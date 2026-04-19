# Book 6.10: RocketMQ 架構與進階特性：業務邏輯的守護神 (RocketMQ Architecture & Advanced Features)

如果有個系統，既需要 Kafka 的高吞吐，又需要傳統 MQ (ActiveMQ/RabbitMQ) 的複雜業務功能 (延遲、事務、過濾)，那就是 **Apache RocketMQ**。

在阿里巴巴的雙 11 大促中，RocketMQ 承擔了兆級別的訊息流轉。它與 Kafka 最大的區別在於：**Kafka 專注於 Log (日誌)，而 RocketMQ 專注於 Business (業務)。**

---

## 1. 第一樂章：為什麼需要 RocketMQ？(Philosophy)
在微服務架構中，RocketMQ 的核心價值在於 **完全解耦 (Decoupling)**。

*   **Sync (RPC/HTTP)**:
    *   Payment Service -> Logistics Service。
    *   **缺點**: 如果 Logistics 掛了，Payment 也會跟著報錯 (連坐法)。且 Payment 必須等待 Logistics 回應，Latency 是所有下游的總和。
*   **Async (RocketMQ)**:
    *   Payment Service -> **RocketMQ** <- Logistics Service。
    *   **優點**: Payment 丟給 MQ 就收工 (Latency 極低)。Logistics 掛了沒關係，訊息堆在 MQ 裡，等它復活再慢慢消費 (削峰填谷)。Payment 完全不用認識下游是誰。

**為什麼不用 Kafka 做這個？**
因為業務系統需要 **「精確的重試」** 與 **「豐富的路由」**：
1.  **Native Retry (原生重試)**: 消費失敗了，RocketMQ 會自動將訊息丟回「重試隊列」，並依照指數退避 (1s, 5s, 10s...) 重推。Kafka 沒這功能，你得自己手寫死信隊列 (DLQ)。
2.  **Tag/SQL Filtering**: 一個 Topic 可以有多種 Tag (e.g., `TagA || TagB`)，讓下游只訂閱自己關心的業務，不用把整個 Topic 拉回來再過濾 (省頻寬)。

RocketMQ 就像一個 **「懂業務的智慧郵差」**，而 Kafka 比較像 **「單純的暴力輸送帶」**。

---

## 2. 第二樂章：核心架構差異 (Storage Design)

為什麼我們說 RocketMQ 可以支援 **數萬個 Topic** 而 Kafka 不行？秘密在於硬碟的寫入方式。

### 1.1 Kafka's Model: Partition-Based
*   **設計**: 每個 Partition 是一個獨立的物理資料夾 (Directory)。
*   **寫入**: 若有 1000 個 Topic，每個 Topic 有 10 個 Partition -> 硬碟上就有 10,000 個資料夾。
*   **瓶頸**: 當 Producer 同時向這 10,000 個 Partition 寫入時，作業系統的磁頭必須在不同磁軌間瘋狂跳轉 (**Random I/O**)。這導致吞吐量隨著 Topic 數量增加而斷崖式下跌。

### 1.2 Deep Dive: The Storage Mechanics (CommitLog 寫入細節)
RocketMQ 的寫入效能之所以能打，全靠 **Sequential Write (順序寫)** 與 **mmap**。

*   **The Structure**:
    *   **CommitLog**: 這是真正的數據倉儲。所有 Topic 混在一起寫。預設每個檔案 **1GB**。
    *   **ConsumeQueue**: 這是索引 (Index)。每個 Element 固定只有 **20 Bytes** (Offset 8 + Size 4 + TagCode 8)。
*   **The Write Path**:
    1.  **Memory Map (mmap)**: Broker 不使用標準的 `write()` Syscall，而是使用 Java 的 `MappedByteBuffer` (對應 Linux `mmap`)。這將檔案直接映射到 User Space 記憶體地址，減少一次 Kernel Copy。
    2.  **Lock**: 為了保證 CommitLog 嚴格順序，寫入時需要**加鎖** (SpinLock / ReentrantLock)。雖然有鎖，但因為只是「追加記憶體操作 (Append RAM)」，速度極快。
    3.  **Flush (刷盤)**:
        *   **Async Flush (預設)**: 寫入 PageCache 就回傳 OK。由 OS 決定何時落盤。速度快，斷電會掉數據。
        *   **Sync Flush**: 啟用 **Group Commit** 機制。累積一小批 Request (或等待 10ms) 後一次 fsync。只有在數據真正落盤後才回傳 ACK。

