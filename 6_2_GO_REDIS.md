# Book 6.2: The Redis Driver (緩存驅動 - 急速快遞)

## 1. 第一樂章：隱喻 (The Metaphor - The Express Courier)

如果說 MySQL 是處理複雜外交事務的 **「領事館」**，那 Redis 就是一個 **「急速快遞中心」**。
這裡不講究繁文縟節 (SQL parse, Optimizer)，只講究一件事：**快進快出**。

Driver (如 `go-redis`) 在這裡的角色就是 **快遞員**。他不需要像外交特使那樣穿西裝打領帶，他只需要把包裹 (Command) 丟上卡車 (Socket)，然後立刻回來。

---

## 2. 第二樂章：RESP 協定 (The Language of Speed)

Redis 為什麼快？除了它是 In-Memory 之外，它的通訊協定 **RESP (REdis Serialization Protocol)** 功不可沒。
這是一個 **「人類可讀」** 但又 **「極易解析」** 的二進位安全協定。

### 結構解析
RESP 用第一個 byte 來區分型別：
*   `+`: 簡單字串 (Simple String) -> `+OK\r\n`
*   `-`: 錯誤 (Error) -> `-Error message\r\n`
*   `:`: 整數 (Integer) -> `:1000\r\n`
*   `$`: 二進位字串 (Bulk String) -> `$6\r\nfoobar\r\n` (6是長度)
*   `*`: 陣列 (Array) -> `*2\r\n` (長度為2的陣列)

### 實戰翻譯
當你在 Go 執行 `rdb.Set(ctx, "key", "val", 0)` 時，Driver 其實是送出了：

```text
*3\r\n      (這是一個 Array，有 3 個元素)
$3\r\nSET\r\n   (第 1 個: "SET")
$3\r\nkey\r\n   (第 2 個: "key")
$3\r\nval\r\n   (第 3 個: "val")
```

這比 SQL 的 Parser 簡單太多了。Go 的 `bufio.Reader` 可以用極高的效率讀取這些 `\r\n` 分隔的 bytes。

> **深度觀念修正 (Parsing vs RTT)**:
> *   **RESP** 解決的是 **CPU 瓶頸** (解析極快)。
> *   **Pipeline** 解決的是 **網路瓶頸** (減少物理來回 RTT)。
> Redis 團隊雙管齊下，才造就了它的高效能。

---

## 3. 第三樂章：Pipeline (消滅 RTT)

Redis 的瓶頸通常不在 CPU，而在 **網路延遲 (RTT)**。
發一個指令要 0.1ms，但網路來回可能要 1ms。如果我有 100 個指令要發：
*   **一般模式 (Ping-Pong)**: 100 * (0.1ms + 1ms) = **110ms**。
*   **Pipeline 模式 (Mailbag)**: 把 100 封信裝進一個大袋子，一次丟給 Server。
    *   Cost: (100 * 0.1ms) + 1ms = **11ms**。
    *   **快了 10 倍！**

### Go 的實作原理
在 `go-redis` 中，Pipeline 並不是什麼黑魔法，它只是利用了 **Buffering**。

```go
pipe := rdb.Pipeline()
pipe.Set(ctx, "k1", "v1", 0) // 寫入 buffer ([]byte)
pipe.Set(ctx, "k2", "v2", 0) // 寫入 buffer
_, err := pipe.Exec(ctx)     // 一次性 conn.Write(buffer)
```

### 什麼時候該用？ (Dos and Don'ts)

| 場景 | 判斷 | 原因 |
| :--- | :--- | :--- |
| **系統預熱 (Warm up)** | ✅ | 一次塞 1 萬筆資料，無順序性，最適合 Pipeline。 |
| **日誌/統計 (Logging)** | ✅ | 每秒累積 100 筆 Counters 一次發送，這能極大降低 IOPS。 |
| **邏輯依賴 (Dependency)** | ❌ | 例如「先 Get 餘額，夠了再 Decr」。Pipeline 是一次發送，你看不到中間結果。這必須用 Lua。 |
| **原子性 (Transactional)** | ❌ | Pipeline 只是批次傳輸，不保證原子性 (中間斷線可能只執行一半)。若需原子性，請用 Multi/Exec 或 Lua。 |

---

## 4. 第四樂章：Redis Cluster (智慧客戶端)

Redis Cluster 是分佈式的 (Sharding)。要準確找到數據，Client 必須內建導航系統。

### A. 定址算法 (HMS CRC16)
怎麼知道 `user:123` 屬於哪個 Slot？Redis 使用一個公開的數學公式，Client 不用問 Server 就能自己算：

> **Formula**: `Slot = CRC16(Key) % 16384`

