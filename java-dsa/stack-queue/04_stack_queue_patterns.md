# Stack & Queue — Pattern Deep Dive
> FAANG Level | Java | Easy → Hard

---

## Pattern Map

```
STACK & QUEUE
├── P1: Monotonic Stack      → next greater/smaller, histogram, span
├── P2: Stack Simulation     → parentheses, decode, calculator, asteroid
├── P3: Monotonic Deque      → sliding window max/min
├── P4: BFS with Queue       → (see Graphs topic — BFS)
└── P5: Design Problems      → Min Stack, Max Stack, Queue using Stacks
```

---

## P1: Monotonic Stack

### Core Idea
> Maintain a stack where elements are always in increasing or decreasing order.
> When you push a new element and it violates the order — pop those elements first.
> Those popped elements found their "answer" (next greater or next smaller).

### Rule of Thumb
```
"Next Greater" → Maintain INCREASING stack (pop when current > stack top)
"Next Smaller" → Maintain DECREASING stack (pop when current < stack top)
```

### Variation 1: Next Greater Element (LC 496)

**Problem:** For each element in `nums`, find the **next greater element** to its right. If no such element exists, output -1. (LC 496 actually uses two arrays with `nums1 ⊆ nums2`; this is the canonical single-array version.)

**Approach (Monotonic decreasing stack of indices):**
- Iterate left to right, maintaining a stack of indices whose answer is **still unknown**.
- For each `nums[i]`, while the stack top is smaller than `nums[i]`, we've found that top's answer → pop and assign.
- Push `i`. At the end, indices remaining in the stack have no greater element → already -1 (from `Arrays.fill`).
- Each index is pushed and popped at most once → O(n) total.
- Time O(n), Space O(n).

```java
int[] nextGreaterElement(int[] nums) {
    int n = nums.length;
    int[] result = new int[n];
    Arrays.fill(result, -1);
    Deque<Integer> stack = new ArrayDeque<>();  // stores indices

    for (int i = 0; i < n; i++) {
        // All elements in stack that are smaller than nums[i]
        // have nums[i] as their next greater element
        while (!stack.isEmpty() && nums[stack.peek()] < nums[i]) {
            result[stack.pop()] = nums[i];
        }
        stack.push(i);
    }
    return result;
}
```

### Variation 2: Next Greater Element II (Circular Array, LC 503)

**Problem:** Same as LC 496 but the array is **circular** — wrapping past the end is allowed when searching for a greater element.

**Approach (Two-pass trick on doubled index):**
- Iterate `i` from `0` to `2n - 1` and use `idx = i % n` to access elements.
- Only **push** in the first pass (`i < n`); the second pass just resolves indices left in the stack via wrap-around.
- This avoids physically duplicating the array while still giving each index a "second chance" to find its next greater.
- Time O(n), Space O(n).

```java
// Process array twice to simulate circular behavior
int[] nextGreaterElements(int[] nums) {
    int n = nums.length;
    int[] result = new int[n];
    Arrays.fill(result, -1);
    Deque<Integer> stack = new ArrayDeque<>();

    for (int i = 0; i < 2 * n; i++) {          // loop twice
        int idx = i % n;
        while (!stack.isEmpty() && nums[stack.peek()] < nums[idx]) {
            result[stack.pop()] = nums[idx];
        }
        if (i < n) stack.push(idx);             // only push in first pass
    }
    return result;
}
```

### Variation 3: Daily Temperatures (LC 739)

**Problem:** Given an array of daily temperatures, return an array where `answer[i]` is the **number of days** you must wait after day `i` to get a warmer temperature. 0 if no such day.

**Approach (Monotonic decreasing stack, store distance):**
- Same as Next Greater Element, but when you pop index `j` because `temps[i] > temps[j]`, record `result[j] = i - j` (the **distance**, not the value).
- Unpopped indices at the end keep their default 0.
- Time O(n), Space O(n).

```java
// "How many days until warmer temperature?" — next greater, store distance
int[] dailyTemperatures(int[] temps) {
    int n = temps.length;
    int[] result = new int[n];
    Deque<Integer> stack = new ArrayDeque<>();

    for (int i = 0; i < n; i++) {
        while (!stack.isEmpty() && temps[stack.peek()] < temps[i]) {
            int idx = stack.pop();
            result[idx] = i - idx;          // distance to next warmer day
        }
        stack.push(i);
    }
    return result;
}
```

