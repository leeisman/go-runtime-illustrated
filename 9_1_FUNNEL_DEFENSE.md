# Book 9.1: 微服務高併發漏斗防禦 (Funnel Defense Architecture)

在微服務與高併發場景（如遊戲開獎、秒殺、紅包搶奪）中，系統最怕的不是沒有資源，而是所有請求在同一時間湧入資料庫，造成連線池打滿、鎖競爭與最終的系統崩潰。

為了解決這個問題，我們不應該只依賴單一技術，而是要建立一個 **「漏斗防禦架構 (Funnel Defense)」**。這是一種層層遞減流量的哲學：在外層用輕量、快速但不保證絕對正確的機制擋下 99% 的無效請求；在最內層用沉重、嚴謹且絕對正確的機制處理那最後 1% 的關鍵請求。

---

## 1. 第一樂章：漏斗隱喻 (The Funnel Metaphor)

想像一座中世紀的巨大城堡（你的系統），正在遭遇敵軍（高併發請求）的突襲。

*   **前哨站 (Redis)**：位於城牆外，配備輕量武器，處理速度極快。它的任務是快速識別並攔截大部分重複或無效的敵人。
*   **最後堡壘 (MySQL)**：位於城堡最深處，擁有最厚重的裝甲與金庫。它的動作緩慢（硬碟 I/O），但絕對不會出錯。

漏斗架構的核心思維：**不要讓所有敵人都衝到最後堡壘前才開始防禦。**

---

## 2. 第二樂章：前哨站 - Redis 分散式鎖 (The Outpost)

當一萬個玩家同時點擊「領取百萬獎金」時，如果全部直接打進 MySQL，資料庫會瞬間癱瘓。第一層防禦，我們交給 **Redis 分散式鎖**。

### 1. 機制：SETNX (Set if Not Exists)
*   我們在 Redis 中設定一個 Key：`lock:jackpot:12345`。
*   一萬個併發請求同時執行 `SETNX`。Redis 是單執行緒模型，只會有**第一個**請求成功寫入並獲得 `1`（取得鎖），其餘 9999 個請求會得到 `0`。
*   **結果**：9999 個請求直接被判定為「搶奪失敗」或「稍後再試」，流量瞬間從 10000 降為 1。

### 2. 優勢
*   **極速**：基於記憶體操作，延遲在毫秒級別。
*   **保護後端**：成功擋下 99% 的無效流量，讓 MySQL 完全感覺不到外面的風暴。

---

## 3. 第三樂章：防線崩潰 - Redis 的隱患 (The Breach)

看起來 Redis 已經完美解決了問題，為什麼我們還需要 MySQL？因為 Redis 作為前哨站，其實是**脆弱且不可靠**的。

### 1. 鎖過期與業務超時 (Lock Expiration)
*   **場景**：A 拿到鎖，設定過期時間 5 秒。但 A 的業務邏輯太複雜，執行了 6 秒。
*   **災難**：第 5 秒時鎖自動釋放，B 趁機拿到了鎖並進入業務邏輯。此時 A 執行完畢，不但刪除了 B 的鎖，還導致 A 和 B 同時修改了狀態（併發漏洞）。
*   **救星 (Watchdog 看門狗)**：當 Pod 成功拿到鎖後，啟動一個背景 Goroutine。只要主要業務邏輯還沒跑完，看門狗就會定時（例如每 2 秒）去 Redis 把這把鎖的 TTL「續命」，徹底解決業務超時導致防線崩潰的問題。

### 2. 腦裂與主從延遲 (Brain Split & Replication Lag)
*   **場景**：Redis 使用主從架構 (Master-Slave)。A 在 Master 成功拿到鎖，但 Master 還沒把資料同步給 Slave 就當機了。
*   **災難**：Slave 升級成新 Master，因為它沒有 A 的鎖紀錄，所以 B 來申請鎖時，新 Master 給了 B。此時 A 和 B 再次同時擁有了鎖。

