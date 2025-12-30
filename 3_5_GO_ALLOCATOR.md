# Book 3.5: 記憶體的分配 (The Allocator)

在 **Book 2.2** 中，我們將 Heap 比喻為 **「大黑板」**。那時候我們說：「只要是 Heap 分配，成本就很高，因為要找空間還要搶鎖。」

但這個說法只對了一半。
如果每次申請記憶體 (例如只為了存一個 `int`) 都要去跟 OS 要，那不僅要發起 **System Call (切換到 Kernel Mode)**，還要跟系統中其他所有 Process (如 Chrome, Docker) 搶奪 **OS 級別的鎖**，效能會非常難看。

**關鍵策略：批發 (Wholesale)**。
Go Runtime 每次向 OS 伸手，都是 **64MB 起跳** 的大開口 (Arena)。
它付一次跟 OS 搶鎖的代價，就能在內部慢慢分配給數百萬個小物件。為了高效管理這筆鉅款，Go 偷學了 **TCMalloc (Thread-Caching Malloc)** 的精髓，把記憶體分配變成了 **三級緩存** 的精妙遊戲。

---

## 1. 第一樂章：P 的私房錢 (mcache) - No Lock

這是 GMP 模型對記憶體分配做出的最大貢獻。

### 1. 隱喻：經理的抽屜
*   **P (Processor)**: 部門經理。
*   **mcache**: 經理辦公桌下的一個 **「私房錢抽屜」**。
*   這個抽屜裡預先放好了一疊一疊的 **小額現金** (Small Objects Pre-allocated Memory)。

### 2. 運作流程 (Lock-Free)
當 G1 (員工) 跑來跟 P (經理) 說：「我要申請一個 **32 bytes** 的空間存 `User` 結構。」
1.  **P 的反應**: P 不會打電話給總務處 (Global Heap)，也不用填申請單。
2.  **直接拿**: P 直接打開自己的抽屜 (`mcache`)，拿出一張 32 塊的鈔票給 G1。
3.  **速度**: 整個過程 **不需要加鎖**。因為這個抽屜只有這一位經理 (P) 能開，別的 P 碰不到，所以絕對不會有衝突。

**結論**: 只要是 **微小物件 (Tiny)** 或 **小物件 (Small)** 的分配，Go 都是走這條 **極速通道**。這就是為什麼我們說「在 Go 裡，`new(int)` 幾乎跟 Stack 分配一樣快」。

## 1.5 插曲：mcache 與 Stack 的糾葛 (The Stack Confusion)

這時你可能會問：「mcache 不需要鎖，又是在 P 身上，那它跟 Stack 有什麼不同？為什麼它還需要 GC？」

### 1. 相似之處 (Why it feels like Stack)
*   **私有**: 都是 Thread/P 私有的，別人搶不到。
*   **極速**: 都是調整一下指標就分配好了。

### 2. 致命差異 (Why it needs GC)
*   **Stack**:
    *   **生命週期**: 嚴格綁定 **函數呼叫**。函數結束，Stack Frame 必定銷毀。
    *   **回收**: **自動彈出 (Pop)**，無需任何人幫忙。
*   **mcache (Heap)**:
    *   **生命週期**: **不確定**。物件可能被傳給別的 G，或被全域變數引用。函數即使結束了，這個物件可能還必須活著。
    *   **回收**: 因為不知道何時會死，必須依賴 **GC** 全盤掃描確認沒人用了，才能回收。

### 3. mcache 多大？
*   **容量**: 很小，通常只有幾十 KB。
*   **限制**: 它只負責 **< 32KB** 的小物件。更大的物件會直接繞過 mcache 去找 mheap。
*   **意義**: 這涵蓋了絕大多數 (99%) 的業務邏輯物件 (structs)。

---

## 2. 第二樂章：部門金庫 (mcentral) - Fine-grained Lock

如果經理的抽屜空了怎麼辦？或者抽屜裡的錢面額不對 (只有 100 塊的，但你需要 64 塊的)？

