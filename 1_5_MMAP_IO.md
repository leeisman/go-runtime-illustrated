# Book 1.5: 記憶體映射 I/O：記憶體與硬碟的時空隧道 (Memory Mapped I/O / mmap)

在高效能的 IO 編程中 (資料庫、Kafka、RocketMQ)，**mmap** 是一個被反覆提及的神級優化。它打破了我們對「讀檔」和「寫檔」的傳統認知。

---

## 1. 第一樂章：傳統 I/O vs mmap

### 1.1 傳統 I/O (`read` / `write`)
這是一次「搬家」的過程。
```go
buf := make([]byte, 1024)
file.Read(buf)
```
1.  **Context Switch**: 切換到 Kernel Mode。
2.  **Copy**: Kernel 把 Page Cache 的資料 **複製** 到 User Space 的 `buf`。
3.  **Return**: 切換回 User Mode。
*   **本質**: Data Copying (資料搬運)。

### 1.2 Memory Mapped I/O (`mmap`)
這是一次「開窗」的過程。
```go
data, _ := syscall.Mmap(fd, 0, 1024, ...)
data[0] = 'A' // 直接修改
```
1.  **Mapping**: OS 修改 **Page Table (分頁表)**，將 User Space 的一段虛擬地址 (Virtual Address) **直接映射** 到 Kernel Space 的 Page Cache (Physical Address)。
2.  **Access**: 當 App 讀寫 `data[0]` 時，CPU 直接操作 Page Cache。
*   **本質**: Data Sharing (資料共享)。

---

## 2. 第二樂章：寫下去後，OS 怎麼知道要存檔？(The Dirty Bit Magic)

您問到：「我只是改了記憶體 `data[0] = 'A'`，OS 沒介入，它怎麼知道這頁髒了？」

答案在 **CPU 硬體 (MMU)** 身上。

### 2.1 The Hardware Hook (硬體勾子)
CPU 的 MMU (Memory Management Unit) 在執行 `Store` 指令 (寫入記憶體) 時，會自動做這件事：
1.  查找 Page Table 找到物理地址。
2.  **檢查 Access Bit**: 如果是第一次訪問，設為 1。
3.  **設定 Dirty Bit (D-Bit)**: **MMU 硬體** 會自動將該 Page Table Entry (PTE) 的 **D 位元** 設為 `1`。
    *   這是一個 **硬體行為**，不需要 OS 軟體參與，所以速度極快。

### 2.2 The OS Scan (軟體掃描)
OS 並不知道您「這一微秒」寫了什麼，但它有背景線程來檢查：
*   **`pdflush` / `flush` threads**: 這是 Linux Kernel 的清潔工。
*   **觸發時機**: 
    1.  定時 (預設 30秒, `dirty_expire_centisecs`)。
    2.  髒頁太多 (超過 `vm.dirty_background_ratio`)。
    3.  User 主動呼叫 `msync()` 或 `fsync()`。

當清潔工醒來時，它會遍歷 Page Table：
*   看到 **D-Bit = 1**？ -> 這是 **Dirty Page**。
*   **Action**: 把這頁的內容 `DMA` 到硬碟對應的扇區 (Block)。
*   **Clear**: 把 D-Bit 清除為 0 (變回 Clean Page)。

---

## 3. 第三樂章：Go 實戰：使用 syscall.Mmap

在 Go 中，我們通常不直接用 `syscall`，而是用封裝好的庫 (如 `gommap` 或 `etcd/pkg/fileutil`)，但原理一樣。

```go
package main

import (
    "fmt"
    "os"
    "syscall"
)

func main() {
    // 1. 開檔 (必須是 ReadWrite)
    f, _ := os.OpenFile("data.bin", os.O_RDWR|os.O_CREATE, 0644)
    f.Truncate(1024) // 檔案必須有實體大小，不能映射空檔
    defer f.Close()

    // 2. 建立映射 (Map)
    // PROT_WRITE: 允許寫入
    // MAP_SHARED: 寫入會同步回檔案 (如果是 MAP_PRIVATE 則是 Copy-On-Write)
    data, err := syscall.Mmap(int(f.Fd()), 0, 1024, syscall.PROT_READ|syscall.PROT_WRITE, syscall.MAP_SHARED)
    if err != nil {
        panic(err)
    }
    defer syscall.Munmap(data) // 必定要解影射

    // 3. 像操作 Slice 一樣操作檔案
    data[0] = 'H'
    data[1] = 'i'
    
    // 此時，該 Page 在 RAM 中變髒 (Dirty)。
    // 硬碟上的檔案還沒變，直到 OS 刷盤。

    // 4. (選用) 強制刷盤
    // 相當於 fsync，會卡住直到寫入完成
    // flags: MS_SYNC (同步), MS_ASYNC (異步)
    syscall.Msync(data, syscall.MS_SYNC)
    
    fmt.Println("Written to file via memory!")
}
```

---

## 4. 第四樂章：mmap 不是萬靈丹 (Drawbacks)

雖然 mmap 少了 Copy，但它也有代價：

1.  **Page Fault (缺頁中斷) 的不可預測性**:
    *   當您讀取 `data[i]` 時，如果該 Page 不在 RAM 裡 (被 Swap 出去了，或還沒載入)，CPU 會觸發 **Major Page Fault**。
    *   這會導致當前 Thread **被暫停 (Block)**，直到硬碟讀取完成。
    *   **風險**: 這比標準 I/O 更難預測 Latency。標準 I/O 您知道 `read()` 可能會卡，但 `mmap` 的隨機讀取可能隨時卡。

2.  **TLB Shootdown (多核同步成本)**:
    *   修改 Mapping 需要同步所有 CPU Core 的 TLB (Translation Lookaside Buffer)。在極高併發下有額外開銷。

3.  **Size Limit**:
    *   在 32-bit 系統上很慘。但在 64-bit 系統上，Address Space 很大，通常不是問題。但 Go/Java 的 Array index 還是 int (2GB限制)，所以 RocketMQ 才會切分 CommitLog。

---

## 5. 第五樂章：總結 (Summary)

| Feature | Standard I/O (`read/write`) | Memory Mapped (`mmap`) |
| :--- | :--- | :--- |
| **System Calls** | 每次讀寫都要 Syscall | 僅 Setup 時一次，之後無 Syscall |
| **Data Copy** | 2次 (Disk -> Kernel -> User) | 1次 (Disk -> Kernel/User Shared) |
| **CPU Usage** | 高 (搬運工) | 低 (直接存取) |
| **Latency** | 穩定 (但慢) | 極快 (但可能有 Page Fault 抖動) |
| **適用場景** | Log 追加、Socket 傳輸 | DB 隨機讀寫、MQ 索引檔 |
