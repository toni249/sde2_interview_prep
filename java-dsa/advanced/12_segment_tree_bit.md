# Segment Tree & Binary Indexed Tree (BIT/Fenwick Tree)
> FAANG Level | Java | Advanced

---

## When Do You Need These?

```
Static array, range queries only        → Prefix Sum (O(1) query)
Many range updates + range queries      → Segment Tree or BIT
Point updates + range sum queries       → BIT (simpler code)
Range updates + range queries           → Segment Tree with lazy propagation
```

---

## P1: Binary Indexed Tree (BIT / Fenwick Tree)

### Core Idea
> `bit[i]` stores the sum of a range ending at index `i`.
> The range length is determined by the lowest set bit of `i`.

**What each BIT node represents:** `tree[i]` is the **partial sum** of array elements over the range `(i - lowbit(i), i]` — a range whose length equals `i & -i` (the lowest set bit of `i`). For example, `tree[12]` (binary 1100) covers indices 9..12 (length 4); `tree[8]` covers 1..8 (length 8).

**Why `i & -i`?** The two's-complement trick isolates the lowest set bit. Adding it walks "up" to the next responsible node during `update`; subtracting it strips that bit during `query`, walking through a logarithmic chain of disjoint partial sums that together cover `[1, i]`.

**Why O(log n)?** Every step of update/query toggles one bit of `i`, so the loop runs at most `log₂(n)` times.

### Template
```java
class BIT {
    int[] tree;
    int n;

    BIT(int n) {
        this.n = n;
        tree = new int[n + 1];
    }

    // Add delta to index i (1-indexed)
    void update(int i, int delta) {
        for (; i <= n; i += i & (-i))   // i & (-i) = lowest set bit
            tree[i] += delta;
    }

    // Sum of [1, i]
    int query(int i) {
        int sum = 0;
        for (; i > 0; i -= i & (-i))
            sum += tree[i];
        return sum;
    }

    // Sum of [l, r]
    int query(int l, int r) {
        return query(r) - query(l - 1);
    }
}
```

### Variation 1: Range Sum Query — Mutable (LC 307)

**Problem:** Given an integer array `nums`, support `update(index, val)` (set `nums[index] = val`) and `sumRange(left, right)` (inclusive sum), with both operations frequent.

**Approach (BIT/Fenwick):**
- Naïve: O(n) per query or O(n) per update. Prefix sums give O(1) query but O(n) update. **BIT gives O(log n) for both.**
- Store deltas: `update(i, val - oldVal)` propagates the change up the BIT chain by `+= lowbit(i)`.
- `sumRange(l, r) = prefix(r+1) - prefix(l)` (BIT is 1-indexed internally).
- Cache `nums[i]` separately so we know the old value to compute the delta.
- Time **O(log n)** per op, space **O(n)**. Segment tree works equally well; BIT is shorter.

```java
class NumArray {
    BIT bit;
    int[] nums;

    NumArray(int[] nums) {
        this.nums = nums.clone();
        bit = new BIT(nums.length);
        for (int i = 0; i < nums.length; i++) bit.update(i + 1, nums[i]);
    }

    void update(int index, int val) {
        bit.update(index + 1, val - nums[index]);
        nums[index] = val;
    }

    int sumRange(int left, int right) {
        return bit.query(left + 1, right + 1);
    }
}
```

### Variation 2: Count of Smaller Numbers After Self (LC 315)

**Problem:** Given an integer array `nums`, return `counts` where `counts[i]` = the number of elements **to the right of `nums[i]`** that are smaller than `nums[i]`.

**Approach (BIT over value ranks, sweep right-to-left):**
- Trick: process **right-to-left**. When we reach index `i`, the BIT contains the **multiset of values already seen on the right**. The answer for `i` = "how many of those are < nums[i]" = `bit.query(rank(nums[i]) - 1)`. Then insert `nums[i]`.
- **Coordinate compression** (rank): values may be huge / negative, but only their relative order matters. Sort uniques and map each value → 1-indexed rank, so the BIT is sized to # of distinct values.
- Why BIT? Each step needs (a) a count of "how many values so far have rank < r" and (b) "add 1 at rank r" — both **O(log V)** operations. Sorting + BIT total **O(n log n)**.
- Alternative: merge sort with index tracking, same complexity.

```java
// Coordinate compress values, then BIT for counting
int[] countSmaller(int[] nums) {
    int n = nums.length;
    int[] result = new int[n];
    int[] sorted = nums.clone();
    Arrays.sort(sorted);
    // Map each num to its rank (1-indexed)
    Map<Integer, Integer> rank = new HashMap<>();
    int r = 1;
    for (int num : sorted) rank.putIfAbsent(num, r++);

    BIT bit = new BIT(r);
    for (int i = n - 1; i >= 0; i--) {   // right to left
        int rankVal = rank.get(nums[i]);
        result[i] = bit.query(rankVal - 1);   // count numbers < nums[i]
        bit.update(rankVal, 1);
    }
    return result;
}
```

---

## P2: Segment Tree

### When BIT Isn't Enough
> BIT handles only **commutative, invertible** operations (sum, xor).
> Segment tree handles: **min, max, gcd, range updates**.

