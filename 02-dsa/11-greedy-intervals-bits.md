# Greedy, Intervals and Bit Manipulation

> **Tier:** T1 · **Related:** [Heaps](08-heaps.md), [Dynamic programming](10-dynamic-programming.md), [Sorting and binary search](06-sorting-binary-search.md), [STL for DSA](../03-languages/c-cpp/04-stl-for-dsa.md) · **Language:** C++17

## Quick revision
- **Greedy:** take the locally best choice and never undo it. Correct only when you can argue that some optimal solution makes the same choice.
- Proof tools: the **exchange argument** (swap the greedy choice into an optimal solution without making it worse) and **greedy stays ahead** (after every step, greedy is at least as good).
- Greedy fails on Coin Change with {1, 3, 4} and on 0/1 knapsack by value-to-weight ratio (it only works for the fractional version). Use DP there.
- Interval patterns: **sort by start** to merge or detect conflicts; **sort by end** to keep the most non-overlapping intervals; a **min-heap of end times** counts rooms; a **sweep line** counts concurrency.
- Half-open `[s, e)` intervals overlap when `a.s < b.e && b.s < a.e`. Decide whether touching endpoints count as overlap before coding.
- Bit tricks: `x & 1` (odd), `(x >> k) & 1` (bit k), `x | (1 << k)` (set), `x & ~(1 << k)` (clear), `x ^ (1 << k)` (toggle).
- `x & (x - 1)` clears the lowest set bit; `x > 0 && (x & (x - 1)) == 0` tests for a power of two; `x & -x` isolates the lowest set bit.
- XOR: `a ^ a = 0`, `a ^ 0 = a`, order does not matter, so pairs cancel out.
- Popcount: Kernighan's loop, `__builtin_popcount` (GCC/Clang), `bitset<32>(x).count()`, or `std::popcount` in C++20.
- Write `1LL << k` or `1u << k` for high bits; shifting by the type's width or more is undefined; left-shifting a negative value is undefined before C++20.
- `==` binds tighter than `&`: write `(x & 1) == 0`, not `x & 1 == 0`.
- Signals: "minimum number of jumps/arrows/rooms", "can you reach", intervals, scheduling; "appears twice except one", "without using + or -", subsets of n <= 20, flags.

## 1. Concept from scratch

### 1.1 Greedy algorithms and why they work
A greedy algorithm builds the answer one irreversible decision at a time, always choosing what looks best right now. It is usually fast (often one sort plus one pass) and short, but it is only correct for problems with the **greedy-choice property**: some optimal solution begins with the greedy choice, and what remains is a smaller instance of the same problem.

**Exchange argument, worked on interval scheduling.** Goal: keep as many non-overlapping intervals as possible. Greedy: always keep the interval that ends first. Take any optimal solution; its first interval ends no earlier than the greedy one. Replace it with the greedy interval: nothing else conflicts (the greedy one ends even sooner), and the count is unchanged. Repeat for the rest. So greedy is optimal.

**When greedy fails:** coins {1, 3, 4} with amount 6 (greedy 4 + 1 + 1 uses three coins, 3 + 3 uses two); 0/1 knapsack by best value-per-weight. If you cannot find an argument, look for a counterexample with three or four small items, and fall back to DP if you find one.

### 1.2 Interval patterns
| Goal | Sort by | Then |
|---|---|---|
| Merge overlapping intervals | start | extend the last merged interval or start a new one |
| Insert one interval into a sorted list | (already sorted) | copy the ones before, absorb the overlaps, copy the rest |
| Keep the most non-overlapping / remove the fewest | end | keep an interval if it starts after the last kept one ends |
| Can one person attend everything? | start | any neighbor overlap means no |
| Minimum rooms / maximum concurrency | start (plus a min-heap of ends) or a sweep over events | see [Heaps](08-heaps.md) |

Decide early whether intervals are closed `[s, e]` or half-open `[s, e)`: it decides whether `[1, 2]` and `[2, 3]` overlap, and the problems in this file differ on that point.

