# Strings — Pattern Deep Dive
> FAANG Level | Java | Easy → Hard

---

## Pattern Map

```
STRINGS
├── P1: Two Pointers          → palindrome, reverse words, compare
├── P2: Sliding Window        → min window, longest without repeat, anagram
├── P3: Hashing / Freq Map    → anagram check, permutation, group anagrams
├── P4: Stack-based           → valid parentheses, decode string, calculator
├── P5: KMP / Z-algorithm     → pattern matching, find all occurrences
└── P6: Trie-based            → prefix search, word dictionary (see Tries topic)
```

---

## P1: Two Pointers on Strings

### Core Idea
Same as arrays — use two indices moving toward each other or same direction.

### Variation 1: Palindrome Check (LC 125)

**Problem:** Given a string `s`, return `true` if it reads the same forward and backward after considering only alphanumeric characters and ignoring case.

**Approach (Two pointers, in-place):**
- Place pointers `l = 0`, `r = n-1` and walk them toward each other.
- Skip non-alphanumeric characters on each side (FAANG trap — punctuation/spaces).
- Compare characters after lowercasing; mismatch → `false`.
- This avoids building a cleaned string (saves O(n) extra space).
- Time: O(n). Space: O(1).

```java
boolean isPalindrome(String s) {
    int left = 0, right = s.length() - 1;
    while (left < right) {
        while (left < right && !Character.isAlphanumeric(s.charAt(left)))  left++;
        while (left < right && !Character.isAlphanumeric(s.charAt(right))) right--;
        if (Character.toLowerCase(s.charAt(left)) != Character.toLowerCase(s.charAt(right)))
            return false;
        left++; right--;
    }
    return true;
}
```
> FAANG trap: "Valid Palindrome" — skip non-alphanumeric chars.

### Variation 2: Palindrome with One Deletion Allowed (LC 680)

**Problem:** Given a string `s`, return `true` if it can become a palindrome by deleting **at most one** character.

**Approach (Two pointers + greedy branch):**
- Walk pointers `l`, `r` inward; while characters match, keep going.
- On the **first mismatch**, the only legal moves are to drop `s[l]` *or* `s[r]`. Recurse/verify both substrings `(l+1, r)` and `(l, r-1)` as plain palindromes.
- If either is a palindrome → answer is `true`. Otherwise `false`.
- Key insight: we only branch once — after the first mismatch we've already "used" our one allowed deletion.
- Time: O(n). Space: O(1).

```java
// LC 680: Valid Palindrome II
boolean validPalindrome(String s) {
    int l = 0, r = s.length() - 1;
    while (l < r) {
        if (s.charAt(l) != s.charAt(r))
            return isPalin(s, l+1, r) || isPalin(s, l, r-1);
        l++; r--;
    }
    return true;
}
boolean isPalin(String s, int l, int r) {
    while (l < r) {
        if (s.charAt(l++) != s.charAt(r--)) return false;
    }
    return true;
}
```

### Variation 3: Reverse Words in String (LC 151)

**Problem:** Given a string `s`, reverse the order of **words** (space-separated tokens). Trim leading/trailing spaces and collapse multiple inner spaces to a single space.

**Approach (Split + reverse join):**
- `trim()` removes outer whitespace; `split("\\s+")` collapses any internal run of whitespace and tokenizes.
- Walk the resulting array right-to-left, appending each word with a single space separator.
- Time: O(n). Space: O(n) for the split array.
- In-place variant (FAANG follow-up): reverse the entire char array, then reverse each word in place — O(1) extra space.

```java
// Trim, split on spaces, reverse order
String reverseWords(String s) {
    String[] words = s.trim().split("\\s+");
    StringBuilder sb = new StringBuilder();
    for (int i = words.length - 1; i >= 0; i--) {
        sb.append(words[i]);
        if (i != 0) sb.append(" ");
    }
    return sb.toString();
}
```

### Questions