因為 CRC16 是確定性的，所以只要 Key 不變，算出來的 Slot 永遠一樣。這就是為什麼不需要 Server 指揮也能分流。

### B. 路由地圖 (Slot -> Node Map)
算出 Slot (例如 3999) 後，去哪台機器找？
1.  **啟動時 (Startup)**: Client 會發送 `CLUSTER SLOTS` 指令，下載完整的「地圖」：
    *   Slots 0-5000: Node A (192.168.0.1)
    *   Slots 5001-10000: Node B (192.168.0.2)
    *   ...
2.  **執行時 (Runtime)**: Client 算出 Slot 3999 -> 查本地地圖 -> 發現是 Node A -> 直接連線。**這是 0 RTT 的完美路徑**。

### C. 異動處理 (Topology Change)
如果運維把 Slot 3999 搬到了 Node B (擴容或故障轉移)，會發生什麼事？
1.  **撞牆 (Fail)**: Client 拿著舊地圖連 Node A -> Node A 發現自己已經沒有 Slot 3999 了 -> 回傳 `-MOVED 3999 NodeB_IP`。
2.  **更新 (Refresh)**:
    *   Smart Client 攔截這個錯誤。
    *   **Lazily Update**: 既然錯了，代表地圖舊了。Client 會重新執行 `CLUSTER SLOTS` 更新整張地圖。
    *   (這是 Redis 為了效能取捨的結果：Server 不主動推播變更，而是讓 Client 撞牆後自己更新)。
### D. 叢集下的分散式鎖隱患 (The Distributed Lock Hazard)
在 Redis Cluster 使用 `SETNX` 做分散式鎖時，有一個經典的 **「鎖丟失 (Lost Lock)」** 風險：

1.  **Client A** 向 `Master` 搶鎖成功。
2.  **Crash**: `Master` 在將鎖複製給 `Slave` 之前突然斷電。 (Redis 預設是**非同步複製**)。
3.  **Failover**: `Slave` 晉升為新 `Master`，但它手裡**沒有這把鎖**。
4.  **Client B** 向新 `Master` 搶鎖，也成功了！
5.  **結果**: A 和 B 同時持有鎖，開始併發修改數據，導致資料損毀。

> **解法 1 (極致安全): Redlock**
> 向 5 台獨立 Redis 節點搶鎖，過半數同意才算成功。適合金融級場景，但實作複雜。
>
> **解法 2 (高 CP 值): WAIT 指令**
> 在 `SETNX` 後立刻執行 `WAIT 1 1000`。
> 這會強迫 Master 等待至少 1 台 Slave 確認收到資料後才回傳成功。
> 雖然寫入會慢一點 (多一個 RTT)，但能確保「鎖」已經備份了，大幅降低 Failover 導致鎖遺失的風險。
>
> *注意：如果您有多個 Slaves (例如 A, B)，`WAIT 1` 只能保證其中一台收到。如果 Failover 時剛好選中另一台沒收到的，鎖還是會丟。要 100% 就在這犧牲可用性改用 `WAIT <All_Slaves>`。*


---

## 5. 第五樂章：調優指南 (Tuning)

### 1. `PoolSize` (連線池大小)
*   **Redis vs SQL**: Redis 每個指令極快 (微秒級)，所以一個連線的復用率極高。通常 **不需要** 像 SQL 那麼多連線。
*   **建議**: `runtime.NumCPU() * 10` 是一個常見起點。
*   **注意**: 如果你要用大量的 **Blocking Data (BLPOP)** 或者 **Pub/Sub**，這些連線會被佔用，這時就需要開大 PoolSize。

### 2. `MinIdleConns` (保溫)
*   Redis 建立連線雖然比 MySQL 快，但 TCP handshake 依然有成本。
*   建議設定一定數量的 Idle 連線，避免流量突增時的 latency抖動。

