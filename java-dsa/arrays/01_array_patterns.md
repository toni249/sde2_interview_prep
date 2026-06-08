# Arrays — Pattern Deep Dive
> Teaching style: Identify pattern → Core trick → Template → Variations (Easy → Hard)
> Questions sourced from: Striver Sheet + LeetCode

---

## Pattern Map (Visualized)

```
ARRAYS
├── P1: Prefix Sum          → "range sum query", "subarray sum = k"
├── P2: Difference Array    → "range updates, query at end"
├── P3: Two Pointers        → "sorted array", "pair/triplet sum", "partition"
├── P4: Sliding Window      → "window of size k", "longest subarray with condition"
├── P5: Kadane's            → "max subarray", "max product subarray"
├── P6: Binary Search       → "sorted / rotated array", "minimize max"
├── P7: Sorting + Greedy    → "merge intervals", "meeting rooms"
├── P8: HashMap/Freq Map    → "count subarrays", "first missing", "duplicates"
├── P9: Monotonic Stack     → "next greater/smaller", "histogram", "stock span"
└── P10: Matrix             → "spiral", "rotate", "diagonal", "set zeros"
```

---

## P1: Prefix Sum

### The Core Idea
> "Precompute running totals so any range sum becomes O(1)"

```
A      = [3, 1, 4, 1, 5]
prefix = [3, 4, 8, 9, 14]

sum(L=1, R=3) = prefix[3] - prefix[0] = 9 - 3 = 6
                                  ^check: 1+4+1 = 6 ✓
```

### Recognition Trigger
- "sum of subarray from L to R"
- "subarray sum equals k" (combine with HashMap)
- "equilibrium index" (leftSum == rightSum)
- 2D version: "sum of rectangle in matrix"

### Template
```java
// Build
int[] prefix = new int[n];
prefix[0] = A[0];
for (int i = 1; i < n; i++) prefix[i] = prefix[i-1] + A[i];

// Query
int rangeSum(int L, int R) {
    return L == 0 ? prefix[R] : prefix[R] - prefix[L-1];
}
```

### The HashMap Trick (★ Very Common) — Subarray Sum Equals K (LC 560)

**Problem:** Given an integer array `nums` (can have negatives) and integer `k`, return the **total count** of contiguous subarrays whose sum equals `k`.

**Approach (Prefix Sum + HashMap):**
- Naive O(n²) checks every subarray. Smart move: walk left→right computing `running = prefix[j]`. Any subarray ending at `j` with sum `k` corresponds to an earlier `prefix[i] = running - k`.
- Store frequencies of prefix sums seen so far in a HashMap; at each step, add `freq[running - k]` to the answer.
- Seed with `freq[0] = 1` so subarrays starting at index 0 are counted (empty prefix before the array).
- Works with negatives — sliding window would not.
- Time O(n), Space O(n).

```java
// Count subarrays with sum = k
int countSubarrays(int[] A, int k) {
    Map<Integer, Integer> freq = new HashMap<>();
    freq.put(0, 1);   // one way to have prefix sum 0 before processing any element
    int running = 0, count = 0;
    for (int x : A) {
        running += x;
        count += freq.getOrDefault(running - k, 0);
        freq.merge(running, 1, Integer::sum);
    }
    return count;
}
```

`freq.put(0, 1)` does not mean the empty prefix has value `1`.
It means prefix sum `0` has been seen once already: the empty prefix before the array starts.
This is what allows subarrays that begin at index `0` to be counted correctly.

### Example: Longest Subarray with Sum K

**Problem:** Given an array `A` (which may contain negative numbers) and integer `k`, return the **length** of the longest contiguous subarray whose sum equals `k`. Return 0 if none exists.

**Approach (Prefix Sum + HashMap, store earliest index):**
- Same observation as LC 560: subarray `(i, j]` sums to `k` iff `prefix[j] - prefix[i] = k`.
- To **maximize length**, we want the earliest `i` for each prefix value — use `putIfAbsent` so the map never overwrites a smaller index.
- For each `j`, look up `running - k`; if present, candidate length = `j - firstIndex[running - k]`.
- Seed `firstIndex.put(0, -1)` so subarrays starting at index 0 are length `j - (-1) = j + 1`.
- If the array is **all positive**, sliding window is simpler and uses O(1) extra space; use this hashmap approach when negatives are possible.
- Time O(n), Space O(n).

```java
int longestSubarrayWithSumK(int[] A, int k) {
    Map<Integer, Integer> firstIndex = new HashMap<>();
    firstIndex.put(0, -1); // prefix sum before the array starts

    int running = 0, maxLen = 0;
    for (int i = 0; i < A.length; i++) {
        running += A[i];

        if (firstIndex.containsKey(running - k)) {
            int start = firstIndex.get(running - k);
            maxLen = Math.max(maxLen, i - start);
        }

        // Keep earliest index only, because earliest gives longest length.
        firstIndex.putIfAbsent(running, i);
    }
    return maxLen;
}
```

