# Two Pointers and Sliding Window

> **Tier:** T1 · **Related:** [Arrays, strings and hashing](01-arrays-strings-hashing.md), [Linked lists](03-linked-lists.md), [Stacks and queues](04-stacks-queues.md), [STL for DSA](../03-languages/c-cpp/04-stl-for-dsa.md) · **Language:** C++17

## Quick revision
- Two pointers replace an O(n^2) pair search with O(n) by using order: every step discards an element that cannot be in any answer.
- **Opposite ends** (`l = 0`, `r = n - 1`, move inward): sorted pair sums, 3Sum, palindromes, container/water problems.
- **Same direction** (read/write or fast/slow): in-place compaction, removing duplicates, cycle and middle detection in lists.
- **Fixed window** of size k: add the entering element, remove the leaving one, O(1) per step.
- **Variable window**: expand `right` every step; shrink `left` while the window is invalid (longest) or while it is still valid (shortest).
- A variable window needs monotone validity: shrinking a valid window keeps it valid (or growing an invalid one keeps it invalid). Positive numbers and character counts qualify; negative numbers do not (use prefix sums).
- Each index enters and leaves the window at most once, so the nested `while` is still O(n) in total.
- Window state is usually a sum, an `int cnt[26]`/`cnt[128]` array, a hash map, or a "how many requirements are still missing" counter.
- Window length is `right - left + 1`.
- Sort first when the problem allows it (3Sum); keep original indices if the answer needs them.
- Signals: "sorted", "pair/triplet", "in place", "palindrome", "contiguous", "substring", "longest/shortest", "at most k".

## 1. Concept from scratch

### 1.1 Why two pointers work
The brute force for "find a pair" checks all n(n - 1)/2 pairs. If the data has structure, usually sorted order, one comparison can rule out a whole row of pairs at once. In a sorted array with `a[l] + a[r] < target`, pairing `a[l]` with anything left of `r` gives an even smaller sum, so `a[l]` can never be part of the answer and `l` moves right. Each step removes one candidate, so at most n steps happen.

### 1.2 The three pointer layouts
| Layout | Pointers | Typical problems |
|---|---|---|
| Opposite ends | `l` from the left, `r` from the right, moving inward | Two Sum II, 3Sum, Valid Palindrome, Container With Most Water, Trapping Rain Water |
| Read/write | `read` scans everything, `write` marks where the next kept element goes | Remove Duplicates, Move Zeroes, in-place filtering |
| Fast/slow | one pointer moves twice as fast | Linked list cycle, middle node (see [Linked lists](03-linked-lists.md)) |

### 1.3 Sliding window
A window is a contiguous range `[left, right]`. Instead of recomputing its sum, counts or distinct letters from scratch for every range (O(n) per window), you update the state in O(1) when an element enters on the right or leaves on the left.
- **Fixed size k:** after the first k elements, every step adds one and removes one.
- **Variable size:** `right` advances every iteration. For the longest valid window, shrink while the window is invalid, then record. For the shortest valid window, shrink while the window is valid, recording before each shrink.

### 1.4 When a variable window is allowed
The window logic assumes that growing the window moves the condition in one direction and shrinking moves it back. Examples where that holds: sum of positive numbers, number of distinct characters, count of a letter, number of zeros. Where it fails: sums with negative numbers ("subarray sum equals k" with negatives needs prefix sums and a hash map, see [Arrays, strings and hashing](01-arrays-strings-hashing.md)). If you cannot say "if `[l, r]` is valid then `[l + 1, r]` is valid", do not use a window.

