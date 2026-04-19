# Book 6.15: Apache Cassandra 底層深潛 (Apache Cassandra Deep Dive)

如果說 MySQL 是「強紀律的金融金庫」，MongoDB 是「彈性快速的 JSON 裝甲車」，那麼 Cassandra 就是一台**「為了高吞吐 Append-Only 寫入而生的無敵流水帳機器」**。

它被 Netflix、Apple、Discord 等公司用來應付每天數 PB 級別的時序資料寫入，而且在任何一台節點故障時，整個叢集依然能無縫提供服務。

---

## 1. 第一樂章：儲存引擎：LSM-Tree 為什麼碾壓 B-Tree 的寫入？

Cassandra 底層採用的是 **LSM-Tree (Log-Structured Merge Tree)**，而不是 MySQL / MongoDB 使用的 B-Tree。這個選擇從根本上決定了它的寫入霸道之處。

### B-Tree 的寫入痛點回顧
```
新資料進來 → 找到正確的葉節點位置（隨機讀取！）
           → 插入資料
           → Page 滿了 → Page Split → 修改父節點（又是隨機 I/O！）
```
B-Tree 的問題：**每次寫入都可能引發隨機 I/O**，對 SSD 的寫入放大極其嚴重。

### LSM-Tree 的寫入路徑：永遠只做順序 I/O

```
新資料進來
    │
    ├──→ ① CommitLog（WAL）：順序 Append 落盤，保證不丟資料
    │       （等同 MySQL 的 Redo Log，這步完成就可以回傳成功）
    │
    └──→ ② MemTable：資料放入記憶體裡的有序跳表 (Skip List)
               │
               │（當 MemTable 達到閾值，約 64MB~256MB）
               ▼
           ③ 背景 Flush → SSTable（Sorted String Table）
               │    順序寫入磁碟的不可變檔案！
               │    內部按 key 排好序了
               ▼
           ④ 背景 Compaction：合併多個 SSTable，清除舊版本
```

**關鍵點**：
- 寫入硬碟的資料（CommitLog + SSTable）**永遠是順序寫入**！
- B-Tree 的隨機 I/O 問題從根源消失了
- SSD 最愛順序寫入，吞吐量可以達到 B-Tree 的數倍到數十倍

---

## 2. 第二樂章：讀取路徑：為什麼 LSM-Tree 讀取比較貴？

寫入的天堂背面，是讀取需要付出的代價：

```
讀取 user_id = 'u001' 的資料

Step 1: 查 MemTable（記憶體，最快）
        │ 沒找到 → 繼續

Step 2: 查 Bloom Filter（每個 SSTable 都有）
        這是一個機率型資料結構：
        「這個 key 肯定不在我這個 SSTable 裡」（100% 準確）
        「這個 key 可能在我這個 SSTable 裡」（偶爾誤報）
        ├─ Bloom Filter 說不在 → 跳過這個 SSTable（省掉一次磁碟 I/O）
        └─ Bloom Filter 說可能在 → Step 3

Step 3: 查 SSTable 的 Index（磁碟）
        找到 key 的大概位置

Step 4: 讀取實際資料
        （可能需要掃描多個 SSTable，每個都有一個版本）

Step 5: 合併多個 SSTable 的版本，取最新的
```

**讀取可能需要掃描多個 SSTable**，這被稱為 **Read Amplification（讀取放大）**。這就是 LSM-Tree 的天然弱點：寫入快，讀取相對慢。
Bloom Filter 是緩解這個問題的核心武器，它能以極小的記憶體代價（幾 KB per SSTable）過濾掉大量不必要的磁碟 I/O。

---

## 3. 第三樂章：資料模型：Partition Key + Clustering Key 的藝術

Cassandra 的資料模型設計是整個系統最重要的架構決定，設計好就飛，設計爛就死。

### 基本概念
```sql
CREATE TABLE wallet_events (
    user_id    TEXT,       -- Partition Key：決定資料去哪台機器
    event_seq  BIGINT,     -- Clustering Key：決定同一個 Partition 內的排序
    tx_id      TEXT,
    amount     BIGINT,
    event_type TEXT,
    PRIMARY KEY (user_id, event_seq)
) WITH CLUSTERING ORDER BY (event_seq ASC);
```

