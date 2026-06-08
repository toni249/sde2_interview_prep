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

**Problem:** Given an integer array `nums` with all distinct elements, return all possible subsets (the power set). Order of output doesn't matter; no duplicates.

**Approach (Include/exclude choice tree):**
- **Choice tree:** at each index, two choices — include `nums[i]` or skip it. That gives 2^n leaves.
- We use a `start` index so subsequent recursion only considers `i ≥ start` — this enforces sorted order on subsets and prevents permutations of the same subset.
- **Base case:** none — every node of the recursion tree is itself a valid subset. So we `result.add(...)` at the top of every call (not just at depth n).
- **Backtrack step:** add → recurse → remove (canonical choose/unchoose).
- **Key pitfall:** must copy `path` (`new ArrayList<>(path)`) — `path` is mutated as we backtrack.
- Time: O(n · 2^n) — 2^n subsets × O(n) to copy each | Space: O(n) recursion depth.

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

**Problem:** Given an integer array `nums` that may contain duplicates, return all possible **unique** subsets.

**Approach (Sort + skip-duplicate-at-same-level):**
- **Sort first** so duplicates are adjacent.
- **Pruning rule:** at the **same recursion depth**, never start with a value identical to the one we just tried — that would generate a subset already produced.
- The condition `if (i > start && nums[i] == nums[i-1]) continue;` is the standard idiom:
  - `i == start` → first pick at this level, always allowed (even if duplicate of parent).
  - `i > start` AND same as previous → sibling duplicate → skip.
- Why `i > start` (not `i > 0`)? Because picking the same value at deeper levels is fine (e.g., `[1,1]` is a valid subset of `[1,1,2]`); we only need to avoid duplicates at the *same* level.
- Time: O(n · 2^n) worst case | Space: O(n).

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

**Problem:** Given an array `nums` of distinct integers, return all possible permutations (n! total).