### 3. `ReadTimeout` / `WriteTimeout` (致命的雙面刃)
*   **設定建議**: 針對不同業務使用不同 Config (快取短、存儲長)。
*   **殘酷的真相 (The Anatomy of a Timeout)**:
    當你的 Go 程式觸發 Timeout 時，背後發生了兩件可怕的事：
    1.  **連線必死 (Client Side Death)**:
        *   Driver 必須強制 **關閉這條 TCP 連線**。不能重複使用。
        *   **原因**: Socket 裡可能殘留了上一個指令的半截數據，重用會導致協議錯亂 (Protocol Desync)。
        *   **代價**: 下一次請求必須重新進行 TCP Handshake (延遲增加)。
    2.  **幽靈運算 (Server Side Waste)**:
        *   **OS 層**: Client 送出 `FIN`，Server OS 收到並標記 Socket 為 `CLOSE_WAIT`。
        *   **App 層 (Redis)**: 但因為 Redis 單執行緒正在忙，**根本沒去檢查 Socket**。它會堅持把慢指令跑完，才發現連線已斷。
        *   **結果**: Server CPU 被白白浪費，且無法中斷。

    3.  **二次傷害 (The Double Whammy)**:
        *   除了這次請求失敗 (Timeout)，你還賠上了這條連線。
        *   下一次請求如果不幸拿到這條已死的連線 (或發現 Pool 空了)，必須支付 **TCP Handshake (1 RTT)** 的代價。
        *   **後果**: 系統若發生大面積 Timeout，會引發 **連線風暴 (Re-connection Storm)**，讓延遲進一步惡化。
    
    > **死亡螺旋 (Death Spiral)**:
    > 如果你設了短 Timeout (1s) 加上 **重試機制 (Retry)**。
    > Client 每一秒斷開一次並重試 -> Redis Server 同時跑著 5 個一樣的慢指令 -> CPU 100% -> 更多 Timeout。
    > **這是自行發動的 DDoS 攻擊。**

    > **逃生指南 (Survival Guide)**:
    > 1.  **指數退避 (Exponential Backoff)**: 失敗後不要馬上重試。等 100ms -> 200ms -> 400ms。給 Server 喘息空間。
    > 2.  **熔斷 (Circuit Breaker)**: 當錯誤率超過閾值 (如 50%)，直接停止發送請求 30秒。防止連線風暴擴大。
    > 3.  **誠實面對**: 如果你的 Query 就是要跑 500ms，請把 Timeout 設為 1s。試圖用短 Timeout 來「加速」系統是自欺欺人。

### 4. Pipeline Batch Size (批次大小)
*   **陷阱**: Pipeline 是將所有指令與結果暫存在記憶體 (`Write Buffer` & `Read Buffer`)。
    *   如果你一次 Pipeline 100 萬筆資料，Client 端記憶體會瞬間飆高 (OOM 風險)。
*   **建議**: 使用 **Chunking (分批)** 策略。
    *   例如每累積 1000 筆指令，就呼叫一次 `pipe.Exec()`。這樣既能享受 Pipeline 的加速，又能控制記憶體用量。

#### 5. 參數決策邏輯 (Decision Logic)

Redis 的參數設定取決於你的 **使用場景 (Use Case)**：

1.  **PoolSize 怎麼算？**
    *   **公式**: `QPS * 平均延遲`。
    *   由於 Redis 極快 (通常 0.5ms~1ms)，單一連線的吞吐量極高。
    *   **案例**: 10,000 QPS * 0.001s = **10 個連線**。
    *   **結論**: 用於單純快取時，通常不需要幾百個連線。預設的 `10 * CPU` 通常綽綽有餘。
    *   **例外 (Blocking Cmds)**:
        *   **原理**: `BLPOP` 指令會**長時間佔用連線** (直到有資料進來才釋放)。這跟 `GET/SET` 用完即倒是完全不同的。
        *   **死鎖場景**: 假設你有 20 個 Worker 在做 `BLPOP`，但 Pool 只有 10 個連線。
            *   前 10 個 Worker 搶走所有連線並開始「佔線等待」。
            *   Pool 變空了。
            *   系統想做一個簡單的 `SET`，卻因為沒連線而卡死。
        *   **黃金法則**: `PoolSize` 必須大於 `Blocking Worker 數量 + 預留給普通指令的數量`。

        > **深度類比 (Deep Analogy)**:
        > 您可以把 `BLPOP` 想像成 **Go Channel** 的跨機器版本。
        > *   **Go Channel**: `val := <-ch` (如果沒資料，Goroutine 休眠)。
        > *   **Redis BLPOP**: `val := rdb.BLPop()` (如果沒資料，Goroutine 也休眠)。
        > *   **差別**: Channel 用的是 Go 內存；Redis 用的是 TCP 連線與 Server 內存。但**行為模式 (Blocking)** 是一模一樣的。

2.  **Timeout 怎麼設？**
    *   **作為 Cache (緩存)**: 建議 **Fail Fast (100ms - 200ms)**。
        *   如果 Redis 慢了，直接 Timeout 當作 Cache Miss 去查 DB。千萬不要因為快取慢而導致整個 API 卡死 (Cascading Failure)。
    *   **作為 Storage (持久存儲)**: 建議設長一點 (1s - 2s)，確保資料能寫入。

---

## 6. 第六樂章：Cluster 的限制 (The Cluster Constraints)

當你從單機切換到 **Redis Cluster** 時，最容易踩到的坑就是 **跨分片操作 (Cross-Slot)**。

