# Book 4.6: Singleflight (防擊穿模式)

在前面的章節中，我們討論的防禦工事（Rate Limit, Worker Pool）都是針對 **「總量」** 的。
但在高併發系統中，有一種此特定的殺手場景：**Hot Key 擊穿 (Cache Stampede)**。

試想：
*   你有一個熱門商品頁面，每秒 5000 次訪問。
*   商品資訊存在 Redis，TTL = 1 分鐘。
*   在 **10:00:00** 這一秒，Cache 剛好過期。
*   在這 0.1 秒內，**500 個請求同時進來**，發現 Cache 沒資料。
*   這 500 個請求 **同時** 穿透 Cache，跑去打 DB。
*   Rate Limit 設了 1000？沒用，因為這 500 個還在限額內。
*   Worker Pool 設了 100？沒用，因為這是一個合法的併發。
*   **結果**：DB 收到 500 個一模一樣的 `SELECT * FROM products WHERE id=1`，瞬間 CPU 飆高掛掉。

我們需要的不是限制，而是 **合併 (Coalescing)**。這就是 **Singleflight**。

---

## 1. 第一樂章：合併飛行

**Singleflight** 的邏輯非常簡單粗暴：
**「只要有一個人正在去 DB 查這筆資料，其他人全部在原地等結果，不要自己去查。」**

### 1.1 流程圖解
1.  **Request A** 進來查 Key="User:1"。
2.  Singleflight 檢查：「現在有人在查 "User:1" 嗎？」
    *   沒有。
    *   **A 去查 DB**。
3.  1 毫秒後，**Request B** 進來查 Key="User:1"。
4.  Singleflight 檢查：「現在有人在查 "User:1" 嗎？」
    *   有！Request A 正在查。
    *   **B 不去 DB，直接 `wg.Wait()` 等 A 的結果**。
5.  **Request C, D, E...** 進來，全部停下來等 A。
6.  **A 查回來了！**
    *   A 拿到結果。
    *   Singleflight 把這個結果同時複製給 B, C, D, E。
    *   大家一起開心返回。

**效果**：原本 500 次 DB 查詢，瞬間變成 **1 次**。DB 負載降低 500 倍。

---

## 2. 第二樂章：Go 實作範例

Go 官方在擴展庫 `golang.org/x/sync/singleflight` 提供了現成的實作。

```go
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
	"time"

	"golang.org/x/sync/singleflight"
)

// 模擬 DB 查詢
var dbCallCount int32

func getDataFromDB(key string) (string, error) {
	fmt.Printf("[DB] Querying %s...\n", key)
	atomic.AddInt32(&dbCallCount, 1)
	time.Sleep(100 * time.Millisecond) // 模擬慢查詢
	return "db_value_" + key, nil
}

func main() {
	var g singleflight.Group
	var wg sync.WaitGroup

	// 模擬 10 個並發請求同時查同一個 Key
	key := "hot_item_123"

	for i := 0; i < 10; i++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()
			
			// Do 的第一個參數是 Key (去重依據)
			// 第二個參數是真正的邏輯 (只會被執行一次)
			v, err, shared := g.Do(key, func() (interface{}, error) {
				return getDataFromDB(key)
			})

			if err != nil {
				fmt.Println("Error:", err)
				return
			}
			
			// shared 為 true 表示這個結果是「蹭」別人的
			fmt.Printf("Req %d: got %v (shared=%v)\n", id, v, shared)
		}(i)
	}

	wg.Wait()
	fmt.Printf("Done. Total DB calls: %d\n", dbCallCount)
}
```

**預期輸出**：
```text
[DB] Querying hot_item_123...  <-- 只出現一次！
Req 0: got db_value_hot_item_123 (shared=true)
Req 9: got db_value_hot_item_123 (shared=true)
...
Done. Total DB calls: 1
```

---

## 3. 第三樂章：底層原理 (Map + WaitGroup)

這聽起來很神奇，其實底層就是 **Map + WaitGroup** 的完美應用。讓我們手刻一個簡化版：

