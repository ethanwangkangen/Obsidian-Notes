# STL Container Reference — Complexities, Iterators, Selection, Patterns

Interview-oriented. HFT-relevant notes marked ⚡. Cross-refs to your ledger where relevant.

---

## 1. Sequence Containers

### `std::vector<T>`

- **Layout:** single contiguous heap array. size/capacity distinction; geometric growth (~1.5–2×).
- **Complexity:** index/`[]` O(1) · push_back amortized O(1) · insert/erase middle O(n) · find O(n).
- **Iterators:** random-access, contiguous.
- **Invalidation:** on **reallocation** — everything. Without reallocation, insert/erase invalidates iterators/refs **at and after** the point only. (Round-4 refinement: not "always".)
- **When:** the default. Contiguity → cache-friendly traversal beats big-O on node containers for n up to thousands. ⚡ `reserve()` when size is known — reallocation is the enemy (and needs `noexcept` moves, see ledger `move_if_noexcept`).
- **Traps:** `vector<bool>` proxy (ledger entry) · iterator use after push_back · `v(5,10)` vs `v{5,10}` (init-list greediness, R1Q2).

### `std::deque<T>`

- **Layout:** map of fixed-size blocks; elements never move.
- **Complexity:** `[]` O(1) (two derefs) · push/pop at **both ends** amortized O(1) · middle insert O(n).
- **Iterators:** random-access, NOT contiguous.
- **Invalidation — the famous asymmetry (R4Q5):** insert at either end invalidates **all iterators** but **no pointers/references**. Middle insert invalidates everything.
- **When:** need front+back growth with indexing (sliding window, 0-1 BFS, monotonic deque). Backing store of `stack`/`queue` by default.

### `std::list<T>` / `std::forward_list<T>`

- **Layout:** doubly/singly linked nodes. Per-element allocation + 2/1 pointers overhead.
- **Complexity:** insert/erase at known position O(1) · access/find O(n) · `splice` O(1).
- **Invalidation:** never, except the erased element itself. Strongest stability in the STL.
- **When:** almost never for performance (cache-hostile ⚡); legit uses: splice-based algorithms, LRU cache (list + hashmap, §5), stable references mandatory. `forward_list` = memory-minimal niche.

### `std::array<T, N>`

