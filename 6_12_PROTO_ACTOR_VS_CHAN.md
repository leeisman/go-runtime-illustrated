# Book 6.12: Proto.Actor vs Go Channel：底層資料結構的降維打擊 (Proto.Actor vs Go Channel)

在 Actor 模式中，「信箱 (Mailbox)」的效能就是整個系統的天花板。
Go 原生的 `channel` 與 Proto.Actor 框架的 MPSC 佇列，本質上都是讓多個 Producer 安全地把訊息丟給一個 Consumer。
但它們的底層實作**完全不同**，這個差異決定了百萬 QPS 時誰生誰死。

---

## 1. 第一樂章：Proto.Actor 的底層：MPSC 無鎖鏈結串列

Proto.Actor 的信箱核心是一個 **MPSC (Multi-Producer, Single-Consumer) 無鎖鏈結串列**。
它的物理佈局長這樣：

```
[Head 指標]  ←──  Actor 消費者從這端拿訊息
    │
    ▼
  Node_1  →  Node_2  →  Node_3  →  nil
                             ▲
                         [Tail 指標]  ←── 所有 Producer 往這端掛訊息
```

### Producer 投遞訊息：只需要一個 CAS！

當上游 Goroutine 要投遞一條訊息時，步驟極度精簡：

```go
// 偽代碼展示 MPSC Linked List 的 Push 操作
func (q *MPSCQueue) Push(node *Node) {
    node.next = nil
    // 核心：一個原子操作，把 tail 指標從 old_tail 換成 new_node
    // 如果有其他 Producer 同時在 Push，CAS 會失敗並自動重試
    // 但爭搶的只是這一個 tail 指標，不是整個佇列的大鎖！
    prev := atomic.SwapPointer(&q.tail, unsafe.Pointer(node))
    // 把前一個尾節點的 next 指向新節點，完成串接
    // 這個寫入是安全的，因為 prev 節點只有我自己持有
    prev.next = node
}
```

**Push 只做兩件事**：
1. `atomic.Swap(tail, new_node)` → 搶佔尾端，O(1)，無鎖，CAS 自旋
2. `prev.next = node` → 串接前一個節點，這步不需要鎖（前一個節點只有我自己持有）

關鍵在於：**多個 Producer 之間只爭搶 `tail` 這一個指標**，而不是整個佇列的 Mutex 大鎖。搶輸了只需 CPU 自旋重試一次，不會陷入 OS 睡眠。

### Consumer 取出訊息：完全不需要鎖！

```go
// 偽代碼展示 MPSC 的 Pop 操作
func (q *MPSCQueue) Pop() *Node {
    head := q.head
    next := head.next  // 因為只有我一個 Consumer，讀 head 不需要鎖！
    if next == nil {
        return nil     // 佇列為空
    }
    q.head = next      // 移動 head，直接普通賦值，無需 atomic！
    return next
}
```

因為**永遠只有一個 Actor (Consumer)** 在讀取 head 端，所以 Pop 操作完全不需要任何原子操作或鎖，純粹的 Linked List 指標移動，速度是極限等級。

---

## 2. 第二樂章：Go Channel 的底層：多步驟的 Mutex + RingBuffer

Go 原生的 `channel` 被設計為通用的 **MPMC (Multi-Producer, Multi-Consumer)** 管道。它永遠要保護自己不被任何方向的併發破壞，所以它的底層結構 `hchan` 複雜很多：

```go
// Go 源碼 runtime/chan.go 中的 hchan 結構
type hchan struct {
    qcount   uint           // 當前佇列中的訊息數量
    dataqsiz uint           // RingBuffer 的容量 (make(chan T, N) 的 N)
    buf      unsafe.Pointer // 指向 RingBuffer 的指標
    sendx    uint           // 寫入指標 (下一個要寫的位置)
    recvx    uint           // 讀取指標 (下一個要讀的位置)
    recvq    waitq          // 等待讀取的 Goroutine 佇列 (睡著的 Consumer)
    sendq    waitq          // 等待寫入的 Goroutine 佇列 (睡著的 Producer)
    lock     mutex          // 保護以上所有欄位的大鎖！
}
```

