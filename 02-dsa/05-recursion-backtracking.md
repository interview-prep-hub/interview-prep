# Recursion and Backtracking

> **Tier:** T1 · **Related:** [Big-O and problem solving](00-big-o-and-problem-solving.md), [Trees and BSTs](07-trees-bst.md), [Dynamic programming](10-dynamic-programming.md), [STL for DSA](../03-languages/c-cpp/04-stl-for-dsa.md) · **Language:** C++17

## Quick revision
- Every recursive function needs a **base case** (stop) and a **recursive case** that moves strictly toward it.
- Trust the recursive call ("leap of faith"): assume it solves the smaller problem, then combine.
- Each call pushes a stack frame; space = maximum depth x frame size. Deep recursion (around 1e5 frames or more) risks stack overflow.
- C++ does not guarantee tail-call optimization; compilers may do it at higher optimization levels, but never rely on it.
- Recursion tree: time = number of nodes x work per node; space = tree height.
- **Backtracking template:** if complete, record; for each choice: prune if invalid, choose, explore, unchoose.
- Pass the path by reference and undo after the call (`push_back` then `pop_back`); copying the path at every level adds an extra factor of n.
- Subsets O(n * 2^n), permutations O(n * n!), combinations of size k O(k * C(n, k)).
- Duplicates in the input: sort, then skip `nums[i] == nums[i - 1]` when `i > start` (same value at the same depth).
- "Reuse allowed" recurses with `i`; "each item once" recurses with `i + 1`; "order matters" (permutations) loops from 0 with a `used` array.
- Prune early: sort and `break` when a candidate exceeds the remaining target; track used columns/diagonals in sets.
- Signals: "all combinations / subsets / permutations", "generate every valid", n up to about 20, grids with paths, puzzles.

## 1. Concept from scratch

### 1.1 Base case, recursive case, leap of faith
A recursive function solves a problem by solving a smaller instance of the same problem. To write one, answer three questions: what is the smallest input I can answer directly (base case)? How do I shrink the input (recursive case)? How do I combine the smaller answer with the current step?

```cpp
long long factorial(int n) {
    if (n <= 1) return 1;            // base case
    return n * factorial(n - 1);     // recursive case: trust factorial(n - 1)
}
```
A missing or unreachable base case means infinite recursion and, in C++, a stack overflow crash.

### 1.2 The call stack
Each call gets a **frame** on the thread's stack holding its parameters, locals and return address. `factorial(3)` pushes frames for 3, 2 and 1; the base case returns 1, then each frame multiplies and pops. Recursion depth is the number of frames alive at once. Default stack sizes are limited (typically around 1 MB on Windows and 8 MB on many Linux setups), so a recursive DFS over a 1e6-node path graph can crash where an iterative version with an explicit `std::stack` (heap memory) is fine.

### 1.3 Recursion trees
Draw each call as a node and its recursive calls as children.
- **Time** = number of nodes x work per node.
- **Space** = height of the tree (the deepest chain of frames), plus any memo tables.

| Problem | Tree shape | Results | Time |
|---|---|---|---|
| Subsets | depth n, include/exclude each item | 2^n | O(n * 2^n) |
| Permutations | depth n, n, n - 1, ... choices | n! | O(n * n!) |
| Combinations of size k | depth k | C(n, k) | O(k * C(n, k)) |
| Phone letter combinations | depth n, up to 4 branches | up to 4^n | O(n * 4^n) |
| Word search | m * n starts, at most 3 branches after the first step | n/a | O(m * n * 3^L) |

The extra factor of n (or k, L) is the cost of copying each finished path into the result.

### 1.4 Backtracking: choose, explore, unchoose
Backtracking is depth-first search over a tree of partial solutions. You extend a partial solution one choice at a time, and abandon it as soon as it cannot lead to a valid answer. The state is shared and mutated, so every change made before the recursive call is undone after it returns.

