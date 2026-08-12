# Squarepoint R1 — FINAL PLAN (v6) — Aug 4–19, Interview Aug 20

# PART 1 — DAY BY DAY

## Aug 5 (Wed) — 3.5h

1. Read Appendix F (protocol) + A.1 (map spec), mumbling the talk-track sentences aloud — the five std::map facts and two-child erase especially (1h).
2. **Map solo attempt #1** — 40 min blank file, narrated. It will be rough; that's the calibration reading. 20-min written review: where exactly did you stall (1h).
3. **Bank: next group** (first of the remaining ten, printed order) (1h).
The mechanism rule starts today: apply it to every bank answer you write.
4. **LC 146 LRU** — solve, then read A.8 (0.5h). 

----
Completed
- Const bank was done yesterday. Have yet to follow up on it
- Map: initial half. Left with rebalancing logic
- std::vector recap
- std::shared_ptr recap
- New: coding checklist
- LRU cache recap

To follow up on
- LFU cache
- Coding checklist - is it good enough? 
	- CANNOT make simple mistakes like syntax stuff (missing return statement, not all paths return, wrong paramters (const, referencedness,..), missing return type declaration, etc.)
	- More common mistakes caught today: 
		- Not checking for null pointer. Huge issue for things like shared_ptr and vector
	- Need to be SURE of syntax and rules for special member functions, including noexcept laws
- Map
	- Templates, concepts and type erasure - follow up (especially during the std::function section)
	- Need to write down the rules for each function. KIV
- vector
	- Revise special member functions. Still not confident
	- const, noexcept etc - coding checklist
	- Emplace_back
		- Unlike push_back, has return value
		- How to use template args... and std foward (`std::forward<Args>` and not `std::forward<Element>`)
	- size and capacity check
		- Overflow? Degenerate vlaue (size of 0)
	- Exception safety levels
		- Can be more precise. Especially when allocating. KIV this, come back to it
	- HUGE mistake
		- Not checking if data_ is nullptr before deallocating
			- Coding checklist - nullptr check
	- **MUST come back to this**
		- Other things that claude pointed out
- shared_ptr
	- Biggest mistake: forgot to tryDelete() before move/copy assignment operators
		- Coding checklist: release before asigning to prevent dangling memory
- LRU cache
	- Unsure of splice() syntax
	- Mistake for map insert() - insert a **pair**
	- Completely dropped condition (common bug) - the capacity check

## Aug 6 (Thu) — 3.5h

1. **Appendix B mechanics sheet — first full read**, slow; mark surprises for daily re-reads from Aug 8 (1h).
2. **Bank: next group** (1h).
3. **Order book on paper** — API + three-container design + the "list iterators survive erasure" sentence, handwritten from A.2 (0.5h).
4. **LC 981** — the member-upper_bound-with-sentinel idiom again (0.5h).
5. **Currency exchange read-through** (A.10) — absorb −log / Bellman-Ford / arbitrage; no coding (0.5h).
6. **Generate both master lists now** — paste the C.1 and C.2 prompts, save the two outputs alongside this file. Five minutes of work that every OS/arch block from Aug 8 onward depends on; doing it on a trip day costs you nothing later (0.25h).

----
Completed
- Bank templates - half done only. 
- Leetcode 981 - solved cleanly BUT
	- Needed help on std::upper_bound with custom comparator
	- Finally got the notes done on this, see [[7.14 std lower bound and upper bound]]
		- **Upper bound is (value, element)**
		- **Upper bound is first TRUE for comp(value, element)**
		- Reverse the 2 above for lower bound.
	- Forgot a `&` for the compartor.
- Generated OS and comp arch lists.
- Order book
	- Did half only. BAD. Need to recap this badly. Mostly forgotten.

## Aug 7 (Fri) — 0h

Travel. If trivially convenient: open HackerRank CodePair once. Otherwise nothing.

## Aug 8 (Sat) — 7.5h — MoldCast day 1

1. MoldCast: SPSC hardening — alignas(64) + cached opposite-index reads (1h).
2. MoldCast: wiring part 1 — RX (epoll/recv/parse/gap-tracking) → SPSC → consumer counter; fixed slots own bytes; drop+counter on full (3.5h).
3. **Order book solo #1** (1h).
4. **OS block 1 — master-list categories 1–3** (processes & lifecycle, threads vs processes, scheduling). Cover-the-answer drill per C.5; misses → ledger (1h).
5. **OSTEP ch. 4** (1h).
6. **LC splice: 1574 Shortest Subarray to be Removed to Make Array Sorted** (1931, array/two-pointer). Offered Aug 2 — if you finished it then, tell me and I'll swap (0.5h).
----
Completed
- SPSC queue (without linking), testing
- OS mater-list categories 1, 2.
- Order book

Backlog
- Order book review
	- Iterators
- Post it notes
	- std::erase
	- Key stl structures, alrogithms, iterator rules
	- coding checklist
- SPSC wiring 

## Aug 9 (Sun) — 7.5h — MoldCast day 2

1. MoldCast: finish wiring — `atomic<bool> `+ jthread shutdown, clean teardown, end-to-end smoke test with the gap path exercised (3.5h).
2. **Locked SPSC blank redo** (A.6a) — bar: all three Jul-31 bug classes clean (1h).
3. **OSTEP ch. 5** — THE fork chapter; do the output examples mentally; retires your ★fork miss (1h).
4. **Git recap** (E.2) — all of it (1h).
5. **Bank: next group** (1h).
6. **LC splice: 1567 Maximum Length Subarray With Positive Product** (1710, DP — deliberately light on a MoldCast-heavy day) (0.5h).
----
Completed
- SPSC queue integration - YAY
- Lock-free SPSC blank redo
- Locked SPSC blank redo
- Cheatsheet post it notes
- Git notes
- 
## Aug 10 (Mon) — 7.5h — MoldCast day 3, then FROZEN

