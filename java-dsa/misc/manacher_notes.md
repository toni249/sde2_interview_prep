# Manacher's Algorithm — Deep Dive
> Find Longest Palindromic Substring in O(n)
> Converts O(n²) expand-around-center to O(n) using already-computed info

---

## The Problem

Given a string s, find the longest palindromic substring.

```
Input:  "babad"
Output: "bab"  (or "aba")

Input:  "cbbd"
Output: "bb"
```

---

## 1. Naive: Expand Around Center — O(n²)

For each character (and gap between characters), expand outward while characters match.

```java
// Two types of palindromes:
// Odd:  "aba"  → center is a character
// Even: "abba" → center is between two characters

String longestPalindrome(String s) {
    int start = 0, maxLen = 1;
    for (int center = 0; center < s.length(); center++) {
        // Odd length
        int len1 = expand(s, center, center);
        // Even length
        int len2 = expand(s, center, center + 1);
        int len = Math.max(len1, len2);
        if (len > maxLen) {
            maxLen = len;
            start = center - (len - 1) / 2;
        }
    }
    return s.substring(start, start + maxLen);
}

int expand(String s, int left, int right) {
    while (left >= 0 && right < s.length() && s.charAt(left) == s.charAt(right)) {
        left--; right++;
    }
    return right - left - 1;
}
```

**Time: O(n²)** — for each center, expansion can go up to O(n).

**Can we do O(n)?** Yes — Manacher's.

---

## 2. The Manacher Insight

When expanding from center i, can we reuse information from previous centers?

**Yes!** If center i falls inside a previously found palindrome (with right boundary R),
then palindrome around i is at least as large as its mirror across the previous center C.

```
Previously found palindrome centered at C, reaching right boundary R:

    L           C           R
    |           |           |
    ←─────── 2*P[C] ──────→

    Mirror of i across C:
    i_mirror = 2*C - i

          i_mirror    C       i
              |        |       |
              ←─P[m]──→       ↑
                              └── P[i] ≥ min(P[i_mirror], R - i)
```

Three cases:
1. i_mirror's palindrome is strictly inside [L, R] → P[i] = P[i_mirror]
2. i_mirror's palindrome extends to/beyond L → P[i] = at least R - i, then expand
3. i_mirror's palindrome exactly touches L → P[i] = R - i, then try to expand

This avoids re-checking characters we already processed.

---

## 3. The "#" Trick — Unify Odd and Even

To handle both odd and even palindromes uniformly, insert `#` between every character:

```
Original: a b b a
           0 1 2 3

With #:   # a # b # b # a #
           0 1 2 3 4 5 6 7 8

Now ALL palindromes have odd length (centered on a character or #).
```

In the transformed string T of length 2n+1:
- Characters at even indices are `#` (act as gaps)
- Characters at odd indices are the original characters

The palindrome radius P[i] in T corresponds to actual substring length = P[i] in original.

---

## 4. The Full Algorithm

```java
String longestPalindrome(String s) {
    // Step 1: Transform s into T with # separators
    StringBuilder sb = new StringBuilder("#");
    for (char c : s.toCharArray()) { sb.append(c); sb.append('#'); }
    String T = sb.toString();
    int n = T.length();

    int[] P = new int[n];   // P[i] = palindrome radius at i (not counting center)
    int C = 0, R = 0;       // center and right boundary of rightmost palindrome

    for (int i = 0; i < n; i++) {
        int mirror = 2 * C - i;   // mirror of i across C

        if (i < R) {
            // i is inside current rightmost palindrome
            P[i] = Math.min(R - i, P[mirror]);
        }
        // else P[i] = 0 (already initialized)

        // Attempt to expand around i
        int left = i - P[i] - 1;
        int right = i + P[i] + 1;
        while (left >= 0 && right < n && T.charAt(left) == T.charAt(right)) {
            P[i]++;
            left--;
            right++;
        }

        // Update rightmost palindrome if we expanded past R
        if (i + P[i] > R) {
            C = i;
            R = i + P[i];
        }
    }

    // Find the maximum palindrome radius and its center
    int maxLen = 0, centerIdx = 0;
    for (int i = 0; i < n; i++) {
        if (P[i] > maxLen) {
            maxLen = P[i];
            centerIdx = i;
        }
    }

    // Convert back to original string indices
    int start = (centerIdx - maxLen) / 2;
    return s.substring(start, start + maxLen);
}
```

---

## 5. Worked Example

Input: `"abba"`
Transformed T: `"#a#b#b#a#"`

```
Index:  0  1  2  3  4  5  6  7  8
Char:   #  a  #  b  #  b  #  a  #
```

**Computing P:**

