# Two Pointers & Sliding Window — Comprehensive Reference

---

## How to Identify

**Primary signals:**

- "Longest/shortest contiguous subarray/substring satisfying X."
- "Count subarrays/substrings with property X."
- Array is sorted and you need to find pairs/triplets.
- "Remove/rearrange elements in place."

**Critical prerequisite for sliding window — the monotonicity check:** Ask: "If my current window is VALID and I ADD one more element (expand right), can it only stay valid or become invalid? And if it's INVALID and I REMOVE from the left, can it only stay invalid or become valid?"

If BOTH are true, the constraint is monotonic and sliding window works. If not, sliding window BREAKS.

**When sliding window does NOT work:**

- **Negative numbers in sum problems** — adding an element could decrease the sum, and removing from the left could increase it. Example: "subarray sum equals K" with negative numbers → use prefix sum + hashmap instead.
- **Non-monotonic constraints** — "subarray where max - min ≤ K" is tricky (you can use a deque-based approach, but not a simple counter).
- **Subsequences (non-contiguous)** — sliding window requires contiguity.

---

## Two Pointers

### Opposite-Direction (Converging)

One pointer at start, one at end, moving toward each other. Works on SORTED arrays.

**Two Sum (sorted):** If sum < target, move left pointer right (need a bigger number). If sum > target, move right pointer left. O(n). LC 167 (Two Sum II).

**Container With Most Water:** Move the SHORTER side inward. Why? The area is limited by the shorter side. Moving the taller side can only decrease width while height stays bottlenecked by the (unchanged) shorter side — so area can only decrease. Moving the shorter side might find a taller line, which could increase the bottleneck. LC 11 (Container With Most Water).

**Trapping Rain Water (two-pointer approach):** Maintain `left_max` and `right_max`. Process the side with the SMALLER max first. Water at that side = `smaller_max - height[that_side]`. Why the smaller side? Water is bounded by the smaller of the two maxes, so the smaller side's water level is fully determined. LC 42 (Trapping Rain Water).

**Valid Palindrome:** Left and right pointers move inward, comparing characters. Skip non-alphanumeric. LC 125 (Valid Palindrome).

### Same-Direction (Fast/Slow)

Both pointers move left to right. Used for in-place rearrangement.

**Remove/Move pattern:** Slow pointer = write position. Fast pointer = read position. When fast finds a "keeper," write it at slow and advance slow. LC 283 (Move Zeroes), LC 26 (Remove Duplicates from Sorted Array), LC 27 (Remove Element).

### Three Pointers / K-Sum

Fix one element with outer loop, apply two pointers on the rest.

**3Sum:** Sort. For each i, two-pointer on [i+1, n-1]. Skip duplicates: after finding a valid triple, advance both pointers past identical values. Also skip if `nums[i] == nums[i-1]` in the outer loop. LC 15 (3Sum), LC 16 (3Sum Closest), LC 18 (4Sum — nest one more loop).

### Dutch National Flag (3-Way Partition)

Three pointers: `low`, `mid`, `high`. Partition into three groups in one pass.

- `nums[mid] == 0`: swap with low, advance both.
- `nums[mid] == 1`: advance mid only.
- `nums[mid] == 2`: swap with high, decrement high only (swapped value is unexamined). LC 75 (Sort Colors).

---

## Sliding Window

### Fixed-Size Window

Window of exactly size K. Slide: add right, remove left.

**How to identify:** "Subarray/substring of exactly size K" + some aggregate.

**Template:**

```
for right = 0 to n-1:
    add nums[right] to window state
    if right >= K:
        remove nums[right - K] from window state
    if right >= K - 1:
        update answer from window state
```

LC 643 (Maximum Average Subarray I), LC 1456 (Max Vowels in Substring of Length K), LC 567 (Permutation in String — fixed window of size len(s1), check frequency match).

### Variable-Size Window (Longest)

Expand right always. Shrink left WHILE window is invalid. The valid window after shrinking is a candidate for the answer.

**Template:**

```
left = 0
for right = 0 to n-1:
    add nums[right] to window state
    while window is INVALID:
        remove nums[left] from window state
        left++
    update answer = max(answer, right - left + 1)
```

LC 3 (Longest Substring Without Repeating Characters), LC 424 (Longest Repeating Character Replacement), LC 1004 (Max Consecutive Ones III — at most K flips), LC 1695 (Maximum Erasure Value — longest subarray with all unique, track sum).

### Variable-Size Window (Shortest)

Expand right until valid. Then shrink left WHILE STILL VALID, updating answer at each valid state.

**Template:**

```
left = 0
for right = 0 to n-1:
    add nums[right] to window state
    while window is VALID:
        update answer = min(answer, right - left + 1)
        remove nums[left] from window state
        left++
```

LC 209 (Minimum Size Subarray Sum), LC 76 (Minimum Window Substring).

### "Exactly K" = "At Most K" − "At Most K−1"

Counting subarrays with EXACTLY K of something is hard directly. Write a helper `atMost(K)` and compute `atMost(K) - atMost(K-1)`.

In `atMost(K)`: standard variable window. At each right pointer position, the number of valid subarrays ending at right is `right - left + 1` (every subarray from any start in [left, right] is valid).

LC 992 (Subarrays with K Different Integers), LC 1248 (Count Number of Nice Subarrays — exactly K odds), LC 930 (Binary Subarrays With Sum).

### "Remove from Both Ends" = Sliding Window on the Middle

If the problem says "remove from the front and/or back to achieve X," the complement is "find a contiguous subarray (what you KEEP) that achieves the inverse of X."

Example: "Remove elements from both ends so that removed sum = X" → "Find longest subarray with sum = total - X."

LC 1658 (Minimum Operations to Reduce X to Zero), LC 2516 (Take K of Each Character From Left and Right).

### Sliding Window with Frequency Map

Track element/character frequencies in the window with a hashmap. Maintain a counter for "how many unique chars meet the target frequency."

**Minimum Window Substring pattern:**

- `required` = number of unique chars in target.
- `formed` = number of unique chars in window meeting target frequency.
- Expand right. When a char's window count hits target count, increment `formed`.
- When `formed == required`, shrink left while maintaining validity.
- When a char's window count drops below target count, decrement `formed`.

LC 76 (Minimum Window Substring), LC 438 (Find All Anagrams — fixed window with freq match), LC 30 (Substring with Concatenation of All Words).

### Sliding Window for "Max − Min ≤ K" (Using Deques)

Maintain two monotonic deques: one for sliding max (decreasing deque), one for sliding min (increasing deque). Expand right. When `max - min > K`, shrink left, removing from deque fronts as they fall out of window.

LC 1438 (Longest Continuous Subarray With Absolute Diff ≤ Limit).