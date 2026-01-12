# Book 5.1: Go BigCache (Zero GC Cache)

## 1. 第一樂章：GC 的恐懼 (The Fear of GC)

在 Go 語言中，寫一個 In-Memory Cache 最簡單的方法就是 `map[string]interface{}`。
但是，當你的 Cache 裡存了 **1000 萬** 個物件時，你會發現 API 變慢了，CPU 被吃光了。

### 為什麼？(Scanning Overhead)
因為 Go 的 GC 是一個勤奮的清潔工。每隔幾毫秒，它就要掃描 Heap 上所有的 **指標 (Pointer)**，確認物件是否存活。
*   **悲劇 Map**: `map[string]*Item`
    *   Key (`string`) 包含指標。
    *   Value (`*Item`) 包含指標。
    *   所以 GC 必須 **逐一檢查** 這 1000 萬個 Entry。這就是悲劇的來源。

### 那什麼時候 GC 不會怕？
Go 編譯器有一個 **黃金特例**：
*   **神蹟 Map**: `map[uint64]uint32` (或任何 Key/Value 都不含指標的 Map)
    *   編譯器知道這裡面只有純數字 (Scalar)，絕對不可能藏著指向其他物件的指針。
    *   因此，GC 會 **直接跳過整個 Map 的內容掃描**。不管 Map 有多大，掃描時間都是 **O(1)**。

**BigCache 的所有設計，就是為了把那個「悲劇 Map」變成這個「神蹟 Map」。**

---

## 2. 第二樂章：BigCache 的核心設計 (Core Design)
BigCache (以及 FreeCache) 的核心目標只有一個：**Zero GC Overhead**。
它怎麼做到的？三個絕招。

### A. 絕不存指標 (No Pointers) - 全鏈路去指標化

BigCache 的魔法不僅僅是把資料變成 `[]byte`，真正的關鍵在於它 **如何索引這些資料**。

#### 1. 致命的 Map
如果我們只做了一半優化，例如使用 `map[string][]byte`：
*   GC 還是要掃描每一個 **Key (string)**，因為 `string` 底層是指標。
*   GC 還是要掃描每一個 **Value (slice header)**，因為 slice header 也是指標。
*   結果：雖然好了一點，但 1000 萬個 Key 還是會讓 GC 掃到崩潰。

#### 2. 徹底的去指標 (The Real Magic)
BigCache 做了決絕的重構，把 Map 變成了這樣：
`map[uint64]uint32`

*   **Key**: 把 `string` Hash 成 `uint64`。 (消滅了 Key 指標)
*   **Value**: 把 `[]byte` 變成 `uint32` (Offset)。 (消滅了 Value 指標)

這張 Map 裡完全沒有指標，只有純數字。配合那個巨大的 `[]byte` 陣列 (也沒指標)，整個 Cache 結構對 GC 來說就是 **完全透明的**。

> **結論**: 把資料壓扁進陣列只是第一步，**把索引壓扁成純數字 Map (Index Transformation)** 才是 BigCache 真正的高明之處。


> **Map vs Array (記憶體視角)**:
> *   **Map**: 1000 萬個小物件，GC 看到的是 **1000 萬個待檢查的節點**，必須逐一訪問。
> *   **BigCache**: 1 個巨大的 `[]byte` 陣列。GC 看到的是 **1 個巨大的不透明黑盒子**。因為 `byte` 絕對不可能是指標，GC 知道這個黑盒子裡面不可能藏著「通往其他堆積物件的鑰匙」，所以 **直接跳過內容掃描**。
>
> 即使這個陣列有 10GB 大，GC 掃描它的時間也是 **接近 0**。

> **Q: 這樣 GC 不檢查它，不會造成 Memory Leak 嗎？**
> **A: 不會。**
> 這裡要區分 **標記 (Marking)** 和 **掃描 (Scanning)**：
> 1.  **標記 (存活確認)**: GC 仍然知道這個巨大的 `[]byte` 陣列是活著的 (因為 BigCache 指向它)。只要陣列有人用，它就不會被回收。
> 2.  **掃描 (內容檢查)**: 優化的是這一步。GC 決定 **不打開** 這個陣列去檢查裡面的內容。這是安全的，因為編譯器保證了 `[]byte` 裡只有純數值 (`byte`)，絕對不可能藏著「指向其他 Heap 物件的指針」。這盒子裡沒有通往外部的鑰匙，所以沒必要浪費時間去翻。
>
> 一旦 BigCache 本身被釋放，這個巨大的陣列也會因為「無人引用」而被 GC 正常回收。

### B. 分片鎖 (Sharding)
為了解決高併發寫入時的鎖競爭 (Lock Contention)，BigCache 內建了 **Sharding**。
它預設創建 1024 個小 Cache (Shards)。
*   `hash(key) % 1024` -> 決定去哪個 Shard。
*   每個 Shard 有自己獨立的 `sync.RWMutex`。
這跟我們在 System Design 裡的分庫分表邏輯一模一樣。

