# C++ Fundamentals

> **Tier:** T1 (know cold) · **Related:** [C essentials](01-c-essentials.md), [C++ OOP](03-cpp-oop.md), [STL for DSA](04-stl-for-dsa.md), [Programming fundamentals](../../01-cs-fundamentals/01-programming-fundamentals.md), [Operating systems](../../01-cs-fundamentals/05-operating-systems.md)
> **On your resume:** C++ in your skills list: the language you used for OOP and DSA at PUCIT and in the OOP and Data Structures labs you assisted as a TA (2023 to 2025).

## Quick revision
- Build with `g++ -std=c++17 -Wall -Wextra -g -fsanitize=address,undefined main.cpp -o main` while learning or debugging.
- **Reference** = alias: must be initialized, never null, never reseated. **Pointer** = object holding an address: nullable, reseatable, supports arithmetic.
- Pass big objects as `const T&`, things you modify as `T&`, cheap types by value. Return by value (copy elision and moves make it cheap).
- A `const` member function cannot modify the object, and only const member functions can be called on const objects.
- Overloads differ by parameter types or count, never by return type alone. Default arguments are trailing and written once, in the declaration.
- `new`/`delete` run constructors and destructors; `malloc`/`free` do not. Pair `new[]` with `delete[]`. Modern code avoids naked `new` entirely.
- **Stack**: automatic, very fast, small (often 1 MB on Windows, 8 MB on Linux). **Heap**: manual or smart-pointer lifetime, large, slower to allocate.
- **RAII**: acquire in the constructor, release in the destructor; destructors run at scope exit and during exception unwinding.
- `unique_ptr` = sole owner, move-only, zero overhead (the default). `shared_ptr` = reference counted. `weak_ptr` = non-owning observer that breaks cycles. Create them with `make_unique` / `make_shared`.
- lvalue = has a name/address; rvalue = temporary. `T&&` binds rvalues. `std::move` is only a cast; the move constructor does the stealing. Moved-from objects are valid but unspecified.
- Mark move constructors `noexcept`, or `std::vector` copies instead of moving when it grows.
- Lambdas: `[x]` copy, `[&x]` reference, `[=]`/`[&]` everything used, `mutable` lets the lambda change its copies. Reference captures that outlive the scope dangle.
- `auto` drops references and top-level const: write `const auto&` in range-for loops over strings and objects.
- Throw by value, catch by `const&`, most specific handler first. Destructors must not throw.
- `nullptr` has its own type (fixes `NULL` overload bugs). `enum class` is scoped with no implicit int conversion.
- C++17 essentials: structured bindings, `if (init; cond)`, `std::optional`, `std::variant`, `std::string_view`, guaranteed copy elision, `inline` variables.

## 1. From C to C++

| Feature | C | C++ |
|---|---|---|
| Paradigm | procedural | procedural, OOP, generic, some functional |
| Data plus behavior | plain structs | classes with access control, constructors, destructors |
| Memory | `malloc`/`free` | `new`/`delete`, RAII, smart pointers, containers |
| Pass by reference | pointers only | references (`int&`) as well |
| Overloading | none | functions and operators |
| Generic code | macros, `void *` | templates |
| Errors | return codes, `errno` | exceptions (plus return codes) |
| Strings and collections | `char` arrays, hand-written lists | `std::string`, STL containers and algorithms |

C++ is close to a superset of C but not exactly: `int *p = malloc(4);` compiles in C and fails in C++ (no implicit conversion from `void *`), and variable-length arrays are not standard C++.

## 2. Namespaces and iostream

```cpp
#include <iostream>
#include <string>

namespace billing {
    double withTax(double amount) { return amount * 1.17; }
}

int main() {
    std::cout << "Total: " << billing::withTax(100.0) << '\n';

    int age;
    std::string fullName;
    std::cin >> age;                               // reads an int, leaves '\n' in the buffer
    std::getline(std::cin >> std::ws, fullName);   // std::ws skips that leftover whitespace first
    std::cout << fullName << " is " << age << '\n';
}
```

