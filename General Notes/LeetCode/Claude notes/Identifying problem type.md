# Technique Identification Guide

How to look at a problem and determine the right approach. Run through these checks in order.

---

## Step 0: Read the Constraints First

The constraint on n is often the single strongest signal. It tells you the maximum acceptable time complexity, which immediately eliminates most techniques.

| Constraint | Max complexity    | Techniques in play                                                           |
| ---------- | ----------------- | ---------------------------------------------------------------------------- |
| n ≤ 20     | O(2^n · n)        | Bitmask DP, backtracking with pruning, meet-in-the-middle                    |
| n ≤ 500    | O(n³)             | Interval DP, Floyd-Warshall, 2D DP with a split-point loop                   |
| n ≤ 5000   | O(n²)             | 2D DP (LCS, edit distance, grid), pairwise two pointers                      |
| n ≤ 10^5   | O(n log n)        | Sorting + greedy, binary search, heap, segment tree / BIT, merge sort tricks |
| n ≤ 10^6   | O(n)              | Sliding window, two pointers, prefix sum, linear DP, monotonic stack/deque   |
| n ≤ 10^9   | O(log n) or O(√n) | Binary search, math, digit DP                                                |

**Example:** n = 16 and the problem says "partition into groups" → bitmask DP. You don't even need to read further. n = 10^5 and it says "subarray" → sliding window or prefix sum, not O(n²) DP.

---

## Step 1: What Does the Problem Ask For?

### "Count the number of ways..."

→ **DP** (almost always). Backtracking is too slow unless n is tiny. Exception: if the answer has a closed-form formula → **math / combinatorics**.

### "Find the minimum / maximum cost, value, length..."

→ **DP** or **greedy**. Check: can a local choice ever hurt you later? No → greedy. Yes → DP. → If "minimize the maximum" or "maximize the minimum" → **binary search on answer**.

### "Is it possible to...?" (feasibility)

→ **DP** (can you reach this state?), **BFS** (can you reach this node?), or **greedy**. → If the feasibility is monotonic in some parameter → **binary search on answer** + greedy check.

### "Find the shortest path / minimum number of steps..."

→ **BFS** (unweighted), **Dijkstra** (weighted non-negative), **Bellman-Ford** (negative weights or edge count limit). → If the "graph" is implicit (states + transitions) → BFS on state space.

### "Generate / list ALL valid..."

→ **Backtracking.** DP gives counts, backtracking gives the actual solutions.

### "Find the longest / shortest subarray / substring with..."

→ Check monotonicity of the constraint (Step 3 below).

### "Find k-th largest / smallest..."

→ **Heap** (size K), **quickselect**, or **binary search on value**.

### "Find the median..."

→ **Two heaps** (max-heap for left half, min-heap for right half).

---

## Step 2: What Is the Input Structure?

### Sorted array

→ **Binary search** for finding elements. → **Two pointers** for pair/sum problems.

### Unsorted array, question about contiguous subarrays

→ Go to Step 3 (monotonicity check).

### Unsorted array, question about subsequences (non-contiguous)

→ **DP** (LIS, LCS). NOT sliding window (sliding window requires contiguity). → If just checking "is A a subsequence of B" → **two pointers**.

### Array of intervals [start, end]

→ **Sort** (by start for merging, by end for scheduling). → "Max non-overlapping" → **greedy (sort by end)**. → "Max value from non-overlapping" → **DP + binary search**. → "How many overlap at any point" → **line sweep (events)**.

### Tree

→ **Recursive DFS** (recurse left, recurse right, combine). This is the default. → Need level information → **BFS**. → BST → exploit sorted inorder property.

### Graph (explicit nodes/edges)

→ See the algorithm table in Step 1 for shortest path. → Connectivity → **DFS / BFS / Union-Find**. → Dependencies/ordering → **topological sort**. → "Can we 2-color?" → **bipartite check (BFS/DFS)**.

### Grid

→ Connectivity / islands / flood fill → **DFS / BFS**. → Shortest path in grid → **BFS** (unweighted) or **Dijkstra** (weighted). → Count paths / min cost path → **DP**. → Rectangle sums → **2D prefix sum**.

### String(s)

→ Two strings, optimal alignment → **2D DP** (LCS, edit distance). → Pattern matching → **KMP / Rabin-Karp**. → Palindromes → **expand from center** or **interval DP**. → Prefix queries → **Trie**.

### Linked list

→ Pointer manipulation. Three tools: **dummy head**, **fast/slow pointer**, **iterative reversal**.

### n ≤ 20

