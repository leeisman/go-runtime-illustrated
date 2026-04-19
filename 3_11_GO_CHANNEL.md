# Book 3.11: The Channel (通道)

Channel 是 Go 最具代表性的併發工具，但它不是魔法管線。
這一章會把它拆成 Runtime 裡的 `hchan`、ring buffer、等待隊列與鎖，讓你知道每一次 send/recv 到底付出了什麼成本。

---

## 1. 第一樂章：隱喻 (The Metaphor - Conveyor Belt)

我們回到皇家圖書館的地下行政區。

如果要讓兩位館長 (Goroutines) 合作，傳統方法 (Shared Memory) 是讓他們共用 **同一塊大黑板**。
*   **慘劇 (Race Condition)**: 兩位館長同時改同一格內容，很容易互相覆蓋，或讀到半成品。
*   **解法 (Lock)**: 必須規定「我寫黑板時你不能動」，這導致嚴重的停頓。

Go 的 Channel 則是在兩位館長之間架了一條 **「任務單輸送帶」**：
*   **輸送帶 (Channel)**: 一條受 Runtime 管理的通道，用來傳遞資料。
*   **動作**: 館長 A 把任務單放上輸送帶 (Send)；館長 B 在另一端取走 (Recv)。
*   **結果**: 雙方不需要直接搶同一塊黑板，而是透過通道交接所有權。

這體現了 CSP 的精隨：
> **Do not communicate by sharing memory; instead, share memory by communicating.**
> (與其搶同一塊黑板，不如把資料裝進任務單，沿著輸送帶交給下一位館長。)

---

## 2. 第二樂章：結構 (The Structure - hchan)

Go Channel 的底層是一個名為 `hchan` 的 Struct (位於 `runtime/chan.go`)。它就像這條任務單輸送帶的控制台。

```go
type hchan struct {
    qcount   uint           // 當前膠囊數量 (len)
    dataqsiz uint           // 管子容量 (cap)
    buf      unsafe.Pointer // 環狀緩衝區 (Ring Buffer) 的指針
    elemsize uint16         // 每個膠囊的大小
    closed   uint32         // 關閉狀態
    elemtype *_type         // 資料型別
    sendx    uint           // 下一個要放入的位置 (Sender Index)
    recvx    uint           // 下一個要拿取的位置 (Receiver Index)
    recvq    waitq          // 等待接收的隊伍 (Receiver Queue) -> 掛著一串 sudog
    sendq    waitq          // 等待發送的隊伍 (Sender Queue) -> 掛著一串 sudog
    lock     mutex          // 保護這個管子的鎖 (就是我們在 3.5 提過的那個 runtime.lock)
}

// 隊伍結構 (Linked List)
type waitq struct {
    first *sudog // 隊頭 (Head)
    last  *sudog // 隊尾 (Tail)
}

// 排隊單 (The Ticker)
type sudog struct {
    g *g              // 是哪個 Goroutine 在排隊
    elem unsafe.Pointer // 貨物 (Data) 所在的地址 (直接指向該 G 的 Stack)
    next *sudog       // 下一張單子
    prev *sudog       // 上一張單子
    ...
}
```

### 2.1 環狀緩衝區 (Ring Buffer)
如果 Channel 是有緩衝的 (`make(chan int, 3)`)，`buf` 會指向一塊連續記憶體，當作 **Ring Buffer** 使用。
*   **特點**: 膠囊先進先出 (FIFO)。
*   **運作**: `sendx` 放入，`recvx` 取出。當指針走到盡頭時，會繞回 0 (Circular)。

### 2.2 等待隊伍 (`recvq` & `sendq`)
這是 Channel 最神奇的地方。如果管子滿了 (Sender Blocked) 或者空了 (Receiver Blocked)，Goroutine 去哪了？
它們會被打包成一個 **`sudog`** (封裝了 G 的結構)，然後掛在 `recvq` 或 `sendq` 上排隊，自己則進入 **休眠 (Parking)** 狀態。

---

## 3. 第三樂章：流程 (The Flow)

