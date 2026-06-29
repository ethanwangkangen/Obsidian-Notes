# Common Techniques
- Sorting before doing something else
- Binary search on final answer, if monotonic
- If we have some sort of algebraic equation, see if we can **rearrange it**
- 

# Dynamic Programming
- Is DP applicable?
	- If I don't care about **how** I got to a state, just that I got to it
	- If a state can be calculated **only from other states**
	- If state helps me to **answer the question**, perhaps even **directly**
	- If theres **monotonically decreasing resourc**e/dimension that guarantees **termination** (eg. decreasing fuel, index moving forward)
- Think about what happens at each "step"
	- eg.
	- **Move** somewhere else (graph/grid problems — Count All Possible Routes
	- Consider a new **index** i (linear DP — House Robber, LIS)
	- Consider **up to** a new **index** i (prefix-style — Coin Change)
	- Take / don't take something (knapsack — 0/1 Knapsack, Target Sum)
	- Match / skip / insert / delete (two-sequence — Edit Distance, LCS)
	- Choose where to **split**/cut an interval (interval DP — Burst Balloons, Matrix Chain)
	- Choose which element to **process last** in a range (interval DP trick — makes subproblems independent)
	- Transition between constraint states (state machine — Stock Buy/Sell with cooldown, holding/not holding)
- Try the brute force approach
	- Write the recursive function that explores all choices
	- Don't think about DP yet — just simulate decisions
	- The function signature becomes your state
	- The function body becomes your recurrence
	- Remember how recursion works
		- **ASSUME IT WORKS** for the sub problems
		- Use it to solve the larger problem
- What is the state
	- What information do I need to retain to get the answer
	- What changes between steps
	- What's the **minimum** I need to know to make **optimal future decisions** without knowing past decisions
	- Can I reduce the size of the state? ie. if one dimension can be computed from the rest
	- Example states
		- Position/index (i, or i,j for 2 sequences)
		- Remaining resource
		- Constraint carried forward
		- Subset of elements used (bitmask when n<=20)
		- Interval boundaries (l,r) for interval DP
- What is the recurrence
	- Think about the brute force recursion
	- To get the answer for `dp[i]`, what do I need to do with all the other `dp[j]` that I have?
		- Then apply optimisations if applicable.
		- Sum over a range: **prefix sum dp**
		- DP is monotonically increasing and we want to find the largest index: **binary search**
			- **Longest Increasing Subsequence (LIS) paradigm**
		- Can precompute so don't have to calculate sum of dp each time? (running max/min maintained during inner loop — Job Scheduling difficulty)
		- Window, and we want to find the max out of the previous ones: **deque**
		- Transition depends on nearest smallest/greater element: monotonic stack
	- Think about what the **new element** can do
		- Eg. for array partition, what existing arrays can the new element extend from
- Base cases
	- Smallest subproblems where answer is trvially known
	- Common patterns
		- Empty sequence - 0 or 1
		- Index OOB - 0 or INF 
		- Single element/ interval of length 1 - directly computable
		- Resource exactly 0 - check if valid end state
- Invalid cases
	- Remember to account for **invalid cases**
		- When initialising
		- Between transitions as well!

## Invalid vs 0?
- eg. https://leetcode.com/problems/cherry-pickup-ii/description/
- Starting values are INT_MIN and not 0
	- Because some states are impossible to reach and should be treated as such by default until proven otherwise


## How to store the state
- 2 states
	- Both fixed
		- `vector<vector<int>>`
	- One fixed, one open
		- `vector<unordered_map<int, int>>`

## Think about what the new index'd element should/can do!
- eg. Array partioning -> what new partition can this array be part of? Then use the recurrence
	- NOT what existing partitions can extend into this array? Much more confusing
- eg. https://leetcode.com/problems/minimum-cost-for-tickets/description/
	- `dp[i]` = cost to satisfy condition until day i
	- WRONG way of thinking
		- At day i, some previous day needs to cover day i.
		- Problematic
	- CORRECT way of thinking
		- What are my options for day i?
		- Buy a 1 day pass -> `costs[0] + dp[i-1]`
		- Buy a 7 day pass -> `costs[1] + dp[i-7]`
		- etc.
	- Also think about what the **recurrence actually means**
		- For `dp[i]`, we have `dp[i-1],....`
		- But all previous dp's only guarantee a cover up to `i-x`, they don't guarantee covering the condition until `i` at all!
		- So we can't think of it like that.
	- Think from the perspective of the current state making a decision, not from the perspective of past states reaching the current one
		- Even in cases like **LIS**, where it seems like we're just extending from past states, the **active choice here** is thatwe're choosing which predecessor to extend from

## 1-indexed or 0-indexed DP?
|Shape|Use|Why|
|---|---|---|
|"First `i` items, skip or take"|1-indexed|Need `dp[0]` = nothing taken yet|
|"Two sequences, align/match"|1-indexed|Need empty prefix for both|
|"Answer ending at index `i`"|0-indexed|No empty state, answer is `max(dp[i])`|
|"Interval `(i, j)`"|0-indexed|Indices are actual positions|
|"Amount/capacity as state"|0-indexed|`dp[0]` naturally means "zero amount"|
- For 1-indexed DP
	- ie. `dp[i]` = i elements considered, ie. up to `nums[i+1]`
	- Settle the base case outside (0 elements considered, 0 groups etc)
	- Then loop from 1 (generally)
	- eg. https://leetcode.com/problems/maximum-score-using-exactly-k-pairs/
- Also, don't confuse **invalid state** with **minimal state**
	- eg. for the above problem also. Setting base case is 0, not LLONG_MIN like invalid states.

## Careful with minimal values
- `std::numeric_limits<double>::lowest()` instead of `std::numeric_limits<double>::min()`!!!

# DP Types
- Partition DP
	- `dp[i]` = best using first i elements
	- Try all previous cut positions
- Weighted interval schedulign DP
	- Sort by end time
	- `dp[i]` = take or skip
	- Binary search previous compatible job
- Prefix sum + DP
- LIS-style DP
- State Machine DP
# Questions to return to
- [3956. Maximum Sum of M Non-Overlapping Subarrays I](https://leetcode.com/problems/maximum-sum-of-m-non-overlapping-subarrays-i/)
- https://leetcode.com/problems/maximum-score-using-exactly-k-pairs/

# How to do prefix sum properly
// 1 indexed prefix
```cpp
vector<long long> prefix(sz + 1, 0);
prefix[0] = 0;
for (int i = 1; i <= sz; ++i) prefix[i] = prefix[i-1] + nums[i-1];
```
`sum(nums[l..r])` (0-indexed inclusive) = `prefix[r+1] - prefix[l]`.
