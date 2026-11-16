# STL for DSA

> **Tier:** T1 (know cold) · **Related:** [Big-O and problem solving](../../02-dsa/00-big-o-and-problem-solving.md), [Arrays, strings and hashing](../../02-dsa/01-arrays-strings-hashing.md), [Heaps](../../02-dsa/08-heaps.md), [Graphs](../../02-dsa/09-graphs.md), [C++ fundamentals](02-cpp-fundamentals.md)
> **On your resume:** C++ as your DSA language at PUCIT and in the Data Structures labs you assisted as a TA (2023 to 2025). Every coding round where you pick C++ leans on this file.

## Quick revision
- STL = containers + iterators + algorithms + functors. Algorithms take iterator ranges `[first, last)`, where `last` is one past the end.
- `vector`: O(1) indexing, amortized O(1) `push_back`, O(n) insert/erase in the middle. `reserve(n)` avoids reallocations.
- `deque`: O(1) at both ends plus indexing. `list`: O(1) insert/erase at a known position, no indexing.
- `stack` and `queue` (default container `deque`) and `priority_queue` (default `vector`) are adaptors. `priority_queue` is a **max**-heap; `greater<>` makes a min-heap.
- `set`/`map` (red-black tree): sorted, O(log n), `lower_bound`. `unordered_set`/`unordered_map` (hash table): O(1) average, O(n) worst, no order, no built-in hash for `pair`.
- `m[key]` inserts a default value when the key is missing; test membership with `count` or `find` (`contains` in C++20).
- `sort` is O(n log n) and not stable; `stable_sort` keeps equal elements in input order. Comparators must be strict: `<`, never `<=`.
- On sorted ranges: `lower_bound` = first element >= x, `upper_bound` = first element > x, `binary_search` = bool.
- `unique` removes only adjacent duplicates and does not shrink the container: `sort`, then `v.erase(unique(v.begin(), v.end()), v.end())`.
- `accumulate` returns the type of its initial value: pass `0LL` for big sums.
- `size()` is unsigned: `v.size() - 1` wraps to a huge number when `v` is empty.
- `INT_MAX` is about 2.1e9: use `long long` (and `1LL * a * b`) for products and large sums.
- Invalidation: `vector` reallocation invalidates every iterator and reference; node containers (`list`, `set`, `map`) invalidate only erased elements.
- Fast I/O: `ios::sync_with_stdio(false); cin.tie(nullptr);` and `'\n'` instead of `endl`.

## 1. Containers, iterators, algorithms, functors

**Containers** hold data, **iterators** are generalized pointers into them, **algorithms** work on iterator ranges without knowing the container, and **functors** (function objects, including lambdas) customize comparisons, predicates and hashes.

```cpp
#include <algorithm>
#include <cstdlib>
#include <functional>
#include <iostream>
#include <vector>
using namespace std;

struct ByAbs {                                       // a functor: a class with operator()
    bool operator()(int a, int b) const { return abs(a) < abs(b); }
};

int main() {
    vector<int> v = {5, -1, 4, -2, 3};               // container
    sort(v.begin(), v.end(), greater<int>());        // algorithm + standard functor: 5 4 3 -1 -2
    sort(v.begin(), v.end(), ByAbs());               // custom functor: -1 -2 3 4 5
    for (auto it = v.begin(); it != v.end(); ++it)   // iterator
        cout << *it << ' ';
    cout << '\n';
}
```

Include the specific headers you use. `#include <bits/stdc++.h>` pulls in the whole library, but it is a GCC-only internal header (MSVC does not have it) and slows compilation: fine in contests, avoid it in real code.

| Iterator category | Containers | What works |
|---|---|---|
| Random access | `vector`, `deque`, `array`, `string` | `it + k`, `it2 - it1`, `sort`, O(log n) binary search |
| Bidirectional | `list`, `set`, `map` and their multi versions | `++`, `--`, `next`, `prev`; `list` has its own `sort()` |
| Forward | `unordered_*`, `forward_list` | `++` only |

`end()` points one past the last element: never dereference it. `rbegin()`/`rend()` walk backwards.

## 2. Complexity reference

