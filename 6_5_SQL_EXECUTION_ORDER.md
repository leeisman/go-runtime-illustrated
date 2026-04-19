# Book 6.5: SQL 的真實生命週期 (SQL Execution Order)

很多人 SQL 寫錯，都是因為誤以為 SQL 是從第一行的 `SELECT` 開始跑的。
其實，MySQL 底層的執行順序跟你寫的順序**完全不同**。

搞懂這個順序，你就能理解 `ON`、`WHERE`、`HAVING` 三兄弟到底在打什麼架。

---

## 1. 第一樂章：真實執行順序 (The Truth)

你寫的 SQL 長這樣：
```sql
SELECT dept, COUNT(*) AS cnt
FROM orders
JOIN users ON orders.uid = users.id
WHERE users.age > 18
GROUP BY dept
HAVING COUNT(*) > 5
ORDER BY cnt DESC
LIMIT 10;
```

但 MySQL 底層的執行順序是這樣的：

```text
┌─────┬──────────────────────────────────────────────────┐
│ 順序 │ 階段                                             │
├─────┼──────────────────────────────────────────────────┤
│  1  │ FROM / JOIN    把表抓出來拼起來 (產生笛卡兒積)     │
│  2  │ ON             牽手條件 (JOIN 時的篩選)            │
│  3  │ WHERE          針對「單筆資料」進行屠殺            │
│  4  │ GROUP BY       把活下來的人，分組組隊              │
│  5  │ HAVING         針對「整個團隊」進行淘汰            │
│  6  │ SELECT         把最終結果打包 (計算欄位、別名)     │
│  7  │ ORDER BY       排序                               │
│  8  │ LIMIT          分頁截取                            │
└─────┴──────────────────────────────────────────────────┘
```

> **關鍵洞察**：`SELECT` 排在第 6 位！這就是為什麼你不能在 `WHERE` 裡面用 `SELECT` 定義的別名 (Alias)，因為 `WHERE` 跑的時候，`SELECT` 根本還沒執行。

---

## 2. 第二樂章：用 Go 虛擬碼還原 MySQL 的腦袋

把 MySQL 的執行引擎想像成一段 Go 程式碼，你就能徹底看透每一層在做什麼：

```go
func ExecuteSQL() []Row {

    // ========== Step 1: FROM / JOIN (把表抓出來) ==========
    // 先把兩張表的所有行做笛卡兒積 (Cartesian Product)
    var rawRows []JoinedRow
    for _, order := range ordersTable {
        for _, user := range usersTable {
            rawRows = append(rawRows, JoinedRow{order, user})
        }
    }
    // 如果 orders 有 1000 行，users 有 500 行
    // rawRows 現在有 500,000 行 (爆炸！)

    // ========== Step 2: ON (牽手條件) ==========
    // 從笛卡兒積中，只保留「真正配對成功」的行
    var matched []JoinedRow
    for _, row := range rawRows {
        if row.Order.UID == row.User.ID { // ON 條件
            matched = append(matched, row)
        }
    }
    // 500,000 行瞬間縮減為 1000 行 (只有真正牽手的)

    // ========== Step 3: WHERE (個人屠宰場) ==========
    // 針對每一筆「單獨的資料行」進行篩選
    var survivors []JoinedRow
    for _, row := range matched {
        if row.User.Age > 18 { // WHERE 條件
            survivors = append(survivors, row)
        }
    }
    // 未成年的全部淘汰

    // ========== Step 4: GROUP BY (分組組隊) ==========
    // 把活下來的人，按照 dept 欄位分組
    groups := make(map[string][]JoinedRow)
    for _, row := range survivors {
        dept := row.Order.Dept
        groups[dept] = append(groups[dept], row)
    }
    // groups = { "工程部": [row1, row2, ...], "行銷部": [row3, ...] }

    // ========== Step 5: HAVING (團隊淘汰賽) ==========
    // 針對「整個團隊」進行篩選
    // 注意：HAVING 操作的對象是「一整個 Group」，不是單獨的 Row
    qualifiedGroups := make(map[string][]JoinedRow)
    for dept, members := range groups {
        if len(members) > 5 { // HAVING COUNT(*) > 5
            qualifiedGroups[dept] = members
        }
    }
    // 人數不足 5 人的部門，整組淘汰

    // ========== Step 6: SELECT (打包結果) ==========
    // 到這一步才計算你要的欄位和別名
    var results []Row
    for dept, members := range qualifiedGroups {
        results = append(results, Row{
            Dept: dept,
            Cnt:  len(members), // COUNT(*) 在這裡計算
        })
    }

    // ========== Step 7: ORDER BY (排序) ==========
    sort.Slice(results, func(i, j int) bool {
        return results[i].Cnt > results[j].Cnt // DESC
    })

    // ========== Step 8: LIMIT (分頁截取) ==========
    if len(results) > 10 {
        results = results[:10]
    }

    return results
}
```

---

## 3. 第三樂章：三兄弟的戰場差異 (ON vs WHERE vs HAVING)

### ON (牽手條件)
*   **執行時機**：Step 2，JOIN 的當下。
*   **操作對象**：兩張表「配對」的條件。
*   **職責**：決定哪些行可以成功牽手。不符合條件的行，在 LEFT JOIN 中會被補 NULL。

```sql
-- ON 決定誰跟誰牽手
SELECT * FROM orders
LEFT JOIN users ON orders.uid = users.id;
-- 如果 orders 有一筆 uid=999 但 users 裡沒有 id=999
-- LEFT JOIN + ON 會保留這筆 order，user 欄位補 NULL
```

