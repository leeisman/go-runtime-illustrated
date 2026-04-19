# Book 7.3: Reliability & Teardown (可靠性的代價)

在 7.2 中，我們看到物理網路是多麼不可靠 (掉包、亂序、MTU 限制)。
但為什麼在 Go 語言裡，我們寫 `conn.Read()`時，讀到的資料永遠是**完整且正確**的？

因為 **TCP (Transport Layer)** 在背後為我們承擔了所有的痛苦。
但這份「可靠性」不是免費的，它帶來了巨大的代價：**延遲 (Latency)** 與 **資源佔用 (State)**。

---

## 1. 第一樂章：連線的解剖 (Anatomy of a Connection)

在進入可靠性討論前，我們先回答兩個最底層的問題：**「連線是真的嗎？」** 與 **「封包怎麼認路？」**

### 1.1 The Physical Truth (連線是一條線嗎？)
*   **答案**: **不是**。世界上沒有一條線連著你跟 Server。
*   **真相**: 所謂 `ESTABLISHED` 連線，只是兩台電腦的 Kernel **「講好了」**。
    *   **Client Kernel**: 在記憶體開一個 `struct sock`，紀錄 "我要連 Server"。
    *   **Server Kernel**: 在記憶體開一個 `struct sock`，紀錄 "我要連 Client"。
    *   **斷線**: 只要一邊把這個 struct 刪掉 (或當機)，這條「線」就斷了。

### 1.2 The Packet Journey (封包如何到達？)
當您呼叫 `connect("8.8.8.8:80")` 時，OS 做了什麼？
1.  **查路由 (Routing Table)**: Kernel 翻開地圖 (`ip route`)，發現去 `8.8.8.8` 要經過 **Gateway (192.168.1.1)**。
2.  **問地址 (ARP)**: Kernel 大吼一聲 (Broadcast)：「誰是 192.168.1.1？你的 MAC 是什麼？」
3.  **封裝 (L2 Frame)**: 拿到 Gateway 的 MAC 後，把 IP 封包包進 **Ethernet Frame**，丟給網卡。
4.  **接力 (Hop-by-Hop)**: Gateway 收到後，拆開看 IP，再轉給下一個 Router (ISP)... 一站站轉發直到 Google。

### 1.3 Why 3-Way Handshake? (為什麼要三次？)
既然沒有線，那握手是為了什麼？**為了「同步 (Sync)」這場集體幻覺的參數。**
*   **2 次握手不行嗎？**
    *   如果 A 發了 `SYN` (迷路了)，A 以為斷了又重發新的。
    *   舊的 `SYN` 突然到了 B。B 回復 `OK`，然後 B 就以為連線建立了，開始傻傻等資料。
    *   A 收到這個「舊的 OK」，因為 A 根本沒打算連線 (它是舊的)，所以 A 會無視。
    *   **結果**: B 的資源被這個「幽靈連線」吃光。
*   **3 次握手 (The Confirmation)**:
    *   B 回復 `SYN-ACK` 後，**還不會** 分配全部記憶體。
    *   必須等到 A 再回傳最後一個 `ACK` ("對，我真的要連")，B 才會正式建立連線。

---

## 2. 第二樂章：連續性的錯覺 (The Illusion of Continuity)

Go 的 `net.Conn` 給你一種「水管 (Stream)」的感覺，但其實底層是「碎片 (Packets)」。

### 1.1 Sequence Number (拼圖編號)
每個 TCP 封包都帶有一個 **Sequence Number (Seq)**。
*   封包 A (Seq 1, Len 100): "Hello "
*   封包 B (Seq 101, Len 100): "World"

如果封包 B 先到了，A 後到。Kernel 的 TCP Stack 會把 B 先放在 **Receive Buffer** 裡暫存，**不給 Go 程式看**。等到 A 到了，拼好 "Hello World" 之後，才喚醒 Go 的 `Read()`。

### 1.2 ACK (掛號信回執)
發送方怎麼知道對方收到了？
*   接收方必須回傳 **ACK (Acknowledgement)**。
*   `ACK 101` 代表：「Seq 101 以前的我都收到了，請給我 101 以後的資料」。
*   **重傳 (Retransmission)**: 如果發送方發了 A，過了 200ms (RTO) 還沒收到 ACK，它就認定 A 掉了，會重發 A。

---

## 3. 第三樂章：斷線的藝術 (The Teardown Nightmare)

TCP 的**建立連線 (3-Way Handshake)** 很簡單，但 **「分手 (4-Way Wave)」** 卻是導致伺服器崩潰的元兇。