Trace:
```
A = [10, 5, 2, 7, 1, 9], k = 15

i  A[i]  running  need(running-k)  longest
0  10    10       -5               0
1   5    15        0               2   -> [10, 5]
2   2    17        2               2
3   7    24        9               2
4   1    25       10               4   -> [5, 2, 7, 1]
5   9    34       19               4
```

### Questions — Easy → Hard

| # | Problem | Pattern Used | LC# |
|---|---------|-------------|-----|
| E1 | Range Sum Query (immutable) | basic prefix | 303 |
| E2 | Find Pivot Index | leftSum == rightSum | 724 |
| E3 | Running Sum of 1D Array | direct prefix | 1480 |
| M1 | Subarray Sum Equals K | prefix + hashmap | 560 |
| M2 | Longest Subarray with Sum K | prefix + hashmap | — |
| M3 | Count of Subarrays with 0s and 1s equal | prefix (0→-1 trick) | — |
| M4 | Contiguous Array (equal 0s and 1s) | prefix + hashmap | 525 |
| M5 | 2D Range Sum Query | 2D prefix | 304 |
| H1 | Count of Subarrays with XOR = k | prefix XOR + hashmap | — |
| H2 | Minimum Moves to Make Array Complementary | prefix + diff array | 1674 |

### The 0→-1 Trick (for 0s and 1s problems)
> Replace 0 with -1. Now "equal 0s and 1s" becomes "subarray sum = 0".
```
[1,0,1,1,0] → [1,-1,1,1,-1]
Longest subarray with sum=0 → longest with equal 0s and 1s.
```

---

## P2: Difference Array

> Already covered in: `difference_array_prefix_sum_notes.md`
> Key: Range updates in O(1), reconstruct in O(n).

### Questions

| # | Problem | LC# |
|---|---------|-----|
| E1 | Corporate Flight Bookings | 1109 |
| M1 | Car Pooling | 1094 |
| M2 | Shifting Letters II | 2381 |
| H1 | Minimum Moves to Make Array Complementary | 1674 |

---

## P3: Two Pointers

### The Core Idea
> Use two indices that move toward/away from each other to avoid O(n²) nested loops.

### When to Use
- Array is **sorted** (or you can sort it)
- Looking for a **pair or triplet** with a condition
- "Remove duplicates", "partition", "palindrome check"

### Three Flavors

#### Flavor A: Converging (opposite ends)
```java
// Pair sum = target in sorted array
int left = 0, right = n - 1;
while (left < right) {
    int sum = A[left] + A[right];
    if (sum == target) { /* found */ }
    else if (sum < target) left++;
    else right--;
}
```

#### Flavor B: Same direction (fast/slow)
```java
// Remove duplicates from sorted array
int slow = 0;
for (int fast = 1; fast < n; fast++) {
    if (A[fast] != A[slow]) {
        A[++slow] = A[fast];
    }
}
// slow+1 is the new length
```

#### Flavor C: Dutch National Flag (3-way partition) — Sort Colors (LC 75)

**Problem:** Array contains only values 0, 1, 2 (representing red/white/blue). Sort it **in-place in one pass** using O(1) extra space — no library sort, no counting then rewriting.

**Approach (3-way partition / Dutch National Flag):**
- Maintain three regions via three pointers: `[0..lo-1]` = 0s, `[lo..mid-1]` = 1s, `[hi+1..n-1]` = 2s. The unknown region is `[mid..hi]`.
- Inspect `A[mid]`:
  - `0` → swap into the 0s region: `swap(lo, mid)`, advance both.
  - `1` → already in place, just advance `mid`.
  - `2` → swap to the back: `swap(mid, hi--)` — do **not** advance `mid` because the swapped-in element is still unknown.
- Loop while `mid <= hi`. Single pass, O(n) time, O(1) space.

```java
// Sort 0s, 1s, 2s
int lo = 0, mid = 0, hi = n - 1;
while (mid <= hi) {
    if (A[mid] == 0) swap(A, lo++, mid++);
    else if (A[mid] == 1) mid++;
    else swap(A, mid, hi--);
}
```

### Key Variations

**Three Sum (LC 15)**

**Problem:** Given an integer array `nums`, return **all unique triplets** `[a, b, c]` such that `a + b + c == 0`. The result must not contain duplicate triplets.

**Approach (Sort + fix one + two pointers):**
- Sort the array — this both enables two-pointer search and makes duplicate skipping easy (equal values are adjacent).
- Fix `A[i]` as the smallest of the triplet; for each `i`, two-pointer search the remaining suffix for a pair summing to `-A[i]`.
- Skip duplicates on `i`, `left`, and `right` after finding/moving to avoid duplicate triplets.
- Early break when `A[i] > 0` (sum can never be 0 anymore).
- Time O(n²), Space O(1) extra (output excluded).

