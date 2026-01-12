# Book 7.2: The Packet Journey (封包的物理旅程)

在 7.1 中，我們討論了 Nginx 如何將資料加密並切成 **TLS Records (16KB)**。
但在這之後，資料離開了 User Space 的保護，被推入了 Kernel，並準備進入那條殘酷的網線。

歡迎來到物理世界。在這裡，沒有無限的 Buffer，只有 **MTU (Maximum Transmission Unit)** 的鐵律。

---

## 1. The Russian Doll (封裝: Encapsulation)

當 Go 的 `conn.Write("Hello")` 發生時，OS 就像打包俄羅斯娃娃一樣，給資料穿上一層又一層的衣服。

### 1.1 The Layers (由內而外)
假設我們傳送一個 HTTPS 請求：

1.  **L7 Application (TLS Record)**: `[TLS Header | Encrypted Data (16KB)]`
    *   這是 Nginx 切好的一塊資料。
2.  **L4 Transport (TCP Header)**: `[Src Port: 443 | Dst Port: 54388 | Seq: 1001]` + `[Payload]`
    *   **職責**: **Reliability**。保證順序、重傳。
    *   **大小**: 通常 20-60 Bytes。
3.  **L3 Network (IP Header)**: `[Src IP: 1.2.3.4 | Dst IP: 8.8.8.8 | TTL: 64]` + `[Payload]`
    *   **職責**: **Routing**。決定這包資料要去地球哪個角落。
    *   **大小**: 通常 20 Bytes。
4.  **L2 Data Link (Ethernet Frame)**: `[Src MAC: AA:BB... | Dst MAC: CC:DD...]` + `[Payload]` + `[FCS]`
    *   **職責**: **Hop-by-Hop Delivery**。決定下一站要丟給哪個鄰居 (Router/Switch)。
    *   **大小**: Header 14 Bytes + FCS 4 Bytes。

---

## 2. The Physical Constraints (MTU & MSS)

為什麼我們不能一次把 16KB 的 TLS Record 直接丟進網線？
因為乙太網 (Ethernet) 的物理標準規定：**一個 Frame 最大只能裝 1500 Bytes**。這就是 **MTU**。

### 2.1 The Math: MSS (Maximum Segment Size)
Kernel 必須把 16KB 的資料切碎。切多大？

*   **MTU**: 1500 Bytes (路規上限)
*   **IP Header**: -20 Bytes
*   **TCP Header**: -20 Bytes
*   **MSS**: **1460 Bytes** (真正能裝資料的空間)

**結果**:
一個 16KB (16384 bytes) 的 TLS Record，會被切成 `16384 / 1460 ≈ 11.2`，也就是 **12 個 TCP Packets**。

### 2.2 The Overhead (效率問題)
*   每個封包只有 1460 bytes 是乾貨，剩下的 40 bytes (IP+TCP) 甚至 54 bytes (含 Ethernet) 是包裝紙。
*   **效率**: `1460 / 1514 ≈ 96.4%`。
*   **隱憂**: 當封包越小 (例如遊戲封包只有 50 bytes)，包裝紙佔比就越高，頻寬利用率越低。

### 2.3 Jumbo Frames (巨型封包: 9000 Bytes)
在 **AWS/GCP 的 VPC 內網**，或者企業的 **Storage Network (iSCSI)**，我們通常會開啟 **Jumbo Frames**。
*   **MTU**: 設定為 **9001 Bytes**。
*   **MSS**: 8961 Bytes。
*   **效益**:
    1.  **效率提升**: Header 佔比大幅下降。
    2.  **CPU 減負**: 傳送 1MB 資料，原本要處理 700 次 Interrupt，現在只要處理 100 次。(這是最主要的效能來源)。
*   **注意**: **絕對不能在 Public Internet 開啟 Jumbo Frames**，因為中間的路由器不支援，封包會被直接丟掉。

### 2.4 Protocol Strategy: The Game Developer's Dilemma (TCP vs UDP)
既然小封包效率這麼差 (Payload 50 bytes, Header 40 bytes)，為什麼遊戲還是要這樣送？
*   **Trade-off**: 遊戲為了 **「快 (Low Latency)」**，寧願犧牲頻寬效率。如果不立刻發送這個封包，玩家看到的畫面就會延遲。**「我就爛！我就浪費頻寬！」** 是即時遊戲的座右銘。

**如何選擇協議？**
1.  **策略類 (RPG / 卡牌 / 德州撲克)**:
    *   **首選 TCP**: 因為 **「準確」** > 「速度」。玩家不能容忍發牌錯誤，但可以容忍 300ms 的動畫延遲。TCP 幫您解決了順序與重傳問題，開發最穩。
2.  **動作類 (FPS 射擊 / MOBA / 賽車)**:
    *   **首選 UDP**: 因為 TCP 的 **Head-of-Line Blocking** 是死罪。
    *   如果 TCP 掉了一個包，畫面會完全凍結 (Freeze) 等待重傳。在 CS:GO 裡，這 0.1 秒的卡頓就代表死亡。UDP 掉了就掉了，直接畫下一幀最新的就好。
3.  **架構鐵則: Traffic Separation (分流)**:
    *   **千萬不要** 把「語音 (Voice)」和「邏輯 (Game Logic)」塞在同一條 TCP 連線。
    *   語音數據量大、容忍掉包；邏輯數據量小、不能掉包。如果混在一起，語音掉包會導致 TCP 卡住，連帶害您的遊戲邏輯卡死。**語音走 UDP，邏輯走 TCP，各走各的路。**

### 2.5 Deep Dive: Stream (TCP) vs Datagram (UDP)
我們常說 TCP 是「串流 (Stream)」，UDP 是「數據報 (Datagram)」，到底在 **OS Kernel 與 網卡** 層級有什麼不同？