1. MoldCast: benchmarks — throughput + gap p50/p99, taskset. **Write down and memorize both numbers** (3h).
2. MoldCast: README + repo public (1h).
3. **Thread pool #1** (A.5), condvars hot from the wiring (1.5h).
4. **OSTEP ch. 6** — syscalls, modes, context switch (1h).
5. **OS block 2 — master-list categories 4–7** (syscalls & privilege, virtual memory, memory layout, heap internals); cover-the-answer, misses → ledger (0.5h). **LC 1396** (0.5h).
----
Completed
- Cheatsheet notes
- Implement file system (without any virtual)
- Thread pool
	- Vector and notes
- std::functional, optional

Tomorrow:
- Pool allocator
- Recap T[], unique ptr etc.
- Memory, array and placement new notes 
- Moldcast wrapup
- Finish OS + OSTEP backlogs
- Map?

## Aug 11 (Tue) — 7.5h — OOD core begins

1. **Map #2 — interviewer-mode.** Target: insert + find + one rotation narrated inside 40 min (1.5h).
2. **std::function wrapper #1** (A.3 variant b) — type-erasure build; write the `Wrapper<R(Args...)>` partial-specialization line three times (1.5h).
3. **Bank: STL-internals / invalidation group** — the priority pull-forward (1.5h).
4. **Comp arch block 1 — master-list categories 1–4** (hierarchy & latency numbers, cache organization, cache-friendly code, coherence & false sharing), then the 60-second cache script aloud five times, numbers included (1h).
5. **OSTEP ch. 13 + 18** (1.5h). Sheet marked items (0.5h).
6. **LC splice: 2008 Maximum Earnings From Taxi** (1871, DP + binary search — the one you deferred from your own Aug 1 set) (0.5h).
Plan for today
- 2.30-3 pack and prepare
- 3-3.30 std::optional, T[] and make_unique recap (missed)
- 3.30-4.30 pool allocator
- 4.30-5 condense sqpt and jump questions, rest
- 5-6 OS until block 4
- 6-8 dinner and rest
- 8-9 Comp arch until block 4
- 9-1 Moldcast wrapup

## Aug 12 (Wed) — 7.5h

1. **Order book #2 — interviewer-mode** ("make cancel O(1)", "modify keeping time priority", "why not deque"). Final 15 min: the **matching extension** — while qty remains and the best opposite level crosses, fill FIFO against the front orders, pop filled orders, erase emptied levels. A later-round sighting demanded a full matching engine in ~15 minutes and rejected the candidate for erase-while-iterating; the `it = c.erase(it)` idiom is the graded moment (1.5h).
2. **Vector blank redo** (A.4) — ledger-audited (1.5h).
3. **Bank: next group** (1.5h).
4. **OS block 3 — master-list categories 11–14** (IPC, file descriptors & I/O, I/O multiplexing, signals); narrate epoll off your own EpollWrapper; misses → ledger (1h).
5. **OSTEP ch. 19 — TLBs**, full read (1h).
6. **⚡ 25-min verbal mini-diagnostic** — I fire OS blocks 1+2 material at interview pace; this is your first calibration of the spoken format, eight days before the mocks would otherwise be it. Misses drive the next four days (0.5h).
7. Networking: TCP vs UDP, handshake, state machine, TIME_WAIT, multicast/IGMP via your project (0.5h).
8. **LC splice: 355 Design Twitter** (design maintenance — heaps + hash maps + k-way merge, half OOD by nature) (0.5h).

## Aug 13 (Thu) — 7.5h

1. **Range Module blank redo #1** (A.7) — both guards in the first draft; run both counterexamples before calling it done (1h).
2. **Filesystem #1** (A.9) — full build (1.5h).
3. **⚡ Debug session #1** — I hand you a deliberately broken class (bugs seeded from your own ledger categories: bytes-vs-elements, growth-from-zero, self-assignment, SWO, off-by-one destroy); you find them against a 25-min clock. This exists because the IITD R1 was debug-a-broken-vector and your Jul-25 attempt at exactly that went 0-for-3 (0.75h).
4. **Bank: concurrency/memory-model group + code-reading group** — code-reading pairs with today's debug work. Add **comp-arch master-list category 12** (how atomics map to hardware) here rather than in a comp-arch block — it's the same material from the other side, and the Oct-2025 "mechanical sympathy" account puts it on the graded surface (1.5h).
5. **Comp arch block 2 — master-list categories 5–8 and 10** (VM hardware/TLB, pipelining, branch prediction, ILP, alignment); mixed 30s drill; misses → ledger (1h).
6. **Reading: Igoro cache-effects gallery + the sorted-array SO answer + a latency-numbers table** (1h).
7. Linux commands quick-fire (0.75h).
8. **LC splice: 1793 Maximum Score of a Good Subarray** (1946, two-pointer/greedy expansion) (0.5h).

## Aug 14 (Fri) — 7.5h

1. **Map #3** — should now hit the 30-min bar (1h).
2. **Thread pool #2 — interviewer-mode**, optimization ladder recited (1.5h).
3. **Currency exchange session** (A.10) — full build + the arbitrage narration (1.5h).
4. **⚡ Trie-as-OOD session** (A.11) — new; converts an accepted risk with an actual Squarepoint OA sighting into covered ground (1.5h).
5. **Bank: next group** (1.5h).
6. OSTEP ch. 20 first half + ch. 39, skim (0.25h).
7. **OS block 4 — master-list categories 8, 9, 10, 15** (synchronization primitives, deadlock, data races vs race conditions, linking & loading). These four had no home block before; 8–10 are the OS-framed version of concurrency you know in C++ terms, and 15 (static vs dynamic linking, what ldd shows) was fully orphaned. Misses → ledger (0.75h).
8. **LC splice: 1705 Maximum Number of Eaten Apples** (1929, greedy + min-heap of expiries) (0.5h).

