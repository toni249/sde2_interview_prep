# Difference Array & Prefix Sum — DSA Notes

> Pattern for "add to many ranges, query later" problems.
> Example problem: **Minimum Moves to Make Array Complementary** (LeetCode 1674)

---

## 1. Prefix Sum — The Warm-Up

### The Idea

Given an array, precompute running totals so any **range sum query** becomes one subtraction.

```
A      = [3, 1, 4, 1, 5, 9]
           0  1  2  3  4  5

prefix = [3, 4, 8, 9, 14, 23]
```

Each `prefix[i] = A[0] + A[1] + ... + A[i]`.

Sum of range `[L, R]` = `prefix[R] - prefix[L-1]` (or just `prefix[R]` if L=0).

### Java Code

```java
public class PrefixSum {
    private int[] prefix;

    public PrefixSum(int[] A) {
        prefix = new int[A.length];
        prefix[0] = A[0];
        for (int i = 1; i < A.length; i++) {
            prefix[i] = prefix[i - 1] + A[i];
        }
    }

    public int rangeSum(int L, int R) {
        if (L == 0) return prefix[R];
        return prefix[R] - prefix[L - 1];
    }
}
```

**Use case:** Static array, many range sum queries. O(n) preprocessing, O(1) per query.

---

## 2. Difference Array — The Reverse Trick

### The Problem It Solves

You have an array of zeros. You receive many instructions like:
- "Add 5 to range [2, 5]"
- "Add 3 to range [1, 4]"
- "Add 2 to range [3, 7]"

**Naive:** Loop over each range and add. O(updates × range_size). Slow.

**Smart:** Record each instruction in O(1), then compute final array in one pass.

### The "Light Switch" Intuition

Walk left to right with a counter starting at 0.

To add `v` to range `[L, R]`:
- At index `L`: **turn ON** `+v` (counter goes up by v)
- At index `R+1`: **turn OFF** `+v` (counter goes down by v)

```
Adding +5 to range [2, 5]:

index:    0   1   2   3   4   5   6   7
action:           +5              -5
counter:  0   0   5   5   5   5   0   0
                  ↑               ↑
                  L=2             R+1=6
```

Counter is 5 exactly at indices 2, 3, 4, 5. That's the range [2, 5]. ✓

### Why R+1, Not R?

R itself should still receive the addition. We turn off **starting at the index after R**.

### The Two-Step Recipe

```
For each range update [L, R] with value v:
    diff[L]   += v
    diff[R+1] -= v

After all updates, prefix sum diff to get the final array:
    A[i] = A[i-1] + diff[i]
```

### Java Code

```java
public class DifferenceArray {
    private int[] diff;
    private int n;

    public DifferenceArray(int size) {
        n = size;
        diff = new int[size + 1]; // +1 for safe R+1 indexing
    }

    public void rangeUpdate(int L, int R, int v) {
        diff[L] += v;
        if (R + 1 < diff.length) {
            diff[R + 1] -= v;
        }
    }

    public int[] build() {
        int[] result = new int[n];
        result[0] = diff[0];
        for (int i = 1; i < n; i++) {
            result[i] = result[i - 1] + diff[i];
        }
        return result;
    }
}
```

### Worked Example

Start: `A = [0, 0, 0, 0, 0]`

Instructions:
1. Add 10 to [0, 2]
2. Add 7 to [2, 4]

After recording:
```
index:   0    1    2    3    4    5
diff:  +10    0   +7  -10    0   -7
```

Prefix sum:
```
index:    0    1    2    3    4
running: 10   10   17    7    7
```

Final `A = [10, 10, 17, 7, 7]`. ✓

---

## 3. The Pattern in One Line

> **When each input contributes a value to a range of some indexed quantity, use a difference array, then prefix-sum at the end.**

This converts `O(updates × range_size)` into `O(updates + array_size)`.

---

## 4. Application: Minimum Moves to Make Array Complementary

### Problem (LeetCode 1674)

Given `nums` of even size `n` and integer `limit`:
- Each `nums[i]` is in `[1, limit]`.
- One **move** = change any `nums[i]` to any value in `[1, limit]`.
- Array is **complementary** if `nums[i] + nums[n-1-i] == x` for all `i`, same `x`.

Find the **minimum number of moves** to make the array complementary.

### Step 1: Analyze One Pair

For a pair `(a, b)` with `mn = min(a,b)`, `mx = max(a,b)`, `s = a+b`:

| Target sum `x`               | Moves needed |
|------------------------------|--------------|
| `x = s`                      | 0            |
| `x` in `[1+mn, limit+mx]` but `x ≠ s` | 1            |
| `x` outside that range       | 2            |

