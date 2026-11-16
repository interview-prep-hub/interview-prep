# Trees and Binary Search Trees

> **Tier:** T1 · **Related:** [Recursion and backtracking](05-recursion-backtracking.md), [Heaps](08-heaps.md), [Graphs](09-graphs.md), [STL for DSA](../03-languages/c-cpp/04-stl-for-dsa.md), [Indexing and query optimization](../12-databases/05-indexing-and-query-optimization.md) · **Language:** C++17

## Quick revision
- A tree is a connected graph with no cycles: n nodes, n - 1 edges, exactly one path between any two nodes.
- **Depth** of a node = edges from the root down to it. **Height** = edges on the longest path from it down to a leaf. LeetCode's "maximum depth" counts nodes, so read each definition.
- Traversals: preorder (node, left, right), inorder (left, node, right), postorder (left, right, node), level order (BFS, one depth at a time).
- Inorder traversal of a BST yields sorted order.
- Recursive DFS uses O(h) stack: O(log n) when balanced, O(n) when skewed. BFS holds up to O(w) nodes, where the widest level can be about n/2.
- **Bottom-up (postorder)** problems: ask both children, combine, return one value to the parent, and keep a global best when the answer bends through a node (diameter, max path sum).
- **Top-down** problems pass information down as parameters (BST bounds, current depth).
- BST invariant: every value in the left subtree < node < every value in the right subtree (not just the direct children).
- BST search, insert and delete are O(h). Delete with two children: copy the inorder successor, then delete it from the right subtree.
- Balanced trees (AVL, red-black) keep h = O(log n). `std::map`/`std::set` are red-black trees in the major implementations.
- Trie: one node per prefix with `children[26]` and an end-of-word flag; insert and lookup in O(L).
- Signals: hierarchy, root-to-leaf paths, subtrees, ancestors, levels, "sorted order with inserts", prefixes.

## 1. Concept from scratch

### 1.1 Terminology
| Term | Meaning |
|---|---|
| Root, parent, child, leaf | top node; a node directly above or below another; a node with no children |
| Subtree | a node plus all of its descendants |
| Binary tree | every node has at most two children, left and right |
| Full | every node has 0 or 2 children |
| Complete | every level is full except possibly the last, which fills left to right (heaps use this shape) |
| Perfect | every level is completely full: 2^(h+1) - 1 nodes |
| Degenerate (skewed) | every node has one child: effectively a linked list, h = n - 1 |

LeetCode uses this node type:
```cpp
struct TreeNode {
    int val;
    TreeNode* left;
    TreeNode* right;
    TreeNode() : val(0), left(nullptr), right(nullptr) {}
    TreeNode(int x) : val(x), left(nullptr), right(nullptr) {}
    TreeNode(int x, TreeNode* left, TreeNode* right) : val(x), left(left), right(right) {}
};
```

### 1.2 Traversals, recursive and iterative
The three depth-first orders differ only in where you visit the node relative to the recursive calls. Iterative versions replace the call stack with an explicit `std::stack`, which matters when the tree can be very deep.
```cpp
#include <algorithm>
#include <stack>
#include <vector>
using namespace std;

void dfs(TreeNode* node, vector<int>& out) {
    if (!node) return;
    // out.push_back(node->val);   // visiting here gives preorder
    dfs(node->left, out);
    out.push_back(node->val);      // visiting here gives inorder
    dfs(node->right, out);
    // out.push_back(node->val);   // visiting here gives postorder
}

vector<int> preorderIterative(TreeNode* root) {
    vector<int> out;
    stack<TreeNode*> st;
    if (root) st.push(root);
    while (!st.empty()) {
        TreeNode* node = st.top(); st.pop();
        out.push_back(node->val);
        if (node->right) st.push(node->right);   // pushed first so the left child pops first
        if (node->left) st.push(node->left);
    }
    return out;
}

vector<int> inorderIterative(TreeNode* root) {
    vector<int> out;
    stack<TreeNode*> st;
    TreeNode* cur = root;
    while (cur || !st.empty()) {
        while (cur) { st.push(cur); cur = cur->left; }   // go as far left as possible
        cur = st.top(); st.pop();
        out.push_back(cur->val);
        cur = cur->right;
    }
    return out;
}

vector<int> postorderIterative(TreeNode* root) {   // collect node-right-left, then reverse
    vector<int> out;
    stack<TreeNode*> st;
    if (root) st.push(root);
    while (!st.empty()) {
        TreeNode* node = st.top(); st.pop();
        out.push_back(node->val);
        if (node->left) st.push(node->left);
        if (node->right) st.push(node->right);
    }
    reverse(out.begin(), out.end());
    return out;
}
```
Level order (BFS) processes the tree one depth at a time with a queue; it is problem 3.6 below.

