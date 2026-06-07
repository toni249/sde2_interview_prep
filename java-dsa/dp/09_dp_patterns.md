# Dynamic Programming — Pattern Deep Dive
> FAANG Level | Java | Aditya Verma Style
> You said this is a weak area — go through EVERY variation carefully.

---

## The DP Framework — Always Follow This

```
Step 1: IDENTIFY    → Can I define the solution of a subproblem in terms of smaller subproblems?
Step 2: RECURSIVE   → Write the plain recursive solution. Get it working.
Step 3: MEMOIZE     → Add a memo table. Top-down DP.
Step 4: TABULATE    → Convert to bottom-up table. Easier to optimize.
Step 5: OPTIMIZE    → Can I reduce space? (Often from 2D → 1D → 2 variables)
```

---

## Pattern Map

```
DP
├── F1: 0/1 Knapsack Family       → choose or skip each item
├── F2: Unbounded Knapsack Family → unlimited reuse of items
├── F3: LCS Family                → two sequences, common subsequence
├── F4: LIS Family                → increasing subsequence
├── F5: MCM / Interval DP Family  → split/merge problems
├── F6: Grid DP                   → paths in a 2D grid
├── F7: DP on Trees               → postorder DP
├── F8: State Machine DP          → buy/sell stocks, cooldown
└── F9: Bitmask DP                → n ≤ 20, assignments
```

---

## F1: 0/1 Knapsack Family

### The Core Problem
> Given n items each with weight[i] and value[i].
> Knapsack capacity = W. Maximize value without exceeding capacity.
> Each item used AT MOST ONCE.

### Step 1: Recursive
```java
int knapsack(int[] wt, int[] val, int W, int n) {
    if (n == 0 || W == 0) return 0;                          // base case
    if (wt[n-1] > W) return knapsack(wt, val, W, n-1);      // can't include
    return Math.max(
        val[n-1] + knapsack(wt, val, W - wt[n-1], n-1),     // include
        knapsack(wt, val, W, n-1)                             // exclude
    );
}
```

### Step 2: Memoize (Top-Down)
```java
int[][] dp = new int[n+1][W+1];   // initialized to -1
int knapsackMemo(int[] wt, int[] val, int W, int n) {
    if (n == 0 || W == 0) return 0;
    if (dp[n][W] != -1) return dp[n][W];
    if (wt[n-1] > W) return dp[n][W] = knapsackMemo(wt, val, W, n-1);
    return dp[n][W] = Math.max(
        val[n-1] + knapsackMemo(wt, val, W - wt[n-1], n-1),
        knapsackMemo(wt, val, W, n-1)
    );
}
```

### Step 3: Tabulate (Bottom-Up) — FAANG prefers this
```java
int knapsackDP(int[] wt, int[] val, int W, int n) {
    int[][] dp = new int[n+1][W+1];
    // dp[i][w] = max value using first i items with capacity w
    for (int i = 1; i <= n; i++) {
        for (int w = 0; w <= W; w++) {
            dp[i][w] = dp[i-1][w];   // exclude item i
            if (wt[i-1] <= w)
                dp[i][w] = Math.max(dp[i][w], val[i-1] + dp[i-1][w - wt[i-1]]);
        }
    }
    return dp[n][W];
}
```

### Step 4: Optimize Space (1D array)
```java
// When computing dp[i][w], we only need dp[i-1][...]
// Traverse w from RIGHT to LEFT to avoid using updated values
int knapsackOptimized(int[] wt, int[] val, int W) {
    int[] dp = new int[W+1];
    for (int i = 0; i < wt.length; i++) {
        for (int w = W; w >= wt[i]; w--) {   // ← RIGHT TO LEFT
            dp[w] = Math.max(dp[w], val[i] + dp[w - wt[i]]);
        }
    }
    return dp[W];
}
```

---

### Variation 1: Subset Sum (LC style)
> Can we find a subset with exactly sum = target?
> **Change:** weight = value = nums[i], capacity = target, optimize for boolean.

