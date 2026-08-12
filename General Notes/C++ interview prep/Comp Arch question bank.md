# Computer Architecture — Interview Master List

### C++ low-latency graduate roles · question + ≤30-second verbal model answer

**How to use this.** Every answer is written to be _said_, not read — 40–70 words, one breath of setup and one of substance. If you find yourself needing 45 seconds, you have added a caveat the interviewer did not ask for. Say the number, say the mechanism, stop. The follow-up is where you earn the points.

**Numbers convention.** Assume a modern x86-64 server core at ~3 GHz, so **1 cycle ≈ 0.3 ns**. All figures are order-of-magnitude — say "roughly" and "order of" out loud. Nobody expects three significant figures; everybody expects you to know a DRAM access is ~50× an L1 hit.

---

## 0. The memorization core

If you retain nothing else, retain this block. These numbers appear in some form in the majority of low-latency screens, and every other section on this list is downstream of them.

|Access|Cycles|Time|Typical size|
|---|---|---|---|
|Register|0 (bypassed)|—|~180 physical|
|**L1 data cache hit**|**~4–5**|**~1.5 ns**|32–48 KB, private|
|**L2 hit**|**~12–15**|**~4 ns**|0.5–2 MB, private|
|**L3 hit**|**~40–50**|**~15 ns**|1.5–3 MB per core, shared|
|**Main memory (DRAM)**|**~200–300**|**~70–100 ns**|GBs|
|Remote NUMA node DRAM|~400–500|~130–150 ns|—|
|Cache-to-cache (other core, same socket)|~100–200|~40–70 ns|—|
|Branch mispredict|~15–20|~5–6 ns|—|
|Uncontended atomic RMW (`lock xadd`)|~15–20|~5–7 ns|—|
|Contended atomic RMW|—|~40–500 ns|—|
|TLB miss + page walk (tables cached)|~10–30|~5–10 ns|—|
|Mutex lock/unlock, uncontended|~20–25|~7 ns|—|
|Mutex contended → futex sleep|thousands|~1–5 µs|—|
|Context switch|—|~1–5 µs|—|
|NVMe SSD read|—|~50–100 µs|—|
|Datacenter network round trip (kernel)|—|~20–100 µs|—|
|Same, kernel-bypass (Solarflare/DPDK)|—|~1–5 µs|—|

**Cache line = 64 bytes.** Say it without hesitating. Everything in sections 2–4 and 12 falls out of that single fact.

**The one-sentence framing to open with:** _"The hierarchy is a latency-versus-capacity trade-off, and the whole discipline of low-latency C++ is arranging your data so the fast, small level is the one you actually touch."_

---

## 1. Memory hierarchy & latency numbers

**Q1.1 — Walk me through the memory hierarchy with rough latencies.** Registers are free, L1 data is about 4 cycles, L2 about 12, L3 about 40, and main memory about 200 to 300 cycles — call it 100 nanoseconds. Sizes go the other way: 32 kilobytes of L1, a megabyte or so of L2, tens of megabytes of shared L3, then gigabytes of DRAM. Each step down is roughly 3 to 5 times slower.

**Q1.2 — Why is there a hierarchy at all? Why not one big fast memory?** Because fast and big are physically opposed. SRAM is fast but needs six transistors per bit, so it's expensive and area-hungry; DRAM is one transistor and a capacitor, so it's dense but slow and needs refreshing. Signal propagation also matters — a larger array is physically further from the core, and distance costs time.

**Q1.3 — How do you convert cycles to nanoseconds?** Divide by the clock. At 3 gigahertz one cycle is a third of a nanosecond, so multiply cycles by 0.33. A 200-cycle memory access is about 70 nanoseconds. I quote cycles when I'm reasoning about the core and nanoseconds when I'm reasoning about wire time or a latency budget.

**Q1.4 — Which caches are private and which are shared?** L1 and L2 are per-core private, L3 is shared across all cores on the socket. That's why L3 is the meeting point for coherence traffic — two cores communicating hit each other through the shared level or through direct cache-to-cache transfer, and that's the ~40 to 70 nanosecond number.

**Q1.5 — What's the split between L1 instruction and L1 data cache?** L1 is split — typically 32 kilobytes of instructions and 32 to 48 of data, separately. That's a Harvard-style split at L1 only; L2 and below are unified. It exists because instruction fetch and data access happen in the same cycle and would otherwise contend for ports. It's also why code size matters — a bloated hot loop can miss in L1i.

**Q1.6 — How long does it take one core to read a value another core just wrote?** Roughly 40 to 70 nanoseconds on the same socket, because the line has to be transferred out of the writer's cache — it isn't a normal L3 hit, it's a coherence transaction. Cross-socket it's more like 100 to 200. That number is the hard floor on any inter-thread handoff, including a lock-free queue.

**Q1.7 — Latency or bandwidth: which one actually limits you?** Depends on the access pattern. Streaming sequentially through a large array is bandwidth-bound — the prefetcher hides latency and you saturate the memory controller, maybe 10 to 20 gigabytes a second per socket. Pointer-chasing is latency-bound, because each access has to complete before you know the next address. Low-latency work is almost always the second case.