| # | Problem | LC# |
|---|---------|-----|
| E1 | Valid Palindrome | 125 |
| E2 | Reverse String | 344 |
| M1 | Valid Palindrome II (one deletion) | 680 |
| M2 | Reverse Words in a String | 151 |
| M3 | Longest Palindromic Substring (expand around center) | 5 |
| H1 | Shortest Palindrome (KMP) | 214 |

### Longest Palindromic Substring — Expand Around Center (LC 5)

**Problem:** Given a string `s`, return the **longest substring** that is a palindrome.

**Approach (Expand around center):**
- Every palindrome has a center. There are `2n - 1` possible centers: `n` single-character centers (odd length) and `n - 1` between-character centers (even length).
- For each center, expand two pointers outward while characters match; the final span length is the longest palindrome centered there.
- Track the global maximum and recompute `start = i - (len - 1) / 2` to map center back to substring start.
- Time: O(n²). Space: O(1).
- Alternatives: DP `dp[i][j]` is palindrome → O(n²) time + O(n²) space; Manacher's algorithm is O(n) but rarely required in interviews.

```java
// For each center (n chars + n-1 between-chars), expand outward
String longestPalindrome(String s) {
    int start = 0, maxLen = 0;
    for (int i = 0; i < s.length(); i++) {
        // Odd length palindromes
        int len1 = expand(s, i, i);
        // Even length palindromes
        int len2 = expand(s, i, i + 1);
        int len = Math.max(len1, len2);
        if (len > maxLen) {
            maxLen = len;
            start = i - (len - 1) / 2;
        }
    }
    return s.substring(start, start + maxLen);
}
int expand(String s, int l, int r) {
    while (l >= 0 && r < s.length() && s.charAt(l) == s.charAt(r)) { l--; r++; }
    return r - l - 1;
}
```

---

## P2: Sliding Window on Strings

### Variation 1: Longest Substring Without Repeating Characters (LC 3)

**Problem:** Given a string `s`, return the length of the **longest substring** that contains no repeated characters.

**Approach (Variable-size sliding window + last-seen map):**
- Maintain a window `[left, right]` that always contains unique characters.
- For each new character `s[right]`, if we've seen it inside the current window (`lastSeen[c] >= left`), **jump** `left` past that prior occurrence — this dodges the inner shrink loop and gives O(n) with one pass.
- Update `lastSeen[c] = right`, then refresh `maxLen` with `right - left + 1`.
- Edge case: if the previous occurrence is *outside* the window (`< left`), do **not** move `left` — that's a stale entry.
- Time: O(n). Space: O(min(n, alphabet)).

```java
// Variable window — shrink when a char repeats
int lengthOfLongestSubstring(String s) {
    Map<Character, Integer> lastSeen = new HashMap<>();
    int left = 0, maxLen = 0;
    for (int right = 0; right < s.length(); right++) {
        char c = s.charAt(right);
        if (lastSeen.containsKey(c) && lastSeen.get(c) >= left) {
            left = lastSeen.get(c) + 1;   // jump left past the duplicate
        }
        lastSeen.put(c, right);
        maxLen = Math.max(maxLen, right - left + 1);
    }
    return maxLen;
}
```
> Key: `lastSeen.get(c) >= left` — only jump if the repeat is inside the window.

### Variation 2: Minimum Window Substring (LC 76) — Classic Hard

**Problem:** Given strings `s` and `t`, return the **smallest** substring of `s` that contains every character of `t` (with at least the required multiplicities). Return `""` if no such window exists.

**Approach (Variable-size sliding window with `have`/`need` counter):**
- Build a `need[]` frequency array from `t`. We want the window's frequencies to "cover" `need[]`.
- Maintain `have` = number of *characters of `t`* currently fully satisfied by the window. Window is valid when `have == total = t.length()`.
- Expand `right`: decrement `need[c]`; if `need[c]` was `> 0` before decrementing, increment `have` (we used a still-needed character).
- When valid, **shrink** from `left` as long as it stays valid; record the smallest window each time we're about to break validity.
- Trick: increment `have` only when `need[c] > 0` *before* decrement — this naturally handles duplicates in `t` (e.g., `t = "aab"`).
- Time: O(|s| + |t|). Space: O(128) for the freq array.