### 1.3 Bit manipulation basics
Integers are stored in binary using **two's complement**, so `-x == ~x + 1`.

| Operator | Meaning | Example with 8-bit values |
|---|---|---|
| `a & b` | 1 where both bits are 1 | `1100 & 1010 = 1000` |
| `a \| b` | 1 where either bit is 1 | `1100 \| 1010 = 1110` |
| `a ^ b` | 1 where the bits differ | `1100 ^ 1010 = 0110` |
| `~a` | flip every bit | `~0000 1100 = 1111 0011` |
| `a << k` | shift left, multiply by 2^k | `0011 << 2 = 1100` |
| `a >> k` | shift right, divide by 2^k (for non-negative values) | `1100 >> 2 = 0011` |

| Trick | Why it works |
|---|---|
| `x & (x - 1)` clears the lowest set bit | subtracting 1 flips the lowest 1 and every 0 below it |
| `x > 0 && (x & (x - 1)) == 0` means power of two | powers of two have exactly one set bit |
| `x & -x` keeps only the lowest set bit | two's complement flips every bit above it |
| XOR of all values cancels pairs | `a ^ a = 0` and XOR is commutative and associative |
| `mask` from 0 to `(1 << n) - 1` enumerates subsets | bit i of the mask says whether item i is included |

Safety rules in C++17: shifting by a count greater than or equal to the bit width is undefined; left-shifting a negative signed value is undefined (C++20 defines it); right-shifting a negative value is implementation-defined (arithmetic in practice). Prefer `unsigned` types for bit work, and `1LL << k` when k can reach 31 or more.

## 2. Core templates
```cpp
#include <algorithm>
#include <utility>
#include <vector>
using namespace std;

// Sweep line: the maximum number of half-open intervals [s, e) active at the same time.
int maxConcurrent(const vector<pair<int, int>>& intervals) {
    vector<pair<int, int>> events;                    // (time, +1 at a start, -1 at an end)
    for (auto [s, e] : intervals) {
        events.push_back({s, +1});
        events.push_back({e, -1});
    }
    sort(events.begin(), events.end());               // at equal times, -1 sorts first: ends free a slot first
    int active = 0, best = 0;
    for (auto [t, delta] : events) {
        active += delta;
        best = max(best, active);
    }
    return best;
}

// Every subset of a small set (n up to about 20) as a bitmask.
vector<vector<int>> allSubsets(const vector<int>& items) {
    int n = items.size();
    vector<vector<int>> result;
    for (int mask = 0; mask < (1 << n); mask++) {
        vector<int> subset;
        for (int i = 0; i < n; i++)
            if ((mask >> i) & 1) subset.push_back(items[i]);   // bit i set: item i is in
        result.push_back(subset);
    }
    return result;
}

bool isPowerOfTwo(long long x) { return x > 0 && (x & (x - 1)) == 0; }

int popcount(unsigned int x) {                        // Kernighan: one iteration per set bit
    int count = 0;
    while (x) {
        x &= x - 1;
        count++;
    }
    return count;
}
```

## 3. Problems

### 3.1 Jump Game (LeetCode 55, Medium)
**Problem (paraphrased):** each array value is the maximum jump length from that index. Starting at index 0, decide whether you can reach the last index.
**Approach:** brute force recursion tries every jump length, exponential; a DP over "can I reach index i" is O(n^2). Greedy: track the farthest index reachable so far. If the loop ever reaches an index beyond that, you are stuck; every index up to `reach` is reachable, so extending `reach` from each of them is enough.
**Complexity:** O(n) time, O(1) space.
```cpp
class Solution {
public:
    bool canJump(vector<int>& nums) {
        int reach = 0;                                   // farthest index reachable so far
        for (int i = 0; i < (int)nums.size(); i++) {
            if (i > reach) return false;                 // index i cannot be reached
            reach = max(reach, i + nums[i]);
        }
        return true;
    }
};
```
**Edge cases:** a single element (true); a zero at index 0 with more elements (false); zeros that can be jumped over.

