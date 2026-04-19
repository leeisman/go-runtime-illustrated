# Book 5.2: Snowflake (Distributed ID - 雪花算法)

分散式 ID 的難題是：不能集中發號，卻又不能重複。
Snowflake 的答案是把時間、機器編號與序列號塞進同一個 64-bit 整數，讓每個節點都能獨立生產全域唯一 ID。

---

## 1. 第一樂章：隱喻 (The Metaphor - The Independent Ticket Booths)

想像一個巨大的遊樂園，有 1000 個售票亭 (微服務節點) 同時在賣票。
我們需要給每一張票一個 **唯一的流水號**。

*   **中央印鈔機 (MySQL Auto Increment)**:
    *   所有售票亭都要打電話給總部：「給我下一個號碼」。
    *   **瓶頸**: 總部電話被打爆 (Lock Contention)。
*   **UUID (亂碼)**:
    *   售票員隨便寫一串亂碼。
    *   **缺點**: 號碼太長 (36 chars)，而且沒順序，資料庫管理員 (DBA) 會氣死 (因為 B+Tree 索引會分裂)。
*   **Snowflake (各自為政)**:
    *   每個售票員看自己的手錶 (Time)。
    *   加上自己的工號 (Machine ID)。
    *   加上自己手邊的計數器 (Sequence)。
    *   **結果**: 不需要打電話回總部，每個人都能獨立產出 **全域唯一** 且 **大致有序** 的號碼。

---

## 2. 第二樂章：位元解剖 (Anatomy of a Snowflake)

Snowflake 的標準實作是生成一個 `int64` (8 bytes)。這 64 bits 被切分成：

```text
| 1 bit |     41 bits (Timestamp)     |  10 bits (MachineID) |   12 bits (Sequence)  |
|-------|-----------------------------|--------------------|-----------------------|
| 0 (正數)| 毫秒級時間戳 (可用 69 年)      |      支援 1024 台機器     |  每毫秒產生 4096 個 ID   |
```

這一刀切下去，決定了系統的極限：
*   **吞吐量**: 單機每毫秒 4096 個 = **409.6 萬 ID / 秒**。 (夠快了吧)
*   **擴容性**: 最多 1024 台機器。

---

## 3. 第三樂章：Go 實作 (Bitwise Magic)

這是一場位元運算的盛宴。在 Go 裡面，我們利用 `<<` (左移) 和 `|` (OR) 來組裝 ID。

```go
package snowflake

import (
    "errors"
    "sync"
    "time"
)

// 常數定義
const (
    // 自定義元年 (Epoch): 2022-01-01 00:00:00 UTC
    // 目的: 讓 41 bits 的時間戳可以從現在開始算還能用 69 年
    epoch = 1640995200000

    timestampBits = 41
    machineBits   = 10
    sequenceBits  = 12

    // 最大的機器 ID (1111111111 -> 1023)
    maxMachineID = -1 ^ (-1 << machineBits)
    
    // 最大的序列號 (111111111111 -> 4095)
    maxSequence  = -1 ^ (-1 << sequenceBits)

    // 位移量
    machineShift   = sequenceBits
    timestampShift = sequenceBits + machineBits
)

type Snowflake struct {
    mu        sync.Mutex
    lastTime  int64
    machineID int64
    sequence  int64
}

// NewNode 初始化一個節點
func NewNode(machineID int64) (*Snowflake, error) {
    if machineID < 0 || machineID > maxMachineID {
        return nil, errors.New("machine ID out of range")
    }
    return &Snowflake{
        lastTime:  time.Now().UnixMilli(),
        machineID: machineID,
        sequence:  0,
    }, nil
}

func (s *Snowflake) NextID() (int64, error) {
    s.mu.Lock()
    defer s.mu.Unlock()

    now := time.Now().UnixMilli()

    // 0. 時鐘回撥檢查 (Clock Moved Backwards)
    if now < s.lastTime {
        // 策略: 報錯拒絕服務，或者阻塞等待
        return 0, errors.New("clock moved backwards")
    }

    // 1. 同一毫秒內的請求
    if now == s.lastTime {
        // 序列號 + 1，並防止溢出 (超過 4095 變回 0)
        s.sequence = (s.sequence + 1) & maxSequence
        
        // 如果溢出了 (變回 0)，代表這毫秒配額用完了，必須等下一毫秒
        if s.sequence == 0 {
            for now <= s.lastTime {
                now = time.Now().UnixMilli()
            }
        }
    } else {
        // 2. 新的一毫秒，序列號歸零
        s.sequence = 0
    }

    s.lastTime = now

    // 3. 拼裝 ID (最精彩的部分)
    // ID = (時間差 << 22) | (機器ID << 12) | (序列號)
    id := ((now - epoch) << timestampShift) | (s.machineID << machineShift) | s.sequence
    return id, nil
}
```

這段代碼完全在記憶體內執行，不涉及任何 I/O。這就是它為什麼快到飛起的原因。

---

## 4. 第四樂章：時鐘回撥的惡夢 (Clock Move Backwards)

分佈式系統最怕的問題之一：**NTP 校時導致時間倒退**。

如果現在是 `10:00:01`，我產生了一個 ID。
結果機器時間突然被校正回 `10:00:00`。
下一秒我又用 `10:00:00` 產生 ID -> **ID 重複了！**

**解法**：
在 `NextID` 裡面檢查 (如上代碼所示)：
1.  **報錯 (Fail Fast)**: 如果時間倒退，直接回傳 error。讓上層重試 (通常因為負載均衡，會打到另一台正常的機器)。
2.  **等待 (Spin Loop)**: 如果倒退時間很短 (例如 < 5ms)，可以直接 `time.Sleep` 等它追上來。

---

## 5. 第五樂章：Machine ID 怎麼來？

這 10 bits 的 Machine ID 其實是最難搞的。你怎麼知道這台 Pod 應該是 1 號還是 2 號？

1.  **靜態設定 (Config)**: 在 K8s Deployment yaml 裡寫死 `ENV MACHINE_ID=1` (難管理)。
2.  **Redis 分配**: Server 啟動時去 Redis `INCR machine_id` 拿一個號碼 (推薦)。
3.  **IP Hash**: 根據 Pod IP 做 Hash (有碰撞風險)。

**最佳實踐**: 使用 Redis 或 Etcd 在啟動時動態註冊，獲取一個臨時的 Worker ID。
