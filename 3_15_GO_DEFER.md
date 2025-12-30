# Book 3.15: The Defer (預約執行 - 最後的守門員)

## 1. 第一樂章：概念 (The Concept)

`defer` 是 Go 語言中最獨特也最優雅的關鍵字之一。它的語義很簡單：**「請在這個函數結束前 (Return 之前)，執行這個動作。」**

它通常用於**資源釋放 (Resource Cleanup)**：
*   `Mutex.Unlock()`
*   `File.Close()`
*   `Database.Close()`

### LIFO (後進先出)
`defer` 維護了一個 **Stack (堆疊)** 結構。
*   先宣告的 `defer` (Bottom) 會 **最後執行**。
*   後宣告的 `defer` (Top) 會 **最先執行**。

這非常符合邏輯：如果你先鎖了門 (Lock)，然後開了窗 (Open)；離開時你應該先關窗 (Close)，最後才開鎖 (Unlock)。

---

## 2. 第二樂章：結構 (The Structure)

在 Go Runtime (src/runtime/runtime2.go) 中，`defer` 是由一個 `_defer` 結構體來表示的：

```go
type _defer struct {
    siz     int32       // 參數大小
    started bool        // 是否已經開始執行
    sp      uintptr     // 呼叫者的 Stack Pointer (SP)
    pc      uintptr     // 呼叫者的 Program Counter (PC)
    fn      *funcval    // 要執行的函數
    link    *_defer     // 指向這條 Goroutine 的下一個 defer (鏈表)
    ...
}
```

這條 `link` 鏈表掛在 `g` (Goroutine) 結構上 (`g._defer`)。當函數返回時，Runtime 會沿著這個鏈表依序取出並執行。

### 進化史 (The Evolution of Performance)

早期的 Go (1.12 之前)，`defer` 確實有點慢，因為它涉及到：
1.  **Heap Allocation**: 每次 `defer` 都要去 Heap 申請一個 `_defer` 物件。
2.  **Deferproc**: 需要呼叫 runtime 函數把這個物件掛到鏈表上。

但現在 (Go 1.14+)，`defer` 已經進化了：

#### 1. Stack Allocation (Go 1.13)
編譯器會盡量把 `_defer` 物件分配在 **Stack** 上 (因為函數結束它就沒用了)，避免了昂貴的 Heap Allocation。

#### 2. Open-coded Defer (Go 1.14+) - **零成本優化**
這是最強的優化。
如果你的 `defer` 出現在函數的頂層 (沒有被包在迴圈或複雜條件裡)，編譯器會直接把 `defer` 的程式碼 **插入 (Inline)** 到函數的 `return` 之前。

**這意味著：完全沒有 `_defer` 物件的創建，也沒有 runtime 呼叫。它就跟手寫放在 return 前面的程式碼一樣快！**

---

## 3. 第三樂章：實戰 (The Application - Graceful Shutdown)

這就是您提到的重點：**Shutdown 時如何確保清理乾淨？**

Graceful Shutdown 的核心是：
1.  **監聽訊號**: 收到 `SIGTERM` / `SIGINT`。
2.  **通知**: 透過 `Context` 通知所有正在跑 worker 停下來。
3.  **清理**: Worker 停下來後，執行 `defer` 裡的資源釋放。

### 為什麼 `defer` 在這裡很重要？
因為當我們呼叫 `main` 裡的 `srv.Shutdown()` 時，它會等待所有連線處理完。
而在每個處理連線的 Goroutine 裡，我們通常會寫：

```go
func handleConnection(conn net.Conn) {
    defer conn.Close() // 確保不管發生什麼錯誤，連線都會關閉
    
    // ... 處理業務 ...
}
```

當 Graceful Shutdown 觸發，業務邏輯檢測到 `Running` 狀態變更或 Context Cancelled 而退出函數時，**`defer conn.Close()` 會自動被執行**。
這保證了：
1.  不會有懸掛的 TCP 連線 (Zombie Connections)。
2.  不會有沒釋放的 DB 連線。
3.  不會有沒解開的鎖。

### 範例代碼

```go
func main() {
    // 1. 建立資源
    db := connectDB()
    // 這裡的 defer 只有在 main return 時才會跑。
    // 如果是用 os.Exit() 強制退出，這個 defer 不會跑！
    defer db.Close() 

    // 2. 啟動服務...
    
    // 3. 監聽訊號 (Block until signal)
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit 

    // 4. 開始關機
    log.Println("Shutting down server...")
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    
    if err := srv.Shutdown(ctx); err != nil {
        log.Fatal("Server forced to shutdown:", err)
    }
    
    log.Println("Server exiting")
    // 到這裡 main 函數結束，觸發 defer db.Close()
}
```

**關鍵點**:
*   `os.Exit(1)`: **殺手**。它會直接讓 Process 消失，**所有的 `defer` 都不會執行**。
*   `return` / `panic` (被 recover): 這些情況下 `defer` **一定會執行**。

所以在做 Graceful Shutdown 時，我們傾向於讓 `main` 函數自然結束 (return)，而不是粗暴地呼叫 `os.Exit`，這樣才能保證所有的 `defer` 都有機會出來掃地。

---

## 4. 第四樂章：拉鋸戰 (Panic & Recover)

`defer` 還有一個特殊能力：它是唯一能看見 `panic` 的人。
當程式發生 panic 時，正常的控制流中斷，但 `defer` 鏈表會被逐一喚醒。

```go
func protect() {
    defer func() {
        if r := recover(); r != nil {
            fmt.Println("復活了！因為：", r)
        }
    }()
    
    panic("Crash!")
}
```

這也是為什麼 Middleware (如 Gin/Echo) 的 `Recovery` 中間件原理：
它在最外層 `defer` 了一個 `recover()`，這樣裡面任何 Handler 寫爛了 panic，都會被這個守門員接住，回傳 500 給用戶，而不是讓整個 Server 掛掉。

---

## 5. 終章：總結 (The Finale)

1.  **結構**: Defer 是 LIFO 的鏈表。
2.  **效能**: 現代 Go (Open-coded defer) 幾乎零成本，請放心使用。
3.  **關機**: Graceful Shutdown 依賴 `defer` 來做最後的資源兜底，前提是不要用 `os.Exit()` 殺死自己。
4.  **守門員**: 它是 Panic 和 Crash 之間的最後一道防線。
