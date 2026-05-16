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

### The HashMap Trick (★ Very Common)
> "Count subarrays with sum = k"

Key insight: if `prefix[j] - prefix[i] = k`, then subarray `[i+1..j]` has sum k.
So we need `prefix[i] = prefix[j] - k`. Count how many such `prefix[i]` exist using a map.

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

Use prefix sum + hashmap when the array can contain **negative numbers**.

Key idea: if current prefix is `running`, we need an earlier prefix `running - k`.
To maximize length, store the **first index** where each prefix sum appeared.

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

#### Flavor C: Dutch National Flag (3-way partition)
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

**Three Sum** = sort + fix one element + two pointers on the rest
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

**Trapping Rain Water** = two pointers with left/right max tracking
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

**Minimum Window Substring** (Classic Hard)
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

### The Core Idea
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

**Maximum Product Subarray:**
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

**Maximum Circular Subarray Sum:**
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

### Rotated Sorted Array
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

### Merge Intervals Template
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

```java
// Two Sum
Map<Integer, Integer> map = new HashMap<>();
for (int i = 0; i < n; i++) {
    if (map.containsKey(target - A[i])) return new int[]{map.get(target-A[i]), i};
    map.put(A[i], i);
}
```

### First Missing Positive (No extra space)
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

### Next Greater Element Template
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

### Largest Rectangle in Histogram
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

**Spiral Order:**
```java
int top=0, bottom=m-1, left=0, right=n-1;
while (top<=bottom && left<=right) {
    for(int i=left; i<=right; i++)   result.add(matrix[top][i]);   top++;
    for(int i=top; i<=bottom; i++)   result.add(matrix[i][right]); right--;
    if(top<=bottom) for(int i=right; i>=left; i--) result.add(matrix[bottom][i]); bottom--;
    if(left<=right) for(int i=bottom; i>=top; i--) result.add(matrix[i][left]);   left++;
}
```

**Rotate 90° Clockwise:**
```
Step 1: Transpose (swap matrix[i][j] with matrix[j][i])
Step 2: Reverse each row
```

**Set Matrix Zeros:**
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