```java
// Find smallest window in s containing all chars of t
String minWindow(String s, String t) {
    int[] need = new int[128];
    for (char c : t.toCharArray()) need[c]++;
    int have = 0, total = t.length();
    int left = 0, minLen = Integer.MAX_VALUE, start = 0;

    for (int right = 0; right < s.length(); right++) {
        char c = s.charAt(right);
        if (need[c] > 0) have++;    // this char was still needed
        need[c]--;

        while (have == total) {     // window is valid — try to shrink
            if (right - left + 1 < minLen) {
                minLen = right - left + 1;
                start = left;
            }
            char lc = s.charAt(left++);
            need[lc]++;
            if (need[lc] > 0) have--;   // window became invalid
        }
    }
    return minLen == Integer.MAX_VALUE ? "" : s.substring(start, start + minLen);
}
```

### Variation 3: Permutation in String (LC 567) — Fixed Window

**Problem:** Given strings `s1` and `s2`, return `true` if `s2` contains a **permutation** of `s1` as a contiguous substring.

**Approach (Fixed-size sliding window over 26-letter frequency):**
- "Permutation of `s1`" = any anagram = same character multiset. So we want a window of size `k = s1.length()` whose multiset equals `s1`'s.
- Initialize `freq[26]` with `s1`'s counts. As we slide the window over `s2`, **subtract** entering chars and **add back** leaving chars; the window contains a permutation when all 26 counts are zero.
- Optimization (avoids 26-element scan per step): keep a `matches` counter — number of letters whose `freq` slot is exactly 0. Maintain it incrementally; window matches when `matches == 26`. O(n) total.
- Time: O(|s1| + |s2|). Space: O(26).

```java
// LC 567: Does s2 contain a permutation of s1?
boolean checkInclusion(String s1, String s2) {
    int[] freq = new int[26];
    for (char c : s1.toCharArray()) freq[c - 'a']++;
    int k = s1.length(), matches = 0;
    // Count how many characters are fully matched
    for (int i = 0; i < 26; i++) if (freq[i] == 0) matches++;

    for (int right = 0; right < s2.length(); right++) {
        int c = s2.charAt(right) - 'a';
        freq[c]--;
        if (freq[c] == 0) matches++;
        if (right >= k) {
            int lc = s2.charAt(right - k) - 'a';
            if (freq[lc] == 0) matches--;
            freq[lc]++;
        }
        if (matches == 26) return true;
    }
    return false;
}
```

### Variation 4: Find All Anagrams in String (LC 438)

**Problem:** Given strings `s` and `p`, return **all starting indices** in `s` where a substring is an anagram of `p`.

**Approach (Fixed-size sliding window — same engine as LC 567):**
- Identical sliding-window logic to "Permutation in String"; instead of returning early on the first match, record `left` (or `right - k + 1`) into the result list every time the window equals `p`'s multiset.
- Use the `matches == 26` shortcut to detect anagram windows in O(1) per step.
- Time: O(|s| + |p|). Space: O(26) plus the answer.

### Questions

| # | Problem | LC# |
|---|---------|-----|
| M1 | Longest Substring Without Repeating Characters | 3 |
| M2 | Permutation in String | 567 |
| M3 | Find All Anagrams in a String | 438 |
| M4 | Longest Repeating Character Replacement | 424 |
| M5 | Max Consecutive Ones III (with k flips) | 1004 |
| H1 | Minimum Window Substring | 76 |
| H2 | Substring with Concatenation of All Words | 30 |

### Longest Repeating Character Replacement (LC 424) — FAANG Favorite

**Problem:** Given a string `s` (uppercase letters) and integer `k`, you may replace at most `k` characters with any letter. Return the length of the **longest substring** of identical characters achievable.