### 1. Pipeline 的破碎
如果你的一個 Pipeline 裡面包含了 `Key A` (Node 1) 和 `Key B` (Node 2)：
*   **Driver 的行為**: 它很聰明，會自動把你的 Pipeline **拆成兩半**。
*   **效能影響**: 分別發送給 Node 1 和 Node 2，然後再等待兩邊都回來。這比單次發送稍微慢一點，且消耗更多網路資源。

### 2. Lua Script 的禁忌 (CROSSSLOT Error)
這是硬傷。Redis 要求單個 Lua Script 裡面的所有 Key **必須落在同一個 Slot**。
*   如果你試圖原子更新 `user:A` 和 `user:B` (假設他們分屬不同 Slot)。
> **深度解析：為什麼 Redis 這麼小氣？ (Why Forbidden?)**
> 您可能會想：「為什麼 Redis 不在後台幫我把請求轉發給 Node B 就好？」
> *   **併行架構 (Parallelism)**: Redis Cluster 的每個節點都是獨立運行的 (Shared Nothing)。Node A 和 Node B 是 **並行 (Parallel)** 工作的。
> *   **原子性代價**: 如果 Lua 允許跨機器操作，這就變成了 **分佈式事務 (Distributed Transaction)**。Redis 必須實作複雜的 **兩階段提交 (2PC)** 來保證 Node A 和 Node B 同時成功或失敗。
> *   **結論**: 2PC 會讓效能從 10萬 QPS 掉到 1千 QPS。Redis 選擇了棄用分佈式事務，來換取極致的擴展性。

### 3. 救世主：Hash Tag `{}`
為了解決上述問題，Redis 支援 **Hash Tag** 語法。
它只會根據 `{}` 裡面的內容來計算 Hash Slot。

*   **用法**:
    *   Key 1: `{game:100}:player:A`
    *   Key 2: `{game:100}:player:B`
*   **效果**:
    *   因為 `{}` 內都是 `game:100`，這兩個 Key 保證會落在 **同一個 Slot** (同一台機器)。
    *   **Lua Script** 可以正常運作。
    *   **Pipeline** 可以一次打包發送，效能最大化。

#### 4. Hash Tag 最佳實踐 (Best Practices)
Hash Tag 的選擇是一門權衡的藝術：

1.  **使用者中心 (User Centric)**: `{user:123}`
    *   **場景**: 電商、社群。
    *   **優點**: 單一用戶的所有資料 (購物車、餘額) 都在一起，操作方便。
    *   **缺點**: 無法原子地轉帳給另一個用戶 (跨 Slot)。

2.  **賽局中心 (Match/Room Centric)**: `{room:99}`
    *   **場景**: 多人對戰遊戲。
    *   **優點**: 確保該局遊戲的所有狀態同步。
    *   **風險**: **熱點 (Hot Spot)**。如果該房間爆滿 (萬人觀戰)，該 Slot 所在的 Node 會超載。

3.  **絕對禁忌**:
    *   不要使用 `{global}` 或 `{server:1}` 這種超大範圍的 Tag。這會導致 **Data Skew (資料傾斜)**，讓 Cluster 退化成單機。

#### 5. 實戰演練：Go + Lua + Hash Tag
場景：**原子轉帳**。在局號 `1001` 中，玩家 A 輸給 玩家 B 50 元。

```go
// 1. 定義 Lua Script (檢查餘額 -> 扣款 -> 加款)
const transferScript = `
    local balance = redis.call("GET", KEYS[1])
    if not balance or tonumber(balance) < tonumber(ARGV[1]) then
        return 0 -- 餘額不足
    end
    redis.call("DECRBY", KEYS[1], ARGV[1])
    redis.call("INCRBY", KEYS[2], ARGV[1])
    return 1 -- 成功
`

func Transfer(ctx context.Context, matchID string, fromUser, toUser string, amount int) error {
    // 2. 建構帶有 Hash Tag 的 Keys
    // 注意：兩個 Key 都必須包含 {match:1001}
    keyFrom := fmt.Sprintf("{match:%s}:user:%s", matchID, fromUser)
    keyTo   := fmt.Sprintf("{match:%s}:user:%s", matchID, toUser)

    // 3. 執行 Lua (原子操作)
    // 因為 Hash Tag 的存在，Redis Cluster 允許這個跨 Key 操作
    res, err := rdb.Eval(ctx, transferScript, []string{keyFrom, keyTo}, amount).Result()
    
    if err != nil {
        return err // 如果沒加 Tag，這裡會報 CROSSSLOT 錯誤
    }
    if res.(int64) == 0 {
        return fmt.Errorf("餘額不足")
    }
    return nil
}
```
**關鍵**: `keyFrom` 和 `keyTo` 雖然不同，但它們都有相同的 `{match:1001}`，所以一定在同一台機器上。