```java
boolean subsetSum(int[] nums, int target) {
    boolean[] dp = new boolean[target + 1];
    dp[0] = true;
    for (int num : nums) {
        for (int w = target; w >= num; w--) {   // RIGHT TO LEFT
            dp[w] = dp[w] || dp[w - num];
        }
    }
    return dp[target];
}
```

### Variation 2: Equal Partition (LC 416)
> Can we partition array into two equal subsets?
> **Change:** Target = totalSum / 2. Check if odd sum (impossible).
```java
boolean canPartition(int[] nums) {
    int sum = Arrays.stream(nums).sum();
    if (sum % 2 != 0) return false;
    return subsetSum(nums, sum / 2);
}
```

### Variation 3: Count Subsets with Sum = Target
> **Change:** dp[w] = count (int), dp[0] = 1, use `dp[w] += dp[w - num]`.
```java
int countSubsets(int[] nums, int target) {
    int[] dp = new int[target + 1];
    dp[0] = 1;
    for (int num : nums) {
        for (int w = target; w >= num; w--) {
            dp[w] += dp[w - num];
        }
    }
    return dp[target];
}
```

### Variation 4: Minimum Subset Sum Difference
> Partition into two subsets minimizing |sum1 - sum2|.
> Run subset sum DP up to totalSum/2. Answer = min(totalSum - 2*dp[i]) for reachable i.
```java
int minSubsetDiff(int[] nums) {
    int total = Arrays.stream(nums).sum();
    boolean[] dp = new boolean[total/2 + 1];
    dp[0] = true;
    for (int num : nums) {
        for (int w = total/2; w >= num; w--) dp[w] = dp[w] || dp[w - num];
    }
    for (int w = total/2; w >= 0; w--) {
        if (dp[w]) return total - 2 * w;
    }
    return total;
}
```

### Variation 5: Target Sum with +/- (LC 494)
> Assign + or - to each num, count ways to reach target.
> **Key:** Let S1 = sum of +nums, S2 = sum of -nums. S1 - S2 = target, S1 + S2 = total.
> → S1 = (target + total) / 2. Count subsets with sum = S1.
```java
int findTargetSumWays(int[] nums, int target) {
    int total = Arrays.stream(nums).sum();
    int t = target + total;
    if (t < 0 || t % 2 != 0) return 0;
    return countSubsets(nums, t / 2);
}
```

### Variation 6: Last Stone Weight II (LC 1049)
> Same as minimum subset sum difference.

### 0/1 Knapsack Questions

| Problem | Variation | LC# |
|---------|-----------|-----|
| Partition Equal Subset Sum | Equal partition | 416 |
| Target Sum (+/-) | Count subsets | 494 |
| Last Stone Weight II | Min subset diff | 1049 |
| Ones and Zeroes | 2D knapsack | 474 |
| Partition Array Into Two Arrays (min diff) | Min subset diff | 2035 |

---

## F2: Unbounded Knapsack Family

### Core Difference from 0/1
> Each item can be used **unlimited times**.
> **Change:** Traverse capacity from LEFT TO RIGHT (not right to left).

```java
int unboundedKnapsack(int[] wt, int[] val, int W) {
    int[] dp = new int[W+1];
    for (int i = 0; i < wt.length; i++) {
        for (int w = wt[i]; w <= W; w++) {   // ← LEFT TO RIGHT
            dp[w] = Math.max(dp[w], val[i] + dp[w - wt[i]]);
        }
    }
    return dp[W];
}
```
> The ONLY difference from 0/1 Knapsack is traversal direction.

### Variation 1: Rod Cutting
> Cut a rod of length n to maximize profit. Price array given.
> Same as unbounded knapsack (length = weight, price = value).

### Variation 2: Coin Change — Minimum Coins (LC 322)
```java
int coinChange(int[] coins, int amount) {
    int[] dp = new int[amount + 1];
    Arrays.fill(dp, amount + 1);   // "infinity"
    dp[0] = 0;
    for (int coin : coins) {
        for (int w = coin; w <= amount; w++) {   // LEFT TO RIGHT
            dp[w] = Math.min(dp[w], 1 + dp[w - coin]);
        }
    }
    return dp[amount] > amount ? -1 : dp[amount];
}
```

