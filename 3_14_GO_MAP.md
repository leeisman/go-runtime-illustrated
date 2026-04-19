# Book 3.14: The Map (雜湊表)

Map 看起來像是語言內建的普通容器，但底下其實是一套會成長、搬遷、處理碰撞的動態索引系統。
這一章會從 `hmap` 與 bucket 開始，看懂查找、寫入、擴容與並發限制。

---

## 1. 第一樂章：隱喻 (The Metaphor - The Limitless Archive)

Go 的 Map 就像是一座 **「自動擴展的檔案館」**。
*   **Key**: 當你想找資料時，你會給管理員一個關鍵字 (Hash Key)。
*   **Bucket (桶子)**: 管理員不只有一個櫃子，而是有一排櫃子。每個櫃子只能放 **8 份文件**。
*   **Hash Function**: 管理員用數學公式算出你的文件應該在第幾個櫃子。

---

## 2. 第二樂章：結構 (The Structure - hmap & bmap)

Map 的表皮是 `hmap`，骨子裡是一堆 `bmap` (Bucket)。

```go
type hmap struct {
    count     int    // 元素總數 (len)
    B         uint8  // 櫃子數量的對數 (2^B 個 buckets)
    buckets   unsafe.Pointer // 指向目前的櫃子陣列
    oldbuckets unsafe.Pointer // 指向擴容前的櫃子 (搬遷時用)
    ...
}
```

### 2. bmap (The Bucket - 存數據的地方)

`hmap` 只是個目錄，真正的數據都存在 **Bucket (`bmap`)** 裡。每個 Bucket 固定能存 **8 個 Key/Value 對**。

雖然 Go 源碼裡隱藏了細節，但對於一個 `map[int64]int8`，編譯器在背後生成的真實結構如下 (這是一個實實在在的 Struct)：

```go
// Runtime 真實結構 (以 map[int64]int8 為例)
type bmap struct {
    tophash  [8]uint8  // 8 bytes (特徵值)
    keys     [8]int64  // 64 bytes (8個 Key 連在一起)
    values   [8]int8   // 8 bytes  (8個 Value 連在一起)
    overflow uintptr   // 8 bytes  (溢出指針)
}
// 總大小: 8 + 64 + 8 + 8 = 88 bytes
```

#### 關鍵優化：鍵值分離 (Key/Value Separation)
為什麼不把 `key/value` 放一起變成 `pair`？因為要 **省記憶體 (Alignment Padding)**。

舉例：`map[int64]int8`
*   **如果交錯 (Naive)**: `Key(8) | Val(1) | Padding(7) | Key(8)...` -> 每個 Slot 浪費 7 bytes。
*   **Go 的做法 (Separated)**: 把所有 Key 放一起，所有 Value 放一起。
    `Key(8)...Key(8) | Val(1)...Val(1)` -> **完全零浪費**。

#### 特例說明：String 與 Huge Value
1.  **String Key**: 
    Go 的 `string` 底層是 16 bytes 的 `StringHeader` (Ptr + Len)。所以在 Bucket 裡，字串 Key 永遠佔用固定的 16 bytes。真正的文字內容存在外部 (Heap)。
2.  **Huge Value (>128 bytes)**:
    如果 Key 或 Value 的大小超過 128 bytes，Go 會自動轉為 **存指針 (Indirect)**。Bucket 裡只存指向該物件的指針，本體分配在 Heap 上。這保證了 Bucket 不會變得過於巨大而影響 Cache 效能。

#### 3. Key 的型別限制 (Struct Key)
*   **Struct 可以當 Key 嗎？** 可以！只要 Struct 裡的所有欄位都是「可比較 (Comparable)」的 (例如沒有 Slice/Map/Func)。
*   **如何比對？** Runtime 會逐一比對 Struct 內的所有欄位。只要有一個欄位不同，就是不同的 Key。
*   **應用**: 這非常適合做 **複合鍵 (Composite Key)**。例如 `Key{Region: "US", ID: 101}`。
    *   **VS `fmt.Sprintf`**: 千萬別用 `fmt.Sprintf("%s_%d")` 來拼 Key！
    *   **Sprintf 缺點**: 會產生新的字串物件 (Heap Allocation -> GC 壓力)，且需要解析格式與反射 (慢)。
    *   **Struct 優點**: 分配在 Stack 上 (Zero Alloc)，且 Runtime 對 Struct Hashing 有專門優化 (快)。
