# C and C++: Rapid-Fire Interview Questions

> How to use: cover the answer, say yours out loud, then check. Deeper explanations live in the topic files linked under each section. Output questions assume a typical 64-bit Linux or Windows compiler (4-byte `int`, 8-byte pointers).

## C essentials (see [C essentials](01-c-essentials.md))

**Q1. What are the four stages of building a C program?**
Preprocessing (expand `#include` and macros, conditional compilation), compilation (C to assembly, type checking), assembly (object file), linking (combine object files and libraries, resolve symbols).

**Q2. What is the difference between a declaration and a definition?**
A declaration introduces a name and its type (`extern int count;`, `int add(int, int);`). A definition also allocates storage or provides the body. You may declare a name many times but define it only once (the one-definition rule).

**Q3. What does `sizeof` return, and when is it evaluated?**
The size in bytes, as a `size_t`, computed at compile time (except for C variable-length arrays). `sizeof(char)` is always 1.

**Q4. Null pointer vs wild pointer vs dangling pointer?**
Null: deliberately points nowhere and can be tested. Wild: never initialized, holds garbage. Dangling: once valid, but the memory was freed or went out of scope. Dereferencing any of them is undefined behavior.

**Q5. What is the difference between `char s[] = "hi";` and `char *s = "hi";`?**
The first is a writable 3-byte array initialized from the literal. The second points at the literal itself, which lives in read-only storage, so writing through it is undefined (declare it `const char *`). `sizeof` gives 3 for the array and 8 for the pointer.

**Q6. Why does `scanf("%d", &x)` need the `&`?**
C passes arguments by value, so `scanf` needs the address of `x` to store the parsed value there. Forgetting it passes garbage as an address: undefined behavior.

**Q7. What does `volatile` mean?**
The value may change outside the program's visible control flow (hardware registers, signal handlers), so the compiler must perform every read and write. It does not make code thread-safe; use atomics or locks for that.

**Q8. How is `typedef` different from `#define` for type aliases?**
`typedef` is handled by the compiler, respects scope and works for complex types such as function pointers. `#define` is text replacement: with `#define PTR int*`, the line `PTR a, b;` makes only `a` a pointer, whereas `typedef int *Ptr; Ptr a, b;` makes both pointers.

**Q9. What happens if you `free` the same pointer twice, or free memory that did not come from `malloc`?**
Both are undefined behavior: heap corruption, crashes, or exploitable security bugs. Set pointers to `NULL` after freeing (`free(NULL)` is safe).

**Q10. What are alignment and padding?**
CPUs access data fastest (sometimes only) at addresses that are multiples of its size. Compilers insert padding bytes so each struct member is aligned, plus trailing padding so arrays of the struct stay aligned. Ordering members from largest to smallest reduces padding.

**Q11. What is an `enum` in C, and how big is it?**
A set of named integer constants, starting at 0 unless assigned. Its size is implementation-defined, usually that of `int`, and C does not type-check enum values against other integers.

**Q12. What is the difference between `#include <file.h>` and `#include "file.h"`?**
Angle brackets search the system and configured include paths. Quotes search the including file's directory first, then fall back to the same paths as angle brackets.

**Q13. What is a segmentation fault?**
The OS stopping the process for touching memory it may not access: null dereference, out-of-bounds access into unmapped pages, writing to read-only memory, stack overflow. It is one possible symptom of undefined behavior, not a guaranteed one.

**Q14. `const int` vs `#define` for constants in C?**
`const` gives a typed, scoped variable that debuggers can see; `#define` is untyped text substitution. In C (unlike C++), a `const int` is not a constant expression, so it cannot size a file-scope array; that is why `#define` and `enum` constants are still common in C.

## C++ fundamentals (see [C++ fundamentals](02-cpp-fundamentals.md))

**Q15. Can a reference be null or be reseated?**
No. A reference must be initialized to an existing object and always refers to it; assigning to the reference assigns to that object. A "null reference" can only come from undefined behavior such as dereferencing a null pointer.

**Q16. What is the difference between `int&` and `const int&` parameters?**
`int&` lets the function modify the caller's variable and binds only to modifiable lvalues. `const int&` promises no modification and also binds to temporaries and literals, so `f(5)` works only with the const version.

