# Recursion & Backtracking — Pattern Deep Dive
> FAANG Level | Java | Easy → Hard

---

## Pattern Map

```
BACKTRACKING
├── P1: Subsets              → all subsets, subsets with duplicates
├── P2: Permutations         → all permutations, with duplicates
├── P3: Combinations         → combination sum, phone number letters
├── P4: Grid Backtracking    → word search, N-Queens, Sudoku, rat in maze
└── P5: Pruning Strategies   → how to make backtracking efficient
```

---

## The Backtracking Template

```java
void backtrack(result, currentPath, choices) {
    if (baseCase) {
        result.add(new ArrayList<>(currentPath));  // ← always copy!
        return;
    }
    for (choice : choices) {
        if (isValid(choice)) {
            currentPath.add(choice);       // choose
            backtrack(result, currentPath, remaining choices);
            currentPath.remove(last);      // unchoose (undo)
        }
    }
}
```
> **The #1 mistake:** not copying `currentPath` when adding to result. Pass `new ArrayList<>(currentPath)`.

---

## P1: Subsets

### Variation 1: All Subsets — No Duplicates (LC 78)
```java
List<List<Integer>> subsets(int[] nums) {
    List<List<Integer>> result = new ArrayList<>();
    backtrack(nums, 0, new ArrayList<>(), result);
    return result;
}
void backtrack(int[] nums, int start, List<Integer> path, List<List<Integer>> result) {
    result.add(new ArrayList<>(path));    // add at every node, not just leaf
    for (int i = start; i < nums.length; i++) {
        path.add(nums[i]);
        backtrack(nums, i + 1, path, result);
        path.remove(path.size() - 1);
    }
}
// Time: O(n × 2^n) | Space: O(n) recursion depth
```

### Recursion Tree Visualization
```
subsets([1,2,3]):
                    []
           /        |        \
         [1]       [2]       [3]
        /   \       |
     [1,2] [1,3]  [2,3]
       |
    [1,2,3]

Result: [], [1], [1,2], [1,2,3], [1,3], [2], [2,3], [3]  → 2^3 = 8 subsets
```

### Variation 2: Subsets with Duplicates (LC 90)
```java
// Sort first, then skip duplicates at the same level
List<List<Integer>> subsetsWithDup(int[] nums) {
    Arrays.sort(nums);                      // ← must sort first
    List<List<Integer>> result = new ArrayList<>();
    backtrack(nums, 0, new ArrayList<>(), result);
    return result;
}
void backtrack(int[] nums, int start, List<Integer> path, List<List<Integer>> result) {
    result.add(new ArrayList<>(path));
    for (int i = start; i < nums.length; i++) {
        if (i > start && nums[i] == nums[i-1]) continue;  // skip duplicate at same level
        path.add(nums[i]);
        backtrack(nums, i + 1, path, result);
        path.remove(path.size() - 1);
    }
}
```
> Key: `i > start` (not `i > 0`). This allows the same value in a deeper level but not at the same sibling level.

---

## P2: Permutations

### Variation 1: All Permutations — No Duplicates (LC 46)
```java
List<List<Integer>> permute(int[] nums) {
    List<List<Integer>> result = new ArrayList<>();
    backtrack(nums, new boolean[nums.length], new ArrayList<>(), result);
    return result;
}
void backtrack(int[] nums, boolean[] used, List<Integer> path, List<List<Integer>> result) {
    if (path.size() == nums.length) {
        result.add(new ArrayList<>(path));
        return;
    }
    for (int i = 0; i < nums.length; i++) {
        if (used[i]) continue;
        used[i] = true;
        path.add(nums[i]);
        backtrack(nums, used, path, result);
        path.remove(path.size() - 1);
        used[i] = false;
    }
}
```

### Variation 2: Permutations with Duplicates (LC 47)
```java
// Sort + skip same value if previous identical element was NOT used
void backtrack(int[] nums, boolean[] used, List<Integer> path, List<List<Integer>> result) {
    if (path.size() == nums.length) {
        result.add(new ArrayList<>(path)); return;
    }
    for (int i = 0; i < nums.length; i++) {
        if (used[i]) continue;
        // Skip if same value and previous wasn't used (would create duplicate permutation)
        if (i > 0 && nums[i] == nums[i-1] && !used[i-1]) continue;
        used[i] = true;
        path.add(nums[i]);
        backtrack(nums, used, path, result);
        path.remove(path.size() - 1);
        used[i] = false;
    }
}
```
> The condition `!used[i-1]` ensures we always pick duplicates in order (left before right), eliminating duplicates.

