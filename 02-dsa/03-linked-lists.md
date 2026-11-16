# Linked Lists

> **Tier:** T1 · **Related:** [Two pointers and sliding window](02-two-pointers-sliding-window.md), [Stacks and queues](04-stacks-queues.md), [C++ fundamentals](../03-languages/c-cpp/02-cpp-fundamentals.md), [STL for DSA](../03-languages/c-cpp/04-stl-for-dsa.md), [Caching and performance](../04-web-fundamentals/08-caching-and-performance.md) · **Language:** C++17

## Quick revision
- A node holds a value and a pointer to the next node; the list is a chain from `head` to `nullptr`.
- Access by position is O(n); insert/delete after a node you already hold is O(1); no random access, poor cache locality, extra pointer per node.
- **Dummy (sentinel) head:** `ListNode dummy(0); dummy.next = head;` removes special cases for an empty list or a changing head. Return `dummy.next`.
- **Fast/slow pointers:** middle (`fast` moves 2, `slow` moves 1), cycle detection (Floyd), k-th from the end (start `fast` k steps ahead).
- **Reversal:** keep `prev`, `curr`, `next`; save `next` before overwriting `curr->next`.
- Cycle start (Floyd): after slow and fast meet, restart one pointer at `head`; moving both one step at a time, they meet at the cycle entry.
- Many medium problems combine three moves: find the middle, reverse a half, merge or compare.
- Always check `fast && fast->next` before `fast->next->next`.
- `std::list` is doubly linked (O(1) `splice`, iterators stay valid); `std::forward_list` is singly linked.
- LRU cache = doubly linked list (recency order) + hash map (key to node): O(1) get and put.
- In real C++, decide who owns each node: delete what you unlink, avoid `shared_ptr` cycles, tear down long `unique_ptr` chains iteratively.

## 1. Concept from scratch

### 1.1 The node and the list
LeetCode problems use this definition (some problems only have the `int` constructor):
```cpp
struct ListNode {
    int val;
    ListNode* next;
    ListNode() : val(0), next(nullptr) {}
    ListNode(int x) : val(x), next(nullptr) {}
    ListNode(int x, ListNode* next) : val(x), next(next) {}
};
```
A **singly linked** list only points forward. A **doubly linked** list also stores `prev`, which allows O(1) removal of a node you hold and backward traversal at the cost of one more pointer. A **circular** list links the tail back to the head.

### 1.2 Array versus linked list
| | Array / `vector` | Linked list |
|---|---|---|
| Memory layout | contiguous | scattered nodes joined by pointers |
| Access by index | O(1) | O(n) |
| Insert/delete at the front | O(n) (shift) | O(1) |
| Insert/delete in the middle | O(n) | O(1) once you hold the neighbor node, O(n) to find it |
| Append | amortized O(1) | O(1) with a tail pointer |
| Extra memory | none per element (some spare capacity) | one or two pointers per node |
| Cache behavior | excellent | poor (pointer chasing) |
| Binary search | yes | no |

In practice a `vector` wins most benchmarks even for moderate middle inserts, because shifting contiguous memory is cheap and pointer chasing causes cache misses. Linked lists shine when you hold node handles and splice often: LRU caches, free lists in allocators, and the kernel's intrusive lists.

### 1.3 The dummy head
When the head can change (deleting the first node, merging, inserting before the head), a stack-allocated sentinel in front of the list means every real node has a predecessor, so one code path handles everything.

### 1.4 Fast and slow pointers
- **Middle:** `fast` moves two steps while `slow` moves one; when `fast` reaches the end, `slow` is in the middle.
- **Cycle:** if a cycle exists, `fast` gains one step per iteration on `slow` inside the cycle, so the gap shrinks by one each step and they must meet. If there is no cycle, `fast` hits `nullptr`.
- **K-th from the end:** move `fast` k steps first, then move both; when `fast` hits the end, `slow` is k nodes from it.