### 一次 `ch <- msg` 的完整步驟拆解

```
Goroutine A 執行 ch <- msg

Step 1: 加鎖 (lock.Lock())
        ─── 所有其他 Producer 此刻全部被擋在這裡 ───

Step 2: 檢查是否有正在等待的 Consumer (recvq 是否非空)？
        ├─ 是 → 直接把 msg 複製給等待中的 Consumer Goroutine
        │        並呼叫 goready() 喚醒它 → 解鎖 → 完成
        └─ 否 → 進入 Step 3

Step 3: 檢查 RingBuffer 是否還有空位 (qcount < dataqsiz)？
        ├─ 有空位 → 把 msg 複製進 buf[sendx] 的記憶體位置
        │            sendx++ (移動寫入指標)
        │            qcount++ → 解鎖 → 完成
        └─ 沒空位 → 進入 Step 4 (最慘的情況)

Step 4: RingBuffer 已滿！
        打包成 sudog 結構體丟進 sendq 等待佇列
        呼叫 gopark() → 讓出 CPU，Goroutine 進入睡眠
        ─── 解鎖 (讓其他人繼續) ───
        ─── 等到有人 Recv 後，才被 goready() 喚醒 ───
```

### 一次 `msg := <-ch` 的完整步驟拆解

```
Goroutine B 執行 msg := <-ch

Step 1: 加鎖 (lock.Lock())

Step 2: 檢查是否有正在等待的 Producer (sendq 是否非空)？
        ├─ 是 → 直接從等待的 Producer 拿 msg (繞過 RingBuffer)
        │        呼叫 goready() 喚醒 Producer → 解鎖 → 完成
        └─ 否 → 進入 Step 3

Step 3: 檢查 RingBuffer 是否有資料 (qcount > 0)？
        ├─ 有資料 → 從 buf[recvx] 複製出 msg
        │            recvx++ (移動讀取指標)
        │            qcount-- → 解鎖 → 完成
        └─ 沒資料 → 打包成 sudog 丟進 recvq，gopark() 睡眠

```

---

## 3. 第三樂章：兩者的物理代價對比

| 操作 | Proto.Actor MPSC | Go Channel (有 buffer) |
|------|-----------------|----------------------|
| Producer 投遞 | 1 次 CAS 自旋 | 1 次 Mutex 加鎖 + 記憶體複製 + 解鎖 |
| Consumer 取出 | 0 個原子操作，純指標移動 | 1 次 Mutex 加鎖 + 記憶體複製 + 解鎖 |
| 佇列塞滿時 | 不存在（Linked List 動態擴展） | `gopark()` 把 Goroutine 打入睡眠 |
| 同時 100 個 Producer | 100 個 CAS 輪流自旋 tail，互不阻塞 | 100 個競爭搶同一把 Mutex，99 個在等 |
| 記憶體佈局 | 動態 Linked List，無上限 | 固定 RingBuffer，容量在 make() 時決定 |

**核心差異的本質**：
- Channel 的 Mutex 是**「悲觀鎖」**：假設一定有衝突，所以每次都先把大門鎖死再說。
- MPSC 的 CAS 是**「樂觀鎖」**：假設通常沒有衝突，如果真的撞上了就快速自旋重試，從不讓 Goroutine 睡死。

---

## 4. 第四樂章：那 Go Channel 什麼時候用？

