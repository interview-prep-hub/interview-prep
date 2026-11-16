# Sorting and Binary Search

> **Tier:** T1 · **Related:** [Big-O and problem solving](00-big-o-and-problem-solving.md), [Two pointers and sliding window](02-two-pointers-sliding-window.md), [Heaps](08-heaps.md), [STL for DSA](../03-languages/c-cpp/04-stl-for-dsa.md), [Indexing and query optimization](../12-databases/05-indexing-and-query-optimization.md) · **Language:** C++17

## Quick revision
- Comparison sorts cannot beat O(n log n) comparisons in the worst case.
- **Merge sort:** O(n log n) always, stable, O(n) extra memory; natural for linked lists and external (disk) sorting.
- **Quick sort:** O(n log n) average, O(n^2) worst case with bad pivots; in place, not stable. Use a random pivot, and a three-way partition when there are many duplicates.
- **Heap sort:** O(n log n) worst case, O(1) extra, not stable, cache-unfriendly.
- **Insertion sort:** O(n^2), but O(n) on nearly sorted input; hybrid sorts use it for small ranges.
- **Counting sort:** O(n + k) for integers in a small range k; radix sort repeats a stable counting sort per digit.
- **Stable** means equal keys keep their input order. It matters for multi-pass, multi-key sorting.
- `std::sort` is an introsort (quick sort + heap sort fallback + insertion sort): O(n log n) worst case since C++11, not stable. `std::stable_sort` is stable.
- Comparators must be a strict weak ordering: use `<` or `>`, never `<=`. A bad comparator is undefined behavior.
- Binary search needs a monotone predicate (false...false true...true) over a random-access space; O(log n).
- Boundary template: `lo = 0, hi = n; while (lo < hi) { mid = lo + (hi - lo) / 2; if (ok(mid)) hi = mid; else lo = mid + 1; }` returns the first true index (or n).
- `lower_bound(x)` = first element >= x; `upper_bound(x)` = first element > x; occurrences of x = upper - lower.
- Binary search on the answer: "smallest speed/capacity/time that works" when feasibility is monotone.
- Write `mid = lo + (hi - lo) / 2` so `lo + hi` cannot overflow.

## 1. Concept from scratch

### 1.1 Why sorting is a building block
Sorting turns many hard problems into easy ones: duplicates become neighbors, two pointers and binary search become possible, and greedy interval algorithms need a sorted order. When an interviewer's constraints allow O(n log n), "sort first" is often the first optimization to try.

### 1.2 The simple O(n^2) sorts (briefly)
- **Bubble sort:** repeatedly swap adjacent out-of-order pairs. With an early exit when a pass makes no swaps, it is O(n) on sorted input. Stable.
- **Selection sort:** find the minimum of the unsorted part and swap it to the front. Always O(n^2) comparisons but at most n - 1 swaps. Not stable, because the long-distance swap can jump an element over an equal one.
- **Insertion sort:** grow a sorted prefix by inserting each new element into place. O(n^2) worst case, O(n) on nearly sorted data, stable and in place, which is why library sorts switch to it for small ranges.

```cpp
#include <vector>
using namespace std;

void insertionSort(vector<int>& a) {
    for (int i = 1; i < (int)a.size(); i++) {
        int key = a[i], j = i - 1;
        while (j >= 0 && a[j] > key) {   // strict >: equal elements keep their order (stable)
            a[j + 1] = a[j];             // shift larger elements one slot right
            j--;
        }
        a[j + 1] = key;
    }
}
```

### 1.3 Merge sort
Split the range in half, sort each half recursively, then merge the two sorted halves with two pointers. T(n) = 2T(n/2) + O(n) = O(n log n) in every case. Taking from the left half on ties keeps it stable. Arrays need an O(n) buffer; linked lists can be merged by relinking nodes (Sort List, LeetCode 148). Merge sort is also the basis of external sorting (sort chunks that fit in memory, then k-way merge them) and of Timsort, the hybrid merge/insertion sort used by Python and by Java for object arrays.