**Q17. What does `new` do if allocation fails?**
Throws `std::bad_alloc`. `new (std::nothrow) T` returns `nullptr` instead. `malloc` returns `NULL`.

**Q18. What is the "most vexing parse"?**
`Widget w();` declares a function named `w` that returns a `Widget`; it does not create an object. Write `Widget w;` or `Widget w{};`.

**Q19. Why should you avoid `using namespace std;` in headers?**
It injects every `std` name into every file that includes the header, causing silent name clashes and ambiguous calls far from the cause. In a `.cpp` file or contest code it is acceptable.

**Q20. What does `inline` mean in modern C++?**
Mainly that a function or variable may be defined in several translation units without violating the one-definition rule, which allows definitions in headers. Actual inlining is the optimizer's decision.

**Q21. What is `constexpr`?**
It marks variables and functions that can be evaluated at compile time, so their results can size arrays or serve as template arguments. A `constexpr` function can still run at run time with non-constant arguments.

**Q22. What do `unique_ptr` and `shared_ptr` cost?**
`unique_ptr` with the default deleter is the size of a raw pointer with no runtime overhead. `shared_ptr` holds two pointers (object and control block), needs a heap-allocated control block (combined with the object in one allocation by `make_shared`), and updates its count atomically on every copy.

**Q23. How do you transfer ownership of a `unique_ptr`?**
Move it: `auto b = std::move(a);` leaves `a` null. Returning a local `unique_ptr` from a function moves it automatically. Copying does not compile.

**Q24. What does `weak_ptr::lock()` return?**
A `shared_ptr` that keeps the object alive while you use it, or an empty `shared_ptr` if the object has already been destroyed. Check it before use.

**Q25. What is an rvalue reference?**
A reference declared `T&&` that binds to temporaries (rvalues). It enables move constructors, move assignment and perfect forwarding.

**Q26. What state is a moved-from object in?**
Valid but unspecified for standard library types: you may destroy it or assign it a new value, but do not rely on its contents (a moved-from `std::string` is usually empty, but that is not guaranteed).

**Q27. What is copy elision (RVO)?**
The compiler constructs a returned object directly in the caller's storage, skipping the copy or move. Since C++17 it is guaranteed when returning a prvalue (`return Widget(...);`); for named locals (NRVO) it is optional but common.

**Q28. Given `const int& r = x;`, what type does `auto a = r;` deduce?**
Plain `int`: `auto` drops references and top-level `const`. Use `auto&` or `const auto&` to keep the reference.

**Q29. What is the difference between `[=]` and `[&]` captures?**
`[=]` copies every variable the lambda uses, freezing their values at creation. `[&]` references them, so the lambda sees later changes but dangles if it outlives those variables.

**Q30. What does `noexcept` do?**
It promises that no exception escapes the function; if one does, `std::terminate` is called. It lets containers move elements safely (for example during `vector` growth). Destructors are `noexcept` by default.

**Q31. When is `std::terminate` called?**
When an exception is not caught, when an exception escapes a `noexcept` function (including a destructor), and when a second exception is thrown out of a destructor during stack unwinding. Destroying a joinable `std::thread` also triggers it.

**Q32. Inside a `catch`, what is the difference between `throw;` and `throw e;`?**
`throw;` rethrows the original exception object with its real dynamic type. `throw e;` throws a new copy of `e` using its static type, slicing a derived exception if `e` was caught as a base reference.

**Q33. What is `std::string_view`, and what is its main danger?**
A non-owning pointer-plus-length view of characters, cheap to pass and copy. It dangles if the underlying string dies first, for example a view of a temporary `std::string` or of a local returned from a function.

**Q34. What is `std::optional` for?**
Representing "maybe a value" without sentinel values like `-1`, null pointers or heap allocation. Test with `has_value()` or in an `if`, read with `*opt` or `value()`, and supply a fallback with `value_or(x)`.

## C++ OOP (see [C++ OOP](03-cpp-oop.md))

**Q35. How does C++ express the four pillars of OOP?**
Encapsulation through access specifiers, abstraction through abstract classes, inheritance through base classes (`: public Base`), and polymorphism through overloading and templates (compile time) and virtual functions (run time). See [OOP concepts](../../01-cs-fundamentals/02-oop.md) for the language-neutral view.

