# Linked Lists — Comprehensive Reference

---

## How to Identify

**Primary signals:** The problem gives you a linked list (or asks you to implement one). Linked list problems are almost always about pointer manipulation, not algorithms.

**The three core tricks that solve 90% of linked list problems:**

1. Dummy head node (eliminates edge cases).
2. Fast/slow pointer (cycle detection, finding middle).
3. Iterative reversal (reverse a sublist or the whole list).

---

## Dummy Head Node

Create a sentinel node before the real head: `dummy.next = head`. Build/modify the list off `dummy`. Return `dummy.next` as the new head.

**When to use:** Any time the head might change (deletion, insertion, merging). Eliminates `if head is null` checks.

LC 2 (Add Two Numbers), LC 21 (Merge Two Sorted Lists), LC 24 (Swap Nodes in Pairs), LC 25 (Reverse Nodes in k-Group).

---

## Fast/Slow Pointer (Floyd's Algorithm)

**Detect cycle:** slow moves 1 step, fast moves 2. If they meet → cycle. If fast reaches null → no cycle.

**Find cycle start:** after detection, reset one pointer to head. Both move 1 step. They meet at the cycle start. Math: if the non-cycle portion has length L and the cycle has length C, slow traveled L + K steps when they meet, fast traveled L + K + nC. Since fast = 2×slow: L + K = nC, so L = nC - K. Starting one pointer at head and both moving at speed 1, they meet at the cycle start after L steps.

**Find middle:** when fast reaches the end, slow is at the middle. `while (fast && fast->next)` puts slow at the first middle node for even-length lists.

**N-th from end:** advance fast by N steps, then move both until fast hits null. Slow is at the N-th from end.

**Find duplicate number (array trick):** treat `nums[i]` as a "pointer" from index i to index nums[i]. This creates an implicit linked list. The duplicate causes a cycle. Use Floyd's to find the cycle entrance = the duplicate.

LC 141 (Cycle), LC 142 (Cycle II — find start), LC 876 (Middle of List), LC 19 (Remove Nth From End), LC 287 (Find the Duplicate Number — Floyd's on array).

---

## Reverse Linked List

**Iterative:**

```
prev = null, curr = head
while curr:
    next = curr.next
    curr.next = prev
    prev = curr
    curr = next
return prev
```

**Reverse a subrange [left, right]:** navigate to position `left-1` (the node before the reversal starts). Reverse the sublist. Reconnect the ends.

**Reverse in groups of K:** reverse the first K nodes. Recursively process the rest. Connect the tail of the reversed group to the result of the recursive call. Need to check if there are ≥ K nodes remaining before reversing.

**Palindrome linked list:** find the middle (fast/slow), reverse the second half, compare with first half. Optionally restore the list.

LC 206 (Reverse), LC 92 (Reverse Linked List II — subrange), LC 25 (Reverse in k-Group), LC 234 (Palindrome Linked List).

---

## Other Patterns

**Merge two sorted lists:** two pointers, compare heads, append smaller. Use a dummy head. LC 21 (Merge Two Sorted Lists).

**Intersection of two lists:** compute lengths. Advance the longer list by the difference. Walk together until they meet. LC 160 (Intersection of Two Linked Lists).

**Copy list with random pointer:** first pass: create new nodes interleaved with originals (A→A'→B→B'). Second pass: set random pointers (A'.random = A.random.next). Third pass: separate the two lists. Or use a hashmap {original → copy}. LC 138 (Copy List with Random Pointer).

**LRU Cache:** hashmap + doubly linked list. Map: key → node. List maintains recency order. On access: move to front. On eviction: remove from back. LC 146 (LRU Cache).