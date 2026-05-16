# Graph Theory — Fundamentals & Terminology
> Read this BEFORE the patterns file (08_graph_patterns.md)
> Everything you need to know conceptually to understand any graph problem.

---

## What Is a Graph?

A graph is a collection of **nodes (vertices)** connected by **edges**.

```
           (1)
          /   \
        (3)   (2)
          \   /
           (4)
             \
             (5)

  Nodes  (circles) : {1, 2, 3, 4, 5}
  Edges  (lines)   : {(1,2), (1,3), (2,4), (3,4), (4,5)}
```

Formal definition: **G = (V, E)** where V = set of vertices, E = set of edges.

---

## 1. Directed vs Undirected

### Undirected Graph
> Edges have **no direction**. You can travel both ways on every edge.

```
  ┌─────────────────────────────┐
  │   UNDIRECTED GRAPH          │
  │                             │
  │    (A)────(B)────(C)        │
  │     │              │        │
  │     └──────(D)─────┘        │
  │                             │
  │   Edge (A,B) means:         │
  │   A → B  ✓                  │
  │   B → A  ✓  (both work)     │
  └─────────────────────────────┘
```

- Real-world examples: **Facebook friendships**, road networks (two-way roads), power grid
- Adding an edge: both directions must be added in code

```java
graph.get(u).add(v);
graph.get(v).add(u);   // must add BOTH
```

---

### Directed Graph (Digraph)
> Edges have a **direction** (arrow). An edge A→B does NOT imply B→A.

```
  ┌──────────────────────────────────────┐
  │   DIRECTED GRAPH (DIGRAPH)           │
  │                                      │
  │    (A) ──→ (B) ──→ (C)              │
  │     ↑                │               │
  │     └────── (D) ←────┘               │
  │                                      │
  │   Edge A→B:  A can go to B  ✓        │
  │              B can go to A  ✗        │
  └──────────────────────────────────────┘
```

- Real-world examples: **Twitter follows**, course prerequisites (A before B), web page links, one-way streets
- Adding an edge: only one direction added

```java
graph.get(u).add(v);   // only u → v
```

---

### Side-by-Side Comparison

```
    UNDIRECTED                    DIRECTED
    ──────────                    ────────
       (1)                           (1)
      /   \                         ↗   ↘
    (2)   (3)                     (2)   (3)
      \   /                         ↘   ↗
       (4)                           (4)

  Any node can reach              Must follow arrows.
  any neighbor freely.            (4) can NOT go to (1).
```

### Key Algorithmic Differences

| Property | Undirected | Directed |
|----------|-----------|---------|
| Cycle detection | DFS + track parent | DFS + 3-colour state |
| Components | Connected Components | Strongly Connected Components (SCC) |
| Topological Sort | Not applicable | Valid only on DAG |
| Edge count | Max n(n-1)/2 | Max n(n-1) |

---

## 2. Weighted vs Unweighted

### Unweighted Graph
> All edges are equal — just "connected" or "not connected".
> BFS gives the shortest path (minimum number of hops).

```
  ┌───────────────────────────────────┐
  │   UNWEIGHTED GRAPH                │
  │                                   │
  │   (A)────(B)────(D)               │
  │    │                              │
  │   (C)                             │
  │                                   │
  │   A→B→D = 2 hops  ← shortest     │
  │   A→C    = 1 hop                  │
  └───────────────────────────────────┘
```

---

### Weighted Graph
> Each edge carries a **cost** (distance, time, price, etc.)
> Minimum hops ≠ minimum cost. Need Dijkstra or Bellman-Ford.

```
  ┌──────────────────────────────────────────┐
  │   WEIGHTED GRAPH                         │
  │                                          │
  │        5           2                     │
  │   (A)──────(B)──────(D)                  │
  │    │                 │                   │
  │    │ 3               │ 1                 │
  │    │                 │                   │
  │   (C)────────────────┘                   │
  │                                          │
  │   Path A→B→D  cost = 5+2 = 7             │
  │   Path A→C→D  cost = 3+1 = 4  ← cheaper │
  │                                          │
  │   Hops say A→B→D wins (2 hops).          │
  │   Weight says A→C→D wins (cost 4).       │
  └──────────────────────────────────────────┘
```

```java
// Weighted adjacency list: store {neighbor, weight} pairs
List<List<int[]>> graph = new ArrayList<>();
graph.get(u).add(new int[]{v, weight});
```

---

### Negative Weights