**Approach (Sliding window + max-frequency invariant):**
- For any window, the minimum replacements needed = `windowSize - maxFreq` (keep the most-frequent letter, replace the rest).
- Window is valid when `windowSize - maxFreq <= k`.
- Expand `right`, update `freq[c]++` and `maxFreq = max(maxFreq, freq[c])`. If window becomes invalid, slide `left` once (no inner while loop — we never need to *shrink* below the best size achieved so far).
- **Clever trick:** we **don't decrement `maxFreq`** when shrinking. It's a "high-water mark" — it can be stale but never lower than the true max of any window better than the current best. This keeps the algorithm O(n) without re-scanning.
- Time: O(n). Space: O(26).

```java
// LC 424: Replace at most k chars — max length of substring with same char
int characterReplacement(String s, int k) {
    int[] freq = new int[26];
    int left = 0, maxFreq = 0, result = 0;
    for (int right = 0; right < s.length(); right++) {
        freq[s.charAt(right) - 'A']++;
        maxFreq = Math.max(maxFreq, freq[s.charAt(right) - 'A']);
        // window size - maxFreq = replacements needed
        while (right - left + 1 - maxFreq > k) {
            freq[s.charAt(left++) - 'A']--;
        }
        result = Math.max(result, right - left + 1);
    }
    return result;
}
```
> Trick: `maxFreq` never decreases — we never shrink the window unnecessarily.

---

## P3: Hashing / Frequency Map

### Variation 1: Anagram Check (LC 242)

**Problem:** Given two strings `s` and `t`, return `true` if `t` is an anagram of `s` (same characters with same multiplicities).

**Approach (Frequency-count compare):**
- Length mismatch → instantly `false`.
- Build a 26-slot freq array from `s` (assuming lowercase). For each char in `t`, decrement; if any slot goes negative, `t` has more of that letter than `s` → not an anagram.
- Single pass over `t` after the build → O(n) time, O(1) space.
- Unicode follow-up (FAANG): use a `HashMap<Character, Integer>` instead of `int[26]`.
- Alternative: sort both strings and compare — O(n log n), no extra space (besides sort buffer).

```java
boolean isAnagram(String s, String t) {
    if (s.length() != t.length()) return false;
    int[] freq = new int[26];
    for (char c : s.toCharArray()) freq[c - 'a']++;
    for (char c : t.toCharArray()) {
        if (--freq[c - 'a'] < 0) return false;
    }
    return true;
}
```

### Variation 2: Group Anagrams (LC 49)

**Problem:** Given an array of strings, group those that are anagrams of each other.

**Approach (Canonical-key hashing):**
- Two strings are anagrams iff they share a **canonical form**. Choose any deterministic transform invariant under reordering:
  - **Sorted form:** `Arrays.sort(s.toCharArray())` → O(L log L) per word.
  - **Frequency form:** `"a3b1c0..."` (see Variation 3) → O(L) per word.
- Use a `HashMap<canonicalKey, List<String>>` and append each input string to its bucket.
- Time: O(N · L log L) with sort, O(N · L) with freq form. Space: O(N · L).

```java
// Key insight: sorted version of a word is the canonical form
Map<String, List<String>> groupAnagrams(String[] strs) {
    Map<String, List<String>> map = new HashMap<>();
    for (String s : strs) {
        char[] chars = s.toCharArray();
        Arrays.sort(chars);
        String key = new String(chars);
        map.computeIfAbsent(key, k -> new ArrayList<>()).add(s);
    }
    return map;
}
```
> FAANG variation: use freq array as key instead of sorting → O(n) per word.

### Variation 3: Encode String as Key (Frequency canonical form)

**Problem:** Build a canonical key per string so anagrams produce identical keys — without sorting.

