# Book 6.14: MongoDB 原生叢集：分散式架構的統治力 (MongoDB Native Cluster)

如果說 WiredTiger 引擎賦予了 MongoDB 驚人的「單機極限效能」，那麼 MongoDB 真正的殺手鐧，其實是它「打從底層就為了分散式系統而生」的叢集架構。

在傳統的 MySQL 世界裡，一旦單機的 CPU 或是硬碟 I/O 被打滿，想要進行「讀寫分離 (Replica)」或是「分庫分表 (Sharding)」，通常需要額外的中間件、路由層或應用端規則配合。這不是做不到，而是工程成本會明顯上升，跨表 JOIN、分片鍵選擇與維運一致性都會變成架構問題。

MongoDB 則是直接把 Replica Set 與 Sharded Cluster **內建進原生架構**。對於開發者 (Go 程式端) 來說，日常 CRUD API 幾乎不需要因為節點數量增加而改寫；真正需要設計的是 read preference、write concern、shard key 與查詢模式。

---

## 1. 第一樂章：讀寫分離的守護神：Replica Set (副本集)

當遊戲上線，單機 MongoDB 同時要吃下幾萬筆的寫入，又要應付後端系統不斷撈取排行榜的讀取時，I/O 絕對會吃不消。這時我們會啟用 **Replica Set (副本集)** 架構。

### 架構原理
一個標準的 Replica Set 至少由三個節點組成（例如 1 Primary + 2 Secondaries）：
*   **Primary (主節點)**：唯一擁有「寫入權限」的王座。所有的 `INSERT` 與 `UPDATE` 指令都只能砸向這裡。
*   **Secondary (次節點)**：不斷透過非同步的方式，去拉取 Primary 的 `Oplog` (類似 MySQL 的 Binlog)，並在自己的記憶體中重播，保持資料一致。

### 效能解放：讀寫分離
在 Application (Go) 端的 Driver 設定中，你可以輕易設定 **Read Preference**：
*   **Primary (預設)**：讀寫都找主節點，保證絕對的強一致性 (不會讀到舊資料)。
*   **SecondaryPreferred**：所有的寫入去找 Primary，但**所有的查詢 (Read) 全部導向 Secondary 節點！**
透過這個簡單的參數，你瞬間把沉重的讀取壓力全部從主節點卸除，讓主節點可以專心應付極限的寫入 QPS。

### 高可用性 (HA) 的自動容災
如果今天 Primary 所在的那台機器突然燒毀了怎麼辦？
在 MySQL，你可能需要額外的 HA 工具或 DBA 流程來切換主節點。但在 MongoDB 中，剩下的 Secondary 節點會在內部發起選舉流程，在短短幾秒內推舉出新的 Primary。Application 端仍可能遇到短暫 retry、timeout 或重新選路，但 Driver 通常能協助重新連線，讓故障恢復變成可預期的流程。

---

## 2. 第二樂章：突破物理極限的終極武器：Sharded Cluster (分片叢集)

如果今天你的遊戲爆紅，單一 Primary 節點的 CPU 和 RAM 就算不處理讀取，光是「寫入」就已經達到 10 萬 QPS 的物理極限了，該怎麼辦？

這時候就要祭出資料庫的終極型態：**橫向擴展 (Horizontal Scaling)**。在 MySQL 叫分庫分表，而在 MongoDB 叫 **Sharding**。MongoDB 的 Sharding 架構設計得極度龐大且優雅。

### Sharding 的三大核心元件
當你的系統龐大到啟動 Sharding 時，架構會變成這樣：

1. **Mongos (路由代理伺服器)**
   這是一個沒有儲存任何資料的純 Route 節點。 Application (Go API) 不再直接連線到資料庫，而是把 query 統統丟給 `mongos`。對於開發者來說，`mongos` 看起來就跟普通的 MongoDB 完全一樣。
2. **Config Servers (大腦中樞)**
   這是一個小型的 Replica Set，裡面儲存了「全域的資料地圖 (Metadata)」。裡面記載著「哪一個範圍的資料，被放在哪一台實體硬碟上」。
3. **Shards (分片資料庫)**
   這才是真正儲存資料的地方（Shard 01, Shard 02...）。為了保證資料不遺失，**其實 每一個 Shard 本身，都是一個完整的 3 節點 Replica Set ！**

### 寫入流程解密 (Shard Key 的魔法)
假設你設定用 `user_id` 作為 **Shard Key (分片鍵)**，並採用 `Hashed (雜湊)` 分片：
1. 程式端下達：`db.users.insert({user_id: 1005, name: "Frankie"})`。這指令會先抵達 `mongos`。
2. `mongos` 收到指令，立刻對 `1005` 進行 Hash 計算，得出一個數值。
3. `mongos` 去問 Config Server：「請問 Hash 值是這個的資料，該歸誰管？」
4. Config Server 查閱地圖，回答：「這個範圍歸 Shard 02 管喔。」
5. `mongos` 立刻將這筆 Insert 指令精準投遞給 **Shard 02 的 Primary 節點** 寫入硬碟。

### 為什麼 MongoDB 的 Sharding 這麼神？
* **路由透明，但不是成本消失**：開發者下達 `find()` 時，如果不帶 Shard Key，`mongos` 會自動發送廣播 (Broadcast) 給所有的 Shards，等大家各自從 B-Tree 查出結果後，`mongos` 會在記憶體幫你做聚合與排序，然後回傳給你。這讓程式碼簡單，但代價是 scatter-gather 查詢會變貴，所以好的 Shard Key 仍然是架構成敗的核心。
* **自動平衡 (Auto-Balancing)**：當 Shard 01 的硬碟快滿了，而 Shard 03 很空時。MongoDB 背景的 Balancer 程序會覺醒，自動在半夜把 Shard 01 的 Chunk (資料塊) 搬移到 Shard 03，並且自動更新 Config Server 的地圖。全程服務不中斷！

**結論**：MongoDB 在巨量資料與高併發場景中的優勢，不只來自 WiredTiger 單機引擎，也來自它把分散式路由、容災選舉、資料搬移封裝成原生機制。但這不代表可以毫無代價地疊加機器；真正的上限會由 Shard Key、熱點資料、跨分片查詢、網路與硬體容量共同決定。