### 1.3 DFS versus BFS on a tree
| | DFS (recursion or stack) | BFS (queue) |
|---|---|---|
| Memory | O(h) | O(w), the widest level |
| Natural for | paths, subtree properties, anything combining children | levels, "closest" or minimum depth (stop at the first leaf found) |
| Risk | stack overflow on very deep trees if recursive | large memory on very wide trees |

### 1.4 Height versus depth
Depth grows from the root downward (the root has depth 0); height grows from the leaves upward (a leaf has height 0) and the tree's height is the root's height. Many LeetCode problems count nodes instead of edges, so a single node has "maximum depth" 1. Confirm which one the interviewer means before you return off-by-one answers.

### 1.5 BST operations
Ordering lets you discard one subtree at every step, so search, insert and delete all cost O(h).
```cpp
TreeNode* searchBST(TreeNode* root, int key) {
    while (root && root->val != key) root = (key < root->val) ? root->left : root->right;
    return root;
}

TreeNode* insertBST(TreeNode* root, int key) {
    if (!root) return new TreeNode(key);
    if (key < root->val) root->left = insertBST(root->left, key);
    else if (key > root->val) root->right = insertBST(root->right, key);
    return root;                                         // duplicates ignored
}

TreeNode* deleteBST(TreeNode* root, int key) {
    if (!root) return nullptr;
    if (key < root->val) root->left = deleteBST(root->left, key);
    else if (key > root->val) root->right = deleteBST(root->right, key);
    else {
        if (!root->left || !root->right) {               // zero or one child: splice the node out
            TreeNode* child = root->left ? root->left : root->right;
            delete root;
            return child;
        }
        TreeNode* succ = root->right;                    // two children: smallest value on the right
        while (succ->left) succ = succ->left;
        root->val = succ->val;                           // copy it up...
        root->right = deleteBST(root->right, succ->val); // ...and delete it from the right subtree
    }
    return root;
}
```

### 1.6 Balanced trees
Insert sorted keys into a plain BST and you get a chain: h = n and every operation is O(n). **Self-balancing** trees rotate nodes after inserts and deletes to keep h = O(log n). AVL trees keep the two child heights within 1 of each other (stricter, slightly faster lookups); red-black trees use node colors to guarantee the height is at most about 2 log2(n), with fewer rotations on updates. `std::map`, `std::set` and their multi versions are red-black trees in the major standard libraries, which is why they guarantee O(log n) and iterate in sorted order. Databases use **B-trees and B+ trees**: each node holds many keys and fits a disk page, so a lookup touches only a few pages.

### 1.7 Tries
A trie (prefix tree) stores strings character by character: each node represents the prefix spelled on the path from the root, and a flag marks where a complete word ends. Insert, search and prefix checks cost O(L) for a string of length L, independent of how many words are stored. With an array of 26 children per node it is fast but memory-hungry; use a hash map of children for large alphabets. Uses: autocomplete, spell checking, longest-prefix matching in IP routing, and word games (Word Search II).

## 2. Core templates
```cpp
#include <algorithm>
using namespace std;

// Bottom-up (postorder): combine the children, update a global answer, return what the parent needs.
int heightAndBest(TreeNode* node, int& best) {
    if (!node) return 0;                          // empty subtree contributes nothing
    int l = heightAndBest(node->left, best);
    int r = heightAndBest(node->right, best);
    best = max(best, l + r);                      // an answer that bends through this node
    return 1 + max(l, r);                         // a parent can extend only one side
}

// Top-down: push constraints from the parent into the children.
bool withinBounds(TreeNode* node, long long low, long long high) {
    if (!node) return true;
    if (node->val <= low || node->val >= high) return false;
    return withinBounds(node->left, low, node->val) && withinBounds(node->right, node->val, high);
}
```

## 3. Problems

### 3.1 Maximum Depth of Binary Tree (LeetCode 104, Easy)
**Problem (paraphrased):** return the number of nodes on the longest path from the root down to a leaf.
**Approach:** this is the bottom-up template in its simplest form: depth is one plus the deeper child. BFS that counts levels works equally well and avoids deep recursion.
**Complexity:** O(n) time, O(h) stack.
```cpp
class Solution {
public:
    int maxDepth(TreeNode* root) {
        if (!root) return 0;
        return 1 + max(maxDepth(root->left), maxDepth(root->right));
    }
};
```
**Edge cases:** empty tree (0); a skewed tree (depth n, recursion depth n).

