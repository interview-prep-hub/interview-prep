# Heaps and Priority Queues

> **Tier:** T1 · **Related:** [Trees and BSTs](07-trees-bst.md), [Graphs](09-graphs.md), [Greedy, intervals and bits](11-greedy-intervals-bits.md), [Sorting and binary search](06-sorting-binary-search.md), [STL for DSA](../03-languages/c-cpp/04-stl-for-dsa.md) · **Language:** C++17

## Quick revision
- A binary heap is a **complete** binary tree with the **heap property**: in a min-heap every parent is <= its children, so the minimum is at the root. Siblings are not ordered.
- Stored in an array with no pointers: for index i, parent `(i - 1) / 2`, children `2i + 1` and `2i + 2`.
- push: append, then **sift up**, O(log n). pop: move the last element to the root, then **sift down**, O(log n). top: O(1).
- Building a heap from n elements bottom-up is **O(n)**, not O(n log n).
- `std::priority_queue<T>` is a **max-heap** by default. Min-heap: `priority_queue<int, vector<int>, greater<int>>`.
- The comparator means "a has lower priority than b" (a sits below b), which is why `greater` produces a min-heap.
- In C++17 a lambda comparator must be passed to the constructor: `priority_queue<T, vector<T>, decltype(cmp)> pq(cmp);`.
- `pair`s compare by `first`, then `second`: `(distance, node)` pairs make Dijkstra's queue.
- No decrease-key, no erase, no iteration: push a new entry and skip stale ones when popped (lazy deletion), or use `std::set`.
- **Top k largest:** min-heap capped at size k, O(n log k). **K smallest:** max-heap capped at k.
- **K-way merge:** min-heap of the current head of each list. **Running median:** two heaps.
- Signals: "k-th largest/smallest", "top k", "closest k", "merge k sorted", "median of a stream", "schedule by earliest/most", "always pick the best available".

## 1. Concept from scratch

### 1.1 Why a heap
A priority queue answers one question fast: "what is the most important item right now?", while items keep arriving. A sorted array answers it in O(1) but inserts cost O(n); an unsorted array inserts in O(1) but finding the best costs O(n). A heap sits in between: O(log n) for both insert and remove-best, O(1) to peek, with a compact array layout.

### 1.2 The heap property and the array layout
The tree is **complete**: every level is full except possibly the last, which fills from the left. That shape lets you store it in an array level by level, with no gaps and no pointers. With 0-based indexes, node i has children `2i + 1` and `2i + 2` and parent `(i - 1) / 2`. The **heap property** only constrains parent versus child, so a heap is "partially sorted": the root is the minimum (or maximum), but the rest is not in order.

### 1.3 Sift up, sift down, and the O(n) build
- **push:** place the new element at the end (the next free slot of the complete tree) and swap it with its parent while it is smaller (sift up). At most h = O(log n) swaps.
- **pop:** move the last element to the root, shrink the array, and swap the root with its smaller child while it is larger than that child (sift down). O(log n).
- **build:** call sift-down on every non-leaf node from the last one back to the root. Half the nodes are leaves and do nothing, a quarter move at most one level, an eighth at most two, and so on. The sum n/4 x 1 + n/8 x 2 + n/16 x 3 + ... is O(n). Pushing elements one at a time would be O(n log n).

### 1.4 std::priority_queue
```cpp
#include <functional>
#include <queue>
#include <vector>
using namespace std;

void priorityQueueDemo() {
    priority_queue<int> maxHeap;                                // largest on top
    priority_queue<int, vector<int>, greater<int>> minHeap;     // smallest on top

    vector<int> v = {5, 1, 8, 3};
    priority_queue<int> fromRange(v.begin(), v.end());          // heapify in O(n)

    // (distance, node) pairs compare by distance first: the queue used by Dijkstra.
    priority_queue<pair<int, int>, vector<pair<int, int>>, greater<pair<int, int>>> pq;
    pq.push({4, 2});

    // Custom order: return true when a should sit BELOW b (a has lower priority).
    auto lowerPriority = [](const pair<int, int>& a, const pair<int, int>& b) {
        return a.second > b.second;                             // the smallest .second rises to the top
    };
    priority_queue<pair<int, int>, vector<pair<int, int>>, decltype(lowerPriority)> custom(lowerPriority);
    custom.push({7, 2});
}
```
The interface is deliberately small: `push`, `emplace`, `top`, `pop`, `size`, `empty`. There is no way to change an element's priority or remove an arbitrary element. When priorities change (Dijkstra relaxations), push a new entry and, when popping, skip entries that are stale. When you truly need arbitrary deletion, use a `std::set` (or `multiset`) as an ordered priority queue with O(log n) erase. The `<algorithm>` functions `make_heap`, `push_heap`, `pop_heap` and `sort_heap` give the same operations on a plain `vector` you control.

