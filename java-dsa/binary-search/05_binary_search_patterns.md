# Binary Search — Pattern Deep Dive
> FAANG Level | Java | Easy → Hard

---

## Pattern Map

```
BINARY SEARCH
├── P1: Classic BS            → sorted array, find target, first/last occurrence
├── P2: Rotated Array         → search, find minimum, count rotations
├── P3: Peak Finding          → peak element, mountain array
├── P4: BS on Answer Space    → minimize max, maximize min — most FAANG problems
└── P5: Advanced              → search in 2D matrix, K-th element, median
```

---

## The Golden Rule of Binary Search

> **Binary search works when you can eliminate half the search space.**
> This means: after computing mid, you can definitively say "the answer is not in [left, mid]" OR "the answer is not in [mid, right]".

### Two Templates

**Template 1: Inclusive [left, right] — for finding exact value**
```java
int left = 0, right = n - 1;
while (left <= right) {
    int mid = left + (right - left) / 2;   // avoids overflow (vs (l+r)/2)
    if (A[mid] == target) return mid;
    else if (A[mid] < target) left = mid + 1;
    else right = mid - 1;
}
return -1;
```

**Template 2: [left, right) — for finding boundary**
```java
// Find first index where condition is true
int left = 0, right = n;   // right is exclusive
while (left < right) {
    int mid = left + (right - left) / 2;
    if (condition(mid)) right = mid;   // shrink right
    else left = mid + 1;
}
return left;   // first true position
```

> Use Template 2 for "first occurrence", "leftmost valid position", "lower bound".

---

## P1: Classic Binary Search

### Variation 1: Standard Search (LC 704)

**Problem:** Given a sorted array `A` and a target value, return the index of `target` in `A`, or -1 if it doesn't exist.

**Approach (Classic BS):**
- **Search space:** index range `[0, n-1]`.
- **Invariant:** at every step, the target (if it exists) lies inside `[left, right]`.
- Compare `A[mid]` to `target`: equal → done; less → answer is in right half (`left = mid+1`); greater → left half (`right = mid-1`).
- Use `mid = left + (right - left) / 2` to avoid integer overflow on large arrays.
- Edge cases: empty array, single element, target smaller than min or larger than max.
- Time: O(log n) | Space: O(1).

```java
int search(int[] A, int target) {
    int left = 0, right = A.length - 1;
    while (left <= right) {
        int mid = left + (right - left) / 2;
        if (A[mid] == target) return mid;
        else if (A[mid] < target) left = mid + 1;
        else right = mid - 1;
    }
    return -1;
}
```

### Variation 2: First and Last Occurrence (LC 34)

**Problem:** Given a sorted array with possible duplicates and a target, return `[firstIndex, lastIndex]` of `target`, or `[-1, -1]` if not found.

**Approach (Biased BS — keep searching after a hit):**
- **Search space:** index range `[0, n-1]`.
- **Predicate:** for first occurrence, after finding `target` at `mid`, the answer might still be further left → record `mid`, then `right = mid - 1`. Symmetric for last occurrence (continue right).
- **Invariant:** `result` always holds the best (smallest for first, largest for last) index seen so far.
- This is essentially two `lower_bound`-style searches; count of occurrences = `last - first + 1`.
- Edge cases: target not present (return -1), all elements equal target (first=0, last=n-1).
- Time: O(log n) | Space: O(1).

```java
// First occurrence (lower bound)
int firstOccurrence(int[] A, int target) {
    int left = 0, right = A.length - 1, result = -1;
    while (left <= right) {
        int mid = left + (right - left) / 2;
        if (A[mid] == target) { result = mid; right = mid - 1; }  // keep looking left
        else if (A[mid] < target) left = mid + 1;
        else right = mid - 1;
    }
    return result;
}

// Last occurrence (upper bound)
int lastOccurrence(int[] A, int target) {
    int left = 0, right = A.length - 1, result = -1;
    while (left <= right) {
        int mid = left + (right - left) / 2;
        if (A[mid] == target) { result = mid; left = mid + 1; }  // keep looking right
        else if (A[mid] < target) left = mid + 1;
        else right = mid - 1;
    }
    return result;
}
// Count occurrences = lastOccurrence - firstOccurrence + 1
```

### Variation 3: Search Insert Position (LC 35)

**Problem:** Given a sorted array and a target, return the index where `target` is found, or where it would be inserted to keep the array sorted.

