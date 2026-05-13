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

| # | Problem | LC# | Difficulty |
|---|---------|-----|------------|
| M1 | Number of Islands | 200 | M |
| M2 | Rotting Oranges | 994 | M |
| M3 | Walls and Gates | 286 | M |
| M4 | Pacific Atlantic Water Flow | 417 | M |
| M5 | Word Ladder | 127 | H |
| M6 | Shortest Path in Binary Matrix | 1091 | M |
| H1 | Minimum Jumps to Reach End | 45 | M |
| H2 | Sliding Puzzle | 773 | H |

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
> Same as Kahn's algorithm — return the `order` list.

### Variation 3: Alien Dictionary (LC 269) — Hard
```java
// Build order from given word list (adjacency based on first differing char)
// Then run topological sort
// Edge case: "ab" before "a" means invalid (longer word before prefix → cycle)
```

### Variation 4: Find Eventual Safe States (LC 802)
> A node is "safe" if it's not on any cycle.
> Using state array (0/1/2) from cycle detection: safe = state 2.

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
```java
// Union all emails belonging to same account
// Then group emails by their root
```

### Variation 4: Making a Large Island (LC 827) — Hard
```java
// Precompute island sizes with DSU
// Then for each 0-cell, check which islands it can connect
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
> Dijkstra from source, return max distance.

### Variation 2: Cheapest Flights Within K Stops (LC 787) — Modified Dijkstra
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
> Maximize probability instead of minimizing distance → max-heap.

### Variation 4: Dijkstra on Grid
```java
// Each cell = node, edge = movement cost
// Use PQ {cost, row, col}
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