```text
backtrack(state):
    if state is a complete solution: record a copy; return
    for each choice available from state:
        if choice breaks a constraint: skip it      (prune)
        apply choice to state                       (choose)
        backtrack(state)                            (explore)
        undo choice                                 (unchoose)
```

### 1.5 Pruning and duplicate handling
- **Prune** as early as possible: sort candidates so you can `break` once one exceeds the remaining target; check constraints (used columns, diagonals) before recursing rather than validating complete boards at the end.
- **Duplicates in the input** (Subsets II, Combination Sum II): sort, then at each depth skip a value equal to the previous one at the same depth (`i > start && nums[i] == nums[i - 1]`). Using the duplicate deeper in the tree is still allowed, so valid combinations like `[1, 1]` survive.
- **Reuse rules:** recurse with `i` when an item may be reused, with `i + 1` when each item is used once; loop from 0 with a `used` array when order matters.

## 2. Core template
```cpp
#include <vector>
using namespace std;

// All size-k combinations of 1..n: a concrete choose / explore / unchoose skeleton.
void combine(int start, int n, int k, vector<int>& path, vector<vector<int>>& out) {
    if ((int)path.size() == k) { out.push_back(path); return; }   // goal: record a copy
    for (int x = start; x <= n; x++) {
        if (n - x + 1 < k - (int)path.size()) break;              // prune: too few numbers left
        path.push_back(x);                                        // choose
        combine(x + 1, n, k, path, out);                          // explore (each number once)
        path.pop_back();                                          // unchoose
    }
}

vector<vector<int>> combinations(int n, int k) {
    vector<vector<int>> out;
    vector<int> path;
    combine(1, n, k, path, out);
    return out;
}
```
Recursion versus iteration: any recursion can be rewritten with an explicit stack of states. Keep recursion when depth is small and the code is clearer (backtracking, balanced trees); switch to iteration when depth can reach the size of the input (linked lists, path-shaped graphs, big grids).

## 3. Problems

### 3.1 Pow(x, n) (LeetCode 50, Medium)
**Problem (paraphrased):** compute x raised to an integer power n, where n can be negative or as small as the minimum 32-bit integer.
**Approach:** brute force multiplies x by itself |n| times, O(n), too slow for n near 2^31. Fast exponentiation: x^n = (x^(n/2))^2, times x when n is odd. Compute the half power once and reuse it; calling it twice would bring back O(n). Widen n to `long long` before negating, because `-INT_MIN` overflows `int`.
**Complexity:** O(log n) time, O(log n) stack.
```cpp
class Solution {
public:
    double myPow(double x, int n) {
        long long e = n;                     // widen first: -INT_MIN does not fit in int
        if (e < 0) { x = 1 / x; e = -e; }
        return power(x, e);
    }
private:
    double power(double x, long long e) {
        if (e == 0) return 1.0;
        double half = power(x, e / 2);       // computed once and reused
        return (e % 2 == 0) ? half * half : half * half * x;
    }
};
```
**Edge cases:** n = 0 (1.0); n = INT_MIN; x = 0 with a positive n. An iterative version squares x and halves e in a loop with O(1) space.

### 3.2 Subsets (LeetCode 78, Medium)
**Problem (paraphrased):** return every subset (the power set) of an array of distinct integers, in any order.
**Approach:** iterating bitmasks from 0 to 2^n - 1 and taking the set bits works and is a fine brute force. The backtracking version records the current path at every node of the tree, then tries each later element as the next pick.
**Complexity:** O(n * 2^n) time, O(n) recursion depth plus the output.
```cpp
class Solution {
public:
    vector<vector<int>> subsets(vector<int>& nums) {
        vector<vector<int>> result;
        vector<int> current;
        build(nums, 0, current, result);
        return result;
    }
private:
    void build(const vector<int>& nums, int start, vector<int>& current, vector<vector<int>>& result) {
        result.push_back(current);                     // every node of the tree is a subset
        for (int i = start; i < (int)nums.size(); i++) {
            current.push_back(nums[i]);                // choose
            build(nums, i + 1, current, result);       // explore
            current.pop_back();                        // unchoose
        }
    }
};
```
**Edge cases:** the empty subset is always included; a single element gives two subsets.