```cpp
#include <vector>
using namespace std;

void mergeSort(vector<int>& a, vector<int>& tmp, int lo, int hi) {   // sorts a[lo, hi)
    if (hi - lo <= 1) return;
    int mid = lo + (hi - lo) / 2;
    mergeSort(a, tmp, lo, mid);
    mergeSort(a, tmp, mid, hi);
    int i = lo, j = mid, k = lo;
    while (i < mid && j < hi) tmp[k++] = (a[j] < a[i]) ? a[j++] : a[i++];   // ties take the left: stable
    while (i < mid) tmp[k++] = a[i++];
    while (j < hi) tmp[k++] = a[j++];
    for (int t = lo; t < hi; t++) a[t] = tmp[t];
}

void mergeSort(vector<int>& a) {
    vector<int> tmp(a.size());
    mergeSort(a, tmp, 0, (int)a.size());
}
```

### 1.4 Quick sort
Choose a pivot, partition the range so smaller elements end up on its left and larger ones on its right, then recurse on both sides. The pivot is in its final position after partitioning.
- **Average O(n log n):** a random pivot splits the range reasonably well most of the time.
- **Worst case O(n^2):** the pivot is always the smallest or largest element, for example the last element of already sorted input. Random pivots make this astronomically unlikely; introsort removes it entirely by switching to heap sort when recursion gets too deep.
- In place, with O(log n) expected stack. Recursing into the smaller side first (and looping on the larger) caps the stack at O(log n) even in the worst case.
- Not stable. With many equal keys, the two-way Lomuto partition degrades toward O(n^2); a three-way partition (less, equal, greater) fixes it, as in Sort Colors below.

```cpp
#include <random>
#include <utility>
#include <vector>
using namespace std;

mt19937 rng(random_device{}());

int partitionLomuto(vector<int>& a, int lo, int hi) {      // partitions a[lo..hi], returns the pivot index
    uniform_int_distribution<int> pick(lo, hi);
    swap(a[pick(rng)], a[hi]);                              // random pivot, moved to the end
    int pivot = a[hi], store = lo;
    for (int i = lo; i < hi; i++)
        if (a[i] < pivot) swap(a[i], a[store++]);           // grow the "< pivot" region
    swap(a[store], a[hi]);                                  // pivot lands in its final place
    return store;
}

void quickSort(vector<int>& a, int lo, int hi) {            // sorts a[lo..hi] inclusive
    if (lo >= hi) return;
    int p = partitionLomuto(a, lo, hi);
    quickSort(a, lo, p - 1);
    quickSort(a, p + 1, hi);
}
```

### 1.5 Heap sort and counting sort (briefly)
- **Heap sort:** build a max-heap in O(n), then repeatedly swap the root to the end of the array and sift the new root down. O(n log n) worst case, O(1) extra space, not stable, and slower in practice than quick sort because its memory accesses jump around. It is introsort's fallback. See [Heaps](08-heaps.md).
- **Counting sort:** for integers in `[0, k]`, count occurrences, then write each value out count times: O(n + k) time and space, no comparisons. For sorting records by a key stably, compute prefix sums of the counts and place records scanning the input from the right. **Radix sort** runs a stable counting sort on each digit, O(d * (n + base)).

```cpp
#include <vector>
using namespace std;

vector<int> countingSort(const vector<int>& a, int maxValue) {   // values in [0, maxValue]
    vector<int> count(maxValue + 1, 0);
    for (int x : a) count[x]++;
    vector<int> out;
    out.reserve(a.size());
    for (int v = 0; v <= maxValue; v++) out.insert(out.end(), count[v], v);   // count[v] copies of v
    return out;
}
```

### 1.6 Stability
A sort is **stable** if elements with equal keys keep their original relative order. Example: sort orders by date, then stable-sort them by customer; each customer's orders stay in date order. Across your stacks: JavaScript's `Array.prototype.sort` is required to be stable since ES2019; Python's `sorted` and `list.sort` are stable; Ruby's `sort` and `sort_by` are not guaranteed stable; and SQL `ORDER BY` gives no guaranteed order among ties, so add a unique tiebreaker such as `id`, especially for pagination.