### Variation 4: Stock Span Problem

**Problem:** For each day, return the **span** = number of consecutive days (ending at and including today) whose price is ≤ today's price.

**Approach (Monotonic decreasing stack of indices):**
- We want, for each `i`, the index of the **previous greater** price. Span = `i - prevGreaterIndex` (or `i + 1` if none).
- Maintain a stack of indices with strictly decreasing prices. Pop while `prices[stack.top] <= prices[i]`. After popping, the stack top is the previous greater.
- Span is the distance between current index and that top.
- Each index pushed/popped at most once → amortized O(1) per query, O(n) total.
- Time O(n), Space O(n).

```java
// How many consecutive days ≤ today's price (including today)?
int[] stockSpan(int[] prices) {
    int n = prices.length;
    int[] span = new int[n];
    Deque<Integer> stack = new ArrayDeque<>();  // stores indices

    for (int i = 0; i < n; i++) {
        while (!stack.isEmpty() && prices[stack.peek()] <= prices[i]) {
            stack.pop();
        }
        span[i] = stack.isEmpty() ? i + 1 : i - stack.peek();
        stack.push(i);
    }
    return span;
}
```

### Variation 5: Largest Rectangle in Histogram (LC 84) — FAANG Classic

**Problem:** Given non-negative bar heights of width 1, return the area of the largest axis-aligned rectangle contained in the histogram.

**Approach (Monotonic increasing stack of indices):**
- Key insight: the largest rectangle's height equals **some single bar `h[j]`**, and its width spans from the **previous smaller** bar to the **next smaller** bar (both exclusive).
- Maintain a stack of indices with strictly increasing heights. When we see `h[i] < h[stack.top]`, the bar at `stack.top` has just found its next smaller (= `i`) and its previous smaller (= the new stack top after popping).
- Pop and compute area = `h[popped] * (i - newTop - 1)`. If stack is empty, width is `i` (extends to the start).
- Append a sentinel 0 at the end (`i == n`) to flush any remaining bars.
- Edge case: all equal heights — only flushed by sentinel.
- Time O(n), Space O(n).

```java
// For each bar: find left and right boundaries where it is the smallest
int largestRectangleArea(int[] heights) {
    int n = heights.length;
    int maxArea = 0;
    Deque<Integer> stack = new ArrayDeque<>();  // increasing stack

    for (int i = 0; i <= n; i++) {
        int h = (i == n) ? 0 : heights[i];     // sentinel 0 at end to flush stack

        while (!stack.isEmpty() && heights[stack.peek()] > h) {
            int height = heights[stack.pop()];
            int width = stack.isEmpty() ? i : i - stack.peek() - 1;
            maxArea = Math.max(maxArea, height * width);
        }
        stack.push(i);
    }
    return maxArea;
}
```
> **Intuition:** When we pop bar at index `j`, it means `heights[i]` is the first bar to its right that is smaller. The last element in the stack is the first bar to its left that is smaller. So the width is exactly `right - left - 1`.

### Variation 6: Trapping Rain Water — Stack Approach (LC 42)

**Problem:** Given an elevation map (array of non-negative ints), compute how many units of water it can trap after raining.

**Approach (Monotonic decreasing stack — fills layer by layer):**
- Maintain a stack of indices with decreasing heights (potential left walls).
- When `height[i] > height[stack.top]`, we've found a right wall. Pop the bottom (`bottom`), and if stack still has a left wall:
  - width = `i - left - 1`
  - bounded height = `min(height[left], height[i]) - height[bottom]`
  - Add `width * bounded_height` to water.
- This fills the water **horizontally**, one layer at a time, between left and right walls.
- Edge case: monotonic ascending/descending arrays trap nothing.
- Time O(n), Space O(n). Alternative: two-pointer O(n) time, O(1) space.

```java
// Think of it as filling containers between walls
int trap(int[] height) {
    int water = 0;
    Deque<Integer> stack = new ArrayDeque<>();

    for (int i = 0; i < height.length; i++) {
        while (!stack.isEmpty() && height[stack.peek()] < height[i]) {
            int bottom = stack.pop();
            if (stack.isEmpty()) break;
            int left = stack.peek();
            int width = i - left - 1;
            int boundedHeight = Math.min(height[left], height[i]) - height[bottom];
            water += width * boundedHeight;
        }
        stack.push(i);
    }
    return water;
}
```

