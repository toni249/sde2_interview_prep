# Heaps / Priority Queue — Pattern Deep Dive
> FAANG Level | Java | Easy → Hard

---

## Pattern Map

```
HEAPS
├── P1: Top K Elements         → k largest, k most frequent, k closest
├── P2: Kth Largest/Smallest   → stream of numbers, in array
├── P3: Merge K Sorted         → k sorted lists/arrays
├── P4: Two Heaps              → median of stream, sliding window median
└── P5: Modified Dijkstra      → graph shortest paths (see Graphs)
```

---

## Java Heap Basics

```java
// Min-Heap (default in Java)
PriorityQueue<Integer> minHeap = new PriorityQueue<>();

// Max-Heap
PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Collections.reverseOrder());

// Custom comparator
PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> a[0] - b[0]);  // sort by first element

// Operations — all O(log n)
pq.offer(x);   // add
pq.poll();     // remove and return min/max
pq.peek();     // view min/max without removing
```

---

## P1: Top K Elements

### Variation 1: Kth Largest Element in Array (LC 215)

**Problem:** Given an unsorted integer array `nums` and integer `k`, return the **kth largest** element (in sorted order, not the kth distinct element).

**Approach (Min-Heap of size k):**
- Greedy invariant: at any time, keep the **k largest values seen so far** in a **min-heap** of fixed size k. The root of that heap is automatically the kth largest.
- Why min-heap (not max)? We want to discard the smallest of our "top k" cheaply — that's the root of a min-heap. A max-heap would force us to scan to find the smallest.
- Steps: offer every element; if heap size exceeds k, poll (removes current smallest of the top group).
- Edge cases: k == nums.length → return min of array; duplicates are fine (sorted order, not distinct).
- Time **O(n log k)**, space **O(k)**. Alternative QuickSelect is **O(n)** average / **O(n²)** worst.

```java
// Min-heap of size k — maintain only k largest
int findKthLargest(int[] nums, int k) {
    PriorityQueue<Integer> minHeap = new PriorityQueue<>();
    for (int num : nums) {
        minHeap.offer(num);
        if (minHeap.size() > k) minHeap.poll();   // remove smallest
    }
    return minHeap.peek();   // kth largest is the smallest in heap of size k
}
// Time: O(n log k) | Space: O(k)
// Alternative: QuickSelect O(n) average — see below
```

### Why min-heap for k LARGEST?
```
If we keep a min-heap of size k:
- heap contains the k LARGEST elements seen so far
- The minimum of those k elements = kth largest
```

### Variation 2: Top K Frequent Elements (LC 347)

**Problem:** Given an integer array `nums` and integer `k`, return the **k most frequent** elements in any order.

**Approach (HashMap + Min-Heap of size k):**
- Two phases: (1) count frequencies in a `HashMap` — O(n). (2) Keep a min-heap of size k **ordered by frequency** so the root is the smallest of the "current top k" — easy to evict.
- Greedy choice: each step, if a new element has higher frequency than the heap's min, the old min is no longer in the top k → evict it.
- Edge cases: k == number of distinct elements; ties in frequency (any order acceptable).
- Time **O(n log k)**, space **O(n)** for map + **O(k)** heap. Alternative **bucket sort** gives **O(n)**: `buckets[freq] = list of values` — walk from high freq down.

```java
List<Integer> topKFrequent(int[] nums, int k) {
    Map<Integer, Integer> freq = new HashMap<>();
    for (int num : nums) freq.merge(num, 1, Integer::sum);

    PriorityQueue<Integer> minHeap = new PriorityQueue<>((a, b) -> freq.get(a) - freq.get(b));
    for (int num : freq.keySet()) {
        minHeap.offer(num);
        if (minHeap.size() > k) minHeap.poll();
    }
    return new ArrayList<>(minHeap);
}
// Alternative: Bucket Sort O(n) — bucket[freq] = list of numbers with that freq
```

### Variation 3: K Closest Points to Origin (LC 973)

**Problem:** Given an array of 2D points and integer `k`, return the `k` points closest to the origin (Euclidean distance, any order).

**Approach (Max-Heap of size k):**
- Mirror image of "k largest": for **k closest**, we keep a **max-heap** of size k ordered by squared distance. Root = current farthest of the closest k — easiest to evict.
- Skip `sqrt` — squared distance preserves ordering and avoids floating-point.
- Time **O(n log k)**, space **O(k)**. Alternative QuickSelect on distances → **O(n)** average.

