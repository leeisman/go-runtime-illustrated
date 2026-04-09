# Book 6.12: MySQL 分庫分表 (Sharding) 的愛與恨

當系統流量與資料量暴增，我們遲早會遇到 MySQL 單機性能的終極天花板。這時架構師們嘴裡總會蹦出四個字：「分庫分表」。
但在你決定把完整的資料庫切碎之前，請先聽聽這句架構界的至理名言：

> **萬用大原則：「只要能不分庫分表，就絕對不要分庫分表！」**

這是一張不可逆的單程車票。一旦切下去，SQL 原生的強大功能（JOIN、ACID Transaction）將有半數直接殘廢。

我們來拆解：為了突破什麼極限，我們被迫簽下這份惡魔契約？又要如何收拾後果？

---

## 1. 第一樂章：單體系統的「分表」 (Table Sharding)

在**同一台**實體伺服器、同一個資料庫實例裡，把一張 1 億筆資料的 `orders` 表，切成 10 張各 1000 萬筆的 `orders_0` 到 `orders_9`。

### 為什麼在同一台機器還要切？(拯救 B+ Tree)
既然都在同一顆硬碟上，總 I/O 上限並沒有改變，為什麼要分表？

1.  **壓低 B+ Tree 的身高**：
    我們在 Book 6.8 學過，3 層的 B+ Tree 可以裝 2000 萬筆資料（需 3 次硬碟 I/O）。若資料爆量到 1~2 億筆，樹高必定突破到 4 層以上。**每一層代表多一次 Random I/O**。
    把大表切成 10 張小表，能確保所有的 B+ Tree 都被強制壓低在 3 層以內，維持極致的查詢速度。
2.  **縮小鎖的影響範圍 (Blast Radius)**：
    高併發寫入時，雖然有 Row Lock，但資料太密集會增加 **Gap Lock (間隙鎖)** 互撞與 Deadlock 的機率。分表可以直接從物理層面隔開不同群體的寫入熱區。

---

## 2. 第二樂章：分散式系統的「分庫」 (Database Sharding)

把資料庫拆分到 **多台不同的實體伺服器** (Server A 裝 Node 0, Server B 裝 Node 1)。

### 為什麼要分庫？(解除物理 I/O 上限)
當你已經優化了所有的索引、快取、連線池，但你的 MySQL 還是發出了這樣的悲鳴：
1.  **CPU 撐不住**：每秒 10 萬次的高頻寫入 (UPDATE/INSERT)，單台 CPU 核心全部被 InnoDB 的 Context Switch 耗盡。
2.  **硬碟 IOPS 撐不住**：SSD 的寫入頻寬（例如 500MB/s）被 Redo Log 刷盤塞滿。
3.  **網路卡撐不住**：每秒幾十 G 的查詢結果把機器的 10G 網卡打滿。

此時，「單機分表」已經救不了你。你必須把壓力分包給別的實體機器。這就是分散式分庫：**徹底粉碎單機能力天花板的唯一方法。**

---

## 3. 第三樂章：惡魔的代價 (Trade-offs & Solutions)

分庫分表換來了無限拓展的能力，但你的業務代碼將迎來兩場恐怖的噩夢：**跨節點 JOIN** 與 **分散式事務**。

### 噩夢 1：JOIN 死了 (The Death of JOIN)

**問題場景**：你想要查「User 42 (在 DB 1) 今年買過的所有商品名稱 (在 DB 2)」。
在單體架構，一個 `JOIN` 加小小的索引瞬間就找到了。
在分散式架構，連線在不同的機器上，SQL 的 `JOIN` 語法直接 **報廢**。

**架構解法 (Workarounds)**：

1.  **應用層組裝 (App-Level JOIN)**：
    用 Go 代碼發兩次 Query。
    *   Query 1 (去 DB 1): `SELECT item_ids FROM orders WHERE user_id = 42`
    *   Query 2 (去 DB 2): `SELECT name FROM items WHERE id IN (...)`
    *   *缺點：增加網路來回延遲 (RTT)。*

