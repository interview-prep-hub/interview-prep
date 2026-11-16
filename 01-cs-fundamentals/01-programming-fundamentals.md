# Programming Fundamentals

> **Tier:** T2 (working knowledge) · **Related:** [OOP](02-oop.md), [Operating systems](05-operating-systems.md), [C++ fundamentals](../03-languages/c-cpp/02-cpp-fundamentals.md), [C essentials](../03-languages/c-cpp/01-c-essentials.md), [Big-O and problem solving](../02-dsa/00-big-o-and-problem-solving.md), [Recursion and backtracking](../02-dsa/05-recursion-backtracking.md), [Dynamic programming](../02-dsa/10-dynamic-programming.md), [Ruby basics](../03-languages/ruby/01-basics.md)
> **On your resume:** C, C++, Java, C#, Python, Ruby and TypeScript in your skills list, plus two years as a PUCIT teaching assistant for the Programming Fundamentals lab.

## Quick revision
- Ahead-of-time compiled (C, C++): native code before running. Interpreted (CRuby, CPython): bytecode run by a VM. JIT (JVM, .NET, V8, Ruby YJIT): hot code compiled to native code at runtime.
- Static typing checks types at compile time (C++, Java, TypeScript); dynamic typing at runtime (Ruby, Python, JavaScript). Strong vs weak describes how freely a language converts types implicitly.
- Typical 64-bit sizes: `char` 1, `int` 4, `long long` 8, `double` 8, pointer 8 bytes. `long` is 4 bytes on Windows and 8 on Linux, so use `<cstdint>` when width matters.
- Signed overflow in C++ is undefined behavior; unsigned arithmetic wraps modulo 2^n. Ruby and Python integers grow without overflow; JavaScript numbers are doubles, exact only up to 2^53 - 1.
- Integer division truncates toward zero in C++, Java and C# (`-7 / 2 == -3`); Ruby and Python floor (`-4`).
- Pass by value copies, pass by reference aliases, pass by pointer passes a (nullable) address. Ruby, JavaScript, Python and Java objects are passed as references by value ("pass by sharing").
- Scope is where a name is visible; lifetime is how long the object exists. Never return a reference or pointer to a local.
- Stack: call frames and automatic variables, fast and small. Heap: dynamic allocations, large, freed manually (`delete`, smart pointers) or by a garbage collector. Every recursive call pushes a frame, so a missing base case overflows the stack.
- Recursion = base case plus a recursive step that shrinks the input. Time comes from the recurrence (calls per level times work per call); stack space equals the maximum depth. Naive `fib` is O(2^n) time but O(n) space; memoization makes it O(n). Tail-call optimization is not guaranteed in C++, CRuby, Python or V8, so deep recursion needs an explicit stack.
- Arrays are contiguous: O(1) indexing, O(n) insertion in the middle. `std::vector::push_back` is amortized O(1).
- Exceptions unwind the stack (RAII cleans up); return codes and `std::optional` make failure explicit at each call site.
- Build pipeline: preprocess, compile, assemble, link. "Undefined reference" is a linker error.
- Debugging loop: reproduce, read the stack trace, isolate, inspect state, fix the cause, add a test.

## 1. Compiled vs interpreted vs JIT

