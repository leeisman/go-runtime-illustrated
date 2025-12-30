# Book 4.4: The Context (上下文與傳遞鏈)

在 **Book 4.2 Worker Pool** 和 **Book 4.3 Rate Limiting** 中，我們學會了如何控制併發數與流量速率。但現實世界比理想情況複雜得多：
1.  **使用者斷線**：使用者發起請求後，瀏覽器崩潰或關閉，這時後端還在跑昂貴的 SQL 查詢嗎？
2.  **超時控制**：如果這個任務跑了 10 秒還沒好，我們是否應該強行終止？
3.  **全域參數**：每一個請求都需要攜帶 `TraceID` 或 `UserID`，要怎麼優雅地穿過層層函數？

這就是 **`context.Context`** 誕生的原因。它在 Go 的並發世界中扮演著 **「神經系統」** 的角色，負責傳遞 **訊號 (Signal)** 與 **養分 (Data)**。

---

## 1. 第一樂章：Context 的兩大職責

`Context` 只有兩個核心功能，簡單而強大：

1.  **控制權 (Control)**: 告訴下游「停下來！」。(Cancellation / Timeout)
2.  **資料鏈 (Data Chain)**: 攜帶請求範圍內的鍵值對。(Request-Scoped Values)

### 1.1 Context 介面解析
讓我們看看 `Context` 最原始的定義：
```go
type Context interface {
    // 當工作該結束時，這個 Channel 會被關閉
    // 使用者應該監聽這個 Channel：<-ctx.Done()
    Done() <-chan struct{}

    // 為什麼被取消？(Canceled or DeadlineExceeded)
    Err() error

    // 截止時間 (如果有的話)
    Deadline() (deadline time.Time, ok bool)

    // 取值 (就像從口袋拿東西)
    Value(key any) any
}
```

---

## 2. 第二樂章：生命週期控制 (Cancellation)

這是 Context 最重要的功能。想像 Context 是一棵樹，當**根節點 (Root)** 被砍斷時，所有的**子節點 (Children)** 也會枯萎。

### 2.1 樹狀結構 (The Tree Hierarchy)

*   `context.Background()`: **樹根**。通常在 `main` 或 `http.Handler` 的最上層建立。
*   `context.WithCancel(parent)`: 建立一個可取消的子節點。
*   `context.WithTimeout(parent, duration)`: 建立一個會自動取消的子節點。

### 2.2 實戰：改進 Worker Pool

讓我們回到 **Worker Pool**，看看加上 Context 後會發生什麼事。

```go
package main

import (
	"context"
	"fmt"
	"time"
)

// 模擬一個耗時的工作，但支援 Cancel
func process(ctx context.Context, id int) {
	select {
	case <-time.After(2 * time.Second): // 模擬需要 2 秒的工作
		fmt.Printf("Worker %d: Done\n", id)
	case <-ctx.Done(): // 監聽取消訊號
		// 只要 ctx 一被 cancel，這個 case 馬上會通
		fmt.Printf("Worker %d: Canceled! (%v)\n", id, ctx.Err())
	}
}

func main() {
	// 1. 建立一個 1 秒後會自動超時的 Context
	// 這就像給任務設了一個「定時炸彈」
	ctx, cancel := context.WithTimeout(context.Background(), 1*time.Second)
	
	// 記得 defer cancel，這是良好的習慣，確保資源釋放
	defer cancel() 

	fmt.Println("Main: Starting workers...")

	// 模擬啟動 3 個 Worker
	for i := 1; i <= 3; i++ {
		go process(ctx, i)
	}

	// 主程式等待足夠長的時間來觀察結果
	time.Sleep(3 * time.Second)
	fmt.Println("Main: Exiting")
}
```

**執行結果預測**：
因為我們設定了 `1s` 超時，而工作需要 `2s`。
所以你會看到所有 Worker 輸出：`Canceled! (context deadline exceeded)`。
**這就是韌性 (Resilience)**。我們成功阻止了系統浪費時間去執行已經註定失敗的任務。