## 2. Core templates
```cpp
#include <algorithm>
#include <climits>
#include <vector>
using namespace std;

// Opposite ends on a sorted array: does any pair add up to target?
bool pairWithSum(const vector<int>& a, long long target) {
    int l = 0, r = (int)a.size() - 1;
    while (l < r) {
        long long sum = (long long)a[l] + a[r];
        if (sum == target) return true;
        if (sum < target) l++;   // a[l] is too small to pair with anything left: drop it
        else r--;                // a[r] is too big to pair with anything right: drop it
    }
    return false;
}

// Read/write pointers: remove duplicates from a sorted array in place, return the new length.
int removeDuplicates(vector<int>& a) {
    int write = 0;
    for (int read = 0; read < (int)a.size(); read++)
        if (write == 0 || a[read] != a[write - 1]) a[write++] = a[read];
    return write;
}

// Fixed window: maximum sum of any k consecutive elements (1 <= k <= a.size()).
long long maxWindowSum(const vector<int>& a, int k) {
    long long sum = 0;
    for (int i = 0; i < k; i++) sum += a[i];
    long long best = sum;
    for (int i = k; i < (int)a.size(); i++) {
        sum += a[i] - a[i - k];  // element i enters, element i - k leaves
        best = max(best, sum);
    }
    return best;
}

// Variable window, longest valid: longest subarray with sum <= limit (values >= 0, limit >= 0).
int longestWithSumAtMost(const vector<int>& a, long long limit) {
    int best = 0, left = 0;
    long long sum = 0;
    for (int right = 0; right < (int)a.size(); right++) {
        sum += a[right];                        // expand
        while (sum > limit) sum -= a[left++];   // shrink until valid again
        best = max(best, right - left + 1);     // [left, right] is valid here
    }
    return best;
}

// Variable window, shortest valid: shortest subarray with sum >= target (values > 0, target > 0).
int shortestWithSumAtLeast(const vector<int>& a, long long target) {
    int best = INT_MAX, left = 0;
    long long sum = 0;
    for (int right = 0; right < (int)a.size(); right++) {
        sum += a[right];
        while (sum >= target) {                 // valid: record it, then try a shorter one
            best = min(best, right - left + 1);
            sum -= a[left++];
        }
    }
    return best == INT_MAX ? 0 : best;
}
```

## 3. Problems

### 3.1 Valid Palindrome (LeetCode 125, Easy)
**Problem (paraphrased):** after ignoring everything except letters and digits, and ignoring case, decide whether a string reads the same forwards and backwards.
**Approach:** brute force builds a cleaned lowercase copy and compares it with its reverse: O(n) time but O(n) extra space. Two pointers from both ends skip non-alphanumeric characters and compare in place.
**Complexity:** O(n) time, O(1) space.
```cpp
class Solution {
public:
    bool isPalindrome(string s) {
        int i = 0, j = (int)s.size() - 1;
        while (i < j) {
            unsigned char a = s[i], b = s[j];        // unsigned: safe for <cctype> functions
            if (!isalnum(a)) { i++; continue; }
            if (!isalnum(b)) { j--; continue; }
            if (tolower(a) != tolower(b)) return false;
            i++;
            j--;
        }
        return true;
    }
};
```
**Edge cases:** empty string or only punctuation (true); a single character; digits mixed with letters.

### 3.2 Two Sum II: Input Array Is Sorted (LeetCode 167, Medium)
**Problem (paraphrased):** in an array sorted in non-decreasing order, find the two positions (1-indexed) whose values sum to a target, using constant extra space. Exactly one answer exists.
**Approach:** brute force checks all pairs, O(n^2); binary-searching the complement for each element is O(n log n). Opposite-end pointers are O(n): if the sum is too small, the left value cannot pair with anything (everything else is at most `numbers[r]`), so move `l`; if too large, move `r`.
**Complexity:** O(n) time, O(1) space.
```cpp
class Solution {
public:
    vector<int> twoSum(vector<int>& numbers, int target) {
        int l = 0, r = (int)numbers.size() - 1;
        while (l < r) {
            int sum = numbers[l] + numbers[r];         // values are small here; use long long if not
            if (sum == target) return {l + 1, r + 1};  // the problem wants 1-indexed positions
            if (sum < target) l++;
            else r--;
        }
        return {};
    }
};
```
**Edge cases:** negative numbers; duplicates such as `[1, 1]` with target 2; the answer at the two ends.

### 3.3 3Sum (LeetCode 15, Medium)
**Problem (paraphrased):** list every distinct triplet of values from the array that adds up to zero, using each position at most once per triplet and never repeating a triplet.
**Approach:** brute force tries all triples, O(n^3), and needs a set to remove repeats. Sort, then fix the first element and run Two Sum II on the rest with two pointers: O(n^2). Skip equal values for the anchor and after each found triplet, so duplicates never appear.
**Complexity:** O(n^2) time, O(1) extra space besides the output (sorting may use O(log n)).
```cpp
class Solution {
public:
    vector<vector<int>> threeSum(vector<int>& nums) {
        sort(nums.begin(), nums.end());
        vector<vector<int>> res;
        int n = nums.size();
        for (int i = 0; i + 2 < n; i++) {
            if (nums[i] > 0) break;                          // smallest value positive: no zero sum left
            if (i > 0 && nums[i] == nums[i - 1]) continue;   // same anchor as before
            int l = i + 1, r = n - 1;
            while (l < r) {
                int sum = nums[i] + nums[l] + nums[r];
                if (sum < 0) l++;
                else if (sum > 0) r--;
                else {
                    res.push_back({nums[i], nums[l], nums[r]});
                    l++;
                    r--;
                    while (l < r && nums[l] == nums[l - 1]) l++;   // skip repeated second values
                    while (l < r && nums[r] == nums[r + 1]) r--;   // skip repeated third values
                }
            }
        }
        return res;
    }
};
```
**Edge cases:** all zeros (exactly one triplet); fewer than three elements; many duplicates. Generalization: kSum is O(n^(k - 1)) by recursing down to the two-pointer case.

