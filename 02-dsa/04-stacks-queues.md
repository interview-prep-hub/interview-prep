# Stacks and Queues

> **Tier:** T1 · **Related:** [Linked lists](03-linked-lists.md), [Recursion and backtracking](05-recursion-backtracking.md), [Graphs](09-graphs.md), [STL for DSA](../03-languages/c-cpp/04-stl-for-dsa.md), [Background jobs with Sidekiq](../05-ruby-on-rails/09-background-jobs-sidekiq.md) · **Language:** C++17

## Quick revision
- **Stack:** last in, first out (LIFO). push, pop, top, all O(1). Think plates, undo, the call stack.
- **Queue:** first in, first out (FIFO). push at the back, pop at the front, O(1). Think lines, BFS, job queues.
- **Deque:** O(1) push and pop at both ends plus O(1) indexing.
- `std::stack` and `std::queue` are adapters over `std::deque` by default. `pop()` returns `void`: read `top()`/`front()` first. Calling `top()`, `front()` or `pop()` on an empty container is undefined behavior.
- A `vector` makes a perfectly good stack (`push_back`, `back`, `pop_back`) and lets you return the contents directly.
- Queue from two stacks: push onto `in`; pop from `out`, refilling `out` from `in` only when `out` is empty. Amortized O(1).
- **Monotonic stack:** keep indices whose values are increasing or decreasing; when a new value breaks the order, pop and resolve the popped items. Each index is pushed and popped once: O(n).
- Next greater element: decreasing stack, pop while `a[top] < a[i]`. Next smaller: increasing stack, pop while `a[top] > a[i]`.
- **Monotonic deque:** sliding window maximum in O(n): drop indices that left the window from the front and smaller values from the back.
- Signals: matching brackets, nested encodings, "most recent", expression evaluation, "next greater/warmer/smaller", spans, histogram areas, window max/min.

## 1. Concept from scratch

### 1.1 Stack, queue, deque
A **stack** only exposes its top: the last item pushed is the first popped. A **queue** is a line: items leave in arrival order. A **deque** (double-ended queue) allows both. All three are restricted interfaces; restricting what you can do makes the intent obvious and the operations O(1).

### 1.2 The STL versions
| Container | Header | Core operations | Default underlying container |
|---|---|---|---|
| `stack<T>` | `<stack>` | `push`, `pop`, `top`, `empty`, `size` | `deque<T>` |
| `queue<T>` | `<queue>` | `push`, `pop`, `front`, `back`, `empty`, `size` | `deque<T>` |
| `deque<T>` | `<deque>` | `push_front/back`, `pop_front/back`, `front`, `back`, `[i]` | (a real container) |
| `priority_queue<T>` | `<queue>` | `push`, `pop`, `top` | `vector<T>` (see [Heaps](08-heaps.md)) |

`pop()` returns nothing because returning the removed element by value could throw while copying after the element is already gone, which would lose it; separating `top()` and `pop()` keeps the operations exception-safe.

### 1.3 Implementing them yourself
- **Stack:** a `vector` with `push_back`/`pop_back`, or a singly linked list that pushes at the head.
- **Queue:** a singly linked list with head and tail pointers, or a **circular buffer**: a fixed array with a head index and a count, where indices wrap with `% capacity`.
- **Queue from two stacks:** shown in problem 3.4.

```cpp
#include <stdexcept>
#include <vector>
using namespace std;

class CircularQueue {                      // fixed capacity (> 0), O(1) push and pop
    vector<int> buf;
    int head = 0, count = 0;
public:
    explicit CircularQueue(int capacity) : buf(capacity) {}
    bool push(int x) {
        if (count == (int)buf.size()) return false;        // full
        buf[(head + count) % buf.size()] = x;               // slot after the last element, wrapping
        count++;
        return true;
    }
    int pop() {
        if (count == 0) throw out_of_range("empty queue");
        int x = buf[head];
        head = (head + 1) % buf.size();
        count--;
        return x;
    }
    bool empty() const { return count == 0; }
};
```

