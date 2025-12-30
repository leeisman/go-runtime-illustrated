# Book 3.9: 地圖的秘密 (sync.Map)

我們在 **Book 3.8** 看到了一場災難：
當 64 個核心同時用 `RWMutex` 讀取同一個 Map 時，`readerCount` 成了那個不幸的乒乓球，導致效能雪崩。

Go 團隊意識到了這個問題，於是他們設計了 **`sync.Map`**。
它的設計哲學只有一句話：**「用空間換時間，用複雜換速度。」**

---

## 1. 第一樂章：雙層地圖 (The Two Maps)

`sync.Map` 的結構體裡，其實藏了 **兩個 Map**。
我們先看看你平常是怎麼用它的：

```go
// 你的代碼：
var m sync.Map
m.Store("K1", "V1") // 寫入 (可能進 dirty)
v, _ := m.Load("K1") // 讀取 (優先查 read)
```

**底層真相 (Runtime Struct)**:

```go
type Map struct {
    mu Mutex
    
    // 1. 唯讀快取 (無鎖)
    // 負責處理大部分的 m.Load()。
    // [重點]: 這裡雖然是 atomic.Value，但它裡面存的是 `*readOnly` 結構體指標。
    read atomic.Value 
    
    // 2. 髒地圖 (有鎖)
    // 這是一個普通的 map，所有新寫入的資料都在這裡。
    // 如果 read 裡找不到 key，就會來這裡找。
    dirty map[any]*entry
    
    misses int // 計數器
}

// [解密] Read Map 的真面目
type readOnly struct {
    m       map[any]*entry // 這才是真正的原生 Map！
    amended bool           // 如果 dirty 裡有 read 沒有的新資料，這裡會是 true
}

// [解密] Value 的容器
type entry struct {
    // 指向真正 value 的指標
    // 透過 CAS (atomic.CompareAndSwapPointer) 來修改這個 p，就能達到無鎖更新
    p unsafe.Pointer 
}
```

這就像是餐廳有兩個倉庫：
*   **Read Map**: 前台保溫箱。服務生 (Goroutine) 可以隨便拿，不用問廚師 (Lock)。
*   **Dirty Map**: 後廚。如果要拿新菜，必須進去問廚師，要排隊 (Lock)。

---

## 2. 第二樂章：寫入流程 (Store - The Dual Write)

當你呼叫 `Store(key, value)` 時，流程會根據 **「這是不是新 Key」** 而分岔成兩條路。

### 情境一：第一次寫入 (The First Time / New Key)
這是最單純、但也最慢的路徑。
1.  **查 Read Map**: `e, ok := read.m[key]`。
2.  **失敗 (ok=false)**: 確認這真的是新資料。
3.  **進庫房 (Dirty Map)**:
    這時候必須走慢車道：
    ```go
    m.mu.Lock() // 1. 上鎖 (因為 Dirty Map 不安全)
    
    // 2. Double Check (防止在等鎖的時候，別人已經把这个 key 加進去了)
    read, _ = m.read.Load().(readOnly)
    if e, ok = read.m[key]; ok {
        // 哎呀，別人幫我加了？那就變成情境二 (tryStore)
    } else {
        // 3. 真的是新 Key，寫入 Dirty Map
        m.dirty[key] = newEntry(value) 
    }
    
    m.mu.Unlock() // 4. 解鎖
    ```
    *   **代價**: 這條路徑就是普通的 `Mutex + Map`，沒有任何神奇優化。

### 情境二：老客戶更新 (The Update / Existing Key)
這是我們夢寐以求的 **無鎖路徑 (Fast Path)**。

**前置條件 (The Missing Link)**:
你可能會問：「剛剛情境一不是只寫入 Dirty Map 嗎？它什麼時候跑進 Read Map 的？」
答：這需要**時間**。
這個 Key 必須經歷多次的 `Load` (Miss) -> **觸發第四樂章的升級 (Promote)** -> 之後才會出現在 Read Map 裡。
一旦升級完成，以後對這個 Key 的更新就會走這一條捷徑：

如果這個 Key 早就存在，而且已經從 Dirty **晉升 (Promote)** 到了 Read Map：

1.  **查 Read Map**: `e, ok := read.m[key]`。
2.  **成功 (ok=true)**: 順利拿到 `entry` 指標 (中間人)。
3.  **原子更新 (CAS)**:
    *   Runtime 直接用 `atomic.CompareAndSwapPointer` 修改 `entry` 裡面的 `p` 指標。
    *   **完全不用加鎖**，也不用進 Dirty Map。
    
    ```go
    // 代碼示意 (entry.tryStore):
    // 只有在 Read Map 找到時 (情境二)，才會執行這段：
    for {
        p := atomic.LoadPointer(&e.p)
        if p == expunged {
            return false // (例外：已被標記刪除，需回到有鎖路徑復活它)
        }
        if atomic.CompareAndSwapPointer(&e.p, p, &newValue) {
            return true // 成功！
        }
    }
    ```