| Container | Structure | Access | Insert / erase | Notes |
|---|---|---|---|---|
| `vector` | dynamic array | O(1) `v[i]` | end: amortized O(1); middle: O(n) | contiguous, cache-friendly: the default |
| `array<T, N>` | fixed array | O(1) | not resizable | size known at compile time |
| `string` | dynamic char array | O(1) | `+=` a char: amortized O(1); `insert`/`erase`: O(n) | `substr(pos, len)` is O(len); `find` worst case O(n*m) |
| `deque` | chunks of arrays | O(1) | both ends O(1); middle O(n) | not contiguous |
| `list` | doubly linked list | O(n) | O(1) at an iterator and at both ends | stable iterators, `splice`; rarely fastest in practice |

| Container | Structure | Operations | Notes |
|---|---|---|---|
| `stack` | adaptor (default `deque`) | `push`, `pop`, `top`: O(1) | LIFO, no iteration |
| `queue` | adaptor (default `deque`) | `push`, `pop`, `front`, `back`: O(1) | FIFO: BFS |
| `priority_queue` | binary heap in a `vector` | `push`/`pop` O(log n), `top` O(1) | max-heap by default; building from a range is O(n) |
| `set` / `multiset` | red-black tree | `insert`, `erase`, `find`, `count`, `lower_bound`: O(log n) | sorted; multiset keeps duplicates (`count` is O(log n + k)) |
| `map` / `multimap` | red-black tree | same operations by key: O(log n) | sorted by key; `m[k]` inserts if missing |
| `unordered_set` / `unordered_map` | hash table | O(1) average, O(n) worst | no order; a rehash costs O(n), amortized away |
| `pair` / `tuple` | plain struct | O(1) member access | compared lexicographically |

## 3. vector in depth

```cpp
vector<int> a;                                     // empty
vector<int> b(5);                                  // five zeros
vector<int> c(5, -1);                              // five -1s
vector<int> d = {3, 1, 2};                         // initializer list
vector<vector<int>> grid(3, vector<int>(4, 0));    // 3 rows x 4 columns of zeros

a.push_back(4);                  // amortized O(1)
a.emplace_back(5);               // constructs in place (handy for pairs and objects)
a.pop_back();                    // O(1); undefined on an empty vector
a.reserve(1000);                 // capacity >= 1000, size unchanged
a.resize(10);                    // size becomes 10, new elements are 0
d.insert(d.begin() + 1, 9);      // O(n) shift: {3, 9, 1, 2}
d.erase(d.begin());              // O(n) shift: {9, 1, 2}
d.at(7);                         // bounds-checked: throws std::out_of_range (d[7] is UB)
```

**Why `push_back` is amortized O(1):** when `size() == capacity()`, the vector allocates a bigger buffer (about 2x in GCC's library, 1.5x in MSVC's; the factor is implementation-defined), moves or copies every element, and frees the old buffer. That one push costs O(n). Because capacity grows geometrically, n pushes copy about 1 + 2 + 4 + ... + n < 2n elements in total, so the average per push is O(1).

```cpp
vector<int> v;
for (int i = 0; i < 10; ++i) {
    v.push_back(i);
    cout << v.size() << '/' << v.capacity() << ' ';
}
// GCC prints: 1/1 2/2 3/4 4/4 5/8 6/8 7/8 8/8 9/16 10/16
```

- Reallocation invalidates all iterators, pointers and references into the vector. Call `reserve` up front when you know the final size.
- `clear()` keeps the capacity; `shrink_to_fit()` is only a request to release it.
- `vector<bool>` is a packed bitset whose elements are proxies (`auto& x = vb[0];` does not compile). Use `vector<char>` or `bitset` when you need real references.

## 4. deque, list and the adaptors

```cpp
deque<int> dq;
dq.push_back(1); dq.push_front(0); dq.pop_back();   // monotonic-deque sliding-window problems

list<int> ls = {1, 2, 3};
auto it = next(ls.begin());       // iterator to 2
ls.erase(it);                     // O(1); iterators to 1 and 3 stay valid

stack<int> st;  st.push(1);  int top = st.top();    st.pop();   // pop() returns void
queue<int> q;   q.push(1);   int front = q.front(); q.pop();

priority_queue<int> maxHeap;                               // largest on top
priority_queue<int, vector<int>, greater<int>> minHeap;    // smallest on top
array<int, 26> freq{};                                     // fixed size, zero-initialized
```

