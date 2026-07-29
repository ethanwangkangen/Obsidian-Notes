# C++ Concurrency — Comprehensive Cheatsheet

Fully self-contained: machinery + all implementations + answered self-test. Ordered simple → hard. ⚡ = HFT-relevant.

---

# PART A — MACHINERY

## A1. Threads

```cpp
#include <thread>
std::thread t(work, 42);          // starts IMMEDIATELY on construction
t.join();                          // block until done
```

- A `std::thread` still joinable at destruction → `std::terminate`. Join or detach exactly once. C++20 `std::jthread` auto-joins in its destructor and adds `std::stop_token` cooperative cancellation:

```cpp
std::jthread jt([](std::stop_token st){ while (!st.stop_requested()) step(); });
// jt.request_stop() or destructor requests + joins
```

- Arguments are **copied/moved** into the thread, even if the callee takes `T&`. Real reference: `std::thread t(work, std::ref(x))`. Dangling classic: passing pointer/ref to a local that dies first.
- `std::thread` is move-only (resource handle, like unique_ptr).
- No return values — use futures (A8) or shared state.
- ⚡ Thread creation ~10–100 μs. Trading systems: create at startup, pin to cores (`pthread_setaffinity_np`), never create on the hot path.

## A2. Mutexes and lock wrappers

|Type|Adds|Note|
|---|---|---|
|`std::mutex`|—|default|
|`std::recursive_mutex`|same-thread relock|design smell — say so|
|`std::timed_mutex`|try_lock_for/until|rare|
|`std::shared_mutex`|many readers OR one writer|read-mostly data|

Never manual `.lock()/.unlock()` (exception between them leaks the lock). Wrappers:

```cpp
{ std::lock_guard lk(m); }              // simplest RAII, zero overhead, no early unlock
{ std::unique_lock lk(m); }             // movable, unlock()/lock() midway, REQUIRED for cv.wait
{ std::scoped_lock lk(m1, m2); }        // multiple mutexes, deadlock-free (C++17)
{ std::shared_lock lk(sm); }            // reader side of shared_mutex
```

Decision rule: lock_guard by default → unique_lock when CV or early unlock → scoped_lock for ≥2 mutexes.

**Deadlock**: T1 locks A then B; T2 locks B then A → circular wait. The four Coffman conditions (all must hold): mutual exclusion, hold-and-wait, no preemption, circular wait — break any one to prevent. Fixes: global lock ordering; `std::scoped_lock`/`std::lock` (try-and-back-off algorithm); hold at most one lock.

⚡ Costs: uncontended lock ≈ one atomic RMW ≈ 20 ns. Contended = kernel futex sleep/wake ≈ 1–10 μs + cold caches. Linux `std::mutex` is a hybrid: brief adaptive spin, then futex.

## A3. Condition variables — cv.wait done right

Three-piece kit, always: mutex + CV + **predicate over shared state**.

```cpp
std::mutex m; std::condition_variable cv; bool ready = false;

// waiter                                       // notifier
std::unique_lock lk(m);                         { std::lock_guard lk(m); ready = true; }
cv.wait(lk, []{ return ready; });               cv.notify_one();
// lock held AND ready == true here
```

`cv.wait(lk, pred)` desugars to:

```cpp
while (!pred())
    wait(lk);          // atomically: unlock, sleep; on wake: relock, re-loop
```

Two failure modes the loop defends against — know cold:

1. **Spurious wakeup**: OS may wake a waiter with no notify (and notify_one may wake several). Without the loop you proceed on a false condition. Predicate re-check under the lock makes wakeups idempotent.
2. **Lost wakeup**: if the notifier writes the flag _without the mutex_: waiter checks pred (false) → notifier sets flag + notifies (nobody asleep yet) → waiter sleeps forever. Holding the mutex for the write makes [check-then-sleep] atomic w.r.t. [set-then-notify]. This is why "it's just a bool" never excuses skipping the lock — not torn writes, the check/sleep race.

