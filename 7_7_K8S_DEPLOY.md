# Book 7.7: K8s 零停機部署：Probes、Rolling Update 與 Go 優雅退場

如果說 **7.6** 講的是「流量怎麼進來」，那 **7.7** 要講的就是「Pod 怎麼安全地上線，以及怎麼完美地下線」。

一個真正穩定的大型微服務/遊戲伺服器，它的發版流程必須同時完成三道保險：
1. **上游保護 (Readiness Probe)**：確保新的 Pod 準備好（連上 DB、熱身完畢）才接客。
2. **路由緩衝 (preStop Sleep)**：確保 K8s 網路拓撲更新完畢，不讓新流量走錯路。
3. **優雅退場 (Graceful Shutdown)**：確保舊 Pod 耐心處理完手上的長連線才關機。

---

## 1. Rolling Update：K8s 的零停機換班機制

當你執行 `kubectl apply` 推送新版本時，K8s 不會把所有舊 Pod 一次殺死再一起重啟（那叫 Recreate 策略，會造成全面停機）。預設採用的是 **Rolling Update (滾動更新)**。

### 底層流程
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 0   # 發版期間，最多允許 0 個 Pod 不可用
    maxSurge: 1         # 允許額外多出 1 個超量 Pod
```

以 3 個副本為例，整個換班過程如下：
```
初始狀態:  [v1] [v1] [v1]

Step 1 (Surge):   [v1] [v1] [v1] [v2←啟動中]
Step 2 (Ready):   [v1] [v1] [v1] [v2✅]      ← v2 通過 Readiness Probe
Step 3 (Drain):   [v1] [v1] [v1↗退場中] [v2✅] ← 舊 v1 開始收 SIGTERM
Step 4 (Done):    [v1] [v1] [v2✅]            ← 繼續下一輪

// maxUnavailable: 0 保證了全程至少有 3 個健康的 Pod 在服務
```

**關鍵**：K8s 只有在確認新 Pod 通過了 **Readiness Probe** 後，才會去指令舊 Pod 退場。這就是整個零停機的核心閘道。

---

## 2. 三種探針 (Probes)：Pod 的生命體徵監測

### (1) Readiness Probe：「我準備好接客了嗎？」
這是最重要的探針。只有當 Readiness Probe 回傳成功，K8s 才會把這個 Pod 的 IP 加入 Service 的 Endpoint 清單裡。
**失敗時**：不會重啟 Pod，只是把它從 Endpoints 移除（暫停接客）。

典型使用場景：
- 服務啟動後需要時間連線 DB 或 Redis connection pool
- gRPC 服務需要預熱 (warm-up) 模型或快取
- 正在做 graceful drain，不想接新流量了

```yaml
readinessProbe:
  httpGet:
    path: /readiness   # 你的 Go 服務需要實作這個 HTTP 端點
    port: 8080
  initialDelaySeconds: 5   # Pod 啟動後，等 5 秒才開始探測
  periodSeconds: 5          # 每 5 秒探測一次
  failureThreshold: 3       # 連續失敗 3 次才判定失敗
```

**Go 實作：Readiness Handler**
```go
var isReady atomic.Bool  // 預設 false，啟動時設為 true

func main() {
    // 初始化：連接 DB、Redis、預熱 cache...
    initDB()
    warmUpCache()

    // 一切就緒，宣布 Ready！
    isReady.Store(true)

    http.HandleFunc("/readiness", func(w http.ResponseWriter, r *http.Request) {
        if isReady.Load() {
            w.WriteHeader(http.StatusOK)
        } else {
            w.WriteHeader(http.StatusServiceUnavailable)
        }
    })
}
```

### (2) Liveness Probe：「我還活著嗎？」
當 Liveness Probe 失敗時，K8s 會直接**重啟**這個 Pod。它防止的是那種「程序還在跑，但已經 Deadlock 或進入無限迴圈，永遠不會自行恢復」的殭屍狀態。

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 20
  failureThreshold: 3
```

⚠️ **重要警告**：Liveness Probe 的閾值絕對不能設得太低！
如果你的服務在 GC 停頓或 DB 短暫超時時，很容易讓 Liveness 失敗，它就會把一個「只是暫時忙碌，並沒有死透」的 Pod 強制重啟，這反而會讓問題雪上加霜！

**黃金守則**：
- `Readiness`（問：準備好了嗎？）→ **用來控制流量路由**，可以敏感一些。
- `Liveness`（問：活著嗎？）→ **用來做最後手段重啟**，應該保守且寬鬆。

### (3) Startup Probe：「給我點時間，我還在啟動」
專門保護那些啟動時間極長的服務（例如：JVM、需要大量預熱的 ML 推理服務）。
在 Startup Probe 成功之前，Liveness 和 Readiness Probe 都不會啟動，確保慢啟動的容器不會在還沒完成初始化時就被殺死。

```yaml
startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  failureThreshold: 30     # 允許失敗 30 次
  periodSeconds: 10        # 每 10 秒一次 = 最長允許 300 秒的啟動時間
```

---

## 3. 舊 Pod 的退場：SIGTERM、preStop 與 Graceful Shutdown

當 K8s 決定要關閉一個舊 Pod 時（不管是因為 Rolling Update 還是縮容），它會執行以下極度精密的「送葬儀式」：

