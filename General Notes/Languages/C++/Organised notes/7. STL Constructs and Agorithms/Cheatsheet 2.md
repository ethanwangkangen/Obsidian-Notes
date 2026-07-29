# C++ STL Master Cheat Sheet — Containers, Algorithms, Idioms

Target: HFT-style OA / OOP interview prep. Assumes C++17 baseline, C++20 features flagged.

---

## 1. Container Map (one-glance overview)

| Container                      | Underlying                 | Ordered?         | Lookup               | Insert/Erase                            | Random access   | Use when                                                                 |
| ------------------------------ | -------------------------- | ---------------- | -------------------- | --------------------------------------- | --------------- | ------------------------------------------------------------------------ |
| `vector<T>`                    | contiguous array           | insertion order  | O(n) (`find`)        | back: amortized O(1); middle: O(n)      | O(1)            | Default. Cache-friendly. 95% of problems.                                |
| `deque<T>`                     | chunked blocks + index map | insertion order  | O(n)                 | both ends: amortized O(1); middle: O(n) | O(1) (2 derefs) | Need push/pop at _both_ ends (sliding window, monotonic deque, BFS 0-1). |
| `list<T>`                      | doubly linked              | insertion order  | O(n)                 | O(1) _given iterator_                   | O(n)            | Need O(1) splice/erase mid-sequence with stable iterators (LRU cache).   |
| `forward_list<T>`              | singly linked              | insertion order  | O(n)                 | O(1) after known node                   | O(n)            | Almost never in interviews.                                              |
| `array<T,N>`                   | fixed C array              | —                | O(n)                 | —                                       | O(1)            | Compile-time size, zero overhead.                                        |
| `string`                       | contiguous (SSO)           | —                | `find` O(n·m)        | like vector                             | O(1)            | Text. Small strings avoid heap (SSO ~15 chars).                          |
| `set/map`                      | red-black tree             | sorted by key    | O(log n)             | O(log n)                                | —               | Sorted order, predecessor/successor queries, range queries.              |
| `multiset/multimap`            | red-black tree             | sorted, dup keys | O(log n)             | O(log n)                                | —               | Sorted with duplicates (interval sweeps, k-largest with removal).        |
| `unordered_set/map`            | hash table (buckets)       | none             | O(1) avg, O(n) worst | O(1) avg                                | —               | Pure membership / key→value, no ordering needed.                         |
| `unordered_multiset`<br>`/map` | hash table                 | none, dup keys   | O(1) avg             | O(1) avg                                | —               | Rare. Counting is usually `unordered_map<K,int>` instead.                |
| `stack<T>`                     | adaptor (default `deque`)  | LIFO             | —                    | O(1)                                    | —               | DFS, monotonic stack, parsing.                                           |
| `queue<T>`                     | adaptor (default `deque`)  | FIFO             | —                    | O(1)                                    | —               | BFS.                                                                     |
| `priority_queue<T>`            | adaptor: heap on `vector`  | heap order       | top O(1)             | push/pop O(log n)                       | —               | Top-k, Dijkstra, merge k lists. **No decrease-key, no iteration.**       |
| `bitset<N>`                    | fixed bits                 | —                | O(1)/bit             | O(1)/bit                                | O(1)            | Bitmask DP, fast set ops on small universe (`count()`, `<<`, `&`).       |

**Adaptors** (`stack`, `queue`, `priority_queue`) expose no iterators. If you need to iterate/inspect, use the underlying container directly (e.g. `multiset` instead of `priority_queue` when you also need arbitrary erase).

---

## 2. Key Methods Per Container

### vector

