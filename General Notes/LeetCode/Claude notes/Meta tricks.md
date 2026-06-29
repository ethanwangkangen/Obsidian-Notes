# Meta-Tricks & Problem Transformation — Comprehensive Reference

These are technique-agnostic patterns that transform hard problems into easier ones, plus a decision framework for choosing the right technique.

---

## Problem Transformation Tricks

### Complement / Inverse Thinking

"Minimum to remove" = total − "maximum to keep."

This reframe often reveals a cleaner subproblem:

- "Min deletions to avoid overlap" = total − max non-overlapping → greedy interval scheduling.
- "Min operations to reduce X from ends" = total length − longest subarray with sum = total - X → sliding window.
- "Min characters to delete for no repeated adjacent" → think about max characters to KEEP.

### Algebraic Rearrangement

Rearrange the problem's equation to reveal a standard form.

**Target Sum:** assign +/- to each number to reach target. Let P = positive set, N = negative set. P - N = target, P + N = total → P = (total + target) / 2 → subset sum (0/1 knapsack).

**Subarray with equal 0s and 1s:** replace 0 with -1, then it's "subarray sum equals 0" → prefix sum + hashmap.

**Matrix cell sum to target:** fix two rows, compute column prefix differences, then "subarray sum equals K" on the 1D column array.

### Reverse Processing

If the problem removes things (edges, nodes, elements), it's often hard to maintain structure. Reverse: ADD things in reverse order. Connectivity increases monotonically with additions, which Union-Find handles naturally.

LC 803 (Bricks Falling When Hit — reverse: add bricks back with Union-Find).

### Circular Array → Double the Array

For circular problems, conceptually concatenate the array with itself. Apply linear techniques on the doubled array, restricting indices to [i, i+n-1].

LC 503 (Next Greater Element II), LC 213 (House Robber II — but two-run approach is cleaner).

Alternative: use modular arithmetic `index % n`.

### "Fix One Dimension, Solve the Other"

When a problem has multiple axes, fix one and apply a known technique on the other.

- **2D max sum rectangle:** fix top and bottom rows (O(n²) pairs). Compute column prefix sums for each pair. Apply Kadane's on columns → O(n²×m).
- **3Sum:** fix one element, two-pointer on the remaining → O(n²).
- **Russian Doll Envelopes:** sort by width, LIS on height → O(n log n).
- **Skyline problem:** process buildings sorted by x-coordinate, use a heap for heights.

### Virtual / Dummy Nodes

**Linked lists:** dummy head node eliminates null-head edge cases.

**Graphs:** add a virtual source connected to all real sources. Turns multi-source BFS/Dijkstra into single-source.

**Trees:** virtual root for a forest (connect all tree roots to a single virtual root).

### Coordinate Compression

When values are huge but only relative ORDER matters, map values to [0, n). Sort unique values, replace each with its sorted index.

**When to use:** before using a BIT (Fenwick tree) or segment tree. These are indexed by value and can't handle values up to 10^9 directly.

LC 315 (Count of Smaller Numbers After Self), LC 493 (Reverse Pairs).

### Contribution Counting

Instead of iterating over all O(n²) subarrays/subsets, count each element's contribution across all subarrays.

For each element, determine how many subarrays it's the min/max/sole-unique of. Use monotonic stack for nearest boundaries.

LC 907 (Sum of Subarray Minimums), LC 828 (Count Unique Characters of All Substrings).

### Think from the Answer

Instead of building toward the answer, ASSUME the answer and verify.

- Binary search on answer: "what's the smallest V such that canDo(V) is true?"
- Reverse verification: "if the answer is X, what constraints must hold?" Check those constraints.

### Process Events, Not Elements

Convert objects into a sorted list of events and sweep through.

- Intervals → (start, +1), (end, -1) events.
- Rectangles → vertical sweep line with horizontal events.
- Any "how many overlap at time T" → event processing.

---

## Decision Flowchart for Technique Selection

When you see a new problem, run through these questions IN ORDER:

### Step 1: What is the answer type?

|Answer type|Likely technique|
|---|---|
|A single number (count, min, max)|DP, greedy, binary search, math|
|A yes/no (feasibility)|DP, greedy, binary search on answer, BFS|
|A list of all solutions|Backtracking|
|A sequence/ordering|Topological sort, greedy, DP|
|Shortest path / min steps|BFS (unweighted), Dijkstra (weighted)|