### Variation 3: Coin Change II — Count Ways (LC 518)
```java
// Order of loops matters!
// Outer = coins, inner = amounts → counts combinations (not permutations)
int change(int amount, int[] coins) {
    int[] dp = new int[amount + 1];
    dp[0] = 1;
    for (int coin : coins) {                   // outer = items
        for (int w = coin; w <= amount; w++) { // inner = capacity, LEFT TO RIGHT
            dp[w] += dp[w - coin];
        }
    }
    return dp[amount];
}
```
> **Combination vs Permutation:**
> - Outer=items, Inner=capacity → Combinations (each combo counted once)
> - Outer=capacity, Inner=items → Permutations (order matters)

### Variation 4: Integer Break / Ribbon Cut
> Break n into integers summing to n, maximize product.

### Unbounded Knapsack Questions

| Problem                     | Variation              | LC# |
| --------------------------- | ---------------------- | --- |
| Coin Change (min coins)     | Unbounded, minimize    | 322 |
| Coin Change II (count ways) | Unbounded, count       | 518 |
| Perfect Squares             | Unbounded knapsack     | 279 |
| Integer Break               | Unbounded, max product | 343 |

---

## F3: LCS (Longest Common Subsequence) Family

### Core Problem
> Given two strings s and t, find the longest subsequence common to both.
> Subsequence: characters in same order but not necessarily contiguous.

### Base DP Table
```java
int lcs(String s, String t) {
    int m = s.length(), n = t.length();
    int[][] dp = new int[m+1][n+1];
    // dp[i][j] = LCS of s[0..i-1] and t[0..j-1]
    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            if (s.charAt(i-1) == t.charAt(j-1))
                dp[i][j] = 1 + dp[i-1][j-1];
            else
                dp[i][j] = Math.max(dp[i-1][j], dp[i][j-1]);
        }
    }
    return dp[m][n];
}
```

### Recurrence Visualization
```
If s[i] == t[j]:  dp[i][j] = 1 + dp[i-1][j-1]    (diagonal)
Else:             dp[i][j] = max(dp[i-1][j],       (skip from s)
                                 dp[i][j-1])         (skip from t)
```

### Variation 1: Longest Common Substring (contiguous)
```java
// Difference: if chars don't match, RESET to 0 (no extending)
int longestCommonSubstring(String s, String t) {
    int m = s.length(), n = t.length(), maxLen = 0;
    int[][] dp = new int[m+1][n+1];
    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            if (s.charAt(i-1) == t.charAt(j-1)) {
                dp[i][j] = 1 + dp[i-1][j-1];
                maxLen = Math.max(maxLen, dp[i][j]);
            } else {
                dp[i][j] = 0;   // ← key difference: reset to 0
            }
        }
    }
    return maxLen;
}
```

### Variation 2: Shortest Common Supersequence (LC 1092)
> Shortest string containing both s and t as subsequences.
> Length = m + n - LCS(s, t).
```java
int shortestSupersequenceLength(String s, String t) {
    return s.length() + t.length() - lcs(s, t);
}
// To reconstruct the actual string: backtrack through DP table
```

### Variation 3: Minimum Insertions/Deletions to Convert s to t (LC 583)
> Deletions from s + Insertions to get t.
> Answer = (m - LCS) + (n - LCS) = m + n - 2 * LCS.

### Variation 4: Edit Distance (LC 72)
> Minimum operations (insert, delete, replace) to convert s to t.
```java
int editDistance(String s, String t) {
    int m = s.length(), n = t.length();
    int[][] dp = new int[m+1][n+1];
    for (int i = 0; i <= m; i++) dp[i][0] = i;   // delete all of s
    for (int j = 0; j <= n; j++) dp[0][j] = j;   // insert all of t

    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            if (s.charAt(i-1) == t.charAt(j-1))
                dp[i][j] = dp[i-1][j-1];
            else
                dp[i][j] = 1 + Math.min(dp[i-1][j-1],    // replace
                                Math.min(dp[i-1][j],       // delete from s
                                         dp[i][j-1]));     // insert into s
        }
    }
    return dp[m][n];
}
```