```java
Arrays.sort(A);
for (int i = 0; i < n - 2; i++) {
    if (i > 0 && A[i] == A[i-1]) continue; // skip duplicates
    int left = i + 1, right = n - 1;
    while (left < right) {
        int sum = A[i] + A[left] + A[right];
        if (sum == 0) { /* found, move both, skip duplicates */ }
        else if (sum < 0) left++;
        else right--;
    }
}
```

**Trapping Rain Water (LC 42)**

**Problem:** Given non-negative integers representing an elevation map (bar heights of width 1), compute how much rain water can be trapped between the bars after raining.

**Approach (Two Pointers with running max):**
- Water above index `i` = `min(maxLeft[i], maxRight[i]) - height[i]`. A prefix-max + suffix-max approach uses O(n) space.
- Two-pointer optimization: at each step, the side with the **smaller** current bar is the bottleneck — water trapped there is fully determined by that side's running max (the other side is guaranteed taller).
- Move the smaller side inward, updating its running max and accumulating `runningMax - currentHeight`.
- Time O(n), Space O(1). Compare with the monotonic stack approach (P9) which also runs O(n) but uses O(n) stack space.

```java
int left = 0, right = n - 1;
int leftMax = 0, rightMax = 0, water = 0;
while (left <= right) {
    if (A[left] <= A[right]) {
        leftMax = Math.max(leftMax, A[left]);
        water += leftMax - A[left];
        left++;
    } else {
        rightMax = Math.max(rightMax, A[right]);
        water += rightMax - A[right];
        right--;
    }
}
```

### Questions — Easy → Hard

| # | Problem | Flavor | LC# |
|---|---------|--------|-----|
| E1 | Two Sum II (sorted array) | Converging | 167 |
| E2 | Valid Palindrome | Converging | 125 |
| E3 | Remove Duplicates from Sorted Array | Fast/Slow | 26 |
| E4 | Move Zeroes | Fast/Slow | 283 |
| M1 | Three Sum | Converging | 15 |
| M2 | Container With Most Water | Converging | 11 |
| M3 | Sort Colors (0,1,2) | Dutch Flag | 75 |
| M4 | Four Sum | Converging | 18 |
| M5 | 3Sum Closest | Converging | 16 |
| H1 | Trapping Rain Water | Converging + tracking | 42 |
| H2 | Minimum Window to Sort (Subarray sort) | Two passes | 581 |

---

## P4: Sliding Window

### The Core Idea
> Maintain a "window" [left, right] over the array. Expand right, shrink left when condition breaks.

### Two Types

#### Type 1: Fixed Window (size = k)
```java
// Max sum of subarray of size k
int windowSum = 0;
for (int i = 0; i < k; i++) windowSum += A[i];
int maxSum = windowSum;

for (int i = k; i < n; i++) {
    windowSum += A[i] - A[i - k];  // slide: add new, remove old
    maxSum = Math.max(maxSum, windowSum);
}
```

#### Type 2: Variable Window (shrink when invalid)
```java
// Longest subarray with sum ≤ target (all positives)
int left = 0, windowSum = 0, maxLen = 0;
for (int right = 0; right < n; right++) {
    windowSum += A[right];                    // expand
    while (windowSum > target) {              // invalid — shrink
        windowSum -= A[left++];
    }
    maxLen = Math.max(maxLen, right - left + 1);
}
```

### Recognition Trigger
- Fixed: "max/min/average of **exactly k** elements"
- Variable: "**longest/shortest** subarray/substring with some property"
- Key constraint: elements are **non-negative** for simple shrinking (else use prefix + hashmap)

### Tricky Case: Negative Numbers
If the array can contain negative numbers, the usual sliding window rule breaks.

Why:
- Sliding window works when expanding the right end only makes the condition "worse" in a predictable way.
- With negative numbers, adding a new element can increase or decrease the sum.
- So shrinking from the left is no longer guaranteed to move you toward the answer.

Example:
```text
A = [2, -1, 2], target = 3
```
If you expand:
- `[2]` sum = 2
- `[2, -1]` sum = 1
- `[2, -1, 2]` sum = 3

The window does not change monotonically, so a simple "expand until invalid, then shrink" rule is not reliable.

For these cases, use:
- **prefix sum + hashmap** for sum-based problems
- or another approach that does not depend on monotonic window movement

### Important Variations

**At Most K Distinct** → `atMost(k) - atMost(k-1)`

This is a counting trick.

`atMost(k)` = number of subarrays that contain **at most k distinct values**.

Then:
- subarrays with **exactly k distinct values**
- = subarrays with at most `k`
- minus subarrays with at most `k - 1`

Reason:
- `atMost(k)` includes all subarrays with 0, 1, 2, ... `k` distinct values
- `atMost(k - 1)` includes all subarrays with 0, 1, 2, ... `k - 1` distinct values
- subtracting removes everything except the ones with exactly `k`

Example:
```text
nums = [1, 2, 1, 2, 3], k = 2
```
Subarrays with exactly 2 distinct values are counted as:
- `atMost(2)` - `atMost(1)`

This pattern is used in problems like:
- `Subarrays with K Different Integers`
- `Fruit Into Baskets`