```java
int[][] kClosest(int[][] points, int k) {
    // Max-heap of size k (remove farthest when overflow)
    PriorityQueue<int[]> maxHeap = new PriorityQueue<>(
        (a, b) -> (b[0]*b[0] + b[1]*b[1]) - (a[0]*a[0] + a[1]*a[1]));
    for (int[] p : points) {
        maxHeap.offer(p);
        if (maxHeap.size() > k) maxHeap.poll();
    }
    return maxHeap.toArray(new int[k][]);
}
```

### Variation 4: Find K Closest Elements (LC 658)

**Problem:** Given a sorted integer array `arr`, integers `k` and `x`, return the **k closest integers to `x`** in the array, sorted ascending. Tie-break: prefer the smaller value.

**Approach (Max-Heap by distance, with tie-break):**
- Custom comparator: order by `|a-x|` descending, then by value descending. Root = the worst candidate among the current best k — easy to evict.
- After collecting k, sort the heap contents ascending for output.
- Edge case: x outside `arr` range — answer is a contiguous prefix/suffix.
- Time **O(n log k)**, space **O(k)**. Optimal is **two-pointer / binary search** on the sorted array → **O(log n + k)**.

```java
// Use custom comparator: sort by |x - mid|, then by value
List<Integer> findClosestElements(int[] arr, int k, int x) {
    PriorityQueue<Integer> maxHeap = new PriorityQueue<>(
        (a, b) -> Math.abs(a - x) != Math.abs(b - x)
                  ? Math.abs(b - x) - Math.abs(a - x)
                  : b - a);
    for (int num : arr) {
        maxHeap.offer(num);
        if (maxHeap.size() > k) maxHeap.poll();
    }
    List<Integer> result = new ArrayList<>(maxHeap);
    Collections.sort(result);
    return result;
}
```

### Variation 5: Kth Largest Element in a Stream (LC 703)

**Problem:** Design `KthLargest(k, initialNums)` and `add(val)` that returns the **kth largest** element after each addition (across all elements seen so far, including duplicates).

**Approach (Min-heap of size k, persistent):**
- Same idea as LC 215 but the heap **persists across add() calls**. The root is always the current kth largest.
- On `add`: offer the new value; if heap grows past k, poll the smallest. The new root is the answer.
- Time **O(log k)** per add, space **O(k)**.

```java
class KthLargest {
    PriorityQueue<Integer> minHeap = new PriorityQueue<>();
    int k;
    KthLargest(int k, int[] nums) {
        this.k = k;
        for (int n : nums) add(n);
    }
    int add(int val) {
        minHeap.offer(val);
        if (minHeap.size() > k) minHeap.poll();
        return minHeap.peek();
    }
}
```

### QuickSelect — O(n) Average Kth Largest
```java
int findKthLargest(int[] nums, int k) {
    int target = nums.length - k;   // kth largest = target-th smallest (0-indexed)
    return quickSelect(nums, 0, nums.length - 1, target);
}
int quickSelect(int[] nums, int left, int right, int target) {
    int pivot = nums[right], p = left;
    for (int i = left; i < right; i++) {
        if (nums[i] <= pivot) swap(nums, i, p++);
    }
    swap(nums, p, right);
    if (p == target) return nums[p];
    if (p < target)  return quickSelect(nums, p + 1, right, target);
    else             return quickSelect(nums, left, p - 1, target);
}
```

### Questions — Top K

| # | Problem | LC# | Difficulty |
|---|---------|-----|------------|
| M1 | Kth Largest Element in an Array | 215 | M |
| M2 | Top K Frequent Elements | 347 | M |
| M3 | K Closest Points to Origin | 973 | M |
| M4 | Find K Closest Elements | 658 | M |
| M5 | Kth Largest Element in a Stream | 703 | E |
| M6 | Sort Characters By Frequency | 451 | M |
| H1 | Top K Frequent Words | 692 | M |

---

## P2: Kth Smallest — Special Cases

### Kth Smallest in Sorted Matrix (LC 378)

**Problem:** Given an `n x n` matrix where each row and each column is sorted ascending, return the **kth smallest** element (in sorted order across the whole matrix).

