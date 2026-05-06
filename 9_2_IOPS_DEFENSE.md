# Book 9.2: 資料庫 IOPS 防禦戰術 (Database IOPS Protection Strategies)

在高併發系統中，關聯式資料庫（MySQL）的 CPU 與連線數可以透過水平擴展或讀寫分離來緩解，但 **磁碟的 IOPS（每秒輸入/輸出次數）** 往往是系統中最難以跨越的物理硬傷。

當系統面臨極高頻的讀寫請求時，保護底層資料庫的 IOPS 不被耗盡，是架構設計的最高指導原則。本章將介紹三大核心防禦戰術，以及架構的最終手段。

---

## 1. 第一樂章：讀取防禦 - Redis 緩存與防護網 (Read IOPS Protection)

資料庫的 I/O 資源極其昂貴，任何可以預測且非強即時變動的資料，都不應該直接對資料庫發起查詢。

### 1. Cache-Aside Pattern (旁路快取模式)
*   **機制**：讀取時先查 Redis，若命中 (Cache Hit) 則直接回傳；若未命中 (Cache Miss)，才去查 MySQL，查完後寫回 Redis 並設定過期時間。更新時，**先更新資料庫，再刪除快取**。
*   **防護原理一：為什麼是先更新 DB，而不是先刪快取？**
    *   假設「先刪快取，再更新 DB」：Thread A 刪除了快取但還沒寫入 DB。此時 Thread B 來讀取，發現快取空了，去 DB 查到了「舊資料」並寫回快取。接著 A 更新了 DB。結果：**DB 是新的，快取是舊的，且在 TTL 過期前永遠不一致。**
*   **防護原理二：為什麼是刪除快取，而不是更新快取？**
    *   基於 Lazy Loading (懶加載) 思想。快取的資料有時是經過複雜計算來的，如果每次改 DB 都去重算並更新快取，會浪費大量 CPU 資源。不如直接刪掉，等真正有人來讀的時候再去 DB 查。
*   **優勢**：將 90% 以上的讀取 I/O 攔截在記憶體層，徹底解放 MySQL 的 Buffer Pool 與讀取 IOPS。

### 2. 快取三大災難與防禦 (Senior 必備)
光有 Redis 不夠，必須防範快取層被擊穿，導致流量瞬間灌入資料庫：

*   **快取穿透 (Cache Penetration)**：駭客狂刷不存在的 ID。
    *   **解法**：在快取層快取空值 (Null Value) 並設定極短 TTL，或在最外層引入 **布隆過濾器 (Bloom Filter)**。
*   **快取擊穿 (Cache Breakdown - 單點突破)**：**單一個**超級熱門的 Key（例如：周杰倫演唱會門票資訊）快取突然過期，瞬間數萬筆併發只針對「這一個 Key」打向 DB。
    *   **解法**：在 Go 中使用 `golang.org/x/sync/singleflight` 機制，確保同一個 Key 瞬間只有「一個 Goroutine」去查 DB，其餘 Goroutine 在記憶體中等待結果共享。
*   **快取雪崩 (Cache Avalanche - 全面崩盤)**：**大量不同**的 Key 在同一秒過期（通常是因為批次載入時設定了相同的 TTL，或是 Redis 叢集當機），導致大面積的 Cache Miss，海量流量同時湧入 DB。
    *   **解法**：在設定 TTL 時加上一段「隨機亂數 (Jitter)」，打散各個 Key 的過期時間；若是 Redis 當機引起的雪崩，則需要依賴高可用架構與限流熔斷機制。

---

## 2. 第二樂章：寫入防禦 - 延遲寫入 (Write-behind)

對於非核心交易數據（例如：`last_login` 登入時間更新、影片觀看次數、日誌收集），直接 `UPDATE` 會引發大量的隨機寫入 (Random I/O) 與行鎖競爭。

### 1. 非同步緩衝區 (Asynchronous Buffer)
*   **機制**：捨棄強一致性，將請求轉向極速的記憶體。在 Go 程式中，將更新事件送入 `channel`，並透過一個背景的 Goroutine (Worker) 配合 `time.Ticker`，定時且定量地拉取資料。
*   **優勢**：資料庫完全感受不到瞬間的寫入洪峰，I/O 曲線變得極度平滑。