### Variation 3: Next Permutation (LC 31) — No Backtracking, Just Observation
```java
void nextPermutation(int[] nums) {
    int n = nums.length, i = n - 2;
    // Step 1: Find first decreasing element from right
    while (i >= 0 && nums[i] >= nums[i + 1]) i--;
    if (i >= 0) {
        // Step 2: Find smallest element to its right that is larger
        int j = n - 1;
        while (nums[j] <= nums[i]) j--;
        swap(nums, i, j);
    }
    // Step 3: Reverse the suffix
    int left = i + 1, right = n - 1;
    while (left < right) swap(nums, left++, right--);
}
```

---

## P3: Combinations

### Variation 1: Combination Sum (LC 39) — Unlimited Reuse
```java
// Can reuse the same element
List<List<Integer>> combinationSum(int[] candidates, int target) {
    List<List<Integer>> result = new ArrayList<>();
    Arrays.sort(candidates);
    backtrack(candidates, target, 0, new ArrayList<>(), result);
    return result;
}
void backtrack(int[] cands, int remaining, int start, List<Integer> path, List<List<Integer>> result) {
    if (remaining == 0) { result.add(new ArrayList<>(path)); return; }
    for (int i = start; i < cands.length; i++) {
        if (cands[i] > remaining) break;     // pruning: sorted, so no need to continue
        path.add(cands[i]);
        backtrack(cands, remaining - cands[i], i, path, result);  // i, not i+1 (reuse allowed)
        path.remove(path.size() - 1);
    }
}
```

### Variation 2: Combination Sum II (LC 40) — No Reuse, No Duplicates
```java
// Each element used once, duplicates in input but not in output
void backtrack(int[] cands, int remaining, int start, List<Integer> path, List<List<Integer>> result) {
    if (remaining == 0) { result.add(new ArrayList<>(path)); return; }
    for (int i = start; i < cands.length; i++) {
        if (i > start && cands[i] == cands[i-1]) continue;  // skip duplicates at same level
        if (cands[i] > remaining) break;
        path.add(cands[i]);
        backtrack(cands, remaining - cands[i], i + 1, path, result);  // i+1, no reuse
        path.remove(path.size() - 1);
    }
}
```

### Variation 3: Combination Sum III (LC 216)
> Choose exactly k numbers from 1-9 summing to n.

### Variation 4: Letter Combinations of Phone Number (LC 17)
```java
String[] map = {"", "", "abc", "def", "ghi", "jkl", "mno", "pqrs", "tuv", "wxyz"};
List<String> letterCombinations(String digits) {
    List<String> result = new ArrayList<>();
    if (digits.isEmpty()) return result;
    backtrack(digits, 0, new StringBuilder(), result);
    return result;
}
void backtrack(String digits, int idx, StringBuilder path, List<String> result) {
    if (idx == digits.length()) { result.add(path.toString()); return; }
    for (char c : map[digits.charAt(idx) - '0'].toCharArray()) {
        path.append(c);
        backtrack(digits, idx + 1, path, result);
        path.deleteCharAt(path.length() - 1);
    }
}
```

### Variation 5: Palindrome Partitioning (LC 131)
```java
// Partition string so every substring is a palindrome
List<List<String>> partition(String s) {
    List<List<String>> result = new ArrayList<>();
    backtrack(s, 0, new ArrayList<>(), result);
    return result;
}
void backtrack(String s, int start, List<String> path, List<List<String>> result) {
    if (start == s.length()) { result.add(new ArrayList<>(path)); return; }
    for (int end = start + 1; end <= s.length(); end++) {
        if (isPalin(s, start, end - 1)) {
            path.add(s.substring(start, end));
            backtrack(s, end, path, result);
            path.remove(path.size() - 1);
        }
    }
}
```

---

## P4: Grid Backtracking

### Variation 1: Word Search (LC 79)
```java
boolean exist(char[][] board, String word) {
    int m = board.length, n = board[0].length;
    for (int i = 0; i < m; i++)
        for (int j = 0; j < n; j++)
            if (dfs(board, word, i, j, 0)) return true;
    return false;
}
boolean dfs(char[][] board, String word, int i, int j, int k) {
    if (k == word.length()) return true;
    if (i < 0 || i >= board.length || j < 0 || j >= board[0].length) return false;
    if (board[i][j] != word.charAt(k)) return false;

    char tmp = board[i][j];
    board[i][j] = '#';    // mark as visited (in-place — no extra space)
    boolean found = dfs(board, word, i+1, j, k+1) ||
                    dfs(board, word, i-1, j, k+1) ||
                    dfs(board, word, i, j+1, k+1) ||
                    dfs(board, word, i, j-1, k+1);
    board[i][j] = tmp;    // restore
    return found;
}
```