### C. 環形緩衝區 (Ring Buffer)
為了避免頻繁的 `make()` 和 `free()` (這會導致 GC 壓力)，BigCache 重複利用底層數組。
它使用 FIFO (先進先出) 的 Ring Buffer 機制來淘汰舊數據。
*   寫入新數據時，指標往後移。
*   如果遇到陣列尾部，繞回頭部覆蓋最舊的數據。
優點：**完全沒有 Allocation** (除了啟動時)。

---

## 3. 第三樂章：源碼解剖 (Source Code Anatomy)

BigCache 的秘密在於它如何管理那塊巨大的記憶體。

### A. 記憶體佈局 (Memory Layout - The Packet)
在 BigCache 的那個 `[]byte` 倉庫裡，每一筆資料並不是隨便亂丟的，而是經過嚴格打包。
當你存入 `Set("myKey", "myValue")` 時，它會在記憶體中寫入這一段連續的 Bytes：

```text
Memory Address: 0x0000       0x0008       0x000A    0x000F
               |------------|------------|---------|----------|...
Laytout:       | Timestamp  | Key Length |   Key   |  Value   |
               | (8 bytes)  | (2 bytes)  | (bytes) | (bytes)  |
               |------------|------------|---------|----------|...
Content:       | 167888...  |     5      | "myKey" | "myValue"|
```

這就是為什麼 GC 掃描不了它：對 CPU 來說，這只是一串連續的數字，沒有任何指向外部的指針。

### B. 寫入流程：Set (Append & Index)

```go
func (s *cacheShard) Set(key string, entry []byte) error {
    s.lock.Lock()
    defer s.lock.Unlock()

    currentTimestamp := uint64(time.Now().Unix())
    
    // 1. 打包數據 (Header + Key + Value)
    // 計算需要的空間 = Time(8) + KeyLen(2) + len(key) + len(val)
    // 並把這些數據序列化成一個 blob (二進位塊)
    blob := makeBlob(key, entry, currentTimestamp)

    // 2. 塞入 RingBuffer (那塊巨大的 []byte)
    // Push 會回傳這筆資料在陣列中的起始位置 (Offset)
    // 如果陣列滿了，它會自動覆蓋最舊的資料 (Wrap Around)
    index := s.entries.Push(blob)

    // 3. 更新索引 (Map Update)
    // 這裡是最關鍵的 Design: Map 的 Value 只存了一個 int (Offset)
    // map[uint64]uint32
    hashKey := s.hash.Sum64(key)
    s.hashmap[hashKey] = index 

    return nil
}
```
**底層動作**:
1.  **記憶體拷貝**: `copy()` 指令將你的資料變成 binary 塞入大陣列尾部。
2.  **索引更新**: Map 只存 HashKey 和 Offset，完全不涉及指標。

### C. 讀取流程：Get (Offset Lookup)

```go
func (s *cacheShard) Get(key string) ([]byte, error) {
    s.lock.RLock()
    defer s.lock.RUnlock()

    // 1. 查索引
    hashKey := s.hash.Sum64(key)
    index, ok := s.hashmap[hashKey] // 拿到 Offset (例如 1024)
    if !ok {
        return nil, ErrEntryNotFound
    }

    // 2. 記憶體定位 (Direct Memory Access)
    // 從大陣列的 index 位置開始讀，讀出 Header
    // 這裡會再次比對 Key (因為 Hash 會有碰撞)
    entry, err := s.entries.Get(index) 

    // 3. 還原物件 (Data Copy)
    // 把 blob 裡的 Value 部分拷貝出來
    return readEntryValue(entry), nil
}
```
**底層動作**:
這就像在 C 語言裡操作陣列一樣：`data[index]`。沒有任何指針跳轉 (Pointer Chasing)，CPU Cache Line 命中率極高。

---

## 4. 第四樂章：實戰應用 (Usage)

### 什麼時候該用 BigCache？
1.  **超大數據量**: 緩存項目超過 100 萬筆 (一般 Map 會導致 GC 停頓數百毫秒)。
2.  **高併發寫入**: 需要 Sharding 來減少鎖競爭。
3.  **不需要 LRU**: BigCache 預設是 FIFO (Ring Buffer)。如果你嚴格需要 "最近最少使用 (LRU)"，BigCache 可能不適合 (或者你要接受它的近似淘汰)。
    *   *註：因為維護精確的 LRU 需要移動 Linked List 指標，這就打破了「無指標」和「順序寫入」的優化。所以高效能 Cache 通常妥協用 FIFO 或 Clock 算法。*

### 範例代碼 (秒殺場景)
這就是所謂的 **「防穿透 (Cache Penetration) 鐵布衫」**。
當駭客或大量用戶瘋狂請求一個「已經賣完」的商品時，如果沒有這一層，流量會直接打穿到 Redis 或 DB，導致後端崩潰。
BigCache 在這裡的作用就是 **擋在最前線，無效請求直接在記憶體層被攔截**。