### 3.2 Invert Binary Tree (LeetCode 226, Easy)
**Problem (paraphrased):** mirror a binary tree by swapping the left and right children of every node.
**Approach:** visit every node once and swap its children; preorder, postorder and BFS all work.
**Complexity:** O(n) time, O(h) stack.
```cpp
class Solution {
public:
    TreeNode* invertTree(TreeNode* root) {
        if (!root) return nullptr;
        swap(root->left, root->right);
        invertTree(root->left);
        invertTree(root->right);
        return root;
    }
};
```
**Edge cases:** empty tree; a single node.

### 3.3 Same Tree (LeetCode 100, Easy)
**Problem (paraphrased):** decide whether two binary trees have the same shape and the same values at every position.
**Approach:** serializing both trees and comparing strings works but costs extra memory. Recursively: two empty trees match; one empty does not; otherwise the values and both pairs of subtrees must match.
**Complexity:** O(n) time, O(h) stack.
```cpp
class Solution {
public:
    bool isSameTree(TreeNode* p, TreeNode* q) {
        if (!p || !q) return p == q;            // both null: true; exactly one null: false
        return p->val == q->val && isSameTree(p->left, q->left) && isSameTree(p->right, q->right);
    }
};
```
**Edge cases:** same values in a different shape; both trees empty. Subtree of Another Tree (LeetCode 572) calls this at every node.

### 3.4 Diameter of Binary Tree (LeetCode 543, Easy)
**Problem (paraphrased):** return the number of edges on the longest path between any two nodes; the path need not pass through the root.
**Approach:** brute force computes left and right heights separately for every node, O(n^2) on skewed trees. In one postorder pass, each node's longest path through it is (left height + right height), measured in nodes along each downward chain; return the height to the parent and keep the best path in a reference variable.
**Complexity:** O(n) time, O(h) stack.
```cpp
class Solution {
public:
    int diameterOfBinaryTree(TreeNode* root) {
        int best = 0;
        height(root, best);
        return best;
    }
private:
    int height(TreeNode* node, int& best) {       // nodes on the longest downward chain from node
        if (!node) return 0;
        int l = height(node->left, best), r = height(node->right, best);
        best = max(best, l + r);                  // the path through node has l + r edges
        return 1 + max(l, r);
    }
};
```
**Edge cases:** a single node (0); the longest path entirely inside one subtree.

### 3.5 Balanced Binary Tree (LeetCode 110, Easy)
**Problem (paraphrased):** decide whether, at every node, the heights of the two subtrees differ by at most one.
**Approach:** brute force calls a height function at every node, O(n^2) on skewed trees (O(n log n) when balanced). Instead return the height from a postorder pass, using -1 as a signal that some subtree is already unbalanced, so the work is done once per node.
**Complexity:** O(n) time, O(h) stack.
```cpp
class Solution {
public:
    bool isBalanced(TreeNode* root) { return check(root) != -1; }
private:
    int check(TreeNode* node) {                   // height, or -1 if this subtree is unbalanced
        if (!node) return 0;
        int l = check(node->left);
        if (l == -1) return -1;                   // stop early
        int r = check(node->right);
        if (r == -1 || abs(l - r) > 1) return -1;
        return 1 + max(l, r);
    }
};
```
**Edge cases:** empty tree (balanced); a tree that is balanced at the root but not deeper down (the check is for every node).

### 3.6 Binary Tree Level Order Traversal (LeetCode 102, Medium)
**Problem (paraphrased):** return the node values grouped by level, from top to bottom and left to right within each level.
**Approach:** DFS with a depth parameter can append into `levels[depth]`, but BFS is the natural fit: the queue size at the start of each round is the number of nodes on the current level.
**Complexity:** O(n) time, O(w) space for the queue.
```cpp
class Solution {
public:
    vector<vector<int>> levelOrder(TreeNode* root) {
        vector<vector<int>> levels;
        if (!root) return levels;
        queue<TreeNode*> q;
        q.push(root);
        while (!q.empty()) {
            int size = q.size();                  // nodes on this level
            vector<int> level;
            for (int i = 0; i < size; i++) {
                TreeNode* node = q.front(); q.pop();
                level.push_back(node->val);
                if (node->left) q.push(node->left);
                if (node->right) q.push(node->right);
            }
            levels.push_back(move(level));
        }
        return levels;
    }
};
```
**Edge cases:** empty tree (empty result). Variations: zigzag order (reverse every other level), averages per level, minimum depth (return at the first leaf).