**Minimum Window Substring (LC 76)** — Classic Hard

**Problem:** Given strings `s` and `t`, return the **shortest substring** of `s` that contains every character of `t` (counting multiplicity). Return `""` if no such substring exists.

**Approach (Variable Sliding Window + frequency maps):**
- Build `need[c]` = required count of each character from `t`. Maintain a `have` count of "characters currently satisfied" — increment only when a character's window count reaches its `need`.
- Expand `right`, adding chars to the window. Once `have == required distinct chars`, the window is **valid**.
- While valid, try to shrink `left`: record `(right - left + 1)` if smaller than best, then drop `s[left]`. If dropping breaks a `need`, decrement `have` and stop shrinking.
- Each character is visited at most twice (once when expanding, once when shrinking) → O(|s| + |t|) time, O(|s| + |t|) space.

```java
// Two frequency maps: need[] and have[]
// Shrink from left once window is valid
```

Meaning:
- `need[]` tells how many of each character the target string requires
- `have[]` tells how many of each character are currently inside the window
- when the window has all required characters in enough quantity, the window is "valid"

Then:
- expand `right` until the window becomes valid
- once valid, try to shrink `left` to make the window as small as possible

Example:
```text
s = "ADOBECODEBANC", t = "ABC"
```
The window becomes valid when it contains at least:
- one `A`
- one `B`
- one `C`

Then you shrink from the left to remove extra characters while keeping the window valid.

This is different from the "sum" window:
- here we are not checking numeric sum
- we are checking whether the window contains enough characters
- so frequency maps are the right tool

### Questions — Easy → Hard

| # | Problem | Type | LC# |
|---|---------|------|-----|
| E1 | Maximum Average Subarray I | Fixed | 643 |
| E2 | Max Sum of Subarray Size K | Fixed | — |
| M1 | Longest Substring Without Repeating Characters | Variable | 3 |
| M2 | Longest Subarray of 1s After Deleting One Element | Variable | 1493 |
| M3 | Fruit Into Baskets (at most 2 distinct) | Variable | 904 |
| M4 | Subarray Product Less Than K | Variable | 713 |
| M5 | Maximum Sum of Almost Unique Subarray | Fixed | 2841 |
| H1 | Minimum Window Substring | Variable + freq map | 76 |
| H2 | Sliding Window Maximum | Fixed + Monotonic Deque | 239 |
| H3 | Subarrays with K Different Integers | Variable: atMost trick | 992 |
| H4 | Minimum Size Subarray in Infinite Array | Circular window | 2875 |

---

## P5: Kadane's Algorithm

### The Core Idea — Maximum Subarray (LC 53)

**Problem:** Given an integer array `nums` (may contain negatives), find the contiguous subarray with the **largest sum** and return that sum.

