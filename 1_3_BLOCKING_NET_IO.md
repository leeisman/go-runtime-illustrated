# Book 1.3: 阻塞式網路模型 (Blocking Network I/O - The Poker Server)

在認識了圖書館的所有角色 (**[Book 1.1: 皇家圖書館](1_1_THE_LIBRARY.md)**) 之後，讓我們把鏡頭拉遠，觀看這部機器如何運作。
這是一場從 Server 啟動 (Startup) 到玩家下注 (Betting) 的完整硬體旅程。

---

### 第一樂章：開張大吉 (Startup: `net.Listen`)

1.  **Shell 的委託**：
    *   你站在 **202 包廂 (Shell Process)** 裡，對著終端機下令 `./poker-server`。
    *   202 包廂的館長按鈴呼叫行政區：「麻煩幫我開一個新房間 (303) 來執行這個程式」。

2.  **機器人搬運 (DMA)**：
    *   行政區啟動，但他不親自搬書，而是命令 **DMA (Direct Memory Access, 自動搬運機器人)**：「去把硬碟架上的 `poker-server` 執行檔，直接搬到 303 包廂的黑板上，搬完叫我」。(這期間 CPU 可以去忙別的)。

3.  **進駐**：
    *   DMA 搬完後，行政區指派一位新館長進駐 303 包廂。

4.  **平民預備 (User Init)**：
    *   新館長整理環境 (Go Runtime Init, GC Setup)，然後開始讀 `func main()`。

5.  **申請總機 (Bind & Listen)**：
    *   讀到 `net.Listen(":8080")`。館長停下手邊工作，準備好申請書，按鈴呼叫行政區。

    *   **物件實體化 (Allocation)**: 當你呼叫 `net.Listen` 時，行政區首先在 **Kernel Heap** 裡申請了一塊記憶體，建立了一個 **無名的 Socket 物件 (`struct sock`)**。這塊記憶體就是「收發大廳」的本體。
    *   **內部隊列**: 這個物件裡包含了兩個特殊的公文籃：
        1.  **SYN Queue (待審區)**：存放正在申請連線的 **「開戶申請書」(SYN 包裹)**。
        2.  **Accept Queue (已核准區)**：存放已經完成握手手續，等待你來領取的新信箱。
    *   **對外 (路由登記)**：
        *   **位置**: 行政區內部的 **「全域插座查找表 (Global Socket Table)」**。
        *   **權限**: 這是一張 **員工專用** 的公告。
        *   **動作**: 行政區在表上插上一張新卡片，內容包含：
            *   **Key (索引)**: `Port 8080`。
            *   **Value (內容)**: 指向 **剛剛建立的那個 Socket 物件** 的指標。
            *   **物件內含資訊** (SoftIRQ 查到後會讀取這些)：
                1.  **狀態 (`sk_state`)**: `TCP_LISTEN` (代表營業中，若非 Listen 則會拒絕連線)。
                2.  **收件地址**: 指向 **SYN Queue** 的記憶體位置 (告訴 SoftIRQ 信要放哪)。
                3.  **處理規則**: 定義了收到信後該執行什麼函式 (Callback)。
        *   **意義**：這是在告訴未來的 SoftIRQ 該如何工作。當 Port 8080 的信進來時，他就會依照這張卡片的指示行動。
    *   **對內 (配發鑰匙 - 建立關聯)**：
        *   **動作**: 行政區在你的 **「檔案描述符表 (FD Table)」** 插上一張卡片：
            *   **Index (你的編號)**: **3** (最小可用空號)。
            *   **Value (關聯)**: 指向 **同一個 Socket 物件**。
        *   **交付**: 行政區把號碼 **3** 作為 Syscall 的回傳值，正式交付給 **303 包廂**。
        *   **意義**：從此以後，303 包廂只要出示 "3 號"，行政區查表就知道是指向那個 Socket 物件。

7.  **結果**：
    *   303 包廂收到 FD 3 後，並不會直接拿著數字到處跑，而是把它包裝成一個 **Go 語言物件 `ln` (`net.Listener`)**。
    *   **比喻**: `ln` 就像是一個 **遙控器**，而 FD 3 是藏在裡面的 **通訊晶片**。
    *   **作用**: 這個遙控器只有一個主要按鈕：`Accept()`。
    *   **對應代碼**:
