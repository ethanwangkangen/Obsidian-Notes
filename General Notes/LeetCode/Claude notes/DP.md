# Dynamic Programming — Comprehensive Reference

---

## How to Identify DP Problems

**Primary signals:**

- "Count the number of ways to..."
- "What is the minimum/maximum cost to..."
- "Is it possible to...?" (feasibility with choices)
- "Longest / shortest [subsequence, path, etc.]"
- You find yourself making a CHOICE at each step, and the best choice depends on the outcomes of future choices.

**The two requirements (both must hold):**

1. **Optimal substructure** — the optimal solution to the problem contains optimal solutions to subproblems.
2. **Overlapping subproblems** — the same subproblems are solved multiple times (this is what makes memoization worthwhile).

**When DP is NOT appropriate:**

- **Greedy works** — if you can prove (via exchange argument) that the locally best choice is always globally best, greedy is simpler and faster. Example: coin change with standard denominations (25, 10, 5, 1) is greedy, but coin change with arbitrary denominations (e.g., [1, 3, 4]) requires DP.
- **No overlapping subproblems** — if each subproblem is solved only once, it's just recursion or divide-and-conquer, not DP. Example: merge sort.
- **State space is too large** — if n = 10^5 and you need 2D DP → 10^10 states, it won't fit in time/memory. Look for a way to reduce dimensions or use a different technique.
- **You need ALL solutions** — DP gives the count or the optimal value, not the enumeration. Use backtracking for "list all."
- **The problem is actually a graph shortest path** — if your DP has weird non-sequential dependencies, you might be better off modeling it as a graph with Dijkstra/BFS.

**Transformation tricks to recognize DP in disguise:**

- **Algebraic rearrangement** — "Target Sum" (assign +/- to each number to reach a target) looks like backtracking, but rearrange: if P is the set of positives and N the set of negatives, then P - N = target and P + N = total. So P = (target + total) / 2. Now it's a standard subset-sum knapsack.
- **Complement thinking** — "minimum elements to remove" = total - "maximum elements to keep." The "keep" version may have clean DP structure.
- **Redefine what's "last"** — in Burst Balloons, instead of "which balloon to burst first," think "which balloon is burst LAST in range [i,j]." This gives clean interval DP because the last balloon's neighbors are the range boundaries.
- **Add a dimension** — if your DP doesn't capture enough info, add a dimension. Stock problems: add "holding / not holding" as a state dimension.
- **Remove a dimension** — if dp[i] only depends on dp[i-1], compress to two variables or a 1D array.

---

## How to Design the DP State

The hardest part of DP is defining the state. Ask yourself:

**"What is the minimum information I need to make the next decision?"**

Step-by-step process:

1. Identify what "decision" you make at each step (include/exclude, take left/right, assign to group, etc.).
2. After making that decision, what changes? That's your state transition.
3. What information from the past affects future decisions? That's your state.
4. Can you order the states so each depends only on previously computed ones? If yes, you have a valid DP. If not, it might be a graph problem.

**Common state patterns:**

- `dp[i]` — considering the first i elements (or ending at element i).
- `dp[i][j]` — two strings of length i, j. Or interval [i, j]. Or first i elements with budget/capacity j.
- `dp[i][k]` — first i elements, having used k transactions/groups/resources.
- `dp[mask]` — which elements have been used (bitmask, n ≤ 20).
- `dp[i][state]` — position i with a finite set of modes (holding stock, cooldown, etc.).

**"Ending at i" vs. "considering first i":**