**結論**: **新 Key 一定是先去 Dirty Map 報到的。** 這就是為什麼接下來的 Load 會找不到它。

---

## 3. 第三樂章：查找流程 (Load - The Happy Path)

現在你知道新資料都在 Dirty Map 了，我們來看讀取流程。
當你呼叫 `Load(key)` 時：

### 1. 先查 Read Map (無鎖快車道)
這是最理想的路徑。
*   Runtime 使用 `atomic.Load` 檢查 `read` 快取。
*   **情境 A (Hit)**: 找到了！直接返回 value。全程無鎖，極速。
*   **情境 B (Miss)**: 找不到。這可能是因為這個 key 是最近才透過 `Store` 新加的，還躺在 `dirty` 倉庫裡，還沒升級到 `read` 前台。

### 2. 轉查 Dirty Map (有鎖慢車道)
既然快車道沒有，只好走慢車道去後廚找。
*   **加鎖**: `mu.Lock()` (必須加鎖，因為 dirty map 不是並發安全的)。
*   **Double Check**: 再查一次 `read` (防止等待鎖的期間剛好有人升級了)。
*   **查 Dirty**: 從 `dirty` map 裡找。
    *   **找到了**: 返回 value。
    *   **還是沒有**: 那就是真的不存在。
*   **懲罰機制 (Misses++)**:
    *   因為這次查詢被迫加了鎖，Runtime 會把 `misses` 計數器 +1。
    *   **潛台詞**: "Read Map 太舊了，害我還要鎖一下才找到，記上一筆。"
*   **解鎖**: `mu.Unlock()`。

**(如果 Misses 次數夠多，就會觸發第四樂章的「數據搬遷」)**。

---

## 4. 第四樂章：數據搬遷 (Promote)

那新數據怎麼從後廚 (Dirty) 跑到前台 (Read) 呢？

*   **觸發條件**: 當 `misses` (查不到的次數) 累積到一定程度 (通常等於 dirty map 的長度)。
*   **動作 (Promotion)**:
    *   Runtime 判斷：「大家老是來後廚找這個 key，太慢了。」
    *   **原子升級**: 它直接把整個 `dirty` map 的指針，**原子替換 (Atomic Store)** 給 `read` map。
    *   `dirty` 變成 `nil` (清空)。
    *   `misses` 歸零。

**結果**: 那些原本要鎖的新數據，瞬間變成了 **Read Only** 的無鎖數據。

---

## 5. 第五樂章：刪除的藝術 (Expunged)

這裡有一個最難懂的概念：**`expunged` (已刪除)**。

當你 `Delete(key)` 時，Go 不會真的把 key 從 map 刪掉 (因為 Read Map 是唯讀的，不能刪 key)。
它會做一個 **邏輯刪除**：
把那個 key 具體指向的 value 指針，改成一個特殊的標記：`expunged`。

*   **讀到 Expunged**: 告訴你「找不到」。
*   **寫入 Expunged**: 如果你又要 `Store` 這個已刪除的 key，它會把標記拿掉，重新啟用這個 key。

---

## 6. 第六樂章：總結與適用場景 (Summary)

`sync.Map` 不是萬能藥，它有非常強烈的性格。

### 1. 什麼時候該用？ (Best Practice)
*   **讀多寫少 (Read Heavy)**: 這是基本盤。
*   **固定名單更新 (Stable Keys)**: 
    *   **場景**: 例如全班 50 個學生，名單固定，只是不斷更新分數。
    *   **原因**: 因為 Key 都在 Read Map 裡了，更新走的是 **CAS 捷徑 (Fast Path)**，完全無鎖。

### 2. 什麼時候 **不** 該用？ (Keep Away)
*   **無限增長的名單 (Growing Keys)**:
    *   **場景**: 例如 Session ID、Request ID、遊客記錄 (一直在 `Store` 新的 Key)。
    *   **原因**: 新 Key 一定要進 Dirty Map (有鎖)。這會導致 Dirty Map 壓力巨大，且頻繁迫使 Read Map 重建 (Promotion)。
    *   **後果**: 效能比普通的 `Mutex + Map` 還要差。

**一句話總結**:
如果你是在 **「維護一個既有的狀態集合」**，用 `sync.Map`。
如果你是在 **「一直產生新資料」**，請用 `Mutex` 或 `Sharded Map`。