**Q1.8 — What is memory-level parallelism?** The ability to have several cache misses outstanding at once. The core tracks them in miss-status holding registers — maybe 10 to 16 of them. So ten independent misses cost roughly one miss's latency, not ten. That's why independent array traversals are cheap and dependent pointer chains are catastrophic: the chain serializes what the hardware wanted to overlap.

**Q1.9 — What does the store buffer do?** It lets a store retire without waiting for the cache line to arrive. The store sits in the buffer, the core moves on, and the write drains later. That's why stores look cheap and loads look expensive. It's also the source of x86's one reordering — a later load can pass an earlier store to a different address, which is exactly what a sequentially-consistent store has to prevent.

**Q1.10 — How would you measure these numbers on a specific machine?** A pointer-chase microbenchmark: build a randomly permuted linked list sized to fit each level, walk it, divide total time by hops. The dependency chain defeats the prefetcher so you measure true latency. Off the shelf I'd use Intel MLC or lmbench, and `perf stat` for the miss counters to confirm I'm hitting the level I think I am.

---

## 2. Cache organization

**Q2.1 — How is a cache organized?** Memory is split into 64-byte lines. The cache is an array of sets, each holding N lines — that's N-way associativity. An address splits into three fields: the low 6 bits are the offset within the line, the next bits index the set, and the rest is the tag. On a lookup you index the set, compare the tag against all N ways in parallel, and hit or miss.

**Q2.2 — Why not fully associative?** Because a fully associative cache has to compare the tag against every line, which needs a comparator per line — too much power, area, and delay for something on the critical path. Direct-mapped is the other extreme: one comparator, but a fixed home for each address, so conflicts are brutal. 8 or 16 ways is the practical compromise.

**Q2.3 — What's a conflict miss, and how does it differ from a capacity miss?** A capacity miss is when your working set simply exceeds the cache. A conflict miss is when it fits, but too many hot addresses map to the same set and evict each other. The classic trigger is a power-of-two stride — striding by 4096 bytes lands everything in one set, so a 16-way cache holds 16 lines and thrashes.

**Q2.4 — Name the categories of cache miss.** Compulsory — first touch, the data was never there. Capacity — working set too big. Conflict — associativity too low for the mapping. And in a multicore system, coherence misses, where a line you had was invalidated by another core's write. That fourth one is the interesting one for us, because it's the one your own code creates.

**Q2.5 — Write-back versus write-through?** Write-through pushes every store to the next level immediately — simple, always consistent, but it burns bandwidth. Write-back marks the line dirty and only writes it out on eviction, so repeated writes to the same line coalesce. Modern CPUs use write-back everywhere; the cost is that eviction of a dirty line is more expensive than a clean one.

**Q2.6 — What is a read-for-ownership?** When you store to a line you don't own, the core has to fetch the whole 64-byte line and get exclusive permission first — even if you're overwriting all of it. So a "write" is really a read plus an invalidation of every other copy. It's why writing to memory costs a full miss, and why write-allocate behavior surprises people benchmarking memsets.

**Q2.7 — Can you avoid the read-for-ownership?** Yes, with non-temporal stores — `movntdq`, or `_mm_stream_si128`. They write straight to memory through a write-combining buffer, bypassing the cache, so you skip the fetch and don't pollute L3. They're for large streaming writes you won't read back. They need a fence afterwards, because they're weakly ordered even on x86.

**Q2.8 — What eviction policy do real caches use?** Approximate LRU — true LRU needs too much state per set, so hardware uses pseudo-LRU trees or re-reference prediction. The practical point is that you can't rely on exact eviction order, and adversarial access patterns can defeat it. You also can't pin a line in cache from userspace, which is why cache warming is done by touching data, not by instruction.

**Q2.9 — Inclusive or exclusive cache hierarchy?** Inclusive means L3 holds a copy of everything in L1 and L2 — simplifies coherence, since a snoop only has to check L3, but wastes capacity and means an L3 eviction forces an L1 eviction. Exclusive maximizes total capacity. Intel moved from inclusive to non-inclusive on the server line; either answer is fine if you name the trade-off.

**Q2.10 — Your struct is 40 bytes. What does that mean for cache?** It means lines and objects don't align — some objects straddle two 64-byte lines, so touching one costs two lines. Sizes that divide or are multiples of 64 keep the mapping regular. It also means iterating an array of them uses 40 of every 64 bytes fetched; the other 24 came along for free but only help if you use them.

---

## 3. Cache-friendly code

**Q3.1 — Define spatial and temporal locality.** Temporal locality is reusing the same data soon — the cache keeps it, so the reuse is free. Spatial locality is using data near what you just used — since the hardware fetched a whole 64-byte line and the prefetcher pulls the next ones, the neighbors are already there. Good code exploits both; the phrase for the second one is "you paid for the line, use it."

**Q3.2 — Why is row-major traversal faster than column-major in C++?** Because C++ arrays are laid out row-major, so consecutive columns are contiguous. Walking a row touches 16 ints per cache line and the prefetcher sees a clean stride. Walking a column strides by the row length, so every access is a new line and possibly a new page — you use 4 bytes of every 64 fetched, a 16× waste of bandwidth.

**Q3.3 — What does the hardware prefetcher actually detect?** Constant strides and sequential streams, tracked per page. There's a next-line prefetcher, an adjacent-line one that pulls a 128-byte pair, and a stride detector in L2. Two things it can't do: cross a 4-kilobyte page boundary, because it works on physical addresses; and follow a pointer, because the address depends on data it hasn't loaded yet.

