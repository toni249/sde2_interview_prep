# Graphs — Pattern Deep Dive
> FAANG Level | Java | Easy → Hard
> You said this is a weak area — go through this VERY carefully.

---

## Pattern Map

```
GRAPHS
├── P1: BFS                   → shortest path (unweighted), multi-source, 0-1 BFS
├── P2: DFS                   → connected components, cycle detection, islands
├── P3: Topological Sort      → dependency order (Kahn's BFS + DFS postorder)
├── P4: Union-Find (DSU)      → connected components, detect cycle, Kruskal MST
├── P5: Dijkstra              → shortest path (weighted, non-negative)
├── P6: Bellman-Ford          → negative edges, detect negative cycle
├── P7: Floyd-Warshall        → all-pairs shortest path
├── P8: MST (Prim + Kruskal)  → minimum spanning tree
└── P9: Advanced              → bridges, SCC (Kosaraju), bipartite check
```

---

## Graph Representations

```java
// Adjacency List (most common for FAANG)
List<List<Integer>> graph = new ArrayList<>();
for (int i = 0; i < n; i++) graph.add(new ArrayList<>());
graph.get(u).add(v);  // directed edge u→v
graph.get(v).add(u);  // undirected: add both

// Adjacency Matrix (for dense graphs or when weights needed)
int[][] matrix = new int[n][n];
matrix[u][v] = weight;

// Edge list (for MST algorithms)
int[][] edges = {{u, v, weight}, ...};
```

---

## P1: BFS — Breadth-First Search

### Core Template
```java
void bfs(List<List<Integer>> graph, int start) {
    boolean[] visited = new boolean[n];
    Queue<Integer> queue = new LinkedList<>();
    visited[start] = true;
    queue.offer(start);

    while (!queue.isEmpty()) {
        int node = queue.poll();
        // process node
        for (int neighbor : graph.get(node)) {
            if (!visited[neighbor]) {
                visited[neighbor] = true;
                queue.offer(neighbor);
            }
        }
    }
}
```

### Shortest Path Template (BFS = shortest path for unweighted graphs)
```java
int[] shortestPath(List<List<Integer>> graph, int start) {
    int[] dist = new int[n];
    Arrays.fill(dist, -1);
    dist[start] = 0;
    Queue<Integer> queue = new LinkedList<>();
    queue.offer(start);

    while (!queue.isEmpty()) {
        int node = queue.poll();
        for (int neighbor : graph.get(node)) {
            if (dist[neighbor] == -1) {
                dist[neighbor] = dist[node] + 1;
                queue.offer(neighbor);
            }
        }
    }
    return dist;
}
```

### Variation 1: Rotting Oranges (LC 994) — Multi-Source BFS

**Problem:** Grid cells are 0 (empty), 1 (fresh orange), 2 (rotten). Every minute, a rotten orange rots all 4-adjacent fresh oranges. Return the minimum minutes until no fresh oranges remain, or -1 if impossible.

**Approach (Multi-Source BFS):**
- A naive BFS from each rotten orange would be O(R × V). Instead, seed the queue with **all** rotten oranges at once → they all "expand" together, one ring per minute.
- Each BFS *level* corresponds to one minute. Track level count to get time.
- Count fresh oranges up front; after BFS, if fresh > 0 some orange was unreachable → return -1.
- This works because BFS level = shortest distance from the nearest source (the rotten orange that reaches it first).

```java
// Multiple starting points simultaneously — put all sources in queue at start
int orangesRotting(int[][] grid) {
    int m = grid.length, n = grid[0].length;
    Queue<int[]> queue = new LinkedList<>();
    int fresh = 0;

    for (int i = 0; i < m; i++) {
        for (int j = 0; j < n; j++) {
            if (grid[i][j] == 2) queue.offer(new int[]{i, j});  // all rotten sources
            if (grid[i][j] == 1) fresh++;
        }
    }
    if (fresh == 0) return 0;

    int minutes = 0;
    int[][] dirs = {{0,1},{0,-1},{1,0},{-1,0}};
    while (!queue.isEmpty()) {
        int size = queue.size();
        for (int i = 0; i < size; i++) {
            int[] cell = queue.poll();
            for (int[] d : dirs) {
                int r = cell[0] + d[0], c = cell[1] + d[1];
                if (r >= 0 && r < m && c >= 0 && c < n && grid[r][c] == 1) {
                    grid[r][c] = 2;
                    fresh--;
                    queue.offer(new int[]{r, c});
                }
            }
        }
        minutes++;
    }
    return fresh == 0 ? minutes - 1 : -1;
}
```

### Variation 2: 0-1 BFS (Deque-based)

**Problem context:** Shortest path in a graph whose edges all have weight 0 or 1 (e.g., "free" vs "costly" moves, or grid with walls passable at cost 1). Dijkstra works but is O((V+E) log V) — overkill. We can do it in O(V+E).

**Approach:**
- Use a **deque** instead of a priority queue.
- A weight-0 edge keeps the cost the same, so push that neighbor to the **front** (process before the current "layer").
- A weight-1 edge increases cost by 1, so push to the **back** (next layer).
- Result: nodes still come out in non-decreasing order of distance, like Dijkstra, but in linear time.

> Edge weights are 0 or 1. Use deque: weight 0 → front, weight 1 → back.
```java
int[] shortestPath01BFS(int[][] graph, int start) {
    int[] dist = new int[n];
    Arrays.fill(dist, Integer.MAX_VALUE);
    dist[start] = 0;
    Deque<Integer> deque = new ArrayDeque<>();
    deque.offerFirst(start);

    while (!deque.isEmpty()) {
        int node = deque.pollFirst();
        for (int[] edge : graph[node]) {  // [neighbor, weight (0 or 1)]
            int next = edge[0], w = edge[1];
            if (dist[node] + w < dist[next]) {
                dist[next] = dist[node] + w;
                if (w == 0) deque.offerFirst(next);   // 0 weight → front
                else        deque.offerLast(next);    // 1 weight → back
            }
        }
    }
    return dist;
}
```

