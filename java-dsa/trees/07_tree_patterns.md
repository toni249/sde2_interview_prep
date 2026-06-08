# Trees — Pattern Deep Dive
> FAANG Level | Java | Easy → Hard

---

## Pattern Map

```
TREES
├── P1: DFS — Preorder/Inorder/Postorder  → serialize, BST, bottom-up
├── P2: BFS — Level Order                 → level traversal, zigzag, right view
├── P3: Path Problems                     → root-to-leaf, any-path max sum
├── P4: Tree Properties                   → height, diameter, balanced check
├── P5: LCA                               → lowest common ancestor variants
├── P6: BST Operations                    → insert, delete, validate, kth smallest
├── P7: Tree Construction                 → build from traversals
└── P8: Advanced                          → serialize/deserialize, Morris traversal
```

---

## Boilerplate

```java
class TreeNode {
    int val;
    TreeNode left, right;
    TreeNode(int val) { this.val = val; }
}
```

---

## P1: DFS Traversals

### The Three Orders

```
        1
       / \
      2   3
     / \
    4   5

Preorder  (Root, Left, Right): 1 2 4 5 3
Inorder   (Left, Root, Right): 4 2 5 1 3
Postorder (Left, Right, Root): 4 5 2 3 1
```

```java
// Recursive — easy but O(n) stack space
void preorder(TreeNode root, List<Integer> result) {
    if (root == null) return;
    result.add(root.val);           // Root first
    preorder(root.left, result);
    preorder(root.right, result);
}

// Iterative Preorder — more common in FAANG interviews
List<Integer> preorderIter(TreeNode root) {
    List<Integer> result = new ArrayList<>();
    if (root == null) return result;
    Deque<TreeNode> stack = new ArrayDeque<>();
    stack.push(root);
    while (!stack.isEmpty()) {
        TreeNode node = stack.pop();
        result.add(node.val);
        if (node.right != null) stack.push(node.right);  // right first (LIFO)
        if (node.left != null)  stack.push(node.left);
    }
    return result;
}

// Iterative Inorder (BST sorted order)
List<Integer> inorderIter(TreeNode root) {
    List<Integer> result = new ArrayList<>();
    Deque<TreeNode> stack = new ArrayDeque<>();
    TreeNode curr = root;
    while (curr != null || !stack.isEmpty()) {
        while (curr != null) { stack.push(curr); curr = curr.left; }  // go left
        curr = stack.pop();
        result.add(curr.val);
        curr = curr.right;                                              // go right
    }
    return result;
}

// Iterative Postorder = reverse of (Root, Right, Left)
List<Integer> postorderIter(TreeNode root) {
    LinkedList<Integer> result = new LinkedList<>();
    Deque<TreeNode> stack = new ArrayDeque<>();
    if (root != null) stack.push(root);
    while (!stack.isEmpty()) {
        TreeNode node = stack.pop();
        result.addFirst(node.val);          // add to FRONT
        if (node.left != null)  stack.push(node.left);
        if (node.right != null) stack.push(node.right);
    }
    return result;
}
```

### Questions — DFS

| # | Problem | LC# |
|---|---------|-----|
| E1 | Binary Tree Preorder Traversal | 144 |
| E2 | Binary Tree Inorder Traversal | 94 |
| E3 | Binary Tree Postorder Traversal | 145 |
| M1 | Binary Tree Zigzag Level Order Traversal | 103 |
| M2 | Flatten Binary Tree to Linked List | 114 |

---

## P2: BFS — Level Order

### Template
```java
List<List<Integer>> levelOrder(TreeNode root) {
    List<List<Integer>> result = new ArrayList<>();
    if (root == null) return result;
    Queue<TreeNode> queue = new LinkedList<>();
    queue.offer(root);

    while (!queue.isEmpty()) {
        int size = queue.size();          // snapshot of current level size
        List<Integer> level = new ArrayList<>();
        for (int i = 0; i < size; i++) {
            TreeNode node = queue.poll();
            level.add(node.val);
            if (node.left != null)  queue.offer(node.left);
            if (node.right != null) queue.offer(node.right);
        }
        result.add(level);
    }
    return result;
}
```
> Key: `int size = queue.size()` BEFORE the inner loop. This captures how many nodes are in the current level.

### Variation 1: Right Side View (LC 199)

**Problem:** Given a binary tree, return the values of nodes you can see when looking at it from the **right** side, ordered top to bottom.

