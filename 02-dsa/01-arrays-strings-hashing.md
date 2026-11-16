# Arrays, Strings and Hashing

> **Tier:** T1 · **Related:** [Big-O and problem solving](00-big-o-and-problem-solving.md), [Two pointers and sliding window](02-two-pointers-sliding-window.md), [Heaps](08-heaps.md), [STL for DSA](../03-languages/c-cpp/04-stl-for-dsa.md) · **Language:** C++17

## Quick revision
- Arrays and vectors: contiguous memory, O(1) indexing, cache-friendly scans; insert/erase in the middle is O(n); `push_back` is amortized O(1).
- `reserve` changes capacity, `resize` changes size. Reallocation invalidates iterators, pointers and references.
- `v.size()` is unsigned: `v.size() - 1` on an empty vector wraps around. Write `(int)v.size() - 1`.
- `std::string` is a mutable, contiguous sequence of chars: `+=` is amortized O(1) per char, `substr` copies (O(k)), `==` is O(n). `string_view` (C++17) slices without copying but must not outlive its source.
- Map lowercase letters with `c - 'a'`; cast to `unsigned char` before calling `isalnum`, `tolower` and friends.
- Hash table = bucket array + hash function. `unordered_map` uses separate chaining; `max_load_factor()` defaults to 1.0 and exceeding it triggers a rehash.
- Hash operations: O(1) average, O(n) worst case (all keys in one bucket), plus O(L) to hash a string key.
- `m[key]` inserts a default value if the key is missing. Test membership with `count` or `find` (`contains` is C++20).
- No `std::hash` for `pair` or your own structs: write a hasher or encode the key into one integer.
- Frequency counting: `int cnt[26]` for lowercase letters, `unordered_map` for anything else.
- Prefix sums: `pre[i + 1] = pre[i] + a[i]`; sum of `a[l..r]` is `pre[r + 1] - pre[l]`. Use `long long`.
- "Count subarrays with sum k" when negatives are allowed: prefix sums + hash map of counts seeded with `{0: 1}`.
- Signals for hashing: "seen before", "count", "group", "pair with sum", "duplicate", "anagram", "first unique".

## 1. Concept from scratch

### 1.1 Arrays and vectors
An array is a block of equally sized slots placed next to each other in memory. Element i lives at `base + i * sizeof(T)`, which is why indexing is O(1) and why scanning is fast: the CPU pulls neighboring elements into cache together.

| Type | Size | Use it for |
|---|---|---|
| `int a[100]` (C array) | fixed at compile time | rarely in modern C++; decays to a pointer and forgets its size |
| `std::array<int, 26>` | fixed at compile time | small fixed tables such as letter counts; supports `==` and `.size()` |
| `std::vector<int>` | grows at runtime, stored on the heap | the default container for almost everything |

Vector facts worth knowing cold: `size()` versus `capacity()`; `reserve(n)` allocates without changing the size; `push_back` is amortized O(1); `insert` and `erase` in the middle shift the tail, O(n); `v[i]` is unchecked (out of range is undefined behavior) while `v.at(i)` throws `std::out_of_range`. A grid is `vector<vector<int>> grid(rows, vector<int>(cols, 0));`.

### 1.2 std::string essentials
| Operation | Cost | Note |
|---|---|---|
| `s[i]`, `s.size()`, `s.back()` | O(1) | `back()` on an empty string is undefined behavior |
| `s += t`, `s.push_back(c)` | amortized O(len t) | appends in place |
| `s + t` | O(len s + len t) | allocates a new string |
| `s.substr(pos, k)` | O(k) | copies; `string_view::substr` does not |
| `s.find(t)` | no complexity guarantee | C++17 adds Boyer-Moore searchers in `<functional>` |
| `s == t`, `s < t` | O(length) | lexicographic comparison |
| `sort(s.begin(), s.end())` | O(n log n) | canonical key for anagrams |
| `to_string(x)`, `stoi(s)`, `stoll(s)` | O(digits) | `stoi` throws on invalid input |

The `<cctype>` helpers (`isdigit`, `isalpha`, `isalnum`, `tolower`, `toupper`) take an `int` that must fit in `unsigned char`. Passing a negative `char` (any non-ASCII byte) is undefined behavior, so write `isalnum(static_cast<unsigned char>(c))`.