```go
// 1. 申請開台 -> 拿到遙控器 ln (裡面包著 FD 3)
ln, err := net.Listen("tcp", ":8080")

// 2. 開始接客 -> 按下遙控器按鈕，進入等待
conn, err := ln.Accept()
```

---

### 第二樂章：新用戶進場 (New User: `ln.Accept`)

1.  **無限迴圈 (The Daily Grind)**：
    *   館長進入 `for` 迴圈，準備受理業務。
    *   **迷思澄清**：雖然是無限迴圈，但这並不代表館長一直繞著桌子跑。這其實是一個 **「睡覺 -> 接客 -> 睡覺」** 的循環。
    *   **真相**：在沒有客人的時候，館長是趴在櫃台上睡覺的，完全不消耗體力 (CPU Usage = 0%)。

2.  **櫃台沒人 (The Blocking Wait)**：
    *   執行到 `ln.Accept()` (按鈕按下)。館長發現 **已核准區 (Accept Queue)** 空空如也。
    *   **動作**: 館長將自己的聯絡方式留在 Socket 的 **「等待清單 (Wait Queue)」** 上，告訴行政區：「有新用戶再叫我」。隨後，他將自己標記為 **睡眠狀態 (WAITING)**，主動讓出 CPU。
    *   (此時 CPU 去忙別的事了，303 包廂一片死寂)

3.  **背景審核 (OS 全權代理 - 3-Way Handshake)**：
    *   **驚人的事實**: 你的 Go 程式 (303 包廂) **完全沒有參與** 握手過程。你還在睡大覺。
    *   **觸發 (The Trigger)**：
        *   **硬體接收 (NIC)**: 遠方傳來的電訊號 (SYN) 抵達網卡。網卡利用 DMA 直接把資料寫入記憶體 (Ring Buffer)，然後按下 **中斷鈴 (Interrupt)**。
        *   **軟體接手 (SoftIRQ)**: 館長(CPU) 被鈴聲打斷，變身為制服模式 (SoftIRQ)，從 Ring Buffer 讀取這封信。
        *   **識別**: 解析發現信件寫著 `Dest: 8080`，查閱 Global Table 後，拿到了 **Socket 物件的記憶體指標** (也就是 303 包廂所謂的 FD 3 本體)。透過這個指標，SoftIRQ 就能直接把信件放入該物件的 SYN Queue。
    *   **執行握手**: 於是 SoftIRQ 依照 FD 3 的指示，開始處理這份申請書：
        
        *   **Step 1: 收到 SYN (申請書)**
            SoftIRQ 為它建立一份 **臨時檔案 (`struct request_sock`)** 放入 **該 Socket 物件內部的 SYN Queue (待審區)**，標記狀態為 `SYN_RECV`，並自動製作回信 **SYN-ACK**。
            *   **發送細節**: SoftIRQ 將這封回信放到 **TX Ring Buffer**，並按下網卡的門鈴 (Doorbell)。網卡收到通知後，啟動 **DMA** 將資料從記憶體搬走，並轉為電訊號發射出去。
            *   **Shell 視角**:
            ```bash
            $ netstat -an | grep 8080
            tcp  0  0  :::8080  1.2.3.4:5678  SYN_RECV
            # (該連線目前正躺在 SYN Queue 裡面)
            ```

        *   **Step 2: 收到 ACK (確認函)**
            SoftIRQ 找到那份臨時檔案，將其 **「轉正」** 為正式的 **Socket 物件**，狀態改為 `ESTABLISHED`。
            *   **Shell 視角**:
            ```bash
            $ netstat -an | grep 8080
            tcp  0  0  :::8080  1.2.3.4:5678  ESTABLISHED
            # (該連線已移入 Accept Queue，正在等待應用程式來領取)
            ```
    *   **審核通過**: 只有在握手徹底完成後，SoftIRQ 才會將這位新用戶的資料從待審區 (SYN Queue) 移入 **已核准區 (Accept Queue)**。此時雖然 303 還在睡覺，但在 OS 層面連線已經成立了。

4.  **叫號 (Notification & Reschedule)**：
    *   **觸發**: SoftIRQ 把文件放入已核准區後，順手查看了一下該 Socket 的 **等待清單 (Wait Queue)**，發現 303 包廂正在上面。
    *   **通知**: SoftIRQ 立刻通知 **行政區排班中心 (Scheduler)**：「貨到了，請把 303 叫醒。」
    *   **排班**: Scheduler 收到通知後，將 303 包廂的狀態由 **睡眠 (WAITING)** 切換為 **就緒 (RUNNABLE)**，並放入執行隊列。
    *   **回歸**: 過了一會兒，館長 (CPU) 真的走進了 303 包廂，繼續執行 `ln.Accept()` 剩下的工作。