```
  ┌──────────────────────────────────────────────┐
  │   NEGATIVE WEIGHT GRAPH                      │
  │                                              │
  │        2           -3                        │
  │   (A)──────(B)──────(C)                      │
  │    │                                         │
  │    │ 5                                       │
  │    │                                         │
  │   (D)                                        │
  │                                              │
  │   A→B→C  cost = 2 + (-3) = -1               │
  │   Dijkstra FAILS with negative edges!        │
  │   Use Bellman-Ford instead.                  │
  └──────────────────────────────────────────────┘
```

### Algorithm Selection by Weight Type

```
  Edge Weight Type              →  Algorithm to Use
  ────────────────────────────────────────────────────
  All edges weight = 1          →  BFS             O(V+E)
  All weights ≥ 0               →  Dijkstra        O((V+E) log V)
  Weights 0 or 1 only           →  0-1 BFS (deque) O(V+E)
  Negative weights (no neg cycle) → Bellman-Ford   O(V×E)
  All-pairs, V ≤ 100            →  Floyd-Warshall  O(V³)
```

---

## 3. Core Terminology

### Degree

```
  ┌────────────────────────────────────────────────────┐
  │   DEGREE = number of edges touching a node         │
  │                                                    │
  │      (1)────(2)────(3)                             │
  │              │                                     │
  │             (4)                                    │
  │                                                    │
  │   degree(1) = 1  ──────────── touches 1 edge       │
  │   degree(2) = 3  ──────────── touches 3 edges      │
  │   degree(3) = 1  ──────────── touches 1 edge       │
  │   degree(4) = 1  ──────────── touches 1 edge       │
  │                                                    │
  │   Sum of all degrees = 2 × |E|  (handshake lemma)  │
  └────────────────────────────────────────────────────┘
```

---

### In-Degree and Out-Degree (Directed Only)

```
  ┌──────────────────────────────────────────────────────────────┐
  │   IN-DEGREE  = edges arriving   (pointing IN to node)        │
  │   OUT-DEGREE = edges leaving     (pointing OUT from node)    │
  │                                                              │
  │                                                              │
  │        out=1          out=1         out=0                    │
  │   (A) ──────→ (B) ──────→ (C)                               │
  │   in=0        in=1  ↗     in=2                              │
  │                    /                                         │
  │   (D) ────────────/                                          │
  │   in=0                                                       │
  │   out=1                                                      │
  │                                                              │
  │   Node  │ In-Degree │ Out-Degree                            │
  │   ──────┼───────────┼───────────                            │
  │     A   │     0     │     1      ← source node              │
  │     B   │     1     │     1                                  │
  │     C   │     2     │     0      ← sink node                 │
  │     D   │     0     │     1      ← source node              │
  │                                                              │
  │  Topological Sort: start from nodes with in-degree = 0      │
  └──────────────────────────────────────────────────────────────┘
```

---

### Path

```
  ┌───────────────────────────────────────────────────────┐
  │   PATH = sequence of nodes connected by edges         │
  │                                                       │
  │   Graph:                                              │
  │   (1)──(2)──(3)──(4)──(5)                            │
  │          \       /                                    │
  │           (6)──(7)                                    │
  │                                                       │
  │   Path from 1 to 5:                                   │
  │   Option A: 1 → 2 → 3 → 4 → 5   length = 4           │
  │   Option B: 1 → 2 → 6 → 7 → 4 → 5  length = 5        │
  │                                                       │
  │   Simple Path:  no node repeated                      │
  │   Walk:         nodes may repeat                      │
  │   Trail:        edges may not repeat                  │
  └───────────────────────────────────────────────────────┘
```

---

### Cycle

```
  ┌───────────────────────────────────────────────────────────┐
  │   CYCLE = a path that returns to its starting node        │
  │                                                           │
  │   UNDIRECTED cycle:           DIRECTED cycle:             │
  │                                                           │
  │      (1)──(2)                    (1)──→(2)               │
  │       │    │                      ↑      │                │
  │      (4)──(3)                    (4)←──(3)               │
  │                                                           │
  │   Cycle: 1→2→3→4→1           Cycle: 1→2→3→4→1            │
  │   (direction doesn't matter) (must follow arrows)         │
  │                                                           │
  │   SELF-LOOP (a node points to itself):                    │
  │      (A) ──→ (A)   ← trivial cycle                        │
  │                                                           │
  │   Acyclic = graph with NO cycles                          │
  └───────────────────────────────────────────────────────────┘
```

---

### DAG — Directed Acyclic Graph

