# Book 4.3: Rate Limiting (流量控制)

在 **Book 4.2 Worker Pool** 中，我們學會了用緩衝 Channel 來應對流量。但如果流量持續大於 Worker 的處理速度，Channel 終究會滿。與其讓請求在隊列中等到超時，不如在門口就先擋下來。

這就是 **Rate Limiting (限流)** 的核心：以人為定義的速率 (Rate) 來主動拒絕或延遲請求。這就像是在高速公路的匝道口設置紅綠燈 (Ramp Metering)。

---

## 1. 第一樂章：兩大經典算法

在電腦科學中，控制流量主要有兩個流派。它們看似相似，但應對 **突發流量 (Burst)** 的態度截然不同。

### 1.1 漏桶 (Leaky Bucket) - 剛性流出
想像一個底部有破洞的水桶：
*   **流入**：水 (請求) 可以用任意速度倒進去。
*   **流出**：水只能以 **固定速率** 從底部漏出來 (例如每秒 1 滴)。
*   **滿溢**：如果水倒太快，桶子滿了，多餘的水就會溢出 (請求被丟棄)。

**特性**：**平滑 (Smoothing)**。它強制把突發流量「整型」成穩定的細水長流。常用於保護對穩定性要求極高的下游 (例如寫入磁帶機、舊式 Mainframe)。

### 1.2 令牌桶 (Token Bucket) - 彈性流出
想像一個存錢筒：
*   **放入**：系統以固定速率 (例如每秒 10 枚) 往裡面放金幣 (Token)。桶子滿了就不放了。
*   **取出**：請求來的時候，必須拿到一枚金幣才能通過。
*   **突發**：如果桶子裡已經存了 50 枚金幣，你是可以**瞬間**拿走 50 枚的 (允許瞬間併發 50 個請求)。

**特性**：**允許突發 (Allow Burst)**。這更符合網路服務的特性——允許使用者在短時間內發起多個請求，只要平均速率不超標即可。**Go 的標準庫採用此算法。**

---

## 2. 第二樂章：Go 的標準實作 (Token Bucket)

Go 在 `golang.org/x/time/rate` 提供了極致高效的 Token Bucket 實作。

### 2.1 核心概念
```go
// 建立一個 Limiter
// Limit: 每秒產生幾個 Token (速率)
// Burst: 桶子最多能存幾個 Token (容量)
limiter := rate.NewLimiter(rate.Limit(10), 5)
```

這樣設定的意思是：
*   **平時**：每 0.1 秒產生一個 Token。
*   **瞬間**：如果你休息了很久，最多可以累積 5 個 Token，允許你瞬間發 5 個請求。

### 2.2 實戰範例

```go
package main

import (
	"context"
	"fmt"
	"golang.org/x/time/rate"
	"time"
)

func main() {
	// 每秒 5 個 Token (每 200ms 一個)，桶子容量 3
	limiter := rate.NewLimiter(5, 3)

	// 模擬 10 個瞬間請求
	for i := 1; i <= 10; i++ {
		// Wait 會阻塞，直到拿到 Token
		// 如果桶子裡有存貨，這裡不會阻塞
		// 如果桶子空了，這裡會睡 200ms
		err := limiter.Wait(context.Background())
		if err != nil {
			fmt.Println("Error:", err)
			return
		}
		fmt.Printf("Request %d passed at %v\n", i, time.Now().Format("15:04:05.000"))
	}
}
```

**預期輸出解讀**：
```text
Request 1 passed at 10:00:00.000 (消耗存貨 1)
Request 2 passed at 10:00:00.000 (消耗存貨 2)
Request 3 passed at 10:00:00.000 (消耗存貨 3 - 桶空了！)
Request 4 passed at 10:00:00.200 (等待 200ms 新 Token)
Request 5 passed at 10:00:00.400 (等待 200ms 新 Token)
...
```
前 3 個請求瞬間通過，後面的請求被迫排隊，變成了每 0.2 秒通過一個 (Leaky Bucket 的效果)。這就是 Token Bucket **「既允許突發，又宏觀限流」** 的美妙之處。

---

## 3. 第三樂章：底層解密 (Lazy Calculation)

你可能會想：「Go 難道真的開了一個背景 Goroutine，每 200ms 往 channel 裡塞一個 Token 嗎？」

**絕對不是！** 那樣做太消耗資源了。
Go 的 `rate.Limiter` 採用的是 **惰性計算 (Lazy Calculation)**，純粹的數學公式。

### 3.1 數學魔法
它不需要真的儲存 Token，它只需要儲存：
1.  **last**: 上一次有人取走 Token 的時間。
2.  **tokens**: 上一次剩下的 Token 數量。

當你現在 (`now`) 來取 `N` 個 Token 時，它計算：
```go
// 1. 算出距離上次過了多久
delta = now - last

// 2. 算出這些時間理論上「生出了」多少新 Token
newTokens = delta * limit

// 3. 更新目前 Token 數量 (不能超過 Burst 容量)
currentTokens = min(oldTokens + newTokens, burst)

// 4. 夠不夠減？
if currentTokens >= N {
    // 夠：扣除，更新 last = now，放行
} else {
    // 不夠：算出還要等多久，讓 Goroutine 睡覺 (Wait)
}
```

這個演算法是 **O(1)** 的，只有簡單的加減乘除，不需要任何 Timer 或背景 Goroutine，極度高效！這在 OS 領域通常被稱為 **GCRA (Generic Cell Rate Algorithm)** 的一種變體。

### 3.2 為什麼不用 Channel？ (The Anti-Pattern)
初學者常會直覺地用 Channel 來實作 Token Bucket：開一個背景 Goroutine，每秒往 Channel 塞 N 個資料。

