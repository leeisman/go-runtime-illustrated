# Book 7.1: The Network Map (網路地圖：IP, Port與DNS)

在深入研究封包的原子結構之前，我們先在 **Application Layer (應用層)** 俯瞰整個網路世界。
作為後端工程師，我們每天都在處理 `Request` 和 `Response`，但這中間到底發生了什麼？

---

## 1. 第一樂章：網路三大基石 (The Three Pillars)

如果把網路比喻成現實世界的物流系統，我們需要三個核心要素：

### 1.1 IP Address (唯一的門牌號碼)
*   **概念**: 網路上每一台機器的身份證。
*   **IPv4 結構**: 由 4 個 Byte 組成 (e.g., `31.10.12.9`)。
    *   **總數量**: `2^32` ≈ **43 億個**。
    *   **現狀**: 早在 2011 年就發完了。現在我們能上網，全靠 **NAT** 技術撐著。

#### 1.1.1 The Routing (怎麼找到 31.10.12.9？)
IP 地址的設計是 **「層級式 (Hierarchical)」** 的，就像現實地址 (國家 -> 城市 -> 街道)。這保證了路由器不需要記住 43 億個地址，只需要記住「方向」。

1.  **Step 1 (洲際)**: 全球核心路由器看前碼 `31`，知道這是屬於 **歐洲區 (RIPE)** 管轄，於是把封包丟往歐洲。
2.  **Step 2 (ISP)**: 歐洲路由器看 `31.10`，知道這被分配給了 **英國 Vodafone**，於是丟進 Vodafone 的骨幹網。
3.  **Step 3 (機房)**: Vodafone 內部路由器看 `31.10.12`，知道這在 **倫敦某個機房**。
4.  **Step 4 (Server)**: 機房交換機最後透過 ARP 協議，精確找到插著 `31.10.12.9` 的那台機器。

#### 1.1.2 NAT & Private IP (公司的省錢魔法)
您可能會問：「公司有 1000 個微服務，難道要買 1000 個公網 IP？」(公網 IP 很貴，一個可能要幾美金/月)。
**當然不是。** 我們使用 **Private IP (私有 IP)** 與 **NAT (網路地址轉換)**：

*   **Public IP (公網)**: 整個公司的 Router 對外只有一個 IP `31.10.12.9`。
*   **Private IP (內網)**: 公司內部的 1000 台機器使用 `192.168.x.x` 或 `10.x.x.x`。這些是 **保留區段**，在公網上是無效的，但在內網可以自由使用。
*   **NAT 魔法**:
    *   當內網機器 A (`192.168.1.5`) 要訪問 Google 時。
    *   Router 會攔截封包，把 **Source IP** 從 `192.168.1.5` 修改為公網 IP `31.10.12.9`，並記下一筆對應關係 表。
    *   Google 回傳給 `31.10.12.9`。
    *   Router 查表，把封包改回 `192.168.1.5`，丟給機器 A。
    *   **結果**: 1000 台機器共用 1 個公網 IP 上網。

### 1.2 Port (分機號碼)
*   **概念**: 既然 IP 找到了機器 (大樓)，那 **Port** 就是用來找 **「是哪個程式在聽電話」** (分機)。
*   **範圍**: 0 ~ 65535。
*   **知名 Port**:
    *   `80`: HTTP (Web Service)
    *   `443`: HTTPS (Secure Web)
    *   `22`: SSH (Remote Login)
    *   `6379`: Redis
    *   `3306`: MySQL
*   **Ephemeral Ports (短暫 Port)**: 當 Client 發起連線時，OS 會隨機分配一個 Port (e.g., 54321)給 Client 用，這樣 Server 回話時才知道回給誰。

### 1.3 DNS (全球電話簿系統)
*   **問題**: 人類記不住 `1.2.3.4`，只記得 `google.com`。
*   **角色**:
    *   **Localhost**: `/etc/hosts` (最優先，手動設定)。
    *   **Resolver (遞迴解析器)**: 這是您連網的第一站。一般用戶預設使用 ISP (中華電信/Comcast) 的 DNS；工程師喜歡手動設為 `8.8.8.8` (Google) 或 `1.1.1.1` (Cloudflare)。它們負責「跑腿」去幫您問出答案。
    *   **Authoritative (權威主機)**: 真正持有該 Domain 資料的主機 (e.g. AWS Route53, GoDaddy)。