### 3.2 Jump Game II (LeetCode 45, Medium)
**Problem (paraphrased):** same jumps, and the last index is guaranteed reachable. Return the minimum number of jumps.
**Approach:** DP over "fewest jumps to reach i" is O(n^2). The greedy view is BFS by levels: the indices reachable with j jumps form a contiguous range. While scanning the current range, track the farthest index the next jump can reach; when you hit the end of the current range, take a jump and move the boundary.
**Complexity:** O(n) time, O(1) space.
```cpp
class Solution {
public:
    int jump(vector<int>& nums) {
        int jumps = 0, curEnd = 0, farthest = 0;
        for (int i = 0; i + 1 < (int)nums.size(); i++) {   // no jump is needed from the last index
            farthest = max(farthest, i + nums[i]);
            if (i == curEnd) {                              // end of the range reachable with `jumps` jumps
                jumps++;
                curEnd = farthest;
            }
        }
        return jumps;
    }
};
```
**Edge cases:** a single element (0 jumps); one jump covering everything.

### 3.3 Gas Station (LeetCode 134, Medium)
**Problem (paraphrased):** stations sit on a circular route; station i provides `gas[i]` and driving to the next station costs `cost[i]`. Return the unique start index from which you can drive the full circle with an initially empty tank, or -1.
**Approach:** brute force simulates from every start, O(n^2). Two facts give O(n): if total gas is less than total cost, no start works; and if the tank goes negative on the way from s to i, no station between s and i can be the start either (each would arrive at i with even less), so restart at i + 1.
**Complexity:** O(n) time, O(1) space.
```cpp
class Solution {
public:
    int canCompleteCircuit(vector<int>& gas, vector<int>& cost) {
        int total = 0, tank = 0, start = 0;
        for (int i = 0; i < (int)gas.size(); i++) {
            int diff = gas[i] - cost[i];
            total += diff;
            tank += diff;
            if (tank < 0) {            // cannot reach i + 1 from start, or from anywhere in between
                start = i + 1;
                tank = 0;
            }
        }
        return total >= 0 ? start : -1;
    }
};
```
**Edge cases:** a single station; total gas equal to total cost (a valid start still exists).

### 3.4 Merge Intervals (LeetCode 56, Medium)
**Problem (paraphrased):** merge all overlapping intervals (touching ones like `[1, 4]` and `[4, 5]` count as overlapping) and return the result.
**Approach:** brute force repeatedly merges any overlapping pair until none remain, O(n^2) or worse. Sort by start; then each interval either overlaps the last merged one (extend its end) or starts a new one.
**Complexity:** O(n log n) time, O(n) space for the output.
```cpp
class Solution {
public:
    vector<vector<int>> merge(vector<vector<int>>& intervals) {
        sort(intervals.begin(), intervals.end());                 // by start, then end
        vector<vector<int>> merged;
        for (auto& cur : intervals) {
            if (merged.empty() || merged.back()[1] < cur[0]) merged.push_back(cur);   // gap: new interval
            else merged.back()[1] = max(merged.back()[1], cur[1]);                  // overlap: extend
        }
        return merged;
    }
};
```
**Edge cases:** one interval nested inside another (`max` keeps the larger end); touching intervals; a single interval.

### 3.5 Insert Interval (LeetCode 57, Medium)
**Problem (paraphrased):** a list of non-overlapping intervals is sorted by start. Insert a new interval, merging where needed, and return the list.
**Approach:** appending and re-running Merge Intervals is O(n log n). One linear pass has three phases: copy intervals that end before the new one starts, absorb every interval that overlaps it, then copy the rest.
**Complexity:** O(n) time, O(n) space for the output.
```cpp
class Solution {
public:
    vector<vector<int>> insert(vector<vector<int>>& intervals, vector<int>& newInterval) {
        vector<vector<int>> result;
        int i = 0, n = intervals.size();
        while (i < n && intervals[i][1] < newInterval[0])         // entirely before
            result.push_back(intervals[i++]);
        while (i < n && intervals[i][0] <= newInterval[1]) {      // overlapping: absorb
            newInterval[0] = min(newInterval[0], intervals[i][0]);
            newInterval[1] = max(newInterval[1], intervals[i][1]);
            i++;
        }
        result.push_back(newInterval);
        while (i < n) result.push_back(intervals[i++]);           // entirely after
        return result;
    }
};
```
**Edge cases:** an empty list; the new interval before everything, after everything, or covering everything.