### Variation 3: Word Ladder (LC 127) — BFS on implicit graph

**Problem:** Given `beginWord`, `endWord`, and a dictionary `wordList`. Each step you may change exactly one character; the resulting word must exist in the list. Return the **minimum number of words** in a transformation sequence (length, not steps), or 0 if impossible.

**Approach (BFS on implicit graph):**
- Build the graph **implicitly**: two words are neighbors iff they differ by exactly one character.
- Don't pre-compute all O(N²) edges. Instead, from a word, generate all `26 × L` one-char variants and check membership in a `HashSet` — O(L × 26) per word.
- BFS from `beginWord`. The level you find `endWord` at + 1 = answer.
- Remove visited words from the set to mark them (saves a separate `visited` set).

```java
// Nodes = words, edge = differ by 1 character
int ladderLength(String beginWord, String endWord, List<String> wordList) {
    Set<String> wordSet = new HashSet<>(wordList);
    if (!wordSet.contains(endWord)) return 0;

    Queue<String> queue = new LinkedList<>();
    queue.offer(beginWord);
    int steps = 1;

    while (!queue.isEmpty()) {
        int size = queue.size();
        for (int i = 0; i < size; i++) {
            String word = queue.poll();
            char[] chars = word.toCharArray();
            for (int j = 0; j < chars.length; j++) {
                char orig = chars[j];
                for (char c = 'a'; c <= 'z'; c++) {
                    chars[j] = c;
                    String newWord = new String(chars);
                    if (newWord.equals(endWord)) return steps + 1;
                    if (wordSet.contains(newWord)) {
                        wordSet.remove(newWord);   // mark visited by removing
                        queue.offer(newWord);
                    }
                }
                chars[j] = orig;
            }
        }
        steps++;
    }
    return 0;
}
```

### BFS Questions

| #   | Problem                        | LC#  | Difficulty |
| --- | ------------------------------ | ---- | ---------- |
| M1  | Number of Islands              | 200  | M          |
| M2  | Rotting Oranges                | 994  | M          |
| M3  | Walls and Gates                | 286  | M          |
| M4  | Pacific Atlantic Water Flow    | 417  | M          |
| M5  | Word Ladder                    | 127  | H          |
| M6  | Shortest Path in Binary Matrix | 1091 | M          |
| H1  | Minimum Jumps to Reach End     | 45   | M          |
| H2  | Sliding Puzzle                 | 773  | H          |

---

## P2: DFS — Depth-First Search

### Core Template
```java
void dfs(List<List<Integer>> graph, int node, boolean[] visited) {
    visited[node] = true;
    // process node (preorder)
    for (int neighbor : graph.get(node)) {
        if (!visited[neighbor]) {
            dfs(graph, neighbor, visited);
        }
    }
    // process node (postorder) — used in topological sort
}
```

### Variation 1: Number of Islands (LC 200)

**Problem:** Given an `m x n` grid of `'1'` (land) and `'0'` (water), count the number of islands. An island is a maximal group of land cells connected 4-directionally.

**Approach:**
- Scan every cell. When we find an unvisited land cell, we've discovered a **new island** → increment count.
- Run DFS/BFS from there to mark **all** land cells in that island as visited so we don't double-count.
- Marking trick: overwrite `'1' → '0'` in-place to save O(m·n) `visited` space (allowed if input can be mutated).
- Time: O(m·n) — each cell visited once. Space: O(m·n) recursion stack worst case.

```java
int numIslands(char[][] grid) {
    int count = 0;
    for (int i = 0; i < grid.length; i++) {
        for (int j = 0; j < grid[0].length; j++) {
            if (grid[i][j] == '1') {
                dfs(grid, i, j);
                count++;
            }
        }
    }
    return count;
}
void dfs(char[][] grid, int i, int j) {
    if (i < 0 || i >= grid.length || j < 0 || j >= grid[0].length || grid[i][j] != '1') return;
    grid[i][j] = '0';   // mark visited (in-place)
    dfs(grid, i+1, j); dfs(grid, i-1, j);
    dfs(grid, i, j+1); dfs(grid, i, j-1);
}
```

### Variation 2: Cycle Detection — Undirected Graph

**Problem:** Given an undirected graph with `n` nodes and edge list, return `true` if it contains a cycle.

**Approach (DFS with parent tracking):**
- In an undirected graph, every edge `u-v` looks like *both* `u→v` and `v→u` in the adjacency list. So when we DFS from `u` to `v`, we'll see `u` again in `v`'s list — that's NOT a cycle, just the same edge.
- Rule: a cycle exists iff we encounter a visited neighbor that is **not our parent** (the node we came from).
- Iterate over all components — graph may be disconnected.
- Alternative: DSU. For each edge, if both endpoints already share a root → cycle.

```java
// DFS: cycle exists if we visit an already-visited node that is NOT the parent
boolean hasCycle(List<List<Integer>> graph, int n) {
    boolean[] visited = new boolean[n];
    for (int i = 0; i < n; i++) {
        if (!visited[i] && dfs(graph, i, -1, visited)) return true;
    }
    return false;
}
boolean dfs(List<List<Integer>> graph, int node, int parent, boolean[] visited) {
    visited[node] = true;
    for (int neighbor : graph.get(node)) {
        if (!visited[neighbor]) {
            if (dfs(graph, neighbor, node, visited)) return true;
        } else if (neighbor != parent) {   // visited and not parent → cycle
            return true;
        }
    }
    return false;
}
```

### Variation 3: Cycle Detection — Directed Graph

**Problem:** Given a directed graph, detect if it contains a cycle. Classic case: Course Schedule (LC 207) — can you finish all courses given prerequisites?

**Approach (3-color DFS):**
- Parent-check trick (used in undirected) fails here because direction matters.
- Use 3 states per node:
  - `0 = unvisited` (white)
  - `1 = in current DFS path` (gray) — on the recursion stack
  - `2 = fully processed` (black) — done with this subtree