**Q3.4 — Array of structs versus struct of arrays?** AoS keeps one object's fields together — good when you touch most fields of one object. SoA keeps one field's values together across objects — good when you sweep one field across many objects, since every byte in the line is useful and it vectorizes. If your hot loop reads one field out of a 64-byte struct, SoA can be an order of magnitude faster.

**Q3.5 — Why is pointer-chasing so slow, beyond just cache misses?** Because it serializes. With an array the core issues many independent loads and overlaps ten misses into roughly one miss of latency. With a linked list, each address comes from the previous load's result, so you pay full DRAM latency per hop with nothing overlapped. A million-node list walk is a million serialized 100-nanosecond stalls.

**Q3.6 — `std::vector` versus `std::list` for a few thousand elements — which and why?** Vector, almost always. Insertion in the middle is O(n) memmove versus list's O(1), but the memmove is a contiguous streaming copy that the hardware does at gigabytes per second, while reaching the list position at all required a chain of cache misses. The complexity class favors the list; the constant factor favors the vector by 10× or more.

**Q3.7 — What is loop blocking or tiling?** Restructuring nested loops so you work on a sub-block that fits in cache, finish with it, then move on. Naive matrix multiply streams a whole row and column per output element and evicts everything. Tiled multiply loads a block once and reuses it O(tile) times. Same arithmetic, same complexity, but the reuse now happens in L1 instead of DRAM.

**Q3.8 — What's hot/cold splitting?** Separating the fields you touch every iteration from the ones you rarely touch, into two structures joined by an index. The hot struct shrinks, so more objects fit per cache line and per cache. Typical case: an order book entry where price and quantity are hot but the client ID, timestamps, and flags are only touched on a fill.

**Q3.9 — Why does field ordering inside a struct matter?** Two reasons. Alignment padding — ordering members largest-to-smallest minimizes the holes the compiler inserts, so the struct is smaller and more fit per line. And grouping — fields used together should sit together, so a hot pair lands on one line rather than straddling two. Reordering members is a free optimization you can do at review time.

**Q3.10 — When would you use a software prefetch?** Only when I know the address well ahead of use and the hardware can't derive it — walking a hash table's bucket array, or processing a batch where I can prefetch element i+8 while working on i. `__builtin_prefetch`. It's easy to make things worse: too early and it's evicted, too late and it does nothing, and it costs issue slots either way. I'd measure.

**Q3.11 — Open addressing or chaining for a latency-sensitive hash map?** Open addressing. Chaining means every collision is a pointer dereference to a separately allocated node — a fresh cache miss with nothing prefetchable. Open addressing probes linearly within the same cache line or the next, so a collision is often free. That's the core reason `std::unordered_map` is avoided in this domain: the standard mandates bucket-and-chain semantics.

**Q3.12 — What's the single highest-leverage cache optimization?** Make the data smaller. Everything else — prefetching, tiling, SoA — is about using lines efficiently, but shrinking the working set until it fits in a faster level moves you an entire order of magnitude at once. Smaller types, fewer indirections, no pointers where an index fits, and arenas instead of scattered allocations.

---

## 4. Coherence & false sharing

**Q4.1 — Explain MESI in thirty seconds.** Every cache line in every core is in one of four states. Modified: I have the only copy and it's dirty. Exclusive: only copy, clean. Shared: several cores have clean copies. Invalid: I don't have it. Reads move you to Shared or Exclusive; a write requires Modified, which means invalidating everyone else's copy first. That invalidation is the cost of sharing.

**Q4.2 — What happens on the wire when I store to a line another core has cached?** My core issues a request for ownership. The coherence fabric sends invalidations to every core holding the line; they mark theirs Invalid and acknowledge; if one held it Modified it writes back or transfers directly. Only then does my store commit. That's the ~40 to 70 nanoseconds — the store itself is one cycle, the permission is everything.

**Q4.3 — What is false sharing?** Two threads writing to different variables that happen to live on the same 64-byte cache line. Logically there's no sharing — but coherence works at line granularity, so each write invalidates the other core's copy. The line ping-pongs between cores and both threads run at coherence speed instead of L1 speed. It's a 10× to 100× slowdown from a layout accident.

**Q4.4 — Give a concrete example.** An array of per-thread counters — `long counts[8]`, one per thread. All eight are 64 bytes total, so they occupy one line, and every increment from every thread invalidates all the others. The fix is padding each counter to its own line, at which point it scales linearly. This is the canonical interview example and it's worth naming as such.

**Q4.5 — How do you fix it in C++?** `alignas(64)` on the member or a padded wrapper struct, so each hot variable owns its line. Standard-blessed version is `std::hardware_destructive_interference_size` from `<new>`, which is 64 on x86 — though libstdc++ warns about ABI stability, so a lot of codebases just hardcode 64 with a `static_assert`. Also pad after the last member, not just between.

**Q4.6 — Why do some codebases pad to 128 bytes instead of 64?** The adjacent-line prefetcher fetches lines in aligned 128-byte pairs, so a write to one line can pull its partner into the neighbouring core and cause the same ping-pong one line away. Some Intel parts also have 128-byte sector behavior. So 128 is the paranoid setting; `hardware_destructive_interference_size` is 64, and on Apple silicon the line itself is 128.

