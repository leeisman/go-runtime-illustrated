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

*   **[Book 1.5: Memory Mapped I/O (mmap)](1_5_MMAP_IO.md)**
    *   **核心隱喻**: 記憶體與硬碟之間的時空隧道。
    *   **重點**: mmap 如何繞過傳統 `read/write` 的 Kernel 緩衝區拷貝，讓硬碟上的 Block 直接映射進 Process 的虛擬位址空間，實現 Zero-Copy 讀取。Kafka/RocketMQ 的底層基石。

*   **[Book 1.6: WAL (Write-Ahead Logging)](1_6_WAL.md)**
    *   **核心隱喻**: 先寫日記再整理房間。
    *   **重點**: Sequential I/O 吊打 Random I/O 的本質原因。WAL 如何透過順序 Append 日誌 + 背景 Checkpoint 落盤，讓 MySQL (Redo Log)、MongoDB (Journal)、PostgreSQL 同時保證「極速寫入」與「斷電不遺失資料」。

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

*   **[Book 3.1: The Scheduler (GMP)](3_1_GO_GMP.md)**: 宏觀調度模型，M, P, G 的三角關係與 Work Stealing。
*   **[Book 3.2: The Goroutine](3_2_GO_ROUTINE.md)**: 微觀視角，G 如何在單一 M 上切換 (g0 stack switch)。
*   **[Book 3.3: The Sysmon](3_3_GO_SYSMON.md)**: 監控線程，負責搶佔 (Preemption) 與死結偵測。
*   **[Book 3.4: The Netpoller](3_4_GO_NETPOLLER.md)**: 網路 I/O 模型，Epoll 與 Runtime 的整合 (非阻塞 I/O)。
*   **[Book 3.5: The Allocator](3_5_GO_ALLOCATOR.md)**: 記憶體分配器 (TCMalloc)，mcache, mcentral, mheap 的三級緩存。
*   **[Book 3.6: The Lock (Mutex)](3_6_GO_LOCK.md)**: 互斥鎖的本質，自旋 (Spinning) 與休眠 (Parking/Futex) 的 Hybrid 機制。
*   **[Book 3.7: The Atomic](3_7_GO_ATOMIC.md)**: 原子操作，硬體總線鎖 (Bus Lock) 與 MESI 協議。
*   **[Book 3.8: The RWMutex](3_8_GO_RWMUTEX.md)**: 讀寫鎖，讀者無鎖路徑與寫者優先級設計。
*   **[Book 3.9: The sync.Map](3_9_GO_SYNC_MAP.md)**: 併發 Map，冷熱分離 (Read/Dirty) 與動態升級機制。
*   **[Book 3.10: The WaitGroup & Cond](3_10_GO_WAITGROUP_COND.md)**: 流程控制的藝術，信號量 (Sema) 與驚群效應 (Thundering Herd)。
*   **[Book 3.11: The Channel](3_11_GO_CHANNEL.md)**: Go 的靈魂。環狀緩衝區 (Ring Buffer) 與鎖的交互，以及 Select 的隨機性實作。
*   **[Book 3.12: The Garbage Collector](3_12_GO_GC.md)**: 記憶體回收。三色標記 (Tri-color)、寫屏障 (Write Barrier) 與 STW 的細節。
*   **[Book 3.13: The Interface](3_13_GO_INTERFACE.md)**: 萬能代理。`eface` 與 `iface` 的底層結構，以及 `itab` 的動態分派成本。
*   **[Book 3.14: The Map](3_14_GO_MAP.md)**: 雜湊表。Hash Bucket (`bmap`) 結構、Tophash 加速與漸進式擴容 (Evacuation)。
*   **[Book 3.15: The Defer & Panic](3_15_GO_DEFER.md)**: 異常處理。Defer 鏈表的執行順序與 `_defer` 結構的演進 (Heap -> Stack -> Open-coded)。
*   **[Book 3.16: The Reflection](3_16_GO_REFLECT.md)**: 鏡像世界。`reflect` 的三大定律，以及 `interface{}` 與 `reflect.Value` 的底層轉換。

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
*   **[Book 4.10: Actor Model](4_10_GO_ACTOR.md)**: 萬物皆 Actor，無鎖併發的終極架構。