```
  ┌───────────────────────────────────────────────────────────────┐
  │   DAG = Directed graph with NO cycles                         │
  │                                                               │
  │   VALID DAG:                    NOT a DAG (has cycle):        │
  │                                                               │
  │   (A)──→(B)──→(D)              (A)──→(B)──→(D)               │
  │    │          ↑                  ↑           │                │
  │    └──→(C)───┘                   └──────────┘                │
  │                                      CYCLE!                   │
  │   Topological order:                                          │
  │   A → C → B → D   or   A → B → C → D  (multiple valid)      │
  │                                                               │
  │   Real-world DAGs:                                            │
  │   • Course prerequisites   (Math → Calculus → Algorithms)    │
  │   • Build systems          (compile A before linking B)       │
  │   • Git commit history     (each commit points to parent)     │
  │   • Recipe steps           (chop before fry)                  │
  └───────────────────────────────────────────────────────────────┘
```

---

### Connected vs Disconnected

```
  ┌─────────────────────────────────────────────────────────────┐
  │   CONNECTED (undirected): every node reachable from any     │
  │   other node                                                │
  │                                                             │
  │      (1)──(2)──(3)──(4)──(5)                               │
  │             └──(6)                                          │
  │   ✓ All 6 nodes form one connected graph                    │
  │                                                             │
  ├─────────────────────────────────────────────────────────────┤
  │   DISCONNECTED: some nodes cannot reach others              │
  │                                                             │
  │    Component 1      Component 2    Component 3              │
  │   ┌──────────┐     ┌──────────┐   ┌───┐                    │
  │   │ (1)─(2)  │     │  (4)─(5) │   │(6)│                    │
  │   │  │       │     │   │      │   └───┘                    │
  │   │ (3)      │     │  (7)     │                             │
  │   └──────────┘     └──────────┘                             │
  │                                                             │
  │   3 connected components: {1,2,3}, {4,5,7}, {6}            │
  │                                                             │
  │   In code: run BFS/DFS from every unvisited node and        │
  │   count how many times you start fresh = # components       │
  └─────────────────────────────────────────────────────────────┘
```

---

### Weakly vs Strongly Connected (Directed)

```
  ┌──────────────────────────────────────────────────────────────┐
  │   WEAKLY CONNECTED: connected if you IGNORE arrow directions │
  │                                                              │
  │   (A) ──→ (B) ──→ (C)                                       │
  │                                                              │
  │   Ignoring arrows: A─B─C  → all connected ✓                 │
  │   But C cannot reach A following arrows.                     │
  │                                                              │
  ├──────────────────────────────────────────────────────────────┤
  │   STRONGLY CONNECTED: every node can reach every other       │
  │   node FOLLOWING the directed edges                          │
  │                                                              │
  │   (A) ──→ (B) ──→ (C)                                       │
  │    ↑                │                                        │
  │    └────────────────┘                                        │
  │                                                              │
  │   A→B→C→A possible ✓    (A reaches C, C can return to A)   │
  │   This whole graph is strongly connected.                    │
  └──────────────────────────────────────────────────────────────┘
```

---

### Strongly Connected Components (SCC)

```
  ┌──────────────────────────────────────────────────────────────────┐
  │   SCC = maximal subgraph where every node reaches every other    │
  │                                                                  │
  │   ┌─────────────────┐                                            │
  │   │   SCC 1         │                                            │
  │   │  (1) ──→ (2)    │──→ (5)   ← SCC 3 (isolated)               │
  │   │   ↑      │      │                                            │
  │   │   └──(3)─┘      │                                            │
  │   └─────────────────┘                                            │
  │          │                                                        │
  │          ↓                                                        │
  │         (4)  ← SCC 2 (isolated)                                  │
  │                                                                  │
  │   SCC 1: {1, 2, 3}  — all can reach each other via arrows       │
  │   SCC 2: {4}         — can't go back up to {1,2,3}              │
  │   SCC 3: {5}         — dead end                                  │
  │                                                                  │
  │   Condensed DAG (each SCC = one super-node):                     │
  │   [SCC1] ──→ [SCC2]                                              │
  │      └──→ [SCC3]                                                 │
  │                                                                  │
  │   Algorithms: Kosaraju (2-pass DFS), Tarjan (1-pass DFS)        │
  └──────────────────────────────────────────────────────────────────┘
```

---

## 4. Graph Representations

### The Same Graph in Three Forms

```
  Graph to represent:
  ┌───────────────────────────┐
  │   (0)───(1)───(2)         │
  │    │           │          │
  │   (3)──────────┘          │
  │                           │
  │  Edges: (0,1),(1,2),(0,3),(2,3)
  └───────────────────────────┘
```