5.  **分配專屬信箱 (Return FD 5)**：
    *   **甦醒 (Recall)**: 館長 (CPU) 恢復意識，此時他 **人還在行政區** (正在執行 `Accept` 系統呼叫的後半段)。
    *   **綁定 (Link)**: 他伸手進 **已核准區**，找到那個無名 Socket 物件 (**注意：物件本體留在行政區，不會被搬走**)。
    *   **註冊 (FD Allocation)**: 他在 303 的 **FD Table** 登記 **FD 5**，並建立一條指向該物件的連結。
    *   **歸隊 (Return)**: 辦完手續後，館長帶著 **數字 5** 離開行政區，回到 303 包廂 (User Space)，將其交給 Go 程式碼。
    *   **對應代碼**:
```go
// 這行代碼經歷了: Blocking -> SoftIRQ Handshake -> Wakeup -> FD Alloc
// conn 是一個新物件，裡面包著 FD 5
conn, err := ln.Accept()
```


---

### 第三樂章：協議升級 (Upgrade: `u.Upgrade`)

#### Step 1: 請求 (Data Arrival)
*   **NIC -> SoftIRQ**: 網卡收到玩家的 TCP 封包 (Payload 內含 "HTTP Upgrade Request")，利用 **DMA** 寫入 Ring Buffer 並觸發中斷。
*   **分派**: SoftIRQ 查表發現這封信屬於 **FD 5** (已建立連線)，於是將資料放入 FD 5 的 **Recv-Q**，並喚醒正在等待讀取的 303。
*   **讀取**: 303 醒來，透過 `read(5)` 將資料領回。
    *   **動作詳解**: 館長 (CPU) 執行 `read` 系統呼叫，親手將資料從 **Kernel Heap (Recv-Q)** 逐字複製到 **User Heap (Go Buffer)**。這是一個消耗 CPU 的體力活。
    *   **對應代碼**:
```go
// 1. 準備盤子 (User Heap)
buf := make([]byte, 1024)

// 2. 搬運 (CPU Copy: Kernel -> User)
// 資料真正從 OS 記憶體流向了 Go 程式變數
n, err := conn.Read(buf)
```

#### Step 2: 回應 (Reply)
*   **對應代碼**:
```go
resp := []byte("HTTP/1.1 101 Switching Protocols...")
// 搬運 (CPU Copy: User -> Kernel)
conn.Write(resp)
```
*   **動作詳解**:
    1.  **搬運**: 館長 (CPU) 將資料從 **User Heap** 複製到 **Kernel Heap (Send-Q)**。注意：只要複製成功，`Write` 就會返回 (資料此時還在電腦裡，尚未送出)。
    2.  **發射**: 隨後 Kernel 將資料放入 TX Ring，並按門鈴通知網卡。
    3.  **硬體**: 網卡啟動 **DMA** 將資料從 Kernel Memory 搬走並轉為電訊號發射。

#### Step 3: 變身 (Switch)
*   **真相**: **FD 5 還是原來那個 TCP Socket**，OS 完全不在乎你們傳什麼。
*   **改變**: 改變的是 **303 包廂的解析邏輯**。
    *   既然雙方已經講好 Upgrade，303 從此不再用 **HTTP Parser** (尋找 `\r\n`, 解析 Header)。
    *   改用 **WebSocket Parser** (讀取 Frame Header, Opcode, Length, Payload)。
*   **OSI 觀點**: 這一切都發生在 **Layer 7 (應用層)**。底下的 **Layer 4 (傳輸層)** 永遠是 TCP，完全沒有變動。
*   **意義**: 就像是同一條電話線，雙方約定好從現在開始「不講中文 (HTTP)，改講摩斯密碼 (WebSocket Binary)」。

---

### 第四樂章：資料輸入 (The I/O Trilogy)
這是不斷重複的日常，也是高效能伺服器的秘密核心。