- Adaptor `pop()` returns `void`, so read `top()` or `front()` first. Calling either on an empty adaptor is undefined: check `empty()`.
- A `priority_queue` comparator answers "does `a` have lower priority than `b`?". So `less` (the default) yields a max-heap and `greater` a min-heap. A lambda comparator needs `decltype` and must be passed to the constructor:

```cpp
auto byDist = [](const pair<int, int>& a, const pair<int, int>& b) { return a.second > b.second; };
priority_queue<pair<int, int>, vector<pair<int, int>>, decltype(byDist)> pq(byDist);   // min by .second
```

## 5. Ordered containers: set, multiset, map, multimap

```cpp
set<int> s = {5, 1, 3};
s.insert(3);                              // ignored: already present
s.count(3);                               // 1 (always 0 or 1 for a set)
s.erase(1);                               // erase by value
auto it = s.lower_bound(4);               // first element >= 4: points to 5
if (it != s.end()) cout << *it;
int smallest = *s.begin(), largest = *s.rbegin();   // O(1)

multiset<int> ms = {2, 2, 3};
ms.erase(ms.find(2));                     // removes ONE 2
ms.erase(2);                              // removes ALL remaining 2s

map<string, int> freq;
freq["apple"]++;                          // inserts {"apple", 0}, then increments
for (const auto& [word, n] : freq)        // iterates in sorted key order
    cout << word << ' ' << n << '\n';
auto pos = freq.find("kiwi");             // freq.end() if missing; nothing is inserted
```

- Use the member `s.lower_bound(x)`, never `std::lower_bound(s.begin(), s.end(), x)`: with set iterators the free function needs O(n) steps.
- Typical uses: sorted unique values, "smallest element >= x" queries, sliding-window order statistics (`multiset`), counting events by time (`map<int, int>`), coordinate compression.
- Custom order: `set<int, greater<int>>`. Struct keys need `operator<` or a comparator.

## 6. Hash containers: unordered_set and unordered_map

```cpp
vector<int> twoSum(const vector<int>& nums, int target) {
    unordered_map<int, int> seen;            // value -> index
    seen.reserve(nums.size());               // pre-size the buckets to avoid rehashing
    for (int i = 0; i < (int)nums.size(); ++i) {
        auto it = seen.find(target - nums[i]);
        if (it != seen.end()) return {it->second, i};
        seen[nums[i]] = i;
    }
    return {};
}
```

| | `map` | `unordered_map` |
|---|---|---|
| Structure | red-black tree | hash table (buckets) |
| Order | sorted by key | none |
| `find` / `insert` / `erase` | O(log n) guaranteed | O(1) average, O(n) worst |
| Key needs | `operator<` (strict weak ordering) | a `std::hash` specialization and `==` |
| Extras | `lower_bound`, range queries, min/max | usually faster lookups |
| Memory | one node per element | buckets plus nodes, usually more |

- O(1) average assumes the hash spreads keys across buckets. Many keys in one bucket (adversarial input, a weak custom hash) make lookups O(n). When the load factor passes `max_load_factor()` (1.0 by default) the table rehashes: O(n) once, amortized O(1).
- Contest sites sometimes have inputs crafted against the default integer hash; on LeetCode-style platforms this is rarely a concern. If in doubt, use `map` or a stronger hash.

There is no standard hash for `pair`, so `unordered_set<pair<int, int>>` does not compile until you supply one:

```cpp
#include <cstdint>

struct PairHash {
    size_t operator()(const pair<int, int>& p) const {
        uint64_t key = (uint64_t(uint32_t(p.first)) << 32) | uint32_t(p.second);   // pack into 64 bits
        return hash<uint64_t>()(key);
    }
};

unordered_set<pair<int, int>, PairHash> visited;
visited.insert({2, 3});
```

Simpler alternatives: encode a grid cell as `r * cols + c` in an `unordered_set<int>`, use `set<pair<int, int>>` (works out of the box at O(log n)), or, fastest of all, a `vector<vector<bool>>` visited grid.

## 7. pair and tuple

