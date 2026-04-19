# Book 4.7: Cache Penetration & Bloom Filter (防穿透模式)

在 **Book 4.6 Singleflight** 中，我們解決了「熱點 Key 過期」導致的並發查詢問題。但攻擊者往往不講武德，他們這一次不打熱點，而是專門打 **「根本不存在的 Key」**。

---

## 1. 第一樂章：穿透與擊穿 (Penetration vs Breakdown)

這兩個名詞非常容易搞混，我們先來個正名運動：

| 名詞 | 英文 | 場景 | 查詢對象 | 結果 | 解法 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **擊穿** | Breakdown / Stampede | 熱門商品 Cache 過期 | **存在的 (有效) Key** | 瞬間大量並發打 DB | Singleflight |
| **穿透** | Penetration | 惡意攻擊 / 爬蟲 | **不存在的 (無效) Key** | 每個請求都打 DB (無法合併) | Bloom Filter |

**攻擊劇本**：
1.  攻擊者生成 100 萬個隨機 UUID (`user:xyz-123`, `user:abc-999`...)。
2.  你的 Cache 一定沒這些資料 (因為根本不存在)。
3.  所以程式碼說：「Cache 沒有 -> 去 DB 查」。
4.  DB 拼命查但什麼都查不到 (Miss)。
5.  **結果**：DB 的 CPU 被這些無效查詢耗盡，正常用戶反而進不來了。

---

## 2. 第二樂章：兩大防禦術

### 2.1 方案 A：快取空值 (Cache Null Value)
這是一招「以退為進」的策略。
*   **邏輯**：當 DB 查不到時，不要直接返回 error，而是把這個 Key 寫進 Cache，值設為 `null` (或特殊標記)，過期時間設短一點 (例如 5 分鐘)。
*   **效果**：下次攻擊者再查同一個無效 Key，Cache 會直接回 `null`，不再打 DB。
*   **缺點**：如果攻擊者用 1 億個**不重複**的亂數 Key，你的 Redis 會被塞滿 1 億個 `null`，記憶體爆炸。

### 2.2 方案 B：布隆過濾器 (Bloom Filter)
這是一招「門神」策略。
*   **邏輯**：在查 Cache 之前，先問 Bloom Filter：「這個 Key **可能**存在嗎？」
    *   Bloom Filter 說 **「沒有」**：那絕對沒有，**直接擋掉**，連 Cache/DB 都不用查。
    *   Bloom Filter 說 **「有」**：那可能真的有 (也可能是誤判)，才放行去查 Cache/DB。
*   **優點**：極致省空間。只需要 **Bit Array**，存 10 億個 Key 可能只要幾百 MB。
*   **缺點**：有 **False Positive (誤判率)**。它可能會把「不存在」說成「有」，但絕不會把「存在」說成「沒有」。對於防穿透來說，這個特性是可以接受的 (誤判只是多打一次 DB，不會死)。

---

## 3. 第三樂章：Bloom Filter 原理

想像你有一個很長的 bit 陣列 (長度 m)，初始全是 0。
還有 k 個雜湊函數 (Hash Function)。

**寫入 (Add)**：
1.  把 `Key="User:1"` 丟進 k 個 Hash 函數，算出 k 個位置。
2.  把陣列中這 k 個位置的 bit 設為 1。

**查詢 (Check)**：
1.  把 `Key="User:1"` 丟進 k 個 Hash 函數，算出 k 個位置。
2.  檢查這 k 個位置的 bit：
    *   只要有 **任何一個是 0** -> 肯定沒存過 -> **不存在 (擋掉)**。
    *   如果 **全都是 1** -> **可能存在 (放行)**。

為什麼是「可能」？因為這些 1 可能是別的 Key 剛好也 Hash 到同樣位置 (Hash Collision) 所留下的。

---

## 4. 第四樂章：Go 實作範例

我們使用 `github.com/bits-and-blooms/bloom` 這個庫。

```go
package main

import (
	"fmt"
	"github.com/bits-and-blooms/bloom/v3"
)

func main() {
	// 初始化 Bloom Filter
	// n: 預計存多少元素 (100萬)
	// p: 容忍的誤判率 (1% = 0.01)
	filter := bloom.NewWithEstimates(1000000, 0.01)

	// 1. 系統啟動時，把所有有效的 ID 載入 Filter (Pre-loading)
	validIDs := []string{"user:1", "user:2", "user:3"}
	for _, id := range validIDs {
		filter.Add([]byte(id))
	}

	// 2. 模擬請求
	checkRequest(filter, "user:1")       // 有效 Key
	checkRequest(filter, "user:99999")   // 無效 Key (攻擊)
}

func checkRequest(f *bloom.BloomFilter, key string) {
	// 這是第一道防線
	if !f.Test([]byte(key)) {
		fmt.Printf("[Blocked] Key '%s' definitely does not exist.\n", key)
		return
	}

	// 通過防線，才去查 Cache/DB
	// 注意：這裡可能會發生極小機率的誤判 (False Positive)
	// 但 99% 的惡意攻擊都在上面被擋掉了
	fmt.Printf("[Allowed] Key '%s' might exist, checking DB...\n", key)
}
```

**輸出結果**：
```text
[Allowed] Key 'user:1' might exist, checking DB...
[Blocked] Key 'user:99999' definitely does not exist.
```

---

## 5. 第五樂章：架構設計思考

雖然 Bloom Filter 很強，但在分散式系統中實作有挑戰：

### 5.1 資料同步 (Consistency)
如果你在 DB 新增了一個 User，記得也要同步加到 Bloom Filter。
*   **難點**：Bloom Filter **不支援刪除 (Delete)**。
*   **原因**：如果你把某個位元從 1 改成 0，可能會誤刪到其他 Hash 碰撞的 Key。
*   **解法**：
    1.  **Cuckoo Filter (布穀鳥過濾器)**：支援刪除的新型態過濾器。
    2.  **定時重建**：每天半夜重建整個 Bloom Filter。這比較適合「商品庫」這種不常刪除的場景。

### 5.2 記憶體 vs Redis
*   **Local BitSet**: 像上面的 Go 程式碼，Filter 存在單機記憶體。速度最快，但每台機器都要維護一份，且重啟會消失。
*   **Redis BitMaps**: 利用 Redis 的 `SETBIT` 指令實作。所有機器共享同一個 Filter。適合分散式架構，但多了一次網路 RTT。

---

## 6. 第六樂章：高併發防護全家桶 (Summary)

至此，我們的 **Book 4** 系列真正圓滿了。讓我們看看這套「全家桶」是如何協同工作的：

1.  請求進來 -> **Rate Limiter** (太快？擋！)
2.  請求有效性 -> **Bloom Filter** (亂猜 Key？擋！)
3.  執行開始 -> **Singleflight** (大家查一樣的？合併！)
4.  執行中 -> **Worker Pool / Semaphore** (人太多？排隊！)
5.  下游掛了 -> **Circuit Breaker** (斷路！別試了！)
6.  單次超時 -> **Context** (太久？取消！)
7.  惡意無效 Key -> **Bloom Filter** (不存在？滾！)

這就是一個 **Production-Ready** 的高併發後端架構。

但在微服務 (Microservices) 的世界裡，我們還有一個常見的需求：**「我要同時呼叫 User, Order, Payment 三個服務，只要有一個失敗就全部取消，該怎麼寫最優雅？」**
手寫 WaitGroup + Channel 太累了。這就是 **Book 4.8: ErrGroup (併發聚合模式)** 的主場。