### 3.1 The 4-Way Wave (四次揮手)
假設 **Client (Active Closer)** 主動想斷線：
1.  **Client**: 發送 `FIN` (我講完了)。進入 `FIN_WAIT_1`。
2.  **Server**: 收到 `FIN`，回傳 `ACK` (知道了)。進入 `CLOSE_WAIT`。
    *   **關鍵**: 此時 Server 的 **Kernel** 知道對方要走了，但 **App (Go)** 可能還沒處理完資料！Kernel 必須等 Go 程式呼叫 `conn.Close()` 才能繼續。
3.  **Server (App)**: 處理完，呼叫 `Close()`，發送 `FIN`。進入 `LAST_ACK`。
4.  **Client**: 收到 `FIN`，回傳 `ACK`。進入 **`TIME_WAIT`**。
5.  **Server**: 收到 `ACK`，正式 `CLOSED`。

---

## 4. 第四樂章：TIME_WAIT vs CLOSE_WAIT 運維複盤 (Ops Review)

這是 SRE 面試必考題，也是線上故障 (Outage) 的常見原因。

### 4.1 CLOSE_WAIT (Server 端的惡夢)
*   **現象**: 伺服器上有成千上萬個連線卡在 `CLOSE_WAIT`，導致 File Descriptor (FD) 耗盡，無法接受新連線。
*   **成因**: **程式碼寫爛了 (Code Bug)**。
    *   對方 (Client) 已經發送 `FIN` 說要走了，Server 的 Kernel 也收到了並回了 ACK。
    *   但是 **Server 的 App (Go 程式碼)** 遲遲不去呼叫 `conn.Close()` (例如卡在死無窮迴圈，或是忘了寫 Close)。
    *   **結論**: `CLOSE_WAIT` 永遠是 **被動關閉方 (Server)** 的錯。請去 fix code。

### 4.2 TIME_WAIT (Client 端的詛咒)

*   **誰會有 TIME_WAIT？**: 永遠是 **「主動喊分手的那個 (Active Closer)」**。
    *   如果是 Client 斷線，TIME_WAIT 在 Client。
    *   如果是 Server (如 Nginx) 主動踢掉連線，TIME_WAIT 在 Server。
*   **為什麼我在壓測 (Load Testing) 時常看到它？**
    *   **數學題 (Port Exhaustion)**:
        *   Linux 可用的臨時 Port (Ephemeral Ports) 範圍只有約 **60,000 個** (`sysctl net.ipv4.ip_local_port_range`).
        *   每個 TIME_WAIT 必須佔用這個 Port **60 秒** (2MSL) 不能釋放。
        *   **容量上限**: `60,000 ports / 60 sec = 1,000 connections/sec`。
    *   **崩潰**: 只要您的壓測 QPS 超過 **1,000** 且使用短連線，第 61 秒時，您的 Port 就乾了。OS 會報錯：`dial: cannot assign requested address`。
*   **為什麼要等 60 秒 (2MSL)？** (不能縮短嗎？)
    *   **理由 1 (可靠性)**: 確保最後一個 ACK 順利到達。如果沒等到，對方重傳 FIN 時我還在，可以再次回 ACK。
    *   **理由 2 (防干擾 - Ghost Packets)**: 這是重點。網路上可能還有迷路的舊封包 (Sequence 100)。如果我們立刻把 ip:port 重用給新連線，這個舊的 Seq 100 剛好撞進新連線的視窗，會導致資料錯亂 (Data Corruption)。所以必須等 2 倍的生命週期 (Life Time)，確保舊封包都死光了。

### 4.3 SRE Practice: Correct Load Testing Strategy (壓測正確姿勢)
很多工程師壓測失敗，是因為「測錯了對象」。

1.  **測「程式邏輯/資料庫」 (Target: Application)**:
    *   **方法**: 使用 **長連接 (Keep-Alive)**。
    *   **目標**: 讓 Go Server CPU 飆到 80% 以上，或是 DB 連線滿載。
    *   **原因**: 我們要測的是「館長處理業務邏輯有多快 (Logic)」，不是「新客人辦入館手續有多快 (Handshake)」。如果你用短連線，壓力全在 OS 握手，程式碼根本還沒用力，且 Client 端容易先耗盡 Port。

2.  **測「網關/防火牆能力」 (Target: Gateway)**:
    *   **方法**: 使用 **短連線 (Short Connection)**。
    *   **目標**: 測試 Nginx/L4 LB 每秒能接受多少 **New Connections** (Handshake Capacity)。