### 1.4 Where they show up in real systems
| Structure | Real use |
|---|---|
| Stack | function call stack, undo/redo, browser back button, parsing nested JSON/HTML, iterative DFS |
| Queue | BFS, job queues (Sidekiq queues are Redis lists consumed in FIFO order), print spoolers, request buffers, message brokers |
| Deque | sliding-window maximum, work-stealing schedulers, a history with a size limit (drop the oldest from the front) |

### 1.5 The monotonic stack
Many problems ask, for every element, "where is the next element bigger (or smaller) than me?" Brute force scans right from each element, O(n^2). Instead, keep a stack of indices still waiting for an answer, with values kept in decreasing order. When a new value arrives that is bigger than the top, it is the answer for that top: pop it and record, and keep popping while the condition holds. Then push the new index. Every index is pushed once and popped at most once, so the whole pass is O(n).

| Question | Stack order (bottom to top) | Pop while |
|---|---|---|
| Next greater to the right | decreasing | `a[top] < a[i]` |
| Next smaller to the right | increasing | `a[top] > a[i]` |
| Previous greater (to the left) | decreasing | `a[top] <= a[i]`, then the top is the answer |
| Circular array | same, loop `i` from 0 to 2n - 1 using `i % n` | same |

### 1.6 The monotonic deque
For "maximum of every window of size k", keep a deque of indices whose values are decreasing from front to back. The front is always the window maximum. Before pushing index i, pop smaller-or-equal values from the back (they can never be the maximum while i is in the window), and pop the front if it has slid out of the window. O(n) in total.

## 2. Core templates
```cpp
#include <deque>
#include <stack>
#include <vector>
using namespace std;

// Next greater value to the right of each element, or -1.
vector<int> nextGreater(const vector<int>& a) {
    int n = a.size();
    vector<int> result(n, -1);
    stack<int> st;                                   // indices still waiting; their values decrease
    for (int i = 0; i < n; i++) {
        while (!st.empty() && a[st.top()] < a[i]) {
            result[st.top()] = a[i];                 // a[i] is the first bigger value to the right
            st.pop();
        }
        st.push(i);
    }
    return result;
}

// Maximum of every window of size k (1 <= k <= a.size()).
vector<int> windowMax(const vector<int>& a, int k) {
    deque<int> dq;                                   // indices; values decrease front to back
    vector<int> result;
    for (int i = 0; i < (int)a.size(); i++) {
        if (!dq.empty() && dq.front() <= i - k) dq.pop_front();          // left the window
        while (!dq.empty() && a[dq.back()] <= a[i]) dq.pop_back();       // can never be the max now
        dq.push_back(i);
        if (i >= k - 1) result.push_back(a[dq.front()]);
    }
    return result;
}
```

## 3. Problems

### 3.1 Valid Parentheses (LeetCode 20, Easy)
**Problem (paraphrased):** a string contains only the six bracket characters. Decide whether every opening bracket is closed by the same type, in the correct nesting order.
**Approach:** brute force repeatedly deletes adjacent pairs like `()` until nothing changes, O(n^2). A stack does it in one pass: push openers; on a closer, the top must be its matching opener.
**Complexity:** O(n) time, O(n) space.
```cpp
class Solution {
public:
    bool isValid(string s) {
        stack<char> st;
        for (char c : s) {
            if (c == '(' || c == '[' || c == '{') { st.push(c); continue; }
            if (st.empty()) return false;                        // closer with nothing open
            char open = st.top();
            st.pop();
            if ((c == ')' && open != '(') || (c == ']' && open != '[') || (c == '}' && open != '{'))
                return false;
        }
        return st.empty();                                       // leftover openers are invalid
    }
};
```
**Edge cases:** a lone closer; only openers; odd length (can return false immediately).