```cpp
pair<int, string> p = {3, "c"};
auto q = make_pair(1, 2.5);                     // pair<int, double>
vector<pair<int, int>> iv = {{1, 3}, {0, 2}, {1, 2}};
sort(iv.begin(), iv.end());                     // by first, then second: {0,2} {1,2} {1,3}
tuple<int, int, int> t{1, 2, 3};
auto [x, y, z] = t;                             // structured bindings (C++17)
int first = get<0>(t);                          // 1
```

Lexicographic comparison makes pairs ideal for "sort by A, then by B": sorting `{-count, word}` pairs orders by count descending, then word ascending.

## 8. Algorithms

```cpp
#include <algorithm>
#include <numeric>

vector<int> v = {4, 1, 3, 1, 5};
sort(v.begin(), v.end());                                        // 1 1 3 4 5
sort(v.rbegin(), v.rend());                                      // descending: 5 4 3 1 1
sort(v.begin(), v.end(), [](int a, int b) { return a > b; });    // descending with a lambda

vector<string> words = {"pear", "fig", "apple", "kiwi"};
sort(words.begin(), words.end(), [](const string& a, const string& b) {
    if (a.size() != b.size()) return a.size() < b.size();        // shorter first
    return a < b;                                                 // ties alphabetically
});                                                               // fig kiwi pear apple
stable_sort(words.begin(), words.end(),
            [](const string& a, const string& b) { return a.size() < b.size(); });   // ties keep their order

vector<int> s = {1, 3, 3, 3, 7};                     // must already be sorted
bool has = binary_search(s.begin(), s.end(), 3);     // true
auto lo = lower_bound(s.begin(), s.end(), 3);        // first >= 3: index 1
auto hi = upper_bound(s.begin(), s.end(), 3);        // first > 3: index 4
int idx = lo - s.begin();                            // 1
int cnt = hi - lo;                                   // 3 copies of 3
auto ins = lower_bound(s.begin(), s.end(), 5);       // where 5 would go: index 4 (points to 7)

reverse(v.begin(), v.end());
int mx = *max_element(v.begin(), v.end());           // returns an iterator: dereference (never on an empty range)
int mnIdx = min_element(v.begin(), v.end()) - v.begin();
long long sum = accumulate(v.begin(), v.end(), 0LL);  // 0LL makes the accumulator a long long
int ones = count(v.begin(), v.end(), 1);
bool found = find(v.begin(), v.end(), 3) != v.end();
sort(v.begin(), v.end());
v.erase(unique(v.begin(), v.end()), v.end());        // dedupe: unique returns the new logical end
fill(v.begin(), v.end(), 0);

vector<int> ids(5);
iota(ids.begin(), ids.end(), 0);                     // 0 1 2 3 4
vector<int> pre(ids.size());
partial_sum(ids.begin(), ids.end(), pre.begin());    // prefix sums: 0 1 3 6 10
string perm = "abc";                                 // start sorted to visit every permutation
do { cout << perm << ' '; } while (next_permutation(perm.begin(), perm.end()));   // abc acb bac bca cab cba
```

| Algorithm | Time | Notes |
|---|---|---|
| `sort` | O(n log n) | introsort, not stable, random-access iterators only |
| `stable_sort` | O(n log n), or O(n log^2 n) without spare memory | merge sort; equal elements keep their order |
| `binary_search`, `lower_bound`, `upper_bound`, `equal_range` | O(log n) | the range must be sorted with the same comparator |
| `min_element`, `max_element`, `count(_if)`, `find(_if)`, `accumulate`, `fill`, `iota`, `partial_sum`, `reverse` | O(n) | `accumulate`, `iota`, `partial_sum` live in `<numeric>` |
| `unique` | O(n) | adjacent duplicates only; pair it with `erase` |
| `next_permutation` | O(n) per call | returns false after the last permutation (and resets to sorted) |
| `nth_element` | O(n) average | puts the k-th smallest in position k (quickselect) |

Also handy: `max({a, b, c})`, `swap(a, b)`, and `gcd`/`lcm` (C++17, `<numeric>`).

## 9. Iterator invalidation

| Container | Insert | Erase |
|---|---|---|
| `vector`, `string` | with reallocation: everything; otherwise: at and after the insertion point | at and after the erased element |
| `deque` | middle: everything; at either end: iterators invalid, references stay valid | middle: everything; at either end: only the erased element (plus `end()` when the last element goes) |
| `list`, `set`, `map` (node-based) | nothing | only the erased element |
| `unordered_*` | on rehash: all iterators (references stay valid); otherwise nothing | only the erased element |