#### 1.3.1 The Resolution Flow (解析流程)
當您查詢 `api.mygame.com` 時，如果這是一個新的請求：
1.  **Cache Check**: 先查 Browser Cache -> OS Cache -> Router Cache。如果有，直接回傳 (0ms)。
2.  **Ask Resolver**: 電腦問 **8.8.8.8**：「你知道 `api.mygame.com` 的 IP 嗎？」(8.8.8.8 說我不知道，但我幫你問)。
3.  **Root Layer**: 8.8.8.8 去問 **Root Server**：「`.com` 是誰管的？」
4.  **TLD Layer**: 8.8.8.8 去問 **.com Server**：「`mygame.com` 是誰管的？」
5.  **Authoritative Layer**: 8.8.8.8 去問 **mygame.com 的權威主機** (e.g. AWS Route53)：「`api` 的 IP 是幾號？」
6.  **Answer**: Route53 回答 `1.2.3.4`。8.8.8.8 把結果傳回給您的電腦，並 **快取 (Cache)** 起來，方便下一個人問的時候直接秒回。

---

## 2. 第二樂章：主從架構 (Client-Server Model)

網路通訊最經典的模式就是 **Client (發起者)** 與 **Server (監聽者)**。

### 2.1 The Flow (一次請求的生命週期)
當您在瀏覽器輸入 `https://api.mygame.com/login` 時：

1.  **Resolve (查號)**: Browser 問 DNS：「`api.mygame.com` 的 IP 是多少？」答：「`1.2.3.4`」。
2.  **Connect (撥號)**: Browser 向 `1.2.3.4` 的 `443 Port` (HTTPS) 發起連線請求。
    *   這裡會發生神奇的 **TCP Handshake** (詳見下方 2.2 節)。
3.  **Request (說話)**: Browser 發送 HTTP 封包：「`POST /login`」。
4.  **Process (思考)**: Server (您的 Go 程式) 收到請求，查 DB，驗證密碼。
5.  **Response (回話)**: Server 回傳：「`200 OK, Token: xyz`」。
6.  **Close (掛斷)**: 雙方互道晚安，斷開連線 (或者保持 Keep-Alive)。

### 2.2 The Complete Handshake Journey (TCP + TLS)
現在幾乎所有網站都是 HTTPS。這意味著在瀏覽器發出請求前，必須先闖過兩道關卡：

#### Phase 1: TCP Handshake (Kernel Space - 全自動) - 1 RTT
這一步全程由 **OS Kernel (TCP Stack)** 自動完成，Application (如 Nginx) 完全不用插手。
1.  **SYN**: Client 喊話：「我想連線，起始序號 (ISN) 是 100」。
2.  **SYN-ACK**: Kernel 自動確認並回覆：「收到 100！我也想連線，我的 ISN 是 500」。
3.  **ACK**: Client 確認：「收到 500！連線成立」。(連線進入 `ESTABLISHED`)
> **Why 3-way?**: 為了防止網路上遲到的「殭屍封包」欺騙 Server，導致資源浪費。

#### Phase 2: TLS Handshake (User Space - Nginx/App) - 2 RTT
當 TCP 連線建立後，Kernel 才會把 Socket 交給 **Nginx**。這時才開始由 Application 介入處理加密：
1.  **Client Hello**: 「哈囉，我支援 TLS 1.3，我有這些加密算法...」
2.  **Server Hello + Certificate**: 「好，這是我的 **憑證** (裡面包含 **公鑰 Public Key**)。」
    > **Nginx Config**: 讀取 `ssl_certificate` 發送給 Client。
3.  **Client Response (Client 行為)**:
    *   **驗證 (Verify)**: Client 檢查這張憑證的 **簽發者 (Issuer)** (例如 DigiCert) 是否在自己電腦的 **「受信任根憑證列表 (Root CA Store)」** 裡。如果簽名驗證通過且沒過期，就信任它。
    *   **生成 Secret**: Client 生成一個 **隨機數 (Secret)**。
    *   **加密**: Client 用憑證裡的 **公鑰** 把這個 Secret **加密**，然後傳回給 Server。
4.  **Key Exchange (Server 解密)**:
    *   Nginx 收到這包加密訊息。
    *   Nginx 拿出藏在硬碟裡的 **私鑰** (`ssl_certificate_key`) 進行 **解密**，成功拿到 Secret。
    *   **結果**: 雙方現在都擁有了同一個 Secret，用它來生成最終通話用的 **Session Key**。
    > **為什麼不直接用公私鑰加密資料？**
    > 因為 **非對稱加密 (RSA/ECC)** 的數學運算太複雜，速度極慢 (比對稱加密慢 100~1000 倍)。如果用它來加密 1GB 影片，Server CPU 會直接燒掉。
    > 所以 TLS 採用 **「混合加密 (Hybrid)」** 策略：用昂貴的公私鑰 **只做一次** 鑰匙交換，之後的海量資料全用極速的 **對稱加密 (AES)** 處理。

