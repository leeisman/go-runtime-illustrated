# Book 8.2: 抓漏神器 (Goroutine Leak & Pprof Profiling)

在 Go 的世界裡，因為有強大的 Tracing GC 存在，傳統 C/C++ 那種「忘記 free() 導致的無主靈魂」已經被徹底消滅。
然而，Go 工程師面臨的是更棘手的敵人：**「邏輯性的記憶體洩漏 (Logical Memory Leak)」**。

因為這類洩漏在語法上完全合法，**純靠 Code Review 幾乎抓不到**。我們必須仰賴「測試期」與「生產期」的專屬探測儀器。

---

## 1. 第一層防禦：測試期的守門員 (`uber-go/goleak`)

超過 90% 的 OOM (Out Of Memory) 事件，凶手都不是單純的變數，而是那些**「永遠等不到結局而死當的 Goroutine」**。它不僅霸佔著 2KB 的 Stack，更挾持了它觸及的所有 Heap 變數。

### 痛點：
在單元測試 (Unit Test) 中，就算 Goroutine 洩漏了，主程式 `t.Run` 測完就關閉了，測試結果依舊會顯示 `PASS`，地雷就這樣被埋進了 Production 區。

### 解法：`goleak`
Uber 開源的 `goleak` 套件能強制盤點系統內的活躍執行緒數量。

```go
import (
    "testing"
    "go.uber.org/goleak"
)

func TestMyActorLogic(t *testing.T) {
    // 魔法指令：在測試束結時，嚴格盤點是否還有 Goroutine 殘留！
    defer goleak.VerifyNone(t)

    // 模擬你的業務邏輯
    ch := make(chan int)
    go func() {
        <-ch // 模擬忘記放資料導致永遠阻塞
    }()
    
    // 測試的主邏輯驗證...
}
```
**執行結果**：測試會無情地 **FAIL**，並直接印出 `goroutine 123 [chan receive]: ...` 告訴你哪一行的 Goroutine 變成了幽靈。把地雷直接攔截在 CI/CD 階段！

---

## 2. 第二層防禦：Go 界的四大記憶體殺手 (Common Leaks)

要在源頭避免 Leak，開發者必須在腦海裡對這「四大天王」有極高的警覺心：

### 1. 挾持大陣列的切片 (Slice Sub-slicing Leak)
*   **情境**：讀取一個 1GB 的超級大陣列，並且只 `slice[0:2]` 截取前 2 個 Bytes 想當作檔頭保存進 Cache 裡。
*   **底層真面目**：Go 切片截取 **不會複製記憶體**。這 2 個 Bytes 的切片檔頭內部的指標，依然指著那 1GB 的大陣列開頭。導致 GC 為了這 2 Bytes，被迫保留 1GB 垃圾。
*   **解方**：需要長久保留的小切片，強制使用 `copy()` 或 Go 1.18 提供的 `strings.Clone()` / `bytes.Clone()` 強烈割離關係。

### 2. 忘了關燈的定時器 (`time.Ticker` Leak)
*   **情境**：(在 Go 1.23 以前) 使用 `time.NewTicker()` 或 `time.Tick()` 產生心跳，但 Goroutine 結束時忘記呼叫 `.Stop()`。
*   **底層真面目**：Ticker 創建時會將自己註冊進底層的 **全域 Timer Heap**。全域指針會永遠挾持著這個 Ticker 與其附帶的 Channel，成為永不回收的 GC Root。
*   **解方**：宣告時無腦配對 `defer ticker.Stop()`。

### 3. 未關閉的連線緩衝區 (`http.Response.Body`)
*   **情境**：執行 `http.Get()` 取得了 API 回應，讀取完 JSON 後就讓函數結束了。
*   **底層真面目**：底層的 `http.Transport` 連線池為了 Keep-Alive，會繼續維持這個 TCP 連線。如果沒有呼叫 `.Close()`，那這個連線專屬的 32KB 讀寫緩衝區會一直駐留在記憶體中，且 FD (File Descriptor) 資源會被耗盡。
*   **解方**：`defer resp.Body.Close()` 是任何網路操作的鐵律。

### 4. 忘記設 TTL 的全域黑洞 (Global Map Without Eviction)
*   **情境**：將使用者的連線物件 `*Client` 存入自定義的 `map[string]*Client`。使用者斷線後，只關了 Socket 卻忘了從 map 移除。
*   **底層真面目**：全域變數是最高權威，GC 看到 map 裡面有紀錄，就絕對不會碰它。
*   **解方**：導入帶有 TTL (過期時間) 或 LRU (最少使用淘汰) 的第三方 Cache 套件。

---

## 3. 終極防禦：生產環境的 X 光機 (`net/http/pprof`)

如果在生產環境 (Production) 監控發現機器的 RAM 從 100MB 緩慢爬升到 8GB 準備 OOM，不要靠靈感瞎猜，請直接祭出核武器 **`pprof`**。

### 如何埋入 X 光探測器？
在你的 `main.go` 加上這短短兩行，系統就會自動開啟一個偵錯專用的 HTTP Server 監控記憶體。
```go
import _ "net/http/pprof" // 引入 pprof (底層會自動註冊 /debug/pprof 路由)

func main() {
    go func() {
        // 開啟在本地端的 6060 port 供工程師查哨
        log.Println(http.ListenAndServe("localhost:6060", nil))
    }()
    // ... 原本的業務邏輯 ...
}
```

### 緊急看診三大指令！
當伺服器記憶體飆高時，在你的終端機打出以下指令：

1. **抓卡死的 Goroutine (查內鬼)**:
   打開瀏覽器輸入 `http://localhost:6060/debug/pprof/goroutine?debug=1`
   它會把當前伺服器中所有的 Goroutine 列出來，你可以一秒鐘看出來：「天阿，為什麼有 50 萬隻 Goroutine 全部卡在 `actor.go:Line44` 的某個 channel 等待？」

2. **抓肥大的記憶體怪獸 (Heap Profiling)**:
   ```bash
   # 這會在指令列開啟互動模式，抓取這瞬間的記憶體快照
   go tool pprof http://localhost:6060/debug/pprof/heap
   ```
   進去後輸入 `top` 指令，它會從第一名排到第十名，極度無情地告訴你哪一個函數、哪一行代碼，挾持了最巨大的 Heap 記憶體空間！
   
3. **網頁視覺化火焰圖 (Web UI)**:
   如果你覺得純文字不好看，可以加上 `-http` 參數：
   ```bash
   go tool pprof -http=:8080 http://localhost:6060/debug/pprof/heap
   ```
   這會自動打開網頁，為你繪製一張**「火焰圖 (Flame Graph)」**。圖上的方塊越寬，代表該模組佔用的記憶體越大。你可以像點地圖一樣，一路點擊最寬的方塊，直到找到深藏在依賴庫底層的記憶體洩漏源頭。

### 總結
優秀的系統架構，並不追求寫出「100% 完美」的代碼（因為不可能）。
而是建構一套 **事前能用 `goleak` 擋下、事後能用 `pprof` 精準剖析** 的工程防禦網！
