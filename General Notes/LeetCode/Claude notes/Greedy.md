# Greedy — Comprehensive Reference

---

## How to Identify Greedy Problems

**Primary signals:**

- You make a sequence of choices, and each choice is final (never reconsidered).
- There's a "locally best" option at each step that feels intuitively right.
- The problem has optimal substructure but NOT overlapping subproblems (or the overlap is trivial).
- Sorting the input "unlocks" a natural processing order.

**Key question to ask yourself:** "If I make the best local choice right now, can that EVER hurt me later?" If no → greedy. If yes → DP.

**Greedy vs. DP:**

|Greedy|DP|
|---|---|
|Make one choice at each step, move on|Consider all choices at each step, pick best|
|O(n log n) or O(n) typically|O(n²) or O(n × states) typically|
|Proof required (exchange argument)|Proof is structural (optimal substructure + overlap)|
|Fails: coin change [1,3,4] amount=6 (greedy: 4+1+1=3 coins; optimal: 3+3=2)|Works for arbitrary coin change|
|Fails: 0/1 knapsack|Works for 0/1 knapsack|

**When greedy FAILS (and you need DP instead):**

- **Coin change with non-standard denominations** — greedy (take largest coin) gives 4+1+1 for [1,3,4] amount 6, but 3+3 is optimal.
- **0/1 Knapsack** — greedy by value/weight ratio fails because you can't take fractions.
- **Longest increasing subsequence** — greedy "always extend with the smallest possible" is actually the patience sort technique, but the naive "always take the next increasing" fails.
- **Any problem where taking a "good" item now blocks a "better" combination later** — this is the hallmark of needing DP.

---

## How to PROVE Greedy is Correct

This is the section you specifically asked about. There are three standard proof techniques.

### Proof Technique 1: Exchange Argument

**The idea:** Take ANY optimal solution. Show that you can "swap" one of its choices for the greedy choice without making the solution worse. By repeating this swap, you transform any optimal solution into the greedy solution — therefore the greedy solution is optimal.

**Template:**

1. Let OPT be an optimal solution that differs from the greedy solution GREEDY.
2. Find the FIRST point where they differ. OPT makes choice X; GREEDY makes choice G.
3. Show that replacing X with G in OPT produces a solution OPT' that is at least as good as OPT.
4. OPT' now agrees with GREEDY on one more step. Repeat until OPT = GREEDY.

**Concrete example — Interval Scheduling (sort by end time, pick earliest-ending non-conflicting):**

1. Let OPT be an optimal set of non-overlapping intervals. Let GREEDY be the greedy solution.
2. Suppose the first interval in OPT is X, and the first greedy choice is G (earliest end time).
3. Since G ends no later than X (G has the earliest end time of all), replacing X with G in OPT cannot create any new conflicts — G ends earlier, so it has MORE room for the next interval, not less.
4. OPT' = OPT with X replaced by G has the same number of intervals and is still valid.
5. Repeat for each subsequent interval. GREEDY is at least as good as OPT.

**When to use:** Most greedy problems. The exchange argument is the "default" proof technique.

---

### Proof Technique 2: Stays-Ahead (Greedy Stays Ahead)

**The idea:** Show that after each step k, the greedy solution is "at least as good" as any other solution by some measure. Formally, prove by induction that greedy's partial solution dominates at step k.

**Template:**

1. Define a measure of "progress" after k steps (e.g., earliest end time after k intervals, farthest reachable position after k jumps).
2. **Base case:** after step 1, greedy is at least as good as any other solution.
3. **Inductive step:** assume greedy is ahead after k steps. Show that after step k+1, it's still ahead.
4. **Conclusion:** greedy is ahead after every step, so its final answer is at least as good.

**Concrete example — Activity Selection:** After selecting k activities:

- Let greedy's k-th activity end at time `g_k`.
- Let any other valid selection's k-th activity end at time `o_k`.
- **Claim:** `g_k ≤ o_k` for all k (greedy always finishes earlier).
- **Base:** g_1 ≤ o_1 because greedy picks the earliest-ending activity.
- **Inductive:** if `g_k ≤ o_k`, then greedy has at least as much room for the (k+1)-th activity. Greedy picks the earliest-ending activity that starts after `g_k`. Since `g_k ≤ o_k`, any activity available to the other solution is also available to greedy. So `g_{k+1} ≤ o_{k+1}`.
- **Conclusion:** greedy always finishes earlier or at the same time, so it can fit at least as many activities.

**When to use:** When there's a natural "progress" metric that monotonically grows.

---