*   **The Dispatch Path (ReputMessageService)**:
    這是一個背景線程，也是 RocketMQ 與 Kafka 最大的架構差異。
    *   **任務**: 它不斷掃描 CommitLog 的新數據。
    *   **Dispatch**: 解析出 Topic, QueueId，然後生成 20 Bytes 的索引，寫入對應的 **ConsumeQueue**。
    *   **Trade-off**: 這意味著「寫入 CommitLog」到「Consumer 可見」之間有微小的**延遲 (Index Lag)** (通常 < 1ms)。但這換來了寫入的極致吞吐。

### 1.3 Deep Dive: The Read Path (隨機讀的代價)
Kafka 讀取是順序的 (Sequential Read)，但 RocketMQ 的讀取在本質上是 **Random Read**。
*   **Flow**:
    1.  Consumer 請求 `Get(TopicA, Queue0, Offset=100)`。
    2.  Broker 查 **ConsumeQueue**: 找到 Offset 100 對應的 CommitLog Offset 是 `55667788`，Size 是 `1KB`。
    3.  Broker **跳轉 (Seek)** 到 **CommitLog** 的 `55667788` 位置讀取數據。
*   **Optimization**: 雖然是隨機讀，但在 **近期數據 (Hot Data)** 場景下，OS 的 PageCache 幾乎都 Cache 住了 CommitLog 的尾端，所以還是很快 (RAM Speed)。只有在讀取 **冷數據 (Backlog)** 時，才會觸發頻繁的 Disk Seek，這時效能會下降。

### 1.4 Deep Dive: The Message Lifecycle (這一生發生了什麼？)
讓我們追蹤一條訊息從進入 Broker 到被 Consumer 帶走的完整物理旅程。

1.  **Arrival (寫入)**:
    *   Producer 透過 Netty (TCP) 送來數據。
    *   Broker 收到後，將數據寫入 `MappedByteBuffer` (這是 Java 對應 `mmap` 的封裝)。
    *   **OS 行為**: OS 將數據直接寫入 Kernel 的 **Page Cache**。此時資料標記為 Dirty，雖然還沒寫入物理硬碟，但對所有 Process (包含稍後的讀取) 都是可見的。

2.  **Indexing (建索引)**:
    *   Broker 有個背景線程 `ReputMessageService` (準實時，延遲 < 1ms)，不斷掃描 CommitLog 的新數據。
    *   它將 (Offset, Size, TagCode) 寫入 **ConsumeQueue** (也是 mmap)。
    *   **關鍵**: 沒有這一步，Consumer 根本不知道有新訊息，因為 Consumer 只認 ConsumeQueue。

3.  **Consumption (長輪詢 Long Polling)**:
    *   RocketMQ 的 Consumer 雖然叫 `PushConsumer`，但底層其實是 **Pull**。
    *   **Step A**: Consumer 發送 `PullRequest` 給 Broker 給我新資料。
    *   **Step B (Suspend)**:
        *   若有新資料：Broker 立刻回傳。
        *   若**沒有**新資料：Broker **不回傳，也不關閉連線**，而是將這個 Request **掛起 (Suspend)** 最多 15 秒。
    *   **Step C (Wake Up)**:
        *   當 Producer 寫入新訊息 (Step 1)，且索引建立完成 (Step 2) 後。
        *   Broker 的 `PullRequestHoldService` 會發現有新資料，立刻 **喚醒** 剛剛被掛起的 Request，將數據寫回給 Consumer。
    *   **效果**: 這結合了 Push 的即時性 (有資料立刻回) 與 Pull 的流量控制 (Consumer 主動發起)。

**OS 的角色總結**:
RocketMQ 與 Kafka 一樣依賴 **Page Cache**。如果 Consumer 追得緊 (Tail Read)，資料全程都在 RAM 裡流轉 (Socket In -> Page Cache -> Socket Out)，完全不觸發物理 IO。

### 1.5 Deep Dive: Consumption Details (Batch & ACK)
很多開發者會好奇：「RocketMQ 也是 Batch 處理嗎？失敗了會怎樣？」