---

#### Adjacency List

```
  ┌──────────────────────────────────────────┐
  │   ADJACENCY LIST                         │
  │                                          │
  │   Node 0  →  [ 1, 3 ]                   │
  │   Node 1  →  [ 0, 2 ]                   │
  │   Node 2  →  [ 1, 3 ]                   │
  │   Node 3  →  [ 0, 2 ]                   │
  │                                          │
  │   Space: O(V + E)   ← efficient!         │
  │   Get neighbors: O(degree)               │
  │   Check edge (u,v): O(degree(u))         │
  │                                          │
  │   ✓ Best for sparse graphs               │
  │   ✓ Used in 95% of FAANG problems        │
  └──────────────────────────────────────────┘
```

```java
List<List<Integer>> graph = new ArrayList<>();
for (int i = 0; i < n; i++) graph.add(new ArrayList<>());
// undirected edge (u, v):
graph.get(u).add(v);
graph.get(v).add(u);
// weighted directed edge (u → v, weight w):
List<List<int[]>> wg = new ArrayList<>();
wg.get(u).add(new int[]{v, w});
```

---

#### Adjacency Matrix

```
  ┌──────────────────────────────────────────┐
  │   ADJACENCY MATRIX                       │
  │                                          │
  │        0   1   2   3                     │
  │      ┌───┬───┬───┬───┐                  │
  │   0  │ 0 │ 1 │ 0 │ 1 │                  │
  │      ├───┼───┼───┼───┤                  │
  │   1  │ 1 │ 0 │ 1 │ 0 │                  │
  │      ├───┼───┼───┼───┤                  │
  │   2  │ 0 │ 1 │ 0 │ 1 │                  │
  │      ├───┼───┼───┼───┤                  │
  │   3  │ 1 │ 0 │ 1 │ 0 │                  │
  │      └───┴───┴───┴───┘                  │
  │                                          │
  │   matrix[i][j] = 1 means edge (i,j)     │
  │   Space: O(V²)   ← expensive!           │
  │   Check edge (u,v): O(1)  ← fast        │
  │   Get all neighbors: O(V) ← slow        │
  │                                          │
  │   ✓ Good for dense graphs               │
  │   ✓ Good for Floyd-Warshall             │
  └──────────────────────────────────────────┘
```

```java
int[][] matrix = new int[n][n];
matrix[u][v] = 1;          // directed
matrix[u][v] = matrix[v][u] = 1;  // undirected
matrix[u][v] = weight;     // weighted
```

---

#### Edge List

```
  ┌──────────────────────────────────────────┐
  │   EDGE LIST                              │
  │                                          │
  │   [ (0,1), (1,2), (0,3), (2,3) ]        │
  │                                          │
  │   With weights:                          │
  │   [ (0,1,5), (1,2,3), (0,3,10) ]        │
  │    ↑  ↑  ↑                               │
  │    u  v  weight                          │
  │                                          │
  │   Space: O(E)                            │
  │   ✓ Best for Kruskal's MST (sort edges)  │
  │   ✗ Can't quickly find neighbors         │
  └──────────────────────────────────────────┘
```

```java
int[][] edges = {{0,1,5}, {1,2,3}, {0,3,10}};
Arrays.sort(edges, (a, b) -> a[2] - b[2]);  // sort by weight for Kruskal
```

---

#### Comparison at a Glance

```
  ┌──────────────────────┬──────────────┬──────────────┬────────────┐
  │ Operation            │ Adj. List    │ Adj. Matrix  │ Edge List  │
  ├──────────────────────┼──────────────┼──────────────┼────────────┤
  │ Space                │ O(V + E)     │ O(V²)        │ O(E)       │
  │ Check edge (u,v)     │ O(deg(u))    │ O(1)         │ O(E)       │
  │ All neighbors of u   │ O(deg(u))    │ O(V)         │ O(E)       │
  │ Add edge             │ O(1)         │ O(1)         │ O(1)       │
  ├──────────────────────┼──────────────┼──────────────┼────────────┤
  │ Best for             │ Most problems│ Dense/FW     │ Kruskal    │
  └──────────────────────┴──────────────┴──────────────┴────────────┘
```

---

## 5.5 Graph Algorithm Intuition

This is the revision layer: the basic idea behind the common graph algorithms.

### BFS
- Think: "visit nodes in waves"
- Why it works for shortest path in unweighted graphs: every edge costs the same, so the first time you reach a node is the shortest hop count
- Uses a queue, so nodes discovered earlier are processed earlier

