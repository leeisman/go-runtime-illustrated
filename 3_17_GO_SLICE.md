# Book 3.17: The Slice (切片 - 動態陣列的奧義)

Slice 是 Go 最常用的資料結構之一，也最容易讓人誤會。
這一章會把 slice header、backing array、append 擴容與 sub-slice 陷阱拆開，讓你知道它什麼時候便宜，什麼時候會偷偷保住大量記憶體。

---

## 1. 第一樂章：結構 (The Structure)

在 Go 裡，Slice 並不是一個「陣列」，它只是一個 **描述符 (Header)**，用來描述底層陣列的一部分。
它的結構非常輕量，只有三個欄位：

```go
type SliceHeader struct {
    Data uintptr // 指向底層陣列的指針
    Len  int     // 目前長度 (len)
    Cap  int     // 最大容量 (cap)
}
```

這意味著：
1.  **傳遞成本低**: 傳遞一個 Slice 給函數，只會複製這 3 個字 (24 bytes)，不會複製底層的大陣列。
2.  **共享記憶體**: 多個 Slice 可以指向同一個底層陣列。

### VS Array & String
*   **Array (`[5]int`)**: 是 **值類型 (Value Type)**。賦值或傳參會 **完整複製** 整個陣列 (慢)。
*   **String (`string`)**: 是一個 **唯讀的 Slice**。結構只有 `Data` 和 `Len` (沒有 Cap，因為不可變)。

---

## 2. 第二樂章：擴容 (The Growth)

當你呼叫 `append(s, val)` 時，如果 `len < cap`，直接寫入；如果 `len == cap`，就需要 **擴容 (Grow)**。

**舊版擴容 (Before 1.18)**:
*   `cap < 1024`: 翻倍 (2x)。
*   `cap >= 1024`: 每次增加 25% (1.25x)。

**新版擴容 (After 1.18)**:
為了讓成長更平滑，Go 改用了更複雜的公式。
*   **小 Slice**: 依然是翻倍。
*   **大 Slice**: 成長幅度會逐漸從 2.0x 遞減到 1.25x，而不是在 1024 處突然變慢。
*   **記憶體對齊**: 計算出的容量會再向上取整，以符合記憶體分配器 (Allocator) 的 class size (例如 48, 64, 80 bytes)，避免碎片浪費。

**關鍵點**:
擴容意味著 **搬家 (Reallocation)**。
Runtime 會去 Heap 申請一塊更大的新陣列，把舊資料 **Copy** 過去。
**所以 `append` 之後，Slice 的 `Data` 指針通常會變！**

---

## 3. 第三樂章：陷阱 (The Pitfalls)

Slice 最強大的功能是「切」(`s[low:high]`)，但這也是最危險的地方。

### 陷阱 1: 記憶體洩漏 (Memory Leak)
```go
func getHeader() []byte {
    bigData := make([]byte, 100*1024*1024) // 100MB
    // ... read file into bigData ...
    return bigData[:10] // 只返回頭 10 bytes
}
```
**發生了什麼事？**
雖然你只返回了 10 bytes 的 Slice，但由於它 **指向** 那個 100MB 的底層陣列，**整個 100MB 都無法被 GC 回收**！
這就是經典的 Memory Leak。

**解法**: 用 `copy`。
```go
small := make([]byte, 10)
copy(small, bigData[:10])
return small // 新的獨立陣列，舊的 100MB 可以被回收
```

### 陷阱 2: 共享修改 (Shared Modification)
```go
a := []int{1, 2, 3, 4, 5}
b := a[0:2] // b = [1, 2], cap=5
b = append(b, 999)
```
**發生了什麼事？**
`b` 還有容量 (cap=5)，所以 `append` **不會擴容**，而是直接寫入底層陣列的 Index 2。
**結果**: `a` 變成了 `[1, 2, 999, 4, 5]`！
原本 `a[2]` 的 `3` 被 `b` 覆蓋掉了。這在併發或共享邏輯中非常致命。

**解法**: 使用 **Full Slice Expression** (`a[0:2:2]`)。
這樣可以強迫 `b` 的 `cap` 也等於 2。再 `append` 時就會觸發擴容 (Copy 到新陣列)，不會影響原本的 `a`。

