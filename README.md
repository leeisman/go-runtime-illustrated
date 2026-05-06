# The Royal Library: OS & Go Runtime Internals

Welcome to the **Royal Library Documentation Series**, a conceptual guide designed to explain Operating System internals and the Go Runtime using the consistent metaphor of a Library.

這系列文檔通過「皇家圖書館」的隱喻，深入淺出地講解作業系統 (OS) 與 Go Runtime 的底層機制。從 Thread、Memory 到 Network I/O，我們將一步步揭開高效能併發的秘密。

---

## 📚 Book 1: OS Fundamentals (作業系統基礎)

這部分建立了整個世界的觀 (Worldview)，介紹核心角色與 I/O 模型。

*   **[Book 1.1: 皇家圖書館 (The Royal Library)](1_1_THE_LIBRARY.md)**: 建立 CPU、Memory、Kernel、I/O 的皇家圖書館世界觀，理解 DMA、Interrupt 與硬體互動的基礎流程。

*   **[Book 1.2: 線程與排程 (The Thread)](1_2_THE_THREAD.md)**: 拆解 Thread Context、Time Slice 與 Context Switch，理解 OS Thread 為什麼昂貴。

*   **[Book 1.3: 阻塞式網路模型 (Blocking Network I/O)](1_3_BLOCKING_NET_IO.md)**: 從 `listen/accept/read/write` 追蹤 Blocking I/O，理解一個連線如何讓 Thread 睡眠與 C10K 問題的根源。

*   **[Book 1.4: 現代網路模型 (The Modern Epoll Server)](1_4_EPOLL_NET_IO.md)**: 解析 epoll instance、ready list 與 RB-tree，理解 Linux 如何支撐高併發 I/O 與 Go Netpoller。

*   **[Book 1.5: Memory Mapped I/O (mmap)](1_5_MMAP_IO.md)**: 說明 mmap 如何把檔案 block 映射進虛擬位址空間，減少傳統 `read/write` 的拷貝成本。

*   **[Book 1.6: WAL (Write-Ahead Logging)](1_6_WAL.md)**: 解釋 Sequential I/O 與 Write-Ahead Logging，理解 Redo Log、Journal、Checkpoint 如何兼顧寫入速度與崩潰恢復。

---

## 🧠 Book 2: Memory Management (記憶體管理)

探索資料如何在 Stack 與 Heap 之間流動，以及 Go 如何自動管理這些資源。

*   **[Book 2.1: Go Stack (堆疊)](2_1_GO_STACK.md)**: 從函式呼叫與作用域理解 Stack 配置，掌握 Goroutine Stack 的成長與低成本特性。

*   **[Book 2.2: Go Heap (堆積)](2_2_GO_HEAP.md)**: 拆解 Heap 配置、指標追蹤與 GC 掃描成本，理解資料逃出作用域後的代價。

*   **[Book 2.3: 逃逸分析 (Escape Analysis)](2_3_GO_ESCAPE_ANALYSIS.md)**: 用實驗理解編譯器如何判斷 Stack 或 Heap，掌握指標、介面與切片造成逃逸的規則。

*   **[Book 2.4: The Pointer (指針)](2_4_GO_POINTER.md)**: 釐清地址、指標變數與物件本體，理解 `unsafe.Pointer`、`uintptr`、二級指針與 GC 安全性。

*   **[Book 2.5: The Type System (型別的物理本質)](2_5_GO_TYPE.md)**: 從 concrete type 到 interface，理解型別如何決定記憶體解讀方式與 type assertion 的底層機制。

---

