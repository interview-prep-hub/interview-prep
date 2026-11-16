# CS Fundamentals: Rapid-Fire Interview Questions

> How to use: cover the answer, say yours out loud, then check. Deeper explanations live in the topic files linked under each section. The C++ "what's the output" snippets assume `#include <iostream>`, `<string>` and `<vector>` plus `using namespace std;` (`<bits/stdc++.h>` is GCC-only, so keep it out of portable code).

## Programming fundamentals (see [Programming fundamentals](01-programming-fundamentals.md))

**Q1. Compiled vs interpreted vs JIT: where do C++, Ruby, JavaScript and Java fit?**
C++ is compiled ahead of time to native machine code. CRuby compiles source to bytecode that its VM (YARV) runs, and YJIT compiles hot methods to native code; V8 interprets JavaScript bytecode and JIT-compiles hot functions; Java compiles to JVM bytecode that the JVM interprets and JIT-compiles. "Compiled" and "interpreted" describe implementations, not languages.

**Q2. Static vs dynamic typing, and strong vs weak typing?**
Static typing checks types before the program runs (C++, Java, TypeScript); dynamic typing checks them at runtime (Ruby, Python, JavaScript). Strong vs weak is about implicit conversions: JavaScript evaluates `"5" * 2` to `10`, while Ruby raises `TypeError` for `"5" + 2`. TypeScript types are erased at runtime, so external input still needs runtime validation (Zod).