**Approach (Lower bound / half-open template):**
- **Search space:** `[0, n]` — note `right = n`, not `n-1`. The answer could be one past the last element (insertion at the end).
- **Predicate:** "is `nums[mid] >= target`?" — we want the leftmost index where this is true.
- **Invariant:** at exit, `left == right` and is the first index where `nums[i] >= target`.
- Use the half-open template (`while (left < right)`, `right = mid`, `left = mid + 1`).
- Edge cases: target larger than all elements → returns `n`; smaller than all → returns 0.
- Time: O(log n) | Space: O(1).

```java
// Find where target would be inserted to keep array sorted
int searchInsert(int[] nums, int target) {
    int left = 0, right = nums.length;
    while (left < right) {
        int mid = left + (right - left) / 2;
        if (nums[mid] < target) left = mid + 1;
        else right = mid;
    }
    return left;   // leftmost valid position
}
```

### Questions

| # | Problem | LC# |
|---|---------|-----|
| E1 | Binary Search | 704 |
| E2 | Search Insert Position | 35 |
| M1 | Find First and Last Position of Element | 34 |
| M2 | Count occurrences in sorted array | — |
| M3 | Sqrt(x) (integer square root) | 69 |

---

## P2: Rotated Sorted Array

### Key Insight
> In a rotated sorted array, **one half is always normally sorted**.
> Use this to decide which half to search.

```
[4, 5, 6, 7, 0, 1, 2]
         ↑ pivot
Left half [4..7] is sorted. Right half [0..2] is sorted.
```

### Variation 1: Search in Rotated Array (LC 33)

**Problem:** A sorted array of distinct integers was rotated at some unknown pivot (e.g., `[4,5,6,7,0,1,2]`). Given a target, return its index, or -1 if not found. Must be O(log n).

**Approach (Half-is-sorted invariant):**
- **Search space:** index range `[0, n-1]`.
- **Key observation:** at any `mid`, **at least one of** `[left..mid]` or `[mid..right]` is properly sorted.
- Identify the sorted half via `nums[left] <= nums[mid]` (sorted left) else sorted right.
- If target lies in the sorted half's value range, search there; else search the other half. This eliminates half the array each iteration.
- **Trap:** use `<=` (not `<`) — when `left == mid`, the single-element half is trivially "sorted."
- Edge cases: array not rotated (still works), target equals an endpoint.
- Time: O(log n) | Space: O(1).

```java
int search(int[] nums, int target) {
    int left = 0, right = nums.length - 1;
    while (left <= right) {
        int mid = left + (right - left) / 2;
        if (nums[mid] == target) return mid;

        // Determine which half is sorted
        if (nums[left] <= nums[mid]) {    // left half is sorted
            if (nums[left] <= target && target < nums[mid])
                right = mid - 1;          // target in left half
            else
                left = mid + 1;
        } else {                          // right half is sorted
            if (nums[mid] < target && target <= nums[right])
                left = mid + 1;           // target in right half
            else
                right = mid - 1;
        }
    }
    return -1;
}
```
> FAANG trap: `nums[left] <= nums[mid]` (not `<`) handles the case when left == mid.

### Variation 2: Find Minimum in Rotated Array (LC 153)

**Problem:** A sorted array of distinct integers was rotated at some pivot. Return the minimum value in O(log n).

