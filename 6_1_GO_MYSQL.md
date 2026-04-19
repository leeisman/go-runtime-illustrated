# Book 6.1: The MySQL Driver (資料庫驅動 - 連線池的藝術)

Go 程式看起來只是呼叫 `db.Query()`，實際上背後牽涉連線池、協議握手、查詢取消與資料庫端資源管理。
這一章要拆的是 `database/sql` 這個「領事館」如何管理昂貴的跨境連線。

---

## 1. 第一樂章：隱喻 (The Metaphor - The Consulate)

如果說 Go Runtime 是一座運作良好的 **「皇家圖書館」** (有館長、任務單、清潔隊與行政區)，那麼 **Database (MySQL/Postgres)** 就是另一個強大的 **「外國檔案館」**。

我們不能直接去操作外國的資源。我們需要一個 **「領事館 (Consulate)」** 或是 **「貿易代表處」**。
在 Go 裡面，這個領事館就是 **`database/sql`** 標準庫，而駐外大使就是 **Driver** (如 `go-sql-driver/mysql`)。

*   **User Code**: 國民。想要請求外國資源 (SELECT/INSERT)。
*   **sql.DB**: 領事館大廳。負責接待國民，管理「簽證」和「特使」。
*   **Real Connection**: 特使 (Diplomat)。真正搭飛機 (TCP) 去外國辦事的人。

---

## 2. 第二樂章：為什麼需要 Connection Pool？ (The Cost)

為什麼我們不能每個 Request 都創建一個新的連線 (`sql.Open` + `Close`)？
因為 **TCP 握手 (Handshake)** 非常昂貴。

**建立一個 MySQL 連線的成本**:
1.  **TCP 三路握手**: Client -> SYN -> Server -> SYN-ACK -> Client -> ACK. (1.5 RTT)
2.  **MySQL 認證**: 發送帳號密碼 -> Server 驗證 -> 回傳 OK. (1-2 RTT)
3.  **資源分配**: MySQL Server 端要為這個連線分配 Thread 和 Buffer。

這一套下來，可能要花掉 **10ms ~ 100ms**。
如果你的 Query 只要 0.5ms，那你 90% 的時間都浪費在「搭飛機 (建立連線)」上。

**解法**: **[Connection Pool (連線池)]**。
特使辦完事不准回國 (Close)，而是留在領事館休息室 (Idle Pool) 待命，隨時準備出下一個任務。

---

## 3. 第三樂章：池的底層實作 (Under the Hood of `sql.DB`)

Go 的 `database/sql` 替我們實作了一個通用的連線池。它的核心結構大約是這樣的：

```go
type DB struct {
    // 互斥鎖，保護下面的狀態
    mu sync.Mutex 

    // 閒置的連線 (Idle Connections) - 休息室
    // 這是一個 Stack (LIFO) 或 Queue，視版本而定
    freeConn []*driverConn 

    // 正在請求的連線 (Connection Request) - 排隊區
    // 當沒有閒置連線且已達 MaxOpen 時，新的請求會在這裡排隊等待
    connRequests map[uint64]chan connRequest

    // 當前已打開的連線數
    numOpen int 
    
    // 配置參數
    maxIdle int // 最多允許多少人閒置 (SetMaxIdleConns)
    maxOpen int // 最多允許多少特使存在 (SetMaxOpenConns)
}
```

### 3.1 獲取連線 (Get Connection)
當你執行 `db.Query()` 時，流程如下：

1.  **檢查 `freeConn` (閒置區)**:
    *   如果有特使在休息，直接叫醒他 (Pop from slice)。
    *   **檢查過期**: 檢查他是否超過 `ConnMaxLifetime`？如果是，強制退休 (Close)，找下一個。
2.  **檢查 `numOpen` (總人數)**:
    *   如果 `numOpen < maxOpen`：可以直接招聘新特使 (Dial New Connection)。
3.  **排隊 (Wait)**:
    *   如果沒閒置且已滿額 (`numOpen >= maxOpen`)：
    *   創建一個 channel，掛到 `connRequests` 等待區。
    *   阻塞等待，直到有別人歸還連線。

### 3.2 歸還連線 (Put Connection / Release)
當你執行 `rows.Close()` 結束查詢時：

1.  **檢查等待區 (`connRequests`)**:
    *   如果有人在排隊，直接把連線過戶給他 (Handoff)。連線不回休息室，無縫接軌。
2.  **放回休息室 (`freeConn`)**:
    *   如果沒人在排隊，檢查 `len(freeConn) < maxIdle`？
    *   **是**: 放入 `freeConn` 列表。
    *   **否**: (休息室爆滿) 依依不捨地裁員，關閉連線 (Close)。

---

## 4. 第四樂章：與 MySQL 的對話 (Wire Protocol)

