# Book 3.8: 讀寫的藝術 (RWMutex)

在 **Book 3.6** 中我們學到了 Mutex (互斥鎖)。它很公平，但也「笨重」：即使大家只是想讀個資料，並沒有要改，它也強迫大家排成一直線。

這在「讀多寫少」的場景 (如 Cache, Config) 非常浪費。
於是，Go 提供了 **`sync.RWMutex`**：允許多個讀者同時進入，但寫者必須獨佔。

它底層是怎麼做到的？其實它只是 **Mutex** 加上 **Atomic 計數器** 的巧妙組合。

---

## 1. 第一樂章：數據結構 (The Anatomy)

RWMutex 的結構體裡藏著這些秘密：

```go
type RWMutex struct {
    w           Mutex  // 一把普通的互斥鎖 (給寫者用的)
    writerSem   uint32 // 寫者等待信號量 (Writer Waiting)
    readerSem   uint32 // 讀者等待信號量 (Reader Waiting)
    readerCount int32  // 當前有多少讀者 (關鍵！)
    readerWait  int32  // 寫者來了之後，還剩多少舊讀者沒走
}
```

---

## 2. 第二樂章：讀者的特權 (RLock - Fast Path)

當你呼叫 `RLock()` 時：

### 1. 動作 (Atomic Add)
```go
// 讀者人數 +1
if atomic.AddInt32(&rw.readerCount, 1) < 0 {
    // 哎呀，有寫者在等！(Slow Path)
    // 去 readerSem 睡覺
}
// 成功！可以讀了。
```

### 2. 解析
*   只要 `readerCount` 是正數，代表沒有寫者。
*   **代價極低**：僅僅是一個 `atomic.Add`。這意味著 100 個讀者可以「並行」拿到鎖，完全不用互搶 Mutex。**這是 RWMutex 效能高的主因。**

---

## 3. 第三樂章：寫者的進擊 (Lock - Hard Path)

當你呼叫 `Lock()` (Write Lock) 時，事情就複雜了。

### 1. 搶麥克風 (Mutex Lock)
寫者必須先搶到那把唯一的 `rw.w` 鎖。
這保證了**同一時間只有一個寫者**能發起挑戰。

### 2. 宣戰 (Invert Reader Count)
寫者要做一件很壞的事：把 `readerCount` 變成 **負數**。
```go
// 原本 count = 5 (有5個讀者)
// 減去一個巨大的常數 (1 << 30)
// 新 count = 5 - 1073741824 = 負數！
r := atomic.AddInt32(&rw.readerCount, -rwmutexMaxReaders)
```
這有什麼用？
*   **嚇阻新讀者**：之後新的讀者在 `RLock` 時，加 1 後發現還是負數，就知道「有寫者來了 (Writer Pending)」，於是乖乖去排隊，不敢進來插隊。
*   這是為了避免 **寫者飢餓 (Writer Starvation)**：如果不嚇阻新讀者，新讀者一直來，寫者可能永遠拿不到鎖。

### 3. 清場 (Wait for Old Readers)
```go
// 剛剛算出 r = 5 (還有5個舊讀者在裡面)
if r != 0 && atomic.AddInt32(&rw.readerWait, r) != 0 {
    // 還有舊讀者沒走光！
    // 寫者去 writerSem 睡覺 (Parking)
}
```
寫者雖然宣戰了，但他很紳士。他會等那 5 個已經在裡面的讀者看完。
等到最後一個舊讀者離開時 (`RUnlock`)，那個讀者會負責去叫醒寫者。

---

## 4. 第四樂章：釋放與喚醒 (Unlock)

### 1. 寫者釋放 (Unlock)
*   把 `readerCount` 加回去 (變回正數)。
*   **廣播**：喚醒所有剛剛被擋在門外的 **新讀者們** (`readerSem`)。
*   大家一起衝進去讀。

### 2. 讀者釋放 (RUnlock)
*   `readerCount - 1`。
*   如果是最後一個讀者，而且發現有寫者在睡覺，就去叫醒寫者 (`writerSem`)。

---

## 5. 第五樂章：總結與實戰 (Summary)

**Q: RWMutex 底層用了什麼？**
A: 它用了 **1 個 Mutex** (保護寫者) + **2 個 Semaphore** (休眠排隊) + **Atomic Count** (極速計數)。

**Q: 什麼時候該用 RWMutex？**
A: 只有在 **「讀遠多於寫 (Read >> Write)」** 的時候 (例如 90% 讀，10% 寫)。
*   如果寫入很頻繁，RWMutex 的維護成本 (Atomic Add + Semaphores) 反而比單純的 Mutex 更高。

### Pro Tip: RLock 不是免費的 (The Hidden Cost)
很多初學者認為「RLock 只是讀，應該跟 atomic load 一樣快」，**這是一個巨大的誤區**。

| 操作 | 行為 (Metaphor) | 代價 (Cost) | 並發瓶頸 (Bottleneck) |
| :--- | :--- | :--- | :--- |
| **Atomic Load** | **隱形人**：安靜地看一眼資料，不留任何痕跡。 | **0** (無副作用)。 | **無**。100 萬人同時看都沒問題 (Shared Cache)。 |
| **RWMutex.RLock** | **打卡機**：看資料前，必須去櫃檯的計數器上**刻一個「+1」**。 | **有** (修改記憶體)。 | **高**。100 萬人雖然只是讀，但都要搶著去修那個計數器，導致 Cache Thrashing。 |

**結論**:
*   如果你追求極致的讀取性能 (Read-Heavy)，請用 `atomic.Value` (Copy-On-Write)。
*   `RWMutex` 比較適合「讀雖然多，但還沒多到會把計數器打爆」的場景。