### 3.3 Subsets II (LeetCode 90, Medium)
**Problem (paraphrased):** like Subsets, but the input may contain duplicates and the output must not contain the same subset twice.
**Approach:** brute force generates all subsets and removes repeats with a `set<vector<int>>` after sorting each one, wasting time on duplicates. Sort first, then skip a value equal to its predecessor at the same depth.
**Complexity:** O(n * 2^n) time, O(n) depth.
```cpp
class Solution {
public:
    vector<vector<int>> subsetsWithDup(vector<int>& nums) {
        sort(nums.begin(), nums.end());                // equal values become neighbors
        vector<vector<int>> result;
        vector<int> current;
        build(nums, 0, current, result);
        return result;
    }
private:
    void build(const vector<int>& nums, int start, vector<int>& current, vector<vector<int>>& result) {
        result.push_back(current);
        for (int i = start; i < (int)nums.size(); i++) {
            if (i > start && nums[i] == nums[i - 1]) continue;   // same value at the same depth: skip
            current.push_back(nums[i]);
            build(nums, i + 1, current, result);
            current.pop_back();
        }
    }
};
```
**Edge cases:** all values equal (`[2, 2, 2]` gives 4 subsets); no duplicates (behaves like Subsets).

### 3.4 Permutations (LeetCode 46, Medium)
**Problem (paraphrased):** return all orderings of an array of distinct integers.
**Approach:** order matters, so every position may take any unused element. Keep a `used` flag per index. Alternatives: the swap-based version, or `std::next_permutation` on a sorted copy.
**Complexity:** O(n * n!) time, O(n) depth plus the output.
```cpp
class Solution {
public:
    vector<vector<int>> permute(vector<int>& nums) {
        vector<vector<int>> result;
        vector<int> current;
        vector<bool> used(nums.size(), false);
        build(nums, used, current, result);
        return result;
    }
private:
    void build(const vector<int>& nums, vector<bool>& used, vector<int>& current,
               vector<vector<int>>& result) {
        if (current.size() == nums.size()) { result.push_back(current); return; }
        for (int i = 0; i < (int)nums.size(); i++) {
            if (used[i]) continue;
            used[i] = true;
            current.push_back(nums[i]);
            build(nums, used, current, result);
            current.pop_back();                        // undo both changes
            used[i] = false;
        }
    }
};
```
**Edge cases:** one element. With duplicates (Permutations II, LeetCode 47), sort and skip `nums[i] == nums[i - 1] && !used[i - 1]`.

### 3.5 Combination Sum (LeetCode 39, Medium)
**Problem (paraphrased):** given distinct positive candidates and a target, list every combination (as a multiset) that sums to the target; a candidate may be used any number of times.
**Approach:** brute force enumerates all multisets up to the target and filters them. Backtracking builds combinations in non-decreasing index order (to avoid permutations of the same multiset), recursing with `i` to allow reuse. Sorting lets you `break` once a candidate exceeds what remains.
**Complexity:** exponential, roughly O(n^(T/m)) for target T and smallest candidate m; O(T/m) depth.
```cpp
class Solution {
public:
    vector<vector<int>> combinationSum(vector<int>& candidates, int target) {
        sort(candidates.begin(), candidates.end());       // enables the early break
        vector<vector<int>> result;
        vector<int> current;
        build(candidates, 0, target, current, result);
        return result;
    }
private:
    void build(const vector<int>& c, int start, int remaining, vector<int>& current,
               vector<vector<int>>& result) {
        if (remaining == 0) { result.push_back(current); return; }
        for (int i = start; i < (int)c.size(); i++) {
            if (c[i] > remaining) break;                  // sorted: later candidates are too big too
            current.push_back(c[i]);
            build(c, i, remaining - c[i], current, result);   // i, not i + 1: reuse allowed
            current.pop_back();
        }
    }
};
```
**Edge cases:** no combination works (empty result); a target equal to one candidate.

