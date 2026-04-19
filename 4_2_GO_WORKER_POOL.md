# Book 4.2: The Worker Pool (工人池模式)

在上一章的 **Pipeline** 中，我們學會了如何將任務拆解成多個階段。其中提到的 **Fan-Out (扇出)** 技巧，讓我們能啟動多個 Goroutine 來並行處理任務。

然而，**不受控制的 Fan-Out 是危險的**。

如果你的上游瞬間湧入 100 萬個請求，而你為每個請求都 `go func()`，你的伺服器可能會因為記憶體耗盡 (OOM) 或過多的 Context Switch 而崩潰。這時候，我們需要一種更有紀律的並發模式：**Worker Pool (工人池)**。

---

## 1. 第一樂章：為什麼需要池化？

**Worker Pool** 的核心思想是 **「資源復用」** 與 **「總量控制」**。

### 1.1 問題：失控的並發 (Unbounded Concurrency)
想像一個處理圖片的服務：
```go
func handleRequests(jobs <-chan Job) {
    for job := range jobs {
        // 危險：來幾個請求就開幾個 Goroutine
        go process(job) 
    }
}
```
*   **記憶體爆炸**：每個 Goroutine 至少佔用 2KB 棧空間，100 萬個就是 2GB。如果有大量遞迴或變數，消耗更大。
*   **CPU 抖動**：過多的 Goroutine 會導致 CPU 頻繁在不同執行緒間切換 (Context Switch)，花在切換的時間比真正工作的時間還多。
*   **下游崩潰**：如果你的處理邏輯包含資料庫查詢，瞬間 100 萬個 DB 連線會直接把資料庫打掛。

### 1.2 解法：固定數量的工人 (Bounded Concurrency)
我們不再為每個任務建立 Goroutine，而是**預先建立固定數量的 Goroutine (Workers)**，讓它們去競爭消費同一個任務隊列 (Job Queue)。

這就像銀行的櫃檯：無論外面排隊的人 (Job) 有多少，櫃檯 (Worker) 只有 5 個。處理速度取決於這 5 個人的效率，但絕對不會因為人太多而讓銀行關門。

---

## 2. 第二樂章：基本結構 (The Structure)

一個標準的 Worker Pool 包含三個角色：
1.  **Job (任務)**：需要被處理的數據。
2.  **Worker (工人)**：固定數量的 Goroutine，不斷從 Queue 拿工作。
3.  **Dispatcher (派發者)**：負責將任務放入 Queue (通常就是 main 或者上游 Pipeline)。

### 2.1 實作範例

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

// 1. Job: 定義任務內容
type Job struct {
	ID       int
	Data     string
}

// 2. Result: 定義處理結果 (可選)
type Result struct {
	JobID    int
	Output   string
	Err      error
}

// worker: 工人的具體工作邏輯
// id: 工人編號，方便除錯
// jobs: 領工作的窗口 (只讀)
// results: 繳交成果的窗口 (只寫)
func worker(id int, jobs <-chan Job, results chan<- Result, wg *sync.WaitGroup) {
	defer wg.Done() // 下班打卡

	// 不斷從 channel 拿任務，直到 channel 被關閉且清空
	for job := range jobs {
		fmt.Printf("Worker %d: started job %d\n", id, job.ID)
		
		// 模擬耗時工作
		time.Sleep(time.Second) 
		
		// 處理完畢，發送結果
		results <- Result{
			JobID:  job.ID,
			Output: job.Data + " processed",
		}
		
		fmt.Printf("Worker %d: finished job %d\n", id, job.ID)
	}
}

