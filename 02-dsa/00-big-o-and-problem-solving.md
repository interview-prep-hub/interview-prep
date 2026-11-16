# Big-O and Problem Solving

> **Tier:** T1 · **Related:** [Arrays, strings and hashing](01-arrays-strings-hashing.md), [Practice problem list](12-practice-problem-list.md), [STL for DSA](../03-languages/c-cpp/04-stl-for-dsa.md), [Interview formats](../00-start-here/03-interview-formats.md) · **Language:** C++17

## Quick revision
- Big-O is an upper bound on growth, Big-Omega a lower bound, Big-Theta a tight bound. In interviews "what's the Big-O?" means the tightest bound for the worst case.
- Drop constants and lower-order terms: `3n^2 + 10n + 7` is O(n^2).
- Sequential blocks add (O(n + m)); nested loops multiply (O(n * m)); halving or doubling a counter gives O(log n).
- `for (j = i; j <= n; j += i)` inside `for i` is the harmonic series: O(n log n).
- Master theorem lite for `T(n) = a T(n/b) + O(n^d)`: if `a < b^d` then O(n^d); if `a = b^d` then O(n^d log n); if `a > b^d` then O(n^(log_b a)).
- Binary search O(log n), merge sort O(n log n), tree traversal O(n), naive Fibonacci O(2^n), all permutations O(n * n!).
- Amortized O(1): `vector::push_back` grows capacity geometrically, so n pushes cost O(n) in total even though one push can cost O(n).
- Space includes the recursion stack: recursive DFS on a skewed tree uses O(n) stack.
- Rule of thumb: about 1e8 simple operations per second. n up to 1e5 needs O(n log n) or better; n up to 20 allows O(2^n).
- Hash map operations are O(1) on average and O(n) in the worst case; `std::map` is O(log n) guaranteed.
- Live-coding loop: clarify, examples, brute force, optimize, code, test, analyze. Narrate every step.
- Always state time and space, and name the variable each depends on (n, m, k, V, E, L).

## 1. Concept from scratch

### 1.1 Why count steps instead of seconds
Wall-clock time depends on the machine, the compiler flags and the input. Asymptotic analysis asks a machine-independent question: as the input size n grows, how fast does the number of basic steps grow? You count the dominant operations (comparisons, array reads, hash lookups), write them as a function of n, and keep only the fastest-growing term. Constants still matter in production, but they do not change which algorithm wins when n gets big.

### 1.2 Big-O, Big-Omega, Big-Theta
| Notation | Meaning | Read it as |
|---|---|---|
| f(n) = O(g(n)) | for large n, f(n) <= c * g(n) for some constant c | "grows at most like g" |
| f(n) = Omega(g(n)) | for large n, f(n) >= c * g(n) for some constant c | "grows at least like g" |
| f(n) = Theta(g(n)) | both of the above | "grows exactly like g, up to constants" |

Two different axes get mixed up:
- **Bound type** (O, Omega, Theta) describes how a function grows.
- **Case** (best, average, worst) chooses which function you are describing. Quick sort's worst case is Theta(n^2) and its average case is Theta(n log n). Linear search's best case is Theta(1) and its worst case is Theta(n).

Saying "linear search is O(n^3)" is technically true and useless. Give the tight bound and say which case you mean.

### 1.3 Common complexity classes
| Class | Name | Typical example |
|---|---|---|
| O(1) | constant | array index, `push_back` (amortized), hash lookup (average) |
| O(log n) | logarithmic | binary search, `std::map::find`, heap push/pop |
| O(sqrt n) | square root | trial-division primality test |
| O(n) | linear | one pass, counting, BFS/DFS (O(V + E)) |
| O(n log n) | linearithmic | merge sort, `std::sort`, n heap operations |
| O(n^2) | quadratic | all pairs, bubble/insertion sort, 2D DP on n x n |
| O(n^3) | cubic | Floyd-Warshall, naive matrix multiplication |
| O(2^n) | exponential | all subsets, naive Fibonacci |
| O(n!) | factorial | all permutations, brute-force travelling salesman |

### 1.4 Analyzing loops
The cost of a loop is (number of iterations) x (cost of one iteration), and the cost of one iteration includes every call inside it.