**模式說明：標準阻塞式 I/O (Standard Blocking I/O)**
> ⚠️ **注意**：本章節演示的是作業系統最原始、最基礎的 **Blocking I/O** 行為。
> 在這個模式下，當沒有資料時，Process (303) 會被 OS 強制催眠。
> 這有助於你理解硬體與 OS 的互動基礎。至於 Go 語言如何透過 **Netpoller (Epoll)** 來「魔改」這個行為以避免睡眠，請參閱 **[Book 3.3: 網路掛號處](3_3_GO_NETPOLLER.md)**。

#### Step 1: 阻塞 (Wait - The Trap)
*   **動作**: 303 執行 `ws.ReadMessage()`，試圖讀取玩家的下一步動作。
*   **判斷**: **303** 拿著 **FD 5** (鑰匙) 向行政區詢問：「5 號信箱有信嗎？」行政區檢查對應的 **Kernel Recv-Q**，發現是空的。
*   **掛號**: 於是 303 **請求行政區** 將自己的聯絡方式留在該信箱的 **「到貨通知單 (Wait Queue)」** 上。
*   **結果**: 303 進入 **睡眠狀態 (WAITING)**，主動讓出 CPU。這就是為什麼 Server 能同時維持數萬個連線而不卡頓。
*   **Linux 視角**:
    *   **`prepare_to_wait()`**: 對應「掛號」動作，把自己加入 Wait Queue，狀態設為 `TASK_INTERRUPTIBLE`。
    *   **`schedule()`**: 對應「讓出」動作。呼叫此函式後，CPU 切換到其他 Process，你的程式碼停在這一行不動 (Block)。
*   **代碼**:
```go
// ws 內部封裝了 Connection (fd: 5)。
// 此函式底層會呼叫 syscall read(5, ...)，如果沒資料就 Block 住。
_, msg, err := ws.ReadMessage()
```

#### Step 2: 喚醒 (Wakeup - The Interrupt)
*   **觸發**: 玩家終於按下了 "All-in"，封包抵達 NIC -> DMA -> Ring Buffer -> SoftIRQ。
*   **通知**: SoftIRQ 把資料放入 **行政區的信箱 (Kernel Recv-Q)**，隨後查看「到貨通知單」，發現 303 正在睡覺，於是通知 Scheduler：「叫醒他」。
*   **排班**: Scheduler 將 303 的狀態標記為 **就緒 (RUNNABLE)**，並將其名牌重新掛回 **CPU 的待辦選秀名單 (Run Queue)**，等待下一次被選中執行。

#### Step 3: 搬運 (Copy - The Heavy Lifting)
*   **甦醒**: 館長 (CPU) 重新進入 303 包廂。
*   **搬運**: 館長親自將資料從 **行政區信箱 (Kernel Recv-Q)** 逐字複製到 **303 的私人緩衝區 (User Buffer, 通常在 Heap)**。
*   **代碼**:
```go
// 這行代碼可能等了很久 (Wait)，直到有資料並 Copy 完成才返回
// 優化提示: 為了避免 GC 壓力，實戰中通常會用 sync.Pool 來重用這個 buffer
// 內部動作: n, err := conn.Read(buf) -> CPU 把資料搬到 buf
_, msg, err := ws.ReadMessage()
```

---

### 第五樂章：資料輸出 (Output: `ws.WriteMessage`)

#### Step 1: 準備 (User Space)
*   303 計算完畢，產生回應資料 `{"result": "ok"}`。

#### Step 2: 寫入 (Copy to Kernel)
*   **動作**: 呼叫 `ws.WriteMessage()`。
*   **搬運**: 館長將資料從 User Buffer 複製到 **Socket Send Buffer (Send-Q)** (Kernel Heap)。
*   **返回**: 注意！只要複製進 Kernel 成功，Function 就會即刻返回 `nil` (此時資料還在電腦裡)。
*   **代碼**:
```go
// 只要 Copy 進 Kernel 就算成功，不代表對方收到
// ws 內部依然是使用同一個 Connection (fd: 5)
// 這次是把資料搬到 Kernel 的 Send-Q
// 底層: n, err := conn.Write(msg) -> write(5, ...)
ws.WriteMessage(websocket.TextMessage, []byte(`{"result": "ok"}`))
```

#### Step 3: 發送 (DMA Transmission)
*   **打包**: OS 擇機將 Send Buffer 資料封裝成 TCP 封包 (加上 Sequence Number 等 Header)。
*   **發射**: 將封包描述符放入 TX Ring，按門鈴通知網卡。網卡啟動 **DMA** 將資料搬走並發射。


