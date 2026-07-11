# Combination Patterns — "Implement XX" Interview Structures (v2)

Expanded rewrite of Part 3 from the STL reference. Every entry: what triggers it, the design, working C++ code, complexity, and the follow-up questions interviewers actually ask. HFT-relevant notes marked ⚡.

**How to use this:** for each pattern, cover the code and re-derive it from the trigger + invariant. The invariant is the thing to memorize; the code falls out of it.

---

## Index

|#|Pattern|Trigger phrase|Core combo|
|---|---|---|---|
|1|LRU Cache|"evict least recently used"|hashmap → list iterators + doubly-linked list|
|2|LFU Cache|"evict least frequently used"|hashmap + freq-bucketed lists|
|3|Ring Buffer (SPSC)|"fixed-size queue, no allocation"|array + head/tail indices, power-of-2|
|4|Median Tracker (stream)|"median after each insert"|two heaps|
|5|Sliding Window Median|"median of last K"|multiset + median iterator (or lazy-deletion heaps)|
|6|ID Allocator|"smallest available ID, huge ID space"|recycled pool + frontier|
|7|Randomized Set|"O(1) insert/delete/getRandom"|vector + hashmap (swap-and-pop)|
|8|Min Stack / Min Queue|"getMin in O(1)"|pair stack / two min-stacks|
|9|Monotonic Deque|"sliding window max"|deque of candidates|
|10|Timer Queue|"schedule callbacks, cancel"|min-heap + lazy cancellation|
|11|Interval Set|"track covered ranges, coalesce"|std::map<start,end>|
|12|Time-based KV Store|"get value as of timestamp"|hashmap → sorted vector + upper_bound|
|13|Hit Counter / Rate Limiter|"count events in last N sec / throttle"|ring of buckets / token bucket|
|14|Object Pool|"reuse objects, no allocation in hot path"|vector storage + free list|
|15|Open-Addressing HashMap|"implement a hashmap"|vector of slots, linear probing, tombstones|
|16|Limit Order Book|"implement an order book"|map<price,level> + hashmap<id,iterator>|
|17|Union-Find|"connected components, merge groups"|parent array + rank + path compression|
|18|Trie|"prefix queries"|node with child array/map|
|19|Snapshot Array|"versioned reads"|per-index vector<(version,val)> + binary search|
|20|Bounded Blocking Queue|"producer/consumer with capacity"|mutex + 2 condition variables|

---

## 1. LRU Cache (LC 146)

**Trigger:** capacity-bounded cache, evict _least recently used_ on overflow, O(1) get/put.

**Invariant:** a doubly-linked list ordered by recency (front = most recent). A hashmap gives O(1) lookup from key → _position in the list_. Every access splices the node to the front; eviction pops the back.

Why this combo: hashmap alone can't order by recency; list alone can't look up in O(1). The trick that makes it work: `std::list` iterators are **stable** — never invalidated by splice/insert/erase of _other_ elements — so storing them in the map is safe.

```cpp
class LRUCache {
    int cap_;
    std::list<std::pair<int,int>> items_;                       // {key, value}, front = MRU
    std::unordered_map<int, std::list<std::pair<int,int>>::iterator> pos_;
public:
    explicit LRUCache(int capacity) : cap_(capacity) {}

    int get(int key) {
        auto it = pos_.find(key);
        if (it == pos_.end()) return -1;
        items_.splice(items_.begin(), items_, it->second);      // move to front, O(1), no realloc
        return it->second->second;
    }

    void put(int key, int value) {
        auto it = pos_.find(key);
        if (it != pos_.end()) {
            it->second->second = value;
            items_.splice(items_.begin(), items_, it->second);
            return;
        }
        if ((int)items_.size() == cap_) {
            pos_.erase(items_.back().first);
            items_.pop_back();
        }
        items_.emplace_front(key, value);
        pos_[key] = items_.begin();
    }
};
```

Complexity: O(1) get/put. Space O(capacity).

**Interviewer follow-ups:**

- _Why `splice` and not erase+insert?_ splice relinks pointers, no allocation, no iterator invalidation, and it's noexcept.
- _Why store the key inside the list node?_ So eviction (which only sees the back node) can delete the map entry.
- ⚡ _Make it faster?_ `std::list` = one heap node per element, cache-hostile. Real answer: intrusive doubly-linked list over a preallocated array (indices instead of pointers), i.e. combine with pattern #14 (Object Pool).

---

## 2. LFU Cache (LC 460)

**Trigger:** evict _least frequently_ used; ties broken by LRU. O(1) everything.

**Invariant:** buckets of equal frequency, each bucket an LRU list. Track `minFreq_`. On access, a key moves from freq-f bucket to freq-f+1 bucket. Eviction: pop LRU tail of the `minFreq_` bucket.

```cpp
class LFUCache {
    struct Node { int key, val, freq; };
    int cap_, minFreq_ = 0;
    std::unordered_map<int, std::list<Node>::iterator> pos_;
    std::unordered_map<int, std::list<Node>> buckets_;          // freq -> LRU list (front = MRU)

    void touch(std::list<Node>::iterator it) {
        int f = it->freq;
        buckets_[f + 1].splice(buckets_[f + 1].begin(), buckets_[f], it);
        it->freq = f + 1;
        if (buckets_[f].empty()) {
            buckets_.erase(f);
            if (minFreq_ == f) minFreq_ = f + 1;
        }
    }
public:
    explicit LFUCache(int capacity) : cap_(capacity) {}

    int get(int key) {
        auto it = pos_.find(key);
        if (it == pos_.end()) return -1;
        touch(it->second);
        return it->second->val;
    }

    void put(int key, int value) {
        if (cap_ == 0) return;
        auto it = pos_.find(key);
        if (it != pos_.end()) { it->second->val = value; touch(it->second); return; }
        if ((int)pos_.size() == cap_) {
            auto& lst = buckets_[minFreq_];
            pos_.erase(lst.back().key);
            lst.pop_back();
            if (lst.empty()) buckets_.erase(minFreq_);
        }
        buckets_[1].emplace_front(Node{key, value, 1});
        pos_[key] = buckets_[1].begin();
        minFreq_ = 1;
    }
};
```

Complexity: O(1) amortized. The subtle part interviewers probe: **why is resetting `minFreq_ = 1` on insert always correct?** Because the new key has frequency exactly 1, which is the global minimum by definition.

---

## 3. Ring Buffer / SPSC Queue

**Trigger:** fixed-capacity FIFO, zero allocation after construction, often "between two threads." ⚡ Near-guaranteed HFT question — this is the market-data → strategy handoff structure (your moldcast M2).

**Invariant:** array + monotonically increasing `head_` (read index) and `tail_` (write index); occupied count = `tail_ - head_`. Power-of-two capacity so `index & (cap-1)` replaces modulo (div is ~20+ cycles; and is 1).

Single-threaded version:

```cpp
template <typename T>
class RingBuffer {
    std::vector<T> buf_;
    size_t mask_, head_ = 0, tail_ = 0;                          // head==tail: empty
public:
    explicit RingBuffer(size_t capPow2) : buf_(capPow2), mask_(capPow2 - 1) {
        assert((capPow2 & mask_) == 0 && capPow2 > 0);           // power of two
    }
    bool push(T v) {
        if (tail_ - head_ == buf_.size()) return false;          // full
        buf_[tail_++ & mask_] = std::move(v);
        return true;
    }
    bool pop(T& out) {
        if (head_ == tail_) return false;                        // empty
        out = std::move(buf_[head_++ & mask_]);
        return true;
    }
};
```

