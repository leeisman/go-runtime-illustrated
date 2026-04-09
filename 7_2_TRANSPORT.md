# Book 7.2: The Transport (傳輸層：TCP 與 UDP)

在 L3 (IP) 層，我們只能把封包從 A 送到 B。
但在 **L4 (Transport)** 層，我們要解決更困難的問題：**「如何保證資料完整、有序且快？」**

這一章我們深入探討網路世界的兩大主角：**TCP** 與 **UDP**。

---

## 1. The Strategy: Stream vs Datagram

這是你在寫程式時的第一個選擇：`net.Dial("tcp")` 還是 `net.Dial("udp")`？
這兩個字串的背後，對應著兩個截然不同的 **Kernel Syscall**：

*   **TCP**: `syscall.Socket(AF_INET, SOCK_STREAM, 0)`
    *   關鍵字是 **STREAM (串流)**。
    *   告訴 OS：這是一條水管，你要幫我負責順序、重組、流量控制。
*   **UDP**: `syscall.Socket(AF_INET, SOCK_DGRAM, 0)`
    *   關鍵字是 **DGRAM (数据报/信件)**。
    *   告訴 OS：這是一封一封的信，我給你什麼你就送什麼，**不需要 Nagle，不需要等待，現在就送！**

### 1.1 TCP (The Stream / 串流)
*   **哲學**: 「我保證送到，而且順序正確。」
*   **抽象**: **Byte Stream (水管)**。
    *   沒有封包邊界。你 Write 兩次 "Hello", "World"，對方可能 Read 一次 "HelloWorld"。
    *   **Buffering (緩衝機制)**:
        *   **MSS (1460 bytes) 怎麼來的？**
            *   這是物理限制。Ethernet 規範的 **MTU (Maximum Transmission Unit)** 通常是 **1500 bytes**。
            *   扣除 **IP Header (20 bytes)** 和 **TCP Header (20 bytes)**，剩下能裝資料的空間就是 **1460 bytes**。
        *   **發送時機 (Nagle's Algorithm)**:
            *   Kernel 為了效率，預設不會讓你「寫 1 byte 就送 1 個封包」(避免 40 bytes Header 只載 1 byte Data 的浪費)。
            *   **規則**: 
                1.  **湊滿**: 等到 Buffer 累積滿 **MSS (1460)** 才發送。
                2.  **清空**: 或者，如果當前網路沒有「未確認的封包」(In-Flight Data 已被 ACK)，就直接發送。
            *   (這意味著：如果網路很慢 ACK 回來慢，你的小封包就會被卡住，直到湊滿 1460 才會送出。這就是 **TCP_NODELAY** 要解決的問題)。
        *   **Go 語言冷知識**:
            *   Go 的 `net` package 預設 **開啟 `TCP_NODELAY`** (即 **關閉 Nagle**)。
            *   Go 認為在微服務架構下，**低延遲 (Low Latency)** 比省那一點點頻寬更重要。這避免了在 RPC 場景下發生莫名其妙的 200ms 延遲。

*   **代價**:
    *   **Head-of-Line Blocking**: 前面封包丟了，後面的就算到了也不能給 App，必須等重傳。
    *   **延遲**: 握手、慢啟動、重傳都會增加 Latency。

### 1.2 UDP (The Datagram / 封包)
*   **哲學**: 「我射後不理，越快越好。」
*   **抽象**: **Message (信件)**。
    *   保留邊界。你 Write 一次，網路上就跑一顆包。
    *   **No Buffering**: 直接封裝成 IP 封包丟給網卡。
*   **優勢**:
    *   **即時性**: 適合語音、視訊、FPS 遊戲。掉包就掉包，不用等。
    *   **簡單**: 支援廣播 (Broadcast) 和群播 (Multicast)。

---

## 2. The Connection (連線的解剖)

TCP 是 **Stateful** 的，UDP 是 **Stateless** 的。

### 2.1 連線的物理真相
*   **沒有線**: 網路上沒有實體線路連著雙方。
*   **State in RAM**: 所謂 `ESTABLISHED`，只是雙方 Kernel 在記憶體裡各開一個 Struct (`TCB`)，記錄對方的 IP、Port 和目前的 Sequence Number。

### 2.2 The 3-Way Handshake (建立連線)
為了同步這場「集體幻覺」，TCP 需要三次握手：
1.  **SYN (Seq=X)**: "我是 A，我從第 X 頁開始講。"
2.  **SYN-ACK (Seq=Y, Ack=X+1)**: "收到。我是 B，我從第 Y 頁開始講。"
3.  **ACK (Ack=Y+1)**: "收到。連線建立。"

**為什麼要 3 次？**
為了防止 **Ghost Packets (歷史殘存封包)** 讓 Server 誤判建立連線而浪費記憶體。

### 2.3 The 4-Way Wave (斷開連線)
因為 TCP 是全雙工 (Full Duplex)，兩個方向要分開關閉。
*   A 發 `FIN` (Close) -> B 回 `ACK`。(A 閉嘴了，但 B 還能說話)。
*   B 發 `FIN` (Close) -> A 回 `ACK`。(雙方都閉嘴 -> `CLOSED`)。

---

## 3. The Reliability (可靠性機制)

TCP 如何在不可靠的 IP 網路上，創造出可靠的幻覺？

### 3.1 Sequence & ACK (封包拆解與確認的藝術)

要保證資料「絕對不掉」，最直覺的想法是給每封信編號：1號信、2號信。但 TCP 作為一個 **Byte Stream (位元組流)**，它編號的方式非常獨特：**TCP 是替「每一個 Byte」編號，而不是替封包編號。**

這麼做的最大好處是能完美對付 **Fragmentation (碎片化 / 封包拆分)**。
假設您在 Go 裡面呼叫了一次 `conn.Write( 4000 Bytes 的照片 )`。由於網卡與乙太網路的限制，一個封包最多只能塞 `MSS = 1460 Bytes` 的資料。
所以您的照片，會在 Kernel 裡面被無情地「切除」成三塊：
*   第 1 塊：Byte 0 ~ 1459
*   第 2 塊：Byte 1460 ~ 2919
*   第 3 塊：Byte 2920 ~ 3999

#### 情況 A：完美的傳輸與 ACK 回條
我們來看看這三個封包在網路上飛行的長相，以及 Server 怎麼回傳確認 (ACK)：

```text
發送方 (Client)                                          接收方 (Server)
──────────────────────────────────────────────────────────────────────
📦 發送第一塊 (長度 1460)
[ TCP 表頭: Seq=0 ] + [ 資料: Byte 0 ~ 1459 ]   ────────▶ 
                                                        (Server 收到了 0~1459)
                                                ◀──────── [回傳表頭: ACK=1460]
                                                  (含義：1460 之前的我都收到了，請給我 1460 開始的資料)

📦 發送第二塊 (長度 1460)
[ TCP 表頭: Seq=1460 ] + [ 資料: Byte 1460 ~ 2919 ]  ────▶ 
                                                        (Server 連續收到了及格資料)
                                                ◀──────── [回傳表頭: ACK=2920]

📦 發送第三塊 (長度 1080)
[ TCP 表頭: Seq=2920 ] + [ 資料: Byte 2920 ~ 3999 ]  ────▶ 
                                                        (Server 照片全部拼湊完畢！)
                                                ◀──────── [回傳表頭: ACK=4000]
```
*   **低開銷計算**: 雖然有 4000 個 Byte，但 TCP 表頭並不會塞 4000 個數字。表頭只會記錄這塊包裹的 **「起始號碼 (Seq)」**，然後 Kernel 會自動加上 payload 大小，推算出尾碼。
*   **預期心理 (ACK)**: Server 回傳的 `ACK=1460`，實質意義是它正在 **「期待」** 下一個字節從 1460 開始，這也是一種「擔保 1459 之前全數抵達」的機制。

#### 情況 B：碎片遺失與重傳 (為何必須是 Byte 編號？)
為何不單純用「第 1 號包、第 2 號包」作為表頭？請看這個混亂的網路情境：
第二個包裹 (Seq=1460) 在路上 **掉包了**。更不幸的是，準備重傳時，網路環境非常差，這一次 OS Kernel 決定把它切得更小塊 (例如一次 1000 Bytes) 來增加發送成功率！

```text
發送方 (Client)                                          接收方 (Server)
──────────────────────────────────────────────────────────────────────
(假設第一塊已送達，Server 正在等待 ACK=1460)

❌ 原本的第二塊丟失了 ( Seq=1460, 長度 1460 )
   (等待超時，Client 準備重傳)

⚠️ 網路環境變差，這一次 Kernel 只敢切小一點 (每次 1000 Bytes) 就發送

📦 碎片化重傳 - 拆包 2-1
[ TCP 表頭: Seq=1460 ] + [ 資料: Byte 1460 ~ 2459 ] ───▶
                                                        (Server 發現剛好接上缺口！)
                                                ◀─────── [回傳表頭: ACK=2460]

📦 碎片化重傳 - 拆包 2-2
[ TCP 表頭: Seq=2460 ] + [ 資料: Byte 2460 ~ 2919 ] ───▶
                                                        (原第二塊的缺口終於補齊！)
                                                ◀─────── [回傳表頭: ACK=2920]
```
這就是精華所在：如果當初我們記錄的是「第 2 號包」，當網路需要重新切分重傳大小時，根本無法標記所謂的「第 2.1 包」和「第 2.2 包」。
可是，因為 TCP 紀錄的是 **「這塊肉是這頭牛身上的第 1460 克到第 2459 克」**，所以不管中途被切得多碎，接收端的 Kernel 永遠能像拼積木一樣，透過 Sequence Number 完美無缺地把它拼回那 4000 Bytes 的原生照片！

這就是 TCP 身為「Byte Stream (位元組流)」能在不可靠環境中保證資料完整性的暴力與優雅。
*   **極限與解法 (PAWS)**:
    *   **問題**: Sequence Number 只有 32-bit，傳輸 4GB 資料後就會歸零 (Wrap Around)。
    *   **風險**: 在 10Gbps 高速網路上，幾秒鐘轉一圈。這時如果一個「3 秒前迷路的舊封包」突然到達，Sequence Number 可能剛好跟現在的新封包一樣，導致資料錯亂。
    *   **解法**: 現代 OS 預設開啟 **TCP Timestamps**。這就像在里程表旁加了日期，用時間戳記來識破並丟棄這些「殭屍封包」 (此機制稱為 PAWS)。
*   **重傳 (Retransmission)**: 如果發送方超時沒收到 ACK，就會視為掉包，重傳該段資料。

### 3.2 Flow Control (滑動窗口 & 背壓)
*   **問題**: 發送方 (Sender) 發太快，接收方 (Receiver) 處理不完怎麼辦？
    *   **場景**: 一萬個玩家同時灌爆 Server，Server 程式 (User Space) 來不及 `Read`，導致 Kernel 裡的 Buffer 被填滿。
*   **解法 (Backpressure / 背壓)**: 
    *   TCP Header 裡有一個 **Window Size** 欄位。
    *   接收方 (Receiver) 會隨時告訴發送方：「我的 Buffer 還剩下多少空間」。
*   **Zero Window (強制暫停)**: 
    *   當 Buffer 滿了，Receiver 回傳 `Window = 0`。
    *   **後果**: Sender 的 Kernel 收到後，會立刻**停止發送**任何資料 (即使 App 呼叫 `Write` 也會被 Block 住)。
    *   **意義**: 這是 TCP 最重要的自我保護機制。它讓 Server 有權力叫 Client 閉嘴，防止系統被流量淹沒而崩潰 (OOM)。

### 3.3 可靠性的代價與極限 (The Reality of "Reliable")
TCP 的「可靠」是指 **資料最終準確 (Accuracy)**，而不是 **時間準時 (Timeliness)**。在現實網路中，您會遇到以下問題：

1.  **Head-of-Line Blocking (隊頭阻塞)**:
    *   **現象**: 一個封包遺失 (1, **X**, 3, 4, 5)。
    *   **後果**: 即使 3, 4, 5 都到了 Kernel，TCP **也不能** 給 User App，必須等到 2 重傳回來。
    *   **影響**: App 感覺到巨大的 **Jitter (抖動)**。平時 Ping 10ms，突然卡住 200ms~1s (等待超時重傳)。這對即時遊戲或高頻交易是致命傷。

2.  **Ghost Connection (殭屍連線)**:
    *   **現象**: 網路線突然被拔掉 (或中間 Router 斷電)，雙方都沒機會發 FIN。
    *   **後果**: 對雙方 Kernel 來說，連線狀態依然是 `ESTABLISHED`。
    *   **OS 的遲鈍**:
        *   **Active (傳輸中)**: OS 必須重傳失敗約 15 次才會死心斷線 (**需耗時 15~30 分鐘**)。
        *   **Idle (閒置中)**: OS 預設要等 2 小時沒動靜才會發送 TCP Keepalive Probe (**需耗時 > 2 小時**)。
    *   **現代最佳解法 (Application Heartbeat)**:
        *   **原則**: 不要被動等待 OS，要主動出擊。
        *   **實作**: App 層自己每幾秒 (e.g., 2s) 傳送一個微小的 **心跳包 (Heartbeat)**。
        *   **邏輯**: 「只要連續 N 次 (e.g., 3次) 沒收到對方的回應，就視為斷線，直接 Close。」
        *   **效益**: 將發現斷線的時間從 「數十分鐘」 縮短到 **「幾秒鐘」**。這才是即時系統 (遊戲、IM、微服務) 的標準做法。

3.  **假性壅塞 (Wireless Loss)**:
    *   **現象**: Wi-Fi/4G 訊號不好偶爾掉包。
    *   **後果**: TCP 誤以為網路塞車 (Congestion Control)，自動把發送速度 **減半**。

4.  **總結：所謂「網路抖動 (Jitter)」的真相**:
    當我們說網路「不穩」或「抖動」時，TCP 底層正在上演這個惡性循環：
    1.  **掉包 (Packet Loss)**: 訊號干擾導致封包沒送到。
    2.  **卡頓 (Latency Spike)**: TCP 為了維持順序 (Order)，強制暫停給應用層資料 (HOL Blocking)，直到重傳成功。**這就是您感覺到的「頓一下」。**
    3.  **降速 (Throttling)**: **發送方 (Sender)** 的 OS Kernel 誤判這是因為網路塞車，於是啟動壅塞控制，**將發送速度減半**。(若是下載慢，就是 Server 降速；上傳慢，就是 Client 降速。**簡單說：誰講話沒人聽，誰就閉嘴**)。
    4.  **斷線 (Timeout)**: 若持續掉包超過極限 (約 15~30 分鐘)，OS 最終放棄，判定斷線。

### 3.4 The Evolution: QUIC (HTTP/3 的逆襲)
既然 TCP 有上述的先天缺陷 (Kernel 僵化、無線誤判、HOL Blocking)，Google 決定另起爐灶，發明了 **QUIC** (基於 **UDP**)。

*   **User Space TCP**: QUIC 骨子裡就是在 UDP 上面重新實作了一套「現代版的 TCP」。
    *   **解除封印 (User Space)**: 把壅塞控制 (Congestion Control) 從 Kernel 搬到 Application 層。這意味著可以隨時更新演算法 (如 **BBR**)，不用等幾年一次的 OS 升級。
    *   **聰明的判斷**: 新的演算法能識別「無線訊號雜訊」與「網路壅塞」的差別，不會因為 Wi-Fi 抖一下就瘋狂降速。
    *   **消除隊頭阻塞 (No HOL Blocking)**: 在 QUIC 裡，Stream A 掉包只會卡住 Stream A，不會影響 Stream B。但在 TCP 裡，一個封包掉就會卡住全世界。

(所以當您看到 Chrome 用 UDP 連線 YouTube 時，別驚訝，那是更先進的可靠傳輸協議。)

---

## 4. Summary
*   **UDP** 是赤裸的 IP，快但不可靠。
*   **TCP** 是精密的重型機械（握手、重傳、窗口），可靠但有代價。
*   **連線** 只是 Kernel 裡的記憶體狀態。

理解了 L4 的傳輸機制後，下一章 **Book 7.3**，我們將進入應用層 (L7)，看看這些 Byte Stream 如何變成我們熟悉的 **HTTP** 與 **Web 應用**。
