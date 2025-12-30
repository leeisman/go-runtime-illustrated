# Book 4.8: ErrGroup (併發聚合模式)

在現代微服務架構中，一個前端請求 (例如「獲取個人首頁」) 往往需要後端同時聚合多個服務的資料：
*   **User Service**: 拿頭像、暱稱。
*   **Order Service**: 拿最近訂單。
*   **Review Service**: 拿待評價商品。
*   **Coin Service**: 拿餘額。

為了效能，我們希望這 4 個請求是 **並行 (Parallel)** 的，而不是串行。
但這帶來了複雜的錯誤處理問題：
1.  **Fail Fast**: 如果 User Service 掛了，我不想等其他 3 個跑完，我想立刻返回錯誤。
2.  **Context Cancel**: 一旦決定要返回錯誤，我希望把其他正在跑的請求也 Cancel 掉，節省資源。
3.  **Error Collection**: 手寫 `WaitGroup` 很難優雅地收集這 4 個可能發生的錯誤。

這就是官方擴展庫 `golang.org/x/sync/errgroup` 的用武之地。它是 **WaitGroup + Context + Error Propagation** 的完美封裝。

---

## 1. 第一樂章：從 WaitGroup 的痛點說起

如果你堅持用 `sync.WaitGroup` 手刻：

```go
// ⛔️ 痛苦的手刻方式
func fetchDashboard(ctx context.Context) error {
    var wg sync.WaitGroup
    var errOnce sync.Once // 只紀錄第一個錯誤
    var firstErr error
    
    // 建立一個可取消的 Context
    ctx, cancel := context.WithCancel(ctx)
    defer cancel()

    // 任務 1
    wg.Add(1)
    go func() {
        defer wg.Done()
        if err := fetchUser(ctx); err != nil {
            errOnce.Do(func() {
                firstErr = err
                cancel() // 只要有人錯，就 Cancel 全場
            })
        }
    }()

    // 任務 2 ... (重複上面那一坨代碼)
    // 任務 3 ... (再重複)
    
    wg.Wait()
    return firstErr
}
```
你會發現代碼非常冗長、容易寫錯 (忘記 cancel, race condition 等)。

---

## 2. 第二樂章：ErrGroup 的優雅

用 `errgroup` 重寫同樣的邏輯：

```go
package main

import (
	"context"
	"fmt"
	"time"
	"golang.org/x/sync/errgroup"
)

func main() {
	// 建立一個 Group 和一個綁定的 Context
	// 只要 Group 裡任何一個任務回傳 error，這個 ctx 就會被 Cancel
	g, ctx := errgroup.WithContext(context.Background())

	// 任務 1: User Service (模擬成功)
	g.Go(func() error {
		// 模擬耗時，必須監聽 ctx.Done()
		select {
		case <-time.After(1 * time.Second):
			fmt.Println("User Service Done")
			return nil
		case <-ctx.Done():
			fmt.Println("User Service Canceled")
			return ctx.Err()
		}
	})

	// 任務 2: Order Service (模擬失敗)
	g.Go(func() error {
		select {
		case <-time.After(500 * time.Millisecond):
			return fmt.Errorf("order service failed")
		case <-ctx.Done():
			return ctx.Err()
		}
	})

	// 任務 3: Coin Service (模擬慢速，會被 Cancel)
	g.Go(func() error {
		select {
		case <-time.After(2 * time.Second): // 這個任務最慢
			fmt.Println("Coin Service Done")
			return nil
		case <-ctx.Done():
			fmt.Println("Coin Service Canceled (Fail Fast)")
			return ctx.Err()
		}
	})

	// 等待所有任務結束
	if err := g.Wait(); err != nil {
		fmt.Printf("Get Error: %v\n", err)
	} else {
		fmt.Println("All Success")
	}
}
```

**輸出結果**：
```text
User Service Done (剛好 1s 做完)
Coin Service Canceled (Fail Fast)  <-- 被 Order 的失敗連累，直接 Cancel
Get Error: order service failed
```

### 2.1 核心特性
1.  **自動 Context 傳遞**: `g.Go(func() error)` 內部的逻辑共享同一個 `ctx`。
2.  **Fail Fast**: 只要有一個 `Go` 回傳 error，`ctx` 立刻被 Cancel，通知其他兄弟任務。
3.  **Wait**: `g.Wait()` 會等待所有 Goroutine 結束（這很重要，防止 Goroutine Leak），並回傳 **第一個** 發生的錯誤。

---

## 3. 第三樂章：限制併發數 (Pipeline Control)

