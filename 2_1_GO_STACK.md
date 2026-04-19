# Book 2.1: 簡單的快樂 (Stack Only)

這一章先看最單純的記憶體命運：資料只活在函數作用域裡，進來時建立，離開時消失。
理解 Stack 的便宜與乾淨，後面才看得懂 Heap 為什麼昂貴。

---

## 1. 第一樂章：純粹的代碼 (The Pure Code)

這是一個簡單的撲克牌比大小邏輯。沒有指標，沒有 goroutine，沒有介面。

```go
package main

// 純粹的 CPU 運算，無任何 I/O
func calculateScore(card1, card2 int) int {
    // 這些變數都只在 Stack 上存活
    sum := card1 + card2
    return sum
}

func main() {
    a := 10
    b := 20
    
    // 純粹的數值傳遞 (Value Copy)
    _ = calculateScore(a, b)
}
```

### 實驗證明 (The Evidence)
我們實際以此代碼執行 `go build -gcflags "-m"` (Escape Analysis)：

```bash
# command-line-arguments
./main.go:4:6: can inline calculateScore
./main.go:10:6: can inline main
./main.go:15:20: inlining call to calculateScore
```

**解讀**:
1.  **`can inline`**: 編譯器發現函數太簡單，甚至不鋪新的 Stack Frame，直接把邏輯合併到 `main` 裡 (效能極致)。
2.  **沒有 `escapes to heap`**: 證明所有變數 (a, b, sum) 都乖乖待在 Stack 上，隨著函數結束自動銷毀。

---

## 2. 第二樂章：從指令到執行 (The Genesis Story)

這段簡單的代碼是如何變成一個運作中的包廂？

### Step 1: 申請包廂 (Shell -> Admin)
*   **起源**: **Shell 本身也是一間閱讀室**。你在這間名為 Shell 的閱讀室裡填單與送出請求：「我要開一間新閱讀室執行 `./experiment_stack`」。
*   **申請 (Syscall)**: 你的請求透過 Shell 向行政區 (Kernel) 遞交申請 (`fork` + `execve`)。
*   **行政區的建設 (The Setup)**:
    1.  **建立空房**: 在圖書館找了一個編號為 **303** 的新閱讀室 (Process)。
    2.  **名冊登記 (Process Table)**: 行政區在 **總預約名冊** 上寫下：「303 號房 -> experiment_stack」。(這就是你打 `ps` 指令時看到的資料)。
    3.  **參考書籍 (ELF Header)**: 行政區翻閱 `experiment_stack` 執行檔，根據裡面的藍圖來佈置房間。
    4.  **上架書籍 (Code Segment / Text Section)**:
        *   行政區依照藍圖，將 `main` 和 `calculateScore` 的 **書本 (Books / Machine Code)** —— 也就是那些細微的 `MOV`、`ADD` 指令 —— **搬進** 了閱讀室，放在館長隨手可得的地方。
        *   **注意**: 這些書是 **唯讀 (Read-Only)** 的。館長只能讀上面的字，不能拿筆在書上亂改 (防止程式自我毀滅)。
    5.  **預置小白紙 (Stack Init)**: **關鍵步驟！** 在館長還沒進來之前，行政區先在牆邊放了一疊 **8MB 的小白紙 (Scratchpad / Main Stack)**。
        *   **目的**: 為了讓館長一進來就有地方可以隨手計算、寫下臨時變數。

### Step 2: 館長進駐 (Entry Point)
*   行政區指派一位 **館長 (CPU/Main Thread)** 走進這個閱讀室。
*   館長徑直走到那疊小白紙前，將 **手指 (Stack Pointer)** 按在第一張紙的頂端。
*   **遊戲開始**: 館長翻開放在架上的 **第一本書** 第一頁 (`main.main`)。

### Step 3: 微觀執行 (Micro Execution - The Loop)
這就是電腦運作的本質：**Fetch-Execute Cycle (指令週期)**。館長不斷重複著「眼看書本、手寫白紙」的循環：

1.  **指令 1 (SUB SP - 開桌)**:
    *   **眼 (Fetch)**: 館長看著書本上 PC 指向的第一行 `48 83 EC 18` (機器碼)。
    *   **腦 (Decode)**: 瞬間理解這是「我要多用 24 bytes 的紙」。
    *   **手 (Execute)**: 手指將 **Stack Pointer** 往紙張下方移，圈出 `main` 要用的書寫範圍。

2.  **指令 2 (MOV - 賦值)**:
    *   **眼**: 視線移到下一行。
    *   **手**: 依照指令，在剛圈好的 **小白紙範圍 (Stack)** 內寫下 `10` (變數 a) 和 `20` (變數 b)。