- `push_back / emplace_back`, `pop_back`, `back()`, `front()`, `[]` (no check) vs `at()` (throws)
- `insert(it, x)`, `erase(it)`, `erase(first, last)` — O(n), shifts tail
- `reserve(n)` (capacity only, no elements) vs `resize(n)` (creates/destroys elements)
- `clear()` — destroys elements, **capacity unchanged**; `shrink_to_fit()` is non-binding
- `data()` — raw pointer; `assign(n, val)`
- 2D init: `vector<vector<int>> g(rows, vector<int>(cols, 0));`
- **`vector<bool>` is a bitset in disguise**: `operator[]` returns a proxy, not `bool&`. `auto& x = vb[0]` fails. Use `vector<char>` or `deque<bool>` if you need real references.

### deque

- Everything vector has, plus `push_front / pop_front / emplace_front`
- No `reserve()`, no `data()` (not contiguous)

### list

- `splice(pos, other, it)` — O(1) move of node(s) between/within lists, iterators stay valid → the whole point of `list` (LRU: `lst.splice(lst.begin(), lst, it)` moves node to front)
- `sort()` **member** (merge sort, O(n log n)) — `std::sort` won't compile (needs random access)
- `merge`, `unique`, `remove/remove_if`, `reverse` — member versions exist for the same reason

### set / map (and multi- variants)

- `insert(x)` → `pair<iterator,bool>` (set/map); multiset returns just iterator
- `emplace`, `emplace_hint(it, ...)` — hint makes insert O(1) amortized if position is right
- `find`, `count`, `contains` (C++20)
- **Member** `lower_bound(k)` (first ≥ k), `upper_bound(k)` (first > k), `equal_range(k)` — O(log n)
- `erase(key)` → returns count erased (**multiset: erases ALL copies!**); `erase(it)` → erases one, returns next iterator
- map only: `operator[]` (**inserts default if missing — never use for read-only lookup, and can't use on const map**), `at()` (throws), `insert_or_assign(k, v)`, `try_emplace(k, args...)` (C++17, no-op if key exists, doesn't construct value unnecessarily)
- `extract(it/key)` (C++17) — remove node without dealloc, mutate `.key()`, re-insert: the only way to "change a key" without copying value
- `merge(other)` — steal nodes from another tree, O(n log n)
- Predecessor/successor: `prev(it)` / `next(it)` after a `lower_bound` — check `begin()`/`end()` bounds first

### unordered_set / unordered_map

- Same insert/find/erase/count/contains API, **no** lower_bound/upper_bound (no order)
- Do `insert({k, v})` instead of `m[k] = v`, since latter won't work if no default constructor
- `reserve(n)` before bulk insert → avoids rehashes (real speed win in OAs)
- `load_factor()`, `max_load_factor(f)`, `rehash(buckets)`
- Custom key: provide hash + equality (see §7)
- Worst case O(n) per op — adversarial inputs on Codeforces; on OAs, fine

### priority_queue

- `top()`, `push/emplace`, `pop()` — **`pop()` returns void**; read `top()` first
- Default = **max-heap** (`less<T>`). Min-heap: `priority_queue<T, vector<T>, greater<T>>`
- Lazy deletion pattern (no decrease-key): push duplicates, on pop skip stale entries by checking against current best (`if (d != dist[u]) continue;` in Dijkstra)

### string

- `substr(pos, len)` — **O(len), makes a copy**; `find(s)` → `size_t`, compare against `string::npos` (not `-1` into an int — npos is huge unsigned)
- `stoi/stol/stoll/stod`, `to_string`
- Split: `stringstream ss(s); string tok; while (getline(ss, tok, ','))` or `while (ss >> tok)`
- Build strings with `+=` / `push_back` (amortized O(1)), **never** `s = s + c` in a loop (O(n²))
- `string_view` (C++17): non-owning window, O(1) substr — but dangles if the source dies. Never return a view of a local.

---

## 3. Iterator & Reference Invalidation — MASTER TABLE

The #1 source of OA bugs and MCQ questions.