Details:

- notify_one when any single waiter can proceed; notify_all when the change may satisfy multiple/different predicates. Wrong choices: notify_one + heterogeneous waiters → wrong thread woken → deadlock; notify_all everywhere → thundering herd.
- Notify inside vs outside the lock: both correct; outside avoids waking a thread that instantly blocks on the held mutex.
- `wait_for/wait_until` return whether pred held or timed out — check it.
- `condition_variable` needs `unique_lock<mutex>`; `condition_variable_any` takes any lockable (slight cost).

## A4. Atomics and CAS

`std::atomic<T>`: (a) indivisible loads/stores/RMWs — no torn values, no data-race UB; (b) ordering contract (A5).

```cpp
std::atomic<int> x{0};
x.store(5);  int v = x.load();          // default order: seq_cst
int old = x.fetch_add(1);               // RMW, returns PREVIOUS
x.exchange(7);
```

- `x++` on atomic IS atomic. On plain int it's load→inc→store: both threads can load the same value, both store the same result, one increment lost — **even on x86**; strong ordering ≠ atomic RMW.
- `is_lock_free()` / `is_always_lock_free`: atomics on large structs silently use a hidden lock. `atomic<int>`, `atomic<T*>` lock-free everywhere relevant.
- `std::atomic_flag`: the only guaranteed-lock-free type; test_and_set/clear (+C++20 test, wait/notify). Exists to build spinlocks (B2).

**CAS** — the primitive under all lock-free code:

```cpp
bool compare_exchange_strong(T& expected, T desired);
// atomic: if (x == expected) { x = desired; return true; }
//         else { expected = x; return false; }        <- reloads on failure
```

Canonical loop — failure-reload means no separate re-read:

```cpp
int cur = x.load();
while (!x.compare_exchange_weak(cur, cur * 2)) ;      // cur already refreshed
```

- `weak` can fail **spuriously**: on LL/SC ISAs (ARM/POWER), the store-conditional aborts if anything disturbs the cache line (or a context switch lands) between load-linked and store-conditional — even when the value matches. x86 `cmpxchg` never fails spuriously; `weak` exists for portability + skipping strong's internal retry loop. Rule: weak inside loops, strong for one-shot attempts.

## A5. Memory ordering — the full story

Compilers and CPUs reorder memory operations; single-threaded code can't tell, cross-thread code can. Orderings constrain this, per _operation_.

**`relaxed`** — atomicity only, zero ordering. Surrounding ops move freely past it; threads may disagree on interleavings. Use: statistics counters where only the final total matters; your own index in SPSC (sole writer).

**`release` / `acquire`** — the workhorse pair:

- `store(v, release)`: nothing sequenced-before the store may reorder after it → it "publishes" everything above.
- `load(acquire)`: nothing sequenced-after the load may reorder before it.
- When an acquire load **reads the value written by** a release store → the store _synchronizes-with_ the load → everything the writer did before release _happens-before_ everything the reader does after acquire → the reader can safely touch **non-atomic** data the writer prepared.

```cpp
int payload;  std::atomic<bool> flag{false};          // payload NOT atomic

// writer                                             // reader
payload = 42;                                         while (!flag.load(std::memory_order_acquire));
flag.store(true, std::memory_order_release);          use(payload);   // guaranteed 42
```

Both relaxed → no synchronizes-with edge → reading payload is a data race → UB.

