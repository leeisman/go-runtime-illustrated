# Book 4.5: Circuit Breaker (斷路器模式)

在前面的章節中，我們學會了：
*   **Worker Pool**: 限制同時處理的量。
*   **Rate Limiter**: 限制進入的速率。
*   **Context**: 限制單個任務的時間。

但如果下游服務（例如 Database 或 Payment Gateway）**徹底掛了**，這些機制還不夠完美。
*   Worker Pool 會被滿滿的「卡住的任務」塞爆。
*   Rate Limiter 雖然限制了速率，但每秒 1000 個請求進去，還是會產生 1000 個 Error，浪費你的 CPU 和下游的頻寬。

這時候，我們需要一個 **總開關 (Master Switch)**，在下游故障時直接切斷通路，讓系統進入「降級模式 (Degraded Mode)」。這就是 **Circuit Breaker (斷路器)**。

---

## 1. 第一樂章：電路隱喻

Circuit Breaker 的概念來自家庭電路。當電流過載（短路）時，保險絲會熔斷（或是開關跳開），防止電線走火。

在微服務中，它是一個 **狀態機 (State Machine)**，有三種狀態：

1.  **Closed (閉合/通路)**：
    *   **正常狀態**。電流（請求）可以流通。
    *   斷路器會計數失敗次數。如果失敗率超過閥值（例如 50%），就會跳閘，轉為 **Open**。

2.  **Open (斷開/斷路)**：
    *   **故障狀態**。主要開關切斷。
    *   **所有請求直接失敗 (Fail Fast)**，不會真的去呼叫下游。這樣可以保護下游不被重試流量打死，也讓上游不用等待超時。
    *   經過一段「冷靜期 (Sleep Window)」後，會自動轉為 **Half-Open**。

3.  **Half-Open (半開/測試)**：
    *   **測試狀態**。
    *   系統會放行 **一個 (或少量)** 請求通過，當作「探路鳥 (Canary)」。
    *   如果這個請求**成功** -> 下游恢復了 -> 轉回 **Closed**。
    *   如果這個請求**失敗** -> 下游還沒好 -> 轉回 **Open** 並重新計時。

---

## 2. 第二樂章：為什麼需要它？

你可能會問：「Context Timeout 不就夠了嗎？」

**差異在於「全域觀點」 vs 「個體觀點」**：
*   **Context**: 只關心**當前這個請求**有沒有超時。它不知道前一個請求是不是也失敗了。
*   **Circuit Breaker**: 它統計**過去一段時間的失敗率**。如果過去 100 個請求都失敗了，第 101 個請求連試都不用試，直接報錯。

**好處**：
1.  **保護下游**：給掛掉的 DB 喘息回復的機會，而不是持續用重試流量轟炸它。
2.  **保護自己**：避免自己的 Worker Goroutine 全部卡在等待 I/O Timeout，導致自己也由內而外崩潰 (Resource Exhaustion)。
3.  **快速響應**：使用者會立刻收到 Error (或降級頁面)，而不是看著瀏覽器轉圈圈 30 秒然後才失敗。

### 2.1 決策指南：什麼時候該用？
如果不想引入 Circuit Breaker 的複雜度，只用 Context Timeout 可以嗎？
答案取決於你的**流量規模**。

1.  **Context Timeout (基本款)**
    *   **適用場景**：**小流量** (例如 < 100 QPS)、內部後台系統、非核心路徑。
    *   **代價**：當下游掛掉時，請求會卡住直到超時。如果單機由 10 個 Goroutine 變 1000 個，記憶體可能會爆，但在低流量下可接受。

2.  **Circuit Breaker (進階款)**
    *   **適用場景**：**高流量** (例如 Peak QPS > 1000)、對外 API、支付/交易核心路徑。
    *   **必要性**：在高並發下，**"Wait for Timeout" 本身就是一種災害**。
    *   **例子**：如果有 1 萬個請求同時湧入，每個都等 3 秒超時，你的 Server 會瞬間積壓 3 萬個閒置連線，導致 CPU 切換過熱或 OOM。這時候你需要斷路器在 **0.1ms** 內直接拒絕請求 (Fail Fast)，保命要緊。

---

## 3. 第三樂章：Go 實作範例

Go 社群最常用的庫是 `github.com/sony/gobreaker`。這裡我們用簡化版邏輯來演示其核心：

