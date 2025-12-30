# The Royal Library: OS & Go Runtime Internals

Welcome to the **Royal Library Documentation Series**, a conceptual guide designed to explain Operating System internals and the Go Runtime using the consistent metaphor of a Library.

這系列文檔通過「皇家圖書館」的隱喻，深入淺出地講解作業系統 (OS) 與 Go Runtime 的底層機制。從 Thread、Memory 到 Network I/O，我們將一步步揭開高效能併發的秘密。

---

## 📚 Book 1: OS Fundamentals (作業系統基礎)

這部分建立了整個世界的觀 (Worldview)，介紹核心角色與 I/O 模型。

*   **[Book 1.1: 皇家圖書館 (The Royal Library)](1_1_THE_LIBRARY.md)**
    *   **核心隱喻**: CPU (館長), Memory (閱覽區), Kernel (行政區), I/O (外部出版社)。
    *   **重點**: 認識所有角色與硬體互動的基礎流程 (DMA, Interrupt)。

*   **[Book 1.2: 線程與排程 (The Thread)](1_2_THE_THREAD.md)**
    *   **核心隱喻**: 線程 (裝備), Context Switch (換裝備), Time Slice (工時)。
    *   **重點**: 為什麼 OS Thread 切換這麼貴？深入理解 CPU 的執行單元。

*   **[Book 1.3: 阻塞式網路模型 (Blocking Network I/O)](1_3_BLOCKING_NET_IO.md)**
    *   **核心隱喻**: 銀行櫃台模式。
    *   **重點**: 傳統 `read/write` 如何運作？為什麼一個連線卡住會導致整個 Thread 睡眠？C10K 問題的根源。

*   **[Book 1.4: 現代網路模型 (The Modern Epoll Server)](1_4_EPOLL_NET_IO.md)**
    *   **核心隱喻**: 監控室 (Epoll Instance), 點名單 (Ready List), 紅黑樹 (RB-Tree)。
    *   **重點**: Linux 如何用 O(1) 的效率解決 C10K 問題。Go Netpoller 的底層基石。

---

## 🧠 Book 2: Memory Management (記憶體管理)

探索資料如何在 Stack 與 Heap 之間流動，以及 Go 如何自動管理這些資源。

*   **[Book 2.1: Go Stack (堆疊)](2_1_GO_STACK.md)**
    *   **核心隱喻**: 隨身筆記本 (Tissue Box)。
    *   **重點**: 函式呼叫的本質，Goroutine Stack 的自動擴展 (Continuous Stack)。

*   **[Book 2.2: Go Heap (堆積)](2_2_GO_HEAP.md)**
    *   **核心隱喻**: 公共黑板。
    *   **重點**: 動態記憶體分配，GC 的掃描區域。

*   **[Book 2.3: 逃逸分析 (Escape Analysis)](2_3_GO_ESCAPE_ANALYSIS.md)**
    *   **核心隱喻**: 審查員。
    *   **重點**: 編譯器如何決定變數該放在 Stack 還是 Heap？指針逃逸的規則。

*   **[Book 2.4: The Pointer (指針)](2_4_GO_POINTER.md)**
    *   **核心隱喻**: 繫繩 (Anchor) 與坐標 (Coordinate)。
    *   **重點**: `unsafe.Pointer` 與 `uintptr` 的區別。為什麼 `uintptr` 是 GC 不安全的？二級指針 (`**T`) 的應用。

*   **[Book 2.5: The Type System (型別的物理本質)](2_5_GO_TYPE.md)**
    *   **核心隱喻**: 裸體 (Naked) 與制服 (Uniform)。
    *   **重點**: 為什麼具體型別沒有 Type ID？Type Assertion 的底層機制。

---

## 3. Go Runtime Internals (腰包裡的秘密)
這系列深入 Go Runtime 的底層機制，從調度器到記憶體管理，再到併發原語的實作細節。