### 1.5 Heap versus BST versus sorted array
| Operation | Binary heap | Balanced BST (`std::set`) | Sorted array |
|---|---|---|---|
| Peek min (or max) | O(1) for one end | O(1) via `begin()` / `rbegin()` | O(1) |
| Insert | O(log n) | O(log n) | O(n) |
| Remove min | O(log n) | O(log n) | O(n) from the front |
| Search for any value | O(n) | O(log n) | O(log n) |
| Delete any value | O(n) to find it | O(log n) | O(n) |
| Sorted iteration | no | yes | yes |
| Build from n items | O(n) | O(n log n) | O(n log n) |
| Memory | one compact array | one node with pointers per element | one compact array |

### 1.6 The recurring heap patterns
- **Top k largest:** keep a min-heap of at most k items; when it grows past k, pop the smallest. The heap holds the k largest seen so far, and its top is the k-th largest. O(n log k) time, O(k) memory, and it works on streams.
- **K-way merge:** a min-heap holds the current front of each sorted list; pop the smallest, then push the next item from the same list. Used for merging sorted files in external sorting.
- **Two heaps:** a max-heap for the smaller half and a min-heap for the larger half give the running median.
- **Greedy scheduling:** always take the most frequent remaining task, or reuse the room that frees up earliest.
- **Graphs:** Dijkstra's shortest paths and Prim's minimum spanning tree (see [Graphs](09-graphs.md)).

## 2. Core template
```cpp
#include <stdexcept>
#include <utility>
#include <vector>
using namespace std;

class MinHeap {
    vector<int> a;

    void siftUp(int i) {
        while (i > 0) {
            int parent = (i - 1) / 2;
            if (a[parent] <= a[i]) break;          // heap property holds
            swap(a[parent], a[i]);
            i = parent;
        }
    }
    void siftDown(int i) {
        int n = a.size();
        while (true) {
            int smallest = i, l = 2 * i + 1, r = 2 * i + 2;
            if (l < n && a[l] < a[smallest]) smallest = l;
            if (r < n && a[r] < a[smallest]) smallest = r;
            if (smallest == i) break;              // both children are larger or absent
            swap(a[i], a[smallest]);
            i = smallest;
        }
    }

public:
    MinHeap() = default;
    explicit MinHeap(vector<int> values) : a(move(values)) {        // O(n) bottom-up build
        for (int i = (int)a.size() / 2 - 1; i >= 0; i--) siftDown(i);
    }
    void push(int x) { a.push_back(x); siftUp((int)a.size() - 1); }
    int top() const {
        if (a.empty()) throw out_of_range("top on empty heap");
        return a[0];
    }
    void pop() {
        if (a.empty()) throw out_of_range("pop on empty heap");
        a[0] = a.back();                           // last leaf replaces the root
        a.pop_back();
        if (!a.empty()) siftDown(0);
    }
    bool empty() const { return a.empty(); }
    int size() const { return (int)a.size(); }
};
```
Heap sort falls out of this: build a max-heap in O(n), then repeatedly swap the root with the last element of the shrinking heap and sift down, O(n log n) in place (see [Sorting](06-sorting-binary-search.md)).

## 3. Problems