### 3.7 Binary Tree Right Side View (LeetCode 199, Medium)
**Problem (paraphrased):** standing to the right of the tree, list the values you see from top to bottom, which is the last node of each level.
**Approach:** BFS and take the last node of each level. A shorter DFS visits the right child first and records the first node it reaches at each new depth.
**Complexity:** O(n) time, O(h) stack.
```cpp
class Solution {
public:
    vector<int> rightSideView(TreeNode* root) {
        vector<int> view;
        dfs(root, 0, view);
        return view;
    }
private:
    void dfs(TreeNode* node, int depth, vector<int>& view) {
        if (!node) return;
        if (depth == (int)view.size()) view.push_back(node->val);   // first node seen at this depth
        dfs(node->right, depth + 1, view);                          // right side first
        dfs(node->left, depth + 1, view);
    }
};
```
**Edge cases:** a left-leaning tree, where some visible nodes are left children; empty tree.

### 3.8 Validate Binary Search Tree (LeetCode 98, Medium)
**Problem (paraphrased):** decide whether a binary tree satisfies the BST rule: every value in a node's left subtree is strictly smaller and every value in its right subtree strictly larger.
**Approach:** comparing each node only with its direct children is the classic wrong answer (a deep descendant can break the rule). Correct options: an inorder traversal must be strictly increasing, or pass down an open interval `(low, high)` that each node must fall inside. Use `long long` bounds because node values can be `INT_MIN` or `INT_MAX`.
**Complexity:** O(n) time, O(h) stack.
```cpp
class Solution {
public:
    bool isValidBST(TreeNode* root) { return valid(root, LLONG_MIN, LLONG_MAX); }
private:
    bool valid(TreeNode* node, long long low, long long high) {   // node->val must be in (low, high)
        if (!node) return true;
        if (node->val <= low || node->val >= high) return false;
        return valid(node->left, low, node->val) && valid(node->right, node->val, high);
    }
};
```
**Edge cases:** duplicates (invalid here); values at the `int` limits; a right-subtree grandchild smaller than the root.

### 3.9 Kth Smallest Element in a BST (LeetCode 230, Medium)
**Problem (paraphrased):** return the k-th smallest value stored in a BST (k is 1-indexed and valid).
**Approach:** brute force collects the full inorder traversal into a vector and returns element k - 1, O(n) time and space. An iterative inorder traversal can stop as soon as it has visited k nodes, O(h + k).
**Complexity:** O(h + k) time, O(h) space.
```cpp
class Solution {
public:
    int kthSmallest(TreeNode* root, int k) {
        stack<TreeNode*> st;
        TreeNode* cur = root;
        while (cur || !st.empty()) {
            while (cur) { st.push(cur); cur = cur->left; }   // smallest values are leftmost
            cur = st.top(); st.pop();
            if (--k == 0) return cur->val;                   // k-th node in sorted order
            cur = cur->right;
        }
        return -1;                                           // unreachable for a valid k
    }
};
```
**Edge cases:** k = 1 (leftmost node); k = n (rightmost). Follow-up with frequent inserts: store subtree sizes in each node to answer in O(h).

### 3.10 Lowest Common Ancestor of a Binary Tree (LeetCode 236, Medium)
**Problem (paraphrased):** given two nodes p and q in a binary tree (not necessarily a BST), return their deepest shared ancestor; a node counts as its own ancestor.
**Approach:** brute force records the root-to-node path for p and for q, then finds where the paths diverge: O(n) time with O(n) extra space and two passes. One postorder pass: return p or q when you meet one; if a node receives non-null results from both sides, p and q are split across it, so it is the LCA.
**Complexity:** O(n) time, O(h) stack.
```cpp
class Solution {
public:
    TreeNode* lowestCommonAncestor(TreeNode* root, TreeNode* p, TreeNode* q) {
        if (!root || root == p || root == q) return root;
        TreeNode* left = lowestCommonAncestor(root->left, p, q);
        TreeNode* right = lowestCommonAncestor(root->right, p, q);
        if (left && right) return root;           // p and q are on different sides
        return left ? left : right;               // both are on one side (or not found here)
    }
};
```
**Edge cases:** p is an ancestor of q (returns p); p and q in different subtrees of the root. For a BST (LeetCode 235), compare values and walk down from the root without recursion.