3.  **關於連線數 (Connections vs QPS)**:
    *   **100 連線打 10 萬 QPS**: 這是測 **「邏輯極限」**。Go 只需開 100 個 Goroutine，Context Switch 極少，跑分最高。
    *   **1 萬連線打 10 萬 QPS**: 這是測 **「高併發穩定性」**。Go 需維護 1 萬個 Goroutine 與 Stack，記憶體與調度成本較高。
    *   **建議**: 先用少連線 (如 100) 測出邏輯極限，再慢慢拉高連線數 (如 5000) 看效能衰退幅度。

### 4.4 SRE Case Study: Debugging TIME_WAIT (找出兇手)
當您在 Grafana 或機器上發現大量 TIME_WAIT 時，`netstat` 通常看不到 PID (因為 Socket 已歸還給 Kernel)。
我們必須用 **Port** 來進行推理 (`netstat -tn | grep TIME_WAIT`)：

| Local Address | Foreign Address | 嫌疑犯 | 原因與對策 |
| :--- | :--- | :--- | :--- |
| **`Server:80`** | `User:Random` | **Nginx (Inbound)** | **原因**: Nginx 主動切斷 User (Active Close)。<br>**對策**: 檢查 `keepalive_timeout` 或 `keepalive_requests` (預設 1000，超過就踢)。 |
| `Server:Random` | **`Go:8080`** | **Nginx (Upstream)** | **原因**: Nginx (作為 Client) 主動切斷 Go。<br>**對策**: Nginx 預設走 HTTP/1.0 短連線。請加上 `proxy_http_version 1.1;` 和 `keepalive 32;`。 |
| `Server:Random` | **`DB:3306`** | **Go App (Outbound)** | **原因**: Go App 主動切斷 DB。<br>**對策**: **危險！** 代表 Connection Pool 配置錯誤 (MaxIdleConns 太小)，導致連線用完即丟。 |

**口訣**:
*   看到 **Local Port 固定 (80/443)**: 代表您是 Server，您踢了 Client。
*   看到 **Foreign Port 固定 (3306/8080)**: 代表您是 Client，您踢了 Backend/DB。


### 4.5 Risk Assessment: Inbound vs Outbound (危險程度)
為什麼要在意 TIME_WAIT 出現在哪一側？因為 **「資源消耗」** 不同，導致 **「死亡紅線」** 不同。

| 類型 | 方向 | 消耗資源 | 上限瓶頸 | 危險程度 | 後果 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Inbound** | 別人連我 (我是 Server) | **記憶體 (RAM)** | 取決於 RAM (通常可達百萬) | **低 (Low)** | 除非多到吃光 RAM，否則通常沒事。您可以容忍 10 萬個 Inbound TIME_WAIT。 |
| **Outbound** | 我連別人 (我是 Client) | **Port (65535)** | **約 28,000 - 60,000** (視 OS 設定) | **極高 (Critical)** | 一旦耗盡，**所有對外連線 (DB/Redis/API) 全部失敗**。OS 報錯 `cannot assign requested address`。 |

**SRE 判斷法則**:
*   **Outbound TIME_WAIT > 20,000**: **紅色警戒**。請檢查 Connection Pool 或開啟 `tcp_tw_reuse`。
*   **Inbound TIME_WAIT > 20,000**: **綠色/黃色**。通常是正常的，除非記憶體報警。

---

## 5. 第五樂章：優化策略 (Optimization Strategy)

### 5.1 解決 TIME_WAIT (Port 耗盡)
如果您的 Nginx (連後端 Go) 出現大量 TIME_WAIT：
1.  **長連接 (Keep-Alive)**: 最治本的方法。不要斷線，就沒有 TIME_WAIT。
2.  **`net.ipv4.tcp_tw_reuse = 1`**: 允許 Kernel 安全地重用 TIME_WAIT 狀態的 Port (僅限 Outbound 連線)。
    *   *注意*: `tcp_tw_recycle` 在新版 Linux 已經被廢除 (因為 NAT 環境下會出事)，不要再用了。

### 5.2 解決 CLOSE_WAIT (FD 耗盡)
*   **Fix Code**: 檢查您的 Go 程式碼。是不是忘了 `defer conn.Close()`？是不是 Goroutine 卡死導致沒執行到 Close？
*   **Read Timeout**: 設定 `SetReadDeadline`，強迫卡住的連線超時斷開。

---

## 6. 第六樂章：總結 (Summary)
*   **Reliability**: 是靠 Seq + ACK + Retransmit 換來的。代價是延遲。
*   **CLOSE_WAIT**: 是 **程式 Bug** (Server 忘了關門)。
*   **TIME_WAIT**: 是 **TCP 機制** (Client 為了保險故意停留)。可以透過 Keep-Alive 或 Reuse 優化。
