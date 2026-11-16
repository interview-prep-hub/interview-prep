# Dynamic Programming

> **Tier:** T1 · **Related:** [Recursion and backtracking](05-recursion-backtracking.md), [Greedy, intervals and bits](11-greedy-intervals-bits.md), [Big-O and problem solving](00-big-o-and-problem-solving.md), [STL for DSA](../03-languages/c-cpp/04-stl-for-dsa.md) · **Language:** C++17

## Quick revision
- DP applies when a problem has **overlapping subproblems** (the same smaller question is asked many times) and **optimal substructure** (the best answer is built from best answers to smaller questions).
- **Memoization** (top-down): plain recursion plus a cache. **Tabulation** (bottom-up): loops that fill a table from the base cases up.
- Framework: define the **state** in words, write the **transition**, set the **base cases**, pick an evaluation **order**, and locate the **answer**.
- Complexity = number of states x work per transition.
- **Space optimization:** if a row only reads the previous row (or the last two values), keep only those.
- 1D: Climbing Stairs, House Robber, Decode Ways, Word Break, Coin Change, LIS. 2D: grid paths, LCS, Edit Distance, knapsack.
- Kadane (maximum subarray): `cur = max(x, cur + x)`, `best = max(best, cur)`.
- 0/1 knapsack in 1D: loop capacity **downward** so each item is used at most once; upward means unlimited reuse.
- Coin Change (fewest coins) is unbounded knapsack: `dp[a] = min(dp[a], dp[a - c] + 1)`; greedy fails for coins like {1, 3, 4}.
- LIS: O(n^2) DP, or O(n log n) with a `tails` array and `lower_bound`.
- Two-sequence DPs index prefixes: `dp[i][j]` covers `a[0, i)` and `b[0, j)`, with an extra row and column of zeros.
- Use `long long` (or a guard value like `amount + 1` for infinity) to avoid overflow when adding to "infinity".
- Signals: "number of ways", "minimum/maximum cost", "can you reach/make", "longest/shortest subsequence", choices at each step that affect later options.

## 1. Concept from scratch

