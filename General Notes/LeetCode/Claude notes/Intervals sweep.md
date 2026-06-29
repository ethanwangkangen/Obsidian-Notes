# Intervals & Line Sweep — Comprehensive Reference

---

## How to Identify

**Primary signals:**

- Input is a list of intervals [start, end].
- "Merge overlapping," "find conflicts," "minimum rooms/resources."
- "Maximum number of non-overlapping intervals."
- "Cover a range with minimum intervals."

**Sort criterion is everything:** The choice between sorting by START vs. END determines the algorithm.

- Sort by START: merging intervals, inserting intervals.
- Sort by END: greedy interval scheduling (maximizing non-overlapping).
- Convert to EVENTS: line sweep for "how many overlap at any point."

---

## Merge Overlapping Intervals

Sort by start time. Maintain a "current" interval. If the next interval's start ≤ current end, merge (extend end). Else, close current and start a new one.

LC 56 (Merge Intervals).

**Insert Interval:** add the new interval to the list, then merge. Or: find the insertion point, merge with overlapping neighbors. LC 57 (Insert Interval).

---

## Line Sweep (Event-Based Processing)

Convert intervals into events: `(time, +1)` for start, `(time, -1)` for end. Sort events. Sweep through, maintaining a running count. The max count = maximum simultaneous overlap.

**Event ordering at the same timestamp:** Depends on problem semantics.

- Meeting rooms: process ENDS before STARTS at the same time (a room freed at time T is available for a meeting starting at T). Sort by `(time, type)` where end = -1 comes before start = +1.
- Point containment: process STARTS before ENDS if the interval is inclusive at the endpoint.

LC 253 (Meeting Rooms II), LC 1094 (Car Pooling), LC 731 (My Calendar II), LC 732 (My Calendar III).

**When line sweep is overkill:** If you just need "do any two intervals overlap?" sort by start and check consecutive pairs. O(n log n). LC 252 (Meeting Rooms — just check for any overlap).

---

## Greedy Interval Scheduling (Sort by End)

To select the MAXIMUM number of non-overlapping intervals: sort by end time. Greedily pick the earliest-ending interval that doesn't conflict with the last picked.

"Minimum intervals to REMOVE" = total − max non-overlapping.

LC 435 (Non-overlapping Intervals), LC 452 (Minimum Number of Arrows — treat touching intervals as overlapping).

---

## Greedy Interval Covering (Sort by Start)

To cover a target range [0, T] with minimum intervals: sort by start time. Among all intervals whose start ≤ current position, pick the one that extends FARTHEST. Advance current position to that interval's end.

LC 1024 (Video Stitching), LC 45 (Jump Game II — conceptually intervals of reachable range).

---

## Weighted Interval Scheduling (DP + Binary Search)

When each interval has a VALUE and you want to maximize total value of non-overlapping selections.

1. Sort by end time.
2. For each interval i, binary search for the last interval that ends before i starts → `last[i]`.
3. `dp[i] = max(dp[i-1], value[i] + dp[last[i]])`.

The binary search is the key insight that makes this O(n log n) instead of O(n²).

LC 1235 (Maximum Profit in Job Scheduling), LC 1751 (Maximum Number of Events That Can Be Attended II).

---

## Interval Intersection / Difference

Two-pointer through two sorted interval lists. Compare endpoints to determine overlap.

LC 986 (Interval List Intersections — two pointers, advance the one that ends first).