```go
// ❌ 錯誤示範：低效的 Channel 實作
func tokenGenerator(tokens chan struct{}) {
    ticker := time.NewTicker(100 * time.Millisecond)
    for range ticker.C {
        select {
        case tokens <- struct{}{}:
        default: // 桶滿了
        }
    }
}
```

**這種做法有三大致命傷**：
1.  **資源浪費 (Resource Heavy)**：如果你要對 1 萬個用戶分別限流，你就得開 1 萬個 Goroutine 和 Ticker！這會吃光你的 CPU 和記憶體。
2.  **無法預支 (No Reservation)**：Channel 只能表達「現在有幾個」，無法表達「我要預借未來的 Token」。
3.  **精確度低**：Go 的 Timer 依賴 P 的調度，在高負載下會不準，導致限流不精確。

相比之下，官方庫的 **Lazy Calculation** 只需要幾個浮點數變數，**0 Goroutine，0 Channel**，這才是系統程式設計的極致。

---

## 4. 第四樂章：與 Worker Pool 的結合

現在我們可以把 Rate Limiter 加到 Worker Pool 的門口了：

```go
func main() {
    jobs := make(chan Job, 100)
    limiter := rate.NewLimiter(100, 20) // 限制整體流量每秒 100

    // Dispatcher
    go func() {
        for {
            job := generateJob()
            
            // 在發送給 Worker 之前，先問 Limiter
            // Allow() 是非阻塞的，如果沒 Token 就回傳 false
            if !limiter.Allow() {
                // 429 Too Many Requests
                fmt.Println("Drop job due to rate limit")
                continue
            }
            
            select {
            case jobs <- job:
                // OK
            default:
                // Channel 還是滿了 (Worker 處理太慢)
                fmt.Println("Drop job due to full queue")
            }
        }
    }()
}
```

這形成了一個 **雙重保護**：
1.  **Rate Limiter**: 控制 **進入速率** (例如防止每秒超過 1000 request)。
2.  **Worker Pool**: 控制 **併發總數** (防止同時開啟超過 50 個 DB 連線)。

這就是高可用系統的黃金組合。

### 4.1 黃金公式：如何設定參數？ (Little's Law)
很多開發者會困惑：「Worker 要開幾個？Rate Limit 要設多少？」這其實是有數學依據的，稱為 **利特爾法則 (Little's Law)**：

$$L = \lambda \times W$$

*   **$\lambda$ (Rate)**: 預期的流量 (Requests/sec)。
*   **$W$ (Wait/Process Time)**: 平均處理一個請求的時間 (sec)。
*   **$L$ (Length/Workers)**: 為了消化流量，系統平均需要的併發 Worker 數。

**實戰應用**：
假設你的 API 需要寫入 DB，平均耗時 **50ms (0.05s)**，你的目標是支撐 **2000 QPS**。
1.  **Rate Limit**: 設定為 `rate.Limit(2000)`。
2.  **Worker Count**: $2000 \times 0.05 = 100$。你至少需要 **100 個 Worker**。
    *   *建議*：開大一點 (例如 120)，作為緩衝以應對偶發的慢查詢。
3.  **Channel Buffer**: 建議設定為 `Rate Limiter` 的 **Burst** 值。
    *   如果允許突發 200 個請求，Channel Buffer 至少要有 200，否則突發進來會被 Channel 阻塞或丟棄，導致限流器的 Burst 設定失效。

**結論**：先測出單個請求的平均耗時 ($W$)，再根據業務目標 ($\lambda$)，就能算出合理的 Worker 數量 ($L$)。

### 4.2 起手式懶人包 (Rule of Thumb)
正常的系統設計邏輯是 **「由下而上」**：先看你的下游 (DB) 能扛多少，再決定你能接多少客。

1.  **設定 Worker Count (物理限制)**:
    *   假設你的 DB 連線池最大只能開 **50 個** 連線 (再多 DB 會掛)。
    *   所以直接設定 **Worker Count = 50**。這是一個硬指標。
2.  **測量平均耗時 (Process Time)**:
    *   透過監控得知 API 平均回應時間為 **50ms (0.05s)**。
3.  **推導 Rate Limit (最大產能)**:
    *   計算：$Rate = \frac{Worker}{Time} = \frac{50}{0.05} = 1000$。
    *   **設定 Rate Limit = 1000**。這就是你的系統極限。
    *   *驗證*：如果超過 1000 QPS 進來，你的 50 個 Worker 根本來不及消化，多餘的請求會在 Buffer 堆積直到超時。所以必須在入口用 Limit = 1000 擋掉。
4.  **設定 Buffer**: **50** (等於 Worker 數，做為對應突發的緩衝)。

**口訣**：**我有 50 個工人，每人花 0.05 秒處理，所以我每秒最多只能接 1000 張單。**

---

## 5. 第五樂章：總結

| 算法 | 對應物理隱喻 | 特性 | Go 應用 |
| :--- | :--- | :--- | :--- |
| **Leaky Bucket** | 底部有洞的水桶 | 強制平滑，無突發 | `time.Tick` 或 Channel 消費端 |
| **Token Bucket** | 存錢筒 | 允許突發，長期平穩 | `x/time/rate`, 網關限流 |

了解了如何控制「流入」，以及如何「並行處理」，最後一塊拼圖就是：**當 Worker 正在跑，但我們想叫他停下來 (例如超時或關閉)**，這時候就不能只靠 Channel 關閉了，而需要更細緻的控制信號。

這就是我們下一章 **Book 4.4: The Context** 的舞台。
