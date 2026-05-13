# Boyer-Moore Voting Algorithm — Deep Dive
> Find the majority element in O(n) time and O(1) space
> One of the most elegant "vote cancellation" algorithms

---

## The Problem

Given an array of n elements, find the element that appears more than n/2 times
(guaranteed to exist).

```
Input:  [3, 2, 3, 1, 3, 3, 2]
Output: 3   (appears 4 times, n/2 = 3.5, so 4 > 3.5)
```

---

## 1. Naive Approaches

### Option A: HashMap
Count frequency of each element. Return the one with count > n/2.
- **Time: O(n) | Space: O(n)**

### Option B: Sorting
Sort the array. The middle element `arr[n/2]` must be the majority.
- **Time: O(n log n) | Space: O(1)**

**Can we do O(n) time AND O(1) space?** Yes — Boyer-Moore Voting.

---

## 2. The Voting Intuition

Imagine an election. The majority candidate appears more than half the time.

**Key idea:** Pair up every minority vote with a majority vote and cancel them.
After all cancellations, the majority candidate is left standing.

```
[A, B, A, A, C, A, B]
 ↑  ↑
 A wins → cancel → nothing left from this pair

 Result after all cancellations: A (majority) survives
```

More precisely:
- Keep track of one **candidate** and a **count**.
- When you see the candidate: count++
- When you see someone else: count-- (one vote cancels one counter-vote)
- When count hits 0: the current candidate is "eliminated", pick a new one

Because the majority has more than half the votes, it can never be permanently
eliminated — it will always be the last candidate standing.

---

## 3. Step-by-Step Trace

Input: `[2, 2, 1, 1, 1, 2, 2]`

```
Candidate: ?     Count: 0

Step 1: See 2 → candidate=2, count=1
┌─────────────────────────────────┐
│ candidate: 2    count: 1        │
└─────────────────────────────────┘

Step 2: See 2 → same as candidate, count=2
┌─────────────────────────────────┐
│ candidate: 2    count: 2        │
└─────────────────────────────────┘

Step 3: See 1 → different, count=1
┌─────────────────────────────────┐
│ candidate: 2    count: 1        │
└─────────────────────────────────┘

Step 4: See 1 → different, count=0
        count hits 0 → candidate wiped out
┌─────────────────────────────────┐
│ candidate: -    count: 0        │
└─────────────────────────────────┘

Step 5: See 1 → new candidate=1, count=1
┌─────────────────────────────────┐
│ candidate: 1    count: 1        │
└─────────────────────────────────┘

Step 6: See 2 → different, count=0
        candidate wiped out again
┌─────────────────────────────────┐
│ candidate: -    count: 0        │
└─────────────────────────────────┘

Step 7: See 2 → new candidate=2, count=1
┌─────────────────────────────────┐
│ candidate: 2    count: 1        │
└─────────────────────────────────┘

Final candidate: 2 ✓  (majority element)
```

---

## 4. Java Code

```java
int majorityElement(int[] nums) {
    int candidate = nums[0];
    int count = 1;

    for (int i = 1; i < nums.length; i++) {
        if (count == 0) {
            candidate = nums[i];
            count = 1;
        } else if (nums[i] == candidate) {
            count++;
        } else {
            count--;
        }
    }
    return candidate;
    // Note: the problem guarantees a majority exists.
    // If not guaranteed, verify candidate with a second pass.
}
```

**Compact version:**
```java
int majorityElement(int[] nums) {
    int candidate = 0, count = 0;
    for (int num : nums) {
        if (count == 0) candidate = num;
        count += (num == candidate) ? 1 : -1;
    }
    return candidate;
}
```

---

## 5. Why Does This Work? (The Proof)

Let the majority element be M (appears > n/2 times).
Let all other elements together appear < n/2 times.

**Claim:** M will be the final candidate.

**Proof:**
Every time M's count decreases by 1, some non-M element's count also effectively
decreases by 1 (they "cancel"). Since M appears more than all others combined,
M can cancel every non-M element AND still have votes left.

```
Cancellation budget:
  M votes:     > n/2
  non-M votes: < n/2

Even if every M vote cancels one non-M vote,
M still has leftover votes → M is the final survivor.
```

---

## 6. Verification Step (When Majority Not Guaranteed)

The algorithm finds the **candidate**, not proof that it's a majority.
If the problem doesn't guarantee a majority exists, add a verification pass:

```java
int findMajority(int[] nums) {
    // Phase 1: find candidate
    int candidate = 0, count = 0;
    for (int num : nums) {
        if (count == 0) candidate = num;
        count += (num == candidate) ? 1 : -1;
    }

    // Phase 2: verify
    int freq = 0;
    for (int num : nums) if (num == candidate) freq++;
    return (freq > nums.length / 2) ? candidate : -1;
}
```

---

## 7. Extension: Majority II — Elements Appearing > n/3 Times (LC 229)

```
At most 2 such elements can exist (since 3 elements each > n/3 would sum > n).
Use 2 candidates and 2 counters.
```

```java
List<Integer> majorityElement(int[] nums) {
    int cand1 = 0, cand2 = 1;   // different initial values
    int cnt1 = 0, cnt2 = 0;

    for (int num : nums) {
        if (num == cand1)       cnt1++;
        else if (num == cand2)  cnt2++;
        else if (cnt1 == 0)   { cand1 = num; cnt1 = 1; }
        else if (cnt2 == 0)   { cand2 = num; cnt2 = 1; }
        else                  { cnt1--; cnt2--; }
    }

    // Verify both candidates
    cnt1 = 0; cnt2 = 0;
    for (int num : nums) {
        if (num == cand1) cnt1++;
        else if (num == cand2) cnt2++;
    }

    List<Integer> result = new ArrayList<>();
    if (cnt1 > nums.length / 3) result.add(cand1);
    if (cnt2 > nums.length / 3) result.add(cand2);
    return result;
}
```

**Trace for n/3 majority:**
```
Input: [1, 1, 1, 3, 3, 2, 2, 2]

Step-by-step candidates:
 1→ cand1=1,cnt1=1
 1→ cnt1=2
 1→ cnt1=3
 3→ cand2=3,cnt2=1
 3→ cnt2=2
 2→ cnt1--, cnt2-- → cnt1=2,cnt2=1
 2→ cnt1--, cnt2-- → cnt1=1,cnt2=0
 2→ cand2=2,cnt2=1

Final candidates: 1 and 2 ✓
```

---

## 8. Generalization: Elements Appearing > n/k Times

For elements appearing > n/k times, maintain k-1 candidates.

```
k=2 → 1 candidate  (> n/2 majority)
k=3 → 2 candidates (> n/3 majority)
k=4 → 3 candidates (> n/4 majority)
```

---

## 9. Cheat Sheet

```
Boyer-Moore Voting — Core Logic:
  count == 0       → switch to current element as new candidate
  num == candidate → count++
  num != candidate → count--

Final candidate = majority element (if guaranteed to exist)
Add verification pass if majority not guaranteed.

For > n/k majority: maintain k-1 (candidate, count) pairs.
```

### Pattern Recognition: When to Use Boyer-Moore
- [ ] Find element appearing > n/2 times (O(1) space required)
- [ ] Find elements appearing > n/3 times (two candidates)
- [ ] Any "majority vote" style problem

---

## 10. Related Problems

| # | Problem | LC# | Difficulty |
|---|---------|-----|------------|
| E1 | Majority Element | 169 | E |
| M1 | Majority Element II (> n/3) | 229 | M |
| M2 | Check Array Formation | - | varies |