```cpp
// 1. Sequential blocks add: O(n + m)
for (int i = 0; i < n; i++) work();
for (int j = 0; j < m; j++) work();

// 2. Nested loops with independent bounds multiply: O(n * m)
for (int i = 0; i < n; i++)
    for (int j = 0; j < m; j++) work();

// 3. Dependent bound: n + (n - 1) + ... + 1 = n(n + 1) / 2, still O(n^2)
for (int i = 0; i < n; i++)
    for (int j = i; j < n; j++) work();

// 4. Halving: about log2(n) iterations, O(log n)
for (int x = n; x > 0; x /= 2) work();

// 5. An early break does not change the worst case: O(n)
for (int i = 0; i < n; i++) if (a[i] == target) break;
```

Hidden linear costs are the classic trap: `s.substr(i)`, `v.erase(v.begin())`, `find(v.begin(), v.end(), x)`, `s = s + t`, and passing a `vector` by value are all O(n) per call. A loop around any of them adds a factor of n.

### 1.5 Recursion: recurrences and the master theorem lite
Write the cost of a call as "work done here + cost of the recursive calls", then picture the recursion tree: total work = sum over levels of (calls on that level x work per call).

| Recurrence | Example | Result |
|---|---|---|
| T(n) = T(n/2) + O(1) | binary search | O(log n) |
| T(n) = T(n - 1) + O(1) | recursive factorial, walking a list | O(n) |
| T(n) = T(n - 1) + O(n) | recursive selection sort | O(n^2) |
| T(n) = 2T(n/2) + O(1) | traversal of a balanced tree | O(n) |
| T(n) = 2T(n/2) + O(n) | merge sort | O(n log n) |
| T(n) = 2T(n - 1) + O(1) | Tower of Hanoi, naive Fibonacci (roughly) | O(2^n) |

**Master theorem lite.** For `T(n) = a T(n/b) + O(n^d)` (a subproblems of size n/b, O(n^d) work to split and combine):
- `a < b^d`: the root's work dominates, so O(n^d).
- `a = b^d`: every level does the same work and there are log n levels, so O(n^d log n).
- `a > b^d`: the leaves dominate, so O(n^(log_b a)).

Merge sort: a = 2, b = 2, d = 1, and 2 = 2^1, so O(n log n). Binary search: a = 1, b = 2, d = 0, and 1 = 2^0, so O(log n).

**Branching rule.** A recursion with b branches and depth d makes about O(b^d) calls. Naive Fibonacci has two branches and depth n, so O(2^n) (more precisely about 1.618^n). Memoization changes the count to (number of distinct states) x (work per state): O(n) for Fibonacci.