> **實戰範例 (nginx.conf)**:
> ```nginx
> server {
>     listen 443 ssl;
>     server_name api.mygame.com;
> 
>     # 步驟 2: 把這張紙 (含公鑰) 給 Client 看
>     ssl_certificate     /etc/nginx/certs/mygame.crt;
>     
>     # 步驟 4: 用這把鑰匙 (私鑰) 解開 Client 傳回來的秘密
>     ssl_certificate_key /etc/nginx/certs/mygame.key;
> 
>     ssl_protocols       TLSv1.2 TLSv1.3;
> }
> ```

#### Phase 3: HTTP Request (AES 加密傳輸)
現在，雙方都有了那把 Session Key (假設協商結果是 AES-256-GCM)，真正的資料傳輸開始了。

1.  **切片 (Segmentation)**:
    *   Browser 要傳送 `POST /login` (內含 JSON Body)。
    *   TLS 層會先將資料切成一個個 **TLS Records** (每個 Record 上限 16KB)。
2.  **加密 (Encryption with AES-GCM)**:
    *   對於每一個 Record，Browser 會生成一個獨一無二的 **IV (隨機向量)**。
    *   運算公式：`CipherText, Tag = AES_GCM_Encrypt(SessionKey, IV, PlainText)`
    *   **產出**: 最終送出的封包結構是 **`[TLS Header] [IV] [密文 CipherText] [GCM AuthTag]`**。
    *   **GCM AuthTag (防偽封條)**: 這是關鍵。如果有駭客在路中間偷改了密文的一個 Bit，Nginx 在驗證 Tag 時就會失敗並立刻斷線。
3.  **傳輸 (Transmission)**:
    *   這串看不懂的 Bytes 被丟進 TCP Socket 出發。
4.  **解密 (Decryption @ Nginx)**:
    *   Nginx 收到封包，取出 IV 和密文。
    *   使用手上的 **Session Key** 進行 `AES_GCM_Decrypt`。
    *   如果 Tag 驗證通過，Nginx 就還原出明文 `POST /login`，並將其透過 HTTP (明文) 轉發給後端的 Go Server。

> **效能迷思: 每個 Byte 都要解密，會慢嗎？**
> *   **AES-NI (硬體加速)**: 現代 CPU (Intel/AMD) 都有專門的硬體指令集來跑 AES，速度極快 (GB/s)，對 CPU 的負擔其實很小。
> *   **真正的隱憂 (Head-of-Line Blocking)**:
>     *   TLS Record 預設大小是 **16KB**。
>     *   但乙太網路的 **MTU (最大傳輸單元)** 只有 **1500 Bytes**。
>     *   這意味著 **1 個 TLS Record 會被切成 11 個 TCP 封包**。
>     *   **後果**: 只要這 11 個小包裡面 **掉 1 個**，Nginx 就無法解密整個 Record，必須卡住等待重傳。這就是為什麼網路稍微不穩，網頁就會「轉圈圈」的原因 (也是 HTTP/3 為何要拋棄 TCP 改用 UDP 的主因)。

> **為什麼有了 Key 還需要 IV (隨機向量)？**
> 如果只用 Key 加密，**「相同的明文」會產生「相同的密文」**。
> *   駭客若看到您傳送了兩次一模一樣長度的亂碼，就能猜到：「喔，他肯定又按了一次登入」。
> *   **加上 IV (每次隨機)**: 即使您傳送一模一樣的 `POST /login`，因為每次 IV 不同，加密出來的亂碼 (Ciphertext) 就會長得完全不一樣。這防止了 **模式分析 (Pattern Analysis)**。

**代價總結**: 建立一個 HTTPS 連線至少需要 **3 個 RTT** (1 TCP + 2 TLS)。這就是為什麼 **Keep-Alive (長連接)** 對效能至關重要，因為它能省下這幾百毫秒的開銷。