如果你有 1000 個任務要跑（例如爬蟲），直接 `g.Go` 1000 次雖然方便，但會瞬間炸開 1000 個 Goroutine。
`errgroup` 從 Go 1.20 開始支援 **SetLimit**:

```go
g, ctx := errgroup.WithContext(context.Background())
g.SetLimit(10) // 限制同時最多只有 10 個 Goroutine 在跑

for i := 0; i < 1000; i++ {
    i := i
    g.Go(func() error {
        return process(ctx, i)
    })
}
```

### 3.1 為什麼要由 Group 管理 Goroutine?
自己開 `go func()` 雖然快，但如果在 func 內部發生 `panic`，整個程式會直接崩潰。
`errgroup` 的 `Go` 方法內部其實做了一層 **Recover** 保護 (雖然官方版比較陽春，但通常企業級框架會再包一層)，並且保證了 `wg.Done()` 一定會被執行，避免死鎖。

---

## 4. 第四樂章：源碼解密 (Under the Hood)

`ErrGroup` 的源碼非常精簡，它展示了如何將基礎的並發原語組合出強大的功能。
核心結構如下 (簡化版)：

```go
type Group struct {
    cancel func()     // 用來通知大家停下來
    wg     sync.WaitGroup
    
    errOnce sync.Once // 保證只記錄第一個錯誤
    err     error     // 儲存第一個錯誤
    
    sem chan struct{} // 用於 SetLimit 的信號量
}
```

### 4.1 關鍵機制：只記錄第一個錯
為什麼要用 `sync.Once`？

```go
func (g *Group) Go(f func() error) {
    g.wg.Add(1)
    go func() {
        defer g.wg.Done()
        
        if err := f(); err != nil {
            // 只有第一個搶到鎖的人，才有資格寫入 g.err
            // 並且觸發 cancel()
            g.errOnce.Do(func() {
                g.err = err
                if g.cancel != nil {
                    g.cancel()
                }
            })
        }
    }()
}
```
**設計哲學**：在並發聚合中，通常我們只關心 **「導致失敗的那個元兇」**。一旦失敗已經發生 (Context Canceled)，後續產生的 `context.Canceled` 錯誤都是雜訊，不值得記錄。

### 4.2 關鍵機制：並發限制
`SetLimit(n)` 其實就是初始化了一個 Buffer Channel：
```go
func (g *Group) tryGo(f func() error) bool {
    if g.sem != nil {
        select {
        case g.sem <- struct{}{}: // 嘗試佔坑
            // 成功佔到坑，繼續執行
        default:
            return false // 沒坑了，回傳 false
        }
    }
    // ... 啟動 goroutine ...
    // Goroutine 結束時 <-g.sem (釋放坑)
}
```
這完全就是我們在 **Book 4.2 Worker Pool** 學到的概念！

### 4.3 殘酷現實：Rollback 怎麼辦？
**注意：ErrGroup 只能 Cancel (停止) 正在跑的任務，無法 Rollback (復原) 已經完成的任務。**

舉例：
1.  Task A (扣款): **成功**。
2.  Task B (發貨): **失敗**。
3.  **結果**: ErrGroup 會回傳 B 的錯誤。但 A 已經扣款成功了！使用者的錢沒了，貨也沒收到。

**解法**：這已經超出了 Go 語言層面的範疇，屬於 **分散式事務 (Distributed Transaction)** 的領域。
你需要引入 **Saga Pattern**：
*   為每個操作定義一個 **補償函數 (Compensate)**。
*   (Action: 扣款) -> (Compensate: 退款)。
*   當 `g.Wait()` 回傳錯誤時，你的主程式必須去檢查哪些任務成功了，並呼叫對應的補償函數把它「修」回來。

**結論**：`ErrGroup` 最適合 **Read-Only** 的聚合查詢 (Fan-Out Query)。如果是涉及多個 **Write** 操作，請務必三思。

---

## 5. 總結

`errgroup` 是 Go 併發模式中的 **「聚合器 (Aggregator)」**。
它解決了微服務中最常見的「多對一」呼叫場景。

*   **WaitGroup**: 只有同步等待，無錯誤處理。
*   **ErrGroup**: 同步等待 + 錯誤傳播 + 上下文取消 + 並發限制。

現在，我們已經掌握了無序併發 (Worker Pool) 和聚合並發 (ErrGroup)。
最後，我們還剩下一個挑戰：**如何讓並發處理是有順序的？** (例如處理 Binlog 或即時對戰訊息)。

這就是 **Book 4.9: Sharding (有序分片)** 的課題。
