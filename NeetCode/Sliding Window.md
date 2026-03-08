# Dynamic Sliding Window
- Subsequence array questions.
- Start by brute force: all possible subarrays
- Then optimise

We start with L, where L is a subarray that extends rightwards as far as it can go.
- [.......]
- Is it possible for another subarray to start before and end after L?
- ie.
- ( .... [ ....] ..... )
- If no, we can use dynamic sliding window

Start with a right pointer that increments from 0 to end of array.
- Then have a left pointer that only ever moves rightwards.
- We know that once right pointer invalidates a subarray and left pointer has to increment, we no longer have to consider any answer with the left pointer before this.

# Longest Substring Without Repeating characters
eg.
[zxyxyz]
Start with 
[z]
[zx]
[zxy]
[zxyx] -> duplicate found, increment left pointer until duplicate removed
[yx] -> there are no more valid subarrys with zx already since this 'x' has invalidated it, so we can stop looking at it and move the left pointer forward.

# Longest Repeating Character Replacement
Since charactersw are bound by uppercase english letters, we can try max substrings that are all A, then B and so on.

Take a substring for example
- Keep a count of the non-A characters.
[AABBBAA]
- Need replace 3 B's
- Then we extend the substring.
	- If added 'A', no problem
	- If not, may not have enough replacements
[AABBBAAC]
- Now too many replacements
- Shift left pointer left until we remove one replacement (the first B)