**Q4.7 — How would you detect false sharing on a running system?** `perf c2c` — it's built exactly for this, and reports HITM events: loads that hit a line Modified in another core's cache, attributed down to the source line and the offset within the line. Symptom signature is a parallel program that gets slower as you add threads, with a high HITM rate and low instructions-per-cycle.

**Q4.8 — False sharing versus true sharing?** False sharing is an accident of layout — separate the variables and it disappears. True sharing is two threads genuinely contending for the same variable, and no amount of padding helps; the line has to move. The fix there is algorithmic: per-thread accumulators reduced at the end, sharding, or restructuring so the hot variable isn't shared at all.

**Q4.9 — Where does false sharing bite in an SPSC queue?** The head and tail indices. The producer writes head every push and the consumer writes tail every pop; if they share a line, every operation invalidates the other side's cache line and you've converted a lock-free queue into a coherence ping-pong. You `alignas(64)` both, pad after the last one, and cache the opposite index locally to avoid even reading it.

**Q4.10 — Is padding ever the wrong call?** Yes — it's a space-versus-contention trade. Padding a hot pair of variables that one thread uses together pushes them onto two lines and doubles the misses. The rule is: pad what different threads write, group what one thread reads together. Blanket `alignas(64)` on everything wastes cache and can make single-threaded code slower.

---

## 5. Virtual memory hardware

**Q5.1 — How does a virtual address become a physical one?** On x86-64 it's a four-level page table walk. The top 48 bits of the address are split into four 9-bit indices, each selecting an entry in the next table down, starting from the base in CR3, ending at a physical frame number that's combined with the 12-bit page offset. Five-level paging exists for very large address spaces.

**Q5.2 — Why isn't that catastrophically slow?** Because of the TLB — a cache of recent translations. A hit is essentially free, resolved in parallel with the L1 lookup. Only a miss triggers the walk. Typical sizes are 64 entries in the L1 data TLB and 1500 or more in the second-level TLB, so a well-behaved working set almost never walks.

**Q5.3 — What does a TLB miss cost?** Around 10 to 30 cycles if the page table entries are themselves in cache, since there's a dedicated page-walk unit and page-structure caches. If the tables have been evicted, each of the four levels is a potential DRAM access, so worst case it's several hundred cycles for a single translation.

**Q5.4 — What is TLB reach, and why does it matter?** Reach is entries times page size — the amount of memory you can address without a miss. With 4-kilobyte pages and 1500 second-level entries that's about 6 megabytes. So a 1-gigabyte working set thrashes the TLB no matter how cache-friendly your access pattern is. That's the specific problem huge pages solve.

**Q5.5 — Explain huge pages.** x86-64 supports 2-megabyte and 1-gigabyte pages alongside 4K. One TLB entry then covers 2 megabytes instead of 4 kilobytes, so reach goes up 512×, and the walk is shorter because it terminates a level early. For a large in-memory book or a big buffer pool that can be the difference between constant TLB thrash and effectively zero misses.

**Q5.6 — Downsides of huge pages?** Internal fragmentation — a 2-megabyte page for a small allocation wastes memory. And transparent huge pages specifically can hurt: the kernel's `khugepaged` compacts memory in the background, which causes latency spikes, and a fault on a huge page zeroes 2 megabytes synchronously. Latency-sensitive shops usually disable THP and reserve explicit hugepages instead.

**Q5.7 — Minor versus major page fault?** A minor fault means the page is in physical memory but not mapped in this process's tables — first touch of a freshly `mmap`ed region, or a copy-on-write break. Microseconds. A major fault means the page must come from disk. Milliseconds. In a latency system a major fault on the hot path is a catastrophic, unrecoverable outlier.

**Q5.8 — How do you keep faults off the hot path?** Pre-fault everything at startup: allocate, then touch every page, or `mmap` with `MAP_POPULATE`. Then `mlockall` with `MCL_CURRENT | MCL_FUTURE` so the kernel can't reclaim or swap it. Same principle as cache warming — you move the cost from the moment it matters to the moment it doesn't.

**Q5.9 — What is a TLB shootdown?** TLBs aren't coherent in hardware, so when one core changes a mapping the kernel must send an inter-processor interrupt to every core that could have cached the stale translation, and each one flushes and acknowledges. It costs microseconds and it stops your threads. Practical consequence: avoid `munmap`, `mprotect`, and `madvise` on the hot path.

**Q5.10 — What's a PCID, and why does it exist?** A process-context identifier tags TLB entries with an address space, so a context switch doesn't have to flush the whole TLB. Without it every switch throws away all your translations and you rebuild from cold. It became much more important with the Meltdown mitigations, which otherwise force a TLB flush on every kernel entry and exit.

---

## 6. Pipelining & hazards

**Q6.1 — What is pipelining, in one breath?** Splitting instruction execution into stages — fetch, decode, execute, memory, writeback — so different instructions occupy different stages simultaneously. It doesn't reduce the latency of any one instruction; it raises throughput toward one instruction retired per cycle, and it lets you clock higher because each stage does less work.