### DFS
- Think: "go as deep as possible, then backtrack"
- Why it is useful: it naturally exposes parent-child relationships, recursion stack, and component structure
- Good for cycle detection, topological sort, SCC, and problems that need exhaustive exploration

### Undirected Cycle Detection
- Use DFS with a `parent`
- If you reach a visited node that is not the parent, you found a cycle
- Why: in an undirected graph, every edge appears in both directions, so the parent edge must be ignored

### Directed Cycle Detection
- Use DFS colors: `WHITE`, `GRAY`, `BLACK`
- `GRAY` means "this node is still in the current recursion stack"
- If you see a `GRAY` node again, that is a back edge, which proves a cycle

### Topological Sort
- Only possible on a DAG
- Idea: place nodes so every edge `u -> v` keeps `u` before `v`
- Kahn's algorithm starts from nodes with in-degree `0`
- DFS-based topo sort pushes a node after all of its outgoing neighbors are processed

### Shortest Path
- Unweighted graph: BFS
- Weighted graph with non-negative weights: Dijkstra
- Weighted graph with possible negative weights: Bellman-Ford
- Reason: shortest path choice depends on whether edge weights can change the "best next step"

### MST
- Goal: connect all nodes with minimum total cost
- Think: choose the cheapest edges that do not create a cycle
- Kruskal sorts edges first, then uses DSU to avoid cycles
- Prim grows one connected tree outward using the cheapest available frontier edge

### DSU
- Think: maintain groups of connected nodes
- `find(x)` tells you which group `x` belongs to
- `union(a, b)` merges two groups
- Used when the main question is "are these in the same component?"

---

## 5. Tree vs Graph

```
  ┌──────────────────────────────────────────────────────────────┐
  │   GRAPH (can have cycles):                                   │
  │                                                              │
  │      (1)──(2)                                                │
  │       │    │   ← cycle: 1→2→4→3→1                           │
  │      (3)──(4)                                                │
  │                                                              │
  ├──────────────────────────────────────────────────────────────┤
  │   TREE (connected, acyclic, n-1 edges):                      │
  │                                                              │
  │            (1)                                               │
  │           /   \                                              │
  │         (2)   (3)           5 nodes, 4 edges  ✓             │
  │         / \                 No cycles          ✓             │
  │       (4) (5)               One path between   ✓             │
  │                             any two nodes                    │
  └──────────────────────────────────────────────────────────────┘

  A tree is just a SPECIAL CASE of a graph.
  Every tree is a graph. NOT every graph is a tree.

  ┌──────────────┬────────────────┬──────────────────────────┐
  │ Property     │ Tree           │ Graph                    │
  ├──────────────┼────────────────┼──────────────────────────┤
  │ Cycles       │ Never          │ Possible                 │
  │ Connected    │ Always         │ May be disconnected      │
  │ Edges        │ Exactly n-1    │ 0 to n(n-1)/2            │
  │ Paths (u,v)  │ Exactly one    │ Zero or more             │
  │ Root         │ Yes (rooted)   │ No special root          │
  └──────────────┴────────────────┴──────────────────────────┘
```

---

## 6. BFS vs DFS — Visual Traversal Order

### BFS (Breadth-First Search)

```
  ┌────────────────────────────────────────────────────┐
  │   BFS explores LEVEL BY LEVEL (like ripples)       │
  │                                                    │
  │   Start at (1):                                    │
  │                                                    │
  │              (1)        ← Level 0  visit: 1        │
  │             /   \                                   │
  │           (2)   (3)     ← Level 1  visit: 2, 3     │
  │           / \     \                                 │
  │         (4) (5)   (6)   ← Level 2  visit: 4, 5, 6  │
  │                                                    │
  │   BFS Order: 1 → 2 → 3 → 4 → 5 → 6                │
  │   Uses: QUEUE (FIFO)                               │
  │   Guarantees SHORTEST PATH in unweighted graph     │
  └────────────────────────────────────────────────────┘
```

---

### DFS (Depth-First Search)

```
  ┌────────────────────────────────────────────────────┐
  │   DFS goes DEEP before backtracking                │
  │                                                    │
  │   Start at (1):                                    │
  │                                                    │
  │              (1)①        ① visit 1                 │
  │             /   \        ② go deep left → visit 2  │
  │           (2)②  (3)⑤    ③ go deeper → visit 4     │
  │           / \     \      ④ dead end, backtrack     │
  │         (4)③(5)④  (6)⑥  ⑤ backtrack to 1, go right│
  │                          ⑥ visit 3, then 6         │
  │                                                    │
  │   DFS Order: 1 → 2 → 4 → 5 → 3 → 6               │
  │   Uses: STACK (or recursion call stack)            │
  │   Does NOT guarantee shortest path                 │
  └────────────────────────────────────────────────────┘
```