### 1.7 Comparison table
| Algorithm | Best | Average | Worst | Extra space | Stable |
|---|---|---|---|---|---|
| Bubble (early exit) | O(n) | O(n^2) | O(n^2) | O(1) | yes |
| Selection | O(n^2) | O(n^2) | O(n^2) | O(1) | no |
| Insertion | O(n) | O(n^2) | O(n^2) | O(1) | yes |
| Merge | O(n log n) | O(n log n) | O(n log n) | O(n) | yes |
| Quick (random pivot) | O(n log n) | O(n log n) | O(n^2) | O(log n) stack expected | no |
| Heap | O(n log n) | O(n log n) | O(n log n) | O(1) | no |
| Counting (range k) | O(n + k) | O(n + k) | O(n + k) | O(n + k) | yes (prefix-sum version) |
| `std::sort` (introsort) | O(n log n) | O(n log n) | O(n log n) | O(log n) | no |
| `std::stable_sort` | O(n log n) | O(n log n) | O(n log^2 n) without spare memory | O(n) if available | yes |

### 1.8 std::sort and comparators
- `std::sort` needs random-access iterators, so it works on `vector`, `deque` and arrays but not `std::list` (use `list.sort()`).
- Related algorithms: `stable_sort`; `partial_sort` (the smallest k in order, O(n log k)); `nth_element` (puts the k-th element in place with smaller ones before it, O(n) average).
- A comparator answers "should a come before b?". It must be a **strict weak ordering**: `comp(a, a)` is false, it is transitive, and equivalence is transitive. Writing `<=` violates the first rule and is undefined behavior; in practice `std::sort` can run past the end of the array and crash.

```cpp
#include <algorithm>
#include <string>
#include <tuple>
#include <vector>
using namespace std;

struct Order { string customer; int total; int id; };

void sortOrders(vector<Order>& orders) {
    // Highest total first; ties broken by id so the result is fully deterministic.
    sort(orders.begin(), orders.end(), [](const Order& a, const Order& b) {
        if (a.total != b.total) return a.total > b.total;
        return a.id < b.id;
    });
    // Several ascending keys at once: tuples compare lexicographically.
    sort(orders.begin(), orders.end(), [](const Order& a, const Order& b) {
        return tie(a.customer, a.total) < tie(b.customer, b.total);
    });
}
```
For plain descending order, pass `greater<int>()`: `sort(v.begin(), v.end(), greater<int>());`.

### 1.9 Binary search: conditions and invariants
Binary search works when two things hold:
1. **Random access** to the search space: an index range of an array, or a range of candidate answer values.
2. **A monotone predicate:** some yes/no question that is false for a prefix of the space and true for the rest (or the reverse). In a sorted array, "is `a[i] >= x`?" has this shape.

Each comparison throws away half of the remaining space, so it takes about log2(n) steps: 20 steps for a million elements. Keep an explicit invariant in mind and the off-by-one bugs disappear. For the boundary template: the answer is always inside `[lo, hi]`, everything before `lo` is false, and everything from `hi` on is true. The loop ends when `lo == hi`, which is the first true index (or n if none).

If you ever write `lo = mid` instead of `lo = mid + 1`, compute an upper middle (`mid = lo + (hi - lo + 1) / 2`), otherwise a two-element range can loop forever.

### 1.10 lower_bound and upper_bound
| Call on a sorted range | Returns | Typical use |
|---|---|---|
| `lower_bound(b, e, x)` | first element `>= x` | insert position, first occurrence |
| `upper_bound(b, e, x)` | first element `> x` | one past the last occurrence |
| `binary_search(b, e, x)` | `bool` | membership only |
| `equal_range(b, e, x)` | pair of the two bounds | all copies of x |

```cpp
#include <algorithm>
#include <vector>
using namespace std;

void boundsDemo() {
    vector<int> a = {1, 2, 2, 2, 5};
    int first = lower_bound(a.begin(), a.end(), 2) - a.begin();   // 1: first element >= 2
    int after = upper_bound(a.begin(), a.end(), 2) - a.begin();   // 4: first element > 2
    int copies = after - first;                                    // 3
    bool hasFive = binary_search(a.begin(), a.end(), 5);           // true
    (void)copies; (void)hasFive;
}
```
On a `std::set` or `std::map`, call the member functions (`s.lower_bound(x)`, O(log n)). The free function `std::lower_bound` on their iterators is O(n), because tree iterators are not random access.

