# Book 3.6: 鎖的本質 (The Lock - sync.Mutex)

我們常聽說：「不要用大鎖，因為鎖很慢。」
但你是否想過，鎖到底慢在哪裡？是誰讓它慢的？是 CPU？是 OS？還是 Runtime？

這一章，我們將解開 **Mutex (互斥鎖)** 的底層機制，看看當一個 M 搶不到鎖時，究竟發生了什麼事。

---

## 1. 第一樂章：鎖的兩種型態 (Two Faces of Lock)

本章我們將以標準庫最常用的 **`sync.Mutex`** 為例，解析它如何在底層運作。

### 0. 核心結構 (The Anatomy)
但在進入運作流程前，我們先看看它長什麼樣子。`sync.Mutex` 驚人的精簡，只有兩個欄位：

```go
type Mutex struct {
    state int32  // 狀態 (複合型 BitMap)
    sema  uint32 // 信號量 (用來掛起 Goroutine 的地址)
}
```
*   **`state` (32 bits)**: 這是一個極度壓縮的狀態機。
    *   **Locked (1 bit)**: 鎖是否被持有 (0=Free, 1=Locked)。
    *   **Woken (1 bit)**: 是否有 Goroutine 剛被叫醒 (用來通知自旋者「你可以休息了」)。
    *   **Starving (1 bit)**: 是否進入飢餓模式 (後續會詳解)。
    *   **Waiter Count (29 bits)**: 紀錄目前有多少個 Goroutine 在排隊等待 (Waiting)。
*   **`sema`**: 這是用來管理等待隊列的 **信號量 (Semaphore)**。當 G 需要休眠時，就會把自己掛在這個地址對應的 Treap 上。

了解到它不是一個單純的 Boolean 開關後，我們就能理解它如何實現 **複合型機制 (Hybrid)**。

### 1. 樂觀模式：自旋 (Spinning) - User Space
*   **情境**: M1 想要拿鎖，發現鎖被 M2 拿走了。
*   **心態**: M1 想：「M2 應該馬上就會用完吧？我就在門口轉兩圈等等看，不要去麻煩 OS 了。」
*   **動作**: M1 執行一個 **空迴圈 (PAUSE instruction)**。
    *   `for i := 0; i < 30; i++ { cpu_relax() }`
*   **代價**:
    *   **CPU**: 100% 運轉 (正在空轉)。
    *   **OS**: 完全不知情。
    *   **Cache**: 保持熱度 (Warm)。
*   **結果**: 如果 M2 在這幾奈秒內放手了，M1 立刻搶到鎖。**這是最快的情況 (Fast Path)。**

### 2. 悲觀模式：休眠 (Goroutine Parking) - Runtime Scheduler
*   **情境**: M1 轉了 4 次圈圈 (Spinning)，M2 還沒放手。
*   **心態**: M1 想：「不行，M2 可能去上廁所了。但我不想直接去吵 OS (太貴了)，我先跟 Go 的管家 (Scheduler) 說一聲，我不玩了，把機會讓給別人。」
*   **動作**: **Runtime 介入 (gopark)**。
    *   **掛起**: Runtime 把目前的 Goroutine (G1) 標記為 `Waiting`。
    *   **入隊**: 
        *   Runtime 會有一個全域的 hash table (`semtable`)。
        *   根據 `lock` 的地址，找到對應的 Bucket (Treap Root)。
        *   **競爭保護**: 這裡會有一把極小的 **內部鎖 (Root Lock)**。如果 G1 和 G2 同時要入隊，他們會在這裡排隊拿到這把小鎖，確保 Treap 的操作是 Thread-Safe 的。
        *   **順序確立 (FIFO)**: 既然必須排隊拿鎖才能入隊，這裡自然就形成了 **先進先出** 的順序。先搶到 Root Lock 的 G，就會被排在隊伍的前面。
        *   最後將 G1 插入 Treap。
    *   **換人**: 這時候 **M (執行緒) 並沒有睡著**！它只是把 G1 放到一邊，然後去 P 的 Run Queue 抓下一個 G2 來跑。
    *   **🔗 呼應 Book 4.1 GMP 模型**: 這就是 GMP 的強大之處。M 不用隨著 G 一起睡 (不像 Java Thread)，而是持續在 CPU 上工作，發揮 **M:N 排程** 的最大效能。