---

### BFS vs DFS: When to Use Which

```
  ┌──────────────────────────────┬──────────────────────────────┐
  │   Use BFS when...            │   Use DFS when...            │
  ├──────────────────────────────┼──────────────────────────────┤
  │ Shortest path (unweighted)   │ Detect cycles                │
  │ Level-order traversal        │ Topological sort             │
  │ "Closest" or "nearest"       │ Find all paths               │
  │ Multi-source spreading       │ Connected components         │
  │   (rotting oranges)          │ Backtracking problems        │
  │ Word ladder                  │ SCCs (Kosaraju/Tarjan)       │
  └──────────────────────────────┴──────────────────────────────┘
```

---

## 7. Special Graph Types

### Bipartite Graph

```
  ┌────────────────────────────────────────────────────────────┐
  │   BIPARTITE: nodes split into 2 groups (U and V),          │
  │   all edges go BETWEEN groups — never WITHIN a group       │
  │                                                            │
  │      Group U        Group V                                │
  │      ───────        ───────                                │
  │      [Alice] ──────→ [Math]                                │
  │        │    ╲         [Science]                            │
  │      [Bob]   ╲──────→ [History]                            │
  │        │                                                   │
  │      [Carol] ────────→ [Math]                              │
  │                                                            │
  │   Students on left, courses on right.                      │
  │   No edge between two students or two courses.             │
  │                                                            │
  │   KEY: A graph is bipartite ⟺ it has NO odd-length cycles  │
  │                                                            │
  │   NOT Bipartite (has triangle — odd cycle of length 3):    │
  │      (A)──(B)                                              │
  │       │  /                                                  │
  │      (C)    ← A-B-C-A is a cycle of length 3 (odd)        │
  │                                                            │
  │   Check: 2-color with BFS. Conflict = not bipartite.      │
  │   LC 785 — Is Graph Bipartite?                             │
  └────────────────────────────────────────────────────────────┘
```

---

### Complete Graph

```
  ┌────────────────────────────────────────────────────────────┐
  │   COMPLETE GRAPH K_n: every node connected to every other  │
  │                                                            │
  │   K_3 (3 nodes):     K_4 (4 nodes):     K_5 (5 nodes):    │
  │                                                            │
  │      (1)              (1)──(2)            (1)              │
  │     /   \            / │╲  /│╲           /│╲╲             │
  │   (2)──(3)          /  │  X  │  \        / │  ╲╲          │
  │                   (3)──│──╱──│──(4)     (5)─(2)           │
  │   3 edges             (3)────(4)         │╲/│╲/│          │
  │                        6 edges          (4)─(3)           │
  │                                          10 edges          │
  │                                                            │
  │   K_n has n(n-1)/2 edges                                  │
  └────────────────────────────────────────────────────────────┘
```

---

### Eulerian Path and Circuit

```
  ┌────────────────────────────────────────────────────────────┐
  │   EULERIAN PATH: visits every EDGE exactly once            │
  │   EULERIAN CIRCUIT: does same AND returns to start         │
  │                                                            │
  │   Famous puzzle: Königsberg Bridges                        │
  │                                                            │
  │   Can you cross all bridges exactly once?                  │
  │                                                            │
  │       Land A                                               │
  │      /  │  \                                               │
  │   B1   B2   B3   ← 7 bridges                              │
  │    │    │    │                                             │
  │   Land B   Land C                                          │
  │       \   /                                                │
  │        B4─B5─B6─B7                                         │
  │        Land D                                              │
  │                                                            │
  │   Answer: NO — some nodes have odd degree.                 │
  │                                                            │
  │   RULES:                                                   │
  │   Eulerian CIRCUIT exists  ⟺  ALL nodes have EVEN degree   │
  │   Eulerian PATH exists     ⟺  EXACTLY 2 nodes have ODD    │
  │                               degree (they are start/end)  │
  │                                                            │
  │   Example (circuit possible):                              │
  │      (A)──(B)   All degrees are even:                      │
  │       │    │    deg(A)=2, deg(B)=2                         │
  │      (D)──(C)   deg(C)=2, deg(D)=2   ✓                    │
  │                                                            │
  │   LC 332 — Reconstruct Itinerary (Eulerian Path)          │
  └────────────────────────────────────────────────────────────┘
```