1.  **Two-Layer Batching (雙層批量)**:
    *   **Network Layer (Pull)**: 為了節省網路 RTT，Consumer 底層預設會一次拉取 **32 條** (`PullBatchSize`)。
    *   **App Layer (Consume)**: 為了追求低延遲與精細控制，SDK 預設會**一條一條** (`ConsumeMessageBatchMaxSize=1`) 餵給您的 Listener 處理。
    *   **對比 Kafka**: Kafka 通常是把整桶 (e.g. 500條) 直接丟給應用層，傾向於高吞吐但處理粒度較粗。

2.  **ACK & Retry (不卡單設計 - 關鍵差異)**:
    *   **Success**: 您回傳 `ConsumeSuccess` -> Broker 推進 Consumer Offset。
    *   **Fail / No ACK**: 如果您回傳 `ReconsumeLater`，或者 Consumer 在處理過程中 Crash (沒給 ACK)。
        *   **Kafka**: 會 **卡在原地** 無限重試，後面的新訊息全部被擋住 (Head-of-Line Blocking)。
        *   **RocketMQ**: 會自動將這條失敗訊息 **「踢出」** 到一個特殊的 **`%RETRY%` Topic**。
        *   **結果**: Main Queue 的 Offset **繼續往前推進**，系統不會被一條毒藥訊息 (Poison Message) 卡死。那條失敗的訊息會在 `1s, 5s, 10s...` 後由 Retry Topic 再次投遞給您。

---

## 3. 第三樂章：分散式事務訊息 (Transactional Messages)

這是 RocketMQ 最強大的武器，完美解決了微服務中的 **"Dual Write"** 問題 (要寫 DB 又要發 MQ，如何保證原子性？)。

### 2.1 The 2PC Flow (二階段提交)
RocketMQ 實作了一個半消息 (Half Message) 機制：

1.  **Phase 1: Prepare (Half Message)**
    *   Producer 發送一條 "Half Message" 給 Broker。
    *   Broker 將其存入特殊的內部 Topic (`RMQ_SYS_TRANS_HALF_TOPIC`)，對 Consumer **不可見**。
    *   Broker 回傳 "OK"。
2.  **Phase 2: Local Transaction**
    *   Producer 執行本地 DB 事務 (例如：扣款)。
3.  **Phase 3: Commit/Rollback**
    *   **若 DB 失敗**: Producer 發送 "Rollback"。Broker 標記該訊息刪除，Consumer 永遠看不見。

**Performance Cost (代價分析)**:
*   **High Latency**: 完整的 2PC 流程涉及 **3 次 Network RTT** (Send Half -> Ack -> Send Commit) 加上 **2 次 Disk I/O** (Broker 寫 Half Log + DB 寫 Redo Log)。
*   **Throughput Drop**: 相比於普通訊息 (Fire-and-Forget)，事務訊息的吞吐量通常會下降 **40% ~ 50%**。
*   **Best Practice**: 這是昂貴的操作。請僅在 **「涉及金錢/核心狀態流轉」** 時使用，千萬不要拿來送 Log。

### 2.2 The Safety Net (回查機制)
如果 Producer 在執行完 DB (Step 2) 後，還沒來得及發 Commit (Step 3) 就**當機**了，怎麼辦？
*   **Broker 的主動權**: Broker 發現某條 Half Message 已經卡很久 (預設 60s) 沒有下文。
*   **CheckLocalTransaction**: Broker 會主動連線到 Producer (任意一台)，發起回查：「這筆 TX_ID 的訂單，你們 DB 到底寫進去沒？」
*   Producer 檢查 DB，若存在則補發 Commit，不存在則補發 Rollback。
*   **結論**: 這保證了 **最終一致性 (Eventual Consistency)**。

### 2.3 Go Implementation Pattern
在使用 `apache/rocketmq-client-go/v2` 時，我們需要實作 `TransactionListener` 介面：

