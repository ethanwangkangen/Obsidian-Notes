# Trees — Comprehensive Reference

---

## How to Identify

**Primary signals:**

- The input is explicitly a tree (binary tree, BST, N-ary tree).
- "Root-to-leaf path," "depth," "height," "diameter."
- "Lowest common ancestor."
- "Serialize / deserialize."

**Default approach for almost all tree problems:** Recurse on left subtree, recurse on right subtree, combine results. This is the DFS recursive pattern. Before trying anything fancier, check if this basic template solves the problem.

**When to use BFS instead:** When you need level-order information (level averages, right-side view, zigzag traversal).

**When to use iterative instead of recursive:** When you need explicit control over traversal order, or when the tree could be deep enough to overflow the call stack (rare in LC, but relevant in practice).

---

## Core Recursive Pattern

### "Recurse Left, Recurse Right, Combine"

Almost all tree problems follow this template:

```
function solve(node):
    if node is null: return base_case
    left_result = solve(node.left)
    right_result = solve(node.right)
    return combine(node.val, left_result, right_result)
```

**Examples:**

- Max depth: `return max(left, right) + 1`. Base: null → 0.
- Sum of all nodes: `return node.val + left + right`. Base: null → 0.
- Is symmetric: compare left.left with right.right AND left.right with right.left.

LC 104 (Maximum Depth), LC 100 (Same Tree), LC 101 (Symmetric Tree), LC 226 (Invert Binary Tree).

### Returning vs. Updating a Global (The Crucial Trick)

Some problems need the function to RETURN one thing to the parent while TRACKING a different global answer.

**Diameter of Binary Tree:**

- **Return** to parent: the HEIGHT of this subtree = `max(left_h, right_h) + 1`.
- **Update global** with: `left_h + right_h` (the diameter passing through this node).
- The diameter might not pass through the root. You must check at EVERY node.

**Binary Tree Maximum Path Sum:**

- **Return** to parent: the max "one-branch" sum extending upward = `node.val + max(0, max(left, right))`. Can't go both ways because the parent needs a single chain.
- **Update global** with: `node.val + max(0, left) + max(0, right)` (the full path through this node, going both left and right).
- Use `max(0, ...)` because a negative branch is worse than not taking it.

**How to know when you need this pattern:** If the answer at a node involves combining left AND right results in a way that can't be extended upward (e.g., a path going left-through-node-right can't also extend to the parent), you need the split. The return value gives the parent what it CAN use; the global update captures what it CAN'T propagate.

LC 543 (Diameter), LC 124 (Binary Tree Max Path Sum), LC 687 (Longest Univalue Path).

---

## BST (Binary Search Tree) Properties

**Fundamental property:** Inorder traversal of a BST produces a SORTED sequence.

### Validate BST

Two approaches:

1. **Inorder traversal:** traverse inorder, check that each value is greater than the previous.
2. **Range passing:** recursively pass a `(min, max)` range. Each node must be within its range. Left child gets `(min, node.val)`, right child gets `(node.val, max)`.

LC 98 (Validate BST).

### K-th Smallest in BST

Inorder traversal, decrement K at each visit. When K hits 0, that's the answer. O(H + K) where H is height.

LC 230 (Kth Smallest Element in BST).

### LCA in BST

Exploit sorted property: if both target values are less than current, go left. Both greater, go right. Otherwise, current node is the LCA (the split point).

LC 235 (LCA of BST).

### LCA in General Binary Tree

Recurse left and right. If both return non-null, current node is the LCA. If only one returns non-null, propagate that upward. Base case: if current node is null or equals either target, return current node.

LC 236 (LCA of Binary Tree).

### Insert / Delete in BST

Insert: binary search to find the correct leaf position. Delete: three cases — leaf (just remove), one child (replace with child), two children (replace with inorder successor or predecessor, then delete that).

LC 450 (Delete Node in BST), LC 701 (Insert into BST).

---

## Tree Construction from Traversals

### Preorder + Inorder

Preorder's FIRST element is the root. Find that root in inorder — everything to its left is the left subtree, everything to the right is the right subtree. Recurse.

**Optimization:** Use a hashmap to get O(1) lookup of root position in inorder (avoids O(n) linear search at each level).

LC 105 (Construct from Preorder + Inorder).

### Postorder + Inorder

Postorder's LAST element is the root. Same partition logic. But build RIGHT subtree first (because postorder visits right before the root in reverse).

LC 106 (Construct from Postorder + Inorder).

---

## Level-Order Traversal (BFS)

Use a queue. Track level boundaries by processing all nodes at the current queue size.

```
queue = [root]
while queue:
    level_size = queue.size()
    for i = 0 to level_size - 1:
        node = queue.pop_front()
        process(node)
        if node.left: queue.push(node.left)
        if node.right: queue.push(node.right)
```

**Right side view:** take the LAST element of each level. **Zigzag:** alternate reading direction each level (or always read left-to-right but reverse every other level in the output). **Level averages / maximums:** aggregate within each level.

LC 102 (Level Order), LC 199 (Right Side View), LC 103 (Zigzag), LC 515 (Largest Value in Each Row), LC 637 (Average of Levels).

---

## Serialize / Deserialize

### Preorder with Null Markers

Serialize: preorder DFS, recording "null" for missing children. Produces a unique representation. Deserialize: read values sequentially. If value is "null", return null. Else, create node, recurse for left, then right. The preorder sequence determines the tree uniquely when nulls are recorded.

LC 297 (Serialize and Deserialize Binary Tree).

### Level-Order

Serialize: BFS, recording nulls. Deserialize: BFS, using a queue to assign children.

LC 449 (Serialize and Deserialize BST — can omit nulls because BST property constrains reconstruction).

---

## Path Problems

### Root-to-Leaf Paths

DFS, maintaining a running path. When you reach a leaf, record/check the path. Backtrack by removing the last element.

**Path Sum:** check if any root-to-leaf path sums to target. Recurse with `target - node.val`. **Path Sum II:** collect all such paths. **Path Sum III:** count paths (not necessarily root-to-leaf) that sum to target. Use prefix sum + hashmap ON the path (like subarray sum equals K but on a tree path). Backtrack by removing from the hashmap when returning.

LC 112 (Path Sum), LC 113 (Path Sum II), LC 437 (Path Sum III — prefix sum on tree).

---

## Tree DP

Root the tree. DFS post-order. Compute `dp[node]` from children's DP values.

**House Robber III:** `dp[node] = (rob_this, skip_this)`. If rob: can't rob children, take `skip(left) + skip(right) + node.val`. If skip: take `max(rob, skip)` for each child.

**Binary Tree Cameras:** Greedy/DP with three states per node: covered (by parent or child), has camera, not covered. Process bottom-up.

LC 337 (House Robber III), LC 968 (Binary Tree Cameras).

---

## Other Tree Patterns

**Flatten to Linked List (preorder):** reverse postorder (right, left, root). Set each node's right to the previously processed node, left to null. LC 114 (Flatten Binary Tree to Linked List).

**Vertical Order Traversal:** BFS with (column, row) coordinates. Sort by column, then row, then value. LC 987 (Vertical Order Traversal).

**Count Good Nodes:** DFS tracking the max value from root to current node. If current ≥ max, it's "good." LC 1448 (Count Good Nodes in Binary Tree).