**結論**：Redis 可以用來「擋流量（防禦）」，但絕對不能用來「保證資料絕對正確（金流）」。

---

## 4. 第四樂章：最後堡壘 - MySQL 樂觀鎖 (The Citadel)

當那個幸運的 1% 請求穿越 Redis 來到 MySQL 時，我們必須祭出最嚴謹的防禦：**樂觀鎖 (Optimistic Lock)**。

### 1. 版本號機制 (Version Control)
在資料表中加入一個 `version` 欄位。

```sql
SELECT id, balance, version FROM account WHERE id = 1;
-- 假設讀到 version = 10
```

### 2. 帶條件的更新 (Conditional Update)
在更新時，將剛剛讀到的 `version` 作為 `WHERE` 條件，並將版本號加 1。

```sql
UPDATE account 
SET balance = balance + 1000, version = version + 1 
WHERE id = 1 AND version = 10;
```

### 3. 底層原理 (Under the Hood)
*   如果 A 和 B 同時來到這裡，MySQL 的 InnoDB 引擎會在執行 `UPDATE` 時對該行資料加**排他鎖 (X-Lock)**。
*   A 先拿到鎖，執行成功，`version` 變成 `11`。
*   B 接著執行，因為條件 `version = 10` 已經不成立，`UPDATE` 影響的行數 (Affected Rows) 為 0。
*   **結果**：即便 Redis 防線崩潰讓 A 幾乎同時進來，MySQL 的樂觀鎖依然保證了資料的一致性。
*   **⚠️ 致命陷阱 (Gap Lock)**：必須確保 `id` 欄位有建立 **Primary Key** 或 **Unique Index**。如果沒有適當的索引，InnoDB 在執行 `UPDATE` 時會退化成全表掃描，並鎖住所有行與間隙 (Gap Lock)，這會引發嚴重的 Deadlock（死結），直接讓最後堡壘崩潰。

---

## 5. 第五樂章：寫入流與交易 (The Write Flow & Transaction)

樂觀鎖只是保護單一列資料，但在真實業務中，扣款和發放獎勵通常牽涉多個步驟，我們必須將其包裹在一個完整的 **Transaction (交易)** 中。

### 1. 交易流水線 (Transaction Pipeline)
一個標準的高併發寫入流 (Write Flow) 應該長這樣：

```go
func ClaimJackpot(ctx context.Context, userID int) error {
    // 產生當前請求的唯一識別碼 (避免誤刪別人的鎖)
    requestUUID := uuid.New().String()
    
    // 1. 前哨站：嘗試取得 Redis 鎖 (防禦 99% 流量)
    if !TryRedisLock("jackpot_lock", requestUUID) {
        return ErrBusy
    }
    // ⚠️ 警告：解鎖絕對不能只用 DEL，必須透過 Lua Script 保證「比對 UUID 且相同才刪除」的原子性操作
    defer ReleaseRedisLockWithLua("jackpot_lock", requestUUID)

    // 2. 開啟 MySQL 交易 (進入最後堡壘)
    tx, _ := db.BeginTx(ctx, nil)
    defer tx.Rollback()

    // 3. 讀取狀態與版本號
    var status, version int
    tx.QueryRow("SELECT status, version FROM jackpot WHERE id = 1").Scan(&status, &version)
    
    // 4. 業務檢查 (是否已被領取)
    if status == CLAIMED {
        return ErrAlreadyClaimed
    }

    // 5. 樂觀鎖更新
    result, _ := tx.Exec("UPDATE jackpot SET status = ?, version = version + 1 WHERE id = 1 AND version = ?", CLAIMED, version)
    rows, _ := result.RowsAffected()
    if rows == 0 {
        // 發生併發衝突，代表有其他人剛好改了版號。
        // 業務決策 A (搶紅包)：Fail-fast，直接回傳失敗，獎品被搶走了。
        // 業務決策 B (扣手續費)：實作 Retry 迴圈，重新 SELECT 最新版號再更新一次。
        return ErrConcurrentUpdate
    }

    // 6. 寫入領獎流水紀錄 (Audit Log)
    tx.Exec("INSERT INTO claim_log (user_id, jackpot_id) VALUES (?, ?)", userID, 1)

    // 7. 提交交易
    return tx.Commit()
}
```