**Approach (Min-Heap, multi-source frontier):**
- The smallest unvisited element is always at the "frontier" — for each row, the leftmost unprocessed cell. Seed a min-heap with the first column.
- Each poll yields the global minimum; push that cell's right neighbor (same row, next column) — preserves the invariant that the heap contains the row-frontier.
- After k polls, the last value polled is the kth smallest.
- Time **O(k log n)** (heap of size ≤ n). Alternative: **binary search on value space** → **O(n log(max-min))**, often faster when k is large.

```java
// Min-heap starting from top-left
int kthSmallest(int[][] matrix, int k) {
    int n = matrix.length;
    PriorityQueue<int[]> minHeap = new PriorityQueue<>((a, b) -> a[0] - b[0]);
    for (int i = 0; i < n; i++) minHeap.offer(new int[]{matrix[i][0], i, 0});

    int result = 0;
    while (k-- > 0) {
        int[] curr = minHeap.poll();
        result = curr[0];
        int row = curr[1], col = curr[2];
        if (col + 1 < n) minHeap.offer(new int[]{matrix[row][col+1], row, col+1});
    }
    return result;
}
```

### Kth Smallest in Multiplication Table (LC 668)
> Binary search on the answer value.

---

## P3: Merge K Sorted

### Merge K Sorted Lists (LC 23)

**Problem:** Given an array of `k` sorted linked lists, merge them into one sorted linked list and return its head.

**Approach (Min-Heap of k frontiers):**
- At any moment the global minimum of all remaining nodes is the head of one of the lists — so seed a min-heap with each list's head.
- Greedy step: poll the heap to get the next smallest, append to the result, push that node's `.next` (if any). The heap always contains ≤ k nodes — one frontier per list.
- Edge cases: some lists null; the entire array empty.
- Time **O(N log k)** where N is total node count; space **O(k)** for the heap. Alternative: **divide & conquer** pairwise merge — same asymptotic time, lower constants, no heap.

```java
// Already covered in Linked List — min-heap pulling from k lists simultaneously
ListNode mergeKLists(ListNode[] lists) {
    PriorityQueue<ListNode> pq = new PriorityQueue<>((a, b) -> a.val - b.val);
    for (ListNode node : lists) if (node != null) pq.offer(node);
    ListNode dummy = new ListNode(0), curr = dummy;
    while (!pq.isEmpty()) {
        ListNode node = pq.poll();
        curr.next = node;
        curr = curr.next;
        if (node.next != null) pq.offer(node.next);
    }
    return dummy.next;
}
```

### Merge K Sorted Arrays
```java
// Same idea: push {value, arrayIndex, elementIndex} into min-heap
PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> a[0] - b[0]);
for (int i = 0; i < k; i++) {
    if (arrays[i].length > 0) pq.offer(new int[]{arrays[i][0], i, 0});
}
// Pull and push next from same array
```

### Smallest Range Covering Elements from K Lists (LC 632)

**Problem:** Given `k` sorted integer lists, find the **smallest range `[a, b]`** that includes at least one number from each list. Tie-break: smaller `a` wins.

**Approach (Min-Heap + running max):**
- Maintain a sliding "selection" of exactly one element per list (a "frontier"). The range = `[heap.min, runningMax]`.
- Each step, poll the min and advance **that list's** pointer by one (this is the only way to shrink the range, since any other move keeps the min unchanged but might increase max). Push the new value, update `runningMax`.
- Stop when any list is exhausted — we can no longer cover all k.
- Why min-heap (not max)? We must always know which element is the **current min** to advance the right list.
- Time **O(N log k)** where N is total element count; space **O(k)**.

```java
// Maintain a window [min, max] across k lists simultaneously
// Advance the list with the minimum element to shrink the range
int[] smallestRange(List<List<Integer>> nums) {
    PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> a[0] - b[0]);
    int max = Integer.MIN_VALUE;
    for (int i = 0; i < nums.size(); i++) {
        pq.offer(new int[]{nums.get(i).get(0), i, 0});
        max = Math.max(max, nums.get(i).get(0));
    }
    int[] result = {0, Integer.MAX_VALUE};
    while (pq.size() == nums.size()) {
        int[] curr = pq.poll();
        int min = curr[0];
        if (max - min < result[1] - result[0]) result = new int[]{min, max};
        int nextIdx = curr[2] + 1;
        if (nextIdx < nums.get(curr[1]).size()) {
            int nextVal = nums.get(curr[1]).get(nextIdx);
            pq.offer(new int[]{nextVal, curr[1], nextIdx});
            max = Math.max(max, nextVal);
        }
    }
    return result;
}
```