### 1.11 Binary search on the answer
Sometimes the array is not sorted, but the answer is a number in a known range and feasibility is monotone: if eating speed k finishes in time, every faster speed also does. Then binary search over the answer values and run an O(n) feasibility check at each step, for O(n log(range)) in total. Signals: "minimum speed/capacity/time such that...", "minimize the largest...", "maximize the smallest...". Classic examples: Koko Eating Bananas (875), Capacity To Ship Packages Within D Days (1011), Split Array Largest Sum (410).

## 2. Core templates
```cpp
#include <functional>
#include <vector>
using namespace std;

// Exact match on a sorted array: index of target, or -1. Search space is the closed range [lo, hi].
int exactSearch(const vector<int>& a, int target) {
    int lo = 0, hi = (int)a.size() - 1;
    while (lo <= hi) {
        int mid = lo + (hi - lo) / 2;
        if (a[mid] == target) return mid;
        if (a[mid] < target) lo = mid + 1;
        else hi = mid - 1;
    }
    return -1;
}

// Boundary search: first value in [lo, hi) where ok is true, or hi if none.
// ok must be monotone: false ... false true ... true.
long long firstTrue(long long lo, long long hi, const function<bool(long long)>& ok) {
    while (lo < hi) {
        long long mid = lo + (hi - lo) / 2;
        if (ok(mid)) hi = mid;      // mid might be the answer: keep it in range
        else lo = mid + 1;          // mid is definitely not the answer
    }
    return lo;
}
// lower_bound is firstTrue(0, n, [&](long long i) { return a[i] >= x; }).
```

## 3. Problems

### 3.1 Binary Search (LeetCode 704, Easy)
**Problem (paraphrased):** return the index of a target in a sorted array of distinct integers, or -1, in O(log n).
**Approach:** a linear scan is O(n). Binary search over the closed range `[lo, hi]` halves the space each step.
**Complexity:** O(log n) time, O(1) space.
```cpp
class Solution {
public:
    int search(vector<int>& nums, int target) {
        int lo = 0, hi = (int)nums.size() - 1;     // search space is [lo, hi]
        while (lo <= hi) {
            int mid = lo + (hi - lo) / 2;           // no overflow, unlike (lo + hi) / 2
            if (nums[mid] == target) return mid;
            if (nums[mid] < target) lo = mid + 1;
            else hi = mid - 1;
        }
        return -1;
    }
};
```
**Edge cases:** one element; target smaller than all or larger than all.

### 3.2 Search Insert Position (LeetCode 35, Easy)
**Problem (paraphrased):** in a sorted array of distinct values, return the target's index, or the index where it would be inserted to keep the order.
**Approach:** a linear scan for the first element `>= target` is O(n). That is exactly `lower_bound`, so use the boundary template over `[0, n]`.
**Complexity:** O(log n) time, O(1) space.
```cpp
class Solution {
public:
    int searchInsert(vector<int>& nums, int target) {
        int lo = 0, hi = nums.size();              // the answer can be n (insert at the end)
        while (lo < hi) {
            int mid = lo + (hi - lo) / 2;
            if (nums[mid] >= target) hi = mid;      // mid could be the insert position
            else lo = mid + 1;
        }
        return lo;   // same as lower_bound(nums.begin(), nums.end(), target) - nums.begin()
    }
};
```
**Edge cases:** insert at index 0; insert at index n.