---

### Sparse vs Dense

```
  ┌────────────────────────────────────────────────────────────┐
  │   5 nodes. Max possible edges = 5×4/2 = 10                │
  │                                                            │
  │   SPARSE (E << V²):          DENSE (E ≈ V²):              │
  │                                                            │
  │   (1)──(2)                  (1)──(2)                       │
  │    │                        / │╲ /│╲                       │
  │   (3)  (4)──(5)           (3)─(4)─(5)                      │
  │                                                            │
  │   3 edges (few)            9 edges (many)                  │
  │   Use: Adjacency List      Use: Adjacency Matrix           │
  │                                                            │
  │   Most FAANG problems are sparse → Adjacency List          │
  └────────────────────────────────────────────────────────────┘
```

---

## 8. Cycle Detection — How It Works Visually

### Undirected: DFS with Parent Tracking

```
  ┌───────────────────────────────────────────────────────┐
  │   If we visit a node that is already visited AND       │
  │   it's NOT our direct parent → CYCLE FOUND            │
  │                                                        │
  │   Graph:  (1)──(2)──(3)──(1)  ← cycle                │
  │                                                        │
  │   DFS from 1:                                          │
  │   Visit 1 (parent: none)                               │
  │     Visit 2 (parent: 1)                                │
  │       Visit 3 (parent: 2)                              │
  │         See 1 → already visited, not parent of 3       │
  │         → CYCLE DETECTED ✓                             │
  └───────────────────────────────────────────────────────┘
```

### Directed: Three-Color DFS

```
  ┌───────────────────────────────────────────────────────┐
  │   WHITE = unvisited                                    │
  │   GRAY  = currently in DFS stack (being processed)    │
  │   BLACK = fully processed (done)                      │
  │                                                        │
  │   Cycle exists if we reach a GRAY node                │
  │   (means we found a back edge to an ancestor)         │
  │                                                        │
  │   Graph: (A)──→(B)──→(C)──→(A)  ← cycle              │
  │                                                        │
  │   DFS from A:                                          │
  │   A → GRAY                                             │
  │     B → GRAY                                           │
  │       C → GRAY                                         │
  │         See A → A is GRAY → BACK EDGE → CYCLE ✓       │
  └───────────────────────────────────────────────────────┘
```

---

## 9. Topological Sort — Visual

```
  ┌────────────────────────────────────────────────────────────┐
  │   TOPOLOGICAL SORT: linear order of a DAG such that         │
  │   for every edge u→v, u appears BEFORE v                   │
  │                                                            │
  │   Example: Course prerequisites                            │
  │                                                            │
  │   Math──→ Calculus ──→ Algorithms                         │
  │   Physics ──────────→ Algorithms                          │
  │                                                            │
  │   In-degrees:                                              │
  │   Math=0, Physics=0, Calculus=1, Algorithms=2              │
  │                                                            │
  │   Step 1: Queue nodes with in-degree 0                     │
  │   Queue: [Math, Physics]                                   │
  │                                                            │
  │   Step 2: Process Math → reduce Calculus in-degree (0)     │
  │   Queue: [Physics, Calculus]                               │
  │                                                            │
  │   Step 3: Process Physics → reduce Algorithms in-degree(1) │
  │   Queue: [Calculus]                                        │
  │                                                            │
  │   Step 4: Process Calculus → reduce Algorithms in-deg (0)  │
  │   Queue: [Algorithms]                                      │
  │                                                            │
  │   Result: Math → Physics → Calculus → Algorithms           │
  │   (or Math → Calculus → Physics → Algorithms — both valid) │
  └────────────────────────────────────────────────────────────┘
```

---

## 10. MST — Minimum Spanning Tree