這不代表 Go Channel 一無是處。它的優勢是：
- **零依賴、語言原生、易讀易維護**：99% 的業務場景 buffered channel 完全夠用。
- **Select 多路複用**：Go Channel 的 `select` 語法天生支援同時監聽多個訊息源，這是 Proto.Actor 做不到的。
- **背壓 (Backpressure) 天然成型 (但有隱患)**：Channel 塞滿時的 `gopark` 阻塞 Producer，能有效防止下游處理器 (Consumer) 被海量任務打爆。
  * **⚠️ 進階架構師的盲點警告**：雖然這保護了下游，但如果你的最上游（例如 HTTP 伺服器）沒有限制最大連線數，外部流量持續湧入，`net/http` 會不斷生成新的 Goroutine，然後這些 Goroutine 全部卡在 `ch <- msg` 進入 `gopark` 睡眠。這會導致 **Goroutine 數量無限暴增，最終把整台機器的記憶體吃光 (OOM 死亡)**！
  * **正解**：要靠 Channel 達成完美的背壓，最上游一定要搭配連線數限制 (Rate Limiter 或 bounded concurrency)，才能「把壓力真正擋在系統大門外」，而不是讓壓力在系統內部堆積變成喪屍 Goroutine。

**決策準則**：
- 一般業務邏輯、單機服務、QPS < 10 萬 → **Go Channel，簡單可靠**
- 遊戲伺服器全局狀態、百萬 Actor 同時運行、QPS > 50 萬 → **Proto.Actor MPSC，直接消滅 Mutex 瓶頸**

---

## 5. 第五樂章：實戰架構設計：包網高併發錢包系統

現在把前面所有的知識點組裝起來，設計一個能應付包網 (Online Betting) 場景的超高併發錢包系統。
這個場景的極端挑戰在於：同一個玩家可能在百毫秒內接收下注成功、賠付、退款等多筆並發操作，任何一筆帳目都不能出錯、不能重複、不能遺漏。

### 核心架構設計圖

```
                     HTTP/gRPC API
                          │
            ┌─────────────▼──────────────┐
            │     ActorSystem (Go)        │
            │                            │
            │  WalletActor(user_001)      │  ← 每個玩家一個 Actor
            │  WalletActor(user_002)      │  ← 狀態只活在記憶體
            │  WalletActor(user_003)      │  ← MPSC 無鎖信箱保護
            │  ...                       │
            └──┬──────────────────┬──────┘
               │                  │
    ① 每筆操作    │                  │  ③ 每 100 筆快照一次
    以 Event 形式 │                  │
    順序寫入      ▼                  ▼
   ┌────────────────┐      ┌────────────────┐
   │  Cassandra     │      │    MySQL        │
   │  (Event Log)   │      │  (Snapshot)     │
   │                │      │                 │
   │  時序流水帳    │      │  餘額快照表     │
   │  append-only   │      │  wallet_snapshot│
   └────────────────┘      └────────────────┘
```

### 為什麼選 Cassandra 存 Event Log？

包網錢包的每一筆操作（下注、賠付、退款）都是一個**不可變的事件 (Immutable Event)**，這種資料模式叫 **Event Sourcing**。它的特性：
- **只寫入不修改 (Append-Only)**：每次加錢扣錢，不是去 UPDATE 餘額欄位，而是插入一行新紀錄。
- **時序排列**：用 `user_id + timestamp` 作為 Partition Key，可以按時間順序讀出一個玩家的完整操作歷史。
- **寫入量極大**：百萬用戶同時下注 → 瞬間塞入百萬筆

Cassandra 完美命中這三個需求：
- **LSM-Tree 儲存引擎**：Write 不需要隨機尋址，所有寫入都是 Append-only 到記憶體 MemTable，再統一排序合併落盤 (SSTable)，寫入速度吊打 MySQL B-Tree。
- **天生分散式**：Event Log 資料會自動按 Partition Key 分散到多個節點，無需手動 Sharding。
- **不適合複雜查詢**：這正好，Event Log 只需要按 user_id 取出所有事件，根本不需要 JOIN。

### Actor 的核心運作：加錢扣錢操作