```go
type call struct {
    wg  sync.WaitGroup
    val interface{}
    err error
}

type Group struct {
    mu sync.Mutex       // 保護 m
    m  map[string]*call // 紀錄正在飛行的請求
}

func (g *Group) Do(key string, fn func() (interface{}, error)) (interface{}, error) {
    g.mu.Lock()
    if g.m == nil {
        g.m = make(map[string]*call)
    }
    
    // 1. 檢查是否已經有人在飛
    if c, ok := g.m[key]; ok {
        g.mu.Unlock()
        c.wg.Wait() // 有人飛，我就等 (阻塞)
        return c.val, c.err
    }
    
    // 2. 沒人飛，我來飛
    c := new(call)
    c.wg.Add(1)
    g.m[key] = c
    g.mu.Unlock() // 釋放鎖，讓 fn 可以慢悠悠執行，不卡其他人
    
    // 3. 執行真正的邏輯
    c.val, c.err = fn()
    c.wg.Done() // 完成，喚醒所有等待者
    
    g.mu.Lock()
    delete(g.m, key) // 刪除 Key，下次請求要重新飛
    g.mu.Unlock()
    
    return c.val, c.err
}
```

### 3.1 關鍵細節
注意看 `g.mu.Unlock()` 的位置！
*   它只鎖住了 **Map 的檢查與寫入** (極快)。
*   **真正的 fn() 執行期間是沒有鎖的**。
*   這保證了 Singleflight 不會變成由鎖導致的瓶頸。

---

## 4. 第四樂章：隱藏的風險 (Panic)

雖然 Singleflight 很好用，但它有一個致命風險：
**如果 fn() 卡死 (Hang)，所有等待者 (`wg.Wait()`) 也會一起陪葬 (Goroutine Leak)。**

**解決方案**：
總是結合 **Timeout** 使用。
官方庫提供了 `DoChan` 方法，回傳一個 channel，讓你可以用 `select` 監聽超時。

```go
ch := g.DoChan(key, fn)
select {
case <-ctx.Done():
    // 請求超時，我不想等了 (但原本那個還在跑的請求無法被這裡 cancel，這是 Singleflight 的特性)
    return ctx.Err()
case res := <-ch:
    return res.Val, res.Err
}
```

### 4.1 決策指南：Do vs DoChan
應該用哪一個？這取決於你對 **「命運共同體」** 的接受度。

1.  **`g.Do` (同步/阻塞)**
    *   **機制**：所有 Goroutine 卡在 `wg.Wait()`，直到 Leader 跑完或報錯。
    *   **結果**：**同生共死**。如果不幸 Leader 跑了 10 秒，大家都要等 10 秒。如果 Leader 報錯，大家拿到同一個 Error。
    *   **適用**：大部份場景。只要 `fn()` 內部有設定合理的 Timeout (例如 DB Query Timeout)，用 `Do` 最簡單且高效 (少一次 Channel 傳遞開銷)。

2.  **`g.DoChan` (非同步/Channel)**
    *   **機制**：回傳一個 Channel，讓你用 `select` 監聽。
    *   **結果**：**各自飛**。如果 Leader 卡住，Follower A 可以因為自己的 Context 1秒超時而先走 (Abandon)，回傳 Timeout 給用戶，不用陪葬。
    *   **適用**：對延遲極度敏感 (Latency Sensitive) 的 API。例如即時競價系統，規定 50ms 沒拿到結果就要放棄，這時候必須用 `DoChan` 搭配 `select`。

**注意**：`Singleflight` 的設計使得「領頭羊」無法被輕易 Cancel，因為它肩負著後面 999 人等待的責任。如果你強行 Cancel 領頭羊，後面的人可能永遠等不到結果。這是使用時需權衡的點。

---

## 5. 第五樂章：總結

**Book 4.6 Singleflight** 補上了我們在「資料熱點」防護上的最後一塊拼圖。

| 模式 | 解決什麼問題？ | 核心手段 |
| :--- | :--- | :--- |
| **Worker Pool** | 資源耗盡 | 限制並發數 |
| **Rate Limit** | 流量過載 | 限制速率 |
| **Circuit Breaker** | 下游死亡 | 快速失敗 |
| **Singleflight** | **Cache 擊穿** | **請求合併** |

現在，無論是海嘯般的總體流量，還是針對單點的狙擊流量，你的系統都有了應對之道。

但還有一種更陰險的攻擊：**惡意用戶故意查詢一堆「不存在」的 Key**。
*   Singleflight 救不了你，因為每個 Key 都不一樣，無法合併。
*   Rate Limit 救不了你，因為攻擊者可以用分散式 IP。
*   結果這堆無效請求會直接「穿透」Cache，把 DB 打殘。

這就是 **Book 4.7: Cache Penetration (防穿透模式)** 的戰場。我們將召喚機率學的神器 —— **Bloom Filter (布隆過濾器)**。
