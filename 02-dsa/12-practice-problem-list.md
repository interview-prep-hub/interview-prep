# DSA Practice Problem List

> A curated set of about 125 LeetCode problems grouped by pattern, a "first 30" starter set, a six-week daily schedule, and a review system. Each pattern section links to the topic file that explains it, with C++17 solutions for the core problems. Related: [Big-O and problem solving](00-big-o-and-problem-solving.md), [STL for DSA](../03-languages/c-cpp/04-stl-for-dsa.md), [Interview formats](../00-start-here/03-interview-formats.md).

## How to use this list
- Learn the pattern first: read the topic file's concept and template sections, then solve that section's problems in order (they get harder).
- Time-box every attempt (Easy 15 to 20 minutes, Medium 30 to 40, Hard 45 to 60). When stuck past the box, read a hint or the approach, then close it and write the code yourself.
- Tick a box only when you have re-solved the problem cleanly, inside the time box, at the 7-day review (see [How to review](#how-to-review)).
- Solve out loud as if an interviewer were listening, using the seven-step framework from [Big-O and problem solving](00-big-o-and-problem-solving.md).
- Write C++ that would compile without your IDE's help: specific headers, `using namespace std;`, `long long` where sums can overflow.

## The first 30 to do
If you only have two weeks, do these in order. They cover the patterns that come up most in screening rounds.

| # | Problem | Difficulty | Pattern |
|---|---|---|---|
| 1 | [Two Sum](https://leetcode.com/problems/two-sum/) | Easy | hash map |
| 217 | [Contains Duplicate](https://leetcode.com/problems/contains-duplicate/) | Easy | hash set |
| 242 | [Valid Anagram](https://leetcode.com/problems/valid-anagram/) | Easy | counting |
| 49 | [Group Anagrams](https://leetcode.com/problems/group-anagrams/) | Medium | hash map grouping |
| 347 | [Top K Frequent Elements](https://leetcode.com/problems/top-k-frequent-elements/) | Medium | bucket sort / heap |
| 238 | [Product of Array Except Self](https://leetcode.com/problems/product-of-array-except-self/) | Medium | prefix/suffix |
| 128 | [Longest Consecutive Sequence](https://leetcode.com/problems/longest-consecutive-sequence/) | Medium | hash set |
| 125 | [Valid Palindrome](https://leetcode.com/problems/valid-palindrome/) | Easy | two pointers |
| 167 | [Two Sum II - Input Array Is Sorted](https://leetcode.com/problems/two-sum-ii-input-array-is-sorted/) | Medium | two pointers |
| 15 | [3Sum](https://leetcode.com/problems/3sum/) | Medium | sort + two pointers |
| 11 | [Container With Most Water](https://leetcode.com/problems/container-with-most-water/) | Medium | two pointers |
| 121 | [Best Time to Buy and Sell Stock](https://leetcode.com/problems/best-time-to-buy-and-sell-stock/) | Easy | running minimum |
| 3 | [Longest Substring Without Repeating Characters](https://leetcode.com/problems/longest-substring-without-repeating-characters/) | Medium | sliding window |
| 20 | [Valid Parentheses](https://leetcode.com/problems/valid-parentheses/) | Easy | stack |
| 155 | [Min Stack](https://leetcode.com/problems/min-stack/) | Medium | stack design |
| 739 | [Daily Temperatures](https://leetcode.com/problems/daily-temperatures/) | Medium | monotonic stack |
| 206 | [Reverse Linked List](https://leetcode.com/problems/reverse-linked-list/) | Easy | pointer reversal |
| 21 | [Merge Two Sorted Lists](https://leetcode.com/problems/merge-two-sorted-lists/) | Easy | dummy head |
| 141 | [Linked List Cycle](https://leetcode.com/problems/linked-list-cycle/) | Easy | fast/slow pointers |
| 704 | [Binary Search](https://leetcode.com/problems/binary-search/) | Easy | binary search |
| 33 | [Search in Rotated Sorted Array](https://leetcode.com/problems/search-in-rotated-sorted-array/) | Medium | binary search |
| 104 | [Maximum Depth of Binary Tree](https://leetcode.com/problems/maximum-depth-of-binary-tree/) | Easy | tree DFS |
| 226 | [Invert Binary Tree](https://leetcode.com/problems/invert-binary-tree/) | Easy | tree DFS |
| 102 | [Binary Tree Level Order Traversal](https://leetcode.com/problems/binary-tree-level-order-traversal/) | Medium | tree BFS |
| 98 | [Validate Binary Search Tree](https://leetcode.com/problems/validate-binary-search-tree/) | Medium | BST bounds |
| 200 | [Number of Islands](https://leetcode.com/problems/number-of-islands/) | Medium | grid DFS/BFS |
| 207 | [Course Schedule](https://leetcode.com/problems/course-schedule/) | Medium | topological sort |
| 70 | [Climbing Stairs](https://leetcode.com/problems/climbing-stairs/) | Easy | 1D DP |
| 198 | [House Robber](https://leetcode.com/problems/house-robber/) | Medium | 1D DP |
| 322 | [Coin Change](https://leetcode.com/problems/coin-change/) | Medium | unbounded knapsack DP |

## The full list by pattern

### Arrays and hashing (see [topic file](01-arrays-strings-hashing.md))
| Done | # | Problem | Difficulty |
|---|---|---|---|
| [ ] | 1 | [Two Sum](https://leetcode.com/problems/two-sum/) | Easy |
| [ ] | 217 | [Contains Duplicate](https://leetcode.com/problems/contains-duplicate/) | Easy |
| [ ] | 242 | [Valid Anagram](https://leetcode.com/problems/valid-anagram/) | Easy |
| [ ] | 169 | [Majority Element](https://leetcode.com/problems/majority-element/) | Easy |
| [ ] | 49 | [Group Anagrams](https://leetcode.com/problems/group-anagrams/) | Medium |
| [ ] | 347 | [Top K Frequent Elements](https://leetcode.com/problems/top-k-frequent-elements/) | Medium |
| [ ] | 238 | [Product of Array Except Self](https://leetcode.com/problems/product-of-array-except-self/) | Medium |
| [ ] | 36 | [Valid Sudoku](https://leetcode.com/problems/valid-sudoku/) | Medium |
| [ ] | 128 | [Longest Consecutive Sequence](https://leetcode.com/problems/longest-consecutive-sequence/) | Medium |
| [ ] | 560 | [Subarray Sum Equals K](https://leetcode.com/problems/subarray-sum-equals-k/) | Medium |

### Two pointers (see [topic file](02-two-pointers-sliding-window.md))
| Done | # | Problem | Difficulty |
|---|---|---|---|
| [ ] | 125 | [Valid Palindrome](https://leetcode.com/problems/valid-palindrome/) | Easy |
| [ ] | 283 | [Move Zeroes](https://leetcode.com/problems/move-zeroes/) | Easy |
| [ ] | 167 | [Two Sum II - Input Array Is Sorted](https://leetcode.com/problems/two-sum-ii-input-array-is-sorted/) | Medium |
| [ ] | 15 | [3Sum](https://leetcode.com/problems/3sum/) | Medium |
| [ ] | 11 | [Container With Most Water](https://leetcode.com/problems/container-with-most-water/) | Medium |
| [ ] | 42 | [Trapping Rain Water](https://leetcode.com/problems/trapping-rain-water/) | Hard |

### Sliding window (see [topic file](02-two-pointers-sliding-window.md))
| Done | # | Problem | Difficulty |
|---|---|---|---|
| [ ] | 121 | [Best Time to Buy and Sell Stock](https://leetcode.com/problems/best-time-to-buy-and-sell-stock/) | Easy |
| [ ] | 3 | [Longest Substring Without Repeating Characters](https://leetcode.com/problems/longest-substring-without-repeating-characters/) | Medium |
| [ ] | 424 | [Longest Repeating Character Replacement](https://leetcode.com/problems/longest-repeating-character-replacement/) | Medium |
| [ ] | 567 | [Permutation in String](https://leetcode.com/problems/permutation-in-string/) | Medium |
| [ ] | 209 | [Minimum Size Subarray Sum](https://leetcode.com/problems/minimum-size-subarray-sum/) | Medium |
| [ ] | 438 | [Find All Anagrams in a String](https://leetcode.com/problems/find-all-anagrams-in-a-string/) | Medium |
| [ ] | 76 | [Minimum Window Substring](https://leetcode.com/problems/minimum-window-substring/) | Hard |

### Stacks and queues (see [topic file](04-stacks-queues.md))
| Done | # | Problem | Difficulty |
|---|---|---|---|
| [ ] | 20 | [Valid Parentheses](https://leetcode.com/problems/valid-parentheses/) | Easy |
| [ ] | 232 | [Implement Queue using Stacks](https://leetcode.com/problems/implement-queue-using-stacks/) | Easy |
| [ ] | 496 | [Next Greater Element I](https://leetcode.com/problems/next-greater-element-i/) | Easy |
| [ ] | 155 | [Min Stack](https://leetcode.com/problems/min-stack/) | Medium |
| [ ] | 150 | [Evaluate Reverse Polish Notation](https://leetcode.com/problems/evaluate-reverse-polish-notation/) | Medium |
| [ ] | 739 | [Daily Temperatures](https://leetcode.com/problems/daily-temperatures/) | Medium |
| [ ] | 394 | [Decode String](https://leetcode.com/problems/decode-string/) | Medium |
| [ ] | 735 | [Asteroid Collision](https://leetcode.com/problems/asteroid-collision/) | Medium |
| [ ] | 239 | [Sliding Window Maximum](https://leetcode.com/problems/sliding-window-maximum/) | Hard |
| [ ] | 84 | [Largest Rectangle in Histogram](https://leetcode.com/problems/largest-rectangle-in-histogram/) | Hard |

### Linked lists (see [topic file](03-linked-lists.md))
| Done | # | Problem | Difficulty |
|---|---|---|---|
| [ ] | 206 | [Reverse Linked List](https://leetcode.com/problems/reverse-linked-list/) | Easy |
| [ ] | 21 | [Merge Two Sorted Lists](https://leetcode.com/problems/merge-two-sorted-lists/) | Easy |
| [ ] | 141 | [Linked List Cycle](https://leetcode.com/problems/linked-list-cycle/) | Easy |
| [ ] | 876 | [Middle of the Linked List](https://leetcode.com/problems/middle-of-the-linked-list/) | Easy |
| [ ] | 234 | [Palindrome Linked List](https://leetcode.com/problems/palindrome-linked-list/) | Easy |
| [ ] | 160 | [Intersection of Two Linked Lists](https://leetcode.com/problems/intersection-of-two-linked-lists/) | Easy |
| [ ] | 142 | [Linked List Cycle II](https://leetcode.com/problems/linked-list-cycle-ii/) | Medium |
| [ ] | 19 | [Remove Nth Node From End of List](https://leetcode.com/problems/remove-nth-node-from-end-of-list/) | Medium |
| [ ] | 143 | [Reorder List](https://leetcode.com/problems/reorder-list/) | Medium |
| [ ] | 2 | [Add Two Numbers](https://leetcode.com/problems/add-two-numbers/) | Medium |
| [ ] | 138 | [Copy List with Random Pointer](https://leetcode.com/problems/copy-list-with-random-pointer/) | Medium |
| [ ] | 146 | [LRU Cache](https://leetcode.com/problems/lru-cache/) | Medium |

### Binary search and sorting (see [topic file](06-sorting-binary-search.md))
| Done | # | Problem | Difficulty |
|---|---|---|---|
| [ ] | 704 | [Binary Search](https://leetcode.com/problems/binary-search/) | Easy |
| [ ] | 35 | [Search Insert Position](https://leetcode.com/problems/search-insert-position/) | Easy |
| [ ] | 34 | [Find First and Last Position of Element in Sorted Array](https://leetcode.com/problems/find-first-and-last-position-of-element-in-sorted-array/) | Medium |
| [ ] | 33 | [Search in Rotated Sorted Array](https://leetcode.com/problems/search-in-rotated-sorted-array/) | Medium |
| [ ] | 153 | [Find Minimum in Rotated Sorted Array](https://leetcode.com/problems/find-minimum-in-rotated-sorted-array/) | Medium |
| [ ] | 74 | [Search a 2D Matrix](https://leetcode.com/problems/search-a-2d-matrix/) | Medium |
| [ ] | 875 | [Koko Eating Bananas](https://leetcode.com/problems/koko-eating-bananas/) | Medium |
| [ ] | 1011 | [Capacity To Ship Packages Within D Days](https://leetcode.com/problems/capacity-to-ship-packages-within-d-days/) | Medium |
| [ ] | 75 | [Sort Colors](https://leetcode.com/problems/sort-colors/) | Medium |
| [ ] | 215 | [Kth Largest Element in an Array](https://leetcode.com/problems/kth-largest-element-in-an-array/) | Medium |
| [ ] | 912 | [Sort an Array](https://leetcode.com/problems/sort-an-array/) | Medium |
| [ ] | 4 | [Median of Two Sorted Arrays](https://leetcode.com/problems/median-of-two-sorted-arrays/) | Hard |

### Recursion and backtracking (see [topic file](05-recursion-backtracking.md))
| Done | # | Problem | Difficulty |
|---|---|---|---|
| [ ] | 50 | [Pow(x, n)](https://leetcode.com/problems/powx-n/) | Medium |
| [ ] | 78 | [Subsets](https://leetcode.com/problems/subsets/) | Medium |
| [ ] | 90 | [Subsets II](https://leetcode.com/problems/subsets-ii/) | Medium |
| [ ] | 46 | [Permutations](https://leetcode.com/problems/permutations/) | Medium |
| [ ] | 39 | [Combination Sum](https://leetcode.com/problems/combination-sum/) | Medium |
| [ ] | 40 | [Combination Sum II](https://leetcode.com/problems/combination-sum-ii/) | Medium |
| [ ] | 17 | [Letter Combinations of a Phone Number](https://leetcode.com/problems/letter-combinations-of-a-phone-number/) | Medium |
| [ ] | 22 | [Generate Parentheses](https://leetcode.com/problems/generate-parentheses/) | Medium |
| [ ] | 79 | [Word Search](https://leetcode.com/problems/word-search/) | Medium |
| [ ] | 51 | [N-Queens](https://leetcode.com/problems/n-queens/) | Hard |

### Trees, BSTs and tries (see [topic file](07-trees-bst.md))
| Done | # | Problem | Difficulty |
|---|---|---|---|
| [ ] | 104 | [Maximum Depth of Binary Tree](https://leetcode.com/problems/maximum-depth-of-binary-tree/) | Easy |
| [ ] | 226 | [Invert Binary Tree](https://leetcode.com/problems/invert-binary-tree/) | Easy |
| [ ] | 100 | [Same Tree](https://leetcode.com/problems/same-tree/) | Easy |
| [ ] | 543 | [Diameter of Binary Tree](https://leetcode.com/problems/diameter-of-binary-tree/) | Easy |
| [ ] | 110 | [Balanced Binary Tree](https://leetcode.com/problems/balanced-binary-tree/) | Easy |
| [ ] | 102 | [Binary Tree Level Order Traversal](https://leetcode.com/problems/binary-tree-level-order-traversal/) | Medium |
| [ ] | 199 | [Binary Tree Right Side View](https://leetcode.com/problems/binary-tree-right-side-view/) | Medium |
| [ ] | 98 | [Validate Binary Search Tree](https://leetcode.com/problems/validate-binary-search-tree/) | Medium |
| [ ] | 230 | [Kth Smallest Element in a BST](https://leetcode.com/problems/kth-smallest-element-in-a-bst/) | Medium |
| [ ] | 236 | [Lowest Common Ancestor of a Binary Tree](https://leetcode.com/problems/lowest-common-ancestor-of-a-binary-tree/) | Medium |
| [ ] | 105 | [Construct Binary Tree from Preorder and Inorder Traversal](https://leetcode.com/problems/construct-binary-tree-from-preorder-and-inorder-traversal/) | Medium |
| [ ] | 208 | [Implement Trie (Prefix Tree)](https://leetcode.com/problems/implement-trie-prefix-tree/) | Medium |
| [ ] | 211 | [Design Add and Search Words Data Structure](https://leetcode.com/problems/design-add-and-search-words-data-structure/) | Medium |
| [ ] | 124 | [Binary Tree Maximum Path Sum](https://leetcode.com/problems/binary-tree-maximum-path-sum/) | Hard |

### Heaps (see [topic file](08-heaps.md))
| Done | # | Problem | Difficulty |
|---|---|---|---|
| [ ] | 703 | [Kth Largest Element in a Stream](https://leetcode.com/problems/kth-largest-element-in-a-stream/) | Easy |
| [ ] | 1046 | [Last Stone Weight](https://leetcode.com/problems/last-stone-weight/) | Easy |
| [ ] | 973 | [K Closest Points to Origin](https://leetcode.com/problems/k-closest-points-to-origin/) | Medium |
| [ ] | 621 | [Task Scheduler](https://leetcode.com/problems/task-scheduler/) | Medium |
| [ ] | 767 | [Reorganize String](https://leetcode.com/problems/reorganize-string/) | Medium |
| [ ] | 23 | [Merge k Sorted Lists](https://leetcode.com/problems/merge-k-sorted-lists/) | Hard |
| [ ] | 295 | [Find Median from Data Stream](https://leetcode.com/problems/find-median-from-data-stream/) | Hard |

### Graphs (see [topic file](09-graphs.md))
| Done | # | Problem | Difficulty |
|---|---|---|---|
| [ ] | 200 | [Number of Islands](https://leetcode.com/problems/number-of-islands/) | Medium |
| [ ] | 695 | [Max Area of Island](https://leetcode.com/problems/max-area-of-island/) | Medium |
| [ ] | 133 | [Clone Graph](https://leetcode.com/problems/clone-graph/) | Medium |
| [ ] | 994 | [Rotting Oranges](https://leetcode.com/problems/rotting-oranges/) | Medium |
| [ ] | 417 | [Pacific Atlantic Water Flow](https://leetcode.com/problems/pacific-atlantic-water-flow/) | Medium |
| [ ] | 547 | [Number of Provinces](https://leetcode.com/problems/number-of-provinces/) | Medium |
| [ ] | 207 | [Course Schedule](https://leetcode.com/problems/course-schedule/) | Medium |
| [ ] | 210 | [Course Schedule II](https://leetcode.com/problems/course-schedule-ii/) | Medium |
| [ ] | 684 | [Redundant Connection](https://leetcode.com/problems/redundant-connection/) | Medium |
| [ ] | 743 | [Network Delay Time](https://leetcode.com/problems/network-delay-time/) | Medium |
| [ ] | 127 | [Word Ladder](https://leetcode.com/problems/word-ladder/) | Hard |

### Dynamic programming (see [topic file](10-dynamic-programming.md))
| Done | # | Problem | Difficulty |
|---|---|---|---|
| [ ] | 70 | [Climbing Stairs](https://leetcode.com/problems/climbing-stairs/) | Easy |
| [ ] | 198 | [House Robber](https://leetcode.com/problems/house-robber/) | Medium |
| [ ] | 213 | [House Robber II](https://leetcode.com/problems/house-robber-ii/) | Medium |
| [ ] | 53 | [Maximum Subarray](https://leetcode.com/problems/maximum-subarray/) | Medium |
| [ ] | 322 | [Coin Change](https://leetcode.com/problems/coin-change/) | Medium |
| [ ] | 300 | [Longest Increasing Subsequence](https://leetcode.com/problems/longest-increasing-subsequence/) | Medium |
| [ ] | 139 | [Word Break](https://leetcode.com/problems/word-break/) | Medium |
| [ ] | 91 | [Decode Ways](https://leetcode.com/problems/decode-ways/) | Medium |
| [ ] | 62 | [Unique Paths](https://leetcode.com/problems/unique-paths/) | Medium |
| [ ] | 1143 | [Longest Common Subsequence](https://leetcode.com/problems/longest-common-subsequence/) | Medium |
| [ ] | 416 | [Partition Equal Subset Sum](https://leetcode.com/problems/partition-equal-subset-sum/) | Medium |
| [ ] | 5 | [Longest Palindromic Substring](https://leetcode.com/problems/longest-palindromic-substring/) | Medium |
| [ ] | 72 | [Edit Distance](https://leetcode.com/problems/edit-distance/) | Medium |

### Greedy and intervals (see [topic file](11-greedy-intervals-bits.md))
| Done | # | Problem | Difficulty |
|---|---|---|---|
| [ ] | 55 | [Jump Game](https://leetcode.com/problems/jump-game/) | Medium |
| [ ] | 45 | [Jump Game II](https://leetcode.com/problems/jump-game-ii/) | Medium |
| [ ] | 134 | [Gas Station](https://leetcode.com/problems/gas-station/) | Medium |
| [ ] | 763 | [Partition Labels](https://leetcode.com/problems/partition-labels/) | Medium |
| [ ] | 56 | [Merge Intervals](https://leetcode.com/problems/merge-intervals/) | Medium |
| [ ] | 57 | [Insert Interval](https://leetcode.com/problems/insert-interval/) | Medium |
| [ ] | 435 | [Non-overlapping Intervals](https://leetcode.com/problems/non-overlapping-intervals/) | Medium |
| [ ] | 452 | [Minimum Number of Arrows to Burst Balloons](https://leetcode.com/problems/minimum-number-of-arrows-to-burst-balloons/) | Medium |

### Bit manipulation (see [topic file](11-greedy-intervals-bits.md))
| Done | # | Problem | Difficulty |
|---|---|---|---|
| [ ] | 136 | [Single Number](https://leetcode.com/problems/single-number/) | Easy |
| [ ] | 191 | [Number of 1 Bits](https://leetcode.com/problems/number-of-1-bits/) | Easy |
| [ ] | 338 | [Counting Bits](https://leetcode.com/problems/counting-bits/) | Easy |
| [ ] | 268 | [Missing Number](https://leetcode.com/problems/missing-number/) | Easy |
| [ ] | 371 | [Sum of Two Integers](https://leetcode.com/problems/sum-of-two-integers/) | Medium |

## Six-week daily schedule
About three new problems a day, six days a week, with the seventh day for a mock interview and reviews. Every day also includes the reviews that fall due under the 1-3-7 rule below (budget 30 to 45 minutes for them). If you are working full-time, stretch each week to ten days: the order matters more than the pace.

### Week 1: foundations, hashing, two pointers
| Day | Focus | New problems |
|---|---|---|
| 1 | Read [Big-O](00-big-o-and-problem-solving.md) and the [hashing](01-arrays-strings-hashing.md) concepts | 1, 217, 242 |
| 2 | Hash map grouping and counting | 49, 347, 238 |
| 3 | Hash sets and prefix sums | 128, 560, 169 |
| 4 | Grid hashing, opposite-end pointers | 36, 125, 167 |
| 5 | Sorting plus two pointers | 15, 11, 283 |
| 6 | Hard two pointers, first windows | 42, 121, 3 |
| 7 | Mock: two random problems from the week, 45 minutes each, out loud; then reviews | none |

### Week 2: sliding window, stacks, start of linked lists
| Day | Focus | New problems |
|---|---|---|
| 8 | Variable windows | 424, 567, 209 |
| 9 | Window with counts | 438, 76 |
| 10 | Stack basics and design | 20, 155, 150, 232 |
| 11 | Monotonic stack | 739, 496, 394 |
| 12 | Stack simulation, deque, histogram | 735, 239, 84 |
| 13 | Linked list fundamentals | 206, 21, 141, 142 |
| 14 | Mock plus reviews | none |

### Week 3: linked lists, binary search, sorting
| Day | Focus | New problems |
|---|---|---|
| 15 | Fast/slow pointers and dummy heads | 876, 19, 143, 2 |
| 16 | Linked list combinations and design | 234, 138, 146, 160 |
| 17 | Binary search templates | 704, 35, 34 |
| 18 | Rotated arrays and matrices | 33, 153, 74 |
| 19 | Binary search on the answer, hard partition | 875, 1011, 4 |
| 20 | Sorting: implement merge sort and quick sort by hand for 912 | 75, 215, 912 |
| 21 | Mock plus reviews | none |

### Week 4: backtracking and trees
| Day | Focus | New problems |
|---|---|---|
| 22 | Recursion and subsets | 50, 78, 90 |
| 23 | Permutations and combinations | 46, 39, 40 |
| 24 | Constrained generation and grids | 17, 22, 79, 51 |
| 25 | Tree DFS basics | 104, 226, 100, 543 |
| 26 | Balance, BFS, BST bounds | 110, 102, 199, 98 |
| 27 | BST order, ancestors, construction | 230, 236, 105 |
| 28 | Mock plus reviews | none |

### Week 5: hard trees, tries, heaps, graphs
| Day | Focus | New problems |
|---|---|---|
| 29 | Path sums and tries | 124, 208, 211 |
| 30 | Heap basics (redo 347 with a heap) | 703, 1046, 973, 347 |
| 31 | Greedy heaps, k-way merge, two heaps | 621, 767, 23, 295 |
| 32 | Grid and graph traversal | 200, 695, 133 |
| 33 | Multi-source BFS, reverse search, topological sort | 994, 417, 207, 210 |
| 34 | Union-find, Dijkstra, BFS on words | 684, 547, 743, 127 |
| 35 | Mock plus reviews | none |

### Week 6: dynamic programming, greedy, bits, final mocks
| Day | Focus | New problems |
|---|---|---|
| 36 | 1D DP | 70, 198, 213, 53 |
| 37 | Knapsack-style and sequence DP | 322, 300, 139, 91 |
| 38 | Grid and two-string DP | 62, 1143, 416, 72, 5 |
| 39 | Greedy | 55, 45, 134, 763 |
| 40 | Intervals | 56, 57, 435, 452 |
| 41 | Bit manipulation | 136, 191, 338, 268, 371 |
| 42 | Final mock, re-read your mistake log and every topic file's quick revision | none |

## How to review
- **The 1-3-7 rule.** After you first solve (or first learn) a problem, re-solve it from a blank editor one day later, three days later and seven days later. Do not look at your old code. If a review fails, restart that problem's schedule from day one.
- **Time every attempt.** Targets: Easy 15 minutes, Medium 30 to 35, Hard 45. Being correct but slow still counts as a miss, because interviews are timed.
- **Grade yourself.** A: clean and on time. B: needed a hint, had a bug you found only by running, or went over time. C: could not solve it. Only an A at the 7-day review ticks the box.
- **Keep a mistake log.** One line per problem: date, number, pattern, what went wrong, and the key insight in one sentence ("prefix sums because negatives break the window"). Re-read it weekly and the night before interviews.
- **Re-explain, not just re-type.** During reviews, say the approach and complexity out loud before coding, exactly as you would in a live round.

## Online assessments (HackerRank-style tests)
Many companies, including many software houses in Pakistan, screen with a timed online test before any interview; formats vary, so ask the recruiter what to expect (see [interview formats](../00-start-here/03-interview-formats.md)).
- **Typical shape:** 60 to 120 minutes, two to four coding problems, sometimes multiple-choice questions on CS fundamentals or SQL. Hidden test cases decide the score, often with partial credit per test case.
- **Input and output:** depending on the platform and the test, you either complete a given function or write a full program that reads standard input and prints to standard output. Practice both.
- **Strategy:** read every problem first, solve the easiest first, and if the optimal idea is not coming, submit a correct brute force to bank partial points before optimizing. Check the constraints to pick the target complexity.
- **Before submitting:** test n = 1, empty or minimal input, maximum sizes (think about overflow and time), negative numbers and duplicates. Remove debug prints; extra output fails test cases.
- **Environment:** confirm the selected language version (C++17 or newer), and expect webcam proctoring, tab-switch detection or disabled pasting on some platforms. Treat it like an exam.

A full-program skeleton for stdin/stdout tests:
```cpp
#include <iostream>
#include <vector>
using namespace std;

int main() {
    ios::sync_with_stdio(false);   // faster iostreams for large inputs
    cin.tie(nullptr);

    int n;
    if (!(cin >> n)) return 0;
    vector<long long> a(n);
    for (auto& x : a) cin >> x;

    long long sum = 0;             // long long: n values up to 1e9 overflow int
    for (long long x : a) sum += x;
    cout << sum << '\n';           // '\n' instead of endl: no flush on every line
    return 0;
}
```
`#include <bits/stdc++.h>` works on GCC-based judges (most of them) but not with MSVC, so do not rely on it in a shared editor or a local Visual Studio setup.

## Pairing with an interviewer (live coding)
- **Expect a shared editor** that may not compile or run your code. Write carefully, keep functions short, and test by tracing examples by hand.
- **Clarify before coding:** restate the problem, ask about constraints and edge cases, and confirm the expected output format.
- **Think aloud the whole time.** Give the brute force and its complexity, then the improvement, and ask "Does this approach sound good before I code it?"
- **Treat hints as collaboration.** When the interviewer nudges, acknowledge it, connect it to your plan, and continue; do not argue or restart from scratch.
- **Manage the clock:** a working, clearly explained solution beats a half-finished optimal one. If time is short, describe how you would finish.
- **Close well:** walk through a test case, state the final time and space complexity, and mention one improvement or trade-off (memory versus speed, iterative versus recursive).
- Your PUCIT teaching-assistant experience is relevant here: explaining a solution to a student and explaining it to an interviewer are the same skill. Use it.