---

## 7. 第七樂章：空間魔法 (Space Magic)

除了快，Redis 還能極致地省記憶體。這兩個資料結構被稱為「大數據統計神器」。

### 1. Bitmap (位元圖) - DAU 統計
**場景**: 有 1 億個用戶，要統計「日活躍用戶 (DAU)」。
*   **笨方法 (Set)**: 存 1 億個 UserID (int64)。
    *   消耗: 1億 * 8 bytes ≈ **800 MB**。
*   **神操作 (Bitmap)**: 用 1 個 bit 代表一個用戶的登入狀態 (UserID 作為 Offset)。
    *   指令: `SETBIT login:2023-12-30 10086 1`
    *   消耗: 1億 * 1 bit ≈ **12.5 MB**。 (節省 **64倍** 空間)
*   **進階 (Retention)**: 算「連續兩天登入」的人數？
    *   `BITOP AND retention_result login:2023-12-30 login:2023-12-31`
    *   這是一個 Bitwise AND 操作，速度極快，直接算出交集。

### 2. HyperLogLog (HLL) - 巨量計數 (UV)
**場景**: 要統計網站的 Unique Visitors，可能有數十億，且允許 **0.81% 誤差**。
*   **神操作**: `PFADD page:view <ip>`
*   **威力**: 不管你塞進去多少不同的元素 (1萬或10億)，HLL **永遠只佔用 12 KB** 的記憶體。
*   **機制**: 這是利用白努利試驗 (Bernoulli trial) 的機率原理估算出來的基數 (Cardinality)。對於大數據分析 (Big Data) 來說，極小的空間換取極快的速度是非常划算的。

---

## 8. 第八樂章：不只是緩存 (Persistence)

"Redis 重啟後資料還在嗎？" —— 這取決於你的 **持久化 (Persistence)** 設定。這也是 Redis 能勝過 Memcached 的核心原因。

### 1. RDB (快照 Snapshot)
*   **機制**: 每隔一段時間 (如 5 分鐘)，Redis 主進程會 `fork()` 一個子進程，利用 OS 的 **Copy-On-Write** 技術，把當下所有記憶體資料 Dump 成一個 `.rdb` 檔案。
*   **優點**: 檔案小，Redis 重啟時讀取極快。適合做異地備份。
*   **缺點**: 如果 Redis 突然掛掉，你會 **遺失最後 5 分鐘** 的資料。

### 2. AOF (追加日誌 Append Only File)
*   **機制**: 每執行一個寫入命令 (SET/INCR)，就把它 Append 到硬碟的 `.aof` 檔案末尾。
*   **同步策略 (`appendfsync`)**:
    *   `always`: 每個指令都寫硬碟。極慢，但絕對安全。
    *   `everysec` (**推薦**): 每秒寫一次。效能與安全的平衡點。最多掉 1 秒資料。
    *   `no`: 交給 OS 決定。
*   **優點**: 資料安全性高。
*   **缺點**: 檔案會越來越大 (需要定期執行 AOF Rewrite 來瘦身)。

### 建議配置
*   **純緩存 (Cache)**: 如果資料丟了去 DB 查就好，可以 **關閉所有持久化**，追求極致效能。
*   **關鍵資料 (Session/Cart)**: 建議開啟 **AOF (everysec)**。配合 Redis 4.0+ 的 **RDB-AOF 混合持久化**，既安全又重啟快。

---

## 9. 第九樂章：秒殺實戰 - 演唱會搶票 (Flash Sale)

### 1. 戰備清單 (Architecture Specs)
在寫任何代碼之前，我們先定义這場仗的規模。
**目標**: 扛住 **10萬 QPS** 的瞬間搶票流量 (假設總票數 10 萬張)。

*   **Frontend (Microservice)**:
    *   **規模**: **10 台 Pods** (每台 4-Core，扛 1萬 QPS)。
    *   **Redis Pool 設定**: `PoolSize: 100`, `MinIdle: 20`。 (每台 Pod 預留 100 條連線)。
*   **Backend (Redis Cluster)**:
    *   **規模**: **10 台 Master Nodes** (作為庫存分片)。
    *   **負載**: 每台承擔 1萬 QPS (約 10% CPU Loading，非常安全)。
    *   **總連線數**: 10 Pods * 100 Conns = 1,000 條 (Redis 毫無壓力)。

### 2. 延遲預估 (P99 Latency Chain) - 修正版
當用戶按下「搶票」按鈕，預期多久會知道結果？我們模擬 **瞬間爆發 (Burst)** 場景：