### Variation 2: N-Queens (LC 51) — FAANG Classic
```java
List<List<String>> solveNQueens(int n) {
    List<List<String>> result = new ArrayList<>();
    char[][] board = new char[n][n];
    for (char[] row : board) Arrays.fill(row, '.');
    backtrack(board, 0, result);
    return result;
}
void backtrack(char[][] board, int row, List<List<String>> result) {
    if (row == board.length) {
        result.add(constructBoard(board)); return;
    }
    for (int col = 0; col < board.length; col++) {
        if (isValid(board, row, col)) {
            board[row][col] = 'Q';
            backtrack(board, row + 1, result);
            board[row][col] = '.';
        }
    }
}
boolean isValid(char[][] board, int row, int col) {
    int n = board.length;
    for (int i = 0; i < row; i++) if (board[i][col] == 'Q') return false;         // column
    for (int i = row-1, j = col-1; i >= 0 && j >= 0; i--, j--)
        if (board[i][j] == 'Q') return false;      // left diagonal
    for (int i = row-1, j = col+1; i >= 0 && j < n; i--, j++)
        if (board[i][j] == 'Q') return false;      // right diagonal
    return true;
}
```
> Optimization: Use 3 boolean arrays (col, diag1, diag2) for O(1) validity check.

### Variation 3: Sudoku Solver (LC 37)
```java
boolean solveSudoku(char[][] board) {
    for (int i = 0; i < 9; i++) {
        for (int j = 0; j < 9; j++) {
            if (board[i][j] != '.') continue;
            for (char c = '1'; c <= '9'; c++) {
                if (isValid(board, i, j, c)) {
                    board[i][j] = c;
                    if (solveSudoku(board)) return true;
                    board[i][j] = '.';
                }
            }
            return false;    // no valid digit — backtrack
        }
    }
    return true;    // all cells filled
}
boolean isValid(char[][] board, int row, int col, char c) {
    for (int i = 0; i < 9; i++) {
        if (board[row][i] == c) return false;           // row check
        if (board[i][col] == c) return false;           // col check
        // 3×3 box check
        if (board[3*(row/3) + i/3][3*(col/3) + i%3] == c) return false;
    }
    return true;
}
```

---

## P5: Pruning Strategies

### Why Pruning Matters
Without pruning, backtracking is brute-force exponential.
Good pruning can reduce from `O(n!)` to near-polynomial.

### Pruning Techniques

**1. Early termination:** Sort input, break when current choice exceeds remaining target.
```java
if (cands[i] > remaining) break;   // since sorted, rest also too large
```

**2. Deduplication at same level:** Skip duplicate choices at the same recursive level.
```java
if (i > start && nums[i] == nums[i-1]) continue;
```

**3. Constraint checking before recursing:** Don't recurse if current state is already invalid.
```java
if (!isValid(board, row, col)) continue;   // check BEFORE placing
```

**4. Bounding (for optimization problems):** If current partial solution already worse than best, stop.
```java
if (currentCost >= bestCost) return;
```

---

## Questions — Complete List

| Problem | Pattern | LC# | Difficulty |
|---------|---------|-----|------------|
| Subsets | P1 | 78 | M |
| Subsets II (with duplicates) | P1 | 90 | M |
| Permutations | P2 | 46 | M |
| Permutations II (with duplicates) | P2 | 47 | M |
| Next Permutation | Observation | 31 | M |
| Combination Sum | P3 | 39 | M |
| Combination Sum II | P3 | 40 | M |
| Combination Sum III | P3 | 216 | M |
| Letter Combinations of Phone Number | P3 | 17 | M |
| Palindrome Partitioning | P3 | 131 | M |
| Generate Parentheses | P3 | 22 | M |
| Word Search | P4 | 79 | M |
| N-Queens | P4 | 51 | H |
| N-Queens II (count) | P4 | 52 | H |
| Sudoku Solver | P4 | 37 | H |
| Expression Add Operators | P3 | 282 | H |
| Word Search II (Trie + BT) | P4 + Trie | 212 | H |

---

## Subsets vs Permutations vs Combinations — Key Difference

```
Subsets:       Order doesn't matter, each element used ≤ once
               → start index increases, no visited array needed

Permutations:  Order matters, each element used exactly once
               → no start index, use visited[] array

Combinations:  Order doesn't matter, pick exactly k elements
               → start index increases, stop when path.size() == k
```

---

> Back to [Master Index](../DSA_MASTER_INDEX.md)