- Namespaces prevent name clashes between libraries. `using namespace std;` is fine in a `.cpp` file or contest code but never in a header, because it leaks into every file that includes it.
- `std::endl` writes a newline **and flushes**; in loops, `'\n'` is much faster. `std::cin >> x` evaluates to false on failure or end of input, so `while (std::cin >> x)` reads until the input runs out.

## 3. References vs pointers

| | Reference `int& r = x;` | Pointer `int* p = &x;` |
|---|---|---|
| Must be initialized | yes | no (but should be) |
| Can be null | no | yes (`nullptr`) |
| Can be reseated | no: `r = y` assigns to `x` | yes: `p = &y` |
| Syntax | used like the object | `*p`, `p->member` |
| Arithmetic | no | yes |
| Typical use | parameters, returning members, range-for | optional objects, reseating, arrays, low-level data structures |

```cpp
void byValue(int n) { n++; }                 // changes a copy
void byRef(int& n) { n++; }                  // changes the caller's variable
void byPtr(int* n) { if (n) (*n)++; }        // may be null: check first
void show(const std::string& s) { std::cout << s; }   // no copy, read-only

int x = 1;
byValue(x);    // x == 1
byRef(x);      // x == 2
byPtr(&x);     // x == 3

int y = 10;
int& r = x;    // r is another name for x
r = y;         // assigns 10 to x; r still refers to x
```

| Situation | Pass as |
|---|---|
| Cheap types (`int`, `double`, pointers, iterators) | by value |
| Large object, read-only | `const T&` |
| Function must modify the caller's object | `T&` |
| "No object" is a valid input | `T*` (or `std::optional`) |
| Function stores its own copy (a sink) | by value, then `std::move` it into place |

## 4. const correctness

```cpp
class Account {
public:
    explicit Account(double balance) : balance_(balance) {}
    double balance() const { return balance_; }   // const: promises not to modify *this
    void deposit(double amount) { balance_ += amount; }
private:
    double balance_;
};

void report(const Account& acc) {
    std::cout << acc.balance() << '\n';           // OK: balance() is const
    // acc.deposit(5);                            // error: deposit() is not const
}
```

- Inside a const member function, `this` is `const Account*`, so members are read-only. Forgetting `const` on getters breaks every caller that holds a `const Account&`.
- Overload on const for accessors: `T& at(std::size_t i)` and `const T& at(std::size_t i) const`. That is how `std::vector::operator[]` works.
- `mutable` members may change inside const functions (caches, mutexes). `constexpr` values are computed at compile time. A `const T&` binds to a temporary and extends its lifetime: `const std::string& s = std::string("hi");` is safe.

## 5. Overloading, default arguments and inline

```cpp
int area(int side) { return side * side; }
int area(int w, int h) { return w * h; }
double area(double r) { return 3.14159 * r * r; }

area(4);       // area(int)
area(2, 3);    // area(int, int)
area(1.5);     // area(double)
// area(5L);   // error: ambiguous, long converts equally well to int and to double

void greet(const std::string& name, const std::string& greeting = "Hello");   // declaration
void greet(const std::string& name, const std::string& greeting) {            // no default repeated
    std::cout << greeting << ", " << name << '\n';
}
greet("Ali");             // Hello, Ali
greet("Ali", "Salaam");   // Salaam, Ali
```

- Overload resolution happens at compile time (compile-time polymorphism). The compiler encodes parameter types into symbol names (**name mangling**); `extern "C"` turns that off so C code can call the function.
- `inline` today mainly means "this definition may appear in several translation units" (an exemption from the one-definition rule). That is why small functions can live in headers, and why member functions defined inside a class body are implicitly inline. Whether a call is actually inlined is the optimizer's choice.

## 6. Memory: stack vs heap, new/delete vs malloc/free

```cpp
void demo() {
    int local = 5;                    // stack: released automatically at the closing brace
    int* one = new int(5);            // heap: lives until delete
    int* arr = new int[10]();         // 10 ints, value-initialized to 0
    std::vector<int> v(1'000'000);    // the vector object is on the stack, its elements on the heap
    delete one;
    delete[] arr;                     // new[] pairs with delete[]; plain delete here is UB
}
```