Driver (如 `go-sql-driver/mysql`) 的工作是把 Go 的函數呼叫翻譯成 MySQL 能懂的 **Binary Packet**。

MySQL 的協定非常簡單，每個封包都有一個 4 bytes header：
*   **3 bytes**: Payload Length (封包長度)。
*   **1 byte**: Sequence ID (序號，防止封包亂序).
*   **Body**: 實際指令。

**範例：發送查詢 `SELECT 1`**
1.  **Header**: Length=9 (假設 Body 9 bytes), Seq=0.
2.  **Body**:
    *   Command Byte: `0x03` (COM_QUERY - 代表這是一個查詢指令)
    *   Query String: "SELECT 1"

Driver 透過 `net.Conn.Write([]byte)` 把這些 bytes 塞進 TCP Socket。
然後 `Read()` 等待 MySQL 回傳 Resultset 封包。

所以，Go 與 MySQL 的交互，說穿了就是 **「拼裝 Bytes」** 與 **「Socket 讀寫」** 的藝術。

---

## 5. 第五樂章：調優指南 (Tuning Best Practices)

學會了底層原理，我們就知道該如何設定那些神秘參數了：

### 1. `SetMaxIdleConns` (閒置連線數)
*   **建議**: **設為跟 `SetMaxOpenConns` 一樣大**。
*   **原因**: 如果 `MaxIdle < MaxOpen` (例如 Open=100, Idle=10)，當高併發過後 (100 個連線)，有 90 個連線會因為擠不進休息室而被強制關閉。下一波流量來時，又得重新握手 90 次。這會導致 **Connection Thrashing (連線震盪)**，Pool 的效益大打折扣。

### 2. `SetMaxOpenConns` (最大連線數)
*   **建議**: 不要無腦設大。從 50~100 開始測試。
*   **原因**: 資料庫的吞吐量是有限的。過多的連線只會造成 DB Server 的 CPU 在 Context Switch 上空轉。Go 的 Pool 已經幫你做了 Queueing，與其讓 DB 掛掉，不如讓請求在 Go 這邊排隊。

### 3. `SetConnMaxLifetime` (連線最大壽命)
*   **建議**: **一定要設！** 且必須 **小於** 資料庫 Server 端 (或 Load Balancer) 的 `wait_timeout`。
*   **原因**: 如果不設，Go 會以為連線永遠有效。但防火牆或 DB 可能會在 5 分鐘後默默切斷連線。這時 Go 拿舊連線去送 Query，就會收到 EOF 或 broken pipe 錯誤 (雖然 Go 會重試，但這增加了延遲)。建議設為 30 分鐘或 1 小時。

### 4. 微服務標準配置懶人包 (Microservice Cheat Sheet)
如果您不想糾結，這裡有一組適合 **標準 K8s Pod (2 Core / 4GB RAM)** 的黃金參數，直接複製貼上即可安心上線：

```go
// 1. 限制總量，避免將 DB 打掛
db.SetMaxOpenConns(50) 

// 2. 閒置池全開，避免反覆握手 (Thrashing)
db.SetMaxIdleConns(50) 

// 3. 定期輪替 (30分鐘)，避免碰到防火牆或 DB 的 Timeout
db.SetConnMaxLifetime(30 * time.Minute)

// 4. (Go 1.15+) 如果真的太閒，10分鐘後回收，把資源還給 DB
db.SetConnMaxIdleTime(10 * time.Minute)
```

#### 進階公式：高併發怎麼算？ (The Scaling Formula)
很多人以為 QPS 越高，Pool 就要開越大，這是誤解。連線數是由 **Latency** 決定的。

> **公式**: `需要的連線數 = QPS * 平均查詢時間(秒) * 緩衝系數(2~3)`

*   **案例 A (快查詢)**: QPS 1000, 耗時 5ms (0.005s)。
    *   計算: `1000 * 0.005 * 2 = 10`。
    *   **結論**: 其實只要 **10 個連線** 就能扛住 1000 QPS。開多了只是浪費記憶體。

*   **案例 B (慢查詢)**: QPS 100, 耗時 200ms (0.2s)。
    *   計算: `100 * 0.2 * 2 = 40`。
    *   **結論**: 雖然 QPS 低，但因為查詢慢，反而需要更多連線。

**⚠️ 叢集陷阱 (Cluster Trap)**:
如果你有 200 個 Pods，每個都設 `MaxOpen=50`。
*   總連線數 = 200 * 50 = **10,000**。
*   這會直接把 MySQL 打掛 (Too many connections)。
*   **高併發解法**:
    1.  降低每個 Pod 的 `MaxOpen` (例如降到 5-10)。
    2.  或者中間加一層 **Proxy (如 ProxySQL, RDS Proxy)** 來做全局的連線復用。

#### 觀念修正：QPS vs TPS (Supply & Demand)
最後要修正一個常見迷思：**「連線數越多，吞吐量越高？」**
錯！這取決於 **Service QPS (需求)** 與 **DB TPS (供給)** 的平衡。