*   **HTTP Server (CPU Queue)**: **~500ms**。
    *   **修正觀念**: 1萬個請求同時進來，Go 需要解析 1萬次 JSON、做 1萬次 Auth。
    *   **計算**: 假設單次 CPU 耗時 0.2ms。`10,000 * 0.2ms = 2,000ms`。
    *   **分攤**: 4 核心並行處理 -> `2,000 / 4` = **500ms**。
    *   (這就是為什麼高性能場景要避開 Reflection JSON 的原因)。
*   **Network RTT**: **~1ms** (恆定)。
    *   **疑問**: 1萬人同時送出，P99 會不會塞車變慢？
    *   **解答**: **不會累加**。這跟 Redis 排隊不同。
    *   **數學證明 (Packet Size)**: 秒殺指令極短 (EVALSHA)。
        *   **大小**: 約 **120 Bytes** (含 TCP/IP Header)。
        *   **總量**: `10,000 * 120 Bytes` ≈ **1.2 MB/s**。
        *   **路寬**: 10Gbps 網卡 = **1250 MB/s**。
        *   **結論**: 佔用率不到 **0.1%**。這就是為什麼 RTT 完全不變。
    *   **結果**: 第 1 個請求和第 1 萬個請求，都是 **並行 (Parallel)** 在光纖裡跑，互不卡頓。P99 RTT 依然是 1ms。
*   **Redis 排隊 (Queueing)**: **~100ms**。
    *   **原理**: Redis 是單執行緒，依序處理請求。
    *   **計算**: `10,000 * 0.01ms (Lua)` = **100ms**。
    
    > **Micro-View: 0.01ms 到底做了什麼？ (Anatomy of Execution)**
    > 這 10微秒內，Redis 單執行緒完成了以下精密動作：
    > 1.  **Syscall**: 從 Socket 讀取數據 (`read`).
    > 2.  **Parsing**: 解析 RESP 協議，識別出 `EVALSHA`。
    > 3.  **Lua VM**: 載入並執行腳本。
    > 4.  **Memory Access**:
    >     *   `GET ticket:0`: 透過 Hash Table (O(1)) 找到整數物件。
    >     *   `CMP`: 比較庫存是否 > 0。
    >     *   `DECR`: 原地修改記憶體數值。
    > 5.  **Encoding**: 把結果 `:1` 寫入 Output Buffer。
    > *一切都在記憶體發生，沒有 Disk I/O，所以極快。*

*   **總等待時間 (P99)**: `500 (App) + 1 (Net) + 100 (Redis)` ≈ **601ms**。

> **效能逃生門：如何消滅這 500ms? (JSON vs gRPC)**
> 既然 JSON 是 CPU 殺手，解法就是 **不要用 JSON**。
>
> 1.  **Mobile App (最佳解)**: 讓 App 直接發送/接收 **Protobuf** (`application/x-protobuf`)。
>     *   Go 解析 Protobuf 是硬編碼 (Generated Code)，無反射，速度快 10 倍以上。
>     *   **結果**: HTTP Server 排隊時間從 500ms 驟降至 **50ms** 以內。
> 2.  **Web Browser (次佳解)**: 如果受限於瀏覽器必須用 JSON。
>     *   改用 **`bytedance/sonic`** 或 **`easyjson`** 等高效庫。
>     *   **結果**: 透過 JIT 或預生成代碼，速度提升 2~3 倍 (延遲降至 ~200ms)。

> **Deep Dive: OS 網路層為什麼無感？ (The SoftIRQ Journey)**
> 許多人擔心 1 萬個請求會塞爆網路，但從 OS 角度看，這只是「輕量級運動」。
>
> 1. **頻寬與 PPS (頻寬不是瓶頸)**
>    *   **流量**: 10k QPS ≈ 1 MB/s。現代網卡是 10Gbps (1250 MB/s)。佔用率 < 0.1%。
>    *   **PPS**: 20k PPS。現代 Kernel 配合 NAPI 可處理百萬級 PPS。
>
> 2. **封包的旅程 (DMA & SoftIRQ)**
>    *   **Step A (硬體)**: 網卡收到訊號，透過 **DMA (Direct Memory Access)** 直接把資料寫入 RAM (Ring Buffer)。**CPU 全程不參與，不消耗資源**。
>    *   **Step B (硬中斷 Hard IRQ)**: 網卡發出中斷通知 CPU：「貨到了」。
>    *   **Step C (軟中斷 SoftIRQ)**: CPU 暫停手邊工作，執行網卡驅動程式 (NAPI Poll)。
>        *   這時 CPU 才開始消耗算力：解析 TCP/IP Header、檢查 Checksum、剝離封包頭。
>        *   最後把「純資料」丟進 Socket Buffer。
>    *   **Step D (User Space)**: Redis 醒來，執行 `read()` 拿走資料。
>
> **結論**: 在 1萬 QPS 下，Step C (SoftIRQ) 的 CPU 消耗極低 (可能不到 1%)。除非流量再大 50 倍，OS 層面才會有感。

