# Linked List — Pattern Deep Dive
> FAANG Level | Java | Easy → Hard

---

## Pattern Map

```
LINKED LIST
├── P1: Fast & Slow Pointers  → cycle, middle, Nth from end, palindrome
├── P2: Reversal              → reverse list, reverse k-groups, palindrome check
├── P3: Merge & Sort          → merge two sorted, merge k sorted, sort list
└── P4: In-place Tricks       → deep copy, flatten, LRU cache, reorder list
```

---

## P1: Fast & Slow Pointers (Floyd's Algorithm)

### Core Idea
> Two pointers moving at different speeds. Fast moves 2 steps, slow moves 1.

### Variation 1: Detect Cycle (LC 141)

**Problem:** Given the head of a singly linked list, return `true` if the list contains a cycle (some node's `next` points back to a previously visited node), else `false`.

**Approach (Floyd's Tortoise & Hare):**
- Use two pointers: slow advances 1 step, fast advances 2 steps.
- If there's a cycle, the gap between them shrinks by 1 each iteration, so they're guaranteed to meet inside the loop.
- If no cycle, fast hits `null` first → return false.
- Edge cases: empty list, single node with no self-loop, two nodes.
- Time O(n), Space O(1). Alternative: `HashSet<ListNode>` for O(n) extra space.

```java
boolean hasCycle(ListNode head) {
    ListNode slow = head, fast = head;
    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
        if (slow == fast) return true;
    }
    return false;
}
```

### Variation 2: Find Cycle Entry Point (LC 142)

**Problem:** If the linked list has a cycle, return the node where the cycle begins; otherwise return `null`.

**Approach (Floyd's two-phase):**
- Phase 1: detect cycle exactly like LC 141 — slow/fast meet inside the loop.
- Phase 2: reset one pointer to `head`, move both 1 step at a time. They meet at the cycle entry.
- Math justification (see derivation block below): distance from head to entry = distance from meeting point to entry (mod cycle length).
- Edge cases: no cycle (return null), cycle starts at head.
- Time O(n), Space O(1).

```
After slow == fast:
- Reset one pointer to head.
- Move both 1 step at a time.
- They meet at the cycle start.

WHY IT WORKS:
If cycle starts at node k, and loop length is L:
slow traveled: k + m
fast traveled: k + m + n*L (extra loops)
fast = 2×slow → k + m + n*L = 2(k+m) → k = n*L - m
So distance from head to cycle start = distance from meeting point to cycle start.
```
```java
ListNode detectCycle(ListNode head) {
    ListNode slow = head, fast = head;
    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
        if (slow == fast) {
            slow = head;
            while (slow != fast) { slow = slow.next; fast = fast.next; }
            return slow;
        }
    }
    return null;
}
```

### Variation 3: Middle of Linked List (LC 876)

**Problem:** Return the middle node of a singly linked list. If the list has an even number of nodes, return the **second** middle.

**Approach (Fast & Slow):**
- Slow moves 1 step, fast moves 2 steps. When fast reaches the end, slow is exactly at the middle.
- For even length, fast becomes `null` and slow lands on the second middle; to get first middle instead, check `fast.next.next != null` in the loop condition.
- Edge cases: empty list (return null), single node (slow == head).
- Time O(n), Space O(1). Alternative: count length, then walk n/2 steps (two passes).

```java
// When fast reaches end, slow is at middle
ListNode middleNode(ListNode head) {
    ListNode slow = head, fast = head;
    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
    }
    return slow;  // for even length, returns second middle
}
```

### Variation 4: Nth Node From End (LC 19)

**Problem:** Remove the `n`-th node from the end of the list and return the new head. `n` is guaranteed valid (1 ≤ n ≤ length).

**Approach (Two-pointer gap):**
- Use a **dummy node** before head — this elegantly handles the case where the head itself is removed.
- Advance `fast` by `n + 1` steps so `slow` will land on the node just **before** the target.
- Then advance both 1 step until `fast == null`. Now `slow.next` is the node to remove.
- Single pass, no length pre-computation needed.
- Edge cases: removing head (dummy handles it), single-node list with n=1.
- Time O(n), Space O(1).

```java
// Move fast n steps ahead, then both together until fast reaches end
ListNode removeNthFromEnd(ListNode head, int n) {
    ListNode dummy = new ListNode(0);
    dummy.next = head;
    ListNode fast = dummy, slow = dummy;

    for (int i = 0; i <= n; i++) fast = fast.next;  // n+1 ahead (to get prev of target)

    while (fast != null) {
        slow = slow.next;
        fast = fast.next;
    }
    slow.next = slow.next.next;   // remove Nth from end
    return dummy.next;
}
```

### Variation 5: Check Palindrome Linked List (LC 234)

**Problem:** Determine whether a singly linked list reads the same forward and backward. Solve in O(n) time and **O(1) space**.

**Approach (Mid + Reverse + Compare):**
- Find middle via slow/fast. Reverse the second half **in place**.
- Walk both halves simultaneously, comparing values; mismatch → not a palindrome.
- (Optional but polite: re-reverse the second half to restore the list before returning.)
- For odd length, the middle node value is skipped automatically since the second half is shorter or starts after it.
- Edge cases: empty, single node (both trivially palindromes).
- Time O(n), Space O(1). Alternative: copy to array and two-pointer compare (O(n) space).

```java
// 1. Find middle   2. Reverse second half   3. Compare   4. Restore (optional)
boolean isPalindrome(ListNode head) {
    ListNode mid = middleNode(head);
    ListNode secondHalf = reverse(mid);
    ListNode p1 = head, p2 = secondHalf;
    while (p2 != null) {
        if (p1.val != p2.val) return false;
        p1 = p1.next; p2 = p2.next;
    }
    return true;
}
```

### Questions

| # | Problem | LC# |
|---|---------|-----|
| E1 | Linked List Cycle | 141 |
| E2 | Middle of the Linked List | 876 |
| M1 | Linked List Cycle II (entry point) | 142 |
| M2 | Remove Nth Node From End of List | 19 |
| M3 | Palindrome Linked List | 234 |
| H1 | Find the Duplicate Number (Floyd on array indices) | 287 |

---

## P2: Reversal Patterns

### Variation 1: Reverse Entire List (LC 206)

**Problem:** Given the head of a singly linked list, reverse the list and return the new head.

**Approach (Three-pointer flip):**
- Maintain `prev`, `curr`, `next`. At each step: save `next = curr.next`, flip `curr.next = prev`, advance `prev = curr; curr = next`.
- When `curr` becomes null, `prev` is the new head.
- Recursive version: recurse to the tail, then on unwind do `head.next.next = head; head.next = null;`.
- Edge cases: empty list, single node — both return as-is.
- Iterative: Time O(n), Space O(1). Recursive: Time O(n), Space O(n) stack.

```java
// Iterative — O(1) space
ListNode reverse(ListNode head) {
    ListNode prev = null, curr = head;
    while (curr != null) {
        ListNode next = curr.next;
        curr.next = prev;
        prev = curr;
        curr = next;
    }
    return prev;
}

// Recursive — O(n) space
ListNode reverseRec(ListNode head) {
    if (head == null || head.next == null) return head;
    ListNode newHead = reverseRec(head.next);
    head.next.next = head;
    head.next = null;
    return newHead;
}
```

### Variation 2: Reverse Nodes in K-Group (LC 25) — FAANG Favorite

**Problem:** Reverse the nodes of the list `k` at a time and return the modified head. If the number of remaining nodes is less than `k`, leave them as-is. Do it in O(1) extra space (besides recursion stack if using recursion).

**Approach (Count, reverse, recurse/iterate):**
- First, walk `k` steps to confirm `k` nodes are available; if not, return head unchanged.
- Reverse the first `k` nodes using the standard three-pointer trick. After reversal, the original `head` is now the **tail** of the reversed group.
- Recursively (or iteratively) reverse the remaining list, and connect `head.next = reverseKGroup(node, k)`.
- Return `prev` (new head of the reversed group).
- Edge cases: `k = 1` (no-op), list length < k (return head).
- Time O(n) (each node touched twice), Space O(n/k) recursion or O(1) iterative.

```java
// 1. Check if k nodes remain   2. Reverse k nodes   3. Connect to rest
ListNode reverseKGroup(ListNode head, int k) {
    ListNode curr = head;
    int count = 0;
    while (curr != null && count < k) { curr = curr.next; count++; }
    if (count < k) return head;        // less than k nodes — don't reverse

    ListNode prev = null, node = head;
    for (int i = 0; i < k; i++) {
        ListNode next = node.next;
        node.next = prev;
        prev = node;
        node = next;
    }
    head.next = reverseKGroup(node, k);  // head is now the tail of reversed group
    return prev;                          // prev is new head
}
```

### Variation 3: Reverse Between L and R (LC 92)

**Problem:** Reverse the nodes of the list from position `left` to position `right` (1-indexed, inclusive) and return the head. Do it in **one pass**.

**Approach (Head-insertion / "front-insert" trick):**
- Use a dummy node before head. Walk `prev` to the node just before position `left`.
- Now `curr = prev.next` is the first node of the sublist. It will stay put and become the tail of the reversed segment.
- Repeatedly take `curr.next` and splice it to the front of the sublist (right after `prev`). Do this `right - left` times.
- This achieves reversal in place without an extra traversal.
- Edge cases: `left == right` (no-op), reversing from head (dummy handles it), reversing the whole list.
- Time O(n), Space O(1).

```java
// Reverse only the sublist from position left to right
ListNode reverseBetween(ListNode head, int left, int right) {
    ListNode dummy = new ListNode(0);
    dummy.next = head;
    ListNode prev = dummy;

    for (int i = 1; i < left; i++) prev = prev.next;  // move to node before left

    ListNode curr = prev.next;
    for (int i = 0; i < right - left; i++) {
        ListNode next = curr.next;
        curr.next = next.next;
        next.next = prev.next;
        prev.next = next;
    }
    return dummy.next;
}
```

### Variation 4: Rotate List (LC 61)

**Problem:** Rotate the linked list to the **right** by `k` places. (Last node becomes the new head, repeated `k` times.) `k` may exceed list length.

**Approach (Circularize then cut):**
- Compute length `n` and walk to the tail in one pass.
- Effective rotations = `k % n` (handles huge `k`). If 0 → no rotation.
- Connect `tail.next = head` to form a cycle.
- Walk `n - k` steps from head to find the **new tail**. New head is `newTail.next`. Cut by `newTail.next = null`.
- Edge cases: empty list, `k == 0`, `k` a multiple of `n`.
- Time O(n), Space O(1).

```java
// Rotate right by k — connect tail to head, then cut at right point
ListNode rotateRight(ListNode head, int k) {
    if (head == null) return null;
    int n = 1;
    ListNode tail = head;
    while (tail.next != null) { tail = tail.next; n++; }

    k = k % n;
    if (k == 0) return head;

    tail.next = head;   // make circular
    int steps = n - k;
    ListNode newTail = head;
    for (int i = 1; i < steps; i++) newTail = newTail.next;

    ListNode newHead = newTail.next;
    newTail.next = null;
    return newHead;
}
```

### Questions

| # | Problem | LC# |
|---|---------|-----|
| E1 | Reverse Linked List | 206 |
| M1 | Reverse Linked List II (between L and R) | 92 |
| M2 | Rotate List | 61 |
| M3 | Swap Nodes in Pairs | 24 |
| H1 | Reverse Nodes in K-Group | 25 |

---

## P3: Merge & Sort

### Variation 1: Merge Two Sorted Lists (LC 21)

**Problem:** Merge two sorted (ascending) linked lists into one sorted list and return its head. Splice nodes (don't allocate new ones).

**Approach (Two-pointer merge with dummy):**
- Use a dummy head; `curr` points to the tail of the result we're building.
- At each step pick the smaller of `l1.val` vs `l2.val`, splice it, advance that pointer.
- When one list is exhausted, attach the other entirely (its remainder is already sorted).
- Edge cases: either list null (return the other), equal values (either choice works, but stable behavior usually picks l1 on tie).
- Time O(n + m), Space O(1).

```java
ListNode mergeTwoLists(ListNode l1, ListNode l2) {
    ListNode dummy = new ListNode(0), curr = dummy;
    while (l1 != null && l2 != null) {
        if (l1.val <= l2.val) { curr.next = l1; l1 = l1.next; }
        else                  { curr.next = l2; l2 = l2.next; }
        curr = curr.next;
    }
    curr.next = (l1 != null) ? l1 : l2;
    return dummy.next;
}
```

### Variation 2: Merge K Sorted Lists (LC 23) — FAANG Favorite

**Problem:** Given an array of `k` sorted linked lists, merge them into one sorted list.

**Approach (Min-heap of head pointers):**
- Push the head of every non-null list into a min-heap keyed on `val`. Heap size ≤ k at all times.
- Poll the smallest, splice it onto the result tail, and push its `next` (if non-null) back into the heap.
- Each of the N total nodes is pushed/popped once → O(log k) per op → **O(N log k)**.
- Edge cases: empty input array, lists containing nulls.
- Alternative: divide-and-conquer pairwise merge — same O(N log k) without a heap; uses O(log k) recursion. Naive sequential merge is O(N·k).
- Space: O(k) for heap.

```java
// Use a min-heap of size k
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
// Time: O(N log k) where N = total nodes, k = number of lists
```

### Variation 3: Sort List (LC 148) — Merge Sort on LL

**Problem:** Sort a linked list in **O(n log n)** time and ideally O(1) extra space (or O(log n) recursion).

**Approach (Merge Sort on linked list):**
- Why merge sort, not quick sort? Random access is O(n) in a linked list — merge sort needs only sequential access and is natural here.
- Recursively: find middle (slow/fast), split into two halves (set `mid.next = null`), sort each half, merge using LC 21.
- For strict O(1) space: bottom-up iterative merge sort — merge sublists of size 1, then 2, then 4, ... ⌈log n⌉ passes.
- Edge cases: empty/single-node list returns as-is.
- Time O(n log n), Space O(log n) recursion (or O(1) iterative).

```java
// 1. Find middle   2. Split   3. Sort each half   4. Merge
ListNode sortList(ListNode head) {
    if (head == null || head.next == null) return head;

    // Find middle (slow/fast)
    ListNode mid = getMid(head);
    ListNode right = mid.next;
    mid.next = null;  // split

    ListNode left = sortList(head);
    right = sortList(right);
    return mergeTwoLists(left, right);
}
// Time: O(n log n) | Space: O(log n) for recursion
```

### Questions

| # | Problem | LC# |
|---|---------|-----|
| E1 | Merge Two Sorted Lists | 21 |
| M1 | Sort List | 148 |
| H1 | Merge K Sorted Lists | 23 |

---

## P4: In-Place / Tricky Operations

### Variation 1: Add Two Numbers (LC 2)

**Problem:** Two non-empty linked lists represent non-negative integers in **reverse order**, one digit per node. Add them and return the sum as a linked list in the same form.

**Approach (Schoolbook addition with carry):**
- Walk both lists in parallel. At each position, sum = `l1.val + l2.val + carry`. New digit = `sum % 10`, new carry = `sum / 10`.
- Continue while either list has nodes left OR carry != 0 (the final carry can produce one extra node, e.g., 9+1 = 10).
- Build the result with a dummy head.
- Edge cases: different lengths (treat missing digit as 0), final carry-out, both lists single-digit producing 2-digit sum.
- Time O(max(n, m)), Space O(max(n, m)) for the result.

```java
// Simulate addition digit by digit
ListNode addTwoNumbers(ListNode l1, ListNode l2) {
    ListNode dummy = new ListNode(0), curr = dummy;
    int carry = 0;
    while (l1 != null || l2 != null || carry != 0) {
        int sum = carry;
        if (l1 != null) { sum += l1.val; l1 = l1.next; }
        if (l2 != null) { sum += l2.val; l2 = l2.next; }
        carry = sum / 10;
        curr.next = new ListNode(sum % 10);
        curr = curr.next;
    }
    return dummy.next;
}
```

### Variation 2: Copy List with Random Pointer (LC 138)

**Problem:** Each node has `val`, `next`, and a `random` pointer that may point to any node in the list or null. Return a **deep copy** of the list.

**Approach (Interleaved nodes — O(1) extra space):**
- Pass 1: For each original node A, create copy A' and weave it in: `A → A' → B → B' → C → C' → ...`.
- Pass 2: Set the random pointer of each copy: `curr.next.random = curr.random.next` (because `curr.random.next` is the copy of whatever `curr.random` points to).
- Pass 3: Un-weave the two lists by restoring `curr.next` for originals and threading copies via `copy.next`.
- Why this works: the interleaved structure gives O(1) access from any original to its copy.
- Edge case: empty list.
- Alternative: HashMap<original, copy> in two passes — O(n) time but O(n) extra space.
- Time O(n), Space O(1) extra.

```java
// Three-pass approach without extra space:
// 1. Interleave: A→A'→B→B'→...
// 2. Set random pointers for copies
// 3. Deinterleave
Node copyRandomList(Node head) {
    if (head == null) return null;
    // Pass 1: insert copy after each node
    Node curr = head;
    while (curr != null) {
        Node copy = new Node(curr.val);
        copy.next = curr.next;
        curr.next = copy;
        curr = copy.next;
    }
    // Pass 2: set random pointers
    curr = head;
    while (curr != null) {
        if (curr.random != null)
            curr.next.random = curr.random.next;
        curr = curr.next.next;
    }
    // Pass 3: restore original + extract copy
    curr = head;
    Node copyHead = head.next;
    while (curr != null) {
        Node copy = curr.next;
        curr.next = copy.next;
        curr = curr.next;
        copy.next = curr != null ? curr.next : null;
    }
    return copyHead;
}
```

### Variation 3: Reorder List (LC 143)
> L0→L1→...→Ln-1→Ln becomes L0→Ln→L1→Ln-1→...

**Problem:** Reorder the list in place so it interleaves the first half with the reverse of the second half. Do it without modifying node values.

**Approach (Three-step splice):**
- Step 1: find middle with slow/fast.
- Step 2: reverse the second half (from `mid.next`), and detach: `mid.next = null`.
- Step 3: interleave by alternately splicing nodes from the second (reversed) list between nodes of the first.
- This decomposes a tricky-looking reorder into three well-known primitives.
- Edge cases: lists of length 0, 1, 2 (no work needed).
- Time O(n), Space O(1).

```java
// 1. Find middle   2. Reverse second half   3. Merge alternately
void reorderList(ListNode head) {
    ListNode mid = getMid(head);
    ListNode second = reverse(mid.next);
    mid.next = null;

    ListNode first = head;
    while (second != null) {
        ListNode tmp1 = first.next, tmp2 = second.next;
        first.next = second;
        second.next = tmp1;
        first = tmp1;
        second = tmp2;
    }
}
```

### Variation 4: Flatten a Multilevel Doubly Linked List (LC 430)

**Problem:** A doubly linked list where each node also has a `child` pointer (itself a doubly linked list, possibly with deeper children). Flatten into a single-level doubly linked list using DFS order. Set all `child` pointers to null and fix `prev`/`next`.

**Approach (DFS with explicit stack):**
- Walk `curr` from head. When `curr.child` exists:
  - Save `curr.next` on a stack (we'll resume there later).
  - Splice the child list into the main list: `curr.next = curr.child`, `child.prev = curr`, clear `curr.child`.
- When you reach a node with `curr.next == null` and the stack isn't empty, pop the saved node and splice it back in.
- This emulates recursive DFS but with O(depth) stack space explicit and no recursion stack overflow risk.
- Edge case: null head, list with no children (no-op).
- Time O(n), Space O(d) where d is max depth.

```java
// DFS / Stack approach: push next onto stack when child exists
Node flatten(Node head) {
    Node curr = head;
    Deque<Node> stack = new ArrayDeque<>();
    while (curr != null) {
        if (curr.child != null) {
            if (curr.next != null) stack.push(curr.next);
            curr.next = curr.child;
            curr.child.prev = curr;
            curr.child = null;
        } else if (curr.next == null && !stack.isEmpty()) {
            Node saved = stack.pop();
            curr.next = saved;
            saved.prev = curr;
        }
        curr = curr.next;
    }
    return head;
}
```

### Questions

| # | Problem | LC# |
|---|---------|-----|
| M1 | Add Two Numbers | 2 |
| M2 | Odd Even Linked List | 328 |
| M3 | Reorder List | 143 |
| M4 | Flatten Multilevel Doubly LL | 430 |
| H1 | Copy List with Random Pointer | 138 |
| H2 | LRU Cache (HashMap + DLL) | 146 |
| H3 | Design Linked List | 707 |

---

## LRU Cache — FAANG System Design in Code

**Problem (LC 146):** Design a data structure with `get(key)` and `put(key, value)` operations, both in **O(1)** average time. Capacity is fixed; when full, evict the **least recently used** key on insert.

**Approach (HashMap + Doubly Linked List):**
- HashMap gives O(1) lookup from key → node.
- Doubly linked list (with dummy head and dummy tail sentinels) gives O(1) removal from any position **and** O(1) insertion at the front.
- Convention: most recently used (MRU) at head, least recently used (LRU) at tail.
- `get`: lookup in map; if found, move that node to head and return value.
- `put`: if key exists, update value and move-to-head; else create new node, insert at head, add to map; if over capacity, drop the node just before tail and remove from map.
- Why dummy sentinels? They eliminate null-checks in `remove`/`insertFront` — every real node always has non-null `prev` and `next`.
- Why doubly linked (not singly)? Because removing a known node requires its predecessor in O(1).
- For thread safety, see `LLD_07_LRU_Cache.md` (striped locks or `ConcurrentHashMap` + segment-locked DLL).
- Time O(1) per op, Space O(capacity).

```java
// HashMap + Doubly Linked List
class LRUCache {
    private final int capacity;
    private final Map<Integer, Node> map;
    private final Node head, tail;   // dummy nodes

    class Node {
        int key, val;
        Node prev, next;
        Node(int k, int v) { key = k; val = v; }
    }

    public LRUCache(int capacity) {
        this.capacity = capacity;
        map = new HashMap<>();
        head = new Node(0, 0); tail = new Node(0, 0);
        head.next = tail; tail.prev = head;
    }

    public int get(int key) {
        if (!map.containsKey(key)) return -1;
        Node node = map.get(key);
        remove(node);
        insertFront(node);
        return node.val;
    }

    public void put(int key, int value) {
        if (map.containsKey(key)) remove(map.get(key));
        if (map.size() == capacity) {
            Node lru = tail.prev;
            remove(lru);
            map.remove(lru.key);
        }
        Node newNode = new Node(key, value);
        insertFront(newNode);
        map.put(key, newNode);
    }

    private void remove(Node node) {
        node.prev.next = node.next;
        node.next.prev = node.prev;
    }

    private void insertFront(Node node) {
        node.next = head.next;
        node.prev = head;
        head.next.prev = node;
        head.next = node;
    }
}
```

---

## Common Mistakes (FAANG Traps)

| Mistake | Fix |
|---------|-----|
| Forgetting dummy head node | Always use `ListNode dummy = new ListNode(0)` |
| Off-by-one in fast/slow init | Draw it out; trace n=1, n=2 cases |
| Not handling null next in fast pointer | Always check `fast != null && fast.next != null` |
| Losing reference when reversing | Save `next` before changing `.next` |
| Not reconnecting after reversal | Tail of reversed segment must point to remainder |

---

> Back to [Master Index](../DSA_MASTER_INDEX.md)
