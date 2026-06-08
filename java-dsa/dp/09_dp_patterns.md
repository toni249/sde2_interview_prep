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

**Approach (DP — 0/1 Knapsack):**
- **State:** `dp[i][w]` = max value using a subset of the first `i` items with capacity ≤ `w`.
- **Recurrence:** for item i-1 (0-indexed): `dp[i][w] = dp[i-1][w]` (exclude); if `wt[i-1] ≤ w`, also consider `val[i-1] + dp[i-1][w - wt[i-1]]` (include). The "i-1" in the include branch enforces each item is used at most once.
- **Base case:** `dp[0][*] = 0` (no items → 0 value); `dp[*][0] = 0` (zero capacity → 0 value).
- **Direction:** bottom-up over (i, w).
- **Space optimization:** since `dp[i][*]` only reads `dp[i-1][*]`, collapse to a 1D array of size W+1, iterating capacity **right→left** so a freshly-updated value (which represents "item i included") cannot be reused in the same item's iteration (preserving 0/1).
- **Complexity:** O(n·W) time, O(W) space (after optimization).

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

**Problem:** Given an array of non-negative integers `nums` and a target sum, determine whether any subset of `nums` sums to exactly `target`.

**Approach (DP — 0/1 Knapsack boolean variant):**
- **State:** `dp[w]` = true if some subset of the items processed so far sums to exactly `w`.
- **Recurrence:** `dp[w] = dp[w] OR dp[w - num]` — either the current num is excluded (old value) or included (a smaller sum + num).
- **Base case:** `dp[0] = true` (empty subset sums to 0).
- **Direction:** bottom-up. Iterate items outer, capacity inner from RIGHT→LEFT to ensure each item used at most once (0/1 constraint).
- **Complexity:** O(n × target) time, O(target) space.

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

**Problem:** Given a non-empty array `nums`, decide whether it can be partitioned into two subsets whose sums are equal.

**Approach (DP — reduce to Subset Sum):**
- If `totalSum` is odd, partition impossible → return false.
- Otherwise the question becomes: does a subset sum to `totalSum / 2`? If yes, the complement also sums to `totalSum / 2`.
- **State / Recurrence / Base:** identical to Subset Sum.
- **Complexity:** O(n × sum/2) time, O(sum/2) space.

```java
boolean canPartition(int[] nums) {
    int sum = Arrays.stream(nums).sum();
    if (sum % 2 != 0) return false;
    return subsetSum(nums, sum / 2);
}
```

### Variation 3: Count Subsets with Sum = Target

**Problem:** Given `nums` of non-negative integers, count the number of distinct subsets whose elements sum to `target`.

**Approach (DP — 0/1 Knapsack counting variant):**
- **State:** `dp[w]` = number of subsets (using items processed so far) summing to `w`.
- **Recurrence:** `dp[w] += dp[w - num]` — every subset summing to `w - num` extended by the current `num` is a new subset summing to `w`.
- **Base case:** `dp[0] = 1` (exactly one empty subset sums to 0).
- **Direction:** bottom-up, items outer, capacity inner RIGHT→LEFT (each item used at most once).
- **Complexity:** O(n × target) time, O(target) space.

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

**Problem:** Partition `nums` into two subsets S1 and S2 (every element in exactly one) so as to minimize `|sum(S1) - sum(S2)|`. Return that minimum.

**Approach (DP — Subset Sum reachability):**
- Let total = sum(nums). If S1 has sum w, then S2 has sum `total - w`, and the difference is `total - 2w`. To minimize, we want w as close to total/2 as possible (but ≤ total/2).
- **State:** `dp[w]` = true if some subset sums to `w` (run subset-sum DP only up to total/2).
- **Recurrence/Base:** same as Subset Sum.
- After DP, scan w from total/2 downwards and return `total - 2*w` for the first reachable w.
- **Complexity:** O(n × total) time, O(total) space.

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

**Problem:** Given an integer array `nums` and an integer `target`, assign either `+` or `-` to each number. Count the number of distinct sign assignments whose total equals `target`.

**Approach (DP — reduce to Count Subsets):**
- Let P = sum of `+` numbers, N = sum of `-` numbers. We need `P − N = target` and `P + N = total`.
- Adding the two: `2P = target + total` → `P = (target + total) / 2`. So count subsets with sum exactly P.
- Guards: if `(target + total)` is negative or odd, there are 0 ways.
- **State / Recurrence / Base:** same as "Count Subsets with Sum = Target".
- **Complexity:** O(n × P) time, O(P) space.

