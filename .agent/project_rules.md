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

## 3. 關鍵行為描述 (Key Behaviors)

*   **T0 -> M0**: T0 透過 TLS 設定 FS Register 指向 g0，確認身分為 M0。
*   **Work Stealing**: 是全域可見性 (Global Visibility) 的設計結果，允許未來的 M 互相存取 P。
*   **Context Switch (User)**: M0 留在原地 (User Space)，只是轉頭 (Change SP/PC) 看另一張 G。
*   **Context Switch (Kernel)**: M0 必須換房間、脫機甲 (Heavy Operation)。

---
*Last Updated: 2025-12-27*