### 3.1 Kth Largest Element in a Stream (LeetCode 703, Easy)
**Problem (paraphrased):** design a class built from an initial array and k; each time a number is added, return the k-th largest value seen so far.
**Approach:** keeping a sorted vector costs O(n) per insert; re-sorting costs O(n log n). A min-heap capped at k holds exactly the k largest values, and its top is the k-th largest.
**Complexity:** O(log k) per `add`, O(k) space.
```cpp
class KthLargest {
    int k;
    priority_queue<int, vector<int>, greater<int>> minHeap;   // the k largest values seen so far
public:
    KthLargest(int k, vector<int>& nums) : k(k) {
        for (int x : nums) add(x);
    }
    int add(int val) {
        minHeap.push(val);
        if ((int)minHeap.size() > k) minHeap.pop();          // drop the smallest of k + 1
        return minHeap.top();                                 // k-th largest overall
    }
};
```
**Edge cases:** the initial array has fewer than k elements (the problem guarantees k values exist whenever an answer is requested); duplicates.

### 3.2 Last Stone Weight (LeetCode 1046, Easy)
**Problem (paraphrased):** repeatedly smash the two heaviest stones; equal stones both vanish, otherwise the heavier one survives with the difference in weight. Return the last stone's weight, or 0.
**Approach:** re-sorting the list every round is O(n^2 log n). A max-heap gives the two heaviest in O(log n) per round.
**Complexity:** O(n log n) time, O(n) space.
```cpp
class Solution {
public:
    int lastStoneWeight(vector<int>& stones) {
        priority_queue<int> pq(stones.begin(), stones.end());   // O(n) heapify
        while (pq.size() > 1) {
            int y = pq.top(); pq.pop();                         // heaviest
            int x = pq.top(); pq.pop();                         // second heaviest
            if (y != x) pq.push(y - x);
        }
        return pq.empty() ? 0 : pq.top();
    }
};
```
**Edge cases:** one stone; everything cancels out (return 0).

### 3.3 K Closest Points to Origin (LeetCode 973, Medium)
**Problem (paraphrased):** return any k points with the smallest distance from the origin.
**Approach:** sorting all points by distance is O(n log n). A max-heap capped at k keeps the k closest seen so far (evict the farthest), O(n log k). Quickselect is O(n) on average. Compare squared distances: no `sqrt`, no floating point.
**Complexity:** O(n log k) time, O(k) space.
```cpp
class Solution {
public:
    vector<vector<int>> kClosest(vector<vector<int>>& points, int k) {
        priority_queue<pair<long long, int>> maxHeap;          // (squared distance, index); farthest on top
        for (int i = 0; i < (int)points.size(); i++) {
            long long x = points[i][0], y = points[i][1];
            maxHeap.push({x * x + y * y, i});
            if ((int)maxHeap.size() > k) maxHeap.pop();        // evict the farthest
        }
        vector<vector<int>> result;
        while (!maxHeap.empty()) {
            result.push_back(points[maxHeap.top().second]);
            maxHeap.pop();
        }
        return result;
    }
};
```
**Edge cases:** k equal to the number of points; ties in distance (any choice is accepted).

### 3.4 Top K Frequent Elements (LeetCode 347, Medium), heap version
**Problem (paraphrased):** return the k most frequent values in an array.
**Approach:** count with a hash map, then keep a min-heap of `(count, value)` capped at k, so the least frequent candidate is evicted first. The O(n) bucket-sort version is in [Arrays, strings and hashing](01-arrays-strings-hashing.md); the heap version also works when values stream in.
**Complexity:** O(n log k) time, O(n) space for the counts.
```cpp
class Solution {
public:
    vector<int> topKFrequent(vector<int>& nums, int k) {
        unordered_map<int, int> freq;
        for (int x : nums) freq[x]++;
        priority_queue<pair<int, int>, vector<pair<int, int>>, greater<pair<int, int>>> minHeap;   // (count, value)
        for (auto& [value, count] : freq) {
            minHeap.push({count, value});
            if ((int)minHeap.size() > k) minHeap.pop();       // drop the least frequent
        }
        vector<int> result;
        while (!minHeap.empty()) {
            result.push_back(minHeap.top().second);
            minHeap.pop();
        }
        return result;
    }
};
```
**Edge cases:** k equal to the number of distinct values; all values equal.