**Approach (Used-array choice tree):**
- **Choice tree:** at each level, pick any *unused* index (not constrained by `start` because order matters).
- **Base case:** `path.size() == n` → push a copy.
- Track availability with a `boolean[] used` array; mark `used[i] = true` before recursing, false after (classic choose/unchoose).
- **No `start` index** here — at every level we may choose any remaining element (that's what makes it n! not 2^n).
- Alternative: swap-in-place — iterate `i` from `start` to `n-1`, swap `nums[start]` with `nums[i]`, recurse with `start+1`, swap back. Saves the `used` array but mutates input.
- Time: O(n · n!) (n! permutations × O(n) copy) | Space: O(n) recursion + O(n) used array.

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

**Problem:** Array `nums` may contain duplicates. Return all **unique** permutations.

**Approach (Sort + canonical-pick rule):**
- **Sort** so duplicates are adjacent.
- **Pruning rule:** when several positions hold the same value, always use them in **left-to-right order**. Concretely: skip `nums[i]` if `nums[i] == nums[i-1]` AND `used[i-1] == false` (meaning we previously chose to skip the identical earlier copy at this level — so picking this one would just produce a permutation we already produced via the other branch).
- Why `!used[i-1]`? When `used[i-1] == true`, the earlier copy is "consumed" by an ancestor in the recursion, so picking `i` is a genuinely new permutation, not a duplicate.
- Time: O(n · n!) worst case (much better in practice due to pruning) | Space: O(n).

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

**Problem:** Rearrange `nums` into the lexicographically next-greater permutation in-place. If it's already the largest permutation (descending), wrap around to the smallest (ascending). O(1) extra space.

**Approach (Three-step trick):**
- **Step 1 — Find the pivot:** Scan from the right, find the first index `i` such that `nums[i] < nums[i+1]`. Everything to the right of `i` is in descending order (a "suffix" that can't grow further on its own).
- **Step 2 — Swap:** Find the smallest element in the descending suffix that is still larger than `nums[i]` (scan from right, first `j` with `nums[j] > nums[i]`). Swap `nums[i]` and `nums[j]`.
- **Step 3 — Reverse suffix:** After the swap the suffix is still descending; reverse it to make it ascending → smallest possible suffix → exact next permutation.
- Edge case: no pivot found (array is fully descending) → skip step 2, just reverse whole array → wraps to smallest permutation.
- Time: O(n) | Space: O(1). Not actually backtracking — included here because it's a permutation classic.

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

**Problem:** Given an array `candidates` of distinct positive integers and a target. Return all **unique combinations** that sum to target. Each candidate may be used **unlimited** times. Combinations are sets (order doesn't matter).

**Approach (Start-index + same-index recurse):**
- **Choice tree:** at each call, choose any candidate from index `start` onwards (enforces sorted order in output → no duplicate combinations).
- **Reuse trick:** when recursing after picking `cands[i]`, pass `i` (not `i+1`) so the same element can be picked again.
- **Base case:** `remaining == 0` → push a copy of path.
- **Pruning:** sort candidates ascending; then `if (cands[i] > remaining) break` — since the array is sorted, no later candidate can fit either.
- Edge cases: empty result if no combination sums to target; target == 0 (single empty combo).
- Time: O(N^(T/M)) where T = target, M = min(candidates) — exponential but pruning helps massively | Space: O(T/M).

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

**Problem:** Input may contain duplicates. Each element may be used **at most once**. Return all unique combinations summing to target.

**Approach (Sort + skip-at-level + i+1 recurse):**
- **Sort** the array to group duplicates.
- Use the same start-index pattern as LC 39, but recurse with `i + 1` (no reuse).
- **Duplicate suppression** (same as Subsets II): `if (i > start && cands[i] == cands[i-1]) continue;` — at the same recursion level, don't start with a value identical to the previous sibling.
- **Pruning:** sorted + `if (cands[i] > remaining) break`.
- This combines two pruning techniques: same-level dedup (for distinct output) + sorted break (for efficiency).
- Time: O(2^n) worst case, much better with pruning | Space: O(n).

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

**Problem:** Find all unique combinations of exactly `k` numbers from 1..9 that sum to `n`. Each digit used at most once.

**Approach (Bounded combination with two stop conditions):**
- **Choice tree:** at each step pick a digit from `start..9`; recurse with `start = digit + 1`.
- **Base cases (two):**
  - `path.size() == k && remaining == 0` → valid combination, push copy.
  - `path.size() == k || remaining < 0` → invalid branch, return.
- **Pruning:** since digits are 1..9 in order, break early when `digit > remaining`.
- Search space is tiny (9 digits) so brute enumeration is fine; pruning is just for elegance.
- Time: O(C(9, k)) | Space: O(k).

> Choose exactly k numbers from 1-9 summing to n.

### Variation 4: Letter Combinations of Phone Number (LC 17)

**Problem:** Given a string of digits 2–9 (e.g., "23"), return all possible letter combinations the digits could represent using the classic phone keypad mapping. "23" → ["ad","ae","af","bd","be","bf","cd","ce","cf"].

**Approach (Cartesian-product backtracking):**
- **Choice tree:** at depth `idx`, the choices are the 3–4 letters mapped to `digits[idx]`. Branching factor = letters per digit (3 or 4).
- **Base case:** `idx == digits.length()` → push current string.
- Use a `StringBuilder` and append/delete-last for O(1) choose/unchoose (vs string concatenation which is O(n)).
- This is a clean Cartesian product implementation; iterative BFS using a queue is an alternative.
- Edge case: empty input → return empty list (not `[""]`).
- Time: O(4^n · n) where n = digits.length | Space: O(n) recursion + output.

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

**Problem:** Partition string `s` such that every substring of the partition is a palindrome. Return all possible palindromic partitionings.

**Approach (Choose-a-cut backtracking):**
- **Choice tree:** at position `start`, try every possible **end** position; the substring `s[start..end-1]` becomes a partition piece **only if** it is a palindrome (this is the pruning gate).
- **Base case:** `start == s.length()` → push a copy of the partition.
- **Pruning:** `if (isPalin(s, start, end-1))` — skip the entire subtree if the candidate piece isn't a palindrome.
- **Optimization:** precompute an `isPalin[i][j]` DP table in O(n²) to answer palindrome checks in O(1). Without DP, each check is O(n).
- Edge cases: empty string (single empty partition); single character (always palindrome).
- Time: O(n · 2^n) — 2^n partition points × O(n) palindrome check | Space: O(n) recursion.

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

**Problem:** Given an `m × n` board of characters and a `word`, return true if `word` can be constructed from sequentially adjacent (up/down/left/right) cells where the same cell is not used more than once.

**Approach (DFS + in-place visited marker):**
- **Choice tree:** from every starting cell, DFS in 4 directions matching the next character of `word`.
- **Base case:** `k == word.length()` → matched all → return true.
- **Pruning (early termination):**
  - Out-of-bounds → return false.
  - `board[i][j] != word.charAt(k)` → wrong char, dead branch.
- **Visited tracking:** overwrite the visited cell with a sentinel (`'#'`) and restore on backtrack. Saves an O(mn) visited array and naturally fails the char-mismatch check during the current path.
- Short-circuit OR across 4 directions — stop as soon as any direction succeeds.
- Time: O(m·n · 4^L) where L = word length | Space: O(L) recursion.

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

**Problem:** Place `n` queens on an `n × n` chessboard so that no two queens attack each other (share a row, column, or diagonal). Return all distinct solutions.

**Approach (Row-by-row placement with constraint check):**
- **Choice tree:** process **one row at a time** — pick a column for the queen in this row, then recurse to the next row. This automatically prevents two queens sharing a row.
- **Base case:** `row == n` → board is complete → save it.
- **`isValid(row, col)` check:** scan all previous rows for conflicts in (a) same column, (b) left diagonal `(r-1, c-1)`, (c) right diagonal `(r-1, c+1)`. No need to check the current row.
- **Pruning:** if invalid for column, skip — don't recurse. This is the entire reason N-Queens is tractable.
- **Optimization:** maintain three boolean sets — `cols[]`, `diag1[]` (indexed by `row + col`), `diag2[]` (indexed by `row - col + n`) — for O(1) validity. Reduces total work dramatically.
- Time: ~O(n!) (very pruned in practice) | Space: O(n²) for the board.

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

**Problem:** Solve a 9×9 Sudoku puzzle in-place. Empty cells are marked `'.'`. Each row, each column, and each of the nine 3×3 sub-boxes must contain digits 1–9 exactly once. The puzzle is guaranteed to have a unique solution.

**Approach (Cell-by-cell backtracking with three-fold constraint):**
- **Choice tree:** find the next empty cell; for each digit `1..9`, check (a) row, (b) column, (c) 3×3 sub-box for conflicts; if valid, place the digit and recurse.
- **Base case:** scanned all 81 cells with no empty → return true (solution found).
- **Pruning:** if no digit works for the current cell, return false → caller un-places its digit and tries another.
- **Box index:** for cell `(r, c)`, the box's top-left corner is `(3*(r/3), 3*(c/3))`. Iterate offsets `i = 0..8` → check cell `(3*(r/3) + i/3, 3*(c/3) + i%3)`.
- **Optimization:** maintain three `boolean[9][9]` arrays (rowUsed, colUsed, boxUsed) for O(1) validity. Further speedups: pick the cell with the fewest legal candidates first ("most-constrained-variable" / MRV heuristic).
- Time: O(9^(empty cells)) worst case, very fast in practice with constraints | Space: O(1) board is in place; O(81) recursion.

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
