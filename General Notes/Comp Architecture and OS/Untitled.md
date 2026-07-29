# Comp Arch / OS — Full OA Study Reference

---

# PART A: CPU CACHES

## A1. Memory hierarchy & the numbers

|Level|Typical size|Latency|Shared?|
|---|---|---|---|
|Registers|~16 GP + vector|0 cy|per HW thread|
|L1i / L1d|32 KiB each|~4 cy|per core|
|L2|256 KiB – 1 MiB|~12 cy|per core (unified i+d)|
|L3 (LLC)|8 – 64 MiB|~40 cy|shared across cores (unified)|
|DRAM|GiBs|~60–100 ns (~200–300 cy)|system|
|NVMe SSD|—|~10–100 µs||
|HDD|—|~5–10 ms||

Derived numbers worth knowing cold:

- L1 → DRAM gap ≈ 50–100×. One DRAM miss ≈ hundreds of arithmetic instructions wasted.
- Syscall: ~100 ns–1 µs. Context switch (direct): ~1–10 µs. Major page fault: ~ms (≈10⁷ cycles).
- Memory bandwidth is finite too: streaming misses can saturate it; latency ≠ bandwidth.

**Why hierarchies work — locality:**

- _Temporal locality:_ what you touched recently, you'll touch again (loop variables, hot functions).
- _Spatial locality:_ if you touched address X, you'll soon touch X±small (arrays, struct fields, straight-line code).
- Caches bet on both. Code with neither (random pointer chasing over a huge graph) runs at DRAM speed no matter how big the cache is.

## A2. Cache line

- **64 bytes** on essentially all modern x86/ARM server chips. Aligned: lines start at addresses divisible by 64 (low 6 bits = 0).
- The line is the atom for three distinct things:
    1. **Transfer** — a miss fetches the whole line, never a byte.
    2. **Coherence** — MESI state is tracked per line.
    3. **False sharing** — contention granularity is the line.
- Consequences:
    - Touching one `int` costs the same as touching 16 (`64/4`) if they're in one line → pack hot data.
    - A struct straddling two lines needs 2 fetches → `alignas(64)` hot structs; order members so hot fields are adjacent and at the front.
    - Sequential access amortizes: first element misses, next 15 hit.

## A3. Cache organization & associativity

**Structure:** cache = array of **sets**; each set holds **N ways** (slots). A line's address determines its set exactly; it may occupy any way within that set.

**Address decomposition** (64 B lines):

```
| tag (rest) | set index (log2 #sets bits) | offset (6 bits) |
```

- Offset: byte within line.
- Index: selects the set.
- Tag: stored with each cached line; lookup = compare candidate tags in the indexed set, in parallel. Match → hit. No match → miss → evict one way (pseudo-LRU) and fill.

**Geometry formula (drill until instant):**

```
#lines = cache_size / line_size
#sets  = #lines / ways
index bits = log2(#sets);  offset bits = log2(line_size);  tag = addr_bits − index − offset
```

Worked examples:

- 32 KiB, 8-way, 64 B: 512 lines → **64 sets**, 6 index bits, 6 offset bits.
- 256 KiB, 16-way (commonly quizzed): 4096 lines → 256 sets → 8 index bits.
- 48-bit physical addresses, 64 sets: tag = 48 − 6 − 6 = 36 bits.

**Associativity spectrum:**

|Kind|Mapping|Pros|Cons|
|---|---|---|---|
|Direct-mapped (1-way)|addr → exactly 1 slot|fastest, simplest, lowest power|conflict misses: 2 hot lines in one slot thrash forever|
|N-way set-assoc|addr → 1 set, N choices|good tradeoff; N=8 captures most benefit|N-way tag compare|
|Fully associative|anywhere|zero conflict misses|compare ALL tags in parallel → only for tiny structures (TLBs, victim buffers, store buffers)|

**Same-set collisions:** two addresses collide iff they share the index bits, i.e., they differ by a multiple of `#sets × line_size`. For a 64-set L1 that's every **4 KiB**. Pathology: walking a matrix column when row length is a power of 2 (say 4 KiB) → every element maps to the same set → you effectively have an 8-line cache. Symptom: performance cliff at power-of-2 sizes. Fix: pad rows by one line / use non-power-of-2 leading dimension.

**Eviction policy:** true LRU is expensive beyond ~4 ways → **pseudo-LRU** (tree-bit approximation). Assume "approximately least-recently-used gets evicted."

**Write policies:**

