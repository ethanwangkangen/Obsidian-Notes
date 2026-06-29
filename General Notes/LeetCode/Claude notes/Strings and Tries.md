# Strings & Tries — Comprehensive Reference

---

## How to Identify

**Primary signals:**

- "Pattern matching" or "find substring" → KMP or Rabin-Karp.
- "All palindromic substrings" → expand from center or Manacher's.
- "Prefix search" or "autocomplete" → Trie.
- "Word search in a grid with dictionary" → Trie + DFS.
- "Maximum XOR" → Binary Trie.
- Anagram-related → frequency counting.
- Subsequence checking → two pointers.

---

## String Matching

### KMP (Knuth-Morris-Pratt)

**Core idea:** Precompute a "failure function" (prefix table / LPS array) for the pattern. The LPS[i] = length of the longest proper prefix of pattern[0..i] that is also a suffix. During matching, when a mismatch occurs, use LPS to skip ahead instead of restarting.

**When to use:** "Find all occurrences of pattern in text" in O(n + m).

**Key insight:** The LPS array tells you: "if matching fails at position j in the pattern, what is the longest prefix that still matches?" Jump to LPS[j-1] instead of 0.

LC 28 (Find the Index of the First Occurrence — strStr), LC 214 (Shortest Palindrome — KMP on reverse + '#' + original).

### Rabin-Karp (Rolling Hash)

**Core idea:** Hash the pattern. Slide a window over the text, maintaining a rolling hash. When hashes match, verify with actual comparison (to handle collisions).

**Rolling hash formula:** `hash = (hash - old_char * base^(m-1)) * base + new_char`, all mod a large prime.

**When to use:** Multiple pattern matching, 2D pattern matching, or when you need to compare many substrings efficiently.

**Collision handling:** Use two independent hash functions (double hashing) for near-zero collision probability.

LC 187 (Repeated DNA Sequences), LC 1044 (Longest Duplicate Substring — binary search on length + rolling hash).

### Z-Algorithm

**Core idea:** Compute Z-array where Z[i] = length of the longest substring starting at i that matches a prefix of the string. Similar to KMP but sometimes simpler to implement.

**For pattern matching:** concatenate pattern + '$' + text, compute Z-array. Any Z[i] == len(pattern) indicates a match at position i - len(pattern) - 1 in the text.

---

## Palindrome Techniques

### Expand from Center

For each possible center (n single-char centers + n-1 between-char centers), expand outward while characters match. O(n²) total.

LC 5 (Longest Palindromic Substring), LC 647 (Palindromic Substrings — count all).

### Manacher's Algorithm

Find all maximal palindromes in O(n). Transforms the string by inserting separators (e.g., "#a#b#a#" for "aba") to handle even-length palindromes uniformly. Maintains a rightmost palindrome boundary and its center, using symmetry to bootstrap new palindrome computations.

LC 5 (Longest Palindromic Substring — Manacher's is optimal but expand-from-center is usually sufficient).

---

## Trie (Prefix Tree)

### Standard Trie

Each node has children (array[26] or hashmap) and an `is_end` flag. Insert and search in O(word_length).

**Operations:**

- Insert: walk/create nodes for each character.
- Search: walk nodes; return true if the final node has `is_end = true`.
- StartsWith: same as search but don't check `is_end`.

LC 208 (Implement Trie).

### Trie + DFS for Word Search II

Build a trie from the word list. DFS on the grid, following the trie simultaneously. If the current trie node is null, prune (no word starts with this prefix). If `is_end` is true, record the word. This is dramatically faster than running separate DFS for each word.

**Optimization:** After finding a word, remove it from the trie (set `is_end = false` and prune empty branches) to avoid re-finding it.

LC 212 (Word Search II).

### Search with Wildcards

'.' matches any character. At a '.', try all 26 children recursively.

LC 211 (Design Add and Search Words Data Structure).

### Binary Trie for Maximum XOR

Build a trie of all numbers' BINARY representations (MSB to LSB, 32 or 64 bits). To find the max XOR with a query number, greedily walk the trie choosing the OPPOSITE bit at each level (to maximize XOR). If the opposite branch doesn't exist, take the same bit.

LC 421 (Maximum XOR of Two Numbers in Array).

---

## Other String Patterns

### Anagram Detection

Two strings are anagrams if they have the same character frequency. Use a frequency array of size 26 (faster than hashmap for lowercase English letters).

For "find all anagrams of pattern in text": sliding window of size len(pattern). Maintain a frequency difference. When all differences are 0, it's an anagram.

LC 242 (Valid Anagram), LC 438 (Find All Anagrams in a String).

### Subsequence Check

Two pointers: one on the candidate, one on the target. If characters match, advance both. Else advance only the target pointer. If the candidate pointer reaches the end, it's a subsequence.

LC 392 (Is Subsequence).

### Longest Common Prefix

Compare characters at each position across all strings. Stop at the first mismatch. Or use a trie: insert all strings, the longest single-child chain from the root is the LCP.

LC 14 (Longest Common Prefix).