**Q36. Which special member functions can the compiler generate?**
Default constructor, destructor, copy constructor, copy assignment, move constructor and move assignment (the last two since C++11).

**Q37. When is the copy constructor called?**
When an object is initialized from another of the same type (`T b = a;`, `T b(a);`), when passing or returning by value (unless the copy is elided), and when throwing or catching by value.

**Q38. What is a converting constructor, and how do you prevent implicit conversions?**
A constructor callable with a single argument that is not marked `explicit`: the compiler may use it to convert silently, as in `void pay(Money m); pay(500);`. Marking it `explicit` forces callers to write `pay(Money(500))`.

**Q39. What is a delegating constructor?**
A constructor that calls another constructor of the same class in its initializer list (C++11), such as `User() : User("guest", 0) {}`, which removes duplicated setup code.

**Q40. When is the member initializer list mandatory?**
For `const` members, reference members, and members or base classes without a default constructor. It is also more efficient in general, because members are initialized once instead of default-constructed and then assigned.

**Q41. Can you call a destructor explicitly?**
Yes (`obj.~T()`), but only for objects built with placement new into raw storage you manage. Doing it to a normal object leads to a second destruction at scope exit: undefined behavior.

**Q42. What is the constructor of an abstract class for, if the class cannot be instantiated?**
It initializes the base part whenever a derived object is constructed (setting members, validating arguments). It is often `protected` to make the intent obvious.

**Q43. What is the difference between a virtual and a pure virtual function?**
A virtual function has an implementation that derived classes may override. A pure virtual function (`= 0`) must be overridden by every concrete derived class and makes its class abstract.

**Q44. Can a static member function be virtual?**
No. It has no `this`, so there is no object whose dynamic type could select an override. For the same reason it cannot be `const`.

**Q45. Can a class template have virtual functions? Can a member function template be virtual?**
A class template can have virtual functions: each instantiation gets its own vtable. A member function template cannot be virtual, because the set of instantiations is unbounded and a vtable needs a fixed number of slots.

**Q46. Overloading vs overriding vs hiding?**
Overloading: same scope, same name, different parameters, chosen at compile time. Overriding: a derived class redefines a virtual function with the same signature, chosen at run time. Hiding: a derived class declares the same name (non-virtual, or with a different signature), so the base versions become invisible through the derived type.

**Q47. What does `protected` mean, and when should you use it?**
Accessible to the class and its derived classes, but not to outside code. Use it for hooks meant for subclasses (for example in the template method pattern); keep data members private and expose protected functions instead.

**Q48. In what order are constructors called with multiple inheritance?**
Virtual bases first, then direct non-virtual bases in the order they are listed in the class declaration, then members in declaration order, then the constructor body. Destructors run in the reverse order.

**Q49. What is a pure virtual destructor, and why must it have a body?**
`virtual ~Base() = 0;` makes a class abstract when it has no other pure virtual function. Every derived destructor implicitly calls the base destructor, so it still needs a definition outside the class: `Base::~Base() {}`.

**Q50. Can a pure virtual function have an implementation?**
Yes. Derived classes must still override it to be concrete, but they can call the shared default explicitly as `Base::f()`.

**Q51. What is the "virtual constructor" idiom?**
A virtual `clone()` member that returns a copy of the object's dynamic type, usually as `std::unique_ptr<Base>`. It lets you copy objects you only know through a base pointer.

**Q52. When is RTTI unavailable?**
`dynamic_cast` downcasts and `typeid` on dynamic types only work for polymorphic classes (at least one virtual function), and some builds disable RTTI entirely with `-fno-rtti`.

**Q53. What is a friend class?**
`friend class Inspector;` inside `Widget` gives every member function of `Inspector` access to `Widget`'s private members. The grant is one-way: `Widget` gets nothing in return.

**Q54. Composition vs inheritance in C++ terms?**
Composition stores another object as a member (has-a), and its lifetime is tied to the owner. Inheritance embeds a base subobject (is-a) and exposes the base interface. Prefer composition for code reuse, and inheritance when you need substitutability through base pointers or references.

