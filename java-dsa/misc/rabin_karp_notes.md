# Rabin-Karp Algorithm — Deep Dive
> String Matching Using Rolling Hash
> Key technique: compute next hash in O(1) from previous hash

---

## The Problem

Same as KMP: find all occurrences of pattern P in text T.

But Rabin-Karp uses a completely different approach — **hashing**.

---

## 1. Naive Hash — Still O(nm)

Compute hash of P. Slide a window of length m over T.
Compute hash of each window, compare with hash of P.

**Problem:** Computing hash of each new window from scratch = O(m).
Total = O(nm). No better than naive.

**Solution:** Use a **rolling hash** — compute new window hash from previous in O(1).

---

## 2. The Rolling Hash Idea

Represent a string as a polynomial:

```
"abc" = a × B² + b × B¹ + c × B⁰
      = 97×100² + 98×100¹ + 99×100⁰
      (where B = base, e.g., 31 or 131)
```

When the window slides by 1:
```
Old window: [a, b, c]   hash = a*B² + b*B¹ + c
New window: [b, c, d]   hash = b*B² + c*B¹ + d

Rolling: new_hash = (old_hash - a*B^(m-1)) * B + d
```

```
Remove leftmost character: subtract a × B^(m-1)
Shift left by 1:           multiply by B
Add new rightmost char:    add d
```

This updates the hash in **O(1)** per step.

---

## 3. The Modulo Problem

Hashes grow huge (B^m can be astronomical).
Solution: work modulo a large prime M.

```
hash = ((hash - leftChar * power) * BASE + rightChar) % MOD

Where: power = B^(m-1) % MOD
```

**Collision:** Two different strings might have the same hash (mod M).
Handle with: verify actual string equality when hashes match.

---

## 4. Java Implementation

```java
public class RabinKarp {
    static final long BASE = 31;
    static final long MOD = 1_000_000_007L;

    public static List<Integer> search(String text, String pattern) {
        int n = text.length(), m = pattern.length();
        List<Integer> result = new ArrayList<>();
        if (m > n) return result;

        // Compute BASE^(m-1) % MOD
        long power = 1;
        for (int i = 0; i < m - 1; i++) power = power * BASE % MOD;

        // Hash of pattern and first window
        long patHash = 0, winHash = 0;
        for (int i = 0; i < m; i++) {
            patHash = (patHash * BASE + (pattern.charAt(i) - 'a' + 1)) % MOD;
            winHash = (winHash * BASE + (text.charAt(i) - 'a' + 1)) % MOD;
        }

        // Slide the window
        for (int i = 0; i <= n - m; i++) {
            if (winHash == patHash) {
                // Verify to avoid hash collision
                if (text.substring(i, i + m).equals(pattern)) {
                    result.add(i);
                }
            }
            // Roll the hash
            if (i < n - m) {
                winHash = (winHash - (text.charAt(i) - 'a' + 1) * power % MOD + MOD) % MOD;
                winHash = (winHash * BASE + (text.charAt(i + m) - 'a' + 1)) % MOD;
            }
        }
        return result;
    }
}
```

---

## 5. Worked Example

Text: `"abcabc"`, Pattern: `"abc"`, BASE=10, MOD=10007

```
Pattern hash: a*100 + b*10 + c = 97*100 + 98*10 + 99 = 10679  (mod 10007 = 672)
power = BASE^(m-1) = 10^2 = 100

Initial window "abc": hash = 672

i=0: winHash(672) == patHash(672) → verify "abc"=="abc" ✓ → result=[0]
     Roll: remove 'a'(97), shift, add 'd'(100)
     winHash = (672 - 97*100 % 10007 + 10007) % 10007 * 10 + 100) % 10007
     winHash = hash of "bca"

i=1: "bca" hash ≠ 672 → skip. Roll → hash of "cab"

i=2: "cab" hash ≠ 672 → skip. Roll → hash of "abc"

i=3: winHash == patHash → verify "abc"=="abc" ✓ → result=[0, 3]

Final: [0, 3] ✓
```

---

## 6. Rolling Hash Formula Visualized

