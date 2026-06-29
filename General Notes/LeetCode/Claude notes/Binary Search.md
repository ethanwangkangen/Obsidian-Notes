# Binary Search — Comprehensive Reference

---

## How to Identify

**Primary signals:**

- Sorted array and you need to find/locate something.
- "Minimize the maximum" or "maximize the minimum."
- "Can you achieve X with value ≤ Y?" where the feasibility is MONOTONIC in Y (if Y works, Y+1 also works).
- K-th smallest/largest in a structured space (sorted matrix, two sorted arrays).

**The universal principle:** Binary search works when there's a MONOTONIC PREDICATE — a function f(x) that is `false, false, ..., false, true, true, ..., true` (or the reverse). You're finding the transition point.

**When binary search does NOT work:**

- The predicate isn't monotonic (f could alternate true/false).
- The search space isn't orderable.
- The feasibility check itself is too expensive (though this is rare — usually the check is O(n) or O(n log n)).

---

## Standard Binary Search

**Avoiding bugs — use the half-open `[lo, hi)` convention consistently:**

```cpp
int lo = 0, hi = n;  // search space is [0, n)
while (lo < hi) {
    int mid = lo + (hi - lo) / 2;  // avoids overflow
    if (condition(mid))
        hi = mid;       // answer is mid or to the left
    else
        lo = mid + 1;   // answer is to the right
}
// lo == hi == first index where condition is true
```

**Lower bound (first true):** use `hi = mid` when condition is true, `lo = mid + 1` when false. Result: first index where condition holds.

**Upper bound (first false after trues):** use `lo = mid + 1` when condition is true, `hi = mid` when false. Result: first index where condition is false (i.e., one past the last true).

**Common bug:** using `mid = (lo + hi) / 2` causes infinite loops when `lo + 1 == hi` and you set `lo = mid`. Always use `lo = mid + 1` or `hi = mid` (never `hi = mid - 1` with this convention, though that works with the closed `[lo, hi]` convention).

---

## Binary Search on Answer (Parametric Search)

**The most important binary search pattern for competitive programming.**

**Core idea:** The answer is a number V. Write a function `canDo(V)` that returns whether V is achievable. If `canDo` is monotonic in V, binary search on V.

**How to identify:**

- "Minimize the maximum [something]" → binary search on the maximum; `canDo(V)` checks if you can arrange things so the max is ≤ V.
- "Maximize the minimum [something]" → binary search on the minimum; `canDo(V)` checks if you can arrange things so the min is ≥ V.
- "Is it possible to do X in Y time/cost?" and the answer flips from no to yes (or yes to no) as Y increases.

**Template:**

```cpp
int lo = MIN_POSSIBLE_ANSWER, hi = MAX_POSSIBLE_ANSWER;
while (lo < hi) {
    int mid = lo + (hi - lo) / 2;
    if (canDo(mid))
        hi = mid;       // for minimization: try smaller
    else
        lo = mid + 1;
}
return lo;
```

For maximization, flip: if `canDo(mid)`, `lo = mid + 1` (try bigger), else `hi = mid`. Return `lo - 1`.

**The `canDo` function is almost always a GREEDY simulation.** You check feasibility by making greedy choices.

**Examples:**

**Koko Eating Bananas:** Binary search on speed. canDo(speed) = can she finish all piles in H hours? Greedily: each pile takes ceil(pile/speed) hours. Sum them up and check ≤ H. LC 875 (Koko Eating Bananas).

**Split Array Largest Sum:** Binary search on the maximum subarray sum. canDo(maxSum) = can you split into ≤ K subarrays each with sum ≤ maxSum? Greedily: scan left to right, start a new subarray when adding the next element would exceed maxSum. LC 410 (Split Array Largest Sum).

**Capacity to Ship Packages:** Same pattern as above. Binary search on capacity. LC 1011 (Capacity To Ship Packages Within D Days).

**Aggressive Cows / Magnetic Balls:** Binary search on minimum distance. canDo(minDist) = can you place K cows such that every pair is ≥ minDist apart? Greedily: place first cow at first stall, then each subsequent cow at the first stall that's ≥ minDist from the last placed cow.

**House Robber IV:** Binary search on the maximum amount stolen from any single house. canDo(maxVal) = can you rob ≥ K houses where each has value ≤ maxVal? Greedily skip adjacent. LC 2560 (House Robber IV).

---

## Binary Search on Rotated/Modified Arrays

**Search in Rotated Sorted Array:** Compare `nums[mid]` with `nums[lo]`:

- If `nums[lo] <= nums[mid]`, the LEFT half is sorted. Check if target is in `[nums[lo], nums[mid])`. If yes, search left. Else search right.
- Else the RIGHT half is sorted. Check if target is in `(nums[mid], nums[hi]]`. If yes, search right. Else search left.

LC 33 (Search in Rotated Sorted Array).

**With duplicates:** When `nums[lo] == nums[mid] == nums[hi]`, you can't tell which half is sorted. Shrink: `lo++, hi--`. Worst case O(n). LC 81 (Search in Rotated Sorted Array II).

**Find Minimum in Rotated Array:** Compare `nums[mid]` with `nums[hi]`:

- If `nums[mid] > nums[hi]`, min is in right half: `lo = mid + 1`.
- Else, min is in left half (including mid): `hi = mid`. LC 153 (Find Minimum in Rotated Sorted Array), LC 154 (with duplicates).

---

## Binary Search for LIS (Patience Sorting)

Maintain a `tails` array where `tails[i]` = smallest possible tail of any increasing subsequence of length i+1. For each element:

- `lower_bound(tails, element)` for STRICTLY increasing.
- `upper_bound(tails, element)` for NON-DECREASING.
- If position == tails.size(), append (LIS grows).
- Else, replace `tails[pos]` with element. Length of `tails` at the end = LIS length.

**Russian Doll trick:** Sort by width ascending, height DESCENDING within same width. Then LIS on heights. The descending sort within same width prevents using two envelopes with the same width.

LC 300 (LIS), LC 354 (Russian Doll Envelopes), LC 1964 (Longest Valid Obstacle Course).

---

## Find Peak Element

If `nums[mid] > nums[mid+1]`, a peak exists on the left (including mid). Else, a peak exists on the right. This works because endpoints are treated as −∞.

LC 162 (Find Peak Element), LC 852 (Peak Index in Mountain Array).

---

## K-th Smallest in Sorted Matrix

Binary search on VALUE (not index). For a given value V, count how many elements are ≤ V by walking the matrix from the bottom-left corner (go right if value ≤ V, go up if value > V). This count is O(n). Binary search the smallest V where count ≥ K.

LC 378 (Kth Smallest Element in a Sorted Matrix).

---

## Median of Two Sorted Arrays

Binary search on the partition point in the SHORTER array. For a valid partition: `maxLeft1 ≤ minRight2` and `maxLeft2 ≤ minRight1`. If `maxLeft1 > minRight2`, move partition left. If `maxLeft2 > minRight1`, move partition right.

LC 4 (Median of Two Sorted Arrays).

---

## Binary Search on Floating Point

For problems with real-valued answers (e.g., minimize maximum distance), use:

```cpp
double lo = MIN, hi = MAX;
for (int iter = 0; iter < 100; iter++) {  // ~100 iterations gives ~10^-30 precision
    double mid = (lo + hi) / 2;
    if (canDo(mid)) hi = mid;
    else lo = mid;
}
```

Use iteration count instead of `while (hi - lo > eps)` to avoid floating-point convergence issues.

LC 774 (Minimize Max Distance to Gas Station).