**Approach (Frequency-array serialization):**
- Build `freq[26]` from the string, then serialize as `count + letter` pairs (e.g., `"aab" → "2a1b"`).
- Use **count + letter** (not just counts) to avoid ambiguity: `"ab"` and `"ba"` should collide, but `"11ab"` and `"1a1b"` shouldn't be confusable with `"1aabbb"` style strings.
- Time per string: O(L + 26). Space: O(26) per key.
- Useful as a drop-in for LC 49 when L is large and you want better-than `L log L`.

```java
// Frequency-based key: "aab" → "2a1b" (no sorting needed, O(n))
String encode(String s) {
    int[] freq = new int[26];
    for (char c : s.toCharArray()) freq[c-'a']++;
    StringBuilder sb = new StringBuilder();
    for (int i = 0; i < 26; i++) {
        if (freq[i] > 0) sb.append(freq[i]).append((char)('a'+i));
    }
    return sb.toString();
}
```

### Questions

| # | Problem | LC# |
|---|---------|-----|
| E1 | Valid Anagram | 242 |
| E2 | Ransom Note | 383 |
| M1 | Group Anagrams | 49 |
| M2 | Top K Frequent Words | 692 |
| M3 | Sort Characters By Frequency | 451 |
| H1 | Minimum Number of Steps to Make Two Strings Anagram II | 2186 |

---

## P4: Stack-Based String Problems

### Variation 1: Valid Parentheses (LC 20)

**Problem:** Given a string `s` of `()[]{}`, return `true` if every opening bracket is correctly matched and closed in the right order.

**Approach (Stack of opens):**
- Iterate left to right. Push every opening bracket.
- On a closing bracket, the stack top must be the **matching** opener; else invalid.
- After processing, stack must be **empty** (otherwise unmatched opens remain).
- Why a stack: nested brackets close in LIFO order — exactly stack semantics.
- Time: O(n). Space: O(n) in the worst case (all opens).

```java
boolean isValid(String s) {
    Deque<Character> stack = new ArrayDeque<>();
    for (char c : s.toCharArray()) {
        if (c == '(' || c == '[' || c == '{') {
            stack.push(c);
        } else {
            if (stack.isEmpty()) return false;
            char top = stack.pop();
            if (c == ')' && top != '(') return false;
            if (c == ']' && top != '[') return false;
            if (c == '}' && top != '{') return false;
        }
    }
    return stack.isEmpty();
}
```

### Variation 2: Decode String (LC 394)

**Problem:** Decode strings of the form `k[encoded]` where `encoded` may itself contain nested `k[...]`. Example: `"3[a2[c]]" → "accaccacc"`.

**Approach (Two stacks — count + string-builder):**
- Walk the input character by character; track a current count `k` and current `StringBuilder`.
- On a digit: `k = k * 10 + digit` (handles multi-digit counts like `"12[ab]"`).
- On `'['`: push `k` and the current builder onto their stacks, then reset both — we're entering a nested scope.
- On `']'`: pop the saved count and builder; repeat the current builder `count` times and **append** it to the popped builder; that becomes the new current.
- On a letter: append to current builder.
- This essentially simulates recursion iteratively. Time: O(output length). Space: O(nesting depth).

```java
String decodeString(String s) {
    Deque<Integer> countStack = new ArrayDeque<>();
    Deque<StringBuilder> strStack = new ArrayDeque<>();
    StringBuilder current = new StringBuilder();
    int k = 0;

    for (char c : s.toCharArray()) {
        if (Character.isDigit(c)) {
            k = k * 10 + (c - '0');
        } else if (c == '[') {
            countStack.push(k);
            strStack.push(current);
            current = new StringBuilder();
            k = 0;
        } else if (c == ']') {
            int repeat = countStack.pop();
            StringBuilder prev = strStack.pop();
            String repeated = current.toString().repeat(repeat);
            current = prev.append(repeated);
        } else {
            current.append(c);
        }
    }
    return current.toString();
}
```

### Variation 3: Basic Calculator (LC 224)

**Problem:** Implement a calculator that evaluates a string expression containing non-negative integers, `+`, `-`, `(`, `)`, and spaces. No `*` or `/`.