3.  **指令 3 (ADD - 運算)**:
    *   **眼**: 視線移到下一行。
    *   **腦**: 看到 `calculateScore` 的邏輯被內聯 (Inlined)，直接在腦中 (Register) 將 10 和 20 相加得到 `30`。

4.  **指令 4 (ADD SP - 收桌)**:
    *   **眼**: 看到函數結束。
    *   **手**: 將 **SP** 移回原位。剛剛寫的 10, 20 瞬間變成廢紙，被撕掉或無視 (Pop)，邏輯上消失了。

---

## 3. 第三樂章：休息是為了走更長的路 (The Context Switch)

假設館長 (CPU A) 剛做完 **指令 2 (MOV)**，正準備做 **指令 3 (ADD)** 時，行政區廣播：「館長，303 時間到了，請去 404 幫忙！」

1.  **卸下裝備 (Save Context)**:
    *   **小白紙不動**: 閱讀室裡的 8MB 小白紙 (Stack) 完全不用搬動，就留在原處。
    *   **只存裝備**: 館長只需要把身上的 **極少量資訊** (十幾個 Register 數值) 抄寫到 **儲物櫃 (Locker / task_struct)**。
    *   **儲物櫃是誰的？**: 這帶出了一個深層秘密——每個閱讀室裡其實都有一個 **「行政專用區」 (Kernel Space)**。
        *   平常你看不見它（被屏風擋住），只有館長在執行特殊任務 (Syscall/Interrupt) 或下班 (Switch) 時，才會走進這個區域操作儲物櫃。
        *   這就是為什麼即便在 303 號房，一般的代碼也拿不到這些核心資料。
    *   這就是為什麼切換速度可以這麼快。
    *   然後兩手空空地離開。

2.  **切換 (Switch)**:
    *   館長去別的房間忙了一圈 (可能過了 10ms)。

3.  **重新著裝 (Restore Context)**:
    *   館長回到 303，打開儲物櫃。
    *   把 **SP** 數值載入手指 (Pointer)，把 **AX** 數值載入腦袋 (Register)，把 **PC** 數值載入眼睛 (Program Counter)。
    *   **無縫接軌**：瞬間恢復成「剛做完指令 2」的狀態，繼續執行 **指令 3 (ADD)**。

4.  **但代價是什麼？ (The Hidden Cost)**:
    *   雖然「存/取」裝備很快，但**暖身 (Warm-up)** 很慢。
    *   **冷掉的腦袋 (Cold Cache)**：館長剛回來時，雖然知道手指指哪裡，但他腦海中原本暫記的那些「上下文脈絡」 (L1/L2 Cache) 全都被清空了。
    *   他必須重新從慢速的書架和桌子上讀取資料進入腦袋，這段「找回手感」的時間，才是 Context Switch 最昂貴的地方。
    *   *(在本案例中，因為桌上只有兩個數字，館長瞬間就能找回手感，但在真實的大型運算中，這會是效能殺手。)*


---

## 4. 第四樂章：接力 (The Migration)

如果那天館長 A (CPU Core 0) 休假了，或者正在忙別的事，行政區指派了 **館長 B (CPU Core 1)** 來接手 303 號房，會發生什麼事？

1.  **房間沒變**: 303 號房還是那一間，裡面的 **書本 (Code)**、**小白紙 (Stack)**、**行政專用區的儲物櫃 (Task Struct)** 都完好無缺地留在原處。

2.  **無縫交接**:
    *   館長 B 走進房間，徑直走進行政專用區。
    *   他從儲物櫃拿出館長 A 留下的指令字條 (Context)。
    *   他把自己的手指 (SP) 放在 A 指定的位置，把眼睛 (PC) 看向 A 讀到的行數。

3.  **哲學意義 (CPU Migration)**:
    *   對於這段程式碼 (Book) 和變數 (Stack) 來說，**是誰 (Who)** 來執行並不重要，重要的是 **狀態 (State)** 是否正確。
    *   這就像大隊接力，棒子 (Context) 只要交接正確，跑道 (Memory) 並不需要跟著移動。
    *   這就是為什麼現代作業系統可以隨意地把 Process 在不同的 CPU Core 之間丟來丟去。

---

## 5. 第五樂章：Stack 的本質 (Summary)
**Stack** 不是資料結構課本裡的那個 Stack，對 OS 來說，它只是 **"一段連續的記憶體 + 一個會移動的 SP 指標"**。
因為簡單 (只有移動指標/撕紙)，所以它是這世界上最快的記憶體分配方式。