```
Window shift example (m=3, BASE=10):

Old: [a][b][c]    hash = a*100 + b*10 + c
New: [b][c][d]    hash = b*100 + c*10 + d

Step 1: Remove 'a'
  hash - a * BASE^(m-1) = a*100 + b*10 + c - a*100 = b*10 + c

Step 2: Shift left (multiply by BASE)
  (b*10 + c) * 10 = b*100 + c*10

Step 3: Add 'd'
  b*100 + c*10 + d   ← new hash ✓

Formula: new_hash = (old_hash - left_char * power) * BASE + right_char
```

---

## 7. Time Complexity

```
Average: O(n + m)  — hash comparisons O(1), string verify rarely triggered
Worst:   O(nm)     — if all hashes collide (pathological input)

In practice: use a large prime MOD and/or double hashing to minimize collisions.
```

**Double Hashing:** Use two different (BASE, MOD) pairs. Collision probability drops to ~1/MOD².

---

## 8. KMP vs Rabin-Karp

```
KMP:
  ✓ Worst case O(n+m) guaranteed
  ✓ No false positives (exact matching)
  ✗ Complex to implement (LPS table)

Rabin-Karp:
  ✓ Simple rolling hash idea
  ✓ Easily extended to 2D (matrix matching)
  ✓ Easily extended to multiple patterns simultaneously
  ✗ Average O(n+m) but worst case O(nm)
  ✗ Needs collision handling
```

---

## 9. Rabin-Karp Killer Application: Multiple Pattern Search

Find any of k patterns in text T:

```java
// Build hash set of all pattern hashes
Set<Long> patternHashes = new HashSet<>();
for (String p : patterns) patternHashes.add(computeHash(p));

// Slide window over text
// O(n) slide + O(1) hash lookup → O(n + total_pattern_length)
// KMP would need Aho-Corasick for this
```

---

## 10. Application: Longest Duplicate Substring (LC 1044)

```java
// Binary search on length + Rabin-Karp to check if any substring of that length repeats
String longestDupSubstring(String s) {
    int lo = 1, hi = s.length() - 1;
    String result = "";
    while (lo <= hi) {
        int mid = (lo + hi) / 2;
        String dup = check(s, mid);  // Rabin-Karp: any duplicate of length mid?
        if (dup != null) { result = dup; lo = mid + 1; }
        else              { hi = mid - 1; }
    }
    return result;
}

String check(String s, int len) {
    long hash = 0, power = 1;
    long BASE = 31, MOD = 1_000_000_007L;
    Map<Long, List<Integer>> seen = new HashMap<>();

    for (int i = 0; i < len; i++) {
        hash = (hash * BASE + (s.charAt(i) - 'a' + 1)) % MOD;
        if (i > 0) power = power * BASE % MOD;
    }
    seen.computeIfAbsent(hash, k -> new ArrayList<>()).add(0);

    for (int i = 1; i <= s.length() - len; i++) {
        hash = (hash - (s.charAt(i - 1) - 'a' + 1) * power % MOD + MOD) % MOD;
        hash = (hash * BASE + (s.charAt(i + len - 1) - 'a' + 1)) % MOD;
        List<Integer> positions = seen.get(hash);
        if (positions != null) {
            String curr = s.substring(i, i + len);
            for (int pos : positions) {
                if (s.substring(pos, pos + len).equals(curr)) return curr;
            }
        }
        seen.computeIfAbsent(hash, k -> new ArrayList<>()).add(i);
    }
    return null;
}
```

---

## 11. Cheat Sheet

```
Rabin-Karp Rolling Hash:

hash = (char1 * BASE^(m-1) + char2 * BASE^(m-2) + ... + charm) % MOD

Roll:
  new_hash = ((old_hash - leftChar * power) * BASE + rightChar + MOD) % MOD
  where power = BASE^(m-1) % MOD

Note: + MOD before % to handle negative values in Java
```

### Pattern Recognition: When to Use Rabin-Karp
- [ ] Pattern matching (simpler than KMP when approximate is ok)
- [ ] Binary search on string length (substring existence check)
- [ ] Match multiple patterns simultaneously
- [ ] 2D pattern matching in a grid

---

## 12. Related Problems

| # | Problem | LC# | Difficulty |
|---|---------|-----|------------|
| M1 | Find the Index of the First Occurrence | 28 | E/M |
| H1 | Longest Duplicate Substring | 1044 | H |
| H2 | Longest Repeating Substring | 1062 | M |
| M2 | Repeated DNA Sequences | 187 | M |