### 3.11 Construct Binary Tree from Preorder and Inorder Traversal (LeetCode 105, Medium)
**Problem (paraphrased):** rebuild a binary tree with unique values from its preorder and inorder traversals.
**Approach:** preorder's first value is the root; its position in inorder splits the left subtree from the right subtree. Searching inorder linearly for every root is O(n^2) on skewed trees. Store each value's inorder index in a hash map, and consume preorder with a shared index, building the left subtree before the right one to match preorder.
**Complexity:** O(n) time, O(n) space.
```cpp
class Solution {
public:
    TreeNode* buildTree(vector<int>& preorder, vector<int>& inorder) {
        for (int i = 0; i < (int)inorder.size(); i++) indexOf[inorder[i]] = i;   // values are unique
        int next = 0;
        return build(preorder, next, 0, (int)inorder.size() - 1);
    }
private:
    unordered_map<int, int> indexOf;

    TreeNode* build(const vector<int>& preorder, int& next, int lo, int hi) {   // inorder range [lo, hi]
        if (lo > hi) return nullptr;
        int rootVal = preorder[next++];                          // preorder visits the root first
        TreeNode* root = new TreeNode(rootVal);
        int mid = indexOf[rootVal];
        root->left = build(preorder, next, lo, mid - 1);         // left first, matching preorder
        root->right = build(preorder, next, mid + 1, hi);
        return root;
    }
};
```
**Edge cases:** a single node; fully left- or right-skewed trees. Preorder plus postorder alone cannot always identify a tree, because a lone child could be on either side.

### 3.12 Implement Trie (Prefix Tree) (LeetCode 208, Medium)
**Problem (paraphrased):** build a structure over lowercase words supporting insert, exact-word search and "does any word start with this prefix".
**Approach:** a hash set answers exact search in O(L) but needs O(n * L) to check a prefix against every word. A trie shares prefixes: each node has 26 child slots and an end-of-word flag. Owning `unique_ptr` children free the whole trie automatically.
**Complexity:** O(L) per operation, O(total characters x 26) space in the worst case.
```cpp
class Trie {
    struct Node {
        unique_ptr<Node> child[26];               // owning pointers
        bool isEnd = false;
    };
    unique_ptr<Node> root = make_unique<Node>();

    Node* walk(const string& s) const {           // follow s; nullptr if the path breaks off
        Node* cur = root.get();
        for (char c : s) {
            cur = cur->child[c - 'a'].get();
            if (!cur) return nullptr;
        }
        return cur;
    }
public:
    void insert(const string& word) {
        Node* cur = root.get();
        for (char c : word) {
            auto& next = cur->child[c - 'a'];
            if (!next) next = make_unique<Node>();
            cur = next.get();
        }
        cur->isEnd = true;
    }
    bool search(const string& word) const {
        Node* node = walk(word);
        return node && node->isEnd;               // the path exists and a word ends here
    }
    bool startsWith(const string& prefix) const { return walk(prefix) != nullptr; }
};
```
**Edge cases:** a word that is a prefix of another (`app` versus `apple`); searching a prefix that is not itself a word.

### 3.13 Binary Tree Maximum Path Sum (LeetCode 124, Hard)
**Problem (paraphrased):** a path is any sequence of connected nodes, each used at most once, not necessarily through the root. Return the largest possible sum of node values on a path (values can be negative).
**Approach:** brute force considers every pair of endpoints, O(n^2) paths. Use the bottom-up template: each node returns the best downward chain starting at it, dropping a child chain if it is negative. The best path that bends at a node is `val + leftGain + rightGain`; track the maximum in a reference variable, but return only one side to the parent.
**Complexity:** O(n) time, O(h) stack.
```cpp
class Solution {
public:
    int maxPathSum(TreeNode* root) {
        int best = INT_MIN;                        // at least one node must be chosen
        gain(root, best);
        return best;
    }
private:
    int gain(TreeNode* node, int& best) {          // best downward chain starting at node
        if (!node) return 0;
        int l = max(0, gain(node->left, best));    // ignore chains that would lower the sum
        int r = max(0, gain(node->right, best));
        best = max(best, node->val + l + r);       // a path that bends at node
        return node->val + max(l, r);              // a parent can extend only one side
    }
};
```
**Edge cases:** all negative values (the answer is the largest single node, which is why `best` starts at `INT_MIN`); a single node.