#### A. TCP (The Buffering Stream)
TCP 被設計用來榨乾頻寬，所以它很喜歡「湊單」。
1.  **App Layer**: 當您呼叫 `conn.Write([]byte("Hello"))`。
2.  **Kernel Layer**: OS **不會立刻發送**！它只是把這 5 bytes **複製 (Copy)** 到 Socket Send Buffer。
3.  **Transmission**: OS 等一下 (幾微秒到幾毫秒)，看後面還有沒有資料 (Nagle's Algorithm)。然後把它們湊成一個盡量滿的 **MSS (1460 bytes)** 封包，再一次發出去。
4.  **Receiver**: 對方收到的是 `"HelloWord..."` 黏在一起的資料流。
    *   **關鍵**: **TCP 沒有邊界 (No Boundary)**。您發送 10 次 10 bytes，對方可能 1 次讀到 100 bytes。

#### B. UDP (The Direct Shot)
UDP 是直腸子，給什麼送什麼。
1.  **App Layer**: 當您呼叫 `udpConn.Write([]byte("Hello"))`。
2.  **Kernel Layer**: OS **直接** 把它包上 UDP Header (8 bytes) + IP Header，立刻推送到網卡 Queue。
3.  **Transmission**: 網線上面立刻出現一個包含 "Hello" 的電子訊號。
4.  **Receiver**: 對方讀到的 **絕對是** "Hello"。
    *   **關鍵**: **UDP 保留邊界 (Preserves Boundary)**。發送 1 次就是 1 個包。

**底層資料流比較**:
*   **TCP**: `[App]` -> `[Send Buffer]` -> `[MSS Slicing]` -> `[IP Packet]`
*   **UDP**: `[App]` -> `[IP Packet]`

---
## 3. The Ops Nightmare: Path MTU Discovery (PMTUD)

這是 SRE 最常遇到的鬼故事：**「Ping 得到的，但網頁打不開 (或是卡在一半)。」**

### 3.1 The Scenario (黑洞成因)
1.  Server (MTU 1500) 發送一個 1500 bytes 的大包給 Client。
2.  路徑中間經過了一個 **VPN Tunnel** 或 **老舊路由器**，它的 MTU 只有 **1400 bytes**。
3.  Router 收到 1500 的包，過不去。且 TCP header 設定了 `DF (Don't Fragment)` (現代預設)。
4.  Router **丟棄 (Drop)** 封包，並回傳一個 ICMP 訊息：「太大了 (`Type 3 Code 4: Fragmentation Needed`)，請切小一點」。
5.  **悲劇發生**: Server 前面的 **防火牆** 把 ICMP 視為攻擊，**擋掉了**。
6.  **結果**: Server 永遠不知道封包被丟了，只是不斷重傳 1500 bytes；Client 永遠收不到。連線 **Hang** 住。

### 3.2 The Fix (解法)
*   **MSS Clamping**: 在 Router/Firewall 上強制修改 TCP Handshake 的 MSS 值。
    *   Config: `iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu`。
    *   這招會讓 Server 以為 Client 只能收小包，主動切小，避開 MTU 黑洞。

---

## 4. The Journey (Routing & Hops)

當封包被切好 (Fragmented) 並設好大小後，它開始了旅程。

### 4.1 Hop-by-Hop (接力賽)
封包不會直接飛到目的地，而是像 **接力賽跑**：
1.  **Local**: 網卡查 ARP Table，找到 Gateway (Router) 的 MAC Address，丟給它。
2.  **Internet**: Router 拆開 Frame，看 IP。
    *   TTL (Time To Live) 減 1。如果 TTL=0，丟棄 (防止無窮迴圈)。
    *   查 Routing Table (BGP)，決定下一站。
    *   重新封裝 MAC，丟給下一站。
3.  **Repeat**: 這個過程重複 15-20 次。

### 4.2 Queueing & Drop (塞車與車禍)
路由器內部的 Buffer 是有限的。
*   **Tail Drop**: 當 Buffer 滿了，新來的封包直接丟掉。
*   **AQM (RED)**: 為了避免 TCP 全域同步 (Global Synchronization)，路由器會隨機丟棄一些封包，強迫 TCP 提早降速。

這就是為什麼在 **Book 7.1** 我們強烈建議使用 **AWS Backbone**。因為在公網上，您的封包隨時可能因為某個不知名的路由器 Buffer 滿了而被無情丟棄，導致 **TCP Head-of-Line Blocking** (所有後面的封包都要停下來等那個掉失的封包重傳)。

---

## 5. Summary
這一章我們看到了 **物理層** 的殘酷：
1.  **MTU 1500** 限制了我們一次能傳送的大小，導致 16KB 的 TLS Record 必須被切成 11-12 個碎片。
2.  只要其中 **任何一個碎片** 在路上被丟棄 (Drop)，整個 16KB 的解密就會卡住 (Head-of-Line Blocking)。
3.  **Path MTU Discovery** 的失敗會導致連線莫名卡死。
4.  **Jumbo Frames** 是內網加速的神器，但出外網要小心。

既然網路如此不可靠 (Drop, Reorder, Jitter)，那我們的程式為什麼還能讀到完整的 "Hello"？
這全靠 **TCP** 在背後默默承受了一切。

**Next: Book 7.3 - Reliability (可靠性的代價)**
我們將探討 TCP 如何透過 **Sequence Number**、**ACK** 和 **Sliding Window** 來把這個支離破碎的物理世界，拼湊成一個完美的邏輯串流。
以及，為什麼 **CLOSE_WAIT** 和 **TIME_WAIT** 會成為伺服器的夢靨。
