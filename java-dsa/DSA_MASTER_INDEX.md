# DSA Master Index — Pattern-Based Learning
> Style: Aditya Verma / Striver Sheet
> Language: Java
> Each topic = a set of **patterns**. Learn the pattern once, apply it to 10 problems.

---

## How to Use This Index

```
Topic → Pattern → Easy → Medium → Hard
```

For every pattern, ask yourself:
1. What is the **recognition trigger**? (what in the problem hints this pattern?)
2. What is the **core trick**?
3. What is the **template code**?
4. What are the **variations**?

---

## Topics Roadmap

| # | Topic               | Status  | Notes File |
|---|---------------------|---------|------------|
| 1 | **Arrays**          | ✅ Done | [arrays/01_array_patterns.md](arrays/01_array_patterns.md) |
| 2 | **Strings**         | ✅ Done | [strings/02_string_patterns.md](strings/02_string_patterns.md) |
| 3 | **Linked List**     | ✅ Done | [linked-list/03_linkedlist_patterns.md](linked-list/03_linkedlist_patterns.md) |
| 4 | **Stack & Queue**   | ✅ Done | [stack-queue/04_stack_queue_patterns.md](stack-queue/04_stack_queue_patterns.md) |
| 5 | **Binary Search**   | ✅ Done | [binary-search/05_binary_search_patterns.md](binary-search/05_binary_search_patterns.md) |
| 6 | **Recursion & Backtracking** | ✅ Done | [backtracking/06_backtracking_patterns.md](backtracking/06_backtracking_patterns.md) |
| 7 | **Trees (BT + BST)**| ✅ Done | [trees/07_tree_patterns.md](trees/07_tree_patterns.md) |
| 8 | **Graphs**          | ✅ Done | [graphs/00_graph_theory.md](graphs/00_graph_theory.md) → [graphs/08_graph_patterns.md](graphs/08_graph_patterns.md) |
| 9 | **Dynamic Programming** | ✅ Done | [dp/09_dp_patterns.md](dp/09_dp_patterns.md) |
| 10| **Heaps / Priority Queue** | ✅ Done | [heaps/10_heap_patterns.md](heaps/10_heap_patterns.md) |
| 11| **Tries**           | ✅ Done | [tries/11_trie_patterns.md](tries/11_trie_patterns.md) |
| 12| **Segment Tree / BIT** | ✅ Done | [advanced/12_segment_tree_bit.md](advanced/12_segment_tree_bit.md) |

---

## 1. Arrays — Patterns

| Pattern | Trigger Words | Difficulty |
|---------|--------------|------------|
| Prefix Sum | "range sum query", "subarray sum = k" | Easy–Medium |
| Difference Array | "range update then query", "add v to range [L,R]" | Medium |
| Two Pointers | "sorted array", "pair/triplet sum", "palindrome" | Easy–Hard |
| Sliding Window (Fixed) | "subarray of size k", "max/min in window" | Easy–Medium |
| Sliding Window (Variable) | "longest/shortest subarray with condition" | Medium–Hard |
| Kadane's Algorithm | "maximum subarray", "contiguous sum" | Medium–Hard |
| Binary Search on Answer | "minimize the maximum", "feasibility check" | Hard |
| Sorting + Greedy | "interval merge", "meeting rooms", "schedule" | Medium |
| HashMap / Frequency | "count subarrays", "anagram", "first missing" | Medium |
| Monotonic Stack | "next greater", "histogram", "stock span" | Medium–Hard |
| Matrix Traversal | "spiral", "rotate", "diagonal", "set zeros" | Easy–Medium |

---

## 2. Strings — Patterns

| Pattern | Trigger Words |
|---------|--------------|
| Two Pointers | "palindrome", "reverse", "match characters" |
| Sliding Window | "substring with condition", "min window" |
| Hashing / Frequency | "anagram", "permutation in string" |
| KMP / Z-algorithm | "pattern matching", "find all occurrences" |
| Trie | "prefix search", "word dictionary" |
| Stack-based | "valid parentheses", "decode string" |

---

## 3. Linked List — Patterns

| Pattern | Trigger Words |
|---------|--------------|
| Fast & Slow Pointers | "cycle detection", "middle of list", "Nth from end" |
| Reversal | "reverse list", "reverse in k-groups", "palindrome LL" |
| Merge | "merge k sorted lists", "merge two sorted" |
| Clone with Random Pointer | "deep copy", "random pointer" |