**`seq_cst`** (default) — acquire+release PLUS one global order all threads agree on across ALL seq_cst ops. Needed only when correctness depends on total order across _multiple_ atomics (Dekker-style "set mine, check yours" mutual exclusion — with acquire/release both threads can fail to see each other's store; with seq_cst at least one must). Cost: full fence on the store (x86: `mfence` or `xchg`).

**Choosing, and how to narrate it in an interview:** start seq_cst (correct by default) → demote to release/acquire where you can name the publish→consume relationship → relaxed only where provably nothing else depends on the order.

⚡ **x86 probe**: x86 is TSO — normal loads/stores already have acquire/release semantics, so release/acquire compiles to plain MOV (free); only seq_cst stores pay a fence. Why write orderings at all? (1) they constrain the **compiler**, which reorders on any ISA; (2) portability — on ARM, relaxed vs acquire is a real hardware difference (ldar/stlr vs ldr/str).

`std::atomic_thread_fence(order)`: standalone fence not attached to one op — orders everything across it. Know it exists; prefer per-op orderings.

## A6. False sharing ⚡

Coherence operates on **cache lines** (64 B). Two threads writing _different_ variables on the _same_ line → the line ping-pongs between cores (MESI: each write needs the line Exclusive/Modified, invalidating the other core's copy) → performance craters with zero logical contention.

```cpp
struct Counters {
    alignas(64) std::atomic<uint64_t> produced;   // one line each
    alignas(64) std::atomic<uint64_t> consumed;
};
```

- Portable spelling: `alignas(std::hardware_destructive_interference_size)` (usually 64; sometimes 128 for adjacent-line prefetchers).
- Hidden case: `std::atomic<int> perThread[N]` — 16 counters per line.
- Detection: `perf c2c`.
- Side effect: alignas(64) forces sizeof to a multiple of 64 (size % alignment == 0).

## A7. Spinlock vs mutex — precise terminology

- **Mutex = blocking**: contended acquire parks the thread in the kernel (futex wait); unlock wakes it. μs-scale + cold caches, but the core does other work.
- **Spinlock = busy-waiting**: loop on an atomic until free. Burns the core; post-release acquisition is ns-scale.

Spin when: critical sections tiny (<~1 μs), threads pinned to dedicated cores with nothing else to run, holders can't be preempted. That's the HFT profile ⚡. Mutex when: long sections, shared cores, or preemptible holders — spinning on a preempted holder wastes whole timeslices (why kernels pair spinlocks with preemption-off).

## A8. Futures, promises, async

```cpp
std::future<int> f = std::async(std::launch::async, compute);   // ALWAYS pass the policy
int r = f.get();                                // blocks; RETHROWS compute's exception

std::packaged_task<int()> task(compute);        // callable that fills a future
auto f2 = task.get_future();
std::thread(std::move(task)).detach();

std::promise<int> p; auto f3 = p.get_future();  // manual: p.set_value(42) / p.set_exception(...)
```

Traps:

- Default policy is `async | deferred`: implementation may choose deferred → runs lazily inside `.get()` on the calling thread — or **never**, if get is never called. Always write `std::launch::async`.
- **Destructor trap**: a future obtained from `std::async` (only those) **blocks in its destructor** until the task finishes. So `std::async(std::launch::async, f);` with the result discarded is fully synchronous.
- `get()` is one-shot (moves the value); multiple consumers → `shared_future`.
- ⚡ Allocation + sync inside — startup orchestration yes, per-message hot path no.

## A9. One-time / lazy init

```cpp
Widget& instance() { static Widget w; return w; }   // "magic static" — thread-safe since C++11
```

Concurrent first callers: one constructs, others **block** until done. Compiler emits a guard variable + double-checked locking (`__cxa_guard_acquire/release` in the Itanium ABI). This is the Meyers singleton.

```cpp
std::once_flag flag;
std::call_once(flag, []{ init(); });                // same guarantee, arbitrary init
```

Why hand-rolled double-checked locking was famously broken pre-C++11: no memory model — the unsynchronized fast-path read could observe the pointer store before the object's construction stores (compiler/CPU reordering), handing out a pointer to an unconstructed object. Modern rule: never hand-write DCLP — magic static or call_once; if forced, atomic pointer with release store / acquire load (which is just DCLP done with real orderings).

Related: `constinit` forces compile-time (constant) initialization of statics — eliminates the static-initialization-order fiasco (SIOF) for that variable and any runtime init cost, without making it const (that's the difference from constexpr variables).

## A10. thread_local

```cpp
thread_local std::mt19937 rng{std::random_device{}()};   // one PER THREAD, lazy-constructed on first use
```

Per-thread instance, destroyed at thread exit. Uses: per-thread RNGs, caches, scratch buffers, error state (errno is effectively this). ⚡ The zero-synchronization answer to "thread-safe counter": each thread increments its own, aggregate at the end — no atomics, no false sharing. Cost: one TLS indirection (fs-segment addressing on x86-64), near-free. Traps: lazy per-thread init surprises in pools (first-touch per worker); messy teardown order with dlopen'd libraries.

## A11. Hazards vocabulary (rapid definitions)

- **Data race**: same location, two threads, ≥1 write, no happens-before → **UB**. Distinct from a benign "race condition" (timing-dependent logic bug, not UB per se).
- **Deadlock**: circular wait (A2, Coffman).
- **Livelock**: threads keep reacting to each other, no progress (corridor dance). Fix: randomized backoff.
- **Starvation**: a thread never wins a repeatedly-free resource (e.g., writers starved by reader stream on an RW lock without writer preference).
- **Priority inversion**: low-prio holds a lock high-prio needs; medium-prio preempts the holder. Fix: priority inheritance (PI futexes). ⚡ Relevant to pinned/prioritized thread designs.
- **ABA**: CAS reads A, location went A→B→A meanwhile — compare passes, world changed. Classic: lock-free stack pop where the popped node was freed and its address reused. Fixes: tagged pointers (generation counter packed alongside), hazard pointers, epoch-based reclamation. Know the problem cold; name fixes without implementing.
- **Torn read/write**: non-atomic multi-part access observed half-updated. What atomic rules out per-variable.

---

# PART B — IMPLEMENTATIONS (all of them, in full)

Difficulty order. Each: code + the probe points interviewers hit.

## B1. Spinlock (TAS → TTAS)

```cpp
class Spinlock {
    std::atomic_flag f_ = ATOMIC_FLAG_INIT;
public:
    void lock() {
        while (f_.test_and_set(std::memory_order_acquire)) {   // TAS attempt
            while (f_.test(std::memory_order_relaxed))          // TTAS: read-only spin (C++20 test())
                /* _mm_pause(); */;
        }
    }
    void unlock() { f_.clear(std::memory_order_release); }
    bool try_lock() { return !f_.test_and_set(std::memory_order_acquire); }
};
```

**Probes:** acquire-on-lock/release-on-unlock keeps the critical section's operations inside it (same publish handshake as A5). **TAS vs TTAS**: test_and_set is an RMW → needs the cache line Exclusive every attempt → under contention, N spinners generate constant coherence traffic. The inner read-only loop spins on a **Shared** cached copy — zero traffic — and only retries the RMW when the line actually changes. `_mm_pause()` hints the CPU (saves power, helps hyperthread sibling, avoids memory-order mis-speculation flush on exit). Pre-C++20 (no `test()`): use `std::atomic<bool>` with load+exchange instead.

## B2. Counting Semaphore from mutex + CV

```cpp
class Semaphore {
    std::mutex m_; 
    std::condition_variable cv_; 
    int count_;
public:
    explicit Semaphore(int initial) :
     count_(initial) 
    {}
    void acquire() { 
	    std::unique_lock lk(m_); 
	    cv_.wait(lk, [&]{ return count_ > 0; }); 
		--count_; 
	}
    void release() {
	    { 
		     std::lock_guard lk(m_); 
		     ++count_; 
	     } 
	     cv_.notify_one(); 
	}
};
```

C++20 has `std::counting_semaphore` — mention, then implement anyway. Binary semaphore (init 1) ≈ a mutex without ownership (any thread may release — that's the semantic difference from mutex, and an interview probe). Primitive-conversion questions ("build X given only Y") all reduce to this skeleton with a different predicate.

## B3. Latch and Barrier

```cpp
class Latch {                                   // one-shot countdown (std::latch in C++20)
    std::mutex m_; std::condition_variable cv_; int count_;
public:
    explicit Latch(int n) : count_(n) {}
    void count_down() { 
	    std::lock_guard lk(m_); 
	    if (--count_ == 0) cv_.notify_all(); 
	}
    void wait() { 
	    std::unique_lock lk(m_); 
	    cv_.wait(lk, [&]{ return count_ == 0; }); 
	}
};

class Barrier {                                 // REUSABLE rendezvous (std::barrier in C++20)
    std::mutex m_; std::condition_variable cv_;
    int n_, count_; uint64_t gen_ = 0;          // generation counter — THE interview point
public:
    explicit Barrier(int n) : n_(n), count_(n) {}
    void arrive_and_wait() {
        std::unique_lock lk(m_);
        uint64_t g = gen_;
        if (--count_ == 0) { 
	        ++gen_; 
	        count_ = n_; 
	        cv_.notify_all(); 
	        return;
	    }
        cv_.wait(lk, [&]{ return gen_ != g; });
    }
};
```

**Why the generation counter:** with a naive `count_ == 0` predicate, a fast thread can loop around and hit the barrier again _before_ slow threads wake — resetting count and trapping them in the old cycle. Waiting on "generation changed" instead of "count is zero" makes each cycle's wakeup condition distinct. This bug is the entire reason the question gets asked.

## B4. Reader-Writer Lock (writer-preference)

```cpp
class RWLock {
    std::mutex m_; std::condition_variable cv_;
    int readers_ = 0, waitingWriters_ = 0; bool writer_ = false;
public:
    void lock_shared() {
        std::unique_lock lk(m_);
        cv_.wait(lk, [&]{ return !writer_ && waitingWriters_ == 0; });  // writers preferred
        ++readers_;
    }
    void unlock_shared() {
        std::lock_guard lk(m_);
        if (--readers_ == 0) cv_.notify_all();
    }
    void lock() {
        std::unique_lock lk(m_);
        ++waitingWriters_;
        cv_.wait(lk, [&]{ return !writer_ && readers_ == 0; });
        --waitingWriters_; writer_ = true;
    }
    void unlock() {
        std::lock_guard lk(m_);
        writer_ = false;
        cv_.notify_all();               // heterogeneous waiters -> must be notify_all
    }
};
```

**Probes:** without `waitingWriters_` in the reader predicate, a steady reader stream starves writers forever (A11 starvation). Why notify_all: waiters are heterogeneous (readers and writers, different predicates) — notify_one may wake the un-runnable kind and deadlock. Production: `std::shared_mutex`; ⚡ read-mostly hot data often prefers seqlocks or RCU instead (name-drop level).

## B5. Bounded Blocking Queue (producer/consumer)

```cpp
template <typename T>
class BlockingQueue {
    std::mutex m_; std::condition_variable notFull_, notEmpty_;
    std::queue<T> q_; size_t cap_;
public:
    explicit BlockingQueue(size_t cap) : cap_(cap) {}
    void push(T v) {
        std::unique_lock lk(m_);
        notFull_.wait(lk, [&]{ return q_.size() < cap_; });
        q_.push(std::move(v));
        notEmpty_.notify_one();
    }
    T pop() {
        std::unique_lock lk(m_);
        notEmpty_.wait(lk, [&]{ return !q_.empty(); });
        T v = std::move(q_.front()); q_.pop();
        notFull_.notify_one();
        return v;
    }
};
```

**Probes:** why **two** CVs — one CV + notify_one can wake a same-side waiter (producer wakes producer) while the opposite side sleeps → deadlock; one CV forces notify_all → thundering herd. Two CVs make every notify wake exactly the right kind. Extensions: `try_pop`, `pop_for(timeout)` (use wait_for's return), shutdown flag folded into both predicates (`return stop_ || ...`, then check which fired). ⚡ This is the general MPMC tool; B7 removes the lock for the SPSC special case.

## B6. Thread Pool ⭐

```cpp
class ThreadPool {
    std::vector<std::thread> workers_;
    std::queue<std::function<void()>> tasks_;
    std::mutex m_; std::condition_variable cv_;
    bool stop_ = false;
public:
    explicit ThreadPool(size_t n) {
        for (size_t i = 0; i < n; ++i)
            workers_.emplace_back([this] {
                while (true) {
                    std::function<void()> task;
                    {
                        std::unique_lock lk(m_);
                        cv_.wait(lk, [&]{ return stop_ || !tasks_.empty(); });
                        if (stop_ && tasks_.empty()) return;     // drain, THEN exit
                        task = std::move(tasks_.front()); tasks_.pop();
                    }                                            // unlock BEFORE running
                    task();
                }
            });
    }
    void submit(std::function<void()> f) {
        { std::lock_guard lk(m_); tasks_.push(std::move(f)); }
        cv_.notify_one();
    }
    ~ThreadPool() {
        { std::lock_guard lk(m_); stop_ = true; }
        cv_.notify_all();
        for (auto& w : workers_) w.join();
    }
};
```

**The four probe points:** (1) wait predicate `stop_ || !empty` but exit condition `stop_ && empty` — shutdown must wake sleepers, yet queued work drains first. (2) `task()` outside the lock — inside, the pool serializes to one thread. (3) `stop_` written under the mutex despite being "just a bool" — the lost-wakeup race (A3). (4) Extensions: return `std::future` via `packaged_task` (wrap, get_future, push the task); per-thread queues + work stealing for ⚡; exception policy (swallow? std::terminate propagates from a worker otherwise — wrap task() in try/catch).

Future-returning submit, since it's the most-asked extension:

```cpp
template <typename F>
auto submit2(F f) -> std::future<decltype(f())> {
    auto pt = std::make_shared<std::packaged_task<decltype(f())()>>(std::move(f));
    auto fut = pt->get_future();
    { std::lock_guard lk(m_); tasks_.push([pt]{ (*pt)(); }); }
    cv_.notify_one();
    return fut;
}
```

(shared_ptr because packaged_task is move-only and std::function requires copyable — itself a known trap; C++23 move_only_function fixes it.)

## B7. Lock-free SPSC Ring ⚡ (the crown jewel — code + full narration)

```cpp
template <typename T>
class SpscQueue {
    std::vector<T> buf_;
    size_t mask_;
    alignas(64) std::atomic<size_t> head_{0};   // consumer-owned
    alignas(64) std::atomic<size_t> tail_{0};   // producer-owned
public:
    explicit SpscQueue(size_t capPow2) : buf_(capPow2), mask_(capPow2 - 1) {
        assert(capPow2 && (capPow2 & mask_) == 0);
    }
    bool push(T v) {                                              // producer thread only
        size_t t = tail_.load(std::memory_order_relaxed);         // own index
        if (t - head_.load(std::memory_order_acquire) == buf_.size()) return false;  // full
        buf_[t & mask_] = std::move(v);
        tail_.store(t + 1, std::memory_order_release);            // publish
        return true;
    }
    bool pop(T& out) {                                            // consumer thread only
        size_t h = head_.load(std::memory_order_relaxed);
        if (h == tail_.load(std::memory_order_acquire)) return false;                // empty
        out = std::move(buf_[h & mask_]);
        head_.store(h + 1, std::memory_order_release);
        return true;
    }
};
```

**The 8-step interview narration (rehearse out loud):**

1. SPSC is tractable where MPMC isn't because each index has exactly one writer — head_ only the consumer, tail_ only the producer → no CAS anywhere, plain loads/stores with orderings suffice. (MPMC needs CAS loops, per-slot generation counters, ABA defenses — "Vyukov MPMC" is the name-drop, don't implement.)
2. Producer publish edge: element write first, then `tail_.store(t+1, release)` — release forbids the element write sinking below the index bump.
3. Consumer consume edge: `tail_.load(acquire)` seeing t+1 forbids the element read hoisting above it → sees complete element. Exactly A5's publish pattern with the index as the flag.
4. Own-index loads are relaxed: sole writer reading its own last write needs no ordering.
5. Cross-index loads are acquire, pairing with the other side's release store: the full/empty check must both see a sufficiently-fresh index and order the element access.
6. Monotonic indices, mask on access: full = `tail−head == cap`, empty = `head == tail`; no sacrificed slot; unsigned wraparound is well-defined.
7. alignas(64) per index: false sharing (A6) — otherwise every producer store invalidates the consumer's line and vice versa.
8. What breaks all-relaxed: consumer can observe the new tail but stale element bytes → garbage read. Real on ARM, latent on x86 (compiler can still break you).

**Extensions when the interview goes long:** batched pop (drain up to N per acquire, amortizing sync — pairs conceptually with recvmmsg-style batching); **cached remote index** — producer caches last-seen head_ and only re-loads on apparent-full (rigtorp / folly::ProducerConsumerQueue optimization; cuts coherence traffic massively since the common case never touches the other side's line); for non-trivially-destructible T, placement-new/destroy per slot instead of assignment.

## B8. Meyers Singleton + hand-rolled DCLP (for contrast)

```cpp
// The right way — magic static (A9):
Widget& instance() { static Widget w; return w; }

// DCLP with real orderings — only to show you COULD:
std::atomic<Widget*> ptr{nullptr};
std::mutex initM;
Widget* get() {
    Widget* p = ptr.load(std::memory_order_acquire);      // fast path, no lock
    if (!p) {
        std::lock_guard lk(initM);
        p = ptr.load(std::memory_order_relaxed);           // re-check under lock
        if (!p) { p = new Widget(); ptr.store(p, std::memory_order_release); }
    }
    return p;
}
```

Why the acquire/release pair is load-bearing: release ensures the constructor's stores complete before the pointer publishes; acquire ensures a fast-path reader's uses don't hoist above the pointer load. Drop them (pre-C++11 style with a plain pointer) → a reader can get a pointer to an unconstructed object. In an interview: write the magic static, explain DCLP verbally, only write it if pushed.

---

# PART C — SELF-TEST, WITH ANSWERS

Cover the answer column; say it out loud; check.

**1. Why must the CV notifier hold the mutex when writing the flag, even a bool?** Lost wakeup: waiter checks pred (false) → notifier sets flag + notifies while nobody sleeps → waiter sleeps forever. The mutex makes check-then-sleep atomic against set-then-notify. Torn writes are irrelevant; it's the check/sleep race.

**2. What does cv.wait(lk, pred) desugar to, and what two bugs does the loop kill?** `while (!pred()) wait(lk);` where wait atomically unlocks+sleeps and relocks on wake. Kills: spurious wakeups (proceed on false condition) and over-notification (notify_one waking multiple / wrong waiters).

**3. Where exactly does plain `int` counter++ lose updates across two threads?** It's load→inc→store. Both threads load the same value v, both compute v+1, both store v+1 → one increment vanishes. x86's strong ordering orders visible operations; it does not fuse the three into one atomic RMW. atomic<int>++ emits `lock inc` — the lock prefix makes the RMW indivisible.

**4. When does acquire synchronize-with release, and what does it buy for non-atomic data?** When the acquire load reads the value written by the release store (same atomic object). Then writer's pre-release operations happen-before reader's post-acquire operations → reader can access non-atomic data the writer prepared, without a data race.

**5. Why is release/acquire "free" on x86 but not ARM, and why write it anyway?** x86 is TSO — ordinary loads have acquire semantics, ordinary stores release; the orderings compile to plain MOVs, only seq_cst stores need a fence. Still write them because (1) the compiler reorders on every ISA and the orderings constrain it, (2) ARM genuinely distinguishes ldr/ldar.

**6. compare_exchange_weak vs strong — spurious-failure mechanism, which in loops?** LL/SC ISAs implement CAS as load-linked + store-conditional; the SC aborts if the cache line was disturbed or a context switch occurred in between, even with matching values → spurious failure. x86 cmpxchg can't fail spuriously. weak in retry loops (retrying anyway; avoids strong's internal loop), strong for single-shot.

**7. Why alignas(64) on SPSC indices — the MESI story?** Writes require the line in Modified/Exclusive; head_ and tail_ sharing a line means each side's write invalidates the other core's copy → the line ping-pongs on every operation despite no logical sharing. Separate lines → each side's index write stays local.

**8. Precisely define "blocking," and the pathological spinlock case?** Blocking = the thread is descheduled by the kernel (futex sleep) and rescheduled on wake. Pathology: spinning on a lock whose holder was preempted — the spinner burns its whole timeslice waiting for a thread that can't run (possibly the one it's blocking, on one core).

**9. Two dangers of std::async without a launch policy?** Default `async|deferred` may pick deferred → runs inside .get() on the caller's thread, or never runs if get is never called. And separately: async-launched futures block in their destructor, so a discarded async(...) call is synchronous.

**10. Why was hand-written DCLP broken pre-C++11; what replaced it?** No memory model: the pointer store could become visible before the object's construction stores (compiler/CPU reordering), so the unsynchronized fast-path read handed out an unconstructed object. Replaced by magic statics / call_once; correct manual form uses release store + acquire load.

**11. TAS vs TTAS — why does the read-only inner loop cut coherence traffic?** test_and_set is an RMW → demands the line Exclusive on every attempt → N spinners continuously steal the line. Read-only spinning keeps a Shared copy in each spinner's cache — no traffic — retrying the RMW only when the line actually changes (their copy gets invalidated).

**12. In SPSC, which load/store pairs carry release/acquire, and which are relaxed — and why?** Release: each side's store of its OWN index after touching the buffer (tail after write, head after read). Acquire: each side's load of the OTHER side's index (producer reads head for fullness, consumer reads tail for emptiness) — pairing with that side's release store to order buffer access. Relaxed: each side's load of its OWN index — sole writer, no one to synchronize with.

**13. Why two condition variables in a bounded queue?** Waiters are two disjoint populations with different predicates. One CV + notify_one can wake the same side (producer wakes producer while consumers sleep) → deadlock; one CV + notify_all works but thunders. Two CVs target every wake correctly.

**14. Barrier generation counter — what bug does it fix?** A fast thread finishing a cycle can re-arrive before slow threads wake; with a count-based predicate it would decrement the RESET count and corrupt the next cycle / strand old waiters. Waiting on "generation changed since I arrived" gives each cycle a distinct wake condition.

**15. Semaphore vs mutex — the semantic difference?** Mutex has ownership: the locker must unlock (and lock_guard enforces it); semaphore has no ownership — any thread may release, enabling signaling patterns (thread A acquires, thread B releases) that a mutex forbids.

**16. What is ABA and one fix?** CAS validates by value: location goes A→B→A between your read and CAS; compare succeeds but the state changed (classic: freed-and-reallocated node in a lock-free stack). Fix: tag the pointer with a generation counter (CAS on the pair), or hazard pointers / epoch reclamation to prevent reuse while referenced.

---

# Study sequencing

1. A4–A5 (atomics + ordering) → implement B1 and B7 blind, narrating B7's 8 steps out loud.
2. A3 (CVs) → implement B5 and B6 blind; B2–B4 are then ~15 minutes each.
3. A6–A7, A11 anytime — no prerequisites.
4. A8–A10 one evening.
5. Endgame: Part C aloud, answers covered, all 16.