### 3.6 Non-overlapping Intervals (LeetCode 435, Medium)
**Problem (paraphrased):** return the minimum number of intervals to remove so the rest do not overlap. Here intervals that only touch (like `[1, 2]` and `[2, 3]`) do not overlap.
**Approach:** brute force tries all subsets, O(2^n). Removing the fewest is keeping the most, which is interval scheduling: sort by end and keep each interval that starts at or after the last kept end. The exchange argument in section 1.1 proves it.
**Complexity:** O(n log n) time, O(1) extra space.
```cpp
class Solution {
public:
    int eraseOverlapIntervals(vector<vector<int>>& intervals) {
        sort(intervals.begin(), intervals.end(),
             [](const vector<int>& a, const vector<int>& b) { return a[1] < b[1]; });   // by end time
        int kept = 0;
        long long lastEnd = LLONG_MIN;
        for (auto& iv : intervals) {
            if (iv[0] >= lastEnd) {        // starts after the last kept one ends: keep it
                kept++;
                lastEnd = iv[1];
            }
        }
        return (int)intervals.size() - kept;
    }
};
```
**Edge cases:** all intervals identical (keep one); already disjoint (0). Close cousin: Minimum Number of Arrows to Burst Balloons (LeetCode 452).

### 3.7 Can attend all meetings (classic interview problem, Easy)
**Problem (paraphrased):** given meeting times as `[start, end)` intervals, decide whether one person can attend all of them (a meeting may start exactly when another ends).
**Approach:** comparing every pair is O(n^2). After sorting by start, only neighbors can conflict: if any meeting starts before the previous one ends, the answer is no.
**Complexity:** O(n log n) time, O(1) extra space.
```cpp
class Solution {
public:
    bool canAttendMeetings(vector<vector<int>>& intervals) {
        sort(intervals.begin(), intervals.end());
        for (int i = 1; i < (int)intervals.size(); i++)
            if (intervals[i][0] < intervals[i - 1][1]) return false;   // starts before the previous one ends
        return true;
    }
};
```
**Edge cases:** no meetings or one meeting (true); back-to-back meetings (true). The follow-up "how many rooms?" is the heap problem in [Heaps](08-heaps.md).

### 3.8 Single Number (LeetCode 136, Easy)
**Problem (paraphrased):** every value appears exactly twice except one. Find it in O(n) time and O(1) extra space.
**Approach:** a hash map of counts is O(n) space; sorting is O(n log n). XOR all values: equal pairs cancel to 0, leaving the single value.
**Complexity:** O(n) time, O(1) space.
```cpp
class Solution {
public:
    int singleNumber(vector<int>& nums) {
        int x = 0;
        for (int v : nums) x ^= v;     // a ^ a = 0 and a ^ 0 = a
        return x;
    }
};
```
**Edge cases:** negative numbers (XOR works on any bit pattern); a single element. Variant where others appear three times: count each bit position modulo 3.

### 3.9 Number of 1 Bits (LeetCode 191, Easy)
**Problem (paraphrased):** return how many bits are set to 1 in the binary representation of a number.
**Approach:** checking all 32 bits with `(n >> i) & 1` is O(32). Kernighan's trick `n &= n - 1` removes one set bit per iteration, so the loop runs once per set bit.
**Complexity:** O(number of set bits), at most 32 iterations; O(1) space.
```cpp
class Solution {
public:
    int hammingWeight(uint32_t n) {    // unsigned: the bit tricks are well defined
        int count = 0;
        while (n) {
            n &= n - 1;                 // clears the lowest set bit
            count++;
        }
        return count;
    }
};
```
**Edge cases:** 0 (no set bits); all 32 bits set. The signature has varied between an unsigned and a positive `int` parameter in different versions of the problem; the logic is the same. One-liners: `__builtin_popcount(n)` or `bitset<32>(n).count()`.