### Proof Technique 3: Structural Argument (Cut-and-Paste)

**The idea:** Show that any optimal solution MUST have the structure that greedy produces, or can be rearranged to have that structure without losing optimality.

**Template:**

1. Assume OPT doesn't follow the greedy structure.
2. Show this leads to a contradiction — either OPT isn't actually valid, or you can rearrange it to follow the greedy structure with equal or better cost.

**Concrete example — Huffman Coding:** If two least-frequent symbols aren't at the deepest level, you can swap them with deeper symbols, reducing total cost (less frequent × longer code becomes less frequent × shorter code). Contradiction with optimality. Therefore, OPT must pair the two least-frequent symbols at the deepest level — exactly what Huffman does.

**When to use:** When the greedy choice imposes a structural constraint (e.g., the two smallest must be at the bottom, the latest deadline must be last).

---

### How to Check Your Greedy Intuition (Practical)

If you can't formally prove it in a contest, do this:

1. **Construct adversarial test cases.** Try inputs that might break greedy: all same values, sorted ascending, sorted descending, values that "almost" trigger a counterexample.
2. **Compare with brute force.** For small n (≤ 15), enumerate all possibilities and compare with greedy. If they agree on 100+ random cases, greedy is very likely correct.
3. **Ask: "Can taking a good-looking choice now ever block a strictly better choice later?"** If yes, greedy fails.

---

## Greedy Categories

### Category 1: Sorting-Based Greedy

Sort the input, then process in order. The sort criterion IS the greedy insight.

**1a — Sort by End Time (Interval Scheduling Maximization)** To maximize the number of non-overlapping intervals, sort by end time. Greedily pick the earliest-ending interval that doesn't conflict with the last picked. "Minimum intervals to remove" = total − max non-overlapping. LC 435 (Non-overlapping Intervals), LC 452 (Minimum Number of Arrows).

**1b — Sort by Start Time (Interval Covering)** To cover a target range [0, T] with minimum intervals: sort by start time. Among all intervals that start ≤ current position, pick the one that extends farthest. LC 1024 (Video Stitching), LC 45 (Jump Game II — conceptually similar).

**1c — Sort by Deadline (Scheduling with Deadlines)** For tasks with deadlines and penalties, sort by deadline. Use a heap to drop the least valuable task when you're overbooked. LC 1383 (Maximum Performance of a Team — sort by one metric, heap for other).

**1d — Sort by a Derived Key** Sometimes the correct sort key isn't obvious — it's a COMPUTED value.

- **Two City Scheduling:** sort by `cost_A[i] - cost_B[i]`. Send the first n/2 to city A (those who benefit most from A over B), rest to B. Proof (exchange): if person i goes to B but `cost_A[i] - cost_B[i] < cost_A[j] - cost_B[j]` and person j goes to A, swapping them reduces total cost. LC 1029 (Two City Scheduling).
    
- **Largest Number:** sort by comparing concatenations: compare `a+b` vs `b+a` as strings. The sort defines the optimal ordering. LC 179 (Largest Number).
    
- **Queue Reconstruction by Height:** sort by height DESCENDING (ties broken by index ascending). Then insert each person at their index position. Taller people are placed first, so inserting a shorter person at index k correctly counts taller-or-equal people before them. LC 406 (Queue Reconstruction by Height).
    

**1e — Assign Smallest Available (Matching)** Sort both "demands" and "supplies." Greedily match the smallest demand with the smallest sufficient supply. LC 455 (Assign Cookies — sort children and cookies, two pointers).

---

### Category 2: Priority Queue Greedy

Always process the "most urgent" or "most valuable" item first, using a heap.

**2a — Merge Costs (Huffman-style)** Repeatedly merge the two smallest/cheapest items. Use a min-heap. LC 1167 (Minimum Cost to Connect Sticks).

**2b — Greedy with Cooldown (Reorganize)** Use a max-heap. Always place the most frequent character. After placing it, hold it out for one round (to prevent adjacency), then push it back. LC 767 (Reorganize String), LC 358 (Rearrange String k Distance Apart), LC 621 (Task Scheduler).

**2c — Process by Importance, Track with Heap** Sort by one dimension, use a heap for the other. Common pattern: sort by some deadline/requirement, use a heap to maintain the best subset seen so far. LC 1383 (Maximum Performance of a Team — sort by efficiency desc, heap for speed), LC 502 (IPO — sort projects by capital requirement, heap for profit).

---

### Category 3: Tracker / Frontier Greedy

Maintain a running tracker (farthest reach, current end, running surplus) and update it as you scan.

