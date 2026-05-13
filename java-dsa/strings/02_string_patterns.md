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

### Variation 1: Palindrome Check
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

### Variation 2: Palindrome with One Deletion Allowed
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

### Variation 3: Reverse Words in String
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

### Longest Palindromic Substring — Expand Around Center
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

### Variation 1: Longest Substring Without Repeating Characters
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

### Variation 2: Minimum Window Substring (Classic Hard)
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

### Variation 3: Permutation in String (Fixed Window = t.length())
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
> Same as permutation check but collect all valid window start indices.

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

### Longest Repeating Character Replacement (FAANG Favorite)
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

### Variation 1: Anagram Check
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

### Variation 2: Group Anagrams
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

### Variation 3: Encode String as Key
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

### Variation 1: Valid Parentheses
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
> "3[a2[c]]" → "accaccacc"

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
```
pattern = "ababc"
lps     = [0, 0, 1, 2, 0]

lps[i] = length of longest proper prefix of pattern[0..i]
         that is also a suffix.
```

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

### FAANG Use Cases for KMP
1. "Find all occurrences of pattern in text"
2. "Shortest palindrome" — reverse + KMP on s + '#' + reverse(s)
3. "Repeated substring pattern" — check if s is in (s+s)[1..2n-2]

### Repeated Substring Pattern
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

### 1. Longest Common Prefix
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

### 2. String to Integer (atoi) — Edge Cases
> FAANG loves this for edge case handling.
- Skip leading spaces
- Handle optional +/-
- Stop at non-digit
- Clamp to Integer.MIN_VALUE / MAX_VALUE

### 3. Count and Say
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