### 1.3 How a hash table works
1. A **hash function** turns the key into a number.
2. The table picks a bucket: `index = hash(key) % bucket_count`.
3. Two keys in the same bucket is a **collision**. `std::unordered_map` resolves collisions with **separate chaining**: each bucket holds a chain of nodes. Many other tables use **open addressing** (probe the following slots), which is more cache-friendly; Python's `dict` is one example.
4. The **load factor** is `size / bucket_count`. When it would exceed `max_load_factor()` (1.0 by default), the table **rehashes**: it allocates more buckets and reinserts every element. That one insert is O(n), but it happens rarely enough to stay amortized O(1). Rehashing invalidates iterators, but not pointers or references to the elements.
5. **Cost:** O(1) average for find, insert and erase, plus the cost of hashing the key. **Worst case O(n)** per operation when many keys collide, for example with adversarial input built against a known hash function.

```cpp
#include <string>
#include <unordered_map>
#include <unordered_set>
using namespace std;

void hashDemo() {
    unordered_map<string, int> stock;
    stock.reserve(1000);                    // pre-size the buckets to avoid rehashing
    stock["sku-1"] = 5;                     // insert or overwrite
    if (stock.count("sku-2")) { /* ... */ } // membership test that does not insert
    auto it = stock.find("sku-1");          // iterator, or stock.end() if missing
    if (it != stock.end()) it->second++;
    stock.erase("sku-1");

    unordered_set<int> seen;
    bool isNew = seen.insert(42).second;    // false if 42 was already present
    (void)isNew;
}
```

| | `unordered_map` | `map` |
|---|---|---|
| Structure | hash table (buckets + chains) | balanced BST (a red-black tree in the common implementations) |
| find / insert / erase | O(1) average, O(n) worst | O(log n) guaranteed |
| Ordered iteration, `lower_bound` | no | yes |
| Key needs | `std::hash<Key>` and `==` | `operator<` (a strict weak ordering) |

There is no `std::hash` for `pair`. Encode the key into one integer, or pass a hasher:
```cpp
#include <functional>
#include <unordered_set>
#include <utility>
using namespace std;

struct PairHash {
    size_t operator()(const pair<int, int>& p) const {
        // Pack both ints into 64 unsigned bits (unsigned shifts are always well defined), then hash.
        unsigned long long key = ((unsigned long long)(unsigned int)p.first << 32) | (unsigned int)p.second;
        return hash<unsigned long long>()(key);
    }
};

unordered_set<pair<int, int>, PairHash> visited;   // for example, visited grid cells
```

### 1.4 Frequency counting
When keys are small integers or lowercase letters, a plain array beats a hash map: no hashing, no allocations, better cache behavior.
```cpp
int cnt[26] = {0};
for (char c : s) cnt[c - 'a']++;   // valid only when s holds lowercase a-z
```
For arbitrary keys use `unordered_map<int, int> freq; for (int x : nums) freq[x]++;`. Here `operator[]` default-inserting 0 is exactly what you want.

### 1.5 Prefix sums
Precompute running totals once, then answer any range-sum query in O(1).
```cpp
#include <vector>
using namespace std;

vector<long long> buildPrefix(const vector<int>& a) {
    vector<long long> pre(a.size() + 1, 0);              // pre[i] = a[0] + ... + a[i - 1]
    for (size_t i = 0; i < a.size(); i++) pre[i + 1] = pre[i] + a[i];
    return pre;
}
// Sum of a[l..r] inclusive: pre[r + 1] - pre[l]
```
The leading zero removes the special case for `l = 0`. The same idea works for prefix XOR, prefix counts ("how many vowels in `s[l..r]`?") and 2D prefix sums for submatrix totals: `P[r+1][c+1] = a[r][c] + P[r][c+1] + P[r+1][c] - P[r][c]`.

## 2. Core templates