## Aug 15 (Sat) — 7.5h

1. **Lock-free SPSC blank redo** (A.6b) (1.25h).
2. **std::function wrapper #2** (variant a) + withstand the grilling: SBO, allocation, clone/copyability, move_only_function (1h).
3. **⚡ DP-optimization drill** — the specific skill one verified candidate failed R1 on: rolling-array space compression and memo→tabulation, practiced on problems you've already solved (983, 1416) — optimization moves, not new problems. This IS today's LC splice; optionally add **LC 1930 Unique Length-3 Palindromic Subsequences** (1533, quick strings/prefix) only if the drill finishes early (1h).
4. **⚡ Maintenance trio:** smart pointers verbal pass (on the verified Oct-2025 trivia list, currently your strength — keep it warm: unique/shared/weak internals, control block, make_shared vs shared_ptr(new), enable_shared_from_this, deleters) + `char**` drill (argv-style iteration, in-place string ops, const with double pointers — the IITD R2 exercise; your prep is all modern C++ and that question deliberately isn't) + estimate-pi Monte Carlo one-liner (1h).
5. **⚡ Micro-topics block (A.12)** — exception safety levels, std::optional, weak_ptr's two use cases, semaphore vs mutex: spoken 30-second answers each. Closes the gaps you flagged plus two verbatim-sighted questions (1h).
6. **Bank: next group** (1.25h).
7. **LRU as OOD narration**, blank rebuild, 25 min (0.5h). Mixed verbal drill — OS + arch + net + git shuffled, and pick up **comp-arch master-list categories 9 and 11** (SIMD awareness, NUMA) here at one-liner depth; that closes every category in both lists (0.5h).

## Aug 16 (Sun) — 7.5h — MOCK 1

1. **Mock 1, format A:** 5' intro / 35' OS + comp-arch + C++ internals quick-fire / 30' unseen OOD. Graded on conclusions AND mechanisms — the mechanism rule is a scoring line (1.5h incl. debrief).
2. **Patch the top bleeding items** + any bank-group top-ups owed — this window is sacred; it absorbs whatever the mock exposes (1.75h). 2b. **⚡ A.13 build block: hash map from scratch (45') + binary heap (20')** — the two implementation gaps from the Aug-3 audit; the rehash-invalidation payoff sentence is the graded moment (1.25h).
3. **Range Module redo #2 — the clean bar** (0.5h).
4. **Ledger full read, part 1** — categories aloud: G, E, F, C, SWO (1h).
5. **Constructor-call-counting output drills** — the exact IITD R2 exercise: predict ctor/copy/move/dtor counts for snippet sets (elision, pass-by-value, return paths, vector growth). Bank groups 1–2 already covered this territory; one focused hour, not more (1h).
6. Reflex check: cache numbers, fork returns, dangling-vs-leak, benchmark numbers (0.5h).

## Aug 17 (Mon) — 7.5h

1. **Weakest two OOD questions, one round each** — default = map + whatever Mock 1 exposed (2h).
2. **⚡ Debug session #2** — new broken class, different seeded bug set, 25-min clock; the bar is beating your session-#1 find rate (0.75h).
3. **Bank: final remaining group(s)**, graded (1.5h).
4. **Second pass: filesystem OR currency**, whichever felt shakier (0.75h).
5. Mixed verbal drill #2, interview pacing (0.75h), then **patterns + rate-limiter/logger narration** (A.12): Meyers singleton, observer, factory, CRTP one-liners + the token-bucket and async-logger stories (0.75h).
6. **Resume stories aloud** — every bullet, two sentences each, per D.2 (1h).
7. **LC splice: 2439 Minimize Maximum of Array** (1965, binary-search-on-answer / prefix-average greedy) — last fresh problem of the prep; nothing new after this except mock content (0.5h).

## Aug 18 (Tue) — 7.5h — MOCK 2

1. **Mock 2, format B:** 5' / 15' C++ trivia / 35' narrated live coding (style-graded; interval-set or design-LC flavor) / 5' your questions. Mechanism rule scored again (1.5h incl. debrief).
2. **Patch** (2h).
3. **Map blank redo — the clean bar** (0.5h).
4. **Ledger part 2 + recognition-catalog trigger-line skim** (1.5h).
5. Motivation answers aloud + the prepared question (0.5h).
6. Networking + Git re-drill, quick-fire (1h). Sheet full re-read (0.5h).

## Aug 19 (Wed) — 7h, front-loaded, hard evening stop

1. **Morning OOD gauntlet** — two unseen-order questions back-to-back, interviewer-mode, full protocol (2h).
2. **Full verbal sweep** — OS + arch + networking + Git, one pass, ≤30s answers (1.5h).
3. **MoldCast + resume full talk-through**, timed, out loud: every bullet, both numbers, DSO no-concurrency rule, the gap-storage↔interval-set story (1h).
4. **Ledger + Appendix B, final reads** (1.5h).
5. Logistics: invite time in SGT, camera/mic test, quiet room, water, phone (0.5h).
6. One shaky-reflex slot, your pick (0.5h). **Hard stop. Sleep.**

## Aug 20 (Thu) — INTERVIEW

Morning only: cache numbers, fork returns, both benchmark numbers, protocol steps 1–2. In the room: clarify first, narrate always, never invent a mechanism; if silent >60s, say what you're deciding between.

---

# PART 2 — APPENDICES

## Appendix A — The eleven OOD questions

### A.1 AVL tree / implement std::map — highest priority

_London R1 "implement maps"; IIT BHU rotations. Your stated freeze._

- **L0 interface (5'):** `template <typename K, typename V> class Map` — insert, find, erase, operator[], size, empty. Ask about ordered iteration.
- **L1 node + ownership (5'):** `struct Node { K key; V val; Node *left=nullptr,*right=nullptr; int height=1; };` Trade-off aloud: unique_ptr children (auto cleanup, awkward rotations, recursion-depth dtor risk) vs raw + recursive dtor (pragmatic live choice).
- **L2 insert + find (10'):** standard walk. **Two-child erase: swap with in-order successor (leftmost of right subtree), delete it from its old spot (≤1 child by construction).** Drill until boring.
- **L3 balance (5–10'):** "AVL, rebalance walking back up." `rotateLeft` from memory (3 pointer moves + 2 height updates). LL/RR/LR/RL at name level.
- **L4 talk track:** std::map is red-black not AVL (fewer rotations per write vs flatter reads) · node-based → iterators/refs survive others' modifications · erase(it) returns next · operator[] default-constructs · extract/node handles since C++17.
- **30-min bar:** interface + node + insert + find + one rotation, narrated.

### A.2 Order book without matching — _Montreal R1; London OP_

- API: addOrder(id, side, price, qty), cancel(id), modify(id, qty), bestBid/bestAsk, volumeAtLevel.
- `map<Price, list<Order>, greater<>>` bids / `less<>` asks → best = begin(), O(1).
- Graded sentence: `unordered_map<OrderId, pair<LevelIt, ListIt>>` → **O(1) cancel, legal because map/list iterators survive erasure of other elements.**
- list vs deque narrated (stable iterators vs cache-friendliness; cancel-by-id ⇒ list). Erase empty price levels. Follow-ups: modify + time priority; node pooling; **adding matching — practiced, not just talked** (Aug 12 session): while qty remains and the best opposite level crosses, fill FIFO against front orders, pop filled, erase emptied levels. A later-round account demanded a full matching engine in ~15 min and rejected on an erase-while-iterating slip — the erase-loop idiom is the graded moment.

### A.3 std::function wrapper — _confirmed Montreal R2_

- Variant a: wrapper class over `std::function<R(Args...)>` (count/timing/logging/retry); `template <typename R, typename... Args> class Wrapper<R(Args...)>` syntax cold.
- Variant b: mini std::function — CallableBase (virtual invoke/clone/dtor), `CallableImpl<F>`, unique_ptr.
- Grilling: unbounded size ⇒ heap; SBO (~2–3 words inline); type erasure ⇒ indirect dispatch; copyable ⇒ clone ⇒ move-only lambdas excluded ⇒ C++23 move_only_function; perfect-forward Args.

### A.4 Vector (blank redo, ledger-audited)

Growth `cap?cap*2:1` · `data_[--size_].~T()` · copy-assign: bytes = size_*sizeof(T), delete old, self-assignment stance · at() uses >= · move ctor initializes via std::exchange, never swaps; both moves noexcept (else realloc copies) · `new T[n]` vs ::operator new + placement new, narrated · bonus: move_if_noexcept, strong guarantee.

### A.5 Thread pool

`vector<jthread>` + deque<function<void()>> + mutex + cv + stopping flag · worker: wait(pred incl. flag) → pop → **unlock, then execute** · submit→future via packaged_task in shared_ptr (std::function needs copyable — the depth question) · shutdown: flag under lock, notify_all · ladder: coarse mutex → bounded condvar → moves → RAII → padding → work-stealing mention.

### A.6 SPSC queues

**(a) Locked:** two condvars; count in every mutation; consume-through-index-then-advance; notify is part of the mutation; size() locks. **(b) Lock-free:** relaxed own / acquire other / release publish; snapshot atomics once; state the full/empty convention; alignas(64) + cached opposite index. Talk: why SPSC needs no CAS; MPMC breaks (CAS/ABA); false sharing with numbers.

### A.7 Range Module / interval set — _London R2 sighting; your gap storage_

add: skip if `b < left || a > right` (touch merges) else min/max absorb + erase, insert once · remove: skip if `b <= left || a >= right` (touch ≠ overlap) else keep (a,left) if a<left, keep (right,b) if b>right, erase, reinsert after · say the `<` vs `<=` asymmetry aloud · half-open, no −1 arithmetic · query: member upper_bound({left, INT_MAX}), fail only on begin(), --it, second >= right · **clean bar:** both guards first draft, survives {[1,3),[5,10)}+add(0,20) and remove-outside-everything · deploy unprompted: MoldCast gap tracker, duplicate retransmit + missing guard = widened gap = NACK storm.

### A.8 LRU cache (LC 146)

list<pair<K,V>> + unordered_map<K, list::iterator> · get: **splice(order.begin(), order, it)** — moves the node, invalidates nothing · put: update+splice / push_front+insert / evict back (map erase FIRST) · same locator mechanism as order book cancel; both row-3 invalidation applications.

### A.9 Mini Linux filesystem — _IIT KGP_

Composite: Node base (name, raw parent ptr), File (content), Directory (`**std::map**<string, unique_ptr<Node>>` — ls must be sorted, that's the trade-off sentence) · path split '/', walk; mkdir -p = create-as-you-walk; ls on a file returns the file (clarify semantics in step 1) · parent owns children; back-pointer raw non-owning · virtual isDir() over dynamic_cast.

### A.10 Currency exchange as OOP — _IIT BHU_

CurrencyConverter::addRate (clarify implied 1/rate), bestRate → `optional<double>` + path; Graph{unordered_map<string, `vector<Edge>`>} · maximize product → w = −log(rate) → shortest path · **rate>1 ⇒ negative weight ⇒ Dijkstra invalid ⇒ Bellman-Ford** · flex: negative cycle = product>1 cycle = **arbitrage detection** · rates ≤1 or hop-bounded ⇒ Dijkstra/BFS variants, and why then valid.

### A.11 Trie as OOD (NEW — was an accepted risk, now covered)

_Why: LC 1268 (Search Suggestions) is a Squarepoint-attributed OA question in your own recall pool, trie-tagged._

- Node design — this is your map session's Layer 1 with a different struct: `struct Node { std::array<std::unique_ptr<Node>, 26> children; bool isWord = false; };` (unique_ptr array is fine here — no rotations to fight, unlike the map).
- Ops: insert (walk, create-as-you-go), search, startsWith — each ~6 lines. Autocomplete = walk to the prefix node, then DFS collecting words (bounded k with early exit).
- Talk track: O(L) independent of dictionary size · memory trade-off (26 pointers/node → map<char,Node*> or sorted vector for sparse alphabets) · when a trie loses to sort + binary search.
- **The escape hatch, know it cold:** LC 1268 also falls to sorting the products + `lower_bound` on the growing prefix — if a trie question appears and the trie wobbles, the sorted-vector answer is fully accepted there. Two roads to the same answer means you cannot be stuck.

### A.12 Micro-topics (added after the Aug-3 search sweep — spoken 30-second answers, not full sessions)

**Exception safety levels (a flagged gap — verbatim-ready):**

- **Nothrow/noexcept guarantee:** never throws — required of destructors, expected of moves and swap. A `noexcept` function that throws → `std::terminate`.
- **Strong guarantee:** commit-or-rollback — the operation either succeeds or leaves state unchanged. `vector::push_back` gives it (a throwing copy during realloc leaves the vector intact — and this is WHY `move_if_noexcept` exists). Copy-and-swap is the idiom that manufactures it — which is your vector redo's copy-assignment stance, name it as such.
- **Basic guarantee:** no leaks, invariants intact, state valid but unspecified.
- The HFT footnote: many trading codebases build `-fno-exceptions` (unwind tables, icache, determinism) — but the guarantee _vocabulary_ still applies to any mutating API, so know it regardless. **std::optional (a flagged gap):** value-or-empty with **inline storage** — aligned buffer + engaged flag, no heap allocation ever. `nullopt` ≠ default-constructed T. `value()` throws `bad_optional_access`; `*opt` on empty is UB (bounds-check-first reflex applies); `value_or(x)` for defaults. Use as a return type for "not found" instead of sentinels or out-params — your A.10 `bestRate` already does this, say so. `optional<T&>` doesn't exist (until C++26); use `T*` for an optional reference. **weak_ptr — the two use cases (a verbatim R1 sighting):** (1) breaking `shared_ptr` cycles (parent holds shared, child holds weak back-pointer); (2) non-owning observation of a maybe-dead object — caches, observer lists: `lock()` atomically yields owning-shared-or-null; checking `expired()` then dereferencing is a TOCTOU bug, always `lock()`. **Semaphore vs mutex (asked in two accounts):** mutex = mutual exclusion WITH ownership (the locker must unlock); semaphore = counter + signaling with NO ownership (any thread may release), counting vs binary; a binary semaphore is not a mutex (no ownership, no priority-inheritance possibilities). C++20: `std::counting_semaphore` / `binary_semaphore`. vs condvar: a condvar needs mutex+predicate and can wake spuriously; a semaphore carries its count. Thread-pool framing: a tasks-available counting semaphore is the alternative to cv+predicate — be able to sketch both. **R1-depth concurrency trivia (spoken, 30 seconds each — the verbal counterpart to the build-it sessions):** **volatile vs atomic** — volatile means "don't optimize away this access," for MMIO and signal handlers; it gives NO atomicity and NO ordering, so it is never the cross-thread answer (Java's volatile is different — don't conflate), while `std::atomic` gives both. **Spurious wakeup** — `cv.wait` can return without a notify, which is why the predicate overload is mandatory and a bare `wait` is a bug. **What "thread-safe" actually means** — no data races AND invariants hold across concurrent calls; individually-safe operations don't make a check-then-act sequence safe (same TOCTOU shape as `expired()`-then-dereference). **Mutex vs spinlock** — spinning burns CPU but skips the syscall and context switch; worth it only when the critical section is shorter than a park/wake round-trip, which is precisely the HFT framing. Drill these in the Aug 15 A.12 hour alongside semaphore-vs-mutex. **Design-patterns vocabulary (warm-up currency, one sentence each):** thread-safe singleton = Meyers singleton, static local + C++11 magic statics · observer = your market-data/logger callback shape · factory = construction behind an interface · RAII is itself the C++ pattern answer · **CRTP/static polymorphism = the HFT answer to "virtual is too slow"** — compile-time dispatch, no vtable indirection. No UML, ever. **Rate limiter + logger (narration only — common OOD asks, 15 lines each):** token bucket = capacity + refill-rate; `allow()` = refill-by-elapsed-time, then spend-or-reject; inject the clock for testability. Async logger = producer threads → MPSC/SPSC queue → single writer thread — which is _literally your MoldCast consumer pattern with a file at the end_; one sentence and it's answered. **SQL: consciously risk-accepted** — one low-signal mention across every account, Python-track flavored; SELECT/WHERE/GROUP BY/JOIN at read-level only.

### A.13 Hash map + heap (the two implementation gaps found in the Aug-3 audit)

**Hash map from scratch (45-min build, Aug 16):** the missing twin of A.1 — you implement the tree map three times but the hash map zero, and "implement unordered_map" is asked at least as often.

- Separate chaining: `std::vector<std::list<std::pair<K,V>>> buckets_;` + `std::hash<K>{}(k) % buckets_.size()`. insert = find-in-bucket else push; find = walk the bucket; erase = bucket list erase.
- Rehash: when `size_ / buckets_.size() > max_load`, allocate a bigger bucket vector (double, keep it simple live) and re-slot every node. **The payoff sentence, yours uniquely:** rehashing invalidates iterators but NOT references/pointers — the nodes survive, only the bucket array is rebuilt. That is row 4 of your own invalidation grid, explained from the inside; say it unprompted.
- Talk track: open addressing (linear probing, tombstones on erase) vs chaining — latency shops often prefer open addressing because one contiguous array beats pointer-chasing per bucket (connects to your cache script); load factor trade-offs; why keys need Hash + KeyEqual; per-bucket `list` vs `forward_list` (std uses forward_list-style nodes).
- 45-min done bar: buckets + insert + find + erase + a working rehash, narrated. **Binary heap (20-min build, same Aug 16 slot):** vector storage; push = append + sift-up (`(i-1)/2`); pop = swap front/back, pop_back, sift-down (children `2i+1`, `2i+2`, swap with the larger); heapify bottom-up = O(n) (say why: work is bounded by node heights, which sum to O(n)). This is the machine under every priority_queue you've used. **Narration tier (read in the Aug 17 A.12 slot, no build):** **std::deque internals, two sentences** — chunked fixed-size blocks + a small map of block pointers; end-insertion may grow the pointer map (iterators invalidated) but never moves the blocks (references survive) — which is row 2 of your invalidation grid explained from the inside, completing the set: you can now explain all four rows mechanistically (vector realloc, deque blocks, node containers, rehash). Fixed-size **memory pool** = pre-allocated block array + free list threaded through the blocks themselves; allocate = pop head, deallocate = push front — ~10 lines, and it IS the "node pooling" follow-up already named in A.2. **String with SSO** = implement-vector's known variant: small buffer inline in the object (union with the heap pointer), so short strings never allocate — one paragraph, know the shape. **Bloom filter** one-liner: k hash bits set in a bitset; may say yes wrongly, never says no wrongly. **LFU** one-liner: LRU's harder sibling — frequency buckets, each holding an LRU list; O(1) via the same iterator-locator trick a third time. Cuttable: the heap and the narration tier, reluctantly. The hash map build is not — it is the largest genuinely-common implementation question the plan otherwise skips.

## Appendix B — Mechanics sheet

Re-read before every OOD session until nothing surprises you.

**Iterators & searching:** set/map iterators bidirectional — no it+5; free lower/upper_bound on them O(n), **always member versions** · lower_bound(k)=first ≥k, upper_bound(k)=first >k; "last ≤k" = upper_bound then --it after begin() check · sentinel pair keys: last first<=L → upper_bound({L,INT_MAX}) step back; first first>=L → lower_bound({L,INT_MIN}) · erase loop: `it = c.erase(it);` else `++it;` · **copy before erase:** `auto [a,b] = *it;` not auto& · rbegin = last · std::sort needs random access.

**Invalidation grid:**

|Container|insert|erase|
|---|---|---|
|vector|realloc → ALL; else at/after point|at/after point|
|deque|ends: iterators die, refs live; middle: all|same shape|
|list / set / map|nothing|erased element only|
|unordered_map|rehash → iterators die, refs/ptrs survive|erased element only|

Order-book locator, LRU splice, filesystem children, trie children = row-3/ownership applications. Say so when you use them.

**Comparators & heaps:** SWO — strict <, comp(a,a)==false, equals false both ways; <= or equality shortcuts = UB in sort · priority_queue with lambda needs the ctor arg (capturing always; everything pre-C++20) · `greater<T>` = min-heap · pre-C++20 safe answer: named functor.

**Construction & insertion:** push_back({a,b}) = braced list; emplace_back(a,b) = ctor forwarding (the (count,value) trap) · map::operator[] default-constructs; insert won't overwrite, insert_or_assign will · bounds check FIRST in any short-circuit; never lean on s[size()]=='\0'.

**Standing reflexes:** (1) signed/unsigned — hoist int n = size() and USE it · (2) growth cap?cap*2:1 · (3) consume through the index, then advance · (4) move ctors initialize (std::exchange), never swap; moves noexcept · (5) **no dead couts in anything human-read** · (6) widen before multiplying, mod during, never 1<<big, no FP modular arithmetic · (7) ((x%m)+m)%m · (8) mental -Wall; if (a=b) has cost you once · (9) **conclusion + mechanism; unknown mechanism = say so, never invent.**

## Appendix C — OS & comp arch: prompts, facts, reading

### C.1 Paste-ready prompt — OS master list

> Generate a comprehensive interview-prep master list (question + model answer, ≤30-second verbal answers) for a C++ HFT-adjacent graduate role, covering these OS categories: (1) processes & lifecycle — fork/exec/wait, return values, zombies/orphans, fork-output prediction problems; (2) threads vs processes — shared vs private, creation and context-switch costs; (3) scheduling — preemption, priorities, Linux CFS at one-liner depth; (4) syscalls & privilege — user/kernel mode, traps, why syscalls are expensive; (5) virtual memory — multi-level page tables, TLB, page faults, demand paging, swap, why VM exists; (6) process memory layout — stack/heap/data/text, stack vs heap trade-offs; (7) heap internals — malloc, brk vs mmap, fragmentation; (8) synchronization — mutex, semaphore, condition variable, spinlock, when each; (9) deadlock — four conditions, prevention via lock ordering; (10) data races vs race conditions — UB vs logic error, atomics; (11) IPC — pipes (incl. fork+close pattern), named pipes, shared memory, sockets, signals; (12) file descriptors & I/O — open/read/write/close, dup2, everything-is-a-file; (13) I/O multiplexing — select vs poll vs epoll, level vs edge triggered, non-blocking I/O; (14) signals — SIGSEGV/SIGKIL.L/SIGTERM/SIGINT semantics; (15) linking & loading — static vs dynamic, what ldd shows.

### C.2 Paste-ready prompt — comp arch master list

> Generate a comprehensive interview-prep master list (question + model answer, ≤30-second verbal answers) for a C++ low-latency graduate role, covering these computer-architecture categories: (1) memory hierarchy & latency numbers (L1/L2/L3/RAM, cache line size); (2) cache organization — lines, sets, associativity, write-back vs write-through, eviction; (3) cache-friendly code — spatial/temporal locality, hardware prefetching, AoS vs SoA, why contiguous beats pointer-chasing; (4) coherence & false sharing — MESI conceptually, alignas(64) padding; (5) virtual memory hardware — TLB, misses, huge pages; (6) pipelining & hazards; (7) branch prediction — mispredict cost, why sorted data is faster, branchless techniques; (8) ILP — superscalar, out-of-order at one-liner depth; (9) SIMD awareness; (10) alignment — natural alignment, misaligned cost, alignof/alignas; (11) NUMA at one-liner depth; (12) how atomics map to hardware — cache-line locking, contended vs uncontended cost.

### C.3 Non-negotiable facts (regardless of any master list)

- **L1 ~4 cycles · L2 ~12 · L3 ~40 · RAM ~200 · cache line 64 bytes · branch mispredict ~15–20 cycles.**
- ★ fork() → child's PID to parent, **0 to child**, −1 on failure; getppid() for parent's PID.
- ★ Dangling pointer = pointer alive, memory dead. Leak = memory alive, pointer gone.
- Data race = unsynchronized conflicting non-atomic access, ≥1 write ⇒ UB; atomics never data-race but can logic-race (wrong answer, not UB).
- Context switch ~1–10µs, hidden cost cache/TLB pollution; uncontended mutex ≈ one atomic RMW (~20ns), contended = futex syscall + park (~1–10µs).
- Deadlock: 4 conditions; break with lock ordering / scoped_lock.
- The rehearsed 60s cache answer: numbers → contiguity beats pointer-chasing → hot/cold split, AoS vs SoA → false sharing → alignas(64), citing your own SPSC padding.
- Linux CFS keeps runnables in a **red-black tree** (deployable inside the map question).

### C.4 Reading list — complete and final

**Core (scheduled into the days):**

1. OSTEP ch. 4 — Processes (Aug 8, ~45').
2. OSTEP ch. 5 — Process API (Aug 9, ~1h). The fork chapter; retires the ★fork miss; do the output examples mentally.
3. OSTEP ch. 6 — Limited Direct Execution (Aug 10, ~45'). Syscalls, modes, context switch.
4. OSTEP ch. 13 + 18 — Address Spaces + Paging (Aug 11, ~1.5h).
5. OSTEP ch. 19 — TLBs (Aug 12, ~1h). Full read; favorite MCQ territory.
6. Igor Ostrovsky, "Gallery of Processor Cache Effects" (Aug 13, ~30'). Best short cache read in existence.
7. The canonical Stack Overflow answer to "Why is processing a sorted array faster than an unsorted array?" (Aug 13, ~15'). Branch prediction, permanently.
8. A "Latency Numbers Every Programmer Should Know" table (Aug 13, ~5'). Pin it up until Aug 20.

**Skim tier:** OSTEP ch. 15 (with 13 if time) · ch. 20 first half (Aug 14) · ch. 39 Files & Directories interlude (Aug 14, for fd/open/dup semantics).

**Optional tier — only these, only if genuinely ahead or as rest-time viewing:** 9. CppCon: Chandler Carruth, "Efficiency with Algorithms, Performance with Data Structures" (~1h video, 1.5× speed over a meal — it converts your cache knowledge into interview-grade spoken sentences, which is exactly the verbal-fluency gap). 10. CSAPP §6.4–6.5 (cache memories / cache-friendly code) — the long-form version of items 6–8; only if everything else is done. 11. OSTEP ch. 30 (Condition Variables) — ONLY if the Aug 10 thread-pool session shakes your condvar confidence.

**Deliberately skipped, with reasons:** OSTEP concurrency 26–29/31–32 (Williams + your memory-model drilling exceed them) · Drepper (excellent, too long for 16 days) · anything C++20-novelty (ranges/coroutines — not in any sighted account). This list is closed. Adding reading after Aug 13 is procrastination wearing a productivity costume.

### C.5 How the master lists get drilled (the OS/arch equivalent of the bank's grading loop)

The C++ bank has a loop — work the group, paste, get graded, misses to ledger. The master lists need the same or they're just documents you read once. The loop:

1. **Generate both lists Aug 6** from the C.1/C.2 prompts; save them next to this file.
2. **Each block below drills specific numbered categories.** Cover the model answer, say your answer aloud, uncover, compare. Scoring is the mechanism rule: conclusion AND mechanism, or an explicit "not sure of the mechanism."
3. **Every miss goes to the ledger** with topic / what you said / correct answer — same format as C++ misses, so the Aug 16 and Aug 18 ledger reads sweep them up automatically.
4. **Missed categories get re-drilled** in the mixed verbal drills (Aug 15, 17) and the Aug 19 sweep — those drills are where misses go to die, not fresh-material time.
5. **Paste any answer you're unsure of** and I'll grade it, same as a bank group.

**Full coverage map — every category has exactly one home block:**

| Block                         | Day            | OS categories     | Comp-arch categories |
| ----------------------------- | -------------- | ----------------- | -------------------- |
| OS 1                          | Aug 8          | 1, 2, 3           | —                    |
| OS 2                          | Aug 10         | 4, 5, 6, 7        | —                    |
| CA 1                          | Aug 11         | —                 | 1, 2, 3, 4           |
| OS 3                          | Aug 12         | 11, 12, 13, 14    | —                    |
| CA 2 + bank concurrency group | Aug 13         | —                 | 5, 6, 7, 8, 10, 12   |
| OS 4                          | Aug 14         | 8, 9, 10, 15      | —                    |
| Mixed drill                   | Aug 15         | —                 | 9, 11                |
| Mini-diagnostic               | Aug 12         | tests OS 1–2      | —                    |
| Mixed drills + sweep          | Aug 15, 17, 19 | all, misses first | all, misses first    |

OS 1–15 and CA 1–12: fully covered, no orphans.

## Appendix D — MoldCast + resume recap

### D.1 MoldCast — exact finish list (Aug 8–10, then FROZEN)

alignas(64) + cached-index SPSC hardening → RX→SPSC→consumer wiring (fixed slots own bytes, drop+counter on full, atomic+jthread shutdown; SKIP: MPMC, thread-pool wiring, multi-session, retransmit changes, buffer pool, zero-copy) → two benchmarks (localhost throughput; gap p50/p99 via skip-every-k + timestamps + Python percentiles; taskset both; **memorize both numbers**) → README + repo public. Done = clonable, readable, numbers visible.

### D.2 Resume recap — every talking point

- **MoldCast:** bullets true after Aug 10 · both numbers + methodology + localhost caveat · next steps: buffer pool/zero-copy, real-network measurement · signature story, unprompted: the gap tracker is an interval set — the Range-Module interview question — and the overlap guard is load-bearing because duplicate retransmits are routine on multicast; without it a duplicate widens a gap and triggers a NACK storm.
- **DSO/B4Mesh:** BFT in two sentences · your ~8k-line role · one hardest-bug story · **no concurrency claim, ever** — concurrency lives in MoldCast.
- **KPMG:** SAST/web/AWS pentest in two sentences · bridge: security work is why you think in UB, bounds, and untrusted input.
- **NUS:** Honours (Distinction), May 2027 — one sentence.
- **Motivation trio**, spoken once each before Aug 20: why HFT/quant tech (latency work is where the machine model IS the job) · why Squarepoint (one specific true thing, two sentences) · why C++ (the language where abstraction has a cost model).
- **One prepared question** (defaults: first-year project shape on the team, or the research-facing vs infrastructure split).
- **Hardest-bug story**, four sentences: symptom → wrong hypothesis → actual cause → what changed in how you work. Pick the SPSC memory-ordering work or a B4Mesh consensus bug. One story, rehearsed, not improvised.

## Appendix E — Networking + Git

**E.1** TCP vs UDP · market data = UDP multicast + seq numbers + NACK (your project: one sender many receivers, TCP can't multicast, HOL blocking) · SYN→SYN-ACK→ACK; FIN/ACK both ways · states in order: LISTEN, SYN_SENT, SYN_RCVD, ESTABLISHED, FIN_WAIT_1/2, CLOSE_WAIT, LAST_ACK, TIME_WAIT (2×MSL — late duplicates must die before 4-tuple reuse; why SO_REUSEADDR exists, and you've set it) · connection = 4-tuple; ports 16-bit · OSI 7 ↔ TCP/IP 4 in one breath · NAT one-liner · "type a URL" in 60s · multicast: class D, IGMP join. **E.2** fetch vs pull · merge vs rebase (+ never rebase shared branches) · reset --soft/--mixed/--hard vs revert · stash/pop · cherry-pick · HEAD, detached HEAD · checkout vs switch/restore · log --oneline, diff --staged · conflict flow · .gitignore.

## Appendix F — Protocol, checklist, session specs

**Opening protocol (every OOD question):** (1) clarify the API surface — operations, complexity expectations, ordering, threading, ownership; asking is graded · (2) public interface first · (3) representation trade-off, one sentence, aloud · (4) skeleton + spoken rule-of-five stance · (5) one operation at a time, narrating the invariant; never silent >1 min. **Self-review checklist:** clarified API? · interface before internals? · trade-off sentence said? · rule-of-five stance said? · invalidation/iterator guarantee named where used? · complexity per op stated? · dead couts? · any mechanism invented rather than flagged? **Debug sessions (×2):** Claude generates a ~60-line broken class seeded with 4–6 bugs drawn from your ledger categories; 25-min clock; score = found/seeded; session 2 must beat session 1. Complements (not replaces) the bank's code-reading group. **Mini-diagnostic (Aug 12):** 25 min, OS blocks 1–2 material, interview pace, aloud. Purpose: first calibration of the spoken format while there are still six days to fix what it exposes. **Mocks:** Mock 1 (Aug 16), format A: 5'/35' quick-fire/30' unseen OOD. Mock 2 (Aug 18), format B: 5'/15' trivia/35' narrated live coding/5' questions. Aug 19 gauntlet: two unseen OOD back-to-back. All three grade the mechanism rule explicitly. **Meta:** speaking-while-coding is the trained skill — silence reads as freezing even when it's thinking · CodePair runs no -Wall for you · R1 is also a fit screen; the stories and the prepared question are graded material.