**Approach (BFS level order, top-down via queue):**
- Standing on the right, the only node visible per level is the **last node** popped from that level — every other node is occluded by something to its right.
- Use the standard level-order BFS template; record the value when `i == size - 1` for the current level.
- What the BFS "carries": the queue holds nodes per level; nothing else is passed down explicitly — depth is implicit in the level boundary.
- Edge cases: `null` root → empty list. Skewed-left tree → still works (each level has exactly one node = the rightmost).
- Time O(n), space O(w) where w = max width.
- Alternative: reverse-preorder DFS (`root → right → left`) tracking `depth`; first visit at each depth is the rightmost.

```java
// The last element of each level
List<Integer> rightSideView(TreeNode root) {
    List<Integer> result = new ArrayList<>();
    if (root == null) return result;
    Queue<TreeNode> queue = new LinkedList<>();
    queue.offer(root);
    while (!queue.isEmpty()) {
        int size = queue.size();
        for (int i = 0; i < size; i++) {
            TreeNode node = queue.poll();
            if (i == size - 1) result.add(node.val);   // last in level
            if (node.left != null)  queue.offer(node.left);
            if (node.right != null) queue.offer(node.right);
        }
    }
    return result;
}
```

### Variation 2: Zigzag Level Order (LC 103)

**Problem:** Return the level-order traversal of a binary tree's values, but with **direction alternating** per level — level 0 left-to-right, level 1 right-to-left, level 2 left-to-right, and so on.

**Approach (BFS + per-level direction flag):**
- The BFS structure is unchanged — we still enqueue children left-then-right. Only the **way we append into the level's list** flips.
- Maintain a `leftToRight` boolean; on each odd level, insert at index `0` (which reverses the order without reversing the queue or revisiting children).
- Using `LinkedList` (or `ArrayDeque`) as the per-level list avoids O(k) shift cost on `add(0, x)`. With `ArrayList`, prefer building L→R always and `Collections.reverse()` on alternate levels.
- Edge cases: `null` root → empty result; single node → one level with one element regardless of direction.
- Time O(n), space O(w).

```java
// Alternate direction: even levels left-to-right, odd levels right-to-left
// Just toggle the add direction with a boolean
if (leftToRight) level.add(node.val);
else             level.add(0, node.val);   // add to front
leftToRight = !leftToRight;
```

### Variation 3: Level Averages, Max, Min
> Trivially derived from the level order template — iterate level list.

### Variation 4: Connect Next Right Pointers (LC 116, 117)

**Problem:** Each node has a `next` pointer (default `null`). Populate `next` so it points to the next node **on the same level**, or `null` if the node is the rightmost on its level. LC 116 = perfect binary tree, LC 117 = any binary tree. Solve in O(1) extra space (recursion stack aside).