```cpp
for (auto it = m.begin(); it != m.end(); ) {     // erase while iterating a map or set
    if (it->second == 0) it = m.erase(it);      // erase returns the next valid iterator
    else ++it;
}

// erase-remove idiom for vectors: one O(n) pass instead of repeated O(n) erases
v.erase(remove_if(v.begin(), v.end(), [](int x) { return x % 2 == 0; }), v.end());   // C++20: erase_if(v, pred)
```

## 10. DSA idioms

**Frequency counting**

```cpp
unordered_map<char, int> freq;
for (char ch : s) freq[ch]++;            // missing keys start at 0
array<int, 26> cnt{};                    // faster when input is lowercase letters only
for (char ch : s) cnt[ch - 'a']++;
```

**Adjacency list, visited array and BFS**

```cpp
vector<vector<int>> adj(n);
for (const auto& e : edges) {            // edges given as {u, v} pairs
    adj[e[0]].push_back(e[1]);
    adj[e[1]].push_back(e[0]);           // undirected: add both directions
}
vector<bool> visited(n, false);
queue<int> q;
q.push(0);
visited[0] = true;
while (!q.empty()) {
    int u = q.front(); q.pop();
    for (int w : adj[u])
        if (!visited[w]) { visited[w] = true; q.push(w); }   // mark when pushing, not when popping
}
```

For weighted graphs use `vector<vector<pair<int, int>>> adj(n);` holding `{neighbor, weight}`.

**Min-heap of pairs (Dijkstra)**

```cpp
vector<long long> dist(n, LLONG_MAX);
priority_queue<pair<long long, int>, vector<pair<long long, int>>, greater<>> pq;   // {dist, node}, smallest first
dist[src] = 0;
pq.push({0, src});
while (!pq.empty()) {
    auto [d, u] = pq.top(); pq.pop();        // copy, then pop (a reference would dangle)
    if (d > dist[u]) continue;               // stale entry: a shorter path was found already
    for (auto [v, w] : adj[u]) {
        if (d + w < dist[v]) {
            dist[v] = d + w;
            pq.push({dist[v], v});
        }
    }
}
```

**Limits and overflow**

```cpp
#include <climits>

int best = INT_MAX;                  // 2147483647, about 2.1e9
long long big = LLONG_MAX;           // about 9.2e18
int a = 100000, b = 100000;
long long wrong = a * b;             // int * int overflows BEFORE the conversion: undefined
long long right = 1LL * a * b;       // 10000000000
int mid = lo + (hi - lo) / 2;        // avoids overflowing lo + hi
```

Using `INT_MAX` as "infinity" and then adding to it overflows: check before adding, or use `INT_MAX / 2` or `long long`. `numeric_limits<int>::max()` from `<limits>` is the type-safe spelling.

**Grids and 2D vectors**

```cpp
int rows = grid.size(), cols = grid[0].size();
vector<vector<int>> dp(rows + 1, vector<int>(cols + 1, -1));    // memo table filled with -1
int dr[] = {-1, 1, 0, 0}, dc[] = {0, 0, -1, 1};                  // up, down, left, right
for (int k = 0; k < 4; ++k) {
    int nr = r + dr[k], nc = c + dc[k];
    if (nr < 0 || nr >= rows || nc < 0 || nc >= cols) continue;  // bounds check first
    // visit grid[nr][nc]
}
```

**Building strings**

```cpp
string out;
out.reserve(s.size());
for (char ch : s)
    if (isalnum((unsigned char)ch)) out += (char)tolower((unsigned char)ch);   // O(n) total
// out = out + ch;   // copies the whole string every time: O(n^2)
reverse(out.begin(), out.end());
string t = to_string(42) + "!";
int num = stoi("123");
string sub = s.substr(2, 3);      // (start, length): O(length), allocates a new string
```

## 11. Fast I/O

```cpp
int main() {
    ios::sync_with_stdio(false);    // stop syncing C++ streams with C stdio
    cin.tie(nullptr);               // don't flush cout before every cin
    int n;
    cin >> n;
    vector<long long> a(n);
    for (auto& x : a) cin >> x;
    cout << accumulate(a.begin(), a.end(), 0LL) << '\n';   // '\n', not endl (endl flushes)
}
```

