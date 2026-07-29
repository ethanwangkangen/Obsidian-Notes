# Squarepoint R1 Prep — 25 Days / 120 Hours (~4.8h/day)

## Top-level split

- LeetCode — 30h (daily ~1.2h, never batched)
- C++ depth — 30h
- Implementations (from memory) — 14h
- MoldCast M6–M8 — 16h
- OS / POSIX / comp arch — 12h
- Networking — 6h
- Williams ch. 5–7 — 6h
- Git + Linux — 2h
- Mocks + verbal + resume stories — 4h

---

## MoldCast M6–M8 (16h) — current state: M1–M5 done (echo, epoll RAII, multicast, packed header serialization, sequence tracking + gap detection); lock-free SPSC done except padding

### M6 — Concurrency primitives (4h)

- **6.1 SPSC hardening (1.5h)**
  - `alignas(64)` on head, tail, buffer; head/tail on separate cache lines (producer writes tail, consumer writes head; same line = MESI ping-pong)
  - Cached-index optimization: producer caches consumer's head, reloads atomic only when cached value says full (mirror for consumer)
  - Power-of-two capacity, mask instead of modulo
  - Test: 2 threads, 10M sequential ints, assert exact FIFO order + count; run under `-fsanitize=thread` and ASan
- **6.2 Bounded mutex+condvar MPMC queue (1.5h)** — matches verified R3 question (moves, condvar, bounded, RAII, cache-friendliness)
  - Pass 1 (correct): mutex, two condvars (`not_full_`, `not_empty_`), predicate loops (spurious + stolen wakeups), bounded capacity, `close()` wakes all waiters, `pop` drains then returns `nullopt`
  - Pass 2 (optimize, narrating each change): `push(T&&)` overloads + `emplace`; move out of deque in `pop`; notify after unlocking (know the tradeoff); `notify_one` vs `notify_all` and when each is wrong; separate the condvars' hot data
- **6.3 RAII thread wrapper + stop (1h)**
  - Joining-thread wrapper or `std::jthread`; `atomic<bool> stop_` relaxed loads, or `stop_token`
  - No thread pool inside MoldCast; no lock-free MPMC (cut)

### M7 — Threaded receive pipeline + retransmit path (9h)

- **7.1 Architecture doc (0.5h)** — write before coding
  - RX thread: epoll loop → recv → parse MoldUDP64 header → push to SPSC
  - Session thread: pop → sequence check (M5 logic) → in-order delivery via callback; gap → buffer OOO + schedule re-request; retransmits arrive through same SPSC
  - Rationale: RX thread only drains the socket so kernel receive buffer never overflows (UDP overflow = silent drop, invisible gap source); all stateful work off the hot path
- **7.2 RX thread (1.5h)**
  - Move existing epoll recv loop into wrapper thread; non-blocking socket; parse header; push to SPSC
  - Queue-full policy: drop + atomic drop counter (documented). Drop is indistinguishable from network loss → gap recovery handles it; backpressure would stall the drain and cause invisible kernel-buffer loss
- **7.3 Session thread (2h)**
  - Interval-based gap store tracks missing ranges; arrived-but-undeliverable packets in `std::map<uint64_t, Message>`; on gap fill, walk map delivering contiguous run
  - Alternative to know: preallocated ring indexed by seq — O(1) but memory-bounded window
  - Delayed re-request: wait ~5–10ms or N subsequent packets before requesting (UDP reorders); immediate vs delayed = HOL-blocking vs bandwidth tradeoff
  - Delivery callback only with strictly in-order messages; document that it runs on the session thread
- **7.4 Retransmit server, sender side (2h)**
  - Ring buffer of last N sent messages keyed by sequence
  - UDP unicast request listener; request = `{session, start_seq, count}` (mirrors MoldUDP64 request format)
  - Replay if in ring; MoldUDP64-style out-of-range response if aged out
  - Single-threaded (own thread or folded into sender epoll loop)
- **7.5 Client re-request wiring (1h)**
  - Session thread sends unicast requests; retransmitted packets flow through the normal RX → SPSC → sequencing path, no special-casing
- **7.6 Clean shutdown + stats (2h)**
  - Order: stop flag → close/wake SPSC consumer → join session thread → join RX thread → RAII closes sockets/epoll
  - Zero leaks under ASan, zero TSan reports
  - Atomic relaxed counters (independent monotonic, no ordering dependency): packets received, gaps detected, retransmits requested/filled, queue-full drops, delivered; print on shutdown

### M8 — Test, benchmark, ship (3h)