⚠️ **先踩一個現實的煞車**：上面那個「每筆操作立刻寫入 Cassandra」的設計，在真實百萬玩家同時下注的場景下**是行不通的**。
即使 Cassandra 的單節點理論上可以達到 5 萬~10 萬寫入/秒，但每一筆 Event 都發起一次網路 Round Trip，100 萬玩家同時下注就是 100 萬次網路請求同時砸向 Cassandra cluster，網路 I/O 直接打滿，延遲直線飆升。

**正確解法：Actor 內建 Write Buffer，批次 Flush**

Actor 的職責是「保護記憶體狀態的一致性」，而不是「每次操作都打一次資料庫」。真實架構應改為：Actor 在記憶體裡先累積事件，達到一定數量或時間後，才一次批次寫入 Cassandra。

```go
type WalletActor struct {
    userID      string
    balance     int64
    eventSeq    int64
    writeBuffer []WalletEvent  // 記憶體 Buffer，累積尚未落盤的 Events
    flushTimer  *time.Timer    // 50ms 定時強制 Flush，避免 Buffer 永遠不滿
}

const batchFlushSize = 100  // 累積 100 筆就 Flush
const batchFlushMs   = 50   // 或每 50ms 強制 Flush

func (a *WalletActor) Receive(ctx actor.Context) {
    switch msg := ctx.Message().(type) {

    case *DebitMsg:
        if a.balance < msg.Amount {
            msg.RespChan <- errors.New("餘額不足")
            return
        }
        // ① 更新記憶體狀態（這步極速，不碰任何 I/O）
        a.balance -= msg.Amount
        a.eventSeq++
        // ② 把 Event 放進記憶體 Buffer，還沒寫 Cassandra！
        a.writeBuffer = append(a.writeBuffer, WalletEvent{
            Type: "DEBIT", Amount: msg.Amount, Seq: a.eventSeq,
        })
        msg.RespChan <- nil  // 立刻回傳成功（此時 Event 只在記憶體）

        // ③ 達到 batchSize 就立刻 Flush
        if len(a.writeBuffer) >= batchFlushSize {
            a.flushToCassandra()
        }

    case flushTick:  // 定時觸發（每 50ms 一次）
        if len(a.writeBuffer) > 0 {
            a.flushToCassandra()  // 不管幾筆，時間到就強制寫入
        }
    }
}

func (a *WalletActor) flushToCassandra() {
    batch := a.writeBuffer
    a.writeBuffer = a.writeBuffer[:0]  // 清空 Buffer

    // 背景批次寫入，不阻塞 Actor 繼續處理下一批訊息
    go func() {
        cassandra.BatchInsert(batch)  // 100 筆合成一個 Cassandra BATCH 請求
        // 每達到快照條件就更新 MySQL
        if a.eventSeq % 100 == 0 {
            takeSnapshot(a.userID, a.balance, a.eventSeq)
        }
    }()
}
```

**這個設計的代價與取捨 (Durability Tradeoff)**：
- ✅ **吞吐量**：100 萬玩家的操作先在 Actor 記憶體中消化，Cassandra 收到的是「批次請求」的數量（大約 1 萬個 Batch），而不是 100 萬個獨立請求，壓力降低 **100 倍**。
- ⚠️ **風險**：如果 Actor 所在的機器在 Flush 前突然崩潰，最多丟失 `batchFlushMs (50ms)` 內的事件。這個風險需要與業務方協商接受，包網系統如果無法接受，就必須用 WAL 先寫本機磁碟做雙重保護。

### 快照策略：MySQL 的精準定位

每 100 筆操作後，把「當前餘額」與「已處理的最後一個事件序號 (eventSeq)」存入 MySQL：

```sql
-- MySQL 快照表
CREATE TABLE wallet_snapshot (
    user_id    VARCHAR(64) PRIMARY KEY,
    balance    BIGINT      NOT NULL,
    event_seq  BIGINT      NOT NULL,   -- 這個快照對應到第幾筆 Event
    updated_at TIMESTAMP   DEFAULT CURRENT_TIMESTAMP
);
```

快照的意義：把「回放事件」的起點從頭搬到最近的存檔點，讓 Actor 復活的速度從「回放 10 萬筆」變成「從快照 + 回放最後 99 筆以內」。