### 3.2 Min Stack (LeetCode 155, Medium)
**Problem (paraphrased):** design a stack that supports push, pop, top and retrieving the current minimum, all in O(1).
**Approach:** scanning for the minimum is O(n). Store with every element the minimum of the stack up to and including it; the top pair then always knows the current minimum, and popping restores the previous one automatically.
**Complexity:** O(1) per operation, O(n) space.
```cpp
class MinStack {
    stack<pair<int, int>> st;   // (value, minimum of everything at or below this element)
public:
    MinStack() {}
    void push(int val) {
        int m = st.empty() ? val : min(val, st.top().second);
        st.push({val, m});
    }
    void pop() { st.pop(); }
    int top() { return st.top().first; }
    int getMin() { return st.top().second; }
};
```
**Edge cases:** duplicate minimums (handled, since each element stores its own minimum); negative values. The problem guarantees `pop`, `top` and `getMin` are never called on an empty stack.

### 3.3 Evaluate Reverse Polish Notation (LeetCode 150, Medium)
**Problem (paraphrased):** evaluate an arithmetic expression written in postfix form (operands before their operator) with `+`, `-`, `*` and `/`, where division truncates toward zero.
**Approach:** postfix exists so a single stack can evaluate it: push numbers; for an operator, pop the right operand, then the left, and push the result.
**Complexity:** O(n) time, O(n) space.
```cpp
class Solution {
public:
    int evalRPN(vector<string>& tokens) {
        stack<long long> st;
        for (const string& t : tokens) {
            if (t == "+" || t == "-" || t == "*" || t == "/") {      // compare whole tokens: "-11" is a number
                long long b = st.top(); st.pop();                    // right operand comes off first
                long long a = st.top(); st.pop();
                if (t == "+") st.push(a + b);
                else if (t == "-") st.push(a - b);
                else if (t == "*") st.push(a * b);
                else st.push(a / b);                                 // C++ integer division truncates toward zero
            } else {
                st.push(stoll(t));
            }
        }
        return (int)st.top();
    }
};
```
**Edge cases:** negative numbers as tokens; operand order for `-` and `/`; a single number.

### 3.4 Implement Queue using Stacks (LeetCode 232, Easy)
**Problem (paraphrased):** build a FIFO queue (push, pop, peek, empty) using only stack operations.
**Approach:** with a single stack you would move everything out and back for each pop, O(n). Use two stacks: `inbox` receives pushes, `outbox` serves pops. Refill `outbox` from `inbox` only when `outbox` is empty, which reverses the order exactly once per element.
**Complexity:** amortized O(1) per operation (each element moves at most once), O(n) space.
```cpp
class MyQueue {
    stack<int> inbox, outbox;
    void refill() {                                    // only when outbox has run dry
        if (outbox.empty())
            while (!inbox.empty()) { outbox.push(inbox.top()); inbox.pop(); }
    }
public:
    MyQueue() {}
    void push(int x) { inbox.push(x); }
    int pop() { refill(); int x = outbox.top(); outbox.pop(); return x; }
    int peek() { refill(); return outbox.top(); }
    bool empty() { return inbox.empty() && outbox.empty(); }
};
```
**Edge cases:** interleaved pushes and pops (refilling while `outbox` still has items would break the order).

### 3.5 Daily Temperatures (LeetCode 739, Medium)
**Problem (paraphrased):** for each day, output how many days you must wait for a strictly warmer temperature, or 0 if it never comes.
**Approach:** brute force scans forward from each day, O(n^2). A monotonic decreasing stack of day indices holds the days still waiting; a warmer day resolves every cooler day on top of the stack.
**Complexity:** O(n) time, O(n) space.
```cpp
class Solution {
public:
    vector<int> dailyTemperatures(vector<int>& temps) {
        int n = temps.size();
        vector<int> answer(n, 0);
        stack<int> st;                                  // indices of days without a warmer day yet
        for (int i = 0; i < n; i++) {
            while (!st.empty() && temps[st.top()] < temps[i]) {
                answer[st.top()] = i - st.top();        // day i is the first warmer day
                st.pop();
            }
            st.push(i);
        }
        return answer;
    }
};
```
**Edge cases:** strictly decreasing temperatures (all zeros); equal temperatures (not warmer, so they stay on the stack).