**Q6.2 — What are the three classes of hazard?** Structural — two instructions need the same hardware unit at once. Data — an instruction needs a result that isn't ready. Control — a branch, so you don't know which instruction to fetch next. All three are cases of "the pipeline wants to keep going and something says wait," and each has its own hardware fix.

**Q6.3 — Name the data hazards.** Read-after-write is the real one — a true dependency, you need the value. Write-after-read and write-after-write are false dependencies: they're artifacts of reusing the same register name, not of the data. Register renaming eliminates both by giving each write a fresh physical register, which is why only RAW constrains a modern out-of-order core.

**Q6.4 — What is forwarding?** Bypassing — routing a result directly from the output of the execute stage to the input of the next instruction, instead of waiting for it to be written to the register file and read back. It turns a multi-cycle RAW stall into zero cycles for back-to-back ALU ops. The one it can't fix is a load followed immediately by its use, since the data genuinely isn't there yet.

**Q6.5 — How deep is a modern pipeline and why does that matter?** Roughly 15 to 20 stages. It matters because it sets the cost of a flush: if you mispredict a branch, everything in flight is discarded and you refill from the correct target, so the penalty is on the order of the pipeline depth — about 15 to 20 cycles. Deeper pipelines clock higher but pay more per mistake.

**Q6.6 — How does out-of-order execution relate to hazards?** It converts stalls into useful work. Rather than stalling on a data hazard, the scheduler finds later independent instructions and runs them now, retiring everything in program order at the end. So a hazard costs you only if there's nothing else to do — which is exactly why a long dependency chain is slow and a wide, independent workload isn't.

---

## 7. Branch prediction

**Q7.1 — Why does the CPU predict branches at all?** Because the branch outcome isn't known until it executes, maybe 15 stages in, and the fetcher can't sit idle for 15 cycles. So it guesses and speculatively executes down the predicted path. Modern predictors are 95 to 99 percent accurate on typical code, which is what makes deep pipelines viable in the first place.

**Q7.2 — What does a mispredict cost?** About 15 to 20 cycles — roughly 5 nanoseconds. The speculative work is discarded, the front end restarts at the correct address, and the pipeline refills. At 99 percent accuracy in a tight loop that's negligible; at 50 percent, in a loop with a couple of instructions of real work, the branch dominates everything.

**Q7.3 — The classic: why is processing a sorted array faster than an unsorted one, with identical work?** Because of the data-dependent branch inside the loop — something like "if the value exceeds a threshold, accumulate." Sorted, the condition is false for a long run then true for a long run, so the predictor learns it and is right nearly every time. Unsorted, it's a coin flip, so you eat a mispredict roughly half the iterations — often a 3 to 6× difference.

**Q7.4 — How do you make that loop branchless?** Turn the control dependency into a data dependency. Instead of branching, compute a mask from the comparison and AND it with the value, then always add — or let the compiler emit a conditional move. The work is done unconditionally and the wrong result is discarded arithmetically, so there's nothing to mispredict.

**Q7.5 — When is branchless _slower_?** When the branch was predictable. A well-predicted branch is nearly free and lets you skip the work entirely; a conditional move always executes both sides and, critically, introduces a data dependency the scheduler can't speculate past. So branchless helps on unpredictable branches and hurts on predictable ones. It's a measurement question, not a rule.

**Q7.6 — What's a branchless min or absolute value?** `std::min` typically compiles to a compare plus `cmov` already. Absolute value is a mask trick: arithmetic-shift the value right by 31 to broadcast the sign bit, XOR, subtract the mask. The general point is that these are the compiler's job — I'd check the assembly on Compiler Explorer before writing intrinsics by hand.

**Q7.7 — What is the branch target buffer?** A cache of predicted target addresses, so the front end can fetch the target before the branch resolves. Direction prediction says taken or not; the BTB says where. Indirect branches — virtual calls, function pointers, switch jump tables — depend entirely on it, and if a call site alternates between targets, it mispredicts even though the direction is always "taken."

**Q7.8 — So what's the real cost of a virtual call?** Three things. The vtable indirection is a potential cache miss. The indirect branch may mispredict if the site is polymorphic. And it blocks inlining, which forecloses every optimization downstream. A monomorphic call site in a hot loop predicts perfectly and is nearly free — the cost is variance, not the mechanism. CRTP or `std::variant` are the usual escapes.

**Q7.9 — What is the return stack buffer?** A small hardware stack that predicts return addresses by pairing each call with its return — much more accurate than the BTB, since a function returns to many different places. It's why mismatched call/return patterns hurt, and it's the structure that gets corrupted by things like deep recursion overflowing it or hand-rolled coroutine stack switching.

**Q7.10 — Are `[[likely]]` and `__builtin_expect` worth using?** Occasionally. They don't tell the hardware predictor anything — they tell the compiler how to lay out code, so the hot path is fall-through and contiguous in the instruction cache and the cold path is moved out of line. That's an I-cache and front-end win. Profile-guided optimization does the same thing from real data and is more reliable.

**Q7.11 — How would you measure branch behavior?** `perf stat` gives branch instructions, branch misses, and the miss rate directly; `perf record` attributes them to source lines. I'd look at misses per instruction alongside IPC — a low IPC with a high branch-miss rate points at the front end, versus a low IPC with high cache misses pointing at memory.

---

## 8. Instruction-level parallelism