### Variation 5: Longest Palindromic Subsequence (LC 516)
> LPS(s) = LCS(s, reverse(s)).
```java
int longestPalindromeSubseq(String s) {
    return lcs(s, new StringBuilder(s).reverse().toString());
}
```

### Variation 6: Minimum Insertions to Make String Palindrome (LC 1312)
> Answer = s.length() - LPS(s).

### Variation 7: Wildcard Matching (LC 44)
> Extends LCS idea with `*` matching any sequence.
```java
boolean isMatch(String s, String p) {
    int m = s.length(), n = p.length();
    boolean[][] dp = new boolean[m+1][n+1];
    dp[0][0] = true;
    for (int j = 1; j <= n; j++) dp[0][j] = dp[0][j-1] && p.charAt(j-1) == '*';
    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            if (p.charAt(j-1) == '*')
                dp[i][j] = dp[i-1][j] || dp[i][j-1];  // match one char or skip *
            else if (p.charAt(j-1) == '?' || s.charAt(i-1) == p.charAt(j-1))
                dp[i][j] = dp[i-1][j-1];
        }
    }
    return dp[m][n];
}
```

### LCS Questions

| Problem | Variation | LC# |
|---------|-----------|-----|
| Longest Common Subsequence | Base LCS | 1143 |
| Shortest Common Supersequence | SCS | 1092 |
| Delete Operations for Two Strings | Min deletions | 583 |
| Edit Distance | Insert/Delete/Replace | 72 |
| Longest Palindromic Subsequence | LCS(s, rev(s)) | 516 |
| Minimum Insertions for Palindrome | n - LPS | 1312 |
| Wildcard Matching | LCS + wildcards | 44 |
| Regular Expression Matching | LCS + regex | 10 |

---

## F4: LIS (Longest Increasing Subsequence) Family

### Core Problem
> Find the length of the longest strictly increasing subsequence.

### O(n²) DP
```java
int lis(int[] nums) {
    int n = nums.length;
    int[] dp = new int[n];
    Arrays.fill(dp, 1);   // every element is LIS of length 1 alone
    int maxLen = 1;
    for (int i = 1; i < n; i++) {
        for (int j = 0; j < i; j++) {
            if (nums[j] < nums[i]) {
                dp[i] = Math.max(dp[i], 1 + dp[j]);
            }
        }
        maxLen = Math.max(maxLen, dp[i]);
    }
    return maxLen;
}
```

### O(n log n) — Binary Search Approach (FAANG preferred for large n)
```java
// Maintain a list 'tails' where tails[i] = smallest tail element of all IS of length i+1
int lisLogN(int[] nums) {
    List<Integer> tails = new ArrayList<>();
    for (int num : nums) {
        int pos = Collections.binarySearch(tails, num);
        if (pos < 0) pos = -(pos + 1);     // find insertion point
        if (pos == tails.size()) tails.add(num);
        else tails.set(pos, num);            // replace to keep tails as small as possible
    }
    return tails.size();
}
// Time: O(n log n)
```

### Variation 1: Maximum Sum Increasing Subsequence
> Instead of max length, maximize sum.
> Change: `dp[i] = max(dp[j] + nums[i])` for j < i where nums[j] < nums[i].

### Variation 2: Longest Bitonic Subsequence
> A subsequence that first increases then decreases.
```java
// Compute LIS[i] = LIS ending at i (left to right)
// Compute LDS[i] = LIS starting at i (right to left, i.e., LIS on reversed)
// Answer = max(LIS[i] + LDS[i] - 1) for all i where both > 1
```

### Variation 3: Minimum Deletions to Sort Array
> min deletions = n - LIS(nums).

### Variation 4: Russian Doll Envelopes (LC 354) — 2D LIS, FAANG Hard
```java
// Sort by width ascending, height DESCENDING (for same width)
// Then LIS on heights only
// Why descending height for same width? Prevents using two envelopes with same width.
int maxEnvelopes(int[][] envelopes) {
    Arrays.sort(envelopes, (a, b) -> a[0] != b[0] ? a[0] - b[0] : b[1] - a[1]);
    int[] heights = Arrays.stream(envelopes).mapToInt(e -> e[1]).toArray();
    return lisLogN(heights);
}
```

