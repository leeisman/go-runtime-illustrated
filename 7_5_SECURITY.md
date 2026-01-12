# Book 7.5: The Security (網路攻防：黑暗森林)

這是 **Book 7 系列** 的最終章。
我們前面學的所有機制 (Handshake, Stateless UDP, Keep-Alive)，在駭客眼裡都是 **武器**。
這一章從攻擊者視角出發，並探討防禦之道。

---

## 1. SYN Flood (L4 Attack)

針對 TCP 3-Way Handshake 的攻擊。

### 1.1 機制
*   **攻擊**: 駭客發送大量 `SYN`，但死都不回 `ACK` (或用偽造 IP)。
*   **後果**: Server 分配了記憶體 (Syn Queue) 等待握手完成，結果資源耗盡，無法服務正常人。
*   **防禦**: **SYN Cookies**。
    *   Server 收到 SYN 不分配記憶體，而是把狀態加密算成 Cookie 放在 Seq Number 回回去。
    *   只有對方回了正確的 ACK，才真正建立連線。
    *   **設定**: `sysctl -w net.ipv4.tcp_syncookies=1`。

---

## 2. UDP Reflection & Amplification (L3 Attack)

針對 UDP Stateless 特性的攻擊。

### 2.1 機制
*   **反射 (Reflection)**: 駭客偽造 Source IP 成受害者 (Victim IP)，發請求給 DNS/NTP Server。DNS/NTP 把回應寄給受害者。
*   **放大 (Amplification)**: 請求很小 (64 bytes)，回應很大 (3000 bytes)。放大倍率 ~50x。
*   **後果**: 受害者頻寬被塞爆 (Tbps 等級)。
*   **防禦**:
    *   **Anycast CDN**: 用全球節點分散流量。
    *   **流量清洗**: ISP 層級丟棄 UDP 垃圾封包。

---

## 3. HTTP Flood & Slowloris (L7 Attack)

針對 Application Layer 的攻擊。

### 3.1 Slowloris (慢速攻擊)
*   **攻擊**: 建立連線後，每 10 秒才送 1 個 byte header。
*   **後果**: Server (如 Go/Nginx) 為了維持 Keep-Alive 而佔用 Worker/Goroutine，導致資源耗盡。
*   **防禦**: 設定嚴格的 `client_header_timeout`。

### 3.2 CC Attack (Challenge Collapsar)
*   **攻擊**: 模擬正常瀏覽器，瘋狂訪問高耗能 API (如搜索)。
*   **防禦**: **WAF (Web Application Firewall)**。
    *   Rate Limiting (限流)。
    *   JS Challenge (驗證碼)。

---

## 4. Origin Cloaking (隱身術)

### 4.1 核心概念
如果駭客知道您 Origin EC2 的真實 IP，他就可以繞過 CDN/WAF 直接打你。

### 4.2 實作
1.  **Security Group**: 只允許 Cloudflare IP 訪問 Port 443。拒絕所有其他 IP。
2.  **mTLS (Authenticated Origin Pulls)**: Nginx 驗證 Cloudflare 的 Client Certificate，確保連線真的來自 Cloudflare。

---

## 5. Grand Summary (全系列結語)

恭喜您完成了 **Book 7: Network Architecture** 全五冊！

*   **7.1 Infra**: 懂了 IP, DNS, NAT。
*   **7.2 Transport**: 懂了 TCP Stream vs UDP Datagram。
*   **7.3 Application**: 懂了 HTTP/3 與 TLS。
*   **7.4 Ops**: 懂了如何壓測與調優 Nginx。
*   **7.5 Security**: 懂了如何保護這一切。

您現在已經具備了 **架構師 (Architect)** 等級的網路視角。
無論是寫 Go Microservices, 調校 Kubernetes Ingress, 還是設計跨國系統，這些知識都將是您最強大的基石。