### WHERE (個人屠宰場)
*   **執行時機**：Step 3，牽手之後。
*   **操作對象**：每一筆「單獨的資料行」。
*   **職責**：不符合條件的行，直接從結果中消失 (包括 NULL 行)。
*   **限制**：不能使用聚合函數 (`COUNT`, `SUM`, `AVG`)，因為此時還沒分組。

```sql
-- WHERE 在牽手之後屠殺
SELECT * FROM orders
LEFT JOIN users ON orders.uid = users.id
WHERE users.age > 18;
-- 注意：uid=999 那筆 order 的 user.age 是 NULL
-- NULL > 18 判定為 false → 這行也被殺掉了！
-- 這就是 ON 和 WHERE 放的位置不同，結果不同的根本原因
```

### HAVING (團隊淘汰賽)
*   **執行時機**：Step 5，GROUP BY 之後。
*   **操作對象**：一整個「Group (團隊)」。
*   **職責**：用聚合函數對整個團隊做判斷，不合格的整組淘汰。

```sql
-- HAVING 看的是整個團隊的統計值
SELECT dept, COUNT(*) AS cnt, AVG(salary) AS avg_sal
FROM employees
GROUP BY dept
HAVING COUNT(*) > 5 AND AVG(salary) > 50000;
-- 人數不足 5 人，或平均薪資不到 5 萬的部門，整組淘汰
```

---

## 4. 第四樂章：經典陷阱 (Common Traps)

### 陷阱 1：在 WHERE 裡用別名
```sql
-- ❌ 錯誤：WHERE 在 SELECT 之前執行，cnt 別名還不存在
SELECT dept, COUNT(*) AS cnt
FROM orders
GROUP BY dept
WHERE cnt > 5;
```

**用 Go 虛擬碼秒懂為什麼錯：**
```go
// 在 Step 3 (WHERE) 階段時，我們手上的資料是牽手完的原始行 (matched)
for _, row := range matched {
    // if row.cnt > 5 { ... } // 💥 編譯錯誤！
    // 根本還沒有計算 COUNT(*)，哪來的 cnt？
    // cnt 這個變數要到 Step 6 (SELECT) 階段才會被建立出來！
}
```

```sql
-- ✅ 正確：用 HAVING (它在 SELECT 之後...不對，HAVING 其實也在 SELECT 之前)
-- 但 MySQL 對 HAVING 做了特殊優化，允許引用 SELECT 的別名
SELECT dept, COUNT(*) AS cnt
FROM orders
GROUP BY dept
HAVING cnt > 5;
```

### 陷阱 2：LEFT JOIN 的 WHERE 吃掉 NULL
```sql
-- ❌ 本意是「列出所有訂單，順便帶上用戶資訊，只要成年用戶」
-- 但 WHERE 會把 user 為 NULL 的訂單也殺掉，LEFT JOIN 變成了 INNER JOIN
SELECT * FROM orders
LEFT JOIN users ON orders.uid = users.id
WHERE users.age > 18;
```

**用 Go 虛擬碼秒懂為什麼錯：**
```go
// Step 2: LEFT JOIN 牽手
// 假設 uid=999 的訂單找不到 user，LEFT JOIN 會幫它補 nil
// matched 裡面產生了一筆： JoinedRow{Order: {uid:999}, User: nil}

// Step 3: WHERE 殺人
for _, row := range matched {
    // 當迴圈跑到 uid=999 這筆時：
    if row.User.Age > 18 { 
        // 💥 Panic (在 SQL 中就是 NULL > 18，判定為 False)
        // 這行被無情淘汰！
    }
}
// 本來 LEFT JOIN 想保留無主訂單的，結果被 WHERE 殺光了。
```

```sql
-- ✅ 正確：把條件放在 ON 裡面，LEFT JOIN 的語義才不會被破壞
SELECT * FROM orders
LEFT JOIN users ON orders.uid = users.id AND users.age > 18;
-- ON 是過濾「牽手的條件」，uid=999 雖然牽不到成年的 User，但它是 LEFT JOIN，所以 Order 本身還是會存活下來！
```

### 陷阱 3：HAVING 能做的事，WHERE 先做更快
```sql
-- ❌ 效能差：先分組再淘汰，浪費了大量 CPU 在分組上
SELECT dept, COUNT(*) AS cnt
FROM employees
GROUP BY dept
HAVING dept != 'intern';
```

**用 Go 虛擬碼秒懂為什麼慢：**
```go
// ❌ 效能差的流程 (留給 HAVING 殺)
// 10 萬筆員工資料，一筆不漏地全部丟進 map 做分組 (非常耗 CPU 和記憶體)
groups := make(map[string][]Row)
for _, row := range employees {
    groups[row.Dept] = append(groups[row.Dept], row)
}
// 分好組後，才把 "intern" 整個 map entry 刪掉...前功盡棄！

// ✅ 效能好的流程 (用 WHERE 提前殺)
var survivors []Row
for _, row := range employees {
    if row.Dept != "intern" {
        // 一開始就無情刷掉 3 萬個實習生
        survivors = append(survivors, row) 
    }
}
// 剩下 7 萬筆再去做 Hash Map 分組，效能起飛！
```
**原則**：能在 WHERE 階段提前殺掉的資料，就不要留到 HAVING。因為 WHERE 在 GROUP BY 之前執行，越早過濾，後續的分組和排序要處理的資料就越少。

---

## 5. 第五樂章：面試口訣 (Finale)

> **「FROM 拼桌 → ON 牽手 → WHERE 殺人 → GROUP BY 組隊 → HAVING 淘汰隊伍 → SELECT 打包 → ORDER BY 排隊 → LIMIT 截取」**

記住這八個字的順序，你就永遠不會搞混 `ON`、`WHERE`、`HAVING` 的差別，也不會再踩到別名不可用、LEFT JOIN 被吃掉的經典陷阱了。