*   **代價**: **中等 (Medium)**。
    *   **Context Switch**: 是 **User Space (G換G)**，比 Kernel Space (M換M) 便宜非常多。
    *   **Ring Switch**: 無 (還在 User Mode)。
*   **例外**: 只有當整台機器都沒事做，M 真的沒 G 可跑時，M 才會真的去呼叫 OS 睡覺 (`futex`)。這就是 Go `sync.Mutex` 比 OS 原生 Mutex 快的原因。

### 3. 代碼視角：當你寫下 `mu.Lock()` 時發生了什麼？

你只寫了一行簡單的代碼：
```go
var mu sync.Mutex
mu.Lock() // <--- 就在這一行，Runtime 幫你演了一齣三部曲
```
這看似簡單的操作，底層其實是一個 **Hybrid (複合)** 機制。Runtime 會根據競爭狀況，依序嘗試以下三招：

#### 第一招：CAS 秒殺 (Fast Path)
這是最幸運的情況。鎖剛好沒人拿，你直接就搶到了。
```go
// Runtime 內部邏輯：
if atomic.CompareAndSwapInt32(&lock.state, 0, 1) {
    return // 搶到了！代價極低 (CPU 指令級別)
}
// 失敗... 進第二招
// 🔗 呼應 Book 3.1 Atomic: 這就是最底層的 CAS 指令，不僅用於計數，更是實現 Lock 的基石。
```

#### 第二招：樂觀自旋 (Spinning)
如果一次 CAS 失敗，我不甘心，我在 User Space 多試幾次 (燒 CPU 換時間)。
```go
// Runtime 內部邏輯：
// 發現鎖被佔用，但不想馬上放棄，於是幫你偷跑：
for i := 0; i < 30; i++ {
    if atomic.CompareAndSwapInt32(&lock.state, 0, 1) {
        return // 在旋轉過程中搶到了！免除了 System Call
    }
    // 關鍵指令：告訴 CPU "我在空轉，請省點電" -> PAUSE
    cpu_relax() 
}
// 轉了 30 圈還是失敗... 進第三招
```

#### 第三招：悲觀休眠 (Parking)
撐不住了，無法再空轉了，Runtime 決定讓你 (Goroutine) 暫時離場，讓出 P 給別人用。
```go
// Runtime 內部邏輯 (src/runtime/sema.go)：
func semacquire1() {
    // 1. 不需要再空轉了，我們把這個 G 打包，貼上 "Waiting" 標籤
    // 2. 把它掛到這把鎖的 "等待清單 (Treap)" 上
    // 3. 呼叫 gopark，讓出 CPU
    goparkunlock(...) 
    
    // --> 此時 M 會去執行 schedule() 找別的 G
    // --> 如果 M 也沒別的 G 可跑，M 才會真的去執行 OS Syscall (Futex) 睡覺
}
```

**Pro Tip (結論)**: 依照上述原理，**你其實完全不需要自己手寫 SpinLock**。
*   `sync.Mutex` 已經內建了聰明的自旋機制，而且它知道何時該 **停損** (轉太久就去睡覺)。
*   自幹 SpinLock (`for { CAS }`) 最大的風險是 **不懂得停損**。如果持鎖者卡住，你的 SpinLock 會把 CPU 燒到 100% 卻不做任何事 (Busy Waiting)。
*   **最佳實踐**: 直接用 `sync.Mutex`，享受 Runtime 幫你自動切換 Leve1/2/3 的優化。

---

## 2. 第二樂章：為什麼鎖會讓效能變差？(The Cost)

回到你的問題：「用鎖是讓全部 M 暫停嗎？有跟行政區溝通嗎？」

### 1. 沒有讓全部 M 暫停
鎖只會暫停 **「想要搶這把鎖但搶不到」** 的那個 M。其他無辜的 M (M3, M4) 繼續做他們的事，不受影響。

### 2. 效能變差的真兇
鎖的效能殺手主要來自兩個層面：

#### A. 自旋期 (Spinning Cost) - 浪費 CPU
*   當鎖競爭很激烈時，你會發現 CPU 使用率飆高，但程式產出卻沒變多。
*   **原因**: 大家都卡在「樂觀模式」轉圈圈。CPU 雖然是滿的，但都在做無用功 (Busy Waiting)。