### 3.5 Task Scheduler (LeetCode 621, Medium)
**Problem (paraphrased):** tasks are letters; the CPU runs one task or idles per time unit, and identical tasks must be separated by at least n units. Return the minimum total time.
**Approach:** simulate greedily: at each tick run the available task with the most remaining copies (max-heap), and park it in a cooldown queue with the time it becomes available again. Running the most frequent task first keeps the long chains from forcing idles at the end. There is also an O(n) counting formula: `max(tasks.size(), (maxCount - 1) * (n + 1) + numberOfTasksWithMaxCount)`.
**Complexity:** O(T log 26) time where T is the answer (at most tasks x (n + 1)), O(1) extra space (26 letters).
```cpp
class Solution {
public:
    int leastInterval(vector<char>& tasks, int n) {
        int count[26] = {0};
        for (char t : tasks) count[t - 'A']++;
        priority_queue<int> ready;                         // remaining counts of runnable tasks
        for (int c : count) if (c > 0) ready.push(c);
        queue<pair<int, int>> cooling;                     // (remaining count, time it is runnable again)
        int time = 0;
        while (!ready.empty() || !cooling.empty()) {
            time++;
            if (!ready.empty()) {
                int left = ready.top() - 1;                // run the most frequent runnable task
                ready.pop();
                if (left > 0) cooling.push({left, time + n});
            }                                              // otherwise the CPU idles this tick
            if (!cooling.empty() && cooling.front().second == time) {
                ready.push(cooling.front().first);         // its cooldown has ended
                cooling.pop();
            }
        }
        return time;
    }
};
```
**Edge cases:** n = 0 (answer is the number of tasks); a single task type (heavy idling); many task types (no idling needed).

### 3.6 Reorganize String (LeetCode 767, Medium)
**Problem (paraphrased):** rearrange a lowercase string so no two adjacent characters are equal, or return an empty string if impossible.
**Approach:** trying permutations is factorial. Greedy with a max-heap of `(count, letter)`: take the two most frequent letters, append both, decrement and push them back. Because the two letters differ, no adjacent pair repeats. If one letter remains at the end with more than one copy, it is impossible (equivalently, any letter appearing more than `(n + 1) / 2` times).
**Complexity:** O(n log 26) = O(n) time, O(1) extra space.
```cpp
class Solution {
public:
    string reorganizeString(string s) {
        int count[26] = {0};
        for (char c : s) count[c - 'a']++;
        priority_queue<pair<int, char>> pq;                     // (remaining count, letter)
        for (int i = 0; i < 26; i++)
            if (count[i] > 0) pq.push({count[i], char('a' + i)});
        string result;
        while (pq.size() >= 2) {
            auto [c1, a] = pq.top(); pq.pop();                  // the two most frequent letters
            auto [c2, b] = pq.top(); pq.pop();
            result += a;
            result += b;
            if (--c1 > 0) pq.push({c1, a});
            if (--c2 > 0) pq.push({c2, b});
        }
        if (!pq.empty()) {
            if (pq.top().first > 1) return "";                  // the same letter would touch itself
            result += pq.top().second;
        }
        return result;
    }
};
```
**Edge cases:** `"aab"` gives `"aba"`; `"aaab"` is impossible; a single character.

### 3.7 Merge k Sorted Lists (LeetCode 23, Hard)
**Problem (paraphrased):** merge k sorted linked lists into one sorted list.
**Approach:** collecting all values and sorting is O(N log N) for N total nodes; merging lists one at a time is O(k * N). A min-heap of the k current heads pops the smallest node and pushes its successor, O(N log k). Pairwise divide-and-conquer merging has the same complexity.
**Complexity:** O(N log k) time, O(k) heap space.
```cpp
class Solution {
public:
    ListNode* mergeKLists(vector<ListNode*>& lists) {
        auto cmp = [](ListNode* a, ListNode* b) { return a->val > b->val; };   // min-heap by value
        priority_queue<ListNode*, vector<ListNode*>, decltype(cmp)> pq(cmp);
        for (ListNode* head : lists)
            if (head) pq.push(head);                           // skip empty lists
        ListNode dummy(0);
        ListNode* tail = &dummy;
        while (!pq.empty()) {
            ListNode* node = pq.top(); pq.pop();
            tail->next = node;
            tail = node;
            if (node->next) pq.push(node->next);               // the next candidate from that list
        }
        return dummy.next;
    }
};
```
**Edge cases:** an empty vector of lists; some lists empty; k = 1.