### 3.4 Container With Most Water (LeetCode 11, Medium)
**Problem (paraphrased):** vertical lines stand at each index with the given heights. Choose two lines that, with the x-axis, hold the most water; return that area.
**Approach:** brute force tries all pairs, O(n^2). Start with the widest pair. The area is limited by the shorter line, and moving the taller line inward can only shrink the width without raising that limit, so always move the shorter line.
**Complexity:** O(n) time, O(1) space.
```cpp
class Solution {
public:
    int maxArea(vector<int>& height) {
        int l = 0, r = (int)height.size() - 1, best = 0;
        while (l < r) {
            int h = min(height[l], height[r]);
            best = max(best, h * (r - l));           // at most 1e4 * 1e5, fits in int
            if (height[l] < height[r]) l++;          // the shorter line limits every narrower pair
            else r--;
        }
        return best;
    }
};
```
**Edge cases:** two lines; equal heights (moving either side is fine).

### 3.5 Trapping Rain Water (LeetCode 42, Hard)
**Problem (paraphrased):** bars of width 1 have the given heights. Compute how many units of rain water stay trapped between them.
**Approach:** water above bar i is `min(maxLeft(i), maxRight(i)) - height[i]`. Brute force scans left and right for every i, O(n^2). Precomputing prefix and suffix maximum arrays gives O(n) time and O(n) space. Two pointers get O(1) space: move the side with the lower bar. If `height[l] < height[r]`, a bar at least as tall as everything passed on the left exists on the right, so the water at l is decided by `leftMax` alone.
**Complexity:** O(n) time, O(1) space.
```cpp
class Solution {
public:
    int trap(vector<int>& height) {
        int l = 0, r = (int)height.size() - 1;
        int leftMax = 0, rightMax = 0, water = 0;
        while (l < r) {
            if (height[l] < height[r]) {
                leftMax = max(leftMax, height[l]);
                water += leftMax - height[l];        // 0 when height[l] is the new maximum
                l++;
            } else {
                rightMax = max(rightMax, height[r]);
                water += rightMax - height[r];
                r--;
            }
        }
        return water;
    }
};
```
**Edge cases:** fewer than three bars (0); strictly increasing or decreasing heights (0); flat plateaus. A monotonic stack solution also exists (see [Stacks and queues](04-stacks-queues.md)).

### 3.6 Best Time to Buy and Sell Stock (LeetCode 121, Easy)
**Problem (paraphrased):** given daily prices, pick one day to buy and a later day to sell to maximize profit; return 0 if no profit is possible.
**Approach:** brute force tries every buy/sell pair, O(n^2). One pass works because the best sale on day i uses the cheapest price before it: track the minimum so far (the left edge of a shrinking-and-growing window).
**Complexity:** O(n) time, O(1) space.
```cpp
class Solution {
public:
    int maxProfit(vector<int>& prices) {
        int minPrice = INT_MAX, best = 0;
        for (int p : prices) {
            minPrice = min(minPrice, p);         // cheapest day to have bought so far
            best = max(best, p - minPrice);      // profit if you sell today
        }
        return best;
    }
};
```
**Edge cases:** falling prices (0); one day; the minimum appearing after the maximum.

### 3.7 Longest Substring Without Repeating Characters (LeetCode 3, Medium)
**Problem (paraphrased):** find the length of the longest contiguous substring in which no character appears twice.
**Approach:** brute force checks every substring for uniqueness, O(n^3) (O(n^2) with an incremental set). A variable window keeps a repeat-free range; store the last index of every character and, when the incoming character was seen inside the window, jump `left` just past that position.
**Complexity:** O(n) time, O(1) space (256 slots).
```cpp
class Solution {
public:
    int lengthOfLongestSubstring(string s) {
        vector<int> last(256, -1);                    // last index of each byte value
        int best = 0, left = 0;
        for (int right = 0; right < (int)s.size(); right++) {
            unsigned char c = s[right];
            if (last[c] >= left) left = last[c] + 1;  // repeat inside the window: move past it
            last[c] = right;
            best = max(best, right - left + 1);
        }
        return best;
    }
};
```
**Edge cases:** empty string (0); all the same character (1); spaces and symbols count as characters.

