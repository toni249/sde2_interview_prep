# KMP Algorithm — Deep Dive
> Pattern Matching in O(n + m) instead of O(n × m)
> One of the most important "non-obvious" algorithms in FAANG interviews

---

## The Problem

Given a **text** T of length n and a **pattern** P of length m,
find all starting indices in T where P occurs.

```
Text:    A B C A B C A B D
Pattern: A B C A B D
                 ↑
           Found at index 3
```

---

## 1. Naive Approach — Why It's Slow

For each position i in text, try matching P starting at i.
If mismatch at some position j, restart: move i forward by 1, reset j to 0.

```
Text:    A B C A B C A B D
Pattern: A B C A B D
         ✓ ✓ ✓ ✓ ✓ ✗  ← mismatch at j=5

Move i forward by 1, restart from j=0:
Text:    A B C A B C A B D
Pattern:   A B C A B D
           ✗            ← mismatch immediately

Move i forward again...
```

**Time: O(n × m)** — for each of n positions, try up to m comparisons.

**The waste:** After matching "ABCAB", we already know the text has "ABCAB" there.
When we restart, we're re-examining characters we already know.

---

## 2. The KMP Insight

> **Don't throw away what you already matched.**

After a mismatch, instead of restarting from j=0, jump back to the longest
position where the pattern can still match based on what we've already seen.

**Key question:** After matching the first j characters of P, where can we
restart in P without missing a match?

**Answer:** Find the longest proper prefix of P[0..j-1] that is also a suffix.
That prefix will already be matched in the text, so we can restart there.

```
Matched so far: A B C A B
                ↑       ↑
                prefix   suffix

"AB" appears at both the start and end.
So we can resume matching from j=2 instead of j=0.
```

This is the **LPS array** (Longest Proper Prefix which is also a Suffix).

---

## 3. The LPS Array (Failure Function)

For each position i in pattern P, `lps[i]` = length of the longest proper
prefix of P[0..i] that is also a suffix.

**"Proper"** means the prefix/suffix cannot be the entire string itself.

### Building LPS — Step by Step

Pattern: `A B C A B D`

```
Index:   0  1  2  3  4  5
Char:    A  B  C  A  B  D
LPS:     0  0  0  1  2  0
```

Let's trace why:

```
i=0: "A"      → no proper prefix/suffix possible → lps[0] = 0

i=1: "AB"     → prefixes: "A"  | suffixes: "B"  → no match → lps[1] = 0

i=2: "ABC"    → prefixes: "A","AB" | suffixes: "C","BC" → no match → lps[2] = 0

i=3: "ABCA"   → prefixes: "A","AB","ABC" | suffixes: "A","CA","BCA"
               → "A" matches! → lps[3] = 1

i=4: "ABCAB"  → prefixes: "A","AB","ABC","ABCA" | suffixes: "B","AB","CAB","BCAB"
               → "AB" matches! → lps[4] = 2

i=5: "ABCABD" → prefixes: "A","AB","ABC","ABCA","ABCAB"
               → suffixes: "D","BD","ABD","CABD","BCABD"
               → no match → lps[5] = 0
```

### LPS Table for Common Patterns

```
Pattern: "AAAA"     → lps = [0, 1, 2, 3]
Pattern: "ABAB"     → lps = [0, 0, 1, 2]
Pattern: "AABAABAAB"→ lps = [0, 1, 0, 1, 2, 3, 4, 5, 6]
Pattern: "ABCDE"    → lps = [0, 0, 0, 0, 0]  (no overlap)
```

---

## 4. Building the LPS Array in Java

```java
int[] buildLPS(String pattern) {
    int m = pattern.length();
    int[] lps = new int[m];
    lps[0] = 0;   // always 0

    int len = 0;  // length of previous longest prefix-suffix
    int i = 1;

    while (i < m) {
        if (pattern.charAt(i) == pattern.charAt(len)) {
            // Extension: current char extends the prefix-suffix
            len++;
            lps[i] = len;
            i++;
        } else {
            if (len != 0) {
                // IMPORTANT: don't increment i
                // Fall back using lps itself — key insight!
                len = lps[len - 1];
            } else {
                lps[i] = 0;
                i++;
            }
        }
    }
    return lps;
}
```