⚡ **Lock-free SPSC version** — the follow-up that separates candidates. One producer thread calls push, one consumer calls pop, no locks:

```cpp
template <typename T>
class SpscQueue {
    std::vector<T> buf_;
    size_t mask_;
    alignas(64) std::atomic<size_t> head_{0};   // consumer-owned; alignas(64) = own cache line
    alignas(64) std::atomic<size_t> tail_{0};   // producer-owned; prevents false sharing
public:
    explicit SpscQueue(size_t capPow2) : buf_(capPow2), mask_(capPow2 - 1) {}

    bool push(T v) {                                             // producer only
        size_t t = tail_.load(std::memory_order_relaxed);        // own index: relaxed ok
        if (t - head_.load(std::memory_order_acquire) == buf_.size()) return false;
        buf_[t & mask_] = std::move(v);
        tail_.store(t + 1, std::memory_order_release);           // publish: release
        return true;
    }
    bool pop(T& out) {                                           // consumer only
        size_t h = head_.load(std::memory_order_relaxed);
        if (h == tail_.load(std::memory_order_acquire)) return false;
        out = std::move(buf_[h & mask_]);
        head_.store(h + 1, std::memory_order_release);
        return true;
    }
};
```

**Talking points (learn these cold):**

- _Why release on publish / acquire on read?_ Release ensures the element write is visible **before** the index bump; acquire ensures the consumer sees the element **after** seeing the bump. This is the release/acquire handshake — same mechanism as Q2 from your 7/11 round.
- _Why relaxed on your own index?_ Only one thread ever writes it; no synchronization needed with yourself.
- _Why `alignas(64)`?_ head_ and tail_ on the same cache line → every producer write invalidates the consumer's cached line and vice versa (**false sharing**) → cache-coherence ping-pong. Separate lines removes it.
- _Why not wrap indices at capacity?_ Monotonic indices make full/empty distinction free (`tail-head`); wrapped indices need to sacrifice a slot or track a count.

---

## 4. Median Tracker — Growing Stream (LC 295)

**Trigger:** "median after each insertion," no deletions.

**Invariant:** `lo_` = max-heap of the smaller half, `hi_` = min-heap of the larger half. Balance so `lo_.size() == hi_.size()` or `lo_.size() == hi_.size()+1`. Median = `lo_.top()` (odd) or average of both tops (even).

```cpp
class MedianFinder {
    std::priority_queue<int> lo_;                                        // max-heap
    std::priority_queue<int, std::vector<int>, std::greater<>> hi_;      // min-heap
public:
    void addNum(int num) {
        lo_.push(num);
        hi_.push(lo_.top()); lo_.pop();          // push through lo -> guarantees ordering invariant
        if (hi_.size() > lo_.size()) { lo_.push(hi_.top()); hi_.pop(); }
    }
    double findMedian() {
        if (lo_.size() > hi_.size()) return lo_.top();
        return (lo_.top() + (double)hi_.top()) / 2.0;
    }
};
```