**What each segment-tree node stores:** the aggregate value (sum / min / max / gcd / ...) over a **contiguous subarray** `[start, end]`. The root covers `[0, n-1]`; each internal node has two children that split its range exactly in half. Leaves correspond to individual array indices. Array layout uses `tree[2*node+1]` (left child) and `tree[2*node+2]` (right child) at indices 0..4n-1.

**Why O(log n)?**
- `update`: only nodes on the path from the leaf to the root contain that index → O(log n) nodes touched.
- `query(l, r)`: at each level the range can be decomposed into at most 2 fully-covered nodes plus 2 partial nodes that recurse — bounded recursion depth = tree height = log n.

### Template — Point Update, Range Query (Sum)
```java
class SegmentTree {
    int[] tree;
    int n;

    SegmentTree(int[] arr) {
        n = arr.length;
        tree = new int[4 * n];
        build(arr, 0, 0, n - 1);
    }

    void build(int[] arr, int node, int start, int end) {
        if (start == end) {
            tree[node] = arr[start];
        } else {
            int mid = (start + end) / 2;
            build(arr, 2*node+1, start, mid);
            build(arr, 2*node+2, mid+1, end);
            tree[node] = tree[2*node+1] + tree[2*node+2];
        }
    }

    void update(int node, int start, int end, int idx, int val) {
        if (start == end) {
            tree[node] = val;
        } else {
            int mid = (start + end) / 2;
            if (idx <= mid) update(2*node+1, start, mid, idx, val);
            else            update(2*node+2, mid+1, end, idx, val);
            tree[node] = tree[2*node+1] + tree[2*node+2];
        }
    }

    int query(int node, int start, int end, int l, int r) {
        if (r < start || end < l) return 0;          // out of range
        if (l <= start && end <= r) return tree[node]; // fully within range
        int mid = (start + end) / 2;
        return query(2*node+1, start, mid, l, r) +
               query(2*node+2, mid+1, end, l, r);
    }

    // Public API
    void update(int idx, int val) { update(0, 0, n-1, idx, val); }
    int query(int l, int r)       { return query(0, 0, n-1, l, r); }
}
```

### Lazy Propagation — Range Update + Range Query

**Problem context:** Support "add `val` to every element in `[l, r]`" **and** "sum over `[l, r]`" — both in O(log n). Without lazy propagation, a range update would touch O(n) leaves.

**Approach (Lazy tags):**
- **What each node stores:** `tree[node]` = current aggregate of the range (already reflecting prior updates that were *fully applied* at or above this node), plus `lazy[node]` = a pending per-element delta that **applies to this node's entire subtree but hasn't been pushed down yet**.
- **Update**: if the node's range is fully inside `[l, r]`, apply the delta to `tree[node]` (multiplied by range length) and **store** it in `lazy[node]` — no need to recurse. Otherwise, **push down** the existing lazy tag first (so children see the right pre-update state), recurse into both children, then recombine.
- **Query**: same shape — push down lazy before splitting recursion, otherwise children would return stale partial sums.
- This keeps every op **O(log n)** even for range updates because we never recurse below a fully-covered node.
- Common pitfall: forgetting to push down before recursing — leads to phantom updates that never reach leaves.

```java
// When you need to update a range [l,r] and query range sums
// Lazy tag: pending update stored at node, applied only when needed
class LazySegTree {
    long[] tree, lazy;
    int n;

    LazySegTree(int n) {
        this.n = n;
        tree = new long[4 * n];
        lazy = new long[4 * n];
    }

    void pushDown(int node, int start, int end) {
        if (lazy[node] != 0) {
            int mid = (start + end) / 2;
            tree[2*node+1] += lazy[node] * (mid - start + 1);
            tree[2*node+2] += lazy[node] * (end - mid);
            lazy[2*node+1] += lazy[node];
            lazy[2*node+2] += lazy[node];
            lazy[node] = 0;
        }
    }

    void update(int node, int start, int end, int l, int r, long val) {
        if (r < start || end < l) return;
        if (l <= start && end <= r) {
            tree[node] += val * (end - start + 1);
            lazy[node] += val;
            return;
        }
        pushDown(node, start, end);
        int mid = (start + end) / 2;
        update(2*node+1, start, mid, l, r, val);
        update(2*node+2, mid+1, end, l, r, val);
        tree[node] = tree[2*node+1] + tree[2*node+2];
    }

    long query(int node, int start, int end, int l, int r) {
        if (r < start || end < l) return 0;
        if (l <= start && end <= r) return tree[node];
        pushDown(node, start, end);
        int mid = (start + end) / 2;
        return query(2*node+1, start, mid, l, r) +
               query(2*node+2, mid+1, end, l, r);
    }
}
```

---

## Questions

| Problem | Approach | LC# | Difficulty |
|---------|----------|-----|------------|
| Range Sum Query — Mutable | BIT or Seg Tree | 307 | M |
| Count of Smaller Numbers After Self | BIT + coord compress | 315 | H |
| Count of Range Sum | Seg Tree / Merge Sort | 327 | H |
| Falling Squares | Coord compress + Seg Tree | 699 | H |
| My Calendar III | Seg Tree lazy | 732 | H |

---

> Back to [Master Index](../DSA_MASTER_INDEX.md)