### 1.5 Memory ownership in C++
- On LeetCode, the judge allocates the nodes and you rearrange pointers. Unlinking a node without `delete` leaks it, which is accepted there; in your own code, whoever owns the list must free unlinked nodes exactly once.
- Use-after-free is the classic bug: reading `node->next` after `delete node`. Save the pointer first. AddressSanitizer (`-fsanitize=address` on GCC/Clang) catches these.
- Modern C++ ownership: `std::list`/`std::forward_list` own their nodes. If you write your own, `unique_ptr<Node> next` gives single ownership, but the default destructor frees the chain recursively and can overflow the stack on a long list, so tear it down iteratively.
- A doubly linked list with `shared_ptr` in both directions forms reference cycles and never frees; make the back pointer a raw pointer or `weak_ptr`.

```cpp
#include <memory>
#include <utility>

struct Node {
    int val;
    std::unique_ptr<Node> next;   // this node owns the rest of the chain
    explicit Node(int v) : val(v) {}
};

class SinglyList {
    std::unique_ptr<Node> head;
public:
    void pushFront(int v) {
        auto node = std::make_unique<Node>(v);
        node->next = std::move(head);
        head = std::move(node);
    }
    ~SinglyList() {               // iterative teardown: no deep recursion on long lists
        while (head) head = std::move(head->next);
    }
};
```

## 2. Core templates
```cpp
// Uses the ListNode struct from section 1.1; no extra headers needed.

ListNode* reverse(ListNode* head) {          // iterative in-place reversal
    ListNode* prev = nullptr;
    while (head) {
        ListNode* next = head->next;         // 1. save the rest
        head->next = prev;                   // 2. flip the arrow
        prev = head;                         // 3. advance prev
        head = next;                         // 4. advance curr
    }
    return prev;                             // the old tail is the new head
}

ListNode* middle(ListNode* head) {           // second middle for even lengths
    ListNode *slow = head, *fast = head;
    while (fast && fast->next) {
        slow = slow->next;
        fast = fast->next->next;
    }
    return slow;
}

ListNode* buildWithDummy(const int* values, int n) {   // dummy head makes appending uniform
    ListNode dummy(0);
    ListNode* tail = &dummy;
    for (int i = 0; i < n; i++) {
        tail->next = new ListNode(values[i]);
        tail = tail->next;
    }
    return dummy.next;
}
```

## 3. Problems

### 3.1 Reverse Linked List (LeetCode 206, Easy)
**Problem (paraphrased):** reverse a singly linked list and return the new head.
**Approach:** brute force copies the values into a vector and writes them back in reverse, O(n) extra space. Iteratively, flip each `next` pointer while walking with `prev` and `curr`. Recursively, reverse the rest of the list, then hook the current node onto its end.
**Complexity:** iterative O(n) time, O(1) space; recursive O(n) time, O(n) stack.
```cpp
class Solution {
public:
    ListNode* reverseList(ListNode* head) {
        ListNode* prev = nullptr;
        ListNode* curr = head;
        while (curr) {
            ListNode* next = curr->next;   // save before overwriting
            curr->next = prev;
            prev = curr;
            curr = next;
        }
        return prev;
    }

    ListNode* reverseListRecursive(ListNode* head) {
        if (!head || !head->next) return head;               // empty or last node: new head
        ListNode* newHead = reverseListRecursive(head->next);
        head->next->next = head;                             // the node after head points back to it
        head->next = nullptr;                                // head becomes the tail for now
        return newHead;
    }
};
```
**Edge cases:** empty list; one node; a very long list (prefer the iterative version to avoid stack overflow).

