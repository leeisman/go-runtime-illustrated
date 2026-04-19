# Book 2.5: The Type System (型別的物理本質)

型別不是編譯器拿來嚇人的標籤，而是資料在記憶體裡如何被解讀的規則。
這一章從 concrete type 開始，慢慢走到 interface，理解 Go 為什麼能同時保有靜態安全與動態分派。

---

## 1. 第一樂章：裸體數據 (Naked Data)

在 Go 語言中，當你定義一個具體型別 (Concrete Type) 的變數時，它在記憶體中是如何存在的？

```go
var age int64 = 18
var u User = User{id: 1}
```

**物理真相**:
*   `age`: 記憶體裡就只有 **8 bytes** 的 `0x12`。
*   `u`: 記憶體裡就只有 struct 的欄位數據。

**關鍵點**: 
這些變數身上 **沒有** 任何隱藏欄位用來記錄「我是 int64」或「我是 User」。它們是 **裸體 (Naked)** 的。Runtime 在執行時，看著這塊記憶體，其實分不出來它是什麼。

**那誰知道？**
**編譯器 (Compiler)** 知道。
因為編譯器是全知的上帝，它在編譯代碼時，已經把所有對 `age` 的操作都寫死成「對 8 bytes 整數的操作」指令 (如 `MOV`, `ADD`)。Runtime 不需要知道型別，只要照著指令做動作就好。

---

## 2. 第二樂章：靜態轉換 (Static Conversion)

這是一個常見的誤區：**「轉型只是換個方式看記憶體嗎？」**
答案是：對於**數值型別 (`int(f)`)**，絕對不是！記憶體內容會發生劇烈變化。

```go
var f float64 = 3.14
var i int = int(f) // 強制轉型
```

**記憶體視圖 (Bit Changes)**:
這是 **CPU 運算** 的結果，涉及物理重組。

| 變數 | 值 | 記憶體內容 (Hex) | 發生了什麼？ |
| :--- | :--- | :--- | :--- |
| **f** | `3.14` | `0x40091EB851EB851F` | (IEEE 754 浮點數格式) |
|      |        | **↓ CPU 指令 (CVT) ↓** | **位元被徹底打散重組** |
| **i** | `3`    | `0x0000000000000003` | (2 的補數整數格式) |

**(對比) 如果是 Unsafe Pointer 轉型**:
```go
// 這是我們 Book 2.4 教的黑魔法
u := *(*int)(unsafe.Pointer(&f))
```
這時記憶體內容才 **真的沒變**。但結果是：
*   **u** 的值變成：`4614253070214989087` (即 `0x4009...` 的整數解讀)。這就是**亂碼**。

**結論**:
Go 語言標準的 `T(x)` 轉型，是 **「值 (Value)」的轉換**，它會生成 CPU 指令來改變記憶體的 Bits，確保轉換後的數值在數學上是有意義的。

---

## 3. 第三樂章：穿上制服 (The Interface Wrapper)

什麼時候 Runtime 才需要知道型別？當我們使用 **Interface** 時。
這是 Go 語言中唯一會攜帶「動態型別資訊」的時刻。

```go
var any interface{} = age
```

**物理變化**:
這時候發生了 **Boxing (裝箱)**。`any` 這個變數在記憶體中不再是 8 bytes，而是 **16 bytes** 的 Fat Pointer：

| Offset | 內容 | 意義 |
| :--- | :--- | :--- |
| **+0** | **`_type` 指針** | **識別證 (Badage)**：指向 `int64` 的型別描述表。 |
| **+8** | **`data` 指針** | **數據 (Payload)**：指向儲存 `18` 的記憶體地址。 |

**關鍵點**:
只有變成了 Interface，數據才擁有了 **「自我介紹」** 的能力。因為它身上多帶了一個 `_type` 指針。

---

## 4. 第四樂章：動態識別 (Dynamic Assertion)

這就解釋了為什麼 **Type Assertion** (`x.(T)`) 只能用在 Interface 上。
但這還有分兩種情況：**驗明正身 (Identity)** 與 **技能檢定 (Capability)**。

### 4.1 驗明正身 `i.(User)` (Concrete Assertion)
這是最簡單的檢查，問的是 **「你是誰？」**。

```go
var i interface{} = &User{id: 1}
u := i.(*User) // 檢查你是不是 *User 指針
```

**底層動作**:
Runtime 只是簡單地比對這兩個指針地址：
`if i._type == type(*User) { return true }`
這是一次 **O(1) 的整數比較**，極快。就跟檢查身分證一樣。

### 4.2 技能檢定 `i.(io.Reader)` (Interface Assertion)
這就複雜了，問的是 **「你會不會這項技能？」** (不管你是誰)。

```go
var i interface{} = &File{} // File 是一個具體型別，它有 Read 方法
r := i.(io.Reader)          // 檢查你有沒有實作 Reader 接口
```

**底層動作 (itab check)**:
Runtime 必須檢查 `File` 具體型別是否擁有 `io.Reader` 規定的所有方法 (`Read`)。
1.  **查表**: 檢查 cache 裡有沒有 `(File, Reader)` 的紀錄 (`itab`)。
2.  **比對**: 如果沒有，就逐一比對 `File` 的方法表和 `Reader` 的方法表。
    *   **真實案例**: USB 裝置插上電腦時。
    *   **Identity**: 電腦檢查 device ID，「喔，你是三星 SSD T5」。
    *   **Skill**: 電腦檢查 capabilities，「喔，你支援 `Mass Storage` 協議，那我就把你當硬碟用」。(電腦不在乎你是三星還是 SanDisk，只要你會 `Read/Write` 技能就好)。

所以 **Skill Check** 比 Identity Check 稍微昂貴一點 (雖然有 cache)，因為它涉及了方法集的匹配。

---

## 5. 第五樂章：總結 (Finale)：為什麼沒有 `int(interface_var)`？

這是一個常見的新手誤區：

```go
var i interface{} = 100
var n int = int(i) // 錯誤！
```

為什麼不能這樣寫？
*   **`int(x)` 是靜態轉換**: 編譯器必須在編譯時就知道 `x` 的具體數據格式，才能生成 CPU 轉換指令。但編譯器只知道 `i` 是一個 Interface，不知道裡面裝了什麼。
*   **`i.(int)` 是動態斷言**: 這才是正確寫法。你需要請 Runtime 去檢查 `i` 的識別證，確認它是 int 後，才能把值取出來。

**結論**:
*   **具體變數**: 裸體數據，由編譯器靜態管理，效能最高，但無法動態識別。
*   **介面變數**: 數據 + 識別證，由 Runtime 動態管理，靈活性最高，但有額外開銷 (16 bytes)。
