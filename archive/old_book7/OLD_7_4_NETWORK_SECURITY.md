# Book 7.4: The Dark Forest (網路攻防與雲端防禦)

歡迎來到網路的黑暗森林。
在前幾章，我們學習了 Handshake, Packet, Reliability 這些美好的機制。
但在駭客眼裡，每一個機制都是 **武器**。

這章我們將從 **攻擊者 (Attacker)** 的視角，看看如何利用 TCP/IP 的設計缺陷來癱瘓服務，以及身為 **防禦者 (Defender)** 該如何利用雲端架構自保。

---

## 1. 第一樂章：SYN Flood: 癱瘓 TCP 握手 (L4)

這是最經典的攻擊，專門針對 **Book 7.1** 的 TCP 3-Way Handshake。

### 1.1 攻擊原理 (The Weapon)
*   **機制**: Server 收到 `SYN` 後，會回傳 `SYN-ACK`，並在記憶體中建立一個 **半連線 (Half-Open Connection)** 等待對方回 `ACK`。
*   **漏洞**: 駭客發送千萬個 `SYN` 封包，但 **死都不回 ACK** (或是用假 IP 發送)。
*   **後果**: Server 的 **SYN Queue** (半連線佇列) 被塞滿。正常的 User 發送 SYN 進來，Server 因為佇列滿了直接丟棄。服務中斷。

### 1.2 防禦對策 (The Shield)
*   **Kernel 參數**: 加大 `net.ipv4.tcp_max_syn_backlog` (但治標不治本)。
*   **SYN Cookies (絕招)**:
    *   Server 收到 SYN 後，**不分配記憶體**，而是把連線資訊加密算成一個 Hash (Cookie) 放在 Sequence Number 裡回回去。
    *   只有當對方真的回了 `ACK` (且 Cookie 正確)，Server 才真正分配記憶體。
    *   **效果**: 讓握手變成 **Stateless**，攻擊者耗不盡 Server 資源。
    *   **實戰設定**: `sysctl -w net.ipv4.tcp_syncookies=1` (現代 Linux 預設已開啟)。
        *   **觸發機制**: Kernel 平時使用高效能的 Stateful 模式；只有當 **SYN Queue 爆滿** (被流量淹沒) 時，才會自動切換到 SYN Cookie 模式保命。
        *   **代價**: 在攻擊期間，部分 TCP 高級選項 (如 Window Scaling) 可能會失效，但「活著」比什麼都重要。

---

## 2. 第二樂章：UDP Reflection: 流量放大術 (L3)

這是針對 **Book 7.2** UDP 協議的攻擊，專門用來製造 **Tbps** 等級的流量海嘯。

### 2.1 攻擊原理 (The Weapon)
*   **機制**: UDP 是 **Stateless** 的，且不驗證來源 IP (可以偽造)。
*   **放大 (Amplification)**: 駭客找到網路上開放的 DNS 或 NTP Server。
    *   發送一個小請求 (64 bytes): "給我所有紀錄"。
    *   伺服器回傳一個大回應 (3000 bytes): 這是 **50倍** 的放大。
*   **反射 (Reflection)**: 駭客把來源 IP **偽造 (Spoof)** 成受害者 (Target) 的 IP。
    *   NTP Server 以為是 Target 問的，於是把 3000 bytes 的大垃圾倒在 Target 頭上。
*   **後果**: Target 的頻寬瞬間被塞爆，網路癱瘓。

### 2.2 防禦對策 (The Shield)
這種攻擊是 **拚頻寬** (Volumetric Attack)。您的 EC2 肯定扛不住。
*   **CDN (Anycast)**: 把流量分散到全球幾千個節點。
*   **Magic Transit (Cloudflare)**: 上游 ISP 直接幫您清洗流量，把 UDP 垃圾丟掉，只讓乾淨的 TCP 進來。

---

## 3. 第三樂章：Slowloris & HTTP Flood: 應用層癱瘓 (L7)

這是針對 **Book 7.3** 資源佔用的攻擊。

### 3.1 Slowloris (慢速攻擊)
*   **機制**: 駭客建立連線後，發送 HTTP Header，但 **發得非常慢** (每 10 秒發一個 byte)。
*   **後果**: Nginx/Go Server 會傻傻地等它 (Keep-Alive)，導致 Worker/Goroutine 被佔用。只要幾千個這種連線，Server 就無法服務其他人。
*   **解法**: 設定嚴格的 `client_header_timeout` 和 `client_body_timeout` (例如 5 秒)。逾時直接踢掉。

### 3.2 CC Attack (HTTP Flood - Challenge Collapsar)
*   **機制**: 駭客控制千萬台殭屍電腦 (Botnet)，模擬正常人用瀏覽器訪問您的重度 API (例如搜尋介面)。
*   **特點**: 這些請求完全符合 HTTP 標準，**SYN Cookie 和防火牆擋不住**。
*   **後果**: API Server CPU 滿載，資料庫掛點。

### 3.3 防禦對策 (WAF)
這時候只能靠 **Web Application Firewall (WAF)** 進行智力對決：
1.  **Rate Limiting**: 限制單一 IP 每秒只能發 10 次請求。
2.  **JS Challenge (Captcha)**: 回傳一段 JavaScript 給 Client。如果是瀏覽器會自動執行算出答案；如果是簡單腳本就會卡住。
3.  **IP Reputation**: 直接封鎖已知的殭屍 IP 列表。

---

## 4. 第四樂章：來源站隱身術 (Origin Cloaking)

### 4.1 為什麼要隱身？
如果您用了 Cloudflare，但駭客知道了您 Origin EC2 的 **真實 IP (Real IP)**：
*   駭客可以 **繞過 Cloudflare**，直接把 UDP 垃圾倒在您的 EC2 上。
*   您的 WAF、DDoS 防護全部失效。

### 4.2 如何隱身？
1.  **Security Group 白名單**:
    *   在 AWS Security Group 設定：**Inbound 只允許 Cloudflare 的 IP 段** (Cloudflare 有公開 IP List)。
    *   拒絕所有其他 IP 的 port 80/443 連線。
2.  **mTLS (Authenticated Origin Pulls)**:
    *   Cloudflare 連線回您的 Nginx 時，會出示一張特殊的 **Client Certificate**。
    *   您的 Nginx 設定 `ssl_client_verify`，沒有這張證書的連線直接拒絕。

---

## 5. 第五樂章：全系列總結 (Grand Summary)

恭喜！您已經完成了 **Book 7: Go Network & Architecture** 全系列。

1.  **7.1 (The Gateway)**: 我們學會了 Nginx 反代、Keep-Alive 的重要性，以及為什麼要用 CDN 做 Edge Termination。
2.  **7.2 (The Journey)**: 我們看穿了 16KB Record 背後的 11 個封包，以及 MTU 與 UDP 的物理限制。
3.  **7.3 (The Reliability)**: 我們理解了 TCP 的代價 (TIME_WAIT)，以及如何正確進行壓測 (避免 Port Exhaustion)。
4.  **7.4 (The Security)**: 我們學會了如何防禦 L4 (SYN Flood) 與 L7 (CC Attack) 的攻擊。

接下來，您可以帶著這些 **「底層視角」** 回去寫 Go 程式碼。您會發現，每一個 `conn.Write` 不同了，每一個 `http.Get` 都不再單純了。
You are now a Network Architect.