### 3.8 Longest Repeating Character Replacement (LeetCode 424, Medium)
**Problem (paraphrased):** with at most k character replacements in an uppercase string, find the longest substring that can be turned into a single repeated letter.
**Approach:** brute force checks every substring with letter counts, O(26 * n^2). A window is fixable when `length - (count of its most common letter) <= k`. Expand right, shrink left while the window needs more than k replacements. `maxFreq` is never decreased when shrinking: a stale, too-large value can only keep a window the same length as the best one already found, so the answer stays correct and the loop stays O(n).
**Complexity:** O(n) time, O(1) space.
```cpp
class Solution {
public:
    int characterReplacement(string s, int k) {
        int count[26] = {0};
        int left = 0, maxFreq = 0, best = 0;
        for (int right = 0; right < (int)s.size(); right++) {
            maxFreq = max(maxFreq, ++count[s[right] - 'A']);
            while ((right - left + 1) - maxFreq > k) {   // too many letters to replace
                count[s[left] - 'A']--;
                left++;
            }
            best = max(best, right - left + 1);
        }
        return best;
    }
};
```
**Edge cases:** k = 0 (longest run of one letter); k at least the string length (the whole string).

### 3.9 Permutation in String (LeetCode 567, Medium)
**Problem (paraphrased):** decide whether some contiguous piece of `s2` is a rearrangement of `s1` (both lowercase).
**Approach:** generating all permutations of `s1` is O(m!). Checking every window of length m by sorting is O(n * m log m). A fixed window of length m slides over `s2` while two 26-count arrays are compared, O(26) per step.
**Complexity:** O(26 * n) time, O(1) space.
```cpp
class Solution {
public:
    bool checkInclusion(string s1, string s2) {
        int m = s1.size(), n = s2.size();
        if (m > n) return false;
        array<int, 26> need{}, window{};               // std::array supports ==
        for (int i = 0; i < m; i++) {
            need[s1[i] - 'a']++;
            window[s2[i] - 'a']++;
        }
        if (need == window) return true;
        for (int i = m; i < n; i++) {
            window[s2[i] - 'a']++;                     // enters on the right
            window[s2[i - m] - 'a']--;                 // leaves on the left
            if (need == window) return true;
        }
        return false;
    }
};
```
**Edge cases:** `s1` longer than `s2`; a match at the very start or end. A "matches" counter makes each step O(1) instead of O(26).

### 3.10 Minimum Window Substring (LeetCode 76, Hard)
**Problem (paraphrased):** find the shortest contiguous piece of `s` that contains every character of `t`, counting repeats; return an empty string if none exists.
**Approach:** brute force checks every substring against the counts of `t`, O(n^2) windows. A variable window does it in O(n + m): `need[c]` holds how many more copies of c the window requires (negative means surplus), and `missing` counts the characters of `t` not yet covered. Expand right; while `missing == 0`, record the window and shrink from the left.
**Complexity:** O(n + m) time, O(1) space (256 counters).
```cpp
class Solution {
public:
    string minWindow(string s, string t) {
        vector<int> need(256, 0);
        for (unsigned char c : t) need[c]++;
        int missing = t.size();                          // characters of t not yet in the window
        int left = 0, bestStart = 0, bestLen = INT_MAX;
        for (int right = 0; right < (int)s.size(); right++) {
            unsigned char c = s[right];
            if (need[c]-- > 0) missing--;                // this copy was still needed
            while (missing == 0) {                       // window covers t: record, then shrink
                if (right - left + 1 < bestLen) {
                    bestLen = right - left + 1;
                    bestStart = left;
                }
                unsigned char out = s[left++];
                if (++need[out] > 0) missing++;          // dropped a needed copy: coverage breaks
            }
        }
        return bestLen == INT_MAX ? "" : s.substr(bestStart, bestLen);
    }
};
```
**Edge cases:** `t` longer than `s`; repeated letters in `t` such as `"aa"`; the whole string as the answer.