### Variation 7: Sum of Subarray Minimums (LC 907)

**Problem:** Given an array `arr`, return the sum of `min(b)` over **every contiguous subarray** `b`. Return mod 1e9+7.

**Approach (Contribution counting via monotonic stacks):**
- Instead of enumerating O(n²) subarrays, count for each element `arr[i]` how many subarrays have `arr[i]` as the **min**, then sum `arr[i] * count`.
- For `arr[i]`, let `left[i]` = distance to the previous strictly smaller element (or to start if none), `right[i]` = distance to the next smaller-or-equal element. Count = `left[i] * right[i]`.
- Use **strict on one side** and **non-strict on the other** to break ties (avoids double-counting when duplicates exist).
- Compute both arrays via monotonic stacks in O(n) each.
- Time O(n), Space O(n).

```java
// For each element, find how many subarrays it is the minimum of
// Answer = sum of (element × count_of_subarrays_where_it_is_min)
int sumSubarrayMins(int[] arr) {
    int n = arr.length;
    long res = 0;
    final int MOD = 1_000_000_007;
    int[] left = new int[n];   // distance to previous smaller element
    int[] right = new int[n];  // distance to next smaller or equal element
    Deque<Integer> stack = new ArrayDeque<>();

    // previous smaller (strict)
    for (int i = 0; i < n; i++) {
        while (!stack.isEmpty() && arr[stack.peek()] >= arr[i]) stack.pop();
        left[i] = stack.isEmpty() ? i + 1 : i - stack.peek();
        stack.push(i);
    }
    stack.clear();
    // next smaller or equal (non-strict — to avoid double counting)
    for (int i = n - 1; i >= 0; i--) {
        while (!stack.isEmpty() && arr[stack.peek()] > arr[i]) stack.pop();
        right[i] = stack.isEmpty() ? n - i : stack.peek() - i;
        stack.push(i);
    }
    for (int i = 0; i < n; i++) {
        res = (res + (long) arr[i] * left[i] * right[i]) % MOD;
    }
    return (int) res;
}
```

### Questions — Monotonic Stack

| # | Problem | LC# | Difficulty |
|---|---------|-----|------------|
| E1 | Next Greater Element I | 496 | E |
| M1 | Daily Temperatures | 739 | M |
| M2 | Next Greater Element II (circular) | 503 | M |
| M3 | Remove K Digits (smallest number) | 402 | M |
| M4 | 132 Pattern | 456 | M |
| M5 | Buildings With an Ocean View | 1762 | M |
| H1 | Largest Rectangle in Histogram | 84 | H |
| H2 | Maximal Rectangle (2D) | 85 | H |
| H3 | Trapping Rain Water | 42 | H |
| H4 | Sum of Subarray Minimums | 907 | H |
| H5 | Maximum Width Ramp | 962 | H |

---

## P2: Stack Simulation

### Variation 1: Valid Parentheses Variants

**Problem (LC 20):** Given a string containing only `()[]{}`, determine if every opening bracket is properly closed by a matching bracket in the correct order.

**Approach (Stack of expected closers / openers):**
- Push every opening bracket onto a stack.
- On a closing bracket, the top of stack must be the matching opener — pop and compare; else invalid.
- At end, stack must be empty (no unmatched openers).
- Edge cases: empty string is valid; odd-length string is always invalid (early-exit optimization).
- Time O(n), Space O(n).

**Problem (LC 921 — Minimum Add to Make Valid):** Return the minimum number of brackets you must insert anywhere in the string to make it valid.

**Approach (Two counters — no stack needed):**
- `open` = unmatched `(` so far; `close` = unmatched `)` so far.
- On `(`: `open++`. On `)`: if `open > 0` match it (`open--`), else it's unmatched → `close++`.
- Answer = `open + close` (each unmatched needs its partner inserted).
- Equivalent to running the stack but only tracking sizes — O(1) extra space.
- Time O(n), Space O(1).

```java
// Check if balanced — core template
boolean isValid(String s) {
    Deque<Character> stack = new ArrayDeque<>();
    for (char c : s.toCharArray()) {
        if ("([{".indexOf(c) != -1) {
            stack.push(c);
        } else {
            if (stack.isEmpty()) return false;
            char top = stack.pop();
            if ((c == ')' && top != '(') ||
                (c == ']' && top != '[') ||
                (c == '}' && top != '{')) return false;
        }
    }
    return stack.isEmpty();
}
```