## STL (see [STL for DSA](04-stl-for-dsa.md))

**Q55. Which headers provide `sort`, `accumulate` and `priority_queue`?**
`<algorithm>`, `<numeric>` and `<queue>`.

**Q56. What do `set::insert`, `unordered_set::insert` and inserting into the middle of a `vector` cost?**
O(log n); O(1) average and O(n) worst; O(n) because later elements shift.

**Q57. How is `std::set` implemented, and why is it sorted?**
As a balanced binary search tree, in practice a red-black tree. Iterating walks the tree in order, which visits keys in sorted order.

**Q58. How do you find the k-th smallest element efficiently?**
`nth_element` in O(n) average time; a max-heap of size k in O(n log k); or sorting in O(n log n).

**Q59. How do you iterate a `map` in descending key order?**
Use reverse iterators, `for (auto it = m.rbegin(); it != m.rend(); ++it)`, or declare the map as `map<K, V, greater<K>>`.

**Q60. What does `erase` return for a `vector` or a `map`?**
An iterator to the element after the erased one (for associative containers since C++11), which makes `it = c.erase(it)` the safe erase-while-iterating pattern.

**Q61. How do you check whether a key exists in a `map` without inserting it?**
`m.find(k) != m.end()`, `m.count(k)`, or `m.contains(k)` in C++20. Never `m[k]`, which inserts.

**Q62. What is the difference between `size()` and `capacity()`?**
`size()` is the number of elements stored; `capacity()` is how many fit before the next reallocation.

**Q63. How do you sort a `vector<pair<int, int>>` by the second element, descending?**
`sort(v.begin(), v.end(), [](const auto& a, const auto& b) { return a.second > b.second; });` (generic lambdas need C++14).

**Q64. What does `accumulate(v.begin(), v.end(), 0)` return for a `vector<double>`?**
An `int`: the accumulator has the type of the initial value, so each partial sum is truncated. Pass `0.0`.

**Q65. `std::array` vs a C array vs `std::vector`?**
`std::array<T, N>` has a fixed compile-time size like a C array but knows its size, can be copied and returned, and does not decay to a pointer. `std::vector` is dynamically sized and stores its elements on the heap.

**Q66. What does `map::insert` do if the key already exists?**
Nothing: it returns a `pair` whose iterator points to the existing element and whose `bool` is `false`. To overwrite, use `m[k] = v` or `insert_or_assign` (C++17).

## What's the output?

**Q67. What does this print?**
```cpp
#include <iostream>
void f(int a[]) { std::cout << sizeof(a) << ' '; }
int main() {
    int a[10];
    int* p = a;
    std::cout << sizeof(a) << ' ' << sizeof(p) << ' ';
    f(a);
}
```
`40 8 8`: the array is 10 x 4 bytes, while `p` and the decayed parameter are pointers.

**Q68. What does this print?**
```c
char s[] = "hello";
const char *t = "hello";
printf("%zu %zu %zu\n", sizeof(s), strlen(s), sizeof(t));
```
`6 5 8`: the array includes the `'\0'`, `strlen` does not count it, and `t` is a pointer.

**Q69. What does this print?**
```cpp
int a[] = {10, 20, 30, 40};
int* p = a + 1;
std::cout << *p << ' ' << *(p + 2) << ' ' << p[-1] << ' ' << (&a[3] - p);
```
`20 40 10 2`: `p` points at `a[1]`, `p[-1]` is `a[0]`, and pointer subtraction counts elements.

**Q70. What does this print?**
```cpp
struct A { A() { std::cout << "A"; } ~A() { std::cout << "~A"; } };
struct B : A { B() { std::cout << "B"; } ~B() { std::cout << "~B"; } };
int main() { B b; }
```
`AB~B~A`: base constructed first, destroyed last.

**Q71. What does this print?**
```cpp
struct M { M(int i) { std::cout << "M" << i << ' '; } };
struct Base { Base() { std::cout << "Base "; } };
struct D : Base {
    M m2, m1;
    D() : m1(1), m2(2) { std::cout << "D"; }
};
int main() { D d; }
```
`Base M2 M1 D`: the base comes first, then members in declaration order (`m2` before `m1`), whatever the initializer list says.