- **Partition Key**：Cassandra 對它做 Hash，決定這筆資料要放在叢集的哪個節點上。**相同 Partition Key 的資料，保證在同一台機器上，且按 Clustering Key 排好序了。**
- **Clustering Key**：在同一個 Partition 內的排序鍵，決定資料在 SSTable 裡的物理排列。

### 為什麼 Partition 設計這麼重要？

**查詢效率**：
```sql
-- ✅ 極速！只打一台機器，資料本地，且已排序
SELECT * FROM wallet_events
WHERE user_id = 'u001' AND event_seq > 700;

-- 💀 災難！Cassandra 必須廣播到所有節點（類似 Scatter-Gather）
-- 沒有 WHERE user_id，不知道資料在哪台機器
SELECT * FROM wallet_events WHERE amount > 100;
```

**熱點問題**：
如果你用 `event_type`（只有 "DEBIT" / "CREDIT" 兩種值）當 Partition Key，所有資料只會分佈在兩個 Partition，兩台機器承載全部流量。叢集再大也無意義。
**好的 Partition Key 需要高基數 (High Cardinality)**，例如 `user_id` 讓百萬玩家的資料均勻分散。

### 複合 Partition Key 與時序設計

對於錢包事件流水帳，一個更成熟的設計：
```sql
CREATE TABLE wallet_events (
    user_id    TEXT,
    month      TEXT,       -- 用月份做 Partition 的一部分，防止單一 Partition 無限增長
    event_seq  BIGINT,
    ...
    PRIMARY KEY ((user_id, month), event_seq)
);
-- (user_id, month) 是複合 Partition Key
-- 查詢：WHERE user_id = 'u001' AND month = '2026-04'
```

這解決了 **Partition 無限增長 (Unbounded Partition)** 的問題：如果一個玩家打了 10 年球，單一 Partition 可能累積幾百萬筆，Compaction 時這個 Partition 會成為效能黑洞。

---

## 4. 第四樂章：一致性等級：CAP 定理的現實抉擇

Cassandra 是一個 **AP 系統（高可用 + 分區容忍）**，但它允許你在查詢層面動態調整一致性：

```
叢集設定：Replication Factor = 3（每份資料有 3 個副本，分佈在 3 個節點）

寫入時的一致性等級：
  ONE     → 只要 1 個節點確認即可。最快，有可能讀到舊資料
  QUORUM  → 需要過半數節點（3/2+1 = 2 個）確認。速度與一致性的平衡
  ALL     → 需要全部 3 個節點確認。最慢，但保證強一致。1 個節點故障就失敗

讀取時同樣可以設定：
  ONE     → 從最近的 1 個節點讀取（最快，可能讀到舊資料）
  QUORUM  → 從過半節點讀取並取最新版本
  ALL     → 從全部節點讀取
```

**黃金法則**：`Write QUORUM + Read QUORUM` 能在 Replication Factor=3 下實現強一致：
過半寫入 (2) + 過半讀取 (2) > 副本數 (3)，因此讀取一定能讀到最新寫入。

對於錢包系統：
- 寫入事件流水帳：`ONE` 或 `QUORUM`（視業務對資料遺失的容忍度）
- 讀取餘額快照（Rehydration 用）：`QUORUM`，確保看到最新快照

---

## 5. 第五樂章：輕量交易 LWT：Cassandra 的樂觀鎖

前面提到 Cassandra 的 **Lightweight Transaction (LWT)** 可以做冪等性保護：

```sql
-- 只有 tx_id 不存在時才插入（冪等保護）
INSERT INTO wallet_events (user_id, event_seq, tx_id, amount)
VALUES ('u001', 751, 'tx_abc123', -200)
IF NOT EXISTS;
```

```sql
-- Compare-and-Set：只有 version 符合預期才更新（樂觀鎖）
UPDATE wallet_balance
SET balance = 49800, version = 2
WHERE user_id = 'u001'
IF version = 1;
```

**LWT 的底層代價**：
LWT 使用 **Paxos 共識協議**，需要兩輪網路 Round Trip（Prepare + Accept），涉及 Replication Factor 個節點。延遲通常是普通寫入的 **4~10 倍**，吞吐量大幅下降。

**架構師守則**：LWT 是最後防線，不要放在熱路徑上。Actor 模型的記憶體去重才是第一道快速防線，LWT 只在邊緣情況（Actor 崩潰重啟、雙寫衝突）才會被觸發。