*   **注意 (Value Copy)**: Key 是以 **值複製 (Copy)** 的方式存入 Bucket。這意味著：
    1.  存入後修改原變數，Map 內的 Key 不會改變 (保證了 Hash 的穩定性，雖然 Go 本也不許修改 Map Key)。
    2.  如果 Struct 很大，每次操作都會有記憶體複製 (Memcpy) 的開銷。

**Bucket 記憶體佈局圖**:
```text
+-------------------------+
| tophash (8 bytes)       | 0x00 - 0x07
+-------------------------+
| key 0 (type dependent)  | 
| ...                     |
| key 7                   |
+-------------------------+
| value 0 (type dependent)|
| ...                     |
| value 7                 |
+-------------------------+
| overflow pointer        | -> 指向 Next Bucket
+-------------------------+
```

---

## 3. 第三樂章：操作 (The Operation - Init, Put, Get)

有了結構，我們來看看 Go 是怎麼存取資料的。這是一個從建立到讀寫的完整旅程。

### 1. 初始化 (Initialization)
當你執行 `m := make(map[string]int, 100)` 時：
1.  **計算 B**: 根據提示 (100)，Runtime 算出需要 **B=4** (16 個桶子)。
    *   **原因**: Go 的負載因子閾值是 6.5。
    *   計算: `16 * 6.5 = 104`，足以容納 100 個元素，所以不需要擴容。
2.  **分配記憶體**: 向 Heap 申請一個 **連續的 Bucket Array** (大小為 16 * bmapSize)。`hmap.buckets` 指向這個陣列開頭。
3.  **Hash Seed**: 生成一個隨機數種子 (`hmap.hash0`)，這能防止 Hash Collision 攻擊 (讓每次執行的 Hash 結果都不同)。

### 2. 定位 (Locate - Shared)
不管是 Get 還是 Put，第一步都是算 Hash 並找到 Bucket。
*   假設 Key 是 **"hello"**，Hash 值算出來是 `0xAF00...0005`。
*   因為初始化算出的 **B=4** (16 個桶子)，Mask 是 `15` (`1111`)。
*   `bucket_idx = hash & 15` = **`5`**。
*   **動作**: Target 是 **第 5 號 Bucket**。

### 3. 寫入流程 (Put)
目標：寫入 "hello" -> 100。
1.  **掃描**: 遍歷 Bucket 5 的 8 個 Slot。
2.  **查重**: 檢查有無 Tophash=`0xAF` 且 Key="hello" 的？
    *   (假設現在是空的，沒找到)。
3.  **找位**: 記錄下 Slot 0 是第一個空位 (Empty)。
4.  **寫入 (記憶體操作)**:
    因為 `bmap` 是 KV 分離的，Runtime 實際上分別操作了三個陣列：
    *   `bucket.tophash[0] = 0xAF`
    *   `bucket.keys[0] = "hello"`
    *   `bucket.values[0] = 100`

### 4. 讀取流程 (Get)
目標：讀取 "hello"。
1.  **掃描 (Batch Scan)**: 遍歷 Bucket 5 的 8 個 Slot。
    *   **為什麼快？**: 雖然是迴圈，但 CPU 利用 **分支預測** 會將這 8 次比對視為一個 **連續批次 (Batch)** 執行，中間沒有停頓，速度極快。這不只是演算法的巧妙，**更是硬體架構 (Pipeline) 的勝利**。
2.  **快篩**: 
    *   Slot 0 的 Tophash 是 `AF` 嗎？ **是！**
3.  **比對**: 
    *   `keys[0]` 是 "hello" 嗎？ **是！** (注意：這裡是去 keys 陣列拿)
4.  **結果**: 回傳 `values[0]` (100)。

### 5. 刪除流程 (Delete)
目標：刪除 "hello"。
1.  **掃描**: 找到目標 Key (流程同 Get)。
2.  **清空**:
    *   將 `keys[0]` 和 `values[0]` 的記憶體 **清零 (Zeroing)**。
    *   **目的**: 幫助 GC。如果這裡面存的是指針，不清零的話，指向的物件永遠不會被回收 (Memory Leak)。