### Step 2: What is the input structure?

|Input|Consider|
|---|---|
|Sorted array|Binary search, two pointers|
|Unsorted array, contiguous subarray|Sliding window, prefix sum, Kadane's, monotonic stack/deque|
|Unsorted array, subsequence|DP, binary search (LIS)|
|Array with n ≤ 20|Bitmask DP|
|String pair|2D DP (LCS, edit distance)|
|Tree|Recursive DFS, BFS for levels|
|Graph (explicit or implicit)|BFS, DFS, Dijkstra, topo sort, Union-Find|
|Intervals|Sort + sweep, greedy, DP + binary search|
|Grid|BFS/DFS (connectivity), DP (paths), prefix sum (sums)|

### Step 3: Check the monotonicity

**For subarray problems:** "If I expand the window, does the constraint monotonically worsen?"

- YES → sliding window.
- NO (negative numbers, non-monotonic constraint) → prefix sum + hashmap or DP.

**For optimization problems:** "Is the feasibility monotonic in the answer value?"

- YES → binary search on answer.
- NO → DP or other approaches.

### Step 4: Greedy or DP?

Ask: "If I make the locally best choice now, can it EVER hurt me later?"

- NO → greedy. Try to formulate the exchange argument.
- YES → DP. Define the state.
- UNSURE → try greedy, test against brute force on small inputs.

### Step 5: Choose the specific pattern

Match against the patterns in the individual technique documents. Common matches:

|Problem description|Pattern|
|---|---|
|"Subarray sum = K" (negatives possible)|Prefix sum + hashmap|
|"Longest subarray with at most K [something]"|Sliding window|
|"Count subarrays with exactly K [something]"|atMost(K) − atMost(K−1)|
|"Remove from both ends"|Sliding window on the middle (complement)|
|"Next greater/smaller element"|Monotonic stack|
|"Sliding window min/max"|Monotonic deque|
|"K-th largest"|Min-heap of size K|
|"Median in stream"|Two heaps|
|"Merge K sorted things"|Min-heap|
|"Max non-overlapping intervals"|Sort by end, greedy|
|"Max value from non-overlapping intervals"|Sort by end, DP + binary search|
|"Max overlap count"|Line sweep (events)|
|"Minimize the maximum / Maximize the minimum"|Binary search on answer|
|"Shortest path, unweighted"|BFS|
|"Shortest path, weights 0/1"|0-1 BFS (deque)|
|"Shortest path, non-negative"|Dijkstra|
|"Dependencies / ordering"|Topological sort|
|"Connected components / groups"|DFS / BFS / Union-Find|
|"LIS"|Patience sort (binary search)|
|"LCS / edit distance"|2D string DP|
|"Assign n to n (n ≤ 20)"|Bitmask DP|
|"Buy/sell stock with states"|State machine DP|
|"Merge/split a range optimally"|Interval DP|
|"Items with weight, maximize value"|Knapsack DP|
|"Generate all X"|Backtracking|
|"Prefix search / autocomplete"|Trie|
|"Max XOR"|Binary Trie|

---

## Common Pitfalls

**Sliding window with negatives:** won't work. The monotonicity breaks. Use prefix sum + hashmap.

**Greedy when DP is needed:** always test greedy against brute force. Coin change [1,3,4] amount 6 is the classic counterexample.

**Wrong binary search boundary:** using `lo = mid` causes infinite loops with integer division. Always use `lo = mid + 1` or `hi = mid`.

**DP base case errors:** the most common DP bug. Think carefully about what dp[0], dp[0][0], or dp[empty] should be. Is it 0, 1, INF, -INF, true, or false?

**Off-by-one in indices:** "first i elements" (1-indexed DP) vs. "ending at index i" (0-indexed). Be consistent within a problem.

**Forgetting to handle duplicates in backtracking:** sort the input first, skip `nums[i] == nums[i-1]`.

**Using DFS for shortest path:** DFS finds A path, not the SHORTEST path. Use BFS for unweighted shortest path.

**Confusing LCS (subsequence) with longest common substring:** LCS allows gaps; substring requires contiguity. The DP recurrences are different (LCS takes max of skip-one-side; substring resets to 0 on mismatch).