- "Ending at i" = the subarray/subsequence must include element i. Used for: max subarray (Kadane's), LIS, longest valid parentheses.
- "Considering first i" = the best answer using elements 0..i-1, element i may or may not be included. Used for: house robber, knapsack.
- Getting this wrong is a common source of bugs. If your recurrence requires "must include the current element," use "ending at."

---

## DP Categories

### Category 1: Linear DP (1D)

The simplest form. `dp[i]` depends on a fixed number of previous states.

**1a — Fibonacci / Climbing Stairs** `dp[i] = dp[i-1] + dp[i-2]`. Any problem where you arrive at state i from a fixed set of previous states. LC 70 (Climbing Stairs), LC 509 (Fibonacci Number), LC 746 (Min Cost Climbing Stairs).

**1b — House Robber (skip constraint)** `dp[i] = max(dp[i-1], dp[i-2] + nums[i])`. "Can't take two adjacent" constraint. At each house: either skip it (take dp[i-1]) or rob it (take dp[i-2] + value). LC 198 (House Robber), LC 740 (Delete and Earn — reduce to House Robber by grouping values).

**1c — House Robber on a Circle** Can't take both first and last. Run House Robber twice: once on `nums[0..n-2]`, once on `nums[1..n-1]`. Take the max. LC 213 (House Robber II).

**1d — Decode Ways** `dp[i]` depends on whether last 1 digit and last 2 digits form valid codes. `dp[i] = dp[i-1] * (valid1) + dp[i-2] * (valid2)`. Watch edge cases: '0' can't stand alone, leading zeros. LC 91 (Decode Ways).

**1e — Word Break** `dp[i]` = can `s[0..i-1]` be segmented into dictionary words? For each `j < i`, check if `dp[j]` is true AND `s[j..i-1]` is in the dictionary. Use a hashset for O(1) word lookup. LC 139 (Word Break).

**1f — Maximum Subarray (Kadane's)** `dp[i] = max(nums[i], dp[i-1] + nums[i])`. Decision: extend the previous subarray or start fresh at i. Track global max separately. LC 53 (Maximum Subarray).

**1g — Maximum Product Subarray** Track BOTH `max_ending_here` and `min_ending_here` because a negative × negative = positive. When `nums[i]` is negative, swap max and min before updating. LC 152 (Maximum Product Subarray).

---

### Category 2: Grid / 2D DP

`dp[i][j]` typically represents something about reaching cell (i, j).

**2a — Unique Paths** `dp[i][j] = dp[i-1][j] + dp[i][j-1]`. Count paths from top-left to bottom-right, moving only right or down. Add obstacles by setting those cells to 0. LC 62 (Unique Paths), LC 63 (Unique Paths II — with obstacles).

**2b — Minimum Path Sum** `dp[i][j] = grid[i][j] + min(dp[i-1][j], dp[i][j-1])`. LC 64 (Minimum Path Sum).

**2c — Maximal Square** `dp[i][j]` = side length of largest all-1s square with bottom-right corner at (i,j). `dp[i][j] = min(dp[i-1][j], dp[i][j-1], dp[i-1][j-1]) + 1` (if grid[i][j] == 1). Intuition: the square is limited by the smallest adjacent square. LC 221 (Maximal Square).

**2d — Cherry Pickup (Two Agents)** Two people traverse simultaneously. State: `dp[r1][c1][r2]` (c2 is derived since both take the same number of steps). If both land on the same cherry, count it once. LC 741 (Cherry Pickup), LC 1463 (Cherry Pickup II — two robots going down).

---

### Category 3: String DP (Two Strings)

`dp[i][j]` = answer for first i characters of string A and first j characters of string B.

**3a — Longest Common Subsequence (LCS)** If `A[i-1] == B[j-1]`: `dp[i][j] = dp[i-1][j-1] + 1`. Else: `dp[i][j] = max(dp[i-1][j], dp[i][j-1])`. LC 1143 (LCS).

**3b — Longest Common Substring (contiguous)** Same as LCS but when chars don't match: `dp[i][j] = 0` (contiguity breaks). Track the global max. LC 718 (Maximum Length of Repeated Subarray).

**3c — Edit Distance** `dp[i][j]` = min edits to convert `A[0..i-1]` to `B[0..j-1]`. Three operations: insert (`dp[i][j-1]+1`), delete (`dp[i-1][j]+1`), replace (`dp[i-1][j-1] + (A[i-1] != B[j-1])`). Take the min. LC 72 (Edit Distance).

**3d — Interleaving String** `dp[i][j]` = can `C[0..i+j-1]` be formed by interleaving `A[0..i-1]` and `B[0..j-1]`? Transition: `dp[i][j] = (dp[i-1][j] && A[i-1]==C[i+j-1]) || (dp[i][j-1] && B[j-1]==C[i+j-1])`. LC 97 (Interleaving String).

**3e — Regex / Wildcard Matching** For regex (`.*`): `dp[i][j]` with careful handling of `*` (zero or more of the preceding character). For wildcard (`?*`): `*` matches any sequence. LC 10 (Regular Expression Matching), LC 44 (Wildcard Matching).

**3f — Distinct Subsequences** `dp[i][j]` = number of subsequences of `S[0..i-1]` that equal `T[0..j-1]`. If `S[i-1]==T[j-1]`: `dp[i][j] = dp[i-1][j-1] + dp[i-1][j]` (use this char or skip it). Else: `dp[i][j] = dp[i-1][j]`. LC 115 (Distinct Subsequences).

---

### Category 4: Knapsack

**Identification:** "Select items, each with a weight/cost and value, maximize value under a capacity constraint."

**4a — 0/1 Knapsack (each item at most once)** 1D optimization: iterate items in the OUTER loop, capacity in the INNER loop going **RIGHT TO LEFT** (reverse). This ensures each item is counted at most once — processing in reverse means when you compute dp[w], dp[w - weight] still holds the value from the PREVIOUS item iteration.

```
for each item:
    for w = W down to item.weight:
        dp[w] = max(dp[w], dp[w - item.weight] + item.value)
```

LC 416 (Partition Equal Subset Sum — target = total/2, values = weights), LC 474 (Ones and Zeroes — 2D knapsack with two constraints).

**4b — Unbounded Knapsack (unlimited copies)** Same structure but inner loop goes **LEFT TO RIGHT** (forward). This allows reusing the same item — dp[w - weight] may already include the current item.

```
for each item:
    for w = item.weight to W:
        dp[w] = max(dp[w], dp[w - item.weight] + item.value)
```

LC 322 (Coin Change — minimize coins), LC 518 (Coin Change 2 — count ways).

**4c — Common Knapsack Transformations**

- **Partition Equal Subset Sum:** 0/1 knapsack with target = total/2. Weights = values = the numbers.
- **Target Sum (+/- each element):** Rearrange algebra: P - N = target, P + N = total → P = (total + target) / 2. Now it's subset sum (0/1 knapsack). Check that (total + target) is even and non-negative.
- **Coin Change (min coins):** Unbounded knapsack. Initialize dp[0] = 0, all others = INF. Transition: `dp[w] = min(dp[w], dp[w - coin] + 1)`.
- **Coin Change 2 (count ways):** Unbounded knapsack. Initialize dp[0] = 1. Transition: `dp[w] += dp[w - coin]`. **Crucial:** iterate coins in outer loop, amounts in inner loop. If reversed, you count permutations, not combinations.

LC 416 (Partition Equal Subset Sum), LC 494 (Target Sum), LC 322 (Coin Change), LC 518 (Coin Change 2).

---

### Category 5: Interval DP

`dp[i][j]` = optimal answer for the subarray/substring `[i..j]`.

**How to identify:** "Merge/split/break a contiguous range" with a cost that depends on the range. "What's the optimal way to combine adjacent things?"

**Iteration order:** Fill by INCREASING LENGTH. `for len = 2 to n: for i = 0 to n-len: j = i+len-1: for k = i to j-1 (split point)`.

**5a — Burst Balloons** Think in REVERSE: which balloon is burst LAST in range [i, j]? That balloon k sees `nums[i-1]` and `nums[j+1]` as neighbors (everything else already gone). `dp[i][j] = max over k of (dp[i][k-1] + nums[i-1]*nums[k]*nums[j+1] + dp[k+1][j])`. LC 312 (Burst Balloons).

**5b — Palindrome DP** `dp[i][j]` = is `s[i..j]` a palindrome? Or: `dp[i][j]` = length of longest palindromic subsequence in `s[i..j]`. Base: `dp[i][i] = 1`. Transition: if `s[i]==s[j]`, `dp[i][j] = dp[i+1][j-1] + 2`; else `dp[i][j] = max(dp[i+1][j], dp[i][j-1])`. LC 516 (Longest Palindromic Subsequence), LC 5 (Longest Palindromic Substring — expand from center is simpler but interval DP also works).

**5c — Matrix Chain Multiplication** `dp[i][j]` = minimum cost to multiply matrices i through j. Split at k: cost = `dp[i][k] + dp[k+1][j] + dim[i]*dim[k+1]*dim[j+1]`. LC 1039 (Minimum Score Triangulation of Polygon).

**5d — Palindrome Partitioning (min cuts)** `dp[i]` = min cuts for `s[0..i]`. For each `j ≤ i`, if `s[j..i]` is a palindrome, `dp[i] = min(dp[j-1] + 1)`. Precompute palindrome table with interval DP or expand-from-center. LC 132 (Palindrome Partitioning II).

---

### Category 6: Bitmask DP

`dp[mask]` where mask is a bitmask of which elements are used/visited. **Only feasible when n ≤ 20** (2^20 ≈ 10^6).

**How to identify:** "Assign n items to n slots" (n small), "visit all nodes and return" (TSP), "partition into groups of specific sizes."

**6a — TSP (Traveling Salesman)** `dp[mask][i]` = min cost to visit all cities in mask, currently at city i. Transition: for each city j not in mask, `dp[mask | (1<<j)][j] = min(dp[mask][i] + cost[i][j])`. LC 943 (Find the Shortest Superstring).

**6b — Assignment Problem** Assign n people to n tasks. `dp[mask]` = min cost when tasks indicated by mask are completed (in order of people). Person index = `popcount(mask)`. LC 1125 (Smallest Sufficient Team), LC 1986 (Minimum Number of Work Sessions).

**6c — Subset Enumeration** Iterating over all subsets of a mask:

```cpp
for (int sub = mask; sub > 0; sub = (sub - 1) & mask) {
    // sub is a submask of mask
}
// handle sub = 0 separately if needed
```

Total work across all masks: O(3^n) by the inclusion-exclusion counting argument.

---

### Category 7: State Machine DP

Define explicit states/modes and transitions between them.

**How to identify:** Problems with distinct phases or modes (holding/not holding, cooldown, transaction count). Most common: the "Buy and Sell Stock" family.

**7a — Stock Problem Framework** At each day i, you're in one of several states:

```
hold[i]     = max(hold[i-1],     rest[i-1] - prices[i])   // keep holding, or buy today
rest[i]     = max(rest[i-1],     hold[i-1] + prices[i])   // keep idle, or sell today
cooldown[i] = hold[i-1] + prices[i]                       // just sold (if cooldown applies)
```

**Variations by adding/modifying states:**

- **Unlimited transactions:** just hold and rest, no transaction counter.
- **At most K transactions:** add a transaction dimension: `hold[i][k]`, `rest[i][k]`.
- **Cooldown after selling:** after selling, enter a cooldown state for 1 day before you can buy again. `rest[i] = max(rest[i-1], cooldown[i-1])` and `hold[i] = max(hold[i-1], rest[i-1] - prices[i])`.
- **Transaction fee:** subtract fee when selling: `rest[i] = max(rest[i-1], hold[i-1] + prices[i] - fee)`.

LC 121 (I — single transaction, just track min price), LC 122 (II — unlimited), LC 123 (III — at most 2), LC 188 (IV — at most K), LC 309 (With Cooldown), LC 714 (With Transaction Fee).

---

### Category 8: LIS Variants

**8a — O(n²) DP** `dp[i]` = length of LIS ending at index i. `dp[i] = max(dp[j] + 1)` for all `j < i` where `nums[j] < nums[i]`. Answer = max of all dp[i].

**8b — O(n log n) Patience Sorting** Maintain a `tails` array. For each element:

- Binary search for its position in `tails`.
- `lower_bound` for strictly increasing LIS.
- `upper_bound` for non-decreasing.
- If it goes past the end, append (LIS grows). Else, replace `tails[pos]` (keeps tails as small as possible for future extensions).

**8c — Number of LIS** Track `count[i]` alongside `length[i]`. When you find `length[j] + 1 == length[i]`, add `count[j]` to `count[i]`. When you find `length[j] + 1 > length[i]`, reset `count[i] = count[j]`. LC 673 (Number of Longest Increasing Subsequence).

**8d — 2D LIS (Russian Doll Envelopes)** Sort by width ascending, then by height DESCENDING within the same width. Then run LIS on heights. The descending sort prevents using two envelopes with the same width. LC 354 (Russian Doll Envelopes).

**8e — Longest String Chain** Sort words by length. For each word, try removing each character and check if the shorter word's chain length is known. `dp[word] = max(dp[word_with_one_char_removed]) + 1`. LC 1048 (Longest String Chain).

---

### Category 9: Tree DP

Root the tree. DFS from root. Compute `dp[node]` from the DP values of its children (post-order).

**How to identify:** "Optimal value on a tree where node decisions depend on subtree results."

**9a — House Robber III** `dp[node]` = (rob_this_node, skip_this_node). If you rob this node, you can't rob its children. If you skip, you take the max of rob/skip for each child. LC 337 (House Robber III).

**9b — Diameter / Max Path Sum** The function RETURNS one thing (height / single-branch sum) for the parent to use, but UPDATES a global max with a different value (diameter / full path through this node).

Diameter: return `max(left_h, right_h) + 1` to parent, update global with `left_h + right_h`. Max path sum: return `node.val + max(0, max(left, right))` to parent, update global with `node.val + max(0, left) + max(0, right)`.

LC 543 (Diameter), LC 124 (Max Path Sum).

---

### Category 10: DP + Optimization Techniques

**10a — DP + Binary Search (Weighted Interval Scheduling)** Sort intervals by end time. `dp[i] = max(dp[i-1], value[i] + dp[last_compatible])`. Use binary search to find `last_compatible` — the latest interval that ends before interval i starts. LC 1235 (Maximum Profit in Job Scheduling), LC 1751 (Maximum Number of Events That Can Be Attended II).

**10b — DP + Monotonic Deque** When transition is `dp[i] = max/min(dp[j] for j in [i-K, i-1]) + cost[i]`, use a monotonic deque over dp values. Each element enters/exits the deque once → O(n) total. LC 1425 (Constrained Subsequence Sum), LC 239 (underlying technique).

**10c — DP + Prefix Sum** When transition sums over a range of previous states: `dp[i] = sum(dp[j] for j in [a, b])`, precompute prefix sums of the dp array for O(1) range sums. LC 1977 (Number of Ways to Separate Numbers — advanced).

**10d — Space Optimization (Rolling Array)** If `dp[i]` depends only on `dp[i-1]`, keep just two rows (or two variables). For 2D DP where `dp[i][j]` depends on `dp[i-1][...]`, use a single 1D array and process in the correct order. For 0/1 knapsack, process capacity in reverse to avoid double-counting.

---

### When to Suspect Your DP is Wrong

- **Greedy gives correct answer on all test cases you try** — maybe greedy works. Try to construct a counterexample for greedy. If you can't, it might be greedy.
- **Your state doesn't capture enough information** — you get wrong answers on edge cases. Add a dimension.
- **Your state captures TOO MUCH information** — TLE. Think about what information is actually needed. Can you merge states?
- **Transition order is wrong** — you're using dp values that haven't been computed yet. Draw the dependency graph to check.
- **Base cases are wrong** — the most common DP bug. Think carefully: what is dp[0]? dp[0][0]? Do you initialize with 0, 1, INF, or -INF?
- **Off-by-one in indexing** — does `dp[i]` represent "first i elements" (1-indexed semantics) or "ending at index i" (0-indexed)? Be consistent.