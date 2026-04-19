# Book 6.11: NATS 與 JetStream：雲原生的神經網路 (NATS & JetStream)

如果說 Kafka 是為了 **大數據日誌 (Log)** 而生，RocketMQ 是為了 **複雜業務 (Business)** 而生。
那麼 **NATS** 就是為了 **現代微服務 (Microservices) 與 實時通訊 (Real-time)** 而生。

它是 **CNCF (Cloud Native Computing Foundation)** 的畢業專案，也是唯一一個 **Go 原生** 的主流 Messaging System。

---

## 1. 第一樂章：Subject-Based Messaging 的哲學 (Flexible Routing)

NATS 與 Kafka/RocketMQ 最大的不同在於：它沒有 **Topic (實體隊列)** 的概念，只有 **Subject (邏輯標籤)**。

### 1.1 Wildcards (通配符)
這是 NATS 的殺手鐧。
*   **`.` (Dot)**: 分隔層級。 e.g. `time.us.east.atlanta`
*   **`*` (Asterisk)**: 匹配單一層級。
*   **`>` (Greater Than)**: 匹配所有剩餘層級 (尾端)。

**應用場景 (遊戲服務)**:
*   **Server 發送**: `game.server1.room101.player_join`
*   **Logging Service 訂閱**: `game.server1.room101.>` (只監控 101 房)
*   **Global Metrics 訂閱**: `game.*.*.player_join` (監控全服玩家登入)

這種 **動態路由能力** 是 Kafka (只有 Partition) 完全做不到的。

---

## 2. 第二樂章：三大通訊模式 (Communication Patterns)

NATS 不僅僅是 Pub/Sub，它還內建了 **Request-Reply**，這讓它能取代 HTTP/gRPC。

### 2.1 Pub-Sub (廣播)
*   **One-to-Many**: 一個 Publisher，多個 Subscriber。
*   **特性**: **Fire and Forget**。如果當下沒人聽，訊息就丟了 (NATS Core 預設不持久化)。
*   **用途**: 緩存失效通知、設定檔更新、遊戲全服廣播。

### 2.2 Queue Groups (負載均衡)
*   **Many-to-One (Scalability)**:
    *   Subscriber 訂閱時指定 `Queue Group Name` (例如 "workers")。
    *   NATS 會自動在群組內的成員間 **隨機輪詢 (Random Load Balance)**。
*   **用途**: 工作佇列 (Job Queue)。這跟 Kafka 的 Consumer Group 很像，但 **不需要 Partition**，隨時增減 Worker。

### 2.3 Request-Reply (偽同步 RPC)
這是 NATS 最獨特的功能。它可以讓異步 MQ 用起來像同步 API。
*   **Client**: `msg, err := nc.Request("help", []byte("data"), 1*time.Second)`
*   **Server**: `nc.Subscribe("help", func(m *Msg) { m.Respond([]byte("OK")) })`
*   **底層原理**: Client 發送時夾帶一個 **Reply-To Subject** (臨時信箱)，Server 處理完後發回那個臨時信箱。
*   **優點**: **Location Transparency**。Client 完全不需要知道 Server 的 IP，也不用 Service Discovery，只要知道 Subject 叫 "help"。

---

## 3. 第三樂章：持久化引擎 (JetStream)

NATS Core 是 **In-Memory** 的 (速度極快但斷電會丟)。
為了對標 Kafka，NATS 推出了 **JetStream**。

### 3.1 Architecture
*   **Stream**: 相當於 Kafka 的 Topic+Log。負責儲存。
*   **Consumer**: 相當於 Kafka 的 Consumer Group + Offset。負責讀取。
*   **Storage**: 支援 **File** (適合生產) 與 **Memory** (適合測試)。

### 3.2 Comparison with Kafka
| Feature | Kafka | NATS JetStream |
| :--- | :--- | :--- |
| **Language** | Java / Scala | **Go** |
| **Dependency** | ZooKeeper / KRaft | **None** (Single Binary) |
| **Distribution** | Partition-based | RAID-1 / Raft-based |
| **Routing** | Topic (Static) | **Subject (Dynamic Wildcards)** |
| **Latency** | ~5ms | **< 1ms (Microseconds)** |
| **Pattern** | Append-only Log | Log, KV Store, Object Store |

---

## 4. 第四樂章：Go 實作範例 (Implementation Example)

NATS 的 Go Client (`nats.go`) 是所有 MQ Client 裡設計最優雅的 (畢竟是同一個爸媽生的)。

### 4.1 Setup & Request-Reply

```go
package main

import (
	"log"
	"time"
	"github.com/nats-io/nats.go"
)

func main() {
	// 1. Connect (Auto Reconnect is robust)
	nc, _ := nats.Connect(nats.DefaultURL)
	defer nc.Close()

	// 2. Server (Responder)
	// 模擬一個 Game Service
	nc.Subscribe("game.login", func(m *nats.Msg) {
		log.Printf("Received: %s", string(m.Data))
		// 處理邏輯...
		m.Respond([]byte("Login Success! Token: xyz"))
	})

	// 3. Client (Requester)
	// 像打 API 一樣呼叫，等待 1秒 timeout
	resp, err := nc.Request("game.login", []byte("User: Frankie"), 1*time.Second)
	if err != nil {
		log.Fatal("Timeout or No Server")
	}

	log.Printf("Response: %s", string(resp.Data))
    // Output: Response: Login Success! Token: xyz
}
```

### 4.2 JetStream (Persistent Push)