**結論**: 用戶感覺是 **「秒殺」** (0.1秒)，體驗極佳。

### 3. 先發制人：清場 (Reset / Pre-warm)
在 **選位模式 (SETNX)** 下，「預熱」的定義變成了 **確保座位是空的**。
活動開始前，Admin 必須從 MySQL 讀取有效的座位表，並清除 Redis 裡對應的鎖。

```go
func ResetStock(ctx context.Context) error {
    // 1. 從 MySQL 獲取所有合法的座位 ID (Source of Truth)
    // SQL: SELECT ticket_id FROM seats WHERE event_id = 101 AND status = 'open'
    allSeats, err := db.GetAllSeatIDs(ctx) 
    if err != nil {
        return err
    }

    pipe := rdb.Pipeline()
    // 建議設定一個計數器，每 1000 筆 Pipe.Exec() 一次
    batchSize := 1000
    count := 0

    for _, ticketID := range allSeats {
        // ticketID 例如 "A-1-1" (對應 MySQL 中的資料)
        key := fmt.Sprintf("seat:%s", ticketID)
        
        // 刪除 Key (釋放座位)
        pipe.Del(ctx, key)
        
        count++
        if count >= batchSize {
            if _, err := pipe.Exec(ctx); err != nil {
                return err
            }
            pipe = rdb.Pipeline() // 重置 Pipeline
            count = 0
        }
    }
    
    // 執行剩下的指令
    if count > 0 {
        _, err = pipe.Exec(ctx)
        return err
    }
    return nil
}
```

> **Sharding 去哪了？ (Where is the Sharding?)**
> 您可能發現這段代碼沒有寫 `shardID := id % 10`。
> 因為在 **Redis Cluster** 中，分片是全自動的：
> 1.  **Driver**: 當您操作 Key `seat:A-1-1` 時，`go-redis` 會自動計算 `CRC16` 雜湊值。
> 2.  **Routing**: Driver 根據雜湊值，自動將請求路由到對應的 Redis 節點。
> 3.  **亂序發送 (Scattering)**: 雖然我們的 Go 迴圈是連續的 (A-1, A-2...)，但因為 CRC16 的雪崩效應，這些 DEL 指令會被 **隨機打散** 到不同機器。Node 1 收到 A-1，Node 8 收到 A-2... 這保證了清場操作也是負載平衡的。

### 4. 核心戰術：分片 + 選位 (Sharding + Seat Selection)

#### A. Lua Script (原子鎖)
既然是用戶指定座位，我們使用 `SETNX` (Set if Not Exists) 來模擬「佔座」。
```lua
-- KEYS[1]: seat:{ticketID} (例如 seat:A-1-1)
-- ARGV[1]: userID
-- ARGV[2]: expire_seconds (例如 600秒)

-- 1. 嘗試佔座
-- 關鍵: 把 ARGV[1] (UserID) 寫入 Key 的 Value。
-- 成功後，Redis 狀態: seat:A-1-1 = "12345" (這就是你的票)
local acquired = redis.call("SETNX", KEYS[1], ARGV[1])
if acquired == 1 then
    -- 2. 搶到了！設定過期時間
    redis.call("EXPIRE", KEYS[1], ARGV[2])
    return 1
end

return 0 -- 被別人搶先了
```

#### B. Go 實作 (指定座位 + 異步寫入)
```go
func BuyTicket(ctx context.Context, userID int, ticketID string) error {
    // 1. 建構 Key
    key := fmt.Sprintf("seat:%s", ticketID)

    // 2. Redis 原子鎖 (搶票核心)
    // 設定 10 分鐘 (600秒) 的付款窗口，逾時未付則釋放
    res, err := rdb.Eval(ctx, buyScript, []string{key}, userID, 600).Result()
    if err != nil {
        return err // 系統錯誤
    }
    
    // 搶鎖失敗 (回傳 0)
    if res.(int64) != 1 {
        return fmt.Errorf("位置 %s 已被搶走", ticketID)
    }

    // 3. 搶到了！建立訂單 (Async Side-Effect)
    // 關鍵: 這裡解決了 "Redis 雖然有資料但很難撈" 的問題。
    // 我們由 Go 程式負責把「戰果」推送到 Message Queue (Kafka) 或 Global Redis List。
    // 後端 Worker 再慢慢消費這個 Queue，把訂單寫進 MySQL。
    
    // 注意: 這裡建議用 Sync 發送以確保不掉單，或者有完善的 ACK 機制
    err = sendToOrderQueue(ticketID, userID) 
    if err != nil {
        //極端情況：Redis 鎖到了但 Queue 寫入失敗
        // 策略: 讓 Redis Key 10分鐘後自動過期釋放 (用戶需重試)
        log.Printf("Failed to queue order: %v", err)
    }
    
    return nil // 搶座成功
}
```

