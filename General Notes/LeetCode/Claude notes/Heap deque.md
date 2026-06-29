# Heap / Priority Queue & Monotonic Deque — Comprehensive Reference

---

## How to Identify Heap Problems

**Primary signals:**

- "K-th largest/smallest."
- "Top K," "K most frequent."
- "Merge K sorted things."
- "Median of a stream."
- "Always process the most/least [something] next" (greedy + heap).
- Dijkstra's algorithm (min-heap for shortest path).

**Heap vs. sorting:**

- If you need the top K from a static collection, sorting works. Heap is useful when the collection CHANGES (stream, dynamic insertions).
- If you need repeated "give me the min/max," heap gives O(log n) per operation vs. O(n) for rescanning.

**Heap vs. monotonic deque:**

- Heap can't efficiently remove arbitrary elements (only the top). For sliding window min/max, use a MONOTONIC DEQUE — it supports removal from both ends in O(1).
- If you need min/max of a sliding window → deque. If you need the global min/max across all elements → heap.

---

## Heap Patterns

### Top-K Pattern

Use a MIN-HEAP of size K to track the K largest elements. The top of the min-heap is the K-th largest.

**Why min-heap and not max-heap?** You want to discard the smallest among your K candidates. A min-heap surfaces the smallest, making it easy to replace when you find something bigger.

**Template:**

```
for each element:
    if heap.size() < K:
        heap.push(element)
    elif element > heap.top():
        heap.pop()
        heap.push(element)
// heap.top() is the K-th largest
```

**Alternative — Quickselect:** O(n) average for K-th element via partition. But O(n²) worst case without randomized pivot.

**Alternative — Bucket Sort:** For "Top K frequent elements," count frequencies with a hashmap, then use a bucket array indexed by frequency. O(n).

LC 215 (Kth Largest Element), LC 347 (Top K Frequent Elements), LC 973 (K Closest Points to Origin), LC 692 (Top K Frequent Words).

### Two Heaps (Median Maintenance)

Split elements into two halves:

- **Max-heap** for the smaller half (surfaces the largest of the smalls).
- **Min-heap** for the larger half (surfaces the smallest of the larges).
- Median = top of max-heap (if odd count) or average of both tops.

**Balancing rule:** max-heap can have at most 1 more element than min-heap. After each insertion, rebalance by moving tops between heaps.

**Insertion flow:**

1. Push to max-heap.
2. Move max-heap's top to min-heap (ensures max-heap's elements ≤ min-heap's elements).
3. If min-heap is larger than max-heap, move min-heap's top back to max-heap.

LC 295 (Find Median from Data Stream).

### Sliding Window Median

Two heaps + LAZY DELETION. When an element leaves the window, don't remove it from the heap immediately — record it in a "to delete" hashmap. When a heap's top is in the deletion set, pop and discard. This is O(n log n) overall.

LC 480 (Sliding Window Median).

### Merge K Sorted Lists/Streams

Push the head of each sorted list into a min-heap. Pop the smallest, output it, push its successor from the same list.

LC 23 (Merge K Sorted Lists), LC 378 (Kth Smallest in Sorted Matrix — rows as sorted lists).

### Smallest Range Covering Elements from K Lists

Like merge K sorted lists but track the current maximum element alongside the min-heap minimum. The range = [heap.top(), current_max]. Shrink by popping the min and advancing its list.

LC 632 (Smallest Range Covering Elements from K Lists).

### Greedy + Heap Pattern

Sort by one dimension. Use a heap to maintain the best subset by another dimension. Common in scheduling/optimization.

**Pattern:** Sort by criterion A. Scan in order. Use a heap to track criterion B. When you exceed a constraint, pop the worst B from the heap.

LC 1383 (Maximum Performance of a Team — sort by efficiency desc, heap for speeds), LC 502 (IPO — sort projects by capital, max-heap for profit), LC 857 (Minimum Cost to Hire K Workers — sort by ratio, heap for quality).

### Reorganize with Heap + Cooldown

Max-heap of (frequency, char). Pop the most frequent, append to result, hold it out for one round. Push the previously held-out character back. If the heap is empty and the held-out character still has remaining count, it's impossible.

LC 767 (Reorganize String), LC 358 (Rearrange String K Distance Apart — hold out for K rounds), LC 621 (Task Scheduler — can also be solved with math formula).

---

## Monotonic Deque

### Sliding Window Maximum / Minimum

Maintain a deque of INDICES with values in decreasing order (for max) or increasing order (for min).

**For sliding window max (decreasing deque):**

```
for i = 0 to n-1:
    // Remove from BACK: elements smaller than nums[i] can never be the max
    while deque not empty and nums[deque.back()] <= nums[i]:
        deque.pop_back()
    deque.push_back(i)
    // Remove from FRONT: elements outside the window
    while deque.front() <= i - K:
        deque.pop_front()
    // deque.front() is the index of the window max
    if i >= K - 1:
        result.push(nums[deque.front()])
```

**Why deque and not heap?** A heap can't remove elements that fall out of the window (only the top). A deque lets you remove from both ends. Each element enters and exits the deque exactly once → O(n) amortized.

LC 239 (Sliding Window Maximum).

### DP Optimization with Monotonic Deque

When DP transition is `dp[i] = max/min(dp[j]) + cost[i]` for `j in [i-K, i-1]`, maintaining a deque over dp values gives O(n) total.

**Template:**

```
deque stores indices j with dp[j] in decreasing order (for max)
for i = 0 to n-1:
    // Remove expired indices from front
    while deque.front() < i - K:
        deque.pop_front()
    // dp[i] uses the best in window
    dp[i] = dp[deque.front()] + cost[i]
    // Maintain decreasing order from back
    while deque not empty and dp[deque.back()] <= dp[i]:
        deque.pop_back()
    deque.push_back(i)
```

LC 1425 (Constrained Subsequence Sum), LC 862 (Shortest Subarray with Sum ≥ K — on prefix sums, increasing deque).

### 0-1 BFS with Deque

When graph edges have weights 0 or 1: use a deque. Push 0-weight neighbors to the FRONT, 1-weight neighbors to the BACK. This maintains sorted order without a heap.

LC 1368 (Minimum Cost to Make at Least One Valid Path), LC 2290 (Minimum Obstacle Removal).