### 2. 容災與優雅關機 (Graceful Shutdown)
*   **風險**：留在記憶體 (`channel`) 裡的資料，一旦 Pod 發生 OOM 或重啟，資料就會人間蒸發。
*   **防禦機制**：監聽作業系統的 `SIGTERM` 訊號。當 Kubernetes 準備刪除 Pod 時，攔截訊號，停止接收新請求，並強制觸發一次 Flush，將 `channel` 內剩餘的資料全部安全寫入 MySQL 後，程式才正式退出。

---

## 3. 第三樂章：寫入優化 - 寫合併與批次處理 (Write-Combining & Batching)

將資料攔截在記憶體後，如何「最有效率」地倒進資料庫，是壓榨 IOPS 效能的最後一哩路。

### 1. 寫合併 (Write-Combining / Debouncing)
*   **場景**：同一個使用者在 10 秒內快速切換了 5 個頁面，觸發了 5 次 `last_login` 更新。
*   **機制**：在送進 DB 之前，在 Go 內部使用分片鎖 (Sharded Map) 或 `sync.Map` 進行去重。相同的 UserID 只保留最新的一筆時間戳記。
*   **優勢**：將 5 筆寫入合併為 1 筆，大幅減少最終需要處理的資料量。

### 2. 批次寫入 (Batch Insert / Upsert)
*   **機制**：將去重後的數十筆或數百筆資料，組裝成單一條 SQL 語法：
    ```sql
    INSERT INTO users (id, last_login) VALUES (1, 'time1'), (2, 'time2') 
    ON DUPLICATE KEY UPDATE last_login = VALUES(last_login);
    ```
*   **優勢**：
    *   **減少 RTT**：將 100 次的網路往返壓縮成 1 次。
    *   **順序寫入**：MySQL 只需要執行一次 Transaction，寫入一次 Redo Log 與 Binlog。將最消耗效能的「隨機 I/O」極大程度轉化為了磁碟最喜歡的「順序 I/O」。

---

## 4. 第四樂章：終極擴展 - 讀寫分離與分表分庫 (The Ultimate Scaling)

在討論完應用層的防禦後，如果底層 IOPS 依然撐不住，我們才需要動用架構層級的手術。這也是面試中常被問到的：**讀寫分離 (Read/Write Splitting)** 與 **分庫分表 (Sharding)** 在 IOPS 防禦上的定位。

### 1. 讀寫分離 (Read/Write Splitting)：治標不治寫
*   **定位**：主要解決的是 **CPU 負載**與 **讀取 IOPS**，無法解決寫入 IOPS 瓶頸。
*   **代價與陷阱**：主從複製 (Replication) 存在延遲。剛寫完主庫立刻去從庫讀，可能讀不到最新資料。必須在業務層容忍「最終一致性」，或透過特定機制（例如寫入後 N 秒內強制讀主庫）來繞過。

### 2. 分庫分表 (Sharding)：物理極限的終極解法
*   **定位**：當寫入流量連上述的 `Write-behind` 與 `Batching` 都壓不下來時，唯一的物理出路。透過將資料分散到多台獨立的實體機器上，**線性擴展寫入 IOPS**。
*   **代價與陷阱 (極高)**：這是一條不歸路。一旦切分，你將面臨「跨表 JOIN 報廢」、「分散式交易 (Distributed Transaction) 複雜度激增」、「全域唯一 ID 依賴 (如 Snowflake)」、「資料傾斜 (Data Skew)」等史詩級災難。

**結論**：在系統架構演進中，**「應用層優化 (緩存/合併/批次)」永遠優先於「架構層切割 (分離/分表)」**。不要一上來就喊分表分庫，先榨乾單機與記憶體的極限。

---

## 5. 第五樂章：總結 - 資料價值的取捨哲學 (Finale)

防禦 IOPS 的本質，是基於「資料價值分類 (Data Value Categorization)」的取捨：

*   **核心金流/道具**：不妥協，使用 MQ (Kafka) 做緩衝，保證 At-least-once 投遞，配合 MySQL 樂觀鎖與 Transaction 確保絕對一致性。
*   **高頻狀態/行為追蹤**：承擔萬分之一的掉資料風險 (Pod 異常崩潰)，換取 `Write-behind` + `Write-Combining` 帶來的百倍 IOPS 效能解放。

架構沒有絕對的完美，只有在吞吐量、延遲與資料持久性 (Durability) 之間，為當前業務找到最優美的平衡點。