→ **Bitmask DP.** The constraint is screaming at you.

---

## Step 3: The Monotonicity Check (For Subarray Problems)

This is the single most important decision point for subarray/substring problems. Ask TWO questions:

**Q1: "If my window is VALID and I add one element to the right, can it only stay valid or become invalid?"** (i.e., does expanding always maintain or worsen the condition?)

**Q2: "If my window is INVALID and I remove one element from the left, can it only stay invalid or become valid?"** (i.e., does shrinking always maintain or improve the condition?)

**If BOTH are yes → the constraint is MONOTONIC → sliding window works.**

**If either is no → sliding window BREAKS.** Use prefix sum + hashmap, DP, or another technique.

### Common cases:

|Problem|Monotonic?|Technique|
|---|---|---|
|"Subarray sum ≤ K" (all positive)|Yes: adding increases sum, removing decreases|Sliding window|
|"Subarray sum = K" (negatives possible)|**No**: adding can decrease sum|Prefix sum + hashmap|
|"At most K distinct elements"|Yes: adding can only increase distinct count|Sliding window|
|"All elements unique"|Yes: adding can only add a duplicate|Sliding window|
|"Max − min ≤ K"|Yes (with deque): adding can change max or min|Sliding window + 2 deques|
|"Subarray product ≤ K" (all positive)|Yes: adding increases product|Sliding window|
|"Longest with at most K zeros"|Yes: adding can only increase zero count|Sliding window|

### The "exactly K" escape hatch:

"Count subarrays with exactly K [something]" is NOT directly monotonic. But decompose: **exactly(K) = atMost(K) − atMost(K−1)**. Each `atMost` function IS monotonic → sliding window.

### The "remove from both ends" escape hatch:

"Remove from front and back to achieve X" = "keep a contiguous middle subarray that achieves the complement of X" → sliding window on the middle.

---

## Step 4: Greedy or DP?

If the problem asks for an optimal value with choices at each step, you need to distinguish between greedy and DP. This is often the hardest judgment call.

### Lean toward GREEDY when:

- There's a natural sorting order that "unlocks" the problem.
- At each step, one choice is obviously dominant (e.g., earliest deadline, smallest cost).
- The problem involves intervals, scheduling, or matching.
- You can articulate in one sentence why a swap would never improve things (exchange argument).
- The problem has a "tracker" structure: farthest reach, running surplus, current end.

### Lean toward DP when:

- Making a choice now restricts future choices in complex ways.
- You need to "try both options" (include/exclude, take/skip).
- There's a knapsack-like capacity constraint.
- The problem says "contiguous" and you need optimal over all starting/ending positions.
- Small counterexamples exist for greedy (e.g., coin change with non-standard denominations).

### Quick greedy sanity check:

Mentally construct an adversarial input. "Can I create a case where the greedy choice at step 1 blocks a better overall solution?" If yes → DP. If you can't find one after 30 seconds → probably greedy, but verify with brute force on small inputs if time permits.

---

## Step 5: Pattern Matching — Keyword → Technique Lookup

When the above steps narrow it down but don't pinpoint the exact technique, match keywords:

### Subarray / Substring keywords

|Keyword / phrase|Technique|
|---|---|
|"Longest subarray/substring with constraint"|Sliding window (if monotonic)|
|"Shortest subarray/substring with constraint"|Sliding window (shrink while valid)|
|"Subarray sum equals K" (negatives)|Prefix sum + hashmap|
|"Subarray sum equals K" (positives only)|Sliding window|
|"Maximum subarray sum"|Kadane's algorithm|
|"Maximum product subarray"|Kadane's variant (track min AND max)|
|"Count subarrays with exactly K..."|atMost(K) − atMost(K−1)|
|"Remove from both ends"|Sliding window on the kept middle|
|"Sum of min/max over all subarrays"|Contribution technique + monotonic stack|
|"Range update, then query"|Difference array, then prefix sum|

### Optimization keywords

|Keyword / phrase|Technique|
|---|---|
|"Minimize the maximum"|Binary search on answer|
|"Maximize the minimum"|Binary search on answer|
|"Can you do X with budget/speed/capacity Y?"|Binary search on Y + greedy feasibility check|
|"Minimum cost to go from A to B"|Dijkstra (weighted), BFS (unweighted)|
|"Minimum steps to transform X to Y"|BFS on state space|
|"Minimum operations to make array equal/sorted"|Greedy or math|

### Structure keywords