> **安全性總結: 這個會被破解嗎？**
> *   **暴力破解 (Impossible)**: AES-256 有 `2^256` 種組合。即便用全地球的運算力，算到宇宙毀滅也算不出來。
> *   **唯一弱點 (Human Error)**: 通常是 **程式寫爛了** (例如不小心使用了固定的 IV，導致 XOR 被逆推) 或是 **私鑰被內鬼偷走**。只要 Nginx 版本有更新且私鑰保管得當，現階段是絕對安全的。

---

## 3. 第三樂章：HTTPS 效能調校與架構 (Performance Tuning & Architecture)

既然知道了 HTTPS 的層層關卡，身為架構師，我們不需要管 `net.Dial` 怎麼寫 (那是 lib 做的)，我們需要關注的是 **「HTTPS 對系統吞吐量的衝擊」** 以及 **「關鍵調校參數」**。

### 3.1 The Cost of Crypto (解密對吞吐量的影響)
很多人認為 HTTPS 慢是因為「加密運算」。這只對了一半。
*   **CPU (運算)**: 有 **AES-NI** 硬體加速，AES-GCM 的運算速度可達 **數 GB/s**。這通常 **不是瓶頸**。
*   **Memory Copy & Context Switch (真正瓶頸)**:
    *   傳統流程: 硬碟 -> Kernel Buffer -> **Copy to Nginx (User Space)** -> OpenSSL 加密 -> **Copy back to Kernel** -> 網卡。
    *   這中間的 **資料來回搬運 (User-Kernel Copy)** 才是拖慢大檔案傳輸的主因。

### 3.2 Deep Dive: kTLS (Kernel TLS)
kTLS 的核心思想是 **「老闆 (Nginx) 只談生意，苦力 (Kernel) 負責搬磚」**。

1.  **Workflow (運作原理)**:
    *   **Handshake (腦袋)**: 依然由 Nginx (User Space) 負責握手運算。
    *   **Offload (下放)**: 握手完成拿到 Key 後，Nginx 透過 `setsockopt(fd, SOL_TLS, ...)` 系統呼叫，將 **Session Key** 和 **IV** 注入到 Kernel Socket 中。
    *   **Encryption (苦力)**: 之後 Nginx 只要呼叫 `sendfile()`。Kernel 讀取硬碟檔案後，直接在 Kernel 內進行 AES 加密、打包 TLS Record，然後直通網卡。
    *   **結果**: 資料 **從未進入過 Nginx (User Space)**，實現了真正的 Zero-Copy HTTPS (針對靜態檔案)。

2.  **How to Enable (實戰設定)**:
    *   **OS 層級**: 確保載入模組 `modprobe tls` (Linux 4.13+)。
    *   **Nginx 層級**: 在 `nginx.conf` 加入 `ssl_conf_command Options KTLS;` 並搭配 `sendfile on;`。

### 3.3 Nginx Tuning: Buffer & Cache (關鍵參數)
在 Nginx (`nginx.conf`) 中，這兩個參數直接決定了使用者的體驗：

#### A. `ssl_buffer_size` (16KB 問題的控制閥)
這就是控制 TLS Record 大小的參數 (預設 **16k**)。
*   **設為 4k (Low Latency 模式)**: 
    *   **適合**: **Web 前端 (HTML/CSS)** 或 **不穩定的公網環境 (Mobile)**。
    *   **原因**: 瀏覽器可以 **串流渲染 (Streaming Render)**。User 收到前 4k 的 HTML 就能看到畫面 Header，體感極快。
*   **設為 16k (High Throughput 模式)**: 
    *   **適合**: **後端 API (JSON)** 或 **大檔案下載**。
    *   **原因**: 一般 JSON Client (`await res.json()`) 必須等到 **「收到完整 Body」** 才能 Parse。既然 User 都要等最後一刻，那切碎就沒意義了，不如用 16k 減少 Header 開銷，下載速度反而更快。
*   **建議**: 
    *   **Public Web**: **4k** (為了首屏體驗)。
    *   **Microservices (Internal)**: **16k** (內網穩定，追求吞吐)。

#### B. `ssl_session_cache` (節省握手 CPU)
*   **設定**: `ssl_session_cache shared:SSL:10m;` (10MB 記憶體約可緩存 4 萬個連線)。
*   **原理**: 儲存握手協商好的 **Master Key (Session Secret)**。下次 User 帶著 Session ID 來，Server 查表命中後，直接跳過第 3、4 步 (憑證驗證與金鑰交換)，直接開始加密傳輸 (Resume Session)。
*   **滿了會怎樣？**: Nginx 使用 **LRU (Least Recently Used)** 算法。
    *   當 10MB 滿了，最久沒用的 Session 會被踢掉。
    *   **後果**: 被踢掉的 User 下次連線時，必須重新跑一次完整的握手 (Full Handshake)，除了稍微慢一點之外，**不會有任何錯誤**。

