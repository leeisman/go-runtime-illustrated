# Book 6.3: MySQL 五大部門：INSERT & UPDATE 的完整旅程 (MySQL Journey)

在 Book 6.8 我們認識了 InnoDB 的四大護法。
現在我們把所有元件攤開，用一家 **「實體公司」** 的隱喻，完整走過一筆 INSERT 和 UPDATE 從進門到回報成功的所有流程。

---

## 1. 第一樂章：核心部門介紹 (The Cast)

### 1. B+ Tree (目錄導航台)
*   **職責**：查地址。
*   **口頭禪**：「你要找的資料在第幾號資料頁 (Page ID)。」
*   **注意**：它不管資料現在是在桌上還是倉庫，它只負責給出一個 **邏輯地址**。

### 2. Buffer Pool (記憶體辦公桌)
*   **職責**：真實的工作台。
*   **鐵律**：所有的修改與讀取，**絕對只能在這個桌子上進行！** CPU 絕對不去倉庫 (硬碟) 改東西。
*   如果資料不在桌上，就先去倉庫搬上來 (Disk Read)，然後才能動手。

### 3. LRU (辦公桌清理大師)
*   **職責**：管理辦公桌的空間。
*   **動作**：桌子滿了的時候，把最久沒人看的舊資料趕回倉庫 (SSD)，騰出空間給新資料。
*   它決定了每次查詢是 **「快取命中 (Cache Hit)」** 還是 **「快取未命中 (Cache Miss)」**。

### 4. Undo Log (時光機影印機)
*   **職責**：備份舊資料（為了 MVCC 多版本控制，以及出錯時可以 Rollback）。
*   **動作**：只要你要修改 (UPDATE) 資料，**必須先把舊的樣子影印一份放這裡**。
*   **兩大用途**：
    1.  **Rollback**：Transaction 出錯要反悔時，拿影印本還原。
    2.  **MVCC**：其他讀者來查詢時，順著影印本的版本鏈找到屬於自己時間點的舊版本，實現「讀不等寫」。

### 5. Redo Log / WAL (保命流水帳)
*   **職責**：記錄剛剛做了什麼動作，防止突然停電。
*   **動作**：桌上改完資料後，在筆記本尾端寫下「我把 A 改成了 B」。只要這筆記本寫完了 (`fsync`)，就算大功告成 (API 回傳成功)。
*   **為什麼不直接寫資料檔？** 因為資料檔是 Random I/O (慢)，日誌是 Sequential I/O (快 100 倍)。詳見 Book 1.6。

---

## 2. 第二樂章：INSERT 的完整旅程

場景：`INSERT INTO users (id, name, age) VALUES (42, 'Frankie', 30)`

```text
Step 1: B+ Tree 導航
  「id=42 應該放在哪一頁？」
  → 沿著 B+ Tree 從根節點往下走
  → 定位到 Leaf Page #128

Step 2: Buffer Pool 上桌
  「Page #128 在桌上嗎？」
  → 如果不在 (Cache Miss)：去倉庫 (硬碟) 把 Page #128 搬到桌上
  → 如果在 (Cache Hit)：直接用

Step 3: LRU 記錄
  「Page #128 剛剛被碰過，把它移到最前面，別讓清理大師趕走它。」

Step 4: 在桌上動手 (Memory Write)
  → 在 Page #128 裡面找到空位
  → 寫入 (42, 'Frankie', 30)
  → 這一頁現在變成了 Dirty Page (髒頁)

Step 5: Redo Log 寫流水帳 (Sequential I/O)
  → 在日誌尾端追加：「Page #128, Offset 320, INSERT (42, 'Frankie', 30)」
  → fsync() 落地！

Step 6: 回報成功 ✅
  → 向 Go 的 Driver 回傳 OK
  → 此時資料只活在「記憶體 + Redo Log」裡
  → 真正的硬碟資料檔，要等背景 Checkpoint 慢慢補寫
```