### 1. 隱喻：樓層提款機
*   **mcentral**: 每一層樓 (Size Class) 共用的一台 **「提款機」**。
*   這台機器裡存著大量的鈔票，但它是 **所有部門經理 (All Ps) 共享的**。

### 2. 運作流程 (Need Lock)
1.  **P 的困境**: P 發現抽屜裡的 32 塊鈔票發完了。
2.  **補貨**: P 拿著補貨單，走到走廊上的 `mcentral` 提款機。
3.  **搶鎖 (為什麼要鎖？)**: 
    *   因為 `mcentral` 是共享資源，可能同時有 10 個 P 都來補貨。
    *   **風險**: 若不加鎖，P1 和 P2 可能會搶到 **同一疊鈔票 (Span)**。
    *   **後果**: 兩個不同的變數會被分配到同一個記憶體地址，導致數據互相覆蓋 (Data Corruption)，這是系統崩潰的絕對禁忌。所以這裡必需排隊。
4.  **批次領取**: P 不會只領一張，他會一口氣領 **一整疊 (Span)** 回去，把自己的抽屜填滿。這樣下次就不用再來排隊了。

### 3. 本質其實是：預先切好的蛋糕 (Pre-cut)
你可能會問：「Runtime 是現場切蛋糕嗎？」
**並不是。**
*   **預先切分**: mcentral 還沒被借出去之前，就已經把一大塊記憶體 (Span) 依照規格 (例如 8 bytes) **切成無數小塊了**。
*   **領取**: P 來補貨時，只是把這「一整盤已經切好的蛋糕」端走而已。
*   **速度**: 這就是為什麼分配這麼快。它不需要計算大小、尋找空位，它只是 **「從 linked list 拔下一個節點」** 而已。

---

## 3. 第三樂章：總行金庫 (mheap) - Global Lock

如果連樓層提款機都沒錢了？或者 G1 獅子大開口要申請 **1GB** 的超大記憶體？

### 1. 隱喻：中央銀行
*   **mheap**: 整棟大樓的 **「地下金庫」**。
*   這裡掌管著所有的實體紙張 (OS Memory Pages)。

### 2. 運作流程 (Global Lock)
1.  **情況**: 需求太大 (> 32KB) 或者 System 全部缺錢。
2.  **上報**: 請求一路向上，最後來到 `mheap`。
3.  **全域鎖 (與 OS 的關係)**: 
    *   **批發商 (Go Runtime)**: `mheap` 已經向 OS 批發了大量記憶體 (Arenas)。它需要 **User-Space Lock** 來管理這些庫存 (切分或合併 Page)。
    *   **地主 (OS Kernel)**: 只有當 `mheap` 庫存空了，才會真的發起 Syscall (`mmap`) 向 OS 要新的地皮。這時候雖然是 OS 在處理，但 `mheap` 依然要鎖住自己，等待擴充完成。

### 3. 什麼代碼會誤觸陷阱？(Triggers)
什麼時候你的程式會繞過 mcache/mcentral，直接撞上這道最慢的牆？
1.  **超大物件分配 (> 32KB)**:
    *   **魔法數字**: **32KB**。這是 mcache 能處理的上限。
    *   **安全區 (<= 32KB)**: 例如 WebSocket Buffer (4KB, 8KB)。這些都能命中 mcache，享受 Lock-Free 的極速分配。
    *   **危險區 (> 32KB)**: 例如 `make([]byte, 1MB)`。Go Runtime 沒辦法從抽屜拿這麼大的東西，只能被迫去搶 `mheap` 的 Global Lock。
    *   **實戰建議**: 儘量讓單次分配小於 32KB。如果必須很大，請用 `sync.Pool`。
2.  **記憶體總量不足 (System Growth)**:
    *   當你的程式把 mheap 裡的庫存都用光了，mheap 就必須向 OS 申請新的虛擬記憶體 (`mmap` syscall)。這也是最慢的操作。