---

## 🚀 Book 5: Package Internals & Performance (套件深度解析)

這系列進入 "Pure Go" 的極限領域，探討知名開源套件如何利用底層技巧 (Unsafe, Syscall, RingBuffer) 來榨乾硬體效能。

*   **[Book 5.1: BigCache (Zero GC)](5_1_GO_BIGCACHE.md)**: 繞過 Go GC 的藝術。如何利用 Byte Array 與 RingBuffer 構建百萬級 TPS 緩存。
*   **[Book 5.2: Snowflake (Distributed ID)](5_2_GO_SNOWFLAKE.md)**: 推特雪花算法。如何在分散式系統中無狀態地生成全域唯一 ID。

---

## 🌐 Book 6: Infrastructure Drivers (基礎設施驅動)

離開 Go 的舒適圈，與龐大的外部系統協作。這裡的重點是「連接管理」、「協議優化」與「故障處理」。

*   **[Book 6.1: The MySQL Driver](6_1_GO_MYSQL.md)**: 資料庫驅動。Connection Pool 的實作細節 (`freeConn` slice) 與 Wire Protocol。
*   **[Book 6.2: MySQL InnoDB](6_2_MYSQL_INNODB.md)**: 存儲引擎核心。B+Tree 結構、Buffer Pool、WAL 與 Doublewrite Buffer。
*   **[Book 6.3: MySQL Journey](6_3_MYSQL_JOURNEY.md)**: 查詢的一生。Parser, Optimizer, Execution Engine, 到 Storage Engine 的完整路徑。
*   **[Book 6.4: MySQL Concurrency](6_4_MYSQL_CONCURRENCY.md)**: 併發控制。MVCC, Undo Log, Read View 與隔離級別 (RC vs RR)。
*   **[Book 6.5: SQL Execution Order](6_5_SQL_EXECUTION_ORDER.md)**: 執行順序。FROM, JOIN, WHERE 到 SELECT, ORDER BY 的精確物理執行步驟。
*   **[Book 6.6: MySQL Sharding](6_6_MYSQL_SHARDING.md)**: 分庫分表。水平/垂直分區策略、分片鍵 (Sharding Key) 選擇與分散式事務挑戰。
*   **[Book 6.7: The Redis Driver](6_7_GO_REDIS.md)**: 鍵值存儲。Cluster Protocol、Pipeline 優化、Lua 原子操作與秒殺架構實戰。
*   **[Book 6.8: The Kafka Driver (Core IO)](6_8_GO_KAFKA.md)**: 消息隊列 (Producer & IO)。深入 OS Page Cache, Zero-Copy (`sendfile`), Sequential IO 與 AWS Nitro 硬體優化。
*   **[Book 6.9: The Kafka Consumer](6_9_GO_KAFKA_CONSUMER.md)**: 群體智慧。Consumer Group Rebalance (Stop-The-World vs Cooperative), Offset Commit 策略與 KRaft 架構。
*   **[Book 6.10: RocketMQ (Tiered Storage)](6_10_GO_ROCKETMQ.md)**: 分層存儲。冷熱數據分離架構 (Poller -> S3) 與多層級 CommitLog 設計。
*   **[Book 6.11: NATS (Simplicity)](6_11_GO_NATS.md)**: 極簡主義。Core NATS (At-Most-Once) 與 JetStream (Persistent) 的架構差異，以及 Gossiping 協議。
*   **[Book 6.12: Proto Actor vs Chan](6_12_PROTO_ACTOR_VS_CHAN.md)**: 併發模型對決。Actor 模型的狀態封裝與 Lock-Free 優勢，對比 Go Native Channel 的侷限與適用場景。
*   **[Book 6.13: MongoDB WiredTiger (儲存引擎深潛)](6_13_MONGODB_WT.md)**: NoSQL 底層機制。BSON 記憶體指標跳躍 (Pointer Skip)、Multikey Index 陣列爆裂機制、MVCC Update List、Checkpoint 洗盤、Document-Level Lock 與 MySQL vs MongoDB 深度架構對比。
*   **[Book 6.14: MongoDB Cluster (原生叢集架構)](6_14_MONGODB_CLUSTER.md)**: 分散式擴展。Replica Set 讀寫分離與自動選舉、Sharded Cluster 三元件 (mongos / Config Server / Shard)、Shard Key 熱點問題、Batch 拆包路由機制與 Scatter-Gather 查詢代價分析。