---

## 6. 第六樂章：Compaction 策略：Cassandra 的背景「整理書架」

隨著資料越來越多，SSTable 也越積越多，Cassandra 的背景 Compaction 需要定期合併它們。選對策略非常重要：

### STCS (Size-Tiered Compaction Strategy) - 預設
```
小 SSTable 積多了 → 合併成中 SSTable
中 SSTable 積多了 → 合併成大 SSTable
大 SSTable 積多了 → 合併成超大 SSTable
```
- **優點**：寫入密集場景效能好，Compaction 觸發相對少
- **缺點**：讀取放大高，合併大 SSTable 時 I/O 衝擊大
- **適用**：純寫入為主的場景（如 Event Log）

### LCS (Leveled Compaction Strategy)
```
把 SSTable 分成多個 Level，每個 Level 的 SSTable 不重疊
Level 0 → Level 1 → Level 2 → ...
每個 Level 的大小是上一個 Level 的 10 倍
```
- **優點**：讀取放大小，讀取效能好
- **缺點**：寫入放大大，持續頻繁 Compaction，I/O 佔用高
- **適用**：讀多寫少，需要穩定讀取效能

### TWCS (Time-Window Compaction Strategy) - 時序資料首選
```
按時間窗口分組 SSTable（例如每小時一組）
同一個時間窗口內的 SSTable 互相合併
過了時間窗口就不再觸發 Compaction（資料已不可變）
```
- **優點**：對時序資料幾乎沒有讀取放大，TTL 清除效率極高
- **缺點**：只適合有 TTL 的時序資料，不適合需要更新舊資料的場景
- **適用**：錢包事件流水帳、IoT 感測器日誌、聊天訊息（**這是包網錢包的首選**）

---

## 7. 第七樂章：叢集架構：Gossip + 一致性 Hash + Virtual Nodes

Cassandra 叢集沒有 Master，所有節點**完全平等 (Peer-to-Peer)**。

### Gossip 協議：節點之間的八卦通訊
```
節點 A 每秒隨機挑 1~3 個鄰居，互相交換「誰還活著、誰的資料有多新」
幾秒之內，叢集中所有節點都知道了其他所有節點的狀態
（類似八卦傳播：你告訴我，我告訴他，幾輪後全村都知道了）
```

### 一致性 Hash Ring

所有節點圍成一個虛擬的 Hash 環（0 ~ 2^64）：
```
        Node A (0~25%)
       /               \
Node D (75~100%)   Node B (25~50%)
       \               /
        Node C (50~75%)
```

當你寫入 `user_id = 'u001'` 時：
1. Cassandra 對 `user_id` 做 Hash，得到一個數值（例如落在 30%）
2. 這個資料由負責 `25%~50%` 的 Node B 主導
3. 按 Replication Factor=3，繼續複製到 Node C 和 Node D

### Virtual Nodes (vNodes)

為了防止新節點加入時資料搬移不均，每個實體節點會分成 **256 個虛擬節點**分散在 Hash 環上，讓負載更均勻，擴容時更平滑。

---

## 8. 第八樂章：實戰場景總結

| 場景 | 推薦 | 原因 |
|------|------|------|
| 錢包事件流水帳 | ✅ Cassandra + TWCS | Append-Only，時序資料，需要高吞吐 |
| 玩家行為日誌 | ✅ Cassandra | Append-Only，按 user_id 查歷史 |
| IoT 感測器資料 | ✅ Cassandra | 時序，高頻寫入 |
| 玩家檔案（彈性欄位）| ❌ MongoDB | 需要 Schema-less，讀寫混合 |
| 金融交易（強 ACID）| ❌ MySQL | 需要跨表交易，強一致性 |
| 即時排行榜 | ❌ Redis | 需要 Sorted Set，毫秒級讀取 |
| 全局廣播 Event 訂閱 | ❌ Kafka | 需要多消費者重播，流式處理 |

**架構師結語**：Cassandra 的設計哲學是「**用一致性換取可用性和吞吐量**」。它天生為「寫多讀少、按主鍵或時序查詢」的場景而生。選錯它（例如拿來做複雜的 JOIN 查詢）是自找苦吃；選對它（Event Sourcing、時序資料、用戶行為流水帳），則是讓系統在百萬 QPS 下依然如絲般滑順的秘密武器。