### 1.6 Amortized analysis: why push_back is O(1)
When a `vector` is full, `push_back` allocates a bigger block, moves every element and frees the old one: that single push is O(n). The capacity grows geometrically (commonly 2x in GCC's libstdc++ and 1.5x in MSVC; the standard only requires amortized constant time). With doubling, the moves over n pushes total 1 + 2 + 4 + ... + n/2 < n, so n pushes do fewer than 2n element writes: O(1) per push on average over the sequence.

- **Amortized is not average case.** Amortized is a guarantee over any sequence of operations, with no probability involved. Average case is an expectation over random inputs.
- `reserve(n)` removes reallocations when you know the final size.
- Reallocation invalidates all iterators, pointers and references into the vector, a classic source of crashes.
- The same accounting explains the monotonic stack (each element is pushed and popped at most once), the sliding window (the left pointer moves at most n times in total) and `unordered_map` rehashing.

### 1.7 Space complexity, including the stack
- **Auxiliary space** is extra memory beyond the input; that is what interviewers usually mean. Say whether you count the output array (it is usually excluded).
- **The recursion stack counts.** Space is the maximum depth times the frame size, not the total number of calls. Recursive DFS uses O(log n) stack on a balanced tree and O(n) on a skewed one.
- Default stack sizes are typically about 1 MB on Windows and 8 MB on many Linux systems, so recursion that goes 1e5 to 1e6 levels deep can overflow. Switch to an explicit stack when depth can be that large.
- Typical numbers: merge sort O(n) extra, quick sort O(log n) expected stack, BFS O(width of the graph) for the queue, hash set of n items O(n).

### 1.8 Input size to acceptable complexity
Assume roughly 1e8 simple operations per second in C++ and a 1 to 2 second limit.

| Max n | Target complexity | Usual technique |
|---|---|---|
| up to 10 or 11 | O(n!) | permutations, brute force |
| up to 20 to 25 | O(2^n), O(n * 2^n) | subsets, backtracking, bitmask DP |
| up to 100 to 500 | O(n^3) | triple loops, Floyd-Warshall, interval DP |
| up to 1e3 to 5e3 | O(n^2) | all pairs, 2D DP |
| up to 1e5 to 1e6 | O(n log n) | sorting, heaps, binary search |
| up to 1e7 to 1e8 | O(n) | one pass, two pointers, prefix sums, hashing |
| 1e9 and beyond | O(log n) or O(1) | binary search on the answer, math |

Read the constraints first: they tell you the intended complexity before you think about the algorithm.

## 2. Core template: the live-coding framework

### 2.1 The seven steps
| Step | What you do | Time in a 45-minute round |
|---|---|---|
| 1. Clarify | Restate the problem. Ask about input size, value ranges, negatives, duplicates, empty input, sortedness, output format, in place or not. | 2 to 4 min |
| 2. Examples | Write one normal example and one or two edge cases; confirm the expected output. | 2 to 3 min |
| 3. Brute force | State the simplest correct idea and its complexity, even if it is slow. | 1 to 2 min |
| 4. Optimize | Name the bottleneck, then apply a pattern (hashing, sorting, two pointers, heap, DP). Get a go-ahead. | 5 to 10 min |
| 5. Code | Clear names, small helpers, keep talking while you type. | 10 to 15 min |
| 6. Test | Dry-run your example line by line, then the edge cases. Fix bugs calmly. | 5 min |
| 7. Analyze | Final time and space, trade-offs, likely follow-ups. | 1 to 2 min |

### 2.2 Worked example: Two Sum, out loud
**Problem (paraphrased):** given an integer array and a target, return the indices of two different elements whose values add up to the target. Exactly one valid pair exists.

**1. Clarify.** Say: "Can there be negatives or duplicates? Is there always exactly one answer? Can I use the same element twice? Do you want indices or values, and in any order? How large can n be?" Assume: negatives and duplicates allowed, exactly one answer, no reuse, any order, n up to 1e4, and the interviewer hopes for better than O(n^2).

**2. Examples.** `[2, 7, 11, 15]`, target 9 gives `[0, 1]`. Edge case with duplicates: `[3, 3]`, target 6 gives `[0, 1]`.

**3. Brute force.** Say: "The simplest approach checks every pair i < j. That's O(n^2) time and O(1) space. It's correct, so it's my fallback. The inner loop is just searching for one value, so let me make that search faster."

**4. Optimize.** Say: "For each element I need `target - nums[i]`. A hash map gives O(1) average lookups, so if I store each value's index as I go, one pass finds the pair: O(n) time and O(n) space. Sorting plus two pointers would be O(n log n) and would lose the original indices, so the hash map fits better. Shall I code it?"

**5. Code.**
```cpp
#include <unordered_map>
#include <vector>
using namespace std;

class Solution {
public:
    vector<int> twoSum(vector<int>& nums, int target) {
        unordered_map<int, int> indexOf;               // value -> index where it was seen
        for (int i = 0; i < (int)nums.size(); i++) {
            int need = target - nums[i];
            auto it = indexOf.find(need);
            if (it != indexOf.end()) return {it->second, i};
            indexOf[nums[i]] = i;                      // insert after the lookup: i never pairs with itself
        }
        return {};                                     // unreachable when an answer is guaranteed
    }
};
```
While typing, say: "I look up before inserting, so an element can't match itself. That also handles `[3, 3]`: the first 3 is stored when the second one looks for its complement."

**6. Test.** Dry run out loud: "i = 0: need 7, map empty, store 2 at 0. i = 1: need 2, found at index 0, return [0, 1]." Then edge cases: duplicates `[3, 3]`; negatives `[-1, -2, -3, -4, -5]` with target -8 gives `[2, 4]`; the pair at the very end of the array.

**7. Analyze.** Say: "O(n) time on average, since each lookup and insert is O(1) average; it degrades to O(n^2) only if hashing degenerates. O(n) extra space for the map."

Likely follow-ups: "What if the array is sorted?" (two pointers with O(1) space, see [Two Sum II](02-two-pointers-sliding-window.md)). "What if you need every pair?" "What if the data does not fit in memory?" (sort externally, or partition the data by hash and solve each partition).

### 2.3 Pattern-recognition table
| Signal in the problem | Pattern | File |
|---|---|---|
| "Have I seen X?", counting, pairs that sum to a target | Hash map / set | [01](01-arrays-strings-hashing.md) |
| Subarray sum equals k, range sums, negatives allowed | Prefix sums + hash map | [01](01-arrays-strings-hashing.md) |
| Sorted array, pair or triplet with a target, palindrome check | Two pointers from both ends | [02](02-two-pointers-sliding-window.md) |
| Longest or shortest contiguous subarray/substring with a condition | Sliding window | [02](02-two-pointers-sliding-window.md) |
| Cycle, middle, or k-th node from the end of a list | Fast and slow pointers | [03](03-linked-lists.md) |
| Matching brackets, nested structure, undo | Stack | [04](04-stacks-queues.md) |
| Next greater/smaller element, spans, histogram areas | Monotonic stack | [04](04-stacks-queues.md) |
| Max or min of every window of size k | Monotonic deque | [04](04-stacks-queues.md) |
| All subsets, permutations or combinations, n up to about 20 | Backtracking | [05](05-recursion-backtracking.md) |
| Sorted input, or "smallest x such that feasible(x)" | Binary search on index or on answer | [06](06-sorting-binary-search.md) |
| Hierarchy, root-to-leaf paths, subtree properties | Tree DFS | [07](07-trees-bst.md) |
| Prefix matching, autocomplete, word dictionary | Trie | [07](07-trees-bst.md) |
| Top k, k-th largest, running median, merge k sorted streams | Heap | [08](08-heaps.md) |
| Fewest steps in an unweighted graph or grid | BFS | [09](09-graphs.md) |
| Prerequisites, build order, dependencies | Topological sort | [09](09-graphs.md) |
| Merging groups, dynamic connectivity, redundant edge | Union-find | [09](09-graphs.md) |
| Cheapest path with non-negative weights | Dijkstra | [09](09-graphs.md) |
| Count the ways, min/max cost, "can we reach", overlapping choices | Dynamic programming | [10](10-dynamic-programming.md) |
| Intervals: merge, overlap, fewest removals | Sort + greedy | [11](11-greedy-intervals-bits.md) |
| Everything appears twice except one, subsets as masks | Bit manipulation | [11](11-greedy-intervals-bits.md) |

### 2.4 How to practice
- **Pattern blocks, then mixing.** Solve three to five problems of one pattern in a row until its template feels automatic, then do mixed sets so you practice recognizing the pattern from the wording.
- **Timebox.** About 15 to 20 minutes for an Easy and 30 to 40 for a Medium. If you have no idea after 20 minutes, read a hint or the approach (not the code), close it, and write the solution yourself.
- **Spaced repetition.** Re-solve from a blank editor after 1, 3 and 7 days. A problem counts as done only when you solve it cold inside the timebox. Track it in the [practice list](12-practice-problem-list.md).
- **Mistake log.** One line per bug: off-by-one, overflow, empty input, wrong pattern, forgot to restore state. Re-read it the night before an interview.
- **Talk while you practice.** Solve out loud or record yourself; silence is the habit that hurts most in live rounds. Do a few mocks in a plain editor with no autocomplete.
- **Headers.** `#include <bits/stdc++.h>` works on GCC (most online judges) but not on MSVC or Clang with libc++. In interviews, include the specific headers (`<vector>`, `<unordered_map>`, `<algorithm>`, `<queue>`, `<climits>`).

## 3. Problems
Each exercise asks for the time and space complexity of a snippet (the `#include` lines and `using namespace std;` are omitted). Cover the answer, decide, then check.

### 3.1 Exercise: dependent nested loops
```cpp
int countPairs(const vector<int>& a, int k) {
    int n = a.size(), count = 0;
    for (int i = 0; i < n; i++)
        for (int j = i + 1; j < n; j++)
            if (a[i] + a[j] == k) count++;
    return count;
}
```
**Answer:** O(n^2) time, O(1) space. The inner loop runs n - 1, n - 2, ..., 0 times, n(n - 1)/2 in total. Doing half the pairs does not change the class.

### 3.2 Exercise: doubling inner loop
```cpp
long long countOps(int n) {
    long long ops = 0;
    for (int i = 0; i < n; i++)
        for (int j = 1; j < n; j *= 2)
            ops++;
    return ops;
}
```
**Answer:** O(n log n) time, O(1) space. The outer loop runs n times; the inner loop doubles j until it reaches n, about log2(n) iterations.

### 3.3 Exercise: harmonic loop
```cpp
vector<int> countDivisors(int n) {
    vector<int> cnt(n + 1, 0);
    for (int i = 1; i <= n; i++)
        for (int j = i; j <= n; j += i)   // visit every multiple of i
            cnt[j]++;
    return cnt;
}
```
**Answer:** O(n log n) time, O(n) space. The inner loop runs n/i times, and n/1 + n/2 + ... + n/n = n(1 + 1/2 + ... + 1/n), which is about n ln n.

### 3.4 Exercise: square-root loop
```cpp
bool isPrime(long long n) {
    if (n < 2) return false;
    for (long long d = 2; d * d <= n; d++)
        if (n % d == 0) return false;
    return true;
}
```
**Answer:** O(sqrt n) time, O(1) space. The loop stops once d passes sqrt(n). Using `long long` matters: with `int`, `d * d` overflows when n is close to 2^31.

### 3.5 Exercise: naive versus memoized recursion
```cpp
long long fibNaive(int n) {
    if (n < 2) return n;
    return fibNaive(n - 1) + fibNaive(n - 2);
}

long long fibMemo(int n, vector<long long>& memo) {   // memo has n + 1 slots, all -1
    if (n < 2) return n;
    if (memo[n] != -1) return memo[n];
    return memo[n] = fibMemo(n - 1, memo) + fibMemo(n - 2, memo);
}
```
**Answer:** naive is O(2^n) time (about 1.618^n) and O(n) space, because space is the deepest chain of calls, not the number of calls. Memoized is O(n) time and O(n) space: n distinct states, each solved once.

### 3.6 Exercise: divide and conquer
```cpp
int maxOf(const vector<int>& a, int lo, int hi) {          // a[lo..hi], non-empty
    if (lo == hi) return a[lo];
    int mid = lo + (hi - lo) / 2;
    return max(maxOf(a, lo, mid), maxOf(a, mid + 1, hi));
}

long long levelSums(const vector<int>& a, int lo, int hi) { // a[lo..hi], non-empty
    long long s = 0;
    for (int i = lo; i <= hi; i++) s += a[i];               // O(size of the range) in every call
    if (lo == hi) return s;
    int mid = lo + (hi - lo) / 2;
    return s + levelSums(a, lo, mid) + levelSums(a, mid + 1, hi);
}
```
**Answer:** `maxOf` is T(n) = 2T(n/2) + O(1): a = 2 > b^d = 1, so O(n^(log2 2)) = O(n) time. `levelSums` is T(n) = 2T(n/2) + O(n): a = 2 = b^d, so O(n log n) time. Both use O(log n) stack.

### 3.7 Exercise: a hidden linear cost
```cpp
void drain(vector<int>& v) {
    while (!v.empty()) {
        process(v.front());
        v.erase(v.begin());   // shifts every remaining element one slot left
    }
}
```
**Answer:** O(n^2) time, O(1) extra space. Each front erase moves the remaining n - 1, n - 2, ... elements. Fix it by walking an index, using `std::deque::pop_front` (O(1)), or reversing once and calling `pop_back`.

### 3.8 Exercise: building a string
```cpp
string slowJoin(const vector<string>& parts) {
    string s;
    for (const string& p : parts) s = s + p;   // allocates and copies a brand-new string each time
    return s;
}

string fastJoin(const vector<string>& parts) {
    string s;
    for (const string& p : parts) s += p;      // appends in place, amortized O(p.size())
    return s;
}
```
**Answer:** with L the total output length, `slowJoin` is O(n * L) in the worst case (O(L^2) when every part is one character) because `s + p` copies all of `s` on every iteration. `fastJoin` is O(L) amortized. Ruby has the same trap: `str += x` builds a new string while `str << x` appends in place.

## Common pitfalls
- Missing hidden costs inside loops: `substr`, `erase(begin())`, linear `find`, `s = s + t`, or passing containers by value into a recursive helper.
- Counting the total number of recursive calls as space. Space is the maximum depth (plus memo tables).
- Saying a hash lookup is O(1) without "on average".
- Collapsing two variables into one: O(n * m) is not O(n^2) when m is tiny; graph algorithms are O(V + E), not O(V).
- Forgetting overflow when estimating: a sum of 1e5 values up to 1e9 needs `long long`.
- Coding before the interviewer agrees with the approach, or going silent for minutes.
- Jumping to a clever idea without first stating a correct brute force.
- Declaring "done" without dry-running an example.

## How to talk about this in an interview
- You assisted in the Data Structures labs at PUCIT, so lean on that habit of explaining: count loop iterations out loud, draw the recursion tree, and name the bottleneck before optimizing. Interviewers grade the reasoning as much as the final code.
- > **Fill in:** one concept students in your Data Structures lab found hard (recursion, pointers, complexity) and the explanation or example that worked for them.
- Connect complexity to the web work you do: an N+1 query issues one database round trip per record, so preloading turns n queries into one or two; an index turns an O(n) table scan into an O(log n) B-tree lookup (see [indexing](../12-databases/05-indexing-and-query-optimization.md)); matching n imported prices against m catalog items with nested loops is O(n * m), while indexing one side in a hash map first makes it O(n + m).
- > **Fill in:** a place in a real codebase (for example a sync job or a report) where a nested loop or repeated query was slow, and what you changed.
- Useful phrases: "My first idea is O(n^2); the bottleneck is the inner search." "I'm trading O(n) extra memory for a linear-time pass." "Let me dry-run this with the example before we call it done."

## Interview questions

**Q1. What's the difference between Big-O, Big-Omega and Big-Theta, and how do they relate to best and worst case?**
O is an upper bound, Omega a lower bound and Theta a tight bound on how a function grows. Best, average and worst case pick which function you analyze; each case can then be described with any of the three. Quick sort's worst case is Theta(n^2), its average case Theta(n log n).
*Follow-up:* Is it wrong to say binary search is O(n)? Not wrong, just not tight; say O(log n).

**Q2. What does amortized O(1) mean for `vector::push_back`?**
Most pushes write one element, but when capacity runs out the vector reallocates and moves everything (O(n)). Because capacity grows geometrically, the total work for n pushes is O(n), so the cost per push averaged over any sequence is O(1). No probability is involved, unlike average case.
*Follow-up:* How do you avoid reallocations? Call `reserve(n)` when the final size is known.

**Q3. What are the time complexities of common STL operations?**
| Operation | Cost |
|---|---|
| `vector` index, `push_back`, `pop_back` | O(1), amortized O(1), O(1) |
| `vector` insert/erase at front or middle | O(n) |
| `deque` push/pop at either end, index | O(1) |
| `list` insert/erase at an iterator, `splice` | O(1) (finding the spot is O(n)) |
| `map`/`set` find, insert, erase, `lower_bound` | O(log n) |
| `unordered_map`/`unordered_set` find, insert, erase | O(1) average, O(n) worst |
| `priority_queue` push, pop / top | O(log n) / O(1) |
| `sort`, `stable_sort` | O(n log n) |
| `lower_bound`, `binary_search` on a sorted vector | O(log n) |
| `nth_element` | O(n) average |
| `string::substr(pos, k)` | O(k) |

*Follow-up:* Why might `std::find` on a `set` be slower than `set::find`? `std::find` is a linear scan; the member function uses the tree in O(log n).

**Q4. Why do we drop constants, and when do constants matter?**
Asymptotics compare growth rates, which decide the winner for large n. Constants matter for small n and in hot paths: insertion sort beats merge sort on tiny arrays (which is why real sort implementations switch to it for small ranges), and a cache-friendly `vector` often beats a `list` even where the list has better Big-O.
*Follow-up:* Example where O(n log n) beats O(n) in practice? A contiguous sort can beat a hash-based linear pass with poor cache behavior on moderate n.

**Q5. How do you find the space complexity of a recursive function?**
Take the maximum recursion depth times the memory per frame, then add any heap allocations such as memo tables or copies. Naive Fibonacci makes O(2^n) calls but only O(n) are on the stack at once.
*Follow-up:* What happens if the depth is 1e6? Likely a stack overflow; convert to iteration with an explicit stack.

**Q6. The constraints say n is up to 1e5. What do you aim for?**
O(n log n) or better. At about 1e8 simple operations per second, O(n^2) would be 1e10 operations (far too slow), while O(n log n) is about 1.7e6. So I look for sorting, a heap, binary search, hashing or a single pass.
*Follow-up:* And n up to 20? Exponential is fine: 2^20 is about 1e6, so subsets or bitmask DP.

**Q7. Is a hash map lookup really O(1)?**
On average, assuming a good hash function and a bounded load factor. Each lookup still hashes the key (O(L) for a string of length L), and if many keys land in one bucket, operations degrade to O(n). Adversarial inputs can force this, which is why some systems randomize their hash seeds.
*Follow-up:* When would you choose `std::map` instead? When you need ordered iteration, range queries, or guaranteed O(log n).

**Q8. What do you do when you get stuck in a live coding round?**
Say where you are stuck, go back to the brute force and look for the repeated work, try a smaller or concrete example by hand, and walk the pattern list (sort it? hash it? two pointers? heap? DP?). Accept hints gracefully and build on them rather than starting over.
*Follow-up:* What if time is almost up? Explain the remaining steps clearly and finish a working brute force rather than half of an optimal solution.