### 3.2 Merge Two Sorted Lists (LeetCode 21, Easy)
**Problem (paraphrased):** splice two sorted lists into one sorted list and return its head.
**Approach:** brute force collects all values, sorts them and builds a new list, O((n + m) log(n + m)). Better: a dummy head and a tail pointer; repeatedly attach the smaller front node, then attach whatever remains.
**Complexity:** O(n + m) time, O(1) extra space (nodes are reused).
```cpp
class Solution {
public:
    ListNode* mergeTwoLists(ListNode* a, ListNode* b) {
        ListNode dummy(0);                        // lives on the stack; return dummy.next
        ListNode* tail = &dummy;
        while (a && b) {
            if (a->val <= b->val) { tail->next = a; a = a->next; }   // <= keeps equal keys stable
            else { tail->next = b; b = b->next; }
            tail = tail->next;
        }
        tail->next = a ? a : b;                   // attach the leftover chain
        return dummy.next;
    }
};
```
**Edge cases:** one or both lists empty; all of one list smaller than the other.

### 3.3 Linked List Cycle (LeetCode 141, Easy) and Linked List Cycle II (LeetCode 142, Medium)
**Problem (paraphrased):** first, report whether following `next` pointers ever loops. Second, return the node where the loop begins, or null.
**Approach:** brute force stores visited node addresses in a hash set: O(n) time and O(n) space, and the first repeated node is the cycle start. Floyd's algorithm uses O(1) space: slow moves 1, fast moves 2; they meet inside the cycle if one exists. For the entry: let a be the distance from head to the entry, b from the entry to the meeting point, and c the rest of the cycle. Fast traveled twice as far as slow, which gives a = c + (k - 1)(b + c) for some k >= 1, so a pointer from head and a pointer from the meeting point, both moving one step, meet exactly at the entry.
**Complexity:** O(n) time, O(1) space.
```cpp
class Solution {
public:
    bool hasCycle(ListNode* head) {
        ListNode *slow = head, *fast = head;
        while (fast && fast->next) {
            slow = slow->next;
            fast = fast->next->next;
            if (slow == fast) return true;        // compare nodes (addresses), not values
        }
        return false;
    }

    ListNode* detectCycle(ListNode* head) {
        ListNode *slow = head, *fast = head;
        while (fast && fast->next) {
            slow = slow->next;
            fast = fast->next->next;
            if (slow == fast) {                   // inside the cycle
                ListNode* p = head;
                while (p != slow) { p = p->next; slow = slow->next; }
                return p;                         // the cycle entry
            }
        }
        return nullptr;
    }
};
```
**Edge cases:** empty list; a single node pointing to itself; the cycle starting at the head.

### 3.4 Middle of the Linked List (LeetCode 876, Easy)
**Problem (paraphrased):** return the middle node; for an even length, return the second of the two middle nodes.
**Approach:** brute force counts the length, then walks half of it: two passes. Fast/slow pointers do it in one pass.
**Complexity:** O(n) time, O(1) space.
```cpp
class Solution {
public:
    ListNode* middleNode(ListNode* head) {
        ListNode *slow = head, *fast = head;
        while (fast && fast->next) {
            slow = slow->next;
            fast = fast->next->next;
        }
        return slow;   // for 1-2-3-4 this is node 3
    }
};
```
**Edge cases:** one node; two nodes. For the first middle instead, loop while `fast->next && fast->next->next`.

### 3.5 Remove Nth Node From End of List (LeetCode 19, Medium)
**Problem (paraphrased):** delete the n-th node counting from the end and return the head (n is always valid).
**Approach:** brute force computes the length L in one pass and removes node L - n + 1 in a second. One pass: start both pointers at a dummy node, move `fast` n + 1 steps ahead, then move both until `fast` is null; `slow` then sits right before the target. The dummy covers deleting the head.
**Complexity:** O(L) time, O(1) space.
```cpp
class Solution {
public:
    ListNode* removeNthFromEnd(ListNode* head, int n) {
        ListNode dummy(0, head);
        ListNode *fast = &dummy, *slow = &dummy;
        for (int i = 0; i <= n; i++) fast = fast->next;   // gap of n + 1 links
        while (fast) {
            fast = fast->next;
            slow = slow->next;
        }
        slow->next = slow->next->next;   // unlink the target (delete it if your list owns the nodes)
        return dummy.next;
    }
};
```
**Edge cases:** removing the head (n equals the length); a single-node list (returns null).

