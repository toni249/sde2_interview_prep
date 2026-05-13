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

### Variation 1: Range Sum Query Mutable (LC 307)
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
