# Book 4.1: The Pipeline (流水線模式)

在前面三本書中，我們探討了 Go Runtime 的內部構造。現在，我們要運用這些知識來建造宏偉的建築。
第一個要學的模式是 **Pipeline (流水線)**：將一個大任務拆解成多個階段，讓數據像水流一樣穿過。

---

## 1. 第一樂章：為什麼需要流水線？

想像你要處理 100 萬張圖片：`下載 -> 縮圖 -> 上傳`。

*   **單體模式 (Monolith)**:
    *   一個 for loop，依序做下載、縮圖、上傳。
    *   **缺點**: 上傳時，CPU 閒著 (等待 IO)。縮圖時，網卡閒著。資源利用率低。

*   **流水線模式 (Pipeline)**:
    *   **Stage 1 (Downloader)**: 只負責下載，丟進 Channel A。
    *   **Stage 2 (Resizer)**: 從 Channel A 拿圖，縮圖，丟進 Channel B。
    *   **Stage 3 (Uploader)**: 從 Channel B 拿圖，上傳。
    *   **優點**: 三個階段 **同時運作**。CPU 和 IO 都不閒著。

---

## 2. 第二樂章：基礎結構 (The Basic Pattern)

一個標準的 Pipeline Stage 函數通常長這樣：

```go
func Stage1(nums []int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out) // [重點] 做完一定要關門，通知下游
        for _, n := range nums {
            out <- n
        }
    }()
    return out // 返回唯讀 Channel 給下游
}

func Stage2(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in { // [重點] 下游用 range 讀，上游 close 會自動跳出
            out <- n * n
        }
    }()
    return out
}

// 組合
c1 := Stage1(nums)
c2 := Stage2(c1)
```

---

## 3. 第三樂章：擴散與匯聚 (Fan-Out / Fan-In)

如果 Stage 2 (縮圖) 特別慢，是效能瓶頸怎麼辦？
我們可以啟動多個 Resizer Worker 來搶食 Channel A 的工作。

### Fan-Out (擴散)
啟動多個 Goroutine 讀取同一個 Input Channel。
這就是 **Work Stealing** 的應用層實現 (GMP 會自動幫你把這些 G 分配到不同 P 上)。

```go
// 啟動 10 個 Worker
checkers := make([]<-chan int, 10)
for i := 0; i < 10; i++ {
    checkers[i] = Worker(inputChan)
}
```

### Fan-In (匯聚)
下游只有一個 Uploader，所以我們要把這 10 個 Worker 的結果匯總到一個 Channel。

```go
func Merge(cs ...<-chan int) <-chan int {
    var wg sync.WaitGroup
    out := make(chan int)

    output := func(c <-chan int) {
        defer wg.Done()
        for n := range c {
            out <- n
        }
    }

    wg.Add(len(cs))
    for _, c := range cs {
        go output(c)
    }

    // [重點] 必須另外開一個 G 來等 WaitGroup，等大家都做完了才能 Close out
    go func() {
        wg.Wait()
        close(out)
    }()
    return out
}
```

---

## 4. 第四樂章：優雅退出 (Cancellation & Leaks)

這是 Pipeline 模式最致命的陷阱：**Goroutine Leak**。
如果下游發生錯誤提早退出了，上游還傻傻地往 Channel 塞資料，但沒人讀，上游就會 **永遠卡住 (Block Forever)**。

**解法：Done Channel**
所有的 Stage 都必須監聽一個 `done` channel (或者 `context.Done`)。

```go
func Stage(done <-chan struct{}, in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            select {
            case out <- n * n:
                // 正常發送
            case <-done:
                // [重點] 收到停工信號，立刻棄船逃生
                return 
            }
        }
    }()
    return out
}
```

---

## 5. 第五樂章：總結 (Summary)

1.  **Always Close**: 生產者必須負責 `close` channel，這是通知消費者「下班」的唯一方式。
2.  **Done Channel**: 為了防止 Goroutine Leak，一定要有一個機制能從外部「關閉整個流水線」。
3.  **Buffer**: 適當給 Channel 加 Buffer 可以減少 Context Switch，提高吞吐量，但不要依賴 Buffer 來解決死鎖。

**Pro Tip**:
如果你的 Stage 之間處理速度差異巨大 (例如下載很慢，計算很快)，除了 Pipeline，你可能還需要 **Book 4.2 的 Worker Pool** 來限制並發數量，避免把記憶體撐爆。