> **注意**：INSERT 不需要 Undo Log 嗎？其實需要，但它記錄的很簡單，只是「如果要 Rollback，就把剛剛插入的那行刪掉」。

---

## 3. 第三樂章：UPDATE 的完整旅程

場景：`UPDATE users SET age = 31 WHERE id = 42`

```text
Step 1: B+ Tree 導航
  「id=42 住在哪一頁？」
  → 沿著 B+ Tree 定位到 Leaf Page #128

Step 2: Buffer Pool 上桌
  → 確認 Page #128 在桌上 (同 INSERT)

Step 3: Undo Log 影印備份 ⚠️ (INSERT 沒有這步)
  「在動手之前，先把舊資料影印一份！」
  → 記錄：「id=42 的 age 原本是 30」
  → 這份影印本被串進 Undo Log 的版本鏈 (DB_ROLL_PTR 指向它)

Step 4: 在桌上動手 (Memory Write)
  → 把 Page #128 裡 id=42 的 age 從 30 改成 31
  → 更新 DB_TRX_ID 為當前 Transaction ID
  → Page #128 變成 Dirty Page

Step 5: Redo Log 寫流水帳 (Sequential I/O)
  → 追加：「Page #128, Offset 320, age: 30 → 31」
  → fsync() 落地！

Step 6: 回報成功 ✅
```

### UPDATE vs INSERT 的關鍵差異
唯一多出來的就是 **Step 3 (Undo Log 影印)**。
這份影印本有兩個消費者：
1.  **Rollback**：如果 Transaction 最後決定 `ROLLBACK`，系統拿出影印本，把 31 改回 30。
2.  **MVCC 讀者**：如果另一個 Transaction 在你 Commit 之前就開始了 `SELECT age FROM users WHERE id = 42`，它的 ReadView 會判定你的修改「不可見」，然後順著 `DB_ROLL_PTR` 跳到 Undo Log，拿到 `age = 30` 這個舊版本。讀者完全不需要等你的鎖。

### 物理位置標籤版：UPDATE 的硬體真相

場景：`UPDATE users SET balance = 800 WHERE name = 'Frankie'` (餘額從 1000 改成 800)

把每一步動作標上它 **真正發生在哪一層硬體** 上，你就能看清 MySQL 寫入的真理：

```text
┌──────────┬───────────────────────────────────────────────────────────┐
│  [RAM]   │ ① 寫入 Undo Log                                         │
│          │   在 Buffer Pool 找張便條紙寫下：                          │
│          │   「Frankie 餘額原本是 1000」                              │
├──────────┼───────────────────────────────────────────────────────────┤
│  [RAM]   │ ② 修改 Page Data                                         │
│          │   把 Buffer Pool 裡主表格的 1000 塗掉，改成 800             │
│          │   此時這一頁變為 Dirty Page (髒頁)                         │
├──────────┼───────────────────────────────────────────────────────────┤
│  [RAM]   │ ③ 寫入 Redo Log Buffer                                   │
│          │   在記憶體裡的筆記本寫下：「把 Frankie 改成 800 了」         │
├──────────┼───────────────────────────────────────────────────────────┤
│  [Go]    │ ④ 發起 COMMIT                                            │
│          │   你的 Go 程式呼叫 tx.Commit()                            │
├──────────┼───────────────────────────────────────────────────────────┤
│  [SSD]   │ ⑤ 強制 fsync()                                           │
│          │   MySQL 將 Redo Log Buffer 的內容                          │
│          │   強制寫入硬碟的 Redo Log 實體檔案 (Sequential I/O)         │
│          │   ← 這是唯一碰到硬碟的瞬間！也是資料「保命」的關卡          │
├──────────┼───────────────────────────────────────────────────────────┤
│  [Go]    │ ⑥ 收到 OK                                                │
│          │   SSD 寫入成功，Go 收到 nil error，交易正式確立             │
├──────────┼───────────────────────────────────────────────────────────┤
│  [SSD]   │ ⑦ 背景 Checkpoint (非同步，與用戶無關)                     │
│          │   過一陣子，背景線程才把 Dirty Page 刷回硬碟資料檔           │
│          │   這是 Random I/O，但用戶早就收到 OK 了，不影響延遲          │
└──────────┴───────────────────────────────────────────────────────────┘
```

