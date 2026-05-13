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

### Variation: Jump Game VI (LC 1696) — DP + Deque
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