## 🧠 Book 3: Go Runtime Internals (腰包裡的秘密)
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
*   **[Book 3.16: The Reflection](3_16_GO_REFLECTION.md)**: 鏡像世界。`reflect` 的三大定律，以及 `interface{}` 與 `reflect.Value` 的底層轉換。
*   **[Book 3.17: The Slice](3_17_GO_SLICE.md)**: 動態陣列的奧義。Slice Header、Backing Array、append 擴容、Sub-slicing 記憶體挾持與常用演算法。

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
*   **[Book 6.15: Apache Cassandra](6_15_CASSANDRA.md)**: 高吞吐分散式資料庫。LSM-Tree 寫入路徑、Read Amplification、Partition Key / Clustering Key、一致性等級、LWT、Compaction 與 Gossip Ring 架構。

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
*   **[Book 7.9: K8s 裡的 gRPC 負載不平衡陷阱 (The gRPC Load Balancing Trap)](7_9_K8S_GRPC_LOAD_BALANCING.md)**: L4 vs L7 的維度打擊。剖析 kube-proxy (TCP DNAT) 與 HTTP/2 多路複用衝突導致的單點流量傾斜，以及 Client-Side LB 與 Service Mesh 的破局架構。

---

## 🧪 Book 8: Testing & Profiling (測試與效能診斷)

這系列收束到工程驗證：如何在測試期抓出併發錯誤，並在生產環境用 profiling 找出真正的瓶頸與洩漏。

*   **[Book 8.1: Data Race Detector](8_1_GO_TEST_RACE.md)**: 開天眼除錯。`go test -race` 的 ThreadSanitizer 追蹤原理、動態分析盲區、CI/CD 中量壓測與極限壓測的職責分離。
*   **[Book 8.2: Goroutine Leak & Pprof Profiling](8_2_GO_TEST_LEAK_PPROF.md)**: 抓漏神器。`uber-go/goleak` 測試期守門、常見記憶體洩漏模式、`net/http/pprof` 生產診斷與 heap/goroutine profile 排查流程。

---

## ⚔️ Book 9: 複合實戰演練 (Composite Practical Exercises)

這系列結合前面各章節的底層知識，探討在真實微服務架構下的高併發防禦、架構設計與極端場景解決方案。

*   **[Book 9.1: 微服務高併發漏斗防禦 (Funnel Defense Architecture)](9_1_FUNNEL_DEFENSE.md)**: 結合 Redis 分散式鎖與 MySQL 樂觀鎖，構建擋下 99% 流量並保證 1% 絕對正確的交易流水線。
*   **[Book 9.2: 資料庫 IOPS 防禦戰術 (Database IOPS Protection Strategies)](9_2_IOPS_DEFENSE.md)**: 從快取防護網、延遲寫入到寫合併與批次處理，榨乾資料庫物理效能極限的防禦兵法。
*   **[Book 9.3: Go 零分配優化戰術 (Zero Allocation Tactics)](9_3_ZERO_ALLOCATION.md)**: 對抗 GC 的極限優化。透過 sync.Pool、逃逸分析控制與零拷貝，將高頻系統的延遲抖動降至最低。
*   **[Book 9.4: 極限無鎖併發與佇列優化 (Lock-Free Concurrency)](9_4_LOCK_FREE_CONCURRENCY.md)**: 突破 Channel 的互斥鎖瓶頸。借鑑 LMAX Disruptor，利用 CAS 與 Ring Buffer 達成奈秒級延遲。
*   **[Book 9.5: 無鎖鏈結串列與 MPSC 佇列 (Lock-Free Linked List & MPSC)](9_5_LOCK_FREE_MPSC.md)**: Actor 模型的底層引擎。結合 Dummy Node 技巧與 atomic.Swap，打造無限容量且零 GC 的極速信箱。

---

## 🔧 Appendix (附錄)

*   **[.agent/project_rules.md](.agent/project_rules.md)**: 專案撰寫規範、核心隱喻對照表。
*   **[OS_Metaphor.md](OS_Metaphor.md)**: 原始隱喻發想筆記。
*   **[archive/old_book7/](archive/old_book7/)**: Book 7 重構前的歷史稿。主線已由 `7_1_INFRA.md` 到 `7_8_K8S_ACTOR_CLUSTER.md` 取代，舊稿僅作追溯與段落回收使用。

---
*Created by Antigravity Agent & Frankie Li*
