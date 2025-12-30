# Book 1.4: 現代網路模型 (The Modern Epoll Server)

在 **[Book 1.3: 阻塞式網路模型](1_3_BLOCKING_NET_IO.md)** 中，我們看到了傳統 Server 的極限：
因為採用了 **Blocking I/O**，唯一的館長 (Thread) 只要服務到一個沒資料的連線，就必須**原地睡覺等待**。這導致後面即便有 10,000 個客人想進來，也被這一個人完全卡死。

為了解決這個問題，Linux 在 2.6 版引入了 **Epoll** 機制，讓**一位館長**就能同時監控成千上萬個連線。相當於現代高效能伺服器 (Nginx, Redis, Go Netpoller) 的基石。
這一次，館長不再傻傻地等，而是僱用了一個強大的助手：**Epoll 監控中心**。

---

## 1. 第一樂章：佈局 (Setup)

**場景**：**303 包廂 (The Process)** 不再去門口罰站，而是坐在辦公桌前填寫一系列精密的申請單 (System Calls)。

### Step 1: 開餐廳 (The Listen Socket)
這一步驟與 **Book 1.3** 的「申請總機」完全一致。

*   **OS Layer (C System Calls)**:
    ```c
    // 1. 申請物件 -> 拿到 FD 3
    int listen_fd = socket(AF_INET, SOCK_STREAM, 0);
    // 2. 佔據地盤 -> Port 8080
    bind(listen_fd, &addr, sizeof(addr));
    // 3. 正式開張 -> 允許連線進入
    listen(listen_fd, 1024);
    ```
*   **Go Layer (High Level)**:
    ```go
    // 這行代碼 = socket() + bind() + listen()
    ln, err := net.Listen("tcp", ":8080") 
    ```
*   **與行政區的溝通**:
    *   **303**: 「我要申請一台電話 (FD 3) 並綁定 Port 8080。」
    *   **行政區**: 建立 Socket 物件，啟用 **SYN Queue** 與 **Accept Queue**。

### Step 2: 建立監控室 (Epoll Create)
**303 (Process)** 決定採用新策略，而不是自己盯哨。

*   **OS Layer (C System Calls)**:
    ```c
    // 申請一個 "Epoll 實體" (監控室)
    int epfd = epoll_create(1); 
    ```
*   **Go Layer (Runtime Source)**:
    這通常發生在 Go 程式啟動時，Runtime 自動執行：
    ```go
    // src/runtime/netpoll_epoll.go
    func netpollinit() {
        // Go 偷偷幫你蓋了監控室，並把鑰匙 (epfd) 藏在全局變數裡
        epfd = epollcreate1(1) 
    }
    ```
*   **與行政區的溝通**:
    *   **303**: 「我需要一個監控中心。」
    *   **行政區**: 「好了，這是監控室的鑰匙 **FD 4**。」(背後建立了一個空的 **紅黑樹** 和 **就緒清單**)。

### Step 3: 委託監控 (Epoll Ctl)
這一步是 **「訂閱通知」** 的關鍵。

*   **OS Layer (C System Calls)**:
    ```c
    struct epoll_event ev;
    ev.events = EPOLLIN;    // 監控目標：有動靜 (Readable)
    ev.data.fd = listen_fd; // 綁定身分：FD 3
    epoll_ctl(epfd, EPOLL_CTL_ADD, listen_fd, &ev);
    ```
*   **Go Layer (Runtime Source)**:
    `net.Listen()` 成功後，底層立刻呼叫：
    ```go
    // src/runtime/netpoll_epoll.go
    func netpollopen(fd uintptr, ...) {
        // 自動幫你填寫掛號單，把 FD 3 加入監控
        epollctl(epfd, _EPOLL_CTL_ADD, int32(fd), &ev)
    }
    ```
*   **與行政區的溝通**:
    *   **303**: 「我要訂閱 **FD 3** 的來電通知。」
    *   **303 解析**: 「FD 3 是號碼牌，背後是綁定 Port 8080 的 **Socket 物件**。一旦網卡收到信，信會進去該物件的信箱 (**Recv-Q**)，物件就會變 **Readable**。這時候請通知我。」
    *   **行政區**: 「收到。」
