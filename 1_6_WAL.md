# Book 1.6: WAL (Write-Ahead Logging - 先寫日記再整理房間)

資料庫和訊息系統最怕的不是慢，而是「寫到一半突然斷電」。
WAL 要解決的核心問題是：如何先用一條順序日誌保住事實，再慢慢整理真正的資料結構。

---

## 1. 第一樂章：為什麼需要 WAL？(Random I/O vs Sequential I/O)

在 Book 1.1 我們學過，硬碟（不管是 HDD 還是 SSD）做 **Random I/O (隨機讀寫)** 的速度，遠遠慢於 **Sequential I/O (順序讀寫)**。

| 操作 | HDD | SSD |
| :--- | :--- | :--- |
| **Random Write** | ~100 IOPS (≈ 10ms/次) | ~10,000 IOPS |
| **Sequential Write** | ~100 MB/s | ~500 MB/s |

差距可以高達 **100~1000 倍**。

這帶來了一個殘酷的矛盾：
*   **資料庫的資料檔 (Data File)**：資料散佈在硬碟的各個角落。更新一行資料 = 跳到某個隨機位置寫入 = **Random I/O (極慢)**。
*   **使用者的期望**：我 `UPDATE` 完要立刻 Commit，你不能讓我等 10ms 的磁碟跳轉。

**WAL 的天才解法**：
> 別急著把資料寫回它在硬碟上的「正確位置」(Random)。
> 先把「我做了什麼修改」這個動作，**追加 (Append)** 到一個專用的日誌檔案尾端 (Sequential)。
> 等日誌安全落地後，才向用戶回報 Commit 成功。
> 至於真正的資料檔，由背景程序慢慢補寫 (Checkpoint)。

**本質**：WAL 就是一台把 **Random I/O 轉化為 Sequential I/O** 的變壓器。

---

## 2. 第二樂章：底層系統呼叫 (How It Works at OS Level)

WAL 在作業系統層面，歸根結底就是三個系統呼叫的組合技：

### Step 1: `open()` - 打開日誌檔案
```c
// O_WRONLY: 只需要寫入權限
// O_APPEND: 每次寫入都自動定位到檔案尾端 (保證順序追加)
// O_CREAT:  檔案不存在就建立
int fd = open("redo.log", O_WRONLY | O_APPEND | O_CREAT, 0644);
```
**`O_APPEND` 是 WAL 的靈魂旗標**。它告訴整個作業系統：「我永遠只往後寫，絕對不會回頭改。」
這讓整個作業系統的 I/O 排程器可以把寫入請求合併成一條連續的磁碟操作，效能飆升。

### Step 2: `write()` - 寫入修改紀錄
```c
// 把修改動作序列化成 binary 紀錄，追加到日誌尾端
// 例如："Page 42, Offset 128, Old=25, New=31"
write(fd, record_bytes, record_len);
```
此時資料可能只寫進了作業系統的 **Page Cache (記憶體緩衝區)**，還沒有真正落到實體硬碟上。

### Step 3: `fsync()` - 強制刷盤 (The Point of No Return)
```c
// 強制作業系統把 Page Cache 裡的資料，物理性地寫入硬碟磁盤
fsync(fd);
```
**`fsync()` 是 WAL 的生死關卡**。只有當 `fsync()` 成功返回，我們才能 100% 確認這條日誌已經永久刻在硬碟上了。即使下一秒停電，這條日誌也不會消失。

### Go 語言的對應寫法
```go
// Step 1: 打開檔案 (O_APPEND + O_SYNC 可選)
f, err := os.OpenFile("wal.log", os.O_WRONLY|os.O_APPEND|os.O_CREATE, 0644)
defer f.Close()

// Step 2: 追加寫入
f.Write(recordBytes)

// Step 3: 強制刷盤
f.Sync() // 底層就是 fsync()
```

---

## 3. 第三樂章：誰在用 WAL？(Industry Applications)

WAL 不只是一個理論概念，它是整個現代基礎設施的地基：

### 1. MySQL InnoDB - Redo Log
*   **流程**：`UPDATE` → 修改 Buffer Pool (記憶體) → 寫入 Redo Log (Sequential) → `fsync` → 回報 Commit 成功。
*   **Checkpoint**：背景線程定期把 Dirty Page 從記憶體刷回資料檔 (Random I/O)。
*   **Crash Recovery**：重開機後，讀取 Redo Log，把還沒刷到資料檔的修改重新執行一遍。

### 2. Apache Kafka - Commit Log
*   **流程**：Producer 送來訊息 → 直接 `Append` 到 Partition 的 Log 檔案尾端 → `fsync` → 回報 ACK。
*   **為什麼 Kafka 這麼快？** 因為它的資料結構就是一條純粹的 Append-Only Log。所有的寫入都是 Sequential I/O，完全沒有 Random I/O。
*   **消費者怎麼讀？** 靠 Offset (偏移量) 直接定位到檔案中的精確位置，用 `sendfile()` (零拷貝) 直接從 Page Cache 送到網路 Socket。

### 3. Redis - AOF (Append Only File)
*   **流程**：執行 `SET key value` → 把這條指令文字追加到 AOF 檔案尾端。
*   **三種刷盤策略**：
    *   `always`：每條指令都 `fsync` (最安全，最慢)。
    *   `everysec`：每秒 `fsync` 一次 (折衷，最多丟 1 秒資料)。
    *   `no`：交給作業系統決定何時刷盤 (最快，但停電可能丟資料)。

### 4. Etcd / PostgreSQL - WAL
*   **Etcd (Raft 共識協議)**：所有的狀態變更都先寫入 WAL，再 Apply 到狀態機。這保證了分散式節點之間的資料一致性。
*   **PostgreSQL**：與 MySQL 原理相同，使用 WAL 來保證 ACID 的 Durability (持久性)。

---

## 4. 第四樂章：fsync 的代價與權衡 (The Trade-off)

`fsync()` 雖然是安全的保證，但它是 **昂貴的**。
*   **HDD**：一次 `fsync` 可能需要 5~10ms (磁頭要物理移動到位)。
*   **SSD**：一次 `fsync` 大約 0.1~1ms。

所以業界通常不會「每一筆資料都 fsync」，而是採用 **Group Commit (批次提交)** 策略：
> 收集 1ms 內的所有修改，打包成一個大的 Block，只做 **一次 `fsync`**。
> 這樣 1000 筆修改共用了一次磁碟操作，單筆成本被攤薄到接近 0。

MySQL InnoDB 的 `innodb_flush_log_at_trx_commit` 參數就是在控制這件事：
*   `= 1`：每次 Commit 都 `fsync` (最安全，預設值)。
*   `= 2`：Commit 時只寫到 OS Buffer，每秒 `fsync` 一次 (折衷)。
*   `= 0`：完全交給 OS，最多丟 1 秒 (最快，但最危險)。

---

## 5. 第五樂章：一句話總結 (Finale)

> **WAL 的本質：用 Sequential I/O 的日誌先保住資料安全，再用背景程序慢慢做 Random I/O 的落盤。它是所有現代資料庫、訊息佇列、分散式系統的「持久性 (Durability)」基石。**

底層系統呼叫只有三招：`open(O_APPEND)` → `write()` → `fsync()`。
萬變不離其宗。