---

## 4. Stack & Queue — Patterns

| Pattern | Trigger Words |
|---------|--------------|
| Monotonic Stack | "next greater/smaller", "span", "histogram" |
| Min Stack | "O(1) getMin", "stack with extra info" |
| Sliding Window Max | "max in window", "deque" |
| BFS (Queue) | "level order", "shortest path unweighted" |
| Expression Evaluation | "evaluate", "decode", "operator precedence" |

---

## 5. Binary Search — Patterns

| Pattern | Trigger Words |
|---------|--------------|
| Classic BS | "sorted array", "find target" |
| BS on Answer Space | "minimize max", "maximize min", "feasibility" |
| Rotated Array | "rotated sorted", "find pivot" |
| Peak Finding | "mountain array", "peak element" |
| First / Last Occurrence | "leftmost", "rightmost", "count occurrences" |

---

## 6. Recursion & Backtracking — Patterns

| Pattern | Trigger Words |
|---------|--------------|
| Subsets / Power Set | "all subsets", "generate all combinations" |
| Permutations | "all arrangements", "generate all permutations" |
| Combinations | "choose k from n", "combination sum" |
| Constraint Satisfaction | "N-Queens", "Sudoku", "word search" |
| Pruning | same as above, with validity checks |

---

## 7. Trees — Patterns

| Pattern | Trigger Words |
|---------|--------------|
| DFS – Preorder | "serialize", "path root to leaf" |
| DFS – Inorder | "BST sorted order", "kth smallest" |
| DFS – Postorder | "bottom-up", "delete nodes", "diameter" |
| BFS – Level Order | "zigzag", "right view", "level averages" |
| Path Sum | "root to leaf sum", "any path max sum" |
| LCA | "lowest common ancestor", "distance between nodes" |
| BST Operations | "insert/delete/search", "validate BST" |
| Morris Traversal | "O(1) space inorder traversal" |

---

## 8. Graphs — Patterns

| Pattern | Trigger Words |
|---------|--------------|
| BFS | "shortest path unweighted", "level traversal", "0-1 BFS" |
| DFS | "connected components", "cycle detection", "islands" |
| Topological Sort | "dependency order", "course schedule", "DAG" |
| Union-Find (DSU) | "connected components", "Kruskal's MST", "detect cycle" |
| Dijkstra | "shortest path weighted", "non-negative edges" |
| Bellman-Ford | "negative edges", "detect negative cycle" |
| Floyd-Warshall | "all pairs shortest path" |
| Prim's MST | "minimum spanning tree" |
| Bridges / Articulation Points | "critical connections" |

---

## 9. Dynamic Programming — Patterns (Aditya Verma Style)

> **The DP Recipe:** Identify → Memoize → Tabulate → Optimize space

### 9.1 0/1 Knapsack Family
```
Trigger: "choose or not choose", "subset", "binary decision per item"
```
| Problem | Variation |
|---------|-----------|
| 0/1 Knapsack | base |
| Subset Sum | knapsack where value = weight |
| Equal Partition | subset sum = totalSum/2 |
| Count of Subsets with Sum | count variant |
| Minimum Subset Sum Difference | minimize |leftSum - rightSum| |
| Target Sum (+/-) | LeetCode 494 |
| Last Stone Weight II | disguised partition |

### 9.2 Unbounded Knapsack Family
```
Trigger: "unlimited use of items", "coin change", "rod cutting"
```
| Problem | Variation |
|---------|-----------|
| Unbounded Knapsack | base |
| Rod Cutting | same as unbounded |
| Coin Change (min coins) | minimize |
| Coin Change (count ways) | count |
| Maximum Ribbon Cut | variation |

### 9.3 LCS (Longest Common Subsequence) Family
```
Trigger: "two strings/sequences", "common", "edit distance"
```
| Problem | Variation |
|---------|-----------|
| LCS | base |
| Longest Common Substring | contiguous |
| Print LCS | backtracking on DP table |
| Shortest Common Supersequence | SCS length = m+n-LCS |
| Edit Distance | insert/delete/replace |
| Delete Operations for Same String | minimize deletions |
| Longest Palindromic Subsequence | LPS = LCS(s, reverse(s)) |
| Minimum Insertions for Palindrome | n - LPS |