**Q72. What does this print?**
```cpp
struct N {
    int id;
    N(int i) : id(i) { std::cout << '+' << id; }
    ~N() { std::cout << '-' << id; }
};
int main() { N a(1); N b(2); { N c(3); } N d(4); }
```
`+1+2+3-3+4-4-2-1`: `c` dies at its closing brace; the rest die in reverse order at the end of `main`.

**Q73. What does this print?**
```cpp
struct Base {
    virtual void f() { std::cout << "Base::f "; }
    void g() { std::cout << "Base::g "; }
};
struct Derived : Base {
    void f() override { std::cout << "Derived::f "; }
    void g() { std::cout << "Derived::g "; }
};
int main() { Derived d; Base* p = &d; p->f(); p->g(); }
```
`Derived::f Base::g`: `f` is virtual (dynamic type), `g` is not (static type `Base*`).

**Q74. What does this print?**
```cpp
struct Base { virtual std::string name() const { return "Base"; } };
struct Derived : Base { std::string name() const override { return "Derived"; } };
void byValue(Base b) { std::cout << b.name() << ' '; }
void byRef(const Base& b) { std::cout << b.name() << ' '; }
int main() { Derived d; byValue(d); byRef(d); }
```
`Base Derived`: passing by value slices the object down to a `Base`.

**Q75. What does this print?**
```cpp
struct Base {
    Base() { hello(); }
    virtual void hello() { std::cout << "Base "; }
};
struct Derived : Base {
    void hello() override { std::cout << "Derived "; }
};
int main() { Derived d; d.hello(); }
```
`Base Derived`: during `Base`'s constructor the object is still a `Base`, so the virtual call does not reach `Derived`.

**Q76. What does this print?**
```cpp
struct Base { virtual void show(int x = 1) { std::cout << "Base " << x; } };
struct Derived : Base { void show(int x = 2) override { std::cout << "Derived " << x; } };
int main() { Derived d; Base* p = &d; p->show(); }
```
`Derived 1`: the function is chosen at run time, but default arguments come from the static type (`Base*`). Never give overrides different defaults.

**Q77. What does this print?**
```cpp
struct Base { ~Base() { std::cout << "~Base "; } };
struct Derived : Base { ~Derived() { std::cout << "~Derived "; } };
int main() { Base* p = new Derived; delete p; }
```
Undefined behavior, because the destructor is not virtual; in practice usually just `~Base`. With `virtual ~Base()` it prints `~Derived ~Base`.

**Q78. What does this print?**
```cpp
void f() {
    static int a = 0;
    int b = 0;
    a++; b++;
    std::cout << a << b << ' ';
}
int main() { f(); f(); f(); }
```
`11 21 31`: the static local keeps its value between calls; the ordinary local starts at 0 each time.

**Q79. What does this print?**
```cpp
int i = 5;
int a = i++;
int b = ++i;
std::cout << a << ' ' << b << ' ' << i;
```
`5 7 7`: post-increment yields the old value, pre-increment the new one.

**Q80. What does this print?**
```cpp
unsigned int u = 0;
u = u - 1;
std::cout << u << '\n';
int x = INT_MAX;
x = x + 1;
std::cout << x << '\n';
```
First line `4294967295`: unsigned arithmetic wraps modulo 2^32. The second is signed overflow, which is undefined behavior: it often prints `-2147483648`, but the optimizer is allowed to assume it never happens.

**Q81. What does this print?**
```cpp
int a = -1;
unsigned int b = 1;
std::cout << (a < b ? "less" : "not less");
```
`not less`: `a` is converted to unsigned (4294967295) before the comparison. `-Wall` warns about signed/unsigned comparisons.

**Q82. What does this print?**
```cpp
std::cout << 7 / 2 << ' ' << 7 / 2.0 << ' ' << -7 / 2 << ' ' << -7 % 3;
```
`3 3.5 -3 -1`: integer division truncates toward zero, and the remainder takes the sign of the dividend.

**Q83. What does this print?**
```c
#define SQ(x) x * x
printf("%d\n", SQ(2 + 3));
```
`11`: the macro expands to `2 + 3 * 2 + 3`. Parenthesize: `#define SQ(x) ((x) * (x))`.

