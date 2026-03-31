# Extra Characters in a String

**LeetCode 2707** — [Extra Characters in a String](https://leetcode.com/problems/extra-characters-in-a-string/)

## Problem

Given a string `s` and a `dictionary` of words, break `s` into one or more non-overlapping substrings such that each substring is present in the dictionary. There may be extra characters in `s` which are not part of any substring.

**Return the minimum number of extra characters left over.**

### Example

```
s = "leetscode"
dictionary = ["leet", "code"]

Answer: 1  →  "leet" + "s"(extra) + "code"  →  1 extra char
```

---

## Step 1 — Recursive (Brute Force)

### Idea

At every index `i`, we have two choices:
1. **Skip** `s[i]` — treat it as an extra character, pay a cost of 1, move to `i+1`
2. **Match** — try all substrings `s[i..j]` that exist in the dictionary, pay 0 cost, jump to `j+1`

We want the minimum total skipped characters.

### Recurrence

```
solve(i) = minimum extra characters from index i to end of s

Base case : solve(n) = 0

solve(i) = min(
    1 + solve(i + 1),                      # skip s[i]
    min(solve(j + 1)) for j in [i, n-1]    # match s[i..j] if in dict
        where s[i..j] in dictionary
)
```

### Code

```java
import java.util.Arrays;
import java.util.HashSet;
import java.util.Set;

class Solution {
    private Set<String> wordSet;
    private int n;

    public int minExtraChar(String s, String[] dictionary) {
        wordSet = new HashSet<>(Arrays.asList(dictionary));
        n = s.length();
        return solve(s, 0);
    }

    private int solve(String s, int i) {
        if (i == n) return 0;
        // option 1: skip s[i]
        int result = 1 + solve(s, i + 1);
        // option 2: try every substring starting at i
        for (int j = i + 1; j <= n; j++) {
            if (wordSet.contains(s.substring(i, j))) {
                result = Math.min(result, solve(s, j));
            }
        }
        return result;
    }
}
```

### Complexity

| | Value |
|---|---|
| Time | O(2^n) — overlapping subproblems recomputed |
| Space | O(n) recursion stack |

**Problem**: Same subproblems are solved repeatedly → TLE for large inputs.

---

## Step 2 — Top-Down DP (Memoization)

### Idea

Cache the result of `solve(i)` so each state is computed only once.

### Code

```java
import java.util.Arrays;
import java.util.HashSet;
import java.util.Set;

class Solution {
    private Set<String> wordSet;
    private int[] memo;
    private int n;

    public int minExtraChar(String s, String[] dictionary) {
        wordSet = new HashSet<>(Arrays.asList(dictionary));
        n = s.length();
        memo = new int[n + 1];
        Arrays.fill(memo, -1);
        return solve(s, 0);
    }

    private int solve(String s, int i) {
        if (i == n) return 0;
        if (memo[i] != -1) return memo[i];
        int result = 1 + solve(s, i + 1);
        for (int j = i + 1; j <= n; j++) {
            if (wordSet.contains(s.substring(i, j))) {
                result = Math.min(result, solve(s, j));
            }
        }
        memo[i] = result;
        return result;
    }
}
```

### Bottom-Up DP (Iterative)

`dp[i]` = minimum extra characters in the first `i` characters of `s` (`s[0..i-1]`).

```
dp[0] = 0   (empty prefix, no characters)

For i from 1 to n:
    dp[i] = dp[i-1] + 1          # skip s[i-1]
    For j from 0 to i-1:
        if s[j..i-1] in dict:
            dp[i] = min(dp[i], dp[j])   # matched word s[j..i-1], no extra cost
```

#### Dry Run on Example

```
s = "leetscode",  dictionary = {"leet", "code"}

dp[0] = 0
dp[1] = dp[0]+1 = 1            ("l" not in dict)
dp[2] = dp[1]+1 = 2            ("le" not in dict)
dp[3] = dp[2]+1 = 3            ("lee" not in dict)
dp[4] = min(dp[3]+1, dp[0])    s[0..3]="leet" ✓ → dp[4] = min(4, 0) = 0
dp[5] = min(dp[4]+1) = 1       ("s" not in dict)
dp[6] = min(dp[5]+1) = 2       ("sc" not in dict)
dp[7] = min(dp[6]+1) = 3       ("sco" not in dict)
dp[8] = min(dp[7]+1) = 4       ("scod" not in dict)
dp[9] = min(dp[8]+1, dp[5])    s[5..8]="code" ✓ → dp[9] = min(5, 1) = 1

Answer = dp[9] = 1  ✓
```

#### Code

```java
import java.util.Arrays;
import java.util.HashSet;
import java.util.Set;

class Solution {
    public int minExtraChar(String s, String[] dictionary) {
        Set<String> wordSet = new HashSet<>(Arrays.asList(dictionary));
        int n = s.length();
        int[] dp = new int[n + 1];

        for (int i = 1; i <= n; i++) {
            dp[i] = dp[i - 1] + 1;       // default: skip s[i-1]
            for (int j = 0; j < i; j++) {
                if (wordSet.contains(s.substring(j, i))) {
                    dp[i] = Math.min(dp[i], dp[j]);
                }
            }
        }

        return dp[n];
    }
}
```

### Complexity

| | Value |
|---|---|
| Time | O(n² · L) — n² pairs, each substring check O(L) with hash set |
| Space | O(n) for dp array |

**Bottleneck**: inner loop checks `s[j:i] in word_set` which involves hashing an O(L) string for every pair. A Trie avoids this.

---

## Step 3 — Trie + DP (Optimal)

### Why Trie?

Instead of checking if every substring `s[j..i]` is in a hash set (O(L) per check), we traverse the Trie **backwards from position `i`**. Each step moves one character, and we get a match whenever we land on an `is_end` node — all in O(1) per character.

### Trie Structure

```
dictionary = ["leet", "code"]   — inserted in REVERSE order

root
 ├── t → e → e → l*
 └── e → d → o → c*
         (* = isEnd = true)
```

### Algorithm

For each position `i` (end of a potential word), walk **backwards** from `s[i-1]` toward `s[0]` through the Trie:
- If the current character is not in the Trie node's children → **stop early** (no word can extend further left)
- If we reach an `is_end` node at position `j` → word `s[j..i-1]` is matched → update `dp[i] = min(dp[i], dp[j])`

### Code

```java
import java.util.HashMap;
import java.util.Map;

class TrieNode {
    Map<Character, TrieNode> children = new HashMap<>();
    boolean isEnd = false;
}

class Solution {
    public int minExtraChar(String s, String[] dictionary) {
        TrieNode root = new TrieNode();
        for (String word : dictionary) {
            TrieNode node = root;
            for (int k = word.length() - 1; k >= 0; k--) {   // insert reversed
                char ch = word.charAt(k);
                node.children.putIfAbsent(ch, new TrieNode());
                node = node.children.get(ch);
            }
            node.isEnd = true;
        }

        int n = s.length();
        int[] dp = new int[n + 1];

        for (int i = 1; i <= n; i++) {
            dp[i] = dp[i - 1] + 1;       // skip s[i-1]
            TrieNode node = root;
            for (int j = i - 1; j >= 0; j--) {   // walk backwards
                char ch = s.charAt(j);
                if (!node.children.containsKey(ch)) break;   // early exit
                node = node.children.get(ch);
                if (node.isEnd) {
                    dp[i] = Math.min(dp[i], dp[j]);
                }
            }
        }

        return dp[n];
    }
}
```

### Trie Walk Visualization (for `i=4`, matching "leet")

```
i=4, walk backwards (Trie built with reversed words):
  j=3: s[3]='t' → root→t          (not end)
  j=2: s[2]='e' → root→t→e        (not end)
  j=1: s[1]='e' → root→t→e→e      (not end)
  j=0: s[0]='l' → root→t→e→e→l   (isEnd=true) → dp[4] = min(dp[4], dp[0]) ✓
```

### Complexity

| | Value |
|---|---|
| Time | O(n² + D·L) — O(n²) for DP, O(D·L) to build trie |
| Space | O(D·L) for trie + O(n) for dp |

Where `D` = number of dictionary words, `L` = average word length.

The inner loop still runs O(n) in the worst case but each step is **O(1)** (dict lookup by single char), and **early exit** prunes branches where no word starts with that prefix.

---

## Complexity Comparison

| Approach | Time | Space | Notes |
|---|---|---|---|
| Recursive (brute) | O(2^n) | O(n) | Overlapping subproblems |
| DP (memoized/iterative) | O(n² · L) | O(n) | Hash set substring check |
| Trie + DP | O(n² + D·L) | O(D·L + n) | O(1) per char, early exit |

---

## Key Insights

1. **Choice at each index** — skip (cost +1) vs match a dictionary word (cost 0) → classic DP structure.
2. **DP state** — `dp[i]` = min extra chars for prefix `s[0..i-1]` cleanly captures all needed history.
3. **Trie advantage** — scanning backwards from a fixed endpoint through the Trie is equivalent to checking all substrings ending at `i`, but stops early when no prefix matches → practical speedup.