### Q: 如果我有一堆很小的物件 (例如 10 萬個 8 bytes)，把 mcache 和 mcentral 都買空了呢？
Runtime 會啟動 **「補貨鏈 (Refill Chain)」**，而不是把你當大物件處理：
1.  **mcache 缺貨**: P 向 mcentral 要一整疊 (Span)。
2.  **mcentral 缺貨**: mcentral 向 mheap 要一塊原始記憶體 (Pages)，把它 **切碎** 成 8 bytes 的格子，補充庫存。
3.  **mheap 缺貨**: mheap 向 OS 申請新的虛擬記憶體 (`mmap`)。
**結論**: 只要單個物件小，你就永遠走 **SOP (快速與批量)** 流程。總量大只會導致 mheap 更頻繁地去跟 OS 要地皮來蓋新的 mcentral 倉庫。

---

## 4. 第四樂章：總結圖解 (The Grand Flow)

讓我們把鏡頭拉遠，看一次從 **OS 到底層員工** 的完整資金流動：

### 1. 宏觀視角：資金的源頭 (OS -> Runtime)
*   **Step A (批發)**: **OS (大地主)** 劃撥一塊巨大的虛擬記憶體 (e.g., 64MB Arena) 給 **mheap (總行)**。
*   **Step B (加工)**: `mheap` 負責管理這塊地，把它切分成數千個 8KB 的 **Pages**。
*   **Step C (分發)**: `mcentral` **(分行)** 根據需求，向總行領走一些 Pages，並將其切割成特定規格的 **Spans** (例如專存 32byte 物件的格子)。

### 2. 微觀視角：兩位員工的申請 (Two Gs)
現在，**G1** (在 P1 上) 和 **G2** (在 P2 上) 同時都要申請 `new(User)`：

*   **情境 1 (理想路徑 - Retail)**:
    *   G1 伸手進 P1 的 **mcache (私房錢)**，直接拿走一個格子。
    *   G2 伸手進 P2 的 **mcache (私房錢)**，也拿走一個格子。
    *   **結果**: **完全平行，互不干擾 (Lock-Free)**。這是 99% 的情況。

*   **情境 2 (競爭路徑 - Wholesale)**:
    *   P1 發現私房錢用光了，跑去 **mcentral** 補貨。
    *   P2 也剛好用光了，也跑去同一個 **mcentral**。
    *   **結果**: **發生競爭**。`mcentral` 上鎖，P1 先補貨，P2 排隊等待。這就是為什麼共享區域需要鎖，也是我們要極力避免的路徑。

**最終結論**:
## 5. 第五樂章：驗屍報告 (Profiling)

即使有 mcache，Heap 分配始終比 Stack 貴。除了 GC 壓力外，**記憶體翻攪 (Churn)** 也是隱藏的效能殺手。我們可以用 `pprof` 來抓出真兇。

### 1. 抓 GC 浪費 (CPU Profile)
如果你在 CPU Profile (`cpu.prof`) 中看到以下函數佔用很高，代表你的 Heap 分配太多了，CPU 都在忙著打掃：
*   **`runtime.gcBgMarkWorker`**: 代表 GC 背景執行緒在大量耗能。
*   **`runtime.scanobject`**: 用來掃描物件參考的，也代表物件圖太複雜。
*   **真實案例 (The JSON Monster)**:
    *   **情境**: 一個高併發的 API Gateway，每個請求都會呼叫 `json.Unmarshal` 解析大量資料。
    *   **後果**: 標準庫的 JSON 使用反射 (Reflection)，會產生海量的暫時性小物件 (小垃圾)。導致 CPU 有 30%~50% 的時間都在跑 `gcBgMarkWorker` 撿垃圾，而不是處理業務。
    *   **特徵**: QPS 上不去，加機器也沒用 (因為每台機器的 CPU 都被 GC 吃光了)。
    *   **💡 解法**:
        1.  **換庫**: 改用 **sonic** (by ByteDance) 或 **easyjson**，避開執行期反射開銷。
        2.  **複用**: 使用 **`sync.Pool`** 複用 Struct 物件，減少每次 `New` 的負擔。
        3.  **換協議**: (終極解法) 改用 **Protobuf** (搭配 gRPC 或 HTTP)。
            *   **註**: 即使是標準 HTTP API，也可以將 `Content-Type` 設為 `application/x-protobuf` 來傳輸二進位資料。這比 JSON 快 5-10 倍，且幾乎沒有 GC 壓力。
            *   **開發難度?**: 對 Mobile/Backend 其實 **更輕鬆** (自動生成強型別 Struct)。唯獨 Web/Postman 無法直接看，但 Postman 最新版已支援匯入 `.proto`。
            *   **最佳實踐**: 讓 Server 支援 **雙模 (Content Negotiation)**。開發時用 `Accept: application/json`方便除錯，上線時 App 改傳 `Accept: application/x-protobuf` 榨乾效能。兩全其美。