**關鍵洞察**：
*   從 ① 到 ④，所有動作都只發生在 **RAM (記憶體)** 裡，速度是奈秒 (ns) 級。
*   整個流程中 **唯一碰到 SSD 的只有 ⑤ (fsync)**，而且是 Sequential I/O，速度極快。
*   用戶在 ⑥ 收到 OK 的時候，**硬碟的資料檔裡其實還是舊的 1000**！新的 800 只活在記憶體和 Redo Log 裡。
*   這就是 WAL 的精髓：**用戶感知的延遲 = RAM 操作 + 一次 Sequential fsync ≈ 0.1ms**，而不是 Random I/O 的 10ms。

---

## 4. 第四樂章：停電了怎麼辦？(Crash Recovery)

假設 Step 5 (Redo Log fsync) 完成後，Step 6 還沒來得及把 Dirty Page 寫回硬碟，伺服器就停電了。

**重開機後**：
1.  InnoDB 啟動時會掃描 Redo Log。
2.  發現：「有一條記錄說 Page #128 的 age 應該是 31，但硬碟上的資料檔裡 age 還是 30。」
3.  **動作**：把 Page #128 搬到 Buffer Pool，重新執行那條修改 (age → 31)，然後刷回硬碟。
4.  **結果**：資料完全恢復，零丟失。

這就是為什麼 Redo Log 被稱為 **「保命流水帳」**。只要日誌在，天塌下來都能救。

---

## 5. 第五樂章：全景流程圖 (The Complete Picture)

```text
                        INSERT / UPDATE
                              │
                    ┌─────────▼──────────┐
                    │   B+ Tree 導航台    │  「資料在 Page #128」
                    └─────────┬──────────┘
                              │
                    ┌─────────▼──────────┐
                    │   Buffer Pool      │  Page 在桌上嗎？
                    │   (記憶體辦公桌)     │  Miss → 去硬碟搬上來
                    └─────────┬──────────┘
                              │
              ┌───────────────┼───────────────┐
              │ (UPDATE Only) │               │
    ┌─────────▼──────────┐    │     ┌─────────▼──────────┐
    │    Undo Log        │    │     │   LRU 清理大師      │
    │  (影印舊版本)       │    │     │  (標記最近使用)      │
    │  → MVCC + Rollback │    │     └────────────────────┘
    └────────────────────┘    │
                              │
                    ┌─────────▼──────────┐
                    │  修改 Buffer Pool   │  在記憶體中改寫資料
                    │  (產生 Dirty Page)  │
                    └─────────┬──────────┘
                              │
                    ┌─────────▼──────────┐
                    │    Redo Log (WAL)   │  Sequential I/O
                    │    fsync() 落地！   │  ← 保命關卡
                    └─────────┬──────────┘
                              │
                    ┌─────────▼──────────┐
                    │   回報 Commit 成功   │  → Go Driver 收到 OK
                    └─────────┬──────────┘
                              │
                    ┌─────────▼──────────┐
                    │   Checkpoint (背景)  │  慢慢把 Dirty Page
                    │   刷回硬碟資料檔     │  寫回 SSD (Random I/O)
                    └────────────────────┘
```

---

## 6. 第六樂章：面試一句話總結 (Finale)

> **「MySQL 的所有讀寫操作，永遠只發生在 Buffer Pool (記憶體) 裡。B+ Tree 負責查地址，Undo Log 負責保存舊版本 (MVCC + Rollback)，Redo Log 負責保命 (Crash Recovery)。真正寫回硬碟的動作，是由背景的 Checkpoint 慢慢完成的。這就是為什麼 MySQL 的 Buffer Pool 越大，效能越好。」**