### 3.6 Next Greater Element I (LeetCode 496, Easy)
**Problem (paraphrased):** `nums1` is a subset of `nums2` (all values distinct). For each value in `nums1`, find the first larger value to its right in `nums2`, or -1.
**Approach:** brute force finds each value in `nums2` and scans right, O(n * m). Run the monotonic stack once over `nums2`, store each value's next greater value in a hash map, then answer each query in O(1).
**Complexity:** O(n + m) time, O(m) space.
```cpp
class Solution {
public:
    vector<int> nextGreaterElement(vector<int>& nums1, vector<int>& nums2) {
        unordered_map<int, int> nextGreater;          // value -> next greater value in nums2
        stack<int> st;                                // values still waiting, decreasing
        for (int x : nums2) {
            while (!st.empty() && st.top() < x) {
                nextGreater[st.top()] = x;
                st.pop();
            }
            st.push(x);
        }
        vector<int> ans;
        ans.reserve(nums1.size());
        for (int x : nums1) {
            auto it = nextGreater.find(x);
            ans.push_back(it == nextGreater.end() ? -1 : it->second);
        }
        return ans;
    }
};
```
**Edge cases:** the last element of `nums2` (always -1); a decreasing `nums2`. The circular version (LeetCode 503) loops twice over the array with `i % n`.

### 3.7 Decode String (LeetCode 394, Medium)
**Problem (paraphrased):** expand an encoded string where `k[text]` means `text` repeated k times; encodings can nest, as in `3[a2[c]]`, which becomes `accaccacc`.
**Approach:** recursion that parses one bracket level per call works. Iteratively, a stack saves the string built so far and the pending repeat count whenever a `[` opens a new level; `]` pops them and appends the current piece repeated k times.
**Complexity:** O(output length x nesting depth) time in the worst case, because each level copies its expanded text into the level above; O(output length) space.
```cpp
class Solution {
public:
    string decodeString(string s) {
        stack<pair<string, int>> st;     // (text before the '[', repeat count for this bracket)
        string cur;
        int num = 0;
        for (char c : s) {
            if (isdigit(static_cast<unsigned char>(c))) {
                num = num * 10 + (c - '0');            // counts can have several digits
            } else if (c == '[') {
                st.push({cur, num});
                cur.clear();
                num = 0;
            } else if (c == ']') {
                auto [prev, k] = st.top();
                st.pop();
                string repeated;
                repeated.reserve(cur.size() * k);
                for (int i = 0; i < k; i++) repeated += cur;
                cur = prev + repeated;
            } else {
                cur += c;
            }
        }
        return cur;
    }
};
```
**Edge cases:** multi-digit counts like `12[ab]`; letters outside any bracket; deep nesting.

### 3.8 Asteroid Collision (LeetCode 735, Medium)
**Problem (paraphrased):** asteroids move along a line; the sign gives direction (positive right, negative left) and the absolute value gives size. When two collide, the smaller explodes, and equal sizes both explode. Return the survivors.
**Approach:** brute force simulates repeated passes, O(n^2). A stack holds the survivors so far. Only a left-moving asteroid meeting a right-moving top collides; resolve collisions until the newcomer dies or nothing collides.
**Complexity:** O(n) time (each asteroid is pushed and popped at most once), O(n) space.
```cpp
class Solution {
public:
    vector<int> asteroidCollision(vector<int>& asteroids) {
        vector<int> st;                                   // a vector as a stack: easy to return
        for (int a : asteroids) {
            bool alive = true;
            while (alive && a < 0 && !st.empty() && st.back() > 0) {
                if (st.back() < -a) {
                    st.pop_back();                        // the top explodes; keep checking
                } else {
                    if (st.back() == -a) st.pop_back();   // equal sizes: both explode
                    alive = false;                        // the newcomer explodes
                }
            }
            if (alive) st.push_back(a);
        }
        return st;
    }
};
```
**Edge cases:** `[-2, -1, 1, 2]` (no collisions: they move apart); `[8, -8]` (both explode, empty result); a large negative clearing several positives.