- Stack, fixed size in the **type** (no decay — see today's session). O(1) everything, zero overhead vs C array. Aggregate: `std::array<int,3> a;` leaves elements uninitialized for trivial T. When: size known at compile time.

### `std::string`

- `vector<char>` + **SSO** (~15–22 chars inline: move = copy cost for short strings, ledger #6) + null terminator maintained.
- Loop trap (R3Q7): `s += c` amortized O(1) → O(n) total; `s = s + c` copies all → **O(n²) total**.
- `substr` returns a **copy** O(k); `find` is O(n·m) naive, not KMP.

### `std::string_view` / `std::span<T>`

- Non-owning (ptr, len). O(1) `substr`/`subspan`, no allocation.
- **Not null-terminated** (R4Q4): never `printf("%s", sv.data())`; use `%.*s` or format.
- Lifetime: view of a temporary dangles at the semicolon (ledger #10).

---

## 2. Container Adaptors


|Adaptor|Default backing|Ops|Notes|
|---|---|---|---|
|`stack<T>`|deque|push/pop/top O(1)|LIFO; can back with vector|
|`queue<T>`|deque|push/pop/front/back O(1)|FIFO; vector NOT allowed (no pop_front)|
|`priority_queue<T>`|vector + heap algos|push/pop **O(log n)**, top O(1)|**Max-heap by default**; min-heap = `priority_queue<T, vector<T>, greater<T>>`. No decrease-key, no iteration → lazy deletion pattern (§5). Build from range = O(n) via ctor (heapify), not n·log n|

---

## 3. Ordered Associative (red-black trees)

`std::set / map / multiset / multimap`

- **Complexity:** insert/erase/find **O(log n)**. In-order traversal O(n), sorted.
- **Iterators:** bidirectional (no `it + 5`). **Invalidation: never** except the erased node. Pointers/refs to elements: stable forever.
- **The tree-only superpowers** (reason to pick over unordered):
    - `lower_bound` / `upper_bound` / ordered predecessor-successor queries O(log n)
    - range iteration between two keys
    - `begin()`/`rbegin()` = min/max in O(1)-ish
- **Key requirement:** strict weak ordering comparator (`<`, never `<=` — UB, ledger #11). Heterogeneous lookup with `std::less<>` avoids constructing a key.
- **Traps:** `map::operator[]` default-inserts on read (ledger #9 — three strikes now); no `operator[]` on const map; keys are const (can't mutate in place — extract/re-insert, or C++17 `node_handle`).
- `multiset` as a poor-man's ordered multiset with O(log n) erase-one: `ms.erase(ms.find(x))` (erase(x) removes ALL).

---

## 4. Unordered Associative (hash tables)

`std::unordered_set / unordered_map / …`

- **Layout:** bucket array of singly-linked node chains (separate chaining mandated in practice by API).
- **Complexity:** insert/find/erase **O(1) average, O(n) worst** (all keys one bucket — adversarial inputs are a real attack ⚡; some contest sites hack `unordered_map` with anti-hash tests → custom hash with random seed).
- **Iterators:** forward only. **Invalidation:** iterators die on **rehash** only; pointers/references **never** (nodes don't move) — R4Q5.
- **Requirements (R2Q10):** hash **and** equality. Invariant: `a == b ⇒ hash(a) == hash(b)`, or lookups silently return `end()`.
- **Control:** `reserve(n)` pre-buckets ⚡ (rehash mid-run is a latency spike); `max_load_factor`.
- **vs map decision:** need ordering/range/neighbor queries → `map`; pure key→value lookup → `unordered_map`; small n (< ~50) → a flat sorted `vector` + `lower_bound` often beats both ⚡ (see §6).

---

## 5. Combination Patterns (the "hashmap + X" catalog)

|Pattern|Construction|Canonical use|
|---|---|---|
|**hashmap + doubly-linked list**|`unordered_map<K, list<pair<K,V>>::iterator>` + `list`|**LRU cache** (LC 146): O(1) get/put; list stability makes stored iterators safe|
|**hashmap of counts**|`unordered_map<T,int>`|frequency problems, sliding-window "distinct count", anagram checks|
|**prefix-sum + hashmap**|running sum → `map[sum]` = count/first-index|subarray-sum-equals-k (LC 560), longest subarray with sum 0|
|**heap + lazy deletion**|`priority_queue` + hashmap of "dead" entries; pop-and-discard stale tops|Dijkstra without decrease-key; sliding-window max/median variants|
|**two heaps**|max-heap (lower half) + min-heap (upper half)|running median (LC 295)|
|**monotonic stack**|`vector`/`stack`, pop while order violated|next greater element, largest rectangle (LC 84), stock span|
|**monotonic deque**|`deque` of indices, expire front, dominate back|sliding-window maximum (LC 239) O(n)|
|**sorted map for intervals**|`map<int,int>` start→end, `lower_bound` neighbors|insert/merge intervals, calendar booking (LC 729)|
|**set as ordered index**|`set<int>` + `lower_bound`|"nearest active element", seat reservation, snapshot ordering|
|**hashmap + vector**|value→index map, swap-with-last on erase|O(1) insert/delete/getRandom (LC 380)|
|**bucket by count**|`vector<vector<T>>` indexed by frequency|top-k frequent O(n) (LC 347 without heap)|
|**string → canonical key**|sorted string or count-array as hashmap key|group anagrams (LC 49)|
|**Fenwick / segment tree over compressed coords**|sort-unique values → indices → BIT|count smaller after self (LC 315) — your flagged weak paradigm|
|**trie**|array-of-26 or hashmap children|prefix queries, XOR problems (LC 421 via bit-trie — your deferred queue)|

---

## 6. Selection Flowchart

1. **Fixed size, known at compile time?** → `array`.
2. **Just a sequence?** → `vector`. Deviate only for: both-end ops (`deque`), O(1) splice/stable refs (`list`).
3. **Key lookup?** → ordering needed (range / neighbors / sorted iteration)? → `map`/`set`. Else → `unordered_map`/`unordered_set`. Tiny n or read-heavy after build → sorted `vector` + `lower_bound` / `equal_range` (flat-map pattern: best cache behavior, O(log n) find, O(n) insert — ⚡ ubiquitous in HFT code precisely because n is small and reads dominate).
4. **Only need max/min repeatedly?** → `priority_queue` (not a sorted structure).
5. **Duplicates with ordering?** → `multiset`/`multimap`.
6. **Non-owning window over existing data?** → `span`/`string_view`.

⚡ Rule of thumb: below a few thousand elements, contiguous + "worse" big-O usually wins over node-based + "better" big-O. Branch-predictable linear scan through one cache line ≈ one L2 miss.

---

## 7. Iterator Invalidation Master Table

|Container|Insert → iterators|Insert → ptrs/refs|Erase → iterators|Erase → ptrs/refs|
|---|---|---|---|---|
|vector|all IF realloc; else at/after point|same|at/after point|at/after point|
|deque|ends: ALL · middle: all|ends: NONE · middle: all|ends: only erased¹ · middle: all|ends: only erased · middle: all|
|list / forward_list|none|none|only erased|only erased|
|set / map (tree)|none|none|only erased|only erased|
|unordered_*|on rehash: all|**never**|only erased|only erased|
|string|like vector|like vector|like vector|like vector|

¹ deque erase at ends: implementations may invalidate the past-the-end iterator too — don't cache `end()`.

Erase-in-loop idiom (R2Q3 — your Category-E instance): `it = c.erase(it)` **and** no `++it` that pass; or C++20 `std::erase_if(c, pred)`.

---

## 8. Iterator Categories (what `it + 5` compiles on)

contiguous (vector, array, string, span) ⊃ random-access (+deque) ⊃ bidirectional (+list, set, map) ⊃ forward (+forward_list, `unordered_*`) ⊃ input/output.

Consequences: `std::sort` needs random-access (can't sort a `list` with it — `list::sort` member exists); `std::lower_bound` on a `set` iterator is O(n) — use the **member** `set::lower_bound` O(log n). Generic: prefer member versions (`find`, `lower_bound`, `count`) on associative containers over the free algorithms.

---

## 9. Complexity Cheat-Row (memorize cold)

|Op|vector|deque|list|map/set|unordered|
|---|---|---|---|---|---|
|access by index|O(1)|O(1)|—|—|—|
|find by key/value|O(n)|O(n)|O(n)|O(log n)|O(1) avg / O(n) worst|
|insert back|O(1)*|O(1)*|O(1)|—|—|
|insert front|O(n)|O(1)*|O(1)|—|—|
|insert middle/keyed|O(n)|O(n)|O(1)†|O(log n)|O(1) avg|
|erase keyed/at it|O(n)|O(n)|O(1)|O(log n)‡|O(1) avg|
|min/max|O(n)|O(n)|O(n)|O(1)-ish|O(n)|
|predecessor/successor|—|—|—|O(log n)|✗ impossible|

* amortized · † position already known · ‡ amortized O(1) given the iterator

---

## 10. ⚡ HFT Addendum

- **Allocation is the enemy:** `reserve()` everything; node containers allocate per element → prefer flat structures; know `pmr::` (polymorphic allocators / monotonic_buffer_resource) exists as the standard answer to "how do you avoid heap on the hot path."
- **`unordered_map` is often banned on hot paths** (chaining = pointer chase per lookup); open-addressing maps (absl::flat_hash_map style) or flat sorted vectors instead. Being able to _say why_ (cache lines, not big-O) is the interview point.
- **Contiguity → prefetching + SIMD.** vector-of-structs vs struct-of-vectors (AoS vs SoA) is the follow-up question.
- **Stability requirements drive choices:** references into unordered_map/list/map survive anything except their own erase — sometimes the entire reason those containers get picked despite the cache cost.