### 3.3 Find First and Last Position of Element in Sorted Array (LeetCode 34, Medium)
**Problem (paraphrased):** in a sorted array with duplicates, return the first and last index of a target, or `[-1, -1]`, in O(log n).
**Approach:** finding one occurrence and expanding outward is O(n) when the target repeats many times. Run two boundary searches instead: the first index `>= target`, and the first index `>= target + 1` minus one.
**Complexity:** O(log n) time, O(1) space.
```cpp
class Solution {
public:
    vector<int> searchRange(vector<int>& nums, int target) {
        int first = firstAtLeast(nums, target);
        if (first == (int)nums.size() || nums[first] != target) return {-1, -1};
        int last = firstAtLeast(nums, (long long)target + 1) - 1;   // widen: target + 1 could overflow int
        return {first, last};
    }
private:
    int firstAtLeast(const vector<int>& nums, long long x) {
        int lo = 0, hi = nums.size();
        while (lo < hi) {
            int mid = lo + (hi - lo) / 2;
            if (nums[mid] >= x) hi = mid;
            else lo = mid + 1;
        }
        return lo;
    }
};
```
**Edge cases:** empty array; target absent; every element equal to the target. With the STL: `lower_bound` and `upper_bound`, or `equal_range`.

### 3.4 Search in Rotated Sorted Array (LeetCode 33, Medium)
**Problem (paraphrased):** a sorted array of distinct values was rotated at an unknown pivot (for example `[4, 5, 6, 7, 0, 1, 2]`). Find a target's index in O(log n), or return -1.
**Approach:** a linear scan is O(n); finding the rotation point first and then searching one side is two binary searches. In one pass: at any `mid`, at least one half is sorted. Check whether the target lies inside the sorted half's value range; if so search there, otherwise search the other half.
**Complexity:** O(log n) time, O(1) space.
```cpp
class Solution {
public:
    int search(vector<int>& nums, int target) {
        int lo = 0, hi = (int)nums.size() - 1;
        while (lo <= hi) {
            int mid = lo + (hi - lo) / 2;
            if (nums[mid] == target) return mid;
            if (nums[lo] <= nums[mid]) {                          // left half [lo, mid] is sorted
                if (nums[lo] <= target && target < nums[mid]) hi = mid - 1;
                else lo = mid + 1;
            } else {                                              // right half [mid, hi] is sorted
                if (nums[mid] < target && target <= nums[hi]) lo = mid + 1;
                else hi = mid - 1;
            }
        }
        return -1;
    }
};
```
**Edge cases:** no rotation; one or two elements (`<=` in `nums[lo] <= nums[mid]` handles `lo == mid`). With duplicates (LeetCode 81) the worst case becomes O(n).

### 3.5 Find Minimum in Rotated Sorted Array (LeetCode 153, Medium)
**Problem (paraphrased):** return the smallest value of a rotated sorted array of distinct values in O(log n).
**Approach:** a linear scan is O(n). Compare `nums[mid]` with `nums[hi]`: if it is larger, the drop (and the minimum) is strictly right of `mid`; otherwise `mid` itself could be the minimum, so keep it.
**Complexity:** O(log n) time, O(1) space.
```cpp
class Solution {
public:
    int findMin(vector<int>& nums) {
        int lo = 0, hi = (int)nums.size() - 1;
        while (lo < hi) {
            int mid = lo + (hi - lo) / 2;
            if (nums[mid] > nums[hi]) lo = mid + 1;   // minimum is right of mid
            else hi = mid;                            // mid may be the minimum
        }
        return nums[lo];
    }
};
```
**Edge cases:** not rotated (returns `nums[0]`); one element; rotation by one position.

### 3.6 Search a 2D Matrix (LeetCode 74, Medium)
**Problem (paraphrased):** every row of an m x n matrix is sorted, and each row starts with a value larger than the last value of the previous row. Decide whether a target exists.
**Approach:** scanning all cells is O(m * n). The rows laid end to end form one sorted array of length m * n, so binary search a virtual index and convert it with `row = mid / n`, `col = mid % n`.
**Complexity:** O(log(m * n)) time, O(1) space.
```cpp
class Solution {
public:
    bool searchMatrix(vector<vector<int>>& matrix, int target) {
        int m = matrix.size(), n = matrix[0].size();
        int lo = 0, hi = m * n - 1;
        while (lo <= hi) {
            int mid = lo + (hi - lo) / 2;
            int value = matrix[mid / n][mid % n];   // virtual 1D index -> (row, column)
            if (value == target) return true;
            if (value < target) lo = mid + 1;
            else hi = mid - 1;
        }
        return false;
    }
};
```
**Edge cases:** a single row or column. If only rows and columns are sorted separately (LeetCode 240), start at the top-right corner and step left or down, O(m + n).

