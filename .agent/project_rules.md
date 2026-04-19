# OS Concept Project Rules

此檔案定義了本專案 (OS Concept Documentation) 的撰寫規範與核心隱喻。
Agent 在生成或修改文檔時，必須遵循以下規則。

## 1. 章節命名風格 (Naming Convention)

為了保持文檔的史詩感 (Epic Narrative)，章節標題應遵循以下層級與命名：

*   **H1 (`#`)**: 僅用於檔案最上方的書本標題。
    *   格式: `# Book X.Y: [標題] ([English Title])`
*   **H2 (`##`)**: 用於主要章節，必須使用 **「樂章 (Movement)」**。
    *   格式：`## X. 第X樂章：[中文標題] ([English Title])`
    *   範例：`## 1. 第一樂章：起源 (The Origin)`
*   **H3 (`###`)**: 用於樂章內的具體步驟、場景或子主題。
    *   範例：`### Step 1: 建立監控中心` 或 `### 場景 1: 創造`

## 1.1 章節分隔線 (Section Divider)

為了保持閱讀節奏與視覺一致性，書本標題與每一個主要樂章之間必須使用水平分隔線：

*   **H1 書本標題後**：在前言或第一樂章開始前，使用一行 `---`。
*   **每個 H2 樂章之間**：在下一個 `## X. 第X樂章：...` 開始前，使用一行 `---`。
*   **H3 以下子章節之間**：不強制使用 `---`，除非該段落需要明確切換場景。

範例：

```md
# Book X.Y: 中文標題 (English Title)

前言文字。

---

## 1. 第一樂章：起源 (The Origin)

內容。

---

## 2. 第二樂章：展開 (The Expansion)
```

## 1.2 文件骨架 (Document Skeleton)

每一篇技術文件應遵循「先建立直覺，再深入底層，最後收束實戰」的順序：

*   **H1 下方開場**：用 2~5 段短文字說明本篇要解決的核心問題，避免一開始就堆 API 或名詞。
*   **第一樂章**：通常用來建立隱喻、問題背景或核心直覺。
*   **中段樂章**：逐步拆解資料結構、執行流程、系統呼叫、協議流程、併發機制或效能代價。
*   **實戰樂章**：若主題與工程實務有關，應加入故障場景、架構取捨、調參指南、常見誤區或壓測觀察。
*   **最後樂章**：必須是總結、決策準則或面試一句話收束，不要讓文章停在程式碼或細節列表上。

建議樂章命名範式：

*   `隱喻 (The Metaphor)`
*   `結構 (The Structure)`
*   `流程 (The Flow)`
*   `底層原理 (Under the Hood)`
*   `代價與陷阱 (Cost & Pitfalls)`
*   `實戰應用 (Production Practice)`
*   `總結 (Summary / Finale)`

## 1.3 標題語言風格 (Title Style)

標題應兼具中文敘事感與英文技術錨點：

*   中文標題負責建立畫面感，例如「可靠性的幻覺」、「連線的解剖」、「地底下的國家檔案局」。
*   英文括號負責對齊技術名詞，例如 `(Byte Stream)`、`(Under the Hood)`、`(Flow Control)`。
*   避免只有英文標題，也避免只有中文詩意標題而看不出技術主題。
*   若標題已有清楚英文專有名詞，可保留在中文標題中，但仍建議補英文括號。
*   「總結」、「終章」、「Finale」、「Summary」都必須包進樂章格式，不可單獨寫成 `## Summary`。

## 1.4 敘事與語氣 (Narrative Voice)

本專案不是單純 API 筆記，而是技術世界觀教材。撰寫時應保持以下語氣：

