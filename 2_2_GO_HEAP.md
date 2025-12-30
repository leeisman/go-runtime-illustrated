# Book 2.2: 複雜的代價 (The Heap)

## 第一樂章：無法銷毀的紙條 (The Escape)

這是一個稍微複雜一點的變體。我們試圖把函數內的變數「帶出來」。

```go
package main

// createValue 建立一個數值並返回其指標
func createValue(val int) *int {
    // x 需要活得比這個函數久
    x := val
    return &x
}

func main() {
    // 第一次申請：在黑板寫下 10，拿到小白紙條 ptr1
    ptr1 := createValue(10)
    
    // 第二次申請：在黑板另一處寫下 20，拿到小白紙條 ptr2
    ptr2 := createValue(20)
    
    // --- 假設這裡發生了 Context Switch ---
    
    // 必須拿著紙條去黑板找數字
    sum := *ptr1 + *ptr2
    println(sum)
}
```

### 實驗證明 (The Evidence)
執行 `go build -gcflags "-m"`：

```bash
# command-line-arguments
./main.go:4:6: can inline createValue
./main.go:10:6: can inline main
./main.go:12:24: inlining call to createValue
./main.go:15:24: inlining call to createValue
./main.go:6:5: moved to heap: x
```

**解讀**:
1.  **`moved to heap`**: 編譯器發現 `x` 的地址被回傳了 (Escaped)。即使函數被內聯 (inlined)，Go 為了安全起見，依然判定這個變數「逃逸」出了原始作用域。
2.  **Stack Frame 的宿命**: 
    *   **Stack** 是一整疊預先放好的紙 (8MB)。
    *   每次呼叫函數 (`createValue`)，館長只是從這疊紙上面 **暫時劃出一張** (Stack Frame) 來用。
    *   當函數結束，這張紙在邏輯上就會被 **「撕掉」或「回收」** (Pop)。
3.  **危機**: 如果 `x` 寫在這張即將被撕掉的紙上，`main` 拿到的指標就會變成廢紙。所以 Go 強制把 `x` 搬到黑板。

---

## 第二樂章：黑板的啟用 (The Blackboard)

在這個場景中，隱喻發生了什麼變化？

### Step 1: 小白紙的極限 (Stack Limitation)
*   **誤區澄清**: 每個函數 **不會** 去申請新的記憶體。它們只是輪流使用那疊 **早已存在的 8MB 紙堆 (Stack)**。
*   **危機**: 館長在執行 `createValue` 時，原本是寫在 **當下的那張小白紙 (Current Stack Frame)** 上的。
*   **衝突**: 按照規定，一旦 `createValue` 結束，這張紙就會被視為廢紙 (Popped)，隨時會被下一個函數拿去寫新的東西。

### Step 2: 申請黑板 (Heap Allocation)
*   **解決方案**: 館長轉身，走向房間中央那塊 **303 專屬的大黑板 (Heap)**。
    *   **權限**: 這塊黑板完全屬於 303 號房，只有這裡的館長能看能寫，別的房間看不到。
    *   **來源**: 它是 **行政區 (OS/Kernel)** 預先掛在房內的 (Virtual Memory)。若寫滿了，館長可以打電話請行政區派人來擴建 (`mmap`)。
    *   **重點**: 黑板上的字 **不會** 因為翻頁 (Function Return) 而被自動擦掉。