### 3.7 Koko Eating Bananas (LeetCode 875, Medium)
**Problem (paraphrased):** piles of bananas must be finished within h hours. Each hour Koko picks one pile and eats up to k bananas from it. Find the smallest integer speed k that finishes in time.
**Approach:** trying every speed from 1 upward is O(max * n). Feasibility is monotone (a faster speed never takes longer), so binary search k over `[1, max pile]`. Hours at speed k are the sum of `ceil(pile / k)`, which can exceed 32 bits at small k, so accumulate in `long long`.
**Complexity:** O(n log M) time for the largest pile M, O(1) space.
```cpp
class Solution {
public:
    int minEatingSpeed(vector<int>& piles, int h) {
        int lo = 1, hi = *max_element(piles.begin(), piles.end());   // speed = max pile always works
        while (lo < hi) {
            int k = lo + (hi - lo) / 2;
            if (hoursNeeded(piles, k) <= h) hi = k;    // fast enough: try slower
            else lo = k + 1;                           // too slow
        }
        return lo;
    }
private:
    long long hoursNeeded(const vector<int>& piles, int k) {
        long long hours = 0;
        for (int p : piles) hours += (p + (long long)k - 1) / k;     // ceil(p / k) in integers
        return hours;
    }
};
```
**Edge cases:** h equal to the number of piles (answer is the largest pile); one pile; huge piles (overflow without `long long`).

### 3.8 Sort Colors (LeetCode 75, Medium)
**Problem (paraphrased):** sort an array that contains only 0s, 1s and 2s in place, in a single pass, without a library sort.
**Approach:** `std::sort` is O(n log n); counting the three values and rewriting the array takes two passes. The **Dutch national flag** algorithm does one pass with three regions: `[0, low)` holds 0s, `[low, mid)` holds 1s, `(high, n - 1]` holds 2s, and `[mid, high]` is unprocessed. This is the same three-way partition that keeps quick sort fast with duplicates.
**Complexity:** O(n) time, O(1) space.
```cpp
class Solution {
public:
    void sortColors(vector<int>& nums) {
        int low = 0, mid = 0, high = (int)nums.size() - 1;
        while (mid <= high) {
            if (nums[mid] == 0) swap(nums[low++], nums[mid++]);
            else if (nums[mid] == 1) mid++;
            else swap(nums[mid], nums[high--]);   // do not advance mid: the swapped-in value is unchecked
        }
    }
};
```
**Edge cases:** all one color; already sorted; reverse sorted.

### 3.9 Kth Largest Element in an Array (LeetCode 215, Medium)
**Problem (paraphrased):** return the k-th largest value in an unsorted array (by sorted position, so duplicates count separately).
**Approach:** sorting is O(n log n). A min-heap of size k is O(n log k) (see [Heaps](08-heaps.md)). **Quickselect** partitions like quick sort but recurses into one side only, O(n) on average: the k-th largest is at index `n - k` in ascending order. A three-way partition keeps it fast when many values are equal, and a random pivot makes the O(n^2) worst case unlikely.
**Complexity:** O(n) average time (O(n^2) worst), O(1) extra space.
```cpp
class Solution {
public:
    int findKthLargest(vector<int>& nums, int k) {
        int target = nums.size() - k;                 // index of the answer in ascending order
        int lo = 0, hi = (int)nums.size() - 1;
        mt19937 rng(random_device{}());
        while (true) {
            int pivot = nums[uniform_int_distribution<int>(lo, hi)(rng)];
            int lt = lo, i = lo, gt = hi;             // partition into < pivot | == pivot | > pivot
            while (i <= gt) {
                if (nums[i] < pivot) swap(nums[lt++], nums[i++]);
                else if (nums[i] > pivot) swap(nums[i], nums[gt--]);
                else i++;
            }
            if (target < lt) hi = lt - 1;             // answer is among the smaller values
            else if (target > gt) lo = gt + 1;        // answer is among the larger values
            else return pivot;                        // target falls inside the "equal" block
        }
    }
};
```
**Edge cases:** k = 1 (maximum); k = n (minimum); all elements equal. The STL shortcut is `nth_element(nums.begin(), nums.begin() + (nums.size() - k), nums.end())`, average O(n).