| | Stack | Heap (free store) |
|---|---|---|
| What lives there | function locals, parameters | anything from `new`, `make_unique`, container buffers |
| Lifetime | ends at scope exit | until deleted or the owning smart pointer dies |
| Speed | moving the stack pointer | allocator bookkeeping, possible fragmentation |
| Size | small, fixed per thread | limited by available memory |
| Failure mode | stack overflow (deep recursion, huge local arrays) | `std::bad_alloc`, leaks |

| | `new` / `delete` | `malloc` / `free` |
|---|---|---|
| Kind | C++ operators | C library functions |
| Constructors and destructors | called | not called |
| Result type | typed `T*` | `void *` (needs a cast in C++) |
| Size | computed from the type | bytes you pass |
| Failure | throws `std::bad_alloc` | returns `NULL` |
| Arrays | `new T[n]` with `delete[]` | `malloc(n * sizeof(T))` with `free` |

Never mix the pairs: `free` on a `new`ed object or `delete` on `malloc`ed memory is undefined behavior.

## 7. RAII

In C, every early `return` and every error path needs its own cleanup, so leaks creep in. RAII (Resource Acquisition Is Initialization) attaches cleanup to scope instead.

```cpp
#include <cstdio>
#include <stdexcept>

class File {
public:
    File(const char* path, const char* mode) : f_(std::fopen(path, mode)) {
        if (!f_) throw std::runtime_error("cannot open file");
    }
    ~File() { std::fclose(f_); }               // runs at scope exit, even during an exception
    File(const File&) = delete;                // exactly one owner: copying is forbidden
    File& operator=(const File&) = delete;
    std::FILE* get() const { return f_; }
private:
    std::FILE* f_;
};

void riskyStep();                              // may throw

void appendLog() {
    File log("app.log", "a");
    std::fputs("started\n", log.get());
    riskyStep();                               // if this throws, ~File still closes the file
}                                              // normal exit: ~File closes it here
```

`std::vector`, `std::string`, smart pointers, `std::lock_guard` and `std::fstream` are all RAII types, which is why idiomatic C++ rarely calls `delete` or `close` by hand.

In your web stack: Ruby's `File.open(path) { |f| ... }` and `ActiveRecord::Base.transaction do ... end`, Python's `with` and C#'s `using` give the same "cleanup is tied to a scope" guarantee.

## 8. Smart pointers

| | `unique_ptr<T>` | `shared_ptr<T>` | `weak_ptr<T>` |
|---|---|---|---|
| Ownership | exclusive | shared, reference counted | none (observes a `shared_ptr` object) |
| Copyable | no, move only | yes (count goes up) | yes |
| Overhead | none over a raw pointer (default deleter) | control block, atomic count updates | uses the same control block |
| Frees the object when | the owner is destroyed or reset | the last `shared_ptr` goes away | never; `lock()` returns a `shared_ptr` or empty |
| Use for | the default: members, factories, polymorphic objects in containers | genuinely shared ownership | breaking cycles, caches, observers |

```cpp
#include <iostream>
#include <memory>

struct Node {
    int value;
    explicit Node(int v) : value(v) {}
    ~Node() { std::cout << "free " << value << '\n'; }
};

int main() {
    auto a = std::make_unique<Node>(1);       // sole owner
    // auto copy = a;                         // error: unique_ptr cannot be copied
    std::unique_ptr<Node> b = std::move(a);   // ownership transfers; a is now null

    auto s1 = std::make_shared<Node>(2);
    {
        auto s2 = s1;                         // two owners
        std::cout << s1.use_count() << '\n';  // 2
    }                                         // s2 destroyed: count back to 1
    std::weak_ptr<Node> w = s1;               // observes without owning
    if (auto locked = w.lock())               // a shared_ptr if the object is still alive
        std::cout << locked->value << '\n';   // 2
}   // locals die in reverse order: prints "free 2", then "free 1"
```

**Reference cycles** are how `shared_ptr` leaks:

```cpp
struct Person {
    std::string name;
    std::shared_ptr<Person> partner;    // owning in both directions: a cycle
    // std::weak_ptr<Person> partner;   // fix: make one direction non-owning
};

void makeCouple() {
    auto a = std::make_shared<Person>();
    auto b = std::make_shared<Person>();
    a->partner = b;
    b->partner = a;
}   // a and b are gone, but each object is still owned by the other: never freed
```

- Prefer `make_unique` (C++14) and `make_shared`: no naked `new`, exception safe, and `make_shared` puts the object and its control block in one allocation.
- Functions that only use an object should take `T&` or `T*`; take a smart pointer only to share or transfer ownership. Never build two `shared_ptr`s from one raw pointer (two control blocks, double delete).
- A typical tree: `unique_ptr` for children, a raw non-owning pointer back to the parent. `auto_ptr` was deprecated in C++11 and removed in C++17.

## 9. Move semantics and rvalue references

Before C++11, returning or inserting a large object (a vector with a million elements) meant a deep copy, even when the source was a temporary about to die. Moving steals the source's buffer instead: O(1) instead of O(n).

| | lvalue | rvalue |
|---|---|---|
| Meaning | has identity: a name or address you can take | a temporary about to expire |
| Examples | `x`, `v[0]`, `*p`, a function returning `T&` | `42`, `x + 1`, `std::string("hi")`, a function returning `T` by value |
| Binds to | `T&`, `const T&` | `T&&`, `const T&` |

```cpp
#include <algorithm>
#include <cstddef>
#include <utility>

class Buffer {
public:
    explicit Buffer(std::size_t n) : size_(n), data_(new int[n]()) {}
    ~Buffer() { delete[] data_; }

    Buffer(const Buffer& other)                 // copy: allocate and copy n ints, O(n)
        : size_(other.size_), data_(new int[other.size_]) {
        std::copy(other.data_, other.data_ + size_, data_);
    }

    Buffer(Buffer&& other) noexcept             // move: steal the pointer, O(1)
        : size_(other.size_), data_(other.data_) {
        other.size_ = 0;
        other.data_ = nullptr;                  // the source must stay safe to destroy
    }
    // assignment operators: see the Rule of 5 in the OOP file
private:
    std::size_t size_;
    int* data_;
};

Buffer a(1'000'000);
Buffer b = a;              // copy constructor: a is an lvalue
Buffer c = std::move(a);   // move constructor: a is now empty but valid
Buffer d = Buffer(10);     // C++17 guaranteed elision: built directly in d, no copy or move
```

- `std::move(x)` moves nothing. It is a cast to `T&&` meaning "you may steal from x"; overload resolution then picks the move constructor or move assignment. Afterwards, only assign to or destroy `x`.
- `std::move` on a `const` object silently copies: `const T&&` cannot bind to `T&&`, so the copy constructor is chosen.
- Moves happen automatically for temporaries and when returning a local by value (often the copy is elided entirely). Don't write `return std::move(local);`: it blocks that elision.
- `std::vector` moves elements during reallocation only if the move constructor is `noexcept`; otherwise it copies to keep its strong exception guarantee.

### Going deeper
In a template, `T&&` is a **forwarding reference** that binds to anything, and `std::forward<T>(x)` preserves the caller's value category ("perfect forwarding"). That is how `emplace_back` and `make_unique` pass constructor arguments through unchanged.

## 10. auto, range-based for and lambdas

```cpp
std::vector<int> nums = {5, 2, 8, 1};
for (int n : nums) {}              // copies each element (fine for int)
for (auto& n : nums) n *= 2;       // reference: modifies the elements
for (const auto& n : nums) {}      // read-only, no copies: the default for strings and objects

auto it = std::find(nums.begin(), nums.end(), 16);   // std::vector<int>::iterator
auto total = 0LL;                                    // long long
```

`auto` deduces from the initializer and drops references and top-level `const`, so `auto x = names[0];` copies the string. Write `auto&` or `const auto&` when you want a reference.