**Q8.1 — What does superscalar mean?** The core can issue and retire multiple instructions per cycle — modern x86 is roughly 4 to 6 wide. So peak throughput isn't one instruction per cycle, it's several, provided they're independent. Instructions-per-cycle above 1 is normal for good code; below 1 means you're stalling on something.

**Q8.2 — Out-of-order execution in one sentence.** Instructions execute as soon as their inputs are ready rather than in program order, and are retired in order from a reorder buffer, so the architectural state still updates as the program says. It's a hardware mechanism for finding parallelism the compiler couldn't schedule statically.

**Q8.3 — What does register renaming buy you?** It removes false dependencies. Architecturally there are 16 general-purpose registers, but physically there are hundreds; each write gets a fresh physical register, so write-after-read and write-after-write hazards vanish and only true data dependencies constrain the schedule. Without it, register pressure would serialize everything.

**Q8.4 — What limits throughput in a well-optimized loop?** The critical path — the longest chain of dependent operations, since each link must wait for the last. If your chain is five 4-cycle operations, that's 20 cycles per iteration no matter how wide the core is. Breaking the chain into independent sub-chains is the optimization, which is what unrolling with multiple accumulators does.

**Q8.5 — Explain latency versus throughput for an instruction.** Latency is how long one operation takes end to end; throughput is how often you can start a new one. A multiply might be 4-cycle latency but fully pipelined at one per cycle. So four independent multiplies take about 4 cycles total, while four dependent ones take 16. The distinction is the whole reason dependency structure matters more than instruction count.

**Q8.6 — Why does summing an array with four accumulators beat one?** Because floating-point addition has around 4 cycles of latency and the single-accumulator version makes every add depend on the last — one add per 4 cycles. Four independent accumulators fill that latency with useful work, roughly a 4× speedup, then you combine them at the end. The compiler won't do it for you with floats unless you allow reassociation, since it changes rounding.

**Q8.7 — What is the reorder buffer?** The structure holding in-flight instructions from issue to retirement — a couple of hundred entries on a modern core. It's what allows speculation to be undone: results aren't architecturally visible until retirement, so a mispredict or a fault just discards the tail. It also bounds how far ahead the core can look, which caps how many misses you can overlap.

---

## 9. SIMD awareness

**Q9.1 — What is SIMD and what widths exist?** Single instruction, multiple data — one instruction operating on a vector of values. On x86 that's SSE at 128 bits, AVX and AVX2 at 256, and AVX-512 at 512. A 256-bit register holds eight floats or four doubles, so a single add does eight adds. It's throughput parallelism inside one core, orthogonal to threading.

**Q9.2 — When does the compiler auto-vectorize?** When the loop trip count is known or computable, the memory accesses are contiguous or a simple stride, there's no aliasing between input and output, there are no calls or complex control flow in the body, and any reduction is reassociable. Compile with `-O3 -march=native` and check with `-fopt-info-vec-missed`, which tells you which of those failed.

**Q9.3 — What most commonly blocks it?** Possible aliasing. If two pointer parameters might overlap, the compiler must assume a write through one can affect a read through the other and serializes. `__restrict` on the parameters is the fix, or copying into locals. After that, it's branches in the loop body and non-contiguous access.

**Q9.4 — Should you hand-write intrinsics?** Only after checking the generated assembly and failing to get the compiler there with restrict, alignment, and loop restructuring. Intrinsics are unportable, hard to maintain, and easy to get slower than the compiler's version. When they're warranted it's usually for something the compiler can't express — a shuffle, a specific horizontal reduction, or bit-manipulation on packed data.

**Q9.5 — Why are gather and scatter slow?** Because they perform independent memory accesses under one instruction — a gather of eight elements is eight potential cache misses and eight TLB lookups, executed largely serially in the load unit. So a gather isn't 8× faster than eight loads; it's roughly the same, with better code density. Contiguous access is what makes SIMD pay.

**Q9.6 — What's the problem with horizontal operations?** Operations across lanes of one register — a horizontal sum, for example — are multi-instruction, high-latency, and don't pipeline well. The idiom is to keep the reduction vertical throughout the loop, accumulating into a vector register, and do exactly one horizontal fold at the end.

**Q9.7 — Any catch with AVX-512?** Yes — on several Intel generations, heavy use of 512-bit operations triggers a frequency reduction across the core, and there's a transition penalty entering and leaving. So a small AVX-512 kernel can slow down surrounding scalar code. Many low-latency shops cap at AVX2 for that reason; newer parts have improved it substantially.

**Q9.8 — Does SIMD matter for a latency-sensitive path, or only throughput?** Both, but differently. It matters for latency when a single message requires a fixed chunk of work — parsing a fixed-width field, checksumming, comparing keys — where doing it in one instruction shortens the critical path. It matters less when the hot path is one small object at a time, where you're bound by memory and branches, not arithmetic.

---

## 10. Alignment

**Q10.1 — What is natural alignment?** An object of size N sits at an address that's a multiple of N — 4-byte ints at multiples of 4, 8-byte doubles at multiples of 8. Hardware is built around this: it means an access never crosses a boundary the memory subsystem cares about. The compiler enforces it automatically for you, including by padding structs.