**Approach (Use the already-built `next` chain of the current level — top-down):**
- Treat the current level as a **linked list** built by `next` pointers (root's level has just one node, length 1).
- Walk that linked list left to right. Use a `dummy` head + `prev` pointer to splice the **next level's** children together as we encounter them.
- After traversing one full level, jump to `dummy.next` — that's the head of the next level's linked list. Repeat.
- Works for LC 117 (general tree) because we don't assume both children exist: we only attach children that are non-null.
- Why O(1) space: no queue, no recursion needed; the `next` pointers themselves serve as the level queue.
- Edge cases: `null` root; a level with all missing children (next-level head becomes `null`, outer loop ends).
- Time O(n), space O(1).

```java
// O(1) space — use current level to build next level connections
Node connect(Node root) {
    Node curr = root;
    while (curr != null) {
        Node dummy = new Node(0);
        Node prev = dummy;
        while (curr != null) {
            if (curr.left != null)  { prev.next = curr.left;  prev = prev.next; }
            if (curr.right != null) { prev.next = curr.right; prev = prev.next; }
            curr = curr.next;       // move to next node in current level
        }
        curr = dummy.next;          // move to first node of next level
    }
    return root;
}
```

### Questions — BFS

| # | Problem | LC# |
|---|---------|-----|
| E1 | Binary Tree Level Order Traversal | 102 |
| M1 | Binary Tree Right Side View | 199 |
| M2 | Binary Tree Zigzag Level Order Traversal | 103 |
| M3 | Average of Levels in Binary Tree | 637 |
| M4 | Populating Next Right Pointers (perfect tree) | 116 |
| M5 | Populating Next Right Pointers II (any tree) | 117 |
| H1 | Binary Tree Maximum Width | 662 |

---

## P3: Path Problems

### Key Insight
> For ANY path problem, the postorder traversal (bottom-up) is your friend.
> Process children first, then use their results to compute the current node's answer.

### Variation 1: Path Sum (root to leaf) — LC 112

**Problem:** Given a binary tree and an integer `targetSum`, return `true` iff there exists a **root-to-leaf** path whose values add up to exactly `targetSum`.

**Approach (Top-down DFS, pass remaining sum down):**
- At each node, subtract `node.val` from the remaining target and recurse. This is top-down: we **pass info down via parameters** (the remaining target) rather than computing from children's returns.
- A leaf is a node with both children null. Only at a leaf do we compare `remaining == node.val`. Don't accept partial paths at internal nodes.
- Each recursive call returns: a boolean — "does some root-to-leaf path *under me* hit the (already reduced) target?"
- Edge cases: `null` root → `false` (no path exists). A single-node tree → `root.val == target`. Negative values allowed → cannot prune early on sum overshoot.
- Time O(n), space O(h) for recursion (h = height).

```java
boolean hasPathSum(TreeNode root, int target) {
    if (root == null) return false;
    if (root.left == null && root.right == null) return root.val == target;
    return hasPathSum(root.left, target - root.val) ||
           hasPathSum(root.right, target - root.val);
}
```

### Variation 2: All Root-to-Leaf Paths (LC 257)

**Problem:** Return all root-to-leaf paths in any order, each formatted as `"v1->v2->...->vk"`.

**Approach (Top-down DFS with path accumulator):**
- Pass the **partial path string** (or a `List<Integer>` you mutate + backtrack) down as a parameter — top-down style.
- At a leaf (both children null), commit the accumulated path to the result list.
- Each recursive call returns nothing; results are written into a shared `List<String>` collected at the root.
- Using `String` concatenation is O(L) per append but avoids needing backtracking; using `StringBuilder` is faster but requires `setLength()` on return to backtrack.
- Edge cases: `null` root → empty list; a single-node tree → one path `"<val>"`.
- Time O(n·h) due to building string paths of length up to h. Space O(n·h) for output.

```java
List<String> binaryTreePaths(TreeNode root) {
    List<String> result = new ArrayList<>();
    dfs(root, "", result);
    return result;
}
void dfs(TreeNode node, String path, List<String> result) {
    if (node == null) return;
    path += node.val;
    if (node.left == null && node.right == null) { result.add(path); return; }
    dfs(node.left,  path + "->", result);
    dfs(node.right, path + "->", result);
}
```

### Variation 3: Maximum Path Sum (ANY path, LC 124) — FAANG Hard

**Problem:** A path is any sequence of nodes connected by parent-child edges (no node repeated). It does **not** have to pass through the root or end at a leaf. Return the maximum sum of node values along any such path.

**Approach (Bottom-up DFS — separate "what I return" vs "what I record"):**
- The classic two-quantity trick:
  1. **Return value** of `maxGain(node)`: best sum of a path that goes *strictly downward* from `node` (so the parent can extend it further). Only one branch can be picked when going through a node — `node.val + max(leftGain, rightGain)`.
  2. **Global candidate** updated at each node: a path that **bends** at this node, using both children — `node.val + leftGain + rightGain`. This path cannot be extended upward (it already uses two children), so it's a closing candidate.
- Clamp negative gains at 0 (`Math.max(0, gain)`) — if a subtree's best contribution is negative, just drop it.
- This is bottom-up: each call **returns info from subtrees**.
- Edge cases: tree of one negative node → answer is that negative value (initialize `maxSum = Integer.MIN_VALUE`, not 0). All-negative tree → still works because we use a global max, not a sum from root.
- Time O(n), space O(h) recursion.

```java
// Path can start and end anywhere — doesn't have to go through root
// At each node: max gain going down = max(0, left_gain, right_gain)
// Update global max at each node with left_gain + node.val + right_gain
int maxSum = Integer.MIN_VALUE;

int maxPathSum(TreeNode root) {
    maxGain(root);
    return maxSum;
}
int maxGain(TreeNode node) {
    if (node == null) return 0;
    int leftGain  = Math.max(0, maxGain(node.left));   // ignore if negative
    int rightGain = Math.max(0, maxGain(node.right));

    // Update global max: path through current node
    maxSum = Math.max(maxSum, node.val + leftGain + rightGain);

    // Return the max gain if we continue from this node (can only go one direction)
    return node.val + Math.max(leftGain, rightGain);
}
```

### Variation 4: Path Sum III (count paths summing to target, LC 437)

**Problem:** Count the number of paths whose values sum to `targetSum`. The path must go **downward** (parent → child) but can start and end at any node (not necessarily root or leaf).

**Approach (Prefix sum on the root-to-node path + HashMap, top-down DFS with backtracking):**
- Mirrors the **subarray-sum-equals-k** trick, except the "array" is the chain of values from the root to the current node.
- Maintain a running sum `running` from root to current. For a subpath ending at the current node to equal `target`, some ancestor must have prefix `running - target`. The map counts how many times each prefix has been seen on the *current* root-to-node path.
- Critical: after recursing into children, **decrement** that prefix's count (backtracking) so it doesn't leak into sibling paths — only ancestors on the current path are valid starting points.
- Each recursive call returns: count of valid downward paths ending at any node in its subtree.
- Use `long` for `running` because values can be large negatives/positives and overflow `int`.
- Edge case: a single node equal to target → count 1. The seed `map.put(0, 1)` handles paths that start at the root.
- Time O(n) (each node visited once), space O(h) for the map.
- Brute force alternative: at every node call `hasPathFrom(node, target)` — O(n²).

```java
// Use prefix sum technique on tree path
int pathSum(TreeNode root, int target) {
    Map<Long, Integer> prefixCount = new HashMap<>();
    prefixCount.put(0L, 1);
    return dfs(root, 0L, target, prefixCount);
}
int dfs(TreeNode node, long running, int target, Map<Long, Integer> map) {
    if (node == null) return 0;
    running += node.val;
    int count = map.getOrDefault(running - target, 0);
    map.merge(running, 1, Integer::sum);
    count += dfs(node.left,  running, target, map);
    count += dfs(node.right, running, target, map);
    map.merge(running, -1, Integer::sum);   // undo — backtrack
    return count;
}
```

### Questions — Path Problems

| # | Problem | LC# |
|---|---------|-----|
| E1 | Path Sum | 112 |
| E2 | Binary Tree Paths | 257 |
| M1 | Path Sum II (all paths) | 113 |
| M2 | Sum Root to Leaf Numbers | 129 |
| M3 | Path Sum III (any path) | 437 |
| H1 | Binary Tree Maximum Path Sum | 124 |

---

## P4: Tree Properties

### Variation 1: Height / Depth
```java
int height(TreeNode root) {
    if (root == null) return 0;
    return 1 + Math.max(height(root.left), height(root.right));
}
```

### Variation 2: Diameter (LC 543)

**Problem:** The diameter is the **number of edges** on the longest path between any two nodes in the tree. The path may or may not pass through the root. Return that diameter.

**Approach (Bottom-up DFS — height calculation with a side effect):**
- For every node, the longest path **bending at that node** has length `height(left) + height(right)` (in edges).
- Compute heights bottom-up; at each node, update a **global** `diameter = max(diameter, left + right)` before returning.
- Return value: `1 + max(left, right)` — the height contribution if the parent continues downward through this node.
- This is the same "return one thing, record another" pattern as LC 124 max path sum.
- Edge cases: `null` root → diameter 0. Single node → diameter 0 (no edges). Skewed tree of n nodes → diameter n-1.
- Time O(n), space O(h).

```java
// Diameter = longest path between any two nodes (may not pass through root)
// At each node: diameter through it = height(left) + height(right)
int diameter = 0;
int diameterOfBinaryTree(TreeNode root) {
    height(root);
    return diameter;
}
int height(TreeNode node) {
    if (node == null) return 0;
    int left = height(node.left), right = height(node.right);
    diameter = Math.max(diameter, left + right);   // update global
    return 1 + Math.max(left, right);
}
```

### Variation 3: Balanced Binary Tree (LC 110)

**Problem:** A balanced binary tree is one where, for **every** node, the heights of the two subtrees differ by **at most 1**. Return `true` iff the tree is balanced.

**Approach (Bottom-up DFS with sentinel — O(n)):**
- Naive O(n²): call `height` at each node and check the imbalance.
- Smarter: fuse "compute height" and "detect imbalance" into one bottom-up pass. The recursive call returns:
  - the actual height if the subtree is balanced, OR
  - `-1` as a sentinel meaning "subtree is unbalanced — short-circuit upward".
- As soon as any subtree returns `-1`, the parent immediately returns `-1` without further work.
- This is bottom-up: each call **returns info from subtrees**; no parameters are passed down.
- Edge cases: `null` returns height 0 (balanced). Single node → balanced.
- Time O(n), space O(h).

```java
// A tree is balanced if height difference of left and right ≤ 1 for ALL nodes
boolean isBalanced(TreeNode root) {
    return checkHeight(root) != -1;
}
int checkHeight(TreeNode node) {
    if (node == null) return 0;
    int left = checkHeight(node.left);
    if (left == -1) return -1;    // left subtree unbalanced
    int right = checkHeight(node.right);
    if (right == -1) return -1;   // right subtree unbalanced
    if (Math.abs(left - right) > 1) return -1;   // current unbalanced
    return 1 + Math.max(left, right);
}
```

### Variation 4: Symmetric Tree (LC 101)

**Problem:** Return `true` iff the tree is a mirror of itself around its center — i.e., the left subtree is the mirror image of the right subtree.

**Approach (Top-down DFS comparing two pointers simultaneously):**
- Generalize "is this tree symmetric?" into "are these two subtrees mirror images?" — call recursively with `(root.left, root.right)`.
- Two trees `L` and `R` are mirrors iff:
  - both null, OR
  - both non-null and `L.val == R.val` and `mirror(L.left, R.right)` and `mirror(L.right, R.left)` (note the **crossover**: outer-with-outer, inner-with-inner).
- Each call returns a boolean. Two pointers descend in opposite directions — this is top-down with **two** state pointers, no global state.
- Edge cases: `null` root → considered symmetric (`true`). Tree with one node → symmetric. Same value but structurally different (one has a left child, the other has a right child) → false.
- Time O(n), space O(h).
- Iterative variant: use a queue, push pairs `(L, R)`, pop and compare.

```java
boolean isSymmetric(TreeNode root) {
    return isMirror(root.left, root.right);
}
boolean isMirror(TreeNode l, TreeNode r) {
    if (l == null && r == null) return true;
    if (l == null || r == null) return false;
    return l.val == r.val &&
           isMirror(l.left,  r.right) &&
           isMirror(l.right, r.left);
}
```

---

## P5: Lowest Common Ancestor (LCA)

### Variation 1: LCA in Binary Tree (LC 236)

**Problem:** Given a binary tree (no BST property, no parent pointers) and two nodes `p` and `q` guaranteed to exist, return their **lowest common ancestor** — the deepest node that is an ancestor of both.

**Approach (Bottom-up DFS, return signals from subtrees):**
- Each recursive call returns one of three things, treated uniformly as "a candidate ancestor or null":
  - `null` if neither `p` nor `q` lies in this subtree,
  - `p` or `q` if exactly one of them lies here,
  - the LCA node itself once it's been found below.
- Base case: if `root == null || root == p || root == q` → return `root` (we found one of them or hit the bottom).
- Recurse left and right. Then:
  - **Both** non-null → `p` and `q` are split across `root`'s two subtrees → `root` is the LCA.
  - One side non-null → both targets are inside that side (or we've already located the LCA there) → propagate it up.
- Bottom-up: results flow **up** from children; no parameters carry state down.
- Edge cases: one of `p`, `q` is the other's ancestor → that ancestor itself is the LCA (returned at the base case, then never overridden).
- Time O(n), space O(h).

```java
// Key insight: if root is p or q, it's the LCA
// If left and right both return non-null, root is LCA
TreeNode lowestCommonAncestor(TreeNode root, TreeNode p, TreeNode q) {
    if (root == null || root == p || root == q) return root;
    TreeNode left  = lowestCommonAncestor(root.left,  p, q);
    TreeNode right = lowestCommonAncestor(root.right, p, q);
    if (left != null && right != null) return root;   // p and q on opposite sides
    return left != null ? left : right;               // both on same side
}
```

### Variation 2: LCA in BST (LC 235)

**Problem:** Same as LC 236 but the tree is a **BST**. Find the LCA of nodes `p` and `q`.

**Approach (Iterative top-down walk using BST order):**
- BST property: every node in the left subtree is smaller, every node in the right subtree is larger.
- Starting from the root: if **both** `p` and `q` are smaller → LCA must be in the left subtree. If both larger → right subtree. Otherwise (`p` and `q` straddle `root.val`, or one equals it) → `root` is the LCA.
- No recursion needed — iterate down with `root = root.left/right` until the split is found.
- Top-down: state passed down is implicit (we just move the `root` pointer).
- Edge cases: `p == root` or `q == root` → that node is the LCA (one of them being ancestor of the other).
- Time O(h) — O(log n) for balanced, O(n) worst case. Space O(1).

```java
// BST property: if both p,q < root → LCA in left; if both > root → in right
TreeNode lowestCommonAncestor(TreeNode root, TreeNode p, TreeNode q) {
    while (root != null) {
        if (p.val < root.val && q.val < root.val) root = root.left;
        else if (p.val > root.val && q.val > root.val) root = root.right;
        else return root;
    }
    return null;
}
```

### Variation 3: LCA with Parent Pointers

**Problem:** Each node has a `parent` field. Find the LCA of two given nodes without doing a tree-wide DFS.

**Approach (Two-pointer walk-up — "intersection of two lists" trick):**
- With `parent` pointers, the chain of ancestors from any node up to the root is a linked list.
- LCA = first common node of `p`'s ancestor chain and `q`'s ancestor chain.
- Method 1 (HashSet, shown): record all ancestors of `p`; walk up from `q` until you hit one of them.
- Method 2 (O(1) extra space — like LC 160 "Intersection of Two Linked Lists"): two pointers `a = p`, `b = q`; on each step, if a hits `null` redirect to `q`, similarly for `b`. They meet at the LCA after at most `depth(p) + depth(q)` steps.
- This isn't really top-down or bottom-up DFS — it walks **up** along stored parent links.
- Edge cases: `p` is an ancestor of `q` → the set already contains `p`, the walk finds it immediately.
- Time O(h), space O(h) (or O(1) with method 2).

```java
// Find all ancestors of p in a set, then walk up from q until hit that set
Set<TreeNode> ancestors = new HashSet<>();
while (p != null) { ancestors.add(p); p = p.parent; }
while (!ancestors.contains(q)) q = q.parent;
return q;
```

---

## P6: BST Operations

### Key BST Property
> Inorder traversal of BST = sorted array.
> Use this to solve any "kth" problem on BST.

### Variation 1: Validate BST (LC 98)

**Problem:** Determine if a binary tree is a valid BST: every node's value is strictly greater than all values in its left subtree and strictly less than all values in its right subtree (no duplicates).

**Approach (Top-down DFS, propagating `(min, max)` allowed range):**
- Common bug: only comparing `node.val` with `node.left.val` and `node.right.val`. This is **not** sufficient — a node deeper in the left subtree could be larger than the root, violating BST globally but not locally.
- Fix: each node must lie in an **open interval** `(min, max)` that tightens as we descend.
  - Going left: new upper bound becomes the current node's value.
  - Going right: new lower bound becomes the current node's value.
- Top-down: the `(min, max)` is passed **down** via parameters.
- Use `Long.MIN_VALUE / Long.MAX_VALUE` (or wrapper `Integer` nulls) because node values can be `Integer.MIN_VALUE` / `Integer.MAX_VALUE`.
- Each call returns: boolean — "is the entire subtree under me a valid BST under the current bounds?"
- Edge case: empty subtree is valid by definition; duplicates are not allowed in LC's definition (use `<=` / `>=`).
- Time O(n), space O(h).
- Alternative: in-order traversal and check the sequence is strictly increasing (track `prev` value).

```java
// Track min and max allowed values at each node
boolean isValidBST(TreeNode root) {
    return validate(root, Long.MIN_VALUE, Long.MAX_VALUE);
}
boolean validate(TreeNode node, long min, long max) {
    if (node == null) return true;
    if (node.val <= min || node.val >= max) return false;
    return validate(node.left, min, node.val) &&
           validate(node.right, node.val, max);
}
```

### Variation 2: Kth Smallest Element in BST (LC 230)

**Problem:** Given a BST and `k`, return the k-th smallest element (1-indexed).

**Approach (In-order traversal with early termination):**
- BST in-order traversal visits nodes in **ascending order**. The k-th node visited is the k-th smallest.
- Maintain a counter `k`; decrement at each "visit"; when it hits 0, record `node.val` and stop further work.
- Recursion is fine for one query; iterative in-order with an explicit stack is preferred if multiple `kthSmallest` calls or for early-termination clarity.
- Follow-up (frequent at FAANG): "What if the BST is modified frequently and we need many kth-smallest queries?" → augment each node with `count = size of left subtree + 1`. Then locating the k-th becomes O(h). With AVL/RB tree maintenance, updates remain O(log n).
- Edge case: invalid `k` (≤ 0 or > n) — out of contract per LC, but guard if interview pushes for it.
- Time O(h + k), space O(h).

```java
// Inorder gives sorted order — stop at kth element
int k, result;
int kthSmallest(TreeNode root, int k) {
    this.k = k;
    inorder(root);
    return result;
}
void inorder(TreeNode node) {
    if (node == null) return;
    inorder(node.left);
    if (--k == 0) { result = node.val; return; }
    inorder(node.right);
}
```

### Variation 3: Convert Sorted Array to BST (LC 108)

**Problem:** Given an integer array sorted in ascending order, build a **height-balanced** BST.

**Approach (Divide and conquer — top-down recursion on index ranges):**
- A balanced BST results when, at every level, the root partitions the remaining elements roughly evenly.
- Recipe: pick `mid = (left + right) / 2` of the current range as the root; recurse on the left half for `root.left` and right half for `root.right`.
- Top-down: pass index range `(left, right)` down; the call returns the constructed subtree root.
- Picking the exact middle guarantees the height difference between subtrees is ≤ 1 → balanced (LC's definition).
- Variations: picking `mid = (left + right + 1) / 2` (right-biased) also works; both yield valid balanced BSTs but different trees.
- Edge cases: empty array → null tree; single element → leaf node.
- Time O(n), space O(log n) recursion (balanced).

```java
// Always pick the middle element as root — guarantees balanced
TreeNode sortedArrayToBST(int[] nums) {
    return build(nums, 0, nums.length - 1);
}
TreeNode build(int[] nums, int left, int right) {
    if (left > right) return null;
    int mid = left + (right - left) / 2;
    TreeNode node = new TreeNode(nums[mid]);
    node.left  = build(nums, left, mid - 1);
    node.right = build(nums, mid + 1, right);
    return node;
}
```

### Variation 4: Delete Node in BST (LC 450)

**Problem:** Given a BST root and a `key`, delete the node with that value and return the BST root of the modified tree. The result must still be a valid BST.

**Approach (Recursive search + 3-case rewire):**
- **Locate** the node via BST descent (`key < root.val → left`, `key > root.val → right`).
- Once `root.val == key`, three cases:
  1. **No left child** → return `root.right` (could be null) to the caller — effectively unlinking.
  2. **No right child** → return `root.left`.
  3. **Two children** → find the **in-order successor** (the leftmost node in `root.right`). Copy its value to `root`, then recursively delete that successor from the right subtree. This preserves BST ordering because the successor is the smallest value greater than `root.val`.
- Each call returns the (possibly new) subtree root so the parent can reattach correctly via `root.left = ...` / `root.right = ...`.
- This is top-down for the search phase, bottom-up for the rewiring (returning new roots).
- Edge cases: key not found → tree unchanged; deleting the root with one child → the child becomes the new tree root.
- Time O(h), space O(h).

```java
TreeNode deleteNode(TreeNode root, int key) {
    if (root == null) return null;
    if (key < root.val) root.left  = deleteNode(root.left,  key);
    else if (key > root.val) root.right = deleteNode(root.right, key);
    else {
        if (root.left == null)  return root.right;
        if (root.right == null) return root.left;
        // Replace with inorder successor (leftmost node in right subtree)
        TreeNode successor = root.right;
        while (successor.left != null) successor = successor.left;
        root.val = successor.val;
        root.right = deleteNode(root.right, successor.val);
    }
    return root;
}
```

### BST Questions

| # | Problem | LC# |
|---|---------|-----|
| E1 | Search in a BST | 700 |
| E2 | Insert into a BST | 701 |
| M1 | Validate Binary Search Tree | 98 |
| M2 | Kth Smallest Element in a BST | 230 |
| M3 | Convert Sorted Array to BST | 108 |
| M4 | Delete Node in a BST | 450 |
| M5 | Recover Binary Search Tree (two nodes swapped) | 99 |
| M6 | Binary Search Tree Iterator | 173 |
| H1 | Count of Smaller Numbers After Self | 315 |

---

## P7: Tree Construction

### Variation 1: Build from Preorder + Inorder (LC 105)

**Problem:** Given two arrays `preorder` and `inorder` representing the traversals of a unique binary tree (all values distinct), reconstruct and return the tree.

**Approach (Divide and conquer, top-down construction):**
- **Preorder's first element is the root** of the current subtree. The same value's position in `inorder` splits inorder into the left subtree (everything before the index) and the right subtree (everything after).
- The **size of the left subtree** is `inorderRootIndex - inorderLeft`. That tells us how many elements of preorder belong to the left subtree.
- Recurse on the two halves to build `root.left` and `root.right`.
- Use a `HashMap<value → inorder index>` so the "find root in inorder" step is O(1); otherwise the algorithm is O(n²).
- A single advancing `preIdx` pointer (instead of computing preorder bounds explicitly) is the cleanest implementation — it naturally walks preorder root → left subtree → right subtree.
- Top-down: bounds are passed down via parameters; the call returns the constructed subtree root.
- Edge cases: empty arrays → null tree; all values must be distinct (a stated input constraint).
- Time O(n), space O(n) for the map + O(h) recursion.

```java
// Preorder[0] = root. Find root in inorder → left/right sizes.
TreeNode buildTree(int[] preorder, int[] inorder) {
    Map<Integer, Integer> inorderMap = new HashMap<>();
    for (int i = 0; i < inorder.length; i++) inorderMap.put(inorder[i], i);
    return build(preorder, 0, preorder.length - 1, inorder, 0, inorder.length - 1, inorderMap);
}
int preIdx = 0;
TreeNode build(int[] pre, int preL, int preR, int[] in, int inL, int inR, Map<Integer,Integer> map) {
    if (preL > preR) return null;
    int rootVal = pre[preIdx++];
    TreeNode root = new TreeNode(rootVal);
    int mid = map.get(rootVal);
    int leftSize = mid - inL;
    root.left  = build(pre, preL+1, preL+leftSize, in, inL, mid-1, map);
    root.right = build(pre, preL+leftSize+1, preR, in, mid+1, inR, map);
    return root;
}
```

### Variation 2: Build from Inorder + Postorder (LC 106)

**Problem:** Given `inorder` and `postorder` traversals of a unique binary tree, reconstruct it.

**Approach:**
- **Postorder's *last* element is the root** (mirror of preorder's *first*). Locate it in `inorder` to split into left/right subtrees.
- Walk postorder **right to left**: root first, then build the **right** subtree first (because postorder is `L, R, Root` — reading from the end gives Root, then R...), then the left.
- Same `HashMap<value → inorder index>` trick for O(1) lookup.
- Why this works: in postorder, after the root the immediately preceding region is the entire right subtree (in postorder), then the left subtree (in postorder). Slicing by inorder's split tells us where the boundary is.
- Time O(n), space O(n).

> Same idea — postorder's last element is the root.

---

## P8: Serialize / Deserialize (LC 297) — FAANG Hard

**Problem:** Design an algorithm to serialize a binary tree to a string and deserialize that string back into the original tree. The format is up to you.

**Approach (Preorder DFS with null markers):**
- A single traversal sequence (e.g., preorder alone) is **not** enough to reconstruct a tree — you also need to know where missing children are. The classic fix is to emit a sentinel (e.g., `"null"`) whenever you'd recurse into a `null` child.
- **Serialize (top-down recursive preorder):** append `root.val,`, then recurse into left, then right. For null nodes append `"null,"` and return.
- **Deserialize (top-down recursive consume):** split the string on `,` into a `Deque<String>` (FIFO). Each `buildTree` call polls the next token: if `"null"`, return null; otherwise create a node and recursively build `left` then `right` from the queue. The recursion structure mirrors the serializer exactly, so the queue is consumed in the right order automatically.
- Both passes are top-down: serialize writes as it descends; deserialize reads as it descends.
- Each `serialize` call returns the string for the subtree; each `buildTree` call returns the reconstructed subtree's root.
- Edge cases: empty tree → just `"null,"`; values can be negative — split on commas, not by sign.
- Time O(n), space O(n) for the string + O(h) recursion stack.
- Alternative formats: level-order (BFS) with null markers — same idea, queue-based; useful when you want a more "human-readable" string.

```java
// Serialize: preorder DFS, use "null" markers
String serialize(TreeNode root) {
    if (root == null) return "null,";
    return root.val + "," + serialize(root.left) + serialize(root.right);
}
// Deserialize: consume queue in same preorder
TreeNode deserialize(String data) {
    Deque<String> queue = new ArrayDeque<>(Arrays.asList(data.split(",")));
    return buildTree(queue);
}
TreeNode buildTree(Deque<String> queue) {
    String val = queue.poll();
    if ("null".equals(val)) return null;
    TreeNode node = new TreeNode(Integer.parseInt(val));
    node.left  = buildTree(queue);
    node.right = buildTree(queue);
    return node;
}
```

---

## Complete FAANG Tree Problem List

| Problem | Pattern | LC# | Difficulty |
|---------|---------|-----|------------|
| Inorder Traversal | DFS | 94 | E |
| Symmetric Tree | DFS Mirror | 101 | E |
| Maximum Depth of Binary Tree | DFS | 104 | E |
| Convert Sorted Array to BST | BST | 108 | E |
| Balanced Binary Tree | DFS | 110 | E |
| Binary Tree Level Order Traversal | BFS | 102 | M |
| Binary Tree Zigzag Level Order | BFS | 103 | M |
| Validate BST | BST | 98 | M |
| Flatten Binary Tree to Linked List | DFS | 114 | M |
| Path Sum II | Path | 113 | M |
| Binary Tree Right Side View | BFS | 199 | M |
| Kth Smallest in BST | BST + Inorder | 230 | M |
| LCA of Binary Tree | LCA | 236 | M |
| Delete Node in BST | BST | 450 | M |
| Path Sum III | Path + Prefix Sum | 437 | M |
| Diameter of Binary Tree | Properties | 543 | M |
| House Robber III | DP on Tree | 337 | M |
| Construct Tree from Pre+Inorder | Construction | 105 | M |
| Binary Tree Maximum Path Sum | Path | 124 | H |
| Serialize and Deserialize Binary Tree | Advanced | 297 | H |
| Recover BST (two nodes swapped) | BST | 99 | H |
| Binary Tree Cameras | Greedy on Tree | 968 | H |

---

## Decision Guide

```
Tree problem?
│
├─ "Traverse" / "list all"         → DFS (recursive) or BFS (level order)
├─ "Level by level"                → BFS
├─ "Path from root to leaf"        → DFS preorder
├─ "Any path max sum"              → DFS postorder (bottom-up)
├─ "Height / diameter"             → DFS postorder
├─ "LCA"                           → DFS (post-order logic)
├─ "BST + sorted / kth"            → Inorder traversal
├─ "Build tree"                    → Divide & conquer on preorder/inorder
└─ "Count ways / DP on tree"       → Postorder DP
```

---

> Back to [Master Index](../DSA_MASTER_INDEX.md)