### 3.8 Find Median from Data Stream (LeetCode 295, Hard)
**Problem (paraphrased):** support adding numbers one at a time and asking for the median of everything added so far.
**Approach:** inserting into a sorted vector is O(n) per add. Two heaps: a max-heap `low` holds the smaller half and a min-heap `high` holds the larger half, with `low` allowed one extra element. Every new number passes through `low` into `high` (so the halves stay ordered), then sizes are rebalanced.
**Complexity:** O(log n) per add, O(1) per median, O(n) space.
```cpp
class MedianFinder {
    priority_queue<int> low;                                   // max-heap: smaller half
    priority_queue<int, vector<int>, greater<int>> high;       // min-heap: larger half
public:
    MedianFinder() {}
    void addNum(int num) {
        low.push(num);
        high.push(low.top());              // the largest of the small half moves across
        low.pop();
        if (high.size() > low.size()) {    // keep low equal in size or one bigger
            low.push(high.top());
            high.pop();
        }
    }
    double findMedian() {
        if (low.size() > high.size()) return low.top();
        return (low.top() + (double)high.top()) / 2.0;         // double: no int overflow
    }
};
```
**Edge cases:** one number; negative numbers; equal numbers. Follow-up: if all values are in 0..100, a counting array gives O(100) medians.

### 3.9 Minimum meeting rooms (classic interview problem, Medium)
**Problem (paraphrased):** given meetings as `[start, end)` time intervals, return the fewest rooms needed so that no two overlapping meetings share a room. A meeting that ends at time t frees its room for one starting at t.
**Approach:** brute force checks, at every start time, how many meetings are active, O(n^2). Sort by start time and keep a min-heap of end times for the rooms in use: if the earliest-ending room is free by the time the next meeting starts, reuse it (pop), then push the new meeting's end. The heap size is the number of rooms. A sweep over sorted start and end times also works.
**Complexity:** O(n log n) time, O(n) space.
```cpp
class Solution {
public:
    int minMeetingRooms(vector<vector<int>>& intervals) {
        sort(intervals.begin(), intervals.end());                    // by start time
        priority_queue<int, vector<int>, greater<int>> ends;         // end times of rooms in use
        for (auto& m : intervals) {
            if (!ends.empty() && ends.top() <= m[0]) ends.pop();     // the earliest room is free: reuse it
            ends.push(m[1]);
        }
        return ends.size();
    }
};
```
**Edge cases:** no meetings (0 rooms); back-to-back meetings (`[1, 5)` and `[5, 8)` share a room); all meetings overlapping (n rooms). Related: "can one person attend everything" is in [Greedy, intervals and bits](11-greedy-intervals-bits.md).

## Common pitfalls
- Forgetting that `priority_queue` is a max-heap by default and getting the k smallest instead of the k largest.
- Writing the comparator as if it meant "a comes first": it means "a has lower priority".
- Passing a lambda type without the lambda object in C++17 (`pq(cmp)` is required).
- Using a max-heap of all n elements for "top k" (O(n log n) and O(n) memory) instead of a size-k min-heap.
- Calling `top()` or `pop()` on an empty queue.
- Trying to update priorities in place; push a new entry and skip stale ones instead.
- Computing `sqrt` for distance comparisons (slower, floating-point error); compare squared distances in `long long`.
- Averaging two `int`s as `(a + b) / 2` (overflow and integer division) for a median.

## How to talk about this in an interview
- Say why a heap and why that direction: "I only ever need the current minimum, so a min-heap; I cap it at k so memory stays O(k) and each operation is O(log k)."
- Explain the array layout and sift operations on a small example; it is a classic Data Structures lab exercise, and drawing the tree next to the array shows you understand it beyond the STL.
- > **Fill in:** whether you implemented or graded a heap or priority queue exercise in your Data Structures labs, and what students found confusing.
- Real-world links: job schedulers run the next due job first, Sidekiq keeps scheduled and retry jobs in Redis sorted sets ordered by time, and dashboards often show "top N" lists, computed in SQL with `ORDER BY ... LIMIT N` or with a bounded heap in application code when the data streams in.