### 3.4 End-to-End Architecture: From Nginx to Go (完整路徑解密)
這是最經典的生產環境配置：**Nginx 負責 SSL/Gzip**，**Go 負責純粹業務**。

#### A. The Configuration (Standard Architecture)
```nginx
http {
    # 1. 開啟 kTLS (讓 Kernel 幫忙解密)
    ssl_conf_command Options KTLS;

    upstream go_backend {
        server 127.0.0.1:8080;
        # 簡而言之: 這是一個 **Connection Pool** 的大小。
        # 當請求處理完後，Nginx 不會切斷 TCP，而是把它放回這 32 個位置的池子裡。
        # **大小建議**: `公式 = (預期最高 QPS) / (Nginx Worker 數量)`。
        # 如果設太小 (e.g. 1萬 QPS 但只設 32)，Pool 瞬間就會滿，導致多餘的連線被強行關閉並重建，引發 CPU 飆高。
        keepalive 32;
    }

    server {
        listen 443 ssl;
        ssl_certificate     /etc/certs/api.crt;
        ssl_certificate_key /etc/certs/api.key;

        location / {
            # 2. 轉發給 Go (走明文 HTTP)
            proxy_pass http://go_backend;
            
            # 關鍵優化 (偷天換日):
            # Client 可能會傳 "Connection: close"，如果轉發給 Go，Go 就會乖乖斷線，毀了我們的 Connection Pool。
            # 所以這裡強制設為空，欺騙 Go 說「還要繼續連」，讓 Nginx 自己去處理 Client 的斷線，但保留後端連線。
            proxy_set_header Connection "";
            
            # 標準配備: 將 Client 真實 IP 塞進 Header 傳給 Go
            # (因為 Connection Header 只控制連線斷不斷，不包含 IP 資訊)
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
    }
}
```

#### B. Under the Hood (底層發生了什麼？)
當一個 `POST /login` 請求進來時，資料經歷了這趟旅程：
1.  **Ingress (Physical -> Kernel)**: 
    *   網卡收到 TCP 封包 -> **kTLS Decrypt** (Kernel) -> Rx Buffer 裡變回明文。
2.  **Nginx Logic (User Space)**: 
    *   Nginx 醒來呼叫 `read()` (Copy 1: Kernel -> User)。
    *   Nginx 解析 Header，決定轉發給 `go_backend`。
3.  **Egress to Go (Nginx -> Kernel -> Go)**: 
    *   Nginx 透過 Loopback 發起寫入 (Copy 2: User -> Kernel)。
4.  **Go Logic (User Space)**: 
    *   Go Netpoller 偵測到讀取事件，處理請求。

**架構師視角**: kTLS 雖然省去了 Step 2 的 **「AES 解密 CPU」**，但省不了 **「資料搬運」** (Kernel -> Nginx -> Kernel -> Go)。這兩次 Memory Copy 是反向代理架構無法避免的物理成本。

### 3.5 The Ultimate Solution: Cloudflare / CDN Architecture
如果您覺得 `ssl_buffer_size` 和 `kTLS` 很難調，這裡有一個 **「課金玩家」** 的終極解法：使用 CDN (如 Cloudflare)。

這也是目前網路架構的主流，因為它解決了物理距離的問題：

#### A. Edge Termination (就近握手 - 物理外掛)
*   **傳統架構 (Direct + Route53)**: 台灣 User -> 美國 Server。
    *   即使你用了 **Route53 (DNS)** 的 Latency Routing，它也只是告訴 User 去連那個美國 IP。
    *   **結果**: 握手依然要跑 3 個 RTT (假設 Ping 200ms -> 握手 600ms)。Route53 只是導航員，救不了物理距離。
*   **CDN 架構 (Proxy + Backbone)**: 台灣 User -> **CDN 台北節點** -> 美國 Origin。
    *   **握手 (Accelerated)**: User 只跟 **台北節點 (如 CloudFront)** 進行握手。RTT 只有 10ms，握手只要 30ms。
    *   **回源 (Optimized)**:
        *   **0-RTT**: 台北節點與美國 Origin 之間維持長連接池，不需要重新握手。
        *   **Private Fibers**: 走 AWS 自己的海底光纖 (高鐵)，避開公網 (公車) 的 15-20 個路由跳點與擁塞這才是 **Global Accelerator** 真正值錢的地方。


