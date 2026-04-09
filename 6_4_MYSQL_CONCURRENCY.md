# Book 6.10: MySQL 高併發架構心法 (架構師的最終兵器)

在 Book 6.8 我們認識了 InnoDB 的四大護法，Book 6.9 我們走過了 INSERT/UPDATE 的完整旅程。
現在我們把視角再拉高，從「微觀的 Row 隱藏欄位」到「宏觀的架構抉擇」，一氣呵成。

---

## 1. 微觀解剖：資料的物理單位與隱藏基因

### 傳輸單位是 16KB Page
MySQL 管理資料的最小物理單位是 **16KB** 的資料頁 (Page)。
修改一行字，整張 16KB 都會變成「髒頁 (Dirty Page)」並一起刷回硬碟。

### 每一行 (Row) 都有隱藏欄位
你宣告的表格可能只有 `id, name, age`，但 InnoDB 在底層偷偷幫每一行加了兩個隱藏基因：

| 隱藏欄位 | 用途 |
| :--- | :--- |
| **`DB_TRX_ID`** (交易 ID) | 這行資料「最後一次」被誰 (Transaction ID) 寫入或修改的印章 |
| **`DB_ROLL_PTR`** (回滾指標) | 指向 Undo Log 的纜線，用來找回修改前的舊資料 |

這兩個隱藏欄位，就是 MVCC 和 Rollback 的全部根基。

---

## 2. 讀取的藝術：MVCC (快照讀)

普通的 `SELECT` 不會上鎖，而是依賴 **MVCC** 來實現「讀寫不衝突」。

### Read View (活躍黑名單)
執行 `SELECT` 的瞬間，MySQL 會產生一份當下 **「還沒 Commit 的 Transaction ID 名單」**。
這份名單就是 **Read View**，它決定了「哪些資料對你來說是可見的」。

*   **RC (讀已提交 Read Committed)**：每次 `SELECT` 都重新產生一份新的 Read View。
    → 你能看到別人剛剛 Commit 的最新資料。
*   **RR (可重複讀 Repeatable Read)**：整個 Transaction 只有第一次 `SELECT` 會產生，之後死守這份舊的 Read View。
    → 不管別人怎麼改，你看到的世界永遠定格在 Transaction 開始的那一刻。

### 比對與判定 (核心邏輯)
引擎掃描資料時，會拿 Row 身上的 `DB_TRX_ID` 去跟自己的 Read View 比對：

```text
拿到一行資料 →  這行的 DB_TRX_ID 是誰改的？
                    │
        ┌───────────┴───────────┐
        │                       │
  「已 Commit 且在我之前」     「未 Commit 或在我之後」
        │                       │
   ✅ 可見！直接讀取         ❌ 不可見！
                                │
                        順著 DB_ROLL_PTR
                        跳進 Undo Log
                        翻找舊版本
                                │
                        找到屬於我時間點的
                        舊快照，讀取它
```

**效果**：這完美避開了髒讀 (Dirty Read)，讓讀者永遠不需要等寫者的鎖。

---

## 3. 寫入的防禦：鎖的演化與空間封殺 (當前讀)

當執行 `UPDATE`、`DELETE` 或 `SELECT ... FOR UPDATE` 時，會放棄 MVCC 快照，**強制看最新資料並上鎖**。這叫做 **「當前讀 (Current Read)」**。

### Row Lock (行鎖) 的兩種型態

####  隱式鎖 (Implicit Lock) - 省資源
*   **機制**：只要沒人搶，就只靠寫在 Row 上的 `DB_TRX_ID` 佔位子。記憶體裡根本沒有鎖物件存在。
*   **本質**：「你看到這行的 `DB_TRX_ID` 是活躍的 Transaction，就知道有人在用，自己別碰。」
*   **好處**：零記憶體開銷。在沒有衝突的場景下，MySQL 連Lock Manager都不用啟動。

#### 顯式鎖 (Explicit Lock) - 遇衝突
*   **機制**：當別人也想改同一行，發現 `DB_TRX_ID` 還在活躍名單中，就會幫你建立實體鎖物件，然後自己乖乖排隊睡眠 (Wait)。
*   **本質**：衝突觸發的「按需升級」。只有真正打架的時候，才付出鎖的記憶體成本。

### Gap Lock (間隙鎖) - 空間封殺