### Questions — Merge K

| # | Problem | LC# | Difficulty |
|---|---------|-----|------------|
| H1 | Merge K Sorted Lists | 23 | H |
| H2 | Smallest Range Covering Elements From K Lists | 632 | H |
| M1 | Kth Smallest Element in a Sorted Matrix | 378 | M |

---

## P4: Two Heaps — Median

### Median of Data Stream (LC 295) — FAANG Classic

**Problem:** Design a data structure that supports two ops on a stream of integers: `addNum(x)` adds a number, `findMedian()` returns the current median (mean of two middles if even count).

**Approach (Two Heaps — max-heap of lower half + min-heap of upper half):**
- Key insight: median only requires knowing the **middle one or two values**, not the full sorted order. Split the stream into halves at the median:
  - **`maxHeap`** stores the **lower half** — root = largest of lower half.
  - **`minHeap`** stores the **upper half** — root = smallest of upper half.
- Invariant: every element of `maxHeap` ≤ every element of `minHeap`; sizes differ by at most 1. Then median = either root, or average of both roots.
- `addNum` trick to enforce ordering: push to `maxHeap`, then move its top to `minHeap` (ensures the order invariant); rebalance sizes if needed.
- Time: **O(log n)** per add, **O(1)** per median. Space **O(n)**.

```java
// Maintain two heaps:
// maxHeap: lower half of numbers (max at top)
// minHeap: upper half of numbers (min at top)
// Invariant: |maxHeap.size - minHeap.size| <= 1

class MedianFinder {
    PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Collections.reverseOrder());
    PriorityQueue<Integer> minHeap = new PriorityQueue<>();

    public void addNum(int num) {
        maxHeap.offer(num);
        minHeap.offer(maxHeap.poll());           // ensure maxHeap's max <= minHeap's min
        if (minHeap.size() > maxHeap.size()) {
            maxHeap.offer(minHeap.poll());       // balance sizes
        }
    }

    public double findMedian() {
        if (maxHeap.size() > minHeap.size()) return maxHeap.peek();
        return (maxHeap.peek() + minHeap.peek()) / 2.0;
    }
}
```

### Sliding Window Median (LC 480)

**Problem:** Given an array `nums` and window size `k`, return the median of each k-sized sliding window as it moves left to right.

**Approach (Two TreeMaps OR two heaps with lazy deletion):**
- Same two-half invariant as LC 295, but each window slide both **adds** and **removes** an arbitrary element — heaps don't support O(log n) arbitrary removal directly.
- Two options:
  - **Lazy deletion**: keep a "to-delete" hashmap; whenever a heap's root is marked-deleted, pop it. Amortized O(log k).
  - **TreeMap** (used here): supports `firstKey()` (peek), and `remove()` of any key in O(log k). Track sizes separately because keys can have multiplicity.
- Watch out for integer overflow when averaging two ints near `Integer.MAX_VALUE` — cast to `double` first.
- Time **O(n log k)**, space **O(k)**.

```java
// Two heaps + remove arbitrary elements (use lazy deletion or TreeMap)
// TreeMap approach: maintain lo (max of lower half) and hi (min of upper half)
class SlidingWindowMedian {
    TreeMap<Integer, Integer> lo = new TreeMap<>(Collections.reverseOrder()); // max-map
    TreeMap<Integer, Integer> hi = new TreeMap<>();                           // min-map
    int loSize = 0, hiSize = 0;

    void add(int num) {
        lo.merge(num, 1, Integer::sum); loSize++;
        // Balance: move lo's max to hi
        hi.merge(move(lo), 1, Integer::sum); loSize--; hiSize++;
        // Ensure lo.size >= hi.size
        if (hiSize > loSize) {
            lo.merge(move(hi), 1, Integer::sum); hiSize--; loSize++;
        }
    }
    void remove(int num) {
        if (lo.containsKey(num)) { decrement(lo, num); loSize--; }
        else                     { decrement(hi, num); hiSize--; }
        // Rebalance
    }
    int move(TreeMap<Integer, Integer> map) {
        int key = map.firstKey();
        decrement(map, key);
        return key;
    }
    void decrement(TreeMap<Integer, Integer> map, int key) {
        map.merge(key, -1, Integer::sum);
        if (map.get(key) == 0) map.remove(key);
    }
    double getMedian() {
        return loSize > hiSize ? lo.firstKey()
                               : (lo.firstKey() + hi.firstKey()) / 2.0;
    }
}
```

