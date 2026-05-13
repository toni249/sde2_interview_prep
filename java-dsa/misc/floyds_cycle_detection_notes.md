# Floyd's Cycle Detection — Deep Dive
> Find a cycle in O(n) time and O(1) space
> Works on: Linked Lists, Arrays as function mappings

---

## The Problem

Given a sequence that may loop back on itself, detect if a cycle exists.
If yes, find where the cycle starts.

```
Linked List with a cycle:

1 → 2 → 3 → 4 → 5
            ↑       ↓
            8 ← 7 ← 6

Cycle starts at node 3.
```

---

## 1. Naive Approach — HashSet

Store every visited node. If you see one twice, there's a cycle.

```java
boolean hasCycle(ListNode head) {
    Set<ListNode> seen = new HashSet<>();
    ListNode curr = head;
    while (curr != null) {
        if (seen.contains(curr)) return true;
        seen.add(curr);
        curr = curr.next;
    }
    return false;
}
```

**Time: O(n) | Space: O(n)** — works but uses extra memory.

**Can we do O(1) space?** Yes — Floyd's algorithm.

---

## 2. The Tortoise and Hare Intuition

Use two pointers:
- **Slow** (tortoise): moves 1 step at a time
- **Fast** (hare): moves 2 steps at a time

```
No cycle: fast pointer hits null before slow catches up.

With cycle:
- Fast enters the cycle, starts lapping.
- Slow enters the cycle.
- Fast is ALWAYS gaining 1 step on slow per round.
- They MUST eventually meet (like two runners on a circular track).
```

### Visual: Meeting in the Cycle

```
Start:
  slow: 1 → 2 → 3 → 4 → 5
  fast: 1 → 3 → 5 → 7 → ...

After entering cycle, fast gains 1 per step on slow.
In at most (cycle_length) steps, they meet.
```

---

## 3. Phase 1: Detect the Cycle

```java
boolean hasCycle(ListNode head) {
    ListNode slow = head, fast = head;
    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
        if (slow == fast) return true;   // they met → cycle exists
    }
    return false;   // fast hit null → no cycle
}
```

**Why `fast.next != null`?** If cycle length is odd, fast might land on
the last node before null, so we check both `fast` and `fast.next`.

---

## 4. Phase 2: Find the Cycle Start — The Math

This is the non-obvious part. After slow and fast meet:

```
                    ┌──────────cycle (length C)──────────┐
HEAD ──── F ────► ENTRY ──── K ────► MEETING POINT ──────┘

F = distance from head to cycle entry
K = distance from entry to meeting point (within cycle)
```

**How far has each pointer traveled when they meet?**

```
Slow traveled: F + K
Fast traveled: F + K + n*C    (fast went around the cycle n extra times)

Since fast = 2 × slow:
F + K + n*C = 2(F + K)
n*C = F + K
F = n*C - K
F = (n-1)*C + (C - K)
```

**In simpler terms:** `F = C - K` (for n=1, one extra lap).

This means:
- Distance from HEAD to ENTRY = distance from MEETING POINT to ENTRY
  (going around the rest of the cycle)

**So:** Reset one pointer to HEAD, keep the other at MEETING POINT.
Move both 1 step at a time. They will meet exactly at the CYCLE ENTRY.

---

## 5. Phase 2: Find Cycle Start in Code

```java
ListNode detectCycle(ListNode head) {
    ListNode slow = head, fast = head;

    // Phase 1: find meeting point
    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
        if (slow == fast) break;
    }

    // No cycle
    if (fast == null || fast.next == null) return null;

    // Phase 2: find cycle start
    slow = head;                    // reset slow to head
    while (slow != fast) {          // fast stays at meeting point
        slow = slow.next;
        fast = fast.next;           // both move 1 step
    }
    return slow;   // meeting point = cycle start
}
```

---

## 6. Full Worked Example

