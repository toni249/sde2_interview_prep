# Tries — Pattern Deep Dive
> FAANG Level | Java | Easy → Hard

---

## Pattern Map

```
TRIE
├── P1: Standard Trie      → insert, search, startsWith (autocomplete)
├── P2: Word Search        → word search in grid (Trie + DFS)
├── P3: XOR Trie           → maximum XOR pair
└── P4: Trie + DP          → word break, palindrome pairs
```

---

## P1: Standard Trie

### Variation 1: Implement Trie (Prefix Tree) (LC 208)

**Problem:** Build a `Trie` with `insert(word)`, `search(word)` (exact match), and `startsWith(prefix)` (any word that begins with prefix).

**Approach (26-ary character tree):**
- **What each node stores:** an array of 26 children (one per lowercase letter) and a boolean `isEnd` marking whether *some* inserted word ends here. The node itself does not store its letter — its position in the parent's `children[]` is the letter.
- **Each character traversal** = one level down: walking from root following letters of `w` reaches the node "after" `w`.
- Insert: walk/create children for each char; set `isEnd` at the last node.
- `search` requires `isEnd == true` at the final node; `startsWith` only requires that the path exists.
- Time **O(L)** per op (L = word length); space **O(total chars × 26)** (or `O(1)` per node using `HashMap` children).

### Structure
```java
class TrieNode {
    TrieNode[] children = new TrieNode[26];
    boolean isEnd;
}

class Trie {
    TrieNode root = new TrieNode();

    void insert(String word) {
        TrieNode curr = root;
        for (char c : word.toCharArray()) {
            int idx = c - 'a';
            if (curr.children[idx] == null) curr.children[idx] = new TrieNode();
            curr = curr.children[idx];
        }
        curr.isEnd = true;
    }

    boolean search(String word) {
        TrieNode curr = root;
        for (char c : word.toCharArray()) {
            int idx = c - 'a';
            if (curr.children[idx] == null) return false;
            curr = curr.children[idx];
        }
        return curr.isEnd;
    }

    boolean startsWith(String prefix) {
        TrieNode curr = root;
        for (char c : prefix.toCharArray()) {
            int idx = c - 'a';
            if (curr.children[idx] == null) return false;
            curr = curr.children[idx];
        }
        return true;
    }
}
```

### Variation 2: Add and Search Word — Wildcard '.' (LC 211)

**Problem:** Design a data structure supporting `addWord(word)` and `search(word)`, where `word` in search may contain '.' as a wildcard matching **any single letter**.

**Approach (Trie + recursive branching at wildcards):**
- Same trie structure as LC 208 — each node holds 26 children + `isEnd`.
- Search recursion: for a regular char, descend one branch (O(1)); for `'.'`, **branch into all 26 non-null children** and recurse — DFS short-circuits as soon as a match path completes.
- Worst-case time: **O(26^d × L)** where `d` = number of dots (small in practice); typical near O(L).
- Edge cases: search hitting null child = false; pattern of all dots → exhaustive but bounded by depth.

```java
boolean search(String word) {
    return searchHelper(word, 0, root);
}
boolean searchHelper(String word, int idx, TrieNode node) {
    if (idx == word.length()) return node.isEnd;
    char c = word.charAt(idx);
    if (c == '.') {
        for (TrieNode child : node.children) {
            if (child != null && searchHelper(word, idx + 1, child)) return true;
        }
        return false;
    }
    TrieNode next = node.children[c - 'a'];
    return next != null && searchHelper(word, idx + 1, next);
}
```

### Variation 3: Replace Words (LC 648)

**Problem:** Given a list of `dictionary` "roots" and a `sentence`, replace every word in the sentence with the **shortest root** that is a prefix of that word. If no root matches, keep the word as-is.

**Approach (Trie of roots + greedy shortest-prefix walk):**
- Insert all roots into a trie; each `isEnd` node marks the end of a root.
- For each word: walk down the trie char-by-char. The **first** time you hit `isEnd`, you've found the shortest root prefix — return it.
- If a character is missing from the trie before hitting an `isEnd`, no root matches → keep original word.
- Why trie (not HashSet of roots)? With a HashSet, you'd check every prefix length per word — O(L²). Trie does it in **O(L)** per word.