#### B. Protocol Optimization (自動優化 - 軟體外掛)
CDN 就像一個聰明的翻譯官，它能幫您做很多 Nginx 做不到(或很難做)的事：
*   **Dynamic Record Sizing (自動調整)**: CDN 會自動偵測 User 的網路狀況 (TCP Window, RTT)。剛連線時自動切成小包 (1KB) 加速首屏，連線穩了自動放大 (16KB)。您完全不需要煩惱 Nginx 配置。
*   **HTTP/3 (QUIC) 轉換**: 即使您的 Nginx/Go 只支援 HTTP/1.1，CDN 可以對外提供最新的 **HTTP/3 (UDP)** 服務，徹底解決 TCP Head-of-Line Blocking 問題，然後轉成 HTTP/1.1 回源給您的伺服器。

#### C. Bonus Features (基本功: 緩存與防護)
除了上述的動態加速，CDN 還附贈了兩個核心價值：
1.  **Static Caching (靜態緩存)**: 圖片、CSS、JS 等靜態資源會直接緩存在 Edge。User B 下載時直接由 Edge 吐回，**完全不消耗 Origin 的頻寬與 CPU**。這通常能節省 90% 的流量費用。
2.  **DDoS Protection (神盾)**: 因為 CDN 本身就是巨大的分散式系統，它能輕鬆吸收 Tbps 等級的攻擊流量 (L3/L4)，並透過 WAF 過濾惡意請求 (L7)。您的 Origin 只要設定「只允許 CDN IP 存取」，就等於躲在防護罩後面。



---

## 4. 第四樂章：架構師視角總結 (The Architect's View)
這一章我們從最基礎的「門牌號碼 (IP)」一路講到了「雲端架構 (CDN)」。身為架構師，這幾個核心觀念必須刻在腦海裡：

1.  **連線是昂貴的 (The Cost of Connection)**:
    *   建立一個 HTTPS 連線起步價就是 **3 個 RTT** (TCP + TLS)。
    *   **對策**: 死都要用 **Keep-Alive** (Nginx `keepalive` + 清除 Connection Header)，這是最划算的優化。
2.  **解密是硬核的 (The Cost of Crypto)**:
    *   AES 運算本身不是瓶頸 (有 AES-NI)，**Memory Copy** 才是真正的隱形殺手。
    *   **對策**: 開啟 **kTLS** + **sendfile**，或是直接把 TLS Termination 丟給 **CDN (Edge)** 去扛。
3.  **代理是有代價的 (The Cost of Proxy)**:
    *   Nginx 為了具備路由能力 (L7 Logic) 與保護後端 (Buffering)，必須付出 Memory Copy 的代價。
    *   **認知**: 在一般場景 (API/Web) 這是值得的；但在極限吞吐場景 (40Gbps+)，物理限制會教做人，這時請轉用 L4 LB (eBPF)。
4.  **參數是靈活的 (The Art of Tuning)**:
    *   **Web (HTML)**: `ssl_buffer_size 4k` (追求首屏秒開)。
    *   **API (JSON)**: `ssl_buffer_size 16k` (追求最大吞吐)。
5.  **全球化是必須的 (Going Global)**:
    *   **Route53**: 只是地圖 (DNS)，負責指路，救不了物理延遲。
    *   **CloudFront/GA**: 才是任意門 (Proxy)。透過 **Edge Termination** (就近握手 0-RTT) 與 **Private Backbone** (內網專線)，徹底解決跨國傳輸的延遲與掉包問題。

### Next: Book 7.2 - The Packet Journey (封包的物理旅程)
現在，Nginx 已經把資料加密好、切成了 TLS Record (邏輯層)。
但當這些 Record 離開 User Space，跳進 Kernel 的那一刻，它們將面臨物理世界最殘酷的限制：**MTU (1500 Bytes)**。

在下一章，我們將拿著顯微鏡，看著這 16KB 的資料如何被肢解成 11 個小碎片，穿上 IP 和 Ethernet 的層層外衣，在充滿鯊魚 (掉包) 與亂流 (擁塞) 的海底電纜中求生。
*   **Encapsulation**: 俄羅斯娃娃般的封裝結構 (L7 -> L2)。
*   **Fragmentation**: 為什麼切碎是萬惡之源？
*   **Routing**: 封包在網路上是如何被「踢」到目的地的？