```
List: 1 → 2 → 3 → 4 → 5 → 6
                   ↑           ↓
                   └─────── 7 ←┘

Cycle entry = node 4, Cycle = [4→5→6→7→4], length C=4
F = 3 (1→2→3, then enter at 4)
```

**Phase 1 trace:**

```
Step  slow    fast
 0     1       1
 1     2       3
 2     3       5
 3     4       7
 4     5       5     ← MEET (K=1 from entry node 4)
```

Meeting point = node 5.

**Math check:** F = C - K = 4 - 1 = 3. Distance from HEAD to ENTRY = 3. ✓
Distance from MEETING (5) to ENTRY (4) going around = 5→6→7→4 = 3. ✓

**Phase 2 trace:**

```
Step  slow(from head)  fast(from meeting)
 0     1                5
 1     2                6
 2     3                7
 3     4                4     ← MEET at node 4 = cycle entry ✓
```

---

## 7. Applications Beyond Linked Lists

### Application 1: Find the Duplicate Number (LC 287)
```java
// Array of n+1 numbers, values in [1,n]. Exactly one duplicate.
// Treat the array as a linked list: index i → nums[i]
// Since values are in [1,n] and we have n+1 elements, there's a cycle.
int findDuplicate(int[] nums) {
    int slow = nums[0], fast = nums[0];
    do {
        slow = nums[slow];
        fast = nums[nums[fast]];
    } while (slow != fast);

    slow = nums[0];    // reset
    while (slow != fast) {
        slow = nums[slow];
        fast = nums[fast];
    }
    return slow;   // duplicate = cycle entry
}
```

Why this works:
```
nums = [1, 3, 4, 2, 2]
Index:  0  1  2  3  4

Following: 0 → nums[0]=1 → nums[1]=3 → nums[3]=2 → nums[2]=4 → nums[4]=2 → ...
                                         ↑                           ↓
                                         └─────────── cycle ─────────┘
Cycle entry = 2 = the duplicate.
```

### Application 2: Happy Number (LC 202)
```java
// Is n a "happy number"? Sum squares of digits repeatedly.
// If cycle → not happy (and cycle contains 4).
boolean isHappy(int n) {
    int slow = n, fast = n;
    do {
        slow = sumOfSquares(slow);
        fast = sumOfSquares(sumOfSquares(fast));
    } while (slow != fast);
    return slow == 1;   // happy number cycles to 1
}

int sumOfSquares(int n) {
    int sum = 0;
    while (n > 0) { int d = n % 10; sum += d*d; n /= 10; }
    return sum;
}
```

---

## 8. Cycle Length

After detecting the meeting point, how to find cycle length C?

```java
int cycleLength(ListNode meetingPoint) {
    int length = 1;
    ListNode curr = meetingPoint.next;
    while (curr != meetingPoint) {
        curr = curr.next;
        length++;
    }
    return length;
}
```

---

## 9. Cheat Sheet

```
Floyd's Cycle Detection — 2 Phases:

Phase 1: Detect cycle
  slow = fast = head
  loop: slow = slow.next, fast = fast.next.next
  if slow == fast → cycle found

Phase 2: Find start (after phase 1 meeting)
  reset slow = head
  both move 1 step: slow = slow.next, fast = fast.next
  when slow == fast → that's the cycle entry

The Math: F = C - K  (distance to entry = rest of cycle from meeting)
```

### Pattern Recognition: When to Use Floyd's
- [ ] Detect cycle in linked list (O(1) space required)
- [ ] Find cycle START in linked list
- [ ] Find duplicate in array of n+1 numbers (LC 287)
- [ ] Happy number (detect if sequence eventually cycles)

---

## 10. Related Problems

| # | Problem | LC# | Difficulty |
|---|---------|-----|------------|
| E1 | Linked List Cycle | 141 | E |
| M1 | Linked List Cycle II (find start) | 142 | M |
| M2 | Find the Duplicate Number | 287 | M |
| E2 | Happy Number | 202 | E |
| M3 | Middle of the Linked List | 876 | E |