**Approach (Stack of (result, sign) snapshots):**
- Maintain four running variables: `result`, `num` (current number being parsed), `sign` (`+1` or `-1`), and a `stack`.
- Digit → extend `num` (multi-digit safe).
- `+` / `-` → finalize: `result += sign * num`, reset `num = 0`, set new `sign`.
- `(` → push the **current `result`** and **current `sign`** (these are the "context outside the parens"), reset `result = 0, sign = +1` to start fresh on the sub-expression.
- `)` → finalize the inner sub-expression into `result`; pop the saved outer sign and multiply; pop the saved outer result and add. This collapses the parenthesized sub-expression back into the surrounding context.
- After the loop, do one final `result += sign * num` for the trailing number.
- Time: O(n). Space: O(depth of nesting).

```java
// Handle +, -, (, ) with no * or /
int calculate(String s) {
    Deque<Integer> stack = new ArrayDeque<>();
    int result = 0, num = 0, sign = 1;

    for (char c : s.toCharArray()) {
        if (Character.isDigit(c)) {
            num = num * 10 + (c - '0');
        } else if (c == '+') {
            result += sign * num;
            num = 0; sign = 1;
        } else if (c == '-') {
            result += sign * num;
            num = 0; sign = -1;
        } else if (c == '(') {
            stack.push(result);     // save current result
            stack.push(sign);       // save current sign
            result = 0; sign = 1;
        } else if (c == ')') {
            result += sign * num;
            num = 0;
            result *= stack.pop();  // apply outer sign
            result += stack.pop();  // add outer result
        }
    }
    return result + sign * num;
}
```

### Questions

| # | Problem | LC# |
|---|---------|-----|
| E1 | Valid Parentheses | 20 |
| M1 | Decode String | 394 |
| M2 | Remove All Adjacent Duplicates in String | 1047 |
| M3 | Remove K Digits | 402 |
| H1 | Basic Calculator | 224 |
| H2 | Basic Calculator II (with * /) | 227 |
| H3 | Largest Rectangle in String (parentheses height) | — |
| H4 | Minimum Remove to Make Valid Parentheses | 1249 |

---

## P5: KMP — Pattern Matching

### Why KMP?
Naive: O(n × m). KMP: O(n + m).

### The LPS Array (failure function)
LPS = `Longest Proper Prefix which is also Suffix`.

For every index `i` in the pattern:
- look at `pattern[0..i]`
- find the longest prefix that is also a suffix
- store its length in `lps[i]`

Important:
- "proper prefix" means the whole string itself does **not** count
- LPS tells us how much of the pattern we can reuse after a mismatch

```
pattern = "ababc"
lps     = [0, 0, 1, 2, 0]

lps[i] = length of longest proper prefix of pattern[0..i]
         that is also a suffix.
```

Example breakdown:
```text
pattern = a b a b c
index      0 1 2 3 4

i = 0 -> "a"      -> no proper prefix/suffix -> 0
i = 1 -> "ab"     -> no match               -> 0
i = 2 -> "aba"    -> "a" is both prefix and suffix -> 1
i = 3 -> "abab"   -> "ab" is both prefix and suffix -> 2
i = 4 -> "ababc"  -> no match               -> 0
```

Why this helps:
- suppose we matched `pattern[0..j-1]`
- then a mismatch happens at `pattern[j]`
- instead of starting from scratch, we jump to `lps[j-1]`
- that means we keep the part that is still guaranteed to match

### Building LPS
```java
int[] buildLPS(String pattern) {
    int m = pattern.length();
    int[] lps = new int[m];
    int len = 0, i = 1;
    while (i < m) {
        if (pattern.charAt(i) == pattern.charAt(len)) {
            lps[i++] = ++len;
        } else if (len > 0) {
            len = lps[len - 1];   // fall back (don't reset i)
        } else {
            lps[i++] = 0;
        }
    }
    return lps;
}
```