### 3.9 Sliding Window Maximum (LeetCode 239, Hard)
**Problem (paraphrased):** for every window of k consecutive elements, report the largest value in it.
**Approach:** brute force scans each window, O(n * k). A balanced tree (`multiset`) gives O(n log k); a max-heap with lazy removal gives O(n log n). The monotonic deque from section 1.6 gives O(n): its front is always the current window's maximum.
**Complexity:** O(n) time, O(k) space.
```cpp
class Solution {
public:
    vector<int> maxSlidingWindow(vector<int>& nums, int k) {
        deque<int> dq;                                     // indices; values decrease front to back
        vector<int> result;
        for (int i = 0; i < (int)nums.size(); i++) {
            if (!dq.empty() && dq.front() <= i - k) dq.pop_front();          // slid out of the window
            while (!dq.empty() && nums[dq.back()] <= nums[i]) dq.pop_back(); // dominated by nums[i]
            dq.push_back(i);
            if (i >= k - 1) result.push_back(nums[dq.front()]);
        }
        return result;
    }
};
```
**Edge cases:** k = 1 (the array itself); k = n (one value); repeated maximums.

### 3.10 Largest Rectangle in Histogram (LeetCode 84, Hard)
**Problem (paraphrased):** bars of width 1 have the given heights. Find the area of the largest rectangle that fits entirely inside the histogram.
**Approach:** brute force tries every pair of edges or expands from every bar, O(n^2). The best rectangle that uses bar i at full height stretches from the first shorter bar on its left to the first shorter bar on its right. A monotonic increasing stack finds both: when bar i is shorter than the top, i is the top's right boundary and the element below the top is its left boundary. A zero-height sentinel at the end flushes the stack.
**Complexity:** O(n) time, O(n) space.
```cpp
class Solution {
public:
    int largestRectangleArea(vector<int>& heights) {
        int n = heights.size(), best = 0;
        stack<int> st;                                     // indices of bars with increasing heights
        for (int i = 0; i <= n; i++) {
            int h = (i == n) ? 0 : heights[i];             // sentinel bar of height 0 at the end
            while (!st.empty() && heights[st.top()] >= h) {
                int height = heights[st.top()];
                st.pop();
                int left = st.empty() ? -1 : st.top();     // first shorter bar on the left
                best = max(best, height * (i - left - 1)); // i is the first shorter bar on the right
            }
            st.push(i);
        }
        return best;
    }
};
```
**Edge cases:** one bar; all equal heights; strictly increasing heights (everything resolves at the sentinel). Maximal Rectangle (LeetCode 85) reuses this per row of a binary matrix.

## Common pitfalls
- Calling `top()`, `front()` or `pop()` on an empty container: undefined behavior, often a crash. Check `empty()` first.
- Expecting `pop()` to return the element; it returns `void`.
- Storing values instead of indices in a monotonic stack when you need distances or widths.
- Choosing `<` versus `<=` carelessly: it decides whether equal elements count as "greater" and how duplicates are handled.
- Forgetting to flush the stack at the end (the sentinel trick in problem 3.10) and missing the last answers.
- In the two-stack queue, moving items from `inbox` to `outbox` while `outbox` still has elements, which breaks FIFO order.
- Treating `"-"` and `"-11"` the same in RPN by checking only the first character.