### 3.10 Counting Bits (LeetCode 338, Easy)
**Problem (paraphrased):** for every i from 0 to n, return the number of set bits in i, ideally in O(n) total.
**Approach:** calling popcount for each number is O(n log n). DP on bits: `i >> 1` drops the last bit, which is `i & 1`, so `bits[i] = bits[i >> 1] + (i & 1)`. (Another valid recurrence: `bits[i] = bits[i & (i - 1)] + 1`.)
**Complexity:** O(n) time, O(n) space for the output.
```cpp
class Solution {
public:
    vector<int> countBits(int n) {
        vector<int> bits(n + 1, 0);
        for (int i = 1; i <= n; i++)
            bits[i] = bits[i >> 1] + (i & 1);   // bits of i/2, plus i's own last bit
        return bits;
    }
};
```
**Edge cases:** n = 0 returns `[0]`.

### 3.11 Missing Number (LeetCode 268, Easy)
**Problem (paraphrased):** an array holds n distinct numbers from the range 0..n, so exactly one is missing. Find it.
**Approach:** a hash set is O(n) space; sorting is O(n log n). The sum formula `n(n + 1)/2 - sum` works (use `long long` for large n). XOR avoids overflow altogether: XOR every index and every value together with n; everything present cancels.
**Complexity:** O(n) time, O(1) space.
```cpp
class Solution {
public:
    int missingNumber(vector<int>& nums) {
        int n = nums.size(), x = n;                       // n is the one index without a slot
        for (int i = 0; i < n; i++) x ^= i ^ nums[i];     // each present value cancels its index
        return x;
    }
};
```
**Edge cases:** the missing number is 0 or n.

### 3.12 Sum of Two Integers (LeetCode 371, Medium)
**Problem (paraphrased):** add two integers without using `+` or `-`.
**Approach:** this is how an adder circuit works. XOR adds bits without carries; AND finds positions where both bits are 1, and shifting that left by one gives the carries. Repeat until no carry remains. In C++, left-shifting a negative `int` is undefined before C++20, so do the work in `unsigned int`, where shifts are well defined and wrap modulo 2^32.
**Complexity:** O(32) iterations at most, so O(1) time and space.
```cpp
class Solution {
public:
    int getSum(int a, int b) {
        unsigned int x = a, y = b;              // unsigned: shifts are defined and wrap like hardware
        while (y != 0) {
            unsigned int carry = (x & y) << 1;  // bits that carry into the next position
            x ^= y;                             // sum without the carries
            y = carry;
        }
        return (int)x;                          // two's complement wrap back (defined in C++20, universal in practice)
    }
};
```
**Edge cases:** negative operands; results that cancel to 0 (`-1 + 1`).

## Common pitfalls
- Using greedy without an argument or a quick counterexample check; when in doubt, try small cases against a brute force.
- Sorting intervals by the wrong key (start versus end) for the goal at hand.
- Mixing up closed and half-open intervals, so touching intervals are merged or counted wrongly.
- Forgetting that `sort` reorders the caller's vector in place; copy it first if the original order still matters.
- Operator precedence: `x & 1 == 0` means `x & (1 == 0)`. Parenthesize bit expressions.
- `1 << 31` or `1 << k` with k >= 31 on `int`; use `1u << k` or `1LL << k`.
- Bit tricks on negative signed numbers (shifts, conversions); convert to `unsigned` first.
- Summing large values in `int` for formulas like `n(n + 1)/2`; use `long long` or XOR.