3.  **標記**:
    *   將 `tophash[0]` 設為 **`0x01` (emptyOne)**。
    *   **禁忌**: 不能設為 `0x00` (emptyRest)！因為後面 (Slot 1) 可能還有碰撞的 Key ("world")，如果設為 `0x00`，掃描到這裡就會中斷，導致找不到後面的 Key (斷鏈)。
4.  **代價**: 刪除操作 **不會釋放 Bucket 記憶體**。Bucket 只是變成了「充滿墓碑 (Tombstones)」的空殼。這就是為什麼需要「等量擴容」來回收空間。

### 6. 遍歷流程 (Range)
`for k, v := range m` 的行為有兩個特點：
1.  **隨機起點**: Runtime 每次都會生成一個隨機數，決定從哪個 Bucket 和哪個 Slot 開始遍歷。
    *   **目的**: 強制讓 Map 的迭代順序不可預測，防止使用者依賴順序 (因為擴容會打亂順序)。
2.  **擴容中的遍歷**: 如果 Map 正在擴容，Iterator 必須「穿梭時空」：
    *   當它遍歷到一個新 Bucket 時，如果發現這個 Bucket 的資料還在舊家 (`oldbuckets`)，它會跳回去舊 Bucket 遍歷。
    *   這保證了即使在搬家過程中，Range 也能準確地把所有 Key 走訪一遍 (不會重複、不會漏)。
3.  **邊歷邊改 (Mutation Safety)**:
    *   **Delete**: 在 Range 中刪除 Key 是 **安全且合法** 的 (不會像某些語言拋出 Iterator Invalid 異常)。
    *   **Put**: 在 Range 中新增 Key 是 **安全的但不可預測**。新增的 Key 可能會被遍歷到，也可能不會，完全隨機。
    *   **Concurrent**: 注意！如果多執行緒同時 Range 和 Modify，會直接觸發 **Panic** (`concurrent map iteration and map write`)。

### 7. 實戰演練：從零開始的 Map 之旅 (Scenario Walkthrough)
我們繼續觀察這個 **5 號 Bucket** 的演變。
設定：已存在 "hello" (Slot 0, Tophash=`AF`)。新來一個 **"world"** (Hash=`...35`, `35 & 15 = 5` 撞桶, Tophash=`B2`)。

#### Round 1: 發生碰撞 (Put "world")
1.  **定位**: 算出 Index=5，走進 Bucket 5。
2.  **掃描**:
    *   `tophash[0]`=`AF`. Key="hello" != "world". -> 跳過。
    *   `tophash[1]`=`00`. -> **找到空位 Slot 1！**
3.  **寫入**:
    *   `bucket.tophash[1] = 0xB2`
    *   `bucket.keys[1] = "world"`
    *   `bucket.values[1] = 200` (假設值)
4.  Bucket 狀態: `[AF, B2, 00, ...]`

#### Round 2: 塞爆了 (Overflow) 
假設 `tophash[0]~[7]` 都滿了。
1.  掃描發現全滿，且無重複 Key。
2.  **動作**: Runtime 去 Heap 申請一個 **新的 Bucket (Overflow Bucket)**。
3.  **串接**: 把 Bucket 5 的 `overflow` 指針指向這個新 Bucket。
4.  **寫入**: 把新 Key 寫入新 Bucket 的 `Index 0` (同樣分頭寫入 tophash/keys/values)。

#### Round 3: 讀取溢出資料 (Get C)
假設我們要讀取剛剛寫入溢出桶的 Key C。
1.  掃描 **主 Bucket** 5 的 Slot 0~7: 都沒找到。
2.  **跳轉**: 發現 `bucket.overflow` 不是 nil，跟著指針跳到 **溢出桶**。
3.  掃描 **溢出桶** 的 Slot 0:
    *   `tophash` 匹配，Key 匹配 "C"。
    *   **Bingo!** 回傳 Value。

### 8. 硬體視角：為什麼是 8？ (CPU Branch Prediction)
為什麼 Go 選擇 **8** 來做線性掃描，而不是用二分搜尋？這涉及到底層 CPU 的 **分支預測 (Branch Prediction)**。