```cpp
int threshold = 3;
int calls = 0;

auto above = [threshold](int x) { return x > threshold; };   // copies threshold right now
auto bump = [&calls] { ++calls; };                            // refers to calls
auto counter = [n = 0]() mutable { return ++n; };             // init-capture (C++14); mutable edits its copy

std::sort(nums.begin(), nums.end(), [](int a, int b) { return a > b; });   // descending
auto big = std::count_if(nums.begin(), nums.end(), above);
bump(); bump();                            // calls == 2
counter(); counter();                      // returns 1, then 2
std::function<bool(int)> pred = above;     // type-erased holder for storing callbacks (<functional>)
```

| Capture | Meaning |
|---|---|
| `[]` | nothing |
| `[x]` / `[=]` | copy of `x` / of everything used, taken when the lambda is created |
| `[&x]` / `[&]` | reference to `x` / to everything used |
| `[this]` | the object pointer (members are reached through it) |
| `[v = std::move(vec)]` | init-capture: a new member initialized from any expression |

- A lambda is an unnamed class with an `operator()` (a **functor**); captures are its data members. By-value captures are const unless the lambda is `mutable`.
- Danger: a lambda that captures locals by reference and outlives them (returned, stored as a callback, run on another thread) holds dangling references. Capture by value for anything that escapes the scope.

## 11. std::string and string_view

```cpp
std::string s = "hello";
s += " world";                 // append
s.size();                      // 11 (same as length())
s.substr(0, 5);                // "hello": (position, count)
s.find("world");               // 6, or std::string::npos if absent
s[0] = 'H';                    // strings are mutable
bool same = (s == "Hello world");   // true: compares contents, unlike C's ==
std::to_string(42);            // "42"
std::stoi("123");              // 123; throws std::invalid_argument for "abc"
s.c_str();                     // const char* for C APIs
```

- `s = s + c` in a loop creates a new string every time (O(n^2) overall). Use `s += c`, and `reserve` when you know the size.
- `std::string_view` (C++17) is a non-owning pointer plus length: cheap for read-only parameters, but never return or store a view of a temporary string.

## 12. Exceptions

```cpp
#include <iostream>
#include <stdexcept>

double divide(double a, double b) {
    if (b == 0) throw std::invalid_argument("division by zero");
    return a / b;
}

int main() {
    try {
        std::cout << divide(10, 2) << '\n';      // 5
        std::cout << divide(1, 0) << '\n';       // throws before anything is printed
    } catch (const std::invalid_argument& e) {   // most specific handler first
        std::cerr << "bad input: " << e.what() << '\n';
    } catch (const std::exception& e) {          // any standard exception
        std::cerr << "error: " << e.what() << '\n';
    } catch (...) {                              // anything else
        std::cerr << "unknown error\n";
    }
}
```

- Throw by value, catch by `const&`: no extra copies and no slicing of derived exception types.
- **Stack unwinding**: while an exception propagates, the destructors of all fully constructed locals run. That is why RAII code is exception-safe. If nothing catches the exception, `std::terminate` ends the program.
- Destructors are implicitly `noexcept`; an exception escaping one calls `std::terminate`. A constructor reports failure by throwing: its own destructor will not run, but already-built members are destroyed.
- `noexcept` promises no exception escapes (otherwise `std::terminate`). Use it on move operations, `swap` and destructors.
- Standard hierarchy: `std::exception` -> `std::logic_error` (`invalid_argument`, `out_of_range` from `vector::at`) and `std::runtime_error`, plus `std::bad_alloc` from `new`.
- Going deeper: the guarantees are no-throw, strong (succeeds or leaves state unchanged, like `push_back`) and basic (no leaks, objects still valid).

## 13. The meanings of static

| Where | Meaning |
|---|---|
| Variable or function at namespace scope | internal linkage: visible only in this `.cpp` (an unnamed `namespace { }` does the same) |
| Local variable | initialized on first use (thread-safe since C++11), lives until the program ends |
| Data member | one copy shared by all objects; define it in one `.cpp`, or use `inline static` (C++17) |
| Member function | no `this`; touches only static members; called as `Class::f()` |

```cpp
class Logger {
public:
    static Logger& instance() {          // "Meyers singleton"
        static Logger log;               // built on the first call, thread-safe
        return log;
    }
    void write(const std::string& msg) { ++count_; std::cout << msg << '\n'; }
    static int count() { return count_; }
private:
    Logger() = default;
    static inline int count_ = 0;        // C++17: defined right here, no .cpp definition needed
};

Logger::instance().write("ready");
```