### 1.1 From recursion to DP
Many DP problems start as a recursive brute force that tries every choice. It becomes exponential because it solves the same subproblem again and again: naive Fibonacci recomputes `fib(n - 2)` inside `fib(n - 1)`, and so on down the tree. Two properties make DP possible:
- **Overlapping subproblems:** the recursion tree contains repeated states. (Merge sort's subproblems never repeat, which is why it is divide and conquer, not DP.)
- **Optimal substructure:** an optimal answer is assembled from optimal answers to its subproblems. Shortest paths have it; longest simple paths in general graphs do not.

Once each distinct state is solved once and stored, the cost drops from "number of paths through the tree" to "number of distinct states x work per state".

### 1.2 Memoization versus tabulation
| | Memoization (top-down) | Tabulation (bottom-up) |
|---|---|---|
| Shape | recursive function + cache (array or hash map) | loops filling a table |
| States computed | only those actually reached | every state in the table |
| Evaluation order | automatic (recursion resolves dependencies) | you must order loops so dependencies come first |
| Costs | call overhead; recursion depth can overflow the stack | no recursion; easiest to space-optimize |
| Pick it when | the state space is sparse or the order is awkward | the state space is dense and the order is obvious |

A good interview path: write the recursive brute force, add a memo to make it pass, then (if asked) convert to a table and shrink the space.

### 1.3 The state, transition, base case framework
1. **State:** say in words what `dp[i]` (or `dp[i][j]`) means, for example "the most money you can rob from the first i houses". Most DP bugs are a fuzzy state.
2. **Transition:** consider the last decision. For House Robber, house i is either skipped (`dp[i - 1]`) or robbed (`dp[i - 2] + nums[i]`).
3. **Base cases:** the smallest states that need no recursion (`dp[0]`, an empty prefix, an empty string).
4. **Order:** evaluate so that every state's dependencies are already computed.
5. **Answer:** usually `dp[n]` or `dp[m][n]`, sometimes the maximum over all states (LIS, Kadane).
6. **Complexity:** states x transitions per state; then check whether you can drop old rows.

### 1.4 Space optimization
If `dp[i]` depends only on `dp[i - 1]` and `dp[i - 2]`, keep two variables. If row r of a grid DP only reads row r - 1, keep one or two rows. In 1D knapsack-style DPs the loop direction decides whether an item is reused (upward) or used once (downward), because it determines whether `dp[w - weight]` has already been overwritten in this round. You lose the ability to reconstruct the chosen items unless you keep the full table or extra parent pointers.

### 1.5 Common DP families
| Family | Typical state | Examples |
|---|---|---|
| 1D linear | best answer for the first i elements, or ending at i | Climbing Stairs, House Robber, Decode Ways, Word Break, LIS, Kadane |
| Grid | best way to reach cell (r, c) | Unique Paths, Minimum Path Sum |
| Two sequences | answer for prefixes `a[0, i)` and `b[0, j)` | LCS, Edit Distance |
| Knapsack | items considered so far and remaining capacity or sum | 0/1 knapsack, Partition Equal Subset Sum, Coin Change |
| Interval (going deeper) | answer for the range `[l, r]` | Burst Balloons, Longest Palindromic Subsequence |
| Bitmask (going deeper) | answer for a subset encoded as bits | assignment and travelling-salesman variants with n up to about 20 |

## 2. Core templates
The four stages of the same DP, using Fibonacci:
```cpp
#include <vector>
using namespace std;

// 1. Plain recursion: O(2^n), because subproblems are recomputed.
long long fibNaive(int n) { return n < 2 ? n : fibNaive(n - 1) + fibNaive(n - 2); }

// 2. Memoization (top-down): cache each answer. O(n) time, O(n) space (memo + stack).
long long fibMemo(int n, vector<long long>& memo) {      // memo has n + 1 slots, all -1
    if (n < 2) return n;
    if (memo[n] != -1) return memo[n];
    return memo[n] = fibMemo(n - 1, memo) + fibMemo(n - 2, memo);
}

// 3. Tabulation (bottom-up): fill from the base cases up. O(n) time, O(n) space.
long long fibTable(int n) {
    if (n < 2) return n;
    vector<long long> dp(n + 1);
    dp[0] = 0;
    dp[1] = 1;
    for (int i = 2; i <= n; i++) dp[i] = dp[i - 1] + dp[i - 2];
    return dp[n];
}

// 4. Space-optimized: only the last two values are read. O(n) time, O(1) space.
long long fibRolling(int n) {
    long long a = 0, b = 1;                               // fib(0), fib(1)
    for (int i = 0; i < n; i++) {
        long long c = a + b;
        a = b;
        b = c;
    }
    return a;
}
```

## 3. Problems

### 3.1 Climbing Stairs (LeetCode 70, Easy)
**Problem (paraphrased):** you climb n steps taking 1 or 2 steps at a time. Count the distinct ways to reach the top.
**Approach:** brute force recursion `ways(n) = ways(n - 1) + ways(n - 2)` is O(2^n). State: `ways(i)` = number of ways to reach step i; the last move was either 1 or 2 steps. Only the previous two values are needed.
**Complexity:** O(n) time, O(1) space.
```cpp
class Solution {
public:
    int climbStairs(int n) {
        int prev2 = 1, prev1 = 1;          // ways(0) = 1 (stand still), ways(1) = 1
        for (int i = 2; i <= n; i++) {
            int cur = prev1 + prev2;       // last move was a 1-step or a 2-step
            prev2 = prev1;
            prev1 = cur;
        }
        return prev1;                      // n <= 45 keeps this within int
    }
};
```
**Edge cases:** n = 1. Variants: steps of size 1..k, or a cost per step (Min Cost Climbing Stairs, LeetCode 746).

### 3.2 House Robber (LeetCode 198, Medium)
**Problem (paraphrased):** houses in a row hold amounts of money; robbing two adjacent houses is not allowed. Maximize the total.
**Approach:** brute force tries every non-adjacent subset, O(2^n). State: `best(i)` = most money from the first i houses. Transition: skip house i (`best(i - 1)`) or rob it (`best(i - 2) + nums[i]`).
**Complexity:** O(n) time, O(1) space.
```cpp
class Solution {
public:
    int rob(vector<int>& nums) {
        int prev2 = 0, prev1 = 0;                   // best up to two houses back, one house back
        for (int x : nums) {
            int cur = max(prev1, prev2 + x);        // skip this house, or rob it
            prev2 = prev1;
            prev1 = cur;
        }
        return prev1;
    }
};
```
**Edge cases:** one house; two houses (take the larger); all zeros.

### 3.3 House Robber II (LeetCode 213, Medium)
**Problem (paraphrased):** same as House Robber, but the houses form a circle, so the first and last houses are adjacent.
**Approach:** the first and last house cannot both be robbed, so solve the linear problem twice, once excluding the last house and once excluding the first, and take the larger result.
**Complexity:** O(n) time, O(1) space.
```cpp
class Solution {
public:
    int rob(vector<int>& nums) {
        int n = nums.size();
        if (n == 1) return nums[0];
        return max(robRange(nums, 0, n - 2), robRange(nums, 1, n - 1));
    }
private:
    int robRange(const vector<int>& nums, int lo, int hi) {   // linear House Robber on nums[lo..hi]
        int prev2 = 0, prev1 = 0;
        for (int i = lo; i <= hi; i++) {
            int cur = max(prev1, prev2 + nums[i]);
            prev2 = prev1;
            prev1 = cur;
        }
        return prev1;
    }
};
```
**Edge cases:** one house (must be handled separately, or both ranges are empty); two houses.

### 3.4 Maximum Subarray (LeetCode 53, Medium)
**Problem (paraphrased):** find the largest sum of a non-empty contiguous subarray.
**Approach:** brute force tries every subarray with a running sum, O(n^2). **Kadane's algorithm** is a DP where `cur` = best sum of a subarray ending at the current index: either extend the previous run or start fresh at this element.
**Complexity:** O(n) time, O(1) space.
```cpp
class Solution {
public:
    int maxSubArray(vector<int>& nums) {
        int best = nums[0], cur = nums[0];         // cur = best sum of a subarray ending here
        for (int i = 1; i < (int)nums.size(); i++) {
            cur = max(nums[i], cur + nums[i]);     // extend the run or restart here
            best = max(best, cur);
        }
        return best;
    }
};
```
**Edge cases:** all negative numbers (the answer is the largest single element, which is why `best` starts at `nums[0]` rather than 0). A divide-and-conquer O(n log n) solution exists but is rarely better.

### 3.5 Coin Change (LeetCode 322, Medium)
**Problem (paraphrased):** with unlimited coins of given denominations, find the fewest coins that add up to an amount, or -1 if impossible.
**Approach:** greedy (largest coin first) fails: with coins {1, 3, 4} and amount 6 it picks 4 + 1 + 1, but 3 + 3 is better. Brute force recursion tries every coin at every step, exponential. State: `dp[a]` = fewest coins for amount a; transition: the last coin c gives `dp[a - c] + 1`. Use `amount + 1` as infinity so adding 1 never overflows.
**Complexity:** O(amount x number of coins) time, O(amount) space.
```cpp
class Solution {
public:
    int coinChange(vector<int>& coins, int amount) {
        const int INF = amount + 1;                    // more coins than any real answer
        vector<int> dp(amount + 1, INF);               // dp[a] = fewest coins to make a
        dp[0] = 0;
        for (int a = 1; a <= amount; a++)
            for (int c : coins)
                if (c <= a) dp[a] = min(dp[a], dp[a - c] + 1);
        return dp[amount] == INF ? -1 : dp[amount];
    }
};
```
**Edge cases:** amount 0 (0 coins); no combination works (-1). Counting the number of ways instead (Coin Change II) loops coins outside and amounts inside to avoid counting orderings twice.

### 3.6 Longest Increasing Subsequence (LeetCode 300, Medium)
**Problem (paraphrased):** return the length of the longest strictly increasing subsequence (not necessarily contiguous).
**Approach:** brute force checks all 2^n subsequences. DP: `dp[i]` = length of the longest increasing subsequence ending at i, computed from every earlier smaller element, O(n^2). The O(n log n) idea keeps `tails[k]` = the smallest possible tail of an increasing subsequence of length k + 1; `tails` stays sorted, so each new value either extends it or replaces the first tail `>= x` (found with `lower_bound`). Its length is the answer, although `tails` itself is not necessarily a real subsequence.
**Complexity:** O(n log n) time, O(n) space (the quadratic version is O(n^2)).
```cpp
class Solution {
public:
    int lengthOfLIS(vector<int>& nums) {
        vector<int> tails;
        for (int x : nums) {
            auto it = lower_bound(tails.begin(), tails.end(), x);   // first tail >= x
            if (it == tails.end()) tails.push_back(x);              // x extends the longest run
            else *it = x;                                           // a smaller tail for that length
        }
        return tails.size();
    }

    int lengthOfLISQuadratic(vector<int>& nums) {
        int n = nums.size(), best = 0;
        vector<int> dp(n, 1);                                       // dp[i] = LIS ending at i
        for (int i = 0; i < n; i++) {
            for (int j = 0; j < i; j++)
                if (nums[j] < nums[i]) dp[i] = max(dp[i], dp[j] + 1);
            best = max(best, dp[i]);
        }
        return best;
    }
};
```
**Edge cases:** all equal values (1, because the sequence must be strictly increasing); strictly decreasing input (1). For non-decreasing subsequences, use `upper_bound`.

### 3.7 Word Break (LeetCode 139, Medium)
**Problem (paraphrased):** decide whether a string can be split into a sequence of dictionary words (words may be reused).
**Approach:** brute force recursion tries every prefix that is a word and recurses on the rest, exponential for inputs like `"aaaa...b"`. State: `dp[i]` = the prefix `s[0, i)` can be segmented; it is true if some `j < i` has `dp[j]` true and `s[j, i)` in the dictionary. Limit j to the longest word length.
**Complexity:** O(n x L x L) time for maximum word length L (each substring costs O(L) to build and hash), O(n) space plus the dictionary.
```cpp
class Solution {
public:
    bool wordBreak(string s, vector<string>& wordDict) {
        unordered_set<string> dict(wordDict.begin(), wordDict.end());
        int n = s.size(), maxLen = 0;
        for (const string& w : wordDict) maxLen = max(maxLen, (int)w.size());
        vector<bool> dp(n + 1, false);                 // dp[i]: s[0, i) can be segmented
        dp[0] = true;                                  // the empty prefix
        for (int i = 1; i <= n; i++)
            for (int j = max(0, i - maxLen); j < i && !dp[i]; j++)
                if (dp[j] && dict.count(s.substr(j, i - j))) dp[i] = true;   // last word is s[j, i)
        return dp[n];
    }
};
```
**Edge cases:** reused words (`"applepenapple"`); a near miss like `"catsandog"`. A trie over the dictionary avoids building substrings.

### 3.8 Decode Ways (LeetCode 91, Medium)
**Problem (paraphrased):** letters A to Z are encoded as 1 to 26. Count the ways to decode a digit string (a part like `06` is invalid).
**Approach:** brute force recursion tries one or two digits at each position, exponential. State: `dp[i]` = ways to decode the prefix of length i. The last piece is either one digit (valid if it is not `0`) or two digits (valid if between 10 and 26). Only two previous values are needed.
**Complexity:** O(n) time, O(1) space.
```cpp
class Solution {
public:
    int numDecodings(string s) {
        int n = s.size();
        int prev2 = 1;                                    // dp[0]: the empty prefix
        int prev1 = (s[0] != '0') ? 1 : 0;                // dp[1]
        for (int i = 2; i <= n; i++) {
            int cur = 0;
            if (s[i - 1] != '0') cur += prev1;                         // last digit alone: 1..9
            int two = (s[i - 2] - '0') * 10 + (s[i - 1] - '0');
            if (s[i - 2] != '0' && two <= 26) cur += prev2;           // last two digits: 10..26
            prev2 = prev1;
            prev1 = cur;
        }
        return prev1;
    }
};
```
**Edge cases:** leading zero (0 ways); `"10"` (1 way); `"27"` (1 way); `"100"` (0 ways).

### 3.9 Unique Paths (LeetCode 62, Medium)
**Problem (paraphrased):** a robot at the top-left of an m x n grid moves only right or down. Count the paths to the bottom-right corner.
**Approach:** brute force recursion explores every path, exponential. State: `paths(r, c)` = number of ways to reach cell (r, c), which is the paths into the cell above plus the paths into the cell on the left. A single row is enough: `dp[c]` already holds the value from the row above when you add `dp[c - 1]`. The closed form is the binomial coefficient C(m + n - 2, m - 1).
**Complexity:** O(m * n) time, O(n) space.
```cpp
class Solution {
public:
    int uniquePaths(int m, int n) {
        vector<int> dp(n, 1);                      // first row: one way to reach each cell
        for (int r = 1; r < m; r++)
            for (int c = 1; c < n; c++)
                dp[c] += dp[c - 1];                // from above (old dp[c]) + from the left
        return dp[n - 1];
    }
};
```
**Edge cases:** a single row or column (1 path). With obstacles (LeetCode 63), set `dp[c] = 0` on blocked cells.

### 3.10 Longest Common Subsequence (LeetCode 1143, Medium)
**Problem (paraphrased):** return the length of the longest sequence of characters that appears, in order but not necessarily contiguously, in both strings.
**Approach:** brute force enumerates the 2^m subsequences of one string and checks each against the other. State: `dp[i][j]` = LCS of the prefixes `a[0, i)` and `b[0, j)`. If the last characters match, extend `dp[i - 1][j - 1]`; otherwise drop one character from either string.
**Complexity:** O(m * n) time, O(m * n) space (O(min(m, n)) with two rows).
```cpp
class Solution {
public:
    int longestCommonSubsequence(string a, string b) {
        int m = a.size(), n = b.size();
        vector<vector<int>> dp(m + 1, vector<int>(n + 1, 0));   // row/column 0 = empty prefix
        for (int i = 1; i <= m; i++)
            for (int j = 1; j <= n; j++)
                dp[i][j] = (a[i - 1] == b[j - 1]) ? dp[i - 1][j - 1] + 1
                                                  : max(dp[i - 1][j], dp[i][j - 1]);
        return dp[m][n];
    }
};
```
**Edge cases:** no common characters (0); identical strings (their length). Diff tools build on the same idea: the lines a diff keeps unchanged are essentially a longest common subsequence of the two files.

### 3.11 0/1 Knapsack (classic, Medium)
**Problem (paraphrased):** each item has a weight and a value and can be taken at most once. Maximize the total value without exceeding a weight capacity W.
**Approach:** brute force tries all 2^n subsets. State: `dp[i][w]` = best value using the first i items with capacity w; either skip item i or take it (`dp[i - 1][w - weight] + value`). Since row i only reads row i - 1, one array suffices if capacity is looped **downward**, so `dp[w - weight]` still holds the previous row's value (looping upward would let an item be taken repeatedly, which is the unbounded knapsack).
**Complexity:** O(n * W) time, O(W) space. That is pseudo-polynomial: polynomial in the numeric value W, not in the input's size in bits.
```cpp
class Solution {
public:
    long long knapsack(const vector<int>& weight, const vector<int>& value, int capacity) {
        vector<long long> dp(capacity + 1, 0);              // dp[w] = best value within weight w
        for (int i = 0; i < (int)weight.size(); i++)
            for (int w = capacity; w >= weight[i]; w--)      // downward: item i used at most once
                dp[w] = max(dp[w], dp[w - weight[i]] + value[i]);
        return dp[capacity];
    }
};
```
**Edge cases:** an item heavier than the capacity (the inner loop never runs for it); capacity 0. To list the chosen items, keep the 2D table and walk back from `dp[n][W]`.

### 3.12 Partition Equal Subset Sum (LeetCode 416, Medium)
**Problem (paraphrased):** decide whether an array of positive integers can be split into two groups with equal sums.
**Approach:** brute force tries all 2^n assignments. If the total is odd, the answer is no; otherwise ask whether some subset sums to total / 2. That is a 0/1 knapsack on booleans: `can[s]` = some subset of the items seen so far sums to s, updated with a downward loop.
**Complexity:** O(n * total) time, O(total) space.
```cpp
class Solution {
public:
    bool canPartition(vector<int>& nums) {
        int total = accumulate(nums.begin(), nums.end(), 0);
        if (total % 2 != 0) return false;                // odd totals cannot split evenly
        int target = total / 2;
        vector<bool> can(target + 1, false);             // can[s]: some subset sums to s
        can[0] = true;
        for (int x : nums)
            for (int s = target; s >= x; s--)            // downward: each number used once
                if (can[s - x]) can[s] = true;
        return can[target];
    }
};
```
**Edge cases:** a single element (false); one element equal to half the total. A `bitset` version, `bits |= bits << x`, does the same in about total / 64 word operations per item.

### 3.13 Edit Distance (LeetCode 72, Medium)
**Problem (paraphrased):** return the minimum number of single-character inserts, deletes and replacements that turn one string into another.
**Approach:** brute force recursion tries all three operations at each mismatch, exponential. State: `dp[i][j]` = edits to turn `a[0, i)` into `b[0, j)`. Base cases: turning a prefix into an empty string takes i deletions, and building `b[0, j)` from nothing takes j insertions. If the last characters match, no edit is needed; otherwise take 1 + the best of replace, delete or insert.
**Complexity:** O(m * n) time, O(m * n) space (two rows suffice).
```cpp
class Solution {
public:
    int minDistance(string a, string b) {
        int m = a.size(), n = b.size();
        vector<vector<int>> dp(m + 1, vector<int>(n + 1));
        for (int i = 0; i <= m; i++) dp[i][0] = i;                 // delete all i characters
        for (int j = 0; j <= n; j++) dp[0][j] = j;                 // insert all j characters
        for (int i = 1; i <= m; i++)
            for (int j = 1; j <= n; j++) {
                if (a[i - 1] == b[j - 1]) dp[i][j] = dp[i - 1][j - 1];   // matching characters are free
                else dp[i][j] = 1 + min({dp[i - 1][j - 1],             // replace
                                         dp[i - 1][j],                 // delete from a
                                         dp[i][j - 1]});               // insert into a
            }
        return dp[m][n];
    }
};
```
**Edge cases:** one empty string (the other's length); identical strings (0). This is the Levenshtein distance behind spell checkers and fuzzy search; PostgreSQL ships a `levenshtein` function in its `fuzzystrmatch` extension.

## Common pitfalls
- Starting to code before stating what `dp[i]` means; a vague state leads to wrong transitions.
- Off-by-one between "first i elements" (prefix length) and "ending at index i"; pick one and size the table accordingly (`n + 1` for prefixes).
- Wrong loop direction in 1D knapsack DPs: upward reuses items, downward uses each once.
- Overflow when "infinity" is `INT_MAX` and you add 1; use a safe sentinel like `amount + 1` or `long long`.
- Forgetting base cases such as `dp[0] = 1` for counting problems or `dp[0] = true` for Word Break.
- Using greedy where DP is required (Coin Change with arbitrary denominations).
- Deep memoized recursion on large n overflowing the stack; convert to tabulation.
- Optimizing space before the solution works, then being unable to reconstruct the answer when asked.

## How to talk about this in an interview
- Use the five-step script out loud: "My state is... The last choice is... so the transition is... Base case... The answer is dp of n... That's n states with O(1) work, so O(n)." Interviewers reward the clear state definition more than the final code.
- Start from the brute-force recursion, point out the repeated subproblem in the recursion tree, then add the memo. This is the same progression you would show students in a Data Structures lab when introducing DP after recursion.
- > **Fill in:** a recursion or DP exercise you helped students with at PUCIT, and where they usually got stuck (base cases, table indexing, loop order).
- Real-world links: diff tools and code review views rely on LCS-style alignment, spell checkers and fuzzy search use edit distance, and pricing or bundling logic (choose the best combination under a budget) is knapsack-shaped.

## Interview questions

**Q1. Memoization versus tabulation: what is the difference, and when do you prefer each?**
Memoization keeps the natural recursion and caches results, so it computes only reachable states and the dependency order takes care of itself, but it pays call overhead and risks stack overflow on deep inputs. Tabulation fills a table iteratively in an order you choose, has no recursion, and makes space optimization easy, but it computes every state. Prefer memoization for sparse or awkward state spaces and tabulation for dense ones or when depth is large.
*Follow-up:* Same time complexity? Yes, both are O(states x transitions); constants and memory differ.

**Q2. How do you recognize that a problem needs dynamic programming?**
It asks for a count, a minimum or maximum, or a yes/no over many combinations; the brute force makes choices whose results depend on earlier choices; and the recursion tree repeats subproblems. If a locally best choice can be proven safe, greedy may suffice; if the subproblems do not overlap, it is divide and conquer.
*Follow-up:* Name a problem that looks like DP but is greedy. Jump Game: tracking the farthest reachable index is enough.

**Q3. DP versus greedy versus divide and conquer?**
Divide and conquer splits into independent subproblems that never repeat (merge sort). DP handles overlapping subproblems by storing their answers (LCS). Greedy commits to one locally optimal choice per step and never revisits it, which is correct only when you can prove that choice is safe (interval scheduling by earliest end).
*Follow-up:* Example where greedy fails but DP works? Coin Change with coins {1, 3, 4} and amount 6.

**Q4. How do you reduce the memory of a DP, and what do you lose?**
Look at which states each state reads: if `dp[i]` uses only the last one or two entries, keep variables; if row i uses only row i - 1, keep one or two rows (loop direction matters for in-place updates). You lose the full table, so reconstructing the actual choices needs extra bookkeeping or a recomputation.
*Follow-up:* Space for LCS with rolling rows? O(min(m, n)).

**Q5. How do you reconstruct the actual solution, not just its value?**
Keep the full table (or a parent pointer per state recording which transition won), then walk backward from the answer state: for LCS, move diagonally on a match and otherwise toward the larger neighbor, collecting matched characters in reverse.
*Follow-up:* What if there are several optimal solutions? Any consistent tie-break works; enumerating all of them can be exponential.

**Q6. Why does the 0/1 knapsack inner loop run backward, and what changes for the unbounded version?**
Going from high to low capacity means `dp[w - weight]` still holds the previous item's row, so the current item is counted at most once. Going from low to high lets the update see values that already include the current item, which allows unlimited copies: exactly the unbounded knapsack (and Coin Change).
*Follow-up:* In Coin Change II (counting ways), why are coins in the outer loop? So each combination is counted once regardless of order; swapping the loops counts ordered sequences instead.

**Q7. Is the knapsack DP polynomial?**
It is O(n * W), which is pseudo-polynomial: polynomial in the numeric capacity W, but W can be exponential in the number of bits needed to write it. The general 0/1 knapsack problem is NP-hard, so no known algorithm is polynomial in the input size.
*Follow-up:* What do you do when W is huge but n is small? Meet in the middle (enumerate subsets of each half, about 2^(n/2) each) or a DP over values instead of weights.

**Q8. Where does dynamic programming show up in real software?**
Text diffs and version control (LCS-like alignment), spell checkers and fuzzy matching (edit distance), route planning (shortest paths), text layout and line breaking, resource allocation and pricing bundles (knapsack variants), and sequence alignment in bioinformatics.
*Follow-up:* Which of these could you meet in a Rails or Postgres app? Fuzzy search with Postgres `levenshtein` or trigram similarity, and showing record history diffs.