```
  ┌────────────────────────────────────────────────────────────┐
  │   MST: Connect all nodes with MINIMUM total edge weight    │
  │   Uses exactly n-1 edges (like a tree)                    │
  │                                                            │
  │   Original weighted graph:                                 │
  │                                                            │
  │        4         2                                         │
  │   (A)──────(B)──────(C)                                    │
  │    │    7       5    │                                      │
  │    │  ╲          ╱  │                                      │
  │    │3   ╲ (E)  ╱    │6                                     │
  │    │     ╱  1 ╲     │                                      │
  │   (D)──────────────(F)                                     │
  │           8                                                │
  │                                                            │
  │   MST (pick cheapest edges without forming cycle):         │
  │                                                            │
  │        4         2                                         │
  │   (A)──────(B)──────(C)                                    │
  │    │                 │                                      │
  │    │3          1     │6 ← wait, E─F=1 is cheaper           │
  │    │       (E)──(F)  │  than C─F=6, pick E─F               │
  │   (D)                                                      │
  │                                                            │
  │   MST edges: A─D(3), B─C(2), A─B(4), E─F(1), C─F(6)?    │
  │   Actually pick the 4 cheapest that don't form a cycle:    │
  │   E─F(1), B─C(2), A─D(3), A─B(4)  → total = 10           │
  │                                                            │
  │   Kruskal: sort edges, add if no cycle (uses DSU)          │
  │   Prim:    grow MST greedily from a starting node (PQ)     │
  └────────────────────────────────────────────────────────────┘
```

---

## 11. Graph Problem Recognition Guide

```
  Problem describes...                    → Model as...
  ──────────────────────────────────────────────────────────────────
  Grid (rows/cols, islands, maze)         → Implicit grid graph
  Cities and roads / flights               → Weighted directed/undirected
  Task dependencies ("A before B")         → Directed graph (DAG)
  "Can A reach B?"                         → BFS/DFS connectivity
  "Shortest path A to B" (no weights)      → BFS
  "Shortest path A to B" (with weights)    → Dijkstra
  "Min cost to connect ALL nodes"          → MST (Kruskal / Prim)
  Social network (friends, mutual)         → Undirected graph
  Social network (follows, one-way)        → Directed graph
  "Group together connected things"        → DSU / BFS components
  State transitions (puzzle, word ladder)  → Implicit graph + BFS
  "Are X and Y in same group?"             → DSU (find same root)
  "Schedule tasks respecting order"        → Topological Sort
  Network reliability / critical links     → Bridges (Tarjan)
  Matching (left nodes ↔ right nodes)     → Bipartite + Matching
```

---

## 12. Complexity Quick Reference

```
  ┌───────────────────────┬──────────────────┬──────────┬─────────────────────────┐
  │ Algorithm             │ Time             │ Space    │ Use When                │
  ├───────────────────────┼──────────────────┼──────────┼─────────────────────────┤
  │ BFS                   │ O(V + E)         │ O(V)     │ Shortest path, unweighted│
  │ DFS                   │ O(V + E)         │ O(V)     │ Components, cycle detect │
  │ Topological Sort      │ O(V + E)         │ O(V)     │ DAG ordering             │
  │ Dijkstra (PQ)         │ O((V+E) log V)   │ O(V)     │ Shortest, non-neg weights│
  │ Bellman-Ford          │ O(V × E)         │ O(V)     │ Negative weights         │
  │ Floyd-Warshall        │ O(V³)            │ O(V²)    │ All-pairs, V ≤ 100       │
  │ Prim's MST            │ O((V+E) log V)   │ O(V)     │ MST, dense graph         │
  │ Kruskal's MST         │ O(E log E)       │ O(V)     │ MST, sort edges          │
  │ DSU (path compression)│ O(α(V)) ≈ O(1)  │ O(V)     │ Components, cycle, MST   │
  │ Tarjan / Kosaraju SCC │ O(V + E)         │ O(V)     │ Strongly connected comp  │
  └───────────────────────┴──────────────────┴──────────┴─────────────────────────┘
  α = inverse Ackermann function — effectively constant
```

---

## 13. Pre-Solve Checklist (FAANG)

```
  Before writing a single line of code, answer these:

  [ ] Directed or undirected?
        → affects: cycle detection method, DFS parent tracking

  [ ] Weighted or unweighted?
        → affects: BFS vs Dijkstra vs Bellman-Ford

  [ ] Can weights be negative?
        → if yes: Bellman-Ford, not Dijkstra

  [ ] Can there be self-loops or parallel edges?
        → handle in validity checks

  [ ] Is the graph guaranteed connected?
        → if not: loop over all nodes to start BFS/DFS

  [ ] 0-indexed or 1-indexed nodes?
        → adjust array sizes accordingly

  [ ] Is it a DAG?
        → if yes: topological sort is available

  [ ] What are V and E?
        → V,E ≤ 100  → Floyd-Warshall fine
        → E ≤ 10^5   → Dijkstra fine
        → very dense → Adjacency Matrix
```

---

> **Next →** [Graph Patterns & Algorithms](08_graph_patterns.md)
> **Back  ←** [Master Index](../DSA_MASTER_INDEX.md)