*   **先講人能理解的畫面，再講機器真正做的事。**
*   可以使用強烈比喻與場景感，但不能犧牲技術精準度。
*   每個比喻後應回到真實技術名詞，讓讀者知道「故事」對應到哪個 OS、Runtime、Network、Database 或 Distributed System 概念。
*   允許使用「真相」、「代價」、「陷阱」、「幻覺」、「生死線」等敘事詞，增強記憶點。
*   避免堆砌情緒詞；如果語氣很燃，下一段應補上機制、流程或工程判斷。
*   讀者預設是有後端與 Go 經驗的工程師，可以直接使用 `syscall`、`goroutine`、`epoll`、`WAL`、`ACK`、`MVCC` 等術語，但首次出現時最好給直覺解釋。

## 2. 核心隱喻對照表 (Core Metaphors)

請嚴格維持以下隱喻的一致性，避免混用舊版隱喻 (如：裝備、車子)。

| 實體 (Entity) | 隱喻 (Metaphor) | 描述 (Description) |
| :--- | :--- | :--- |
| **M (Machine)** | **館長 (Librarian)** | 擁有執行能力的 CPU Thread。可以切換不同任務。 |
| **M0** | **初始館長** | 程式啟動時的第一個 Thread (T0)。 |
| **g0 (System Stack)** | **主閱讀桌 / 控制台** | M 的固定工作台 (8MB Stack)。處理調度、行政雜務的地方。 |
| **P (Processor)** | **待辦清單區域 (List Area)** | 黑板上劃分出的區域，用來掛 Local Queue。 |
| **G (Goroutine)** | **任務單 / 章節** | 輕量級的任務紙條。包含 PC 和 SP 資訊。 |
| **Heap** | **大黑板 / 公共區** | 所有物件 (P, G, M struct) 存放的公共記憶體空間。 |
| **Netpoller** | **網路掛號處 / Epoll 櫃** | 處理 I/O 阻塞的地方。M 把 G 寄放在這裡，自己不等待。 |

## 2.1 隱喻使用規則 (Metaphor Rules)

*   同一篇文章內的隱喻必須穩定，不要在同一個技術實體上混用多套世界觀。
*   若舊文使用「廚房、廚師、訂單、裝備、車子」等舊隱喻，新增或重寫時應優先轉回「圖書館 / 行政區 / 館長 / 任務單 / 大黑板」體系。
*   隱喻不能替代技術定義。每個重要隱喻第一次出現時，應明確標出真實對應，例如 `P (Processor)`、`G (Goroutine)`、`TCB`、`Page Cache`。
*   實戰案例可以使用使用者既有戰功名錄中的背景：高併發遊戲、Gateway batching、Write-behind log、Redis atomic jackpot、Rate Limiter、BigCache、SingleFlight、Zookeeper jitter。

## 3. 關鍵行為描述 (Key Behaviors)

*   **T0 -> M0**: T0 透過 TLS 設定 FS Register 指向 g0，確認身分為 M0。
*   **Work Stealing**: 是全域可見性 (Global Visibility) 的設計結果，允許未來的 M 互相存取 P。
*   **Context Switch (User)**: M0 留在原地 (User Space)，只是轉頭 (Change SP/PC) 看另一張 G。
*   **Context Switch (Kernel)**: M0 必須換房間、脫機甲 (Heavy Operation)。

## 4. 技術精準度 (Technical Accuracy)

寫作時可以簡化，但必須標明簡化邊界：

*   使用常見數值時要說明條件，例如 `MSS = 1460` 是 Ethernet MTU 1500、IPv4 無 options 時的常見例子，不是固定真理。
*   描述「保證」時要區分層級，例如 TCP 的可靠是 L4 byte stream，不等於 L7 商業成功。
*   避免把相近概念混用，例如 TCP segmentation 不等於 IP fragmentation，flow control 不等於 congestion control。
*   遇到 OS、Runtime、Network、Database 的行為時，應盡量說明「誰在做」：User Space、Kernel、NIC、Go Runtime、Broker、Storage Engine、Coordinator。
*   若為了教學把序號、狀態機或流程簡化，必須用一句話標註「以下為簡化模型」。
*   不確定或版本相關的技術細節，不要寫死成絕對語氣；可用「通常」、「常見」、「在 Linux/Go/Kafka 某版本中」。