### 陷阱 3: Range 中刪除 (Delete inside Range) - 危險！
```go
for i, v := range a {
    if v == target {
        a = append(a[:i], a[i+1:]...) // 刪除元素
    }
}
```
**發生了什麼事？**
1.  **跳過檢查 (Skipping)**: 刪除 `i` 後，原本的 `i+1` 遞補變成了 `i`。但 `range` 下一步會跑 `i+1`，導致這個遞補過來的元素被跳過檢查。
2.  **崩潰 (Panic)**: `range` 在開始時就記住了原本的長度 (例如 5)。如果您刪了一個變 4，當迴圈跑到 Index 4 時，存取 `a[4]` 會導致 **Index Out of Range**。

**正確解法**: 請使用 **[原地過濾 (In-place Filter)]** (5. 常見演算法操作 - Trick 4)。

---

## 4. 第四樂章：效能 (The Performance)

1.  **預分配 (Pre-alloc)**:
    如果你知道大概要裝多少，`make([]int, 0, 100)`。
    這能避免多次擴容帶來的 `malloc` 和 `memmove` 開銷。

2.  **清空 Slice**:
    `s = s[:0]`。這只是把 Header 裡的 `Len` 歸零，底層記憶體還在，下次 `append` 可以直接復用，完全零分配。

3.  **複製**:
    使用 `copy(dst, src)` 是最快的，它是組合語言級別的 `memmove`。

---

## 5. 第五樂章：演算法 (The Algorithms)

Go 的 Slice 非常靈活，透過 `append` 和切片語法，可以實現各種資料結構的操作：

### 1. 刪除元素 (Delete / Cut)
這裡有兩種做法，取決於您是否在乎順序：

*   **保持順序 (Slow - O(N))**:
    ```go
    a = append(a[:i], a[i+1:]...)
    ```
    因為要把 `i` 後面的資料全部往前搬 (Memmove)，若是大陣列會很慢。

*   **快速刪除 (Fast - O(1))**:
    如果不介意順序，可以直接由最後一個補上：
    ```go
    a[i] = a[len(a)-1] // 用最後一個覆蓋它
    a = a[:len(a)-1]   // 切掉最後一個
    ```
    這完全不需要搬移記憶體，效率極高。

    #### 圖解變化 (Under the Hood)
    假設 `a = [A, B, C, D, E]`, Len=5, Cap=8。刪除 Index 2 (C)：
    1.  **Swap**: 把最後一個 (E) 填入 (C) 的位置。底層變成 `[A, B, E, D, E]`。
    2.  **Cut**: `Len` 變為 4。`a` 現在看是 `[A, B, E, D]`。
        *   注意：底層 Index 4 的位置還留著一個 **「幽靈資料 (E)」**，但它已經在 Len 之外了。
    3.  **Append 'F'**: 如果你再 `append(a, 'F')`...
        *   因為 `Len(4) < Cap(8)`，不需要擴容。
        *   直接覆蓋 Index 4 (幽靈 E)。底層變成 `[A, B, E, D, F]`。
        *   `Len` 變回 5。整個過程 **零分配 (Zero Allocation)**。

### 2. 堆疊操作 (Stack)
*   **Push**: `a = append(a, x)`
*   **Pop**: 
    ```go
    x := a[len(a)-1] // 取出最後一個
    a = a[:len(a)-1] // 縮短長度
    ```

### 3. 隊列操作 (Queue)
*   **Enqueue**: `a = append(a, x)`
*   **Dequeue (Shift)**:
    ```go
    x := a[0]     // 取出第一個
    a = a[1:]     // 指針往後移
    ```
    *   **注意**: 這樣做會導致底層陣列的前面部分變成無法回收的廢棄空間。如果是長隊列，建議使用 circular buffer 或 `container/list`，或者定期 `copy` 重整。

### 4. 原地過濾 (In-place Filter)
這是最高效的技巧，完全不申請新記憶體，直接在原陣列上過濾資料：
```go
func filterOdd(nums []int) []int {
    b := nums[:0] // b 指向同一個底層陣列，但 len=0
    for _, x := range nums {
        if x%2 == 0 {
            b = append(b, x) // 覆蓋原陣列的前端
        }
    }
    return b
}
```

---

## 6. 第六樂章：總結 (The Finale)

Slice 是 Go 最偉大的發明之一，它用極低的成本實現了動態陣列。
*   **結構**: 3 個字的 Header (Data, Len, Cap)。
*   **擴容**: 平滑增長，但會改變地址。
*   **風險**: 注意大陣列的小切片 (Memory Leak) 和共享陣列的覆蓋 (Shared Mutation)。

掌握了 Slice，你就掌握了 Go 數據操作的核心。