### Tracing `buildLPS("ABCABD")`

```
lps = [0, 0, 0, 0, 0, 0]
len = 0, i = 1

i=1: P[1]='B' vs P[len=0]='A' → mismatch, len=0 → lps[1]=0, i=2
i=2: P[2]='C' vs P[0]='A'    → mismatch, len=0 → lps[2]=0, i=3
i=3: P[3]='A' vs P[0]='A'    → MATCH → len=1, lps[3]=1, i=4
i=4: P[4]='B' vs P[1]='B'    → MATCH → len=2, lps[4]=2, i=5
i=5: P[5]='D' vs P[2]='C'    → mismatch, len=2 (not 0)
     → len = lps[len-1] = lps[1] = 0  (fall back!)
     → P[5]='D' vs P[0]='A'  → mismatch, len=0 → lps[5]=0, i=6

Result: lps = [0, 0, 0, 1, 2, 0]  ✓
```

**Why `len = lps[len-1]` when falling back?**

```
We had matched: P[0..len-1] at some position.
lps[len-1] tells us: within P[0..len-1], the longest
prefix that is also a suffix.
So we can "reuse" that prefix — we don't need to go all the way back to 0.
This is the CORE insight of KMP.
```

---

## 5. The Search Algorithm

```java
List<Integer> kmpSearch(String text, String pattern) {
    int n = text.length(), m = pattern.length();
    int[] lps = buildLPS(pattern);
    List<Integer> result = new ArrayList<>();

    int i = 0;  // index in text
    int j = 0;  // index in pattern

    while (i < n) {
        if (text.charAt(i) == pattern.charAt(j)) {
            i++;
            j++;
        }

        if (j == m) {
            // Found a match ending at i-1
            result.add(i - j);
            j = lps[j - 1];   // look for next match
        } else if (i < n && text.charAt(i) != pattern.charAt(j)) {
            if (j != 0) {
                j = lps[j - 1];   // fall back in pattern, don't move i
            } else {
                i++;              // no prefix to fall back to, move text
            }
        }
    }
    return result;
}
```

---

## 6. Full Worked Example

Text: `A B C A B C A B D`
Pattern: `A B C A B D`
LPS: `[0, 0, 0, 1, 2, 0]`

```
Step-by-step:

i=0,j=0: T[0]='A'==P[0]='A' → i=1,j=1
i=1,j=1: T[1]='B'==P[1]='B' → i=2,j=2
i=2,j=2: T[2]='C'==P[2]='C' → i=3,j=3
i=3,j=3: T[3]='A'==P[3]='A' → i=4,j=4
i=4,j=4: T[4]='B'==P[4]='B' → i=5,j=5
i=5,j=5: T[5]='C'!=P[5]='D' → MISMATCH, j=lps[4]=2 (don't move i!)

  [We already know T[3..4]="AB" = P[0..1], skip re-checking]

i=5,j=2: T[5]='C'==P[2]='C' → i=6,j=3
i=6,j=3: T[6]='A'==P[3]='A' → i=7,j=4
i=7,j=4: T[7]='B'==P[4]='B' → i=8,j=5
i=8,j=5: T[8]='D'==P[5]='D' → i=9,j=6

j==m=6 → MATCH FOUND at index i-j = 9-6 = 3 ✓
j = lps[5] = 0, continue...
i=9 → loop ends.

Result: [3]
```

---

## 7. Complexity

```
Time:   O(n + m)
        - buildLPS: O(m)
        - search:   O(n)
        Note: i never decreases, j never goes above m.
              Total steps bounded by 2n + 2m.

Space:  O(m) for the LPS array
```

Compare: Naive = O(n × m). KMP is always O(n + m).

---