How the builder works:
- `len` = current length of matched prefix
- `i` = index we are computing
- if `pattern[i] == pattern[len]`, we extend the match
- if not, we do not restart from `0` immediately
- instead, we fall back to `lps[len - 1]`

This fallback is the whole KMP trick.

Example build trace for `ababc`:
```text
pattern: a b a b c
index:   0 1 2 3 4
lps:     0 0 1 2 0

i=1, len=0: b != a -> lps[1]=0
i=2, len=0: a == a -> lps[2]=1, len=1
i=3, len=1: b == b -> lps[3]=2, len=2
i=4, len=2: c != a -> fallback len = lps[1] = 0
            c != a -> lps[4]=0
```

### KMP Search
```java
List<Integer> kmpSearch(String text, String pattern) {
    int[] lps = buildLPS(pattern);
    List<Integer> result = new ArrayList<>();
    int i = 0, j = 0;
    while (i < text.length()) {
        if (text.charAt(i) == pattern.charAt(j)) {
            i++; j++;
            if (j == pattern.length()) {
                result.add(i - j);
                j = lps[j - 1];
            }
        } else if (j > 0) {
            j = lps[j - 1];     // don't move i
        } else {
            i++;
        }
    }
    return result;
}
```

Search intuition:
- `i` walks through the text
- `j` walks through the pattern
- if characters match, move both forward
- if they mismatch and `j > 0`, use `lps[j - 1]` to move `j` back
- `i` does not move backward, so each text character is processed at most once

Mini example:
```text
text    = abxabcabcaby
pattern = abcaby
```
Suppose we matched `abca` and then got a mismatch.
Instead of comparing from pattern start again, KMP asks:
- what is the longest prefix of `abc a` that is also a suffix?
- answer comes from `lps`
- jump `j` to that shorter valid prefix and continue

That is why KMP avoids repeated comparisons.

### FAANG Use Cases for KMP
1. "Find all occurrences of pattern in text"
2. "Shortest palindrome" — reverse + KMP on s + '#' + reverse(s)
3. "Repeated substring pattern" — check if s is in (s+s)[1..2n-2]

### Repeated Substring Pattern (LC 459)

**Problem:** Given a string `s`, return `true` if it can be constructed by repeating some non-empty substring two or more times.

**Approach (Concatenation trick):**
- Form `doubled = s + s` and search for `s` in `doubled[1 : 2n-1]` (strip the very first and very last character).
- Why it works: if `s = p` repeated `k ≥ 2` times, then `s + s` contains `s` starting at index `|p|` (a rotation of `p`-blocks). Conversely, finding `s` in the trimmed `doubled` implies a rotation by some `d < n` maps `s` to itself, which forces a period `d | n`.
- Stripping endpoints prevents the trivial match at index 0.
- Time: O(n²) with `String.contains` (uses naive search). Use **KMP** for O(n) — that's the FAANG-grade variant.

```java
// LC 459: Does s consist of a repeated substring?
boolean repeatedSubstringPattern(String s) {
    String doubled = s + s;
    // If s can be formed by repeating a substring,
    // then s must appear in doubled[1..2n-2]
    return (doubled.substring(1, doubled.length() - 1)).contains(s);
    // Use KMP for O(n) — above is O(n²) due to String.contains
}
```

### Questions

| # | Problem | LC# |
|---|---------|-----|
| M1 | Implement strStr() / Find needle in haystack | 28 |
| M2 | Repeated Substring Pattern | 459 |
| H1 | Shortest Palindrome | 214 |
| H2 | String Matching in Array | 1408 |

---

## FAANG Tricky String Questions

### 1. Longest Common Prefix (LC 14)

**Problem:** Given an array of strings, return their longest common prefix; return `""` if there is none.

**Approach (Horizontal scan / shrink-to-fit):**
- Start with `prefix = strs[0]`. For each subsequent string, **shrink** the prefix from the right until `strs[i].startsWith(prefix)`.
- Early exit if `prefix` becomes empty — no common prefix possible.
- Time: O(S) where S = total characters across all strings. Space: O(1).
- Alternatives: vertical scan (compare char `j` across all strings, stop at first mismatch); divide-and-conquer; binary search on the prefix length (`O(S · log minLen)`).

