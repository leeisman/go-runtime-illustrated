# Book 3.12: The Garbage Collector (垃圾回收)

## 1. 第一樂章：隱喻 (The Metaphor - The Invisible Cleaner)

回到我們的 **迴轉壽司店**。
食客 (Chefs/G) 拼命做壽司、吃壽司。盤子 (Object) 越堆越高。
如果沒人收盤子，這家店 (Heap Memory) 很快就會塞滿，最後爆炸 (OOM)。

於是，Go 聘請了一位 **「隱形清潔工 (GC)」**。
但他有一個幾乎不可能的任務：**他必須在大家「還在吃」的時候，一邊收盤子。**

*   **傳統做法 (STW - Stop The World)**:
    *   清潔工大吼一聲：「全部不准動！」(Stop)。
    *   大家停格。清潔工慢慢收完。大家繼續吃。
    *   **缺點**: 服務停擺，客戶不爽 (Latency 高)。

*   **Go 的做法 (Concurrent Mark & Sweep)**:
    *   清潔工戴著耳機，穿梭在人群中默默貼標籤 (Mark)。
    *   大家繼續吃吃喝喝 (Concurrent)。
    *   **挑戰**: 如果清潔工剛確認某個盤子是「垃圾」，下一秒食客突然又要用那個盤子裝醬油怎麼辦？

---

## 2. 第二樂章：三色標記的實作細節 (The Implementation)

在理解流程之前，我們先來看看 Runtime 到底用什麼資料結構來實作這套系統。

### 核心結構 (The Structures)
Go 使用 **Bitmap (位元圖)** 配合 **Work Queue (工作佇列)** 來分辨物件狀態。但在那之前，我們先回顧一下 Heap 的單位 (Ref: Book 3.5)：

1.  **Heap Arena**: 這是 Heap 記憶體的物理大區塊 (64MB)。
2.  **Object (物件)**: 存活在 Heap 上的資料 (如 `User` struct)。
3.  **gcmarkBits (位元圖)**: 每個 Arena 都有對應的 Bitmap，紀錄「誰活著」。
    *   **對應關係 (Mapping)**: 採用 **1:64** 的比例壓縮 (1 bit 管 8 bytes)。
        *   **Heap**: `[ 8 Bytes ] [ 8 Bytes ] [ 8 Bytes ] ...`
        *   **Map** : `[  Bit 0  ] [  Bit 1  ] [  Bit 2  ] ...`
    *   **例子**: 如果物件在 Offset 16 (第 2 格)，我們就去標記 Bit 2。
    *   **0**: 未標記 (White)。
    *   **1**: 已被發現 (Marked)。
4.  **allocBits (位元圖)**: 紀錄「記憶體佔用狀態」 (給 Allocator 看的空房表)。
    *   **Swap 機制 (雙緩衝)**: 這是理解 GC 的關鍵。
        1.  **平時**: Allocator 看著 `allocBits` 查房。
        2.  **GC 開始**: Runtime 申請一張 **全新的白紙** 給 `gcmarkBits` 使用。
        3.  **GC 結束**: 把畫好的 `gcmarkBits` 扶正變成下一輪的 `allocBits`。
        4.  **結果**: 舊的 `allocBits` 被丟棄，我們總是拿「最新的點名結果」當作「明天的空房表」。
    *   **數值變化實例 (Dead Object)**:
        1.  **GC 前**: `alloc=1` (被佔用)。
        2.  **GC 中**: `mark` 初始化為 0。因為它死了 (沒人指它)，所以 `mark` 保持 **0**。
        3.  **GC 後 (Swap)**: `alloc` 變成了 `mark` 的值。所以 `alloc` 變成了 **0**。
        4.  **結果**: 該位置被釋放，Allocator 視為空房。
5.  **Work Queue (工作佇列)**: 存放待掃描的物件指針。

### 顏色的物理意義 (Physical Meaning)
所謂的「三色」，在底層其實是這樣運作的：
*   **白色 (White)**: **Bit=0**。 (我都還沒看過它)
*   **灰色 (Grey)**: **Bit=1** 且 **指針在 Work Queue 裡**。 (我看到它了，但還沒掃描它的孩子)
*   **黑色 (Black)**: **Bit=1** 且 **指針已離開 Work Queue**。 (我徹底檢查完它了)

---

### 完整的 GC 週期 (The Cycle)

現在，我們用這些「物理動作」來走一遍 GC 流程：

1.  **Mark Start (STW)**:
    *   **做什麼**: 暫停所有 P，開啟全域 **Write Barrier**，準備 Work Queues。
    *   **耗時**: 10~30 µs (O(1))。
    
