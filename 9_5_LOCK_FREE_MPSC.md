# Book 9.5: 無鎖鏈結串列與 MPSC 佇列 (Lock-Free Linked List & MPSC)

在上一章中，我們探討了利用連續記憶體 (Array) 實作的 Ring Buffer。但在微服務與 Actor 模型中，當我們面臨「多個生產者、單一消費者 (MPSC: Multi-Producer Single-Consumer)」的情境時，還有一種極度優雅且被廣泛使用的無鎖設計：**基於 Dummy Node (哨兵節點) 的無鎖鏈結串列**。

這套演算法（源自 Dmitry Vyukov 的設計）是當代高效能 Actor 框架（如 Proto.Actor, Akka）實作底層信箱 (Mailbox) 的標準答案。

---

## 1. 第一樂章：為什麼需要 Dummy Node (哨兵節點)？

在傳統的鏈結串列 (Linked List) 中，操作 `head` 和 `tail` 最怕遇到「佇列為空」或「佇列只剩一個節點」的極端情況，因為這往往需要複雜的 `if-else` 與大鎖來保護邊界。

*   **Dummy Node 魔法**：我們在初始化佇列時，預先塞入一個「不含任何實際資料」的空節點 (Dummy)。
*   **初始狀態**：讓 `head` 和 `tail` 同時指向這個 Dummy Node。
    ```go
    dummy := &Envelope{}
    a.head.Store(dummy)
    a.tail.Store(dummy)
    ```
*   **優勢**：有了 Dummy Node，佇列**永遠不會是空的**（至少會有一個節點），這使得生產者與消費者的指標操作可以完美分離，徹底消除了邊界條件的競態問題 (Race Condition)。

---

## 2. 第二樂章：生產者 - 極速入隊 (Multi-Producer)

生產者可能有多個（例如上百個 Goroutine 同時發訊息給同一個 Actor），必須處理並發寫入。

這段短短三行的程式碼，展現了無鎖併發的暴力美學：
```go
func (a *Actor) sendEnvelope(env *Envelope) {
	env.next.Store(nil)
	
	// 1. 原子交換：把自己的節點設為新的 tail，並把「舊的 tail」拿回來
	prev := a.tail.Swap(env)
	
	// 2. 指標鏈接：把舊 tail 的 next 指向自己的新節點
	prev.next.Store(env)
}
```
*   **無鎖核心 (`Swap`)**：`a.tail.Swap(env)` 是一個硬體級別的原子操作。即使 100 個 Goroutine 同時呼叫，硬體也會排好順序，每個人都能準確拿到自己專屬的「前一個節點 (`prev`)」，指標鏈接絕不會打結。

---

## 3. 第三樂章：消費者 - 巧妙出隊 (Single-Consumer)

在 Actor 模型中，消費者永遠只有一個（單一 Goroutine 負責處理該 Actor 的信箱），這讓出隊邏輯異常純粹，完全不需要 CAS。

```go
func (a *Actor) dequeue() *Envelope {
	head := a.head.Load()
	next := head.next.Load()
    
	if next == nil {
		// ⚠️ 隱藏的併發陷阱：生產者剛執行完 Swap，但還沒執行 prev.next.Store(env)
		if head != a.tail.Load() {
			runtime.Gosched() // 放棄 CPU，等生產者那一行指令跑完
		}
		return nil // 真的沒資料
	}
    
	// 把 head 往前推。
	a.head.Store(next)
	
	PutEnvelope(head) // 回收舊的 Dummy
	return next       // 注意：這個拿出來的 next，會在下一次變成「新的 Dummy Node」！
}
```
*   **微觀的資料不一致 (`runtime.Gosched`)**：因為生產者的 `Swap` 和 `Store` 是兩行獨立指令。消費者在讀取時，可能剛好插在這兩行指令中間。這時 `head != tail` (有人塞資料進來了)，但是 `head.next == nil` (鏈接還沒連上)。我們只需呼叫 `runtime.Gosched()` 稍等一微秒即可。
*   **新陳代謝的 Dummy**：出隊時，其實是把當前的 `head` 丟棄，並把拿出來的節點變成「新的 Dummy Node」。這是一個極度優雅的指標輪替設計。

---

## 4. 第四樂章：與 Ring Buffer 的架構對決

面試官最愛問：「既然有極速的 Ring Buffer，為什麼 Actor 框架還要用 Linked List？」

1.  **記憶體彈性 (Memory Elasticity)**：Ring Buffer 的大小是固定的。如果遇到極端流量突刺 (Spike)，Ring Buffer 一旦塞滿，會導致生產者阻塞 (Block) 或丟棄訊息。Linked List 是動態長度的，只要記憶體的 Heap 沒爆，它可以無限承載（非常適合處理不確定性的 Actor 信箱）。
2.  **Zero Allocation 閉環**：搭配我們在 [Book 9.3 零分配戰術] 中探討的 `sync.Pool`，把用完的 `Envelope` 節點丟回池子裡回收。生產時 `Get()`，消費完 `Put()`。這完美結合了 Linked List 的長度彈性，同時又消滅了瘋狂 `new()` 帶來的 GC 負擔。

---

## 5. 第五樂章：總結 - Actor 模型的基石 (Finale)

**一句話總結**：結合 Dummy Node 與 `atomic.Swap` 的 MPSC 鏈結串列，用三行程式碼完美解決了多生產者的競態問題；搭配 `sync.Pool` 的物件回收，更是為 Actor 模型打造出一個無限容量、無鎖、零 GC 的終極信箱引擎。