```go
import "github.com/allegro/bigcache/v3"

func InitCache() *bigcache.BigCache {
    config := bigcache.Config{
        Shards:             1024,            // 分片數 (必須是 2 的次方)
        LifeWindow:         10 * time.Minute,// 數據存活時間
        CleanWindow:        5 * time.Minute, // 清理週期
        MaxEntriesInWindow: 1000 * 10 * 60,  // 預估吞吐量 (用來預分配內存)
        MaxEntrySize:       500,             // 預估單個 Value 大小 (Byte)
        HardMaxCacheSize:   8192,            // 限制最大內存 (MB)，例如 8GB
    }
    
    cache, _ := bigcache.New(context.Background(), config)
    return cache
}

// 寫入 (標記座位已售完)
func MarkSold(cache *bigcache.BigCache, seatID string) {
    cache.Set(seatID, []byte("1")) 
}

// 讀取 (檢查是否已售完)
func IsSold(cache *bigcache.BigCache, seatID string) bool {
    _, err := cache.Get(seatID)
    return err == nil // 如果拿得到，代表已售完
}
```

---

## 5. 第五樂章：代價 (The Cost of Zero-GC)

天下沒有白吃的午餐。BigCache 雖然殺死了 GC，但它帶來了兩個新的惡魔：**序列化 (Serialization)** 與 **資料拷貝 (Data Copy)**。

### A. 記憶體流動 (The Data Flow)
當你執行 `Set` 和 `Get` 時，資料在記憶體中經歷了多次搬運：

1.  **Struct -> []byte (序列化)**:
    *   `json.Marshal(user)`。這一步最慢，涉及 Reflect 和大量小記憶體分配。
2.  **[]byte -> RingBuffer (Set Copy)**:
    *   BigCache 執行 `copy(buffer[tail:], data)`。這是純記憶體搬運，速度取決於資料大小。
3.  **RingBuffer -> []byte (Get Copy)**:
    *   BigCache 為了線程安全，**不能**直接回傳 RingBuffer 裡的 Slice (否則你在外面修改會影響 Cache 內部)，必須 `make` 一個新的 slice 並 `copy` 出來。
4.  **[]byte -> Struct (反序列化)**:
    *   `json.Unmarshal`。

### B. 損益平衡點
這是一個簡單的數學題：
> **BigCache Win if:**
> (GC 掃描 1000 萬個指標的時間) > (序列化 + 資料拷貝的時間)

*   **小物件 (Small Object)**: BigCache 完勝。拷貝幾個 byte 幾乎不花時間，但 1000 萬個小物件會搞死 GC。
*   **大物件 (Huge Object, >10KB)**: 原生 Map 可能更好。因為拷貝大塊記憶體很耗頻寬 (Memory Bandwidth)，且通常大物件數量不多，GC 壓力較小。

## 6. 第六樂章：競品對決 (BigCache vs FreeCache vs GroupCache)

| Feature | **BigCache** | **FreeCache** | **GroupCache** | **sync.Map** |
| :--- | :--- | :--- | :--- | :--- |
| **GC Overhead** | **Zero** | **Zero** | Low | High (Large size) |
| **Sharding** | Yes | Yes (256) | Yes | Internal |
| **Eviction** | **FIFO** (Ring Buffer) | **LRU** (近似) | LRU | None |
| **Performance** | ★★★★★ | ★★★★★ | ★★★★ | ★★★ |
| **Keys Limit** | Millions | Millions | Millions | Thousands |
| **Serialization**| **Required** | **Required** | Required | **None** (Stores Pointers) |

> **共同的痛點 (Shared Pain)**:
> 注意到上面 **BigCache** 和 **FreeCache** 的序列化都是 "Required" 嗎？
> 只要是打著 **Zero GC** 招牌的 Library，背後原理都一樣 (Off-Heap or Bytes Array)，所以**一定都需要序列化**。
> 而 `sync.Map` 雖然不用序列化 (可以直接存 `*User`)，但代價就是 GC 掃描會卡頓。
>
> **選型指南**:
> *   如果你在意 **極致吞吐量** 且物件極多 -> 選 BigCache/FreeCache。
> *   如果你的物件 **結構複雜且數量少** -> 選 `sync.Map` (省下序列化 CPU)。

*   **BigCache**: 速度最快，因為它是 FIFO (最簡單)。適合「短期存活、過期就丟」的數據 (如 Session, Rate Limit, Sold Out Tags)。
*   **FreeCache**: 支援 LRU，適合「熱點數據緩存」。它用了一套複雜的 Pointer-less Linked List 來模擬 LRU。
*   **GroupCache**: Google 出品，重點在於「分佈式緩存」和「防擊穿 (SingleFlight)」，如果你需要多機協同，選這個。

---

## 6. 結語 (Conclusion)
BigCache 告訴我們，在追求系統極限時，我們甚至要學會「繞過語言的特性」。
Go 的 GC 很棒，但在千萬級物件面前，它是個負擔。這時，回歸原始的 **Byte Array** 和 **Ring Buffer**，不僅是復古，更是對計算機底層原理的致敬。
