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