- During DFS, if we encounter a neighbor with state `1`, that's a **back edge** → cycle.
- State `2` means we already explored this branch with no cycle, safe to skip.
- Alternative: Kahn's topo sort — if processed count < n, cycle exists.

```java
// Three states: 0=unvisited, 1=in current DFS stack, 2=done
// Cycle if we hit a node with state 1 (in current path)
int[] state;
boolean hasCycleDirected(List<List<Integer>> graph, int n) {
    state = new int[n];
    for (int i = 0; i < n; i++) {
        if (state[i] == 0 && dfsDirected(graph, i)) return true;
    }
    return false;
}
boolean dfsDirected(List<List<Integer>> graph, int node) {
    state[node] = 1;   // mark as in-progress
    for (int neighbor : graph.get(node)) {
        if (state[neighbor] == 1) return true;   // back edge → cycle
        if (state[neighbor] == 0 && dfsDirected(graph, neighbor)) return true;
    }
    state[node] = 2;   // mark as done
    return false;
}
```

### Variation 4: Clone Graph (LC 133)

**Problem:** Given a reference to a node in a connected undirected graph, return a **deep copy** of the graph. Each `Node` has a value and a list of neighbors.

**Approach (DFS + hash map memoization):**
- We must clone nodes once and reuse them — otherwise infinite recursion on cycles.
- Maintain `Map<originalNode, clonedNode>`. Before recursing into a neighbor, check the map; if cloned, reuse.
- DFS pattern: clone current → put in map → recursively clone neighbors and link them.
- BFS variant: queue of originals, lazily create clones — same complexity, no recursion.
- Time: O(V + E). Space: O(V) for the map.

```java
Map<Node, Node> map = new HashMap<>();
Node cloneGraph(Node node) {
    if (node == null) return null;
    if (map.containsKey(node)) return map.get(node);
    Node clone = new Node(node.val);
    map.put(node, clone);
    for (Node neighbor : node.neighbors) {
        clone.neighbors.add(cloneGraph(neighbor));
    }
    return clone;
}
```

### DFS Questions

| # | Problem | LC# | Difficulty |
|---|---------|-----|------------|
| M1 | Number of Islands | 200 | M |
| M2 | Course Schedule (cycle detection) | 207 | M |
| M3 | Clone Graph | 133 | M |
| M4 | Number of Provinces | 547 | M |
| M5 | Pacific Atlantic Water Flow | 417 | M |
| H1 | Critical Connections (Bridges) | 1192 | H |

---

## P3: Topological Sort ★★★

### What Is It?
> Linear ordering of nodes in a DAG (Directed Acyclic Graph) such that for every edge u→v, u comes before v.
> Use when: "ordering tasks with dependencies", "course prerequisites".

### Method 1: Kahn's Algorithm (BFS-based) — FAANG Preferred
```java
// Start with all nodes that have in-degree 0 (no prerequisites)
// Remove them one by one, reducing neighbors' in-degree
List<Integer> topoSort(List<List<Integer>> graph, int n) {
    int[] inDegree = new int[n];
    for (int u = 0; u < n; u++) {
        for (int v : graph.get(u)) inDegree[v]++;
    }

    Queue<Integer> queue = new LinkedList<>();
    for (int i = 0; i < n; i++) {
        if (inDegree[i] == 0) queue.offer(i);
    }

    List<Integer> order = new ArrayList<>();
    while (!queue.isEmpty()) {
        int node = queue.poll();
        order.add(node);
        for (int neighbor : graph.get(node)) {
            if (--inDegree[neighbor] == 0) queue.offer(neighbor);
        }
    }
    // If order.size() != n, there's a cycle (not a DAG)
    return order.size() == n ? order : new ArrayList<>();
}
```

### Method 2: DFS Postorder
```java
// Postorder = reverse topological order
// Add node to result AFTER all its descendants are processed
Deque<Integer> stack = new ArrayDeque<>();
boolean[] visited = new boolean[n];

void dfsTopoSort(List<List<Integer>> graph, int node) {
    visited[node] = true;
    for (int neighbor : graph.get(node)) {
        if (!visited[neighbor]) dfsTopoSort(graph, neighbor);
    }
    stack.push(node);   // add AFTER all neighbors are processed
}
// Result: stack.pop() repeatedly
```

### Variation 1: Course Schedule (LC 207) — Can you finish?

**Problem:** `numCourses` courses labeled `0..n-1`. `prerequisites[i] = [a, b]` means you must take `b` before `a`. Return `true` if you can finish all courses.