**Q84. What does this print?**
```cpp
int x = 1, y = 2;
int& r = x;
r = y;
r++;
std::cout << x << ' ' << y;
```
`3 2`: `r = y` copies 2 into `x` (references never reseat), then `r++` increments `x`.

**Q85. What does this print?**
```cpp
struct T {
    T() { std::cout << "D "; }
    T(const T&) { std::cout << "C "; }
    T& operator=(const T&) { std::cout << "A "; return *this; }
};
int main() { T a; T b = a; T c; c = a; }
```
`D C D A`: `T b = a;` is copy construction even though it uses `=`; only `c = a;` is assignment.

**Q86. What does this print?**
```cpp
int x = 1;
auto byVal = [x] { return x; };
auto byRef = [&x] { return x; };
x = 10;
std::cout << byVal() << ' ' << byRef();
```
`1 10`: the value capture was copied when the lambda was created.

**Q87. What does this print?**
```cpp
struct B { void f(int) { std::cout << "B"; } };
struct D : B { void f(double) { std::cout << "D"; } };
int main() { D d; d.f(1); }
```
`D`: `D::f(double)` hides `B::f(int)`, and `1` converts to `double`.

**Q88. What does this print?**
```cpp
struct E {};
struct V { virtual void f() {} };
std::cout << sizeof(E) << ' ' << sizeof(V);
```
`1 8`: an empty class still occupies one byte so distinct objects have distinct addresses, and a polymorphic class carries a vptr.

**Q89. What does this print?**
```cpp
try { throw 42; }
catch (double) { std::cout << "double"; }
catch (int) { std::cout << "int"; }
```
`int`: catch matching does not apply arithmetic conversions (only derived-to-base and qualification conversions).

**Q90. What does this print?**
```cpp
int i = 1;
int j = i++ + ++i;
std::cout << j;
```
Trick question: undefined behavior. `i` is modified twice in one expression with no sequencing between the operands of `+`, so compilers may print different values. Recognizing it is the answer.

## Spot the bug

**Q91. What is wrong here?**
```cpp
const std::string& greeting() {
    std::string s = "hi";
    return s;
}
```
It returns a reference to a local that is destroyed when the function returns, so the caller gets a dangling reference. Return `std::string` by value (copy elision makes it cheap).

**Q92. What is wrong here?**
```cpp
class Buffer {
    int* data_;
public:
    explicit Buffer(int n) : data_(new int[n]) {}
    ~Buffer() { delete[] data_; }
};
Buffer a(10);
Buffer b = a;
```
The compiler-generated copy is shallow: `a` and `b` share one array, and both destructors delete it (double free). Follow the Rule of 3/5 with a deep copy, delete the copy operations, or hold a `std::vector<int>` (Rule of 0).

**Q93. What is wrong here?**
```cpp
for (auto it = v.begin(); it != v.end(); ++it)
    if (*it % 2 == 0) v.erase(it);
```
`erase` invalidates `it`, so the following `++it` is undefined behavior (and elements get skipped). Use `it = v.erase(it);` without incrementing in that branch, or the erase-remove idiom.

**Q94. What is wrong here?**
```c
p = realloc(p, new_size);
```
If `realloc` fails it returns `NULL`, overwriting the only pointer to the still-allocated old block: a leak. Assign to a temporary, check it, then update `p`.

**Q95. What is wrong here?**
```cpp
struct Shape { virtual double area() const = 0; };
struct Circle : Shape {
    std::vector<double> samples;
    double area() const override { return 0; }
};
std::vector<std::unique_ptr<Shape>> shapes;
shapes.push_back(std::make_unique<Circle>());
```
`Shape` has no virtual destructor, so when the vector destroys its elements it deletes `Circle` objects through `Shape*`: undefined behavior, and `samples` is never freed. Add `virtual ~Shape() = default;`.

**Q96. What is wrong here?**
```cpp
std::vector<int> v = {1, 2, 3};
int& first = v.front();
v.push_back(4);
std::cout << first;
```
`push_back` may reallocate, leaving `first` referring to freed memory. Re-read the element after modifying the vector, or keep an index instead of a reference.