| Container               | Operation                         | Iterators                               | References/Pointers                      |
| ----------------------- | --------------------------------- | --------------------------------------- | ---------------------------------------- |
| **vector**              | `push_back` with spare capacity   | only `end()` invalid                    | valid                                    |
|                         | `push_back` triggering realloc    | **ALL invalid**                         | **ALL invalid**                          |
|                         | `insert/erase` at position p      | invalid from p onward (all, if realloc) | same                                     |
|                         | `reserve/resize` growing capacity | ALL invalid                             | ALL invalid                              |
| **deque**               | `push/pop` at **either end**      | **ALL iterators invalid**               | **references still VALID** ← classic MCQ |
|                         | insert/erase in middle            | ALL invalid                             | ALL invalid                              |
| **list / forward_list** | any insert                        | nothing invalidated                     | nothing                                  |
|                         | erase                             | only the erased element                 | only erased                              |
| **set/map (tree)**      | insert                            | nothing                                 | nothing                                  |
|                         | erase                             | only erased element                     | only erased                              |
| **unordered_***         | insert **without** rehash         | nothing                                 | nothing                                  |
|                         | insert **causing rehash**         | **ALL iterators invalid**               | **references still VALID** ← classic MCQ |
|                         | erase                             | only erased                             | only erased                              |
| **string**              | like vector                       | like vector                             | like vector                              |

Consequences to internalize:

- Never hold `it`/`&v[i]`/`v.data()` across a `push_back` on a vector unless you `reserve`d up front.
- `for (auto x : v) if (...) v.push_back(...)` — UB when realloc happens mid-range-for.
- Pointers into a `deque` survive front/back growth (why `deque` is safe for stable-address arenas); iterators don't.
- Node-based containers (list, set, map) = stable everything. That's what you're buying with the cache misses.

---

## 4. Iterator Categories (what algorithms demand)

`input → forward → bidirectional → random access → contiguous (C++17)`

| Category      | Containers                    | Enables                                                 |
| ------------- | ----------------------------- | ------------------------------------------------------- |
| Random access | vector, deque, array, string  | `std::sort`, `nth_element`, `it + n`, `it2 - it1`, `[]` |
| Bidirectional | list, set, map                | `reverse`, `prev(it)`, `stable_partition`               |
| Forward       | `forward_list`, `unordered_*` | `find`, `remove`, single-pass+ algorithms               |

- `std::sort(lst.begin(), lst.end())` **does not compile** for `list` → use `lst.sort()`.
- `it + 5` doesn't compile on map iterators → `std::advance(it, 5)` (O(n) there) or `std::next(it, 5)`.
- `std::distance(a, b)`: O(1) random access, O(n) otherwise.
- **Member vs free binary search**: `m.lower_bound(k)` is O(log n); `std::lower_bound(m.begin(), m.end(), ...)` is O(n) advances on tree iterators (still O(log n) comparisons, but linear traversal).
	- Tree containers = **set, multiset, map, multimap**
- **Always use the member function on set/map.** Free `std::lower_bound` is for sorted vectors/arrays.

---

## 5. Algorithm Toolbox (by task)

All in `<algorithm>` / `<numeric>` unless noted. Ranges are `[first, last)` half-open — always.

### Searching (sorted range required where noted)

|Algorithm|Complexity|Notes|
|---|---|---|
|`find(f,l,x)` / `find_if(f,l,pred)`|O(n)|returns iterator, `== l` if absent|
|`lower_bound(f,l,x)`|O(log n)*|first element **≥ x** (sorted). *log only with random access|
|`upper_bound(f,l,x)`|O(log n)*|first element **> x**|
|`equal_range(f,l,x)`|O(log n)*|`{lower, upper}` pair; count = distance|
|`binary_search(f,l,x)`|O(log n)*|bool only — usually you want lower_bound anyway|
|`partition_point(f,l,pred)`|O(log n)*|first element where pred is false — generalized binary search on a predicate|
|`count(f,l,x)` / `count_if`|O(n)||
|`all_of / any_of / none_of`|O(n)|short-circuit|
|`adjacent_find(f,l)`|O(n)|first pair of equal neighbors (dup detection in sorted range)|
|`mismatch(f1,l1,f2)` / `equal`|O(n)||
|`search(f,l, sf,sl)`|O(n·m)|subsequence find|