**Approach (Kahn's algorithm = cycle check on DAG):**
- Model as a directed graph: edge `b → a` for each `[a, b]`.
- The schedule is feasible iff the graph is a DAG (no cycle).
- Run Kahn's: process nodes with `in-degree = 0`, decrement their neighbors' in-degree, repeat.
- If we process all `n` nodes → no cycle → feasible. Otherwise some nodes remained "stuck" in a cycle.

```java
// Just check if topological sort is possible (no cycle)
boolean canFinish(int numCourses, int[][] prerequisites) {
    List<List<Integer>> graph = new ArrayList<>();
    for (int i = 0; i < numCourses; i++) graph.add(new ArrayList<>());
    int[] inDegree = new int[numCourses];
    for (int[] pre : prerequisites) {
        graph.get(pre[1]).add(pre[0]);
        inDegree[pre[0]]++;
    }
    Queue<Integer> q = new LinkedList<>();
    for (int i = 0; i < numCourses; i++) if (inDegree[i] == 0) q.offer(i);
    int count = 0;
    while (!q.isEmpty()) {
        int course = q.poll(); count++;
        for (int next : graph.get(course)) if (--inDegree[next] == 0) q.offer(next);
    }
    return count == numCourses;
}
```

### Variation 2: Course Schedule II (LC 210) — Return the order

**Problem:** Same setup as LC 207, but instead of true/false, return **any valid course order** (empty array if impossible).

**Approach:**
- Run Kahn's algorithm and append nodes to a result list as they're processed.
- If the final result has fewer than `n` nodes → cycle → return empty.
- The order Kahn's produces is one valid topological order (multiple may exist when in-degrees tie).

### Variation 3: Alien Dictionary (LC 269) — Hard

**Problem:** A list of words from an alien language is given in their dictionary order. Derive the order of letters in the alien alphabet. Return any valid order, or `""` if impossible.

**Approach (Build graph from adjacent pairs + topo sort):**
- Letters = nodes. To find edges, compare every adjacent pair `(words[i], words[i+1])` letter by letter.
- The **first differing letter** tells us `a comes before b` → add directed edge `a → b`. Stop comparing that pair (later letters give no info).
- Edge case: if `words[i]` is a longer string and `words[i+1]` is its prefix (e.g., `"abc"` then `"ab"`), the input is invalid → return `""`.
- Run Kahn's. If processed count < total distinct letters → cycle (contradiction) → return `""`.

```java
String alienOrder(String[] words) {
    Map<Character, Set<Character>> graph = new HashMap<>();
    Map<Character, Integer> inDeg = new HashMap<>();
    for (String w : words) for (char c : w.toCharArray()) inDeg.putIfAbsent(c, 0);

    for (int i = 0; i + 1 < words.length; i++) {
        String a = words[i], b = words[i + 1];
        if (a.length() > b.length() && a.startsWith(b)) return "";  // invalid
        for (int j = 0; j < Math.min(a.length(), b.length()); j++) {
            char x = a.charAt(j), y = b.charAt(j);
            if (x != y) {
                graph.putIfAbsent(x, new HashSet<>());
                if (graph.get(x).add(y)) inDeg.merge(y, 1, Integer::sum);
                break;
            }
        }
    }

    Queue<Character> q = new LinkedList<>();
    for (var e : inDeg.entrySet()) if (e.getValue() == 0) q.offer(e.getKey());
    StringBuilder sb = new StringBuilder();
    while (!q.isEmpty()) {
        char c = q.poll(); sb.append(c);
        if (graph.containsKey(c))
            for (char nxt : graph.get(c))
                if (inDeg.merge(nxt, -1, Integer::sum) == 0) q.offer(nxt);
    }
    return sb.length() == inDeg.size() ? sb.toString() : "";
}
```

### Variation 4: Find Eventual Safe States (LC 802)

**Problem:** A directed graph. A node is **safe** if every path starting from it leads to a terminal node (out-degree 0). Return all safe nodes in ascending order. Practically: a node is safe iff it's **not on or reachable to a cycle**.

**Approach (Reverse topo sort / 3-color DFS):**
- Method A — 3-color DFS:
  - Run DFS; mark nodes as `0` (unvisited), `1` (on current path), `2` (safe).
  - If a DFS hits a `1`-node → cycle on this path → current node is **not safe**.
  - If all DFS branches complete with `2`s only → mark current node `2`.
- Method B — reverse Kahn's:
  - Reverse all edges. Terminal nodes (original out-deg 0) become sources in the reversed graph.
  - Run Kahn's on the reverse graph; every node processed is safe.

```java
List<Integer> eventualSafeNodes(int[][] graph) {
    int n = graph.length;
    int[] state = new int[n];  // 0/1/2
    List<Integer> res = new ArrayList<>();
    for (int i = 0; i < n; i++) if (dfsSafe(graph, i, state)) res.add(i);
    return res;
}
boolean dfsSafe(int[][] g, int u, int[] state) {
    if (state[u] != 0) return state[u] == 2;
    state[u] = 1;
    for (int v : g[u]) if (!dfsSafe(g, v, state)) return false;
    state[u] = 2;
    return true;
}
```

### Topological Sort Questions

| # | Problem | LC# | Difficulty |
|---|---------|-----|------------|
| M1 | Course Schedule | 207 | M |
| M2 | Course Schedule II | 210 | M |
| M3 | Find Eventual Safe States | 802 | M |
| M4 | Minimum Height Trees | 310 | M |
| H1 | Alien Dictionary | 269 | H |
| H2 | Sequence Reconstruction | 444 | M |
| H3 | Parallel Courses III | 2050 | H |

---

## P4: Union-Find (Disjoint Set Union — DSU) ★★★

### Core Idea
> Group elements into sets. Supports two operations:
> - `find(x)`: which set does x belong to?
> - `union(x, y)`: merge the sets of x and y.

### Template (with Path Compression + Union by Rank)
```java
class DSU {
    int[] parent, rank;

    DSU(int n) {
        parent = new int[n];
        rank = new int[n];
        for (int i = 0; i < n; i++) parent[i] = i;
    }

    int find(int x) {
        if (parent[x] != x) parent[x] = find(parent[x]);  // path compression
        return parent[x];
    }

    boolean union(int x, int y) {
        int px = find(x), py = find(y);
        if (px == py) return false;   // already in same set (cycle!)
        if (rank[px] < rank[py]) { int tmp = px; px = py; py = tmp; }
        parent[py] = px;
        if (rank[px] == rank[py]) rank[px]++;
        return true;
    }

    boolean connected(int x, int y) { return find(x) == find(y); }
}
```
> **Path Compression:** Make every node on the path point directly to root → near O(1).
> **Union by Rank:** Always attach smaller tree under larger → balanced.

### Variation 1: Redundant Connection (LC 684) — Find the cycle edge

**Problem:** A tree on `n` nodes has exactly one extra edge added, making one cycle. Edges are given in order. Return the **last** edge that, if removed, makes the graph a tree again.

**Approach (DSU — process edges in order):**
- Iterate edges left to right. For each edge `[u, v]`, try `union(u, v)`.
- If `u` and `v` are **already in the same set**, this edge closes a cycle → this is the redundant edge. Return it.
- Because we process in order, this is automatically the "last" such edge — exactly what the problem asks.

```java
// For each edge, if union returns false → that edge creates a cycle
int[] findRedundantConnection(int[][] edges) {
    DSU dsu = new DSU(edges.length + 1);
    for (int[] edge : edges) {
        if (!dsu.union(edge[0], edge[1])) return edge;
    }
    return null;
}
```

### Variation 2: Number of Connected Components (LC 323)

**Problem:** Given `n` nodes labeled `0..n-1` and an undirected edge list, count the number of connected components.

**Approach (DSU):**
- Start assuming all `n` nodes are isolated → `components = n`.
- For each edge, `union(u, v)`. Every successful merge reduces the count by 1 (two trees become one).
- Already-connected edges (same root) don't change the count.
- Alternative: BFS/DFS — iterate nodes, run a search from each unvisited node, count the kicks.

```java
int countComponents(int n, int[][] edges) {
    DSU dsu = new DSU(n);
    int components = n;
    for (int[] edge : edges) {
        if (dsu.union(edge[0], edge[1])) components--;
    }
    return components;
}
```

### Variation 3: Accounts Merge (LC 721)

**Problem:** Each account is `[name, email1, email2, ...]`. Two accounts belong to the same person if they share **any** email. Merge all accounts of the same person and return them as `[name, sortedEmails...]`.

**Approach (DSU on emails):**
- Treat each email as a node. For each account, **union the first email with each other email** in that account. Transitively all emails of one person end up in one set.
- Also keep `emailToName` for output formatting.
- After unions, group emails by their DSU root. Sort each group, prepend the name.
- Indexing: map each unique email to an integer id so we can use array-based DSU.

```java
List<List<String>> accountsMerge(List<List<String>> accounts) {
    Map<String, Integer> emailId = new HashMap<>();
    Map<String, String> emailName = new HashMap<>();
    for (List<String> acc : accounts) {
        String name = acc.get(0);
        for (int i = 1; i < acc.size(); i++) {
            emailId.putIfAbsent(acc.get(i), emailId.size());
            emailName.put(acc.get(i), name);
        }
    }
    DSU dsu = new DSU(emailId.size());
    for (List<String> acc : accounts) {
        int first = emailId.get(acc.get(1));
        for (int i = 2; i < acc.size(); i++) dsu.union(first, emailId.get(acc.get(i)));
    }
    Map<Integer, List<String>> groups = new HashMap<>();
    for (var e : emailId.entrySet())
        groups.computeIfAbsent(dsu.find(e.getValue()), k -> new ArrayList<>()).add(e.getKey());
    List<List<String>> res = new ArrayList<>();
    for (List<String> emails : groups.values()) {
        Collections.sort(emails);
        List<String> entry = new ArrayList<>();
        entry.add(emailName.get(emails.get(0)));
        entry.addAll(emails);
        res.add(entry);
    }
    return res;
}
```

### Variation 4: Making a Large Island (LC 827) — Hard

**Problem:** Binary grid of 0s and 1s. You may flip **at most one** 0 to 1. Return the maximum island size achievable (4-connected).

**Approach (DSU / island labeling + boundary scan):**
- Pass 1: label every island with a unique id (e.g., 2, 3, 4, ...) and record `size[id]`. Use DFS/BFS or DSU.
- Pass 2: for each `0`-cell, look at its 4 neighbors, collect the **distinct island ids** around it, and the candidate size is `1 + sum of those islands' sizes`.
- Using a **Set** when collecting neighbor ids matters: two neighbors might belong to the same island — counting it twice would over-count.
- Edge case: if the grid is all 1s, the answer is `m × n` (no zero to flip — but with "at most one flip", current island already is the max).
- Time: O(m·n).

```java
int largestIsland(int[][] grid) {
    int n = grid.length, id = 2;
    Map<Integer, Integer> size = new HashMap<>();
    int[][] dirs = {{1,0},{-1,0},{0,1},{0,-1}};
    for (int i = 0; i < n; i++)
        for (int j = 0; j < n; j++)
            if (grid[i][j] == 1) size.put(id, paint(grid, i, j, id++));

    int best = size.values().stream().max(Integer::compare).orElse(0);
    for (int i = 0; i < n; i++)
        for (int j = 0; j < n; j++) if (grid[i][j] == 0) {
            Set<Integer> seen = new HashSet<>();
            for (int[] d : dirs) {
                int r = i + d[0], c = j + d[1];
                if (r >= 0 && r < n && c >= 0 && c < n && grid[r][c] > 1) seen.add(grid[r][c]);
            }
            int s = 1;
            for (int k : seen) s += size.get(k);
            best = Math.max(best, s);
        }
    return best;
}
int paint(int[][] g, int i, int j, int id) {
    if (i < 0 || i >= g.length || j < 0 || j >= g.length || g[i][j] != 1) return 0;
    g[i][j] = id;
    return 1 + paint(g,i+1,j,id) + paint(g,i-1,j,id) + paint(g,i,j+1,id) + paint(g,i,j-1,id);
}
```

### DSU Questions

| # | Problem | LC# | Difficulty |
|---|---------|-----|------------|
| M1 | Number of Provinces | 547 | M |
| M2 | Redundant Connection | 684 | M |
| M3 | Number of Connected Components | 323 | M |
| M4 | Accounts Merge | 721 | M |
| M5 | Satisfiability of Equality Equations | 990 | M |
| H1 | Making a Large Island | 827 | H |
| H2 | Swim in Rising Water | 778 | H |

---

## P5: Dijkstra's Algorithm ★★★

### When To Use
> Shortest path in a weighted graph where ALL edge weights are NON-NEGATIVE.

### Template
```java
int[] dijkstra(int[][] graph, int src) {  // graph as adjacency list of [neighbor, weight]
    int n = graph.length;
    int[] dist = new int[n];
    Arrays.fill(dist, Integer.MAX_VALUE);
    dist[src] = 0;

    PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> a[0] - b[0]);
    pq.offer(new int[]{0, src});   // {distance, node}

    while (!pq.isEmpty()) {
        int[] curr = pq.poll();
        int d = curr[0], node = curr[1];
        if (d > dist[node]) continue;   // stale entry — skip

        for (int[] edge : graph[node]) {
            int next = edge[0], weight = edge[1];
            if (dist[node] + weight < dist[next]) {
                dist[next] = dist[node] + weight;
                pq.offer(new int[]{dist[next], next});
            }
        }
    }
    return dist;
}
```
> Time: O((V + E) log V)

### Variation 1: Network Delay Time (LC 743)

**Problem:** A network with `n` nodes. `times[i] = [u, v, w]` is a directed edge with travel time `w`. A signal sent from node `k` reaches all reachable nodes; return the time for **all** to receive it, or `-1` if some are unreachable.

**Approach:**
- We need the max of all shortest distances from `k`.
- Run Dijkstra from `k`. After it terminates, scan `dist[]`:
  - If any node still has `INF` → return `-1`.
  - Else return `max(dist[i])`.

### Variation 2: Cheapest Flights Within K Stops (LC 787) — Modified Dijkstra / Bellman-Ford

**Problem:** Cities are `0..n-1`. `flights[i] = [from, to, price]`. Find the cheapest price from `src` to `dst` using **at most `k` stops** (so at most `k+1` edges). Return `-1` if impossible.

**Approach:**
- Plain Dijkstra fails because a "cheap but long" path might exhaust the stops budget and lock out a "pricier but shorter" path that's actually valid.
- Two clean options:
  - **Bellman-Ford-style relaxation** (shown below): run `k+1` rounds, each round relaxes every edge using the previous round's snapshot. The snapshot prevents using two edges within the same round.
  - **Modified Dijkstra with stops state**: PQ entries are `{cost, node, stopsUsed}`; only enqueue when `stopsUsed ≤ k`. Track `bestStops[node]` to prune dominated states.
- Time: `O(k·E)` for the BF approach.

```java
// State = {cost, node, stops_remaining}
// Use BFS (not Dijkstra) to explore level by level (stops = levels)
int findCheapestPrice(int n, int[][] flights, int src, int dst, int k) {
    int[] cost = new int[n];
    Arrays.fill(cost, Integer.MAX_VALUE);
    cost[src] = 0;

    for (int i = 0; i <= k; i++) {   // k stops = k+1 edges = k+1 BFS rounds
        int[] tmp = cost.clone();
        for (int[] flight : flights) {
            int from = flight[0], to = flight[1], price = flight[2];
            if (cost[from] != Integer.MAX_VALUE && cost[from] + price < tmp[to]) {
                tmp[to] = cost[from] + price;
            }
        }
        cost = tmp;
    }
    return cost[dst] == Integer.MAX_VALUE ? -1 : cost[dst];
}
```

### Variation 3: Path With Maximum Probability (LC 1514)

**Problem:** Undirected graph; each edge has a **success probability** in `[0,1]`. Find the path from `start` to `end` that maximizes the **product** of edge probabilities.

**Approach (Dijkstra with max-heap):**
- Multiplying probabilities along a path → we want to **maximize**, not minimize. Flip the priority queue ordering: use a **max-heap on probability**.
- Track `prob[node]` (best probability to reach `node`). Relax: `prob[v] = max(prob[v], prob[u] * edge.p)`.
- Why Dijkstra still works: probabilities are in `[0,1]`, so multiplying by an edge only **decreases** the value — analogous to "distances never decrease" in regular Dijkstra (just monotone in the other direction).
- Alternative (avoids floating-point underflow on long paths): take `-log(p)` and run regular shortest-path Dijkstra on positive weights.

```java
double maxProbability(int n, int[][] edges, double[] succ, int start, int end) {
    List<double[]>[] g = new List[n];
    for (int i = 0; i < n; i++) g[i] = new ArrayList<>();
    for (int i = 0; i < edges.length; i++) {
        g[edges[i][0]].add(new double[]{edges[i][1], succ[i]});
        g[edges[i][1]].add(new double[]{edges[i][0], succ[i]});
    }
    double[] prob = new double[n];
    prob[start] = 1.0;
    PriorityQueue<double[]> pq = new PriorityQueue<>((a, b) -> Double.compare(b[1], a[1]));
    pq.offer(new double[]{start, 1.0});
    while (!pq.isEmpty()) {
        double[] cur = pq.poll();
        int u = (int) cur[0];
        if (u == end) return cur[1];
        if (cur[1] < prob[u]) continue;
        for (double[] e : g[u]) {
            int v = (int) e[0];
            double np = cur[1] * e[1];
            if (np > prob[v]) { prob[v] = np; pq.offer(new double[]{v, np}); }
        }
    }
    return 0.0;
}
```

### Variation 4: Dijkstra on Grid — Swim in Rising Water (LC 778)

**Problem (LC 778):** `n×n` grid; `grid[i][j]` is the elevation. At time `t`, water level is `t`, and you can move to a 4-adjacent cell only if both its elevation ≤ `t`. Return the **earliest time** you can reach `(n-1, n-1)` from `(0, 0)`.

**Approach (Dijkstra on grid, "minimax path"):**
- Reframe: the time to reach a cell = **maximum elevation along any path used to get there**. We want the path that **minimizes that maximum** (a "minimax path").
- Run Dijkstra-like search with PQ ordered by current path's max-elevation. Each cell = node; edges = 4-adjacency; "edge weight" = `max(neighbor elevation, current best)`.
- First time we pop the destination, that pop's priority is the answer.
- Alternative: DSU + sort cells by elevation, "open" them one at a time, stop once `(0,0)` and `(n-1,n-1)` are in the same set.

```java
int swimInWater(int[][] grid) {
    int n = grid.length;
    int[][] dirs = {{1,0},{-1,0},{0,1},{0,-1}};
    boolean[][] seen = new boolean[n][n];
    PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> a[0] - b[0]);  // {time, r, c}
    pq.offer(new int[]{grid[0][0], 0, 0});
    while (!pq.isEmpty()) {
        int[] cur = pq.poll();
        int t = cur[0], r = cur[1], c = cur[2];
        if (r == n - 1 && c == n - 1) return t;
        if (seen[r][c]) continue;
        seen[r][c] = true;
        for (int[] d : dirs) {
            int nr = r + d[0], nc = c + d[1];
            if (nr >= 0 && nr < n && nc >= 0 && nc < n && !seen[nr][nc])
                pq.offer(new int[]{Math.max(t, grid[nr][nc]), nr, nc});
        }
    }
    return -1;
}
```

### Dijkstra Questions

| # | Problem | LC# | Difficulty |
|---|---------|-----|------------|
| M1 | Network Delay Time | 743 | M |
| M2 | Path With Maximum Probability | 1514 | M |
| M3 | Minimum Cost to Reach Destination | — | M |
| H1 | Cheapest Flights Within K Stops | 787 | M |
| H2 | Swim in Rising Water | 778 | H |
| H3 | Find the City With Smallest Reachable Neighbors | 1334 | M |

---

## P6: Bellman-Ford

### When To Use
> Shortest path with NEGATIVE edge weights. Also detects negative cycles.
> Slower than Dijkstra: O(V × E).

**Problem (canonical):** Single-source shortest paths in a directed graph that **may have negative edges** (e.g., currency arbitrage, time-travel-style problems). Also: detect whether a **negative cycle** is reachable from the source — that would make the shortest path undefined (you can loop forever, reducing cost).

**Approach:**
- **Relaxation property:** if `dist[u] + w(u,v) < dist[v]`, the path through `u` is shorter — update `dist[v]`.
- A shortest path uses at most `V-1` edges (otherwise it repeats a vertex and could be shortened). So `V-1` rounds of "relax every edge" are enough to converge.
- After `V-1` rounds, do one more pass: if **any** edge can still be relaxed, that edge participates in a negative cycle.
- Why Dijkstra can't do this: Dijkstra commits a node's distance the moment it pops from the PQ. A later, very negative edge could invalidate that commitment.

```java
int[] bellmanFord(int n, int[][] edges, int src) {
    int[] dist = new int[n];
    Arrays.fill(dist, Integer.MAX_VALUE);
    dist[src] = 0;

    for (int i = 0; i < n - 1; i++) {    // relax all edges n-1 times
        for (int[] edge : edges) {
            int u = edge[0], v = edge[1], w = edge[2];
            if (dist[u] != Integer.MAX_VALUE && dist[u] + w < dist[v]) {
                dist[v] = dist[u] + w;
            }
        }
    }
    // Check for negative cycle: if any edge can still be relaxed, negative cycle exists
    for (int[] edge : edges) {
        if (dist[edge[0]] != Integer.MAX_VALUE && dist[edge[0]] + edge[2] < dist[edge[1]])
            return null;   // negative cycle
    }
    return dist;
}
```

---

## P7: Floyd-Warshall

### When To Use
> All-pairs shortest paths. O(V³). Use when V is small (≤ 100).

**Problem (canonical — LC 1334):** Given `n` cities and weighted edges with a distance threshold `d`, find the **city with the fewest reachable neighbors** within distance `d`. Tie → return the one with the largest index.

**Approach (Floyd-Warshall):**
- We want shortest distance between **every** pair `(i, j)` — Dijkstra from every node is `O(V·E log V)`, but for small V the cleaner DP is Floyd-Warshall.
- DP recurrence: `dist[i][j] = min(dist[i][j], dist[i][k] + dist[k][j])`.
- Interpretation of `k`: "what if we *also allow* node `k` as an intermediate?" After we've iterated `k` over all nodes, every possible intermediate has been considered.
- **Loop order matters**: `k` must be the outermost loop. Inverting it gives wrong answers.
- For LC 1334: after computing `dist`, for each city count `j` with `dist[i][j] ≤ d`; pick the minimum count (largest index on tie).

```java
void floydWarshall(int[][] dist) {
    int n = dist.length;
    for (int k = 0; k < n; k++) {          // k = intermediate node
        for (int i = 0; i < n; i++) {
            for (int j = 0; j < n; j++) {
                if (dist[i][k] != INF && dist[k][j] != INF)
                    dist[i][j] = Math.min(dist[i][j], dist[i][k] + dist[k][j]);
            }
        }
    }
}
```
> LC 1334: Find the City With Smallest Reachable Neighbors → Floyd-Warshall.

---

## P8: Minimum Spanning Tree (MST)

**Problem (canonical — LC 1584 "Min Cost to Connect All Points"):** Given `n` points in the plane, the cost to connect two is the Manhattan distance between them. Return the minimum total cost to connect **all** points so that any two are reachable.

**Approach (MST):**
- An MST is a subset of edges that connects every vertex with **minimum total weight** and **no cycles** (exactly `V-1` edges).
- Two equally famous greedy algorithms — both rely on the **cut property**: the minimum-weight edge crossing any cut is safe to include in some MST.
  - **Kruskal:** sort all edges by weight; add edge if its two endpoints are in different DSU sets. O(E log E).
  - **Prim:** grow the MST one vertex at a time from an arbitrary start; at each step add the cheapest edge that crosses out of the current tree. O((V+E) log V) with a heap.
- Pick Kruskal when edges are easy to enumerate / graph is sparse; pick Prim when the graph is dense or implicit (LC 1584 uses Prim well — every pair is an edge).

### Kruskal's Algorithm (Sort edges + DSU)
```java
// Sort edges by weight, add if it doesn't create a cycle (DSU)
int kruskal(int n, int[][] edges) {
    Arrays.sort(edges, (a, b) -> a[2] - b[2]);   // sort by weight
    DSU dsu = new DSU(n);
    int totalCost = 0;
    for (int[] edge : edges) {
        if (dsu.union(edge[0], edge[1])) {   // no cycle
            totalCost += edge[2];
        }
    }
    return totalCost;
}
```

### Prim's Algorithm (Greedy + Min-Heap)
```java
// Start from any node, greedily add cheapest edge crossing the cut
int prim(int n, List<List<int[]>> graph) {   // graph[u] = {v, weight}
    boolean[] inMST = new boolean[n];
    PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> a[0] - b[0]);
    pq.offer(new int[]{0, 0});   // {weight, node}
    int totalCost = 0;

    while (!pq.isEmpty()) {
        int[] curr = pq.poll();
        int w = curr[0], node = curr[1];
        if (inMST[node]) continue;
        inMST[node] = true;
        totalCost += w;
        for (int[] edge : graph.get(node)) {
            if (!inMST[edge[0]]) pq.offer(new int[]{edge[1], edge[0]});
        }
    }
    return totalCost;
}
```

### MST Questions

| # | Problem | LC# | Difficulty |
|---|---------|-----|------------|
| M1 | Min Cost to Connect All Points | 1584 | M |
| H1 | Optimize Water Distribution | 1168 | H |

---

## P9: Advanced Graph Algorithms

### Bridges (Critical Connections) — LC 1192

**Problem:** A network has `n` servers and undirected connections. A **critical connection** is one whose removal disconnects the network. Return all critical connections (bridges).

**Approach (Tarjan's bridge-finding):**
- Run DFS, numbering nodes with a discovery time `disc[u]` as they're first visited.
- Track `low[u]` = the smallest `disc` reachable from `u`'s DFS subtree using **at most one back edge** (an edge to an ancestor).
- After exploring child `v` of `u`: `low[u] = min(low[u], low[v])`.
- **Bridge test:** edge `(u, v)` (where `v` is a child of `u` in DFS) is a bridge iff `low[v] > disc[u]`. Intuition: the subtree rooted at `v` has *no* back edge that climbs above `u`; so removing `(u, v)` disconnects that subtree.
- Important: when comparing against the parent, skip only **the parent edge once** (handle multi-edges if the input allows them).
- Time: O(V + E).

```java
// Tarjan's algorithm: an edge (u,v) is a bridge if disc[v] > low[u] after DFS
// low[u] = min discovery time reachable from subtree of u
int[] disc, low;
boolean[] visited;
int timer = 0;
List<List<Integer>> bridges;

void dfs(int node, int parent, List<List<Integer>> graph) {
    visited[node] = true;
    disc[node] = low[node] = timer++;
    for (int neighbor : graph.get(node)) {
        if (!visited[neighbor]) {
            dfs(neighbor, node, graph);
            low[node] = Math.min(low[node], low[neighbor]);
            if (low[neighbor] > disc[node])
                bridges.add(Arrays.asList(node, neighbor));
        } else if (neighbor != parent) {
            low[node] = Math.min(low[node], disc[neighbor]);
        }
    }
}
```

### Bipartite Check (LC 785)

**Problem:** Given an undirected graph, return `true` if it is bipartite — i.e., its vertices can be split into two sets such that every edge has one endpoint in each set. Equivalent: graph is **2-colorable**.

**Approach (BFS 2-coloring):**
- A graph is bipartite iff it has **no odd-length cycle**.
- BFS (or DFS) from every uncolored node. Color the source `1`, then color each neighbor with the **opposite** color (`-color`).
- If a neighbor is already colored the **same** as the current node → odd cycle → not bipartite.
- Loop over all components — graph may be disconnected.
- Time: O(V + E).

```java
// 2-colorable? BFS/DFS, assign colors, conflict = not bipartite
boolean isBipartite(int[][] graph) {
    int n = graph.length;
    int[] color = new int[n];  // 0=uncolored, 1 or -1
    for (int i = 0; i < n; i++) {
        if (color[i] != 0) continue;
        Queue<Integer> q = new LinkedList<>();
        q.offer(i); color[i] = 1;
        while (!q.isEmpty()) {
            int node = q.poll();
            for (int neighbor : graph[node]) {
                if (color[neighbor] == 0) {
                    color[neighbor] = -color[node];
                    q.offer(neighbor);
                } else if (color[neighbor] == color[node]) return false;
            }
        }
    }
    return true;
}
```

---

## Complete FAANG Graph Problem List

| Problem | Pattern | LC# | Difficulty |
|---------|---------|-----|------------|
| Number of Islands | DFS/BFS | 200 | M |
| Clone Graph | DFS | 133 | M |
| Course Schedule | Topo Sort | 207 | M |
| Course Schedule II | Topo Sort | 210 | M |
| Number of Provinces | DSU/DFS | 547 | M |
| Rotting Oranges | Multi-source BFS | 994 | M |
| Pacific Atlantic Water Flow | Multi-source DFS | 417 | M |
| Is Graph Bipartite? | BFS Coloring | 785 | M |
| Find Eventual Safe States | DFS State | 802 | M |
| Redundant Connection | DSU | 684 | M |
| Accounts Merge | DSU | 721 | M |
| Word Ladder | BFS Implicit Graph | 127 | H |
| Network Delay Time | Dijkstra | 743 | M |
| Cheapest Flights K Stops | Bellman-Ford | 787 | M |
| Min Cost to Connect All Points | Kruskal/Prim MST | 1584 | M |
| Path with Maximum Probability | Dijkstra | 1514 | M |
| Critical Connections | Tarjan Bridges | 1192 | H |
| Swim in Rising Water | Dijkstra/DSU | 778 | H |
| Making a Large Island | DSU | 827 | H |
| Alien Dictionary | Topo Sort | 269 | H |

---

## Graph Algorithm Selection Table

```
Problem type                              → Algorithm
──────────────────────────────────────────────────────────────────
Shortest path, unweighted                 → BFS (O(V+E))
Shortest path, weighted, non-negative     → Dijkstra (O((V+E) log V))
Shortest path, negative weights           → Bellman-Ford (O(VE))
All-pairs shortest path                   → Floyd-Warshall (O(V³))
Ordering with dependencies                → Topological Sort (Kahn's)
Minimum spanning tree                     → Kruskal (sort + DSU) or Prim
Connected components                      → DSU or BFS/DFS
Cycle detection (undirected)              → DSU or DFS with parent
Cycle detection (directed)                → DFS with 3-state coloring
Bridge / articulation points              → Tarjan's (DFS + low link)
Bipartite check                           → BFS 2-coloring
Multiple starting sources                 → Multi-source BFS
0/1 edge weights                          → 0-1 BFS (deque)
```

---

> Back to [Master Index](../DSA_MASTER_INDEX.md)
