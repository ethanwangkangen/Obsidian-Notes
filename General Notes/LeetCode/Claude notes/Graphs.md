# Graphs — Comprehensive Reference

---

## How to Identify Graph Problems

**Explicit graphs:** the problem gives nodes and edges directly.

**Implicit graphs (the harder case):** the problem doesn't mention "graph" but the structure is there:

- **Grid → graph:** each cell is a node, adjacent cells are edges. "Shortest path in grid" = BFS.
- **State space → graph:** each state is a node, valid transitions are edges. "Minimum moves to reach target state" = BFS on the state graph.
- **String transformations → graph:** each string is a node, one-character changes are edges. "Shortest transformation sequence" = BFS.
- **Dependencies → DAG:** prerequisites, build order, course scheduling → topological sort.
- **Connectivity → Union-Find or DFS:** "are X and Y connected?" or "how many groups?"

**Choosing the right algorithm:**

|Problem type|Algorithm|
|---|---|
|Shortest path, unweighted|BFS|
|Shortest path, weights 0/1|0-1 BFS (deque)|
|Shortest path, non-negative weights|Dijkstra|
|Shortest path, negative weights|Bellman-Ford|
|Shortest path, at most K edges|Bellman-Ford (K iterations)|
|All-pairs shortest path|Floyd-Warshall|
|Connected components|DFS / BFS / Union-Find|
|Cycle detection (directed)|DFS with 3 colors|
|Cycle detection (undirected)|DFS with parent / Union-Find|
|Dependency ordering|Topological sort|
|Minimum spanning tree|Kruskal's / Prim's|
|Bipartite check|BFS/DFS 2-coloring|

**When graph techniques are NOT appropriate:**

- If the structure has optimal substructure with overlapping subproblems, it's probably DP, not shortest path.
- If you need ALL paths (not just shortest), it's backtracking/DFS, not BFS.

---

## BFS (Breadth-First Search)

### Standard BFS (Shortest Path, Unweighted)

BFS from source. First visit = shortest distance. Use a queue.

**Template:**

```
queue = [source]
visited = {source}
distance = 0
while queue:
    for each node in current level:
        if node == target: return distance
        for each neighbor:
            if neighbor not visited:
                visited.add(neighbor)
                queue.push(neighbor)
    distance++
```

LC 1091 (Shortest Path in Binary Matrix), LC 752 (Open the Lock).

### Multi-Source BFS

Push ALL source nodes into the queue initially with distance 0. BFS outward. Each cell gets the distance to the NEAREST source.

**How to identify:** "For every cell, find distance to nearest X." Reversal of perspective: instead of BFS from each cell to find X, BFS from all X's simultaneously.

LC 994 (Rotting Oranges — all rotten oranges are sources), LC 286 (Walls and Gates), LC 1162 (As Far from Land as Possible).

### BFS on State Spaces (Implicit Graphs)

Encode the state as a hashable key (string, tuple, int). BFS over states.

**Key trick:** the "state" must capture everything needed to determine future moves. For sliding puzzles, the state is the board configuration. For word ladder, the state is the current word.

LC 127 (Word Ladder), LC 773 (Sliding Puzzle), LC 815 (Bus Routes — BFS on bus stops, edges = routes).

### Bidirectional BFS

BFS from both source and target simultaneously. Alternate expanding the smaller frontier. When frontiers meet, path found. Reduces time from O(b^d) to O(b^{d/2}).

**When to use:** Large branching factor, known source AND target.

LC 127 (Word Ladder — bidirectional is the optimal approach).

---

## DFS (Depth-First Search)

### Cycle Detection in Directed Graphs (3-Color DFS)

Three states: WHITE (unvisited), GRAY (in current DFS path), BLACK (fully processed).

- Start DFS at a WHITE node, color it GRAY.
- If you encounter a GRAY node → back edge → CYCLE.
- When done with all neighbors, color node BLACK.

**Why 3 colors?** In directed graphs, reaching a visited node isn't necessarily a cycle — it could be a cross edge. Only a back edge (reaching a GRAY node) indicates a cycle.

LC 207 (Course Schedule — detect cycle), LC 802 (Find Eventual Safe States).

### Cycle Detection in Undirected Graphs

DFS with parent tracking. If you reach a visited node that ISN'T your parent → cycle. Or: Union-Find. If `find(u) == find(v)` before `union(u, v)` → adding edge (u,v) creates a cycle.

LC 684 (Redundant Connection).

### Connected Components

Run DFS/BFS from each unvisited node. Count how many times you start a new traversal.

LC 200 (Number of Islands), LC 323 (Number of Connected Components), LC 547 (Number of Provinces).

### Paint from the Boundary

Instead of finding "enclosed" regions (hard), DFS/BFS from the BOUNDARY and mark everything reachable. What's left is enclosed.

LC 130 (Surrounded Regions — DFS from border O's first, then flip remaining O's), LC 417 (Pacific Atlantic Water Flow — BFS from both oceans inward, intersect).

---

## Topological Sort

### How to Identify

"Prerequisites / dependencies," "ordering constraints," "course schedule," "build order," "alien dictionary." Any DAG where you need a valid ordering.

### Kahn's Algorithm (BFS-based)

1. Compute in-degree of every node.
2. Push all nodes with in-degree 0 into a queue.
3. Process: pop node, add to result, decrement in-degree of all neighbors. If a neighbor hits 0, push it.
4. If result has fewer than n nodes → CYCLE exists.

**For lexicographically smallest order:** use a min-heap instead of a queue.

### DFS-based

Run DFS on all unvisited nodes. Push to result on POST-ORDER (after all children processed). Reverse the result.