*   **行政區內部 (關鍵動作)**: 
    1.  **註冊 (RB-Tree)**: 把 FD 3 寫入 **FD 4 的紅黑樹** (監控名冊)。
    2.  **設定回調 (Set Callback)**: 在 FD 3 的 Socket 物件上安裝 **自動觸發裝置**。
    3.  **觸發機制**: 當 FD 3 變 Readable 時，裝置會自動把 "FD 3" 抄寫到 **FD 4 的就緒清單 (Ready List)** 上。(注意：只抄號碼，不搬運資料)。

---

## 2. 第二樂章：Reactor 迴圈 (The Loop)

現在，**303** 搬了張椅子坐在監控室 (FD 4) 門口。

### 1. 詢問 (Wait)
*   **OS Layer**: `epoll_wait(epfd, ...)`。
*   **Go Layer**: `runtime.netpoll` 呼叫 `epollwait`。
*   **狀態**: 303 睡在 **FD 4 門口**，等待門鈴響起。

### 2. 客人進場 (Administrative Workflow)
這是行政區最忙碌的時刻，發生在你完全不知情的時候。

*   **Step A: 握手 (Handshake)**
    *   網卡收到 `SYN` -> 回覆 `SYN-ACK` -> 收到 `ACK`。
    *   連線建立 (ESTABLISHED)。
    *   **(觀念澄清)：這整個過程是網卡與 FD 3 的直接互動，數據流不經過 FD 4。**
*   **Step B: 入隊 (Enqueue)**
    *   行政區建立內部 Socket 物件，塞入 **FD 3 的 Accept Queue**。
*   **Step C: 觸發 (Trigger)**
    *   Accept Queue 有資料 -> **FD 3 變 Readable**。
    *   **回調發動**: 觸發裝置自動按下 **FD 4 的門鈴**，並將 "FD 3" 寫入 **FD 4 的就緒清單**。

### 3. 處理 (Process)
*   **甦醒**: 303 被 FD 4 的門鈴叫醒，拿到清單：「FD 3 有事」。
*   **判斷**: 「FD 3 是我的 **總機 (Listen Socket)**，有事代表 **有新連線**。」
*   **行動**: 呼叫 `accept(3)`。
*   **結果**: 拿到新連線的 **FD 5**。

---

## 3. 第三樂章：擴展 (Scale)

拿到 FD 5 (新連線) 後，303 依然很聰明。

1.  **再次委託 (Register)**:
    *   303 立刻呼叫 `epoll_ctl` (透過 Go 的 `netpollopen`)，把 **FD 5** 也加入 **FD 4 的監控名單**。
2.  **Go 的魔法**:
    *   現在 FD 5 也在監控中了。
    *   當你呼叫 `conn.Read(buf)` 時，如果沒資料，Goroutine 就去睡覺。
    *   等到 FD 5 真的有資料 (Readable) -> 觸發 Callback -> FD 4 通知 -> 喚醒 Goroutine。

---

## 4. 第四樂章：核心機制解密 (The Mechanics)

為什麼這套機制能解決 C10K？

### 1. 紅黑樹的價值 (Management)
*   **問題**: 既然 Callback 會通知，為什麼還要紅黑樹？
*   **答案**: 為了 **管理 (新增/刪除)**。
*   **場景**: 當 FD 55688 斷線時，行政區需要從 100,000 個名單中找到它並移除 Callback。
*   **效率**: 紅黑樹玩的是 **「終極密碼 (Binary Search)」**。
    *   比 Root 大 -> 往右；比右子樹小 -> 往左。
    *   **O(log N)** vs **O(N)**：3 次比較 vs 5 萬次掃描。

### 2. 終極比較 (Epoll vs Select)
*   **Select (老師巡堂)**:
    *   老師 (Kernel) 必須主動問遍全班一萬個學生：「你想上廁所嗎？」(Polling)。
    *   **效率**: **O(N)**。人越多越慢。
*   **Epoll (學生舉手)**:
    *   老師睡覺。學生想上廁所自己 **舉手 (Callback)**。
    *   老師睜眼只看 **舉手名單 (Ready List)**。
    *   **效率**: **O(1)**。不管班上有多少人，老師只處理有動作的人。
