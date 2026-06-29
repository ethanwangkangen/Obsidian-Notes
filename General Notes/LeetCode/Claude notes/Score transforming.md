# Score Transformation & Reduction Paradigms

Problems that look complex until you realise the objective can be rewritten as a per-element score, after which a classic algorithm (Kadane's, prefix sum, two pointers, DP) finishes the job. This is one of the least-named but most common paradigms at the 1800–2200 difficulty range.

---

## The Core Idea

You have some complex objective over subarrays, subsequences, or element selections. It seems hard to optimise directly. But if you fix a parameter (a target value, a character, a threshold), the contribution of each element to the objective becomes a simple local value — often just +1, -1, or 0. Once every element has a score, you've reduced the problem to a classic 1D algorithm.

**The general reduction:**

```
for each candidate value v (or parameter):
    assign each element a score based on v
    run classic algorithm on scores (Kadane's, prefix sum, sliding window, DP)
    update global answer
```

Time complexity: O(|candidates| × n) or O(|candidates| × n log n). Since |candidates| is often small (e.g., values in [1, 50]), this is effectively O(n).

---

## Paradigm 1: +1 / -1 Scoring → Kadane's

This is the paradigm from the problem you posted. The net gain from an operation over a subarray = sum of per-element scores. Maximise the sum of a subarray = Kadane's.

### Template

```
base_count = count of already-good elements outside the operation
for each candidate value v:
    for each element:
        score[i] = +1 if element[i] == v (operation turns it into the target)
                   -1 if element[i] == target (operation destroys an existing target)
                    0 otherwise
    gain = max subarray sum of score[] via Kadane's
    answer = max(answer, base_count + gain)
```

### Why Kadane's?

The gain is additive over the subarray. You want the subarray that maximises the net count change. This is exactly "maximum subarray sum," which Kadane's solves in O(n).

### Example: Maximum Frequency After Subarray Operation (LC 3434)

You can add any constant x to a subarray. Choose x to turn some value v into k. Inside the subarray, every v becomes k (+1 contribution), every k that isn't v gets shifted away (-1 contribution). Outside the subarray, nothing changes.

Fix v. Assign: `+1` if `nums[i] == v`, `-1` if `nums[i] == k` (and v ≠ k), `0` otherwise. Kadane's gives the best subarray for this v. Iterate over all v ∈ [1, 50]. Answer = `count(k) + max_gain`.

**Why this is non-obvious:** the operation acts on a subarray and adds an arbitrary constant. You might try to search over the constant x. But fixing the SOURCE value v (what gets mapped TO k) instead of the constant x collapses everything: the gain from any subarray is fully determined by how many v's and k's are in it.

### Other problems using +1/-1 scoring → Kadane's

**Maximum Subarray Score with Penalty:** if inside-subarray elements above a threshold give +1 and below give -1, Kadane's gives the best subarray.

**Minimum Swaps to Group All 1s Together (LC 1151):** count of 1s in the window = desired window size. Score each element +1 if 1, -1 if 0, find max subarray of exact length = count(1s). (Alternatively: sliding window since array is binary and monotonic. But the +1/-1 framing gives the general form.)

**Minimum Operations to Make Array Alternating:** for each position, score +1 if it already matches the desired pattern, else 0. Find pattern that maximises matches. This is score-counting, not Kadane's, but the per-element scoring step is the same.

---

## Paradigm 2: 0/1 Scoring → Prefix Sum for Range Counts

Fix a parameter, assign 0/1 per element, answer a range query with prefix sum.

### Example: Count Subarrays with Majority Element

Fix the majority candidate (Boyer-Moore). Assign +1 for candidate, -1 for others. A subarray has the candidate as majority iff its score sum > 0, i.e., prefix[r] > prefix[l-1]. Query pairs of prefix sums with a hashmap.

**The key insight:** "majority" is a non-local property of a subarray, but re-encoding as +1/-1 makes it equivalent to a prefix sum comparison.

LC 169 related, LC 2183 (Count Array Pairs Divisible by K — scoring by gcd relationship).

### Example: Contiguous Array (LC 525)

"Longest subarray with equal 0s and 1s." Direct: hard. Score: replace 0 with -1, 1 stays +1. Now "equal 0s and 1s" = "subarray sum = 0" = prefix sum + hashmap.

**The transformation:** 0 and 1 are asymmetric in count. Replacing 0 with -1 makes the balance symmetric — the problem becomes "longest zero-sum subarray," a classic.

---

## Paradigm 3: Fix One Value, Reduce to Classic Problem

A broader version: the problem has two or more "moving parts." Fix one and the other becomes tractable.

### Template

```
for each fixed value / index / threshold:
    solve the reduced 1D problem
    update global answer
```

### Example: Maximum Equal Frequency (LC 1224)

"Longest prefix where you can remove exactly one element to make all frequencies equal." Fix the frequency target f and think about what structure would allow removal of one element. This is complex — but fixing f and scanning reveals a few structural cases, each checkable in O(1) with running counts.

### Example: Best Time to Buy and Sell Stock with Transaction Fee (LC 714)

Two moving parts: buy price and sell price. Fix the buy (hold it in `dp[hold]`). The sell decision becomes: is `price - fee > 0` net gain? State machine DP.

### Example: Trapping Rain Water

Direct: what's the water at position i? Two moving parts: left max and right max. Fix the smaller of the two (two-pointer: always process the smaller side). The water at the current position is then fully determined.

---

## Paradigm 4: Rewrite the Objective Function

The problem asks you to maximise/minimise something that, when you expand the formula, simplifies into a form you know how to handle.

### Template

```
expand the objective algebraically
identify: what's fixed? what's varying?
the varying part is your optimisation target
```

### Example: Maximum Subarray Sum After One Operation (LC 2321 variant)

"Select one element and square it, then find maximum subarray sum." Naive: try all O(n) choices of squaring. But for each choice, the subarray sum = (standard Kadane) + (extra from squaring one element). Precompute prefix Kadane from left and suffix Kadane from right. For each i, best answer using position i as the squared element = `best_left_ending_at[i-1] + nums[i]^2 + best_right_starting_at[i+1]`.

**The insight:** the "one modification" interacts with Kadane's — but only locally at one index. Precompute Kadane results from both directions and combine.

LC 53 related, LC 2321 (Score of a String variant).

### Example: Maximum Score from Removing Substrings (LC 1717)

"Remove 'ab' for p points and 'ba' for q points, maximise score." Algebraic insight: always greedily remove the higher-valued pair first (because removing 'ab' first doesn't block removing 'ba' from the remainder, but the reverse can). Use a stack for each pass.

**The rewrite:** "which pair to remove first" looks like it needs DP over all states, but the objective separates into two independent greedy passes.

### Example: Target Sum (LC 494)

"Assign +/- to each element to reach target." Expand: let P = sum of positives, N = sum of negatives. P + N = total, P - N = target → P = (total + target) / 2. Now it's subset sum.

**Why this matters:** the algebraic rewrite turns an infinite search (try all sign assignments) into a finite DP (find subsets with a specific sum). The search space collapses.

### Example: Minimum Cost to Connect Sticks (LC 1167)

"Merge sticks, cost = sum of merged lengths, minimise total cost." Rewrite: each stick's length appears in the total cost once for every time it's involved in a merge (i.e., its "depth" in the merge tree). Minimise total cost = minimise weighted depth. This is the Huffman coding problem → greedy min-heap.

---

## Paradigm 5: Contribution Counting

Instead of iterating over all pairs/subarrays/subsets and computing their aggregate property, count how many times each element contributes.

### Template

```
for each element i:
    determine how many (subarrays / pairs / subsets) include element i
        in a "relevant" way (e.g., as the minimum, as a participant)
    multiply by element's value
    sum across all elements
```

### Example: Sum of Subarray Minimums (LC 907)

"Sum of min(subarray) over all subarrays." Direct: O(n²) subarrays. Instead: for each element, count the number of subarrays where IT is the minimum. Use monotonic stack to find the nearest smaller on left (L) and right (R). Element i is minimum in `(i - L) * (R - i)` subarrays. Contribution = `nums[i] * (i - L) * (R - i)`.

**Key:** the "property" (minimum) is defined locally per element, and monotonic stack gives the range for each element in O(n) total.

LC 907, LC 2104 (Sum of Subarray Ranges = sum of maxs − sum of mins), LC 1856 (Maximum Subarray Min-Product — same idea + prefix sum).

### Example: Sum of Distances in Tree (LC 834)

"Sum of distances from each node to all others." Naive: BFS from each node, O(n²). Instead: one DFS to compute sum of distances from the root. Second DFS: when you move the "center" from a parent to a child, distances to the child's subtree decrease by subtree_size, distances to everything else increase by n − subtree_size. Reuse parent's computation.

**The insight:** reframing as "how does the answer CHANGE when we shift the center" gives an O(n) rerooting DP.

LC 834.

### Example: Count Number of Texts (LC 2266)

"How many ways to decode a string of digit-keypresses?" Each character's contribution to the count depends on how many consecutive identical digits precede it. This is a DP where each element contributes to future states — but thinking in terms of "contribution" clarifies the recurrence.

---

## Paradigm 6: Reduce to "Is It in the Set?" via XOR / Bitmask

Fix a threshold, encode each element as a bit or XOR value, use prefix XOR or bitmask to answer range queries in O(1).

### Example: Count Triplets That Can Form Two Arrays of Equal XOR (LC 1442)

"Count triplets (i, j, k) where XOR(i..j-1) == XOR(j..k)." This equals XOR(i..k) == 0 (since equal XOR values XOR to 0). Use prefix XOR: `prefix[k+1] == prefix[i]`. Count pairs with equal prefix XOR.

**The transformation:** "two subarrays have equal XOR" → "the combined range has XOR = 0" → "prefix XOR equality" → hashmap count.

### Example: Decode XORed Array (LC 1720)

Given XOR of adjacent elements and the first element, reconstruct the array. `a[i] = encoded[i-1] XOR a[i-1]`. Prefix XOR directly gives all elements.

---

## Paradigm 7: Running Invariant / Balance Variable

Maintain a running variable that encodes a balance or surplus. When the balance hits a condition, you've found an answer. Most famous: the gas station circular scan.

### Example: Gas Station (LC 134)

Running surplus. If total gas ≥ total cost, a solution exists. The start is the point after the last place where the running surplus went negative. The "balance variable" is the running surplus, and its minimum tells you the start.

### Example: Minimum Add to Make Parentheses Valid (LC 921)

Running balance of open brackets. When balance goes negative, you need an insertion (and reset). Final balance = unmatched opens.

### Example: Valid Parentheses with Wildcards (LC 678)

Instead of a single balance, maintain `[lo, hi]` range of possible balances. `*` expands the range by 1 in both directions. Clamp lo at 0 (can't have negative balance). If `hi < 0` at any point, invalid. If `lo == 0` at the end, valid.

**The paradigm:** wildcards create UNCERTAINTY in the balance. Track the range of possible balances rather than a single value.

---

## Paradigm 8: Two-Pass / Prefix + Suffix Combination

Precompute answers from the left (prefix) and from the right (suffix). Combine at each index.

### Template

```
prefix[i] = best answer considering only elements 0..i
suffix[i] = best answer considering only elements i..n-1
answer[i] = combine(prefix[i-1], current element, suffix[i+1])
```

### Example: Product of Array Except Self (LC 238)

`result[i]` = product of all elements except i = `prefix_product[i-1] * suffix_product[i+1]`.

### Example: Maximum Subarray After One Modification

Best subarray that contains exactly one "squared" element. Precompute best Kadane ending at each index (prefix), best Kadane starting at each index (suffix). For each index i as the squared element: `prefix[i-1] + nums[i]^2 + suffix[i+1]`.

### Example: Sum of Beauty of All Substrings — variants

For problems of the form "find the best pairing between the left half and right half of the array," prefix + suffix precomputation is canonical. The combination at each split point is O(1).

### Example: Trapping Rain Water (LC 42)

`water[i] = min(max_left[i], max_right[i]) - height[i]`. Precompute prefix max and suffix max. Each cell's water is determined by local maxes from both directions.

---

## How to Recognise These Paradigms in a New Problem

**Signal for +1/-1 Kadane's:**

- "You can perform one operation on a subarray (shift all elements by x)"
- "Find the subarray that maximises [count of something] − [cost of something else]"
- The gain from a subarray is a sum of local, per-element contributions
- You'd otherwise search over all possible operation values (but fixing the source value collapses this)

**Signal for algebraic rewrite:**

- The objective has two or more terms that seem coupled (both changing together)
- You're searching over two variables (e.g., buy price and sell price in stocks)
- Expanding P - N = target, P + N = total type equations reveals a subset sum

**Signal for contribution counting:**

- "Sum of [min/max/some aggregate] over all subarrays/subsets"
- Each subarray's contribution to the total is dominated by one element (its min or max)
- You need to count "how many subarrays is element X the bottleneck of"

**Signal for running invariant:**

- "Find the point where the accumulated [something] first exceeds / equals / turns negative"
- The problem involves a sequence of decisions that each affect a running score
- Parentheses, surplus, balance problems

**Signal for prefix + suffix:**

- "Each element's answer depends on information from BOTH sides"
- "Remove one element and compute something for the rest"
- "Best split of the array at each index"

---

## Summary Table

|Paradigm|Key operation|Classic algorithm|Trigger words|
|---|---|---|---|
|+1/-1 score → max sum|Score per element, Kadane's|Kadane's|"subarray operation", "shift elements in subarray", "maximise frequency after operation"|
|0/1 score → count|Score per element, prefix sum|Prefix sum + hashmap|"longest balanced", "equal counts of X and Y"|
|Fix one variable|Iterate over candidates|Any (Kadane's, DP, binary search)|Two coupled variables, small value range|
|Algebraic rewrite|Expand formula, simplify|Subset sum, knapsack|"+/- assignment", "partition into groups", "target = P - N"|
|Contribution counting|Count appearances as bottleneck|Monotonic stack|"sum of min/max over all subarrays"|
|XOR reduction|Prefix XOR, hash pairs|Prefix XOR + hashmap|"equal XOR", "XOR of subarray = 0"|
|Running invariant|Track balance/surplus, note conditions|Scan + conditional|"parentheses", "gas station", "circular validity"|
|Prefix + suffix|Precompute both directions|Linear scan|"exclude one element", "best left and right", "split at each index"|