### Actor 的 Passivation（休眠下線）與 Rehydration（重新復活）

包網平台有大量玩家，不可能把百萬個 Actor 全部常駐在記憶體裡。Proto.Actor 支援 **Passivation (被動化)** 機制：

```
玩家 user_001 的 WalletActor 超過 30 分鐘沒有收到任何訊息
          │
          ▼
ActorSystem 觸發 Passivation
          │
          ├─ 立刻把 actor 從記憶體移除（釋放 RAM）
          └─ 此時 MySQL 快照是遺留的存檔，Cassandra 保有完整事件流
```

當玩家再次上線，系統收到第一筆訊息時，觸發 **Rehydration (重新水化)** 流程：

```
收到 user_001 的訊息，但 Actor 已不在記憶體
          │
          ▼
【Step 1】 從 MySQL 讀取最新快照
          → balance = 50,000, event_seq = 700

【Step 2】 從 Cassandra 讀取 event_seq > 700 的所有後續事件
          → [701: DEBIT -200], [702: CREDIT +500], [703: DEBIT -100]

【Step 3】 在記憶體中重播 (Replay) 這些事件
          → balance: 50,000 - 200 + 500 - 100 = 50,200

【Step 4】 WalletActor 以最新狀態重新上線，開始接客！
```

這個設計的優雅之處在於：**MySQL 提供了「O(1) 的快速跳點」，Cassandra 提供了「從跳點到現在的精確事件流」**，兩者合力讓 Actor 的復活成本從 O(N) 的全量回放，降低到 O(N/100) 的局部回放。

### 整體資料流總結

```
API: user_001 下注 200 元
  │
  ▼
ActorSystem 路由到 WalletActor(user_001) 的 MPSC 信箱
  │ (無鎖投遞，Actor 依序處理，無 Race Condition)
  ▼
Actor 檢查餘額是否足夠
  │
  ├─ 不足 → 立刻回傳錯誤，Cassandra 不寫入任何東西
  └─ 足夠 → ①更新記憶體 balance → ②立刻回傳成功給 API
               → ③把 Event 放進 writeBuffer（記憶體）
               → ④ 若 buffer 滿 100 筆 or 50ms 到期 → 批次 Flush → Cassandra
               → ⑤ 若 seq % 100 == 0，背景快照 → MySQL

玩家 30 分鐘沒上線 → Passivation，Actor 從 RAM 消失
玩家再次請求 → Rehydration，MySQL 快照 + Cassandra 差量回放 → Actor 重生
```

---

## 6. 第六樂章：終極殺招：Cluster Sharding 與動態路由 (Location Transparency)

在微服務的架構裡，最簡單的分流方式是「靜態路由 (Static Sharding)」，例如在 Nginx 寫死 `uid % 1024`，讓 1024 個 Pod 固定負責綁定的玩家。
但這種靜態設計有三個微服務的致命傷：
1. **單點故障中斷**：如果 Pod 5 死掉，K8s 重新拉起 Pod 需要 30~60 秒。這期間 `uid % 1024 == 5` 的玩家全部陷入系統中斷，這對高頻交易是災難。
2. **擴容雪崩 (Rehashing Storm)**：過年想擴容到 2000 個 Pod，除數變成 `% 2000`，99% 玩家的 Hash 值全部改變。所有 Actor 必須在新的 Pod 重新去 DB 撈快照重播，瞬間產生毀滅級的資料庫讀取風爆。
3. **熱點無法遷移**：兩個瘋狂下注的超大戶剛好 Hash 到同一個 Pod，把那台機器的 CPU 打滿，系統卻無法動態把他們拆開。

### 動態路由：Virtual Shards (虛擬分片)

為了解決這個問題，Proto.Actor 與 Akka 採用了 **Cluster Sharding** 機制，做到真正的「位置透明 (Location Transparency)」：