### Questions — Two Heaps

| # | Problem | LC# | Difficulty |
|---|---------|-----|------------|
| H1 | Find Median from Data Stream | 295 | H |
| H2 | Sliding Window Median | 480 | H |
| M1 | IPO (maximize capital) | 502 | H |

---

## P5: Task Scheduling with Heaps

### Task Scheduler (LC 621)

**Problem:** Given an array of CPU task labels (chars A–Z) and an integer `n`, the same task must be spaced **at least n units** apart. Return the **minimum total time** (including idle slots) to finish all tasks.

**Approach (Greedy formula based on max frequency):**
- The bottleneck is the **most frequent** task: if it appears `maxFreq` times, you must lay out `maxFreq - 1` "cycles" of `n + 1` slots (for cooldown) plus one final placement.
- If multiple tasks tie at `maxFreq`, they all occupy the last partial cycle → add `maxCount` to the formula.
- Final answer: `max(formula, tasks.length)` — if there are many low-frequency tasks, no idling is needed and time is just `tasks.length`.
- A heap-based simulation also works (max-heap by frequency, process `n+1` at a time), but the closed-form is O(N + 26).
- Time **O(N)**, space **O(1)**.

```java
// Maximum frequency task determines the minimum idle slots needed
int leastInterval(char[] tasks, int n) {
    int[] freq = new int[26];
    for (char c : tasks) freq[c - 'A']++;
    int maxFreq = 0;
    for (int f : freq) maxFreq = Math.max(maxFreq, f);
    int maxCount = 0;
    for (int f : freq) if (f == maxFreq) maxCount++;
    // Cycles of (n+1) slots, last cycle may be shorter
    int result = (maxFreq - 1) * (n + 1) + maxCount;
    return Math.max(result, tasks.length);
}
```

### Reorganize String (LC 767)

**Problem:** Given a string `s`, rearrange its characters so that **no two adjacent characters are the same**. Return any valid rearrangement, or `""` if impossible.

**Approach (Max-Heap by frequency, alternate placement):**
- Greedy: at each step, the safest character to place next is the **currently most frequent** one — but we can't place the *same* char twice in a row. So pull the **top two** from the max-heap, place both, decrement their counts, push back if > 0.
- Why max-heap? Highest-frequency character is the constraint — if we delay it too long, we get stuck with consecutive duplicates at the end.
- Impossibility check: if any char's frequency > `(n + 1) / 2`, no arrangement exists; this naturally surfaces when the heap has size 1 with count > 1 at the end.
- Time **O(N log 26) = O(N)**, space **O(26)**.

```java
// Greedily place most frequent char, alternate using max-heap
String reorganizeString(String s) {
    int[] freq = new int[26];
    for (char c : s.toCharArray()) freq[c - 'a']++;
    PriorityQueue<int[]> maxHeap = new PriorityQueue<>((a, b) -> b[1] - a[1]);
    for (int i = 0; i < 26; i++) if (freq[i] > 0) maxHeap.offer(new int[]{i, freq[i]});
    StringBuilder sb = new StringBuilder();
    while (maxHeap.size() > 1) {
        int[] first = maxHeap.poll(), second = maxHeap.poll();
        sb.append((char)('a' + first[0]));
        sb.append((char)('a' + second[0]));
        if (--first[1] > 0) maxHeap.offer(first);
        if (--second[1] > 0) maxHeap.offer(second);
    }
    if (!maxHeap.isEmpty()) {
        if (maxHeap.peek()[1] > 1) return "";
        sb.append((char)('a' + maxHeap.poll()[0]));
    }
    return sb.toString();
}
```

---

## Heap Recognition Triggers

```
Problem says...                    → Think...
─────────────────────────────────────────────────────
"k largest / k smallest"           → Heap of size k (opposite type)
"kth element"                      → Heap of size k or QuickSelect
"merge k sorted"                   → Min-heap pulling from k sources
"median of stream"                 → Two heaps (max + min)
"schedule tasks with cooldown"     → Max-heap (greedy frequency)
"running median"                   → Two heaps
"smallest range from k lists"      → Min-heap + track max
```

---

> Back to [Master Index](../DSA_MASTER_INDEX.md)
