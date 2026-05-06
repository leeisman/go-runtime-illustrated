# Book 7.9: K8s 裡的 gRPC 負載不平衡陷阱 (The gRPC Load Balancing Trap)

在 Kubernetes 中，我們習慣依賴 `Service` (ClusterIP) 來幫我們做負載平衡。當一個 Service 後面掛了多個 Pod 時，我們理所當然地認為流量會平均分散。

但當你使用 **gRPC** (底層為 HTTP/2) 從一個 Pod 透過 Service 打向另一個微服務時，你很可能會遇到一個可怕的災難：「**一旦連線建立，永遠只有其中一個 Pod 在狂接流量，其他 Pod 全部在閒置。**」

本章將從網路封包 (Header) 的底層原理解開這個謎團。

---

## 1. 第一樂章：L4 vs L7 的維度打擊 (The Dimension Gap)

要理解這個問題，必須先看懂 Kubernetes Service 與 gRPC 所處的「網路維度」完全不同。

### K8s Service (kube-proxy) 活在 L4 (傳輸層)
kube-proxy 底層是透過 `iptables` 或 `IPVS` 實作的。它只看得懂 TCP/UDP 的封包 Header。
它做負載平衡的最小單位是 **「一條 TCP 連線 (Connection)」**。

### gRPC (HTTP/2) 活在 L7 (應用層)
gRPC 的底層是 HTTP/2，而 HTTP/2 的核心特徵是 **連線多工 (Connection Multiplexing)**。
為了極致效能，gRPC 客戶端只會建立 **「唯一一條 TCP 長連線」**，然後把成千上萬個並發的 RPC 請求 (Request)，包裝成 HTTP/2 的 Frame，全部塞進這同一條 TCP 水管裡送出去。

---

## 2. 第二樂章：封包的奇幻漂流 (The Packet Journey)

讓我們模擬一次 Pod A (10.0.1.5) 透過 Service B (ClusterIP: 10.96.0.100) 呼叫 Pod B1 (10.0.2.10) 的完整封包漂流：

### 🚚 第一階段：建立 TCP 連線 (kube-proxy 的唯一舞台)
1.  **發送 TCP SYN**：Pod A 發起 TCP 握手。
    `發送封包：[Src: 10.0.1.5 | Dst: 10.96.0.100 | Payload: TCP SYN]`
2.  **kube-proxy 攔截與 DNAT**：Linux 內核的 netfilter 攔截到目的地為 ClusterIP 的封包。它根據負載平衡策略選中 Pod B1，並寫入 **conntrack (連線追蹤表)**，記錄下這條連線的死規定。
    `轉換後：[Src: 10.0.1.5 | Dst: 10.0.2.10 | Payload: TCP SYN]`
3.  **連線確立**：TCP 三方交握完成，Pod A 與 Pod B1 之間建起了一條堅固的 TCP 長連線。

### 🚚 第二階段：資料傳輸 (gRPC 與 HTTP/2 封包)
TCP 水管接好之後，gRPC 開始在這條水管裡面瘋狂丟 HTTP/2 的資料。

1.  **發送 gRPC 請求**：Pod A 上的業務邏輯發起呼叫，將 Request 序列化成 Protobuf，包裝成 HTTP/2 的 DATA Frame，然後塞進 TCP 封包送出去。
    `發送封包：[Src: 10.0.1.5 | Dst: 10.96.0.100 | Payload: HTTP/2 Frame (etcd.Get)]`
2.  **持續的 DNAT 轉換 (災難發生)**：因為 TCP 連線已經建立，Linux 內核的 `conntrack` 表裡面已經有這條連線的死規定。接下來的「每一個封包」，只要經過 Node 網路層，都會**瞬間被自動轉換，kube-proxy 不會再重新做負載平衡決策**：
    `轉換後：[Src: 10.0.1.5 | Dst: 10.0.2.10 | Payload: HTTP/2 Frame]`
3.  **Pod B 處理並回傳**：Pod B1 算完結果，回傳 HTTP/2 DATA Frame。同樣經過 `conntrack` 復原 IP，最後送回 Pod A。

**殘酷的結果**：因為 HTTP/2 的連線多工特性，後續的 10,000 個並發請求全都在這「同一條 TCP 水管」裡送出。kube-proxy 看不懂 HTTP/2 封包，也無法把水管中途切斷分給 B2，所有的流量都只會精準地打在 Pod B1 身上，Pod B2 永遠 0 流量。

---

## 3. 第三樂章：破局之道 (The Solutions)

為了解決這個 L4 無法處理 L7 負載平衡的硬傷，架構師通常有兩條路可走：

### 解決方案 A：客戶端負載平衡 (Client-Side Load Balancing)
這是效能最高、最原生的解法。捨棄 K8s 的 ClusterIP，改用 **Headless Service (`ClusterIP: None`)**。
1.  **DNS 解析**：當 Pod A 查詢這個 Service 的 DNS 時，K8s DNS 不再回傳單一虛擬 IP，而是直接回傳 Pod B1, Pod B2 的所有真實 IP 列表。
2.  **gRPC 內建 LB**：在 Pod A 初始化 gRPC Client 時，使用 `dns:///` resolver 並指定 `round_robin` 策略。
3.  **行為改變**：gRPC Client 會自己去跟 B1, B2 各建立一條 TCP 長連線，然後**在應用程式內部**把 10,000 個 HTTP/2 Request 平均分發到這兩條水管上。

**Golang 實作範例**：
```go
import (
	"log"
	
	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
	// 必須匿名引入，以註冊底層的 dns resolver
	_ "google.golang.org/grpc/resolver"
)

func main() {
	// 1. 使用 dns:/// 開頭，告訴 gRPC 透過 DNS 解析所有 Pod 的真實 IP
	// 2. target 是你的 Headless Service 名稱與 Port
	target := "dns:///my-service.default.svc.cluster.local:50051"

	// 3. 配置 DefaultServiceConfig，強制指定 loadBalancingPolicy 為 round_robin
	conn, err := grpc.Dial(
		target,
		grpc.WithTransportCredentials(insecure.NewCredentials()),
		grpc.WithDefaultServiceConfig(`{"loadBalancingPolicy":"round_robin"}`),
	)
	if err != nil {
		log.Fatalf("did not connect: %v", err)
	}
	defer conn.Close()

	// 接下來的客戶端呼叫，gRPC 會自動在底層建立好的多條 TCP 連線上進行 Round-Robin 分發
}
```

### 解決方案 B：Service Mesh (L7 代理)
引入如 Istio 或 Linkerd 這樣的 Service Mesh。
1.  **Envoy Sidecar**：在 Pod A 和 Pod B 旁邊都掛一個 Envoy 代理。
2.  **L7 解析**：Envoy 看得懂 HTTP/2。當 Pod A 送出 gRPC 請求時，Envoy 會截斷連線，把 HTTP/2 的 Frame 拆解開來，並且基於「單一 Request」的維度，分別路由給 B1 和 B2。
3.  **代價**：雖然解決了負載不均，但封包多繞了兩層 Proxy，會增加網路延遲 (Latency) 與 CPU 消耗。

---

## 4. 第四樂章：總結 - 選擇適合的武器 (Finale)

**一句話總結**：K8s 原生的 Service 只能做 L4 (TCP 連線) 級別的平衡，面對 gRPC (HTTP/2) 「一條連線多工所有請求」的特性會直接破功。追求極限低延遲請用 Headless Service (Client-Side LB)，追求運維治理方便則選用 Service Mesh。