```java
// Minimum add to make valid (LC 921)
int minAddToMakeValid(String s) {
    int open = 0, close = 0;
    for (char c : s.toCharArray()) {
        if (c == '(') open++;
        else {
            if (open > 0) open--;   // match with an open
            else close++;            // unmatched close
        }
    }
    return open + close;
}
```

### Variation 2: Asteroid Collision (LC 735)

**Problem:** Each asteroid moves right (positive) or left (negative) at the same speed. When two collide, the smaller explodes; equal-size both explode; same direction never collide. Return the state after all collisions.

**Approach (Stack simulation):**
- The only collisions happen when a **negative** (moving left) asteroid meets the top of the stack which is **positive** (moving right).
- For each incoming asteroid `a`:
  - While `a < 0` and `stack.peek() > 0`: compare magnitudes.
    - `|top| < |a|` → top explodes, keep looping with `a` alive.
    - `|top| == |a|` → both explode, `a` dies.
    - `|top| > |a|` → `a` dies.
  - If `a` survives, push it.
- Stack invariant: any negatives on the stack are at the **bottom** (they never collide with subsequent negatives).
- Time O(n), Space O(n).

```java
int[] asteroidCollision(int[] asteroids) {
    Deque<Integer> stack = new ArrayDeque<>();
    for (int a : asteroids) {
        boolean alive = true;
        while (alive && a < 0 && !stack.isEmpty() && stack.peek() > 0) {
            if (stack.peek() < -a)       stack.pop();   // top destroyed
            else if (stack.peek() == -a){ stack.pop(); alive = false; }  // both destroyed
            else                          alive = false;   // new one destroyed
        }
        if (alive) stack.push(a);
    }
    int[] res = new int[stack.size()];
    for (int i = res.length - 1; i >= 0; i--) res[i] = stack.pop();
    return res;
}
```

### Variation 3: Remove Duplicate Letters (LC 316) — Greedy + Stack
> Remove duplicates so result is lexicographically smallest.

**Problem:** Given a string `s`, remove duplicate letters so that every letter appears exactly once and the resulting string is **lexicographically smallest** among all such results, preserving relative order.

