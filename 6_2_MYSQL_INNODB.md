# Book 6.2: MySQL InnoDB：地底下的國家檔案局 (MySQL InnoDB)

前面我們聊的是「如何在 Go 應用層優雅地把請求送過去（連線池 & Context）」。
現在我們終於要推開 MySQL 這座「外國領事館」的大門，走進他們地底下的 **「國家檔案局 (Storage Engine)」** 了。

---

## 1. 第一樂章：核心引擎 InnoDB (全能檔案局)

在早期的 MySQL (5.5 以前)，預設引擎是 **MyISAM**。它追求極致的讀取速度，但缺點是：
*   **不支援 Transaction (事務)**。
*   **鎖的粒度是表鎖 (Table Lock)**：如果有人在修改第 1 頁，整本書都會被鎖起來不准別人看。這在微服務高併發場景下是災難。

現代 MySQL 最常用、也幾乎是唯一指定的預設引擎是 **InnoDB**。
它之所以強大，是因為它具備了企業級檔案局的 **四大護法**：

### 1. Row-Level Locking (行級鎖)
只鎖住你正在修改的 **那一行**，其他人可以自由修改同一張表的其他行。
大幅提升併發寫入能力。

### 2. MVCC (多版本並發控制)
讀寫分離的極致。當你正在修改某一行時，InnoDB 會利用 **Undo Log** 保留一個「舊版本」。
這時其他來查詢的讀者，可以直接讀取舊版本，達成 **「讀不阻塞寫，寫不阻塞讀」**。

### 3. Buffer Pool (記憶體緩衝池)
InnoDB 其實是一個高度依賴記憶體的怪獸。它會把常用的硬碟資料 (Page) 快取在記憶體裡。
你以為你在查硬碟，其實 **90% 以上的查詢都是在記憶體裡完成的**。

### 4. Redo Log (重做日誌 - WAL)
為了效能，修改其實是先寫在記憶體裡的。萬一突然停電怎麼辦？
InnoDB 會先把修改的「動作」循序寫入實體的 **Redo Log (Sequential I/O 極快)**。
只要 Log 在，重開機就能把資料救回來。這就是 **WAL (Write-Ahead Logging)** 技術。

---

## 2. 第二樂章：索引的本質 (B+ Tree - The Filing System)

MySQL 之所以能在上億筆資料中 **O(log N)** 秒殺找到你要的那一行，靠的就是 **B+ Tree 索引**。

### 為什麼是 B+ Tree 而不是 Binary Tree？
如果用普通的二元樹 (Binary Search Tree)，1 億筆資料的樹高約 **27 層**。
每一層代表一次硬碟 I/O (隨機讀取 ≈ 10ms)。27 次 × 10ms = **270ms**。太慢了。

B+ Tree 的核心思想是：**把樹壓扁，讓每個節點塞滿一整頁 (16KB Page) 的資料。**
*   每個節點可以存放約 **1000 個指標** (取決於 Key 大小)。
*   三層 B+ Tree = 1000 × 1000 × 1000 = **10 億筆資料**。
*   也就是說，查找 10 億筆資料，最多只需要 **3 次硬碟 I/O**。

### Clustered Index (聚簇索引) vs Secondary Index (二級索引)

#### Clustered Index (主鍵索引 / 聚簇索引)
*   InnoDB 的表資料 **就是按照主鍵 (Primary Key) 排列的 B+ Tree**。
*   葉節點 (Leaf Node) 存放的是 **完整的整行資料 (Row Data)**。
*   所以用主鍵查詢是最快的，直接命中葉節點拿到所有欄位。

#### Secondary Index (二級索引 / 非主鍵索引)
*   例如你在 `name` 欄位建了索引。
*   這棵 B+ Tree 的葉節點存的 **不是整行資料**，而是 **主鍵值 (Primary Key)**。
*   所以用 `name` 查詢時，要先在 Secondary Index 找到主鍵，再回到 Clustered Index 去拿完整資料。
*   這個過程叫做 **回表 (Table Lookup / Bookmark Lookup)**。

#### Covering Index (覆蓋索引 - 避免回表)
如果你的查詢所需要的欄位，剛好都包含在索引裡面，就不需要回表了。
```sql
-- 假設有聯合索引 idx_name_age (name, age)
SELECT name, age FROM users WHERE name = 'Frankie';
-- name 和 age 都在索引的葉節點裡，不需要回表！極速！
```

---

## 3. 第三樂章：鎖的機制 (Lock Granularity)

### Row Lock 的真相：鎖的是索引！
很多人以為 InnoDB 鎖的是「資料行」。**錯！InnoDB 鎖的是索引記錄 (Index Record)。**

這帶來了一個致命的坑：
```sql
-- 假設 name 欄位「沒有索引」
UPDATE users SET age = 30 WHERE name = 'Frankie';
```
因為 `name` 沒有索引，InnoDB 無法精確定位到哪一行。
它只能做 **全表掃描 (Full Table Scan)**，在掃描的過程中，它會把 **每一行都鎖住**。
結果：**行級鎖退化成了表級鎖！** 整張表被鎖死，併發歸零。