```cpp
#include <unordered_map>
#include <unordered_set>
#include <vector>
using namespace std;

// Complement lookup: does any pair add up to target?
bool hasPairWithSum(const vector<int>& a, long long target) {
    unordered_set<long long> seen;
    for (int x : a) {
        if (seen.count(target - x)) return true;   // the partner appeared earlier
        seen.insert(x);
    }
    return false;
}

// Prefix sum + hash map: count subarrays whose sum is exactly k (negatives allowed).
int countSubarraysWithSum(const vector<int>& a, long long k) {
    unordered_map<long long, int> countOf{{0, 1}};  // the empty prefix has sum 0
    long long prefix = 0;
    int result = 0;
    for (int x : a) {
        prefix += x;
        if (auto it = countOf.find(prefix - k); it != countOf.end()) result += it->second;  // C++17 if-init
        countOf[prefix]++;
    }
    return result;
}
```
The third template, **group by a canonical key**, is one line: `groups[keyOf(x)].push_back(x)`, where the key is a sorted string, a count signature or a normalized pattern.

## 3. Problems

### 3.1 Two Sum (LeetCode 1, Easy)
**Problem (paraphrased):** return the indices of two different positions in an integer array whose values add up to a target. Exactly one answer exists.
**Approach:** brute force checks every pair, O(n^2). Better: walk once and keep a map from value to index; for each element, look up `target - nums[i]` before inserting the element itself. The full out-loud walk-through is in [Big-O and problem solving](00-big-o-and-problem-solving.md).
**Complexity:** O(n) time on average, O(n) space.
```cpp
class Solution {
public:
    vector<int> twoSum(vector<int>& nums, int target) {
        unordered_map<int, int> indexOf;                 // value -> index
        for (int i = 0; i < (int)nums.size(); i++) {
            auto it = indexOf.find(target - nums[i]);    // |values| <= 1e9, so the difference fits in int
            if (it != indexOf.end()) return {it->second, i};
            indexOf[nums[i]] = i;
        }
        return {};
    }
};
```
**Edge cases:** duplicates such as `[3, 3]` with target 6; negatives; the pair at the very end.

### 3.2 Contains Duplicate (LeetCode 217, Easy)
**Problem (paraphrased):** report whether any value appears at least twice in an array.
**Approach:** brute force compares all pairs, O(n^2). Sorting and comparing neighbors is O(n log n) with O(1) extra space. A hash set gives O(n): `insert` tells you whether the value was already present.
**Complexity:** O(n) average time, O(n) space.
```cpp
class Solution {
public:
    bool containsDuplicate(vector<int>& nums) {
        unordered_set<int> seen;
        seen.reserve(nums.size());
        for (int x : nums)
            if (!seen.insert(x).second) return true;   // insert fails: already seen
        return false;
    }
};
```
**Edge cases:** one element; all equal; when memory is tight, prefer the in-place sort.

### 3.3 Valid Anagram (LeetCode 242, Easy)
**Problem (paraphrased):** decide whether two lowercase strings use exactly the same letters with the same counts.
**Approach:** sorting both and comparing is O(n log n). Counting is O(n): add for one string, subtract for the other. With equal lengths, if no count ever drops below zero, all counts end at zero.
**Complexity:** O(n) time, O(1) space (26 counters).
```cpp
class Solution {
public:
    bool isAnagram(string s, string t) {
        if (s.size() != t.size()) return false;
        int cnt[26] = {0};
        for (char c : s) cnt[c - 'a']++;
        for (char c : t)
            if (--cnt[c - 'a'] < 0) return false;   // t uses a letter more often than s
        return true;
    }
};
```
**Edge cases:** different lengths. Follow-up for Unicode: switch to `unordered_map<char32_t, int>` over decoded code points.

### 3.4 Group Anagrams (LeetCode 49, Medium)
**Problem (paraphrased):** split a list of words into groups where every group contains words that are anagrams of each other.
**Approach:** brute force compares every pair with the anagram check, O(n^2 * k). Instead give each word a canonical key that all its anagrams share: its sorted letters (O(k log k)) or its 26 letter counts (O(k)). Then group by key in a hash map.
**Complexity:** O(n * k log k) time with sorted keys, O(n * k) space, for n words of length up to k.
```cpp
class Solution {
public:
    vector<vector<string>> groupAnagrams(vector<string>& strs) {
        unordered_map<string, vector<string>> groups;
        for (const string& s : strs) {
            string key = s;
            sort(key.begin(), key.end());          // anagrams share the same sorted form
            groups[key].push_back(s);
        }
        vector<vector<string>> result;
        result.reserve(groups.size());
        for (auto& [key, group] : groups) result.push_back(move(group));   // C++17 structured binding
        return result;
    }
};
```
**Edge cases:** empty strings (they group together); a single word. Count-signature key: build a string such as `"1#0#2#..."` from the 26 counts.