## 14. nullptr, enum class, structured bindings, optional, variant

```cpp
void f(int);
void f(char*);
f(NULL);       // NULL is an integer constant: picks f(int) or is ambiguous, depending on the compiler
f(nullptr);    // f(char*): nullptr has its own type, std::nullptr_t

enum Color { Red, Green };                      // unscoped: names leak, converts to int silently
enum class Status : char { Active, Banned };    // scoped, with a chosen underlying type
Status st = Status::Active;
// int n = st;                                  // error: no implicit conversion
int n = static_cast<int>(st);                   // 0
```

```cpp
#include <iostream>
#include <map>
#include <optional>
#include <string>
#include <variant>
#include <vector>

std::optional<int> indexOf(const std::vector<int>& v, int target) {
    for (std::size_t i = 0; i < v.size(); ++i)
        if (v[i] == target) return static_cast<int>(i);
    return std::nullopt;                                 // "no value" without a magic -1
}

int main() {
    std::map<std::string, int> stock = {{"apple", 3}, {"pear", 0}};
    for (const auto& [name, qty] : stock)                // structured bindings
        std::cout << name << ": " << qty << '\n';
    auto [pos, inserted] = stock.insert({"kiwi", 5});    // pair<iterator, bool>
    if (auto it = stock.find("pear"); it != stock.end()) // if with an initializer
        std::cout << it->second << '\n';

    int idx = indexOf({4, 7}, 7).value_or(-1);           // 1

    std::variant<int, std::string> result = 42;          // type-safe union: one of the listed types
    result = std::string("timeout");
    if (std::holds_alternative<std::string>(result))
        std::cout << std::get<std::string>(result) << '\n';   // the wrong type throws std::bad_variant_access
}
```

In your web stack: `std::optional` is a checked version of returning `nil` or `undefined`, and `std::variant` is the C++ cousin of a TypeScript discriminated union (`std::visit` plays the exhaustive `switch`).

## 15. Compiling and debugging with g++

```bash
g++ -std=c++17 -Wall -Wextra -O2 main.cpp -o main                               # optimized build
g++ -std=c++17 -Wall -Wextra -g -fsanitize=address,undefined main.cpp -o main   # debug build
./main < input.txt
gdb ./main      # break main, run, next, step, print x, bt (backtrace after a crash)
```

| Flag | Purpose |
|---|---|
| `-std=c++17` | language standard (`c++20` for newer features) |
| `-Wall -Wextra` | warnings: uninitialized variables, sign comparisons, unused values, missing returns |
| `-g` | debug symbols for gdb and readable sanitizer traces |
| `-O0` / `-O2` | no optimization (debugging) / optimized |
| `-fsanitize=address` | catches out-of-bounds access, use after free and leaks at runtime |
| `-fsanitize=undefined` | catches signed overflow, invalid shifts, null dereference and similar UB |
| `-D_GLIBCXX_DEBUG` | bounds-checked containers and iterators in GCC's standard library |

On Windows, g++ comes from MinGW-w64 (for example through MSYS2) or WSL. Sanitizer support in MinGW builds is limited, so WSL is the easy route. MSVC uses `cl /std:c++17 /EHsc /W4` and supports `/fsanitize=address`.

## 16. Undefined behavior in C++

Everything from [C](01-c-essentials.md) applies (out of bounds, use after free, signed overflow, uninitialized reads), plus:

```cpp
std::vector<int> v = {1, 2, 3};
int& first = v[0];
v.push_back(4);          // may reallocate...
first = 10;              // ...so this can write to freed memory

int* p = new int[5];
delete p;                // new[] requires delete[]

std::string_view sv = std::string("temp");   // the temporary dies at the end of this line
std::cout << sv;                             // dangling view
```

Also: dereferencing `end()`, deleting a derived object through a base pointer without a virtual destructor, and data races between threads. Sanitizers catch most of these.

