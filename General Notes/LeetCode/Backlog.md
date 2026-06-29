Dynamic programming
- https://leetcode.com/problems/maximum-number-of-items-from-sale-i/description/
	- Unbounded and bounded knapsack together, how to do it properly
- https://leetcode.com/problems/check-if-there-is-a-valid-parentheses-string-path/description/
	- How to do top down recursion DP properly with memset (and how to think about dp properly)

Bit map back log
- https://leetcode.com/problems/smallest-sufficient-team/description/
- This is actually just another form of knapsack
	- Take, don't take over the people
	- Then the state is the bitmap of the team. We want the bitmap to be all ones
- Remembner how knapsack works
	- Take, don't take then fulfil some constraint each time
	- Instead of the constraint being a set value like amount of money, we set the constraint to be the bitmap of the team
- Also how to trace back to get the path, not just the final state

- set, map, etc. how they work
- how to use comparator for binary search

- Contribution transformation
	- And also, fix one variable
	- https://leetcode.com/problems/maximum-frequency-after-subarray-operation/

| Rank | Transformation                             | Importance |
| ---- | ------------------------------------------ | ---------- |
| 1    | Contribution → Kadane/Prefix               | ⭐⭐⭐⭐⭐      |
| 2    | Greedy exchange arguments                  | ⭐⭐⭐⭐       |
| 3    | Count instead of construct                 | ⭐⭐⭐⭐       |
| 4    | Reverse perspective                        | ⭐⭐⭐⭐       |
| 5    | Local contribution (element-wise counting) | ⭐⭐⭐⭐       |
| 6    | Prefix difference transformations          | ⭐⭐⭐        |
| 7    | Fix one variable                           |            |

https://leetcode.com/problems/count-unique-characters-of-all-substrings-of-a-given-string/
- Instead of for each subarray, measure the prroperty, do 
	- For each element count how many subarrays it contributes to
- Each element, go left and right
	- Same as rectangle histogram paradigm
- https://leetcode.com/problems/sum-of-total-strength-of-wizards/description/
- "Group the sub-objects by their special element" is a valid partition **iff** you can define the special element so that every sub-object has exactly one — and when the natural definition allows ties, you impose a tie-break to force uniqueness.
- https://leetcode.com/problems/apply-operations-to-maximize-score/description/


dp
- Identifying knapsack
	- https://leetcode.com/problems/maximum-total-reward-using-operations-i/description/

Greedy + binary search
- https://leetcode.com/problems/maximum-number-of-tasks-you-can-assign/description/
- Identifying binary search


- **Add the claude notes on implementation-style questions**
- **Add claude notes on bugs**
- **Notes on binary search implementation, etc.**
# Reframing
**1. Swap the axis of summation.**

_Trigger:_ "sum / count [property] over all subarrays / pairs / subsets," and iterating over those sub-objects is too many (O(n²) or worse).

_Operation:_ Stop iterating over the sub-objects. Iterate over the _elements_, and for each element ask "how many sub-objects do I contribute to, and how much?" Sum those contributions.

_Worked instance:_ LC 828. Instead of "for each substring, count its unique chars," you did "for each character, count the substrings where it's the unique one" = `(i−left)(right−i)`.

_Why it works structurally:_ A double sum `∑_substrings ∑_elements f(element, substring)` doesn't care which loop is outer. Swapping the order is just Fubini's theorem — it's always _valid_. It's _useful_ when, with the element on the outside, each element's inner contribution has a closed form (a count of ranges) instead of requiring you to actually visit every substring. The hidden structure it exposes: the property was secretly a sum of independent per-element pieces all along, disguised by being written substring-first.

---

**2. Fix the awkward variable.**

_Trigger:_ You have two or more free choices entangled, and your technique can only manage one at a time. (The dead giveaway: "I'd use a sliding window / Kadane's, but there's this _other_ thing I also get to choose.")

_Operation:_ Identify which variable, once pinned, _forces_ or _simplifies_ the others. Loop over its possible values on the outside; solve the now-one-dimensional problem on the inside.

_Worked instance:_ Max Frequency After Subarray Operation. Two coupled choices: the subarray and the additive `x`. Neither fixes cleanly — but fixing the _source value v_ forces `x = k−v`, and collapses the inner problem to Kadane's. You loop v over 1..50 outside.

_Why it works structurally:_ The reason a problem with two free variables is hard is that the optimal choices are _interdependent_ — the best subarray depends on x, the best x depends on the subarray. Fixing one _breaks the dependency_: now the inner problem has a single objective with no hidden second knob, so a standard 1-D technique applies. You pay a factor of |domain of the fixed variable| in time, which is why this only works when that domain is small (here, 50). The constraint `values ≤ 50` was the problem _telling_ you which variable to fix.

---

**3. Take the complement.**

_Trigger:_ The objective is about what you _remove_, _delete_, _exclude_, or _cut_ — especially "minimum to remove" or "remove from the ends."

_Operation:_ Flip to what _remains_. Re-express the objective in terms of the kept portion, solve that, and subtract from the total at the end.

_Worked instance:_ "Remove elements from the front and back so the removed sum equals X" → "find the _longest contiguous middle_ whose sum equals total−X." The removal problem had two moving boundaries (front and back); the kept-middle problem is a single contiguous subarray → sliding window.

