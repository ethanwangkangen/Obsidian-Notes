# Key signals
- Contiguous subarrays counting
	- Anytime see 'count all subarrays satisfying X', first instinct should be **sliding window/2 pointer**
- Think about, for **each r**, subarrays **ending at that r**
- Most importantly, **monotonically breakable property**
	- If the property fails for window `[l,r]`, it will also fail for any wider window `[l, r+1]`, `[l-1,r]`

# Core property
- If `P([l, r])` holds, then `P([l', r'])` holds for any l ≤ l' ≤ r' ≤ r.
	- **If window is valid, every sub-window is also invalid**

Just ask:
> "If I take a valid window and make it smaller, can it become invalid?"
> If no -> sub-window inheritance holds -> sliding window works


# How can we maintain the window property?
- Keep track of minimum/maximums
	- Deque

# Tricks
- **Exactly k**  = (At most k) - (At most k-1)
	- Exactly k doesnt fit the monotonic property for sliding window
	- But if at most k have monotonicity, then we can use this trick
	- https://leetcode.com/problems/subarrays-with-k-different-integers/description/
	- https://leetcode.com/problems/count-number-of-nice-subarrays/description/