### 3.6 Reorder List (LeetCode 143, Medium)
**Problem (paraphrased):** rearrange `L0, L1, ..., Ln` in place into `L0, Ln, L1, Ln-1, L2, ...` by relinking nodes.
**Approach:** brute force stores the nodes in a vector and relinks them with two indices, O(n) space. O(1) space uses the three standard moves: find the end of the first half, reverse the second half, then weave the two halves together.
**Complexity:** O(n) time, O(1) space.
```cpp
class Solution {
public:
    void reorderList(ListNode* head) {
        if (!head || !head->next) return;
        ListNode *slow = head, *fast = head;              // 1. slow stops at the end of the first half
        while (fast->next && fast->next->next) {
            slow = slow->next;
            fast = fast->next->next;
        }
        ListNode* prev = nullptr;                         // 2. reverse the second half
        ListNode* curr = slow->next;
        slow->next = nullptr;                             // cut the list in two
        while (curr) {
            ListNode* next = curr->next;
            curr->next = prev;
            prev = curr;
            curr = next;
        }
        ListNode *first = head, *second = prev;           // 3. weave: first, second, first, ...
        while (second) {
            ListNode* n1 = first->next;
            ListNode* n2 = second->next;
            first->next = second;
            second->next = n1;
            first = n1;
            second = n2;
        }
    }
};
```
**Edge cases:** lengths 1, 2 and 3; odd versus even length (the first half keeps the extra node).

### 3.7 Add Two Numbers (LeetCode 2, Medium)
**Problem (paraphrased):** two non-negative integers are stored as lists of digits in reverse order (ones digit first). Return their sum as a list in the same format.
**Approach:** converting the lists to integers overflows (up to 100 digits), so simulate grade-school addition with a carry, building the result behind a dummy head. Keep looping while either list or the carry remains.
**Complexity:** O(max(n, m)) time, O(1) extra space besides the output.
```cpp
class Solution {
public:
    ListNode* addTwoNumbers(ListNode* l1, ListNode* l2) {
        ListNode dummy(0);
        ListNode* tail = &dummy;
        int carry = 0;
        while (l1 || l2 || carry) {
            int sum = carry;
            if (l1) { sum += l1->val; l1 = l1->next; }
            if (l2) { sum += l2->val; l2 = l2->next; }
            carry = sum / 10;
            tail->next = new ListNode(sum % 10);
            tail = tail->next;
        }
        return dummy.next;
    }
};
```
**Edge cases:** different lengths; a final carry (`5 + 5` gives `0 -> 1`); both numbers zero.

### 3.8 Palindrome Linked List (LeetCode 234, Easy)
**Problem (paraphrased):** decide whether the list's values read the same in both directions, ideally with O(1) extra space.
**Approach:** brute force copies the values into a vector and uses two pointers, O(n) space. O(1) space: find the middle, reverse the second half, compare it with the first half node by node.
**Complexity:** O(n) time, O(1) space.
```cpp
class Solution {
public:
    bool isPalindrome(ListNode* head) {
        ListNode *slow = head, *fast = head;
        while (fast && fast->next) {               // slow ends at the start of the second half
            slow = slow->next;
            fast = fast->next->next;
        }
        ListNode* prev = nullptr;                  // reverse the second half
        while (slow) {
            ListNode* next = slow->next;
            slow->next = prev;
            prev = slow;
            slow = next;
        }
        for (ListNode *a = head, *b = prev; b; a = a->next, b = b->next)
            if (a->val != b->val) return false;
        return true;
    }
};
```
**Edge cases:** one node; odd length (the middle node is compared with itself, harmless). This modifies the input: mention that you would reverse the second half back if the caller still needs the list.