### Variation 5: Number of LIS (LC 673)
> Count all longest increasing subsequences (not just length).
```java
// dp[i] = length of LIS ending at i
// count[i] = number of LIS ending at i
int findNumberOfLIS(int[] nums) {
    int n = nums.length;
    int[] dp = new int[n], count = new int[n];
    Arrays.fill(dp, 1); Arrays.fill(count, 1);
    int maxLen = 1;
    for (int i = 1; i < n; i++) {
        for (int j = 0; j < i; j++) {
            if (nums[j] < nums[i]) {
                if (dp[j] + 1 > dp[i]) { dp[i] = dp[j] + 1; count[i] = count[j]; }
                else if (dp[j] + 1 == dp[i]) count[i] += count[j];
            }
        }
        maxLen = Math.max(maxLen, dp[i]);
    }
    int result = 0;
    for (int i = 0; i < n; i++) if (dp[i] == maxLen) result += count[i];
    return result;
}
```

### LIS Questions

| Problem | Variation | LC# |
|---------|-----------|-----|
| Longest Increasing Subsequence | Base | 300 |
| Number of Longest Increasing Subsequences | Count | 673 |
| Russian Doll Envelopes | 2D LIS | 354 |
| Largest Divisible Subset | LIS variant | 368 |

---

## F5: MCM / Interval DP Family

### Core Problem (Matrix Chain Multiplication)
> Given matrices A1, A2, ..., An, find the optimal parenthesization to minimize multiplication cost.

### Template — Interval DP
```java
// dp[i][j] = min cost to compute matrices from i to j
// Try all split points k
for (int len = 2; len <= n; len++) {         // length of interval
    for (int i = 0; i <= n - len; i++) {
        int j = i + len - 1;
        dp[i][j] = Integer.MAX_VALUE;
        for (int k = i; k < j; k++) {        // split point
            dp[i][j] = Math.min(dp[i][j],
                dp[i][k] + dp[k+1][j] + cost(i, k, j));
        }
    }
}
```

### Variation 1: Burst Balloons (LC 312) — Classic Hard
```java
// Key trick: think about the LAST balloon to burst (not first)
// Reverse thinking: dp[i][j] = max coins from balloons i to j
int maxCoins(int[] nums) {
    int n = nums.length;
    int[] arr = new int[n + 2];
    arr[0] = arr[n+1] = 1;
    for (int i = 0; i < n; i++) arr[i+1] = nums[i];
    int m = n + 2;
    int[][] dp = new int[m][m];

    for (int len = 2; len < m; len++) {
        for (int i = 0; i < m - len; i++) {
            int j = i + len;
            for (int k = i + 1; k < j; k++) {
                dp[i][j] = Math.max(dp[i][j],
                    dp[i][k] + arr[i] * arr[k] * arr[j] + dp[k][j]);
            }
        }
    }
    return dp[0][m-1];
}
```

### Variation 2: Palindrome Partitioning II (LC 132)
> Min cuts to partition s into all palindromes.
```java
int minCut(String s) {
    int n = s.length();
    boolean[][] isPalin = new boolean[n][n];
    // Precompute palindrome table
    for (int len = 1; len <= n; len++) {
        for (int i = 0; i <= n - len; i++) {
            int j = i + len - 1;
            isPalin[i][j] = s.charAt(i) == s.charAt(j) &&
                            (len <= 2 || isPalin[i+1][j-1]);
        }
    }
    int[] dp = new int[n];
    Arrays.fill(dp, Integer.MAX_VALUE);
    for (int i = 0; i < n; i++) {
        if (isPalin[0][i]) { dp[i] = 0; continue; }
        for (int j = 1; j <= i; j++) {
            if (isPalin[j][i]) dp[i] = Math.min(dp[i], dp[j-1] + 1);
        }
    }
    return dp[n-1];
}
```