## 8. Complete Java Implementation

```java
public class KMP {
    public static int[] buildLPS(String pattern) {
        int m = pattern.length();
        int[] lps = new int[m];
        int len = 0, i = 1;
        while (i < m) {
            if (pattern.charAt(i) == pattern.charAt(len)) {
                lps[i++] = ++len;
            } else if (len != 0) {
                len = lps[len - 1];
            } else {
                lps[i++] = 0;
            }
        }
        return lps;
    }

    public static List<Integer> search(String text, String pattern) {
        int n = text.length(), m = pattern.length();
        int[] lps = buildLPS(pattern);
        List<Integer> result = new ArrayList<>();
        int i = 0, j = 0;
        while (i < n) {
            if (text.charAt(i) == pattern.charAt(j)) { i++; j++; }
            if (j == m) {
                result.add(i - j);
                j = lps[j - 1];
            } else if (i < n && text.charAt(i) != pattern.charAt(j)) {
                j = (j != 0) ? lps[j - 1] : 0;
                if (j == 0) i++;
            }
        }
        return result;
    }
}
```

---

## 9. FAANG Applications of KMP

### Application 1: Repeated Substring Pattern (LC 459)
```java
// Is string S a repetition of some pattern?
// Key trick: S + S contains S at an index other than 0 and n.
boolean repeatedSubstringPattern(String s) {
    String doubled = s + s;
    // Search for s in doubled[1..2n-2]
    List<Integer> matches = KMP.search(doubled.substring(1, doubled.length() - 1), s);
    return !matches.isEmpty();
}
```

### Application 2: Shortest Palindrome (LC 214)
```java
// Find shortest palindrome by prepending chars to s.
// Reverse of s: r = reverse(s)
// Find longest prefix of s that is a palindrome:
// → find LPS of (s + "#" + r)
// The LPS of the last character tells us how much of s is already a palindrome.
String shortestPalindrome(String s) {
    String r = new StringBuilder(s).reverse().toString();
    String combined = s + "#" + r;
    int[] lps = KMP.buildLPS(combined);
    int longest = lps[combined.length() - 1];   // longest palindromic prefix length
    return r.substring(0, s.length() - longest) + s;
}
```

### Application 3: String Rotation Check
```java
// Is t a rotation of s?  (e.g., "abcde" rotated = "cdeab")
// If t is a rotation, then t appears in s+s.
boolean isRotation(String s, String t) {
    if (s.length() != t.length()) return false;
    return !KMP.search(s + s, t).isEmpty();
}
```

---

## 10. Cheat Sheet

```
KMP in 3 steps:
1. Build LPS array for pattern            → O(m)
2. Search using LPS to avoid backtracking → O(n)
3. Total: O(n + m), Space: O(m)

LPS[i] = length of longest proper prefix of P[0..i]
         that is also a suffix.

On mismatch at j:
  if j > 0:  j = lps[j-1]   (fall back, don't move i)
  if j == 0: i++             (no prefix to reuse, advance text)

When j == m: match found at (i - m), then j = lps[m-1]
```

### Pattern Recognition: When to Use KMP
- [ ] Find all occurrences of a pattern in text
- [ ] Check if string B is a rotation of A (search B in A+A)
- [ ] Shortest palindrome by prepending
- [ ] Repeated substring pattern
- [ ] Any problem involving prefix = suffix overlap in strings

---

## 11. Related Problems

| # | Problem | LC# | Key Insight |
|---|---------|-----|-------------|
| E1 | Implement strStr() | 28 | Direct KMP application |
| M1 | Repeated Substring Pattern | 459 | KMP on s+s |
| H1 | Shortest Palindrome | 214 | LPS of s+"#"+reverse(s) |
| M2 | Find All Anagrams in String | 438 | Sliding window (not KMP) |
| M3 | String Rotation | - | Search in s+s |

---

*Practice building the LPS array by hand before coding. The insight of `len = lps[len-1]` is the hardest part — trace it a few times.*
