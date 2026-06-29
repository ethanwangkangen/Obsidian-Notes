# LeetCode Tricks & Techniques — Comprehensive Reference

---

## Table of Contents

1. Arrays & Hashing
2. Two Pointers
3. Sliding Window
4. Binary Search
5. Stack & Monotonic Stack
6. Queue & Monotonic Deque
7. Heap / Priority Queue
8. Greedy
9. Intervals
10. Dynamic Programming
11. Graphs
12. Trees
13. Linked Lists
14. Backtracking
15. Bit Manipulation
16. Tries
17. Math & Number Theory
18. Matrix / Grid
19. Union-Find (DSU)
20. Cross-Cutting Meta-Tricks

---

## 1. Arrays & Hashing

### 1.1 — Prefix Sum

**Core idea:** Precompute cumulative sums so any subarray sum is O(1) via `prefix[r] - prefix[l-1]`.

**How to identify:** Any problem asking about subarray sums, range sums, or "sum equals K" with negative numbers.

**Key tricks:**

- **Subarray sum equals K** — store prefix sums in a hashmap. At each index, check if `prefix[i] - K` exists in the map. This counts subarrays in O(n). The hashmap stores `{prefix_value: count}`. Initialize with `{0: 1}`.
- **Subarray sum divisible by K** — store `prefix[i] % K` in the hashmap instead. Two prefixes with the same remainder form a subarray divisible by K. Watch out for negative modulo in C++ — use `((prefix % K) + K) % K`.
- **2D prefix sum** — for rectangle sum queries on a matrix. `prefix[i][j]` = sum of rectangle from (0,0) to (i,j). Rectangle sum = inclusion-exclusion with four corners.
- **Prefix XOR** — same idea but with XOR. Subarray XOR = `prefix[r] ^ prefix[l-1]`. Used for "subarray XOR equals K" problems.

**Common transformations:**

- "Count subarrays with sum = K" (with negatives) → prefix sum + hashmap, NOT sliding window.
- "Count subarrays with sum = K" (all positive) → sliding window IS valid.
- "Maximum subarray sum of length ≥ K" → prefix sum + monotonic deque.

**Example problems:** LC 560 (Subarray Sum Equals K), LC 974 (Subarray Sums Divisible by K), LC 304 (2D Range Sum Query), LC 1310 (XOR Queries of a Subarray), LC 525 (Contiguous Array — treat 0 as -1, then prefix sum).

---

### 1.2 — Difference Array

**Core idea:** For bulk range-increment operations, instead of updating every element in a range, mark +val at start and -val at end+1, then prefix-sum to reconstruct.

**How to identify:** "Apply K operations, each adds value to a range [l, r]" — any time you're repeatedly incrementing ranges.

**Key trick:** Create array `diff[]` of same size. For each operation adding `val` to `[l, r]`: `diff[l] += val`, `diff[r+1] -= val`. Then prefix-sum `diff` to get the final array.

**Example problems:** LC 370 (Range Addition), LC 1109 (Corporate Flight Bookings), LC 1094 (Car Pooling).

---

### 1.3 — Kadane's Algorithm

**Core idea:** Track `current_max` ending at each position. At each step: `current_max = max(nums[i], current_max + nums[i])`. The decision is: extend the previous subarray, or start fresh.

**How to identify:** "Maximum subarray sum" (contiguous).

**Key tricks:**

- **Maximum product subarray** — track BOTH `current_max` and `current_min` because a negative times a negative becomes positive. At each step, if `nums[i]` is negative, swap max and min before updating.
- **Circular variant** — max circular subarray = max(standard Kadane, total_sum - min_subarray_sum). Edge case: if all elements are negative, the min subarray is the entire array, so don't use the circular formula.

**Example problems:** LC 53 (Maximum Subarray), LC 152 (Maximum Product Subarray), LC 918 (Maximum Sum Circular Subarray).

---

### 1.4 — Boyer-Moore Majority Vote

**Core idea:** Maintain a candidate and a count. For each element: if count is 0, set new candidate; if element equals candidate, increment; else decrement. The majority element (> n/2) survives.

**How to identify:** "Find element appearing more than n/2 times" or "more than n/3 times."

**Key trick:** For n/3 majority, maintain TWO candidates and TWO counts. At most 2 elements can appear > n/3 times. After the pass, verify both candidates with a second pass.

**Example problems:** LC 169 (Majority Element), LC 229 (Majority Element II).

---

### 1.5 — Cyclic Sort

**Core idea:** When you have numbers in range [1, N] in an array of size N, put each number at its "correct" index (num-1). Walk the array: if `nums[i] != i+1`, swap `nums[i]` to its correct position. Then scan for mismatches.

**How to identify:** "Find missing/duplicate number in range [1, N]" with O(1) space.