```java
// Build trie of dictionary words (roots)
// For each word in sentence, find shortest root prefix
String replaceWords(List<String> dictionary, String sentence) {
    Trie trie = new Trie();
    for (String root : dictionary) trie.insert(root);

    StringBuilder sb = new StringBuilder();
    for (String word : sentence.split(" ")) {
        if (sb.length() > 0) sb.append(' ');
        sb.append(findRoot(trie.root, word));
    }
    return sb.toString();
}
String findRoot(TrieNode root, String word) {
    TrieNode curr = root;
    StringBuilder prefix = new StringBuilder();
    for (char c : word.toCharArray()) {
        if (curr.children[c - 'a'] == null) break;
        prefix.append(c);
        curr = curr.children[c - 'a'];
        if (curr.isEnd) return prefix.toString();  // found root
    }
    return word;  // no root found
}
```

### Questions — Standard Trie

| # | Problem | LC# | Difficulty |
|---|---------|-----|------------|
| E1 | Implement Trie (Prefix Tree) | 208 | M |
| M1 | Design Add and Search Words Data Structure | 211 | M |
| M2 | Replace Words | 648 | M |
| M3 | Longest Word in Dictionary | 720 | M |
| H1 | Palindrome Pairs | 336 | H |

---

## P2: Word Search II — Trie + DFS (LC 212)

**Problem:** Given an `m x n` board of letters and a list of `words`, return **all words from the list** that can be formed by walking adjacent (4-directional) cells, with each cell used at most once per word.

**Approach (Trie + backtracking DFS):**
- **What each trie node stores:** the 26 children + the word itself (or `isEnd` flag) at the terminal node. Storing the whole word at the end node avoids reconstructing it via a path-stack.
- **Each character traversal** = matching the next board cell against the current trie node — if `trie.children[c]` is null, **prune immediately** (no word continues this prefix).
- DFS each cell, descending the trie in lockstep with the board walk. Mark visited (e.g., set cell to `'#'`), recurse 4 directions, restore.
- When we hit a node with `isEnd`, add the word to results and clear its flag to avoid duplicates.
- Optimization: after finding a word, prune leaf trie nodes (cleanup) — reduces future branching dramatically.

### Why Trie + DFS?
> Naive: search for each word separately — O(words × m × n × 4^L).
> With Trie: search all words simultaneously in one DFS — O(m × n × 4^L).

```java
List<String> findWords(char[][] board, String[] words) {
    Trie trie = new Trie();
    for (String word : words) trie.insert(word);

    Set<String> result = new HashSet<>();
    int m = board.length, n = board[0].length;

    for (int i = 0; i < m; i++) {
        for (int j = 0; j < n; j++) {
            dfs(board, i, j, trie.root, new StringBuilder(), result);
        }
    }
    return new ArrayList<>(result);
}

void dfs(char[][] board, int i, int j, TrieNode node, StringBuilder path, Set<String> result) {
    if (i < 0 || i >= board.length || j < 0 || j >= board[0].length || board[i][j] == '#')
        return;

    char c = board[i][j];
    TrieNode next = node.children[c - 'a'];
    if (next == null) return;        // no word starts with this prefix

    path.append(c);
    if (next.isEnd) result.add(path.toString());

    board[i][j] = '#';              // mark visited
    dfs(board, i+1, j, next, path, result);
    dfs(board, i-1, j, next, path, result);
    dfs(board, i, j+1, next, path, result);
    dfs(board, i, j-1, next, path, result);
    board[i][j] = c;               // restore
    path.deleteCharAt(path.length() - 1);
}
```

---

## P3: XOR Trie — Maximum XOR

### Core Idea
> Store numbers bit by bit (MSB to LSB) in a Trie.
> For each number, greedily choose the opposite bit at each level to maximize XOR.

**What each XOR-trie node stores:** exactly 2 children — one for bit `0`, one for bit `1`. Depth = bit position (typically 32 for ints), so every number lives on a path of length 32 from the root. **Each level represents one bit, MSB → LSB.**

**Why MSB first?** XOR is maximized when the **highest** differing bit is set — so at each level, if we can take the **opposite bit** (giving us a `1` in this position of the XOR), we get the largest possible contribution `2^i`. Greedy from MSB downward is optimal because higher bits dominate.