**Why?**
- With **1 move**: change one element to anywhere in `[1, limit]`.
  - Smallest reachable sum: `1 + mn` (set the smaller one to `1`... wait, keep `mn` is wrong — keep the larger, set the other to 1 → sum = `1 + mx`. Keep the smaller, set the other to 1 → sum = `1 + mn`. The lowest is `1 + mn`.)
  - Actually: to get the **lowest** sum with one move, set one element to 1, keeping the smaller (`mn`) gives `1 + mn`. Keeping the larger (`mx`) gives `1 + mx`. Both are achievable, so 1-move range is union, which spans `[1 + mn, limit + mx]`.
- With **2 moves**: any sum in `[2, 2·limit]`.

### Step 2: Range Updates on `x`

For each pair, contribute costs across `x ∈ [2, 2·limit]`:

1. Default everywhere: **add 2** to all `x` in `[2, 2·limit]`
2. In 1-move range `[1+mn, limit+mx]`: **subtract 1** (cost is 1, not 2)
3. At `x = s`: **subtract 1 more** (cost is 0, not 1)

That's three range updates per pair on an array indexed by `x`. Perfect fit for the difference array.

### Step 3: Java Solution

```java
public class Solution {
    public int minMoves(int[] nums, int limit) {
        int n = nums.length;
        // Size 2*limit + 2 to safely index R+1 when R = 2*limit
        int[] diff = new int[2 * limit + 2];

        for (int i = 0; i < n / 2; i++) {
            int a = nums[i];
            int b = nums[n - 1 - i];
            int mn = Math.min(a, b);
            int mx = Math.max(a, b);
            int s = a + b;

            // Default 2 moves for every x in [2, 2*limit]
            diff[2] += 2;
            diff[2 * limit + 1] -= 2;

            // In [1+mn, limit+mx]: save 1 (1 move suffices)
            diff[1 + mn] -= 1;
            diff[limit + mx + 1] += 1;

            // At x = s: save 1 more (0 moves)
            diff[s] -= 1;
            diff[s + 1] += 1;
        }

        // Prefix-sum sweep, tracking the minimum
        int best = n;
        int running = 0;
        for (int x = 2; x <= 2 * limit; x++) {
            running += diff[x];
            best = Math.min(best, running);
        }
        return best;
    }
}
```

### Step 4: Walk-Through

`nums = [1, 2, 4, 3]`, `limit = 4`. Pairs: `(1,3)` and `(2,4)`. `x` ranges over `[2, 8]`.

**Pair (1, 3):** mn=1, mx=3, s=4
- `diff[2] += 2`, `diff[9] -= 2`
- `diff[2] -= 1`, `diff[8] += 1`
- `diff[4] -= 1`, `diff[5] += 1`

**Pair (2, 4):** mn=2, mx=4, s=6
- `diff[2] += 2`, `diff[9] -= 2`
- `diff[3] -= 1`, `diff[9] += 1`
- `diff[6] -= 1`, `diff[7] += 1`

Final diff (indices 2–8):
```
x:     2   3   4   5   6   7   8
diff: +3  -1  -1  +1  -1  +1  +1
```

Prefix-sum sweep:
```
x=2: 3
x=3: 2
x=4: 1   ← min
x=5: 2
x=6: 1   ← tied min
x=7: 2
x=8: 3
```

**Answer: 1 move.** Sanity check at `x = 4`: pair (1,3) sums to 4 already (0 moves); pair (2,4) needs to drop from 6 to 4 (change one element, 1 move). Total = 1 ✓

### Complexity

- **Time:** O(n + limit)
- **Space:** O(limit)

Naive would be O(n × limit). Diff array is the speedup.

---

## 5. Cheat Sheet

### Prefix Sum
```java
prefix[0] = A[0];
for (int i = 1; i < n; i++) prefix[i] = prefix[i-1] + A[i];
// range [L, R] sum:
int sum = (L == 0) ? prefix[R] : prefix[R] - prefix[L-1];
```

### Difference Array
```java
// add v to range [L, R]:
diff[L]   += v;
diff[R+1] -= v;

// reconstruct after all updates:
A[0] = diff[0];
for (int i = 1; i < n; i++) A[i] = A[i-1] + diff[i];
```

### Pattern Recognition Checklist

Use a difference array when:
- [ ] You're updating ranges (not points)
- [ ] You only need the final state (not intermediate)
- [ ] Each input contributes to ranges of values
- [ ] Total updates × range size is too large

---

## 6. Related Problems to Practice

1. **LeetCode 1109** — Corporate Flight Bookings (classic range updates)
2. **LeetCode 1893** — Check if All the Integers in a Range Are Covered
3. **LeetCode 2381** — Shifting Letters II
4. **LeetCode 1854** — Maximum Population Year
5. **LeetCode 1674** — Minimum Moves to Make Array Complementary (this problem)

---

*Notes compiled from a walk-through session. Practice the pattern on the related problems above to lock it in.*