### 3.6 Combination Sum II (LeetCode 40, Medium)
**Problem (paraphrased):** candidates may contain duplicates and each position can be used at most once; list every distinct combination that sums to the target.
**Approach:** same skeleton as 3.5, with two changes: recurse with `i + 1` (each item once) and skip equal values at the same depth so `[1, 7]` is not reported twice when there are two 1s.
**Complexity:** O(n * 2^n) time in the worst case, O(n) depth.
```cpp
class Solution {
public:
    vector<vector<int>> combinationSum2(vector<int>& candidates, int target) {
        sort(candidates.begin(), candidates.end());
        vector<vector<int>> result;
        vector<int> current;
        build(candidates, 0, target, current, result);
        return result;
    }
private:
    void build(const vector<int>& c, int start, int remaining, vector<int>& current,
               vector<vector<int>>& result) {
        if (remaining == 0) { result.push_back(current); return; }
        for (int i = start; i < (int)c.size(); i++) {
            if (i > start && c[i] == c[i - 1]) continue;  // duplicate choice at this depth
            if (c[i] > remaining) break;
            current.push_back(c[i]);
            build(c, i + 1, remaining - c[i], current, result);   // i + 1: each item once
            current.pop_back();
        }
    }
};
```
**Edge cases:** many duplicates such as `[1, 1, 1, 1]`; no valid combination.

### 3.7 Letter Combinations of a Phone Number (LeetCode 17, Medium)
**Problem (paraphrased):** each digit 2-9 maps to letters as on a phone keypad. Return every letter string a digit string could represent.
**Approach:** this is a Cartesian product, so the brute force and the optimal solution coincide: one recursion level per digit, one branch per letter. An iterative version grows a result list digit by digit.
**Complexity:** O(n * 4^n) time, O(n) depth.
```cpp
class Solution {
public:
    vector<string> letterCombinations(string digits) {
        if (digits.empty()) return {};
        vector<string> result;
        string current;
        build(digits, 0, current, result);
        return result;
    }
private:
    const vector<string> keys{"", "", "abc", "def", "ghi", "jkl", "mno", "pqrs", "tuv", "wxyz"};

    void build(const string& digits, int i, string& current, vector<string>& result) {
        if (i == (int)digits.size()) { result.push_back(current); return; }
        for (char letter : keys[digits[i] - '0']) {
            current.push_back(letter);
            build(digits, i + 1, current, result);
            current.pop_back();
        }
    }
};
```
**Edge cases:** empty input returns an empty list, not `[""]`; digits 7 and 9 have four letters.

### 3.8 Generate Parentheses (LeetCode 22, Medium)
**Problem (paraphrased):** list every balanced string made of n pairs of parentheses.
**Approach:** brute force generates all 2^(2n) strings and validates each. Backtracking only builds valid prefixes: add `(` while fewer than n are open, add `)` while it would close an open one.
**Complexity:** proportional to the number of answers, the n-th Catalan number (about 4^n / n^1.5), times n to copy each.
```cpp
class Solution {
public:
    vector<string> generateParenthesis(int n) {
        vector<string> result;
        string current;
        build(n, 0, 0, current, result);
        return result;
    }
private:
    void build(int n, int open, int close, string& current, vector<string>& result) {
        if ((int)current.size() == 2 * n) { result.push_back(current); return; }
        if (open < n) {                                 // can still open a new pair
            current.push_back('(');
            build(n, open + 1, close, current, result);
            current.pop_back();
        }
        if (close < open) {                             // can only close what is open
            current.push_back(')');
            build(n, open, close + 1, current, result);
            current.pop_back();
        }
    }
};
```
**Edge cases:** n = 1 gives `["()"]`.