2.  **冗餘欄位 (Data Denormalization)**：
    打破 正規化 (Normalization) 規則。在 `orders` 表裡面直接多塞一個欄位 `item_name`。
    寫入時辛苦一點，更新兩次；查詢時只要查單表！這叫「空間換時間」。

3.  **CQRS (讀寫分離) 與異步同步**：
    寫入時依然分開寫在各個小庫。但在背景用 Kafka / Debezium 監聽 MySQL 的 Binlog。將所有庫的更新事件收集起來，送進一個適合做海量查詢的系統 (例如 **Elasticsearch** 或 Hadoop)。
    *   寫入走 MySQL Shard。
    *   複雜 JOIN 查詢走 ES 總表。

### 噩夢 2：事務斷裂 (Distributed Transactions)

**問題場景**：A 帳號 (在 DB 1) 轉帳 100 元給 B 帳號 (在 DB 2)。
單體架構下 `BEGIN; UPDATE A; UPDATE B; COMMIT;` 就搞定。
分散式架構下，如果 DB 1 成功，但此時網路斷了，DB 2 沒收到更新。A 被扣了錢，B 沒拿到錢，準備上新聞。

**架構解法 (Solutions)**：

1.  **絕對首選：規避法 (Route by Shard Key)**
    治標先治本。為什麼 A 和 B 會在不同機器？
    如果你在做高頻交易系統，你可以針對「公會」或「房間」當作 Shard Key。保證同一場交易的所有參與者，都被 Hash 到 **同一個 DB** 實例內。
    只要在同一個 DB 內，你就可以繼續開心地用原生 `tx.Commit()`！

2.  **強一致性：2PC (Two-Phase Commit)**
    業界非常著名的分散式事務標準 (例如 XA 協議)。
    *   第一階段：協調者問 DB1 和 DB2「你們準備好寫入了嗎？先鎖住資源！」
    *   第二階段：大家回報 OK，協調者再廣播「大家一起 Commit！」
    *   *致命傷：性能極差！鎖資源的時間太長，高併發下會導致系統吞吐量暴跌。互聯網公司極少使用。*

3.  **最終一致性：Outbox Pattern (發件匣模式) + MQ**
    這是微服務架構的終極殺招。（我們在下一章 MQ 會深入討論）
    *   DB 1 在扣錢的同時，在同一個 DB 的「發件匣表 (Outbox)」寫入一筆：[準備給 B 加 100 塊]。這利用了單機事務保證 100% 寫入。
    *   背景程式把 Outbox 的紀錄丟進 Kafka。
    *   DB 2 消費 Kafka 訊息，幫 B 加 100 塊（需實現冪等性防止重試）。
    *   *概念：放棄「同時成功」，接受「短暫的不一致」，換取高吞吐量與最終一致性 (Eventual Consistency)。*

---

## 4. 終章：架構師的決策樹 (The Sharding Matrix)

遇到效能平頸，請嚴格按照這個順序升級（越靠上的越廉價、越安全）：

1.  **SQL 調優**：加 Index，改寫爛 Query（如避免 `SELECT *` 或大分頁）。 (不花錢)
2.  **引入 Cache**：讀取太慢？加 BigCache 或 Redis 防禦。 (架構變更小)
3.  **升級硬體 (Scale Up)**：花錢買更大的 SSD、更多的 RAM，擴大 Buffer Pool。 (最簡單的鈔能力)
4.  **讀寫分離 (Read/Write Splitting)**：把複雜的 SELECT 導向 Read Replica (從庫)。 (改動中等)
5.  **單機分表 (Partitioning / Table Splitting)**：解決 B+ Tree 樹高問題與超大表的鎖競爭。 (改程式碼，痛苦開始)
6.  **分散式分庫分表 (Database Sharding)**：解決實體 I/O 與 CPU 極限。 (分散式事務與 JOIN 問題降臨，地獄模式)

記住，**分庫分表是最後的無奈之舉**。如果在你流量只有 1 萬的時候就提早做了分庫分表，這不叫有遠見，這叫「過度工程 (Over-Engineering) 操作失誤」，你會為了修無法 JOIN 的 Bug 而痛不欲生。
