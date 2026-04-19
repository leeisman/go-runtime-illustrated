# Book 4.9: Sharding & Ordered Concurrency (有序分片模式)

在前面的章節中，我們追求的都是 **「越快越好」**。
*   Worker Pool：哪裡有空閒 Worker 就去哪裡 (Random)。
*   ErrGroup：誰先回來算誰的 (Random)。

但在某些業務場景，**順序 (Order)** 比速度更重要。

**場景舉例 (Order Matters)**：
1.  **金融交易**: 用戶 A 先「存款 100」，再「轉帳 50」。如果這兩個指令被不同的 Worker 並行處理，變成先轉帳再存款，就會因為餘額不足失敗。
2.  **遊戲對戰**: 玩家先「移動」，再「攻擊」。如果亂序，變成原地空揮再移動，邏輯就炸了。
3.  **Binlog Sync**: 資料庫的 Change Log 必須嚴格按照時間順序重放。

挑戰：**我們既想要 Worker Pool 的高併發（多核處理），又想要針對單一用戶的嚴格順序。**

這就是 **Sharding (分片/分區)** 模式的舞台。

---

## 1. 第一樂章：Hash Sharding 原理

核心思想非常簡單：**把世界切成 N 片，每一片由一個專屬的 Worker 負責。**

$$WorkerID = Hash(EntityID) \ \% \ WorkerCount$$

*   如果你是 User A (ID=101)，Hash(101) % 10 = 1。那你這輩子所有的任務都只會交給 **Worker 1**。
*   如果你是 User B (ID=102)，Hash(102) % 10 = 2。你歸 **Worker 2** 管。

**效果**：
1.  **全局並發**: Worker 1 和 Worker 2 是並行跑的。所以 User A 和 User B 互不影響，速度很快。
2.  **局部有序**: User A 的所有指令都在 Worker 1 的 Channel 裡排隊。因為 Channel 是 FIFO 的，且 Worker 1 只有一個 Goroutine，所以保證了 User A 的指令是 **嚴格順序執行** 的。

這就是 **Sequential Consistency (順序一致性)**。

---

## 2. 第二樂章：Go 實作範例

我們需要建立一組 Worker，和一個 Dispatcher (分發器)。

```go
package main

import (
	"fmt"
	"hash/fnv"
	"sync"
	"time"
)

// Task 代表一個有序任務
type Task struct {
	UserID  string
	Payload string
}

// ShardedPool 是我們的有序工作池
type ShardedPool struct {
	workers    []chan Task // 每個 Worker 一個專屬 Channel
	workerNum  int
	wg         sync.WaitGroup
}

// NewPool 初始化 N 個 Worker
func NewPool(n int) *ShardedPool {
	p := &ShardedPool{
		workers:   make([]chan Task, n),
		workerNum: n,
	}

	for i := 0; i < n; i++ {
		p.workers[i] = make(chan Task, 100)
		p.wg.Add(1)
		
		// 啟動 Worker i
		go func(id int, ch chan Task) {
			defer p.wg.Done()
			fmt.Printf("Worker %d started\n", id)
			for task := range ch {
				// 處理邏輯
				fmt.Printf("[Worker %d] User %s: processing %s\n", id, task.UserID, task.Payload)
				time.Sleep(100 * time.Millisecond) // 模擬處理
			}
		}(i, p.workers[i])
	}
	return p
}

// Push 負責分發 (Hash Dispatching)
func (p *ShardedPool) Push(t Task) {
	// 1. 計算 Hash
	h := fnv.New32a()
	h.Write([]byte(t.UserID))
	workerID := int(h.Sum32()) % p.workerNum

	// 2. 丟進對應的 Channel
	// 這裡保證了同一個 UserID 永遠去同一個 Channel
	p.workers[workerID] <- t
}

func (p *ShardedPool) Stop() {
	for _, ch := range p.workers {
		close(ch)
	}
	p.wg.Wait()
}

func main() {
	pool := NewPool(4) // 開 4 個分片

	// 模擬 User A 的操作 (必須保證順序 A1 -> A2 -> A3)
	// 用戶 B 的也可以並行 (B1, B2)
	
	tasks := []Task{
		{"UserA", "1. Login"},
		{"UserB", "1. Login"},
		{"UserA", "2. Deposit"},
		{"UserA", "3. Transfer"},
		{"UserB", "2. Logout"},
	}

	for _, t := range tasks {
		pool.Push(t)
	}

	// 等待執行
	time.Sleep(2 * time.Second)
	pool.Stop()
}
```

**預期輸出**：
```text
[Worker 0] UserA: processing 1. Login
[Worker 3] UserB: processing 1. Login
[Worker 0] UserA: processing 2. Deposit  <-- A2 一定在 A1 後面
[Worker 0] UserA: processing 3. Transfer <-- A3 一定在 A2 後面
[Worker 3] UserB: processing 2. Logout
```

### 2.1 鎖的悖論 (Lock Striping)
你可能會懷疑：「開了這麼多 Channel，豈不是會造成大量的 Lock Contention (鎖競爭)？」

事實恰恰相反！
*   **單一 Channel**: 所有生產者搶 **1 把鎖**。 (High Contention)
*   **Sharding Channels**: 生產者被分散去搶 **N 把鎖**。 (Low Contention)

這叫做 **Lock Striping (鎖分段)**。
只要分片數 `N` 不要大到誇張 (例如 10 萬)，適量的分片 (例如 CPU Cores * 4) 反而能大幅降低鎖競爭，提升吞吐量。

### 2.2 至高視角：GMP 與 Futex 的戰爭
為什麼「減少鎖競爭」這麼重要？讓我們用 **Book 3** 的底層視角來看：