### 3.9 Word Search (LeetCode 79, Medium)
**Problem (paraphrased):** decide whether a word can be traced through a letter grid by moving between horizontally or vertically adjacent cells, never reusing a cell.
**Approach:** try every start cell and run a DFS that matches one character per step. Mark the cell as used during the current path (overwrite it with `#`) and restore it when backtracking. Pruning ideas: return false early if the grid lacks enough of some letter; start from the rarer end of the word.
**Complexity:** O(m * n * 3^L) time for word length L, O(L) recursion depth.
```cpp
class Solution {
public:
    bool exist(vector<vector<char>>& board, string word) {
        for (int r = 0; r < (int)board.size(); r++)
            for (int c = 0; c < (int)board[0].size(); c++)
                if (dfs(board, word, 0, r, c)) return true;
        return false;
    }
private:
    bool dfs(vector<vector<char>>& b, const string& word, int i, int r, int c) {
        if (i == (int)word.size()) return true;                       // matched every character
        if (r < 0 || c < 0 || r >= (int)b.size() || c >= (int)b[0].size() || b[r][c] != word[i])
            return false;
        char saved = b[r][c];
        b[r][c] = '#';                                                // choose: mark as used
        bool found = dfs(b, word, i + 1, r + 1, c) || dfs(b, word, i + 1, r - 1, c) ||
                     dfs(b, word, i + 1, r, c + 1) || dfs(b, word, i + 1, r, c - 1);
        b[r][c] = saved;                                              // unchoose: restore
        return found;
    }
};
```
**Edge cases:** a one-cell grid; a word longer than the number of cells; paths that would need to revisit a cell.

### 3.10 N-Queens (LeetCode 51, Hard)
**Problem (paraphrased):** place n queens on an n x n board so no two share a row, column or diagonal; return every arrangement as rows of `.` and `Q`.
**Approach:** brute force tries all ways to choose n cells (or n^n row placements) and validates. Place one queen per row instead, tracking used columns and both diagonal directions in O(1) lookups: cells on one diagonal share `r - c` and on the other share `r + c`.
**Complexity:** roughly O(n!) time (pruning keeps it far below n^n), O(n^2) space for the board.
```cpp
class Solution {
public:
    vector<vector<string>> solveNQueens(int n) {
        vector<vector<string>> result;
        vector<string> board(n, string(n, '.'));
        vector<bool> col(n, false), diag(2 * n - 1, false), anti(2 * n - 1, false);
        place(0, n, board, col, diag, anti, result);
        return result;
    }
private:
    void place(int r, int n, vector<string>& board, vector<bool>& col, vector<bool>& diag,
               vector<bool>& anti, vector<vector<string>>& result) {
        if (r == n) { result.push_back(board); return; }
        for (int c = 0; c < n; c++) {
            int d = r - c + n - 1, a = r + c;            // shift r - c into the range 0..2n-2
            if (col[c] || diag[d] || anti[a]) continue;  // attacked: prune
            board[r][c] = 'Q';
            col[c] = true; diag[d] = true; anti[a] = true;
            place(r + 1, n, board, col, diag, anti, result);
            board[r][c] = '.';
            col[c] = false; diag[d] = false; anti[a] = false;
        }
    }
};
```
**Edge cases:** n = 1 (one solution); n = 2 and n = 3 (no solutions).

## Common pitfalls
- A base case that is never reached (for example `n == 0` when n can jump past zero), causing a stack overflow.
- Forgetting to undo state after the recursive call: a missing `pop_back`, `used[i] = false`, or cell restore.
- Pushing a reference to the shared path into results, then mutating it. In C++ `result.push_back(current)` copies, which is what you want.
- Passing vectors by value into the recursive helper, multiplying the cost by n.
- Skipping duplicates with `i > 0` instead of `i > start`, which wrongly drops valid combinations like `[1, 1, 6]`.
- Recomputing the same subproblem (as in `power(x, e / 2) * power(x, e / 2)`), which makes the tree exponential. If subproblems repeat, memoize (see [Dynamic programming](10-dynamic-programming.md)).
- Integer overflow on negation (`-n` when n is `INT_MIN`).