2.  **Roots Scanning (Concurrent)**:
    *   **Global/Stacks**: GC 遍歷所有 Goroutine，暫停它們並掃描 Stack。
    *   **動作 (變成灰色)**: 針對 Stack 上的指針指向的 Heap 物件：
        *   **地址換算 (The Math)**: Runtime 算出 `Index = (Ptr - ArenaBase) / 8`。
        *   **標記**: 找到 Bitmap 中第 `Index` 個 Bit，將其 **設為 1**。
        *   **入列**: 將其指針 **丟進 Work Queue**。
    
3.  **Concurrent Marking (BFS)**:
    *   現在 Work Queue 裡有一堆第一層物件。GC Worker 開始消耗佇列：
        *   **Pop (變黑)**: 拿出一個物件 X (它自然就變成了黑色，因為離開了佇列)。
        *   **Push (變灰)**: 掃描 X 內部的指針 (例如 pointing to Y)。如果 Y 的 Bit 是 0：
            *   將 Y 的 **Bit 設為 1**。
            *   將 Y 的指針 **丟進 Work Queue**。
    *   這就是「三色標記」的真面目：不斷的 **Set Bit & Enqueue**，直到 Queue 空掉。
    
4.  **Mark Termination (STW)**:
    *   **為什麼要暫停？**: 在 Step 3 期間，Write Barrier 不斷產生新的灰色物件 (Set Bit=1 & Enqueue)。必須暫停一下，把最後這一小批積壓的佇列掃描乾淨。
    
5.  **Sweep (Concurrent & Lazy)**:
    *   **懶惰清掃**: 現在 Bitmap 裡，Bit=1 的是活的，Bit=0 的是垃圾。
    *   當你需要分配新記憶體時，Runtime 順便檢查一段 Bitmap，把 Bit=0 的區塊撿回來用。
    *   **重置 (Reset)**: 其實是 **替換 (Swap)**。
        *   上一輪標記完的 Bitmap (顯示誰活著)，直接變成下一輪的 **`allocBits`** (顯示誰佔用記憶體)，這樣 Allocator 就知道哪些位置不能用。
        *   而 Runtime 會為下一輪準備一個 **全新的全 0 Bitmap** 作為新的 `gcmarkBits` (白紙)。



---

## 3. 第三樂章：寫屏障 (Write Barrier - The Hybrid)

**誤區澄清**: Write Barrier **不是**一個 Queue，它是編譯器安插在指標寫入操作時的一段 **Hook (攔截程式碼)**。

*   **平時**: `A.ptr = B` 真的就只是賦值。
*   **GC 期間**: 執行 `A.ptr = B` 時，會觸發 Barrier 程式碼：
    1.  **攔截**: "等一下！你在改指針！"
    2.  **塗色**: 把相關的物件 (例如 B 或者原本 A 指向的 C) 標記為 **灰色**。
    3.  **入隊**: 把這些灰色物件 **丟進 Work Queue**。

這就是為什麼在 GC 期間，指針寫入會稍微變慢的原因。

最難的問題來了：**清潔工在掃地時，你在亂丟東西。**

假設清潔工已經把 **A (黑)** 掃描完了，認為它很安全且下面沒東西。
這時，你突然把一個 **白色物件 C** 塞給了 **A** (`A.ptr = C`)，並且把原本指向 C 的引用拿掉。
*   清潔工認為 A 是黑的 (已完成)，不會再回頭檢查 A。
*   結果：清潔工沒看到 C，C 還是白的。
*   **慘劇**: C 被當成垃圾收走，程式崩潰 (Dangling Pointer)。

### 解決方案：混合寫屏障 (Hybrid Write Barrier)
Go Runtime 強制規定：**在 GC 期間，任何人如果想修改指針，必須觸發警鈴 (Barrier)。**

*   **規則**: 無論你怎麼改指針，只要你碰了一個白色物件，**不管三七二十一，先把它塗成灰色！**
*   **意義**: 寧可殺錯 (把垃圾留到下一輪)，不可放過 (誤刪有用代碼)。這保證了 **強三色不變性 (Strong Tri-color Invariant)** 的變體。

這就是為什麼在 3.11 我們說 Channel 的 Direct Copy 也要過 Barrier 的原因——因為它涉及了指針的移動。

---

## 4. 第四樂章：GC Pacer (節奏控制)

清潔工什麼時候出來掃地？
如果每秒都出來，那大家都別吃飯了 (CPU 全在 GC)。
如果十年才出來，店早就被垃圾淹沒了。

Go 有一個 **Pacer (配速員)**，它根據一個公式決定：
*   **GOGC=100 (預設)**: 當 Heap 成長到 **上次 GC 後的 2 倍** 時，觸發下一次 GC。
*   **Live Heap**: 假設上次掃完剩 10MB，那等用到 20MB 時就觸發。
*   **Mark Assist (協助標記)**: 這是防止 GC 永遠跑不完的保險機制。如果在 GC 期間你分配太快 (產生垃圾 > 清理速度)，Runtime 會強制你的 Goroutine 停下來「幫忙標記」，變相減慢你的速度，確保 GC 能追上。