### 2.2 至高視角：GMP 與 Futex 的戰爭
為什麼「減少鎖競爭」這麼重要？讓我們看看當 100 個 G 同時對 1 個 Channel 執行 Send (`ch <-`) 時，情況是如何一步步惡化的：

1.  **第一階段：User Space 內耗 (CPU 空轉)**
    *   **現象**: 99 個 G 搶不到鎖，會在 CPU 上 **Spinning (自旋)**。
    *   **代價**: CPU 100% 運轉，但都在做無效的 CAS 檢查。而且多核爭搶同一個變數 (`lock`) 會導致 **Cache Line Bouncing (快取失效)**，大幅降低硬體效率。

2.  **第二階段：Blocking (G 睡覺)**
    *   **現象**: 如果轉太久還是搶不到，或者搶到了但 **Buffer 滿了**。
    *   **代價**: G 必須放棄 CPU，呼叫 `runtime.gopark` 進入隊列等待。這裡發生了 **Goroutine Context Switch**。

3.  **第三階段：Kernel Space 懲罰 (M 睡覺)**
    *   **現象**: 如果大量 G 都去睡了，P 的隊列空了，導致 M (OS Thread) 無事可做。
    *   **代價**: M 被迫呼叫 `futex(WAIT)` 請求 OS 讓它掛起。這觸發了最昂貴的 **OS Context Switch (> 1000ns)**。

**Sharding 的救贖**：
透過開 100 個 Channel，流量被分散了。絕大多數的 G 都能在 **第一階段 (CAS)** 就瞬間搶到鎖，因此根本不需要 Spinning，更不會惡化到後面阻塞甚至 Futex 的地步。

### Note: 到底是誰在睡 Futex？
這是一個常見誤區。在 Go 裡，**Goroutine (G)** 搶不到鎖時，只是在 Go Runtime 層面被放進隊列 (`gopark`)，成本很低。
真正呼叫 **Futex** 睡覺的是 **M (OS Thread)**。
只有當 **所有 G 都被阻塞**，導致 M 沒工作可做 (No Work to Run) 時，M 才會去睡 Futex。

**Worker Pool 的致命傷** 就在於它容易瞬間阻塞大量 G，導致 P 的隊列被掏空，迫使 M 進入昂貴的 Futex 睡眠。

### 2.3 番外篇：Channel 的 Handoff 魔法 
你可能會擔心：「那些不幸睡著的 G，醒來後豈不是要**重新搶鎖** (Thundering Herd)？」
這在普通的 `Mutex` 是真的會發生，但在 Channel 不會！

Go Channel 有一個特殊的 **Handoff (交接)** 機制：
1.  當 G1 因 Buffer 滿而睡在 `sendq` 裡。
2.  接收者 G2 來拿資料時，它**直接把 G1 的資料拿走** (從 G1 的 Stack 複製過來)。
3.  然後 G2 呼叫 `goready(G1)`。
4.  **G1 醒來時，發現資料已經送出去了！** 它**不需要搶鎖，也不需要再次 Send**，直接繼續跑。

這就是為什麼在做生產者-消費者模型時，**Channel 永遠比手寫 Mutex 優秀** 的底層原因。

---

## 3. 第三樂章：潜在問題 (Hotspot)

雖然 Sharding 解決了順序問題，但它引入了 **Data Skew (資料傾斜)** 的風險。

*   **情境**：如果 User A 是一個超級大戶，每秒發 1000 個請求。User B, C, D 每秒只有 1 個。
*   **結果**：Worker 0 (負責 User A) 會被操死，佇列永遠是滿的。而 Worker 1, 2, 3 都在發呆。
*   **短板效應**：系統的總吞吐量卡在最慢的那個 Worker (A)。

**解法**：
1.  **Virtual Nodes (虛擬節點)**: 一致性雜湊 (Consistent Hashing) 的概念，讓大戶更有機會分散 (但在這裡不適用，因為我們要保序)。
2.  **隔離 (Isolation)**: 把大戶 (Hot User) 單獨識別出來，給他開一個專屬的 VIP Worker，不要讓他影響平民用戶。
3.  **Shuffle Sharding**: 這是 AWS 用的高階技巧，為了提高隔離性。(延伸閱讀)

---

## 4. 第四樂章：總結

**Book 4.9 Sharding** 告訴我們：並發不一定要亂序。

透過 **Partitioning (分區)**，我們可以同時獲得並發的性能和單線程的確定性 (Determinism)。這也是 Kafka Topic Partition、Redis Cluster Slot 背後的核心設計哲學。

現在，回頭看看我們的軍火庫：

| 模式 | 解決什麼問題？ |
| :--- | :--- |
| **Pipeline** | 資料流轉與解耦 |
| **Worker Pool** | 資源控制與隨機並發 |
| **Rate Limiter** | 入口流量整形 |
| **Circuit Breaker** | 下游故障保護 |
| **Singleflight** | 熱點快取擊穿 |
| **Bloom Filter** | 惡意快取穿透 |
| **ErrGroup** | 並發聚合與錯誤傳播 |
| **Sharding** | **有序並發 (Ordered Concurrency)** |

| **Sharding** | **有序並發 (Ordered Concurrency)** |

這九把劍 (4.1 ~ 4.9)，足夠你在 Go 的並發江湖中橫著走了。

但如果你覺得 Sharding 的「固定分片」還不夠靈活，如果你想為 **每一個 User** 都建立一個專屬的「狀態維護者」，徹底拋棄 DB 鎖和 Mutex 鎖。

這就是 **Book 4.10: Actor Model (演員模型)** 的世界。讓我們看看 Go 語言如何實現這個在 Erlang/Akka 中稱霸一方的架構。