*   **[3.1 The Scheduler (GMP)](3_1_GO_GMP.md)**: 宏觀調度模型，M, P, G 的三角關係與 Work Stealing。
*   **[3.2 The Goroutine](3_2_GO_ROUTINE.md)**: 微觀視角，G 如何在單一 M 上切換 (g0 stack switch)。
*   **[3.3 The Sysmon](3_3_GO_SYSMON.md)**: 監控線程，負責搶佔 (Preemption) 與死結偵測。
*   **[3.4 The Netpoller](3_4_GO_NETPOLLER.md)**: 網路 I/O 模型，Epoll 與 Runtime 的整合 (非阻塞 I/O)。
*   **[3.5 The Allocator](3_5_GO_ALLOCATOR.md)**: 記憶體分配器 (TCMalloc)，mcache, mcentral, mheap 的三級緩存。
*   **[3.6 The Lock (Mutex)](3_6_GO_LOCK.md)**: 互斥鎖的本質，自旋 (Spinning) 與休眠 (Parking/Futex) 的 Hybrid 機制。
*   **[3.7 The Atomic](3_7_GO_ATOMIC.md)**: 原子操作，硬體總線鎖 (Bus Lock) 與 MESI 協議。
*   **[3.8 The RWMutex](3_8_GO_RWMUTEX.md)**: 讀寫鎖，讀者無鎖路徑與寫者優先級設計。
*   **[3.9 The sync.Map](3_9_GO_SYNC_MAP.md)**: 併發 Map，冷熱分離 (Read/Dirty) 與動態升級機制。
*   **[3.10 The WaitGroup & Cond](3_10_GO_WAITGROUP_COND.md)**: 流程控制的藝術，信號量 (Sema) 與驚群效應 (Thundering Herd)。
*   **[3.11 The Channel](3_11_GO_CHANNEL.md)**: Go 的靈魂。環狀緩衝區 (Ring Buffer) 與鎖的交互，以及 Select 的隨機性實作。
*   **[3.12 The Garbage Collector](3_12_GO_GC.md)**: 記憶體回收。三色標記 (Tri-color)、寫屏障 (Write Barrier) 與 STW 的細節。
*   **[3.13 The Interface](3_13_GO_INTERFACE.md)**: 萬能代理。`eface` 與 `iface` 的底層結構，以及 `itab` 的動態分派成本。
*   **[3.14 The Map](3_14_GO_MAP.md)**: 雜湊表。Hash Bucket (`bmap`) 結構、Tophash 加速與漸進式擴容 (Evacuation)。
*   **[3.15 The Defer & Panic](3_15_GO_DEFER.md)**: 異常處理。Defer 鏈表的執行順序與 `_defer` 結構的演進 (Heap -> Stack -> Open-coded)。
*   **[3.16 The Reflection](3_16_GO_REFLECT.md)**: 鏡像世界。`reflect` 的三大定律，以及 `interface{}` 與 `reflect.Value` 的底層轉換。

---

## 🛡️ Book 4: Concurrency Patterns & Resilience (併發模式與韌性)

這系列從底層機制轉向應用架構，探討如何利用 Go 的特性構建高可用系統。

*   **[Book 4.1: The Pipeline](4_1_GO_PIPELINE.md)**: 流水線模式，Fan-Out/Fan-In 與錯誤傳遞。
*   **[Book 4.2: The Worker Pool](4_2_GO_WORKER_POOL.md)**: 限制併發度，保護下游資源 (DB/API)。
*   **[Book 4.3: Rate Limiting](4_3_GO_RATELIMIT.md)**: 流量控制，Token Bucket 與 Leaky Bucket 實作。
*   **[Book 4.4: The Context](4_4_GO_CONTEXT.md)**: 請求範疇 (Request Scope) 的取消訊號與數值傳遞底層。
*   **[Book 4.5: Circuit Breaker](4_5_GO_BREAKER.md)**: 斷路器模式，快速失敗 (Fail Fast) 以避免資源耗盡。
*   **[Book 4.6: Singleflight](4_6_GO_SINGLEFLIGHT.md)**: 合併請求模式，解決 Cache 擊穿 (Thundering Herd) 問題。
*   **[Book 4.7: Cache Penetration](4_7_GO_BLOOM_FILTER.md)**: 布隆過濾器，防禦惡意隨機 Key 攻擊。
*   **[Book 4.8: ErrGroup](4_8_GO_ERRGROUP.md)**: 併發聚合模式，處理多任務並行與錯誤傳播。
*   **[Book 4.9: Sharding](4_9_GO_SHARDING.md)**: 有序分片模式，在並發中保證局部順序性。
*   **[Book 4.10: Actor Model](4_10_GO_ACTOR.md)** (待補): 萬物皆 Actor，無鎖併發的終極架構。

---

## 🔧 Appendix (附錄)

*   **[.agent/project_rules.md](.agent/project_rules.md)**: 專案撰寫規範、核心隱喻對照表。
*   **[OS_Metaphor.md](OS_Metaphor.md)**: 原始隱喻發想筆記。

---
*Created by Antigravity Agent & Frankie Li*
