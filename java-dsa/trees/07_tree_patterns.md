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
```java
boolean hasPathSum(TreeNode root, int target) {
    if (root == null) return false;
    if (root.left == null && root.right == null) return root.val == target;
    return hasPathSum(root.left, target - root.val) ||
           hasPathSum(root.right, target - root.val);
}
```

### Variation 2: All Root-to-Leaf Paths (LC 257)
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
> Same idea — postorder's last element is the root.

---

## P8: Serialize / Deserialize (LC 297) — FAANG Hard

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
