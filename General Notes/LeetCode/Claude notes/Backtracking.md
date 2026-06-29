# Backtracking — Comprehensive Reference

---

## How to Identify

**Primary signals:**

- "Generate ALL possible" combinations, permutations, subsets, partitions.
- "Find ALL valid" configurations (N-Queens, Sudoku).
- The constraint space is small enough for brute-force with pruning.
- The problem asks to LIST results, not just count them (counting is usually DP).

**Backtracking vs. DP:**

- Backtracking: enumerate and collect all valid solutions. Exponential but necessary when you need the solutions themselves.
- DP: count solutions or find the optimal. Polynomial (usually) because it avoids re-enumerating.

**When backtracking is NOT appropriate:**

- You only need the COUNT → DP.
- You only need the OPTIMAL → DP or greedy.
- The search space is too large (n > 20 for subsets, n > 10 for permutations) and pruning doesn't help enough.

---

## Core Template

```
function backtrack(state, start_or_choices):
    if is_complete(state):
        result.add(copy_of(state))
        return
    for each choice from start_or_choices:
        if is_valid(choice, state):
            apply(choice, state)
            backtrack(updated_state, updated_choices)
            undo(choice, state)    // backtrack
```

**The three key decisions for any backtracking problem:**

1. **What is a "choice"?** — include/exclude an element, place a queen, assign a value.
2. **When is the state "complete"?** — path length = target, all positions filled, sum = target.
3. **What makes a choice "invalid"?** — sum exceeds target, column conflict, duplicate.

---

## Subsets

### All Subsets (Power Set)

For each element: include it or don't. Use a `start` index to avoid duplicates.

```
backtrack(start):
    result.add(copy of current)    // every partial state is a valid subset
    for i = start to n-1:
        current.push(nums[i])
        backtrack(i + 1)
        current.pop()
```

LC 78 (Subsets).

### Subsets with Duplicates

Sort the input. Skip duplicates: `if i > start and nums[i] == nums[i-1]: continue`.

**Why this works:** Sorting groups duplicates together. The skip condition ensures that among identical elements, you only take them in order (first before second). This prevents generating the same subset via different orderings of identical elements.

LC 90 (Subsets II).

---

## Combinations

### Combination Sum (reuse allowed)

Each element can be used unlimited times. At each step, you can re-choose the same element (pass `i` instead of `i+1`).

```
backtrack(start, remaining_target):
    if remaining_target == 0: record solution
    for i = start to n-1:
        if nums[i] > remaining_target: break    // pruning (requires sorted input)
        current.push(nums[i])
        backtrack(i, remaining_target - nums[i])  // i, not i+1: allow reuse
        current.pop()
```

LC 39 (Combination Sum).

### Combination Sum (no reuse, with duplicates)

Each element used at most once. Sort input. Skip: `if i > start and nums[i] == nums[i-1]: continue`.

LC 40 (Combination Sum II).

### Combinations of Size K

Stop when current.size() == K.

LC 77 (Combinations).

---

## Permutations

### Without Duplicates

Use a `used[]` boolean array. At each position, try all unused elements.

```
backtrack():
    if current.size() == n: record
    for i = 0 to n-1:
        if not used[i]:
            used[i] = true
            current.push(nums[i])
            backtrack()
            current.pop()
            used[i] = false
```

LC 46 (Permutations).

### With Duplicates

Sort input. Skip: `if i > 0 and nums[i] == nums[i-1] and not used[i-1]: continue`.

**Intuition:** Among identical elements, always use the earlier one first. If `used[i-1]` is false and `nums[i] == nums[i-1]`, you're trying to use a later copy before an earlier copy — skip.

LC 47 (Permutations II).

### Swap-Based Permutation

Alternative approach: generate permutations by swapping. For position `start`, swap `nums[start]` with each `nums[i]` for `i >= start`, recurse on `start+1`, swap back. Avoids the `used[]` array but handling duplicates is trickier — use a local set to track what values have been placed at position `start`.

---

## Pruning Techniques

**1. Sort + early exit:** Sort the input. If the current element already exceeds the remaining budget, `break` (not just `continue`), because all subsequent elements are larger.

**2. Constraint propagation:** For N-Queens, track occupied COLUMNS, DIAGONALS (r-c), and ANTI-DIAGONALS (r+c) with sets. O(1) conflict checking.

**3. Bound checking:** If the remaining elements can't possibly reach the target (even if all chosen), prune.

**4. Symmetry breaking:** For problems with symmetric solutions, fix part of the solution to avoid counting duplicates. For N-Queens, you can fix the first queen to the left half of the first row.

---

## Key Backtracking Problems

**N-Queens:** Place queens row by row. At each row, try each column. Check column, diagonal (r-c), and anti-diagonal (r+c) conflicts with sets. LC 51 (N-Queens), LC 52 (N-Queens II — just count).

**Word Search (Grid):** DFS from each cell matching the first character. Mark cells as visited during the search, unmark on backtrack. Optimization: check character frequency before starting. LC 79 (Word Search).

**Palindrome Partitioning:** For each position, try all possible first palindromes, then recurse on the remainder. LC 131 (Palindrome Partitioning).

**Letter Combinations of Phone Number:** Map each digit to its letters. Backtrack through digits, trying each letter. LC 17 (Letter Combinations of a Phone Number).

**Generate Parentheses:** Track open/close counts. Add '(' if open < n. Add ')' if close < open. LC 22 (Generate Parentheses).

**Sudoku Solver:** For each empty cell, try digits 1-9. Check row, column, and 3×3 box constraints. LC 37 (Sudoku Solver).