**Approach (Compare with right end):**
- **Search space:** index range `[0, n-1]`.
- **Invariant:** the minimum is always inside `[left, right]`.
- Compare `nums[mid]` with `nums[right]`:
  - If `nums[mid] > nums[right]`: the pivot (min) must be in the right half, exclusive of mid → `left = mid + 1`.
  - Else: mid could itself be the min, so `right = mid` (don't exclude it).
- Why compare with `right`, not `left`? Comparing with `left` fails when the array is unrotated (left half always looks "smaller" but the answer is at index 0).
- Edge cases: array unrotated → returns nums[0]; n == 1.
- Time: O(log n) | Space: O(1).

```java
int findMin(int[] nums) {
    int left = 0, right = nums.length - 1;
    while (left < right) {
        int mid = left + (right - left) / 2;
        if (nums[mid] > nums[right])
            left = mid + 1;    // min is in right half
        else
            right = mid;       // mid could be the min
    }
    return nums[left];
}
```

### Variation 3: With Duplicates (LC 81, 154)

**Problem:** Same as LC 33 / 153 but the array can contain duplicates (e.g., `[2,2,2,0,2]`). Search for target / find min.

**Approach (Shrink ambiguous ends):**
- Duplicates break the "one half is always sorted" property — when `nums[left] == nums[mid] == nums[right]`, we cannot tell which half is sorted.
- **Fix:** when this ambiguity occurs, just shrink both ends (`left++; right--`) and retry. We give up one element on each side per ambiguous step.
- Otherwise, run the standard LC 33 / 153 logic.
- Edge case: array like `[1,1,1,1]` → ambiguity at every step, degenerates to O(n).
- Time: O(log n) average, **O(n) worst** (e.g., all duplicates) | Space: O(1).

```java
// Duplicates break the "one half is always sorted" guarantee
// When nums[left] == nums[mid] == nums[right], we can't decide → shrink both ends
if (nums[left] == nums[mid] && nums[mid] == nums[right]) {
    left++; right--;
}
```

### Questions

| # | Problem | LC# |
|---|---------|-----|
| M1 | Search in Rotated Sorted Array | 33 |
| M2 | Find Minimum in Rotated Sorted Array | 153 |
| M3 | Search in Rotated Sorted Array II (duplicates) | 81 |
| M4 | Find Minimum in Rotated Sorted Array II (duplicates) | 154 |

---

## P3: Peak Finding

### Variation 1: Find Peak Element (LC 162)

**Problem:** Given an integer array where `nums[i] != nums[i+1]`, find any index `i` such that `nums[i] > nums[i-1]` and `nums[i] > nums[i+1]`. Imagine `nums[-1] = nums[n] = -∞`. Must be O(log n).

**Approach (Follow the ascending slope):**
- **Search space:** index range `[0, n-1]`.
- **Invariant:** there is always at least one peak inside `[left, right]` (because edges are -∞).
- At `mid`: if `nums[mid] < nums[mid+1]`, we're on an ascending slope → a peak must lie to the right → `left = mid + 1`. Otherwise we're descending (or at a peak) → `right = mid`.
- This works even though the array is *not* sorted — we're using local monotonicity, not order.
- Edge cases: single element (return 0); strictly increasing (last element is peak); strictly decreasing (first element).
- Time: O(log n) | Space: O(1).

```java
// Peak: element greater than both neighbors
// Key insight: if nums[mid] < nums[mid+1], peak is on the right
int findPeakElement(int[] nums) {
    int left = 0, right = nums.length - 1;
    while (left < right) {
        int mid = left + (right - left) / 2;
        if (nums[mid] < nums[mid + 1])
            left = mid + 1;    // ascending — peak is to the right
        else
            right = mid;       // descending — peak is at mid or left
    }
    return left;
}
```

### Variation 2: Mountain Array Peak (LC 852)

**Problem:** Given a "mountain" array (strictly increases then strictly decreases), return the peak index. Guaranteed there is exactly one peak.

**Approach (Same as Find Peak, simpler guarantee):**
- **Search space:** index range `[0, n-1]`.
- Since the array is mountain-shaped, the slope at `mid` uniquely tells us which side the peak is on. No multiple-peak ambiguity.
- If `arr[mid] < arr[mid+1]`: still climbing → peak is to the right (`left = mid+1`). Else: descending → peak at `mid` or left (`right = mid`).
- Edge cases: n must be ≥ 3 (problem guarantees this).
- Time: O(log n) | Space: O(1).

> Array increases then decreases — find the peak index.
```java
int peakIndexInMountainArray(int[] arr) {
    int left = 0, right = arr.length - 1;
    while (left < right) {
        int mid = left + (right - left) / 2;
        if (arr[mid] < arr[mid + 1]) left = mid + 1;
        else right = mid;
    }
    return left;
}
```

### Variation 3: Find in Mountain Array (LC 1095) — Hard

**Problem:** A mountain array `arr` and a target. Return the smallest index where `arr[i] == target`, or -1. You may only access elements via a `get(i)` API and should minimize calls (`O(log n)`).

**Approach (Three-phase BS):**
- **Phase 1:** Find the peak index using LC 852 logic → O(log n).
- **Phase 2:** Binary search the ascending half `[0, peak]` for `target` (standard ascending BS).
- **Phase 3:** If not found, binary search the descending half `[peak+1, n-1]` (flip comparisons since it's descending).
- Return the first hit (ascending half gives the smaller index automatically).
- Edge cases: target equals peak, target appears only in descending half, target absent.
- Time: O(log n) (three log-n searches) | Space: O(1).

```java
// 1. Find peak using BS
// 2. Binary search in ascending left half
// 3. If not found, binary search in descending right half
```

---

## P4: Binary Search on Answer Space — ★ FAANG Most Important Pattern

### Core Idea
> "Minimize the maximum" or "maximize the minimum" → the answer lives in a range.
> Binary search ON the answer, not on the array.

### Template
```java
int left = minPossibleAnswer;   // always valid lower bound
int right = maxPossibleAnswer;  // always valid upper bound

while (left < right) {
    int mid = left + (right - left) / 2;
    if (canAchieve(mid)) {
        right = mid;       // mid is feasible, try smaller
    } else {
        left = mid + 1;    // mid not feasible, need larger
    }
}
return left;
```
> Flip `right = mid` / `left = mid + 1` based on whether you're minimizing or maximizing.

### How to identify this pattern
- [ ] "Split into k groups and minimize the max group sum" → BS on max sum
- [ ] "Distribute M bananas to K monkeys, minimize max" → BS on max
- [ ] "At least K workers can finish in D days" → BS on D
- [ ] `canAchieve(mid)` is a simple greedy check

### Variation 1: Koko Eating Bananas (LC 875)

**Problem:** Koko has `n` piles of bananas (`piles[i]`) and `h` hours before the guards return. Each hour she chooses a pile and eats up to `k` bananas (the rest of that pile is wasted that hour). Find the minimum integer `k` (bananas per hour) such that she finishes all piles in `h` hours.

**Approach (BS on answer space — minimize speed):**
- **Search space:** `k ∈ [1, max(piles)]`. Speed > max(piles) is never better; speed 0 doesn't make sense.
- **Predicate `canFinish(k)`:** total hours = `Σ ceil(piles[i] / k)`; feasible iff hours ≤ h.
- **Monotonicity:** if speed `k` works, any speed `> k` also works → predicate is monotone false→true. BS finds the smallest true.
- Hours for pile p at speed k: `(p + k - 1) / k` (ceil division without floats).
- Edge cases: h == n → must eat the largest pile in one hour → k = max(piles). h huge → k = 1.
- Time: O(n · log(max(piles))) | Space: O(1).

```java
// Find min eating speed k such that Koko eats all in h hours
int minEatingSpeed(int[] piles, int h) {
    int left = 1, right = Arrays.stream(piles).max().getAsInt();
    while (left < right) {
        int mid = left + (right - left) / 2;
        if (canFinish(piles, mid, h)) right = mid;
        else left = mid + 1;
    }
    return left;
}
boolean canFinish(int[] piles, int speed, int h) {
    int hours = 0;
    for (int pile : piles) hours += (pile + speed - 1) / speed;  // ceil division
    return hours <= h;
}
```

### Variation 2: Split Array Largest Sum (LC 410) — FAANG Hard

**Problem:** Given an integer array `nums` and an integer `k`, split `nums` into exactly `k` non-empty contiguous subarrays. Minimize the largest subarray sum among the `k` parts.

**Approach (BS on answer space — minimize max sum):**
- **Search space:** answer `S ∈ [max(nums), sum(nums)]`. Lower bound: even single-element groups can't go below the max element. Upper bound: one group containing everything.
- **Predicate `canSplit(S)`:** greedily walk left-to-right; start a new group whenever adding the current num would exceed `S`; feasible iff groups used ≤ k.
- **Monotonicity:** if `S` works, any larger `S` works → BS for smallest feasible `S`.
- Greedy is optimal because larger groups can only reduce group count.
- Edge cases: k == 1 (answer = sum); k == n (answer = max).
- Time: O(n · log(sum)) | Space: O(1).

```java
// Split array into k parts to minimize the largest part sum
int splitArray(int[] nums, int k) {
    int left = Arrays.stream(nums).max().getAsInt();   // min possible (one element per part)
    int right = Arrays.stream(nums).sum();              // max possible (all in one part)

    while (left < right) {
        int mid = left + (right - left) / 2;
        if (canSplit(nums, k, mid)) right = mid;
        else left = mid + 1;
    }
    return left;
}
boolean canSplit(int[] nums, int k, int maxSum) {
    int parts = 1, current = 0;
    for (int num : nums) {
        if (current + num > maxSum) { parts++; current = 0; }
        current += num;
    }
    return parts <= k;
}
```

### Variation 3: Capacity to Ship Packages (LC 1011)

**Problem:** A conveyor has packages with weights `weights[i]` loaded in order. Find the **minimum ship capacity** so all packages can be shipped within `days` days. Each day load packages in given order without exceeding capacity.

**Approach (BS on answer — identical to LC 410):**
- **Search space:** capacity `C ∈ [max(weights), sum(weights)]`. Min: must fit the heaviest single package. Max: ship everything in one day.
- **Predicate `canShip(C)`:** greedy — pack consecutive items into current day until adding the next would exceed `C`, then start a new day; feasible iff days used ≤ given `days`.
- **Monotonicity:** larger capacity always works once smaller does.
- This is the same template as Split Array (LC 410) — packages must stay in given order, just like contiguous subarrays.
- Edge cases: days == 1 → C = sum; days ≥ n → C = max.
- Time: O(n · log(sum)) | Space: O(1).

```java
// Exactly same structure as Split Array Largest Sum
int shipWithinDays(int[] weights, int days) {
    int left = Arrays.stream(weights).max().getAsInt();
    int right = Arrays.stream(weights).sum();
    while (left < right) {
        int mid = left + (right - left) / 2;
        if (canShip(weights, days, mid)) right = mid;
        else left = mid + 1;
    }
    return left;
}
boolean canShip(int[] weights, int days, int cap) {
    int day = 1, load = 0;
    for (int w : weights) {
        if (load + w > cap) { day++; load = 0; }
        load += w;
    }
    return day <= days;
}
```

### Variation 4: Aggressive Cows (Classic — SPOJ / GFG)

**Problem:** Given stall positions `stalls[]` (sorted after sort) and `k` aggressive cows, place the cows so that the **minimum** distance between any two is **maximized**. Return that maximum minimum distance.

**Approach (BS on answer — maximize the minimum):**
- **Search space:** distance `d ∈ [1, stalls[n-1] - stalls[0]]`. Smallest possible gap is 1, largest is the span.
- **Predicate `canPlace(d)`:** greedy — place first cow at stalls[0], then place next cow at the first stall ≥ last placed + d; feasible iff we placed all k cows.
- **Monotonicity:** if distance `d` works, any smaller `d` works (looser constraint) → predicate is monotone true→false. We want the largest true.
- **Trap (maximization):** to avoid infinite loop when `left = mid`, use `mid = left + (right - left + 1) / 2` (upper-mid), and `right = mid - 1` on infeasible.
- Edge cases: k == 2 (answer = span); k > n (impossible, but problem usually guarantees k ≤ n).
- Time: O(n · log(span)) | Space: O(1).

```java
// Place k cows in stalls, maximize minimum distance between any two cows
// Binary search on the minimum distance
Arrays.sort(stalls);
int left = 1, right = stalls[n-1] - stalls[0];
while (left < right) {
    int mid = left + (right - left + 1) / 2;   // NOTE: +1 for maximization
    if (canPlace(stalls, k, mid)) left = mid;   // feasible → try larger
    else right = mid - 1;
}
```
> Maximization: use `left = mid + (right - left + 1) / 2` and `left = mid` when feasible.

### Variation 5: Minimize Maximum Distance to Gas Station (LC 774)

**Problem:** Sorted positions of `n` gas stations on the x-axis. You may add `k` new stations anywhere. Minimize the maximum gap between adjacent stations. Return that minimum (a real number).

**Approach (BS on real-valued answer):**
- **Search space:** continuous `d ∈ [0, maxGap]` where maxGap is the largest existing adjacent gap.
- **Predicate `canAchieve(d)`:** for every adjacent gap `g`, we need `ceil(g / d) - 1` new stations to bring all sub-gaps ≤ d; feasible iff total new stations needed ≤ k.
- **Monotonicity:** larger `d` is easier → BS for smallest feasible `d`.
- Real-valued BS — stop when `right - left < 1e-6` (or fixed ~100 iterations) since exact integer convergence isn't possible.
- Edge cases: k == 0 → answer = maxGap; k huge → answer → 0.
- Time: O(n · log((maxGap)/eps)) | Space: O(1).

### Variation 6: Minimum Days to Make m Bouquets (LC 1482)

**Problem:** `bloomDay[i]` = day flower `i` blooms. To make a bouquet you need `k` **adjacent** bloomed flowers. Return the minimum day to make `m` bouquets, or -1 if impossible.

**Approach (BS on day count):**
- **Search space:** day `d ∈ [min(bloomDay), max(bloomDay)]`.
- **Predicate `canMake(d)`:** scan array; count consecutive flowers with `bloomDay[i] ≤ d`; every time the streak hits `k`, increment bouquets and reset streak. Feasible iff bouquets ≥ m.
- **Monotonicity:** more days only adds bloomed flowers → predicate is monotone false→true. BS for smallest true.
- Feasibility check: if `m * k > n`, return -1 immediately.
- Edge cases: m == 0 (return 0 / day 0); single bouquet; all bloom same day.
- Time: O(n · log(maxDay)) | Space: O(1).

### Questions — BS on Answer

| # | Problem | LC# | Difficulty |
|---|---------|-----|------------|
| M1 | Koko Eating Bananas | 875 | M |
| M2 | Minimum Days to Make m Bouquets | 1482 | M |
| M3 | Capacity to Ship Packages Within D Days | 1011 | M |
| M4 | Find the Smallest Divisor Given a Threshold | 1283 | M |
| H1 | Split Array Largest Sum | 410 | H |
| H2 | Minimize Max Distance to Gas Station | 774 | H |
| H3 | Magnetic Force Between Two Balls | 1552 | M |
| H4 | Minimum Speed to Arrive on Time | 1870 | M |
| H5 | Maximum Running Time of N Computers | 2141 | H |

---

## P5: Advanced Binary Search

### Variation 1: Search in 2D Matrix (LC 74)

**Problem:** Given an `m × n` matrix where each row is sorted left-to-right and the first integer of each row is greater than the last integer of the previous row (i.e., row-major sorted). Find `target` in O(log(mn)).

**Approach (Flatten + BS):**
- **Search space:** virtual flat index `[0, m*n - 1]`.
- **Trick:** map flat index `i` to `(row, col) = (i / n, i % n)`. The matrix behaves exactly like a sorted 1D array.
- Standard binary search on the flat index.
- Edge cases: empty matrix, target smaller than [0][0] or larger than [m-1][n-1].
- Time: O(log(mn)) | Space: O(1).

```java
// Treat 2D matrix as a flattened sorted array
boolean searchMatrix(int[][] matrix, int target) {
    int m = matrix.length, n = matrix[0].length;
    int left = 0, right = m * n - 1;
    while (left <= right) {
        int mid = left + (right - left) / 2;
        int val = matrix[mid / n][mid % n];
        if (val == target) return true;
        else if (val < target) left = mid + 1;
        else right = mid - 1;
    }
    return false;
}
```

### Variation 2: Search in 2D Matrix II (LC 240)

**Problem:** Matrix where each **row** is sorted L→R and each **column** is sorted T→B (but **not** globally sorted — row i+1 may start lower than row i's end). Find `target`.

**Approach (Staircase / elimination from corner):**
- Start at the **top-right** corner `(0, n-1)`. From here:
  - If `matrix[r][c] == target` → found.
  - If `matrix[r][c] > target` → entire column `c` below is also too big → eliminate column (`col--`).
  - If `matrix[r][c] < target` → entire row `r` to the left is also too small → eliminate row (`row++`).
- Each step removes a full row or column → at most `m + n` steps.
- Top-left or bottom-right corners don't work — they don't give a clean decision (both moves go the same direction).
- Edge cases: target out of overall range, single row/col.
- Time: O(m + n) | Space: O(1).

```java
// Each row and column is sorted but not globally
// Start from top-right corner
boolean searchMatrix(int[][] matrix, int target) {
    int row = 0, col = matrix[0].length - 1;
    while (row < matrix.length && col >= 0) {
        if (matrix[row][col] == target) return true;
        else if (matrix[row][col] > target) col--;
        else row++;
    }
    return false;
}
```

### Variation 3: Kth Smallest Element in Sorted Matrix (LC 378)

**Problem:** Given an `n × n` matrix where rows and columns are each sorted ascending, return the kth smallest element (1-indexed).

**Approach (BS on value space + countLessEqual):**
- **Search space:** value range `[matrix[0][0], matrix[n-1][n-1]]` (NOT indices).
- **Predicate `count(v) = #elements ≤ v`** computed in O(n) using the staircase trick from LC 240 (start at bottom-left).
- **Invariant:** answer = smallest `v` such that `count(v) ≥ k`. Monotone false→true.
- The answer must be an actual matrix element (since BS converges to a value where `count` jumps from <k to ≥k, and that's exactly a matrix value).
- Alternative: min-heap O(k log k) — worse for large k.
- Edge cases: k == 1 (answer = matrix[0][0]); k == n² (answer = matrix[n-1][n-1]).
- Time: O(n · log(maxVal - minVal)) | Space: O(1).

```java
// BS on value range, not index
int kthSmallest(int[][] matrix, int k) {
    int n = matrix.length;
    int left = matrix[0][0], right = matrix[n-1][n-1];
    while (left < right) {
        int mid = left + (right - left) / 2;
        int count = countLessEqual(matrix, mid, n);
        if (count < k) left = mid + 1;
        else right = mid;
    }
    return left;
}
int countLessEqual(int[][] matrix, int val, int n) {
    int count = 0, row = n - 1, col = 0;
    while (row >= 0 && col < n) {
        if (matrix[row][col] <= val) { count += row + 1; col++; }
        else row--;
    }
    return count;
}
```

### Variation 4: Median of Two Sorted Arrays (LC 4) — FAANG Hard

**Problem:** Two sorted arrays `A` (size m) and `B` (size n). Return the median of the combined sorted array. Must run in O(log(min(m, n))).

**Approach (BS on partition of the shorter array):**
- **Key idea:** to find the median, we need to split the combined `m + n` elements into a left half (size `(m+n+1)/2`) and a right half such that every element in left ≤ every element in right.
- **Search space:** `partA ∈ [0, m]` — how many of A's elements go in the left half. Then `partB = (m+n+1)/2 - partA` (derived, not searched).
- **Invariant we want at convergence:** `A[partA-1] ≤ B[partB]` AND `B[partB-1] ≤ A[partA]`. Use `±∞` sentinels at edges.
- **BS step:** if `A[partA-1] > B[partB]` → partA too big, move right boundary left. Else partA too small, move left boundary right.
- Always BS the shorter array → ensures partB stays in `[0, n]`.
- Median: combined odd → `max(leftA, leftB)`; even → average of `max(left*)` and `min(right*)`.
- Edge cases: one array empty, all elements of one < other, m == n.
- Time: O(log(min(m, n))) | Space: O(1).

```java
// Binary search on partition point of shorter array
double findMedianSortedArrays(int[] A, int[] B) {
    if (A.length > B.length) return findMedianSortedArrays(B, A);
    int m = A.length, n = B.length;
    int left = 0, right = m;

    while (left <= right) {
        int partA = left + (right - left) / 2;
        int partB = (m + n + 1) / 2 - partA;

        int maxLeftA  = (partA == 0) ? Integer.MIN_VALUE : A[partA - 1];
        int minRightA = (partA == m) ? Integer.MAX_VALUE : A[partA];
        int maxLeftB  = (partB == 0) ? Integer.MIN_VALUE : B[partB - 1];
        int minRightB = (partB == n) ? Integer.MAX_VALUE : B[partB];

        if (maxLeftA <= minRightB && maxLeftB <= minRightA) {
            if ((m + n) % 2 == 0)
                return (Math.max(maxLeftA, maxLeftB) + Math.min(minRightA, minRightB)) / 2.0;
            else
                return Math.max(maxLeftA, maxLeftB);
        } else if (maxLeftA > minRightB) right = partA - 1;
        else left = partA + 1;
    }
    return 0;
}
```

---

## BS Pattern Recognition Table

| Problem phrase | BS type | left/right boundaries |
|---------------|---------|----------------------|
| "find in sorted array" | Classic | [0, n-1] |
| "first/last occurrence" | Classic + direction | [0, n-1] |
| "rotated sorted array" | Rotated | [0, n-1] |
| "find peak element" | Peak | [0, n-1] |
| "minimize the maximum" | BS on answer | [max_element, sum] |
| "maximize the minimum" | BS on answer | [1, max_possible] |
| "at least k elements satisfy X in D days" | BS on D | [1, max_days] |
| "kth smallest in matrix/two arrays" | BS on value | [min_val, max_val] |

---

> Back to [Master Index](../DSA_MASTER_INDEX.md)