### 3.9 Copy List with Random Pointer (LeetCode 138, Medium)
**Problem (paraphrased):** each node has `next` and a `random` pointer to any node or null. Build a deep copy whose pointers only point into the new list.
**Approach:** the obvious approach maps each original node to its clone in a hash map (two passes: create clones, then wire `next` and `random`). An O(1)-extra-space variant interleaves clones into the original list (A, A', B, B', ...), sets `clone->random = original->random->next`, then separates the two lists.
**Complexity:** O(n) time, O(n) space for the map.
```cpp
class Solution {
public:
    Node* copyRandomList(Node* head) {
        unordered_map<Node*, Node*> cloneOf;                    // original -> copy
        for (Node* p = head; p; p = p->next) cloneOf[p] = new Node(p->val);
        for (Node* p = head; p; p = p->next) {
            cloneOf[p]->next = p->next ? cloneOf[p->next] : nullptr;
            cloneOf[p]->random = p->random ? cloneOf[p->random] : nullptr;
        }
        return head ? cloneOf[head] : nullptr;
    }
};
```
**Edge cases:** empty list; `random` pointing to the node itself; several nodes sharing the same `random` target.

### 3.10 LRU Cache (LeetCode 146, Medium)
**Problem (paraphrased):** design a fixed-capacity key-value cache with `get` and `put` in O(1) average time. When full, inserting a new key evicts the least recently used key; both `get` and `put` count as a use.
**Approach:** brute force keeps a vector of (key, value, last-used time) and scans it, O(n) per operation. O(1): a doubly linked list keeps entries in recency order (front is most recent), and a hash map from key to list iterator finds any entry instantly. `std::list::splice` moves a node to the front in O(1) without invalidating its iterator.
**Complexity:** O(1) average per operation, O(capacity) space.
```cpp
class LRUCache {
    int capacity;
    list<pair<int, int>> items;                                // (key, value); front = most recent
    unordered_map<int, list<pair<int, int>>::iterator> where;  // key -> its node in items

public:
    LRUCache(int capacity) : capacity(capacity) {}

    int get(int key) {
        auto it = where.find(key);
        if (it == where.end()) return -1;
        items.splice(items.begin(), items, it->second);        // mark as most recently used
        return it->second->second;
    }

    void put(int key, int value) {
        auto it = where.find(key);
        if (it != where.end()) {                               // update and refresh
            it->second->second = value;
            items.splice(items.begin(), items, it->second);
            return;
        }
        if ((int)items.size() == capacity) {                   // evict the least recently used
            where.erase(items.back().first);
            items.pop_back();
        }
        items.emplace_front(key, value);
        where[key] = items.begin();
    }
};
```
**Edge cases:** capacity 1; `put` on an existing key (update, no eviction); `get` on a missing key. Real-world versions: Redis can evict keys with an approximated LRU policy, and Rails' in-memory cache store prunes the least recently used entries when it grows too large.

## Common pitfalls
- Dereferencing `fast->next->next` without checking `fast && fast->next` first.
- Losing the rest of the list during reversal by overwriting `curr->next` before saving it.
- Forgetting the dummy head, then special-casing "the head changed" incorrectly.
- Comparing node values instead of node addresses in cycle detection (duplicate values are allowed).
- Returning `dummy` instead of `dummy.next`, or allocating the dummy with `new` and leaking it.
- Forgetting to cut the list (`slow->next = nullptr`) before reversing half of it, creating a cycle.
- Using a node after `delete`, or deleting nodes the caller still owns.
- In LRU, forgetting to erase the evicted key from the hash map, or storing raw positions that go stale.