### Variation 3: Strange Printer (LC 664)
### Variation 4: Minimum Cost to Merge Stones (LC 1000)

### MCM Questions

| Problem | LC# |
|---------|-----|
| Burst Balloons | 312 |
| Palindrome Partitioning II | 132 |
| Strange Printer | 664 |
| Minimum Cost Tree from Leaf Values | 1130 |

---

## F6: Grid DP

### Variation 1: Unique Paths (LC 62)
```java
int uniquePaths(int m, int n) {
    int[] dp = new int[n];
    Arrays.fill(dp, 1);
    for (int i = 1; i < m; i++) {
        for (int j = 1; j < n; j++) dp[j] += dp[j-1];
    }
    return dp[n-1];
}
```

### Variation 2: Minimum Path Sum (LC 64)
```java
int minPathSum(int[][] grid) {
    int m = grid.length, n = grid[0].length;
    int[] dp = grid[0].clone();
    for (int j = 1; j < n; j++) dp[j] += dp[j-1];   // first row
    for (int i = 1; i < m; i++) {
        dp[0] += grid[i][0];
        for (int j = 1; j < n; j++) dp[j] = grid[i][j] + Math.min(dp[j], dp[j-1]);
    }
    return dp[n-1];
}
```

### Variation 3: Triangle (LC 120) — Top-Down Path
```java
int minimumTotal(List<List<Integer>> triangle) {
    int n = triangle.size();
    int[] dp = new int[n];
    for (int i = 0; i < n; i++) dp[i] = triangle.get(n-1).get(i);
    for (int i = n - 2; i >= 0; i--)
        for (int j = 0; j <= i; j++)
            dp[j] = triangle.get(i).get(j) + Math.min(dp[j], dp[j+1]);
    return dp[0];
}
```

### Variation 4: Maximal Square (LC 221)
```java
// dp[i][j] = side length of max square with (i,j) as bottom-right
int maximalSquare(char[][] matrix) {
    int m = matrix.length, n = matrix[0].length, max = 0;
    int[][] dp = new int[m+1][n+1];
    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            if (matrix[i-1][j-1] == '1') {
                dp[i][j] = 1 + Math.min(dp[i-1][j], Math.min(dp[i][j-1], dp[i-1][j-1]));
                max = Math.max(max, dp[i][j]);
            }
        }
    }
    return max * max;
}
```

### Variation 5: Dungeon Game (LC 174) — DP from bottom-right
```java
// Must reach princess — work backwards from destination
int calculateMinimumHP(int[][] dungeon) {
    int m = dungeon.length, n = dungeon[0].length;
    int[][] dp = new int[m+1][n+1];
    for (int[] row : dp) Arrays.fill(row, Integer.MAX_VALUE);
    dp[m][n-1] = dp[m-1][n] = 1;
    for (int i = m - 1; i >= 0; i--) {
        for (int j = n - 1; j >= 0; j--) {
            int need = Math.min(dp[i+1][j], dp[i][j+1]) - dungeon[i][j];
            dp[i][j] = Math.max(need, 1);
        }
    }
    return dp[0][0];
}
```

---

## F7: State Machine DP — Buy/Sell Stocks

### Framework
> Model the problem as states: "holding stock", "not holding stock", "cooldown".

### Variation 1: Best Time to Buy and Sell Stock (LC 121) — One transaction
```java
int maxProfit(int[] prices) {
    int minPrice = Integer.MAX_VALUE, maxProfit = 0;
    for (int price : prices) {
        minPrice = Math.min(minPrice, price);
        maxProfit = Math.max(maxProfit, price - minPrice);
    }
    return maxProfit;
}
```

### Variation 2: Unlimited Transactions (LC 122)
```java
// Greedy: add all positive differences
int maxProfit(int[] prices) {
    int profit = 0;
    for (int i = 1; i < prices.length; i++)
        profit += Math.max(0, prices[i] - prices[i-1]);
    return profit;
}
```