---

## 🌐 Book 7: Network & Architecture (網路架構與攻防)

這系列經過重構，從底層基礎設施一路講到高層應用與攻防，構建完整的網路世界觀。

*   **[Book 7.1: The Infrastructure](7_1_INFRA.md)**: 基礎設施 (L2/L3)。MAC vs IP、Routing Table 邏輯、NAT (SNAT/DNAT) 原理、**K8s Service (ClusterIP)** 與 DNS 解析流程。
*   **[Book 7.2: The Transport](7_2_TRANSPORT.md)**: 傳輸層 (L4)。TCP vs UDP 核心差異、可靠性的代價 (Jitter & HOL Blocking)、**Backpressure (背壓)** 機制與 **QUIC (HTTP/3)** 的逆襲。
*   **[Book 7.3: The Application](7_3_APPLICATION.md)**: 應用層 (L7)。HTTP 的演進 (1.1 -> 2 -> 3/QUIC)、TLS 握手成本與 WebSocket 升級機制。
*   **[Book 7.4: The Operations](7_4_OPS.md)**: 運維實戰。TIME_WAIT vs CLOSE_WAIT 除錯指南、壓測策略 (Load Testing) 與 Nginx 關鍵調校。
*   **[Book 7.5: The Security](7_5_SECURITY.md)**: 網路攻防。SYN Flood (L4), UDP Reflection (L3), Slowloris (L7) 攻擊原理與雲端防禦架構 (Origin Cloaking)。
*   **[Book 7.6: K8s Game Traffic (遊戲流量零代理架構)](7_6_K8S_GAME_TRAFFIC.md)**: 超低延遲遊戲後端。Headless Service 零代理 (Zero-Proxy) 設計、Client-Side gRPC Load Balancing (`dns:///` + `round_robin`)、HTTP/2 多路複用取代 Connection Pool、Service Mesh 的 L7 Sidecar 效能代價分析。
*   **[Book 7.7: K8s 零停機部署 (Probes & Graceful Shutdown)](7_7_K8S_DEPLOY.md)**: 完整發版生命週期。Rolling Update 滾動換班機制、Readiness / Liveness / Startup 三種探針的職責差異、preStop Sleep 解決網路拓撲更新空窗期、Go `server.Shutdown()` 與 gRPC `GracefulStop()` 優雅退場完整實作。
*   **[Book 7.8: K8s 分散式 Actor 叢集 (Smart Gateway & Consistent Hashing)](7_8_K8S_ACTOR_CLUSTER.md)**: 去中心化動態路由。白嫖 K8s API Server 作為註冊中心 (Kubernetes Provider)、Headless Service Endpoints Watch 機制、Smart Gateway 零轉發直連優化，以及基於記憶體的一致性雜湊 (Consistent Hashing) 演算法推演。

---

## 🔧 Appendix (附錄)

*   **[.agent/project_rules.md](.agent/project_rules.md)**: 專案撰寫規範、核心隱喻對照表。
*   **[OS_Metaphor.md](OS_Metaphor.md)**: 原始隱喻發想筆記。

---
*Created by Antigravity Agent & Frankie Li*
