# Prefix sums and difference arrays
- Plain prefix sum
	- For querying a range of indexes
		- **Sum** **between** a range of values, **contribution** **between** indexes, etc.
- Prefix sum plus hashmap
	- eg. count subarrays whose sum equals k
	- `Normalisation for 'divisible by k' = ((P[i] % k) + k) % k`
- Prefix XOR
	- Same skeleton
- 2D prefix sum
	- eg. submatrix sum

- Difference array
	- Walk the difference array to count contributions and subtractions
- Sweep line
	- Similar instead you don't walk the difference array, just walk the boundaries of each interval.
	- eg. Number of Flowers in Full Bloom
		- Keep vectors of additions and removals
		- Then **binary search** at each relevant point to know how many additions and subtractions

# 2 Pointers, sliding window
- Sliding window
	- Invariant: **shrinking MUST make the predicate easier**
	- The 'at most' trick
		- `exactly k = atMost(k) - atMost(k-1`
	- Counting windows
		- Whencounting subarrays, fix right endpoint and add `r-l+1` (the number of valid left endpoints)
- Opposite direction 2 pointers
	- On sorted array
	- Invariant: **is value condition symmetric**
- Monotonic deque window
	- Max/min of current window, or 'max minus k'

# Binary search
- Binary search on answer
	- Invariant: **monotonicity**
	- Problem reduces to a **feasibility check**
	- Look out for **max of minimum**, **max of threshold** etc.
	- Remember for set and map
		- Use member `.upper_bound()` instead of general variant
- Binary search inside DP
	- eg. job scheduling DP -> DP is monotonic on i, so just need to find the best previous j

# Greedy
- Regret greedy
	- **I'm able to change the past** upon regret
	- Only make choices in hindsight, and only when I have to.
	- Take everything as you go, push into heap, when budget breaks, evict the worst thin you have taken
- Interval greedy
	- eg. Minimum number of points/arrows to cover
	- Fixed intervals, pick a max non-conflicting subset, etc.
	- **Sort by RIGHT endpoint, take greedily**
- Greedy on parity/structure
	- Operations that flip, pair up or cancel
	- Greedy settle from one end, and move to the other end
- Greedy vs DP
	- Ask: what's the choice at i? 
		- If no obvious choice at the current moment-> greedy
	- Will I regret this choice later?
		- No -> greedy
		- Yes -> DP over the states (unless it's **regret greedy** where the alternative choice is obvious)
	- Counting is **never greedy**

# Heaps
- Top-k and k-way merge
	- Remember that the comparator for the heap sorts **downwards**
	- Keep the 'worst' one at the top so you can always pop the worst one, and maintain the size of the heap
- 2 heaps
	- **Streaming median**, or any 'balance around a pivot'
		- Rebalance so sizes differ by at most one
- Scheduling heeps
	- Eg. meeting rooms, CPU tasks, etc.
	- min-heap on end times
- Lazy deletion
	- Tombstone, when need arbitrary erase from a heap
	- Push replacements, and at pop, skip stale entries
- When considering heap
	- Consider `std::set` as well!
	- **Set** is better when there's
		- **Arbitary erase/predecessor/range query**

# Monotonic stack, contribution counting
- Nearest greatest/smaller
- Contribution counting
	- Iterate over elements, for each one, count how many subarrays it's responsible for
	- Extend left and right
	- eg. 
	- Related: rectangle histogram

# Hashing
- Complement lookup
- Group by canonical key

# Graphs
- BFS, DFS
	- Remember to settle invariant for the **entry point** ie (0, 0) or source node
	- Multi-source BFS
	- Binary search on BFS
	- Cycle detection
		- **3 state DFS: 0 unvisited, 1 on path, 2 done**
		- Node has no cycles if
			- Children DFS all return false
			- Not on current path
			- Only after this, then can set it to 'done'
- Djikstra's
	- Remember 2 optimisation checks
		- One right after popping, check `<`
		- Then another for children, check `<=`

# DP
- Always ask first, **what's the CHOICE I'm making here**
	- Think about it recursively with **wishful thinking**
- Linear/sequence DP
- State-machine DP
- Knapsack DP
	- Can I get this sum by taking/not taking these elements?
- Partition DP
	- eg. Split into k parts, at most k subarrays
	- Best over the first i elements split into j groups
- Take/don't take DP
- Interval DP
	- Eg. cutting ruler, subsequence palindrome
	- Trigger: decision carves out an inner region that must be solved separately from the region around it
		- ie. **CANNOT** fix one end of the partition (eg. 0)
	- Loop by increasing interval length (or just use recursion, easier)
- Grid DP
	- Obvious
	- Often paired with binary search
- LIS family
	- Tail array
- Digit DP
- Bitmask DP
	- `dp[mask]` when next/previous move can be seen from the set alone
	- `dp[mask][last]` when the cost of the next move depends on which elements was last used
	- Submask enumeration
		- `for (int sub = mask; sub; sub = (sub - 1 ) & mask)`
- DP optimisations
	- Prefix/suffix max or min
	- Monotonic deque
	- Hashmap for a pre-solved value
	- Bitset

# Range query - sets
- Just remember that `std::set` and `std::map` are ordered and this can be very useful
- Implementation questions
	- Ordered set for both ordering and O(1) erasing (with iterator) or O(logn) erasing with key
	- 


# Implementatino questions


**Container decision table**:

| Mutations needed                    | Structure                     |
| ----------------------------------- | ----------------------------- |
| static after build                  | sorted vector + `lower_bound` |
| insert, read/pop min only           | heap                          |
| arbitrary erase, predecessor, range | ordered tree (set/map)        |
| exact key, no order                 | hash map                      |
| dense small integer key             | plain vector                  |