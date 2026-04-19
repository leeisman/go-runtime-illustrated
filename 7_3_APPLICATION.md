# Book 7.3: The Application (應用層：HTTP, TLS 與進化)

有了這條可靠的 TCP 水管後，我們要傳送什麼內容？
這一章探討 L7 協議的演進，看人類如何從簡單的文字傳輸 (**HTTP/1.1**)，進化到極致效能的 UDP 傳輸 (**HTTP/3**)。

---

## 1. 第一樂章：Web 協議的進化 (HTTP Evolution)

**【大前提：HTTP 本質依然是 TCP 連線】**
無論是 HTTP/1.1 還是 HTTP/2，它們的底層傳輸都是依託於同一條 **TCP 連線 (L4)**。
也就是說，保證封包不遺失、處理掉包重傳、保證所有 Bytes 抵達順序正確，這些「可靠性傳輸」的髒活，全是底層 OS 的 TCP 模組在負責的。
既然底層已經這麼完美地保證了可靠性，為什麼 HTTP 的版本還要不斷升級進化？因為即使底層很可靠，**「塞車」**的方式卻不一樣。

### 1.1 HTTP/1.1 (純文字的排隊地獄)
*   **特徵**: 纯文字協議 (Text-based)，例如 `GET / HTTP/1.1`，人類肉眼可讀。
*   **通訊模式**: 為了省去反覆建立連線的昂貴成本，HTTP/1.1 廣泛使用了 `Connection: keep-alive`，讓多個 Request 可以重複利用同一條 TCP 水管。
*   **缺陷原罪 (L7 隊頭阻塞 - Application HOL Blocking)**:
    *   **底層設計問題**: 因為 HTTP/1.1 原本就是純文字流，它的傳輸規範中**「沒有幫請求貼上 ID 標籤」**。雙方只能透過「嚴格遵守先後順序 (FIFO)」來對應答案。
    *   **阻塞過程**: 當你同時發送了 Req 1、Req 2。Server 在處理時，**必須**等 Res 1 的最後一個字全數回傳完畢，才能接著吐出 Res 2。如果 Req 1 只是卡在資料庫拉報表要算 10 秒，整條 TCP 水管就會被霸佔並空等 10 秒。這時就算 Req 2 早就準備好了也完全送不出去！這就是應用層級的隊頭阻塞。

### 1.2 HTTP/2 (切片、貼標籤與多路復用)
為了解決這蠢翻天的排隊問題，HTTP/2 對應用層資料進行了超級魔改：
*   **特徵**: 放棄人類可讀的純文字，改為 **二進位分幀 (Binary Framing)**。這對人類不直觀，但對機器解析極度高效。
*   **代表作 (gRPC)**: 知名微服務框架 **gRPC 就是強制建立在 HTTP/2 之上**。gRPC 的傳輸底層利用 HTTP/2 切片，而 Payload 內容又採用 Protobuf (也是二進位編碼而非 JSON 純文字)，達成了從頭到尾都是 Binary 的極致效能。
*   **多路復用 (Multiplexing)**:
    *   **底層設計**: HTTP/2 把每一個很大的 Request 或 Response，用斬骨刀剁成極小的「影格 (Frame)」，並**在每一個小碎片上強制烙印一個 Stream ID** (例如：此碎片隸屬 Stream 1)。
    *   **完美超車**: 既然每個碎片都有了自己的身分證，Client 與 Server 就能在**同一條 TCP 水管內**毫無忌憚地「混合發送」REQ 1、REQ 2 的碎片。Server 可以先回傳瞬間算好的 RES 2，Client 只要看著 ID 就能像拼圖一樣組裝起來。彼此完全不用互相等待！
*   **HPACK (標頭壓縮)**:
    *   在 HTTP/1.1 時代，你每次發 Request 都要重新帶一次幾百 Bytes 的純文字 Header (包含超長的 Cookie、User-Agent 等)，這非常浪費頻寬。
    *   HTTP/2 發明了 HPACK。Client 跟 Server 雙方會默默維護一本「字典」。如果這個 Header 之前傳過了，第二次傳送時只要給一個索引代號 (例如 `#62`) 即可！極大幅度節省了無謂的傳輸量。
*   **微服務架構的真正救星 (Microservice MVP)**:
    *   **HTTP/1.1 的悲劇**: 現代微服務通常是「扇出 (Fan-out)」架構。一筆用戶請求進來，Service A 可能要「並行發出 100 個 Request」去跟 Service B 要資料。如果用 HTTP/1.1，因為不能多路復用，Service A **必須向 OS 申請打開 100 條全裸的實體 TCP 連線**。這會瞬間榨乾系統的 File Descriptor，並引發海量的 Context Switch 與 TCP 握手開銷。
    *   **HTTP/2 的救贖**: 有了多路復用，不管 Service A 要並行發送 100 個還是 1000 個並發請求給 Service B，它**永遠只需要維持「1 條」實體 TCP 連線** (這條連線被稱為 Connection Pool 裡的單一長連接)。所有的併發請求全都在這條實體連線上被切片傳送。這就是為什麼當代微服務 (如 gRPC) 能夠支撐幾萬 QPS 的核心物理原因：**極致地壓榨單一 Socket 資源，絕不濫開連線。**