- **8.1 Lossy-link test (1h)**
  - Drop injection: sender flag "drop every Nth send but advance sequence," or forwarding proxy with p = 1–5% random drop
  - Assert: delivered stream complete and in-order despite loss
- **8.2 Benchmark (1h)**
  - Localhost sender → receiver: throughput (msgs/sec at fixed size); latency via `steady_clock` timestamp in payload, p50/p99 at delivery
  - If p99 >> p50, be able to explain (scheduling jitter, gap-recovery stalls = visible HOL blocking)
- **8.3 README + publish (1h)**
  - ASCII architecture diagram; protocol summary + MoldUDP64 fidelity notes; design decisions (NACK-over-ACK, drop policy, delayed re-request, HOL blocking); build/test incl. sanitizer targets; benchmark table
  - Tidy commit history; make repo public

### Interview payoff map

- 6.2 → R3 thread-safe queue question verbatim
- 6.1 → false-sharing / cache trivia
- 7.2 → TCP-vs-UDP depth (which TCP features rebuilt, which deliberately omitted, why)
- 7.6 → concurrency-correctness questions
- 8.2 → "performance problem you investigated"

---

## C++ depth (30h)

### 1. Value categories & move semantics (3h)

- lvalue / xvalue / prvalue with one example each
- `std::move` is a cast; when a move silently becomes a copy (const members; no-noexcept move during vector growth)
- `std::forward`; reference collapsing rules written out; forwarding vs rvalue reference (`T&&` deduced vs non-deduced context)
- Rule of 5, rule of 0
- Const-member move behavior re-verified (logged misconception)
- `noexcept` on move ctors; `move_if_noexcept` in `vector::resize`

### 2. Object lifetime & ctor counting (3h)

- Drill format: given snippet, write exact ctor/copy/move/dtor output
  - Pass-by-value; return with/without RVO/NRVO (guaranteed elision C++17 vs optional)
  - `push_back` vs `emplace_back`; initializer lists always copy; temporaries bound to const refs
- Rvalue lifetime extension re-verified (logged inverted item)
- Dangling patterns: ref to local, `string_view` of temporary, lambda capturing reference past scope
- 10 written drills with traced outputs

### 3. STL internals & invalidation (4h) — worst logged area

- Grid: {vector, deque, list, map/set, unordered_map} × {insert, erase, growth/rehash} → {iterators dead?, references dead?}
  - vector growth kills everything; vector erase kills at-and-after; unordered rehash kills iterators but not references
- `std::map` as balanced BST: single left/right rotation pseudocode cold; where RB/AVL rebalance triggers
- unordered_map: buckets, load factor, rehash policy
- deque chunked block-map, why iterators are fat; `vector<bool>`; SSO

### 4. Smart pointers & RAII (2h)

- Control block contents (strong count, weak count, deleter, object-or-pointer)
- `make_shared` single allocation vs two; weak_ptr memory-pinning downside
- `weak_ptr::lock` race-safety; `enable_shared_from_this` and the double-control-block bug
- Deleters: type-erased in shared, part-of-type in unique; `unique_ptr<T[]>`; aliasing constructor
- RAII tied to EpollWrapper and thread wrapper

### 5. Virtual dispatch & OOP (2.5h)

- vtable/vptr mechanics; one vptr in single inheritance; vtable layout with overrides
- Virtual destructor (delete through base pointer without = UB); pure virtual / abstract
- Virtual calls in ctors dispatch statically — why
- Object slicing; `final` / `override`; devirtualization
- Diamond inheritance + virtual bases at explain-level (problem, cost: extra vptr, virtual base offset)
- Operator overloading conventions: member vs free, `operator<<` must be free, canonical `operator=`
- Four OOP pillars, each with a ready 5-line code example; 10-minute clean-hierarchy drill

### 6. Casting & C++14/17 (2h)

- static / dynamic / reinterpret / const_cast: one example + one failure mode each
- `dynamic_cast`: pointer → null, reference → throw
- `reinterpret_cast` + strict aliasing = UB; C-style cast resolution order
- C++17 vs 14: structured bindings, if/switch-init, guaranteed elision, `optional`/`variant`/`any`/`string_view`, fold expressions, `if constexpr`, CTAD, inline variables

### 7. Memory model & atomics (2h)

- Acquire/release pairing explained line-by-line on own SPSC code
- Relaxed: where safe (counters, own-index loads); seq_cst default + cost
- CAS weak vs strong; false sharing with own alignas fix as example
- Cache latencies: L1 ~4 / L2 ~12 / L3 ~40 / RAM ~200 cycles — physical sticky note, recite daily

### 8. Trap bank + initialization & linkage (1.5h)