## Common pitfalls
- Validating a BST by comparing only parent and children; the rule applies to whole subtrees.
- Using `int` bounds in BST validation when node values can equal `INT_MIN` or `INT_MAX`.
- Confusing depth and height, or edges and nodes, and returning answers off by one.
- Recomputing heights at every node (O(n^2)) instead of returning them from one postorder pass.
- Returning the "bent" path value to the parent in diameter or max path sum problems; the parent can only extend one side.
- Recursion depth on skewed trees: a 1e5-node chain can overflow the stack; use iterative traversals.
- Forgetting the null checks before touching `node->left` or `node->right`.
- Leaking nodes in your own code (fine on LeetCode, not in production): give the tree an owner.

## How to talk about this in an interview
- Announce the traversal and direction: "This is bottom-up: each node needs its children's heights, so postorder." or "This is top-down: I'll pass the allowed range into each call." That framing is how you would explain tree recursion to students in a Data Structures lab, and it makes your code easy to follow.
- Before coding, draw a five-node tree and run your recursion on it out loud, including the null children.
- Real-world trees you have touched: the DOM and React's component tree, nested categories or comment threads in Rails apps, JSON payloads from third-party APIs, and PostgreSQL B-tree indexes behind every `WHERE id = ?` lookup (see [indexing](../12-databases/05-indexing-and-query-optimization.md)).
- > **Fill in:** a hierarchical feature you worked on (for example nested menus, categories, organizations within a multi-tenant platform) and how the hierarchy was stored and queried.

## Interview questions

**Q1. What is a balanced tree, and why does it matter?**
A tree whose height stays O(log n), so search, insert and delete stay O(log n). The interview definition for binary trees: at every node, the left and right subtree heights differ by at most one. Self-balancing trees such as AVL and red-black trees rotate nodes after updates to keep this property; without it, sorted inserts degrade a BST into a linked list with O(n) operations.
*Follow-up:* Which standard containers are balanced trees? `std::map`, `std::set` and their multi versions, typically red-black trees.

**Q2. BFS or DFS on a tree: which uses more memory?**
DFS uses O(h): small for balanced trees, O(n) for a chain. BFS uses O(w), the widest level, which is about n/2 for a perfect tree. So DFS is lighter on bushy trees and BFS is lighter on deep, thin ones.
*Follow-up:* Which finds the minimum depth faster? BFS, because it can stop at the first leaf it meets.

**Q3. Which traversal of a BST gives sorted output, and why?**
Inorder: it visits the entire left subtree (all smaller values), then the node, then the right subtree (all larger values), recursively. That also gives an O(n) BST validity check and k-th smallest queries.
*Follow-up:* Which traversal copies or serializes a tree naturally? Preorder (with null markers), because each node is written before its children.

**Q4. How do you delete a node from a BST?**
Find it. A leaf is removed; a node with one child is replaced by that child; a node with two children takes the value of its inorder successor (the smallest value in its right subtree), and that successor is then deleted from the right subtree, where it has at most one child. O(h) time.
*Follow-up:* Can you use the predecessor instead? Yes, the largest value in the left subtree works symmetrically.

**Q5. BST versus hash table for a dictionary: when would you use each?**
A hash table gives O(1) average lookups but no order. A balanced BST gives O(log n) guaranteed and supports sorted iteration, range queries, floor/ceiling and k-th element queries (with augmentation). Pick the BST when order or worst-case guarantees matter.
*Follow-up:* What does a database use for range queries? B+ tree indexes, whose leaves are linked in key order.

**Q6. When is a trie better than a hash set of words?**
When you need prefix queries (autocomplete, "any word starting with"), longest-prefix matching, or to share storage among many words with common prefixes. Lookups cost O(L) regardless of dictionary size. A hash set is simpler and lighter when you only need exact membership.
*Follow-up:* How do you cut a trie's memory? Use hash-map children, or compress single-child chains (a radix tree).

**Q7. What can go wrong with a recursive DFS on a tree of 1e5 nodes?**
If the tree is skewed, recursion depth reaches 1e5, which can overflow the default stack (often about 1 MB on Windows). Use an iterative traversal with an explicit stack, or increase the stack size where the platform allows.
*Follow-up:* Is this a real-world risk? Yes, with user-controlled nesting such as deeply nested JSON, comment threads or category trees.

**Q8. How do databases use trees for indexes?**
Most relational indexes are B+ trees: each node holds hundreds of keys and fits a disk page, so even millions of rows need only three or four levels. Leaves hold keys in sorted order and are linked, which makes range scans and `ORDER BY` on the indexed column efficient.
*Follow-up:* Why not a binary tree? It would be about 20 levels deep for a million rows, with one disk read per level.