**Example problems:** LC 268 (Missing Number — also solvable with XOR), LC 287 (Find the Duplicate — also Floyd's cycle), LC 442 (Find All Duplicates), LC 448 (Find All Numbers Disappeared).

---

### 1.6 — Contribution Technique

**Core idea:** Instead of iterating over all subarrays and computing their property, count how much each element contributes across all subarrays.

**How to identify:** "Sum of (min/max/some property) over all subarrays."

**Key trick:** For each element, determine how many subarrays it's the min/max of. Use monotonic stack to find the nearest smaller/larger on left and right. The element at index `i` is the min of `left_count * right_count` subarrays.

**Example problems:** LC 907 (Sum of Subarray Minimums), LC 2104 (Sum of Subarray Ranges), LC 828 (Count Unique Characters of All Substrings).

---

## 2. Two Pointers

### 2.1 — Opposite-Direction Pointers

**Core idea:** One pointer at start, one at end, moving inward.

**How to identify:** Sorted array + "find pair with property" (sum, difference). Or any shrinking-from-both-sides logic.

**Key tricks:**

- **Two Sum (sorted)** — if sum < target, move left pointer right; if sum > target, move right pointer left. O(n).
- **Container With Most Water** — always move the shorter side inward, because moving the taller side can never increase the area (width decreases, height is bottlenecked by the shorter side).
- **Trapping Rain Water** — two-pointer approach: maintain `left_max` and `right_max`. Process the side with the smaller max first, because water is bounded by the smaller side.

**Example problems:** LC 167 (Two Sum II), LC 11 (Container With Most Water), LC 42 (Trapping Rain Water), LC 125 (Valid Palindrome).

---

### 2.2 — Same-Direction Pointers

**Core idea:** Both pointers move left to right, one "slow" and one "fast."

**How to identify:** "Remove/overwrite in place," "partition array."

**Key tricks:**

- **Remove element in place** — slow pointer tracks write position, fast pointer scans. When fast finds a value to keep, write it at slow and advance slow.
- **Move zeroes** — same pattern. Slow pointer tracks next non-zero position.

**Example problems:** LC 283 (Move Zeroes), LC 26 (Remove Duplicates from Sorted Array), LC 27 (Remove Element).

---

### 2.3 — Three Pointers / K-Sum

**Core idea:** Fix one element with an outer loop, then use two pointers on the rest.

**How to identify:** "Find triplets/quadruplets with sum = target" or "3Sum / 4Sum."

**Key tricks:**

- **Sort first.** This enables two-pointer on the remaining elements.
- **Skip duplicates** — after finding a valid pair, skip while `nums[left] == nums[left+1]` and `nums[right] == nums[right-1]`.
- **Generalize to K-Sum** — recursively reduce K-Sum to 2-Sum. Fix one element, recurse on (K-1)-Sum.

**Example problems:** LC 15 (3Sum), LC 16 (3Sum Closest), LC 18 (4Sum).

---

### 2.4 — Dutch National Flag (3-Way Partition)

**Core idea:** Three pointers: `low`, `mid`, `high`. Partition array into three regions (e.g., 0s, 1s, 2s) in a single pass.

**How to identify:** "Sort array with only 3 distinct values" or "partition around a pivot."

**Key trick:** `mid` scans left to right. If `nums[mid] == 0`, swap with `low`, advance both. If `nums[mid] == 2`, swap with `high`, decrement `high` only (because the swapped value is unexamined). If 1, just advance `mid`.

**Example problems:** LC 75 (Sort Colors).

---

## 3. Sliding Window

### 3.1 — Fixed-Size Window

**Core idea:** Maintain a window of exactly size K. Slide it: add the new right element, remove the old left element.

**How to identify:** "Subarray/substring of exact size K" + some aggregate (max sum, number of distinct, etc.).

**Example problems:** LC 643 (Maximum Average Subarray I), LC 567 (Permutation in String — window of size len(s1)), LC 1456 (Maximum Number of Vowels in a Substring of Given Length).

---

### 3.2 — Variable-Size Window

**Core idea:** Expand right pointer to include more elements. When the window violates a constraint, shrink from the left until valid again. Track the best valid window.

**How to identify:** "Longest/shortest subarray/substring with constraint" where all elements are non-negative or the constraint is monotonic (adding elements can only make it worse/better).

**Key tricks:**

- **Longest variant** — expand right always, shrink left while invalid. Update answer after shrinking (the window is now the longest valid ending at right).
- **Shortest variant** — expand right until valid, then shrink left while STILL valid, updating answer at each valid state.
- **Why this doesn't work with negative numbers** — the monotonicity breaks. Adding a negative number can decrease the sum, so shrinking from the left might not help. Use prefix sum + hashmap instead.

**Example problems:** LC 3 (Longest Substring Without Repeating Characters), LC 76 (Minimum Window Substring), LC 209 (Minimum Size Subarray Sum), LC 424 (Longest Repeating Character Replacement), LC 1004 (Max Consecutive Ones III).

---

### 3.3 — "Exactly K" = "At Most K" − "At Most K−1"

**Core idea:** Counting subarrays with exactly K of something is hard directly. Instead, write a helper `atMost(K)` that counts subarrays with ≤ K, then the answer is `atMost(K) - atMost(K-1)`.

**How to identify:** "Count subarrays with exactly K distinct elements" or "exactly K zeros" — any "exactly K" counting problem over subarrays.

**Key trick:** The `atMost(K)` function is a standard variable-size sliding window. At each position of the right pointer, the number of valid subarrays ending there is `right - left + 1` (every subarray starting from any index in [left, right]).

**Example problems:** LC 992 (Subarrays with K Different Integers), LC 1248 (Count Number of Nice Subarrays), LC 930 (Binary Subarrays With Sum).

---

### 3.4 — "Remove From Both Ends" = Sliding Window on the Middle

**Core idea:** If the problem says "remove elements from the left and/or right side" to minimize something, the complement is "keep a contiguous middle subarray" — which is a sliding window.

**How to identify:** "Remove from front and back to achieve X" → equivalent to "find contiguous subarray (the part you keep) that achieves the complement of X."

**Example:** "Minimum operations to reduce array to size 0 by removing from ends where removed sum = total" → find longest subarray with sum = total - target.

**Example problems:** LC 1658 (Minimum Operations to Reduce X to Zero), LC 2516 (Take K of Each Character From Left and Right).

---

### 3.5 — Sliding Window with Frequency Map

**Core idea:** Maintain a hashmap of character/element frequencies within the window. Track a `formed` or `valid` counter to know when the window satisfies the constraint.

**How to identify:** "Smallest window containing all characters of pattern," "window with at most K distinct."

**Key trick for Minimum Window Substring:** maintain `required` (number of unique chars in target) and `formed` (number of unique chars in window meeting target frequency). Expand right until `formed == required`, then shrink left while maintaining validity.

**Example problems:** LC 76 (Minimum Window Substring), LC 438 (Find All Anagrams), LC 30 (Substring with Concatenation of All Words).

---

## 4. Binary Search

### 4.1 — Standard Binary Search

**Core idea:** On a sorted array, repeatedly halve the search space.

**Key trick — avoiding bugs:** Use the `[lo, hi)` half-open convention consistently. `lo = 0`, `hi = n`. While `lo < hi`: `mid = lo + (hi - lo) / 2`. For lower bound (first true): if condition, `hi = mid`; else `lo = mid + 1`. For upper bound (last true): if condition, `lo = mid + 1`; else `hi = mid`, then answer is `lo - 1`.

---

### 4.2 — Binary Search on Answer (Parametric Search)

**Core idea:** When the answer itself is monotonic (if X works, then X+1 also works, or vice versa), binary search on the answer value and check feasibility.

**How to identify:** Look for these patterns:
- "Minimize the maximum" or "Maximize the minimum."
- "Can you achieve X with value ≤ Y?" where the feasibility is monotonic in Y.
- The answer is a number you're optimizing, and you can write a `canDo(value)` check.

**Key trick:** Frame the problem as: "What is the smallest/largest value V such that `canDo(V)` is true?" Then binary search on V. The `canDo` function is usually a greedy simulation.

**Common transformations:**
- "Split array into K subarrays, minimize the max sum" → binary search on the max sum, greedily check if you can split into ≤ K parts.
- "Place K items with maximum minimum gap" → binary search on the gap, greedily check placement.
- "Speed to finish in H hours" → binary search on speed.

**Example problems:** LC 875 (Koko Eating Bananas), LC 410 (Split Array Largest Sum), LC 1011 (Capacity To Ship Packages), LC 774 (Minimize Max Distance to Gas Station), LC 2560 (House Robber IV).

---

### 4.3 — Binary Search on Rotated / Modified Arrays

**Core idea:** Determine which half is sorted, then decide which half to search.

**Key trick:** Compare `nums[mid]` with `nums[lo]` (or `nums[hi]`). If `nums[lo] <= nums[mid]`, the left half is sorted. Check if target is in the sorted half; if yes, search there; else search the other half.

**For finding minimum:** Compare `nums[mid]` with `nums[hi]`. If `nums[mid] > nums[hi]`, the min is in the right half. Else it's in the left half (including mid).

**Duplicate handling:** When `nums[lo] == nums[mid] == nums[hi]`, you can't determine which half is sorted. Shrink both ends: `lo++`, `hi--`. This makes worst case O(n).

**Example problems:** LC 33 (Search in Rotated Sorted Array), LC 81 (Search in Rotated II — with duplicates), LC 153 (Find Minimum in Rotated), LC 154 (Find Minimum II — with duplicates).

---

### 4.4 — Binary Search for LIS (Patience Sorting)

**Core idea:** Maintain a `tails` array where `tails[i]` = smallest tail element of all increasing subsequences of length `i+1`. For each element, binary search for its position in `tails`. The length of `tails` is the LIS length.

**How to identify:** "Longest increasing subsequence" in O(n log n). Also applies to envelope problems.

**Key trick:** Use `lower_bound` (bisect_left) for strictly increasing. Use `upper_bound` (bisect_right) for non-decreasing.

**Common transformation — Russian Doll Envelopes:** Sort by width ascending, then by height DESCENDING (within same width). Then LIS on heights. The descending sort within same width prevents using two envelopes with the same width.

**Example problems:** LC 300 (Longest Increasing Subsequence), LC 354 (Russian Doll Envelopes), LC 1964 (Find the Longest Valid Obstacle Course).

---

### 4.5 — Median of Two Sorted Arrays

**Core idea:** Binary search on the partition point in the shorter array. For a given partition, check if the cross-boundary elements satisfy `maxLeft1 <= minRight2` and `maxLeft2 <= minRight1`.

**How to identify:** "Median of two sorted arrays in O(log(min(m,n)))."

**Example problems:** LC 4 (Median of Two Sorted Arrays).

---

### 4.6 — Find Peak Element

**Core idea:** If `nums[mid] > nums[mid+1]`, a peak exists on the left side (including mid). Else it exists on the right. This works because the endpoints are defined as −∞.

**How to identify:** "Find any local maximum in an unsorted array."

**Example problems:** LC 162 (Find Peak Element), LC 852 (Peak Index in Mountain Array).

---

## 5. Stack & Monotonic Stack

### 5.1 — Monotonic Stack

**Core idea:** Maintain a stack where elements are always in increasing (or decreasing) order. When a new element violates the order, pop elements — those popped elements have found their "answer" (next greater, next smaller, etc.).

**How to identify:** "Next greater/smaller element" for each position. Or any problem where for each element, you need to know the nearest element that is larger/smaller on the left or right.

**Key tricks:**

- **Next Greater Element (right)** — iterate left to right. Maintain a decreasing stack (of indices). When `nums[i] > stack.top()`, pop and record `nums[i]` as the answer for the popped element.
- **Previous Smaller Element (left)** — iterate left to right. Maintain an increasing stack. When `nums[i] <= stack.top()`, pop. After popping, `stack.top()` is the previous smaller element for `nums[i]`.
- **Which direction to iterate?** If you want the NEXT greater/smaller (looking right), iterate left to right. If you want the PREVIOUS greater/smaller (looking left), iterate left to right and check the stack top after popping.
- **Strict vs non-strict** — for "strictly greater", pop when `>=`. For "greater or equal", pop when `>`. Getting this wrong causes subtle bugs.

**Example problems:** LC 496 (Next Greater Element I), LC 503 (Next Greater Element II — circular: double the array), LC 739 (Daily Temperatures), LC 907 (Sum of Subarray Minimums).

---

### 5.2 — Largest Rectangle in Histogram

**Core idea:** For each bar, find how far it can extend left and right as the shortest bar. Use a monotonic increasing stack (of indices). When a bar is popped by a shorter bar, the popped bar's rectangle has known width boundaries.

**How to identify:** "Largest rectangle under histogram" or "maximal rectangle in binary matrix."

**Key trick for Maximal Rectangle in matrix:** Treat each row as the "ground floor" of a histogram. For each row, compute the histogram heights (cumulative from top: if cell is 1, height = height above + 1; if 0, height = 0). Then apply largest rectangle in histogram on each row.

**Example problems:** LC 84 (Largest Rectangle in Histogram), LC 85 (Maximal Rectangle).

---

### 5.3 — Stack for Expressions & Parentheses

**Core idea:** Use a stack to handle nested structure.

**Key tricks:**

- **Valid parentheses** — push open brackets, pop on matching close. Stack should be empty at the end.
- **Basic Calculator** — use stack to save the current result and sign when you see '('. Process the sub-expression. On ')', pop and combine.
- **Decode string "3[abc]"** — push current string and repeat count onto stack when you see '['. On ']', pop and repeat.
- **Longest valid parentheses** — push indices onto stack. Initialize stack with -1 as a boundary marker. On ')', pop. If stack is empty, push current index as new boundary. Else, current length = `i - stack.top()`.

**Example problems:** LC 20 (Valid Parentheses), LC 224 (Basic Calculator), LC 394 (Decode String), LC 32 (Longest Valid Parentheses).

---

### 5.4 — Monotonic Stack for Greedy Deletion

**Core idea:** Build the result by greedily keeping/removing elements. Use a stack: if the current element is "better" than the top and you still have removals left, pop.

**How to identify:** "Remove K digits to make smallest/largest number" or "remove duplicate letters maintaining order."

**Key trick for Remove K Digits:** iterate through digits. While the stack top is larger than current digit and K > 0, pop (remove that digit). Push current digit. After processing, remove remaining K digits from the end.

**Example problems:** LC 402 (Remove K Digits), LC 316 (Remove Duplicate Letters), LC 321 (Create Maximum Number).

---

### 5.5 — The 132 Pattern

**Core idea:** Iterate right to left. Maintain a monotonic decreasing stack. The "2" in the 132 pattern is the largest value that was ever popped from the stack.

**How to identify:** "Find i < j < k such that nums[i] < nums[k] < nums[j]."

**Key trick:** As you iterate right to left, track the maximum popped value as `third` (the "2" in 132). If you encounter a value less than `third`, you've found the "1" and the pattern exists.

**Example problems:** LC 456 (132 Pattern).

---

## 6. Queue & Monotonic Deque

### 6.1 — Monotonic Deque for Sliding Window Min/Max

**Core idea:** Maintain a deque of indices where the values are in decreasing order (for max) or increasing order (for min). The front of the deque is always the current window's max/min.

**How to identify:** "Sliding window maximum/minimum." Or any DP where the transition looks at the best value in a sliding range of previous states.

**Key tricks:**

- **For sliding window max:** deque stores indices in decreasing order of value. Before adding a new element, pop from the BACK while `nums[back] <= nums[i]` (they can never be the max again). Pop from the FRONT if the front index is out of the window.
- **Why a deque and not a heap?** A heap can't efficiently remove elements that fall out of the window. A deque gives O(1) amortized per element.
- **DP optimization** — if your DP transition is `dp[i] = max(dp[j]) + something` for `j` in a sliding range `[i-K, i-1]`, use a monotonic deque on the `dp` values.

**Example problems:** LC 239 (Sliding Window Maximum), LC 1425 (Constrained Subsequence Sum — DP + deque), LC 862 (Shortest Subarray with Sum at Least K — prefix sum + deque).

---

### 6.2 — 0-1 BFS with Deque

**Core idea:** When edge weights are only 0 or 1, use a deque instead of a priority queue. Push 0-weight edges to the front, 1-weight edges to the back.

**How to identify:** Graph shortest path where edges have weights 0 or 1 (e.g., grid where moving on roads is free but crossing walls costs 1).

**Example problems:** LC 1368 (Minimum Cost to Make at Least One Valid Path in a Grid), LC 2290 (Minimum Obstacle Removal to Reach Corner).

---

## 7. Heap / Priority Queue

### 7.1 — Top-K Pattern

**Core idea:** Use a min-heap of size K to track the K largest elements. When a new element is larger than the heap top, replace it.

**How to identify:** "K-th largest," "top K frequent," "K closest."

**Key trick:** For K-th largest, use a MIN-heap of size K (not a max-heap of all elements). The top of this heap is the K-th largest. This is O(n log K) instead of O(n log n).

**Alternative — Quickselect:** O(n) average for K-th element, but O(n²) worst case without randomization.

**Example problems:** LC 215 (Kth Largest Element), LC 347 (Top K Frequent Elements — also solvable with bucket sort in O(n)), LC 973 (K Closest Points to Origin).

---

### 7.2 — Two Heaps (Median Finding)

**Core idea:** Maintain a max-heap for the smaller half and a min-heap for the larger half. The median is the top of the larger heap (or average of both tops).

**How to identify:** "Find median from data stream" or "median maintenance."

**Key trick:** Balance the heaps so their sizes differ by at most 1. Always push to max-heap first, then move max-heap's top to min-heap. If min-heap grows larger, move its top back to max-heap.

**Example problems:** LC 295 (Find Median from Data Stream), LC 480 (Sliding Window Median — two heaps + lazy deletion).

---

### 7.3 — Merge K Sorted Lists/Streams

**Core idea:** Push the head of each list into a min-heap. Pop the smallest, push its next element.

**How to identify:** "Merge K sorted arrays/lists" or "smallest range covering one element from each list."

**Example problems:** LC 23 (Merge K Sorted Lists), LC 378 (Kth Smallest Element in Sorted Matrix — treat rows as sorted lists), LC 632 (Smallest Range Covering Elements from K Lists).

---

### 7.4 — Lazy Deletion from Heap

**Core idea:** When you need to "remove" arbitrary elements from a heap but can't (heaps only support top removal), mark elements as deleted and skip them when they surface at the top.

**How to identify:** Sliding window median, or any heap where you need to remove non-top elements.

**Key trick:** Use a separate hashmap/multiset to record "pending deletions." When popping from the heap, check if the popped element is in the deletion set. If so, discard it and pop again.

**Example problems:** LC 480 (Sliding Window Median).

---

### 7.5 — Greedy Scheduling with Heaps

**Core idea:** Use a max-heap to greedily process the most urgent/frequent item first. After using an item, it becomes available again after a cooldown.

**How to identify:** "Schedule tasks with cooldown," "reorganize string so no two adjacent are the same."

**Key trick for Reorganize String:** pop the most frequent character from the max-heap, append it, then push the previously used character back (it was held out for one round to prevent adjacency).

**Example problems:** LC 767 (Reorganize String), LC 621 (Task Scheduler), LC 358 (Rearrange String K Distance Apart).

---

## 8. Greedy

### 8.1 — How to Identify Greedy Problems

**Key signal:** The problem has an "optimal substructure" where making the locally best choice at each step leads to the global optimum, AND there are no overlapping subproblems requiring revisiting decisions.

**Validation:** Try to construct a counterexample. If you can't find one, greedy likely works. The formal tool is the "exchange argument": show that any non-greedy solution can be improved by swapping to the greedy choice.

**When greedy fails → use DP:** If a locally optimal choice at step K might prevent a better choice at step K+3, you need DP.

---

### 8.2 — Interval Scheduling (Sort by End Time)

**Core idea:** To maximize the number of non-overlapping intervals, sort by END time and greedily pick the earliest-ending interval that doesn't conflict.

**How to identify:** "Maximum non-overlapping intervals," "minimum intervals to remove to eliminate all overlaps."

**Key trick:** "Minimum removals" = total − "maximum non-overlapping." Don't solve it as a removal problem.

**Example problems:** LC 435 (Non-overlapping Intervals), LC 452 (Minimum Number of Arrows to Burst Balloons).

---

### 8.3 — Jump Game Pattern

**Core idea:** Track the farthest reachable index. If at any point your current index exceeds farthest, you're stuck.

**Key tricks:**

- **Jump Game I (can you reach the end?)** — maintain `farthest = max(farthest, i + nums[i])`. If `i > farthest`, return false.
- **Jump Game II (minimum jumps)** — maintain `current_end` (farthest you can go with current number of jumps) and `farthest`. When `i` reaches `current_end`, you must jump: `current_end = farthest`, `jumps++`.

**Example problems:** LC 55 (Jump Game), LC 45 (Jump Game II), LC 1024 (Video Stitching).

---

### 8.4 — Sorting by a Derived Key

**Core idea:** Sometimes the greedy criterion isn't obvious — you need to sort by a computed value.

**Key tricks:**

- **Custom comparison** — for "arrange numbers to form the largest number," compare by concatenation: compare `a+b` vs `b+a` as strings.
- **Weighted job scheduling** — sort by end time, then DP with binary search.
- **Two-dimensional sorting** — sort by one dimension, then solve the other. Example: "Queue reconstruction by height" — sort by height descending (then by index ascending), then insert into result at the index position.

**Example problems:** LC 179 (Largest Number), LC 406 (Queue Reconstruction by Height), LC 1029 (Two City Scheduling — sort by `cost_A - cost_B`).

---

### 8.5 — Gas Station (Circular Greedy)

**Core idea:** If total gas ≥ total cost, a solution exists. Find the starting point by tracking a running surplus. Whenever surplus goes negative, reset the start to the next station.

**Why this works:** If you can't reach station j from station i, then you can't reach j from any station between i and j either (because you'd arrive at those intermediate stations with less gas).

**Example problems:** LC 134 (Gas Station).

---

## 9. Intervals

### 9.1 — Merge Overlapping Intervals

**Core idea:** Sort by start time. Maintain a running interval. If the next interval overlaps (its start ≤ current end), merge by extending end. Else, close the current interval and start a new one.

**Example problems:** LC 56 (Merge Intervals), LC 57 (Insert Interval — insert then merge).

---

### 9.2 — Line Sweep / Event-Based Processing

**Core idea:** Convert intervals into events: `(start, +1)` and `(end, -1)`. Sort events. Sweep through, maintaining a running count. The maximum count is the answer for "maximum overlap."

**How to identify:** "Maximum number of overlapping intervals at any point," "minimum meeting rooms," "minimum platforms."

**Key trick for event ordering:** When a start and end happen at the same time, the ordering depends on semantics. For meeting rooms, an end at time T frees the room before a start at time T needs it, so process ends before starts at the same timestamp. Use `(time, +1 for start / -1 for end)` and sort — the -1 naturally comes first.

**Example problems:** LC 253 (Meeting Rooms II), LC 1094 (Car Pooling), LC 731/732 (My Calendar II/III).

---

### 9.3 — Interval DP + Binary Search

**Core idea:** Sort intervals by end time. For each interval, binary search for the last non-overlapping previous interval. DP transition: `dp[i] = max(dp[i-1], value[i] + dp[last_non_overlapping])`.

**How to identify:** "Maximum weight/profit from non-overlapping intervals" (weighted interval scheduling).

**Key trick:** After sorting by end, use `upper_bound` (or `bisect_right`) on the start of interval i to find the latest interval that ends before interval i starts.

**Example problems:** LC 1235 (Maximum Profit in Job Scheduling), LC 1751 (Maximum Number of Events That Can Be Attended II).

---

## 10. Dynamic Programming

### 10.1 — How to Identify DP Problems

**Key signals:**
- "Count the number of ways."
- "Minimum/maximum cost/value."
- "Can you achieve X?" (feasibility with choices).
- Overlapping subproblems: the same sub-computation appears multiple times.
- Optimal substructure: optimal solution builds on optimal sub-solutions.

**DP vs. greedy:** If making a choice now affects future choices in complex ways (not just "take the best"), it's DP.

**DP vs. backtracking:** If you need ALL solutions, it's backtracking. If you need the COUNT or the OPTIMAL, it's DP.

---

### 10.2 — 1D DP Patterns

**Core patterns:**

- **Climbing stairs / Fibonacci** — `dp[i] = dp[i-1] + dp[i-2]`. Any problem where you arrive at state i from a fixed set of previous states.
- **House Robber** — `dp[i] = max(dp[i-1], dp[i-2] + nums[i])`. "Can't take adjacent" constraint.
- **House Robber circular** — run House Robber twice: once on `nums[0..n-2]`, once on `nums[1..n-1]`. Take the max.
- **Decode Ways** — `dp[i]` depends on whether the last 1 digit and last 2 digits form valid decodings.
- **Coin Change (minimum coins)** — `dp[amount] = min(dp[amount - coin] + 1)` for each coin. Unbounded knapsack variant.
- **Word Break** — `dp[i]` = true if `s[0..i]` can be segmented. Check all `j < i`: if `dp[j]` is true and `s[j..i]` is in the dictionary.

**Example problems:** LC 70 (Climbing Stairs), LC 198/213 (House Robber I/II), LC 91 (Decode Ways), LC 322 (Coin Change), LC 139 (Word Break).

---

### 10.3 — 2D DP Patterns

**Core patterns:**

- **Grid paths** — `dp[i][j] = dp[i-1][j] + dp[i][j-1]`. Count paths from top-left to bottom-right.
- **LCS (Longest Common Subsequence)** — `dp[i][j]` = LCS of first i chars of s1 and first j chars of s2. If `s1[i] == s2[j]`, `dp[i][j] = dp[i-1][j-1] + 1`. Else `max(dp[i-1][j], dp[i][j-1])`.
- **Edit Distance** — `dp[i][j]` = min edits for s1[0..i] to s2[0..j]. Three operations: insert, delete, replace.
- **Interleaving String** — `dp[i][j]` = can s3[0..i+j] be formed by interleaving s1[0..i] and s2[0..j].

**Space optimization:** If `dp[i]` only depends on `dp[i-1]`, keep only two rows (or even one row with careful ordering).

**Example problems:** LC 62/63 (Unique Paths I/II), LC 1143 (LCS), LC 72 (Edit Distance), LC 97 (Interleaving String), LC 10 (Regular Expression Matching), LC 44 (Wildcard Matching).

---

### 10.4 — Knapsack

**0/1 Knapsack** — each item used at most once. Iterate items in the outer loop, capacity in the INNER loop going RIGHT TO LEFT (so each item is only used once).

```
for each item:
    for w = W down to item.weight:
        dp[w] = max(dp[w], dp[w - item.weight] + item.value)
```

**Unbounded Knapsack** — each item used unlimited times. Same structure but inner loop goes LEFT TO RIGHT.

```
for each item:
    for w = item.weight to W:
        dp[w] = max(dp[w], dp[w - item.weight] + item.value)
```

**How to identify:** "Select items with weights/costs, maximize value under capacity." "Partition into two subsets with equal sum" is 0/1 knapsack with target = total/2.

**Common transformations:**
- "Target Sum (+/- each element)" → partition into two groups P and N where P - N = target, P + N = total → subset sum with target = (total + target) / 2.
- "Coin Change (count ways)" → unbounded knapsack counting.

**Example problems:** LC 416 (Partition Equal Subset Sum), LC 494 (Target Sum), LC 518 (Coin Change 2 — count ways), LC 474 (Ones and Zeroes — 2D knapsack).

---

### 10.5 — Interval DP

**Core idea:** `dp[i][j]` = optimal answer for the subarray `[i..j]`. Try every possible split point k between i and j.

**How to identify:** "Merge adjacent things," "break/burst things from a range," "cost of combining."

**Iteration order:** Fill by increasing length. `for len = 2 to n: for i = 0 to n-len: j = i+len-1`.

**Key trick for Burst Balloons:** Think in reverse — instead of removing balloons, think of which balloon is LAST to be burst in range [i, j]. That balloon k gets `nums[i-1] * nums[k] * nums[j+1]` (because everything else in the range is already gone).

**Example problems:** LC 312 (Burst Balloons), LC 1039 (Minimum Score Triangulation), LC 516 (Longest Palindromic Subsequence — `dp[i][j]`), LC 5 (Longest Palindromic Substring), LC 87 (Scramble String).

---

### 10.6 — Bitmask DP

**Core idea:** `dp[mask]` where `mask` is a bitmask representing which elements have been used/visited. Used when n is small (n ≤ 20).

**How to identify:** "Assign n items to n slots" (n ≤ 20), "visit all nodes" (TSP), "partition into groups."

**Key tricks:**

- **Iterate over subsets of a mask:** `for (int sub = mask; sub > 0; sub = (sub - 1) & mask)`.
- **TSP** — `dp[mask][i]` = minimum cost to visit all cities in `mask`, ending at city `i`. Transition: try extending from each city `j` in `mask` that isn't `i`.
- **Count set bits** — `__builtin_popcount(mask)` in C++.

**Example problems:** LC 1986 (Minimum Number of Work Sessions), LC 691 (Stickers to Spell Word), LC 943 (Find the Shortest Superstring), LC 1125 (Smallest Sufficient Team).

---

### 10.7 — State Machine DP (Stock Problems)

**Core idea:** Define states explicitly: holding a stock, not holding, in cooldown, etc. Transitions between states at each day.

**How to identify:** The "Buy and Sell Stock" family, or any problem with distinct modes/phases.

**Framework:**

```
At each day i:
    hold[i] = max(hold[i-1], not_hold[i-1] - prices[i])       // keep holding, or buy today
    not_hold[i] = max(not_hold[i-1], hold[i-1] + prices[i])    // keep idle, or sell today
```

**Variations:**
- Cooldown after selling → add a `cooldown[i]` state; can only buy from cooldown.
- Transaction limit K → add a transaction dimension: `hold[i][k]`, `not_hold[i][k]`.
- Transaction fee → subtract fee on sell.

**Example problems:** LC 121 (Best Time I — single transaction), LC 122 (Best Time II — unlimited), LC 123 (Best Time III — at most 2), LC 188 (Best Time IV — at most K), LC 309 (With Cooldown), LC 714 (With Transaction Fee).

---

### 10.8 — DP on Strings: Subsequences vs. Substrings

**Subsequence DP:** Usually 2D. `dp[i][j]` considers prefix of length i of string A and prefix of length j of string B. Characters don't need to be contiguous.

**Substring/subarray DP:** Usually `dp[i]` or `dp[i][j]` where the constraint is that elements MUST be contiguous. Often the DP requires "ending at position i" semantics.

**Key distinction:** For longest common subsequence, when chars don't match, you take `max(dp[i-1][j], dp[i][j-1])`. For longest common substring, when chars don't match, `dp[i][j] = 0` (reset, since contiguity breaks).

---

### 10.9 — Space Optimization

**Rolling array:** If `dp[i]` only depends on `dp[i-1]`, use two arrays and alternate. Or even one array with careful ordering (for 0/1 knapsack, iterate capacity in reverse).

**When to apply:** Almost all 2D DP where one dimension is "step/index" can be compressed to 1D if you only look back one step.

---

### 10.10 — DP + Monotonic Deque Optimization

**Core idea:** When the DP transition is `dp[i] = min/max over dp[j] for j in [i-K, i-1]` plus some cost, maintaining a deque over the DP values gives O(n) total instead of O(nK).

**How to identify:** DP where the "lookback" is a sliding window of previous states.

**Example problems:** LC 1425 (Constrained Subsequence Sum), LC 862 (Shortest Subarray with Sum ≥ K).

---

### 10.11 — Longest Increasing Subsequence (LIS) Variants

**O(n²) DP:** `dp[i]` = length of LIS ending at index i. `dp[i] = max(dp[j] + 1)` for all `j < i` where `nums[j] < nums[i]`.

**O(n log n) with patience sorting:** See Binary Search section (4.4).

**Common variants:**
- Longest non-decreasing: use `upper_bound` instead of `lower_bound`.
- Number of LIS: track `count[i]` alongside `dp[i]`.
- Longest string chain: sort by length, DP where you try removing each character.

**Example problems:** LC 300 (LIS), LC 673 (Number of LIS), LC 1048 (Longest String Chain), LC 354 (Russian Doll Envelopes).

---

### 10.12 — Tree DP

**Core idea:** Root the tree. DFS from the root. For each node, compute `dp[node]` from the `dp` values of its children.

**How to identify:** "Optimal value on a tree where the decision at a node depends on its subtrees."

**Key trick — returning multiple values:** The diameter of a tree requires returning the HEIGHT of each subtree to the parent, while updating a global max with `left_height + right_height` at each node. The value you return (height) is different from the value you're computing (diameter).

**Example problems:** LC 337 (House Robber III — `dp[node] = (rob_this, skip_this)`), LC 543 (Diameter of Binary Tree), LC 124 (Binary Tree Maximum Path Sum).

---

## 11. Graphs

### 11.1 — BFS for Shortest Path (Unweighted)

**Core idea:** BFS from the source. The first time you reach a node is the shortest path.

**How to identify:** "Shortest path / minimum steps" in an unweighted graph or grid.

**Key tricks:**

- **Multi-source BFS** — push ALL sources into the queue initially with distance 0. Useful for "distance to nearest X for every cell."
- **BFS on implicit graphs** — the graph isn't given explicitly. The "states" are derived (e.g., board configurations, string transformations). Encode states as keys for a visited set.
- **Bidirectional BFS** — BFS from both source and target. When frontiers meet, you're done. Useful when the branching factor is large.

**Example problems:** LC 994 (Rotting Oranges — multi-source), LC 127 (Word Ladder — BFS on string transformations), LC 1091 (Shortest Path in Binary Matrix), LC 752 (Open the Lock).

---

### 11.2 — DFS: Cycle Detection & Connectivity

**Key tricks:**

- **Cycle in directed graph** — use 3 colors: WHITE (unvisited), GRAY (in current DFS path), BLACK (fully processed). If you reach a GRAY node, there's a cycle. This is the standard approach for topological sort cycle detection.
- **Cycle in undirected graph** — DFS with parent tracking. If you reach a visited node that isn't the parent, there's a cycle. Or use Union-Find.
- **Connected components** — run DFS/BFS from each unvisited node, count the number of times you start a new traversal.

**Example problems:** LC 200 (Number of Islands — grid DFS/BFS), LC 323 (Number of Connected Components), LC 547 (Number of Provinces).

---

### 11.3 — Topological Sort

**Core idea:** Order nodes so every edge (u, v) has u before v. Only possible in a DAG.

**Two approaches:**
- **Kahn's (BFS):** compute in-degrees. Push all nodes with in-degree 0 into a queue. Process: decrement in-degree of neighbors, push any that hit 0. If the final order has fewer than n nodes, there's a cycle.
- **DFS:** run DFS and push to result on POST-ORDER (after processing all children). Reverse the result.

**How to identify:** "Prerequisites / dependencies," "ordering constraints," "course schedule."

**Key trick:** If you need the LEXICOGRAPHICALLY smallest topological order, use a min-heap instead of a queue in Kahn's.

**Example problems:** LC 207 (Course Schedule — just detect if possible), LC 210 (Course Schedule II — return the order), LC 269 (Alien Dictionary), LC 2115 (Find All Possible Recipes).

---

### 11.4 — Dijkstra's Algorithm

**Core idea:** Min-heap BFS for weighted graphs with non-negative edges. Pop the node with the smallest distance, relax its neighbors.

**How to identify:** "Shortest path in weighted graph (non-negative weights)."

**Key trick:** Don't use a visited set that prevents re-processing — instead, skip a node if its popped distance is greater than its current known distance. (This handles the lazy heap approach where you push duplicates.)

**Modification — K shortest paths:** allow each node to be visited up to K times. The K-th visit gives the K-th shortest path.

**Example problems:** LC 743 (Network Delay Time), LC 1631 (Path With Minimum Effort), LC 778 (Swim in Rising Water), LC 787 (Cheapest Flights Within K Stops — Bellman-Ford or modified Dijkstra/BFS).

---

### 11.5 — Bellman-Ford

**Core idea:** Relax all edges V-1 times. If any edge can still be relaxed after that, there's a negative cycle.

**How to identify:** "Shortest path with negative weights" or "at most K edges."

**Key trick for K edges:** Run only K iterations of Bellman-Ford. Copy the distance array before each iteration to avoid using paths that are too long (this is the "copy before relax" trick).

**Example problems:** LC 787 (Cheapest Flights Within K Stops).

---

### 11.6 — Bipartite Check

**Core idea:** Try to 2-color the graph. BFS/DFS, alternating colors. If you reach a neighbor with the same color, it's not bipartite.

**How to identify:** "Can you split into two groups such that no two in the same group are connected?"

**Example problems:** LC 785 (Is Graph Bipartite), LC 886 (Possible Bipartition).

---

### 11.7 — Graph Trick: Process in Reverse

**Core idea:** If the problem involves removing edges/nodes, reverse it — ADD edges/nodes and use Union-Find.

**How to identify:** "Process deletions" is hard (connectivity decreases). Process insertions in reverse using Union-Find (connectivity increases).

**Example problems:** LC 1319 (Number of Operations to Make Network Connected), LC 803 (Bricks Falling When Hit — reverse: add bricks back).

---

### 11.8 — Minimum Spanning Tree

**Kruskal's:** Sort edges by weight. Use Union-Find. Add edge if it connects two different components. Stop after n-1 edges.

**Prim's:** Start from any node. Use min-heap of edges from the current tree. Pop the lightest edge to an unvisited node.

**Example problems:** LC 1584 (Min Cost to Connect All Points), LC 1135 (Connecting Cities With Minimum Cost).

---

### 11.9 — Paint from the Boundary

**Core idea:** Instead of finding "enclosed" regions (hard), start DFS/BFS from the boundary and mark everything reachable. What's left is enclosed.

**How to identify:** "Surrounded regions," "pacific atlantic water flow."

**Example problems:** LC 130 (Surrounded Regions — mark border-connected O's first), LC 417 (Pacific Atlantic Water Flow — BFS from both oceans inward).

---

## 12. Trees

### 12.1 — Core Recursive Pattern

**Core idea:** Almost all tree problems follow: (1) recurse on left subtree, (2) recurse on right subtree, (3) combine results.

**Key trick — returning vs. updating a global:** Some problems need the function to return one thing (used by the parent's computation) while tracking a different global answer.

**Classic example — Diameter:** The function returns the HEIGHT of the subtree (parent needs this). But the DIAMETER at each node is `left_height + right_height`, which you compare against a global max.

**Classic example — Max Path Sum:** The function returns the max "single branch" sum (extending upward to parent). But the max path through a node is `node.val + max(0, left) + max(0, right)`, which you compare globally.

**Example problems:** LC 104 (Maximum Depth), LC 543 (Diameter), LC 124 (Max Path Sum), LC 110 (Balanced Binary Tree), LC 100 (Same Tree).

---

### 12.2 — BST Properties

**Core idea:** In a BST, inorder traversal produces a sorted sequence.

**Key tricks:**

- **Validate BST** — inorder traversal and check sorted order. Or, recursively pass a `(min, max)` range down. Each node must be within its range.
- **K-th smallest in BST** — inorder traversal, count to K.
- **LCA in BST** — if both nodes < current, go left. Both > current, go right. Else, current is the LCA.
- **LCA in generic binary tree** — recurse left and right. If both return non-null, current node is LCA. If one returns non-null, propagate that.

**Example problems:** LC 98 (Validate BST), LC 230 (Kth Smallest in BST), LC 235 (LCA of BST), LC 236 (LCA of Binary Tree).

---

### 12.3 — Tree Construction from Traversals

**Core idea:** Preorder's first element is the root. Find that root in inorder — everything left is the left subtree, everything right is the right subtree. Recurse.

**Key trick:** Use a hashmap to get O(1) lookup of root position in inorder array (avoids O(n) search each time).

**Postorder + inorder:** Postorder's LAST element is the root. Otherwise same logic but build right subtree first (process postorder from the end).

**Example problems:** LC 105 (Construct from Preorder + Inorder), LC 106 (Construct from Postorder + Inorder).

---

### 12.4 — Level-Order Traversal (BFS)

**Core idea:** BFS with a queue. Track level boundaries by processing all nodes at the current queue size before moving to the next level.

**Key tricks:**

- **Right side view** — take the LAST element of each level.
- **Zigzag level order** — alternate the direction of reading each level.
- **Average / maximum of levels** — aggregate within each level.

**Example problems:** LC 102 (Level Order), LC 199 (Right Side View), LC 103 (Zigzag Level Order), LC 515 (Find Largest Value in Each Tree Row).

---

### 12.5 — Serialize / Deserialize

**Core idea:** Use preorder traversal, recording null nodes as a sentinel (e.g., "#" or "null"). This uniquely defines the tree.

**Key trick:** During deserialization, use a queue/iterator on the serialized data. Recursively build: read a value, build left subtree, build right subtree. When you hit the null sentinel, return null.

**Example problems:** LC 297 (Serialize and Deserialize Binary Tree), LC 449 (Serialize and Deserialize BST).

---

## 13. Linked Lists

### 13.1 — Dummy Head Node

**Core idea:** Create a dummy node that points to the head. This eliminates edge cases where the head itself might change.

**When to use:** Any problem where you build or modify a linked list and the head might be removed/replaced.

**Example problems:** LC 2 (Add Two Numbers), LC 21 (Merge Two Sorted Lists), LC 24 (Swap Nodes in Pairs).

---

### 13.2 — Fast/Slow Pointer (Floyd's Cycle)

**Core idea:** Slow moves 1 step, fast moves 2 steps.

**Key tricks:**

- **Detect cycle** — if fast catches slow, there's a cycle.
- **Find cycle start** — after detection, move one pointer to head. Now both move 1 step at a time. They meet at the cycle start.
- **Find middle** — when fast reaches the end, slow is at the middle. For even-length lists, slow lands on the first of the two middle nodes (with the standard `while fast and fast.next` loop) or the second (with `while fast.next and fast.next.next`).
- **Find n-th from end** — advance fast by n steps, then move both until fast reaches the end.

**Example problems:** LC 141 (Linked List Cycle), LC 142 (Linked List Cycle II), LC 876 (Middle of Linked List), LC 19 (Remove Nth Node From End), LC 287 (Find the Duplicate Number — treat array as linked list with cycle).

---

### 13.3 — Reverse Linked List

**Core idea (iterative):** Three pointers: `prev`, `curr`, `next`. For each node: save next, point current to prev, advance prev and curr.

**Key tricks:**

- **Reverse in groups of K** — reverse K nodes, then recurse/iterate on the rest. Track the tail of each reversed group to connect them.
- **Palindrome linked list** — find middle, reverse second half, compare with first half, (optionally) restore.

**Example problems:** LC 206 (Reverse Linked List), LC 25 (Reverse Nodes in k-Group), LC 234 (Palindrome Linked List), LC 92 (Reverse Linked List II — reverse a subrange).

---

## 14. Backtracking

### 14.1 — Core Template

```
def backtrack(state, choices):
    if is_goal(state):
        result.append(copy of state)
        return
    for choice in choices:
        if is_valid(choice):
            make(choice)
            backtrack(updated_state, remaining_choices)
            undo(choice)
```

**How to identify:** "Generate all possible," "find all combinations/permutations/subsets."

---

### 14.2 — Handling Duplicates

**Core idea:** Sort the input first. Skip a choice if it's the same as the previous choice AND the previous choice wasn't used in the current path.

**Key rule:** `if i > start and nums[i] == nums[i-1]: continue` (for combinations). For permutations with duplicates: `if i > 0 and nums[i] == nums[i-1] and not used[i-1]: continue`.

**Why this works:** Sorting groups duplicates together. The skip condition ensures you only use duplicate values in order — e.g., if you have two 1s, you always use the first 1 before the second 1.

**Example problems:** LC 47 (Permutations II), LC 40 (Combination Sum II), LC 90 (Subsets II).

---

### 14.3 — Pruning

**Key tricks:**

- **Sum exceeds target** — if running sum > target, skip (for Combination Sum with positive values).
- **Sorted + early exit** — if `nums[i] > remaining target`, break (not just continue), since all subsequent elements are also too large.
- **N-Queens** — track occupied columns, diagonals (r-c), and anti-diagonals (r+c) with sets for O(1) conflict checking.
- **Start index** — pass a `start` parameter to avoid generating duplicate combinations. Each recursive call starts from `start` (or `start + 1` for no-reuse variants).

**Example problems:** LC 39 (Combination Sum), LC 51 (N-Queens), LC 79 (Word Search — DFS on grid with backtracking).

---

## 15. Bit Manipulation

### 15.1 — XOR Tricks

**Core idea:** XOR has these properties: `a ^ a = 0`, `a ^ 0 = a`, XOR is commutative and associative.

**Key tricks:**

- **Single number (all others appear twice)** — XOR everything together. Pairs cancel, leaving the single number.
- **Two single numbers** — XOR all gives `a ^ b`. Find any set bit (using `x & -x` for the lowest), partition all numbers into two groups by that bit, and XOR each group.
- **Missing number in [0, n]** — XOR all numbers with XOR of [0..n].

**Example problems:** LC 136 (Single Number), LC 260 (Single Number III — two unique), LC 137 (Single Number II — every other appears 3 times; use bitwise counting mod 3).

---

### 15.2 — Bit Operations

**Key tricks:**

- `n & (n - 1)` — removes the lowest set bit. Use to count set bits: loop until n = 0, counting iterations.
- `n & (-n)` — isolates the lowest set bit (gives you a mask with only that bit).
- Check i-th bit: `n & (1 << i)`.
- Set i-th bit: `n | (1 << i)`.
- Clear i-th bit: `n & ~(1 << i)`.
- Toggle i-th bit: `n ^ (1 << i)`.
- Power of 2 check: `n > 0 && (n & (n - 1)) == 0`.

**Example problems:** LC 191 (Number of 1 Bits), LC 338 (Counting Bits — `dp[i] = dp[i & (i-1)] + 1`), LC 231 (Power of Two), LC 371 (Sum of Two Integers — add without +, using XOR for sum and AND+shift for carry).

---

### 15.3 — Enumerating Subsets of a Bitmask

**Core idea:** Given a bitmask `mask`, enumerate all its submasks:

```cpp
for (int sub = mask; sub > 0; sub = (sub - 1) & mask) {
    // sub is a subset of mask
}
// don't forget to handle sub = 0 (empty set) separately if needed
```

**How to identify:** Bitmask DP where transitions require trying all subsets of the remaining elements.

---

## 16. Tries

### 16.1 — Standard Trie (Prefix Tree)

**Core idea:** Tree where each edge represents a character. Shared prefixes share paths. Each node stores children and an `is_end` flag.

**How to identify:** "Prefix search," "auto-complete," "word search in a board with dictionary."

**Key trick for Word Search II:** Build a trie from the word list. DFS on the grid, following the trie. This prunes the search — you stop exploring a path as soon as it leaves the trie. Much faster than running separate DFS for each word.

**Example problems:** LC 208 (Implement Trie), LC 211 (Design Add and Search Words), LC 212 (Word Search II).

---

### 16.2 — Binary Trie for Maximum XOR

**Core idea:** Build a trie of all numbers' binary representations (MSB to LSB). To find max XOR with a number, greedily walk the trie choosing the opposite bit at each level.

**How to identify:** "Maximum XOR of two numbers in an array."

**Example problems:** LC 421 (Maximum XOR of Two Numbers in Array).

---

## 17. Math & Number Theory

### 17.1 — Modular Arithmetic

**Key rules:**
- `(a + b) % m = ((a % m) + (b % m)) % m`
- `(a * b) % m = ((a % m) * (b % m)) % m`
- `(a - b) % m = ((a % m) - (b % m) + m) % m` (add m to handle negative results in C++)
- Division requires modular inverse: `a / b mod m = a * b^(-1) mod m`, where `b^(-1) = b^(m-2) mod m` (when m is prime, via Fermat's little theorem).

**Key trick — overflow prevention in C++:** When multiplying two numbers mod 10^9+7, cast to `long long` before multiplying.

---

### 17.2 — Fast Exponentiation

**Core idea:** Compute `a^n mod m` in O(log n) using repeated squaring.

```
result = 1
while n > 0:
    if n is odd: result = result * a % m
    a = a * a % m
    n /= 2
```

**Example problems:** LC 50 (Pow(x, n)), LC 372 (Super Pow).

---

### 17.3 — Sieve of Eratosthenes

**Core idea:** To find all primes up to n, mark multiples of each prime starting from p². O(n log log n).

**Example problems:** LC 204 (Count Primes).

---

### 17.4 — GCD & LCM

`gcd(a, b) = gcd(b, a % b)`, base case `gcd(a, 0) = a`.
`lcm(a, b) = a / gcd(a, b) * b` (divide first to avoid overflow).

**Example problems:** LC 1071 (Greatest Common Divisor of Strings), LC 365 (Water and Jug Problem).

---

## 18. Matrix / Grid

### 18.1 — Rotation = Transpose + Reverse

**Core idea:** Rotate 90° clockwise = transpose the matrix (swap `[i][j]` with `[j][i]`) then reverse each row.

**Other rotations:**
- 90° counter-clockwise = transpose, then reverse each column (or reverse each row then transpose).
- 180° = reverse each row, then reverse each column.

**Example problems:** LC 48 (Rotate Image).

---

### 18.2 — Spiral Order Traversal

**Core idea:** Maintain four boundaries: `top`, `bottom`, `left`, `right`. Traverse right along top row, then down along right column, then left along bottom row, then up along left column. Shrink boundaries after each traversal.

**Example problems:** LC 54 (Spiral Matrix), LC 59 (Spiral Matrix II — fill instead of read).

---

### 18.3 — Island / Flood Fill Problems

**Core idea:** For each unvisited land cell, run DFS/BFS to mark the entire island. Count how many times you start a new traversal.

**Key tricks:**

- **Mark visited in-place** — change '1' to '0' (or a sentinel) to avoid a separate visited array.
- **Area of island** — count cells during DFS.
- **Number of distinct islands** — normalize the island shape (record relative coordinates) and use a set.
- **Making the grid connected** — think in terms of components and required additions.

**Example problems:** LC 200 (Number of Islands), LC 695 (Max Area of Island), LC 733 (Flood Fill), LC 827 (Making a Large Island — tag islands, check borders).

---

### 18.4 — Diagonal Properties

**Core idea:** Cells on the same top-left-to-bottom-right diagonal have the same `row - col`. Cells on the same top-right-to-bottom-left diagonal have the same `row + col`.

**When to use:** N-Queens conflict checking. Or grouping cells by diagonal.

---

### 18.5 — 2D → 1D Reduction

**Core idea:** Fix one dimension (e.g., enumerate all pairs of rows) and reduce the 2D problem to a 1D problem on the other dimension.

**Example:** "Maximum sum rectangle in a 2D matrix" → fix top and bottom rows, compute column sums, apply Kadane's algorithm on the column sums.

**Example problems:** LC 363 (Max Sum of Rectangle No Larger Than K).

---

## 19. Union-Find (Disjoint Set Union)

### 19.1 — Standard Union-Find

**Core idea:** Track connected components. Two operations: `find(x)` (which component is x in?) and `union(x, y)` (merge components).

**Optimizations:**
- **Path compression** in `find`: point every node directly to the root.
- **Union by rank/size**: attach the smaller tree under the larger tree.
With both optimizations, amortized O(α(n)) per operation (nearly constant).

**How to identify:** "Are x and y connected?" "How many connected components?" "Merge groups." Any problem where Union-Find gives cleaner code than DFS.

**Key tricks:**

- **Count components** — start with n components, decrement on each successful union.
- **Weighted Union-Find** — track a weight/offset on each edge. Useful for "ratio" problems.
- **Detect cycle in undirected graph** — if `find(u) == find(v)` before `union(u, v)`, the edge creates a cycle.

**Example problems:** LC 684 (Redundant Connection), LC 721 (Accounts Merge), LC 947 (Most Stones Removed), LC 1319 (Number of Operations to Make Network Connected), LC 399 (Evaluate Division — weighted UF).

---

## 20. Cross-Cutting Meta-Tricks

These are technique-agnostic patterns that transform hard problems into easier ones.

---

### 20.1 — Complement / Inverse Thinking

**Core idea:** If the problem asks for "minimum to remove," compute "maximum to keep" and subtract.

**Examples:**
- "Min deletions to avoid overlap" = total − max non-overlapping (Interval Scheduling).
- "Min operations to reduce X from ends" = total length − "max subarray with sum = total - X" (Sliding Window).
- "Longest subsequence with ≤ K changes" = complement of what you're NOT changing = sliding window.

---

### 20.2 — Circular Array → Double the Array

**Core idea:** For problems on circular arrays, concatenate the array with itself. Then apply linear techniques on the doubled array, restricting window/range to ≤ n.

**Alternative:** Use modular arithmetic with `index % n`.

**Example problems:** LC 503 (Next Greater Element II — circular), LC 918 (Max Sum Circular Subarray — but the complement trick is better here: max circular = total - min subarray).

---

### 20.3 — Virtual / Dummy Nodes

**For linked lists:** Add a dummy head to avoid null-head edge cases.

**For graphs:** Add a virtual source that connects to all real sources (or a virtual sink). Turns multi-source shortest path into single-source.

---

### 20.4 — Coordinate Compression

**Core idea:** When values are huge but only their relative order matters, map values to [0, n). Sort the unique values and replace each value with its sorted index.

**When to use:** Before using a BIT (Fenwick tree) or segment tree — these are indexed by value and can't handle values up to 10^9 directly.

**Example problems:** Count of smaller numbers after self (LC 315), Reverse pairs (LC 493).

---

### 20.5 — Think From the Answer

**Core idea:** Instead of building toward the answer, assume the answer and verify. This is the basis of binary search on answer, but it also applies more broadly.

**Examples:**
- "Is it possible to achieve X?" → assume X, check consistency.
- "Find the minimum time/cost such that Y" → binary search on time/cost, check Y feasibility.

---

### 20.6 — Process Events, Not Elements

**Core idea:** Convert a collection of objects into a sorted list of events and sweep through.

**Examples:**
- Intervals → (start, +1), (end, -1) events for meeting rooms.
- Rectangles → sweep line with events for area/perimeter.
- Points → events at coordinates for closest pair / sweep line algorithms.

---

### 20.7 — "Fix One Dimension, Solve the Other"

**Core idea:** When a problem has multiple axes of freedom, fix one (enumerate all possibilities or sort by it) and solve the remaining dimension with a known technique.

**Examples:**
- 2D array max sum → fix top/bottom rows, Kadane's on columns.
- 3Sum → fix one element, two-pointer on the rest.
- LIS in 2D (envelopes) → sort by width, LIS on height.

---

### 20.8 — Contribution Counting (Repeated)

**Core idea:** Instead of iterating over all O(n²) subarrays/subsets, figure out each element's contribution to the total.

**When to use:** "Sum of [property] over all subarrays" where the property is min, max, XOR, etc.

---

### 20.9 — Offline Processing: Sort Queries

**Core idea:** When online processing of queries is hard, sort the queries by some criterion and process them in an order that makes updates easier.

**Example:** "Count of elements ≤ K in a range" — sort queries by K, process in increasing K, add elements to a BIT.

---

### 20.10 — Monotonicity Check for Technique Selection

Before choosing a technique, ask: "If I add one more element, does the answer only go up / only go down / could go either way?"

- **Monotonic** → sliding window, binary search, two pointers, or greedy are likely.
- **Non-monotonic** → prefix sum + hashmap, DP, or more complex approaches.

This is the most fundamental decision point. Sliding window REQUIRES monotonicity of the window constraint. If adding an element could make a bad window good again, sliding window breaks.

---

### 20.11 — Common Problem Transformations Summary

| Problem says...                        | Actually means...                          |
|-----------------------------------------|---------------------------------------------|
| "Remove from both ends"                | Keep a contiguous middle (sliding window)   |
| "Minimum to remove"                    | Total − maximum to keep                     |
| "Circular array"                       | Double the array or modular arithmetic      |
| "Exactly K"                            | At most K − at most K−1                     |
| "Minimize the maximum"                 | Binary search on the answer                 |
| "Sum of property over all subarrays"   | Contribution of each element                |
| "Shortest path, unweighted"            | BFS                                         |
| "Shortest path, weights 0/1"           | 0-1 BFS with deque                          |
| "Shortest path, non-negative weights"  | Dijkstra                                    |
| "Subarray sum = K" (with negatives)    | Prefix sum + hashmap (NOT sliding window)   |
| "Subarray sum = K" (all positive)      | Sliding window                              |
| "Count ways / optimal cost with choices"| DP                                          |
| "Generate ALL possibilities"           | Backtracking                                |
| "Next greater / smaller"               | Monotonic stack                             |
| "Sliding window min/max"               | Monotonic deque                             |
| "K-th largest/smallest"                | Heap of size K or quickselect               |
| "Median in a stream"                   | Two heaps                                   |
| "Connected / groups / islands"         | DFS / BFS / Union-Find                      |
| "Dependencies / ordering"              | Topological sort                            |
| "Prefix search / dictionary"           | Trie                                        |
| "Non-overlapping interval selection"   | Sort by end, greedy                         |
| "Weighted interval selection"          | Sort by end, DP + binary search             |

---

### 20.12 — Decision Flowchart

When you see a new problem, run through this:

1. **Is it about a contiguous subarray/substring?**
   → Check if the constraint is monotonic (adding elements only makes it worse/better).
   → Monotonic → sliding window. Non-monotonic → prefix sum + hashmap. Max subarray → Kadane's.

2. **Is the array sorted, or should you sort it?**
   → Sorted + pair/triple search → two pointers.
   → Sort + greedy → interval scheduling, activity selection, custom comparators.

3. **Is it asking for shortest path or minimum steps?**
   → Unweighted → BFS. Weighted non-negative → Dijkstra. Negative → Bellman-Ford. Weights 0/1 → 0-1 BFS.

4. **Is it asking for ALL possibilities or combinations?**
   → Backtracking (with pruning).

5. **Is it asking for optimal (max/min) with choices at each step?**
   → Are there overlapping subproblems? → DP.
   → No overlapping subproblems, greedy choice works? → Greedy.

6. **Is the answer itself a number you're optimizing, and feasibility is monotonic?**
   → Binary search on answer.

7. **Does "for each element, find the nearest X" appear?**
   → Monotonic stack or monotonic deque.

8. **Is it about a tree?**
   → Default: recurse on left and right, combine.
   → Need level info → BFS. Need sorted order in BST → inorder.

9. **Is it about connected components, cycles, or graph structure?**
   → Union-Find or DFS/BFS.

10. **Does n ≤ 20?**
    → Bitmask DP might be the intended approach.