```java
class XORTrie {
    int[][] children;
    int idx = 1;

    XORTrie(int maxNumbers) {
        children = new int[maxNumbers * 32][2];  // binary trie
    }

    void insert(int num) {
        int curr = 0;
        for (int i = 31; i >= 0; i--) {
            int bit = (num >> i) & 1;
            if (children[curr][bit] == 0) children[curr][bit] = idx++;
            curr = children[curr][bit];
        }
    }

    int maxXOR(int num) {
        int curr = 0, result = 0;
        for (int i = 31; i >= 0; i--) {
            int bit = (num >> i) & 1;
            int opposite = 1 - bit;
            if (children[curr][opposite] != 0) {
                result |= (1 << i);
                curr = children[curr][opposite];
            } else {
                curr = children[curr][bit];
            }
        }
        return result;
    }
}
```

### Maximum XOR of Two Numbers in Array (LC 421)

**Problem:** Given an integer array `nums`, return the maximum value of `nums[i] XOR nums[j]` for any `i != j`.

**Approach (Binary trie + greedy MSB walk):**
- Insert every number into a 32-level binary trie (bits MSB → LSB).
- For each `num`, walk the trie from the root: at each level, look for the **opposite bit** child — if it exists, taking it sets a `1` in that bit of the XOR (best possible); else fall back to the same bit.
- Accumulate bits set, return the max across all numbers.
- Time **O(32n) = O(n)**, space **O(32n)**.
- Alternative: bitmask + HashSet trick — also O(n).

```java
int findMaximumXOR(int[] nums) {
    XORTrie trie = new XORTrie(nums.length);
    for (int num : nums) trie.insert(num);
    int max = 0;
    for (int num : nums) max = Math.max(max, trie.maxXOR(num));
    return max;
}
```

---

## P4: Trie + DP

### Word Break (LC 139)

**Problem:** Given a string `s` and a dictionary of words `wordDict`, return `true` if `s` can be **segmented** into a space-separated sequence of one or more dictionary words (words can be reused).

**Approach (Trie + 1-D DP):**
- `dp[i]` = "can `s[0..i)` be segmented?". Base `dp[0] = true`.
- For every reachable position `i`, walk the trie from index `i` forward. Every time we hit a trie `isEnd`, position `j+1` is reachable → `dp[j+1] = true`.
- Trie prevents O(L) substring hash lookups per attempt — we **prune immediately** when no dictionary word extends the current prefix.
- Time **O(n²)** worst-case (vs O(n × L_max) with HashSet); space **O(n + dict size)**.

```java
// Build trie of dictionary, then DP
boolean wordBreak(String s, List<String> wordDict) {
    Trie trie = new Trie();
    for (String w : wordDict) trie.insert(w);

    int n = s.length();
    boolean[] dp = new boolean[n + 1];
    dp[0] = true;

    for (int i = 0; i < n; i++) {
        if (!dp[i]) continue;
        TrieNode curr = trie.root;
        for (int j = i; j < n; j++) {
            int c = s.charAt(j) - 'a';
            if (curr.children[c] == null) break;
            curr = curr.children[c];
            if (curr.isEnd) dp[j + 1] = true;
        }
    }
    return dp[n];
}
```

---

## P5: Streaming Patterns

### Stream of Characters (LC 1032)

**Problem:** Implement `StreamChecker(words)` and `query(char)` that returns `true` if **any of the last k characters typed so far** spells a word in `words` (for any k).

**Approach (Reversed-word trie + suffix matching):**
- Key trick: insert each dictionary word **reversed** into the trie. Then on each `query(c)`, append `c` to a running buffer and walk the trie from the **end of the buffer backwards** — this checks whether any suffix of the typed stream is a reversed dictionary word, i.e. any dictionary word is a suffix of the stream.
- Stop early on a `null` child (no dictionary word continues this suffix) or on `isEnd` (match found).
- Bound the buffer length to the longest dictionary word to keep query O(maxLen).
- Time per query **O(L_max)**, space **O(total dict chars)**.

---

## Trie vs HashMap

```
Use Trie when:                          Use HashMap when:
─────────────────────────────────────── ─────────────────────────
Prefix-based queries                    Exact key lookup
Autocomplete                            Frequency counting
Word search with shared prefixes        Simple membership check
XOR maximization                        Order not important
Space matters (shared prefix storage)   (HashMap uses more space)
```

---

> Back to [Master Index](../DSA_MASTER_INDEX.md)
