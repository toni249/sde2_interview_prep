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

### Variation 1: K Largest Elements (LC 215)
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

### Variation 2: K Most Frequent Elements (LC 347)
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

### Variation 4: K Closest Numbers in Array (LC 658)
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