```go
type OrderListener struct{}

// 執行本地事務 (Step 2)
func (l *OrderListener) ExecuteLocalTransaction(msg *primitive.Message) primitive.LocalTransactionState {
    // 1. 解析 msg
    // 2. 執行 DB: INSERT INTO orders ...
    
    if err != nil {
        return primitive.RollbackMessageState // 告訴 Broker 刪掉訊息
    }
    return primitive.CommitMessageState // 告訴 Broker 放行訊息
}

// 應對 Broker 的回查 (Safety Net)
func (l *OrderListener) CheckLocalTransaction(msg *primitive.MessageExt) primitive.LocalTransactionState {
    orderID := msg.GetProperty("order_id")
    
    // 查詢 DB: SELECT count(*) FROM orders WHERE id = ?
    if exists {
        return primitive.CommitMessageState
    }
    return primitive.RollbackMessageState
}

// Main
func main() {
    p, _ := rocketmq.NewTransactionProducer(
        &OrderListener{}, // 註冊監聽器
        producer.WithNsResolver(primitive.NewPassthroughResolver([]string{"127.0.0.1:9876"})),
    )
    p.Start()
    
    // 發送事務訊息 (Step 1)
    p.SendMessageInTransaction(context.Background(), &primitive.Message{
        Topic: "Topic_Pay_Success",
        Body:  []byte("order_payload"),
    })
}
```

---

## 4. 第四樂章：Other Advanced Features (其他特異功能)

### 3.1 Delay Scheduling (延時訊息)
*   **Kafka**: 原生不支援。需要自行開發中間層 (Timer Wheel)。
*   **RocketMQ**: 原生支援。
    *   **4.x**: 18 個固定等級 (`1s 5s 10s 30s 1m 2m... 2h`)。
    *   **5.0+**: 支援任意時間 (`TimerWheel` 實作)。
*   **原理**: Broker 先把訊息偷換到 `SCHEDULE_TOPIC_XXXX`，等到時間到了，再還原回原 Topic。

### 3.2 Tag Filtering (服務端過濾)
*   Producer: `msg.SetTag("TagA")`
*   Consumer: `Subscribe("Topic", "TagA || TagB")`
*   **原理**: Broker 在 CommitLog 轉發到 ConsumeQueue 時，會把 Tag 的 HashCode 存進 Index。Consumer 拉取時 (Pull)，Broker 直接比較 HashCode，不符合的連網卡頻寬都不用浪費，直接過濾掉。

---

## 5. 第五樂章：總結 (Summary) Comparision

| Feature | Kafka | RocketMQ |
| :--- | :--- | :--- |
| **Storage** | Partition (Random IO danger) | CommitLog (Sequential IO) |
| **Topic Limit** | Low (< 1000s) | High (100k+) |
| **Throughput** | Extreme (Millions/s) | High (100k/s) |
| **Latency** | Medium (Batching focus) | Low (Push focus) |
| **Features** | Log Stream | Transaction, Delay, Broadcast |
| **Replication** | ISR (Leader/Follower) | Master/Slave (Sync/Async) |
| **Dependencies** | ZooKeeper / KRaft | NameServer (Lightweight) |

**結論**:
*   做 **Log / Metrics / Data Pipeline** -> 選 **Kafka**。
*   做 **Business Logic / Order / Payment / Game** -> 選 **RocketMQ**。

---

## 6. 第六樂章：Operational Survival Guide (維運生存指南)
使用 RocketMQ 進行微服務解耦 (Decoupling) 是「兩面刃」。好處是上游發完就跑，壞處是**下游死活上游完全不知情**。為了避免這種「異步變遺忘」，必須嚴格執行以下規範：

### 5.1 Physical Isolation (物理隔離原則)
絕對不要把 **核心交易 (Trade)** 與 **日誌監控 (Log)** 放在同一個 RocketMQ Cluster。
*   **風險**: Log 服務若發生流量暴衝 (Log Spam)，會瞬間打滿 Page Cache 與 網卡頻寬。
*   **後果**: 導致訂單訊息發不進來 (Write Latency 飆高)，影響營收。
*   **對策**: 建立獨立的 `Trade-Cluster` (SSD, 高規) 與 `Log-Cluster` (HDD, 低規)。

### 5.2 Vital Metrics (保命監控指標)
如果沒有看這三個指標，你的系統就是在裸奔：
1.  **Consumer Lag (消費堆積)**: **最重要指標**。這代表下游服務是否健康。堆積過高會導致 Page Cache 失效，引發 Read IO 雪崩。
2.  **CommitLog Append Latency (寫入延遲)**: 用來監控 Broker 狀態。正常應 < 5ms。若飆高代表硬碟或 OS 出問題。
3.  **DLQ (Dead Letter Queue) Count**: RocketMQ 自動重試 16 次失敗後，訊息會進入 DLQ。**DLQ 增加代表業務邏輯有 Bug 或是數據異常**，必須設告警人工介入，這通常意味著「丟單」了 (雖然沒報錯)。
