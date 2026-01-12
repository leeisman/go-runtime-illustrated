# Book 7.4: The Operations (運維實戰：故障與優化)

當理論遇到現實，災難往往就發生了。
這一章是 SRE 與 Backend Lead 必須精通的戰場生存指南：如何除錯 TCP 狀態、如何正確壓測、以及 Nginx 關鍵調校。

---

## 1. TCP State Debugging (狀態機故障)

### 1.1 CLOSE_WAIT (Server 端的惡夢)
*   **症狀**: Server 有大量 `CLOSE_WAIT`，導致 FD 耗盡。
*   **原因**: **Application Bug**。Server 收到 Client 的 FIN，Kernel 回了 ACK，但 **Go 程式忘了呼叫 `conn.Close()`**。
*   **解法**: Fix your code (defer close)。

### 1.2 TIME_WAIT (Client 端的詛咒)
*   **症狀**: Client (或反代 Nginx) 無法建立新連線，報錯 `cannot assign requested address`。
*   **原因**: **Port Exhaustion**。
    *   主動關閉連線的一方 (Active Closer) 必須進入 TIME_WAIT 等待 **2MSL (60秒)**。
    *   Linux Ephemeral Ports 只有約 **60,000** 個。
    *   `60,000 / 60s = 1000 TPS`。只要每秒建立超過 1000 個短連線，Port 就會耗盡。
*   **解法**:
    1.  **Keep-Alive (治本)**: 不要頻繁斷線。
    2.  **`net.ipv4.tcp_tw_reuse = 1`**: 允許重用 TIME_WAIT socket (僅限 Outbound)。

### 1.3 Debugging Table (誰是兇手？)
| Local | Foreign | 嫌疑犯 | 原因 |
| :--- | :--- | :--- | :--- |
| **`:80`** | `User:xxx` | **Nginx** | Nginx 主動踢 User (Inbound)。無害 (耗 RAM)。 |
| `Random` | **`:8080`** | **Nginx** | Nginx 主動踢 Upstream。**有害** (耗 Port)。 |
| `Random` | **`DB:3306`** | **App** | Go 主動踢 DB。**有害** (Bad Pool)。 |

---

## 2. Load Testing Strategy (壓測正確姿勢)

### 2.1 對象決定策略
1.  **測程式邏輯 (Logic)**:
    *   用 **長連接 (Keep-Alive)**。我們想看 CPU 能跑多快，不要讓 TCP 握手成為瓶頸。
2.  **測網關吞吐 (Gateway)**:
    *   用 **短連線 (Short Conn)**。測試 Nginx 每秒能處理多少握手。

### 2.2 連線數迷思
*   **少連線 (100) 打高 QPS**: 測極限吞吐 (Throughput)。
*   **多連線 (10k) 打高 QPS**: 測高併發穩定性 (Concurrency / Scheduler Overhead)。

---

## 3. Nginx Tuning (反代優化)

Nginx 是最常見的 Gateway，預設配置通常不夠用。

### 3.1 Upstream Keepalive
Nginx 預設對 Upstream (Go) 用 **HTTP/1.0 短連線**。這會導致上面提到的 TIME_WAIT 災難。
**正確配置**:
```nginx
upstream backend {
    server 127.0.0.1:8080;
    keepalive 32; # 每個 Worker 維持 32 個長連接
}
server {
    location / {
        proxy_pass http://backend;
        proxy_http_version 1.1; # 必須啟用 1.1
        proxy_set_header Connection ""; # 清除 Close header
    }
}
```

---

## 4. Summary
*   **CLOSE_WAIT** 是您的 `Close()` 沒寫好。
*   **TIME_WAIT** 是 TCP 為了可靠性而付出的代價 (Port 佔用)。
*   **壓測** 時要清楚自己在測什麼 (Logic vs Handshake)。
*   **Nginx** 必須手動開啟 Upstream Keepalive。

系統穩定了，但安全嗎？
下一章 **Book 7.5**，我們進入最後的戰場：**網路攻防 (Security)**。