|Keyword / phrase|Technique|
|---|---|
|"Next greater / next smaller element"|Monotonic stack|
|"Sliding window maximum / minimum"|Monotonic deque|
|"Largest rectangle"|Monotonic stack (histogram technique)|
|"Merge K sorted lists/arrays"|Min-heap|
|"Top K / K-th largest"|Min-heap of size K (or quickselect)|
|"Median of a stream"|Two heaps|
|"Prerequisites / dependencies"|Topological sort|
|"Connected / how many groups / islands"|DFS / BFS / Union-Find|
|"Can split into two groups, no conflict"|Bipartite check|
|"Process deletions on a graph"|Reverse: add edges with Union-Find|
|"Prefix search / autocomplete"|Trie|
|"Maximum XOR"|Binary Trie|
|"Buy and sell stock"|State machine DP|
|"House robber / can't take adjacent"|Linear DP (skip constraint)|
|"Partition into subsets with equal sum"|0/1 Knapsack (target = total/2)|
|"Assign +/- to reach target"|Rearrange algebra → subset sum knapsack|
|"Merge/burst/break a contiguous range"|Interval DP|
|"Longest increasing subsequence"|Patience sort (binary search on tails)|
|"Edit distance / LCS"|2D string DP|
|"Match with wildcards / regex"|2D string DP|
|"Non-overlapping intervals (maximize count)"|Sort by end, greedy|
|"Non-overlapping intervals (maximize value)"|Sort by end, DP + binary search|
|"How many intervals overlap at once"|Line sweep (events)|
|"Distance to nearest X for every cell"|Multi-source BFS|

---

## Step 6: "It Looks Like X but Is Actually Y"

Common misidentifications and how to catch them:

**"Subarray sum = K" → you reach for sliding window** WRONG if negatives exist. Adding an element could decrease the sum, breaking monotonicity. → Prefix sum + hashmap.

**"Longest subsequence" → you reach for sliding window** WRONG. Subsequences aren't contiguous. Sliding window only works on contiguous subarrays/substrings. → DP.

**"Count the number of ways" → you try backtracking** WRONG unless n is tiny. Backtracking enumerates all solutions (exponential). Counting is DP (polynomial). → DP.

**"Shortest path" → you do DFS** WRONG. DFS finds A path, not the SHORTEST. → BFS for unweighted, Dijkstra for weighted.

**"Minimum coins for amount" → you try greedy (take largest coin)** WRONG for non-standard denominations. Coins [1, 3, 4], amount 6: greedy gives 4+1+1 = 3 coins, optimal is 3+3 = 2. → DP (unbounded knapsack).

**"Connected components" → you build a full adjacency list + DFS** POSSIBLY OVERKILL. If you only need connectivity queries and edges arrive dynamically → Union-Find is simpler.

**"Maximum subarray" → you try 2D DP** OVERKILL. Kadane's is O(n) with O(1) space. Always use Kadane's for max subarray sum.

**"Find cycle in directed graph" → you use Union-Find** WRONG. Union-Find detects cycles in UNDIRECTED graphs. For directed graphs → 3-color DFS (white/gray/black).

**"Remove from both ends to get sum X" → you try DP on two ends** OVERCOMPLICATED. Take the complement: keep a contiguous middle with sum = total − X → sliding window. Much simpler.

**"Minimize the maximum distance/time/cost" → you try DP** POSSIBLE but check first: is feasibility monotonic? If "can I do it with max ≤ V" flips from no to yes as V increases → binary search on V with a greedy check. Usually O(n log(answer_range)) which is faster than DP.

**"n ≤ 15 and 'assign items to groups'" → you try backtracking** POSSIBLE but slow. Bitmask DP with submask enumeration is faster: O(3^n) vs potentially O(n!) for backtracking.

---

## Step 7: When You're Stuck

If none of the above gives you a clear path:

1. **Simplify.** Solve a smaller or constrained version. "What if the array were sorted?" "What if all values were positive?" "What if n ≤ 10?" The simplified version often reveals the technique for the full problem.
    
2. **Work backwards from the answer.** What does the optimal answer LOOK like structurally? Is it a subarray? A subset? A path? A partition? The structure constrains the technique.
    
3. **Draw the dependency graph.** For DP: what subproblems does each state depend on? Draw arrows. If it forms a DAG, you have a valid DP ordering. If there are cycles, you might need a graph algorithm instead.
    
4. **Think about what information you need at each step.** That's your DP state. If the state space is manageable, it's DP. If the state is just "current best" or "current position," it might be greedy.
    
5. **Check if the problem is a known pattern in disguise.** Many problems are transformed versions of classic ones. "Can I rearrange this to look like knapsack / interval scheduling / LIS / shortest path?"