```java
// Binary search on the length
String longestCommonPrefix(String[] strs) {
    if (strs.length == 0) return "";
    String first = strs[0];
    for (int i = 1; i < strs.length; i++) {
        while (!strs[i].startsWith(first)) {
            first = first.substring(0, first.length() - 1);
        }
    }
    return first;
}
```

### 2. String to Integer / atoi (LC 8) — Edge Cases

**Problem:** Convert a string to a 32-bit signed integer following these rules: skip leading whitespace; optional `+`/`-` sign; consume digits until a non-digit; clamp the result to `[Integer.MIN_VALUE, Integer.MAX_VALUE]`.

**Approach (Single-pass state machine):**
- Walk left to right with an explicit phase: whitespace → sign → digits.
- Skip leading spaces only once (not interleaved).
- Read at most one `+`/`-`; remember the sign and advance.
- Accumulate digits: `result = result * 10 + digit`. **Before** the multiply/add, check for overflow:
  - If `result > MAX/10`, or `result == MAX/10 && digit > MAX%10` → clamp to `MAX_VALUE` (negative side: `MIN_VALUE`).
- Stop at the first non-digit.
- FAANG loves this because mishandling any one rule fails dozens of test cases (e.g., `"   -42"`, `"+-12"`, `"  0000001"`, `"-2147483649"`).

### 3. Count and Say (LC 38)

**Problem:** Define `countAndSay(1) = "1"`. For `n > 1`, `countAndSay(n)` is the **run-length encoding** of `countAndSay(n-1)`, where each run of identical digits is read as "count then digit" (e.g., `"1211" → "111221"` because it has *one* 1, *one* 2, *two* 1s). Return `countAndSay(n)`.

**Approach (Iterative run-length encoding):**
- Start with `s = "1"`. Loop `n - 1` times.
- In each iteration, scan `s` and group consecutive identical characters; for each run, append `count` then the character to a `StringBuilder`.
- Two-pointer style: inner pointer `j` advances while `s[j]` matches the run's leading char.
- Time: O(n · L) where L is the length of the final string (grows roughly like the Conway constant ~ 1.303ⁿ). Space: O(L).

```java
String countAndSay(int n) {
    String s = "1";
    for (int i = 1; i < n; i++) {
        StringBuilder sb = new StringBuilder();
        int j = 0;
        while (j < s.length()) {
            char c = s.charAt(j);
            int count = 0;
            while (j < s.length() && s.charAt(j) == c) { j++; count++; }
            sb.append(count).append(c);
        }
        s = sb.toString();
    }
    return s;
}
```

### Complete FAANG String Question List

| Problem | Pattern | Difficulty | LC# |
|---------|---------|------------|-----|
| Valid Palindrome | Two Pointers | E | 125 |
| Longest Common Prefix | Greedy | E | 14 |
| Roman to Integer | HashMap | E | 13 |
| Longest Substring Without Repeating | Sliding Window | M | 3 |
| Longest Palindromic Substring | Expand Center | M | 5 |
| Group Anagrams | HashMap | M | 49 |
| Valid Anagram | Freq Array | E | 242 |
| String to Integer (atoi) | Simulation | M | 8 |
| Decode String | Stack | M | 394 |
| Permutation in String | Sliding Window | M | 567 |
| Find All Anagrams | Sliding Window | M | 438 |
| Longest Repeating Char Replacement | Sliding Window | M | 424 |
| Minimum Window Substring | Sliding Window | H | 76 |
| Basic Calculator | Stack | H | 224 |
| Shortest Palindrome | KMP | H | 214 |
| Wildcard Matching | DP | H | 44 |
| Regular Expression Matching | DP | H | 10 |
| Word Break | DP | M | 139 |

---

> Back to [Master Index](../DSA_MASTER_INDEX.md)