*   **前端 Web 的真實寫照 (AJAX 與 6 條連線極限)**:
    *   **HTTP/1.1 的作弊與極限**: 延續上述邏輯，當你在前端用 `AJAX` 或 `fetch` 同時發起 10 個非同步請求時。為了繞過排隊阻塞，瀏覽器只好在背後「偷偷跟 OS 申請打開多條實體 TCP 連線」。但為了保護伺服器不被 100 張圖片的網頁炸癱瘓，主流瀏覽器 (Chrome) 都強制規定：**「同一個網域 (Domain) 最多只能同時開 6 條 TCP 連線」**。所以你剩下的 4 個請求會被強制 Pending 扣留排隊，這就是前端有名的網路塞車瓶頸。(這也是舊時代為什麼要把圖片拆分到 `static1.xxx`, `static2.xxx` 不同子網域的根本原因：為了騙瀏覽器多開 6 條通訊管線)。
    *   **HTTP/2 的解放**: 一旦網頁伺服器升級到 HTTP/2，不管你的 AJAX 併發發出 10 個還是 100 個非同步請求，瀏覽器**只會打開 1 條實體的 TCP 連線**。所有的 100 個 Request 會瞬間被切成二進位碎片發射出去，完全不需要在瀏覽器端排隊 Pending。不僅伺服器的實體連線負載大幅降低，網頁的平行載入速度也迎來了本質上的飛躍。
*   **撞上真牆 (TCP HOL Blocking)**:
    *   成也 TCP，敗也 TCP。因為大家都在「同一條 TCP 物理連線」裡飆車。回顧 7.2 章，只要底層網卡掉了一個 TCP 封包，OS Kernel 為了保證「絕對有序」，會強制卡死整條通道等待重傳。
    *   **結論**: 這時你會發現，明明只是 Stream 1 的一塊小碎片遺失了，但底層 TCP 卻把整條馬路封鎖，導致後面無辜的 Stream 2 和 Stream 3 碎片全部一起陪葬！**(這也就是為什麼在 Wi-Fi 訊號差的地方，使用 HTTP/2 的體驗反而比開了好幾條連線的 HTTP/1.1 還要爛)**。

### 1.3 HTTP/3 & QUIC (The UDP Revolution)
*   **核心**: **拋棄 TCP，改用 UDP**。
*   **QUIC 協議**:
    *   在 UDP 之上自己實作了 Reliability (重傳) 和 Congestion Control。
    *   **無隊頭阻塞**: 每個 Stream 獨立。Stream A 掉包只會卡住 Stream A，Stream B 照常運作。
    *   **Connection Migration**: 切換網路 (Wi-Fi 轉 4G) 不用重連 (用 Connection ID 取代 IP:Port)。

---

## 2. 第二樂章：加密層 (TLS)

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

## 3. 第三樂章：即時與雙向 (WebSocket)

HTTP 是 "Request-Response" 模型，Server 不能主動說話。
WebSocket 解決了這個問題。

### 3.1 Upgrade Mechanism
*   **握手**: Client 發送一個特殊的 HTTP Request (`Upgrade: websocket`)。
*   **切換**: Server 同意後 (`101 Switching Protocols`)，這條 TCP 連線就脫離了 HTTP 規範，變成全雙工的 Raw Socket。
*   **應用**: 聊天室、股票報價、遊戲信令。

### 3.2 世紀對決：HTTP/2 vs WebSocket (遊戲架構選型)
在現代系統架構（尤其是高併發的遊戲伺服器）中，遇到 `HTTP/2` 和 `WebSocket` 時該選哪一個？勝負的唯一判斷標準是：**「伺服器需不需要『無預警地主動』狂敲客戶端？」**

*   **當你需要「全雙工主動廣播」 👉 唯一選擇 WebSocket**
    *   **場景**: 麻將對手出牌、拉霸機中獎全服跑馬燈、即時聊天室。
    *   **原因**: 不管 HTTP 升級到第幾代，它骨子裡始終是傲嬌的「一問一答 (Request-Response)」模型，Client 不發 Request，Server 絕對不會理你。但 WebSocket 握手後就是一條直接相連的雙向水管，Server 可以隨時無預警把資料塞給 Client，這對於即時互動的情境無可取代。
*   **當你需要「單純的查詢與操作」 👉 強烈建議 HTTP/2**
    *   **場景**: 查詢商城明細、打開背包看道具、帳號登入、甚至內部伺服器群互相拉取資料 (Server-to-Server 微服務通訊)。
    *   **原因**: 在 WebSocket 這種純水管裡，如果你丟了一個 `{action: "get_item"}`，你很難輕易分辨「下一秒流出來的資料」，到底是你查詢的背包道具結果？還是哪位路人剛好在此刻傳給你的一句聊天訊息？你必須手刻極度複雜的 `Msg_ID` 去做分流。但在 HTTP/2 裡，Request 一定完美對應著專屬的 Response，而且有著「多路復用」的無阻塞效能加持，絕不塞車，是開發這類邏輯的首選。
*   **業界主流：雙拼架構 (我全都要)**
    真實的大型遊戲公司（如 IGS 等）通常採用混合架構：
    1. **遊戲牌局 / 戰鬥大廳**: 專門拉一條 `WebSocket` 長連線，維持所有即時的雙向廣播。
    2. **周邊系統 / 微服務 / 登入**: 走 `HTTP/2` (REST 或 gRPC)，一問一答，拿完資料就乾淨結案。完美隔離並發揮兩種協議的所有優勢。

---

## 4. 第四樂章：總結 (Summary)
*   **HTTP/1.1**: 簡單，但有 L7 阻塞。
*   **HTTP/2**: 解決 L7 阻塞，但受限於 TCP L4 阻塞。
*   **HTTP/3**: 用 UDP 解決一切阻塞，是未來的趨勢。
*   **TLS**: 是必要的安全層，但要注意握手延遲 (RTT)。

有了高效的應用協議，接下來我們要面對現實世界的問題：**當流量太大時，系統會怎麼崩潰？**。
下一章 **Book 7.4** 進入運維視角 (Ops)。