Index from iterator on vector: `int i = it - v.begin();` (or `distance(v.begin(), it)`). Count of x in sorted vector: `upper_bound(...) - lower_bound(...)`.

**lower_bound with custom sort order must use the SAME comparator** you sorted with: `sort(v.begin(), v.end(), cmp); auto it = lower_bound(v.begin(), v.end(), x, cmp);` Descending sorted → `lower_bound(v.begin(), v.end(), x, greater<>())`.

### Sorting & selection

|Algorithm|Complexity|Notes|
|---|---|---|
|`sort(f,l,cmp?)`|O(n log n)|introsort; unstable; comparator must be **strict weak ordering** (use `<`, never `<=` → UB/segfault)|
|`stable_sort`|O(n log n) w/ extra mem, else O(n log²n)|preserves relative order of equals — "sort by score, keep input order on ties"|
|`partial_sort(f, mid, l)`|O(n log k)|smallest k sorted at front|
|`nth_element(f, nth, l)`|**O(n) avg**|element at `nth` is what it would be if sorted; left ≤ it ≤ right (unordered). Kth smallest / median without full sort|
|`is_sorted / is_sorted_until`|O(n)||
|`partition(f,l,pred)`|O(n)|true-group first, returns boundary; `stable_partition` keeps order|
|`min_element / max_element / minmax_element (f,l,cmp?)`|O(n)|return **iterators** — deref them. minmax in ~1.5n comparisons|
|`min/max({a,b,c})`, `clamp(x, lo, hi)`|O(1)|initializer-list overloads exist|

Top-k idioms, know all three trade-offs:

- `nth_element` then look at prefix — O(n), unordered k
- `partial_sort` — O(n log k), sorted k
- size-k **min**-heap over a stream — O(n log k), doesn't need all data in memory (say this in interviews)

### Modifying

|Algorithm|Notes|
|---|---|
|`remove(f,l,x)` / `remove_if`|**Doesn't erase!** Shifts kept elements forward, returns new logical end. Container size unchanged → must pair with `erase` (§6)|
|`unique(f,l)`|removes **consecutive** duplicates only → sort first for global dedupe; same erase pairing|
|`reverse(f,l)`|O(n)|
|`rotate(f, mid, l)`|O(n); `mid` becomes the new first element. Left-rotate by k: `rotate(v.begin(), v.begin()+k, v.end())`|
|`fill(f,l,x)` / `generate(f,l,fn)`||
|`transform(f,l,out,fn)`|map: `transform(v.begin(), v.end(), v.begin(), [](int x){return x*x;});`|
|`copy / copy_if(f,l,out,pred?)`|with `back_inserter(dst)` to append: `copy_if(v.begin(), v.end(), back_inserter(evens), pred);`|
|`swap_ranges`, `iter_swap`||
|`replace / replace_if`||
|`shuffle(f,l,rng)`|needs `mt19937 rng(seed)`; `random_shuffle` is removed|
|`sample(f,l,out,k,rng)`|C++17 reservoir sampling|

### Numeric (`<numeric>`)

|Algorithm|Notes|
|---|---|
|`accumulate(f,l,init)`|**init's TYPE sets the accumulator type**: `accumulate(v.begin(), v.end(), 0)` overflows int on big sums → use `0LL`. (Your Category-G classic.)|
|`accumulate(f,l,init,op)`|fold with custom op, e.g. `multiplies<>()`|
|`reduce` (C++17)|like accumulate but order-unspecified/parallelizable — op must be associative|
|`iota(f,l,start)`|fill 0,1,2,... — index arrays for argsort|
|`partial_sum(f,l,out)`|prefix sums — same `0` vs `0LL` trap via value_type|
|`adjacent_difference(f,l,out)`|deltas; first output = first input|
|`inner_product(f1,l1,f2,init)`|dot product|
|`gcd(a,b)`, `lcm(a,b)`|C++17, free functions|