## 5. 程式碼、圖解與流程 (Code, Diagrams & Flow)

*   程式碼區塊必須標註語言，例如 `go`、`text`、`sql`、`mermaid`。
*   程式碼應服務於概念，不要為了完整可執行而塞入大量樣板。
*   系統流程建議使用 `text` 時序圖、步驟列表或 Mermaid，尤其適合描述 syscall、TCP handshake、GC cycle、Kafka journey、K8s traffic flow。
*   步驟型 H3 可使用 `### Step 1: ...`；場景型 H3 可使用 `### 場景 1: ...`。
*   每段流程後應補一句「這代表什麼工程意義」，避免讀者只看懂順序，卻不知道代價與取捨。

## 6. 實戰與總結 (Production Practice & Summary)

每篇文章若涉及後端、分散式、資料庫、網路或 Go Runtime，應盡量補上實戰視角：

*   **故障徵兆**：例如 `Send-Q` 堆積、CPU 100%、GC pause、rebalance storm、cache penetration、goroutine leak。
*   **排查方向**：先看哪些指標、哪個元件可能是瓶頸、如何避免誤判。
*   **架構取捨**：說明為什麼選 A 不選 B，以及 A 的代價。
*   **常見誤區**：用明確句子打破錯誤直覺，例如「Write 成功不代表對方收到」。
*   **最後樂章**：必須收斂成可記憶的條列、決策樹或一句話總結。

## 7. 維護與改稿原則 (Maintenance Rules)

*   只做格式正規化時，不改正文技術內容。
*   技術內容改寫時，必須同時檢查標題格式、分隔線、隱喻一致性與總結是否存在。
*   批次修改大量文件時，優先使用機械化規則，避免混入主觀重寫。
*   舊稿 `OLD_*` 也應遵守格式規則，但除非明確要求，不主動重寫舊稿技術內容。
*   若檔名是 `X_Y_...md`，H1 必須對應 `# Book X.Y: ... (...)`。
*   `README.md`、`OS_Metaphor.md`、`.agent/*` 屬於專案輔助文件，不強制套用 Book 樂章格式。

## 8. README 目錄規格 (README Index Rules)

`README.md` 是整個系列的入口頁與路線圖，不是單篇 Book 文章，因此使用獨立格式：

*   **H1**：使用專案總標題，例如 `# The Royal Library: OS & Go Runtime Internals`。
*   **H2**：以 Book 群組為主，例如 `## Book 3: Go Runtime Internals`，可保留 emoji 作為視覺索引。
*   **每個 Book 檔案必須在 README 收錄**：根目錄下所有 `X_Y_*.md` 都要出現在 README；README 也不能連到不存在的 Book 檔案。
*   **列表格式**：每篇使用一行 bullet：
    *   `*   **[Book X.Y: Short Title](X_Y_FILE.md)**: 一句話說明。`
*   **一句話說明**：應包含該篇的核心技術關鍵字與讀者會學到的能力，不要只寫抽象形容詞。
*   **順序**：必須依 Book 編號排序，先 `1.1 -> 1.2`，再 `2.1 -> ...`。
*   **群組開場**：每個 Book 群組下方可有 1~2 句路線說明，交代這組文章在整個系列中的角色。
*   **Archive**：歷史稿或舊版內容應放在 Appendix，明確標註「僅作追溯與段落回收使用」，不要混入主線 Book 列表。
*   **更新檢查**：新增、移動、刪除任何 `X_Y_*.md` 時，必須同步更新 README，並檢查：
    *   README 連結是否存在。
    *   是否有實際 Book 未列入 README。
    *   是否有 README 連到非主線或舊檔。

---
*Last Updated: 2026-04-19*
