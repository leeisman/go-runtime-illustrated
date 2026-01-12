# Book 7.3: The Application (應用層：HTTP, TLS 與進化)

有了這條可靠的 TCP 水管後，我們要傳送什麼內容？
這一章探討 L7 協議的演進，看人類如何從簡單的文字傳輸 (**HTTP/1.1**)，進化到極致效能的 UDP 傳輸 (**HTTP/3**)。

---

## 1. HTTP Evolution (Web 協議的進化)

### 1.1 HTTP/1.1 (The Standard)
*   **特徵**: 纯文本 (Text-based)，易讀。
*   **Keep-Alive**: 用 `Connection: keep-alive` 讓同一條 TCP 連線可以送多個 Request，省去握手成本。
*   **缺陷 (Head-of-Line Blocking)**:
    *   這是 **應用層 (L7)** 的阻塞。
    *   你必須等 Request 1 回來，才能發 Request 2 (Serial Processing)。
    *   如果第一個請求卡住 (如 DB 慢)，整條連線都癱瘓。

### 1.2 HTTP/2 (The Multiplexing)
*   **特徵**: 二進制分幀 (Binary Framing)。
*   **多路復用 (Multiplexing)**:
    *   在同一條 TCP 連線上，同時跑多個 Stream (Stream ID)。
    *   Request 1 和 Request 2 可以並行發送，互不等待。解決了 L7 的阻塞。
*   **缺陷 (TCP HOL Blocking)**:
    *   因為底層還是 **一條 TCP**。如果掉了一個 TCP 封包，Kernel 會卡住整條 Stream 等重傳，導致所有 Request (即使沒掉包的) 都被卡住。
    *   **結論**: 在弱網環境下，HTTP/2 比 HTTP/1.1 還慢。

### 1.3 HTTP/3 & QUIC (The UDP Revolution)
*   **核心**: **拋棄 TCP，改用 UDP**。
*   **QUIC 協議**:
    *   在 UDP 之上自己實作了 Reliability (重傳) 和 Congestion Control。
    *   **無隊頭阻塞**: 每個 Stream 獨立。Stream A 掉包只會卡住 Stream A，Stream B 照常運作。
    *   **Connection Migration**: 切換網路 (Wi-Fi 轉 4G) 不用重連 (用 Connection ID 取代 IP:Port)。

---

## 2. TLS: The Secure Layer (加密層)

現在幾乎所有 L7 協議都跑在 TLS 之上 (HTTPS, MQTTS, WSS)。

### 2.1 TLS Handshake (握手成本)
TLS 握手發生在 TCP 握手 **之後**，是額外的開銷。
*   **TLS 1.2**: 需要 **2-RTT** (往返兩次) 才能交換 Key。
*   **TLS 1.3**: 優化到 **1-RTT** (甚至 0-RTT Session Resumption)。這對手機端延遲至關重要。

### 2.2 Encryption Overhead
*   **Symmetric (AES/Chacha20)**: 資料傳輸用。現代 CPU (AES-NI) 跑極快，通常不是瓶頸。
*   **Asymmetric (RSA/ECDSA)**: 握手交換 Key 用。計算量大，消耗 CPU。
*   **kTLS**: Linux Kernel 提供的 Zero-Copy TLS，讓網卡或 Kernel 直接加密，減少 User/Kernel Copy 次數 (Book 7.1 舊內容)。

---

## 3. WebSocket (即時與雙向)

HTTP 是 "Request-Response" 模型，Server 不能主動說話。
WebSocket 解決了這個問題。

### 3.1 Upgrade Mechanism
*   **握手**: Client 發送一個特殊的 HTTP Request (`Upgrade: websocket`)。
*   **切換**: Server 同意後 (`101 Switching Protocols`)，這條 TCP 連線就脫離了 HTTP 規範，變成全雙工的 Raw Socket。
*   **應用**: 聊天室、股票報價、遊戲信令。

---

## 4. Summary
*   **HTTP/1.1**: 簡單，但有 L7 阻塞。
*   **HTTP/2**: 解決 L7 阻塞，但受限於 TCP L4 阻塞。
*   **HTTP/3**: 用 UDP 解決一切阻塞，是未來的趨勢。
*   **TLS**: 是必要的安全層，但要注意握手延遲 (RTT)。

有了高效的應用協議，接下來我們要面對現實世界的問題：**當流量太大時，系統會怎麼崩潰？**。
下一章 **Book 7.4** 進入運維視角 (Ops)。