### 3.5 Top K Frequent Elements (LeetCode 347, Medium)
**Problem (paraphrased):** return the k values that occur most often in an array (the answer is unique, any order).
**Approach:** count with a hash map, then sorting the (value, count) pairs by count is O(n log n); a size-k heap is O(n log k) (see [Heaps](08-heaps.md)). **Bucket sort** is O(n): a frequency is between 1 and n, so put each value into `bucket[frequency]` and read buckets from high to low until you have k values.
**Complexity:** O(n) time, O(n) space.
```cpp
class Solution {
public:
    vector<int> topKFrequent(vector<int>& nums, int k) {
        unordered_map<int, int> freq;
        for (int x : nums) freq[x]++;
        int n = nums.size();
        vector<vector<int>> bucket(n + 1);                 // bucket[f] = values seen exactly f times
        for (auto& [value, f] : freq) bucket[f].push_back(value);
        vector<int> result;
        for (int f = n; f >= 1 && (int)result.size() < k; f--) {
            for (int value : bucket[f]) {
                result.push_back(value);
                if ((int)result.size() == k) break;
            }
        }
        return result;
    }
};
```
**Edge cases:** k equal to the number of distinct values; every value appears once.

### 3.6 Product of Array Except Self (LeetCode 238, Medium)
**Problem (paraphrased):** for each index, output the product of all other elements, without using division, in O(n).
**Approach:** brute force multiplies everything else for each index, O(n^2). Dividing the total product by `nums[i]` breaks on zeros and is not allowed. The answer at i is (product of everything left of i) x (product of everything right of i): store the left products in the output array, then sweep from the right with one running variable.
**Complexity:** O(n) time, O(1) extra space (the output array is not counted).
```cpp
class Solution {
public:
    vector<int> productExceptSelf(vector<int>& nums) {
        int n = nums.size();
        vector<int> answer(n, 1);
        for (int i = 1; i < n; i++) answer[i] = answer[i - 1] * nums[i - 1];   // product of nums[0..i-1]
        int right = 1;                                                        // product of nums[i+1..n-1]
        for (int i = n - 1; i >= 0; i--) {
            answer[i] *= right;
            right *= nums[i];
        }
        return answer;
    }
};
```
**Edge cases:** one zero (only that index is non-zero); two zeros (all zeros); the problem guarantees every prefix and suffix product fits in 32 bits.

### 3.7 Longest Consecutive Sequence (LeetCode 128, Medium)
**Problem (paraphrased):** find the length of the longest run of consecutive integers (like 4, 5, 6, 7) that can be formed from the array's values, in O(n).
**Approach:** sorting and scanning is O(n log n). With a hash set, start counting only at values that begin a run (`x - 1` is absent), then walk `x + 1, x + 2, ...`. Each value is visited by at most one walk, so the total is O(n).
**Complexity:** O(n) average time, O(n) space.
```cpp
class Solution {
public:
    int longestConsecutive(vector<int>& nums) {
        unordered_set<int> s(nums.begin(), nums.end());
        int best = 0;
        for (int x : s) {                       // iterate the set, not nums: duplicates would repeat walks
            if (s.count(x - 1)) continue;       // not the start of a run
            int len = 1;
            while (s.count(x + len)) len++;
            best = max(best, len);
        }
        return best;
    }
};
```
**Edge cases:** empty array (0); duplicates; negatives. Values are within plus or minus 1e9, so `x - 1` cannot overflow.

