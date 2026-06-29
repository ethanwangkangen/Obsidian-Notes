# Stack & Monotonic Stack — Comprehensive Reference

---

## How to Identify

**Primary signals:**

- "Next greater / next smaller element."
- "For each element, find the nearest element that is larger/smaller on the left or right."
- "Largest rectangle" in a histogram or matrix.
- Nested structures: parentheses, brackets, nested expressions.
- "Remove elements to make the result optimal" (greedy deletion with a stack).

**The monotonic stack principle:** If you need to know, for each element, where the first bigger (or smaller) element is, a stack that maintains a monotonic order lets you "resolve" each element the moment its answer appears. Elements waiting on the stack haven't found their answer yet; when a new element violates the order, all popped elements have just found theirs.

**When a stack is NOT the right tool:**

- If you need the min/max over a SLIDING WINDOW (use a monotonic deque instead — the stack can't efficiently remove from the bottom).
- If you need the min/max over ALL previous elements (just track a running min/max variable).

---

## Monotonic Stack

### Next Greater Element (Right)

For each element, find the first element to its RIGHT that is greater.

**Approach:** Iterate LEFT to RIGHT. Maintain a DECREASING stack (top is smallest). When `nums[i]` is greater than `stack.top()`, pop — and `nums[i]` is the answer for the popped element. Push `nums[i]`.

**At the end,** anything remaining on the stack has no next greater element (answer = -1).

LC 496 (Next Greater Element I), LC 739 (Daily Temperatures — "how many days until warmer").

### Next Greater Element (Circular Array)

Double the array (conceptually iterate `0` to `2n-1`, using `i % n`). Process the same way. Elements in the "second copy" help resolve elements near the end of the original array.

LC 503 (Next Greater Element II).

### Previous Smaller Element (Left)

For each element, find the nearest element to its LEFT that is smaller.

**Approach:** Iterate LEFT to RIGHT. Maintain an INCREASING stack. Pop while `stack.top() >= nums[i]`. After popping, `stack.top()` is the previous smaller element for `nums[i]`. Push `nums[i]`.

### Which Direction to Iterate? What Order to Maintain?

|What you want|Iterate|Stack order|Pop when|
|---|---|---|---|
|Next greater (right)|Left → right|Decreasing|`nums[i] > top`|
|Next smaller (right)|Left → right|Increasing|`nums[i] < top`|
|Prev greater (left)|Left → right|Decreasing|`nums[i] >= top`|
|Prev smaller (left)|Left → right|Increasing|`nums[i] >= top`|

**Strict vs. non-strict:** For "strictly greater," pop when `>=`. For "greater or equal," pop when `>`. Getting this wrong causes subtle bugs, especially in contribution-counting problems.

### Contribution Counting with Monotonic Stack

"Sum of minimums (or maximums) over all subarrays." For each element, find how many subarrays it's the min/max of. This requires knowing the distance to the previous and next smaller (or larger) elements.

**Formula:** Element at index i with previous smaller at index `left` and next smaller at index `right` is the minimum of `(i - left) * (right - i)` subarrays. Its contribution = `nums[i] * (i - left) * (right - i)`.

**Handling duplicates:** Use strict inequality on one side and non-strict on the other to avoid double-counting. Convention: previous strictly smaller, next smaller-or-equal (or vice versa).

LC 907 (Sum of Subarray Minimums), LC 2104 (Sum of Subarray Ranges — sum of maxs minus sum of mins).

---

## Largest Rectangle in Histogram

For each bar, find how far left and right it can extend as the SHORTEST bar. Use a monotonic INCREASING stack of indices.

**Process:** Iterate through bars. When a bar is shorter than `stack.top()`, pop. The popped bar's width extends from the new `stack.top() + 1` to `i - 1`. Width = `i - stack.top() - 1` (if stack is empty, width = `i`).

**Sentinel trick:** Push a bar of height 0 at the end to flush all remaining bars from the stack. Optionally push -1 at the beginning as a boundary.

LC 84 (Largest Rectangle in Histogram).

### Maximal Rectangle in Binary Matrix

Build a histogram for each row: for each cell, the "height" = number of consecutive 1s above (including current row). If cell is 0, height resets to 0. Apply Largest Rectangle in Histogram on each row's heights.

LC 85 (Maximal Rectangle).

---

## Stack for Expressions & Parentheses

### Valid Parentheses

Push open brackets. On close bracket, check if top matches. If not (or stack is empty), invalid. At the end, stack must be empty.

LC 20 (Valid Parentheses).

### Longest Valid Parentheses

Push INDICES onto the stack. Initialize stack with [-1] as a boundary. On ')': pop. If stack becomes empty, push current index as new boundary. Else, `length = i - stack.top()`, update max.

LC 32 (Longest Valid Parentheses).

### Basic Calculator

Use a stack to handle nested parentheses. Track current number, current result, and current sign. On '(': push result and sign to stack, reset. On ')': pop sign and previous result, combine.

LC 224 (Basic Calculator), LC 227 (Basic Calculator II — no parens, handle precedence), LC 772 (Basic Calculator III — full with parens and precedence).

### Decode String "3[abc]"

On '[': push current string and repeat count. On ']': pop and repeat. On digit: build the number. On letter: append to current string.

LC 394 (Decode String).

---

## Greedy Deletion with Stack (Monotonic Stack Greedy)

### Remove K Digits

Build the smallest number by maintaining a monotonic increasing stack of digits. When a new digit is smaller than the top and K > 0, pop (remove the larger digit). After processing all digits, remove remaining K digits from the end. Strip leading zeros.

**Exchange argument proof:** Removing a larger digit before a smaller digit always produces a smaller number than any other removal choice.

LC 402 (Remove K Digits).

### Remove Duplicate Letters

Same idea but with the constraint that each letter appears exactly once. Track "last occurrence" of each character and "in stack" set. Pop only if the top character appears again later (so it can be re-added).

LC 316 (Remove Duplicate Letters / LC 1081 Smallest Subsequence of Distinct Characters).

---

## Other Stack Patterns

### Asteroid Collision

Simulate with a stack. Positive asteroids move right, negative move left. Collision only happens when top is positive and incoming is negative. Pop while collision destroys the top. If incoming survives (or stack is empty or top is negative), push it.

LC 735 (Asteroid Collision).

### The 132 Pattern

Iterate RIGHT to LEFT. Maintain a decreasing stack. Track the maximum popped value as `third` (the "2" in 132). If you encounter a value less than `third`, you've found the "1" — pattern exists.

**Why this works:** The stack maintains potential "3"s (the largest). Popped values become the "2" (second largest). A new value smaller than the "2" is the "1."

LC 456 (132 Pattern).

### Min Stack

Maintain a parallel stack (or pairs) that tracks the current minimum. When pushing, push `min(val, current_min)` onto the min stack. When popping, pop from both.

LC 155 (Min Stack).

### Trapping Rain Water (Stack Approach)

Maintain a decreasing stack. When `height[i] > stack.top()`, pop. Water trapped = `(min(height[i], height[new_top]) - height[popped]) * (i - new_top - 1)`. This captures water layer by layer horizontally. The two-pointer approach (section 2) captures it column by column vertically — either works.

LC 42 (Trapping Rain Water).