func main() {
	const numJobs = 10
	const numWorkers = 3 // 限制只有 3 個並發

	// 建立任務隊列 (Job Queue) 和結果隊列
	jobs := make(chan Job, numJobs)
	results := make(chan Result, numJobs)
	
	// 用來等待所有工人下班的 WaitGroup
	var wg sync.WaitGroup

	// 1. 啟動 Worker (Fan-Out)
	// 就像銀行開門前，先把 3 個行員叫到櫃檯坐好
	for w := 1; w <= numWorkers; w++ {
		wg.Add(1)
		go worker(w, jobs, results, &wg)
	}

	// 2. 派發任務 (Dispatcher)
	// 開始把客戶放進排隊紅龍
	for j := 1; j <= numJobs; j++ {
		jobs <- Job{ID: j, Data: fmt.Sprintf("data-%d", j)}
	}
	// 關鍵：發完任務一定要關閉 jobs channel
	// 這會告訴 workers：「沒工作了，處理完手上的就下班」
	close(jobs)

	// 3. 等待所有工人下班 (Wait)
	// 我們需要一個獨立的 Goroutine 來等，因為主程式還要忙著收結果
	go func() {
		wg.Wait()
		// 確定工人都走了，才關閉結果通道
		close(results) 
	}()

	// 4. 收集結果 (Collect)
	// 主程式在這裡讀取結果，直到 results 被關閉
	for res := range results {
		fmt.Printf("Result: Job %d -> %s\n", res.JobID, res.Output)
	}
}
```

---

## 3. 第三樂章：深層機制解析

這段程式碼看似簡單，卻包含了幾個重要的 Runtime 行為：

### 3.1 Channel 作為鎖 (Auto-Balancing)
你可能會問：「三個 Worker 同時去讀 `jobs`，會不會打架？」
不會。
我們在 **Book 3.1 GMP** 和 **Book 3.10 Cond** 裡學過，Channel 內部由 `hchan.lock` (Mutex) 保護。
*   當 **Worker 1** 執行 `<-jobs` 時，它獲得鎖。
*   如果 Channel 裡沒資料，**Worker 1** 會被包裝成一個 `sudog` (Runtime 的結構體)，並乖乖排進 Channel 的 **`recvq` (接收者隊列)**。然後它會釋放鎖並去睡覺 (`gopark`)。
*   這是一個 **FIFO 隊列**。當 **Worker 2** 也來要資料時，它會排在 Worker 1 後面。
*   **公平性 (Fairness)**: 當發送者放入一個數據時，Runtime 會直接把這個數據 **複製 (Direct Copy)** 給 `recvq` 第一名的 Worker 1，並喚醒它。這保證了先來的 Goroutine 一定先拿到工作，防止飢餓。
*   Go Runtime 自動處理這種競爭與排隊，實現了**最純粹的公平負載平衡**：誰先空閒排隊，誰就先拿任務。

### 3.2 緩衝區的重要性 (The Buffer)
`jobs := make(chan Job, numJobs)`
這裡我們使用了 **Buffered Channel**。
*   **如果不緩衝**：`main` (派發者) 必須等有一個 Worker 空閒取走任務，才能發送下一個。這會導致 `main` 被 Worker 的處理速度拖慢。
*   **使用緩衝**：這是一個 **「解耦 (Decoupling)」** 的過程。`main` 可以瞬間把任務全丟進 Queue (只要 Queue 夠大)，然後去忙別的，Worker 則按照自己的步調慢慢消化。這就是典型的 **「生產者-消費者 (Producer-Consumer)」** 模型。

### 3.3 優雅關閉 (Graceful Shutdown)
Worker Pool 最容易寫錯的地方就是 **死鎖 (Deadlock)** 或 **洩漏 (Leak)**。請記住這個口訣：
1.  **發送者關閉輸入**：`main` 發完數據後 `close(jobs)`。
2.  **接收者感知結束**：Worker 的 `range jobs` 迴圈會在 channel 關閉且空了之後自動跳出。
3.  **等待者關閉輸出**：必須等所有 Worker 都退出 (`wg.Wait()`)，才能 `close(results)`。
    *   *錯誤示範*：如果你在 `main` 直接 `wg.Wait()` 然後 `close(results)`，會發生 Deadlock，因為 `main` 被 `wg.Wait` 卡住，沒人去讀 `results`，導致 Worker 想寫 `results` 卻寫不進去（如果 results 滿了），Worker 卡住 -> wg 無法 Done -> main 永遠卡住。

### 3.4 底層解密：Range Loop 的真面目

你可能會好奇，為什麼 `for job := range jobs` 這麼神奇，既能等待資料，又能自動偵測關閉？
其實這只是編譯器給的 **語法糖 (Syntactic Sugar)**。在編譯階段，它會被轉換成類似這樣的虛擬碼：

```go
// 這是 range jobs 的真實面貌
for {
    // 呼叫 runtime.chanrecv，這是阻塞式的讀取
    // ok 回傳 true 代表讀到資料，false 代表 channel 已關閉且空了
    job, ok := <-jobs 
    
    if !ok {
        break // Channel 關閉且沒資料了，跳出迴圈
    }
    
    // 執行你的 Loop Body
    // ...
}
```

這個機制依賴於 Go Runtime 的 `chanrecv` 函數：
1.  **阻塞 (Blocking)**：如果 `jobs` 裡沒資料但還沒 Close，`chanrecv` 會呼叫 `gopark` 讓 Goroutine 進入睡眠 (Waiting 狀態)，釋放 MP 資源。
2.  **喚醒 (Waking)**：當有新 Job 進來，Sender 會呼叫 `goready` 喚醒這個 Goroutine。
3.  **退出 (Exit)**：當 `jobs` 被 Close 且 Buffer 空了，`chanrecv` 會立刻返回 `false`，迴圈也就結束了。

所以，`range loop` 是最優雅且高效的 Channel 消費方式，完全貼合 Go 的調度模型。

---

## 4. 第四樂章：進階 - 動態與回壓 (Dynamic & Backpressure)

上面的例子是「靜態」的，但在真實高吞吐系統中，你需要考慮更多：

### 4.1 任務隊列爆滿怎麼辦？
如果 `jobs` channel 滿了，`main` 塞不進去，這就是一種天然的 **Backpressure (背壓)**。

**如果不丟棄會發生什麼後果？**
1.  **延遲飆升 (Latency Spike)**：客戶端的請求不會失敗，但會一直轉圈圈等待。如果後端處理要 1 秒，隊列排了 1000 個，最後一個進來的請求就要等 1000 秒，這比直接報錯還糟糕 (使用者體驗極差)。
2.  **記憶體堆積 (OOM Risk)**：如果上游是 HTTP Server，每個等待寫入 Channel 的請求都對應一個 Goroutine 和 HTTP Connection，這些都會佔用記憶體。如果不快速失敗 (Fail Fast)，伺服器最終會 OOM。
3.  **雪崩效應 (Cascading Failure)**：上游的 Load Balancer 或 API Gateway 可能因為你的回應太慢，判定你掛了或超時，於是發起 **重試 (Retry)**。這些重試請求更加劇了隊列的擁堵，導致系統徹底癱瘓。

預設情況下，發送端會阻塞 (Block)。但如果你希望採取 **「丟棄 (Drop)」** 策略而不是等待，可以使用 `select` + `default`：

```go
// Non-Blocking Enqueue (非阻塞入隊)
func enqueue(job Job, jobs chan<- Job) bool {
    select {
    case jobs <- job:
        return true // 成功入隊
    default:
        // 通道滿了，直接丟棄請求，或者回傳 503 Service Unavailable
        fmt.Println("Queue is full, dropping job", job.ID)
        return false
    }
}
```

這種模式常見於高吞吐的網關服務，寧可丟棄過載的請求，也不要讓整個系統卡死 (Fail Fast)。
另一種策略是 **擴容 (Scale Up)**，即動態增加 Worker 數量，但這需要更複雜的監控機制。

### 4.2 Worker 裡的 Panic
如果 Worker 處理某個 Job 時 Crash 了怎麼辦？
```go
func worker(...) {
    defer wg.Done()
    for job := range jobs {
        // 安全網
        func() {
            defer func() {
                if r := recover(); r != nil {
                    fmt.Println("Recovered in worker", r)
                }
            }()
            // 執行實際邏輯
            process(job)
        }()
    }
}
```
**切記：** 永遠要在 Worker 迴圈內部做 `recover`，確保一個 Job 失敗不會殺死整個 Worker（或者讓 Worker 死掉後由監控機制重啟，這涉及更高級的 Supervisor 模式）。

---

## 5. 第五樂章：總結

**Worker Pool** 是 Go 並發模式中的「中流砥柱」。

| 特性 | Fan-Out Only | Worker Pool |
| :--- | :--- | :--- |
| **並發數** | 無限制 (Unbounded) | 固定限制 (Bounded) |
| **記憶體消耗** | 不可預測，高風險 | 可預測，穩定 |
| **實作難度** | 低 | 中 (需管理 Lifecycle) |
| **資源保護** | 無 (易打掛下游) | 有 (保護 DB/API) |
| **適用場景** | 短期、少量任務 | 長期運行、大流量服務 |

掌握了 Worker Pool，你就掌握了保護伺服器不被流量衝垮的第一道防線。接下來，我們將探討如何讓這些 Goroutine 更有智慧地溝通與取消 —— **Book 4.3: The Context**。