### 2. 抓鎖競爭 (Contention)
如果你頻繁申請 **中大型物件 (>32KB)** 或 **特定 Size Class** 耗盡，導致 mcache 穿透，你會看到：
*   **`runtime.lock`**: 通用鎖競爭。大多發生在 mheap 分配大物件時。
    *   *(註: 這是 Runtime 內部的底層鎖。正常情況下佔比應極低 (<1%)。若它飆高，通常是三個原因： 1. **Allocator** (大物件分配) 2. **Scheduler** (Goroutine 頻繁建立/銷毀/調度) 3. **Stack Growth** (深度遞迴導致 Stack 頻繁擴容))*
    *   **常見誤區**: 很多人看到 `runtime.lock` 直覺就是記憶體問題，這是錯的！**Channel** (`chan.lock`)、**Timer** (`timers.lock`) 和 **Netpoller** 也都共用這個底層鎖機制。
*   **`runtime.(*mcentral).cacheSpan`**: 代表 P 頻繁被迫去 mcentral 補貨。
    *   **如何抓兇手**: 不要只看 Runtime 函數，這沒有意義。在 pprof 圖表中，順著這些 Runtime 函數 **往上找 Caller (呼叫者)**，看是 `mallocgc` 呼叫的 (記憶體問題)，還是 `chansend` 呼叫的 (Channel 競爭)，這才是真相。

### 3. 抓「敗家子」 (Heap Profile - Alloc)
這才是最關鍵的。請使用 `alloc_objects` (累積分配次數) 而不是預設的 `inuse_objects`。
*   **指令**: `go tool pprof -alloc_objects http://.../heap`
*   **現象**: 如果某個函數的 `Alloc` 數值極大 (例如累積 1000 萬次)，但 `Inuse` 很小 (只有 10 個)。
*   **數據對比 (The Truth)**:
    *   **預設視角 (`-inuse_objects`)**: 騙人的假象。看起來一切正常，因為垃圾剛好被 GC 撿走了。
        ```bash
        Flat   Cum
        10     10     main.BadPattern  (看起來很無辜)
        ```
    *   **真相視角 (`-alloc_objects`)**: 殘酷的現實。原來它已經偷偷製造了一億個垃圾！
        ```bash
        Flat   Cum
        100M   100M   main.BadPattern  (兇手就是你！)
        ```