## How to talk about this in an interview
- Say which order you need first: "I need the most recent unmatched item, so a stack" or "I need to process in arrival order, so a queue." That one sentence shows you picked the structure for a reason.
- For monotonic stacks, explain the invariant and the amortized argument: "The stack holds indices still waiting for a bigger value, in decreasing order. Each index is pushed and popped once, so it's O(n)." As a Data Structures lab TA you can present it the way you would on the lab whiteboard: draw the bars and pop them as the new bar arrives.
- Connect queues to your backend work: Sidekiq jobs wait in Redis-backed queues and are processed by worker threads; retries and scheduled jobs live in separate sorted structures. See [background jobs](../05-ruby-on-rails/09-background-jobs-sidekiq.md).
- > **Fill in:** a background job in your projects (for example a marketplace sync or a notification sender) and what happens when its queue backs up or a job fails.

## Interview questions

**Q1. Stack versus queue: what is the difference, and where is each used in real systems?**
A stack is LIFO: the newest item leaves first (call stack, undo, back button, parsing nested structures, iterative DFS). A queue is FIFO: the oldest item leaves first (BFS, job queues, print and request buffers, message brokers). Both offer O(1) insert and remove at their designated ends.
*Follow-up:* Which one does BFS use and why? A queue, so nodes are processed in order of distance from the source.

**Q2. How are `std::stack` and `std::queue` implemented?**
They are container adapters: thin wrappers that expose a restricted interface over another container, `std::deque` by default. You can choose another backing container, such as `stack<int, vector<int>>`. They provide no iterators.
*Follow-up:* Why `deque` and not `vector` for `queue`? A queue needs O(1) removal at the front, which `vector` cannot do.

**Q3. What is a monotonic stack, and why is it O(n) despite the inner loop?**
A stack whose values stay sorted (increasing or decreasing). New elements pop everything that breaks the order, and each pop resolves an answer such as "next greater". Each index is pushed once and popped at most once, so all the inner loops together do at most n pops.
*Follow-up:* Name three problems that use it. Daily Temperatures, Largest Rectangle in Histogram, Trapping Rain Water.

**Q4. How would you implement a queue with a fixed-size array?**
A circular buffer: keep a head index and a count; push writes at `(head + count) % capacity`, pop reads at `head` and advances it with wraparound. Both are O(1), with no shifting. Full and empty are distinguished by the count.
*Follow-up:* What if it must grow? Allocate a bigger array and copy the elements in logical order, starting from head (amortized O(1), like `vector`).

**Q5. How do you implement a stack using queues?**
With one queue: after pushing x, rotate the queue by popping and re-pushing the previous size elements so x reaches the front. Push becomes O(n), and pop and top are O(1) (Implement Stack using Queues, LeetCode 225).
*Follow-up:* Which is cheaper, queue-from-stacks or stack-from-queues? Queue from two stacks, with amortized O(1) for every operation.

**Q6. `deque` versus `vector` versus `list`: when do you use each?**
`vector` for contiguous storage and push/pop at the back; `deque` when you also need O(1) push and pop at the front (it stores fixed-size blocks, so it is less contiguous); `list` when you splice nodes you already hold and need iterators to stay valid.
*Follow-up:* Does `deque` keep references valid on `push_front`? References to elements yes, iterators no.

**Q7. How does the call stack relate to recursion and stack overflow?**
Each function call pushes a frame (arguments, locals, return address) onto the thread's stack, and returning pops it. Deep or infinite recursion exhausts the fixed stack size and crashes with a stack overflow. Any recursion can be rewritten with an explicit `std::stack` on the heap.
*Follow-up:* See [Recursion and backtracking](05-recursion-backtracking.md) for when to convert.

**Q8. Where do queues show up in backend systems you have worked with?**
Background job systems such as Sidekiq push jobs onto Redis-backed queues and worker threads pop them. Queues also smooth traffic spikes (buffering webhooks), decouple services (message brokers), and drive BFS-style crawls or sync pipelines.
*Follow-up:* What guarantee do most job queues give? At-least-once delivery, so jobs must be idempotent.
