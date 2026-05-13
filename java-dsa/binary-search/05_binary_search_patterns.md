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

### Variation 1: Standard Search
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

### Variation 2: First and Last Occurrence
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

### Variation 4: Aggressive Cows (Classic — Not on LC directly)
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
### Variation 6: Minimum Days to Make m Bouquets (LC 1482)

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