*   **診斷**: 這就是典型的 **Memory Churn (記憶體翻攪)**。
    *   **意思**: 你的程式在瘋狂申請記憶體然後馬上丟掉 (例如在迴圈裡 `make([]byte)`)。
    *   **後果**: 雖然沒有 Memory Leak (Inuse 很低)，但這會導致 **GC 頻繁啟動 (Stop The World)**，直接導致 **Latency 飆高**。
    *   **代碼範例 (兇手長這樣)**:
        ```go
        func BadPattern() {
            for {
                // 1. 迴圈內分配 (Loop Alloc)
                tmp := make([]byte, 1024*1024) 
                
                // 2. 字串拼接 (String Concat)
                s := ""
                for i:=0; i<100; i++ { s += "data" } 
                
                // 3. 介面裝箱 (Interface Boxing)
                fmt.Sprintf("ID: %d", i) 

                // 4. 型別轉換 (Type Conversion)
                b := []byte(s)
                _ = string(b)

                DoSomething(tmp, s)
            }
        }
        ```
    *   **詳解：四大效能地雷**:
        1.  **迴圈內分配 (Loop Alloc)**
            *   **問題**: 每次迴圈都向 Heap 申請新記憶體，用完即丟。
            *   **解法**: 將 `make` 移到迴圈外，並使用 `tmp = tmp[:0]` 重置。
            *   *(註: 即使是小物件，高頻率也會導致 GC 掃描成本飆升)*
        2.  **字串拼接 (String Concat)**
            *   **問題**: Go String 是 **不可變 (Immutable)** 的。每次 `+=` 都會引發 Alloc + Copy，複雜度是 O(N^2)。
            *   **解法**: 改用 `strings.Builder`。
        3.  **介面裝箱 (Interface Boxing)**
            *   **問題**: `fmt.Sprintf` 接受 `interface{}`。傳入 `int` 時，Runtime 必須分配一個小物件來「裝」這個 `int` (逃逸分析)。
            *   **解法**: 使用強型別的 Logger (如 Zap 的 `Int` 欄位)。
        4.  **型別轉換 (Type Conversion)**
            *   **問題**: `string` 轉 `[]byte` 是 **Deep Copy** (為了安全)。這涉及記憶體分配。
            *   **解法**: 架構上盡量減少轉型。若為 Read-Only 且效能關鍵，可考慮 `unsafe` Zero-Copy 技術 (Go 1.20+)。
                ```go
                // String -> []byte (Zero-Copy)
                func StrToBytes(s string) []byte {
                    return unsafe.Slice(unsafe.StringData(s), len(s))
                }
                // []byte -> string (Zero-Copy)
                func BytesToStr(b []byte) string {
                    return unsafe.String(unsafe.SliceData(b), len(b))
                }
                ```

### 結論：當 API 變慢時，如何判斷是記憶體問題？

我們應該採取 **「觀察 -> 診斷 -> 處方」** 的策略：

1.  **策略**: 當 **Latency 飆高** 時，先檢查 **CPU 使用率**。
2.  **現象 A (CPU 也飆高)**:
    *   **檢查**: 打開 CPU Profile，發現 `runtime.gcBgMarkWorker` (撿垃圾) 或 `runtime.mallocgc` (製造垃圾) 佔用前三名。
        ```bash
        (pprof) top  // 這就是標準的 top 指令輸出格式
        Flat    Flat%   Sum%    Cum     Cum%
        30.00s  30.00%  30.00%  30.00s  30.00%  runtime.gcBgMarkWorker (忙著掃垃圾)
        10.00s  10.00%  40.00%  15.00s  15.00%  runtime.mallocgc       (忙著切蛋糕)
        ```
        *(註: **Flat** = 函數自己做事的時間；**Cum** = 函數自己 + 叫小弟做事的時間)*
        *(記憶法: **Flat 是苦力** (真正耗油的引擎)，**Cum 是老闆** (包含整個部門的產出))*
        *(注意: `top` 指令只會列出排行榜，不會顯示 Call Stack。想知道是誰呼叫了它，請輸入 `list runtime.mallocgc` 或使用 Web UI 的 Graph 視圖)*
    *   **診斷**: **Memory Churn (敗家子)**。你的程式正在瘋狂製造垃圾，GC 忙不過來。
    *   **處方**: 用 `go tool pprof -alloc_objects` 抓出 `Alloc` 最高的函數，改用 `sync.Pool` 或 `strings.Builder` 優化。
3.  **現象 B (鎖競爭)**:
    *   **特徵**: CPU 表現不一 (搶鎖自旋時高，等待 Syscall 時低)。
    *   **檢查**: 不要只看 CPU Load，請務必打開 **Block/Mutex Profile**，若發現 `runtime.lock` 或 `mcentral.cacheSpan` 佔比高，就是它了。
    *   **診斷**: **Allocator Contention (搶分行)**。大量的中/大物件導致 P 被迫排隊補貨。(關於鎖的詳細原理與排查，請參閱 **Book 3.6 鎖與競爭**)
    *   **處方**: 盡量讓物件小於 32KB (走 mcache)，或使用 Object Pool 減少申請頻率。

---
**Next: [Book 3.6 鎖與競爭 (Lock & Contention)](./3_6_GO_LOCK.md)**
