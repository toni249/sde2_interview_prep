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

### Variation: With Wildcard '.' (LC 211)
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

### Variation: Replace Words with Roots (LC 648)
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

### Maximum XOR of Two Numbers (LC 421)
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