---

## 3. 第三樂章：底層原理 (Under the Hood)

Context 是如何做到「父死子亡」的連鎖反應的？

### 3.1 傳播機制 (Propagation)
當你呼叫 `WithCancel(parent)` 時：
1.  Go Runtime 會建立一個 `cancelCtx` 結構體。
2.  如果不只有自己，它會**把自己掛到 Parent 的 `Config.children` 列表裡**。
    *   這是一個雙向連結：Child 持有 Parent，Parent 也持有 Children。
3.  當 Parent 被 `cancel()` 時：
    *   Parent 關閉自己的 `done` channel。
    *   Parent 遍歷 `children` 列表，遞歸呼叫每一個 Child 的 `cancel()`。
    *   **結果**：整棵子樹瞬間收到通知。

### 3.2 效能開銷
*   **建立 Context**: 非常輕量，只是一個 struct 的分配。
*   **監聽 Done**: `<-ctx.Done()` 幾乎無開銷，因為只是在等一個 channel close。
*   **注意**: 雖然輕量，但不要濫用。不要把長期的全局物件 (Global Cache對象) 放在 Context 裡，那不屬於 Request Scope。

---

## 4. 第四樂章：隱藏的陷阱 - Value

`context.WithValue` 允許我們在樹上掛載數據。

```go
ctx := context.WithValue(parent, "userID", 12345)
uid := ctx.Value("userID")
```

### 4.1 它是怎麼找資料的？
它是一個 **Linked List (鏈結串列)**。
*   `ctx.Value` 先看自己有沒有這個 key。
*   如果沒有，就往上找 `Parent.Value`。
*   一直找到根節點 (`Background`) 為止。

### 4.2 致命陷阱
1.  **O(N) 查找**：如果你包了 100 層 Context，查找一個 key 最壞情況要往上跳 100 次。雖然記憶體跳轉很快，但如果在熱點路徑 (Hot Path) 上還是要小心。
2.  **類型不安全**：`Value` 返回 `interface{}`，你必須自己做 Type Assertion (`val.(int)`). 如果斷言錯誤會 Panic (如果不檢查的話)。
3.  **Key 的衝突**：永遠不要用 `string` 作為 Key！
    *   *錯誤*: `ctx.WithValue(ctx, "user", u)` -> 其他 Library 可能也用 "user" 這個字串，導致覆蓋。
    *   *正確*: 用自定義的 struct 或 int 類型作為 Key，確保全域唯一。

```go
type key int
const userKey key = 0

// 封裝 Set
func WithUser(ctx context.Context, u *User) context.Context {
    return context.WithValue(ctx, userKey, u)
}

// 封裝 Get
func GetUser(ctx context.Context) *User {
    if u, ok := ctx.Value(userKey).(*User); ok {
        return u
    }
    return nil
}
```

---

## 5. 總結：使用的黃金法則

1.  **Context 均為第一個參數**：函數簽名裡，`ctx` 永遠放第一個參數 (`func DoExample(ctx context.Context, ...)`).
2.  **不要存在 Struct 裡**：Context 是屬於流程調用的，不應該變成 Struct 的成員變數 (Member Field)。
3.  **寧可傳遞也不要存 Global**：讓 Context 隨著 Call Stack 流動。
4.  **只有 Request-Scoped 的數據才放 Value**：TraceID, UserID, AuthToken 是合適的。DB Connection, Logger 物件通常不建議放進去 (顯式傳遞更好)。

掌握了 Context，你的應用程式就有了**停損點**。這是建構高可用 (High Availability) 系統的基石。

接下來，我們要面對一個更現實的問題：如果下游服務 (如 DB) 真的掛了，我們應該繼續重試嗎？或者應該快速切斷連結以避免災難擴大？
這就是 **Book 4.5: Circuit Breaker (斷路器)** 的課題。