## How to talk about this in an interview
- Narrate the tree: "At each level I decide one element; the branches are include or exclude, so there are 2^n leaves." Then state time as leaves x copy cost and space as depth.
- Say the template explicitly: "choose, explore, unchoose", and point to the line that undoes each change. As a Data Structures lab TA you likely traced recursive calls with students; do the same on a small input such as n = 3 before coding.
- > **Fill in:** a recursion topic from your PUCIT labs (tree traversal, Tower of Hanoi, recursive linked-list code) and how you helped students trace the call stack.
- Real-world link: recursive descent shows up in rendering nested comments or category trees, walking JSON payloads from third-party APIs, and permission checks over nested resources. Mention the stack-depth risk for user-controlled nesting.

## Interview questions

**Q1. Recursion versus iteration: when do you use each, and what causes a stack overflow?**
Recursion expresses self-similar problems (trees, backtracking, divide and conquer) clearly; iteration avoids call overhead and stack limits. Every call consumes a stack frame, and the stack is small and fixed (often 1 to 8 MB), so recursion depth proportional to n (a long linked list, a path-shaped graph) can overflow. Convert such code to a loop or an explicit `std::stack`.
*Follow-up:* Does C++ guarantee tail-call optimization? No; compilers may apply it with optimizations on, but you cannot rely on it.

**Q2. How do you estimate the complexity of a backtracking solution?**
Count the nodes of the recursion tree (branching factor to the power of depth, reduced by pruning) and multiply by the work per node, including copying each solution. Subsets: 2^n solutions x O(n) copy. Permutations: n! x O(n).
*Follow-up:* Why is the bound loose for N-Queens? Pruning removes most branches, so the real count is far below n^n; O(n!) is the usual estimate.

**Q3. What is the difference between recursion, DFS and backtracking?**
Recursion is the mechanism (a function calling itself). DFS is a traversal order that goes deep first and is often written recursively. Backtracking is DFS over a space of partial solutions with shared mutable state that is undone on the way back, plus pruning of dead branches.
*Follow-up:* Is every backtracking problem exponential? In the worst case yes; pruning and memoization change the practical cost.

**Q4. How do you avoid duplicate results without a set?**
Sort the input and, at each recursion depth, skip a candidate equal to the previous one at that depth (`i > start && a[i] == a[i - 1]`). For permutations with duplicates, skip `a[i] == a[i - 1]` when `a[i - 1]` is not currently used.
*Follow-up:* Why is a set worse? It still generates every duplicate (wasted work) and costs memory plus O(n log n) comparisons per insert.

**Q5. When does memoization turn backtracking into dynamic programming?**
When different paths reach the same subproblem state and the answer depends only on that state, not on the path. Then you cache by state (for example, index and remaining target) and the exponential tree collapses to states x transitions. Problems that must list every solution usually cannot be memoized usefully.
*Follow-up:* Example? Counting the combinations that sum to a target (Combination Sum IV, LeetCode 377) is DP; listing them (Combination Sum) is backtracking.

**Q6. Why pass the path by reference and undo changes, instead of passing copies?**
A copy at every call costs O(n) per node, multiplying the total by n and adding memory pressure. One shared path with `push_back`/`pop_back` keeps each step O(1) and copies only finished solutions.
*Follow-up:* When is copying acceptable? When paths are tiny or the language makes immutable copies cheap, but say the cost out loud.

**Q7. How would you convert a recursive DFS into an iterative one?**
Push the start state on an explicit `std::stack`; loop: pop a state, process it, push its children (in reverse order if the visiting order must match the recursion). For postorder or backtracking that must undo work, push a marker or store an iterator/index per frame to resume where you left off.
*Follow-up:* Which order do children come out? Reversed from the push order; push the right child before the left for preorder.

**Q8. How does Pow(x, n) get O(log n), and what breaks if you call the half power twice?**
Each level halves the exponent, giving log n levels with O(1) work each. Calling `power(x, e / 2)` twice creates two branches per level, 2^(log n) = n leaves, which is back to O(n).
*Follow-up:* Where does fast exponentiation matter in practice? Modular exponentiation in cryptography and hashing, and matrix exponentiation for linear recurrences.
