# Miscellaneous Algorithms — Deep Dive Notes
> Non-obvious algorithms that require understanding WHY they work, not just how.
> Same format as: prefix_sum / difference_array notes.

---

## What Goes Here?

These are algorithms where:
1. The naive approach is obvious but too slow
2. The clever solution is counterintuitive at first
3. The core insight (once understood) is reusable across many problems

---

## Index

| File | Algorithm | Key Idea | Complexity |
|------|-----------|----------|------------|
| [kmp_notes.md](kmp_notes.md) | KMP — String Matching | LPS failure function avoids restart | O(n+m) |
| [rabin_karp_notes.md](rabin_karp_notes.md) | Rabin-Karp — Rolling Hash | Update hash in O(1) as window slides | O(n+m) avg |
| [manacher_notes.md](manacher_notes.md) | Manacher's — Palindromes | Mirror symmetry avoids re-expansion | O(n) |
| [floyds_cycle_detection_notes.md](floyds_cycle_detection_notes.md) | Floyd's — Cycle Detection | Hare meets tortoise, math gives start | O(n), O(1) space |
| [boyer_moore_voting_notes.md](boyer_moore_voting_notes.md) | Boyer-Moore Voting | Votes cancel, majority survives | O(n), O(1) space |
| [difference_array_prefix_sum_notes.md](../difference_array_prefix_sum_notes.md) | Prefix Sum + Diff Array | Range update/query tricks | O(n) |

---

## Quick Pattern Recognition

```
Problem says...                         → Use...
────────────────────────────────────────────────────────────
"find pattern in text"                  → KMP (guaranteed O(n+m))
"multiple patterns" or "binary search   → Rabin-Karp (rolling hash)
  on substring length"
"longest palindromic substring"         → Manacher's (O(n)) or expand O(n²)
"cycle in linked list / find start"     → Floyd's two-pointer
"find duplicate in array [1..n]"        → Floyd's (array as linked list)
"majority element (> n/2)"              → Boyer-Moore voting
"majority elements (> n/3)"             → Boyer-Moore with 2 candidates
"range updates, query final values"     → Difference Array
"range sum queries, static array"       → Prefix Sum
```

---

## Why These Algorithms Are Grouped Together

All of them share the same meta-pattern:

```
Naive: O(n²) or O(n × m)
       doing redundant work for each position

Clever: O(n) or O(n + m)
        reuse information already computed for nearby positions

The insight is always:
"We already know X about what we've seen.
 Use X to skip unnecessary work going forward."
```

| Algorithm | Information Reused |
|-----------|-------------------|
| KMP | LPS table: longest border of matched prefix |
| Rabin-Karp | Hash: subtract left, multiply, add right |
| Manacher's | P array: mirror palindrome radius |
| Floyd's | Meeting point: math gives cycle start |
| Boyer-Moore | Vote count: majority can't be eliminated |
| Diff Array | Boundary events: defer reconstruction |

---

> Back to [Master Index](../DSA_MASTER_INDEX.md)