_Why it works structurally:_ "What you remove" and "what you keep" partition the same whole, so optimizing one optimizes the other (max keep ⇔ min remove). The reframe is worth doing when the _kept_ set has nicer structure than the _removed_ set. Removing-from-both-ends leaves a contiguous middle — and contiguity is exactly what sliding window needs. The removed parts (a prefix + a suffix) are two disjoint pieces, which is awkward; the kept part is one piece, which is clean. You're trading a two-piece object for a one-piece object.

---

**4. Reverse time.**

_Trigger:_ The process _destroys_ structure as it runs — edges/nodes get deleted, things get disconnected, the graph falls apart. Maintaining the answer through destruction is hard.

_Operation:_ Run the process backward. Deletions become insertions. Process the events in reverse order and _build up_ instead of tearing down.

_Worked instance:_ "Bricks falling when hit" (LC 803): hitting a brick disconnects regions — hard. Reverse it: _add_ the bricks back in reverse order of the hits, and use Union-Find to watch connectivity _grow_.

_Why it works structurally:_ This one is specific to a deep asymmetry in data structures: **union is easy, un-union is hard.** Union-Find can merge components in near-O(1) but has no efficient "split." Connectivity, MST membership, "is this still attached to the roof" — all of these are cheap to maintain under _additions_ and expensive-to-impossible under _removals_. Reversing time converts every removal into an addition, putting you back in the regime your data structures are good at. The hidden structure: the problem only looked hard because it ran in the direction your tools dislike.

---

**5. Move the center (delta / rerooting).**

_Trigger:_ You must compute some quantity for _every_ position / every node, and the naive method recomputes from scratch each time (n separate scans or n separate DFS traversals → O(n²)).

_Operation:_ Compute the answer for _one_ position fully. Then, when moving to an adjacent position, express its answer as the previous answer _plus a small delta_, rather than recomputing.

_Worked instance:_ "Sum of distances to all other nodes, for every node" (LC 834). Naive: BFS from each node, O(n²). Reroot: compute the root's total distance once; when you move the center from a parent to a child, every node in the child's subtree gets _one closer_ (−subtree_size) and every other node gets _one farther_ (+(n−subtree_size)). One DFS down updates all of them.

_Why it works structurally:_ Adjacent positions share almost all of their computation — moving the center by one step changes the relationship to only a _local_ set of elements, leaving the rest in a predictable bulk shift. The naive approach throws away the (n−1)/n of the work that didn't change. The delta captures only what actually moved. This is the same insight as prefix sums or Kadane's ("don't recompute the overlapping part"), generalized to trees and 2-D. Hidden structure: enormous redundancy between consecutive answers.

---

**6. Rewrite the objective algebraically.**

_Trigger:_ The objective is an equation or expression that doesn't match any technique — but it's _manipulable_. Sums of signed terms, absolute values, products, "i+j" style coupled indices, constraints like "P − N = target."

_Operation:_ Do algebra on it. Expand, separate variables, substitute, complete a known identity — until the expression turns into a shape you recognize.

_Worked instances:_ (a) Target Sum: "assign ± to each number to hit `target`." Let P = sum of +'s, N = sum of −'s. Then P−N = target and P+N = total, so P = (total+target)/2 → it's now subset-sum. (b) LC 1499, max of `y_i + y_j + |x_i − x_j|` with `|x_i−x_j| ≤ k`: split the absolute value by assuming `i<j` so it becomes `(y_j + x_j) + (y_i − x_i)`, which _separates_ the two indices — now for each j you just want the best `(y_i − x_i)` in a window, which is a monotonic-deque max.

_Why it works structurally:_ Techniques attach to _shapes_ (a subset-sum shape, a "maximize A_i + B_j over a window" shape). An objective written in the problem's natural language is rarely in canonical shape, but it's often _algebraically equivalent_ to something that is. The blocker is presentational, not fundamental. The two highest-value sub-moves: **separating coupled indices** (get all the `i` stuff on one side, all the `j` stuff on the other → now you can optimize them independently) and **splitting absolute values by case** (assume an ordering to remove the `| |`, which unlocks separation). Hidden structure: the problem already _is_ a standard one, wearing different clothes.

---

**7. Assume the answer and verify (binary search on the answer).**

_Trigger:_ You're optimizing a single numeric value, _and_ feasibility is monotone — if value V works, every value past V works too (or every value before it). Especially "minimize the maximum" / "maximize the minimum."

_Operation:_ Stop trying to _construct_ the optimum. Instead, write a checker `canDo(V)` — "is V achievable?" — and binary search for the boundary where it flips from false to true.

_Worked instance:_ "Split array into ≤ K parts, minimize the largest part-sum." Don't try to find the split. Binary search on the answer S; `canDo(S)` greedily walks the array making a new part whenever the running sum would exceed S, and checks if you used ≤ K parts. Smallest feasible S is the answer.

_Why it works structurally:_ Some problems are _much_ easier to _check_ than to _solve directly_. Constructing the optimal partition is fiddly; verifying "can I stay under S" is a trivial greedy scan. Monotonicity is what makes the search valid: because feasibility is a step function (false…false, true…true), the optimum is exactly the flip point, and binary search finds a flip point in log time. Hidden structure: the answer lives on an ordered axis with a single threshold, so you can _probe_ it instead of _building_ it.