The "push through" trick (always insert into lo, immediately move lo's max to hi, rebalance) avoids all case analysis about which heap the new element belongs to. O(log n) insert, O(1) median.

---

## 5. Sliding Window Median (LC 480) — the eviction variant

**Trigger:** "median of the **last K**" — pattern 4 plus deletion of the oldest element. This is the one from your 7/11 mock: you knew #4, didn't recognize that eviction breaks plain heaps (arbitrary-position deletion).

**Approach A — multiset + median iterator (cleaner):**

**Invariant:** `std::multiset` holds the window; iterator `mid_` points at the median element (the lower one for even K). Every insert/erase shifts `mid_` by at most one position, decided by comparing the changed value against `*mid_`.

```cpp
class SlidingMedian {
    int k_;
    std::multiset<int> win_;
    std::multiset<int>::iterator mid_;
    std::deque<int> order_;                       // insertion order for eviction
public:
    explicit SlidingMedian(int k) : k_(k) {}

    double insert(int x) {
        order_.push_back(x);
        win_.insert(x);
        if (win_.size() == 1) mid_ = win_.begin();
        else if (x < *mid_) { if (win_.size() % 2 == 0) --mid_; }   // grew on the left
        else                { if (win_.size() % 2 == 1) ++mid_; }   // grew on the right (x >= *mid_)

        if ((int)order_.size() > k_) {                              // evict oldest
            int old = order_.front(); order_.pop_front();
            // adjust mid BEFORE erasing, in case old == *mid_'s node
            if (old < *mid_)      { if (win_.size() % 2 == 0) ++mid_; win_.erase(win_.find(old)); }
            else if (old > *mid_) { if (win_.size() % 2 == 1) --mid_; win_.erase(win_.find(old)); }
            else {                                                  // erasing a copy of the median value
                auto victim = win_.find(old);                       // find() returns SOME copy; fine, all equal
                if (victim == mid_) {
                    auto next = std::next(mid_);
                    if (win_.size() % 2 == 1) mid_ = std::prev(mid_);
                    else mid_ = next;
                    win_.erase(victim);
                } else {
                    if (old < *mid_ || victim == win_.lower_bound(old)) {} // duplicate handling collapses:
                    // simplest correct rule: treat victim's side by comparing its position:
                    bool left = std::distance(win_.begin(), victim) < std::distance(win_.begin(), mid_); // O(k)! see note
                    if (left) { if (win_.size() % 2 == 0) ++mid_; }
                    else      { if (win_.size() % 2 == 1) --mid_; }
                    win_.erase(victim);
                }
            }
        }
        size_t n = win_.size();
        if (n % 2 == 1) return *mid_;
        return (*mid_ + (double)*std::next(mid_)) / 2.0;
    }
};
```

**Honesty note:** the duplicate-equal-to-median case is genuinely fiddly (the `std::distance` fallback above is O(K) — in an interview, say "I'd erase a copy that isn't the mid iterator: `win_.erase(victim == mid_ ? handle-specially : victim)`" and handle it by always erasing `win_.lower_bound(old)` and pre-adjusting `mid_` if that happens to _be_ `mid_`). If the interviewer lets you, **Approach B is easier to get right under pressure.**

**Approach B — lazy-deletion two heaps:**

Keep pattern-4 heaps plus `std::unordered_map<int,int> dead_` (value → pending-delete count) and _live_ size counters for each heap. On eviction, increment `dead_[old]` and decrement the live counter of whichever side `old` belongs to (compare against `lo_.top()`). Before reading any `top()`, **prune**: while the top is in `dead_`, pop it and decrement its count. Rebalance using live sizes only.

- Insert: O(log K) · Median: amortized O(log K) · Space: O(K + windows' worth of dead entries).
- Why "lazy"? Heaps can't delete from the middle, so mark-dead-and-purge-when-surfaced. Same idiom as Timer Queue cancellation (#10).

**Ledger cross-ref:** identification trigger = "known aggregate over a _sliding window_ where the base structure can't delete arbitrarily" → either (a) switch to an ordered structure with erase (multiset), or (b) keep the structure and add lazy deletion.

---

## 6. ID Allocator — Recycled Pool + Frontier (LC 2336, LC 379)

**Trigger:** "return the smallest available X" where the space of X is **too large to materialize** (10^9 IDs, unbounded sequence numbers) but the _live_ set is small.

**Invariant:** two disjoint sources of available IDs:

1. `recycled_` — ordered set of previously-issued, now-freed IDs;
2. `frontier_` — a single int: the smallest **never-issued** ID. Everything ≥ frontier is available; everything < frontier is either live or recycled.

Every recycled ID < frontier by construction (it was issued before frontier advanced past it), so _smallest available = min(recycled) if nonempty, else frontier_.

```cpp
class IdAllocator {
    std::set<int> recycled_;
    std::unordered_set<int> live_;                 // exists only to validate free()
    int frontier_ = 0;
    int maxIds_;
public:
    explicit IdAllocator(int maxIds) : maxIds_(maxIds) {}

    int allocate() {
        int id;
        if (!recycled_.empty()) { id = *recycled_.begin(); recycled_.erase(recycled_.begin()); }
        else if (frontier_ < maxIds_) id = frontier_++;
        else return -1;
        live_.insert(id);
        return id;
    }
    void free(int id) {
        if (live_.erase(id) == 0) return;          // invalid / double-free -> no-op
        recycled_.insert(id);
    }
};
```

O(log L) per op where L = live count; memory O(L), **independent of ID-space size** — that's the whole point.

**Follow-ups:**

- ⚡ _Hot-path version?_ `recycled_` as a min-heap over a vector (no per-node allocation); tolerate double-free via lazy deletion, or drop the ordering requirement entirely if "any available ID" is acceptable (then a plain vector free-list, O(1)).
- _Frees cluster into ranges?_ Interval coalescing — pattern #11 — `std::map<int,int>` of free ranges. This is exactly moldcast NAK gap tracking: sequence space too large to materialize, track missing _ranges_.
- _Why not a bitmap?_ 10^9 bits = 125 MB and O(n) scan for smallest. Fine for small dense spaces, wrong here.

---

## 7. Randomized Set — O(1) insert/delete/getRandom (LC 380)

**Trigger:** the getRandom requirement. Hash structures can't sample uniformly in O(1); arrays can't delete in O(1) — so combine.

**Invariant:** `vec_` holds the elements densely; `idx_` maps value → its position in vec_. Delete = **swap-and-pop**: move the last element into the victim's slot, fix its map entry, pop_back.

```cpp
class RandomizedSet {
    std::vector<int> vec_;
    std::unordered_map<int,int> idx_;
    std::mt19937 rng_{std::random_device{}()};
public:
    bool insert(int val) {
        if (idx_.count(val)) return false;
        idx_[val] = vec_.size();
        vec_.push_back(val);
        return true;
    }
    bool remove(int val) {
        auto it = idx_.find(val);
        if (it == idx_.end()) return false;
        int pos = it->second, last = vec_.back();
        vec_[pos] = last; idx_[last] = pos;        // note: works even when val IS last
        vec_.pop_back(); idx_.erase(it);           // erase via iterator AFTER using it
        return true;
    }
    int getRandom() {
        return vec_[std::uniform_int_distribution<int>(0, vec_.size()-1)(rng_)];
    }
};
```

⚡ Swap-and-pop is a general idiom: O(1) unordered removal from a vector. Used constantly in engine/HFT code for "order lists where order doesn't matter."

---

## 8. Min Stack (LC 155) and Min Queue

**Min Stack trigger:** push/pop/top plus getMin, all O(1). **Invariant:** each entry stores the min _of the stack up to and including itself_ — min state is snapshotted per level, so pop needs no recomputation.

```cpp
class MinStack {
    std::vector<std::pair<int,int>> s_;            // {value, min-so-far}
public:
    void push(int v)   { s_.push_back({v, s_.empty() ? v : std::min(v, s_.back().second)}); }
    void pop()         { s_.pop_back(); }
    int  top()  const  { return s_.back().first; }
    int  getMin() const{ return s_.back().second; }
};
```

**Min Queue:** a queue built from two min-stacks (`in_`, `out_`; dequeue from out_, refilling by draining in_ when empty — amortized O(1)). `getMin() = min(in_.getMin(), out_.getMin())`. This is the standard answer when an interviewer extends Min Stack — and it's an alternative implementation of sliding-window min (#9).

---

## 9. Monotonic Deque — Sliding Window Max (LC 239)

**Trigger:** max (or min) of every window of size K over an array/stream, O(n) total.

**Invariant:** deque holds **indices** whose values are strictly decreasing front→back. Front = current window max. New element kills everything smaller from the back (they can never be a future max — dominated: newer AND smaller). Front expires when its index leaves the window.

```cpp
std::vector<int> maxSlidingWindow(const std::vector<int>& a, int k) {
    std::deque<int> dq;                            // indices, values decreasing
    std::vector<int> out;
    for (int i = 0; i < (int)a.size(); ++i) {
        if (!dq.empty() && dq.front() <= i - k) dq.pop_front();   // expire by index
        while (!dq.empty() && a[dq.back()] <= a[i]) dq.pop_back();// dominated
        dq.push_back(i);
        if (i >= k - 1) out.push_back(a[dq.front()]);
    }
    return out;
}
```

Each index enters and leaves the deque once → O(n). **Why indices, not values?** Expiry needs positions; values alone can't tell you when the front leaves the window. ⚡ Streaming max/min over a time window (e.g., "high of the last second") is this exact structure.

---

## 10. Timer Queue

**Trigger:** "schedule callbacks at deadlines, fire in order, support cancellation."

**Invariant:** min-heap keyed by deadline. Cancellation is the interesting part — heaps can't remove from the middle, so **lazy cancellation**: cancel marks an ID dead; expiry pops and skips dead entries. (Same lazy-deletion idiom as #5B.)

```cpp
class TimerQueue {
    struct Timer { uint64_t deadline; uint64_t id; std::function<void()> cb; };
    struct Cmp { bool operator()(const Timer& a, const Timer& b) const { return a.deadline > b.deadline; } };
    std::priority_queue<Timer, std::vector<Timer>, Cmp> heap_;
    std::unordered_set<uint64_t> cancelled_;
    uint64_t nextId_ = 0;
public:
    uint64_t schedule(uint64_t deadline, std::function<void()> cb) {
        heap_.push({deadline, nextId_, std::move(cb)});
        return nextId_++;
    }
    void cancel(uint64_t id) { cancelled_.insert(id); }            // O(1), lazy

    void runExpired(uint64_t now) {
        while (!heap_.empty() && heap_.top().deadline <= now) {
            Timer t = std::move(const_cast<Timer&>(heap_.top())); heap_.pop();
            if (cancelled_.erase(t.id)) continue;                  // skip dead
            t.cb();
        }
    }
    // next wakeup for the event loop's epoll_wait timeout:
    std::optional<uint64_t> nextDeadline() const {
        return heap_.empty() ? std::nullopt : std::optional{heap_.top().deadline};
    }
};
```

⚡ This slots directly into an epoll loop (moldcast M2): `epoll_wait` timeout = `nextDeadline() - now`. Follow-ups: _memory growth from cancelled entries?_ Bounded by scheduled count; purge on pop. _`std::function` overhead?_ Type erasure + possible allocation — hot path uses fixed callback types or intrusive handles (your existing ledger item).

---

## 11. Interval Set — Range Tracking with Coalescing (LC 352, LC 715, LC 57)

**Trigger:** "track which ranges are covered / add ranges / query membership / merge adjacent" — sequence-number gap tracking (⚡ moldcast NAKs), calendar booking, memory-region tracking.

**Invariant:** `std::map<int,int> m_` of disjoint, non-adjacent intervals `[start] -> end`. Every operation: find neighbors with `upper_bound`, merge overlaps, keep the disjointness invariant.

```cpp
class IntervalSet {                                // half-open [start, end)
    std::map<int,int> m_;
public:
    void add(int s, int e) {
        auto it = m_.upper_bound(s);               // first interval with start > s
        if (it != m_.begin() && std::prev(it)->second >= s)  // predecessor overlaps/touches
            { --it; s = it->first; e = std::max(e, it->second); }
        while (it != m_.end() && it->first <= e)   // swallow everything overlapping
            { e = std::max(e, it->second); it = m_.erase(it); }
        m_[s] = e;
    }
    bool contains(int x) const {
        auto it = m_.upper_bound(x);
        return it != m_.begin() && std::prev(it)->second > x;
    }
};
```

Amortized O(log n) per add (each interval inserted once, erased once). **The `upper_bound` + `prev` dance is the memorizable core** — it appears in every interval problem. Removal (LC 715) is the same dance plus possibly _splitting_ one interval into two.

---

## 12. Time-Based Key-Value Store (LC 981)

**Trigger:** `set(key, val, timestamp)` with nondecreasing timestamps; `get(key, t)` returns the value with the largest timestamp ≤ t.

**Invariant:** per key, an append-only vector of `{timestamp, value}` — sorted for free because writes arrive in time order. Read = binary search.

```cpp
class TimeMap {
    std::unordered_map<std::string, std::vector<std::pair<int,std::string>>> m_;
public:
    void set(const std::string& k, std::string v, int t) { m_[k].emplace_back(t, std::move(v)); }
    std::string get(const std::string& k, int t) {
        auto it = m_.find(k);
        if (it == m_.end()) return "";
        auto& v = it->second;
        auto p = std::upper_bound(v.begin(), v.end(), t,
                    [](int t, const auto& e){ return t < e.first; });
        return p == v.begin() ? "" : std::prev(p)->second;
    }
}; 
```

⚡ This is the general shape of "as-of" queries — versioned config, historical snapshots (see #19). The `upper_bound` then `prev` step is the same move as #11. Follow-up: _timestamps not monotonic?_ Insert with `lower_bound` (O(n) shift) or switch to `std::map`.

---

## 13. Hit Counter & Rate Limiter

**Hit Counter (LC 362) trigger:** "how many events in the last N seconds," coarse granularity acceptable. **Invariant:** ring of N per-second buckets; each bucket remembers which second it represents so stale data self-invalidates without a cleanup pass.

```cpp
class HitCounter {                                  // window = 300s
    static constexpr int N = 300;
    std::array<int, N> count_{};                    // hits in that second
    std::array<int, N> stamp_{};                    // which second the slot currently represents
public:
    void hit(int t) {
        int i = t % N;
        if (stamp_[i] != t) { stamp_[i] = t; count_[i] = 0; }   // slot recycled
        ++count_[i];
    }
    int getHits(int t) {
        int total = 0;
        for (int i = 0; i < N; ++i)
            if (t - stamp_[i] < N) total += count_[i];
        return total;
    }
};
```

O(1) hit, O(N) query with N fixed = O(1). Zero allocation, bounded memory regardless of hit rate — say that sentence; it's the point.

**Token Bucket (rate limiter) trigger:** "allow at most R requests/sec with bursts up to B." **Invariant:** a bucket holds up to B tokens, refilled at R/sec, computed **lazily** — no background thread:

```cpp
class TokenBucket {
    double rate_, burst_, tokens_;
    double last_;                                   // last refill time (seconds)
public:
    TokenBucket(double ratePerSec, double burst)
        : rate_(ratePerSec), burst_(burst), tokens_(burst), last_(0) {}
    bool allow(double now) {
        tokens_ = std::min(burst_, tokens_ + (now - last_) * rate_);
        last_ = now;
        if (tokens_ >= 1.0) { tokens_ -= 1.0; return true; }
        return false;
    }
};
```

⚡ Exchanges impose order-rate limits; every OMS has one of these. Follow-up: _sliding-window-log vs token bucket?_ Log (deque of timestamps, pop-expired) is exact but O(rate) memory; bucket is O(1) and allows controlled bursts.

---

## 14. Object Pool / Free List

**Trigger:** ⚡ "no allocation in the hot path" — orders, messages, nodes churned at high rate.

**Invariant:** preallocated slab; free slots form an intrusive singly-linked list threaded _through the slots themselves_ (a union: a slot is either a live T or a next-free index). Allocate = pop head of free list; free = push.

```cpp
template <typename T>
class ObjectPool {
    union Slot { T obj; int next; Slot() {} ~Slot() {} };
    std::vector<Slot> slab_;
    int freeHead_;
public:
    explicit ObjectPool(int n) : slab_(n) {
        for (int i = 0; i < n; ++i) slab_[i].next = i + 1;
        slab_[n - 1].next = -1;
        freeHead_ = 0;
    }
    template <typename... Args>
    T* alloc(Args&&... args) {
        if (freeHead_ == -1) return nullptr;
        Slot& s = slab_[freeHead_];
        freeHead_ = s.next;
        return new (&s.obj) T(std::forward<Args>(args)...);      // placement new
    }
    void free(T* p) {
        p->~T();                                                 // explicit dtor
        Slot* s = reinterpret_cast<Slot*>(p);
        s->next = freeHead_;
        freeHead_ = int(s - slab_.data());
    }
};
```

O(1) alloc/free, zero syscalls, contiguous memory (cache-friendly), no fragmentation. You built a first-fit heap allocator already — this is its constant-time cousin for fixed-size objects. Follow-ups: _thread safety?_ Per-thread pools, or a lock-free Treiber stack for the free list (mention ABA problem). _Relation to #6?_ Same recycled-pool idea; here order doesn't matter so it's O(1) instead of O(log n).

---

## 15. Open-Addressing HashMap

**Trigger:** "implement a hashmap." Chaining is the easy answer; ⚡ open addressing is the answer that shows you know why `std::unordered_map` is slow (per-node allocation, pointer chasing).

**Invariant:** one flat array of slots, each EMPTY / OCCUPIED / TOMBSTONE. Linear probing: collision → next slot. Tombstones keep probe chains intact after erase. Resize at ~0.7 load factor.

```cpp
template <typename K, typename V>
class FlatHashMap {
    enum class State : uint8_t { Empty, Occupied, Tombstone };
    struct Slot { K key; V val; State st = State::Empty; };
    std::vector<Slot> slots_;
    size_t size_ = 0, mask_;

    size_t probe(const K& k) const {                       // returns insert-or-found slot
        size_t i = std::hash<K>{}(k) & mask_;
        size_t firstTomb = SIZE_MAX;
        while (true) {
            const Slot& s = slots_[i];
            if (s.st == State::Empty)     return firstTomb != SIZE_MAX ? firstTomb : i;
            if (s.st == State::Tombstone) { if (firstTomb == SIZE_MAX) firstTomb = i; }
            else if (s.key == k)          return i;
            i = (i + 1) & mask_;
        }
    }
    void maybeGrow() {
        if (size_ * 10 < slots_.size() * 7) return;        // load factor 0.7
        std::vector<Slot> old = std::move(slots_);
        slots_.assign(old.size() * 2, {}); mask_ = slots_.size() - 1; size_ = 0;
        for (auto& s : old) if (s.st == State::Occupied) insert(std::move(s.key), std::move(s.val));
    }
public:
    explicit FlatHashMap(size_t capPow2 = 16) : slots_(capPow2), mask_(capPow2 - 1) {}
    void insert(K k, V v) {
        maybeGrow();
        size_t i = probe(k);
        if (slots_[i].st != State::Occupied) ++size_;
        slots_[i] = {std::move(k), std::move(v), State::Occupied};
    }
    V* find(const K& k) {
        size_t i = probe(k);
        return slots_[i].st == State::Occupied ? &slots_[i].val : nullptr;
    }
    bool erase(const K& k) {
        size_t i = probe(k);
        if (slots_[i].st != State::Occupied) return false;
        slots_[i].st = State::Tombstone; --size_;
        return true;
    }
};
```

**Talking points:** why tombstones (erasing to Empty breaks probe chains for keys that probed past this slot); why linear probing beats chaining on modern CPUs (sequential probes stay in one/two cache lines vs a pointer chase per collision); tombstone accumulation → periodic rehash; robin-hood hashing / SIMD probing (Swiss tables) as the "what does production use" flex.

---

## 16. Limit Order Book ⚡

**Trigger:** the HFT interview classic. "Support add/cancel/execute; best bid/ask in O(1)-ish; price levels in order."

**Invariant:** per side, `std::map<price, Level>` (sorted price levels; best = `begin()`/`rbegin()`); each Level is a FIFO `std::list<Order>` (time priority); a global `unordered_map<orderId, {level-map iterator, list iterator}>` makes cancel O(1)-after-hash. Iterator stability of map + list is what makes this legal (same trick as LRU).

```cpp
struct Order { uint64_t id; int qty; };
struct Level { int totalQty = 0; std::list<Order> fifo; };

class OrderBook {
    using LevelMap = std::map<int, Level, std::greater<>>;      // bids: high->low
    LevelMap bids_;                                             // (asks_: std::less, omitted for brevity)
    struct Handle { LevelMap::iterator lvl; std::list<Order>::iterator ord; };
    std::unordered_map<uint64_t, Handle> byId_;
public:
    void addBid(uint64_t id, int price, int qty) {
        auto [lit, _] = bids_.try_emplace(price);
        lit->second.totalQty += qty;
        lit->second.fifo.push_back({id, qty});
        byId_[id] = {lit, std::prev(lit->second.fifo.end())};
    }
    void cancel(uint64_t id) {
        auto it = byId_.find(id);
        if (it == byId_.end()) return;
        auto [lit, oit] = it->second;
        lit->second.totalQty -= oit->qty;
        lit->second.fifo.erase(oit);
        if (lit->second.fifo.empty()) bids_.erase(lit);
        byId_.erase(it);
    }
    // best bid: bids_.begin() -> {price, level.totalQty}
};
```

**The design-choice interrogation (they will ask):**

- _Why `map` not `unordered_map` for levels?_ Need ordered traversal (best price, walking the book on a sweep). Also ⚡ price levels cluster near the top → hot map nodes stay cached.
- _Why `list` per level, not vector?_ O(1) cancel-from-middle with a stored iterator; FIFO time priority. Vector cancel is O(n) shift or breaks time order with swap-and-pop.
- ⚡ _Production version?_ Prices are ticks → dense array of levels indexed by (price − base)/tick, best-price pointer walks; intrusive lists over pooled order objects (#14). map/list is the correct _interview_ answer; knowing the array version exists is the flex.
- _Match/execute?_ Repeatedly hit `begin()` of the opposite side, consume FIFO front, erase empty levels.

---

## 17. Union-Find (DSU)

**Trigger:** "merge groups / are these connected / count components," online (queries interleaved with merges).

```cpp
class DSU {
    std::vector<int> p_, r_;
public:
    explicit DSU(int n) : p_(n), r_(n, 0) { std::iota(p_.begin(), p_.end(), 0); }
    int find(int x) { return p_[x] == x ? x : p_[x] = find(p_[x]); }   // path compression
    bool unite(int a, int b) {
        a = find(a); b = find(b);
        if (a == b) return false;
        if (r_[a] < r_[b]) std::swap(a, b);
        p_[b] = a;
        if (r_[a] == r_[b]) ++r_[a];
        return true;
    }
};
```

Near-O(1) amortized (inverse Ackermann) with both optimizations. Know: Kruskal's MST, accounts-merge, "number of islands II" (incremental), cycle detection in undirected graphs. Interview probe: _why does path compression alone (no rank) still work well?_ Amortized O(log n); the pair gives α(n).

---

## 18. Trie

**Trigger:** prefix queries — autocomplete, startsWith, word search with wildcards, XOR-maximization (bitwise trie, LC 421 — on your deferred list).

```cpp
class Trie {
    struct Node { std::array<Node*, 26> ch{}; bool end = false; };
    Node* root_ = new Node();
public:
    void insert(const std::string& w) {
        Node* n = root_;
        for (char c : w) {
            auto& nxt = n->ch[c - 'a'];
            if (!nxt) nxt = new Node();
            n = nxt;
        }
        n->end = true;
    }
    bool search(const std::string& w) const { const Node* n = walk(w); return n && n->end; }
    bool startsWith(const std::string& p) const { return walk(p) != nullptr; }
private:
    const Node* walk(const std::string& s) const {
        const Node* n = root_;
        for (char c : s) { n = n->ch[c - 'a']; if (!n) return nullptr; }
        return n;
    }
};
```

(Leaks in this form — interview-acceptable; mention unique_ptr children or pool allocation (#14) as the fix.) **Bitwise trie variant:** children = {0,1}, insert numbers MSB-first, "max XOR with x" = greedily walk taking the opposite bit when it exists. That's the whole of LC 421.

---

## 19. Snapshot Array (LC 1146)

**Trigger:** "set values, take snapshots, read as-of snapshot k" — versioned reads.

**Invariant:** per index, an append-only vector of `{snapId, value}`; read = binary search for largest snapId ≤ k. Snapshots are O(1) (just bump a counter) because unchanged indices store nothing — sparse versioning.

```cpp
class SnapshotArray {
    std::vector<std::vector<std::pair<int,int>>> hist_;   // per index: {snapId, val}
    int snap_ = 0;
public:
    explicit SnapshotArray(int n) : hist_(n, {{-1, 0}}) {}
    void set(int i, int v) {
        if (hist_[i].back().first == snap_) hist_[i].back().second = v;
        else hist_[i].push_back({snap_, v});
    }
    int snap() { return snap_++; }
    int get(int i, int snapId) {
        auto& h = hist_[i];
        auto it = std::upper_bound(h.begin(), h.end(), std::pair{snapId, INT_MAX});
        return std::prev(it)->second;
    }
};
```

Same as-of binary search as #12 — recognize them as one family: **append-only history + upper_bound**.

---

## 20. Bounded Blocking Queue (LC 1188)

**Trigger:** "producer/consumer, block when full/empty" — the standard concurrency-implementation question, and the mutex-based counterpart to #3.

**Invariant:** mutex guards the queue; **two** condition variables — `notFull_` (producers wait on it) and `notEmpty_` (consumers) — so a push wakes only consumers and vice versa. Waits use predicates (spurious wakeups).

```cpp
template <typename T>
class BlockingQueue {
    std::mutex m_;
    std::condition_variable notFull_, notEmpty_;
    std::queue<T> q_;
    size_t cap_;
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

**Talking points:** why the predicate lambda (spurious wakeups + lost-wakeup races — the CV re-checks under the lock); why two CVs not one (one CV + notify_one can wake the wrong side → deadlock; one CV requires notify_all, thundering herd); `notify_one` outside vs inside the lock (either is correct; outside avoids the woken thread immediately blocking on the still-held mutex). Williams ch4 territory. ⚡ Contrast with #3: this is the general MPMC tool; SPSC ring is the specialization that removes the lock entirely.

---

## Cross-Cutting Idioms (the meta-patterns)

These recur across the 20 above — recognizing THEM is what generalizes:

1. **Hashmap → stable iterator/index into an ordered structure.** LRU (#1), LFU (#2), Order Book (#16). Enables O(1) jump into the middle of something ordered. Requires iterator stability: list/map yes, vector/deque no (your invalidation cluster is exactly why this matters).
2. **Lazy deletion.** Sliding median B (#5), Timer Queue (#10), heap-based allocators. When the structure can't delete from the middle, mark dead and purge when it surfaces. Cost: memory grows with dead entries; always mention the bound.
3. **Swap-and-pop.** Randomized Set (#7). O(1) unordered vector removal.
4. **Recycled pool + frontier / free list.** ID Allocator (#6), Object Pool (#14). Never materialize a huge space; track only what's been touched.
5. **Append-only history + upper_bound ("as-of" queries).** TimeMap (#12), SnapshotArray (#19).
6. **upper_bound + prev neighbor dance.** Interval Set (#11), as-of queries (#12, #19). THE std::map interview move.
7. **Per-slot self-invalidating timestamps.** Hit Counter (#13). Avoids cleanup passes.
8. **Monotonic index pairs.** Ring buffer (#3): never wrap the logical index, mask on access.
9. **Two-structure balance.** Median heaps (#4/#5): maintain a size invariant across two halves.
10. **Dominance pruning.** Monotonic deque (#9): discard anything that can never again be the answer.

## Suggested drill order (given your current state)

1. LC 2336 (ID allocator — lock in the new pattern), LC 480 (sliding median — the mock miss), both cold.
2. Re-implement #3 SPSC and #20 BlockingQueue from scratch after Williams ch5 — they're the applied form of release/acquire and CVs respectively.
3. #16 Order Book from a blank file, timed 25 min — near-guaranteed at Optiver/IMC/Citadel Securities.
4. #15 FlatHashMap once, untimed — it teaches tombstones + load factor, common "design" probe.
5. The rest: read-and-rederive; implement only if the invariant doesn't feel obvious.

---

# Part 2 — Algorithmic Combination Patterns

The combos above are class-design structures; these are in-solve techniques. Same format: trigger, invariant, code, probes.

## 21. Hashmap of Counts

**Trigger:** "how many of each," "contains duplicates," anagram equality, sliding window with "at most K distinct."

**Invariant:** `unordered_map<T,int>` (or `array<int,26>` for lowercase letters — always prefer the array when the alphabet is fixed: no hashing, no allocation, cache-resident). The sliding-window form maintains counts incrementally: enter → `++cnt[x]`, and track `distinct` by checking the 0→1 and 1→0 transitions.

```cpp
// LC 340-style: longest substring with at most k distinct
int longestKDistinct(const std::string& s, int k) {
    std::array<int, 128> cnt{};
    int distinct = 0, best = 0, l = 0;
    for (int r = 0; r < (int)s.size(); ++r) {
        if (cnt[s[r]]++ == 0) ++distinct;
        while (distinct > k)
            if (--cnt[s[l++]] == 0) --distinct;
        best = std::max(best, r - l + 1);
    }
    return best;
}
```

The transition-counting trick (`++x == 0` / `--x == 0`) is the memorizable core — it makes "number of distinct" O(1) to maintain instead of O(alphabet) to recount.

## 22. Prefix-Sum + Hashmap

**Trigger:** "count/find subarrays with sum exactly K" (or divisible by K, or equal 0s and 1s). The giveaway: _subarray_ + an _exact_ aggregate → you cannot two-pointer it when negatives exist.

**Invariant:** `sum[i..j] == K ⟺ prefix[j] − prefix[i−1] == K ⟺ prefix[i−1] == prefix[j] − K`. So walk once, and for each position ask the hashmap "how many earlier prefixes equal `cur − K`?" Seed with `{0: 1}` (the empty prefix) — forgetting the seed is the classic bug.

```cpp
// LC 560: count subarrays summing to k
int subarraySum(const std::vector<int>& a, int k) {
    std::unordered_map<long long, int> seen{{0, 1}};
    long long cur = 0; int ans = 0;
    for (int x : a) {
        cur += x;
        if (auto it = seen.find(cur - k); it != seen.end()) ans += it->second;
        ++seen[cur];
    }
    return ans;
}
```

Variants: store **first index** instead of count for "longest subarray with sum K"; store `prefix % K` for divisibility (mind negative mods — `((cur % k) + k) % k`, your Category-G subtraction-guard cousin); map 0→−1 to turn "equal 0s and 1s" into "sum = 0."

## 23. Monotonic Stack

**Trigger:** "next/previous greater/smaller element for each position," largest rectangle, stock span, trapping water. Distinct from #9: stack = nearest-element queries over the whole array; deque = extremum of a _fixed-width window_.

**Invariant:** stack of indices whose values are (say) strictly decreasing. When a new element violates the order, everything it pops has just found its "next greater" — answer recorded at pop time.

```cpp
// next greater element index for each i (-1 if none)
std::vector<int> nextGreater(const std::vector<int>& a) {
    std::vector<int> ans(a.size(), -1), st;        // st: indices, values decreasing
    for (int i = 0; i < (int)a.size(); ++i) {
        while (!st.empty() && a[st.back()] < a[i]) { ans[st.back()] = i; st.pop_back(); }
        st.push_back(i);
    }
    return ans;
}
```

Each index pushed/popped once → O(n). LC 84 (largest rectangle) = for each bar, previous-smaller and next-smaller bound its rectangle — one pass with a sentinel. The four-way choice (next/prev × greater/smaller) is controlled by the comparison direction and iteration direction; derive it, don't memorize four templates.

## 24. Set as Ordered Index

**Trigger:** "nearest active element to X," seat assignment, "delete elements but query neighbors of survivors" — you need order + arbitrary erase + predecessor/successor, which vector (O(n) erase) and hashmap (no order) each fail.

**Invariant:** `std::set<int>` of live positions; every query is the `lower_bound` + neighbor dance (same move as #11/#12):

```cpp
std::set<int> live;
// nearest live element to x:
auto it = live.lower_bound(x);                 // first >= x
int best = INT_MAX;
if (it != live.end())   best = *it;
if (it != live.begin()) {
    int lo = *std::prev(it);
    if (x - lo <= best - x) best = lo;         // tie -> lower
}
```

O(log n) per op. ⚡ Also the honest first draft of a price-level index before you justify the full order book (#16). Probe: _why not sort a vector once?_ Fine if no deletions; the set earns its log factor only when the live set mutates.

## 25. Bucket by Count

**Trigger:** "top-K frequent" or anything keyed by frequency, when O(n log n) via heap feels lazy and counts are bounded by n.

**Invariant:** a frequency can't exceed n, so `vector<vector<T>> bucket(n+1)` indexed by count is a valid "sort" — O(n) total.

```cpp
// LC 347: top-k frequent, O(n)
std::vector<int> topK(const std::vector<int>& a, int k) {
    std::unordered_map<int,int> cnt;
    for (int x : a) ++cnt[x];
    std::vector<std::vector<int>> bucket(a.size() + 1);
    for (auto& [v, c] : cnt) bucket[c].push_back(v);
    std::vector<int> out;
    for (int c = a.size(); c > 0 && (int)out.size() < k; --c)
        for (int v : bucket[c]) { out.push_back(v); if ((int)out.size() == k) break; }
    return out;
}
```

Same family as counting sort: whenever the key range is O(n), you can bucket instead of compare-sort. LFU cache (#2) is this idea made incremental.

## 26. String → Canonical Key

**Trigger:** "group things that are equivalent under some transformation" — anagrams, shifted strings, isomorphic patterns.

**Invariant:** map each item to a canonical representative of its equivalence class; equal canon ⟺ same group; hashmap keyed by canon does the grouping.

```cpp
// LC 49: group anagrams — canon = sorted string (or 26-count signature for O(len))
std::unordered_map<std::string, std::vector<std::string>> groups;
for (auto& s : strs) {
    std::string key = s;
    std::sort(key.begin(), key.end());
    groups[key].push_back(s);
}
```

Count-array canon (`"1#0#2#..."` from the 26 counts) beats sorting for long strings: O(len) vs O(len log len). The generalizable idea: **design the key so the hashmap does the reasoning** — shifted strings → normalize first char to 'a'; isomorphic → first-occurrence-index pattern.

## 27. Fenwick / Segment Tree + Coordinate Compression

**Trigger:** "for each element, count earlier elements smaller/greater than it," inversions, dynamic prefix sums over a huge value range. Your flagged weak paradigm — the recognition cue is **order statistics computed online while sweeping**.

**Invariant, two halves:**

- _Compression:_ values up to 1e9 but only n distinct → sort-unique them; a value's rank (via `lower_bound`) becomes its index. You never index by the value, only by its rank.
- _Fenwick:_ array where `tree[i]` covers a range ending at i of length `i & (-i)` (lowest set bit). point-update and prefix-sum both O(log n) by walking the bit pattern.

```cpp
struct Fenwick {
    std::vector<int> t;
    explicit Fenwick(int n) : t(n + 1, 0) {}
    void add(int i, int d)  { for (++i; i < (int)t.size(); i += i & (-i)) t[i] += d; }
    int  sum(int i) const   { int s = 0; for (++i; i > 0; i -= i & (-i)) s += t[i]; return s; } // prefix [0..i]
};

// LC 315: count smaller after self — sweep right-to-left
std::vector<int> countSmaller(std::vector<int>& a) {
    std::vector<int> sorted(a); 
    std::sort(sorted.begin(), sorted.end());
    sorted.erase(std::unique(sorted.begin(), sorted.end()), sorted.end());
    auto rank = [&](int v) { return std::lower_bound(sorted.begin(), sorted.end(), v) - sorted.begin(); };

    Fenwick fw(sorted.size());
    std::vector<int> ans(a.size());
    for (int i = (int)a.size() - 1; i >= 0; --i) {
        int r = rank(a[i]);
        ans[i] = r > 0 ? fw.sum(r - 1) : 0;        // how many already-seen values are strictly smaller
        fw.add(r, 1);
    }
    return ans;
}
```

**When Fenwick vs segment tree:** Fenwick when the operation is an invertible prefix aggregate (sum, count); segment tree when you need arbitrary-range min/max, non-invertible ops, or lazy range updates (your LC 2407 / 2839 KIV items are segment-tree-with-max, not Fenwick). **The sweep template to internalize:** iterate in one order; per element, _query_ the structure about what's already inserted, then _insert_ the element. The direction of the sweep and the definition of "rank" are the only two decisions.

## Updated cross-cutting idioms (additions)

11. **Transition counting** (`++x == 1` / `--x == 0`) — maintain "number of distinct/active" in O(1). (#21)
12. **Aggregate difference → hashmap lookup** — turn "exact subarray property" into "have I seen `cur − K`." (#22)
13. **Pop-time answering** — in monotonic structures, the answer for an element is finalized the moment it's popped. (#23, #9)
14. **Key design as reasoning** — canonical keys make equivalence a hashmap hit. (#26)
15. **Compress, then index by rank** — huge value domain + order queries → sort-unique + lower_bound. (#27, and #6's "don't materialize the space" is the same instinct)

---

# Part 3 — Patterns Added on Review

## 28. K-Way Merge (heap of cursors)

**Trigger:** "merge K sorted lists/streams," "k-th smallest in a sorted matrix," "smallest range covering one element from each list." ⚡ The direct trading-systems form: merge K sorted exchange feeds into one time-ordered stream — a genuinely asked HFT question.

**Invariant:** min-heap of K _cursors_ (current element + which source + position). Pop the global minimum, advance that cursor, push it back if not exhausted. The heap never exceeds K entries — that's the entire memory story.

```cpp
// LC 23: merge k sorted lists
ListNode* mergeKLists(std::vector<ListNode*>& lists) {
    auto cmp = [](ListNode* a, ListNode* b){ return a->val > b->val; };
    std::priority_queue<ListNode*, std::vector<ListNode*>, decltype(cmp)> pq(cmp);
    for (auto* l : lists) if (l) pq.push(l);
    ListNode dummy; ListNode* tail = &dummy;
    while (!pq.empty()) {
        ListNode* n = pq.top(); pq.pop();
        tail = tail->next = n;
        if (n->next) pq.push(n->next);
    }
    return dummy.next;
}
```

O(N log K). Probes: _why not merge pairwise?_ Also O(N log K) via divide-and-conquer — know both. _Streaming version?_ Identical structure; cursors are live feed handles, pop = publish next event. LC 632 (smallest range) is this plus tracking the current max across cursors.

## 29. Sweep Line — Sort Events + Heap/Ordered Multiset

**Trigger:** intervals asking for a _global aggregate over time_: "minimum rooms," "max concurrent," skyline. Distinct from #11: interval _set_ maintains coverage incrementally online; sweep line answers a batch question by sorting time itself.

**Invariant:** convert each interval to two events (start:+1, end:−1), sort by time (ends before starts at ties, or after — decide from the problem's boundary semantics; this tie rule is where the bugs live), sweep accumulating.

```cpp
// Meeting Rooms II (LC 253): min rooms
int minRooms(std::vector<std::vector<int>>& iv) {
    std::vector<std::pair<int,int>> ev;                 // {time, +1/-1}
    for (auto& i : iv) { ev.push_back({i[0], +1}); ev.push_back({i[1], -1}); }
    std::sort(ev.begin(), ev.end());                    // -1 sorts before +1 at same time: end frees room first
    int cur = 0, best = 0;
    for (auto& [t, d] : ev) best = std::max(best, cur += d);
    return best;
}
```

When the aggregate is more than a count — skyline needs "current max height" — replace the counter with a `multiset` (insert height on start, erase one copy on end, answer = `*rbegin()`): that's LC 218. Heap-of-end-times is the other classic formulation of 253 (sort by start; pop ends ≤ current start; heap size = rooms). Family: #11 = online coverage, #29 = offline aggregate, #13 = fixed-window count.

## 30. Difference Array

**Trigger:** "apply many range increments, then read final values" — bookings, car pooling, resource load. The offline cousin of Fenwick: if all updates happen before all reads, you don't need a tree.

**Invariant:** `diff[l] += v; diff[r+1] -= v;` for each update; one prefix-sum pass materializes the array. O(1) per update, O(n) finalize.

```cpp
// LC 1109: corporate flight bookings
std::vector<int> corpFlightBookings(std::vector<std::vector<int>>& b, int n) {
    std::vector<int> d(n + 1, 0);
    for (auto& x : b) { d[x[0]-1] += x[2]; d[x[1]] -= x[2]; }
    for (int i = 1; i < n; ++i) d[i] += d[i-1];
    d.pop_back();
    return d;
}
```

Decision rule: updates-then-reads → difference array; interleaved updates/reads → Fenwick (#27); interleaved with _range_ reads and _range_ updates → lazy segment tree. LC 1094 (car pooling) is the same with a fixed 1001-slot axis. 2D version exists (four corner adjustments) for grid stamping problems.

## 31. Rolling Hash (often + Binary Search on Answer)

**Trigger:** substring equality at scale — "longest duplicate substring," repeated patterns, comparing many windows of the same length.

**Invariant:** polynomial hash `h = s[0]*B^(L-1) + ... + s[L-1] (mod M)`; sliding the window is O(1): `h' = (h − s[i]*B^(L-1)) * B + s[i+L]` (all mod M, with the add-M-before-subtract guard — **Category G alert**, this is exactly your no-guard/modpow bug family). "Does any length-L substring repeat" becomes hashset membership; since that predicate is monotone in L, binary search the length.

```cpp
// core: all length-L window hashes of s, O(n)
std::vector<uint64_t> windowHashes(const std::string& s, int L) {
    const uint64_t B = 131, M = (1ULL << 61) - 1;       // or double-hash to dodge collisions
    auto mulmod = [&](uint64_t a, uint64_t b){ return (__uint128_t)a * b % M; };
    uint64_t h = 0, pw = 1;
    for (int i = 0; i < L; ++i) { h = (mulmod(h, B) + s[i]) % M; if (i) pw = mulmod(pw, B); }
    pw = mulmod(pw, B);                                  // B^(L-1)... careful: pw = B^(L-1)
    std::vector<uint64_t> out{h};
    for (int i = L; i < (int)s.size(); ++i) {
        h = (h + M - mulmod(s[i - L], pw) % M) % M;      // subtract guard
        h = (mulmod(h, B) + s[i]) % M;
        out.push_back(h);
    }
    return out;
}
```

Probes: collision handling (double hash, or verify candidates by direct compare); why mod a large prime near 2^61 (fast reduction, birthday-bound safety). LC 1044 = binary search L + this. Honest interview note: say "hashes can collide; for correctness I'd verify or double-hash" unprompted — it's a credibility marker.

## Brief mentions (know they exist; full sections not warranted)

- **Skip list** (LC 1206): probabilistic ordered map — tower of linked lists, level ~ geometric. Worth one read; Redis sorted sets use it; occasionally asked as "implement an ordered map without trees."
- **Iterator designs**: BST iterator (LC 173 — stack of left-spines, O(h) memory lazy in-order), Flatten Nested List (LC 341 — stack of {list, index} cursors), Peeking Iterator (LC 284 — one-element buffer). All the same idea: **externalize a traversal's call stack into an object**.
- **Prefix bitmask parity** (variant of #22): when the property is "each char/state appears an even number of times," replace the running sum with a running XOR mask; "seen this mask before" via hashmap. LC 1371, 1915.
- **Two-stacks editors** (LC 71-adjacent designs, browser history LC 1472): cursor = boundary between two stacks; undo/redo = pop/push across. Trivial once seen, annoying to invent live.

---

# Practice Problem Map

Ratings ≈ Zerotrac where known; ⭐ = do cold this week (highest yield for you specifically). Premium marked (P).

|#|Pattern|Problems|
|---|---|---|
|1|LRU|146 ⭐ (from scratch, timed 15 min)|
|2|LFU|460|
|3|Ring buffer|622, 641 (then the SPSC version from this doc, no LC equivalent)|
|4|Two heaps|295|
|5|Sliding median|480 ⭐ (the mock miss — both approaches)|
|6|ID allocator|2336 ⭐, 379 (P)|
|7|Randomized set|380, 381 (dups variant — forces you to rethink swap-and-pop)|
|8|Min stack/queue|155, 232|
|9|Monotonic deque|239, 1696, 862 (hard: deque over _prefix sums_ — great stretch)|
|10|Timer/TTL|1797, 362|
|11|Interval set|57, 352, 729 → 731 → 732 (escalating), 715 (the boss fight), 2276|
|12|As-of KV|981|
|13|Hit counter|362|
|14|Object pool|(no LC — implement from this doc)|
|15|Hashmap impl|706 (then upgrade to open addressing yourself)|
|16|Order book|(no LC — implement from this doc, timed 25 min ⭐)|
|17|Union-Find|547, 684, 721 ⭐ (accounts merge — DSU + canonical grouping combo), 947|
|18|Trie|208, 211, 212, 421 ⭐ (your deferred bit-trie), 1707 (deferred; offline sort + trie)|
|19|Snapshot|1146|
|20|Blocking queue|1188 (P), 1115/1116 (free concurrency alternatives)|
|21|Count maps|438, 567, 76 (the hard template), 340 (P)|
|22|Prefix + hashmap|560 ⭐, 525, 974 (⚠ negative-mod, Category G), 325 (P)|
|23|Monotonic stack|739, 496, 901, 907 (contribution counting — subtle), 84 ⭐, 42|
|24|Ordered index|220 (set + lower_bound in a window), 855|
|25|Bucket by count|347, 451, 1636|
|26|Canonical key|49, 205, 890|
|27|Fenwick + compression|315 ⭐ (your flagged paradigm), 493, 327 (hard: range-sum counts), 1649; segment tree: 2407 (your KIV), 218|
|28|K-way merge|23 ⭐, 373, 378, 632 (hard), 355 (design form)|
|29|Sweep line|253 (P — or the identical free 2406 via heap), 1094, 218, 731|
|30|Difference array|1109, 1094, 2381|
|31|Rolling hash|187, 28 (as hash practice), 1044 ⭐ (binary search + hash, hard)|
|—|Iterators|173, 341, 284|
|—|Bitmask parity|1915, 1371|
|—|Skip list|1206|

**Suggested 2-week sequencing** (interleave with your DP/knapsack track, don't replace it):

- Days 1–3: the ⭐ set minus hards — 480, 2336, 146, 560, 23, 721.
- Days 4–7: 315 + 421 (both flagged/deferred paradigms), 84, 212, 907.
- Days 8–11: interval escalation 729→731→732→715; sweep line 2406, 218.
- Days 12–14: hards — 1044, 632, 862, 327. Plus the two no-LC implementations (order book, SPSC) timed.