Argsort idiom (sort indices, keep data intact):

```cpp
vector<int> idx(n);
iota(idx.begin(), idx.end(), 0);
sort(idx.begin(), idx.end(), [&](int a, int b){ return v[a] < v[b]; });
```

### Set operations (require SORTED ranges; outputs sorted)

`set_union`, `set_intersection`, `set_difference`, `set_symmetric_difference`, `includes` — all O(n+m), all take an output iterator:

```cpp
vector<int> out;
set_intersection(a.begin(), a.end(), b.begin(), b.end(), back_inserter(out));
```

`merge(f1,l1,f2,l2,out)` — merge two sorted ranges O(n+m). `inplace_merge(f,mid,l)` — merge two sorted halves of one container (merge-sort building block).

### Heap algorithms (raw heap on a vector — when priority_queue is too rigid)

`make_heap(f,l,cmp?)` O(n) — note: linear, not n log n. `push_heap` (after push_back), `pop_heap` (moves top to back, then pop_back), `sort_heap` O(n log n), `is_heap`. Default comparator `less` = max-heap, same convention as priority_queue.

### Permutations

`next_permutation(f,l)` / `prev_permutation` — O(n) per step, in-place, returns false after wrapping past the last permutation. **Start from sorted** to enumerate all n! :

```cpp
sort(v.begin(), v.end());
do { /* use v */ } while (next_permutation(v.begin(), v.end()));
```

---

## 6. Core Idioms (memorize cold)

**Erase–remove** (vector/deque/string):

```cpp
v.erase(remove(v.begin(), v.end(), 3), v.end());
v.erase(remove_if(v.begin(), v.end(), [](int x){ return x < 0; }), v.end());
// C++20 one-liners:
erase(v, 3);
erase_if(v, [](int x){ return x < 0; });
```

Why: `remove` can't change container size (it only sees iterators). Forgetting the `erase` wrapper leaves garbage at the tail — favorite MCQ.

**Sort + unique + erase** (global dedupe):

```cpp
sort(v.begin(), v.end());
v.erase(unique(v.begin(), v.end()), v.end());
```

Also the basis of coordinate compression: dedupe, then `lower_bound` gives each value its rank.

**Erase while iterating** — the ONLY correct patterns:

```cpp
// map/set/list: erase returns the next valid iterator (C++11+)
for (auto it = m.begin(); it != m.end(); ) {
    if (cond(*it)) it = m.erase(it);
    else ++it;
}
// vector: same pattern works, or just use erase_if / erase-remove
```

`for (auto& x : m) if (...) m.erase(x.first);` = UB. Range-for + mutation of structure = UB, always.

**Multiset: erase ONE copy, not all:**

```cpp
ms.erase(ms.find(x));   // one copy (find returns some iterator to x)
ms.erase(x);            // ALL copies — returns count removed
```

**Map read without inserting:**

```cpp
auto it = m.find(k);
if (it != m.end()) use(it->second);
// m[k] would insert a default-constructed value — mutates the map, breaks const, skews .size()
```

**Sliding-window max — monotonic deque** (why deque, not stack): pop stale indices from front, pop dominated values from back, both O(1) → O(n) total.

**Custom comparator + tie-breaks with `tie`:**

```cpp
sort(people.begin(), people.end(), [](const P& a, const P& b){
    return tie(a.age, a.name) < tie(b.age, b.name);   // age asc, then name asc
});
// mixed directions: return tie(b.score, a.name) < tie(a.score, b.name); // score DESC, name asc
```

**Streaming top-k (k largest) — size-k MIN-heap:**

```cpp
priority_queue<int, vector<int>, greater<int>> pq;
for (int x : stream) {
    pq.push(x);
    if (pq.size() > k) pq.pop();   // evict the smallest of the current top-k
}
```

**Dijkstra skeleton (lazy deletion):**