3.  **到底什麼是 Blocking I/O？ (與 DMA 的關係)**
    *   **定義**: 所謂 **Blocking (阻塞)**，是指當你的程式呼叫 `read()` 時，如果 **DMA (硬體層)** 還沒把資料搬運完成。
    *   **後果**: OS 會把你的 **Thread 強制休眠 (Sleep)**，踢出 CPU。這段時間該 Thread 什麼事都不能做，完全停擺。
    *   **隱喻**: 就像你去郵局領包裹。
        *   **Blocking I/O**: 櫃台說「貨車 (DMA) 還沒到」，你被迫 **在櫃檯前罰站 (Thread Block)**，直到車來了才能走。
        *   **Epoll**: 你留了電話就走了。等貨車到了，郵局傳簡訊 **(Callback)** 通知你再來拿。
    *   **重點**: Blocking 就是「為了等硬體 (DMA)，浪費了軟體 (Thread) 的自由」。

---

## 5. 第五樂章：結語 (Conclusion)

你可能會問：「這不就只是讓你少開一點 Thread 嗎？跟 I/O 效能有什麼關係？」

**答對了！核心價值正是「少開 Thread」。**

1.  **為什麼少開 Thread = 高 I/O 效能？**
    *   回想 **[Book 1.2](1_2_THE_THREAD.md)**，我們學過 Thread 的 **Context Switch (換裝備)** 是非常昂貴的。
    *   在舊模式 (Blocking I/O) 下，為了服務 10,000 個連線，你被迫開 10,000 個 Thread。CPU 會把 90% 的時間花在「切換 Thread」，而不是「處理 I/O 資料」。
    *   **Epoll** 讓你用 **1 個 Thread** 就能處理 10,000 個連線的等待。省下的 CPU 資源，全部可以用來處理真正的業務邏輯。這就是效能提升的來源。

2.  **什麼是 I/O Multiplexing (多路復用)？**
    *   **你的疑問**: 「讀資料時不還是要個別去讀 FD 5, FD 6 嗎？」
    *   **解答**: 沒錯！讀資料 (Data Transfer) 確實是個別的。**「復用 (Multiplexing)」是指「等待 (Waiting)」這個動作。**
    *   **餐廳隱喻**:
        *   **沒有復用 (Blocking)**: 餐廳有 100 桌客人。你請了 **100 個服務生**，每個人只准站在一張桌子旁，死死盯著那桌客人有沒有要點餐。沒人舉手時，這 100 個人都在發呆 (浪費薪水/CPU)。
        *   **有復用 (Epoll)**: 你只請了 **1 個服務生** (Thread)。他站在高處同時由看這 100 桌。
        *   **關鍵**: 雖然客人舉手時，他還是要走過去那桌點餐 (個別處理 FD)，但他一個人就能 **「同時等待」** 100 桌的訊號。
    *   **定義**: 將 **10,000 個等待的需求**，合併 (復用) 給 **1 個 Thread** 來監控。這才是它的真諦。
    *   **一句話總結**: 沒錯！正如你所說：**「只要等一條線 (FD 4)，就能復用所有線 (FD 3, 5, 6...)」**。
    *   **(觀念澄清)**: 這裡講的 I/O 主要是指 **網路 I/O (Network I/O)**，也就是 Socket 的讀寫。雖然 Linux 裡「一切皆檔案」，但硬碟 (Disk) I/O 通常有其他處理方式 (如 Cached I/O)，而不完全依賴 Epoll。

---

## 附錄：Q&A (Global Epfd)

**Q: 如果我同時開了兩個 Server (Port 8080 和 9090)，會有兩個監控室嗎？**

**A: 只有一個。**

這背後的技術原因是：**在同一個 Process 內，File Descriptor (FD) 是全域唯一的 Key。**

1.  **唯一性**:
    *   就算你開了兩個 Server (Port 8080, 9090)，OS 會發給你不同的 FD 號碼 (例如 FD 3 和 FD 6)。
    *   這兩個號碼絕對不會重複。
2.  **共用基礎**:
    *   正因為 Key 是唯一的，所以我們可以放心地把所有連線 (不管是來自 8080 還是 9090) 通通丟進 **同一個紅黑樹** 裡。
    *   紅黑樹只需要用 **FD ID** 當作索引，就能精準區分每一個連線，不會打架。
3.  **Go Runtime**:
    *   所以 `epfd` 在 Go 裡是一個 **全局單例 (Global Singleton)**。
    *   Poller (M0) 只需要死守這唯一的一扇門，就能掌控全域。