- `[=]` captures `this` (C++20 deprecation of implicit this-capture)
- Signed/unsigned comparison — 5 variants (recurring item); integer promotion/overflow UB
- Static init order fiasco + Meyers singleton
- Six initialization forms (default/value/direct/copy/list/aggregate); `int x;` vs `int x{};`; `vector<int> v(3,5)` vs `{3,5}`; most vexing parse
- `const` vs `constexpr` vs `consteval`; top-level vs low-level const
- Internal vs external linkage (`static`, anonymous namespaces, `inline` variables); ODR in one sentence
- `string_view` dangling pitfalls

### 9. Lambdas & std::function (3h)

- Desugaring: lambda = anonymous class with captured members + `operator()`; drill: write the equivalent class by hand for a given lambda
- Captures: `[=]` / `[&]` / explicit; by-value captures const in `operator()` unless `mutable` (mutable changes the member copy, per lambda object, not the original)
- Init-capture `[x = std::move(v)]` (C++14) = how to move into a lambda
- `this` vs `[*this]` (C++17); dangling from captured refs/`this` past scope
- Generic lambdas = templated `operator()`; captureless lambda → function pointer (unary `+` trick); why capturing lambdas can't convert
- `std::function`: type erasure over any callable of the signature
  - Costs: possible heap allocation (SBO when callable fits), indirect call blocks inlining, requires copyable callable
  - vs templates (zero-cost, infects signatures) vs function pointers (no state)
  - "When not to use it": hot paths — allocation + indirect call

### 10. Exceptions (2.5h)

- Stack unwinding: throw → catch; dtors of all fully-constructed locals run — why RAII composes with exceptions
- Exception escaping a destructor during unwind → `std::terminate`; dtors implicitly noexcept since C++11
- Exception in ctor: object never existed, its dtor doesn't run; dtors of already-constructed members/bases do run → argument for RAII members; `new T` with throwing ctor frees the allocation automatically
- Guarantees: nothrow / strong / basic; `vector::push_back` as worked strong-guarantee example
- `throw;` vs `throw e;` (rethrow keeps dynamic type; latter slices); catch by const ref
- `noexcept` as contract; `move_if_noexcept` interaction
- Cost model: table-based, ~free happy path, expensive throw → exceptions for exceptional paths

### 11. Memory allocation, placement new, alignas (3h)

- `new T` = operator new then ctor; `delete p` = dtor then operator delete; placement new `new (ptr) T(...)` = ctor only on provided memory; inverse = explicit `p->~T()`
- Why vector needs this: capacity > size means raw objectless memory; `new T[cap]` would default-construct and require default-constructibility
- `operator new` overloading (class + global); `new[]`/`delete[]` mismatch UB + size-cookie reason; `malloc` vs `new`
- Alignment: `alignof`, natural alignment, struct padding + member reordering (drill: predict `sizeof` for 3 structs)
- `alignas`: cache-line padding (performance, SPSC) and aligned raw buffers for placement new (correctness — misaligned construction = UB); `std::aligned_alloc`; `std::launder` name-recognition only
- Arena/pool allocators conceptually; revision of own first-fit allocator (free list, headers, split, coalesce, alignment)
- Stack vs heap: stack = pointer bump; malloc internals at 2-min depth (free lists/bins, sbrk vs mmap for large blocks)

### 12. Templates & compile-time (1.5h)

- Deduction basics: when `T&&` is forwarding; array/function decay
- Full vs partial specialization; CRTP recap on own past example
- `if constexpr` vs SFINAE (use if constexpr; sketch one `enable_if`)
- Variadic packs + fold expressions
- Instantiation, why templates live in headers, link to ODR

---

## Implementations track (14h) — blank file, no references, timed, then diff vs reference; every gap → ledger

- Target 25–45 min per item; compile + minimally test

- **I1. unique_ptr — 1h (D6)**
  - `template<class T, class D = default_delete<T>>`; deleted copy; move ctor/assign via `exchange`; `release`/`reset`/`get`/`*`/`->`/`bool`; deleter in dtor + reset
  - Probes: deleter as template param here vs type-erased in shared_ptr (zero overhead, EBO); why nothing atomic
- **I2. Spinlock — 0.5h (D8)**
  - `atomic_flag`: `test_and_set(acquire)` loop, `clear(release)`
  - TTAS improvement (spin on plain load first) + why (avoid hammering line in exclusive state)
- **I3. vector — 2.5h, two passes**
  - Pass 1 (D10, 1.5h): raw `operator new` storage; size/capacity; growth factor (2× vs 1.5× amortization argument); placement-new in `push_back`; explicit dtor calls in `pop_back`/`clear`/dtor; `operator[]`; reallocation loop
  - Pass 2 (D12, 1h): exception safety — build new buffer fully before touching old (strong guarantee); `move_if_noexcept` relocation; `emplace_back` w/ perfect forwarding; `push_back(v[0])` self-reference edge case
  - Probes: invalidation grid recited against own code; why not `realloc`; `reserve` guarantees