## Common pitfalls
- Range-for without `&` over strings or objects (copies everything, and edits go to the copies).
- Returning a reference or pointer to a local, or a `string_view` of a temporary.
- `delete` vs `delete[]` mismatches, mixing `malloc` with `delete`.
- Using an object after `std::move`, or calling `std::move` on a `const` object and expecting a move.
- A move constructor without `noexcept`, so `vector` growth silently copies.
- Lambdas capturing locals by reference and running later.
- `shared_ptr` cycles; two `shared_ptr`s created from one raw pointer; `using namespace std;` in a header.
- Mixing `cin >>` with `getline` without skipping the leftover newline; `int * int` overflowing before it reaches a `long long` (write `1LL * a * b`).

## How to talk about this in an interview
- Frame it honestly: "C++ is the language I used for OOP and DSA at university, and I was a TA for the OOP and Data Structures labs. Day to day I now work in Rails and TypeScript, but I'm comfortable with pointers, memory and object lifetimes."
- TA experience is a strength here: you have seen the classic bugs up close (dangling pointers, missing `delete[]`, shallow copies, uninitialized variables) and explained them. Use one as a concrete example.
- Map concepts to your stack when it helps: RAII vs Ruby blocks and `ensure`, smart pointers vs garbage collection, `const&` parameters vs passing object references in JavaScript.
- Be clear about depth: confident with C++11/14/17 (smart pointers, move semantics, lambdas, `auto`, structured bindings, `optional`); lighter on C++20 (concepts, ranges, coroutines).
- > **Fill in:** which compiler and IDE your labs used (g++ through Code::Blocks or Dev-C++, Visual Studio), and the most common student bug you helped debug.

## Interview questions

**Q1. What is the difference between a pointer and a reference?**
A reference is an alias: it must be initialized, cannot be null in valid code and cannot be reseated, and you use it with normal object syntax. A pointer is an object holding an address: it can be null, reassigned and used in arithmetic, and needs `*` or `->`. Use references when there is always an object; use pointers when "no object" is valid or you need to reseat.
*Follow-up:* Can you have an array of references? No: references are not objects, so they cannot be stored in arrays (use `std::reference_wrapper`).

**Q2. Stack vs heap allocation in C++: when do you use each?**
The stack holds locals with automatic lifetime: allocation is nearly free and cleanup is automatic, but the space is small (often 1 MB on Windows, 8 MB on Linux). The heap holds objects created with `new`, `make_unique` or inside containers: size is limited only by memory and lifetime is under your control, but allocation is slower and something must free it. Default to stack objects and RAII; use the heap for large or variable-size data, objects that outlive the scope, and polymorphic objects.
*Follow-up:* Where do a `std::vector`'s elements live? On the heap; the vector object itself (pointer, size, capacity) can sit on the stack.

**Q3. Compare `malloc`/`free` with `new`/`delete`.**
`new` allocates and runs the constructor, returns a typed pointer and throws `std::bad_alloc` on failure; `delete` runs the destructor, then frees. `malloc` returns raw `void *` bytes (or `NULL`) and `free` only releases them: no constructors or destructors. Never mix the pairs, and match `new[]` with `delete[]`.
*Follow-up:* What if you `free` an object created with `new`? Undefined behavior: the destructor never runs and the allocators may differ.

**Q4. What is RAII and why does C++ rely on it?**
Resource Acquisition Is Initialization ties a resource to an object's lifetime: the constructor acquires (memory, file, lock, socket) and the destructor releases. Destructors run automatically at scope exit, including during exception unwinding, so cleanup cannot be forgotten on early returns or errors. C++ has no garbage collector, so RAII is its answer to resource management.

**Q5. Explain `unique_ptr`, `shared_ptr` and `weak_ptr`.**
`unique_ptr` is exclusive, move-only ownership with no overhead: the default choice. `shared_ptr` shares ownership through a reference count in a control block; the last owner deletes the object. `weak_ptr` observes a shared object without owning it; `lock()` returns a `shared_ptr` if it is still alive. `weak_ptr` breaks cycles and suits caches and observers.
*Follow-up:* Is `shared_ptr` thread-safe? Its count updates are atomic, but access to the pointed-to object is not synchronized.