**Q10.2 — What does a misaligned access actually cost on x86?** For ordinary loads and stores, usually nothing — if it stays within one cache line. The penalty arrives when it straddles a boundary: a cache-line split costs a few extra cycles because two lines are accessed, and a 4-kilobyte page split is much worse, on the order of tens of cycles, because it's two translations and potentially two faults.

**Q10.3 — And on other architectures?** Historically a fault. Older ARM and SPARC trap on misaligned access, and even where the hardware allows it, doing it in C++ through a reinterpreted pointer is undefined behavior regardless of platform. That's why the portable way to read a value out of a byte buffer is `memcpy` into a properly-typed local, which compiles to a single load anyway.

**Q10.4 — `alignof` versus `alignas`?** `alignof` queries the required alignment of a type; `alignas` imposes a stricter one on a variable or a type. You can only strengthen, never weaken. The everyday use is `alignas(64)` for cache-line isolation, and `alignas(32)` for a buffer you want to load with AVX.

**Q10.5 — How does alignment interact with `new`?** Before C++17, `operator new` only guaranteed `max_align_t` — typically 16 bytes — so heap-allocating an `alignas(64)` type was silently unsupported. C++17 added aligned `new` overloads that take an `align_val_t`, so over-aligned types now work correctly on the heap. It's worth knowing because a lot of older codebases still use `posix_memalign` or a hand-rolled aligned allocator.

**Q10.6 — Why does struct member ordering change `sizeof`?** Because the compiler inserts padding so each member lands on its natural alignment, and pads the whole struct up to its strictest member's alignment so arrays stay aligned. A char, then an int64, then a char is 24 bytes; reordering to int64, char, char is 16. Declaring largest to smallest is the rule of thumb.

**Q10.7 — What's wrong with `#pragma pack`?** It removes padding, which shrinks the struct but produces misaligned members — so every access risks a split, and taking a reference or pointer to a misaligned member is undefined behavior. It's justified for wire and file formats where the layout is dictated externally, and the safer pattern there is a byte buffer plus `memcpy` accessors.

**Q10.8 — Why must an atomic be naturally aligned?** Because the lock-free guarantee comes from the hardware performing the operation on a single cache line atomically. An object straddling two lines can't be updated atomically by a single instruction — historically x86 handled it with a full bus lock, which is disastrously slow, and it's not guaranteed at all. So `std::atomic` over-aligns its storage, and a misaligned 64-bit atomic would silently fall back to a lock.

---

## 11. NUMA

**Q11.1 — What is NUMA?** Non-uniform memory access: on a multi-socket machine each socket has its own memory controller and attached DRAM, so accessing your local node is fast and reaching another socket's memory goes over the interconnect. Local is roughly 80 nanoseconds, remote roughly 130 to 150 — call it 1.5 to 2×, plus lower bandwidth.

**Q11.2 — What is the first-touch policy?** Linux allocates a physical page on the node of the thread that first _writes_ to it, not the one that called `malloc`. So if a main thread allocates and zeroes a large buffer and then worker threads on other sockets use it, every access is remote. The fix is to have each worker touch its own portion first, in parallel.

**Q11.3 — How do you control NUMA placement?** `numactl` at launch — `--cpunodebind` and `--membind` — or programmatically with `libnuma` and `mbind`. In practice a low-latency process pins each thread to a specific core with `sched_setaffinity`, allocates its data from that core's node, and keeps the NIC on the same socket the handling thread runs on.

**Q11.4 — What does NUMA imply for a thread pool?** That a work-stealing pool with a shared queue is quietly hostile to it: a task can migrate to a thread on another socket and then touch memory allocated on the original one. The NUMA-aware design is per-node pools with node-local queues and allocation, stealing only within a node as a first preference.

**Q11.5 — Simplest way to avoid the whole problem?** Run on one socket. Pin the process and all its memory to a single NUMA node, and if you need the other socket, run a second independent process there. A lot of trading systems do exactly this — the interconnect variance isn't worth the extra cores.

---

## 12. Atomics on the hardware

**Q12.1 — What does an atomic increment compile to, and what does the hardware do?** On x86, `lock xadd`. The `lock` prefix isn't a bus lock on any modern CPU — the core acquires the cache line in Modified state through the coherence protocol and holds it for the duration of the read-modify-write, so no other core can observe or alter it mid-operation. Cache-line locking, not bus locking.

**Q12.2 — What does an uncontended atomic RMW cost?** About 15 to 20 cycles, call it 5 nanoseconds, when the line is already in your L1 in Modified state. That's more than the roughly 1 cycle of a plain increment, because it can't be reordered or buffered in the store buffer — it has to complete at the cache. But it's cheap in absolute terms.

**Q12.3 — And contended?** Now the line has to migrate. Every operation is a coherence transaction pulling the line from whichever core last wrote it — so you're at 40 to 100 nanoseconds per operation same-socket, more across sockets, and it gets worse as you add cores because they all queue for the same line. Ten threads on one counter can be slower than one thread.

**Q12.4 — Why is `memory_order_relaxed` free on x86 for loads and stores?** Because x86 is already strongly ordered — it's TSO, total store order. Loads have acquire semantics and stores have release semantics in hardware, so a relaxed, acquire, or release load all compile to a plain `mov`. The memory order argument is mostly instructions to the _compiler_ about what it may reorder. On ARM it genuinely changes the emitted instructions.