這解釋了 **Book 3.5 Allocator** 的教訓：
如果你瘋狂製造「短命的小物件」，Heap 增長會極快 -> GC 頻繁觸發 -> CPU 都在做標記 -> 服務變慢。

---

## 5. 第五樂章：效能調優 (Performance Tuning)

GC 的效能不只取決於 Heap 多大，更取決於 **「有多少指針需要掃描」**。

### 1. 指針密度 (Pointer Density - The Killer)
這常是被忽略的瓶頸。
*   **慢**: `map[string]*User` 或 `[]*User`。這包含數百萬個指針。GC 必須一個個追蹤 (Deference)。
*   **快**: `map[string]User` (若 User 內無指針) 或 `[]byte`。
    *   對於不含指針的記憶體區塊，GC 會將其標記為 **NoScan**，掃描時直接整塊跳過，速度極快。
*   **建議**: 盡量減少 Heap 上的指針數量。例如用 `ID (int)` 代替 `Pointer` 關聯，或用大的 `[]struct`。

### 2. GOMEMLIMIT (Go 1.19+)
這對 Kubernetes 環境至關重要。
*   **問題**: 以前我們只設 `GOGC`。如果設太大，容易 OOM (被 K8s 殺掉)；設太小，GC 又太頻繁浪費 CPU。
*   **解法**: 設定 `GOMEMLIMIT` (例如 Pod Limit 的 90%)。
    *   Runtime 會盡量少做 GC。
    *   但當記憶體用量逼近 Limit 時，它會自動變得「勤勞」起來，防止 OOM。這比手動調 GOGC 更智慧。

## 6. 極簡記憶法：GC 運行四部曲 (Executive Summary)

拋開教科書上那些死板的「白灰黑三色標記」術語，用架構師的語言，我們完全可以用這四個具象的步驟來精準還原流程：

1. **第一聲鳴槍 (微秒級 STW)**：全球暫停僅約 10 微秒，只為做一件事：「打開 Write Barrier (寫屏障) 全域警報器！」宣佈進入回收期，然後立刻讓大家繼續跑。**這確保了接下來的掃描期間，全場所有的 Goroutine 只要發生了「指標改寫/位移」，都必須無條件被攔截，並將該有風險的指標強制丟進 `Work Queue` 排隊檢查。**
2. **局部麻醉抽線索 (Concurrent Scan)**：清潔工 (GC Worker) 遊走在成千上萬個 Goroutine 之間。暫停 G1，抽提它的 Stack，把指向 Heap 的指標丟進 `Work Queue` (工作佇列) 後，立刻喚醒 G1。接著去麻醉 G2，這樣化整為零的搜查，完全不干擾整體伺服器的高速運作。
3. **順藤摸瓜與警報防呆 (Tracing & Hook)**：
    * 追蹤：清潔工拿出 `Work Queue` 裡的原始指針，沿著 Heap 往下追查，確保碰到的活躍物件都在新畫布 (`gcmarkBits`) 上塗為 1。
    * 防呆：掃描的同時，世界照常運轉。如果有 Goroutine 執行改寫指標，**寫屏障 (Write Barrier)** 警報就會大響，強制把這個有可能發生「漏看風險」的指標丟進 Queue 裡排隊覆查。這做到了寧可緩刑錯殺，也絕不誤刪。
4. **第二聲鳴槍與偷天換日 (STW 2 & Swap)**：為了避免有人發瘋似地改指標導致 Queue 永遠清不完，系統發動第二次短短微秒的 STW。暫停全場，把殘餘的任務清空，接著大喊「關閉警報器」，然後直接拔掉舊空房表，把剛剛順藤摸瓜畫好的 `gcmarkBits` 存活名單直接改名成明天的 `allocBits`。零成本瞬間物理消滅所有死去的垃圾！

---

## 7. 終章 (Finale)

Go GC 的哲學是 **「低延遲 (Low Latency)」** 而非高吞吐。
它願意犧牲一點程式執行的 CPU (Write Barrier 攔截開銷)，換取 **幾乎感受不到的微秒級 STW**。

*   **Stack**: 免費 (自己彈出銷毀，完全不歸 GC 管)。
*   **Heap**: 昂貴 (需要被加入追蹤隊列 + 承受寫屏障警報開銷)。
*   **優化**: 盡量讓變數留在 Stack (徹底理解第二章的逃逸分析)，Heap 上活著的指針越少，這套麻醉與順藤摸瓜的流程就跑得越快！

---
**Next: [Book 3.13 介面 (Interface)](./3_13_GO_INTERFACE.md)** (預計建立)