1.  **指令流水線 (Pipeline)**:
    現代 CPU 就像工廠流水線，當它正在「執行」判斷指令 (`if`) 時，其實已經在「預取」後面的指令了。
2.  **預測 (Guess)**:
    遇到迴圈或判斷時，CPU 必須「猜」下一條要執行誰，才能保持流水線滿載。
3.  **猜錯的代價 (Pipeline Flush)**:
    如果猜錯了，CPU 必須把已經預取、解碼的指令 **全部作廢 (Flush)**，這會浪費 **10~20 個時脈週期**。

對於一個 `i=0...7` 的小迴圈，CPU 的預測器非常容易猜對「繼續執行」，所以這 8 次掃描幾乎沒有分支預測失敗的懲罰 (除了最後一次退出)。相比之下，二分搜尋或其他複雜邏輯會導致更多難以預測的跳轉，反而更慢。

---

## 4. 第四樂章：擴容 (The Growth - Evacuation)

當 Map 遇到壓力時，Go 會根據情況觸發兩種不同模式的 **擴容 (Resizing)**：

### 1. 翻倍擴容 (2x Growth)
*   **條件**: `Load Factor > 6.5` (平均每個 Bucket 裝超過 6.5 個元素)。
*   **原因**: 太擠了！Hash 碰撞太高，影響效能。
*   **對策**: 把 Bucket 數量 **翻倍** (B+1)，讓大家住寬敞一點。

### 2. 等量擴容 (Same Size Growth)
*   **條件**: `Overflow Buckets` 數量太多 (接近主 Bucket 數量)。
*   **原因**: 太亂了！雖然總元素不多 (Load Factor 低)，但因為頻繁刪除，導致 Bucket 裡充滿空洞，卻掛了一堆溢出桶。
*   **對策**: Bucket 數量 **不變**，但是觸發搬遷機制，把資料重新排列緊實 (Compact)，丟掉無用的溢出桶。

---



### 漸進式搬遷 (Incremental Evacuation)
擴建是一個大工程。如果一次把所有文件搬到新櫃子，程式會卡住很久 (Latency Spike)。
Go 採用了 **「螞蟻搬家」** 策略：

1.  **宣告擴建**: `hmap.B++`，把現在的 `buckets` 掛到 `oldbuckets`，建立雙倍大小的新 `buckets`。
2.  **不急著搬**: 此時，資料還在舊櫃子裡。
3.  **邊用邊搬**: 
    *   每次你對 Map 進行 **寫入** 或 **刪除** 操作時，管理員會「順便」幫忙搬移 **2 個 Bucket** 的舊資料到新家。
    *   這把擴容的成本攤平到了之後的每一次操作中，避免了單次巨大的延遲。

### 讀取時的雙重檢查
在搬遷期間 (`oldbuckets != nil`)：
*   **讀取**: 必須先去舊櫃子看，如果舊櫃子裡的資料還沒搬走，就讀舊的 (Evacuated 標記)。
*   **寫入**: 寫到新櫃子，並順手觸發搬遷。

---

## 5. 第五樂章：並發陷阱 (The Trap)

**Map 是不安全的 (Not Thread-Safe)！**
除了 `sync.Map` (Book 3.9) 之外，標準 Map **完全沒有鎖**。

*   **為什麼不加鎖？**: 為了效能。如果要加鎖，每次讀寫都要 Lock，對於單執行緒程式太慢了。
*   **Crash**: Go Runtime 內建了並發偵測。如果你在多個 Goroutine 同時讀寫 Map，程式會直接 **Fatal Panic** (無法 Recover)，因為這通常代表資料結構已經損壞 (Corruption)。
*   **解法**: 請配合 `sync.RWMutex` 使用 (Refer to Book 3.8)。

---

## 6. 第六樂章：總結 (Finale)

Go Map 是一個高度優化的 Hash Map，它的設計哲學是：
1.  **空間效率**: 通過 KV 分離排列減少 Padding。
2.  **查找效率**: 使用 Tophash 快速過濾。
3.  **擴容平滑**: 漸進式搬遷避免卡頓。

掌握了這些，你就知道為什麼 Map 在 Go 裡這麼快，但也這麼脆弱 (並發讀寫必死)。

---
**Next: [Book 3.15 Defer & Panic (異常處理)](./3_15_GO_DEFER.md)** (預計建立)