```
K8s 決定終止 Pod
       │
       ├──→ 【同時觸發兩件事！】
       │         │                    │
       │   Pod 從 Endpoints 移除  執行 preStop Hook
       │   (停止接收新流量)       (給你一段緩衝時間)
       │         │                    │
       │         └────────┬───────────┘
       │                  ▼
       │        preStop Hook 執行完畢後
       │                  │
       │                  ▼
       │        發送 SIGTERM 給容器主程序
       │                  │
       │                  ▼ (開始计时 terminationGracePeriodSeconds)
       │        Go 程式接收訊號，開始優雅退場
       │        (拒絕新連線、等待進行中的 Request 完成)
       │                  │
       │          如果超過 terminationGracePeriodSeconds
       │                  ↓
       │        強制發送 SIGKILL (立即殺死，不管三七二十一)
```

### 為什麼需要 preStop Sleep？(最容易被忽略的死角)
K8s 從 Endpoints 移除 Pod 和觸發 SIGTERM 是**幾乎同時**的，但「把更新後的 Endpoints 清單傳播到叢集中所有 kube-proxy 節點」需要幾百毫秒的時間。

如果你的 Go 程式在收到 SIGTERM 的瞬間就立刻關閉 Server，那麼在這幾百毫秒的空窗期內，仍然有其他 Pod 或外部 LB 把新的請求發給這個「已經在關閉」的 Pod，導致連線錯誤！

**解法：在 preStop 加一個 sleep**：
```yaml
lifecycle:
  preStop:
    exec:
      command: ["/bin/sleep", "5"]   # 給 K8s 5 秒讓網路拓撲更新完畢

terminationGracePeriodSeconds: 60    # 總超時：5s preStop + 55s 讓進行中 req 完成
```

### Go 的 Graceful Shutdown 完整實作
```go
package main

import (
    "context"
    "log"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"
)

func main() {
    server := &http.Server{Addr: ":8080", Handler: setupRoutes()}

    // 啟動 Server
    go func() {
        if err := server.ListenAndServe(); err != http.ErrServerClosed {
            log.Fatalf("Server error: %v", err)
        }
    }()

    // 監聽 OS 的終止訊號 (SIGTERM 來自 K8s，SIGINT 來自 Ctrl+C)
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGTERM, syscall.SIGINT)
    <-quit  // 阻塞在這裡，直到收到終止訊號

    log.Println("收到 SIGTERM，開始優雅退場...")

    // 1. 把 Readiness 設為 false，讓 K8s 知道不要再給我流量
    isReady.Store(false)

    // 2. 等待進行中的 Request 全部完成，最多等 55 秒
    ctx, cancel := context.WithTimeout(context.Background(), 55*time.Second)
    defer cancel()

    // server.Shutdown 會立刻停止接受新連線，
    // 但會等待已建立的連線完成請求後才關閉
    if err := server.Shutdown(ctx); err != nil {
        log.Fatalf("強制關閉 Server: %v", err)
    }

    // 3. 清理資源 (關閉 DB 連線池、flush buffer...)
    cleanupResources()

    log.Println("Server 已安全關閉，再見！")
}
```

**gRPC 的 Graceful Shutdown 版本**：
```go
// gRPC Server 的優雅退場
grpcServer.GracefulStop()
// GracefulStop() 會停止接受新的 RPC 呼叫，
// 但會等待所有進行中的 RPC Stream 自行完成後才終止
```

---

## 4. 完整生命週期時序圖

```
時間軸 ──────────────────────────────────────────────────►

新 Pod v2 啟動
  │ [initialDelaySeconds]
  │ Startup Probe 通過 ✅
  │ Readiness Probe: /readiness 回傳 503 (DB 還在連接中)
  │ ...熱身中...
  │ Readiness Probe: /readiness 回傳 200 ✅
  │ K8s 把 v2 的 IP 加進 Endpoints ← 開始接收流量！
  │
  │   舊 Pod v1 收到退場指令
  │     ├─ preStop: sleep 5s (等網路拓撲更新)
  │     ├─ 收到 SIGTERM
  │     ├─ isReady = false (Readiness Probe 開始回傳 503)
  │     ├─ server.Shutdown() (停止接新連線，等舊的跑完)
  │     ├─ 所有 gRPC Stream 與 HTTP 請求完成
  │     └─ 安全退場 ✅ (在 terminationGracePeriodSeconds 內完成)
```

---

## 5. 常見的致命錯誤

| 錯誤 | 後果 | 解法 |
|------|------|------|
| 沒有 Readiness Probe | 新 Pod 還沒連上 DB 就開始接客，返回 500 | 實作 `/readiness` 並檢查 DB 連線 |
| 沒有 preStop sleep | 舊 Pod 關閉太快，仍有流量進來，出現連線錯誤 | `preStop: sleep 5` |
| 沒有 Graceful Shutdown | 長連線 (gRPC Streaming / WebSocket) 被暴力中斷 | `server.Shutdown(ctx)` 或 `grpcServer.GracefulStop()` |
| `terminationGracePeriodSeconds` 太短 | 大量進行中的請求來不及完成就被 SIGKILL | 設定為 `preStop時間 + 最長請求時間` |
| Liveness 閾值太低 | GC 停頓或 DB 短暫超時觸發重啟，系統自傷 | Liveness 的 `failureThreshold` 設得比 Readiness 寬鬆 |

**架構師結論**：從「gRPC 慢不慢」切入，到 OS 底層 Socket、DB I/O 瓶頸，最後領悟 K8s 叢集的零停機部署邏輯。這代表你看系統，已經從「點 (程式碼)」、「線 (網路連線)」，進化到了「面 (分散式叢集狀態)」的立體視角！