#### B. 休眠期 (Scheduling Cost) - 排程介入
*   當自旋失敗，Runtime 會呼叫 `gopark` 把你的 Goroutine 掛起。
*   **代價**:
    1.  **Re-scheduling**: P 必須切換去跑別的 Goroutine (雖然比 OS 切換快，但仍有成本)。
    2.  **Cache Thrashing**: 最慘的。當你的 Goroutine 醒來時，可能已經被搬到另一個 P/M 上了，原本 L1/L2 Cache 裡的資料全沒了 (變冷)。
    3.  **OS Overhead (Optional)**: 只有當 M 真的沒事做時，才會真的睡著 (Futex)，那時候才有昂貴的 OS 成本。

---



## 3. 第三樂章：Go 的優化 (Handoff & Starvation)

Go 的 `sync.Mutex` 最大的秘密，就在於它如何處理 **`Unlock()`** 那一瞬間的權力交接。

### 1. 戰場設定：解鎖的瞬間 (The Race)
當持鎖者呼叫 `Unlock()` 時，並不是直接把鎖交給下一個人，而是鳴槍開跑。場上有兩類選手同時競爭：
*   **選手 A (Woken G)**: 剛被叫醒的 Goroutine (來自 Queue)。
    *   **狀態**: **迷糊 (Cache Cold)**。
    *   **原因**: 它剛被排程到 CPU 上，CPU 的 **L1/L2 Cache** 裡並沒有它需要的變數或指令。它必須去主記憶體 (RAM) 搬運資料，這非常慢。
*   **選手 B (Spinning G)**: 正在 CPU 上自旋的 Goroutine (New)。
    *   **狀態**: **發熱 (Cache Hot)**。
    *   **原因**: 它已經在 CPU 上跑了好幾個 Cycle，需要的資料都在 **L1/L2 Cache** 裡，存取速度是 RAM 的 100 倍。

這場比賽的規則，決定了系統是「高效」還是「公平」：

### 2. 正常模式 (Normal Mode) - 偏心 (Unfair)
*   **戰況**: 裁判 (Runtime) 允許兩者自由搏擊。
*   **結果**: **選手 B (自旋者) 幾乎必勝**。
    *   因為 B 已經在 CPU 上了，而 A 還要經過 Scheduler 排程、Context Switch。
    *   等到 A 醒來睜開眼，鎖早就被 B 搶走並再次上鎖了。
*   **為什麼這樣設計？**
    *   **🔗 呼應 Book 2.1 CPU Cache**: 為了 **Performance**。讓 CPU 熱的人繼續跑，比切換給冷的人快得多。吞吐量 (Throughput) 極大化。

### 3. 飢餓模式 (Starvation Mode) - 公平 (Fair)
*   **觸發**: 如果 **選手 A** (Woken G) 輸太慘，連續 **1ms** 都搶不到鎖。
*   **戰況**: 裁判看不下去了，切換成「飢餓模式」。
*   **結果**: **直接保送選手 A**。
    *   Runtime 強制規定：**選手 B 不准搶** (即使他在 CPU 上轉也不行)，也不准自旋。
    *   鎖釋放時，直接 **Handoff (手遞手)** 交給隊列頭部的 A。
*   **代價**:
    *   犧牲了吞吐量 (因為強迫熱 CPU 的 B 去睡覺)，但換來了 **Tail Latency** 的穩定 (保證沒人餓死)。

鎖的代價是一個光譜：

1.  **不需鎖 (Lock-Free / Atomic)**:
    *   代價: 0 (或極低)。
    *   範例: `mcache` 分配。

2.  **輕量鎖 (Spin Lock / Mutex Fast Path)**:
    *   代價: 小。浪費一點點 CPU 空轉。
    *   狀態: **User Space**。

3.  **重量鎖 (Futex / Mutex Slow Path)**:
    *   代價: **大**。
    *   狀態: **Kernel Space (驚動 OS)**。
    *   特徵: Context Switch + Cache Miss。