```
i=0: T[0]='#'. No mirror (i>=R=0). Expand: left=-1 → stop. P[0]=0.

i=1: T[1]='a'. No mirror. Expand: T[0]='#'==T[2]='#'? Yes → P[1]=1.
     T[-1]? No → stop. P[1]=1. R=2, C=1.

i=2: T[2]='#'. i<R=2? No (i==R). Expand: T[1]='a'==T[3]='b'? No. P[2]=0.

i=3: T[3]='b'. i<R=2? No. Expand: T[2]='#'==T[4]='#'? Yes → P[3]=1.
     T[1]='a'==T[5]='b'? No → stop. P[3]=1. R=4, C=3.

i=4: T[4]='#'. i<R=4? No (i==R). Expand: T[3]='b'==T[5]='b'? Yes → P[4]=1.
     T[2]='#'==T[6]='#'? Yes → P[4]=2.
     T[1]='a'==T[7]='a'? Yes → P[4]=3.
     T[0]='#'==T[8]='#'? Yes → P[4]=4.
     T[-1]? No → stop. P[4]=4. R=8, C=4.

i=5: T[5]='b'. i<R=8? Yes. mirror=2*4-5=3. P[mirror]=P[3]=1.
     P[i] = min(R-i, P[mirror]) = min(8-5, 1) = min(3,1) = 1.
     Try expand: T[5-1-1]=T[3]='b', T[5+1+1]=T[7]='a'? No → stop.
     P[5]=1.

i=6: T[6]='#'. i<R=8. mirror=2*4-6=2. P[2]=0.
     P[i]=min(8-6,0)=0. Expand: T[5]='b'==T[7]='a'? No. P[6]=0.

i=7: T[7]='a'. i<R=8. mirror=2*4-7=1. P[1]=1.
     P[i]=min(8-7,1)=min(1,1)=1. Expand: T[5]='b'==T[9]? Out of bounds. P[7]=1.

i=8: T[8]='#'. i>=R=8. Expand: T[7]='a'? left=-1? T[-1]? No. P[8]=0.

P = [0, 1, 0, 1, 4, 1, 0, 1, 0]
```

Maximum P[i] = 4 at i=4 (center '#' between the two b's).
`maxLen = 4`, `start = (4 - 4) / 2 = 0`.
Result: `s.substring(0, 0 + 4) = "abba"` ✓

---

## 6. Complexity

```
Time:   O(n)
        - Each character in T is processed at most twice (once normally, once during expansion)
        - The right boundary R only ever moves right → total expansions = O(n)

Space:  O(n) for the P array and transformed string T
```

Compare: naive expand-around-center = O(n²). Manacher = O(n).

---

## 7. Cheat Sheet

```
Manacher's in 4 steps:

1. Transform:  "abc" → "#a#b#c#"  (unified odd/even)

2. P[i] = palindrome radius at center i (in transformed string)

3. For each i:
   - If i < R: P[i] = min(R-i, P[mirror])  (use mirror info)
   - Expand from current P[i]
   - Update (C, R) if we expanded past R

4. Answer:
   maxLen = max(P[i])
   start  = (centerIdx - maxLen) / 2  (back to original)

Key variables:
  C = center of rightmost palindrome so far
  R = right boundary of that palindrome (exclusive)
  mirror = 2*C - i
```

### Pattern Recognition: When to Use Manacher's
- [ ] Longest palindromic substring in O(n)
- [ ] Count all palindromic substrings (sum all P[i])
- [ ] Any problem where you need palindrome radii for all positions

---

## 8. Counting All Palindromic Substrings

Using Manacher's P array, total palindromes = sum of ceil(P[i]/1 + 1) for each center.

Simpler: count palindromes = `sum of (P[i] + 1) / 2` across all centers (roughly).

More precisely:
```java
// After computing P for transformed string:
int countPalindromes(String s) {
    // P[i] for transformed T
    // Each P[i] in T corresponds to (P[i]+1)/2 palindromes in original
    // But it's easier to just use expand-around-center for counting.
    int count = 0;
    for (int i = 0; i < n; i++) count += (P[i] + 1) / 2;
    // Wait, need to subtract # centers
    // Simpler: for each odd center: P[odd_i]/2 palindromes
    //          for each even center (#): P[even_i]/2 palindromes
    return count;
}
// For LC 647, expand-around-center is cleaner for counting.
```

---

## 9. Related Problems

| # | Problem | LC# | Difficulty |
|---|---------|-----|------------|
| M1 | Longest Palindromic Substring | 5 | M |
| M2 | Palindromic Substrings (count) | 647 | M |
| H1 | Shortest Palindrome (KMP approach) | 214 | H |
| M3 | Longest Palindromic Subsequence | 516 | M (DP, not Manacher) |

*For most FAANG interviews, the O(n²) expand-around-center is acceptable.
Manacher's is asked in specialized or competitive-programming contexts.*