### 3.10 Median of Two Sorted Arrays (LeetCode 4, Hard)
**Problem (paraphrased):** return the median of the combined values of two sorted arrays in O(log(m + n)).
**Approach:** merging the arrays (or walking two pointers to the middle) is O(m + n). Instead, binary search how many elements the left half takes from the smaller array, i; the other array contributes `j = half - i`. The split is correct when everything on the left is `<=` everything on the right: `aLeft <= bRight` and `bLeft <= aRight`. If `aLeft > bRight`, too many came from `a`; otherwise too few.
**Complexity:** O(log(min(m, n))) time, O(1) space.
```cpp
class Solution {
public:
    double findMedianSortedArrays(vector<int>& a, vector<int>& b) {
        if (a.size() > b.size()) return findMedianSortedArrays(b, a);   // search the smaller array
        int m = a.size(), n = b.size();
        int half = (m + n + 1) / 2;                  // size of the left half (gets the extra one if odd)
        int lo = 0, hi = m;                          // how many elements the left half takes from a
        while (lo <= hi) {
            int i = lo + (hi - lo) / 2, j = half - i;
            int aLeft  = (i == 0) ? INT_MIN : a[i - 1];
            int aRight = (i == m) ? INT_MAX : a[i];
            int bLeft  = (j == 0) ? INT_MIN : b[j - 1];
            int bRight = (j == n) ? INT_MAX : b[j];
            if (aLeft <= bRight && bLeft <= aRight) {                   // valid split
                if ((m + n) % 2 == 1) return max(aLeft, bLeft);
                return (max(aLeft, bLeft) + (double)min(aRight, bRight)) / 2.0;   // double avoids overflow
            }
            if (aLeft > bRight) hi = i - 1;          // took too many from a
            else lo = i + 1;                         // took too few from a
        }
        return 0.0;                                  // unreachable for valid input
    }
};
```
**Edge cases:** one array empty; all of one array smaller than the other; odd versus even total length.

## Common pitfalls
- `(lo + hi) / 2` overflowing `int` for large indices or value ranges; use `lo + (hi - lo) / 2`.
- Mixing templates: `while (lo <= hi)` with `hi = mid` loops forever; `while (lo < hi)` with `hi = mid - 1` can skip the answer. Pick one template and keep its invariant.
- Using `lo = mid` with a lower middle, which never shrinks a two-element range.
- Binary searching data that is not sorted, or a predicate that is not monotone.
- Comparators that use `<=` or are inconsistent (for example comparing floating point values that can be NaN): undefined behavior in `std::sort`.
- Assuming `std::sort` is stable; use `stable_sort` when equal keys must keep their order.
- Calling `std::lower_bound` on `set`/`map` iterators (O(n)) instead of the member function (O(log n)).
- Forgetting `long long` when a feasibility check sums large values (Koko, ship capacity).

## How to talk about this in an interview
- State the binary search invariant before coding: "The answer is always in `[lo, hi]`; everything before `lo` fails the check." Then each update line has an obvious justification, which is the habit you would push in a Data Structures lab when students fight off-by-one errors.
- For sorting questions, compare on four axes: time (average and worst), extra memory, stability, and practical speed (cache behavior). Mention that real libraries use hybrids: introsort in C++, Timsort-style merging in Python.
- Connect to databases: a B-tree index is a disk-friendly multiway search tree, so an indexed lookup is a few page reads instead of a full table scan, and `ORDER BY` without a unique tiebreaker gives unstable pagination. See [indexing](../12-databases/05-indexing-and-query-optimization.md).
- > **Fill in:** a sorted or paginated listing you built (for example inventory or applications) and how you made the order deterministic.