### 3.1 發送 (Send): `ch <- data`

當我們執行發送操作時，圖書館管理員會執行以下 SOP：

1.  **上鎖 (`lock`)**: 保護 `hchan`，避免其他人同時塞膠囊。
2.  **情況 A (直接面交)**:
    *   檢查 `recvq` 有沒有人在排隊？
    *   **有的話 (Direct Copy)**: 既然 G2 (Receiver) 已經拿著盤子在排隊了，那就不必放在轉盤 (Buffer) 了。
        *   **動作**: G1 (Sender) 直接把資料從自己的 Stack，**跨越空間寫入** 到 G2 正在休眠的 Stack 變數地址裡。省去二次複製。
    *   喚醒那個 Receiver (`goready`)。
3.  **情況 B (放入緩衝)**:
    *   沒人排隊，但 `qcount < dataqsiz` (緩衝未滿)。
    *   **寫入 Heap**: 把資料從 Stack 複製到 `buf[sendx]` (注意：Buffer 位於 Heap 上)。
    *   `sendx++`, `qcount++`。
4.  **情況 C (阻塞等待)**:
    *   緩衝滿了，也沒人收。
    *   **打包 (`sudog`)**: 把自己 (G) 打包成 `sudog` (這是 Runtime 內部分配的物件，通常位於 Heap/P-Cache)。
    *   **資料去哪了？**: **資料還留在 Sender 的 Stack 上！** 沒有複製進 Buffer。Sender 只是把「Stack 地址」記在 `sudog` 裡，然後就睡著了。
    *   **G 進入停車場 (gopark)**，讓出 CPU，等待被喚醒。
5.  **解鎖 (`unlock`)**。

### 3.2 接收 (Receive): `data := <- ch`

流程幾乎是鏡像的：

1.  **上鎖**。
2.  **情況 A (緩衝有貨 - 維持 FIFO)**:
    *   **我拿貨**: 既然 Buffer 有東西，那最舊的資料一定在 Buffer 裡。
    *   **動作**: 從 `buf[recvx]` (Heap) 複製到我的 Stack。
    *   **幫忙補貨**: 如果 `sendq` 有人 (G_Sender) 在排隊，代表 Buffer 其實是滿的。
        *   為了不讓 G_Sender 繼續等，我把 G_Sender Stack 裡的新資料，**搬運到** 我剛剛清出來的 `buf[recvx]` 位置裡。
        *   **注意**: 這裡 **不是** Direct Copy (面交)。因為我拿的是「舊貨 (Heap)」，G_Sender 補進去的是「新貨」。這是為了維持嚴格的 **FIFO** 順序。
        *   **FIFO 原理**: 你可能疑惑「把新資料放在剛拿走的舊位置，順序不會亂嗎？」
            *   別忘了這是 **Ring Buffer**。當 Buffer 滿時，`recvx` (讀取頭) 和 `sendx` (寫入頭) 指向同一個位置。
            *   我讀完後，`recvx` 會往前走一格 (`recvx++`) 去讀下一筆舊資料。
            *   剛空出來的那一格，正好就是 `sendx` 要寫的位置 (隊尾)。所以新資料放進去，我要繞一圈後才會讀到它。FIFO 完美成立。
3.  **情況 B (直接面交)**: 緩衝是空的，但 `sendq` 有人 (Unbuffered Channel 的情況)。直接從 Sender 拿資料。
4.  **情況 C (阻塞等待)**: 沒貨也沒人送。把自己掛在 `recvq`，睡覺去。
5.  **解鎖**。

---

## 4. 第四樂章：特性與陷阱 (Features & Pitfalls)

