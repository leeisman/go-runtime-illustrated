# Book 3.10: 流程的指揮棒 (WaitGroup & Cond)

如果說 Mutex 是為了保護「資料」，那 **WaitGroup** 和 **Cond** 就是為了控制 **「流程 (Flow)」**。
它們不關心變數是 1 還是 2，它們只關心 **「誰該停，誰該走」**。

---

## 1. 第一樂章：WaitGroup (計數器指揮家)

`sync.WaitGroup` 的底層出乎意料地簡單。它沒有複雜的 Queue，核心只是一個 **64-bit 的整數**。

### 結構解密
```go
type WaitGroup struct {
    noCopy noCopy
    
    // 這是一個 64位元的複合狀態：
    // [ 高32位元 ]: Counter (還要等幾個任務 Finish)
    // [ 低32位元 ]: Waiter  (有多少個 Goroutine 在呼叫 Wait() 等待中)
    state1 uint64 
    
    sema   uint32 // 信號量 (地址)。Runtime 會用這個地址作為 Key，把等待的 G 掛在上面。
}
```

**Runtime 內部邏輯**:
*   Runtime 內部有一個全局的 **Treap (樹狀堆積)** 或 Hash Table。
*   當 `Wait()` 時，它是把 **G (sudog)** 存進這個全域表中，Key 就是 `&wg.sema`。
*   當 `Done()` 時，它用 `&wg.sema` 去查表，把掛上面的 G 抓出來喚醒。
**(所以 sema 本身的值不重要，重要的是它的地址 `&sema` 是一個獨一無二的 ID)**。

### 1. Add(delta) - 派工
*   直接對 `state1` 的 **高 32 位** 做 `atomic.Add`。
*   這是純 CPU 指令，**極快**。

### 2. Done() - 完工
*   就是 `Add(-1)`。
*   每次減完，檢查 Counter 是否歸零。
    *   **沒歸零**: 沒事，繼續跑。
    *   **歸零了**: 
        *   檢查 **低 32 位** (Waiter 數量)。
        *   如果有 5 個 Waiter，就連續呼叫 5 次 `runtime_Semrelease` (搖醒它們)。

### 3. Wait() - 等待並睡覺
*   檢查 Counter 是否為 0？
    *   是 0: 直接返回 (不用等)。
    *   非 0: 對 `state1` 的 **低 32 位** (Waiter) 加 1。
    *   然後呼叫 `runtime_Semacquire` **去睡覺 (Park)**。

**效能分析**:
*   **優點**: `Add/Done` 是無鎖的 (Atomic)，非常高效。
*   **瓶頸**: 唯一的瓶頸是當 Counter 歸零那瞬間，如果有幾萬個 Waiter 在等，Runtime 要一次喚醒幾萬個 G，這會造成短暫的調度壓力。

---

## 2. 第二樂章：Cond (條件廣播站)

`sync.Cond` 是完全不同的野獸。它是用來解決 **「特定條件滿足才執行」** 的問題。

### 結構解密
```go
type Cond struct {
    L Locker // 必須搭配一把鎖 (通常是 Mutex)
    
    // 這裡沒有 queue，只有一個指向 Runtime 內部的 List 指標
    notify  notifyList 
    checker copyChecker
}
// runtime/sema.go
type notifyList struct {
    wait   uint32
    notify uint32
    head   *sudog // 鏈結串列 (Linked List) 的頭
    tail   *sudog // 鏈結串列 的尾
}
```

### 1. Wait() - 加入訂閱名單
*   把當前的 Goroutine (G) 打包成 `sudog`。
*   掛到 `notifyList` 的 **尾巴 (Tail)**。
*   **釋放鎖 (L.Unlock)** (這是關鍵！讓別人有機會改狀態)。
*   **Sleep (gopark)**。

### 2. Signal() - 叫醒一個
*   從 `notifyList` 的 **頭部 (Head)** 取下一個 G。
*   呼叫 `goready` 把它叫醒。
*   **用途**: 一對一通知 (例如：Queue 有一個空位了)。

### 3. Broadcast() - 全員起床 (驚群效應)
*   **這是大招，也是殺招。**
*   它會把 `notifyList` 上 **所有** 的 G 全部取下來。
*   **全部** 呼叫 `goready`。

**效能危機：Thundering Herd (驚群效應)**
想像你有 10,000 個 G 在 `Wait()` (例如等待搶購開始)。
*   你呼叫 `Broadcast()`。
*   **瞬間**：10,000 個 G 全部變成 `Runnable`。
*   **GMP 衝擊**: P 的 Local Queue 瞬間爆滿，Global Queue 也爆滿。
*   **CPU 衝擊**: 所有 CPU 開始瘋狂做 Context Switch。
*   **鎖競爭**: 這 10,000 個 G 醒來的第一件事，就是去搶那把 `L.Lock()` (因為 Wait 返回前要加鎖)。
*   **結果**: 10,000 人搶 1 把鎖。系統瀕臨崩潰，不僅沒人搶到，連其他正常的 G 都被卡死。

---

## 3. 第三樂章：總結 (Summary)

| 元件 | 核心機制 | 效能特徵 | 危險指數 |
| :--- | :--- | :--- | :--- |
| **WaitGroup** | Atomic Counter + Sem | **極快**。適合「分散聚合 (Fan-Out / Fan-In)」模式。 | ⭐ (安全) |
| **Cond.Signal** | Notify List (Head) | **快**。適合「生產者-消費者」一對一通知。 | ⭐ (安全) |
| **Cond.Broadcast** | Notify List (All) | **極重**。喚醒所有人去搶同一把鎖。 | ⭐⭐⭐⭐⭐ (危險！慎用) |

**Pro Tip**:
除非業務邏輯真的需要「一次通知所有人」(例如 Config Reload)，否則盡量用 `Channel` 或 `Signal` 取代 `Broadcast`。在高併發下，`Broadcast` 往往是系統延遲飆升的元兇。