**Approach (Greedy with monotonic stack + frequency / in-stack flags):**
- Precompute `freq[c]` for each character.
- Walk the string; decrement `freq[c]` as we pass each `c`.
- If `c` is already in the stack, skip (we keep the leftmost good occurrence).
- Else, while `stack.top > c` AND `freq[stack.top] > 0` (we'll see another occurrence later), pop the top — a smaller char in front is better since we'll get another chance to place the popped letter.
- Push `c`, mark it in-stack.
- Why correct: the popped char will be re-added later (its freq > 0), and replacing it with a smaller letter at this position lex-decreases the result.
- Time O(n) (each char pushed/popped at most once), Space O(26).

```java
String removeDuplicateLetters(String s) {
    int[] freq = new int[26];
    boolean[] inStack = new boolean[26];
    for (char c : s.toCharArray()) freq[c - 'a']++;

    Deque<Character> stack = new ArrayDeque<>();
    for (char c : s.toCharArray()) {
        freq[c - 'a']--;
        if (inStack[c - 'a']) continue;
        // Pop larger chars if they appear later
        while (!stack.isEmpty() && stack.peek() > c && freq[stack.peek() - 'a'] > 0) {
            inStack[stack.pop() - 'a'] = false;
        }
        stack.push(c);
        inStack[c - 'a'] = true;
    }
    StringBuilder sb = new StringBuilder();
    for (char c : stack) sb.append(c);
    return sb.reverse().toString();
}
```

### Questions — Stack Simulation

| # | Problem | LC# |
|---|---------|-----|
| E1 | Valid Parentheses | 20 |
| M1 | Minimum Add to Make Parentheses Valid | 921 |
| M2 | Asteroid Collision | 735 |
| M3 | Remove Duplicate Letters | 316 |
| M4 | Remove K Digits | 402 |
| M5 | Evaluate Reverse Polish Notation | 150 |
| H1 | Basic Calculator | 224 |
| H2 | Minimum Remove to Make Valid Parentheses | 1249 |

---

## P3: Monotonic Deque — Sliding Window Maximum

### Core Idea
> Use a deque (double-ended queue) to maintain the maximum of the current window.
> Front = max, back = candidates. Remove from front when out of window, from back when smaller than incoming.

### Variation 1: Sliding Window Maximum (LC 239)

**Problem:** Given an array `nums` and a window size `k`, return an array where each element is the maximum of the corresponding sliding window of size `k`.

**Approach (Monotonic decreasing deque of indices):**
- Maintain a deque of indices such that the corresponding values are **strictly decreasing** front-to-back. Front is always the max of the current window.
- For each `i`:
  1. Pop from **front** while the front index is out of window (`< i - k + 1`).
  2. Pop from **back** while `nums[back] < nums[i]` — those candidates are smaller AND older than `i`, so they can never be max again while `i` is alive.
  3. Add `i` to the back.
  4. Once `i >= k - 1`, record `nums[deque.peekFirst()]` as the window max.
- Each index is added and removed at most once → **amortized O(1)** per window, O(n) total.
- Alternative: max-heap O(n log k) with lazy deletion.
- Time O(n), Space O(k).

```java
// LC 239: Sliding Window Maximum
int[] maxSlidingWindow(int[] nums, int k) {
    int n = nums.length;
    int[] result = new int[n - k + 1];
    Deque<Integer> deque = new ArrayDeque<>();  // stores indices

    for (int i = 0; i < n; i++) {
        // Remove elements outside window
        while (!deque.isEmpty() && deque.peekFirst() < i - k + 1) {
            deque.pollFirst();
        }
        // Maintain decreasing order — remove smaller elements from back
        while (!deque.isEmpty() && nums[deque.peekLast()] < nums[i]) {
            deque.pollLast();
        }
        deque.offerLast(i);
        if (i >= k - 1) result[i - k + 1] = nums[deque.peekFirst()];
    }
    return result;
}
```

### Variation 2: Jump Game VI (LC 1696) — DP + Deque

**Problem:** Starting at index 0, at each step you can jump up to `k` indices forward. The score of a path is the sum of `nums` at visited indices. Return the **maximum** score to reach the last index.

**Approach (DP with sliding-window max via deque):**
- Let `dp[i]` = max score to reach index `i`. Recurrence: `dp[i] = nums[i] + max(dp[i-k], ..., dp[i-1])`.
- Naively this is O(nk). Optimize the "max over last k" using a monotonic decreasing deque, exactly like LC 239 — but on `dp` values.
- Same deque ops: pop front if out of window, pop back if `dp[back] <= dp[i]`, push `i`.
- Final answer is `dp[n - 1]`.
- Edge case: `k >= n` reduces to picking all (max score = sum of positives + nums[0] forced).
- Time O(n), Space O(n).

```java
// dp[i] = max score to reach index i
// dp[i] = nums[i] + max(dp[i-k], ..., dp[i-1])
// Use deque to maintain max of sliding window of size k
int maxResult(int[] nums, int k) {
    int n = nums.length;
    int[] dp = new int[n];
    dp[0] = nums[0];
    Deque<Integer> deque = new ArrayDeque<>();
    deque.offer(0);

    for (int i = 1; i < n; i++) {
        while (!deque.isEmpty() && deque.peekFirst() < i - k) deque.pollFirst();
        dp[i] = nums[i] + dp[deque.peekFirst()];
        while (!deque.isEmpty() && dp[deque.peekLast()] <= dp[i]) deque.pollLast();
        deque.offer(i);
    }
    return dp[n - 1];
}
```

---

## P4: Design Problems

### Min Stack (LC 155)

**Problem:** Design a stack with `push`, `pop`, `top`, and `getMin` — **all in O(1)** time.

**Approach (Auxiliary "min so far" stack):**
- Use a second stack `minStack` that stores the **min of all values up to that depth**.
- On `push(v)`: push `v` to the main stack; push `min(v, minStack.top)` to `minStack` (or just `v` if empty).
- On `pop`: pop both stacks in tandem so heights stay synced.
- `top()` reads main top; `getMin()` reads `minStack` top.
- Invariant: `minStack` has the same height as the main stack, with the running min at each level.
- Space optimization: only push to `minStack` when value `<= currentMin`, and on pop check if popped value equals min. Saves space for non-decreasing input.
- Time O(1) all ops, Space O(n).

```java
// Key: maintain a second stack tracking the minimum at each level
class MinStack {
    Deque<Integer> stack = new ArrayDeque<>();
    Deque<Integer> minStack = new ArrayDeque<>();

    public void push(int val) {
        stack.push(val);
        minStack.push(minStack.isEmpty() ? val : Math.min(val, minStack.peek()));
    }
    public void pop() {
        stack.pop(); minStack.pop();
    }
    public int top()    { return stack.peek(); }
    public int getMin() { return minStack.peek(); }
}
```

### Queue using Two Stacks (LC 232)

**Problem:** Implement FIFO queue (`push`, `pop`, `peek`, `empty`) using only two LIFO stacks. Operations should be **amortized O(1)**.

**Approach (Push-stack + Pop-stack):**
- `inStack` accepts all pushes. `outStack` serves pops/peeks.
- When `outStack` is empty and a pop/peek is needed, drain all of `inStack` into `outStack` — this reverses the order, so the bottom of `inStack` (oldest element) is now on top of `outStack`.
- Each element is pushed/popped at most twice across its lifetime → amortized O(1).
- Worst-case single op is O(n) (when transfer happens).
- Time amortized O(1), Space O(n).

```java
// Amortized O(1) per operation
class MyQueue {
    Deque<Integer> inStack = new ArrayDeque<>();
    Deque<Integer> outStack = new ArrayDeque<>();

    public void push(int x) { inStack.push(x); }

    public int pop() {
        move();
        return outStack.pop();
    }
    public int peek() {
        move();
        return outStack.peek();
    }
    private void move() {
        if (outStack.isEmpty()) {
            while (!inStack.isEmpty()) outStack.push(inStack.pop());
        }
    }
    public boolean empty() { return inStack.isEmpty() && outStack.isEmpty(); }
}
```

### Stack using Two Queues (LC 225)

**Problem:** Implement LIFO stack (`push`, `pop`, `top`, `empty`) using only FIFO queues.

**Approach (Push is heavy — single-queue rotation):**
- On `push(x)`: offer `x` to `q2`, drain all of `q1` into `q2`, swap `q1`/`q2`. Now `q1.front` = latest pushed = "top of stack".
- `pop` and `top` are O(1) reads of `q1.front`.
- Alternative one-queue variant: push `x` then rotate the queue by `size - 1` positions; same complexity, no second queue needed.
- Time: push O(n), pop/top O(1). Space O(n).

```java
class MyStack {
    Queue<Integer> q1 = new LinkedList<>(), q2 = new LinkedList<>();

    public void push(int x) {
        q2.offer(x);
        while (!q1.isEmpty()) q2.offer(q1.poll());
        Queue<Integer> tmp = q1; q1 = q2; q2 = tmp;
    }
    public int pop()  { return q1.poll(); }
    public int top()  { return q1.peek(); }
    public boolean empty() { return q1.isEmpty(); }
}
```

---

## Complete FAANG Stack/Queue Problem List

| Problem | Pattern | LC# | Difficulty |
|---------|---------|-----|------------|
| Valid Parentheses | Stack | 20 | E |
| Min Stack | Design | 155 | M |
| Implement Queue using Stacks | Design | 232 | E |
| Daily Temperatures | Mono Stack | 739 | M |
| Next Greater Element I | Mono Stack | 496 | E |
| Next Greater Element II | Mono Stack | 503 | M |
| Evaluate Reverse Polish Notation | Stack | 150 | M |
| Decode String | Stack | 394 | M |
| Asteroid Collision | Stack | 735 | M |
| Remove Duplicate Letters | Mono Stack + Greedy | 316 | M |
| Remove K Digits | Mono Stack | 402 | M |
| 132 Pattern | Mono Stack | 456 | M |
| Sliding Window Maximum | Mono Deque | 239 | H |
| Largest Rectangle in Histogram | Mono Stack | 84 | H |
| Trapping Rain Water | Mono Stack / Two ptr | 42 | H |
| Basic Calculator | Stack | 224 | H |
| Maximal Rectangle | Mono Stack + Histogram | 85 | H |
| Sum of Subarray Minimums | Mono Stack | 907 | H |
| Maximum Frequency Stack | Design | 895 | H |

---

> Back to [Master Index](../DSA_MASTER_INDEX.md)