- **I4. shared_ptr + weak_ptr — 2.5h (D13–14)**
  - Control block struct: strong count, weak count, type-erased deleter (virtual `destroy()` on block); atomic counts
  - Rules: last strong ref destroys object; last ref of either kind destroys block
  - `weak_ptr::lock` = CAS increment-if-nonzero (attempt, then study)
  - `make_shared` combined allocation in comments + weak_ptr pinning tradeoff
  - Probe: ordering on decrement — `fetch_sub(acq_rel)`, thread hitting zero must see all prior writes before dtor
- **I5. std::function — 1.5h (D17)**
  - Abstract `callable_base` (virtual `invoke`/`clone`/dtor); templated `callable<F>` holder; heap-allocating ctor template; copy via clone; `operator()`
  - SBO explained verbally: aligned inline buffer + placement new
- **I6. Thread pool — 2h (D19)**
  - N jthreads (or wrapped); task queue of `std::function<void()>`; mutex + condvar
  - `submit` returning `std::future` via `packaged_task` (warm-up: fire-and-forget void version)
  - Shutdown: stop flag → notify_all → workers drain then exit → join in dtor
  - Probes: notify_all on shutdown vs notify_one on submit; task submitting a task / join-inside-worker deadlock hazard
- **I7. First-fit allocator revision — 1.5h (D20)**
  - Re-derive: block header layout, free-list walk, split on alloc, coalesce on free (boundary tags or sorted list), alignment rounding
  - Probes: internal vs external fragmentation; first-fit vs best-fit; thread safety (per-thread arenas > one big lock)
- **I8. SPSC + MPMC blank-file re-dos — 1.5h (D21)**
  - SPSC with all orderings + padding + cached indices ≤ 30 min
  - Bounded MPMC with condvars + close semantics ≤ 25 min
  - Any gap → taper-week drill list

- Excluded: string/SSO, lock-free MPMC, full RB-tree insert (rotation pseudocode is the verified bar)

---

## LeetCode (30h, ~1.2h daily)

- **DP + optimize-follow-up — 10h**
  - Two-pass drills: correct DP, then compress (2D→1D; monotonic-queue transition optimization; prefix-max/min folding — LC 1937 as canonical dimension-reduction drill)
  - Knapsack variants; digit DP once for exposure; revisit LC 2401 (flagged fail)
- **Cyclic/wraparound — 4h**
  - LC 1610 (torch recall) cold, re-derive wraparound (duplicate angles +360, sliding window)
  - LC 918, LC 213, LC 2134; findMinimumShifts re-solved from scratch checking all three logged bugs
- **Strings — 4h**
  - Sliding window w/ char counts; KMP concept-level; palindrome DP; parsing-heavy implementation strings
- **Two-pointers + stdin implementation — 4h**
  - Overlapping-ranges problems (stock-price OA recall)
  - One full Employee-Salary-v3-style drill: messy stdin → parse → aggregate → output, written with proper structs/functions (style-graded)
- **Graph + OOP-framed design — 4h**
  - Currency-exchange/Dijkstra rewritten with classes (Graph, Edge, interfaces)
  - One BFS-grid, one topo sort, one union-find; present algorithms as designed code
- **Weak-area rotation — 2h**
  - 5 greedy-vs-DP recognition problems with justification
- **Timed sims — 2h**
  - Two 35-min single-problem sessions, talking aloud, R1 format
- Deferred: tries, Fenwick, segment trees (no verified Squarepoint signal)
- Standing rule: paradigm never revealed up front

---

## OS / POSIX / comp arch (12h)

- **fork() drills — 3h**
  - Process count from N forks in a loop; fork + printf buffering (why output duplicates without newline/fflush); fork+exec; wait/waitpid; zombies vs orphans
  - Copied vs shared: fds shared → interleaved writes; COW memory; fork in a threaded program
  - 8 output-prediction drills with traced answers
- **VM & page tables — 3h**
  - Drawable translation walk: VPN split, multi-level walk, TLB hit/miss, page fault path
  - Why multi-level (sparsity); TLB shootdowns conceptually; huge pages; COW page-fault mechanism
- **Semaphores & races — 2h**
  - Semaphore vs mutex vs condvar (when each is wrong); producer-consumer with semaphores written cold
  - Deadlock: four conditions + breaking each; race-spotting drill on given code