## Common pitfalls
- Using two pointers on unsorted data where the "discard" argument does not hold. Sort first, or use a hash map.
- Forgetting to skip duplicates in 3Sum, or skipping them before recording the first occurrence.
- `int r = nums.size() - 1` with an unsigned subtraction on an empty vector; cast first.
- Applying a sliding window to sums with negative numbers.
- Recording the answer at the wrong moment: longest windows record after shrinking, shortest windows record inside the shrink loop.
- Off-by-one in the window length: it is `right - left + 1`.
- Rebuilding window counts from scratch each step, which turns O(n) into O(n * k).
- Overflow in sums of many large values: keep window sums in `long long`.

## How to talk about this in an interview
- Lead with the invariant: "Everything left of `l` and right of `r` can't be in any answer, because..." or "The window `[left, right]` is always valid after the inner loop." Interviewers want the reason the pointer moves, not only the code.
- Explain why it is linear: "Each index enters once and leaves once, so the nested loop is O(n) in total."
- From your Data Structures labs, the board-friendly way to teach this is to draw the array, the two arrows and the window bracket, and move them step by step. Do the same on the interviewer's screen or whiteboard.
- Real-world link: rate limiters count requests in a sliding time window, and analytics dashboards compute rolling averages with fixed windows. Pagination with `LIMIT`/`OFFSET` is a fixed window over sorted rows.
- > **Fill in:** any rolling-window computation you have seen in real code (rolling averages, request throttling, recent-activity feeds) and how it was implemented.

## Interview questions

**Q1. When does the two-pointer technique work, and why does it usually need sorted input?**
It works when one comparison lets you safely throw away one candidate. In a sorted array, if `a[l] + a[r]` is too small, no partner for `a[l]` remains on the left of `r`, so `l` can move. Without sorted order (or another monotone property), that argument fails.
*Follow-up:* What does sorting cost you? O(n log n) time and the original indices, unless you sort pairs of (value, index).

**Q2. Why is a variable sliding window O(n) even though it has a nested loop?**
`right` moves n times in total, and `left` only moves forward and never past `right`, so it moves at most n times in total across all iterations. Total work is O(2n). This is amortized analysis.
*Follow-up:* What makes it degrade? Recomputing the window state from scratch instead of updating it in O(1).

**Q3. How do you recognize a fixed versus a variable window problem?**
Fixed: the window size is given ("every k consecutive", "substring of length m"). Variable: the problem asks for the longest or shortest window satisfying a condition ("at most k distinct", "sum at least target").
*Follow-up:* What if it asks for "exactly k distinct"? Compute atMost(k) - atMost(k - 1).

**Q4. When does a sliding window fail, and what do you use instead?**
When validity is not monotone, typically sums with negative numbers: growing the window can lower the sum. Use prefix sums with a hash map for "sum equals k", or a monotonic deque over prefix sums for "shortest subarray with sum at least k" with negatives.
*Follow-up:* Name the deque problem. Shortest Subarray with Sum at Least K (LeetCode 862).

**Q5. In Container With Most Water, why is it safe to move the shorter line?**
Every container that keeps the shorter line and moves the other inward is narrower and still capped by the shorter height, so none of them can beat the current one. Discarding the shorter line loses nothing.
*Follow-up:* What if heights are equal? Move either; both are capped at that height for every narrower pair.

**Q6. How do you avoid duplicate triplets in 3Sum without a set?**
Sort the array, skip an anchor equal to the previous anchor, and after recording a triplet move both pointers past equal values. Equal values are adjacent after sorting, so this removes repeats in O(1) extra space.
*Follow-up:* Hash-set approach complexity? Also O(n^2), but with extra memory and messier deduplication.

**Q7. How would you handle Unicode in the anagram or permutation window problems?**
Fixed 26-slot arrays assume lowercase ASCII. For Unicode, decode the bytes into code points first and count them in a hash map, being careful that a window must not split a multi-byte character.
*Follow-up:* Does `std::string` know about characters? No, it stores bytes; `size()` is a byte count.

**Q8. Give three ways to solve Trapping Rain Water.**
Prefix and suffix maximum arrays (O(n) time, O(n) space); two pointers with running maxima (O(n) time, O(1) space); a monotonic decreasing stack that fills water between a bar and the next taller bar (O(n) time, O(n) space).
*Follow-up:* What about the 2D version? Trapping Rain Water II uses a min-heap from the borders inward (LeetCode 407).