| Model | How code runs | Examples | Trade-offs |
|---|---|---|---|
| Ahead-of-time compiled | source to native machine code before running | C, C++, Go, Rust | fast execution, errors caught early; rebuild per platform |
| Bytecode plus interpreter | source to bytecode, executed by a virtual machine | CRuby (YARV), CPython | portable and flexible; slower per operation |
| JIT compiled | the VM profiles hot code and compiles it to machine code at runtime | JVM (Java), .NET CLR (C#), V8 (JavaScript, Node), Ruby with YJIT | near-native speed on hot paths; warm-up time and extra memory |

"Interpreted" describes an implementation, not a language. Java is compiled to bytecode by `javac` and then interpreted and JIT-compiled by the JVM. TypeScript is compiled (transpiled) to JavaScript, which V8 then JIT-compiles.

## 2. Typing: static vs dynamic, strong vs weak

| Language | When types are checked | Implicit conversions | Example |
|---|---|---|---|
| C++ | compile time | many (numeric promotions, `int` to `bool`, array to pointer) | `int x = 3.9;` compiles and `x` is 3 |
| Java, C# | compile time | few | `int x = 3.9;` is a compile error |
| TypeScript | compile time, erased at runtime | JavaScript rules at runtime | type errors appear in the editor and `tsc`, not in production |
| Ruby, Python | runtime | very few (strong) | `"1" + 1` raises `TypeError` |
| JavaScript | runtime | many (weak) | `"1" + 1` is `"11"`, `"3" * "4"` is `12` |

Strong vs weak has no formal definition and is a spectrum, so in interviews describe concrete behavior ("JavaScript coerces in `==` and `+`, Ruby raises instead").

## 3. Data types, sizes and overflow

```cpp
#include <climits>
#include <cstdint>
#include <iostream>
using namespace std;

int main() {
    cout << sizeof(char) << ' ' << sizeof(int) << ' ' << sizeof(long long) << ' '
         << sizeof(double) << ' ' << sizeof(void*) << '\n';   // 1 4 8 8 8 on a typical 64-bit build

    int32_t a = INT_MAX;                // 2147483647
    // a + 1 would be signed overflow: undefined behavior
    int64_t widened = int64_t{a} + 1;   // widen first: 2147483648
    uint8_t u = 255;
    u = u + 1;                          // unsigned values wrap around: 0
    cout << widened << ' ' << int(u) << '\n';

    int x = 100000, y = 100000;
    // long long wrong = x * y;         // int * int overflows BEFORE the result is widened
    long long right = 1LL * x * y;      // promote first: 10000000000
    cout << right << '\n';

    cout << 7 / 2 << ' ' << -7 / 2 << ' ' << -7 % 2 << ' ' << 7 / 2.0 << '\n';  // 3 -3 -1 3.5
}
```

- Undefined behavior (UB) means the standard places no requirements: the program may appear to work, crash, or be "optimized" in surprising ways. Signed overflow, out-of-bounds access and use-after-free are UB.
- C++ integer division truncates toward zero and `%` takes the sign of the dividend. Ruby and Python floor instead (`-7 / 2 == -4`, `-7 % 2 == 1`); JavaScript has only floating-point division (`-7 / 2 === -3.5`, so use `Math.trunc` or `Math.floor`).
- `0.1 + 0.2 != 0.3` in every IEEE 754 language. Compare floats with a tolerance and store money as integer cents or a decimal type.
- `char` is a small integer: `'a' + 1 == 98` and `char('a' + 1) == 'b'`. A `std::string` holds bytes, so a UTF-8 string with accented characters has `size()` larger than its character count.

> **In your web stack:** Ruby Integers have arbitrary precision, so overflow never happens. JavaScript loses precision above `Number.MAX_SAFE_INTEGER` (9007199254740991), so large database ids or external ids should travel in JSON as strings (or use `BigInt`).

## 4. Control flow

```cpp
switch (status / 100) {                  // switch on an integer
    case 2: label = "success"; break;
    case 4: label = "client error"; break;
    case 5: label = "server error"; break; // forgetting break falls through to the next case
    default: label = "other";
}
do { ++attempts; } while (attempts < 3);  // do-while runs the body at least once
if (p != nullptr && *p > 0) { /* ... */ } // short-circuit: *p never runs when p is null
```

Loops (`for`, range-based `for`, `while`, `do-while`), `break` and `continue` work the same way across C-family languages. Short-circuit evaluation (`&&` stops at the first false, `||` at the first true) is what makes guard expressions safe everywhere: `user && user.admin?` in Ruby, `user?.isAdmin` in TypeScript.

## 5. Functions and parameter passing

```cpp
#include <iostream>
#include <string>
#include <vector>
using namespace std;

void byValue(int x) { x = 100; }               // works on a copy: caller unchanged
void byReference(int& x) { x = 100; }          // an alias: caller changed
void byPointer(int* x) { if (x) *x = 100; }    // an address: caller changed, may be null
size_t countSkus(const vector<string>& skus) { return skus.size(); }  // no copy, read-only

int main() {
    int a = 1, b = 1, c = 1;
    byValue(a);
    byReference(b);
    byPointer(&c);
    cout << a << ' ' << b << ' ' << c << '\n';  // 1 100 100
}
```

| Mode | Syntax | Copies the argument? | Can change the caller's value? | Can be null? | Use for |
|---|---|---|---|---|---|
| By value | `T x` | yes | no | n/a | small types, or when you need a copy anyway |
| By reference | `T& x` | no | yes | no | output parameters, in-place modification |
| By const reference | `const T& x` | no | no | no | large read-only inputs (strings, vectors) |
| By pointer | `T* x` | copies the address | yes | yes | optional arguments, C APIs |

Returning by value is cheap in modern C++ thanks to move semantics and copy elision (guaranteed for temporaries since C++17).

> **In your web stack:** Ruby, JavaScript, Python and Java pass object references by value ("pass by sharing"). Mutating the object is visible to the caller; reassigning the parameter is not.

The value copied for an object argument is the *reference* (the address of the object), not the object itself. The parameter and the caller's variable then lead to the same object, which gives two rules:

1. **Mutating through the parameter is visible to the caller**, because both names reach the same object.
2. **Reassigning the parameter is not visible**, because assignment only changes which object the local name points to. The caller's name still points at the original.

Primitives such as Java `int` are copied outright, so they follow the C++ by-value rules. Immutable objects (Python `int` and `str`, JavaScript strings) cannot be mutated at all, so every apparent change is a reassignment and stays local.

```ts
function addItem(cart: string[]) { cart.push("sku-4"); }  // mutation: the caller sees sku-4
function replaceCart(cart: string[]) { cart = []; }        // reassignment: the caller's array is untouched

const cart = ["sku-1"];
addItem(cart);
replaceCart(cart);
console.log(cart);                                          // ["sku-1", "sku-4"]
```

```python
def add_item(cart: list[str]) -> None:
    cart.append("sku-4")                 # mutates the shared list: the caller sees it

def replace_cart(cart: list[str]) -> None:
    cart = ["sku-9"]                     # rebinds the local name only

def swap(a: list[str], b: list[str]) -> None:
    a, b = b, a                          # swaps two local names: the caller is unchanged

cart = ["sku-1"]
add_item(cart)
replace_cart(cart)
print(cart)                              # ['sku-1', 'sku-4']
```

```ruby
def add_item(cart)
  cart << "sku-4"                       # mutates the shared array: the caller sees it
end

def replace_cart(cart)
  cart = ["sku-9"]                     # rebinds the local name only
end

cart = ["sku-1"]
add_item(cart)
replace_cart(cart)
p cart                                 # ["sku-1", "sku-4"]
```

```java
static void addItem(List<String> cart) { cart.add("sku-4"); }                            // visible to the caller
static void replaceCart(List<String> cart) { cart = new ArrayList<>(); }                 // local only
static void swap(List<String> a, List<String> b) { List<String> t = a; a = b; b = t; }   // does nothing outside
```

Two gotchas that trip people up:
- Ruby: `cart += ["sku-4"]` looks like `<<` but builds a new array and rebinds the local name, so the caller sees nothing. `cart << "sku-4"` mutates.
- Python: `items += ["x"]` on a list calls `list.__iadd__` and mutates in place, while `items = items + ["x"]` builds a new list and rebinds.

### How C++ differs

C++ does not share objects by default. Writing a parameter as `T x` copies the whole object, so neither mutation nor reassignment reaches the caller. Sharing exists only when you ask for it, and the two ways to ask map onto the other languages:

- `T*` (a pointer passed by value) is pass by sharing written out by hand. The pointer is copied, but it leads to the same object, so mutation through it is visible and reassigning the pointer is not.
- `T&` (a reference) is an alias for the caller's variable, so both mutation and reassignment are visible. This is stronger than sharing, and it is why `std::swap(a, b)` works on C++ objects while the Java `swap` above does nothing.

```cpp
#include <iostream>
#include <string>
#include <vector>
using namespace std;

void addCopy(vector<string> cart) { cart.push_back("sku-4"); }          // works on a copy: caller unchanged
void addViaPointer(vector<string>* cart) { cart->push_back("sku-4"); }  // same object: caller sees it
void resetPointer(vector<string>* cart) { cart = nullptr; }             // only the local pointer changes
void addViaRef(vector<string>& cart) { cart.push_back("sku-4"); }       // alias: caller sees it
void replaceViaRef(vector<string>& cart) { cart = {"sku-9"}; }          // alias: caller's vector is replaced

int main() {
    vector<string> cart = {"sku-1"};
    addCopy(cart);                 // still 1 item
    addViaPointer(&cart);          // 2 items
    resetPointer(&cart);           // still 2 items: the copied pointer was reassigned, not the vector
    addViaRef(cart);               // 3 items
    replaceViaRef(cart);
    cout << cart.size() << '\n';   // 1
}
```

| Mechanism | Mutation seen by caller? | Reassignment seen by caller? | What is copied | Closest equivalent |
|---|---|---|---|---|
| Object argument in Ruby, JavaScript, Python, Java | yes | no | the reference | the default for all four |
| C++ `T x` | no | no | the whole object | C++ default |
| C++ `T* x` | yes | no | the address | pass by sharing |
| C++ `T& x` | yes | yes | nothing | C# `ref` parameter |
| C++ `const T& x` | no (compile error) | no | nothing | read-only, no copy |

Because a C++ object lives in its owner's scope or on the heap, a pointer or reference you keep after the call must not outlive it (see section 6). Garbage-collected languages avoid that by keeping an object alive for as long as any reference to it exists, which is why sharing is the safe default there and an explicit choice in C++.

## 6. Scope and lifetime

```cpp
#include <iostream>
using namespace std;

int totalRequests = 0;               // global: lives for the whole program

int nextId() {
    static int counter = 0;          // static local: initialized once, survives between calls
    return ++counter;
}
int& dangling() {
    int local = 42;                  // automatic storage: destroyed when the function returns
    return local;                    // BUG: reference to a dead object (compilers warn)
}
int main() {
    { int inner = 1; }               // block scope: invisible after the closing brace
    int first = nextId();
    int second = nextId();
    cout << first << ' ' << second << '\n';   // 1 2
}
```

Scope is a compile-time question (block, function, class, namespace, file). Lifetime, or storage duration, is a runtime question: automatic (stack frame), static (globals and `static` locals), dynamic (heap, until freed) and thread-local.

> **In your web stack:** in garbage-collected languages, an object lives as long as something references it. A closure that captures a local keeps it alive after the function returns (see [Ruby closures](../03-languages/ruby/04-blocks-procs-lambdas.md) and [JavaScript closures](../03-languages/javascript/02-scope-closures-hoisting.md)). JavaScript `var` is function-scoped while `let` and `const` are block-scoped.

## 7. Stack vs heap

```text
 high addresses
 +----------------------------------+
 | stack (grows downward)           |
 |   main():  count, prices header  |  <- every call pushes a frame:
 |   build(): i, tmp, return addr   |     locals, arguments, return address
 |          v   free space   ^      |
 | heap (grows upward)              |  <- vector elements, new Node{...},
 |   [10][20][30]     Node{2}       |     long strings
 +----------------------------------+
 | static data: globals, statics    |
 | code (text segment)              |
 +----------------------------------+
 low addresses
```

```cpp
#include <memory>
#include <vector>
using namespace std;

struct Node { int value; Node* next; };

void demo() {
    int count = 3;                                   // stack
    vector<int> prices = {10, 20, 30};               // header on the stack, elements on the heap
    Node* raw = new Node{1, nullptr};                // heap, manual lifetime
    delete raw;                                      // forget it: leak; twice: undefined behavior
    auto owned = make_unique<Node>(Node{2, nullptr}); // heap, freed automatically (RAII)
}                                                    // count, prices and owned are cleaned up here
```

| | Stack | Heap |
|---|---|---|
| Allocation | automatic, just moves the stack pointer (very fast) | allocator call (slower), can fragment |
| Lifetime | ends when the function returns | until freed (`delete`, smart pointer, garbage collector) |
| Size | small and fixed per thread (commonly 1 MB on Windows, 8 MB for the Linux main thread) | large, limited by available memory |
| Typical bugs | stack overflow, dangling references to locals | leaks, double free, use-after-free |
| Access pattern | LIFO, cache friendly | anywhere through pointers |

RAII (Resource Acquisition Is Initialization) ties a resource to an object's lifetime, so destructors release memory, files and locks even when an exception unwinds the stack. See [C++ fundamentals](../03-languages/c-cpp/02-cpp-fundamentals.md).

> **In your web stack:** Ruby, JavaScript, Python, Java and C# put objects on a garbage-collected heap while stack frames hold references. You never call `free`, but you can still leak memory by keeping references alive: global caches, event listeners that are never removed, closures stored in long-lived objects.

## 8. Recursion and the call stack

Recursion means a function solves a problem by calling itself on a smaller version of the same problem. Every correct recursive function has two parts:

1. **Base case:** an input small enough to answer directly, with no further call. It is what stops the recursion.
2. **Recursive case:** reduce the input toward the base case, call the function on the smaller input, and combine the result with the current step. "Smaller" must be real progress (`n - 1`, half the range, a child node), otherwise the calls never end.

Three questions to ask for any recursive problem: What is the smallest input I can answer without recursing? How do I shrink the input by one step? How do I combine the answer for the smaller input into the answer for this one? Then take the "leap of faith": assume the recursive call already returns the right answer for the smaller input, and only check that the current step is correct.

### 8.1 The call stack, step by step

```cpp
#include <iostream>
using namespace std;

long long factorial(int n) {
    if (n <= 1) return 1;              // base case stops the recursion
    return n * factorial(n - 1);       // each call waits for the next: n frames deep
}
long long factorialIter(int n) {       // same result, constant stack usage
    long long result = 1;
    for (int i = 2; i <= n; ++i) result *= i;
    return result;
}
int main() { cout << factorial(5) << ' ' << factorialIter(20) << '\n'; }  // 120 2432902008176640000
```

Each call gets its own stack frame holding its own copy of `n` and a return address. Execution has two phases: **winding** (calls pile up until the base case) and **unwinding** (frames return one by one, and the pending work, here the multiplication, finishes on the way back).

```text
factorial(4)
  -> 4 * factorial(3)              frame 1 waits
       -> 3 * factorial(2)         frame 2 waits
            -> 2 * factorial(1)    frame 3 waits
                 -> returns 1      frame 4: base case, winding stops
            <- returns 2 * 1 = 2   unwinding starts
       <- returns 3 * 2 = 6
  <- returns 4 * 6 = 24
```

The stack therefore holds as many frames as the recursion is deep, which is why recursion costs O(depth) memory even when the function allocates nothing. `factorial(21)` overflows `long long`, so the recursion limit is rarely the first thing to break here (see section 3).

### 8.2 Classic examples

```cpp
#include <iostream>
#include <string>
#include <vector>
using namespace std;

// Sum of digits: shrink by dropping the last digit
int sumDigits(int n) {
    if (n < 10) return n;                         // base case: a single digit
    return n % 10 + sumDigits(n / 10);
}

// Greatest common divisor (Euclid): the second argument shrinks every call
int gcd(int a, int b) {
    if (b == 0) return a;
    return gcd(b, a % b);
}

// Palindrome check: compare the ends, then recurse on the inside
bool isPalindrome(const string& s, int l, int r) {
    if (l >= r) return true;                      // 0 or 1 characters left
    if (s[l] != s[r]) return false;
    return isPalindrome(s, l + 1, r - 1);
}

// Fast exponentiation: halve the exponent, O(log n) instead of O(n)
long long power(long long base, int exp) {
    if (exp == 0) return 1;
    long long half = power(base, exp / 2);        // compute ONCE and reuse
    return exp % 2 == 0 ? half * half : half * half * base;
}

// Recursive binary search on a sorted vector: search the half that can hold the target
int binarySearch(const vector<int>& a, int target, int lo, int hi) {
    if (lo > hi) return -1;                       // empty range: not found
    int mid = lo + (hi - lo) / 2;                 // avoids (lo + hi) overflow
    if (a[mid] == target) return mid;
    return a[mid] < target ? binarySearch(a, target, mid + 1, hi)
                           : binarySearch(a, target, lo, mid - 1);
}

// Tower of Hanoi: move n-1 disks away, move the big one, move n-1 disks back on top
void hanoi(int n, char from, char to, char via, int& moves) {
    if (n == 0) return;
    hanoi(n - 1, from, via, to, moves);
    ++moves;                                      // move disk n from 'from' to 'to'
    hanoi(n - 1, via, to, from, moves);
}

int main() {
    cout << sumDigits(9045) << ' ' << gcd(48, 18) << ' '
         << isPalindrome("racecar", 0, 6) << ' ' << power(2, 10) << '\n';   // 18 6 1 1024
    vector<int> sorted = {2, 5, 8, 12, 16, 23};
    cout << binarySearch(sorted, 12, 0, 5) << ' ' << binarySearch(sorted, 7, 0, 5) << '\n';  // 3 -1
    int moves = 0;
    hanoi(3, 'A', 'C', 'B', moves);
    cout << moves << '\n';                        // 7, which is 2^3 - 1
}
```

Recursion fits naturally when the data is itself recursive: nested arrays, trees, file systems, JSON, comment threads.

```ruby
def flatten(items)
  items.flat_map { |x| x.is_a?(Array) ? flatten(x) : [x] }   # an Array element is a smaller instance of the same problem
end
flatten([1, [2, [3, 4]], 5])   # [1, 2, 3, 4, 5]
```

```ts
interface CommentNode { id: number; text: string; replies: CommentNode[] }

function countComments(c: CommentNode): number {
  return 1 + c.replies.reduce((sum, r) => sum + countComments(r), 0);   // this node plus every reply subtree
}
```

### 8.3 Before or after the recursive call

Where the work sits relative to the recursive call changes the order of effects. Work before the call runs on the way down (winding), work after it runs on the way back (unwinding).

```cpp
struct Node { int value; Node* next; };

void printForward(Node* n)  { if (!n) return; cout << n->value << ' '; printForward(n->next); }   // 1 2 3
void printBackward(Node* n) { if (!n) return; printBackward(n->next); cout << n->value << ' '; }  // 3 2 1

void f(int n) {                    // f(3) prints 3 2 1 1 2 3
    if (n == 0) return;
    cout << n << ' ';              // on the way down
    f(n - 1);
    cout << n << ' ';              // on the way back up
}
```

Printing a singly linked list backwards without reversing it is the textbook use: the stack remembers the nodes for you.

### 8.4 Kinds of recursion

| Kind | Shape | Example |
|---|---|---|
| Direct | a function calls itself | `factorial` |
| Indirect (mutual) | `A` calls `B`, `B` calls `A` | `isEven` and `isOdd` below |
| Linear | one recursive call per invocation, depth equals n | `factorial`, `sumDigits`, linked list traversal |
| Binary or tree | two recursive calls per invocation | naive Fibonacci, tree traversal, merge sort |
| Multiple | a loop makes several recursive calls | permutations, subsets, N-Queens (see [backtracking](../02-dsa/05-recursion-backtracking.md)) |
| Tail | the recursive call is the very last action (section 8.8) | `factTail` |
| Head | the recursive call comes first, the work after it | `printBackward` |

```cpp
bool isOdd(unsigned n);                                              // forward declaration
bool isEven(unsigned n) { return n == 0 ? true  : isOdd(n - 1); }   // isEven(4) -> isOdd(3) -> ... -> isEven(0)
bool isOdd(unsigned n)  { return n == 0 ? false : isEven(n - 1); }
```

### 8.5 Time and space complexity

Time: write the **recurrence**, then expand it or draw the **recursion tree** (calls per level times work per call, summed over levels). Space: the **maximum depth** of the stack times the size of a frame, plus any heap memory you allocate.

| Function | Recurrence | Time | Stack space |
|---|---|---|---|
| `factorial`, `sumDigits` | T(n) = T(n-1) + O(1) | O(n), and O(log n) for `sumDigits` | O(n), and O(log n) for `sumDigits` |
| Recursive binary search | T(n) = T(n/2) + O(1) | O(log n) | O(log n) (an iterative version uses O(1)) |
| Fast `power` | T(n) = T(n/2) + O(1) | O(log n) | O(log n) |
| Naive Fibonacci | T(n) = T(n-1) + T(n-2) + O(1) | O(2^n), tight bound about 1.618^n | O(n) (depth, not call count) |
| Memoized Fibonacci | each n computed once | O(n) | O(n) |
| Tower of Hanoi | T(n) = 2T(n-1) + O(1) | O(2^n) | O(n) |
| Merge sort | T(n) = 2T(n/2) + O(n) | O(n log n) | O(log n) stack plus the merge buffer |
| Subsets, permutations | one branch per choice | O(n * 2^n), O(n * n!) | O(n) |

Master theorem shortcut for divide and conquer, `T(n) = a*T(n/b) + O(n^d)`: if `d < log_b(a)` the answer is `O(n^(log_b a))`; if `d == log_b(a)` it is `O(n^d * log n)`; if `d > log_b(a)` it is `O(n^d)`. Merge sort has a = 2, b = 2, d = 1, so it lands in the middle case: `O(n log n)`.

A common mistake is to confuse the **number of calls** with the **depth**. Naive `fib(n)` makes exponentially many calls, but at any moment the stack holds only n frames, so its space is O(n).

### 8.6 Overlapping subproblems and memoization

Naive Fibonacci recomputes the same values again and again: `fib(5)` calls `fib(3)` twice, `fib(2)` three times, and `fib(50)` makes about 40 billion calls. When a recursion revisits the same inputs, **cache the results** (memoization, top-down dynamic programming).

```cpp
long long fibNaive(int n) { return n <= 1 ? n : fibNaive(n - 1) + fibNaive(n - 2); }   // O(2^n)

long long fibMemo(int n, vector<long long>& memo) {      // O(n) time, O(n) space
    if (n <= 1) return n;
    if (memo[n] != -1) return memo[n];                   // already solved
    return memo[n] = fibMemo(n - 1, memo) + fibMemo(n - 2, memo);
}                                                        // call with vector<long long> memo(n + 1, -1)

long long fibIter(int n) {                               // bottom-up: O(n) time, O(1) space
    long long a = 0, b = 1;
    for (int i = 0; i < n; ++i) { long long next = a + b; a = b; b = next; }
    return a;                                            // fib(92) is the largest that fits in long long
}
```

```python
from functools import cache

@cache                                    # memoization in one line (Python 3.9+)
def fib(n: int) -> int:
    return n if n < 2 else fib(n - 1) + fib(n - 2)
```

```ts
const memo = new Map<number, bigint>();
function fib(n: number): bigint {
  if (n < 2) return BigInt(n);
  const hit = memo.get(n);
  if (hit !== undefined) return hit;
  const value = fib(n - 1) + fib(n - 2);
  memo.set(n, value);
  return value;
}
```

Memoization works only when the function is pure (same input, same output) and the subproblems overlap. Divide and conquer such as merge sort has no overlap, so caching does not help there. See [dynamic programming](../02-dsa/10-dynamic-programming.md).

### 8.7 Stack overflow: causes, limits and fixes

Causes: a missing or unreachable base case, a recursive step that does not shrink the input, or legitimate recursion deeper than the stack allows (walking a million-node linked list recursively).

| Runtime | What you see | Rough depth for a small frame |
|---|---|---|
| C++ | segmentation fault (Linux), stack overflow crash (Windows) | tens of thousands to a few hundred thousand, set by the 1 MB or 8 MB stack and the frame size |
| Python | `RecursionError` | 1000 by default (`sys.getrecursionlimit()`) |
| Ruby | `SystemStackError` ("stack level too deep") | around 10,000 |
| JavaScript (V8) | `RangeError: Maximum call stack size exceeded` | around 10,000 |
| Java | `StackOverflowError` | around 10,000 to 20,000 with the default `-Xss` |

The depths are approximate and vary with the version and the size of each frame. Raising the limit (`sys.setrecursionlimit`) can turn a clean exception into a hard interpreter crash, so prefer one of these fixes:

- **Iterate with an explicit stack.** Any recursion can be rewritten this way, because the call stack is just a stack you stop managing by hand.
- **Reduce the depth.** Divide and conquer keeps it at O(log n).
- **Use tail recursion** only where the language guarantees tail-call optimization (section 8.8).
- **Memoize** to cut calls when the real problem is repeated work.

```cpp
#include <algorithm>
#include <stack>
#include <utility>
using namespace std;

struct TreeNode { int val; TreeNode* left; TreeNode* right; };

int maxDepth(TreeNode* root) {                       // recursive: depth of the tree = stack depth
    return root ? 1 + max(maxDepth(root->left), maxDepth(root->right)) : 0;
}

int maxDepthIter(TreeNode* root) {                   // same answer, explicit stack on the heap
    if (!root) return 0;
    stack<pair<TreeNode*, int>> st;
    st.push({root, 1});
    int best = 0;
    while (!st.empty()) {
        auto [node, depth] = st.top();
        st.pop();
        best = max(best, depth);
        if (node->left)  st.push({node->left,  depth + 1});
        if (node->right) st.push({node->right, depth + 1});
    }
    return best;
}
```

A classic Rails version of the bug is a model method that calls itself by accident, for example overriding `name` and calling `name` inside it instead of `super` or `self[:name]`.

### 8.8 Tail recursion and tail-call optimization

A call is a **tail call** when nothing remains to do after it returns, so its result is returned unchanged. A compiler or VM that supports tail-call optimization (TCO) reuses the current frame and turns the call into a jump, giving O(1) stack space.

```cpp
long long factorial(int n)                     { return n <= 1 ? 1 : n * factorial(n - 1); }   // NOT a tail call: the multiply is pending

long long factTail(int n, long long acc = 1) {         // accumulator carries the partial result
    if (n <= 1) return acc;
    return factTail(n - 1, acc * n);                   // tail call: nothing left to do afterwards
}
```

TCO is **guaranteed** in Scheme, Haskell and (for self-calls) Scala. It is **not guaranteed** in C++ (GCC and Clang often do it at `-O2`, but a debug build will not), CRuby (opt-in only), Python (deliberately never), Java, or JavaScript (the ES2015 spec requires it, but only Safari's engine implements it, not V8). In interviews, say that writing in tail form is good practice but you should not rely on it for stack safety in these languages.

### 8.9 Recursion vs iteration

| | Recursion | Iteration |
|---|---|---|
| Readability | very clear for trees, graphs, nested data and divide and conquer | clear for simple counting and linear scans |
| Memory | O(depth) stack frames | usually O(1), or an explicit stack on the heap |
| Speed | function call overhead per step | faster in tight loops |
| Failure mode | stack overflow | none from depth, but can loop forever |
| Natural fit | tree traversal, backtracking, parsing, `n!`-style generation | loops over arrays, accumulations, streaming data |

Rule of thumb: use recursion when the problem or data is recursive and the depth is bounded (balanced trees, logarithmic splits); use a loop when the recursion would be linear and deep, or when you are on a hot path.

### 8.10 Recursion in the web stack

- **Nested data:** comment threads, category trees, menus and file trees are recursive. A React component can render itself:

```tsx
function CommentThread({ comment }: { comment: CommentNode }) {
  return (
    <li>
      {comment.text}
      <ul>{comment.replies.map((r) => <CommentThread key={r.id} comment={r} />)}</ul>
    </li>
  );
}
```

- **Deep clone and JSON walks:** `structuredClone`, schema validators and serializers recurse over objects. A circular reference without cycle detection recurses forever, which is why `JSON.stringify` throws on cycles.
- **SQL:** PostgreSQL walks a parent-child table with a recursive CTE, with no application-level recursion and no N+1 queries per level:

```sql
WITH RECURSIVE tree AS (
  SELECT id, parent_id, name, 1 AS depth FROM categories WHERE parent_id IS NULL   -- base case: the roots
  UNION ALL
  SELECT c.id, c.parent_id, c.name, t.depth + 1
  FROM categories c JOIN tree t ON c.parent_id = t.id                              -- recursive step: the children
)
SELECT * FROM tree;
```

- **Rails:** a `has_many :children` association loaded level by level causes an N+1 query per level. Load the tree with a recursive CTE or a gem such as `ancestry`, and add a depth limit or cycle check so bad data cannot loop.

### 8.11 A recipe for solving and debugging recursion

1. Define precisely what the function returns for an input of size n (its contract).
2. Write the base case(s) first: the smallest inputs, including empty, null and one element.
3. Write the recursive step assuming the contract holds for the smaller input.
4. Check that every path makes progress toward a base case.
5. Trace a tiny input (n = 0, 1, 2, 3) by hand on paper, drawing the call tree.
6. If it misbehaves, print the arguments with indentation by depth (`cout << string(depth * 2, ' ') << "f(" << n << ")\n";`) or use the debugger's call stack view.

For problems that build up a path and undo choices (subsets, permutations, N-Queens, Sudoku), the recursive pattern is backtracking: see [Recursion and Backtracking](../02-dsa/05-recursion-backtracking.md).

## 9. Arrays and strings

- C arrays are fixed-size, contiguous and unchecked: `arr[10]` on a 10-element array is undefined behavior, and an array passed to a function decays to a pointer, losing its size.
- `std::array<T, N>` has a fixed size with `.size()`. `std::vector<T>` grows geometrically, so `push_back` is amortized O(1); a reallocation moves the elements and invalidates iterators and pointers into the vector. `.at(i)` checks bounds and throws `std::out_of_range`; `[i]` does not check.
- C strings are `char` arrays terminated by `'\0'`: `strlen` is O(n) and overflowing the buffer is a classic security bug. `std::string` tracks its size, manages memory and is mutable.
- Strings are immutable in Java, C#, Python and JavaScript (build large strings with `StringBuilder` or `join`) and mutable in C++ and Ruby.

```cpp
#include <iostream>
#include <string>
#include <vector>
using namespace std;

int main() {
    vector<int> v;
    v.reserve(4);                               // one allocation when the size is known
    for (int i = 1; i <= 4; ++i) v.push_back(i * 10);
    cout << v[2] << ' ' << v.size() << '\n';    // 30 4
    // v.at(10) throws std::out_of_range; v[10] is undefined behavior
    string sku = "zelda-01";
    sku[0] = 'Z';                               // std::string is mutable
    sku += "-EU";
    cout << sku << ' ' << sku.size() << '\n';   // Zelda-01-EU 11
}
```

More in [Arrays, strings and hashing](../02-dsa/01-arrays-strings-hashing.md) and [STL for DSA](../03-languages/c-cpp/04-stl-for-dsa.md).

## 10. Error handling: exceptions vs return codes

```cpp
#include <iostream>
#include <optional>
#include <stdexcept>
#include <string>
using namespace std;

// Return-value style: the type says "may be missing" and the caller must check
optional<int> parsePort(const string& s) {
    try {
        size_t used = 0;
        int port = stoi(s, &used);
        if (used != s.size() || port < 1 || port > 65535) return nullopt;
        return port;
    } catch (const exception&) {        // stoi throws invalid_argument or out_of_range
        return nullopt;
    }
}

// Exception style: bad input is exceptional here, so callers higher up decide
int requirePort(const string& s) {
    if (auto port = parsePort(s)) return *port;
    throw invalid_argument("invalid port: " + s);
}

int main() {
    cout << parsePort("8080").value_or(-1) << ' ' << parsePort("80x").value_or(-1) << '\n';  // 8080 -1
    try {
        requirePort("http");
    } catch (const invalid_argument& e) {
        cout << "error: " << e.what() << '\n';     // error: invalid port: http
    }
}
```

| | Return codes, `optional`, result types | Exceptions |
|---|---|---|
| Visibility | explicit in the signature and at each call site | hidden control flow: any call may throw |
| Forgetting to handle | silently ignored unless the type forces a check (`optional`, `[[nodiscard]]`) | propagates loudly until caught |
| Cost | cheap and predictable | near zero when not thrown in common implementations, expensive when thrown |
| Cleanup | manual at every exit point, unless RAII | stack unwinding runs destructors |
| Best for | expected failures: parse errors, not found, validation | unexpected failures, broken invariants, constructor failures |

> **In your web stack:** Ruby raises exceptions, and Rails pairs status-returning and raising variants (`save` returns `false`, `save!` raises). JavaScript uses `throw` plus rejected Promises (`try/catch` around `await`). Many TypeScript codebases return result objects (`{ ok: true, data } | { ok: false, error }`) for expected failures. See [Ruby exceptions](../03-languages/ruby/06-advanced-internals.md).

## 11. From source to executable: compile and link

1. **Preprocess:** expand `#include` (paste headers in), macros and `#ifdef` blocks. Each `.cpp` becomes one translation unit.
2. **Compile:** check syntax and types, optimize, and emit assembly.
3. **Assemble:** produce an object file (`.o` or `.obj`) containing machine code plus a symbol table of defined and undefined symbols.
4. **Link:** combine object files and libraries, resolve every symbol to an address, and write the executable. Static libraries (`.a`, `.lib`) are copied in; shared libraries (`.so`, `.dll`, `.dylib`) are loaded at runtime.

```bash
g++ -std=c++17 -Wall -Wextra -g -c main.cpp -o main.o       # compile only
g++ -std=c++17 -Wall -Wextra -g -c pricing.cpp -o pricing.o
g++ main.o pricing.o -o app                                  # link
./app
g++ -std=c++17 -E main.cpp | head                            # inspect preprocessor output
```

Compile errors point at a line. Linker errors such as "undefined reference to applyDiscount(int)" mean a function was declared but never defined in anything being linked: a missing source file, a missing library flag, or a header signature that differs from the implementation. Include guards (`#pragma once` or `#ifndef`) prevent double inclusion, and the One Definition Rule forbids defining the same non-inline function in two translation units. `bits/stdc++.h` is a GCC-only header that pulls in the whole standard library: handy in contests, but not portable (MSVC lacks it) and slow to compile, so include specific headers.

> **In your web stack:** a "build" in TypeScript means type checking (`tsc`), transpiling and bundling (Vite, esbuild, Metro for React Native), not linking machine code. Ruby app code has no build step, but native-extension gems such as `pg` compile C code during `bundle install`, which is why missing system libraries break installs in Docker images.

## 12. Debugging basics

1. **Reproduce** reliably with the smallest input and exact steps, ideally in the same environment.
2. **Read the error and stack trace:** the top frame is where it failed; the first frame in your own code is usually where to look.
3. **Hypothesize and test:** log values, step through with a debugger, and check assumptions (inputs, null or `nil`, off-by-one, types, time zones).
4. **Narrow down:** bisect the code path, or let `git bisect` find the commit that introduced the bug.
5. **Fix the cause, not the symptom,** then add a test that would have caught it.

| Stack | Tools |
|---|---|
| C, C++ | compile with `-g`; `gdb`/`lldb` (`break`, `run`, `next`, `step`, `print`, `bt`); `-fsanitize=address,undefined`; Valgrind |
| Ruby, Rails | `debugger` from the `debug` gem, `binding.irb`, `rails console`, logs, `method(:x).source_location` |
| JavaScript, TypeScript, React | browser DevTools breakpoints, the `debugger;` statement, React DevTools, `node --inspect` |
| React Native, Expo | Expo dev tools, device logs (Xcode console, `adb logcat`) |

`git bisect` automates step 4: mark the current commit `bad` and an old one `good`, then Git checks out midpoints for you to test until it names the first bad commit (see [Git](../14-devops-and-tools/01-git.md)).

## Common pitfalls
- Overflow in an intermediate calculation (`x * y` computed in `int` before being stored in a `long long`).
- Assuming integer division and `%` behave the same in C++, Ruby, Python and JavaScript.
- Comparing floating-point values with `==`, or storing money in floats.
- Returning a reference or pointer to a local variable.
- Recursion without a base case, or too deep for the stack.
- A recursive step that does not shrink the input (`f(n)` calling `f(n)`), or a base case that is skipped for some inputs (negative `n`, empty list, null node).
- Recomputing overlapping subproblems (naive Fibonacci), or calling the recursive function twice where once would do (`power(b, e/2) * power(b, e/2)` is O(n), not O(log n)).
- Confusing the number of calls with the stack depth when stating space complexity.
- Relying on tail-call optimization in C++, Ruby, Python or JavaScript.
- Forgetting that work placed after the recursive call runs in reverse order.
- `switch` cases without `break` falling through.
- Off-by-one loops (`i <= n` over an array of size `n`) and unchecked `[]` access.
- Swallowing exceptions or ignoring return codes.
- Believing Ruby or JavaScript copies objects when passing them to functions.

## How to talk about this in an interview
- You were a TA for the Programming Fundamentals lab at PUCIT from 2023 to 2025, so interviewers will expect crisp answers here. Explain each concept the way you would to a first-year student: a one-line definition, a tiny example, then a real-world consequence.
- Bridge to your daily stack to show breadth: answer pass-by-value in C++ terms, then add how Ruby and JavaScript behave ("pass by sharing"), and mention JavaScript's 2^53 precision limit or Ruby's arbitrary-precision integers when overflow comes up.
- > **Fill in:** which language the PF lab used, two mistakes students made most often (for example missing base cases or confusing pass by value and pass by reference), and how you explained them.

## Interview questions

**Q1. What is the difference between compiled, interpreted and JIT-compiled languages? Where do Ruby, JavaScript and Java fit?**
Compiled languages are translated to machine code before running (C++); interpreted ones run through a VM that executes bytecode (CRuby, CPython); JIT systems compile hot code to machine code while running. Java compiles to bytecode, then the JVM interprets and JIT-compiles it; JavaScript runs on JIT engines like V8; Ruby is interpreted by YARV with YJIT as an optional JIT.
*Follow-up:* Is TypeScript compiled? It is transpiled to JavaScript and its types are erased.

**Q2. Static vs dynamic typing, and strong vs weak typing?**
Static typing checks types before running (C++, Java, TypeScript); dynamic typing checks them at runtime (Ruby, Python, JavaScript). Strong vs weak describes implicit conversions: JavaScript happily turns `"1" + 1` into `"11"`, while Ruby and Python raise `TypeError`.
*Follow-up:* Which combination is Ruby? Dynamic and strong.

**Q3. Explain pass by value, by reference and by pointer. What do Ruby and JavaScript do?**
By value copies the argument, by reference makes the parameter an alias of the caller's variable, and by pointer passes an address that can be null and must be dereferenced. Ruby and JavaScript pass object references by value: mutations through the parameter are visible to the caller, but reassigning the parameter is not.
*Follow-up:* How do you avoid copying a large object in C++ without allowing changes? Pass `const T&`.

**Q4. What is the difference between the stack and the heap?**
The stack holds call frames (locals, arguments, return addresses), is allocated automatically, is very fast and small, and frees memory when functions return. The heap holds dynamically allocated objects with flexible lifetimes, is larger and slower to allocate, and needs manual freeing or a garbage collector.
*Follow-up:* Where does a `std::vector`'s data live? Its small header can be on the stack, its elements are on the heap.

**Q5. What causes a stack overflow and how do you fix it?**
Recursion without a reachable base case, or recursion deeper than the stack can hold, since every call adds a frame. Fix the base case, convert to iteration with an explicit stack, or restructure so depth is logarithmic.
*Follow-up:* What does Ruby raise? `SystemStackError` with "stack level too deep".

**Q6. What happens on integer overflow in C++, Ruby and JavaScript?**
In C++, signed overflow is undefined behavior and unsigned overflow wraps modulo 2^n. Ruby integers are arbitrary precision, so they never overflow. JavaScript numbers are doubles, so integers above 2^53 - 1 silently lose precision unless you use `BigInt`.
*Follow-up:* How do you multiply two `int`s safely in C++? Promote first: `1LL * a * b`.

**Q7. Why does `-7 / 2` give different answers in different languages?**
C++, Java and C# truncate toward zero (`-3`), Ruby and Python floor toward negative infinity (`-4`), and JavaScript performs floating-point division (`-3.5`). The remainder operator follows the same choice, so `-7 % 2` is `-1` in C++ and `1` in Ruby.
*Follow-up:* Why does that matter in practice? Pagination or bucketing logic ported between languages can break on negative numbers.

**Q8. What is the difference between scope and lifetime?**
Scope is where a name can be used in the source code; lifetime is how long the object exists at runtime. A `static` local has function scope but program lifetime; a heap object outlives the scope of the pointer that created it.
*Follow-up:* What is a dangling reference? A reference to an object whose lifetime has ended, such as a returned reference to a local variable.

**Q9. Exceptions or error codes: which do you prefer?**
Return values (`optional`, result types) for expected failures that callers must handle, such as parse errors or "not found"; exceptions for unexpected failures and broken invariants, where unwinding to a higher-level handler is cleaner. In C++, RAII makes exceptions safe because destructors run during unwinding.
*Follow-up:* How does Rails reflect this choice? `save` returns `false` while `save!` raises.

**Q10. What does a linker do, and what does "undefined reference" mean?**
The linker combines object files and libraries and resolves every symbol to an address. "Undefined reference" means some code uses a function or variable that was declared but never defined in anything being linked, often because a source file or library was left out of the build or a signature does not match.
*Follow-up:* Static vs dynamic linking? Static copies library code into the executable; dynamic loads shared libraries at runtime.

**Q11. Why is `vector::push_back` amortized O(1)?**
When capacity runs out, the vector allocates a larger buffer (a constant factor bigger, for example double) and moves the elements. Because capacity grows geometrically, the total copying across n pushes is O(n), so the average cost per push is constant.
*Follow-up:* What gets invalidated on reallocation? All iterators, pointers and references to the elements.

**Q12. What is undefined behavior?**
Code whose behavior the C or C++ standard does not define, such as signed overflow, out-of-bounds access, use-after-free or reading uninitialized variables. The compiler may assume it never happens, so results range from "seems to work" to crashes and security holes. Sanitizers (`-fsanitize=address,undefined`) catch many cases.
*Follow-up:* Does Ruby have undefined behavior? Not at the language level; errors raise exceptions, though C extensions can still crash.

### Recursion

**Q13. What is recursion, and what must every recursive function have?**
A function that solves a problem by calling itself on a smaller instance of the same problem. It needs a base case that returns directly, and a recursive case that moves strictly closer to the base case and combines the result. Without the base case, or without progress toward it, the calls never stop and the stack overflows.
*Follow-up:* Can a function have more than one base case? Yes, for example `fib` has two (`n == 0` and `n == 1`).

**Q14. Walk me through what happens on the call stack for `factorial(4)`.**
Each call pushes a frame with its own `n` and a return address. Frames for `factorial(4)`, `(3)`, `(2)` and `(1)` pile up; `factorial(1)` hits the base case and returns 1. Then the frames pop in reverse order, each finishing its pending multiplication: 2, then 6, then 24.
*Follow-up:* How many frames exist at the deepest point? Four, so the space is O(n).

**Q15. Recursion or iteration: how do you choose?**
Use recursion when the data or the problem is recursive (trees, graphs, nested structures, divide and conquer, backtracking) and the depth is bounded. Use a loop when the recursion would be linear and deep, because recursion adds call overhead and O(depth) stack space and risks a stack overflow. Every recursion can be turned into a loop with an explicit stack.
*Follow-up:* Is recursion ever faster? Not by itself; it can win only when it makes the algorithm better, such as divide and conquer with memoization.

**Q16. What is the time and space complexity of naive recursive Fibonacci, and how do you fix it?**
Time is O(2^n) (tightly about 1.618^n) because each call spawns two more and the same subproblems are solved repeatedly; `fib(50)` makes about 40 billion calls. Space is O(n), the maximum depth, not the number of calls. Fix it with memoization (a cache keyed by `n`) for O(n) time, or a bottom-up loop with two variables for O(n) time and O(1) space.
*Follow-up:* When does memoization not help? When subproblems do not overlap (merge sort) or the function has side effects.

**Q17. What is tail recursion, and do C++, Ruby, Python and JavaScript optimize it?**
A tail call is the last action of a function, with nothing left to do after it returns, usually achieved with an accumulator parameter (`factTail(n - 1, acc * n)`). With tail-call optimization the compiler reuses the frame for O(1) space. Scheme and Haskell guarantee it; C++ compilers often do it at `-O2` but do not promise to; CRuby has it only as an opt-in; Python never does it; JavaScript specifies it but only Safari implements it.
*Follow-up:* So how do you handle a deep recursion in Python? Rewrite it as a loop or with an explicit stack.

**Q18. What causes a stack overflow in recursion, and how do you prevent it?**
A missing or unreachable base case, a step that does not shrink the input, or a legitimately deep recursion such as a million-node linked list. Prevent it by checking base cases for all edge inputs (empty, null, negative), reducing depth with divide and conquer, converting to iteration with an explicit stack, or memoizing to cut repeated work. Raising the language limit is a last resort because it can crash the process.
*Follow-up:* What are the error names? `RecursionError` (Python), `SystemStackError` (Ruby), `RangeError` (JavaScript), `StackOverflowError` (Java), a segmentation fault (C++).

**Q19. Write a function to reverse a singly linked list recursively.**
Recurse to the last node, then on the way back make each node point to the one before it.
```cpp
Node* reverse(Node* head) {
    if (!head || !head->next) return head;   // empty list or last node becomes the new head
    Node* newHead = reverse(head->next);     // reverse the rest first
    head->next->next = head;                 // the next node now points back at us
    head->next = nullptr;                    // we become the tail for now
    return newHead;
}
```
Time is O(n) and stack space is O(n), so for very long lists prefer the iterative three-pointer version with O(1) space.
*Follow-up:* What if the list is empty? The first check returns `head` (null), so it is handled.

**Q20. Solve the Tower of Hanoi. How many moves does it take for n disks?**
To move n disks from A to C: move n-1 disks from A to B, move disk n from A to C, then move n-1 disks from B to C. The recurrence is `T(n) = 2T(n-1) + 1`, which gives `2^n - 1` moves, so 7 for three disks and over a million for 20. The time is exponential, but the stack depth is only n.
*Follow-up:* Can it be done faster? No, `2^n - 1` is the proven minimum for three pegs.

**Q21. What is the output of this code?**
```cpp
void f(int n) {
    if (n == 0) return;
    cout << n << ' ';
    f(n - 1);
    cout << n << ' ';
}
// f(3)
```
`3 2 1 1 2 3`. The first print runs while winding down (3, 2, 1), the base case returns, and the second print runs while unwinding (1, 2, 3).
*Follow-up:* What if the first print is removed? Then it prints `1 2 3` only.

**Q22. Why is a recursive binary search O(log n) space while the iterative one is O(1)?**
Each recursive call halves the range and adds a stack frame, and there are about log2(n) halvings before the range is empty, so the stack holds O(log n) frames. The loop version just updates `lo` and `hi` in place.
*Follow-up:* Is the recursive binary search tail recursive? Yes, each branch returns the recursive call directly, so a language with TCO would run it in O(1) space.

**Q23. Compute `x^n` efficiently. Where do people go wrong?**
Use exponentiation by squaring: `half = power(x, n / 2)`, then return `half * half` for even n or `half * half * x` for odd n. That is O(log n) time and stack. The classic mistake is writing `power(x, n/2) * power(x, n/2)`, which makes two calls per level and degrades to O(n). Others: forgetting negative exponents (`1 / power(x, -n)`, and `-n` overflows for `INT_MIN`), and overflow in intermediate products.
*Follow-up:* How would you do it with a loop? Square the base, halve the exponent, multiply into the result when the low bit is set.

**Q24. What is the difference between recursion, divide and conquer, dynamic programming and backtracking?**
Recursion is the technique of a function calling itself. Divide and conquer is a recursive strategy that splits the problem into independent subproblems and merges the answers (merge sort, binary search). Dynamic programming is recursion (or a table) for problems with overlapping subproblems and optimal substructure, caching each answer once. Backtracking is recursion that builds a candidate step by step and undoes (backtracks) a choice when it leads to a dead end (permutations, N-Queens).
*Follow-up:* How do you tell if a recursion needs memoization? Draw the call tree and look for the same arguments appearing more than once.

**Q25. What is direct versus indirect (mutual) recursion?**
Direct recursion means a function calls itself. Indirect recursion means a cycle through several functions: `isEven(n)` calls `isOdd(n - 1)`, which calls `isEven(n - 2)`. The same rules apply: every cycle needs a base case and progress, and in C++ you need a forward declaration for the function defined later.
*Follow-up:* Where does this appear in real code? Recursive-descent parsers, where `expression`, `term` and `factor` call each other.

**Q26. Can every recursive function be written iteratively?**
Yes. A recursive call is a push of the local state onto the call stack, and a return is a pop, so an explicit stack data structure can simulate it. Simple linear and tail recursion becomes a plain loop; tree and backtracking recursion needs a manual stack that stores the node plus where to resume. The iterative version is not always clearer, which is why recursion is kept for naturally recursive problems.
*Follow-up:* Name a case where the iterative form is clearly better. Walking a very long linked list, or a depth-first search on a graph that may be deeper than the stack allows.

**Q27. How would you render nested comments or a category tree in a web app, and what can go wrong?**
Render recursively: a component or partial renders one node and calls itself for each child. Problems: very deep or cyclic data causes a stack overflow or an infinite loop (add a depth limit and cycle detection), and loading children level by level causes N+1 queries. Fetch the whole subtree in one query with a recursive CTE (`WITH RECURSIVE`) or a tree gem, build a parent-to-children map in memory, then render from the map.
*Follow-up:* What if the tree is huge? Paginate or lazy-load children on expand instead of rendering the whole tree.