- **Write-back** (modern default): stores update only the cache; the line is marked _dirty_; memory updated on eviction. Saves bandwidth.
- **Write-through:** every store also goes to the next level. Simpler, more traffic (used in some L1s historically, niche now).
- **Write-allocate** (pairs with write-back): a store miss first fetches the line, then modifies it. So even pure writes pull lines in.
- Non-temporal stores (`movnt`) bypass the cache for huge streaming writes (avoid polluting cache with data you won't reread).

**Inclusive vs exclusive (occasionally quizzed):** inclusive L3 contains a copy of everything in L1/L2 (eases coherence snooping — evicting from L3 forces eviction from inner levels); exclusive maximizes total capacity. Intel historically inclusive L3, AMD mostly exclusive/victim L3.

**L1i vs L1d (split L1, "Harvard"):**

- Separate so the front-end fetches instructions in the same cycle the back-end accesses data (independent ports), and each is tuned for its access pattern.
- L1i is effectively **read-only**: no stores target it → no dirty bits, no write logic.
- JIT / self-modifying code: instructions are written _as data_ through L1d. **x86 hardware keeps L1i coherent automatically** (snoops detect the overlap and flush stale L1i/pipeline). **ARM does not** — you must explicitly clean D-cache + invalidate I-cache (`__builtin___clear_cache`).
- L2/L3 are **unified** (hold both code and data).

## A4. Cache miss taxonomy — the 3 C's (+ coherence)

|Miss|Definition|Litmus test|Mitigation|
|---|---|---|---|
|**Compulsory (cold)**|first-ever access to the line|misses even with an _infinite_ cache|prefetch (HW/SW), larger lines, batch layout|
|**Capacity**|working set exceeds cache size; lines evicted for space and re-fetched|misses even in a _fully-associative_ cache of same size|shrink working set: blocking/tiling, streaming algorithms, SoA layout|
|**Conflict**|target set is full although other sets have room|disappears when fully associative|more ways, padding to break power-of-2 strides|
|**Coherence**|line invalidated by another core's write|multicore only|reduce sharing; eliminate false sharing|

Classification algorithm (apply in order): Would it miss in an infinite cache? yes → compulsory. Else, would it miss in a fully-associative cache of the same total size? yes → capacity. Else → conflict.

Canonical MCQ scenarios:

- **Stream a 1 GiB array once, never revisit → compulsory** dominated. (Not capacity: capacity implies _re_-fetching evicted data you revisit.) More associativity: useless. Prefetcher: hides most of it.
- Loop repeatedly over an array 2× the cache size → **capacity** (every pass re-misses everything under LRU).
- Column-walk with power-of-2 row stride → **conflict**.
- Two threads ping-ponging a shared counter → **coherence**.

## A5. MESI coherence protocol

Problem: each core has private L1/L2 caches holding copies of the same memory. Coherence keeps them consistent. Tracked **per cache line**, enforced by snooping or a directory.

**States** (for a given line in a given core's cache):

- **M — Modified:** sole copy in any cache; **dirty** (≠ memory). This core may read/write freely. On another core's request, must supply the data (and write back / hand off ownership).
- **E — Exclusive:** sole copy; **clean** (== memory). May read freely; a write upgrades **silently to M with zero bus traffic** — this is E's entire reason to exist (private data pays no coherence tax).
- **S — Shared:** clean; other caches may also hold it. **Read-only.** Writing requires an upgrade.
- **I — Invalid:** unusable; any access is a miss.

**Transitions to know:**

|Event|Result|
|---|---|
|Read miss, no other cache holds it|fetch from memory → **E**|
|Read miss, some cache holds it (M/E/S)|holder supplies/downgrades → both/all **S** (M holder writes back or transfers dirty data)|
|Write while **S**|broadcast **RFO** (read-for-ownership / invalidate); all other copies → **I**; writer → **M**|
|Write while **E**|silent → **M**|
|Write miss (line in I/absent)|RFO fetches line with exclusive ownership → **M**|
|Snooped read while you're **M**|supply data; → **S**|
|Snooped RFO while you're M/E/S|→ **I** (M supplies dirty data first)|

Extensions (name-recognition level): **MOESI** (AMD; O = Owned, dirty-but-shared, avoids write-back on M→shared), **MESIF** (Intel; F = Forward, designates which sharer responds).

**False sharing — the flagship exam question:**

- Setup: `struct { int64_t a; int64_t b; }`, thread 1 hammers `a` on core 1, thread 2 hammers `b` on core 2. No data race, no shared variable.
- Mechanism: both variables occupy **one line**. Each write needs M state → RFO invalidates the other core's copy → its next write re-RFOs it back. The line **ping-pongs** across the interconnect; each transfer costs tens of cycles. Throughput collapses ~10×.
- Detection: `perf c2c`, HITM events.
- Fix: separate the variables by ≥64 B — `alignas(64)` each, or pad with `char pad[64]`, or `alignas(std::hardware_destructive_interference_size)`.
- Distractor answers to reject: TLB flushing (translation is untouched), capacity misses (working set is 16 bytes), page faults.
- _True sharing_ (both cores write the SAME variable) has the same ping-pong cost — the fix there is algorithmic (per-thread counters, combine at the end).

## A6. Store buffer (bridge to memory ordering)

- Each core has a small FIFO **store buffer**: retired stores wait there before the cache accepts them (e.g., while the RFO completes). The core's own loads check ("store-to-load forwarding") so it always sees its _own_ stores.
- Other cores do NOT see buffered stores → the store appears delayed to the outside world → this is the hardware cause of **store→load reordering** on x86 (see Part D).

---

# PART B: VIRTUAL MEMORY

## B1. Purpose & mechanics

Virtual memory gives every process the illusion of a private, contiguous address space. Wins:

1. **Isolation/protection** — process A cannot name B's memory; kernel pages protected by supervisor bit.
2. **Lazy allocation & overcommit** — `malloc` reserves address space; physical frames arrive on first touch.
3. **Sharing** — one physical copy of libc mapped into everyone; shared memory segments; fork's COW.
4. **Per-page permissions** — r/w/x, enabling W^X, guard pages, read-only .text.
5. **Swap/paging to disk** — address space can exceed RAM.

- **Page = 4 KiB** (default). Virtual address = virtual page number (VPN) + 12-bit offset. Physical = physical frame number (PFN) + same offset. Translation maps VPN→PFN; the offset passes through untouched.
- **PTE contents** (know the bits): PFN + **present**, **writable**, **user/supervisor**, **execute-disable (NX)**, **accessed** (HW sets on any use — feeds LRU approximations), **dirty** (HW sets on write — decides if eviction must write to swap).

## B2. Multi-level page tables

**The problem with a flat table:** hardware indexes it as one array — entry #N must exist at slot N whether or not page N is mapped. 48-bit space / 4 KiB pages = 2³⁶ entries × 8 B = **512 GiB per process**. Also would need to be physically contiguous. Absurd.

**The tree:** x86-64 splits the 48-bit VA:

```
| 9: PML4 idx | 9: PDPT idx | 9: PD idx | 9: PT idx | 12: offset |
```

Each level is one 4 KiB page holding 512 × 8 B entries. An entry at levels 1–3 points to the _physical address of the next table_; a leaf PTE points to the data frame.

**Walk:** CR3 (per-process register holding PML4's physical address) → index by top 9 bits → next table → … → PTE → PFN‖offset. **Four sequential dependent memory accesses** — each read's result is the next read's address, so nothing parallelizes.

**Where the savings actually come from — precise statement:**

- Mapped pages still cost exactly one leaf PTE each. No compression, no magic.
- A **null entry at level k prunes the entire subtree**: the tables below it are _never allocated_. One null PML4 entry stands in for 512 GiB of address space at the cost of 8 bytes.
- Real processes are tiny islands (text+heap low, stack high, libs in the middle) in an empty 256 TiB ocean → nearly the whole tree is pruned → total table size is typically a few hundred KiB.
- **Degenerate case:** map the entire address space → all leaves must exist → multi-level is slightly _worse_ than flat (adds ~1/512 + 1/512² … ≈ 0.2% for the directories).
- Secondary win: every table is an ordinary 4 KiB page, allocated independently — no giant contiguous allocation.

**Worked toy example** (32-bit VA, 10+10+12, 4 B entries): process maps 4 MiB at the bottom (code+heap) and 4 MiB at the top (stack).

- Flat: 2²⁰ entries × 4 B = 4 MiB, always.
- Two-level: 1 directory (4 KiB) + 1 leaf table for the bottom 4 MiB (4 KiB) + 1 leaf table for the top (4 KiB) = **12 KiB**; the other 1022 directory entries are null.

**Levels are growing:** 5-level paging (57-bit VA) exists for huge-memory machines — same idea, one more indirection.

## B3. TLB

- The Translation Lookaside Buffer caches **completed translations**: VPN → PFN + permissions. It does _not_ cache PTEs; it caches the _end result_ of the whole walk, so a hit costs ~1 cycle and touches the page tables **zero** times.
- Why not "just cache the PTEs in L1d"? The walk would still be 4 sequential lookups per memory access — even at 4 cycles each that's ~16 extra cycles on _every load and store_. The TLB collapses it to one associative lookup done in parallel with the L1 access.
- Sizes: L1 dTLB ~64 entries, iTLB similar, unified L2 TLB ~1.5–2K entries. High associativity (often fully assoc for L1 TLB).
- **TLB miss:** on x86 a **hardware page walker** does the 4-level walk transparently (no OS involvement). The walker's reads are normal cached reads — PTEs frequently hit in L1d/L2, and dedicated _paging-structure caches_ skip upper levels. Cost: ~tens of cycles typically.
- **TLB miss ≠ page fault.** Miss: translation not cached → walk it → if a valid PTE is found, refill TLB, done (pure HW). **Fault:** the walk finds present=0 or a permission violation → CPU raises an exception → OS handler runs.
- **Context switches:** page tables are per-process → loading a new CR3 invalidates the old translations. Either (a) flush the whole TLB (old, expensive: subsequent code re-misses everything), or (b) tag each entry with an **ASID/PCID** so entries from multiple processes coexist and only matching ones are used — the modern default. Switching _threads of the same process_ keeps CR3 → TLB untouched → one reason thread switches are cheaper.
- **Huge pages (2 MiB / 1 GiB):** the PD (or PDPT) entry directly maps a big page → walk ends a level early, and one TLB entry now covers 512× (or 262,144×) the memory → massive drop in TLB misses for big heaps. Costs: internal fragmentation, harder to swap. `madvise(MADV_HUGEPAGE)` / explicit hugetlbfs. Standard answer to "how do you reduce TLB pressure."
- **TLB shootdown** (advanced but appears): when the OS changes a mapping (munmap, permission change), other cores may hold the stale entry → kernel sends IPIs (inter-processor interrupts) forcing each core to invalidate → expensive, scales badly with core count. Why frequent mmap/munmap churn is costly in multithreaded processes.

## B4. Page faults

CPU raises a fault when the walk can't complete legally. The OS handler classifies:

|Kind|Meaning|Disk?|Cost|Examples|
|---|---|---|---|---|
|**Minor (soft)**|frame is already in RAM; only this process's PTE is missing/invalid|No|~µs|first touch of lazily-allocated heap (zero-page), library page already resident from another process, **COW break**|
|**Major (hard)**|data must come from disk|Yes|~ms; process sleeps|swapped-out page, first access to a non-resident mmap'd file region|
|**Invalid**|no vma / permission violation|—|**SIGSEGV** (or SIGBUS)|null deref, write to .text, use-after-munmap|

**Demand paging:** nothing is populated eagerly. `malloc`/`mmap` just record "this range is valid" in kernel bookkeeping (VMAs); the first touch of each page minor-faults and gets a frame. Consequence: `malloc(1 GiB)` is instant; touching it costs 262,144 minor faults (why HFT prefaults + `mlock`s memory at startup).

**Copy-on-write (fork):** `fork()` duplicates the page _table_, not the pages; both processes' PTEs point at the same frames, all marked **read-only**. First write by either → protection fault → kernel sees it's COW → copies that one page, remaps writable → resumes. `fork`+`exec` therefore copies almost nothing. This is _the_ canonical minor-fault example.

**Thrashing:** aggregate working set > RAM → every eviction is soon re-faulted → system spends all time paging. Signature: high major-fault rate, low CPU utilization. Remedy: fewer processes, more RAM, better locality.

**Page replacement** (one-liner level): true LRU too costly → **clock/second-chance** approximation using the PTE _accessed_ bit; dirty pages cost extra (write-back before reuse).

---

# PART C: PROCESSES, THREADS, SCHEDULING

## C1. Hardware execution resources

- **Processor/CPU/socket:** the physical chip.
- **Core:** independent execution engine within it — own registers, ALUs, pipeline, L1i/L1d, L2; shares L3 + memory controller with sibling cores. N cores = N instructions streams truly in parallel.
- **Hardware thread (SMT / hyperthreading):** one core exposes 2 logical CPUs by duplicating _architectural state_ (register file, PC) while **sharing execution units, L1/L2, TLB, branch predictor**. When thread A stalls (cache miss), thread B uses the idle slots → throughput +10–30%, not 2×. Costs: the two threads contend for cache/TLB → jitter, side channels → **HFT typically disables SMT** (or pins one critical thread per physical core).
- Hierarchy: processor ⊃ cores ⊃ HW threads ← [OS scheduler maps] ← SW threads ⊂ processes.

## C2. Process vs thread (software)

|Resource|Process|Threads in one process|
|---|---|---|
|Virtual address space / page table|own|**shared**|
|Heap, globals, code|own|shared|
|File descriptors, signal handlers, cwd|own|shared|
|Registers, PC, stack|own|**own** (each thread its own stack)|
|errno, TLS|—|own|

- Process = resource container + isolation boundary. Thread = unit of scheduling/execution.
- Communication: threads share memory natively (hence need synchronization); processes need **IPC** — pipes, sockets, shared memory (`shm`/`mmap` — fastest, kernel only sets up the mapping), message queues, signals.
- Tradeoff: threads are cheap + fast to communicate but one thread's crash/corruption kills all; processes are isolated but heavier.
- Creation: `fork()` (clone the process, COW) + `exec()` (replace image); threads via `pthread_create`/`std::thread` (much cheaper: allocate a stack, share everything else — on Linux both are `clone()` with different flags).

## C3. Context switching

Mechanics: timer interrupt or block/yield → kernel entry → save old task's registers into its task struct → pick next task (scheduler) → restore its registers, switch kernel stack → if different **process**, load its CR3 → return to user mode.

- **Thread→thread (same process):** no CR3 change → TLB and much cache content stay warm → cheapest.
- **Process→process:** CR3 change → TLB entries stale (flush or PCID) → the _indirect_ cost (cold caches, cold TLB, cold branch predictor for the incoming task) usually exceeds the ~1–10 µs direct cost.
- **Mode switch ≠ context switch:** a syscall enters the kernel but the same task continues — much cheaper than switching tasks.

## C4. Scheduling

Goals in tension: throughput vs latency/responsiveness vs fairness.

Algorithms (recognize + one property each):

- **FCFS:** simple; convoy effect (short job stuck behind long).
- **SJF/SRTF:** optimal average waiting time; needs future knowledge; starves long jobs.
- **Round-robin:** FIFO + time quantum. **Quantum tradeoff (classic MCQ):** too small → context-switch overhead dominates (e.g., 1 µs switch, 10 µs quantum = 10% overhead); too large → degenerates to FCFS, interactive latency dies. Rule: quantum ≫ switch cost, still ms-scale.
- **Priority:** starvation of low-prio → fix with **aging** (waiting raises effective priority).
- **MLFQ:** multiple RR queues by priority; new/I/O-bound tasks start high, CPU hogs sink → auto-classifies interactive vs batch.
- **Linux CFS:** red-black tree ordered by virtual runtime; always run the task with least vruntime → weighted fairness. `SCHED_FIFO`/`SCHED_RR` = real-time classes that preempt everything (HFT uses these + isolated cores).

Concepts:

- **Preemptive** (timer interrupt can yank the CPU — all modern OSes) vs **cooperative** (must yield voluntarily).
- I/O-bound tasks block early → schedulers boost them (better latency, keeps devices busy); CPU-bound consume full quanta.
- **Priority inversion:** L holds a lock H needs; M (middle priority) preempts L → H waits on M. Mars Pathfinder bug. Fix: **priority inheritance** (L temporarily inherits H's priority) or priority ceiling.
- **Affinity/pinning** (`taskset`, `pthread_setaffinity_np`): keep a thread on one core → preserves L1/L2/TLB warmth, avoids migration; `isolcpus`/`nohz_full` for dedicated cores (HFT staple).

---

# PART D: SYNCHRONIZATION & MEMORY ORDERING

## D1. Data races & critical sections

- **Data race:** two threads access the same memory location concurrently, at least one is a write, no synchronization → **undefined behavior** in C++ (not "you get a stale value" — UB, the compiler may miscompile).
- Distinct from **race condition** (a correctness bug from timing/interleaving, possible even with no data race — e.g., check-then-act across two properly-locked calls: `if (!m.count(k)) m[k]=v;` under two separate lock acquisitions).

## D2. Locks

- **Mutex:** ownership-based blocking lock. Linux implementation = **futex**: uncontended acquire/release is a single userspace CAS (no syscall); only contention takes the syscall path and sleeps the thread. Sleeping frees the CPU for other work; wakeup costs a scheduling round-trip.
- **Spinlock:** loop on CAS until acquired. Correct choice only when: critical section is tiny (tens of ns) AND holder is guaranteed running on another core (so the wait is short). Pathologies: on one core, spinning while the holder is _preempted_ burns the whole quantum achieving nothing; naive spinning also hammers the line with RFOs — better spin on a _read_ (test-and-test-and-set) + `_mm_pause()`.
- **Reader-writer lock:** many concurrent readers XOR one writer. Wins only when reads are long and vastly outnumber writes; the shared cacheline for the reader count itself becomes a bottleneck at scale. Writer starvation possible depending on policy.
- **Semaphore:** counter + wait/post; N-resource pool or signaling; no ownership concept (any thread may post — unlike a mutex).
- `std::lock_guard` (RAII, scope-bound), `std::unique_lock` (movable, needed for CVs), `std::scoped_lock` (multiple mutexes acquired atomically → deadlock-free).

## D3. Condition variables — exam favorite

Purpose: sleep until a predicate over shared state becomes true, without burning CPU.

Canonical pattern (memorize verbatim):

```cpp
// waiter                                  // notifier
std::unique_lock<std::mutex> lk(m);        {
cv.wait(lk, []{ return ready; });             std::lock_guard<std::mutex> g(m);
// == while(!ready) cv.wait(lk);              ready = true;
// here: lock held AND predicate true      }
                                           cv.notify_one();
```

Rules and their reasons:

1. **Always loop on the predicate** (`while`, never `if`):
    - **Spurious wakeups:** `wait` may return with no notify at all (permitted by POSIX/C++ for implementation efficiency).
    - **Stolen wakeups:** between the notify and your thread actually running, a third thread may have consumed the state.
2. **Post-condition of `wait` returning:** you hold the mutex; **the predicate is NOT guaranteed** — that's exactly why you re-check.
3. `wait` performs _atomically_: unlock mutex + go to sleep; on wakeup it reacquires the mutex before returning. The atomicity closes the gap where a notify could land between your unlock and your sleep.
4. **The predicate must be written under the mutex** (otherwise: **lost wakeup** — waiter checks `ready==false`, notifier sets it and notifies _before_ the waiter sleeps, waiter sleeps forever). Calling `notify_*` itself without the mutex is legal (and can avoid a pointless wakeup-then-block).
5. `notify_one` wakes ≥1 waiter; `notify_all` wakes all (they serialize reacquiring the mutex; use when waiters wait on different predicates or all must proceed).

## D4. Deadlock & friends

**Coffman conditions — all four necessary simultaneously:**

1. Mutual exclusion (resource not shareable)
2. Hold and wait (hold one, request another)
3. No preemption (can't forcibly take it back)
4. Circular wait (cycle in the wait-for graph)

Breaking any one prevents deadlock. Practical fixes, mapped to the condition broken:

- **Global lock ordering** (break circular wait) — the standard answer.
- Acquire all at once / `std::scoped_lock` / try-lock + release-and-retry (break hold-and-wait).
- Timeouts / try_lock (a preemption-ish escape).
- Detection + recovery (databases: build wait-for graph, abort a victim).

Related failure modes:

- **Livelock:** threads are active but perpetually reacting to each other (both back off, both retry, collide again) — no progress with full CPU usage.
- **Starvation:** some thread never gets the resource/CPU although others progress (unfair lock, priority scheduling without aging).

## D5. Atomics & lock-freedom

- `std::atomic<T>`: indivisible loads/stores/RMWs. `fetch_add`, `exchange`, `compare_exchange_{weak,strong}` (CAS) compile to single locked instructions on x86 (`lock xadd`, `lock cmpxchg`) — a lone counter needs no mutex.
- CAS loop pattern: read expected → compute desired → `compare_exchange`; on failure `expected` is refreshed → retry. `_weak` may fail spuriously → fine inside loops, use `_strong` for one-shot.
- **ABA problem:** value went A→B→A between your read and CAS → CAS succeeds though state changed (dangling in lock-free stacks). Fixes: tagged/versioned pointers, hazard pointers.
- Progress guarantees: **lock-free** = _some_ thread completes in bounded steps (no mutual blocking; immune to a holder being preempted); **wait-free** = _every_ thread does. Lock-free ≠ faster; it's about progress under adversity.
- `atomic<T>::is_lock_free()`: true for ≤8 B (16 B with cmpxchg16b); larger T falls back to an internal lock.

## D6. Memory ordering — x86 TSO

Two independent reordering sources; both must be tamed:

1. **Compiler** — may reorder/eliminate/hoist ordinary accesses. Constrained by the C++ memory model (atomics, fences). `volatile` prevents _elision_ for MMIO but provides **no** inter-thread ordering/atomicity — classic trap answer.
2. **CPU** — x86 = **Total Store Order**:
    - Load-load: preserved. Store-store: preserved. Load-store: preserved.
    - **Store→load: the one visible reordering.** A later load may complete before an earlier store becomes globally visible, because the store sits in the **store buffer**. (The core sees its own store via store-to-load forwarding; others don't yet.)

- **Litmus test (Dekker):** `T1: x=1; r1=y;` `T2: y=1; r2=x;` (x=y=0 initially). On x86, **r1==0 && r2==0 is possible** — both stores buffered when the loads execute. Preventing it requires a full fence (`mfence`) or seq_cst atomics.
- C++ → x86 mapping (why acquire/release is "free" on x86):
    
    |C++ operation|x86 code|
    |---|---|
    |load relaxed/acquire/seq_cst|plain `mov`|
    |store relaxed/release|plain `mov`|
    |**store seq_cst**|`mov` + `mfence` (or `xchg`) — the expensive one|
    |any atomic RMW|`lock`-prefixed op = full barrier|
    
- Semantics ladder: **relaxed** = atomicity only, no ordering (counters). **acquire** (on loads): later ops can't move before it. **release** (on stores): earlier ops can't move after it. Release-store → acquire-load that reads it = _synchronizes-with_: the acquirer sees everything the releaser did before the store. This is the flag/data-handoff pattern:
    
    ```cpp
    // producer                          // consumerdata = 42;                           while (!flag.load(std::memory_order_acquire));flag.store(true, std::memory_order_release);assert(data == 42);                  // guaranteed
    ```
    
- **seq_cst** additionally imposes one single global order over all seq_cst ops (needed for Dekker-style algorithms; default for `std::atomic`).
- ARM/POWER are _weakly ordered_ (all four reorderings possible) → acquire/release compile to real barrier instructions there; code that "worked on x86" with relaxed ordering breaks on ARM.

---

# PART E: PIPELINE, PREDICTION, ILP

## E1. Pipelining

- Classic 5 stages: **IF → ID → EX → MEM → WB**; modern x86: 14–20 stages, decoding x86 into µops.
- Throughput vs latency: pipelining raises instruction _throughput_ (one finishing per cycle ideally); each instruction's latency is unchanged or worse.
- **Hazards:**
    - _Data_: instruction needs a prior result → solved by **forwarding/bypass** (route result straight from EX output to the next EX input) or a stall (classic unavoidable one: load-use — a load followed immediately by its consumer stalls 1 cycle).
    - _Control_: which instruction follows a branch? → prediction (below).
    - _Structural_: two instructions need the same hardware unit at once → duplicate units / stall.
- Deeper pipeline → higher clock, but every flush (mispredict) discards more work.

## E2. Branch prediction

- The front-end fetches _predicted_ instructions speculatively; a **misprediction flushes the pipeline ≈ 15–20 cycles**.
- Predictors: BTB (target addresses), direction predictors using global/local history (modern: TAGE/perceptron — assume "learns any stable pattern, even long ones"). Loop-closing branches and consistent patterns: ~99%+ accuracy. **Data-dependent 50/50 branches are the killer** (`if (a[i] < threshold)` on random data).
- Canonical demo: summing elements above a threshold is several× faster on a **sorted** array — sorted makes the branch pattern predictable (all-false then all-true).
- Mitigations: branchless (`cmov`, arithmetic masks, `(x < t) * x`), lookup tables, `[[likely]]/[[unlikely]]`, sorting data, converting control dependency to data dependency.
- Return address stack predicts `ret`; deep/mismatched call-return breaks it.

## E3. Superscalar & out-of-order execution

- **Superscalar:** issue/execute several instructions per cycle (4–6 wide).
- **Out-of-order:** instructions wait in a scheduler; each executes as soon as _its inputs_ are ready, regardless of program order; results **retire in order** (via the reorder buffer, so exceptions/interrupts see a precise architectural state). Register renaming removes false (name) dependencies.
- What OoO hides: it runs independent work under a cache miss. What it cannot hide: a **serial dependency chain** — if each operation needs the previous result, ILP = 1. Pointer chasing (`p = p->next`) is the extreme case: each load's _address_ depends on the previous load → one long chain of DRAM latencies, nothing to overlap.
- **Hardware prefetcher:** watches L1/L2 miss streams; sequential or constant-stride patterns get fetched ahead → streaming an array can approach memory bandwidth with near-zero visible miss latency. Random access defeats it entirely.
- The `vector` vs `list` verdict, fully justified (assemble all three): (1) spatial locality — 16 ints/line vs 1 node + padding; (2) prefetcher — linear stride predicted vs random node addresses; (3) dependency chain — independent `a[i]` loads overlap vs serial `p->next`.

---

# PART F: KERNEL INTERFACE

## F1. User vs kernel mode, syscalls

- CPU privilege rings: user code can't touch page tables, devices, or other processes; it must ask the kernel via **syscalls** (`syscall` instruction → mode switch to kernel, kernel stack, dispatch; `sysret` back).
- **Mode switch ≠ context switch:** the same task keeps running, just in kernel mode. Cost ~100 ns–1 µs direct, plus cache/branch-predictor/TLB pollution → why hot paths avoid syscalls: batching (`writev`, `sendmmsg`), `io_uring` (submit/complete via shared-memory rings), kernel-bypass networking (DPDK/OpenOnload — NIC DMA-mapped into userspace, zero syscalls per packet), `vDSO` (`gettimeofday` etc. as userspace reads of a kernel-shared page).
- Blocking syscall (`read` on an empty socket): kernel puts the task to sleep on a wait queue, schedules another; data arrival → interrupt → wake.

## F2. Interrupts & exceptions

- **Interrupt:** asynchronous, from hardware (timer, NIC, disk). CPU finishes current instruction, saves state, vectors through the IDT into a kernel handler.
- **Exception/trap:** synchronous, caused by the executing instruction (page fault, divide-by-zero, `int3`, syscall itself).
- **The timer interrupt is the mechanism of preemption** — without it, a spinning user task could never be descheduled (cooperative-only).
- Handler discipline: _top half_ minimal (ack device, queue work); _bottom half_ (softirq/tasklet/workqueue) does the heavy lifting later with interrupts enabled. Network RX under load switches to polling (NAPI) to avoid interrupt storms.
- Interrupts steal cycles + trash caches on whichever core they land → HFT routes IRQs away from critical cores (irq affinity) and isolates cores.

## F3. Odds & ends that appear in MCQs

- `mmap` vs `read`: `read` copies kernel→user buffer per call; `mmap` maps the page cache into your address space — pages fault in lazily, no copy, shared across processes. Random access to a big file: mmap wins; small sequential reads: buffered read is fine.
- **Zero-copy**: `sendfile`/`splice` move file→socket without a userspace round-trip.
- `malloc` is a _library_ (arena management in userspace); it gets memory from the kernel via `brk`/`mmap` only when its arenas run dry. Most `malloc` calls involve **no syscall**.
- Stack vs heap: stack allocation = pointer bump, freed by scope, hot in cache; heap = allocator search + fragmentation + (rarely) syscall.
- Prefaulting + `mlock`: touch and pin memory at startup so the trading path never takes a fault.

---

# PART G: LAST-HOUR QUICK-FIRE

Numbers: line 64 B · page 4 KiB · L1 ~4cy · DRAM ~100 ns · mispredict ~15–20cy · syscall ~µs⁻ · ctx switch ~µs · major fault ~ms.

- `#sets = size / (line × ways)`; collisions at stride = #sets × 64 B.
- Streamed-once data → **compulsory** misses; associativity irrelevant; prefetcher hides it.
- Conflict misses exist in direct-mapped and N-way; **fully associative eliminates conflict only**.
- Coherence unit = **line**; translation unit = **page**. Never mix them in an answer.
- E state exists so private data can be written with zero bus traffic; write in S = RFO/invalidate broadcast.
- **False sharing = MESI line ping-pong; fix `alignas(64)`.** Not TLB. Not capacity.
- Multi-level paging: savings = **subtrees for unmapped regions never allocated**; mapped pages still 1 PTE each; fully-mapped space → no savings; walk = 4 _sequential_ reads from CR3.
- TLB caches **final translations**, not PTEs; hit → **zero** table accesses; miss → HW walk; walk failure → page fault (≠ TLB miss).
- Context switch: new CR3 → TLB flush unless **PCID/ASID**; same-process thread switch keeps CR3.
- Huge pages: shorter walk + 512× TLB reach.
- Minor fault: frame in RAM (lazy alloc, COW, resident lib) — no disk. Major: disk. `fork` cheap via COW; first write = minor fault.
- Thread vs process: threads share address space/heap/FDs; own stack/registers. Thread switch < process switch.
- SMT: shares core's execution units + L1/L2; duplicates registers; +10–30%; off in HFT.
- RR quantum: small → overhead; big → FCFS. Priority → starvation → aging. Inversion → inheritance.
- Mutex: futex, no syscall uncontended. Spinlock: only for tiny sections, holder running; disaster if holder preempted.
- **CV: `while(!pred) wait;` — spurious + stolen wakeups; on return: mutex held, predicate NOT guaranteed; predicate writes under mutex or lost wakeup.**
- Deadlock = all 4 Coffman conditions; lock ordering breaks circular wait; `scoped_lock` breaks hold-and-wait. Livelock = active, no progress.
- Data race = UB; race condition possible without data race (check-then-act).
- **x86 TSO: only store→load reorders (store buffer). Dekker needs mfence/seq_cst.** Acquire/release ≈ free on x86; seq_cst store pays a fence. `volatile` ≠ synchronization. ARM = weak → real barriers.
- CAS loop + ABA (fix: version tags). Lock-free = someone progresses; wait-free = everyone.
- OoO hides misses behind independent work; can't fix serial dependency chains → pointer chasing worst case. `vector` > `list`: locality + prefetcher + independent loads.
- Mode switch (syscall) ≠ context switch. Timer interrupt ⇒ preemption. mmap: lazy, zero-copy, page-cache-backed.