After `sync_with_stdio(false)`, do not mix `printf`/`scanf` with `cin`/`cout`. LeetCode handles I/O for you; this matters on platforms and online assessments that read from standard input.

## Common pitfalls

```cpp
map<string, int> m;
if (m["x"] == 0) {}                    // BUG: inserts "x" -> 0; m.size() is now 1
if (m.count("x")) {}                   // correct (or m.find("x") != m.end())

vector<int> v;
for (int i = 0; i < v.size() - 1; ++i) {}       // BUG: size() - 1 wraps to SIZE_MAX when v is empty
for (int i = 0; i + 1 < (int)v.size(); ++i) {}  // correct

sort(v.begin(), v.end(), [](int a, int b) { return a <= b; });   // BUG: not a strict weak ordering
sort(v.begin(), v.end(), [](int a, int b) { return a < b; });    // correct

for (int x : v) if (x == 0) v.push_back(1);     // BUG: push_back may reallocate under the loop
```

- Holding a reference into `pq.top()` or `v[i]` across `pop()` or `push_back()`.
- `std::lower_bound` on a `set` (O(n)) instead of `s.lower_bound`.
- Forgetting to sort before `unique`, `binary_search` or `next_permutation`.
- `accumulate(v.begin(), v.end(), 0)` on large values: the `int` accumulator overflows.
- `v.reserve(n)` followed by `v[i] = x`: the size is still 0 (use `resize` or `push_back`).
- Passing vectors by value into recursive functions (a copy per call): pass by reference.
- `endl` in big output loops, causing time-limit failures.

## How to talk about this in an interview
- Narrate container choices with their complexity: "I'll use an `unordered_map` for O(1) average lookups; if I needed keys in order or 'next larger key' queries, I'd switch to `map` at O(log n)."
- Mention that at university and as a Data Structures TA you implemented linked lists, stacks, queues and BSTs from scratch before relying on the STL, so you can explain what each container does underneath.
- If the interviewer prefers another language, map it: `unordered_map` is a Ruby `Hash`, a JavaScript `Map` or a Python `dict`; Python has `heapq` (a min-heap) while JavaScript and Ruby have no built-in heap, so you would implement one.
- > **Fill in:** which data structures your labs implemented by hand, and how much LeetCode-style practice you have done in C++ (platforms, rough counts).

## Interview questions

**Q1. `std::map` vs `std::unordered_map`: when do you use each?**
`map` is a red-black tree: keys stay sorted and every operation is O(log n) guaranteed, with `lower_bound` and range queries. `unordered_map` is a hash table: O(1) average lookups but O(n) worst case, no order, and keys need a hash. Default to `unordered_map` for pure lookups and counting; choose `map` when you need ordering, predecessor/successor queries, or guaranteed bounds.
*Follow-up:* Which uses more memory? Usually `unordered_map`, because of the bucket array on top of the nodes.

**Q2. How does `vector` grow, and why is `push_back` amortized O(1)?**
When it runs out of capacity it allocates a larger buffer (a constant factor, about 2x in GCC and 1.5x in MSVC), moves the elements over and frees the old buffer. That push costs O(n), but geometric growth means n pushes do less than about 2n element copies in total, so the average is O(1) per push.
*Follow-up:* What does growth do to iterators? Reallocation invalidates every iterator, pointer and reference into the vector.

**Q3. What is the difference between `reserve` and `resize`?**
`reserve(n)` only ensures capacity, so the size and the elements are unchanged and you still `push_back`. `resize(n)` changes the size, creating value-initialized elements (or destroying extras), so `v[i]` is valid afterwards. Writing `v[i]` after only `reserve` is undefined behavior.

**Q4. What is iterator invalidation? Give an example.**
An iterator, pointer or reference becomes invalid when the container operation moves or destroys what it refers to. Example: holding `&v[0]` and then `push_back` causing a reallocation, or calling `m.erase(it)` and then `++it`. Node-based containers are friendlier: erasing invalidates only the erased element, which is why `it = m.erase(it)` works.