**Q12.5 — Then which memory order does cost something on x86?** Sequential consistency, on the store side. A `seq_cst` store must prevent the one reordering x86 allows — a later load passing an earlier store — so the compiler emits `xchg`, or a `mov` followed by `mfence`. That's a store-buffer drain, on the order of 20 to 30 cycles. It's why `release` versus `seq_cst` on a publishing store is a real, measurable choice.

**Q12.6 — Why is a CAS loop expensive under contention beyond the instruction cost?** Because failure is wasted work at coherence speed. Each attempt acquires the line exclusively, recomputes, and loses the race, then the line moves to the winner and back. Throughput can degrade as threads are added. `fetch_add` is strictly better where it applies, because it always succeeds — the hardware does the retry internally without a round trip.

**Q12.7 — What does `is_lock_free` really tell you?** Whether operations on that type use a hardware instruction rather than a hidden lock. It's true for types up to the natural word size — and for 16 bytes on x86-64 with `cmpxchg16b`, though `std::atomic<T>` over a 16-byte struct historically fell back to a mutex in libstdc++ unless you enabled the instruction. Anything larger is always locked. `is_always_lock_free` is the compile-time version.

**Q12.8 — How does ARM differ mechanically?** ARM uses load-linked / store-conditional — `ldxr` and `stxr` — rather than a single locked instruction. You load with a reservation, and the store fails if anything touched the line in between, so you loop. That means even a `fetch_add` is a retry loop at the hardware level, and ARM's weak memory model makes acquire and release genuinely emit barrier instructions.

**Q12.9 — Sketch the hardware story of a spinlock.** `lock`'s fast path is one atomic exchange or CAS on a flag — 20 cycles uncontended. On failure you spin, but you spin on a _plain load_ — test, then test-and-set — because spinning on the atomic keeps the line in Modified state and starves the holder. Add a `pause` instruction in the loop to reduce power and avoid a memory-order-violation flush on exit.

**Q12.10 — So when is a mutex better than spinning?** When the wait is longer than the cost of sleeping. Spinning burns a core and wins only if the critical section is shorter than a context switch — sub-microsecond. Beyond that the futex path is cheaper in aggregate. The framing I'd give is that a mutex _is_ built out of an atomic; the question is never atomic versus mutex, it's what you do when you lose the race — pay ~100 cycles for a cache line, or thousands for the scheduler plus a cold cache.

**Q12.11 — Why does an atomic counter shared by every thread scale so badly?** Because it's true sharing — one line, every core writing it, so the line serializes the whole system at coherence speed. Padding doesn't help. The fix is structural: per-thread counters, each on its own cache line, summed on read. That converts a contended line into N uncontended ones and it scales linearly.

---

## Rapid-fire drill card

Cover the right column and go down the list out loud. Target: answer inside three seconds.

|Prompt|Answer|
|---|---|
|Cache line|64 bytes|
|L1 hit|~4 cycles / ~1.5 ns|
|L2 hit|~12 cycles / ~4 ns|
|L3 hit|~40 cycles / ~15 ns|
|DRAM|~200–300 cycles / ~100 ns|
|Branch mispredict|~15–20 cycles|
|Uncontended atomic RMW|~20 cycles|
|Contended cache line transfer|~40–100 ns|
|Remote NUMA penalty|~1.5–2×|
|Huge page size|2 MB (and 1 GB)|
|Cycles → ns at 3 GHz|÷3|
|Fix for false sharing|`alignas(64)`|
|Fix for TLB thrash|huge pages|
|Fix for pointer-chasing|contiguous layout|
|Fix for unpredictable branch|branchless / mask|
|Fix for one-field-of-many access|SoA|
|Fix for FP reduction chain|multiple accumulators|
|Fix for remote memory|first-touch on the using thread|

---

## Three things to say when you don't know

1. **Bound it instead of guessing.** "I don't know the exact figure, but it has to be between a cache hit and a DRAM access, so tens of nanoseconds." An interviewer will take a correct order of magnitude with honest framing over a confident wrong number every time.
2. **Name the measurement.** "I'd check that with `perf c2c`" or "I'd look at IPC and the miss rate together" turns a gap into a demonstration of method.
3. **State the trade-off you do know.** Most architecture questions are trade-offs wearing a fact's clothing. If you can't recall whether L3 is inclusive on a given part, you can still say what inclusivity buys and costs.

---

## Where this list is deliberately shallow

Worth knowing that these exist, in case an interviewer pulls the thread — you are not expected to go deep at graduate level, but "I know the name and roughly what it's for" beats silence.

- **Speculative-execution side channels** (Spectre, Meltdown) and their mitigation cost — relevant because retpolines and KPTI are why syscalls and indirect branches got more expensive.
- **Memory controller and DRAM internals** — row buffers, bank conflicts, `tRCD`/`tCAS`. Explains why sequential DRAM access beats random by more than the cache story alone predicts.
- **Uncore and mesh interconnect** — the on-die network connecting cores, L3 slices, and memory controllers; explains why L3 latency varies with which slice holds your line.
- **Cache allocation technology** — partitioning L3 between processes so a noisy neighbour can't evict your hot data.
- **Micro-op cache / loop stream detector** — the front end can bypass decode for small hot loops, which is one reason loop body size has a cliff.