- **Pipes, fds, mini-FS — 2h**
  - fd table → open-file table → inode table; who shares what after fork vs dup2
  - pipe + fork pattern written cold
  - Mini filesystem sketch in C++ classes (inode, dentry, directory-as-file) — 20 min so it's not novel live
- **Comp arch — 2h**
  - Cache lines, associativity, row-major loop-order example
  - Branch prediction, sorted-array-faster phenomenon; pipeline hazards at 2-min depth; TLB as another cache

---

## Networking (6h)

- TCP state machine drawn from memory until fluent: all 11 states, both close sequences, TIME_WAIT + why 2MSL
- Three-way handshake with sequence numbers
- TCP vs UDP as MoldCast-anchored answer: retransmission/ordering rebuilt, flow/congestion control deliberately omitted, multicast forces UDP
- OSI layers + common ports (22, 53, 80, 443; DNS-over-UDP nuance)
- HOL blocking, cited from own delivery path

---

## Git + Linux (2h)

- Git: rebase vs merge; reset soft/mixed/hard; cherry-pick; stash; reflog; commit/tree/blob storage; detached HEAD
- Linux: grep/find/awk-lite; ps/top/kill + signals (9 vs 15); chmod bits; hard vs symlinks; /proc; pipes/redirection incl. `2>&1`

---

## Day-by-day (LC ~1.2h every day, not repeated below)

### Phase 1 — MoldCast sprint (D1–D6)

- D1: Mold 6.1 + 6.2 (3h)
- D2: Mold 6.3 + 7.1 + 7.2 (3h) · Williams ch. 5 begin (0.5h)
- D3: Mold 7.3 + start 7.4 (3h) · Williams (0.5h)
- D4: Mold finish 7.4 + 7.5 (3h) · Williams (0.5h)
- D5: Mold 7.6 + 8.1 lossy test (2.5h) · Williams ch. 6 (1h)
- D6: Mold 8.2 benchmark + 8.3 README + repo public (1.5h) · I1 unique_ptr (1h) · Williams ch. 6 finish (1h)

### Phase 2 — core grind (D7–D19)

- D7: cat. 7 atomics narrating own SPSC (2h) · Williams ch. 7 skim (1.5h)
- D8: cat. 1 value/move (3h) · I2 spinlock (0.5h)
- D9: cat. 2 lifetime + ctor-counting drills (3h) · OS fork pt. 1 (1h)
- D10: cat. 3 pt. 1 invalidation grid (2.5h) · I3 vector pass 1 (1.5h)
- D11: cat. 11 allocation/placement new/alignas (3h) · cat. 3 pt. 2 map rotations cold (1h)
- D12: cat. 10 exceptions (2.5h) · I3 vector pass 2 exception-safe realloc (1h)
- D13: cat. 4 smart pointers (2h) · I4 shared_ptr pt. 1 (1.5h)
- D14: I4 shared_ptr pt. 2 + weak_ptr::lock (1h) · OS VM & page tables (2.5h)
- D15: cat. 5 virtual/OOP + diamond + operators (2.5h) · OS fork pt. 2 (1h)
- D16: cat. 9 lambdas & std::function (3h) · Git pt. 1 (0.5h)
- D17: I5 std::function build (1.5h) · cat. 6 casts + 14/17 (2h)
- D18: OS semaphores/races + pipes/fds/mini-FS (3.5h)
- D19: I6 thread pool (2h) · cat. 12 templates (1.5h)

### Phase 3 — breadth + verbalization (D20–D22)

- D20: Networking TCP state machine drawable cold (2.5h) · I7 allocator revision (1.5h)
- D21: Networking TCP-vs-UDP aloud + OSI/ports/HOL (2h) · I8 SPSC + MPMC blank-file re-dos (1.5h)
- D22: cat. 8 trap bank + init/linkage (1.5h) · comp arch (2h) · Git/Linux quick-fire finish (1h)

### Phase 4 — taper (D23–D25, no new material)

- D23: full ledger review, every file, re-tested (2h) · resume stories 2 min each aloud (1h) · LC timed sim #1
- D24: mock R1 #1 exact format — 5' intro / 15' trivia / 35' LC / 5' questions (1h) · patch gaps (1.5h) · sticky notes: cache numbers, invalidation grid, TCP states (0.5h)
- D25: mock R1 #2 or trivia-only round (1h) · MoldCast pitch aloud, README reread, flashcards · stop by evening; LC light or none

### Cut order (pre-decided, in order)

1. cat. 12 templates
2. I7 allocator revision
3. comp arch → 1h
- Never cut: daily LC, D1–D6 MoldCast sprint, I8, taper mocks