## Interview questions

**Q1. How is a priority queue implemented?**
Usually as a binary heap stored in an array: a complete binary tree where each parent is no larger (min-heap) or no smaller (max-heap) than its children. push appends and sifts up, pop moves the last element to the root and sifts down, both O(log n); peeking is O(1); building from n items is O(n). `std::priority_queue` wraps a `vector` with the heap algorithms.
*Follow-up:* Other implementations? A balanced BST or `std::set` (O(log n) with arbitrary delete), or d-ary and Fibonacci heaps for special cases.

**Q2. BST versus heap: what is the difference, and when do you use each?**
A BST orders everything (left < node < right), so it supports search, sorted iteration and range queries in O(log n) when balanced. A heap only guarantees that parents beat children, so it supports fast access to the single best item: O(1) peek, O(log n) push and pop, an O(n) build, and a compact array with no pointers. Use a heap when you only need the min or max; use a BST when you need lookups, ordered traversal or deletion of arbitrary items.
*Follow-up:* Can you search a heap efficiently? No, finding an arbitrary value is O(n).

**Q3. Why is building a heap O(n) and not O(n log n)?**
Bottom-up heapify calls sift-down from the last internal node back to the root. Most nodes are near the bottom and sift only a level or two: n/4 nodes move at most 1 level, n/8 at most 2, and so on. The sum is bounded by 2n, so the build is O(n). Pushing one element at a time costs O(n log n).
*Follow-up:* Does this make heap sort O(n)? No, the n pops still cost O(n log n).

**Q4. Why use a min-heap to find the k largest elements?**
The min-heap of size k keeps the k largest seen so far with the smallest of them on top, so each new element only has to beat that top to get in. That is O(n log k) time and O(k) memory, and it works on streams. A max-heap of everything costs O(n) memory and O(n + k log n) time.
*Follow-up:* When is quickselect better? For a one-off query on an in-memory array: O(n) average, but not streaming.

**Q5. How do you write a custom comparator for `std::priority_queue`, and why does it look backwards?**
The comparator is a "less than" for ordering: it returns true when a has lower priority than b, and the element that is greatest under it ends up on top. So `less` gives a max-heap and `greater` gives a min-heap; for a min-heap on a field, return `a.field > b.field`. In C++17, a lambda must be passed to the constructor along with `decltype(lambda)` as the template argument.
*Follow-up:* Simpler alternative? Push pairs or tuples and rely on lexicographic comparison, negating a key to flip its direction.

**Q6. How do you change the priority of an element that is already in the queue?**
`std::priority_queue` cannot. The common trick is lazy deletion: push the element again with its new priority and ignore outdated entries when they are popped (check against a `dist` array or a version number). Alternatively use a `std::set` of `(priority, id)` pairs: erase the old pair and insert the new one, O(log n) each.
*Follow-up:* What does lazy deletion cost? The heap can hold up to E entries, so operations are O(log E), which is O(log V) asymptotically for Dijkstra.

**Q7. How does heap sort work, and is it stable?**
Build a max-heap in place in O(n), then repeatedly swap the root (the maximum) with the last element of the heap region, shrink the region, and sift down: O(n log n) time with O(1) extra space. It is not stable, because the swaps move equal elements past each other.
*Follow-up:* Why is it slower than quick sort in practice? Sift-down jumps around the array, so it has poor cache locality.

**Q8. Where do heaps or priority queues appear in real systems?**
Timer and job schedulers (run the soonest deadline first), Dijkstra-based routing, event-driven simulations, merging sorted log files or database runs, top-N analytics over streams, and operating-system process schedulers that pick the highest-priority task.
*Follow-up:* What does Redis offer for this? Sorted sets (skip lists keyed by score), which tools such as Sidekiq use for scheduled and retry jobs.