### 3.8 Subarray Sum Equals K (LeetCode 560, Medium)
**Problem (paraphrased):** count the contiguous subarrays whose elements add up to exactly k. Values can be negative.
**Approach:** brute force fixes a start and extends the end with a running sum, O(n^2). A sliding window fails because negatives break the "grow to increase, shrink to decrease" logic. Use prefix sums: subarray `(j, i]` sums to k exactly when `prefix[i] - prefix[j] = k`, so at each i count how many earlier prefixes equal `prefix[i] - k`.
**Complexity:** O(n) average time, O(n) space.
```cpp
class Solution {
public:
    int subarraySum(vector<int>& nums, int k) {
        unordered_map<long long, int> seen{{0, 1}};   // prefix sum -> occurrences; the empty prefix counts
        long long prefix = 0;
        int count = 0;
        for (int x : nums) {
            prefix += x;
            auto it = seen.find(prefix - k);
            if (it != seen.end()) count += it->second;
            seen[prefix]++;                           // insert after the lookup
        }
        return count;
    }
};
```
**Edge cases:** k = 0; the subarray starting at index 0 (that is what the `{0, 1}` seed handles); all negatives.

### 3.9 Majority Element (LeetCode 169, Easy)
**Problem (paraphrased):** return the value that occupies more than half of the array (it always exists).
**Approach:** counting with a hash map is O(n) time and O(n) space; sorting and taking the middle is O(n log n). **Boyer-Moore voting** is O(n) time and O(1) space: keep a candidate and a counter, add one for a match and subtract one otherwise, switching candidates when the counter is zero. Pairing each majority element with a different element cancels at most half of the array, so the majority value survives.
**Complexity:** O(n) time, O(1) space.
```cpp
class Solution {
public:
    int majorityElement(vector<int>& nums) {
        int candidate = 0, count = 0;
        for (int x : nums) {
            if (count == 0) candidate = x;
            count += (x == candidate) ? 1 : -1;
        }
        return candidate;   // if a majority were not guaranteed, verify with a second counting pass
    }
};
```
**Edge cases:** one element. Variant: values appearing more than n/3 times need two candidates.

### 3.10 Valid Sudoku (LeetCode 36, Medium)
**Problem (paraphrased):** check a partially filled 9 x 9 board: no digit may repeat within a row, a column or a 3 x 3 box. Solvability does not matter.
**Approach:** brute force validates each row, each column and each box separately (27 scans). One pass is enough with three tables of seen flags; the box index is `(r / 3) * 3 + c / 3`.
**Complexity:** O(81), so O(1) time and space for a fixed board (O(n^2) for an n x n board).
```cpp
class Solution {
public:
    bool isValidSudoku(vector<vector<char>>& board) {
        bool row[9][9] = {}, col[9][9] = {}, box[9][9] = {};
        for (int r = 0; r < 9; r++) {
            for (int c = 0; c < 9; c++) {
                if (board[r][c] == '.') continue;
                int d = board[r][c] - '1';         // digits '1'..'9' become 0..8
                int b = (r / 3) * 3 + c / 3;       // which 3 x 3 box
                if (row[r][d] || col[c][d] || box[b][d]) return false;
                row[r][d] = col[c][d] = box[b][d] = true;
            }
        }
        return true;
    }
};
```
**Edge cases:** an empty board is valid; a duplicate inside a box but in different rows and columns.

## Common pitfalls
- Using `m[key]` to check membership: it silently inserts the key with a default value, growing the map and corrupting counts.
- `v.size() - 1` or `i < v.size() - k` with unsigned arithmetic when the vector is short: the subtraction wraps to a huge number.
- `int` overflow in sums and products: use `long long` for prefix sums and anything that multiplies.
- Passing a negative `char` to `isalnum`/`tolower`: cast to `unsigned char` first.
- `c - 'a'` on input that may contain uppercase letters, digits or spaces: out-of-bounds writes. Check the constraints.
- Copying strings in a loop (`s.substr`, `s + t`) inside an O(n) loop, giving an O(n^2) solution by accident.
- Assuming `unordered_map` iteration order is stable or sorted. It is neither; use `map` or sort the output.
- Iterating `nums` instead of the set in Longest Consecutive Sequence, which repeats walks for duplicates.
- Modifying a container (inserting into a vector or rehashing a map) while holding iterators into it.