> **分佈疑問：這樣真的會均勻嗎？**
> 1.  **雪崩效應 (Avalanche Effect)**: Redis 使用的 CRC16 算法特性是「輸入微小變動，輸出巨大跳躍」。
>     *   `seat:A-1-1` -> Slot 5302 (Node 2)
>     *   `seat:A-1-2` -> Slot 1209 (Node 8)
> 2.  **優勢 (防止熱區)**: 這反而比人工分配更好。
>     *   如果我們人工規定 "A區都在 Node 1"，一旦 A區熱門，Node 1 就掛了。
>     *   透過 CRC16，鄰近的座位被打散到不同機器。即便大家都搶 A區，流量也會被 10 台機器共同分擔。
> 3.  **統計**: 在 10萬個 Key 的樣本下，分佈會非常均勻。

> **架構代價 (The Cost of Choice)**:
> 這種「指定座位」模式雖然用戶體驗好，但有一個致命傷：**熱點無法分散**。
> 如果 10 萬人都想搶 `A1-Row1` (第一排)，這 10 萬個請求全部會打向 **同一台 Redis** (因為 Key 相同)。
> 這時 Redis Sharding 就失效了，這是業務需求帶來的物理限制。

> **Cluster 架構備註 (No Hash Tag Needed)**:
> 在這個「選位模式」下，我們每次只操作一個 Key (`seat:A-1`)。
> 因此，我們 **不需要** 使用 Hash Tag `{}`。
> 讓每個座位的 Key 自然地透過 CRC16 演算法，均勻散落在不同的 Redis 節點上，這才是最佳的負載均衡策略。

### 5. 架構陷阱：Sharding vs Hash Tag
最後提醒最容易犯的錯：**不要在需要分散流量的 Key 上加 Hash Tag**。

*   **目標：分散流量 (Load Balancing)**
    *   **正確 (我們採用的)**: `seat:A-1`, `seat:B-2`
        *   因為 Key 全名不同，Redis Cluster 會把它們打散到不同機器。
        *   **結果**: 10 台機器一起工作，吞吐量是 **10倍**。
    *   **錯誤 (千萬別做)**: `{seat}:A-1`, `{seat}:B-2`
        *   因為大括號 `{seat}` 相同，Redis 只會看這個標籤，導致所有座位都存在 **同一台機器**。
        *   **結果**: 只有 1 台機器在工作，吞吐量是 **1倍**，且該機器會瞬間過載。

*   **目標：聚合邏輯 (Atomic Transaction)**
    *   **正確**: 只有當你需要對「同一用戶」的多個 Key 做原子操作時才用。
    *   例如: `{user:1}:profile`, `{user:1}:order` (強迫綁在同一台，以便 Lua 跨 Key 操作)。

### 6. 終極防禦：本地快取 (Local Cache Strategy)

1.  **售完即擋 (Sold Out Shield) - BigCache (Zero GC)**
    *   **問題**: 原生 Map 或 `sync.Map` 在存儲數十萬個 Key 時，會導致 Go GC 掃描時間顯著增加 (GC Scan Overhead)，且高併發寫入時鎖競爭嚴重。
    *   **解法**: 使用 **BigCache** 或 **FreeCache**。
    *   **原理**: 這些庫利用 **Sharding (分片鎖)** 減少競爭，並將數據序列化存入巨大的 `[]byte` 陣列 (RingBuffer)，實現 **Zero-GC** (無指標，GC 不會掃描)。
    *   **效果**: 當 Go 發現 `seat:A-1` 搶票失敗，寫入 BigCache。後續請求查 Cache 發現存在，直接攔截。這能扛住百萬級 QPS 且不影響 GC。

2.  **熱點攔截 (Hot Seat Shield) - 針對單一熱門座位**
    *   **情境**: 10 萬人同時搶 "第一排中間" (`seat:A-1-50`)。這會造成單個 Redis Key 寫入過熱。
    *   **解法**: Go 程式內部可以使用 `Mutex` 或 `SingleFlight`。
    *   **邏輯**: 對於同一個座位的請求，Go 節點在 10ms 內只允許 **這 10 萬人中的 1 個人** 去 Redis 執行 `SETNX`。
    *   **結果**: 其他 99,999 人在 Go 層直接被告知「搶票中...失敗」，保護了 Redis 不被熱點擊穿。