**Q5. Why can `unordered_map` degrade to O(n)?**
All keys might hash into the same bucket (adversarial input or a poor custom hash), turning the bucket into a list that every lookup scans. Mitigations: a stronger or randomized hash, `reserve` to limit rehashing, or switching to `map` for guaranteed O(log n).

**Q6. How do you build a min-heap with `priority_queue`?**
`priority_queue<int, vector<int>, greater<int>>`. The comparator says whether `a` has lower priority than `b`, so the default `less` gives a max-heap and `greater` a min-heap. For pairs, `greater<>` orders by first then second; with a lambda, pass `decltype(lambda)` as the type and the lambda to the constructor.

**Q7. Explain `lower_bound`, `upper_bound` and `binary_search`. How do you count occurrences of x in a sorted vector?**
`lower_bound` returns the first position whose element is >= x, `upper_bound` the first position > x, and `binary_search` returns a bool. All are O(log n) on a sorted random-access range. Count = `upper_bound(...) - lower_bound(...)`, or use `equal_range`.

**Q8. How do you remove duplicates from a vector?**
Sort it, then `v.erase(unique(v.begin(), v.end()), v.end())`. `unique` only collapses adjacent duplicates, shifting the kept elements to the front and returning the new logical end; `erase` actually shrinks the vector. Total O(n log n). If order must be kept, filter through an `unordered_set` instead.

**Q9. Why must a `sort` comparator be a strict weak ordering? What goes wrong with `<=`?**
`sort` assumes `comp(a, a)` is false and that "equivalent" behaves consistently. With `<=`, two equal elements each claim to come before the other, which can push the algorithm's internal loops past the range: wrong results, infinite loops or crashes. Always express "strictly less than".

**Q10. What is the difference between `sort` and `stable_sort`?**
Both are O(n log n). `sort` (typically introsort) may reorder equal elements; `stable_sort` (merge sort) keeps them in their original order and uses extra memory. Use stability for multi-key sorting done in passes, or when equal records must keep input order.

**Q11. Why doesn't `unordered_map<pair<int, int>, int>` compile, and how do you fix it?**
The standard library has no `std::hash` specialization for `pair`. Provide a hash functor as the third template argument (for example, pack both ints into a 64-bit key), use `map<pair<int, int>, int>`, which only needs `operator<`, or encode the pair as one integer such as `r * cols + c`.

**Q12. What does `m[key]` do when the key is missing?**
It inserts the key with a value-initialized value (0 for numbers, empty for strings) and returns a reference to it. That is handy for counting (`freq[x]++`) but a bug for lookups: just checking `m[key] == 0` grows the map. Use `find`, `count` or `contains` (C++20) to test membership; `at()` throws if the key is missing.

**Q13. `vector` vs `list` vs `deque`: which would you pick?**
`vector` almost always: contiguous memory makes it cache-friendly, and indexing is O(1). `deque` when you need O(1) pushes and pops at both ends with indexing (sliding windows). `list` only when you insert and erase in the middle through saved iterators or need `splice`; its O(1) insert rarely beats a vector in practice because of pointer chasing.

**Q14. Why is `for (int i = 0; i < v.size() - 1; i++)` dangerous?**
`size()` returns an unsigned `size_t`. When `v` is empty, `v.size() - 1` wraps to the maximum value, so the loop runs and indexes out of bounds. Write `i + 1 < v.size()` or cast the size to `int` first.

**Q15. What is the difference between a container and a container adaptor? Why can't you iterate a `stack`?**
Adaptors (`stack`, `queue`, `priority_queue`) wrap an underlying container (by default `deque`, `deque` and `vector`) and expose only a restricted interface, deliberately hiding iterators so the LIFO, FIFO or heap rules cannot be broken. Use the underlying container directly if you need to iterate.

**Q16. What is the difference between `push_back` and `emplace_back`?**
`push_back` takes an already constructed object and copies or moves it in. `emplace_back` forwards its arguments to the element's constructor and builds it in place, saving a temporary: `v.emplace_back(1, 2)` for a `vector<pair<int, int>>`. For simple types the difference is negligible.

**Q17. What is a functor, and how does it relate to a lambda?**
A functor is an object of a class with `operator()`, so it can be called like a function and can carry state; `greater<int>` is one. A lambda is shorthand: the compiler generates an unnamed functor class whose data members are the captures. Both can be passed to algorithms and used as comparators.