LC 207 (Course Schedule — detect if possible), LC 210 (Course Schedule II — return the order), LC 269 (Alien Dictionary — build graph from character ordering), LC 2115 (Find All Possible Recipes).

---

## Dijkstra's Algorithm

Min-heap BFS for weighted graphs with NON-NEGATIVE edges.

**Template:**

```
heap = [(0, source)]
dist = {source: 0}
while heap:
    d, u = heap.pop()
    if d > dist[u]: continue  // stale entry (lazy deletion)
    for each (v, weight) in neighbors(u):
        if d + weight < dist.get(v, INF):
            dist[v] = d + weight
            heap.push((dist[v], v))
```

**Key trick — lazy deletion:** Don't use a `visited` set that prevents revisiting. Instead, skip a node if its popped distance exceeds its current known distance. This avoids the need for decrease-key operations.

**Modification for K-th shortest path:** Allow each node to be popped up to K times. The K-th pop gives the K-th shortest distance.

**Common transformations:**

- "Path with minimum effort" → edge weight = abs difference in heights. Dijkstra on max edge weight (not sum). Relaxation: `new_dist = max(dist[u], weight(u,v))`.
- "Swim in rising water" → same idea: minimize the maximum height along the path.
- "Minimum cost to reach destination with constraints" → add constraint as a state dimension.

LC 743 (Network Delay Time), LC 1631 (Path With Minimum Effort), LC 778 (Swim in Rising Water), LC 1514 (Path with Maximum Probability — max-heap, multiply probabilities).

---

## Bellman-Ford

Relax all edges V-1 times. If any edge can still be relaxed after V-1 iterations, there's a negative cycle.

**K-edge constraint (Cheapest Flights within K stops):** Run only K+1 iterations. **Critical:** copy the distance array BEFORE each iteration to avoid using paths with too many edges. Without the copy, you might use a distance that was updated in the current iteration (which implies more edges).

```
for i = 0 to K:
    prev = copy of dist
    for each edge (u, v, w):
        dist[v] = min(dist[v], prev[u] + w)  // use PREV, not current dist
```

LC 787 (Cheapest Flights Within K Stops).

---

## 0-1 BFS (Deque-Based)

When edge weights are ONLY 0 or 1, use a deque instead of a priority queue. Push 0-weight transitions to the FRONT, 1-weight transitions to the BACK. This maintains the sorted order of the deque.

**How to identify:** Grid where some moves are free and others cost 1 (e.g., moving in direction of arrows is free, changing direction costs 1).

LC 1368 (Minimum Cost to Make at Least One Valid Path), LC 2290 (Minimum Obstacle Removal to Reach Corner).

---

## Union-Find (Disjoint Set Union)

### Core Operations

`find(x)`: return the root of x's component. With PATH COMPRESSION: point every node directly at the root during find.

`union(x, y)`: merge components. With UNION BY RANK/SIZE: attach smaller tree under larger tree.

Both optimizations together give amortized O(α(n)) ≈ O(1) per operation.

### How to Identify

"Are X and Y connected?", "How many connected components?", "Merge groups," "Process edges and track connectivity."

**Union-Find vs. DFS for connectivity:** Union-Find supports DYNAMIC edge additions efficiently. DFS is better for one-shot traversal of a static graph.

### Key Tricks

**Count components:** Start with n components. Each successful union (where find(x) ≠ find(y)) decrements the count by 1.

**Detect cycle in undirected graph:** If `find(u) == find(v)` BEFORE `union(u, v)`, this edge creates a cycle.

**Process in reverse:** If the problem REMOVES edges/nodes, it's hard to maintain connectivity. REVERSE IT: add edges/nodes in reverse order using Union-Find (connectivity increases monotonically with additions).

**Weighted Union-Find:** Track a weight/offset on each edge for ratio/relationship problems. During find, accumulate weights along the path.

LC 684 (Redundant Connection), LC 721 (Accounts Merge), LC 947 (Most Stones Removed), LC 1319 (Number of Operations to Make Network Connected), LC 399 (Evaluate Division — weighted UF), LC 803 (Bricks Falling When Hit — reverse processing).

---

## Minimum Spanning Tree

**Kruskal's:** Sort edges by weight. Use Union-Find. Add edge if it connects two different components. Stop after n-1 edges. O(E log E).

**Prim's:** Start from any node. Use min-heap of (weight, node) for unvisited nodes. Pop cheapest, add to MST, push its neighbors. O(E log V).

**When to use which:** Kruskal's is simpler for sparse graphs (fewer edges). Prim's is better for dense graphs.

LC 1584 (Min Cost to Connect All Points), LC 1135 (Connecting Cities With Minimum Cost).

---

## Bipartite Check

Try to 2-color the graph. BFS/DFS: alternate colors. If a neighbor has the same color as the current node, NOT bipartite.

**How to identify:** "Split into two groups such that no two in the same group are connected," "is the graph 2-colorable?"

LC 785 (Is Graph Bipartite), LC 886 (Possible Bipartition).

---

## Graph Modeling Tricks

**Virtual source/sink:** Add a dummy node connected to all "real" sources (or sinks). Converts multi-source/sink problems into single-source/sink.

**Layer graph / state graph:** When movement has multiple dimensions (position + fuel, position + keys collected), create a LAYERED graph where each layer represents a state. Dijkstra/BFS on this expanded graph.

LC 864 (Shortest Path to Get All Keys — state = position + bitmask of keys), LC 787 (position + stops remaining).

**Reverse the graph:** If the problem asks "can X reach Y?", it's sometimes easier to reverse all edges and ask "can Y reach X?" (or BFS from Y in the reversed graph). Useful for "which nodes can reach the target."

LC 802 (Find Eventual Safe States — reverse graph or DFS with memoization).