**Q3. How are arguments passed in Ruby and JavaScript compared with C++?**
Ruby and JavaScript pass object references by value ("pass by sharing"): mutating the object is visible to the caller, reassigning the parameter is not. C++ passes by value (a copy) unless the parameter is a reference (`T&`, an alias for the caller's variable) or a pointer. Use `const T&` for large read-only arguments.

**Q4. Stack vs heap?**
The stack holds per-thread call frames (locals, return addresses): allocation is just moving the stack pointer, memory is freed automatically on return, and it is small (often 8 MB on Linux, 1 MB on Windows). The heap holds dynamically allocated objects with flexible lifetimes, freed by `delete`, smart pointers or a garbage collector. In Ruby and JavaScript almost every object lives on the heap.

**Q5. What causes a stack overflow, and how do you fix it?**
A call chain deeper than the stack: usually recursion with a missing or unreachable base case, or recursion whose depth grows with the input (a DFS down a 100,000-node chain). Fix the base case or convert to iteration with an explicit stack. Ruby raises `SystemStackError`; JavaScript throws `RangeError: Maximum call stack size exceeded`.

**Q6. Can a garbage-collected program leak memory?**
Yes. A GC frees only unreachable objects, so anything still referenced stays: ever-growing class-level caches, event listeners and intervals never removed, closures capturing big objects, unbounded memoization. In C++ the classic leak is `new` without `delete`; RAII and `unique_ptr` prevent it, and `weak_ptr` breaks `shared_ptr` cycles.

**Q7. What is undefined behavior in C++?**
Code the standard places no requirements on: the program may crash, seem to work, or be optimized into something surprising. Examples: signed integer overflow, out-of-bounds indexing, dereferencing null or dangling pointers, reading uninitialized variables, data races. GCC and Clang sanitizers (`-fsanitize=address,undefined`) catch many of them.

**Q8. What do `-7 / 2` and `-7 % 2` give in C++ and in Ruby?**
C++ truncates toward zero: `-3` and `-1` (JavaScript's `%` also gives `-1`). Ruby and Python floor: `-4` and `1`. So `n % 2 == 1` misses negative odd numbers in C++ and JavaScript; test `n % 2 != 0` instead.

**Q9. Why is `0.1 + 0.2 != 0.3`, and what does that mean for money?**
IEEE 754 doubles store binary fractions, and 0.1 has no exact binary form, so the sum is `0.30000000000000004`. Compare floats with a tolerance, and never store money in floats: use integer minor units (cents) or a decimal type (Ruby `BigDecimal`, Python `Decimal`, Postgres `numeric`).

**Q10. Shallow copy vs deep copy?**
A shallow copy duplicates the outer object but shares nested objects; a deep copy duplicates recursively. Ruby `dup`/`clone` and JavaScript spread or `Object.assign` are shallow; `structuredClone` is deep (but cannot copy functions). A C++ class with a raw owning pointer gets a shallow copy from the default copy constructor, and two destructors then free the same memory.

**Q11. Value equality vs identity?**
Value equality compares contents; identity asks whether two references point to the same object. Ruby: `==` vs `equal?`. JavaScript: `===` compares objects by reference, so `{} === {}` is `false`. Java: `equals` vs `==`. React's change detection relies on identity, which is why state updates must create new objects.

**Q12. Why prefer immutable data?**
Immutable values can be shared across threads without locks, cached safely and compared cheaply by reference. Watch the shallow traps: JavaScript `const` freezes the binding, not the object, and both `Object.freeze` and Ruby `freeze` are shallow.

**Q13. Exceptions or error codes?**
Exceptions keep the happy path clean and cannot be silently ignored, but they hide control flow and are slow when thrown. Return values (`std::optional`, C++23 `std::expected`, result objects) make failure explicit at each call site. A common rule: exceptions for unexpected failures, return values for expected outcomes, like Rails `save` returning `false` while `save!` raises.

**Q14. What does the linker do, and what does "undefined reference" mean?**
The compiler turns each `.cpp` file into an object file; the linker combines object files and libraries and resolves symbols between them. "Undefined reference to `foo`" means `foo` was declared but its definition never reached the linker: a `.cpp` missing from the build, a missing library flag, or a function or static member declared but never defined.

**Q15. Recursion vs iteration: does tail-call optimization save you?**
Each recursive call costs a stack frame; a loop does not. C++ compilers may turn tail calls into jumps as an optimization, but nothing guarantees it, and Ruby, Python and most JavaScript engines do not do it by default. For deep inputs, write the loop.

## Object-oriented programming (see [OOP](02-oop.md))

**Q16. What are the four pillars of OOP? Give a web example of each.**
Encapsulation: an `Order` changes its total only through `add_item`, so the total can never disagree with the items. Abstraction: a `PaymentGateway` exposes `charge` and hides the provider's HTTP calls. Inheritance: `AdminUser < User`, or every model inheriting from `ApplicationRecord`. Polymorphism: a sync job calls `channel.sync_listings` on any sales channel without knowing which one it has.

**Q17. Class vs object?**
A class is a blueprint (a type) that defines state and behavior; an object is an instance with its own state and identity. In Ruby, classes are themselves objects (instances of `Class`), which is what makes class-level macros like `has_many` possible.

**Q18. Abstraction vs encapsulation?**
Abstraction is a design decision about what to expose (`charge(amount)`); encapsulation is the technique that hides and protects state (private members, methods that validate). You can encapsulate behind a poor abstraction; good designs do both.

**Q19. Abstract class vs interface?**
An abstract class can have state, constructors and implemented methods next to abstract ones, and in Java and C# a class extends only one. An interface is a contract without instance state (Java 8+ and C# 8+ allow default methods), and a class can implement many. C++ has no keyword: an "interface" is a class with only pure virtual functions and a virtual destructor. Use an abstract class for a family that shares code, an interface for a capability (`Comparable`, `Notifier`).

**Q20. Overloading vs overriding?**
Overloading: same name, different parameter lists, resolved at compile time from the argument types. Overriding: a subclass redefines a virtual method with the same signature, resolved at runtime from the object's type. Ruby and JavaScript have no overloading (the last definition wins), so they use default or keyword arguments instead.

**Q21. How does a virtual call work under the hood in C++?**
Each class with virtual functions has a vtable of function pointers, and each object stores a hidden vptr to its class's vtable. A virtual call loads the vptr, picks the slot and calls through it, so the object's dynamic type decides. Costs: one indirection, a pointer per object, and usually no inlining. (Vtables are an implementation technique, not a standard requirement, but every major compiler uses them.)

**Q22. Why does a polymorphic base class need a virtual destructor?**
Deleting a derived object through a base pointer whose destructor is not virtual is undefined behavior; in practice only the base destructor runs, so the derived part's resources leak. Rule: if a class has any virtual function, give it `virtual ~Base() = default;`.

**Q23. Can a constructor be virtual?**
No. During construction the object's type is fixed and its vptr is still being set up. For "virtual copying", add a virtual `clone()` that returns `std::unique_ptr<Base>`. Destructors can be virtual, and in polymorphic bases they must be.

**Q24. What do `override` and `final` do?**
`override` makes the compiler check that the function really overrides a base virtual, catching mismatches such as a missing `const`. `final` on a virtual function forbids further overriding; on a class it forbids inheriting from it, which also lets the compiler devirtualize calls.

**Q25. Dependency vs association vs aggregation vs composition?**
Dependency: A only uses B for a moment, such as a method parameter (`checkout(payment_method)`). Association: A keeps a link to B as a field (an `Order` knows its `Customer`). Aggregation: a whole has parts that live independently (a `Team` has `Player`s). Composition: the whole owns parts that die with it (an `Order` and its line items, `dependent: :destroy` in Rails).

**Q26. Why prefer composition over inheritance?**
Inheritance couples a subclass to its parent's internals (the fragile base class problem), is fixed at compile time, and explodes into combinations (`RetryingLoggingHttpClient`). Composition holds collaborators behind interfaces and delegates to them, so you can swap them at runtime or in tests. React components, Ruby modules and Rails service objects all lean on composition.

**Q27. What is the diamond problem, and how do C++, Java, Ruby and Python handle it?**
`D` inherits from `B` and `C`, which both inherit from `A`, so `D` gets two `A` subobjects and ambiguous member access. C++ fixes it with virtual inheritance (`class B : virtual public A`); Java and C# allow one base class plus many interfaces; Ruby has single inheritance plus mixins, and method lookup walks the `ancestors` chain linearly, so the most recently included module wins. Python allows several base classes and orders them into one line (the MRO, C3 linearization), so each class appears once and `super()` follows that order.

**Q28. What do public, protected and private mean, and how do inheritance modes change them?**
Public: accessible everywhere; protected: the class and its subclasses; private: the class itself (and its friends). Public inheritance keeps the base's levels, protected inheritance turns public members into protected ones, private inheritance makes both private. Only public inheritance models "is-a".

**Q29. `class` vs `struct` in C++?**
Only the defaults differ: `struct` members and base classes are public by default, `class` ones private. By convention, `struct` is for plain data with few invariants and `class` for types that protect invariants.

**Q30. What are static members?**
Members that belong to the class rather than to an object: one copy shared by all instances, usable without an object. A static member function has no `this`, so it cannot touch instance members directly. Ruby's equivalents are class methods (`def self.find_by_slug`) and class-level instance variables.

**Q31. What are the rule of three, five and zero?**
If a class needs a custom destructor, copy constructor or copy assignment (usually because it owns a raw resource), it needs all three; C++11 adds the move constructor and move assignment for five. Rule of zero: let members such as `std::string`, `std::vector` and `std::unique_ptr` own the resources, and write none of them.

**Q32. How do Ruby, Python and TypeScript get polymorphism without classic interfaces?**
Ruby and Python use duck typing: any object that responds to the method works, and modules (Ruby) or mixin classes (Python) share behavior; Python also offers `abc.ABC` and `typing.Protocol` when you want an explicit contract. TypeScript uses structural typing: a value satisfies an interface if it has the right shape, no `implements` needed, and interfaces disappear at runtime.

**Q33. What are coupling and cohesion?**
Cohesion is how focused a module is on one job (aim high); coupling is how much modules depend on each other's internals (aim low). A 2,000-line `User` model handling billing, notifications and search has low cohesion; a controller reaching into another model's internals is tightly coupled.

**Q34. What is a friend function, and when is it acceptable?**
A non-member function (or class) granted access to a class's private members by a `friend` declaration inside the class. The classic legitimate use is `operator<<` for printing. Frequent friends are a sign of broken encapsulation.

**Q35. What is a constructor, and can it be private or virtual?**
A constructor runs when an object is created and puts it into a valid starting state. It has no return type and cannot be virtual (the real type is still being built), but it can be private, which is how singletons and factory methods (`create(...)`) control who makes objects. Kinds: default, parameterized, copy and (C++) move; Ruby, Python and JS have just one constructor (`initialize`, `__init__`, `constructor`) and use default arguments instead of overloads.

**Q36. Static (early) vs dynamic (late) binding?**
Static binding connects a call to code at compile time using the declared type (overloads, templates, non-virtual functions). Dynamic binding does it at runtime using the object's real type (virtual functions through a pointer or reference). Redefining a non-virtual function in a subclass only hides the base one, so a call through a base pointer still runs the base version; `override` catches that mistake. In Ruby, Python and JS every method call is dynamic.

**Q37. Data hiding vs encapsulation?**
Encapsulation bundles data with the methods that use it in one class; data hiding means outsiders cannot reach the data directly (`private`). Encapsulation is the bundle, data hiding is the lock. A class with public fields and methods is bundled but not hidden, so it cannot protect its rules.

**Q38. How do you read a UML class diagram?**
Each box is a class (`+` public, `-` private, `#` protected). Solid line with a hollow triangle: inheritance. Dashed line with a hollow triangle: implements an interface. Plain solid line: association. Dashed arrow: dependency. Hollow diamond: aggregation. Filled diamond: composition (the part dies with the whole). Numbers at the line ends (`1`, `0..1`, `*`) are multiplicities.

**Q39. What is a mixin, and how does it differ from inheritance and an interface?**
A mixin is a small bundle of reusable behavior added to a class without making it a child in an is-a hierarchy: Ruby `include Comparable`, Django's `LoginRequiredMixin`, TypeScript mixin functions. Inheritance says what a thing is, an interface says what it must be able to do (no code), a mixin supplies ready-made code for one ability. Keep mixins small, because lookup order (Ruby `ancestors`, Python MRO) decides which method wins.

**Q40. What is an immutable value object, and why must you override the hash when you override equality?**
A value object (`Money`, `Address`) has no identity; it is defined by its values, compared by value and best made immutable, so it is safe to share and has no copy bugs ("changing" it builds a new one). Hash maps and sets use the hash to find a bucket, so two equal objects with different hashes would be treated as different; that is why `equals`/`hashCode` (Java), `eql?`/`hash` (Ruby) and `__eq__`/`__hash__` (Python) go together.

**Q41. How would you design the classes for a parking lot?**
Clarify requirements, then turn nouns into classes (`ParkingLot`, `Spot`, `Ticket`) and verbs into methods on the class that owns the data (`park`, `leave`, `canFit`). Put an interface where the rules will change (`FeePolicy`, so flat or weekend pricing is a new class), let the lot own its spots (composition), skip class hierarchies the problem does not need, then walk through a scenario and say what you would add next (floors, payments, locking). Full worked code is in [OOP section 17](02-oop.md).

**Q42. What kinds of polymorphism are there?**
Subtype (one base type, many subclasses, chosen at runtime: virtual functions, interfaces), ad hoc (same name, different code per argument type, chosen at compile time: overloading), and parametric (one piece of code for any type: C++ templates, Java, C# and TypeScript generics). Duck typing gives the subtype effect without a declared base type.

## OOP in C++: what's the output? (see [OOP](02-oop.md) and [C++ OOP](../03-languages/c-cpp/03-cpp-oop.md))

**Q43. What's the output?**
```cpp
struct Engine {
    Engine()  { cout << "Engine "; }
    ~Engine() { cout << "~Engine "; }
};
struct Vehicle {
    Vehicle()  { cout << "Vehicle "; }
    ~Vehicle() { cout << "~Vehicle "; }
};
struct Car : Vehicle {
    Engine engine;
    Car()  { cout << "Car "; }
    ~Car() { cout << "~Car "; }
};

int main() {
    Car c;
    cout << "| ";
}
```
**Output:** `Vehicle Engine Car | ~Car ~Engine ~Vehicle`. Construction runs base classes first, then members in declaration order, then the constructor body; destruction runs in exactly the reverse order.

**Q44. Spot the bug: what does this print?**
```cpp
struct Range {
    int size;
    int start;
    Range(int s) : start(s), size(start + 10) {}
};

int main() {
    Range r(5);
    cout << r.start << ' ' << r.size;
}
```
**Answer:** `5` followed by garbage (undefined behavior). Members are initialized in declaration order, not initializer-list order, so `size` is computed from `start` before `start` is set. Declare members in the order you initialize them; `-Wall` includes `-Wreorder`, which warns about this.

**Q45. What's the output?**
```cpp
struct Base {
    ~Base() { cout << "~Base "; }
};
struct Derived : Base {
    ~Derived() { cout << "~Derived "; }
};

int main() {
    Base* p = new Derived();
    delete p;
}
```
**Answer:** formally undefined behavior; in practice it prints only `~Base`, so `Derived`'s cleanup never runs. With `virtual ~Base()` it prints `~Derived ~Base`.

**Q46. What's the output?**
```cpp
struct Animal {
    virtual void speak() const { cout << "Animal "; }
    void move() const { cout << "walks "; }
    virtual ~Animal() = default;
};
struct Dog : Animal {
    void speak() const override { cout << "Woof "; }
    void move() const { cout << "runs "; }
};

int main() {
    Dog d;
    Animal* p = &d;
    p->speak();
    p->move();
    d.move();
}
```
**Output:** `Woof walks runs`. `speak` is virtual, so the object's dynamic type (`Dog`) decides. `move` is not virtual, so the pointer's static type (`Animal*`) decides; `Dog::move` only hides it.

**Q47. What's the output?**
```cpp
struct Animal {
    virtual string name() const { return "Animal"; }
    virtual ~Animal() = default;
};
struct Cat : Animal {
    string name() const override { return "Cat"; }
};

void byValue(Animal a) { cout << a.name() << ' '; }
void byRef(const Animal& a) { cout << a.name() << ' '; }

int main() {
    Cat c;
    byValue(c);
    byRef(c);
    vector<Animal> zoo{c};
    cout << zoo[0].name();
}
```
**Output:** `Animal Cat Animal`. Copying a `Cat` into an `Animal` slices off the derived part, and the copy is a genuine `Animal`. References and pointers keep polymorphism; store polymorphic objects as `vector<unique_ptr<Animal>>`.

**Q48. What's the output?**
```cpp
struct Base {
    Base() { hello(); }
    virtual void hello() const { cout << "Base::hello "; }
    virtual ~Base() = default;
};
struct Derived : Base {
    Derived() { hello(); }
    void hello() const override { cout << "Derived::hello "; }
};

int main() { Derived d; }
```
**Output:** `Base::hello Derived::hello`. While `Base`'s constructor runs, the `Derived` part does not exist yet, so virtual calls resolve to `Base`. Destructors behave the same way, so never rely on virtual calls during construction or destruction.

**Q49. What's the output?**
```cpp
void f(int)    { cout << "int "; }
void f(long)   { cout << "long "; }
void f(double) { cout << "double "; }

int main() {
    f(5);
    f(5L);
    f('a');
    f(5.0f);
}
```
**Output:** `int long int double`. Exact matches win, then promotions (`char` to `int`, `float` to `double`) beat conversions. Adding `f(5u)` would not compile: `unsigned` to `int`, `long` or `double` are all conversions of equal rank, so the call is ambiguous.

**Q50. What's the output?**
```cpp
struct Base {
    void print(int x) const { cout << "Base int " << x; }
};
struct Derived : Base {
    void print(double x) const { cout << "Derived double " << x; }
};

int main() {
    Derived d;
    d.print(42);
}
```
**Output:** `Derived double 42`. Name lookup stops at the first scope that declares `print`, so `Derived::print` hides every `Base::print` overload and `42` converts to `double`. Add `using Base::print;` inside `Derived` and the call prints `Base int 42`.

**Q51. What's the output?**
```cpp
struct Base {
    virtual void greet(string who = "base") const { cout << "Base " << who; }
    virtual ~Base() = default;
};
struct Derived : Base {
    void greet(string who = "derived") const override { cout << "Derived " << who; }
};

int main() {
    Derived d;
    const Base& b = d;
    b.greet();
}
```
**Output:** `Derived base`. The function body is chosen at runtime, but default arguments come from the static type of the expression (`Base`). Never redefine default arguments in an override.

**Q52. What's the output?**
```cpp
struct Tracer {
    Tracer() { cout << "ctor "; }
    Tracer(const Tracer&) { cout << "copy "; }
    Tracer& operator=(const Tracer&) { cout << "assign "; return *this; }
    ~Tracer() { cout << "dtor "; }
};

Tracer make() { return Tracer(); }
void take(Tracer) {}

int main() {
    Tracer a;
    Tracer b = a;
    b = a;
    take(a);
    Tracer c = make();
    cout << "| ";
}
```
**Output:** `ctor copy assign copy dtor ctor | dtor dtor dtor`. `Tracer b = a` is initialization (the copy constructor), not assignment; passing by value copies the argument and later destroys the parameter; `make()` returns a prvalue, and C++17 guarantees copy elision, so `c` is constructed directly. Locals are destroyed in reverse order (`c`, `b`, `a`).

**Q53. What's the output, and what is the bug?**
```cpp
struct Session {
    static int alive;
    Session()  { ++alive; }
    ~Session() { --alive; }
};
int Session::alive = 0;

int main() {
    Session a;
    {
        Session b, c;
        cout << Session::alive << ' ';
    }
    Session d = a;
    cout << Session::alive;
}
```
**Output:** `3 1`. `b` and `c` are destroyed at the end of the block. `Session d = a` uses the compiler-generated copy constructor, which does not increment, yet `d`'s destructor still decrements, so the count ends at `-1`. Rule of three: a class with a custom destructor usually needs a custom copy constructor too.

**Q54. What's the output?**
```cpp
struct Shape { virtual ~Shape() = default; };
struct Circle : Shape {};

void draw(const Shape&)  { cout << "shape "; }
void draw(const Circle&) { cout << "circle "; }

int main() {
    Circle c;
    const Shape& s = c;
    draw(c);
    draw(s);
}
```
**Output:** `circle shape`. Overload resolution uses static types at compile time; only virtual member functions dispatch on the runtime type. Choosing behavior by the runtime types of arguments needs a virtual method on `Shape` or double dispatch (the Visitor pattern).

**Q55. Spot the bug: why does this not compile?**
```cpp
struct Shape {
    virtual double area() const = 0;
    virtual ~Shape() = default;
};
struct Square : Shape {
    double side = 2;
    double area() { return side * side; }
};

int main() { Square s; }
```
**Answer:** `Square::area()` is not `const`, so it does not override `Shape::area() const`; it is a new function that hides it. `Square` stays abstract and cannot be instantiated. Write `double area() const override`: with `override`, the compiler reports the mismatch right at the declaration.

**Q56. What's the output?**
```cpp
struct Meters {
    Meters(double v) : value(v) {}
    double value;
};

void run(Meters m) { cout << m.value; }

int main() { run(5); }
```
**Output:** `5`. A single-argument constructor doubles as an implicit conversion: `5` becomes `5.0`, then a `Meters`. Mark it `explicit Meters(double v)` to forbid this; then only `run(Meters{5})` compiles.

## SOLID and design principles (see [SOLID and design principles](03-solid-and-design-principles.md))

**Q57. What does SOLID stand for?**
Single Responsibility (one reason to change), Open/Closed (extend by adding code, not by editing working code), Liskov Substitution (subtypes work wherever the base type is expected), Interface Segregation (small, client-specific interfaces), Dependency Inversion (high-level policy depends on abstractions, and details implement them).

**Q58. Give an SRP violation from a Rails app and the fix.**
A controller action that validates input, calculates prices, charges the card, sends the receipt and enqueues an inventory sync has five reasons to change. Move validation into the model, the workflow into a `CheckoutService`, the email into a mailer and the sync into a job; the controller only parses params and renders.

**Q59. Give an Open/Closed example.**
Adding a sales channel or payment provider should mean adding one class that implements the shared interface and registering it, not editing `case provider` branches in five files. Polymorphism and the Strategy pattern are the usual tools; add extension points only where change actually happens.

**Q60. Explain Liskov Substitution with an example.**
Code that works with a base type must work with any subtype without surprises. A `Square` subclass of a mutable `Rectangle` breaks callers that set the width and expect the height to stay the same; a subclass that raises `NotImplementedError` for an inherited method breaks callers too. Fix it by modeling a different abstraction instead of forcing inheritance.

**Q61. Explain Interface Segregation.**
Clients should not depend on methods they do not use. Instead of one `Exporter` interface with `to_csv`, `to_pdf` and `email`, define small role interfaces so a CSV-only exporter does not stub the rest. In TypeScript, give each component a props type with only what it needs.

**Q62. Dependency Inversion vs dependency injection vs inversion of control?**
DIP is the principle (depend on abstractions); DI is a technique (pass dependencies in, usually through the constructor, instead of creating them inside); IoC is the broader idea that a framework or container calls your code and wires its dependencies (Rails calls your controllers; NestJS and ASP.NET Core inject services).

**Q63. Is DRY always right?**
DRY is about knowledge, not similar-looking code. Two validations that look alike but change for different reasons should stay separate, because merging them couples unrelated rules. Many teams wait for the third repetition before extracting: the wrong abstraction costs more than some duplication.

**Q64. What are KISS and YAGNI?**
KISS: choose the simplest design that solves today's problem. YAGNI: do not build features or flexibility for imagined requirements. Both fight speculative abstractions, such as a plugin system for a single payment provider.

**Q65. What is the Law of Demeter?**
Talk only to direct collaborators: `order.customer_city` instead of `order.customer.address.city`. Long chains couple the caller to the whole object graph and break when it changes. Rails `delegate :city, to: :customer, prefix: true` is the common fix; do not wrap every chain mechanically.

**Q66. What does "tell, don't ask" mean?**
Tell an object to do the work instead of pulling its data out and deciding for it: `account.withdraw(amount)`, which checks the balance itself, rather than comparing `account.balance` in the caller and then assigning a new balance. It keeps invariants inside the object that owns the data.

**Q67. How do SOLID principles make testing easier?**
SRP gives small units with few behaviors to test; DIP and DI let you pass fakes for HTTP clients, mailers and clocks; ISP keeps those fakes small; LSP guarantees a fake can stand in for the real thing.

**Q68. When does applying SOLID become over-engineering?**
When you add interfaces with one implementation "just in case", layers that only forward calls, or factories for objects that never vary. Apply the principles where change is real or likely, and refactor toward them when the second use case arrives.

## Design patterns (see [Design patterns](04-design-patterns.md))

**Q69. What is a design pattern, and what are the three classic categories?**
A named, reusable solution to a recurring design problem; its main value is shared vocabulary. Creational patterns (how objects are created): Singleton, Factory Method, Abstract Factory, Builder. Structural (how objects are composed): Adapter, Decorator, Facade, Proxy, Composite. Behavioral (how objects collaborate): Strategy, Observer, Command, State, Template Method, Iterator.

**Q70. What are the pitfalls of Singleton?**
It is global mutable state in disguise: hidden dependencies, state leaking between tests, awkward mocking, thread-safety and initialization-order problems. It is also only one per process: Puma workers, Sidekiq processes and servers each get their own instance, so a singleton cannot coordinate anything globally (use Redis or the database for that). Prefer passing the dependency in, and keep singletons for stateless or read-only things such as configuration.

**Q71. How do you write a thread-safe singleton in modern C++?**
```cpp
class Config {
public:
    static Config& instance() {
        static Config config;  // initialized once, thread-safe since C++11
        return config;
    }
    Config(const Config&) = delete;
    Config& operator=(const Config&) = delete;
private:
    Config() = default;
};
```
The function-local static (a "Meyers singleton") replaces double-checked locking, and the deleted copy operations stop accidental copies.

**Q72. Factory Method vs Abstract Factory vs Builder?**
Factory Method: one method decides which concrete class to create (`Notifier.for(user)`). Abstract Factory: an object that creates families of related products (light-theme vs dark-theme widgets). Builder: constructs one complex object step by step, which beats a constructor with ten optional parameters (query builders, test data factories).

**Q73. Adapter vs Decorator vs Facade vs Proxy?**
Adapter converts one interface into the one clients expect (a third-party API client wrapped behind your `SalesChannel` interface). Decorator adds behavior behind the same interface (caching or logging around a client). Facade puts a simple new interface in front of a complex subsystem (`Checkout.call`). Proxy controls access behind the same interface (lazy loading, permission checks, remote objects).

**Q74. Strategy vs State?**
Both delegate to interchangeable objects. In Strategy, the client chooses the algorithm (a shipping cost calculator per carrier). In State, the object switches its own behavior as its state changes (an order that is `pending`, then `paid`, then `shipped`, each allowing different transitions).

**Q75. Strategy vs Template Method?**
Template Method fixes the algorithm's skeleton in a base class and lets subclasses override steps (inheritance). Strategy injects the varying step as an object or function (composition), so it can change at runtime and is easier to test. Modern code usually prefers Strategy.

**Q76. Observer vs Pub/Sub?**
In Observer, subscribers register directly with the subject, which calls them (DOM event listeners, model callbacks). Pub/Sub puts a broker in between, so publishers and subscribers never know each other and may live in different processes (Redis pub/sub, Action Cable). `ActiveSupport::Notifications` is an in-process pub/sub used for instrumentation.

**Q77. How is a background job an example of the Command pattern?**
The job packages a request (an action plus its arguments) as an object, so it can be queued, retried, scheduled, logged or run on another machine. Undo stacks and task queues are other examples.

**Q78. How does middleware relate to Chain of Responsibility?**
Each handler receives the request, does its part (authentication, logging, rate limiting), then either passes it to the next handler or stops the chain. Rack middleware in Rails and `app.use()` in Express and Hono implement this pattern.

**Q79. Repository vs Active Record?**
Active Record puts persistence on the domain object itself (`user.save`): fast to build, and the Rails default. Repository moves persistence into a separate object (`userRepository.save(user)`), keeping domain objects free of database code and easy to fake in tests; it is common in clean architecture, .NET and TypeScript backends.

**Q80. What is the Null Object pattern?**
Replace `nil` checks with an object that implements the same interface with harmless default behavior: a `GuestUser` whose `admin?` returns `false` and whose cart is empty, so callers never write `if current_user`. Ruby's `&.` handles nil at each call site; a null object removes the branch entirely.

**Q81. Name some common anti-patterns.**
God object (one class that knows everything, such as a 3,000-line model), spaghetti code, golden hammer (one tool for every problem), premature optimization, magic numbers and strings, and copy-paste programming. Naming them in a code review makes the discussion faster and less personal.

## Operating systems (see [Operating systems](05-operating-systems.md))

**Q82. Process vs thread?**
A process is a running program with its own address space, file descriptors and at least one thread; a crash stays contained. Threads live inside a process, share its memory and have their own stack and registers: cheaper to create and switch, but shared memory needs synchronization. Puma uses both: worker processes, each running several threads.

**Q83. Concurrency vs parallelism, and where does Ruby's GVL fit in?**
Concurrency is making progress on several tasks by interleaving them; parallelism is running them at the same instant on several cores. CRuby's GVL lets only one thread run Ruby code at a time per process, but it is released during blocking I/O, so threads still help I/O-bound web apps; CPU parallelism comes from multiple processes (Puma workers).

**Q84. What happens in a context switch, and why can too many threads hurt?**
The OS saves the running task's registers and program counter and restores another task's; switching between processes also switches address spaces. Each switch costs CPU time and cache warmth, and each thread needs memory for its stack, so thousands of busy threads spend more time switching than working.

**Q85. What are the process states?**
New, ready (waiting for a CPU), running, waiting or blocked (on I/O or a lock), and terminated. The scheduler moves processes between ready and running; I/O moves them to waiting and back to ready.

**Q86. Compare FCFS, SJF, Round Robin and priority scheduling.**
FCFS is simple, but short jobs wait behind long ones (the convoy effect). SJF minimizes average waiting time but needs burst estimates and can starve long jobs. Round Robin gives each task a time slice: responsive and fair, with overhead if slices are tiny. Priority scheduling can starve low priorities, which aging (raising priority over time) fixes.

**Q87. What is a race condition? Give a web example.**
A result that depends on the timing of unsynchronized access to shared data. `count += 1` from two threads is a read-modify-write that can lose updates. Web apps hit the same race across processes: two requests both read `stock = 1` and both sell the item. Fix it with a mutex or atomic in memory, and with atomic SQL updates, row locks or unique constraints in the database.

**Q88. Mutex vs semaphore?**
A mutex allows one holder at a time, and only the thread that locked it should unlock it. A counting semaphore admits up to N holders (for example, at most 10 concurrent API calls); a binary semaphore looks like a mutex but has no owner, so any thread can signal it.

**Q89. What is a deadlock, and how do you prevent it?**
Two or more tasks each wait forever for a resource another one holds. It needs all four Coffman conditions: mutual exclusion, hold and wait, no preemption and circular wait. Break one: acquire locks in a global order (the most practical fix), use timeouts or try-lock, or take all locks at once. Databases detect deadlocks and abort one transaction (Postgres reports `deadlock detected`), so lock rows in a consistent order, for example sorted by id, and retry.

**Q90. Starvation vs deadlock vs livelock?**
Deadlock: everyone is blocked waiting. Livelock: tasks keep running and reacting to each other but make no progress (two people stepping aside in a corridor forever). Starvation: one task never gets the resource because others keep winning, fixed with fairness or aging.

**Q91. What are virtual memory, page faults and thrashing?**
Each process sees a private virtual address space that the MMU maps to physical memory in pages. A page fault happens when an accessed page is not in RAM and must be loaded (from disk or swap). Thrashing is when the working set exceeds RAM and the system spends its time paging instead of working; the fix is using less memory or adding RAM.

**Q92. User mode vs kernel mode, and what is a system call?**
Application code runs in user mode with restricted access; the kernel runs in kernel mode with full hardware access. A system call (`read`, `write`, `open`, `fork`, socket calls) switches into the kernel to do privileged work for the program, which costs more than a normal function call.

**Q93. How do processes communicate?**
Pipes (`ps aux | grep puma`), sockets (TCP, or Unix domain sockets such as Nginx talking to Puma), shared memory, message queues, signals and files. Across machines the usual choices are HTTP, gRPC and message brokers.

**Q94. What do `fork` and copy-on-write mean for a Rails server?**
`fork` creates a child process with a copy of the parent's memory, but pages stay shared until one side writes to them (copy-on-write). Puma's `preload_app!` loads the app once in the master and then forks workers, so they share most memory pages. Connections opened before the fork must not be shared, so they are re-established in each worker (Rails does this for Active Record; check other clients such as Redis).

**Q95. Zombie vs orphan process?**
A zombie has exited, but its parent has not yet read its exit status with `wait`, so its process-table entry lingers. An orphan's parent died first; it is re-parented to PID 1, which reaps it when it exits. In containers your app may be PID 1, so use a tiny init such as `tini` (`docker run --init`) to reap zombies and forward signals.

**Q96. SIGTERM vs SIGKILL?**
SIGTERM asks a process to stop and can be caught: web servers finish in-flight requests, and Sidekiq stops fetching jobs, gives running ones a timeout to finish and pushes the rest back to Redis. SIGKILL cannot be caught; the process dies immediately. `docker stop` sends SIGTERM and, by default, SIGKILL 10 seconds later, so keep graceful shutdowns shorter than the grace period.

**Q97. What does "Too many open files" mean?**
The process hit its file descriptor limit; every open file, socket and pipe uses a descriptor. Causes: files or connections that are opened and never closed, or a limit too low for the traffic. Check with `ulimit -n` and `lsof -p <pid>`, fix leaks first, then raise the limit.

## Networking (see [Networking](06-networking.md))

**Q98. OSI vs TCP/IP: where do HTTP, TLS, TCP and IP sit?**
OSI has seven layers (physical, data link, network, transport, session, presentation, application); TCP/IP has four (link, internet, transport, application). IP is the network (internet) layer, TCP and UDP the transport layer, HTTP the application layer. TLS sits between transport and application; in OSI terms it is usually placed at the session or presentation layer.

**Q99. TCP vs UDP?**
TCP is connection-oriented, reliable and ordered, with retransmission, flow control and congestion control; it costs a handshake and suffers head-of-line blocking. UDP sends independent datagrams with no delivery, ordering or duplicate guarantees, but with low overhead and latency. TCP: HTTP/1.1 and HTTP/2, database connections, SSH. UDP: DNS queries, video calls, games, and QUIC (HTTP/3).

**Q100. Explain the TCP three-way handshake. Why three steps?**
The client sends SYN with its initial sequence number, the server replies SYN-ACK with its own, and the client sends ACK. Three messages are the minimum for both sides to announce their sequence numbers and confirm the other side received them. It costs one round trip before any data flows, plus the TLS handshake.

**Q101. What identifies a TCP connection?**
The 4-tuple: source IP, source port, destination IP and destination port. That is why one server port (443) can serve thousands of clients at once: each client connection has a different source IP and port.

**Q102. What are private IP ranges and NAT, and why can't a webhook reach your laptop?**
Private ranges (10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16) are not routable on the internet. NAT lets many private hosts share one public address by rewriting addresses and ports on outgoing connections, so nothing outside can open a connection to your laptop unless you use a tunnel or deploy to a public host.

**Q103. What does CIDR notation like `10.0.1.0/24` mean?**
The first 24 bits are the network prefix, leaving 8 bits for hosts: 256 addresses, 254 usable (the network and broadcast addresses are reserved). A `/16` gives 65,536 addresses; a `/32` is a single host.

**Q104. How does DNS resolution work?**
The browser and OS caches are checked first; then a recursive resolver (your ISP's or a public one) asks a root server, which points to the TLD servers (`.com`), which point to the domain's authoritative servers, which return the record. Every answer is cached for its TTL, which is why DNS changes "propagate" slowly; lower the TTL ahead of a migration.

**Q105. A vs AAAA vs CNAME vs MX vs TXT records?**
A maps a name to an IPv4 address, AAAA to an IPv6 address; CNAME aliases one name to another (`www` to a hosting provider's name); MX lists mail servers; TXT holds text for domain verification, SPF and DKIM; NS names the authoritative servers. A CNAME cannot coexist with other records for the same name, so it is not allowed at the zone apex (providers offer ALIAS records or CNAME flattening).

**Q106. What happens when you type a URL and press Enter?**
The browser parses the URL and checks its caches (and HSTS); DNS resolves the host to an IP; it opens a TCP connection and runs the TLS handshake (or a QUIC handshake for HTTP/3); it sends the HTTP request, which passes a CDN or load balancer and a reverse proxy such as Nginx before reaching the app server (Puma, Node), which runs the route, queries the database and responds. The browser then parses the HTML, fetches CSS, JavaScript and images, builds the DOM and CSSOM, lays out, paints and runs scripts.

**Q107. HTTP/1.1 vs HTTP/2 vs HTTP/3?**
HTTP/1.1: a text protocol with one outstanding request per connection, so browsers open about six connections per host. HTTP/2: binary framing, many multiplexed streams over one TCP connection, header compression; but one lost TCP packet stalls every stream. HTTP/3: HTTP over QUIC on UDP, with per-stream loss recovery, faster connection setup and migration across networks.

**Q108. How does HTTPS work?**
During the TLS handshake the server presents a certificate signed by a trusted certificate authority, and the browser checks the chain, the hostname and the validity dates. Both sides use asymmetric cryptography (ECDHE key exchange) to agree on symmetric session keys, then encrypt traffic with fast symmetric ciphers (AES-GCM, ChaCha20). TLS 1.3 needs one round trip; resumed sessions can send 0-RTT data, with replay caveats.

**Q109. What are keep-alive and connection pooling, and why do they matter?**
Keep-alive reuses a TCP (and TLS) connection for many requests, avoiding repeated handshakes. Connection pools apply the same idea to databases and HTTP clients: a fixed set of open connections shared by threads. In Rails, the database pool must be at least the number of Puma threads, or requests queue waiting for a connection.

**Q110. L4 vs L7 load balancer?**
An L4 load balancer routes by IP and port without reading the payload: fast and protocol-agnostic. An L7 load balancer understands HTTP, so it can route by host, path or header, terminate TLS, add `X-Forwarded-For` and health-check specific endpoints. Nginx and cloud application load balancers work at L7.

**Q111. Forward proxy vs reverse proxy?**
A forward proxy acts for clients (a corporate proxy that filters outbound traffic); a reverse proxy acts for servers, and clients think they are talking to the origin. Nginx in front of Puma terminates TLS, serves static files, buffers slow clients, compresses responses and balances load across app servers.

**Q112. What is a CDN, and how do you control what it caches?**
A network of edge servers that cache content close to users, cutting latency and load on your origin. You control it with `Cache-Control` (`max-age`, `s-maxage`, `private`, `no-store`), fingerprinted asset names for long-lived caching, and purge APIs for changes. Never let a CDN cache personalized responses: mark them `private`.

**Q113. What do ping and traceroute use?**
`ping` sends ICMP echo requests and measures the round-trip time. `traceroute` sends probes with increasing TTL values; each router that drops a probe returns an ICMP "time exceeded" message, which reveals the path (Unix `traceroute` uses UDP probes by default, Windows `tracert` uses ICMP echo). Many hosts block ICMP, so no reply does not mean the host is down.

**Q114. A production server cannot reach a third-party API. How do you debug it?**
Work up the layers: `dig api.example.com` (DNS), `curl -v https://api.example.com` (TCP connect, TLS, HTTP status), then outbound firewall or security-group rules, proxies, IP allowlists on the provider's side, expired certificates or an outdated CA bundle, and the provider's status page. Running the same call from your laptop separates "our network" from "their API".

**Q115. Latency vs bandwidth vs throughput?**
Latency is the time for a round trip (milliseconds); bandwidth is the maximum data rate of a link; throughput is the rate you actually achieve. Many small sequential requests are latency-bound (fix with fewer round trips, batching, HTTP/2 and CDNs); large downloads are bandwidth-bound.