**3a — Jump Game (Farthest Reachable)** Maintain `farthest = max(farthest, i + nums[i])`. If `i > farthest`, stuck. LC 55 (Jump Game).

**3b — Jump Game II (Minimum Jumps)** Track `current_end` (farthest with current jump count) and `farthest` (overall farthest). When `i` hits `current_end`, you must jump: `jumps++`, `current_end = farthest`. LC 45 (Jump Game II).

**3c — Gas Station (Running Surplus)** If `total_gas ≥ total_cost`, a solution exists. Track running surplus. Whenever it goes negative, reset start to the next station. Why it works: if you can't reach station j from i, you can't reach j from any station between i and j either (you'd have even less gas at those intermediate stations). LC 134 (Gas Station).

**3d — Candy (Two-Pass)** First pass left-to-right: if rating[i] > rating[i-1], candy[i] = candy[i-1] + 1. Second pass right-to-left: if rating[i] > rating[i+1], candy[i] = max(candy[i], candy[i+1] + 1). Each pass handles one direction of the constraint. LC 135 (Candy).

---

### Category 4: Exchange-Argument Greedy

The greedy choice is justified by showing that any deviation can be "fixed" by swapping back, as described in the proof section above.

**4a — Boats to Save People** Sort weights. Pair the lightest with the heaviest. If they fit in one boat, great. If not, the heaviest goes alone (they can never pair with anyone heavier). Two pointers: left and right. Proof: if the heaviest can't pair with the lightest, they can't pair with anyone, so they go alone. If they CAN pair, pairing them is at least as good as any other pairing (exchange: if heavy pairs with someone heavier than light, that's worse because the light person is wasted). LC 881 (Boats to Save People).

**4b — Minimum Number of Platforms (Event Processing)** Convert arrivals/departures to events. Sort. Sweep through, tracking active count. Max active count = answer. LC 253 (Meeting Rooms II).

**4c — Task Scheduler with Cooldown** Mathematical: the answer is max(total_tasks, (max_freq - 1) × (n + 1) + count_of_max_freq). The bottleneck is the most frequent task. Proof: you can always fill idle slots with other tasks. LC 621 (Task Scheduler).

---

### Category 5: Greedy on Sequences

**5a — Remove K Digits (Monotonic Stack Greedy)** To make the smallest number, greedily remove digits that are larger than the digit after them (a "descent" makes the number smaller). Use a stack; pop while top > current and removals remain. This IS greedy — the exchange argument says: removing a large digit before a small digit is always better than removing the small digit. LC 402 (Remove K Digits).

**5b — Subsequence Greedy** When selecting a subsequence with constraints (e.g., longest increasing), greedy choices sometimes work: always extend with the smallest possible tail (patience sorting for LIS).

**5c — Parentheses Greedy** For "minimum additions to make parentheses valid": scan left to right, track balance. When balance goes negative, you need an insertion (and reset balance). At the end, remaining positive balance = unmatched opens. LC 921 (Minimum Add to Make Parentheses Valid).

---

### Category 6: Greedy with Algebraic Insight

Sometimes the greedy criterion comes from algebraic manipulation.

**6a — Minimum Moves to Equal Array Elements** "Increment n-1 elements by 1" is equivalent to "decrement 1 element by 1." So the answer is `sum - n * min`. LC 453 (Minimum Moves to Equal Array Elements).

**6b — Advantage Shuffle (Greedy Matching)** Sort both arrays. For each element of the opponent (largest first), if your largest beats it, pair them. If not, sacrifice your smallest. This is "Tian Ji's horse racing." LC 870 (Advantage Shuffle).

---

## How to Verify Greedy in Practice

1. **Brute force comparison** for small inputs (n ≤ 12). Generate random inputs, compare greedy output with brute-force optimal.
    
2. **Look for the exchange argument.** Can you always swap a non-greedy choice for the greedy choice without hurting the solution? If you can articulate the swap in one sentence, greedy is almost certainly correct.
    
3. **Identify the greedy invariant.** What property does the greedy solution maintain after each step? (e.g., "the last selected interval ends as early as possible"). Check that this invariant is preserved at every step.
    
4. **Try to break it.** Construct adversarial inputs:
    
    - All elements equal.
    - Sorted ascending, sorted descending.
    - Values specifically designed so that the "obvious" local choice backfires.
    - If you CAN break it, you need DP.
5. **Check if the problem is a KNOWN greedy pattern** (intervals, scheduling, matching, two-pointer pairing). Known patterns have known proofs.