*   **Service QPS**: 你的程式每秒想發送多少請求。
*   **DB TPS**: 資料庫每秒 **真正能處理** 多少請求 (受限於 CPU/Disk)。

當 **QPS > TPS** 時 (例如 10000 請求湧入，但 DB 只能處理 5000)：
*   如果你開 10000 個連線硬塞：資料庫會忙於 **Context Switch**，導致 TPS 下降 (例如變 3000)，加速死亡。
*   **正確做法**: 利用 `SetMaxOpenConns` 限制併發 (例如 100)。
    *   多出來的 9900 個請求會在 **Go 端排隊** (Goroutine Park，成本低)。
    *   DB 則維持在最佳負載 (100 concurrent)，TPS 穩定輸出 (5000)。

**結論**: Connection Pool 不只是為了復用，更是為了 **保護下游資料庫**，將壓力留在應用層。

---

## 6. 第六樂章：Context 護身符 (Query Timeout & Cancellation)

連線池幫你管好了「連線的數量」，但還有一個致命問題它管不了：**「一條連線被某個慢查詢霸佔多久？」**

### 6.1 沒有 Context 的災難
假設你的 Pool 設了 `MaxOpenConns = 50`。
某天 DBA 不小心漏加了索引，一條 SQL 從 5ms 變成 30 秒。
*   50 條連線全部卡在這條慢 SQL 上，沒有人回來。
*   後續所有請求全部在 Go 端排隊 (`connRequests`)，堆積上萬個 Goroutine。
*   前端用戶看到的是：所有 API 全面 Timeout，系統癱瘓。

**解法**：每一次 DB 操作，都必須帶上 `context.WithTimeout`，強制設定死線！

```go
func GetUser(db *sql.DB, id int64) (*User, error) {
    // 給這條查詢最多 3 秒的生命，超時直接斬殺！
    ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
    defer cancel()

    var user User
    // 使用 QueryRowContext 而非 QueryRow
    err := db.QueryRowContext(ctx, "SELECT name, age FROM users WHERE id = ?", id).
        Scan(&user.Name, &user.Age)
    
    if err != nil {
        return nil, err // 超時會回傳 context.DeadlineExceeded
    }
    return &user, nil
}
```

當 3 秒一到，Go 的 Driver 會主動向 MySQL 發送 `KILL QUERY` 指令，把這條 SQL 斬斷。
連線被釋放回 Pool，不會繼續霸佔資源。

### 6.2 上游斷線，下游還在傻傻做 (Orphaned Query)
這是更隱蔽的災難。想像一個完整的請求鏈路：

```
用戶 (Browser) → API Gateway → Go Service → MySQL
```

用戶等了 2 秒不耐煩，直接關掉瀏覽器 (取消請求)。
API Gateway 收到斷線通知，把上游的 `context` 取消了。

**如果你的 Go Service 沒有把上游的 Context 傳遞下去**：
*   Go Service 完全不知道用戶已經走了。
*   它還在傻傻等 MySQL 跑完一條 10 秒的慢查詢。
*   查完之後，結果送回去才發現：「人呢？沒人收貨了！」
*   **白白浪費了 10 秒的 DB 連線 + CPU 算力。**

**正確做法**：把 HTTP Handler 的 `request.Context()` 一路往下傳遞！

```go
func HandleGetUser(w http.ResponseWriter, r *http.Request) {
    // r.Context() 會在用戶斷線時自動被取消 (Cancel)
    // 加上 WithTimeout 做雙重保險：就算用戶沒斷，最多也只等 3 秒
    ctx, cancel := context.WithTimeout(r.Context(), 3*time.Second)
    defer cancel()
    
    user, err := GetUserWithContext(ctx, db, userID)
    // ...
}

func GetUserWithContext(ctx context.Context, db *sql.DB, id int64) (*User, error) {
    var user User
    // 這裡的 ctx 同時繼承了「用戶斷線取消」和「3 秒超時」兩道保險
    err := db.QueryRowContext(ctx, "SELECT name, age FROM users WHERE id = ?", id).
        Scan(&user.Name, &user.Age)
    return &user, err
}
```

這樣一來，不管是「用戶主動斷線」還是「查詢超過 3 秒」，任何一個條件先觸發，Go 都會立刻斬斷 MySQL 查詢並歸還連線。

### 6.3 鐵律總結
1. **永遠使用 `XXXContext` 系列方法** (`QueryContext`, `ExecContext`, `QueryRowContext`)，而非無 Context 版本。
2. **上游的 Context 必須一路傳遞到最底層的 DB 呼叫**，確保取消信號能貫穿整條鏈路。
3. **每一層都可以加上自己的 `WithTimeout` 做雙重保險**，防止某一層的超時設定過於寬鬆。