```cpp
priority_queue<pair<long long,int>, vector<pair<long long,int>>, greater<>> pq;
pq.push({0, src}); dist[src] = 0;
while (!pq.empty()) {
    auto [d, u] = pq.top(); pq.pop();
    if (d != dist[u]) continue;              // stale entry — the "decrease-key" substitute
    for (auto [v, w] : adj[u])
        if (d + w < dist[v]) { dist[v] = d + w; pq.push({dist[v], v}); }
}
```

**LRU cache = `list` + `unordered_map<key, list::iterator>`** — works precisely because list iterators are never invalidated by other insertions/erasures; `splice` moves a node to the front in O(1).

**Predecessor/successor queries — ordered set:**

```cpp
auto it = s.lower_bound(x);          // first >= x
if (it != s.begin()) auto pred = *prev(it);   // greatest < x
```

**Interval scheduling / sweep** — `multimap<time, event>` or sort a vector of pairs; `map<int,int>` of boundary deltas + prefix sum for "max concurrent intervals".

---

## 7. Comparators & Hashing — the rules

**Strict weak ordering (required by sort, set, map, pq, binary search):**

1. Irreflexive: `cmp(a,a)` is false → **use `<`, never `<=`**. `<=` violates this and gives UB (real-world symptom: `std::sort` segfaults or infinite-loops).
2. Antisymmetric + transitive, and "equivalence" (`!cmp(a,b) && !cmp(b,a)`) must itself be transitive.
3. Comparator must be **stateless w.r.t. the elements being sorted** — comparing against data you're mutating mid-sort = UB.

**Equivalence, not equality, in ordered containers:** a `set<T,Cmp>` treats a and b as duplicates iff `!cmp(a,b) && !cmp(b,a)`. So `set<int, decltype(cmp)>` with `cmp = |a|<|b|` keeps only one of {3, -3}. MCQ favorite.

**priority_queue direction (the eternal gotcha):**

- `less<T>` (default) → **MAX-heap**. `greater<T>` → min-heap.
- Mnemonic: pq sorts like `sort` would, then serves you from the **back**.
- Custom: `cmp(a,b) == true` means a sits **further from the top** than b.

```cpp
auto cmp = [](const T& a, const T& b){ return a.cost > b.cost; };  // min-heap by cost
priority_queue<T, vector<T>, decltype(cmp)> pq(cmp);
```

**Custom-ordered set/map with a lambda:**

```cpp
auto cmp = [](const P& a, const P& b){ return a.x < b.x; };
set<P, decltype(cmp)> s(cmp);      // pre-C++20 must pass cmp to ctor
```

Or write a struct with `bool operator()(a,b) const` — no ctor argument needed.

**Sort descending, three equivalent ways:** `greater<>()` comparator · sort then `reverse` · `sort(v.rbegin(), v.rend())`.

**Heterogeneous (transparent) lookup — avoid temporary constructions:**

```cpp
set<string, less<>> s;             // less<void>, has is_transparent
s.find("abc"sv);                   // no temporary std::string built
// C++20: unordered version needs a transparent hash+eq pair
```

**Hashing user types for unordered containers:**

```cpp
struct PairHash {
    size_t operator()(const pair<int,int>& p) const {
        return hash<long long>()(((long long)p.first << 32) ^ (unsigned)p.second);
    }
};
unordered_set<pair<int,int>, PairHash> seen;
```

No std::hash for `pair`/`tuple`/`vector` out of the box — you must supply one (or use `set`, or encode to `long long` / string key). Requirement: `a == b ⇒ hash(a) == hash(b)`; hash and `==` must agree.

---

## 8. Container Choice & Combination — decision notes