## How to talk about this in an interview
- Draw boxes and arrows before writing code, and narrate each pointer change: "Save next, flip the arrow, move prev, move curr." Pointer manipulation is exactly what you walked students through in the Data Structures labs, so explain it the way you would at a lab bench.
- > **Fill in:** a pointer bug students commonly hit in your Data Structures lab (lost head, null dereference, leaked nodes) and how you taught them to avoid it.
- Connect LRU to your web work: page and fragment caches, Redis eviction policies, and API response caches all need "evict the least recently used entry". See [caching](../04-web-fundamentals/08-caching-and-performance.md).
- Be honest that in Ruby, TypeScript or Python you rarely write linked lists by hand; the point is reasoning about references, which shows up in object graphs, ActiveRecord associations and React state updates that must not mutate shared objects.

## Interview questions

**Q1. Array versus linked list: when would you choose each?**
Arrays give O(1) indexing, cache-friendly scans and binary search, but inserting or deleting at the front or middle is O(n). Linked lists give O(1) insert and delete when you already hold the neighboring node, never need a big contiguous block, and keep node addresses stable, but indexing is O(n) and pointer chasing is cache-unfriendly. Default to arrays; choose a list when you splice nodes you hold, as in an LRU cache.
*Follow-up:* Why is `vector` often faster even for middle inserts? Shifting contiguous memory is fast and cache-friendly, while finding the position in a list costs O(n) cache misses.

**Q2. How do you detect a cycle in a linked list?**
Floyd's tortoise and hare: slow moves one step, fast moves two. If fast reaches null there is no cycle; if they meet, there is one. O(n) time, O(1) space. A hash set of visited nodes also works with O(n) space.
*Follow-up:* How do you find where the cycle starts? Reset one pointer to the head after the meeting and advance both one step at a time; they meet at the entry.

**Q3. Why does Floyd's algorithm always meet, and in O(n)?**
Once both pointers are inside the cycle, fast closes the gap by exactly one node per step, so the gap reaches zero within one lap of the cycle. Slow enters the cycle after at most n steps, so the total is O(n).
*Follow-up:* Would moving fast three steps work? It still detects cycles, but the entry-finding math changes; two is the simple, standard choice.

**Q4. What is a dummy node and why use it?**
A sentinel placed before the real head so every real node has a predecessor. Deleting the head, merging and inserting at the front then need no special case. Allocate it on the stack and return `dummy.next`.
*Follow-up:* Where else do sentinels help? Doubly linked lists often use head and tail sentinels so insertion and removal never check for null.

**Q5. Why does an LRU cache need both a hash map and a doubly linked list?**
The map finds a key's node in O(1); the list keeps recency order and lets you move or remove that node in O(1). A singly linked list cannot remove a node in O(1) without its predecessor, which is why the list is doubly linked.
*Follow-up:* How would you make it thread-safe? A mutex around get and put, or sharded caches each with their own lock.

**Q6. How do you manage memory for linked structures in modern C++?**
Prefer `std::list` or `std::forward_list`. For custom nodes, use `unique_ptr` for the owning `next` pointer and a raw pointer or `weak_ptr` for `prev`, and destroy long chains iteratively to avoid deep recursive destructors. Delete unlinked nodes exactly once and never touch them afterwards.
*Follow-up:* What tool finds use-after-free bugs? AddressSanitizer or Valgrind.

**Q7. How would you find where two singly linked lists intersect?**
Walk two pointers; when one reaches the end, restart it at the other list's head. After at most n + m steps both have traveled the same distance and meet at the intersection node, or both reach null together. O(n + m) time, O(1) space (Intersection of Two Linked Lists, LeetCode 160).
*Follow-up:* The hash-set alternative? Store one list's nodes, then scan the other for the first hit, with O(n) space.

**Q8. How would you reverse a list in groups of k?**
Check that k nodes remain, reverse those k with the standard loop, connect the previous group's tail to the new group head, and continue; leave a final short group as is (Reverse Nodes in k-Group, LeetCode 25). O(n) time, O(1) space.
*Follow-up:* What makes it error-prone? Keeping track of the previous group's tail; a dummy head simplifies the first group.
