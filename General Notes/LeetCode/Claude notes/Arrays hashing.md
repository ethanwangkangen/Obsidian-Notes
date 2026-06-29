# Arrays & Hashing — Comprehensive Reference

---

## How to Identify

**Primary signals:**

- "Subarray sum equals K" → prefix sum + hashmap.
- "Count subarrays with property X" → prefix sum or sliding window (check monotonicity).
- "Find duplicates / missing numbers in range [1, N]" → cyclic sort or XOR.
- "Two sum / group sum" → hashmap for O(1) lookup.
- "Range update operations" → difference array.
- "Sum of [property] over all subarrays" → contribution technique.

**When to use hashing vs. sorting:**

- Hashing: O(1) lookup, unordered. Use for existence checks, frequency counting, prefix sum queries.
- Sorting: O(n log n) but enables two pointers, binary search, greedy. Use when order matters.

---

## Prefix Sum

**Core idea:** `prefix[i] = sum(nums[0..i-1])`. Any subarray sum `sum(l, r) = prefix[r+1] - prefix[l]` in O(1).

### Subarray Sum Equals K (with negatives)

Store prefix sums in a hashmap `{prefix_value: count}`. At each index i, check if `prefix[i] - K` exists in the map. Initialize with `{0: 1}` (empty prefix).

**Why this works:** If `prefix[i] - prefix[j] = K`, then the subarray `[j+1, i]` sums to K. The hashmap tells you how many valid j's exist.

**Critical:** This does NOT work with sliding window when negatives are present.

LC 560 (Subarray Sum Equals K), LC 525 (Contiguous Array — treat 0 as -1, then subarray sum = 0).

### Subarray Sum Divisible by K

Store `prefix[i] % K` in the hashmap. Two prefixes with the same remainder form a subarray divisible by K. In C++, handle negative modulo: `((prefix % K) + K) % K`.

LC 974 (Subarray Sums Divisible by K).

### 2D Prefix Sum

`prefix[i][j]` = sum of rectangle from (0,0) to (i-1, j-1). Rectangle sum from (r1,c1) to (r2,c2) = `prefix[r2+1][c2+1] - prefix[r1][c2+1] - prefix[r2+1][c1] + prefix[r1][c1]`.

LC 304 (2D Range Sum Query).

### Prefix XOR

`prefix_xor[i] = nums[0] ^ nums[1] ^ ... ^ nums[i-1]`. Subarray XOR = `prefix_xor[r+1] ^ prefix_xor[l]`.

LC 1310 (XOR Queries of a Subarray), LC 1442 (Count Triplets That Can Form Two Arrays of Equal XOR).

---

## Difference Array

For bulk range-increment operations. Instead of updating every element in [l, r], mark `diff[l] += val` and `diff[r+1] -= val`. Prefix-sum `diff` to reconstruct.

LC 370 (Range Addition), LC 1109 (Corporate Flight Bookings), LC 1094 (Car Pooling).

---

## Kadane's Algorithm

`current_max = max(nums[i], current_max + nums[i])` — extend previous subarray or start fresh.

**Maximum product variant:** track BOTH max and min at each step. When nums[i] is negative, swap max and min before updating.

**Circular variant:** max circular = max(standard Kadane, total_sum - min_subarray). Edge case: if all negative, min subarray = entire array, so the circular formula gives 0 — use standard Kadane only.

LC 53 (Maximum Subarray), LC 152 (Maximum Product Subarray), LC 918 (Max Sum Circular Subarray).

---

## Hashing Patterns

### Two Sum

Store `{value: index}` in a hashmap. For each element, check if `target - element` exists.

LC 1 (Two Sum).

### Group Anagrams

Key = sorted characters (or character frequency tuple). Group strings by key.

LC 49 (Group Anagrams).

### Longest Consecutive Sequence

Put all numbers in a hashset. For each number that is the START of a sequence (`num - 1` not in set), count consecutive elements. O(n) total because each element is visited at most twice.

LC 128 (Longest Consecutive Sequence).

---

## Boyer-Moore Majority Vote

Candidate + count. For each element: count == 0 → new candidate. Element == candidate → count++. Else → count--. The majority element (> n/2) survives.

**For n/3 majority:** two candidates, two counts. At most 2 elements can appear > n/3 times. Verify with a second pass.

LC 169 (Majority Element), LC 229 (Majority Element II).

---

## Cyclic Sort

Numbers in [1, N] in an array of size N. Place each number at index `num - 1`. Then scan for mismatches.

LC 268 (Missing Number), LC 442 (Find All Duplicates), LC 448 (Find Disappeared Numbers), LC 41 (First Missing Positive — harder: ignore negatives and values > n).

---

## Contribution Technique

Instead of iterating over all O(n²) subarrays, determine each element's contribution to the total.

Use monotonic stack to find the nearest smaller/larger on left and right. Element at index i with boundaries `left` and `right` contributes to `(i - left) * (right - i)` subarrays.

LC 907 (Sum of Subarray Minimums), LC 2104 (Sum of Subarray Ranges), LC 828 (Count Unique Characters of All Substrings).

---

## Next Permutation

1. Find the largest index i such that `nums[i] < nums[i+1]` (the "descent" from the right).
2. Find the largest j > i such that `nums[j] > nums[i]`.
3. Swap nums[i] and nums[j].
4. Reverse the suffix after index i.

If no descent exists (array is fully descending), reverse the whole array (wrap to smallest permutation).

LC 31 (Next Permutation).