### 4.1 為什麼 Channel 是 "Slow" 的？
相較於 Mutex，Channel 的開銷確實較大 (涉及記憶體複製與調度)。
但為什麼我們說它效能還是很好？
*   **鎖的優化 (Spinning)**: `hchan.lock` 並非普通的鎖，它是 `runtime.lock`。在競爭不激烈時，它會 **自旋 (Spinning)** 幾次嘗試搶鎖，而不會立刻進入 OS 核心睡眠 (Syscall)。這讓它的開銷維持在用戶態 (User Space)。
*   **臨界區極短**: 鎖住的期間只做「指標移動」和「記憶體複製」，速度極快，搶完就跑。

 
也就是說，除非超高頻競爭 (百萬 QPS)，否則 Channel 的鎖開銷幾乎可以忽略。
它的價值主要在於 **「解耦」** 與 **「安全性」**。

### 4.2 Panic 觸發點
*   **關閉 nil channel**: `panic`
*   **發送給 closed channel**: `panic` (這是最常見的 Bug)
*   **關閉 closed channel**: `panic`

### 4.3 Select 多路復用的底層魔法 (`runtime.selectgo`)

我們常說 `select` 能同時監聽多個 Channel，但它到底怎麼做到的？
在編譯期，Go 編譯器會把 `select` 語句轉換成對 Runtime 底層 `runtime.selectgo` 函數的呼叫。這個函數做了一系列非常精妙的操作：

1.  **打亂順序 (Shuffle Order)**
    為了防止寫在前面的 `case` 永遠優先觸發（造成飢餓問題），`selectgo` 第一步是產生一個隨機數陣列，把所有 `case` 的檢查順序做一次 Fisher-Yates 洗牌。這是為什麼多個 Channel 同時就緒時，觸發是隨機的原因。
2.  **記憶體地址鎖排序 (Lock Order)**
    `select` 要同時操作多個 Channel，這意味著它**必須同時鎖住所有參與的 Channel**。
    如果有兩個 Goroutine 寫了相反順序的 `select`，直接上鎖 100% 會引發 Deadlock。
    為了避免死結，`selectgo` 會把所有參與的 Channel **按照它們在 Heap 上的記憶體地址由小到大排序**。然後依照這個物理地址順序去要求鎖 (`lock`)。地址排序保證了全局一致的上鎖順序，巧妙避開了死結。
3.  **第一輪尋訪 (Fast Path)**
    拿到所有鎖之後，依照「第 1 步打亂的順序」輪詢每個 Channel，看（有沒有資料可讀 / 緩衝有沒有空位可寫）。
    *   只要發現**任何一個**已經 Ready，立刻解開所有其他 Channel 的鎖，並執行該 `case`。
    *   如果都沒 Ready，而且有寫 `default`，就解開所有鎖，執行 `default`。
4.  **分身排隊並休眠 (Parking)**
    如果都沒 Ready，也沒有 `default`，這個 Goroutine 就要睡覺了。但它要怎麼同時等好幾個 Channel？
    *   Runtime 會把這個 Goroutine 包裝成好幾個 `sudog` 結構體（像是分身）。
    *   把這些 `sudog` 分別掛進每一個參與 Channel 的 `recvq` 或 `sendq` 等待隊列裡。
    *   然後呼叫 `gopark` 把 Goroutine 催眠。
5.  **喚醒與拔除分身 (Wake Up & Cleanup)**
    *   當**其中某一個** Channel 來了資料，那個 Channel 的發送者會發現隊列裡有 `sudog`，然後把資料交給它，並喚醒這個 Goroutine (`goready`)。
    *   Goroutine 醒來後的第一件事，就是**趕快去其他的 Channel 隊列裡，把剩下的 `sudog` 分身全部拔掉 (dequeue)**，避免之後被誤喚醒。
    *   最後解開所有鎖，執行中獎的那個 `case`。

---

## 5. 第五樂章：總結 (Finale)

Channel 是一個 **「帶鎖的環狀緩衝區 + 排程器整合機制」**。
*   它不只是資料結構，它還與 Runtime Scheduler 深度綁定 (負責讓 G 睡覺和起床)。
*   它用 **Copy** 取代 **Pointer Sharing**，實現了 CSP 併發模型。

---
**Next: [Book 3.12 垃圾回收 (Garbage Collector)](./3_12_GO_GC.md)** (預計建立)