### 2. 工程意義
這個 Flow 將**讀取、校驗、防護 (樂觀鎖)、日誌 (流水)** 封裝在同一個 ACID 交易內。即使發生 Redis 腦裂，只要 `Affected Rows == 0` 或發生回滾，資料庫依然堅若磐石。

---

## 6. 第六樂章：架構的深度延伸 (Architectural Extensions)

如果要把「漏斗防禦」做到極致應付千萬人搶票，兩層防禦其實還不夠，我們可以擴展為**「四層防禦」**：

### 1. 第零防線：本地快取 (Local Cache)
*   **痛點**：Redis 雖然快，但終究需要**網路 I/O**。瞬間湧入十萬請求也可能打滿網卡。
*   **做法**：在微服務的 Pod 記憶體內建立 Local Cache（例如 Go 的 `sync.Map` 或 `Ristretto`）。當大獎被抽走時，除了更新 MySQL 與 Redis，同時廣播通知所有 Pod 標記 `is_claimed = true`。
*   **效果**：接下來的百萬個無效請求，在 Go 程式內部就會被擋下，連網路線都不用出，達成極限效能。

### 2. 削峰填谷：訊息佇列 (Message Queue)
*   **痛點**：目前的流程是「同步 (Synchronous)」等待 MySQL 寫入完成才回傳，吞吐量受限於資料庫 I/O。
*   **做法**：如果業務允許「非同步 (Asynchronous)」，將第五樂章的 MySQL 寫入流替換成「將領獎事件推入 Kafka/RabbitMQ」。
*   **效果**：讓資料庫從「被動承受衝擊」變成「依照自己舒服的速度拉取任務寫入」，將系統吞吐量上限再拉高一個數量級。

### 3. 戰場上帝視角：可觀測性閉環 (Observability)
*   **痛點**：漏斗架構由多層異質元件組成，若發生效能抖動，瞎猜只會錯失黃金救援時間。一個好的漏斗不只要能「擋」，還要能「看」。
*   **做法**：在漏斗的每個收束點埋設 Metrics 指標。例如：
    *   **外圍防線**：Local Cache 與 Redis 的「阻擋率 (Block Rate)」。
    *   **緩衝區**：MQ 的「任務堆積深度 (Lag Depth)」。
    *   **最後堡壘**：MySQL 樂觀鎖觸發 `Affected Rows == 0` 或 Rollback 的次數。
*   **效果**：當大流量襲來，只需看一眼 Grafana 上的「漏斗轉化率」，就能立刻釐清是哪一層防線遭遇瓶頸。有資料佐證，才是維運這套複雜系統的真正底氣。

---

## 7. 第七樂章：總結 (Finale)

**漏斗防禦架構** 的精髓在於承認沒有單一技術是完美的：

*   **第零防線 (Local Cache)**：記憶體級別，擋下海量重試與已知無效請求。
*   **前哨站 (Redis + Lua + Watchdog)**：網路級別，不可靠但極快，負責承載 99% 的砲火。
*   **緩衝區 (MQ)**：非同步削峰，保護底層資料庫。
*   **最後堡壘 (MySQL Optimistic Lock)**：很慢，但絕對可靠。負責守護那 1% 的真實交易，確保最終一致性。

**一句話總結**：用 快取與佇列 擋流量（追求可用性），用 MySQL 樂觀鎖 擋併發（追求一致性），這才是高併發微服務的終極防禦兵法。
