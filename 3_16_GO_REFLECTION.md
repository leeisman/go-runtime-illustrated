# Book 3.16: The Reflection (鏡像世界 - 動態型別的黑魔法)

Reflection 讓 Go 在執行期重新觀察型別，但這份自由不是免費的。
這一章會把 interface 裡的型別資訊拆出來看，理解三大定律、可修改性，以及為什麼反射程式一定要重視快取。

---

## 1. 第一樂章：緣起 (The Origin)

Go 是一個 **靜態強型別 (Statically Typed)** 語言。這意味著編譯器在編譯時就知道變數是 `int` 還是 `string`。
這帶來了極致的效能，但也帶來了僵化。

想像你要寫一個 `json.Unmarshal` 函數：
*   你根本不知道使用者會傳什麼 Struct 給你。
*   你也不知道那個 Struct 有哪些欄位 (Fields)。

這時候，你需要一種能力，讓你能在 **Run time (執行期)** 檢查變數的型別與結構。這就是 **反射 (Reflection)**。

---

## 2. 第二樂章：基石 (The Foundation - eface)

我們在 3.13 章學過，`interface{}` (空介面) 的底層結構是：
```go
type eface struct {
    _type *_type          // 指向型別元數據 (Metadata)
    data  unsafe.Pointer  // 指向實際資料
}
```

**這就是反射的一切根源！**
當你呼叫 `reflect.TypeOf(x)` 或 `reflect.ValueOf(x)` 時，Go 其實只是把這個變數轉成 `eface`，然後把裡面的 `_type` 和 `data` 拿出來給你看而已。

---

## 3. 第三樂章：定律 (The Three Laws)

Rob Pike 曾提出反射的三大定律，我們用白話文來解釋：

### 第一定律：Interface ➜ Reflection Object
> Reflection goes from interface value to reflection object.

你可以把一個變數轉換成 `reflect.Type` (看型別) 或 `reflect.Value` (看/改數值)。

```go
var x float64 = 3.14
t := reflect.TypeOf(x)  // 獲取 _type
v := reflect.ValueOf(x) // 獲取 data
fmt.Println("Type:", t) // float64
fmt.Println("Value:", v) // 3.14
```

### 第二定律：Reflection Object ➜ Interface
> Reflection goes from reflection object to interface value.

你可以把 `reflect.Value` 變回 `interface{}`，然後再 Assert 回原始型別。

```go
y := v.Interface().(float64) // 變回 3.14
```

### 第三定律：要修改，必須傳指針 (Settability)
> To modify a reflection object, the value must be settable.

這是最多人踩坑的地方。

```go
var x float64 = 3.14
v := reflect.ValueOf(x)
v.SetFloat(7.1) // Panic! reflect: reflect.Value.SetFloat using unaddressable value
```

**為什麼？**
因為 `reflect.ValueOf(x)` 傳遞的是 `x` 的 **副本 (Value Copy)**。
如果你在反射裡改了這個副本，外面的 `x` 根本不會變。Go 認為這是在做白工，所以直接 Panic 禁止你這麼做。

**正確做法 (傳指針)**：
```go
v := reflect.ValueOf(&x) // 傳指針!
p := v.Elem()            // 透過 Elem() 拿到指針指向的實體
p.SetFloat(7.1)          // 成功！x 變成了 7.1
```
這其實跟函式呼叫 `func(x *int)` 是一模一樣的道理，只是換成了反射的語法。

---

## 4. 第四樂章：內幕 (The Inside - Under the Hood)

當我們用反射讀取 Struct 的欄位時，到底發生了什麼？

```go
type User struct {
    Name string
    Age  int
}
u := User{Name: "Frankie", Age: 30}
v := reflect.ValueOf(u)
name := v.Field(0) // 讀取第一個欄位
```

**底層動作**:
1.  **取得偏移量 (Offset)**: `v` 裡的 `_type` 包含了 `User` 結構的元數據。Runtime 查表發現 Field 0 (Name) 的 Offset 是 0。
2.  **指針運算**: Runtime 用 `unsafe.Pointer` 計算：`base_address + offset`。
3.  **讀取**: 從那個記憶體地址讀出 16 bytes (String Header)。
4.  **包裝**: 把讀出來的資料包裝成一個新的 `reflect.Value` 回傳給你。

**這就是為什麼反射慢！**
普通的 `u.Name` 是一條簡單的 CPU 指令 (Load)；而 `v.Field(0)` 是一連串的查表、計算、封裝，這中間差了幾百倍的效能。

---

## 5. 第五樂章：實戰 (The Application - JSON)

最經典的反射應用就是 JSON 序列化：

```go
type User struct {
    Name string `json:"user_name"` // Struct Tag
}
```

當 `json.Marshal` 跑起來時：
1.  它用反射遍歷 `User` 的所有欄位。
2.  它讀取 `StructTag` (`json:"user_name"`)。這其實只是一個字串，存在 Type Metadata 裡。
3.  它解析這個字串，發現 key 應該叫 `user_name`。
4.  它利用反射讀取 `Name` 的值，拼出 JSON 字串。

這就是為什麼所有 Go 的 ORM (GORM) 和 Web 框架 (Gin Binding) 都極度依賴反射。它們是用 **Runtime 的效能** 換取 **開發的靈活性**。

#### 反射的效能救星：緩存 (Caching)
既然反射慢，為什麼 GORM 還敢用？
答案是 **Schema Caching**。
1.  **只做一次**: 當程式第一次用到 `User` Struct 時，GORM 會用反射把所有欄位的 Offset 和 Tag 解析出來，存進一個 Map 裡。
2.  **之後查表**: 第二次用到 `User` 時，直接查表拿 Offset，進行快速的記憶體操作 (類似 Unsafe Pointer)。
3.  **IO 才是瓶頸**: 資料庫查詢通常要 5ms，而反射緩存後可能只花 0.01ms。所以在 IO-Bound 應用中，這點損耗是可以接受的。

**極致效能派**:
如果你連這 0.01ms 都要省，可以使用 **程式碼生成 (Code Generation)** 工具 (如 `sqlc` 或 `easyjson`)。它們在編譯時就生成好解析代碼，完全不使用反射，效能等同手寫。

---

## 6. 第六樂章：謝幕 (Series Finale)

至此，**Go Memory & Runtime Series** 全書完結！

我們從最底層的 **Hello World (Book 1)** 開始，一路探索了：
*   **Memory**: Stack, Heap, Allocator, GC。
*   **Concurrency**: GMP, Netpoller, Sysmon。
*   **Data Structures**: Slice, Map, Channel, Interface。
*   **Optimization**: Pointer, Escape Analysis, Defer, Reflection。

您現在已經具備了 **穿透語法糖，直視記憶體本質** 的能力。
這不僅僅是學會了 Go，更是學會了如何像電腦一樣思考。

**Congratulations! You are now a Go Runtime Expert.**