**Q6. How can `shared_ptr` leak memory?**
Through cycles: if A holds a `shared_ptr` to B and B holds one to A, neither count reaches zero after outside owners disappear. Make one direction a `weak_ptr` (typically child-to-parent). A related bug is constructing two `shared_ptr`s from the same raw pointer, which double-deletes.

**Q7. What are move semantics, and what does `std::move` actually do?**
Moving transfers resources, such as a heap buffer pointer, from an object that no longer needs them instead of deep-copying: O(1) instead of O(n). `std::move` does nothing at runtime; it casts to an rvalue reference so overload resolution picks the move constructor or move assignment. The moved-from object stays valid but unspecified.
*Follow-up:* When does the compiler move without `std::move`? For temporaries and when returning a local by value (if the copy is not elided outright).

**Q8. What is the difference between an lvalue and an rvalue?**
An lvalue has identity: a name or address you can take (`x`, `v[i]`, `*p`). An rvalue is a temporary about to expire (`42`, `a + b`, a function returning by value). `T&` binds only to lvalues, `T&&` only to rvalues, and `const T&` to both, which is why const references accept temporaries.

**Q9. Why should move constructors be `noexcept`?**
When `std::vector` grows it must keep its strong exception guarantee. If an element's move constructor could throw, a failure halfway would leave the old buffer partly moved-from, so the vector copies instead. With `noexcept`, it moves, which is far cheaper for heavy elements.

**Q10. What does a const member function guarantee?**
It cannot modify the object's state (except `mutable` members): inside it, `this` points to const. Only const member functions can be called on const objects or through const references, so every getter should be const.

**Q11. Can you overload functions by return type only?**
No. Overloads must differ in the number or types of parameters (or in the const qualifier of a member function); the return type plays no part in choosing a call. Ambiguous calls such as `area(5L)` with `area(int)` and `area(double)` are compile errors.

**Q12. What does `static` mean in C++?**
At namespace scope: internal linkage. On a local variable: one instance, initialized on first use (thread-safe since C++11), alive until program exit. On a data member: one copy shared by all objects. On a member function: no `this`, only static members are accessible, called as `Class::f()`.

**Q13. Why use `nullptr` instead of `NULL`?**
`NULL` is an integer constant, so `f(NULL)` may call an `int` overload or be ambiguous. `nullptr` has its own type, `std::nullptr_t`, converts only to pointer types, and behaves correctly in overloads and templates.

**Q14. Explain lambda captures. What is the risk with `[&]`?**
`[x]` copies `x` when the lambda is created, `[&x]` stores a reference, `[=]` and `[&]` capture everything used, and `[this]` captures the object pointer. Value captures are read-only unless the lambda is `mutable`. With `[&]`, a lambda that outlives the scope (stored callback, returned lambda, another thread) reads dangling references: undefined behavior.

**Q15. What happens when an exception is thrown? Can destructors throw?**
The runtime unwinds the stack to the nearest matching `catch`, destroying every fully constructed local on the way; with no handler, `std::terminate` is called. Destructors are implicitly `noexcept`, so an exception escaping one terminates the program (two active exceptions during unwinding cannot be handled). Catch and handle errors inside the destructor.

**Q16. Why prefer `make_unique` and `make_shared` over `new`?**
They avoid naked `new`, create the object and its owner in one expression (no leak window) and read better. `make_shared` also allocates the object and control block together, which is faster and more cache-friendly. Trade-off: with `make_shared`, that single block stays allocated until the last `weak_ptr` is gone.

**Q17. What is the difference between `delete` and `delete[]`?**
`delete p` destroys one object; `delete[] p` destroys every element of an array from `new T[n]`, using a count the runtime stored. Mismatching them is undefined behavior (often only one destructor runs, or the heap is corrupted). Prefer `std::vector` or `std::unique_ptr<T[]>`.

**Q18. How do you find memory bugs and undefined behavior in C++ code?**
Compile with `-Wall -Wextra` and fix the warnings, run debug builds with `-g -fsanitize=address,undefined`, use `-D_GLIBCXX_DEBUG` for checked containers, Valgrind on Linux, and gdb for backtraces. Then shrink the failing input until the bug is obvious.