*   **動作**:
    1.  **第一次呼叫**: 
        *   館長在黑板左上角寫下 `10` (存留)。
        *   接著，把黑板座標 `0x100` **帶回給 `main`**，寫在 `main` 函數保留的 **小白紙區域 (main's Stack Frame)** 上。
        *   至於 `createValue` 自己的那張小白紙，隨後就被撕掉了，但座標已經安全地交給 `main` 了。
    2.  **第二次呼叫**: 
        *   館長在黑板右下角寫下 `20`。
        *   同樣地，把座標 `0x900` 帶回給 `main` 的小白紙上。
    3.  **現狀**: `main` 的小白紙上記錄著兩個座標，指向黑板上兩個**不連續**的位置。

---

## 第三樂章：更昂貴的切換 (The Context Switch II)

還記得之前提到的 **Context Switch 效能損耗** 嗎？在 Heap 的場景下，這個損耗會被**倍數放大**。

我們來慢動作播放一次：

### 1. 館長 A 的最後一刻 (The Interruption)
*   **進度**: 館長 A 剛執行完 `ptr2 := createValue(20)`。
*   **狀態**:
    *   **小白紙 (Stack)**: 上面寫著 `ptr1=0x100`, `ptr2=0x900`。
    *   **黑板 (Heap)**: 這兩個座標已經寫好了數字 10 和 20。
*   **中斷**: 就在館長 A 準備要把這兩個數字加起來 (`sum := ...`) 的瞬間，行政區廣播響了：「303 號房暫停，館長 A 下班！」

### 2. 存檔 (Save Context)
*   館長 A 嘆了口氣，走進 **行政專用區**，打開 **儲物櫃 (Locker)**。
*   他在字條上寫下：「我做到第 20 行 (PC)，手指指在 main 的紙張位置 (SP)」。
*   **關鍵**: 他不需要搬運黑板上的資料，也不需要影印小白紙。他只是存個檔就走了。

### 3. 館長 B 的接手與困境 (The Resume & Cache Miss)
一段時間後，**館長 B** 被指派進來接手。

1.  **接手時刻**: 館長 B 看了儲物櫃的字條，坐回書桌前，視線落在 `main` 函數的這一行：
    `sum := *ptr1 + *ptr2`
2.  **指令解讀**: 書上寫著：「把 `ptr1` 指向的數字 和 `ptr2` 指向的數字加起來」。
3.  **第一重打擊 (讀取指標)**:
    *   B 必須先低頭看小白紙上的 `ptr1` 變數。
    *   從這變數得到一個座標 `0x100`。
4.  **第二重打擊 (Pointer Chasing / 追蹤黑板)**:
    *   B 必須抬頭離開小白紙，依照 `0x100` 走到黑板的左上角去讀那個數字 `10`。
    *   **代價**: 這就是所谓的 **Pointer Chasing**。因為資料不在手邊 (Stack)，而是在遠處 (Heap)，這一來一回的「抬頭找座標」過程，導致 CPU Cache 難以預測，容易發生 Cache Miss。
5.  **第三重打擊 (Spatial Locality Poor)**:
    *   B 讀完 `10`，接著要讀 `ptr2` (座標 `0x900`)。
    *   **慘劇**: 因為這兩個數字是分兩次寫上去的，它們可能一個在黑板最左邊，一個在最右邊。
    *   B 必須再折返跑去黑板另一頭讀 `20`。
    *   **再發生一次 Cache Miss**。

**結論**: 相比於 Stack 上變數緊挨在一起（一次讀取全家飽），Heap 上的指標就像**藏寶圖**。你必須不斷地跑來跑去 (Pointer Chasing)，這對 CPU Cache 非常不友善。

---

## 第四樂章：代價 (The Allocation Cost)

為什麼我們不乾脆全部用黑板就好？

1.  **距離遠 (Latency)**:
    *   小白紙 (Stack) 就在手邊，寫字超快。
    *   走去黑板 (Heap) 寫字需要時間，而且黑板很大，找空位也要時間 (`malloc` is slow)。

2.  **管理難 (GC Pressure)**:
    *   小白紙用完即撕 (Auto-cleanup)，完全不用操心。
    *   黑板上的字寫了就不會消失。如果一直寫、一直寫，黑板很快就滿了 (OOM)。

---

## 第五樂章：清潔時間 (Garbage Collection)

這就是為什麼我們需要 **GC**。在我們的圖書館裡，**沒有專職的清潔工 (Janitor)**。

*   **兼職清潔**:
    *   **書本 (Code)** 裡其實夾著一本「清潔指南」。
    *   當館長覺得黑板快滿了，或者 **翻到了書本上規定的清潔時間 (GC Trigger)**，館長就被迫**停下手邊的工作 (Stop The World)**。
    *   他必須拿著板擦，去檢查黑板上每一個座標：
        *   「這個座標，還有任何小白紙 (Stack) 指向它嗎？」
        *   如果有，保留 (Mark)。
        *   如果沒有，擦掉 (Sweep)。
*   **負擔**: 這就是為什麼 Heap 分配多了程式會變慢，因為館長把時間都花在擦黑板，而不是在讀書 (執行業務邏輯)。

---

## 第六樂章：生存法則 (The Survival Rule)

讀完這一章，你只需要記住一個最簡單的判斷準則，就能像編譯器一樣思考：

**「這個變數，在它出生的 `{}` (作用域/函數) 結束後，還需要活著嗎？」**

*   **YES (活得比 `{}` 久)** -> 它必須去 **黑板 (Heap)**。因為 `{}` 結束代表那張小白紙 (Stack Frame) 會被撕掉。
*   **NO (只在 `{}` 內活)** -> 它就留在 **小白紙 (Stack)**。這是最高效的選擇。

這就是 **Escape Analysis** 的全部真義：不是因為變數大或小，而是因為它**想活得比它出生的 `{}` 還久**。

---

## 第七樂章：反轉思維 (The Optimization)

你可能會有個絕妙的想法：「既然後面還有 `main` 的小白紙可以用，為什麼館長不能直接寫在 `main` 的紙上，這樣就不用跑去黑板了？」

**沒錯！你是對的！** 這就是 Go 優化的終極技巧：**向下傳遞 (Passing Down)**。

### 技巧：老闆給紙 (Caller Alloc)
如果我們改變寫法，由 `main` (老闆) 先準備好紙，交給 `fillValue` (下屬) 去填：

```go
package main

// fillValue 這次不回傳指標，而是接收 main 給的紙條 (pointer) 並填寫
func fillValue(ptr *int, val int) {
    // 寫在老闆給的紙上 (main's stack frame)
    *ptr = val
}

func main() {
    // 第一次申請：main 自己準備紙
    var x1 int
    fillValue(&x1, 10) 
    
    // 第二次申請：main 再準備一張紙
    var x2 int
    fillValue(&x2, 20)
    
    // 這裡可以直接相加，因為 x1, x2 都在 main 自己的手邊 (Stack)
    sum := x1 + x2
    println(sum)
}
```

### 結果
執行 `go build -gcflags "-m"`：

```bash
# command-line-arguments
./main.go:4:6: can inline fillValue
./main.go:9:6: can inline main
./main.go:12:11: inlining call to fillValue
./main.go:16:11: inlining call to fillValue
./main.go:4:16: ptr does not escape
```

*   **Escape Analysis**: `ptr does not escape`。這是最關鍵的一行！因為 `ptr` 雖然是指標，但它指向的地址 (`&x1`, `&x2`) 一直都在 `main` 的 Stack Frame 內，沒有超出 `main` 的生命週期。
*   **隱喻**:
    1.  `main` 拿出一張小白紙，劃好兩個空位 `x1`, `x2`。
    2.  `main` 把 `x1` 的位置指給 `fillValue` 看說：「幫我填上 10」。
    3.  `fillValue` 填完交還。
    4.  **完全不需要黑板**！

    3.  `fillValue` 填完交還。
    4.  **完全不需要黑板**！

我們再來慢動作播放一次，看看這次的切換有多快樂：

### 1. 館長 A 的輕鬆工作 (The Stack Write)
*   **動作 1 (準備小白紙)**: 館長 A 用手 (SP) 在 **Main 的小白紙區域** (Main's Stack Frame) 劃出一塊區域，標上 `x1` 和 `x2`。
*   **動作 2 (進入子作用域)**: 
    *   他跳轉進 `fillValue` 的書頁 (Function Call / Scope)。
    *   書上寫著：「請在傳進來的這個位置填上 10」。
    *   於是他直接在 **Main 的那張小白紙** 的 `x1` 格子裡填上 10。
    *   任務完成，跳回 `main` 的書頁，接著對 `x2` 做一樣的事，填上 20。
*   **狀態**:
    *   **小白紙 (Stack)**: `x1=10`, `x2=20` 緊緊挨在一起 (例如記憶體位址 0x0100 和 0x0108)。
    *   **黑板 (Heap)**: **完全空白**。館長連轉頭看黑板一眼都不需要。
*   **中斷**: 就在準備相加時，廣播響了：「館長 A 下班！」

### 2. 存檔 (Save Context)
*   館長 A 依然只是走進儲物櫃，寫下 PC (眼睛讀到的行數) 和 SP (手指按著的位置)。
*   **重點**: 因為沒有任何黑板資料，他心裡非常輕鬆，沒有「待會回來要重新找座標」的負擔。

### 3. 館長 B 的快樂接手 (The Happy Resume)
館長 B 接手，這次的體驗與第三樂章截然不同：

1.  **接手時刻**: 館長 B 坐下，視線落在 `sum := x1 + x2`。
2.  **第一重驚喜 (Stack Hit)**:
    *   B 低頭看小白紙 (Stack)。
    *   發現 `x1` 和 `x2` 就在手邊。
3.  **無敵的空間局部性 (Spatial Locality Excellent)**:
    *   當 B 的眼睛瞄到 `x1` 時，CPU 的硬體機制 (Cache Line Prefetch) 發現隔壁就是 `x2`。
    *   它自動把 `x1` 和 `x2` **整塊一起** 讀進了大腦 (L1 Cache)。
    *   B 甚至還沒意識到要讀 `x2`，`x2` 的數值已經在他的腦袋裡了。
4.  **結果**:
    *   沒有抬頭。
    *   沒有跑黑板。
    *   沒有 Cache Miss。
    *   運算在 1 個 Cycle 內結束。

**結論**: 這就是 **Stack Allocation** 的威力：不僅省去了寫黑板的時間，更重要的是它讓資料**緊湊排列**。館長不需要拿著藏寶圖東奔西跑，所有資料都像自助餐盤一樣擺在面前，伸手即得。

### 最終結論：方向決定命運 (Direction is Destiny)

| 方向 | 行為 |結果 | 隱喻 |
| :--- | :--- | :--- | :--- |
| **向上傳遞 (Up)** | 函數回傳指標 (`return &x`) | **Heap** | 試圖把 **子作用域 (Child Scope)** 的紙條傳回給 **父作用域 (Parent Scope)** -> **危險！因為子作用域結束紙條就會失效，強迫寫黑板**。 |
| **向下傳遞 (Down)**| 函數接收指標 (`func(&x)`) | **Stack** | 把 **父作用域** 的紙條傳進去給 **子作用域** 使用 -> **安全，因為父作用域還沒結束，紙條依然有效**。 |