## How to talk about this in an interview
- Name the trade-off out loud: "I'll spend O(n) memory on a hash map to turn the inner search into an O(1) lookup."
- Tie hashing to your stack: Ruby's `Hash`, JavaScript's `Map`, and Python's `dict` are hash tables, and Rails' `index_by` or `group_by` builds one so you can match records without nested loops. A Rails N+1 fix is often "preload, then index the results by id".
- > **Fill in:** a feature or job (for example a price or stock sync, or a report) where you matched two lists of records by key; did you use a hash lookup, a SQL join, or nested loops at first?
- As a Data Structures lab TA you explained this material before; reuse that: draw the buckets, show two keys colliding, and explain why the worst case is O(n).
- > **Fill in:** a hashing or array bug you helped a student or teammate find (off-by-one, overflow, map default insertion) and how you spotted it.

## Interview questions

**Q1. How does a hash map work, and what is its worst case?**
A hash function maps the key to a bucket index (`hash % bucket_count`). Colliding keys share a bucket; `std::unordered_map` chains them in a list, while open-addressing tables probe nearby slots. When the load factor passes its maximum, the table rehashes into more buckets. Operations are O(1) on average and O(n) in the worst case, when many keys land in the same bucket.
*Follow-up:* How do you defend against deliberate collisions? Use a randomized or keyed hash (many languages seed their string hash per process), or a balanced tree map with guaranteed O(log n).

**Q2. When would you pick `std::map` over `std::unordered_map`?**
When you need sorted iteration, range queries (`lower_bound`, `upper_bound`), a guaranteed O(log n) bound, or keys that are easy to compare but awkward to hash (such as pairs). Otherwise `unordered_map` is usually faster.
*Follow-up:* Which one does a `pair<int, int>` key work with out of the box? `map`, because `pair` has `operator<` but no `std::hash`.

**Q3. What happens to iterators and pointers when a vector grows?**
If `push_back` exceeds the capacity, the vector allocates a new block and moves the elements, so every iterator, pointer and reference into it becomes dangling. Without reallocation, only iterators at or after the insertion point are invalidated.
*Follow-up:* How do you make it safe? Call `reserve` up front, or store indices instead of pointers.

**Q4. Why use prefix sums, and how do they extend to 2D?**
They turn each range-sum query from O(n) into O(1) after an O(n) precomputation. In 2D, `P[r][c]` holds the sum of the rectangle from the origin, and any submatrix sum is four lookups combined by inclusion-exclusion.
*Follow-up:* What if values change between queries? Use a Fenwick tree or a segment tree for O(log n) updates and queries.

**Q5. Why can't a sliding window solve "subarray sum equals k" when values can be negative?**
The window logic assumes that adding an element increases the sum and removing one decreases it. Negatives break that monotonicity, so you cannot decide which end to move. Prefix sums with a hash map handle any sign.
*Follow-up:* When does the window work? When all values are positive, as in Minimum Size Subarray Sum.

**Q6. Array versus `std::array` versus `std::vector`: which do you use?**
`vector` by default because it knows its size and grows. `std::array` when the size is a compile-time constant (letter counts, fixed directions) because it avoids a heap allocation and still has value semantics and `==`. Raw C arrays mostly in C code or tiny local buffers.
*Follow-up:* What does passing a `vector` by value cost? A full O(n) copy; pass by `const&` unless you need a copy.

**Q7. How would you detect duplicates with very little extra memory?**
Sort in place and compare neighbors: O(n log n) time, O(1) extra space (beyond the sort's own stack). If values are bounded integers, a bitset uses one bit per possible value. For huge data, a Bloom filter answers "possibly seen" with a tunable false-positive rate.
*Follow-up:* What if you cannot modify the input? Copy it, or accept the hash set's O(n) memory.

**Q8. How is a string different in C, C++ and your web languages?**
A C string is a `char` array terminated by `'\0'`, so its length is O(n) to compute. `std::string` stores its length and is mutable. Ruby strings are mutable (unless frozen); JavaScript and Python strings are immutable, so repeated concatenation creates new strings.
*Follow-up:* Why does that matter for complexity? Building a string with repeated `s = s + x` can turn an O(n) loop into O(n^2).

**Q9. What is the time complexity of building an anagram key by sorting versus counting?**
Sorting a word of length k costs O(k log k); a 26-count signature costs O(k) to build and O(26) to turn into a key. For long words the count signature wins; for short words sorting is simpler and fast enough.
*Follow-up:* What if the alphabet is all of Unicode? Use a map of counts, or sort the code points.