## How to talk about this in an interview
- For greedy, say the choice and the reason: "I always keep the interval that ends first, because it leaves the most room for the rest; if an optimal answer picked a different first interval, swapping in mine can't hurt." That one exchange-argument sentence separates a guess from a solution.
- For bits, write a small binary example (like 12 = 1100) and walk the trick on it; explaining `x & (x - 1)` on paper is a standard lab-style explanation and easy to follow.
- Real-world links: booking and scheduling features (rooms in a co-living house, events, rent periods) are interval problems, and PostgreSQL can enforce "no overlapping bookings" with range types and an exclusion constraint. Bit flags appear as permission masks, Unix file modes (`chmod 755`) and compact feature flags.
- > **Fill in:** any feature you built with dates or time ranges (events, tasks, rent periods, subscriptions) and how overlaps or ordering were handled.

## Interview questions

**Q1. How do you convince an interviewer that a greedy algorithm is correct?**
State the greedy choice, then give an exchange argument: take any optimal solution, show that replacing its first decision with the greedy decision keeps it valid and no worse, and repeat. Alternatively show greedy stays ahead after every step. Then sanity-check small cases against a brute force.
*Follow-up:* What if you cannot prove it? Look for a counterexample; if you find one, switch to DP.

**Q2. Give an example where greedy fails but DP works.**
Coin Change with coins {1, 3, 4} and amount 6: greedy takes 4 + 1 + 1 (three coins), but 3 + 3 uses two. The 0/1 knapsack is another: taking items by best value per weight can waste capacity. DP explores all combinations of subproblems instead of committing early.
*Follow-up:* Why does greedy work for everyday coins such as 1, 2, 5, 10? That system is canonical: greedy happens to be optimal for those denominations, a special property rather than a general rule.

**Q3. Why sort by end time to keep the most non-overlapping intervals, but by start time to merge?**
Keeping the interval that ends first leaves the most room for later intervals, which the exchange argument proves optimal. Merging needs overlapping intervals to be adjacent in the processing order so one pass can extend the current block, which sorting by start guarantees.
*Follow-up:* What does sorting by start give for scheduling? It can keep a long early interval that blocks many short ones, so it is not optimal.

**Q4. How do you check whether a number is a power of two, and why does `x & (x - 1)` work?**
`x > 0 && (x & (x - 1)) == 0`. Subtracting 1 flips the lowest set bit to 0 and all the zeros below it to 1, so `x & (x - 1)` clears exactly the lowest set bit. A power of two has a single set bit, so the result is 0.
*Follow-up:* How do you count set bits with the same trick? Repeat `x &= x - 1` until x is 0, counting iterations.

**Q5. What are the pitfalls of shifts and signed integers in C++?**
Shifting by a count at least the bit width is undefined; left-shifting a negative value is undefined before C++20; right-shifting a negative value is implementation-defined before C++20 (usually arithmetic); and `1 << 31` on a 32-bit `int` reaches the sign bit. Use unsigned types for bit manipulation and `1LL` or `1u` literals for high bits.
*Follow-up:* What changed in C++20? Signed integers are defined as two's complement, and left shifts of negative values became well defined.

**Q6. Why not use the XOR swap trick in real code?**
`a ^= b; b ^= a; a ^= b;` swaps without a temporary, but it zeroes the value when both names refer to the same object, it is harder to read, and compilers already generate optimal code for `std::swap`. It is a curiosity, not an optimization.
*Follow-up:* Where is XOR genuinely useful? Finding the unpaired element, parity checks, checksums, and simple toggling of flags.

**Q7. Where do bit flags show up in real systems?**
Permission masks (read/write/execute in Unix file modes), feature flags packed into one integer column, network masks and IP addressing, compact sets in bitmask DP, and Bloom filters, which use bit arrays for fast probabilistic membership tests.
*Follow-up:* Why not always pack booleans into bits in a database? Readability and querying suffer; a few separate boolean columns or an array/JSON column are often clearer.

**Q8. When do interval problems need a heap rather than a single sort?**
When you need to know how many intervals are active at once or which one frees up next, such as minimum meeting rooms or assigning tasks to servers. Sort by start and keep a min-heap of end times; the heap's size is the number of rooms in use. A sweep over start and end events gives the same count.
*Follow-up:* Complexity? O(n log n) for the sort plus O(log n) per heap operation.