## Interview questions

**Q1. How does quick sort work, and what is its worst case?**
Pick a pivot, partition the range into elements smaller and larger than it (the pivot lands in its final position), and recurse on both sides. Average O(n log n), in place with O(log n) expected stack. The worst case is O(n^2), when every pivot is the smallest or largest element, for example choosing the last element on already sorted input. Random pivots, median-of-three, three-way partitioning for duplicates, and introsort's heap sort fallback avoid it.
*Follow-up:* How do you bound the stack? Recurse into the smaller partition and loop on the larger one: O(log n) stack even in the worst case.

**Q2. Merge sort versus quick sort: which do you choose?**
Merge sort guarantees O(n log n), is stable, works well on linked lists and for external sorting, but needs O(n) extra memory for arrays. Quick sort is in place and usually faster in practice thanks to cache locality, but is not stable and has an O(n^2) worst case. For arrays in memory without a stability requirement, quick sort (introsort); for stability, linked lists or data on disk, merge sort.
*Follow-up:* What does `std::stable_sort` use? A merge sort that falls back to an O(n log^2 n) in-place merge when it cannot allocate a buffer.

**Q3. What is a stable sort, and when does it matter?**
A stable sort keeps elements with equal keys in their original relative order. It matters when you sort by several keys in passes (sort by date, then stable-sort by customer) or when the existing order carries meaning, like insertion time. Merge, insertion and counting sort are stable; quick, heap and selection sort are not.
*Follow-up:* How do you make an unstable sort stable? Add the original index as a final tiebreaker in the comparator.

**Q4. What conditions must hold for binary search to work?**
Random access to the search space and a monotone predicate over it: false for a prefix and true for the rest (sorted data gives this for comparisons against a target). It also applies to an implicit space of answers when feasibility is monotone. Without monotonicity, discarding half of the space is not safe.
*Follow-up:* Can you binary search a linked list? Not efficiently: reaching the middle costs O(n), which defeats the purpose.

**Q5. What does `std::sort` use internally, and what is a strict weak ordering?**
Common implementations use introsort: quick sort that switches to heap sort when recursion gets too deep, and insertion sort for small ranges, giving O(n log n) worst case. A strict weak ordering is a "less than" relation that is irreflexive, transitive, and whose equivalence is transitive; `<` qualifies, `<=` does not.
*Follow-up:* What happens with a broken comparator? Undefined behavior: wrong results, crashes, or reads past the end of the array.

**Q6. Explain `lower_bound` versus `upper_bound`.**
On a sorted range, `lower_bound(x)` returns the first position whose element is `>= x` and `upper_bound(x)` the first whose element is `> x`. Their difference is the number of copies of x, and `lower_bound` is the insertion point that keeps the range sorted.
*Follow-up:* How do you find the last element `<= x`? `upper_bound(x)` minus one, after checking it is not `begin()`.

**Q7. How would you sort 10 GB of data with 1 GB of RAM?**
External merge sort: read about 1 GB at a time, sort it in memory and write it out as a sorted run; then k-way merge the roughly ten runs with a min-heap holding the current head of each run, streaming the output to disk. Total I/O is about two passes over the data.
*Follow-up:* Which structure does the merge step use? A min-heap of size k, O(log k) per output element.

**Q8. When would you use counting sort or radix sort instead of a comparison sort?**
When keys are integers (or fixed-length strings) from a small range: counting sort is O(n + k), and radix sort is O(d * (n + base)) for d-digit keys. They beat O(n log n) when k or d is small, at the cost of extra memory and only working on those key types.
*Follow-up:* Why can they beat the O(n log n) lower bound? That bound only applies to comparison-based sorting; these sorts index by key value instead of comparing.

**Q9. How do you recognize a "binary search on the answer" problem?**
The question asks for the minimum or maximum value of something (speed, capacity, days, distance), checking a candidate is easy, and feasibility is monotone in that value. Search the value range with the boundary template, calling the check at each step.
*Follow-up:* Give the complexity of Koko Eating Bananas. O(n log M), with M the largest pile.