1. **維護活著名冊**：所有 Node (Pod) 啟動時向 etcd/Consul 報到，維護一份活著的清單 `[Pod_A, Pod_B, Pod_C]`。
2. **虛擬分片 (Virtual Shard)**：框架內部劃分出固定幾萬個 Shard (例如 10,000 個)。玩家 `uid=001` 固定映射到 `Shard_45`。
3. **一致性雜湊分配**：這 10,000 個 Shard 會被「均勻且動態」地分配給目前活著的 Pod_A, B, C。
4. **無腦轉發 (基本設定)**：API Gateway 收到 `uid=001` 的請求，隨便丟給任何一台中繼 Pod。這台中繼 Pod 掐指一算「Shard_45 歸 Pod_A 管」，再透過 gRPC **轉發 (Forward)** 給 Pod_A。優點是開發者完全不需要知道 Actor 在哪。
5. **聰明網關優化 (Smart Gateway / Zero-Hop)**：為了解決中繼轉發多一次 RTT 延遲的問題，可以讓最外層的 API Gateway 直接訂閱 etcd 的「名片簿 (包含所有 Pod 的實體 IP 與 Shard 分配)」。當 `uid=001` 抵達時，Gateway 自己算出版圖，直接拿 gRPC 對準 `Pod_A` 的實體 IP (例如 10.244.1.5:8080) 發射！**達成零代理、零轉發延遲的極限效能！**

### 神仙級的容災劇本 (Failover)

藉由「Cluster Sharding」加上「Kafka/Cassandra 重播過濾」，Actor 框架展現了分散式系統最極致的容災能力：

1. **災難發生**：`Pod_A` 被拔插頭瞬間死機。
2. **瞬間被發現**：2 秒內，etcd 發現 `Pod_A` 心跳遺失。更新名冊變為 `[Pod_B, Pod_C]`。
3. **動態接管**：框架立刻把原本屬於 Pod_A 的 `Shard_45`，重新分配給 `Pod_C` 接管。（只有死掉的那個 Node 的 Shard 被轉移，原本 B 和 C 的 Shard 完全不動，沒有擴容雪崩！）
4. **無縫復活**：玩家覺得卡卡的按下了 Retry，請求被打進叢集，一路被導航到新主人 `Pod_C`。
5. **Rehydration (借錄影帶重播)**：`Pod_C` 發現自己身上沒有這個 Actor，立刻啟動復活程序。
   * **跳點**：先去 MySQL 讀取快照（得知進度停在第 800 筆，餘額 5000）。
   * **借帶子**：去 Cassandra 查詢 `WHERE uid=001 AND seq > 800`，只拿回尚未結算的歷史事件（例如第 801~805 筆）。
   * **播放重算**：在記憶體中將這 5 筆帳「重播 (Replay)」，一步步推演出最新的餘額。
6. **濾水器 (過濾髒資料)**：在重播的同時也是在洗資料！如果在先前的「腦裂(Split-Brain)」或「網路重試」期間，導致 Cassandra 被重複寫入了兩筆一樣的 `txid_999`。Actor 肚子裡的 `txid map` 會在重播第一筆時記住它，播到第二筆時直接判定為重複並**丟棄**。
7. **滿血接客**：錄影帶播完（通常不到 1 毫秒），Actor 帶著 100% 正確純淨的餘額狀態在 `Pod_C` 滿血復活，成功處理剛剛卡住的請求並回傳給玩家。

**架構師結語**：真實的高吞吐架構永遠是「記憶體速度回應 + 批次降壓落盤」的組合。Actor 負責無鎖保護記憶體一致性，Write Buffer 負責把 N 倍的 DB 壓力合併攤平，MySQL Snapshot 負責縮短 Rehydration 的回放成本，Cassandra 負責作為不可竄改的事件真相來源，而 **Cluster Sharding 負責讓這一切在節點死傷慘重的物理世界裡，依然能如絲般滑順地 99.99% 高可用運作**。每一層都有其不可替代的職責，缺一不可。
