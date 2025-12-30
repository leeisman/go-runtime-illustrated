# Book 2.3: 逃逸分析實驗 (The Escape Analysis)

這裡是我們驗證「生存法則」的實驗記錄。所有的隱喻都必須經得起編譯器的考驗。

## 實驗一：切片擴容 (Slice Append)

Slice 是 Go 最常用的資料結構，它的行為完美展示了 Stack 到 Heap 的過渡。

### 代碼 (`experiment_slice.go`)
```go
package main

func main() {
    // Case 1: 小刷子 (Small Slice)
    // cap=2, 預期在 Stack
    s := make([]int, 0, 2) 
    s = append(s, 1)
    s = append(s, 2)
    s = append(s, 3) // 觸發擴容 (Grow)
    println(s[0])
    
    // Case 2: 超大刷子 (Large Slice)
    // cap=100w, 預期在 Heap (太大放不下書桌)
    s2 := make([]int, 0, 1000000)
    println(s2[0])
}
```

### 結果
```bash
go build -gcflags "-m" experiment_slice.go

./experiment_slice.go:5:11: make([]int, 0, 2) does not escape
./experiment_slice.go:14:12: make([]int, 0, 1000000) escapes to heap
```

### 結論
1.  **擴容不一定逃逸**：Case 1 雖然 `append` 超過了容量導致擴容，但因為資料量小且沒有傳出作用域，館長只是在桌上換了一張稍大的紙。
2.  **太長必逃逸**：Case 2 證明了除了「作用域」，**大小 (Size)** 也是限制。如果紙大到桌子 (Stack Limit) 放不下，行政區會強制你去黑板。

---

## 實驗二：物件傳遞 (Struct Passing)

這驗證了我們最核心的「向上 vs 向下」法則。

### 代碼 (`experiment_struct.go`)
```go
package main

type User struct {
    ID   int
    Name string
}

// Case 1: 傳值 (Value Copy)
func createUserStack() User {
    u := User{ID: 1, Name: "Stack"}
    return u // 影印一份帶走
}

// Case 2: 傳指標 (Pointer) - 向上傳遞
func createUserHeap() *User {
    u := User{ID: 2, Name: "Heap"}
    return &u // 試圖帶走紙條座標
}

func main() {
    u1 := createUserStack()
    println(u1.ID)

    u2 := createUserHeap()
    println(u2.ID)
}
```

### 結果
```bash
go build -gcflags "-m" experiment_struct.go

./experiment_struct.go:14:2: moved to heap: u
```

### 結論
1.  **傳值即安全**：`createUserStack` 雖然創造了物件，但因為回傳的是 Value (影本)，原本的紙可以放心撕掉。
2.  **向上指標必死**：`createUserHeap` 試圖回傳 Pointer，觸發了生存法則（想活得比函數久），被踢去 Heap。

---

## 實驗三：介面陷阱 (Interface)

Interface 是最容易讓人困惑的地方，這裡揭露了它的雙重性格。

### 代碼 (`experiment_interface.go`)
```go
package main

type Printer interface {
    Print()
}

type MyStruct struct {
    val int
}

func (m MyStruct) Print() {
    println(m.val)
}

// Case 1: 本地轉型 (Local Assignment)
func localInterface() {
    var p Printer
    s := MyStruct{val: 10}
    p = s // 這裡常會導致逃逸
    p.Print()
}

// Case 2: 向下傳遞 (Passing Down)
func useInterface(p Printer) {
    p.Print()
}

// Case 3: 向上傳遞 (Returning Up)
func returnInterface() Printer {
    s := MyStruct{val: 30}
    return s
}

func main() {
    localInterface()
    
    s := MyStruct{val: 20}
    useInterface(s)
    
    i := returnInterface()
    i.Print()
}
```

### 結果
```bash
go build -gcflags "-m" experiment_interface.go

./experiment_interface.go:19:6: s escapes to heap  <-- Case 1: Local
./experiment_interface.go:24:19: leaking param: p  <-- Case 2: Param
./experiment_interface.go:31:9: s escapes to heap  <-- Case 3: Return
./experiment_interface.go:35:16: s escapes to heap <-- localInterface call
./experiment_interface.go:38:15: s does not escape <-- useInterface call (Surprise!)
```

### 結論
1.  **向上必逃 (Case 3)**：不意外，只要往回傳，不管是 Struct 還是 Interface 都要逃。
2.  **向下安全 (Case 2)**：**這是亮點！** 即使是 Interface，只要是向下傳遞 (main -> useInterface)，`s does not escape`。這證明了「老闆給紙」的技巧連 Interface 都適用。
3.  **本地也不一定安全 (Case 1)**：將 Struct 塞進 Interface 變數時，為了構建動態表格 (Itab)，編譯器往往會保守地選擇 Heap。

这告訴我們：**盡量保持型別確定 (Concrete Type)**，僅在必要時 (例如函數參數需要多態) 才使用 Interface，可以減少不必要的逃逸。