```java
int findTargetSumWays(int[] nums, int target) {
    int total = Arrays.stream(nums).sum();
    int t = target + total;
    if (t < 0 || t % 2 != 0) return 0;
    return countSubsets(nums, t / 2);
}
```

### Variation 6: Last Stone Weight II (LC 1049)

**Problem:** Given stones with weights, repeatedly smash any two; if equal both vanish, else the heavier becomes `|a-b|`. Return the smallest possible weight of the remaining stone (0 if all destroyed).

**Approach (DP — reduces exactly to Min Subset Sum Difference):**
- After all smashes, every stone has been signed `+` or `-`, and the final remaining value equals `|sum(plus) − sum(minus)|`. So we want to split stones into two groups minimizing absolute difference.
- Identical DP to "Minimum Subset Sum Difference".
- **Complexity:** O(n × totalSum) time, O(totalSum) space.

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

**Approach (DP — Unbounded Knapsack):**
- **State:** `dp[w]` = max value achievable using any number of copies of items, with capacity ≤ `w`.
- **Recurrence:** `dp[w] = max(dp[w], val[i] + dp[w - wt[i]])` — including item i lets us still use item i again, so we read `dp[w - wt[i]]` from the *current* row (already updated in this same item's pass).
- **Base case:** `dp[0] = 0`.
- **Direction:** for each item, iterate capacity LEFT→RIGHT (the exact opposite of 0/1) so updates flow forward and allow reuse.
- **Complexity:** O(n·W) time, O(W) space.

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

**Problem:** Given a rod of length `n` and a `price[]` where `price[i]` is the price of a piece of length `i+1`, cut the rod into pieces (zero or more cuts) so as to maximize total revenue.

**Approach (DP — Unbounded Knapsack):**
- Pieces of each length can be used unlimited times → unbounded knapsack mapping: length = weight, price = value, capacity = n.
- **State:** `dp[w]` = max revenue achievable from a rod of length `w`.
- **Recurrence:** `dp[w] = max(dp[w], price[i] + dp[w - len[i]])` for every length `i`.
- **Base case:** `dp[0] = 0`.
- **Direction:** bottom-up, capacity inner LEFT→RIGHT (allows reuse of a piece length).
- **Complexity:** O(n²) time, O(n) space.

### Variation 2: Coin Change — Minimum Coins (LC 322)

**Problem:** Given coin denominations `coins[]` (unlimited supply of each) and a target `amount`, return the minimum number of coins needed to make `amount`, or -1 if impossible.

**Approach (DP — Unbounded Knapsack, minimize count):**
- **State:** `dp[w]` = minimum coins needed to make amount `w` (∞ if unreachable).
- **Recurrence:** `dp[w] = min(dp[w], 1 + dp[w - coin])` — use one more coin of denomination `coin` on top of the best way to make `w - coin`.
- **Base case:** `dp[0] = 0` (zero coins make amount 0). Initialize rest to a sentinel "infinity" (`amount + 1` works since we'll never need more than `amount` 1-rupee coins).
- **Direction:** bottom-up, capacity inner LEFT→RIGHT (unbounded — each coin is reusable).
- **Complexity:** O(amount × coins.length) time, O(amount) space.

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

**Problem:** Given coin denominations (unlimited supply) and an `amount`, return the number of distinct *combinations* (sets — order doesn't matter) of coins that sum to `amount`.

**Approach (DP — Unbounded Knapsack counting):**
- **State:** `dp[w]` = number of combinations summing to `w` using coins considered so far.
- **Recurrence:** `dp[w] += dp[w - coin]` — every way to make `w - coin` extended by one more `coin` gives a way to make `w`.
- **Base case:** `dp[0] = 1` (the empty combo).
- **Direction:** outer loop = coins, inner = amount LEFT→RIGHT. This ordering is CRITICAL — fixing a coin before iterating amounts prevents counting the same multiset twice (otherwise `{1,2}` and `{2,1}` are counted separately → permutations, not combinations).
- **Complexity:** O(amount × coins.length) time, O(amount) space.

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

### Variation 4: Integer Break / Ribbon Cut (LC 343)

**Problem:** Given integer `n ≥ 2`, break it into the sum of at least two positive integers and maximize the product of those integers. Return that maximum product.

**Approach (DP — Unbounded "items" are integers 1..n-1):**
- **State:** `dp[i]` = max product obtainable from breaking `i` into ≥ 1 part (we then enforce ≥ 2 parts at the top level).
- **Recurrence:** `dp[i] = max over j in [1..i-1] of max(j, dp[j]) * max(i - j, dp[i - j])` — for each split point j, take either j as-is or its best breakdown, and similarly for `i - j`.
- **Base case:** `dp[1] = 1`.
- **Direction:** bottom-up; fill from 2 up to n.
- **Complexity:** O(n²) time, O(n) space. (A pure-math greedy "use as many 3s as possible" gives O(1).)

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

### Base LCS (LC 1143)

**Problem:** Given two strings `s` and `t`, return the length of their longest common subsequence (characters appear in the same relative order in both strings; not necessarily contiguous).

**Approach (DP — LCS):**
- **State:** `dp[i][j]` = length of LCS of prefix `s[0..i-1]` and `t[0..j-1]`.
- **Recurrence:**
  - If `s[i-1] == t[j-1]`: `dp[i][j] = 1 + dp[i-1][j-1]` (take the matched char).
  - Else: `dp[i][j] = max(dp[i-1][j], dp[i][j-1])` (skip one char from either string and take the better option).
- **Base case:** `dp[0][*] = dp[*][0] = 0` (empty string has LCS 0 with anything).
- **Direction:** bottom-up, fill row by row.
- **Complexity:** O(m·n) time, O(m·n) space → can be optimized to O(min(m,n)) using two rolling rows.

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

**Problem:** Given two strings `s` and `t`, return the length of the longest contiguous substring that appears in both.

**Approach (DP — LCS variant with reset):**
- **State:** `dp[i][j]` = length of the longest common substring ending exactly at `s[i-1]` and `t[j-1]` (must include those two characters).
- **Recurrence:**
  - If `s[i-1] == t[j-1]`: `dp[i][j] = 1 + dp[i-1][j-1]`.
  - Else: `dp[i][j] = 0` — the key difference vs LCS: a mismatch breaks contiguity, so the run ends.
- **Base case:** `dp[0][*] = dp[*][0] = 0`.
- **Answer:** `max(dp[i][j])` over all cells (not `dp[m][n]`).
- **Complexity:** O(m·n) time, O(m·n) space (or O(min(m,n)) with rolling rows).

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

**Problem:** Given strings `s` and `t`, return the length (or actual string) of the shortest string that contains BOTH `s` and `t` as subsequences.

**Approach (DP — LCS reduction):**
- Every character of the LCS is shared by both strings, so it should appear once in the supersequence. Non-LCS characters from each string must appear separately.
- **Formula:** length = `m + n − LCS(s, t)`.
- To reconstruct the actual string, walk the LCS DP table backwards: when chars match, take that char; otherwise step into the larger neighbor and emit the skipped char.
- **Complexity:** O(m·n) time, O(m·n) space.

```java
int shortestSupersequenceLength(String s, String t) {
    return s.length() + t.length() - lcs(s, t);
}
// To reconstruct the actual string: backtrack through DP table
```

### Variation 3: Minimum Insertions/Deletions to Convert s to t (LC 583)

**Problem:** Given strings `s` and `t`, return the minimum number of insertions + deletions required to convert `s` into `t` (only insert or delete allowed — no replace).

**Approach (DP — LCS reduction):**
- The LCS represents the characters we keep. Everything in `s` outside the LCS must be **deleted**, and everything in `t` outside the LCS must be **inserted**.
- **Formula:** answer = `(m − LCS) + (n − LCS) = m + n − 2·LCS`.
- **Complexity:** O(m·n) time, O(m·n) space.

### Variation 4: Edit Distance (LC 72)

**Problem:** Given two strings `s` and `t`, return the minimum number of operations (insert one char, delete one char, replace one char) required to transform `s` into `t`. (Levenshtein distance.)

**Approach (DP — 2D string DP):**
- **State:** `dp[i][j]` = minimum operations to convert `s[0..i-1]` to `t[0..j-1]`.
- **Recurrence:**
  - If `s[i-1] == t[j-1]`: `dp[i][j] = dp[i-1][j-1]` (free; chars already match).
  - Else: `dp[i][j] = 1 + min(dp[i-1][j-1], dp[i-1][j], dp[i][j-1])` — replace, delete from s, or insert into s respectively.
- **Base case:** `dp[i][0] = i` (delete all i chars of s), `dp[0][j] = j` (insert all j chars of t).
- **Direction:** bottom-up, fill row-by-row.
- **Complexity:** O(m·n) time, O(m·n) space → can be reduced to O(n) using two rolling rows.

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

**Problem:** Given a string `s`, return the length of the longest subsequence of `s` that is a palindrome.

**Approach (DP — LCS trick):**
- A palindromic subsequence reads the same forwards and backwards, so it appears identically in `s` and `reverse(s)`. Therefore `LPS(s) = LCS(s, reverse(s))`.
- Alternative direct formulation: `dp[i][j]` = LPS of `s[i..j]`; `dp[i][i] = 1`; if `s[i] == s[j]` then `dp[i][j] = 2 + dp[i+1][j-1]`; else `dp[i][j] = max(dp[i+1][j], dp[i][j-1])`. Fill by increasing interval length.
- **Complexity:** O(n²) time, O(n²) space.

```java
int longestPalindromeSubseq(String s) {
    return lcs(s, new StringBuilder(s).reverse().toString());
}
```

### Variation 6: Minimum Insertions to Make String Palindrome (LC 1312)

**Problem:** Given a string `s`, return the minimum number of character insertions (anywhere) required to make `s` a palindrome.

**Approach (DP — LPS reduction):**
- Every character outside the longest palindromic subsequence must be "mirrored" by an insertion to balance the palindrome.
- **Formula:** answer = `s.length() − LPS(s)`.
- **Complexity:** O(n²) time, O(n²) space (driven by the underlying LPS DP).

### Variation 7: Wildcard Matching (LC 44)

**Problem:** Given a string `s` and a pattern `p` containing lowercase letters, `?` (matches any single character), and `*` (matches any sequence including empty), determine whether `p` matches the **entire** `s`.

**Approach (DP — 2D string DP):**
- **State:** `dp[i][j]` = true if `s[0..i-1]` matches `p[0..j-1]`.
- **Recurrence:**
  - If `p[j-1] == '*'`: `dp[i][j] = dp[i-1][j]` (star consumes one more char from s) `OR dp[i][j-1]` (star matches empty).
  - Else if `p[j-1] == '?'` or `s[i-1] == p[j-1]`: `dp[i][j] = dp[i-1][j-1]`.
  - Else: false.
- **Base case:** `dp[0][0] = true`. `dp[0][j] = true` only while `p[j-1] == '*'` (a prefix of all stars matches empty s).
- **Direction:** bottom-up, row-by-row.
- **Complexity:** O(m·n) time, O(m·n) space.

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

### Base LIS (LC 300)

**Problem:** Given an integer array `nums`, return the length of the longest strictly increasing subsequence.

**Approach (DP — O(n²) classic):**
- **State:** `dp[i]` = length of the longest increasing subsequence that **ends at** index `i` (must include `nums[i]`).
- **Recurrence:** `dp[i] = 1 + max(dp[j])` over all `j < i` with `nums[j] < nums[i]`. If no such j, dp[i] stays 1.
- **Base case:** every `dp[i]` initialized to 1 (the element by itself).
- **Answer:** `max(dp[i])` over all i.
- **Direction:** bottom-up, left to right.
- **Complexity:** O(n²) time, O(n) space. See next section for O(n log n).

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

### LIS O(n log n) — Patience Sorting

**Problem:** Same as Base LIS but with large `n` (~10⁵) where O(n²) is too slow.

**Approach (Greedy + Binary Search — "patience sorting"):**
- Maintain a list `tails` where `tails[k]` = the smallest possible tail value of any increasing subsequence of length `k+1` seen so far.
- For each `num`: binary-search the first index in `tails` that is ≥ num. If found, replace it (we found a smaller tail for that length); else append (we extended the longest subsequence by one).
- **Key invariant:** `tails` is sorted at all times, even though it is **not itself an LIS** of `nums` (it's just bookkeeping of best tails).
- **Answer:** `tails.size()`.
- **Complexity:** O(n log n) time, O(n) space.

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

**Problem:** Given `nums`, find an increasing subsequence with the maximum possible sum (not length).

**Approach (DP — LIS variant):**
- **State:** `dp[i]` = max sum of an increasing subsequence **ending at** index `i`.
- **Recurrence:** `dp[i] = nums[i] + max(dp[j])` over all `j < i` with `nums[j] < nums[i]`. If no valid j, `dp[i] = nums[i]`.
- **Base case:** `dp[i] = nums[i]` initially (the element alone).
- **Answer:** `max(dp[i])`.
- **Complexity:** O(n²) time, O(n) space.

### Variation 2: Longest Bitonic Subsequence

**Problem:** Given `nums`, find the length of the longest subsequence that first strictly increases and then strictly decreases (a "mountain"). The peak must be a single element.

**Approach (DP — two LIS passes):**
- **State:** `LIS[i]` = length of LIS ending at i (left→right). `LDS[i]` = length of longest decreasing subseq starting at i (equivalently, LIS ending at i on reversed array).
- **Recurrence:** standard LIS recurrence on each direction.
- **Answer:** `max(LIS[i] + LDS[i] − 1)` over all i (subtract 1 because `nums[i]` is counted twice as the peak). Require both arms ≥ 2 if strict bitonic is required.
- **Complexity:** O(n²) time, O(n) space.

```java
// Compute LIS[i] = LIS ending at i (left to right)
// Compute LDS[i] = LIS starting at i (right to left, i.e., LIS on reversed)
// Answer = max(LIS[i] + LDS[i] - 1) for all i where both > 1
```

### Variation 3: Minimum Deletions to Sort Array

**Problem:** Given `nums`, return the minimum number of deletions so the remaining elements are sorted in non-decreasing order.

**Approach (DP — LIS reduction):**
- After deletions, the remaining elements are a sorted subsequence. The largest such subsequence is the (non-strict) LIS.
- **Formula:** answer = `n − LIS(nums)`. Use `nums[j] ≤ nums[i]` if non-decreasing allowed, `<` if strictly increasing required.
- **Complexity:** O(n log n) using patience sorting.

### Variation 4: Russian Doll Envelopes (LC 354) — 2D LIS, FAANG Hard

**Problem:** Given a list of envelopes `[w, h]`, you can nest envelope A inside B only if `A.w < B.w` AND `A.h < B.h` (strict). Return the maximum number that can be nested.

**Approach (Sort + LIS on heights):**
- Sort by width ascending. Now the question reduces to "find LIS on heights" — but with a twist: envelopes with **equal widths** must not both be selected (strict `<` rule).
- **Trick:** for ties in width, sort heights **descending**. Then within a tie group, the LIS algorithm on heights can pick at most one (because heights are decreasing, no two can both be increasing).
- Run O(n log n) LIS on the resulting heights array.
- **Complexity:** O(n log n) time, O(n) space.

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

**Problem:** Given `nums`, return the number of distinct longest increasing subsequences (not just their length).

**Approach (DP — LIS with companion count array):**
- **State:** `dp[i]` = length of LIS ending at i. `count[i]` = number of LIS ending at i with that length.
- **Recurrence:** for each j < i with `nums[j] < nums[i]`:
  - If `dp[j] + 1 > dp[i]`: we found a strictly longer LIS — set `dp[i] = dp[j] + 1`, `count[i] = count[j]` (inherit ways).
  - Else if `dp[j] + 1 == dp[i]`: another way to achieve the same length — `count[i] += count[j]`.
- **Base case:** `dp[i] = 1`, `count[i] = 1`.
- **Answer:** sum of `count[i]` for all i with `dp[i] == maxLen`.
- **Complexity:** O(n²) time, O(n) space.

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

**Approach (Interval DP — MCM template):**
- **State:** `dp[i][j]` = minimum scalar multiplications needed to fully parenthesize matrices i..j.
- **Recurrence:** `dp[i][j] = min over split k in [i, j) of (dp[i][k] + dp[k+1][j] + cost(i, k, j))` — try every place to "make the last cut", solve the two halves, add the cost of combining their results.
- **Base case:** `dp[i][i] = 0` (one matrix, no multiplication).
- **Direction:** fill by **increasing interval length** (not by i/j), so that both halves are already computed when we need them.
- **Complexity:** O(n³) time, O(n²) space.

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

**Problem:** Given `nums` representing balloon coin values, bursting balloon `i` earns `nums[left] * nums[i] * nums[right]` where `left/right` are the **currently adjacent** unburst balloons (or 1 at the boundary). Find the max coins from bursting all balloons.

**Approach (Interval DP with "last burst" trick):**
- Forward DP fails: when you burst i, its neighbors change → state is too complex.
- **Key insight — think backwards:** in an optimal interval `(i..j)`, ask "which balloon `k` is bursted LAST in this interval?" When k is last, balloons at index `i` and `j` (the boundary balloons just OUTSIDE the interval) are still alive → contribution is exactly `nums[i] * nums[k] * nums[j]`. Now the left part `(i..k)` and right part `(k..j)` become independent.
- **Setup:** pad nums with 1 on both ends → `arr` of size `n+2`. Use open interval indices.
- **State:** `dp[i][j]` = max coins from bursting all balloons strictly between i and j (exclusive endpoints).
- **Recurrence:** `dp[i][j] = max over k in (i, j) of dp[i][k] + arr[i]*arr[k]*arr[j] + dp[k][j]`.
- **Base case:** `dp[i][i+1] = 0` (no balloon in between).
- **Direction:** fill by increasing interval length.
- **Complexity:** O(n³) time, O(n²) space.

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

**Problem:** Given a string `s`, partition it so every substring is a palindrome. Return the **minimum number of cuts** required.

**Approach (DP — 1D with palindrome precomputation):**
- **Pre-step:** build `isPalin[i][j]` table in O(n²): `isPalin[i][j] = (s[i] == s[j]) && (j - i ≤ 1 || isPalin[i+1][j-1])`. Fill by increasing length.
- **State:** `dp[i]` = min cuts needed to partition `s[0..i]` so every piece is a palindrome.
- **Recurrence:**
  - If `isPalin[0][i]`: `dp[i] = 0` (no cut needed).
  - Else: `dp[i] = min over j in [1..i] of (dp[j-1] + 1)` for every j with `isPalin[j][i]`.
- **Base case:** `dp[0] = 0` (single character is already a palindrome).
- **Direction:** bottom-up over i.
- **Complexity:** O(n²) time, O(n²) space (dominated by palindrome table). Pure MCM-style O(n³) also works but is slower.

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

**Problem:** A printer can only print a sequence of the same character at a time, and a new print can cover existing characters. Given a target string `s`, return the minimum number of "print runs" required.

**Approach (Interval DP):**
- **State:** `dp[i][j]` = minimum turns to print `s[i..j]`.
- **Recurrence:** start with `dp[i][j] = dp[i+1][j] + 1` (print `s[i]` independently, then the rest). Then for every k in `(i, j]` where `s[k] == s[i]`, we can "extend" the run that paints `s[i]` to also cover position k for free: `dp[i][j] = min(dp[i][j], dp[i+1][k-1] + dp[k][j])`.
- **Base case:** `dp[i][i] = 1`.
- **Direction:** fill by increasing interval length.
- **Complexity:** O(n³) time, O(n²) space.

### Variation 4: Minimum Cost to Merge Stones (LC 1000)

**Problem:** Given `n` piles in a row with weights, you may merge **exactly K consecutive** piles in one move, paying their total weight as cost. Return the min total cost to merge all piles into one, or -1 if impossible.

**Approach (Interval DP with feasibility check):**
- Feasibility: each merge reduces pile count by `K-1`, so we need `(n - 1) % (K - 1) == 0`, else return -1.
- **State:** `dp[i][j]` = min cost to merge stones[i..j] into the minimum possible piles (`((j-i) % (K-1)) + 1` piles).
- **Recurrence:** `dp[i][j] = min over t in [i..j-1] step (K-1) of (dp[i][t] + dp[t+1][j])`; if `(j-i) % (K-1) == 0`, also add `prefixSum(i..j)` (the final merge cost).
- **Base case:** `dp[i][i] = 0`.
- **Direction:** fill by increasing interval length.
- **Complexity:** O(n³ / K) time, O(n²) space.

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

**Problem:** Given an `m × n` grid, a robot starts at `(0,0)` and can move only right or down. Return the number of distinct paths to `(m-1, n-1)`.

**Approach (DP — Grid):**
- **State:** `dp[i][j]` = number of paths from `(0,0)` to `(i, j)`.
- **Recurrence:** `dp[i][j] = dp[i-1][j] + dp[i][j-1]` (arrive from above or from left).
- **Base case:** entire first row and first column = 1 (only one path along the edge).
- **Direction:** bottom-up, row by row.
- **Space optimization:** since we only need the previous row, use a single 1D array of size n: `dp[j] += dp[j-1]`.
- **Complexity:** O(m·n) time, O(n) space. Closed-form alternative: `C(m+n-2, m-1)`.

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

**Problem:** Given a grid filled with non-negative integers, find a path from top-left to bottom-right that minimizes the sum of numbers along the path. Move only right or down.

**Approach (DP — Grid):**
- **State:** `dp[i][j]` = minimum path sum to reach `(i, j)`.
- **Recurrence:** `dp[i][j] = grid[i][j] + min(dp[i-1][j], dp[i][j-1])`.
- **Base case:** `dp[0][0] = grid[0][0]`; first row and first column are prefix sums along the edge (only one direction available).
- **Space optimization:** 1D array of size n; process row by row, where `dp[j] = grid[i][j] + min(dp[j], dp[j-1])`.
- **Complexity:** O(m·n) time, O(n) space.

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

**Problem:** Given a triangle of numbers, find the minimum path sum from top to bottom. At each step you may move to one of the two adjacent indices in the row below.

**Approach (DP — bottom-up rows):**
- **State:** `dp[j]` = min path sum from triangle cell `(current_row, j)` down to the bottom.
- **Recurrence:** `dp[j] = triangle[i][j] + min(dp[j], dp[j+1])` (move down-left or down-right).
- **Base case:** initialize `dp` to the last row of the triangle.
- **Direction:** bottom-up — start at the last row and roll upward; this avoids handling boundary conditions for "two parents" and reuses one 1D array.
- **Answer:** `dp[0]`.
- **Complexity:** O(n²) time, O(n) space (n = number of rows).

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

**Problem:** Given an `m × n` binary matrix of `'0'` and `'1'`, find the largest square containing only `'1'`s and return its area.

**Approach (DP — Grid):**
- **State:** `dp[i][j]` = the side length of the largest all-1s square whose bottom-right corner is `(i-1, j-1)` (1-indexed for convenience).
- **Recurrence:** if `matrix[i-1][j-1] == '1'`: `dp[i][j] = 1 + min(dp[i-1][j], dp[i][j-1], dp[i-1][j-1])`. The min ensures the three overlapping squares (top, left, diagonal) are all valid; the smallest restricts how far this corner can grow. If the cell is `'0'`, `dp[i][j] = 0`.
- **Base case:** outer row/col of dp (index 0) all zero.
- **Answer:** `max(dp[i][j])²` (max side length squared).
- **Space optimization:** can be reduced to O(n) by tracking the previous diagonal value in a single variable.
- **Complexity:** O(m·n) time, O(m·n) space.

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

**Problem:** A knight starts at top-left of a dungeon grid (each cell adds or subtracts health), moves only right/down, and must reach the princess at bottom-right with HP ≥ 1 at every step. Return the minimum starting HP.

**Approach (DP — reverse direction):**
- Forward DP fails: from a cell, the future minimum required HP depends on the path you'll take — you can't decide locally what the best move is using only past info, because future damage can dominate.
- **Reverse trick:** `dp[i][j]` = the minimum HP the knight must have **when entering** cell `(i, j)` to survive the rest of the journey to the bottom-right.
- **Recurrence:** `need = min(dp[i+1][j], dp[i][j+1]) − dungeon[i][j]`. Then `dp[i][j] = max(need, 1)` (must always remain ≥ 1).
- **Base case:** beyond the grid, set sentinels to `INF`; `dp[m][n-1] = dp[m-1][n] = 1` so the knight needs exactly 1 HP arriving on the princess cell.
- **Answer:** `dp[0][0]`.
- **Complexity:** O(m·n) time, O(m·n) space (reducible to O(n)).

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

**Problem:** Given daily stock `prices[]`, you may complete **at most one** buy-then-sell transaction. Return the maximum profit, or 0 if no profit possible.

**Approach (DP collapsed to two variables):**
- **State (intuition):** at day i, track the minimum price seen so far; the best sell on day i is `prices[i] − minSoFar`.
- **Recurrence:** `minPrice = min(minPrice, prices[i])`; `maxProfit = max(maxProfit, prices[i] − minPrice)`.
- **Base case:** `minPrice = ∞`, `maxProfit = 0`.
- Equivalently: state-machine with 2 states (`held`, `sold`) and only 1 buy allowed.
- **Complexity:** O(n) time, O(1) space.

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

**Problem:** Given daily `prices[]`, you may complete as many buy-then-sell transactions as you like (but you can hold at most one share at a time). Maximize total profit.

**Approach (Greedy — equivalent to state-machine DP):**
- Any upward sequence `p[i-1] < p[i] < p[i+1]` can be broken into consecutive day-to-day positive gains without losing profit.
- **Greedy rule:** add `prices[i] − prices[i-1]` whenever it's positive.
- The equivalent DP: `held = max(held, cash − price)`, `cash = max(cash, held + price)`. Both formulations are O(n).
- **Complexity:** O(n) time, O(1) space.

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

**Problem:** Given daily `prices[]`, unlimited transactions are allowed but **after selling you must rest one day** before buying again. Maximize profit.

**Approach (DP — explicit 3-state machine):**
- **States** for each day:
  - `held` = max profit if currently holding a stock at end of day.
  - `sold` = max profit if we sold TODAY (so tomorrow is cooldown).
  - `rest` = max profit if we are not holding and not just-sold (idle/cooldown done).
- **Transitions:**
  - `sold = held + price` (must have been held to sell).
  - `held = max(held, rest − price)` (continue holding, or buy from rest — NOT from sold, that's the cooldown enforcement).
  - `rest = max(rest, prevSold)` (continue resting, or finish cooldown from yesterday's sell).
- **Base case:** `held = −∞` (impossible to be holding before day 0), `sold = rest = 0`.
- **Answer:** `max(sold, rest)` — never beneficial to end while holding.
- **Complexity:** O(n) time, O(1) space.

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

**Problem:** Given daily `prices[]` and a flat transaction `fee`, unlimited buy/sell transactions allowed but each sale incurs `fee`. Maximize net profit.

**Approach (DP — 2-state machine):**
- **States:** `held` = max profit if holding a stock; `cash` = max profit if not holding.
- **Transitions:**
  - `held = max(held, cash − price)` (continue holding, or buy today).
  - `cash = max(cash, held + price − fee)` (continue not holding, or sell today and pay fee).
- **Base case:** `held = −prices[0]`, `cash = 0`.
- **Answer:** `cash` after the last day (no reason to end while holding).
- **Complexity:** O(n) time, O(1) space.

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

**Problem:** Given daily `prices[]` and an integer `k`, you may complete at most `k` buy-then-sell transactions (one share at a time). Maximize profit.

**Approach (DP — state machine with transaction counter):**
- If `k ≥ n/2`, no constraint is binding → reduce to "unlimited transactions" (LC 122).
- **State:** `dp[i][0]` = max profit using ≤ i transactions, currently NOT holding. `dp[i][1]` = max profit using ≤ i transactions, currently holding.
- **Recurrence (per price):**
  - `dp[i][0] = max(dp[i][0], dp[i][1] + price)` — sell (does NOT increment i; we count a transaction at buy time).
  - `dp[i][1] = max(dp[i][1], dp[i-1][0] − price)` — buy (uses one of the i transactions, hence `dp[i-1][0]`).
  - Iterate i from k down to 1 so that updates to lower-i states don't bleed into higher-i state in the same day.
- **Base case:** `dp[*][0] = 0`, `dp[*][1] = −∞`.
- **Answer:** `dp[k][0]`.
- **Complexity:** O(n·k) time, O(k) space.

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

**Problem:** Given `nums` and integer `k`, decide if `nums` can be partitioned into `k` non-empty subsets each with equal sum.

**Approach (Bitmask DP — n ≤ ~16):**
- Required target per bucket = `total / k`. If `total % k != 0`, impossible.
- **State:** `dp[mask]` = the current partial sum **modulo target** in the bucket currently being filled, given that `mask` is the set of already-used elements. Sentinel `-1` means mask is unreachable.
- **Recurrence:** from a valid mask, for every index i not in mask: let `next = mask | (1 << i)`. If `dp[mask] + nums[i] ≤ target`, set `dp[next] = (dp[mask] + nums[i]) % target`. The mod cleanly "rolls over" to the next bucket when we hit `target`.
- **Base case:** `dp[0] = 0`.
- **Answer:** `dp[(1 << n) − 1] == 0` (all used and current bucket cleanly closed).
- **Complexity:** O(2ⁿ · n) time, O(2ⁿ) space.

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

**Problem:** Given two arrays of equal length `nums1` and `nums2`, rearrange `nums2` such that `sum(nums1[i] XOR nums2[i])` over all i is minimized. Return that minimum.

**Approach (Bitmask DP — assignment problem):**
- This is the classic "match n items to n slots optimally". n ≤ 14 so 2ⁿ ≤ 16384 — fits.
- **State:** `dp[mask]` = min XOR-sum after we have processed the first `popcount(mask)` elements of `nums1` and used exactly the bits in `mask` as indices into `nums2`.
- **Recurrence:** let `i = popcount(mask) − 1` (the nums1 index we just paired). Then `dp[mask] = min over j in bits(mask) of dp[mask ^ (1 << j)] + (nums1[i] XOR nums2[j])`.
- **Base case:** `dp[0] = 0`.
- **Answer:** `dp[(1 << n) − 1]`.
- **Complexity:** O(2ⁿ · n) time, O(2ⁿ) space.

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