**Approach (Kadane's DP):**
- Define `maxEndingHere[i]` = max sum of any subarray ending **exactly** at index `i`.
- Recurrence: `maxEndingHere[i] = max(A[i], maxEndingHere[i-1] + A[i])`. Either start fresh at `i`, or extend the previous best.
- The answer is `max` over all `maxEndingHere[i]`.
- Intuition: if accumulated sum so far is negative, it can only hurt — discard and restart.
- Time O(n), Space O(1) (one variable suffices; the DP collapses).

> At each index, decide: "Do I extend the previous subarray, or start fresh?"

```java
// Maximum Subarray Sum
int maxSoFar = A[0], maxEndingHere = A[0];
for (int i = 1; i < n; i++) {
    maxEndingHere = Math.max(A[i], maxEndingHere + A[i]);
    maxSoFar = Math.max(maxSoFar, maxEndingHere);
}
```

**Why it works:** `maxEndingHere` is the max sum of any subarray ending at index `i`.
If `maxEndingHere` goes negative, it's better to restart from `A[i]`.

### Trace Example
```
A = [-2, 1, -3, 4, -1, 2, 1, -5, 4]

i    A[i]  maxEndingHere  maxSoFar
0    -2    -2             -2
1     1     1              1
2    -3    -2              1
3     4     4              4
4    -1     3              4
5     2     5              5
6     1     6              6  ← answer
7    -5     1              6
8     4     5              6
```

### Variations

**Print the subarray (not just sum):**
```java
// Track start and end indices
int start = 0, end = 0, tempStart = 0;
for (int i = 1; i < n; i++) {
    if (A[i] > maxEndingHere + A[i]) {
        maxEndingHere = A[i];
        tempStart = i;
    } else {
        maxEndingHere += A[i];
    }
    if (maxEndingHere > maxSoFar) {
        maxSoFar = maxEndingHere;
        start = tempStart;
        end = i;
    }
}
```

**Maximum Product Subarray (LC 152):**

**Problem:** Given an integer array `nums`, find the contiguous subarray with the **largest product** and return that product.

**Approach (Kadane variant — track max AND min):**
- Sum is monotonic under addition, but product is not: a large negative × another negative can flip to the new maximum. So we cannot just track the running max.
- Track both `maxProd` and `minProd` ending at the current index. When `A[i] < 0`, swap them — the previous max times a negative is the new candidate for min, and vice versa.
- At each step: `maxProd = max(A[i], maxProd * A[i])`, `minProd = min(A[i], minProd * A[i])`. Update global answer with `maxProd`.
- Edge cases: zeros reset both to `A[i]`; single-element arrays return `A[0]`.
- Time O(n), Space O(1).

> Key insight: track both max and min (because negative × negative = positive)
```java
int maxProd = A[0], minProd = A[0], result = A[0];
for (int i = 1; i < n; i++) {
    if (A[i] < 0) { int tmp = maxProd; maxProd = minProd; minProd = tmp; }
    maxProd = Math.max(A[i], maxProd * A[i]);
    minProd = Math.min(A[i], minProd * A[i]);
    result = Math.max(result, maxProd);
}
```

**Maximum Circular Subarray Sum (LC 918):**

**Problem:** Given a **circular** integer array `nums` (the end wraps to the beginning), find the maximum possible sum of a non-empty contiguous subarray. A subarray may wrap around.

**Approach (Two-case Kadane):**
- Case A — best subarray does **not** wrap: standard Kadane on `nums`.
- Case B — best subarray **wraps**: the unused middle part is then a contiguous non-wrapping subarray with the **minimum** sum. So `bestWrap = totalSum - minSubarraySum`.
- Answer = `max(caseA, caseB)`. Equivalent trick: `min` Kadane = `-Kadane(negated array)`.
- Edge case: if all numbers are negative, `minSubarraySum == totalSum`, making `caseB = 0` (empty subarray), which is invalid → return `caseA`.
- Time O(n), Space O(1).

```
Answer = max(
    normal Kadane's,                         // non-wrapping
    totalSum - Kadane's on negated array     // wrapping = total - min subarray
)
```

### Questions — Easy → Hard

| # | Problem | Variation | LC# |
|---|---------|-----------|-----|
| E1 | Maximum Subarray | base Kadane | 53 |
| M1 | Maximum Product Subarray | track min too | 152 |
| M2 | Maximum Sum Circular Subarray | total - min | 918 |
| M3 | Longest Turbulent Subarray | up-down pattern | 978 |
| H1 | Maximum Subarray Sum with One Deletion | DP variant | 1186 |
| H2 | K-Concatenation Maximum Sum | Kadane + formula | 1191 |

---

## P6: Binary Search on Arrays

### Two Scenarios

#### Scenario A: Direct Binary Search on Sorted Array
```java
int left = 0, right = n - 1;
while (left <= right) {
    int mid = left + (right - left) / 2;
    if (A[mid] == target) return mid;
    else if (A[mid] < target) left = mid + 1;
    else right = mid - 1;
}
```

#### Scenario B: Binary Search on **Answer Space** (★★★ Very Important)
> "Minimize the maximum" or "Maximize the minimum" → think BS on answer

**Template:**
```java
int left = minPossibleAnswer, right = maxPossibleAnswer;
while (left < right) {
    int mid = left + (right - left) / 2;
    if (isFeasible(mid)) right = mid;  // or left = mid + 1 for maximize
    else left = mid + 1;
}
return left;
```

**isFeasible(mid)** = "can we achieve this answer with mid as the constraint?"

### Rotated Sorted Array — Search in Rotated Sorted Array (LC 33)

**Problem:** A sorted ascending array of **distinct** values has been rotated at an unknown pivot (e.g., `[4,5,6,7,0,1,2]`). Given a target, return its index, or `-1` if absent. Must run in O(log n).

**Approach (Modified Binary Search):**
- Key invariant: at any `mid`, **at least one of the two halves** `[left..mid]` and `[mid..right]` is fully sorted (no pivot inside it). Check which.
- If `A[left] <= A[mid]`, the left half is sorted. Check if `target` is in `[A[left], A[mid])` — if yes, go left; else go right.
- Otherwise the right half is sorted. Check if `target` is in `(A[mid], A[right]]` — if yes, go right; else go left.
- This narrows by half each step → O(log n) time, O(1) space.
- Caveat (LC 81 variant): with duplicates, `A[left] == A[mid] == A[right]` makes the sorted-half test ambiguous; in that case advance `left++` / `right--` and continue → worst-case O(n).

```java
// Search in rotated sorted array
// Key: one half is always sorted
int left = 0, right = n - 1;
while (left <= right) {
    int mid = left + (right - left) / 2;
    if (A[mid] == target) return mid;
    // Left half sorted
    if (A[left] <= A[mid]) {
        if (A[left] <= target && target < A[mid]) right = mid - 1;
        else left = mid + 1;
    } else { // Right half sorted
        if (A[mid] < target && target <= A[right]) left = mid + 1;
        else right = mid - 1;
    }
}
```

### Questions — Easy → Hard

| # | Problem | Type | LC# |
|---|---------|------|-----|
| E1 | Binary Search | Classic | 704 |
| E2 | First Bad Version | Classic | 278 |
| M1 | Search in Rotated Sorted Array | Rotated | 33 |
| M2 | Find Minimum in Rotated Sorted Array | Rotated | 153 |
| M3 | Peak Index in Mountain Array | Peak finding | 852 |
| M4 | Koko Eating Bananas | BS on answer | 875 |
| M5 | Find Peak Element | BS | 162 |
| H1 | Split Array Largest Sum | BS on answer | 410 |
| H2 | Aggressive Cows | BS on answer (maximize min) | — |
| H3 | Median of Two Sorted Arrays | BS | 4 |
| H4 | Capacity to Ship Packages Within D Days | BS on answer | 1011 |

### "BS on Answer" Recognition Checklist
- [ ] The answer lies in a **range** [lo, hi]
- [ ] You can write a **feasibility function**: "can we do it with this value?"
- [ ] The feasibility function is **monotonic** (once feasible, always feasible above/below)

---

## P7: Sorting + Greedy

### When to Use
- "Merge overlapping intervals"
- "Meeting rooms / scheduling"
- "Minimum platforms"
- Sort + scan

### Merge Intervals Template — Merge Intervals (LC 56)

**Problem:** Given an array of intervals `[[start, end], ...]`, merge all overlapping intervals and return the resulting list (intervals are considered overlapping if they share at least one point).

**Approach (Sort by start + linear sweep):**
- Sort intervals by start time. After sorting, any interval that overlaps the current "open" interval must come right after it (no later interval has an earlier start).
- Maintain the last merged interval in the result. For the next interval `cur`:
  - If `cur.start <= last.end` → overlap; extend `last.end = max(last.end, cur.end)`.
  - Else → no overlap; push `cur` as a new entry.
- One pass after sort. Time O(n log n) for sort, O(n) sweep. Space O(n) for output.
- Variants: LC 57 (Insert Interval), LC 435 (Non-overlapping — sort by `end` and greedy).

```java
Arrays.sort(intervals, (a, b) -> a[0] - b[0]);
List<int[]> result = new ArrayList<>();
result.add(intervals[0]);
for (int i = 1; i < intervals.length; i++) {
    int[] last = result.get(result.size() - 1);
    if (intervals[i][0] <= last[1]) {           // overlap
        last[1] = Math.max(last[1], intervals[i][1]);
    } else {
        result.add(intervals[i]);
    }
}
```

### Questions — Easy → Hard

| # | Problem | LC# |
|---|---------|-----|
| E1 | Sort Colors | 75 |
| M1 | Merge Intervals | 56 |
| M2 | Insert Interval | 57 |
| M3 | Non-overlapping Intervals | 435 |
| M4 | Minimum Number of Platforms (Meeting Rooms II) | 253 |
| M5 | Jump Game | 55 |
| H1 | Jump Game II | 45 |
| H2 | Gas Station | 134 |
| H3 | Largest Number | 179 |

---

## P8: HashMap / Frequency Map

### Core Uses

1. **Count frequencies** → O(1) lookup
2. **Subarray with sum k** → prefix + map (covered in P1)
3. **First missing positive** → mark visited in-place or use set
4. **Two sum** → store complements in map

**Two Sum (LC 1)**

**Problem:** Given an integer array `nums` and an integer `target`, return the **indices** of the two numbers that add to `target`. Exactly one solution exists; you may not use the same element twice.

**Approach (HashMap of complements, one pass):**
- For each `A[i]`, the complement needed is `target - A[i]`. If we've already seen the complement, we have our pair.
- Walk the array once; before inserting `A[i]` into the map, check if `target - A[i]` is already a key.
- Insert **after** checking so the same index isn't reused.
- Time O(n), Space O(n). Compare: sort + two-pointer is O(n log n) and loses original indices.

```java
// Two Sum
Map<Integer, Integer> map = new HashMap<>();
for (int i = 0; i < n; i++) {
    if (map.containsKey(target - A[i])) return new int[]{map.get(target-A[i]), i};
    map.put(A[i], i);
}
```

### First Missing Positive — LC 41 (No extra space)

**Problem:** Given an unsorted integer array `nums`, return the **smallest positive integer** missing from it. Must run in O(n) time and use O(1) extra space.

**Approach (Cyclic sort / in-place hashing):**
- Observation: the answer must lie in `[1, n+1]` where `n = nums.length`. Any value `≤ 0` or `> n` is irrelevant.
- Use the array itself as a hash: place value `x` at index `x - 1` (so `A[i] == i + 1` becomes the "in place" condition).
- For each index, while `A[i]` is in `[1, n]` and `A[A[i]-1] != A[i]` (avoid infinite loop on duplicates), swap `A[i]` into its correct slot.
- Second pass: the first index where `A[i] != i + 1` gives answer `i + 1`. If all match, answer is `n + 1`.
- Time O(n) amortized — each swap places one value permanently. Space O(1).

```java
// Use array itself as a hash map
// Place each number x at index x-1
for (int i = 0; i < n; i++) {
    while (A[i] > 0 && A[i] <= n && A[A[i]-1] != A[i]) {
        swap(A, i, A[i]-1);
    }
}
for (int i = 0; i < n; i++) {
    if (A[i] != i + 1) return i + 1;
}
return n + 1;
```

### Questions — Easy → Hard

| # | Problem | LC# |
|---|---------|-----|
| E1 | Two Sum | 1 |
| E2 | Contains Duplicate | 217 |
| M1 | Subarray Sum Equals K | 560 |
| M2 | Longest Consecutive Sequence | 128 |
| M3 | Group Anagrams | 49 |
| M4 | Top K Frequent Elements | 347 |
| H1 | First Missing Positive | 41 |
| H2 | Minimum Window Substring | 76 |
| H3 | Count of Subarrays with XOR = k | — |

---

## P9: Monotonic Stack

### The Core Idea
> Maintain a stack that is always **increasing** or **decreasing**.
> This gives O(n) solution to "next greater/smaller element" problems.

### Next Greater Element Template (LC 496 / 503 / 739)

**Problem:** For each index `i`, find the value of the **next** element to its right that is strictly greater than `A[i]`. If none exists, output `-1` for that index.

**Approach (Monotonic Decreasing Stack):**
- Maintain a stack of **indices** whose values are in decreasing order from bottom to top (i.e., waiting for their "next greater").
- When processing `A[i]`, pop every index whose value is `< A[i]` — `A[i]` is their answer.
- Push `i`. Indices left in the stack at the end have no next greater → keep their default `-1`.
- Each index is pushed and popped at most once → O(n) amortized. Space O(n).
- Variants: LC 503 (circular array) — iterate `2n` times with `i % n`. LC 739 (Daily Temperatures) — same template but store the **distance** `i - stack.pop()`.

```java
int[] result = new int[n];
Arrays.fill(result, -1);
Deque<Integer> stack = new ArrayDeque<>();  // stores indices

for (int i = 0; i < n; i++) {
    // Pop elements smaller than current (current is their "next greater")
    while (!stack.isEmpty() && A[stack.peek()] < A[i]) {
        result[stack.pop()] = A[i];
    }
    stack.push(i);
}
```

### Largest Rectangle in Histogram (LC 84)

**Problem:** Given non-negative integers representing the heights of histogram bars (each width 1), return the area of the **largest rectangle** that fits entirely within the histogram.

**Approach (Monotonic Increasing Stack):**
- For each bar `h`, the largest rectangle with `h` as its **minimum height** spans from "previous smaller bar + 1" on the left to "next smaller bar - 1" on the right. Compute these boundaries with one stack pass.
- Keep a stack of indices in increasing height order. When the new bar's height is smaller than `heights[stack.peek()]`, pop and finalize the popped bar's rectangle: `height = heights[popped]`, `width = i - stack.peek() - 1` (or `i` if stack is empty).
- Sentinel trick: append a virtual `height = 0` at the end (`i == n`) so all remaining bars get flushed cleanly.
- Each bar pushed/popped once → O(n) time, O(n) space.
- This is the key building block for LC 85 (Maximal Rectangle in a binary matrix): run this row-by-row on accumulated histogram heights.

```java
// Classic hard problem — uses monotonic stack
// Stack stores indices of bars in increasing height order
Deque<Integer> stack = new ArrayDeque<>();
int maxArea = 0;
for (int i = 0; i <= n; i++) {
    int h = (i == n) ? 0 : heights[i];
    while (!stack.isEmpty() && heights[stack.peek()] > h) {
        int height = heights[stack.pop()];
        int width = stack.isEmpty() ? i : i - stack.peek() - 1;
        maxArea = Math.max(maxArea, height * width);
    }
    stack.push(i);
}
```

### Questions — Easy → Hard

| # | Problem | LC# |
|---|---------|-----|
| E1 | Next Greater Element I | 496 |
| M1 | Daily Temperatures | 739 |
| M2 | Stock Span Problem | — |
| M3 | Next Greater Element II (circular) | 503 |
| M4 | Sum of Subarray Minimums | 907 |
| H1 | Largest Rectangle in Histogram | 84 |
| H2 | Maximal Rectangle (matrix) | 85 |
| H3 | Trapping Rain Water (stack approach) | 42 |

---

## P10: Matrix Traversal

### Common Tricks

**Spiral Order — Spiral Matrix (LC 54):**

**Problem:** Given an `m × n` matrix, return all elements in **spiral order**: traverse the outermost ring clockwise (right, down, left, up), then move inward and repeat.

**Approach (Four shrinking boundaries):**
- Maintain `top`, `bottom`, `left`, `right` boundaries. In each loop iteration, walk one full ring:
  1. Left → right along `top`, then `top++`.
  2. Top → bottom along `right`, then `right--`.
  3. Right → left along `bottom` (only if `top <= bottom`), then `bottom--`.
  4. Bottom → top along `left` (only if `left <= right`), then `left++`.
- The two guard conditions handle non-square matrices (single remaining row or column) without revisiting cells.
- Time O(m·n), Space O(1) extra (excluding output).

```java
int top=0, bottom=m-1, left=0, right=n-1;
while (top<=bottom && left<=right) {
    for(int i=left; i<=right; i++)   result.add(matrix[top][i]);   top++;
    for(int i=top; i<=bottom; i++)   result.add(matrix[i][right]); right--;
    if(top<=bottom) for(int i=right; i>=left; i--) result.add(matrix[bottom][i]); bottom--;
    if(left<=right) for(int i=bottom; i>=top; i--) result.add(matrix[i][left]);   left++;
}
```

**Rotate 90° Clockwise — Rotate Image (LC 48):**

**Problem:** Given an `n × n` 2D matrix representing an image, rotate it 90 degrees clockwise **in-place** (no extra matrix).

**Approach (Transpose + reverse rows):**
- Observation: rotating 90° clockwise = transpose (reflect along main diagonal) + reverse each row.
- Step 1: For `i < j`, swap `matrix[i][j]` with `matrix[j][i]`. After this, columns become rows (but mirrored).
- Step 2: Reverse each row to fix the mirroring.
- Counter-clockwise rotation = transpose + reverse each **column** (or reverse rows then transpose).
- Time O(n²), Space O(1).

```
Step 1: Transpose (swap matrix[i][j] with matrix[j][i])
Step 2: Reverse each row
```

**Set Matrix Zeros — Set Matrix Zeroes (LC 73):**

**Problem:** Given an `m × n` matrix, if a cell is `0`, set its **entire row and column** to `0`. Do it **in-place** with O(1) extra space (the obvious O(m+n) auxiliary arrays are not allowed for the optimal solution).

**Approach (First row / first column as in-place markers):**
- Use the matrix's first row and first column themselves to record which rows/cols need zeroing — saves the auxiliary arrays.
- Pre-pass: separately record whether the first row and first column originally contain a zero (two booleans), because they'll be overwritten as markers.
- Scan the interior `(i, j)` with `i >= 1, j >= 1`: if `matrix[i][j] == 0`, set `matrix[i][0] = 0` and `matrix[0][j] = 0`.
- Second pass on interior: if `matrix[i][0] == 0` or `matrix[0][j] == 0`, set `matrix[i][j] = 0`.
- Finally, zero out the first row / first column based on the two saved booleans.
- Time O(m·n), Space O(1).

```
Step 1: Use first row and first column as markers
Step 2: Process inner matrix, then boundaries
```

### Questions — Easy → Hard

| # | Problem | LC# |
|---|---------|-----|
| E1 | Transpose Matrix | 867 |
| M1 | Spiral Matrix | 54 |
| M2 | Rotate Image | 48 |
| M3 | Set Matrix Zeroes | 73 |
| M4 | Search a 2D Matrix | 74 |
| M5 | Diagonal Traverse | 498 |
| H1 | Maximal Rectangle | 85 |
| H2 | Number of Islands (array DFS) | 200 |

---

## Pattern Decision Tree

```
Array problem?
│
├─ "range sum" / "subarray sum = k"
│    └─→ Prefix Sum (+ HashMap if counting)
│
├─ "range updates, final values"
│    └─→ Difference Array
│
├─ "pair/triplet/palindrome" + sorted or can sort
│    └─→ Two Pointers
│
├─ "window", "substring", "subarray with condition"
│    ├─ fixed size k? → Sliding Window Fixed
│    └─ longest/shortest? → Sliding Window Variable
│
├─ "maximum/minimum subarray (contiguous)"
│    └─→ Kadane's (track min too if product)
│
├─ "sorted/rotated" + "find element"
│    └─→ Binary Search
│
├─ "minimize max" / "maximize min"
│    └─→ Binary Search on Answer
│
├─ "intervals", "meeting rooms", "schedule"
│    └─→ Sort + Greedy
│
├─ "next greater/smaller", "histogram"
│    └─→ Monotonic Stack
│
├─ "count occurrences", "anagram", "duplicates"
│    └─→ HashMap / Frequency Map
│
└─ "2D matrix traversal"
     └─→ Spiral / Rotation tricks
```

---

## Complexity Quick Reference

| Pattern | Time | Space | Notes |
|---------|------|-------|-------|
| Prefix Sum | O(n) build, O(1) query | O(n) | Static arrays |
| Difference Array | O(n) | O(n) | Range updates |
| Two Pointers | O(n) or O(n log n) with sort | O(1) | Sorted required |
| Sliding Window | O(n) | O(1) or O(k) | Single pass |
| Kadane's | O(n) | O(1) | |
| Binary Search | O(log n) | O(1) | |
| BS on Answer | O(n log(range)) | O(1) | |
| Monotonic Stack | O(n) amortized | O(n) | Each element pushed/popped once |
| HashMap | O(n) | O(n) | |

---

> **Next:** [String Patterns](../02_string_patterns.md) | [Back to Index](../DSA_MASTER_INDEX.md)