### 9.4 LIS (Longest Increasing Subsequence) Family
```
Trigger: "increasing/decreasing subsequence", "envelope problem"
```
| Problem | Variation |
|---------|-----------|
| LIS | base O(n²) / O(n log n) |
| Maximum Sum Increasing Subsequence | |
| Minimum Deletions to Make Sorted | n - LIS |
| Longest Bitonic Subsequence | LIS from left + LIS from right |
| Russian Doll Envelopes | 2D LIS |
| Number of LIS | count variant |

### 9.5 Matrix Chain Multiplication (MCM) Family
```
Trigger: "burst balloons", "split array", "optimal way to break/merge"
```
| Problem | Variation |
|---------|-----------|
| MCM | base (interval DP) |
| Palindrome Partitioning II | min cuts |
| Boolean Parenthesization | count true evaluations |
| Scramble String | tree-based interval DP |
| Burst Balloons | reverse thinking |
| Egg Drop | classic |

### 9.6 Grid DP
```
Trigger: "m×n grid", "paths", "robot"
```
| Problem | Variation |
|---------|-----------|
| Unique Paths | base |
| Unique Paths with Obstacles | |
| Minimum Path Sum | |
| Triangle | top-down grid |
| Cherry Pickup | two robots on grid |
| Dungeon Game | reverse DP |

### 9.7 DP on Trees
```
Trigger: "tree", "subtree", "root choices affect children"
```
| Problem | Variation |
|---------|-----------|
| Diameter of Binary Tree | postorder |
| Max Path Sum in Tree | |
| Binary Tree Cameras | |
| House Robber III | |

### 9.8 Bitmask DP
```
Trigger: "n ≤ 20", "assign tasks to people", "visiting all cities"
```
| Problem | Variation |
|---------|-----------|
| TSP (Travelling Salesman) | base |
| Minimum XOR Sum | assignment |
| Partition to K Equal Subsets | |

---

## 10. Heaps — Patterns

| Pattern | Trigger Words |
|---------|--------------|
| Top K Elements | "k largest", "k most frequent" |
| Kth Largest/Smallest | "kth element in stream" |
| Merge K Sorted | "k sorted lists/arrays" |
| Median of Stream | "running median", two heaps |
| Sliding Window Maximum | monotonic deque |
| Task Scheduler | "cooldown", "frequency-based scheduling" |

---

## 11. Tries — Patterns

| Pattern | Trigger Words |
|---------|--------------|
| Insert/Search/StartsWith | "prefix search", "autocomplete" |
| Word Search in Board | "DFS + Trie" |
| Replace Words | "dictionary roots" |
| Maximum XOR | "bit manipulation + Trie" |
| Palindrome Pairs | "word pairs forming palindrome" |

---

## 12. Segment Tree / BIT — Patterns

| Pattern | Trigger Words |
|---------|--------------|
| Range Sum Query (Mutable) | "update + range query" |
| Range Min/Max Query | "RMQ" |
| Count of Smaller Numbers | "inversions", "BIT" |
| Falling Squares | "interval + coordinate compression" |

---

## Pattern Recognition Cheat Sheet

```
Problem says...                 → Think...
─────────────────────────────────────────────────
"subarray sum = k"              → Prefix Sum + HashMap
"range updates then query"      → Difference Array
"sorted array, pair sum"        → Two Pointers
"longest/shortest subarray"     → Sliding Window (Variable)
"fixed window of size k"        → Sliding Window (Fixed)
"maximum subarray"              → Kadane's
"minimize the maximum"          → Binary Search on Answer
"next greater element"          → Monotonic Stack
"cycle in list"                 → Fast & Slow Pointers
"shortest path unweighted"      → BFS
"dependency / ordering"         → Topological Sort
"connected components"          → DSU / DFS
"choose or skip each item"      → 0/1 Knapsack DP
"two strings, common"           → LCS DP
"increasing sequence"           → LIS DP
"split into parts optimally"    → MCM / Interval DP
"k largest / kth"               → Heap
"prefix match / autocomplete"   → Trie
"n ≤ 20, assign tasks"          → Bitmask DP
```

---

> **Next Step:** Start with [Arrays → Pattern Deep Dive](arrays/01_array_patterns.md)