```go
func jetstreamExample() {
    nc, _ := nats.Connect(nats.DefaultURL)
    js, _ := nc.JetStream()

    // 1. Define Stream (Storage)
    js.AddStream(&nats.StreamConfig{
        Name:     "ORDERS",
        Subjects: []string{"orders.*"}, // 攔截所有 orders 開頭的 subject 存起來
    })

    // 2. Publish (Async)
    js.Publish("orders.us", []byte("Order #1"))

    // 3. Persistent Subscribe (Push Consumer)
    js.Subscribe("orders.>", func(m *nats.Msg) {
        log.Printf("Processing persistent msg: %s", string(m.Data))
        m.Ack() // 重要：JetStream 需要 Ack
    })
}
```

---

## 5. 第五樂章：總結 (Summary): When to use NATS?

*   **Go Team**: 首選。除錯、維運都與 Go 應用程式無縫接軌。
*   **Internal Microservices**: 取代 gRPC/HTTP 做服務間通訊。享受 **Location Transparency** 和 **Load Balancing**。
*   **Gaming / IoT**: 需要極大連線數 (Client Connections) 與極低延遲。
*   **Edge Computing**: Binary 只有 15MB，可以在樹莓派甚至更小的設備上跑。

**什麼時候不要用？**
*   需要大數據生態整合 (Hadoop/Spark) -> 用 Kafka。
*   需要複雜的分散式事務 (2PC) -> 用 RocketMQ。

---

## 6. 第六樂章：NATS 底層實作揭密 (Internals)
NATS Server (`gnatsd`) 是純 Go 寫成的，它的源碼展現了如何用 Go 寫出 C++ 級別的效能。

### 6.1 The Subject Trie (路由核心)
NATS 如何在微秒內知道 `game.server1.room101.chat` 應該送給哪些 Subscriber (包含 `game.>`, `*.server1.*`...)？
它使用了一個高度優化的 **Radix Tree (Trie)** 結構：
*   **節點 (Node)**: 每一層 Subject (例如 `game`, `server1`) 都是 Tree 上的一個節點。
*   **匹配算法**: 當訊息進來時，它不需要遍歷所有訂閱者。它只需要順著 Tree 往下走，找到所有匹配的葉子節點。
*   **效能**: 時間複雜度與 **訂閱者數量無關**，只與 **Subject 的長度** 有關 (O(k))。這就是為什麼它能支撐千萬級 Topic。

### 6.2 Zero-Allocation Parser (零分配優化)
NATS 的通訊協議是 **純文字 (Text-Based)** 的，類似 HTTP/1.1 或 Redis。
### 6.2 Zero-Allocation Parser (零分配優化)
NATS 的通訊協議是 **純文字 (Text-Based)** 的，類似 HTTP/1.1 或 Redis。
e.g. `PUB subject start_len\r\npayload\r\n`

```go
// 簡化版 NATS Parser 邏輯 (State Machine)
func (c *client) parse(buf []byte) error {
    var i int
    var b byte

    // State Machine Loop (逐字解析)
    for i = 0; i < len(buf); i++ {
        b = buf[i]

        switch c.state {
        case OP_START:
            // 讀取第一個字元 'P' (PUB) 或 'S' (SUB)
            if b == 'P' || b == 'p' {
                c.state = OP_PUB 
            }
        
        case OP_PUB:
            // 讀取 Subject (完全不產生新 String)
            if b == ' ' {
                // 直接使用 Slice 引用，Zero Copy!
                // 注意：這裡的 argStart 是上一輪紀錄的 index
                // c.subject 是一個 []byte，它指向跟 buf 同一塊記憶體
                c.subject = buf[c.argStart:i] 
                c.state = OP_MSG_PAYLOAD
            }
        }
    }
    return nil
}
```

*   **優化技巧**:
    1.  **Slice Reference**: `c.subject = buf[start:i]` 只是 pointer 操作，沒有發生資料拷貝。
    2.  **No `string()`**: 在 Go 裡 `string([]byte)` 會觸發 `malloc` (Escape to Heap)。NATS 內部盡量全程使用 `[]byte` 操作，甚至 Map Key 也是特製的。
    3.  **Sync.Pool**: 讀取 Network Socket 的 `buf` 是從 Pool 裡拿的，用完就還回去，所以幾乎沒有 GC 壓力。

### 6.3 Write Coalescing (Smart Flush 機制)
為了減少 System Call (`write`) 的次數，NATS 採用了一種 **自適應 (Adaptive)** 的 Batch 策略，完全不需要使用者設定參數 (不像 Kafka 需要調 `linger.ms`)。

**核心邏輯**:
1.  **Client Buffer**: 每個連線對象 (Client Connection) 都有一個內部的 Write Buffer。
2.  **Read Loop**: Server 從 Socket 讀到一批資料 (假設包含 100 條 Pub 訊息)。
3.  **Process**: Server 快速解析這 100 條訊息，並將它們塞入對應 Subscriber 的 Write Buffer 中 (**注意：此時只是 Append 到 RAM，還沒發送**)。
4.  **Flush**: 等到這批 Input 資料 **全部處理完**，Server 準備去讀下一批 Input 之前，才會觸發一次真實的 `flush()` (System Call)。

**效果**:
*   **高負載時 (High Load)**: Input 源源不絕，Server 處理時間變長，Write Buffer 自然在這期間累積了大量訊息 -> **一次 System Call 送走大包資料 (High Throughput)**。
*   **低負載時 (Low Load)**: Input 只有 1 條，處理完馬上就閒下來了 -> **馬上 Flush，幾乎零延遲 (Low Latency)**。

這種機制讓 NATS 在吞吐量與延遲之間自動取得了完美的平衡。