```go
package main

import (
	"errors"
	"fmt"
	"sync"
	"time"
)

// 狀態定義
type State int

const (
	StateClosed State = iota
	StateOpen
	StateHalfOpen
)

type CircuitBreaker struct {
	mu           sync.Mutex
	state        State
	failureCount int
	threshold    int           // 失敗多少次跳閘
	lastOpenTime time.Time     // 上次跳閘時間
	sleepWindow  time.Duration // 冷靜期長度
}

func NewCircuitBreaker() *CircuitBreaker {
	return &CircuitBreaker{
		state:       StateClosed,
		threshold:   3, // 測試用，連續失敗 3 次就跳
		sleepWindow: 2 * time.Second,
	}
}

// Execute 這是斷路器的核心包裝函數
// 它接受一個函數 req (你的業務邏輯)，並由斷路器決定是否執行它
func (cb *CircuitBreaker) Execute(req func() error) error {
	cb.mu.Lock()
	
	// 1. 檢查是否 Open
	if cb.state == StateOpen {
		// 檢查是否過了冷靜期，可以嘗試 Half-Open?
		if time.Since(cb.lastOpenTime) > cb.sleepWindow {
			fmt.Println("State Change: Open -> Half-Open (Testing...)")
			cb.state = StateHalfOpen
		} else {
			cb.mu.Unlock()
			return errors.New("circuit breaker is open (fail fast)")
		}
	} else if cb.state == StateHalfOpen {
        // 如果已經有人在試了，其他人拒絕 (簡單實作)
        cb.mu.Unlock()
		return errors.New("circuit breaker is half-open (waiting for probe)")
    }
	cb.mu.Unlock()

	// 2. 執行真正的請求
	err := req()

	cb.mu.Lock()
	defer cb.mu.Unlock()

	// 3. 根據結果更新狀態
	if err != nil {
		// 請求失敗
		if cb.state == StateHalfOpen {
			fmt.Println("Probe Failed! State Change: Half-Open -> Open")
			cb.state = StateOpen
			cb.lastOpenTime = time.Now()
		} else {
			cb.failureCount++
			if cb.failureCount >= cb.threshold {
				fmt.Println("Threshold Reached! State Change: Closed -> Open")
				cb.state = StateOpen
				cb.lastOpenTime = time.Now()
			}
		}
		return err
	}

	// 請求成功
	if cb.state == StateHalfOpen {
		fmt.Println("Probe Success! State Change: Half-Open -> Closed")
		cb.state = StateClosed
		cb.failureCount = 0
	} else {
        // Closed 狀態下成功，重置失敗計數 (或是依滑動窗口遞減)
		cb.failureCount = 0 
	}
	
	return nil
}

func main() {
	cb := NewCircuitBreaker()

	// 模擬不穩定的服務
	unstableService := func() error {
		// 模擬總是失敗
		return errors.New("service timeout")
	}
    
    // 成功服務
    fixedService := func() error { return nil }

	for i := 1; i <= 10; i++ {
        // 前 5 次模擬失敗
        var err error
        if i <= 5 {
             err = cb.Execute(unstableService)
        } else {
             // 第 6 次後服務修好了，但要看斷路器給不給過
             time.Sleep(500 * time.Millisecond) // 慢慢執行
             err = cb.Execute(fixedService)
        }
		
		if err != nil {
			fmt.Printf("Req %d: error: %v\n", i, err)
		} else {
			fmt.Printf("Req %d: success\n", i)
		}
		time.Sleep(200 * time.Millisecond)
	}
}
```

### 3.1 效能悖論：鎖的代價 (The Performance Paradox)
你的直覺可能已經警鈴大作：**「如果我在高流量下使用 Circuit Breaker，這個 `cb.mu.Lock()` 豈不是會變成新的瓶頸？」**

沒錯！
*   **標準實作 (`sony/gobreaker`)**: 使用 `sync.Mutex`。在 QPS < 20,000 時通常沒問題 (Mutex 在無競爭下極快)。
*   **極致優化 (Hystrix/Google SRE)**: 在 **10萬+ QPS** 的場景，我們會改用 **Atomic CAS** (Book 3.7) 來實作計數器，完全移除 Mutex，或者採用 **Striping (分片鎖)** 技術，將請求分散到多個小 Breaker 以減少競爭。

**結論**：Circuit Breaker 是必要的惡 (Necessary Evil)。對於超高併發系統，請務必選擇 **Lock-Free** 的實作版本。

### 3.2 實戰細節：什麼算失敗？(Error Classification)
很多初學者會犯一個錯：把所有 error 都當成失敗。
**千萬不要！** 否則用戶輸入錯誤參數 (400) 或查無資料 (404) 也會導致 Circuit Breaker 跳閘，引發誤殺。

我們需要一個 **過濾器 (Filter)**：

```go
// 自定義判斷邏輯
func isFailure(err error) bool {
    if err == nil {
        return false
    }
    // 1. 必殺名單 (System Errors): 這些代表下游真的掛了
    if errors.Is(err, context.DeadlineExceeded) || // Timeout (最重要!)
       errors.Is(err, syscall.ECONNREFUSED) ||     // 連不上
       errors.Is(err, sql.ErrConnDone) {           // 連線中斷
        return true
    }
    
    // 2. 白名單 (Business Errors): 這些是邏輯錯誤，下游其實是健康的
    // e.g. record not found, validation error, duplicate key
    return false
}
```
**法則**：只計算那些 **「重試也沒用」** 或 **「代表系統過載」** 的錯誤（特別是 Timeout）。

---

## 4. 第四樂章：與 Worker Pool & Rate Limiter 的協奏

這三個模式是如何協同工作的？

1.  **最外層 (Rate Limiter)**：先檢查你的流量是否超標？超標直接擋掉。
2.  **中間層 (Circuit Breaker)**：檢查下游還活著嗎？如果死了，直接擋掉，不要浪費時間。
3.  **內層 (Worker Pool)**：如果前兩關都過了，分配一個 Worker 給你。
4.  **執行層 (Context)**：Worker 用 Context 控制執行時間，如果超時就 Cancel。

這形成了一個 **同心圓防禦體系 (Defense in Depth)**：
*   **Rate Limiter** 保護 **系統容量**。
*   **Circuit Breaker** 保護 **外部依賴**。
*   **Worker Pool** 保護 **硬體資源**。
*   **Context** 保護 **單次請求**。

---

## 5. 總結

**Book 4: Concurrency Patterns & Resilience** 到此告一段落。

從底層的 **GMP** 調度模型 (Book 3)，到上層的 **Pipeline** 數據流 (Book 4.1)，再到 **Rate Limit**、**Worker Pool**、**Circuit Breaker** 構成的堅固堡壘。
不過，還有一個特殊場景值得我們探討：當 1000 個請求**同時**查詢**同一個**熱門資料 (Hot Key) 時，上述機制可能都擋不住 (Rate Limit 允許通過，Worker Pool 全開)，結果是 DB 被重複查詢打死。

這就是 **Book 4.6: Singleflight (防擊穿模式)** 的舞台。