**教訓**：高併發場景下，`WHERE` 條件的欄位 **一定要有索引**。

### Gap Lock & Next-Key Lock (間隙鎖)
在 `REPEATABLE READ` 隔離級別下，InnoDB 不只鎖住現有的行，還會鎖住 **行與行之間的間隙 (Gap)**。
*   **目的**：防止幻讀 (Phantom Read)。避免你在同一個 Transaction 裡面同樣的 `SELECT` 兩次，卻因為別人 INSERT 了新資料而拿到不同的結果。
*   **代價**：可能導致意想不到的鎖等待 (Lock Wait) 和死鎖 (Deadlock)。

---

## 4. 第四樂章：MVCC 深潛 (Read Without Locking)

MVCC 是 InnoDB 實現高併發讀取的靈魂。它的原理是：**每一行資料都有隱藏的版本號。**

每一行隱藏了兩個欄位：
*   **`DB_TRX_ID`**：最後一次修改這行的 Transaction ID。
*   **`DB_ROLL_PTR`**：指向 **Undo Log** 中這行的舊版本 (形成一條版本鏈)。

### ReadView (快照)
當你開啟一個 `SELECT` 查詢時，InnoDB 會生成一個 **ReadView (快照)**。
這個快照記錄了「此刻有哪些 Transaction 正在進行中 (尚未 Commit)」。

讀取規則：
1.  如果這行的 `DB_TRX_ID` 是在我開始之前就 Commit 的 → **可見** (讀取它)。
2.  如果這行的 `DB_TRX_ID` 是在我之後才開始的 → **不可見** (沿著 `DB_ROLL_PTR` 去 Undo Log 找舊版本)。
3.  如果是正在進行中的 Transaction 修改的 → **不可見** (一樣找舊版本)。

**效果**：讀者永遠不需要等待寫者鎖釋放。它自己跳進時光機 (Undo Log)，拿到那一刻的歷史版本就走了。

---

## 5. 第五樂章：WAL 與 Crash Recovery (停電不怕)

### 寫入流程 (Write Path)
當你執行 `UPDATE users SET age = 31 WHERE id = 1` 時：
1.  **修改 Buffer Pool**：在記憶體中找到 Page，修改 `age` 為 31。此時這個 Page 變成 **Dirty Page (髒頁)**。
2.  **寫入 Redo Log Buffer**：把「Page X, Offset Y, 修改為 31」這個動作記錄下來。
3.  **Flush Redo Log to Disk**：Transaction Commit 時，Redo Log 被 **fsync** 到硬碟 (Sequential I/O，極快)。
4.  **後台刷髒頁**：由背景線程 (Checkpoint) 慢慢把 Dirty Page 寫回硬碟的資料檔 (Random I/O，較慢)。

### 為什麼先寫 Log 而不是直接寫資料？
*   **資料檔**：存在硬碟的各個角落 (Random I/O)，每次寫要花大約 **10ms**。
*   **Redo Log**：只追加在檔案尾端 (Sequential I/O)，每次寫只需 **0.01ms**。

這就是 **WAL (Write-Ahead Logging)** 的精髓：
> 「先把動作記在日記本上 (Sequential)，再慢慢把房間整理好 (Random)。就算整理到一半停電了，開機照著日記本重做一遍就好。」

---

## 6. 第六樂章：架構全景圖 (The Big Picture)

```text
┌─────────────────────────────────────────────┐
│                Go Application               │
│    sql.DB (Connection Pool + Context)       │
└──────────────────┬──────────────────────────┘
                   │ TCP (MySQL Protocol)
┌──────────────────▼──────────────────────────┐
│              MySQL Server                    │
│  ┌─────────────────────────────────────┐    │
│  │         Query Parser / Optimizer     │    │
│  └──────────────┬──────────────────────┘    │
│  ┌──────────────▼──────────────────────┐    │
│  │           InnoDB Engine              │    │
│  │  ┌───────────┐  ┌────────────────┐  │    │
│  │  │Buffer Pool│  │   Redo Log     │  │    │
│  │  │ (Memory)  │  │ (WAL on Disk)  │  │    │
│  │  └───────────┘  └────────────────┘  │    │
│  │  ┌───────────┐  ┌────────────────┐  │    │
│  │  │ Undo Log  │  │  B+ Tree Data  │  │    │
│  │  │  (MVCC)   │  │   (On Disk)    │  │    │
│  │  └───────────┘  └────────────────┘  │    │
│  └─────────────────────────────────────┘    │
└─────────────────────────────────────────────┘
```

從 Go 的連線池 (Book 6.1) 到 MySQL 的 B+ Tree 葉節點，整條鏈路一覽無遺。
理解了這些底層機制，你在設計高併發系統時，就能精準地判斷：
*   **瓶頸在連線池？** 調 `MaxOpenConns`。
*   **瓶頸在索引？** 檢查是否觸發了全表掃描導致行鎖退化。
*   **瓶頸在磁碟 I/O？** 確認 Buffer Pool 是否夠大 (命中率)。
*   **瓶頸在鎖競爭？** 審視 Transaction 的範圍是否太大、是否有 Gap Lock 死鎖。