### Variation 3: With Cooldown (LC 309) — State Machine
```java
// States: held (holding stock), sold (just sold, cooldown), rest (idle)
int maxProfit(int[] prices) {
    int held = Integer.MIN_VALUE, sold = 0, rest = 0;
    for (int price : prices) {
        int prevSold = sold;
        sold = held + price;                      // sell
        held = Math.max(held, rest - price);       // buy (from rest, not sold)
        rest = Math.max(rest, prevSold);           // rest or continue resting
    }
    return Math.max(sold, rest);
}
```

### Variation 4: With Transaction Fee (LC 714)
```java
int maxProfit(int[] prices, int fee) {
    int held = -prices[0], cash = 0;
    for (int price : prices) {
        held = Math.max(held, cash - price);
        cash = Math.max(cash, held + price - fee);
    }
    return cash;
}
```

### Variation 5: At Most K Transactions (LC 188)
```java
int maxProfit(int k, int[] prices) {
    int n = prices.length;
    if (k >= n / 2) { /* unlimited */ return unlimitedProfit(prices); }
    // dp[i][0] = max profit with i transactions, not holding
    // dp[i][1] = max profit with i transactions, holding
    int[][] dp = new int[k+1][2];
    for (int[] d : dp) d[1] = Integer.MIN_VALUE;
    for (int price : prices) {
        for (int i = k; i >= 1; i--) {
            dp[i][0] = Math.max(dp[i][0], dp[i][1] + price);  // sell
            dp[i][1] = Math.max(dp[i][1], dp[i-1][0] - price); // buy
        }
    }
    return dp[k][0];
}
```

---

## F8: Bitmask DP

### When To Use
> n ≤ 20. "Assign tasks to workers", "visit all cities", "cover all elements".

### Template
```java
// mask = bitmask of visited states
int[] dp = new int[1 << n];
dp[0] = initial;
for (int mask = 0; mask < (1 << n); mask++) {
    for (int i = 0; i < n; i++) {
        if ((mask & (1 << i)) == 0) {   // i not in mask
            int newMask = mask | (1 << i);
            dp[newMask] = Math.min(dp[newMask], dp[mask] + cost[i]);
        }
    }
}
```

### Variation 1: Partition to K Equal Subsets (LC 698)
```java
boolean canPartitionKSubsets(int[] nums, int k) {
    int total = Arrays.stream(nums).sum();
    if (total % k != 0) return false;
    int target = total / k;
    int n = nums.length;
    int[] dp = new int[1 << n];  // dp[mask] = sum in current bucket for that state
    Arrays.fill(dp, -1); dp[0] = 0;
    for (int mask = 0; mask < (1 << n); mask++) {
        if (dp[mask] == -1) continue;
        for (int i = 0; i < n; i++) {
            if ((mask & (1 << i)) != 0) continue;
            int next = mask | (1 << i);
            int val = (dp[mask] + nums[i]) % target;
            if (dp[mask] + nums[i] <= target) dp[next] = val;
        }
    }
    return dp[(1 << n) - 1] == 0;
}
```

### Variation 2: Minimum XOR Sum (LC 1879)
> Assign nums2 to nums1, minimize sum of XOR. Classic bitmask DP assignment.

---

## DP Quick Reference — All Patterns

| Pattern | Key Idea | Space Opt |
|---------|---------|-----------|
| 0/1 Knapsack | include/exclude each item | 2D → 1D (right to left) |
| Unbounded Knapsack | unlimited use | 1D (left to right) |
| LCS | match or skip two sequences | 2D → O(min(m,n)) |
| LIS | at each index, check all previous | O(n) with binary search |
| Interval DP (MCM) | try all split points | 2D, fill by length |
| Grid DP | from top-left to bottom-right | 2D → 1D |
| State Machine | model states explicitly | O(1) variables |
| Bitmask DP | mask = set of visited nodes | O(2^n) |

---

## The 5 Questions to Ask for Any DP Problem

```
1. What is dp[i] or dp[i][j] representing?
2. What is the base case?
3. What is the recurrence (how do I compute dp[i] from smaller subproblems)?
4. What is the final answer (dp[n], dp[n][W], max(dp[i])...)?
5. Can I optimize space?
```

---

> Back to [Master Index](../DSA_MASTER_INDEX.md)