### 4. 等待隊列的順序 (Queue Ordering)
關於「誰先拿到鎖」，Go Runtime 的內部隊列 (Treap) 本身是 **Strict FIFO (先進先出)** 的。
*   **純粹佇列 (No Spinners)**: 如果沒有新來的 Goroutine (CAS) 在門口自旋插隊，Runtime 絕對是依照順序叫醒 Goroutine 並把鎖交給它。這完全不依賴 OS，而是 Go Runtime 自己的秩序。
*   **唯一變數 (The Interruption)**: 順序之所以會被打亂，純粹是因為 **正常模式** 允許「新來的插隊」。如果沒有這些插隊者，`sync.Mutex` 就是一個標準的 FIFO 鎖。

---

## 4. 第四樂章：驗屍報告 (Profiling)

既然鎖的代價這麼大，我們怎麼知道程式現在是卡在鎖上？還是單純跑得慢？
**`pprof`** 提供了兩個專用視角，請務必學會看這兩個指標：

### 1. 抓「卡住」 (Block Profile)
*   **指令**: `go tool pprof http://.../block`
*   **專注指標**: **等待時間 (Delay)**。
*   **真實案例 (The IO Logger)**:
    *   **情境**: 你寫了一個 Logger，在 `Lock()` 之後做了 **寫入硬碟 (Disk IO)** 的動作，然後才 `Unlock()`。
    *   **後果**: 硬碟很慢 (ms 級)，導致成千上萬個 Goroutine 都在排隊睡覺等鎖。
    *   **特徵**: CPU 使用率很低 (都在睡)，但 API 回應時間極長。
    *   **💡 解法**: 千萬不要在鎖裡面做 IO。請改用 **Async Logger (如 zap/zerolog)**，它們會把 Log 先寫到 Memory Buffer (極快)，再由背景 Thread 慢慢寫入硬碟。
*   **看到什麼 (Example)**:
    ```bash
    (pprof) top
    Flat   Flat%   Sum%    Cum     Cum%
    0s     0.00%   0.00%   30min   30.0%  sync.(*Mutex).Lock
    ```
*   **數據解讀**:
    *   **Cum (30min)**: 這不是說一次睡了 30 分鐘，而是這段採樣時間內，所有 Goroutine **加總** 睡了 30 分鐘。
    *   **診斷**: 如果這個數字佔比 (Cum%) 很高，但 CPU 使用率不高，代表你的程式是 **IO Bound** 或 **Lock Bound** (都在等)。

### 2. 抓「競爭」 (Mutex Profile)
*   **指令**: `go tool pprof http://.../mutex`
*   **專注指標**: **競爭次數 (Contention Count)**。
*   **真實案例 (The Hot Map)**:
    *   **情境**: 你有一個全域的 `map[string]int` (例如用來記 Metrics)，每秒有 10 萬個請求都要去 `Lock()` 更新它。
    *   **後果**: 雖然 Map 更新很快 (ns 級)，但因為人太多，大部分時間都花在 CPU 上 **自旋 (Spinning)** 搶鎖。
    *   **特徵**: CPU 飆高 (User CPU)，但真正的業務邏輯產出卻不高 (都在搶鎖)。
*   **看到什麼 (Example)**:
    ```bash
    (pprof) top
    Flat   Flat%   Sum%    Cum     Cum%
    1.2M   10.0%   10.0%   1.2M    10.0%  sync.(*Mutex).Unlock
    ```
*   **數據解讀**:
    *   **Cum (1.2M)**: **注意！這裡的單位是次數 (Count)**。代表這段時間內，有 120 萬次 `Lock()` 請求因為搶不到鎖而被迫排隊或自旋。
    *   **診斷**: 如果這個數字很高 (百萬級)，說明該鎖是 **熱點 (Hotspot)**。即便每次只卡 1ns，但 100 萬次累積起來的 **CPU 自旋成本** 是巨大的。這就是典型的 **High Contention**。

### 結論：如何判斷是否為「鎖」的問題？
當你發現 API 變慢，且 **懷疑是 Lock 造成** 時，請用以下矩陣確認：
1.  **CPU 低 + Block Profile 有 Mutex**: 確診為 **睡著了 (Blocking)**。代表有人長時間持鎖 (例如在鎖內做 IO)。
2.  **CPU 高 + Mutex Profile 有 Mutex**: 確診為 **打起來了 (Contention)**。代表搶鎖太激烈 (例如 Hot Map)。
*(註：如果 CPU 低，且 Block Profile 裡沒看到 Mutex 相關堆疊，那通常是純粹的 Network/DB IO 等待，與鎖無關)*