*   **目的**：物理封殺 **「幻讀 (Phantom Read)」**。
*   **機制**：不只鎖住實體的 Row，還會把你條件區間的 **「Index 空隙」** 全部封死。任何企圖在這區間 `INSERT` 新資料的操作，全部撞牆排隊。
*   **範例**：
    ```sql
    -- RR 隔離級別下
    SELECT * FROM users WHERE age BETWEEN 20 AND 30 FOR UPDATE;
    ```
    InnoDB 不只鎖住 age=20~30 的現有 Row，還鎖住了 20~30 之間所有的「空位」。
    如果有另一個 Transaction 想 `INSERT INTO users (age) VALUES (25)`，它會被 Gap Lock 擋住。

*   **透明人效應**：一般的 `SELECT` (快照讀) 完全無視 Gap Lock。它不申請鎖也不排隊，直接穿牆秒回舊資料 (MVCC)。只有「當前讀」才會被 Gap Lock 影響。

---

## 4. 架構師的實戰抉擇 (Trade-off)

高併發不等於高衝突。鎖的策略完全取決於「業務場景」：

### 樂觀鎖 (CAS 原子操作)
```sql
-- 利用 WHERE 條件做原子判斷，完全不排隊
UPDATE users SET balance = balance - 100 WHERE id = 42 AND balance >= 100;
-- affected_rows == 1 → 成功
-- affected_rows == 0 → 餘額不足，業務層處理
```
*   **本質**：交給 CPU Latch 決定勝負，不進入 Lock Manager。
*   **適合**：極高併發、低衝突（如各自升級裝備、扣庫存）。
*   **缺點**：高衝突場景下會引發重試風暴，反而打死 CPU。

### 悲觀鎖 (FOR UPDATE)
```sql
BEGIN;
SELECT balance FROM users WHERE id = 42 FOR UPDATE;  -- 先鎖住
-- 業務邏輯判斷...
UPDATE users SET balance = balance - 100 WHERE id = 42;
COMMIT;
```
*   **本質**：強制觸發 Row Lock (+ Gap Lock) 排隊。
*   **適合**：高衝突、零容錯（如金流扣款、轉帳）。
*   **缺點**：保護了 CPU，但會拖垮整體吞吐量 (大家在排隊)。

### 頂級大廠的降維打擊
因為 MySQL 的 Gap Lock 範圍難控且極易死鎖 (Deadlock)，最高端的做法是：

> **將 MySQL 降級為 RC 隔離級別 (關閉 Gap Lock 釋放效能)，
> 並把防併發的「排隊邏輯」往上推給 Redis (分散式鎖) 或 Kafka (MQ 序列化) 來扛！**

```text
┌─────────────────────────────────────────────┐
│            Architecture Evolution           │
│                                             │
│  Level 1: MySQL RR + Gap Lock (簡單但易死鎖) │
│      ↓                                      │
│  Level 2: MySQL RC + 樂觀鎖 CAS (高吞吐)    │
│      ↓                                      │
│  Level 3: Redis 分散式鎖 + MySQL RC          │
│           (排隊邏輯上推到 Redis)              │
│      ↓                                      │
│  Level 4: Kafka MQ 序列化 + MySQL RC         │
│           (徹底消滅鎖，用佇列排隊)            │
└─────────────────────────────────────────────┘
```

*   **Level 1 → 2**：關掉 Gap Lock，用 CAS 取代 FOR UPDATE，吞吐量翻倍。
*   **Level 2 → 3**：當 CAS 重試風暴出現，把排隊邏輯交給 Redis `SETNX`，MySQL 只負責最終寫入。
*   **Level 3 → 4**：當 Redis 也扛不住，把所有寫入請求丟進 Kafka Topic，Consumer 端序列消費，MySQL 徹底免鎖。

---

## 5. 終章：從記憶體位址到分散式架構

| 層級 | 元件 | 職責 |
| :--- | :--- | :--- |
| **Row 微觀** | `DB_TRX_ID` + `DB_ROLL_PTR` | MVCC 版本判定 + Undo Log 時光機 |
| **Page 物理** | Buffer Pool + LRU | 記憶體快取 + 冷熱淘汰 |
| **持久化** | Redo Log (WAL) + Checkpoint | 保命日記 + 背景刷盤 |
| **併發控制** | Row Lock + Gap Lock | 寫寫互斥 + 防幻讀 |
| **架構抉擇** | 樂觀鎖 / 悲觀鎖 / Redis / Kafka | 根據衝突密度選擇戰場 |

> **這套從「記憶體位址」貫穿到「分散式架構」的思維，就是支撐千萬級流量系統最堅實的地基。**