|Need|Reach for|
|---|---|
|"kth largest / top-k stream"|size-k min-heap (`pq` + `greater`)|
|"kth smallest, all data in memory, once"|`nth_element` — O(n)|
|top-k **with arbitrary deletion** (order book!)|`multiset` (`*ms.rbegin()` = max, `ms.erase(ms.find(x))`) — pq can't delete middle|
|membership only, no order|`unordered_set` (+`reserve`)|
|"smallest element ≥ x", floor/ceiling, range [lo,hi)|`set/map` + member `lower_bound/upper_bound`|
|sorted once, then many lookups, no more inserts|**sorted `vector` + `std::lower_bound`** — beats `set` on cache/memory; say this trade-off out loud|
|counts / frequency|`unordered_map<K,int>`; `map` only if you also need sorted iteration|
|sliding window max/min|monotonic `deque`|
|next-greater-element, spans|monotonic `stack` (or vector as stack)|
|BFS / FIFO|`queue`; 0-1 BFS → `deque` (push_front for 0-edges)|
|stable addresses while growing|`deque` (refs survive end-growth) or node containers|
|O(1) reorder of live elements (LRU/MRU)|`list` + hash map of iterators|
|merge k sorted lists|min-heap of (value, list-index, iter)|
|median of a stream|two heaps (max-heap lower half, min-heap upper half), rebalance to size diff ≤ 1|
|set algebra on int universe ≤ ~10⁶|`bitset` — `&`, `|
|dedupe + rank (coordinate compression)|sort + unique + erase, then `lower_bound` for index|

**vector vs list, the interview answer:** vector wins almost always despite worse big-O for middle insertion — contiguous memory → cache lines + prefetching; list pays a cache miss per node plus 2 pointers overhead. Choose list only when you need iterator/reference stability or O(1) splice given an existing iterator (you already paid O(n) to find it otherwise).

**map vs unordered_map:** unordered for pure key→value speed; map when you need ordered iteration, lower_bound-style queries, or worst-case O(log n) guarantees. unordered iterators die on rehash; tree iterators are stable.

---

## 9. Rapid-Fire Gotchas (pre-OA checklist)

1. `remove/remove_if/unique` don't shrink the container — pair with `erase`.
2. `map::operator[]` inserts on read-miss; use `find`/`at`/`contains`.
3. `multiset::erase(value)` nukes ALL copies; `erase(find(value))` for one.
4. `accumulate(..., 0)` sums in `int` → overflow; use `0LL`. Same trap: `partial_sum` into `vector<int>`.
5. pq default is a MAX-heap; `greater` for min. `pop()` returns void.
6. Comparators use strict `<`; `<=` is UB in sort.
7. `std::sort` needs random access → `list.sort()` member instead; member `lower_bound` on set/map, free-function on sorted vectors.
8. Iterator invalidation: vector realloc kills everything; deque end-ops kill iterators but not references; unordered rehash kills iterators but not references; node containers kill nothing but the erased.
9. Never `push_back` into the vector you're range-for-ing over.
10. `v.size()` is **unsigned**: `v.size() - 1` underflows when empty; `for (int i = 0; i < (int)v.size() - 1; ++i)` — cast or use `ssize` (C++20).
11. `string::find` returns `npos` (huge unsigned), not -1 — compare with `string::npos`.
12. `front()/back()/top()/pop()` on an empty container = UB, not an exception. Guard with `empty()`.
13. `vector<bool>` proxy references — avoid when you need `bool&` / `auto&`.
14. `reserve` ≠ `resize`: reserve gives capacity (indexing into it is still UB); resize creates elements.
15. Set-operation algorithms and `binary_search`-family require sorted input; `unique` requires adjacency (sort first).
16. `end()` is one-past-last — never dereference; ranges are half-open everywhere.
17. `nth_element`/`partition` leave the rest unordered — don't assume sortedness after.
18. Lambda captures in comparators: capture by `&` for big data (`[&](int a, int b){ return v[a] < v[b]; }`), and never mutate what you compare on.
19. `auto [k, val] : m` copies; `auto& [k, val] : m` to mutate values (key stays const).
20. `next_permutation` needs a sorted start to enumerate all permutations.