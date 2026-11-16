# C#: Rapid-Fire Interview Questions

> How to use: cover the answer, say yours out loud, then check. Deeper explanations live in the topic files linked under each section.

## Platform and types (see [C# fundamentals](01-fundamentals.md))

**Q1. What is the difference between .NET Framework, .NET Core and .NET?**
.NET Framework (up to 4.8.x) is the original Windows-only runtime, now in maintenance mode. .NET Core was the cross-platform, open-source rewrite; from version 5 it is simply ".NET". New work targets modern .NET, whose LTS releases (such as .NET 8 and .NET 10) get three years of support.

**Q2. What is the CLR?**
The Common Language Runtime: it loads assemblies, JIT-compiles IL to native code, manages memory with the garbage collector, enforces type safety and handles exceptions and threads.

**Q3. What is an assembly?**
The compiled unit of deployment (`.dll` or `.exe`) containing IL, metadata and a manifest (name, version, dependencies). A project usually produces one assembly, and `internal` means "visible inside this assembly".

**Q4. What is JIT compilation, and what are the alternatives?**
The JIT compiles IL to native code at run time, method by method on first call; tiered compilation later recompiles hot methods with more optimization. ReadyToRun precompiles code to speed up startup, and Native AOT compiles the whole app ahead of time (fast startup and small memory, but limits on reflection and dynamic code).

**Q5. `decimal` vs `double`?**
`double` is binary floating point: fast, but values like 0.1 are not exact. `decimal` is a 128-bit base-10 type with 28-29 significant digits: slower, but exact for money. Use `decimal` for currency (`25.99m`).

**Q6. What is `var`, and how is it different from `dynamic`?**
`var` lets the compiler infer the static type from the initializer; the variable is still strongly typed. `dynamic` defers member binding to run time, so mistakes become runtime exceptions. Use `dynamic` only for interop scenarios.

**Q7. What does `default` produce?**
The default value of a type: zero, `false` or `'\0'` for value types (every field zeroed for structs) and `null` for reference types. `default(T)` works inside generics.

**Q8. What do `HasValue`, `Value` and `GetValueOrDefault` do on `int?`?**
`HasValue` reports whether a value is present, `Value` returns it or throws `InvalidOperationException` when empty, and `GetValueOrDefault()` returns the value or `default`.

**Q9. What do `?.`, `??` and `??=` do?**
`a?.B` evaluates to null instead of throwing when `a` is null; `a ?? b` returns `a` unless it is null, otherwise `b`; `x ??= y` assigns `y` only when `x` is null.

**Q10. When does boxing happen implicitly?**
Whenever a value type is converted to `object` or to an interface: assigning to an `object` variable, adding to a non-generic collection, passing to a `params object[]` method, or using a struct through an interface reference. Each box is a heap allocation.

**Q11. Is `string` a value type or a reference type?**
A reference type that behaves like a value: it is immutable, and `==` is overloaded to compare contents. Literals are interned, so identical literals share one instance.

**Q12. How should you compare strings case-insensitively?**
`string.Equals(a, b, StringComparison.OrdinalIgnoreCase)`. Stating the comparison avoids culture surprises (the Turkish "I" problem), and ordinal comparison is the fast, correct choice for identifiers, keys and file names.

## OOP in C# (see [C# fundamentals](01-fundamentals.md))

**Q13. How does C# express the four OOP pillars?**
Encapsulation through access modifiers and properties, abstraction through interfaces and abstract classes, inheritance through single class inheritance, and polymorphism through `virtual`/`override`, interfaces, overloading and generics. See [OOP concepts](../../01-cs-fundamentals/02-oop.md).

**Q14. Can a class inherit from several classes?**
No: one base class plus any number of interfaces, which avoids the diamond problem with state that C++ solves using virtual inheritance.

**Q15. What happens when two interfaces declare the same method?**
One public method can satisfy both. If they need different behavior, implement them explicitly (`void IA.Run()` and `void IB.Run()`), and each version is reachable only through its interface type.

**Q16. What does `sealed` do on a class and on a method?**
On a class it forbids inheritance (`string` is sealed). On an `override` method it stops further overriding in subclasses. Sealing also lets the JIT devirtualize some calls.

**Q17. What is a static class?**
A class marked `static`: it cannot be instantiated or inherited and contains only static members. Typical for helpers and for extension methods.

**Q18. What is a static constructor?**
A parameterless `static` constructor that the runtime calls once, automatically, before the type is first used. It initializes static state and cannot be called directly.

**Q19. In what order do initializers and constructors run in an inheritance chain?**
Derived field initializers, then base field initializers, then the base constructor body, then the derived constructor body. Unlike Java, C# runs a class's field initializers before calling the base constructor.

**Q20. What are `this` and `base` used for?**
`this` refers to the current instance, and `this(...)` chains to another constructor of the same class. `base` reaches the parent's members (`base.Describe()`), and `base(...)` calls the parent constructor.

**Q21. Can you override a non-virtual method?**
No. You can only hide it with `new`, and then the compile-time type decides which version runs.

**Q22. Overloading vs overriding?**
Overloading: same name, different parameter lists, resolved at compile time. Overriding: a derived class replaces a `virtual` or `abstract` member using `override`, resolved at run time from the object's actual type.

**Q23. What is a partial class?**
A class split across several files with `partial`, merged at compile time. It is mostly used to keep generated code (designers, source generators) separate from handwritten code.

**Q24. Why use properties instead of public fields?**
They keep the public API stable when you later add validation, lazy loading or change notification; they can appear in interfaces; and data binding, serializers and EF Core work with properties.

## Collections, strings and exceptions (see [C# fundamentals](01-fundamentals.md))

**Q25. `List<T>` vs an array?**
Arrays have a fixed size and slightly less overhead. `List<T>` wraps an array that grows by doubling and adds `Add`, `Insert` and `Remove`. Use arrays for fixed-size data, lists otherwise.

**Q26. How does `Dictionary<TKey, TValue>` work?**
It is a hash table: the key's `GetHashCode` picks a bucket and `Equals` resolves collisions, giving O(1) average operations. A key's hash must not change while it is stored, and custom key types need consistent `Equals` and `GetHashCode`.

**Q27. Why prefer `TryGetValue` over `ContainsKey` plus the indexer?**
`TryGetValue` does one lookup and never throws; `ContainsKey` followed by `dict[key]` hashes the key twice.

**Q28. `IEnumerable<T>` vs `ICollection<T>` vs `IList<T>`?**
`IEnumerable<T>` only iterates; `ICollection<T>` adds `Count`, `Add`, `Remove` and `Contains`; `IList<T>` adds indexing and `Insert`. Accept the least specific interface you need, and return read-only interfaces where possible.

**Q29. What read-only and immutable collection options exist?**
`IReadOnlyList<T>` and `IReadOnlyDictionary<TKey, TValue>` are read-only views (the underlying collection can still change), `ReadOnlyCollection<T>` is a wrapper, the `System.Collections.Immutable` types never change, and .NET 8 added frozen collections optimized for read-heavy lookups.

**Q30. What is `HashSet<T>` for?**
Unique items with O(1) average `Add` and `Contains`, plus set operations: `UnionWith`, `IntersectWith`, `ExceptWith`.

**Q31. Which collections are safe for concurrent use?**
The `System.Collections.Concurrent` types: `ConcurrentDictionary`, `ConcurrentQueue`, `ConcurrentBag` and `BlockingCollection`. Ordinary collections are not safe for concurrent writes.

**Q32. What is string interning?**
The runtime keeps a single shared instance of each literal string, so equal literals reference the same object; `string.Intern` adds strings at run time. It saves memory but is never a reason to compare strings by reference.

**Q33. When does `finally` run?**
Always: after normal completion, after an exception (handled or not, as it propagates), and even after a `return` inside `try`. It is for cleanup; `using` is the idiomatic shortcut for disposable resources.

**Q34. What are exception filters?**
`catch (HttpRequestException ex) when (ex.StatusCode == HttpStatusCode.NotFound)` runs the handler only if the condition is true. When it is false, the stack is not unwound, which preserves debugging information.

**Q35. How should you design custom exceptions?**
Derive from `Exception` (or a more specific base), end the name with `Exception`, offer the standard constructors (message, and message plus inner exception), and use them for exceptional situations, not ordinary control flow.

## LINQ, delegates and events (see [C# advanced](02-advanced.md))

**Q36. Query syntax vs method syntax?**
The compiler translates query syntax (`from x in xs where ... select ...`) into method calls. Method syntax covers more operators (`Take`, `Distinct`, `Any`) and is the usual team style; query syntax can read better for joins and `let` clauses.

**Q37. `Select` vs `SelectMany`?**
`Select` maps each element to one result. `SelectMany` maps each element to a sequence and flattens the results, like Ruby's `flat_map` or JavaScript's `flatMap`.

**Q38. `First` vs `FirstOrDefault` vs `Single`?**
`First` returns the first match and throws if there is none; `FirstOrDefault` returns `default` instead; `Single` demands exactly one match and throws on zero or several (`SingleOrDefault` throws only on several).

**Q39. Which LINQ operators execute immediately?**
Materializers (`ToList`, `ToArray`, `ToDictionary`) and operators that return a single value (`Count`, `Sum`, `Max`, `First`, `Any`, `Single`). Most others are deferred.

**Q40. How do you join two collections with LINQ?**
`customers.Join(orders, c => c.Id, o => o.CustomerId, (c, o) => new { c.Name, o.Total })`, or `join o in orders on c.Id equals o.CustomerId` in query syntax. `GroupJoin` produces the shape of a left join.

**Q41. What is an expression tree?**
A data structure describing code, such as `Expression<Func<Order, bool>>`, instead of compiled IL. Providers like EF Core walk the tree and translate it into SQL.

**Q42. What is a delegate?**
A type-safe reference to one or more methods with a particular signature. Callbacks, events and lambdas are all built on delegates.

**Q43. What is a multicast delegate?**
A delegate whose invocation list holds several methods (combined with `+=`). Invoking it calls them in order; if it returns a value, you get the last one.

**Q44. What is the classic closure bug with `for` loops?**
Lambdas capture variables, not values. Every lambda created in a `for` loop shares the single loop variable, so they all see its final value after the loop. Copy it into a local inside the loop. `foreach` creates a fresh variable per iteration (since C# 5).

**Q45. What is the standard .NET event pattern?**
`public event EventHandler<TArgs>? Changed;` with the `(object? sender, TArgs e)` signature, raised with `Changed?.Invoke(this, args)` so that having no subscribers does not throw.

**Q46. How do events cause memory leaks?**
The publisher's invocation list holds strong references to subscribers. If a long-lived publisher (a static or singleton service) outlives subscribers that never unsubscribe, those subscribers can never be collected.

**Q47. `Func<T, bool>` vs `Predicate<T>`?**
Same shape, different delegate types: LINQ uses `Func<T, bool>`, while older APIs such as `List<T>.Find` and `RemoveAll` take `Predicate<T>`. Delegates of different types do not convert into each other implicitly.

## Async, memory and resources (see [C# advanced](02-advanced.md))

**Q48. What is a `Task`?**
An object representing an asynchronous operation that may produce a value (`Task<T>`). You can await it, compose it (`WhenAll`, `WhenAny`) and inspect its status and exceptions; it plays the role of a JavaScript `Promise`.

**Q49. Task vs Thread?**
A thread is an OS execution path with its own stack. A task is a unit of work, usually run on thread-pool threads, or pure I/O that occupies no thread while waiting. Prefer tasks; create dedicated threads only for special long-running work.

**Q50. Does `async` move code to a background thread?**
No. An async method runs on the calling thread until its first `await` on an incomplete task, and waiting for I/O needs no thread at all. Use `Task.Run` explicitly for CPU-bound work.

**Q51. `Task.WhenAll` vs `Task.WhenAny`?**
`WhenAll` completes when every task finishes and returns all results; `WhenAny` completes when the first finishes, which helps with timeouts and racing redundant requests.

**Q52. How do exceptions work in async code?**
Awaiting a faulted task rethrows its exception, so ordinary `try/catch` around the `await` works. With `WhenAll`, `await` rethrows the first exception, while the returned task's `Exception` property holds all of them.

**Q53. What is a `CancellationToken`?**
A cooperative cancellation signal passed down through async calls. Code checks `IsCancellationRequested` or passes the token to APIs that throw `OperationCanceledException`. ASP.NET Core provides one per request (`HttpContext.RequestAborted`).

**Q54. What is `ValueTask<T>`?**
A struct alternative to `Task<T>` that avoids an allocation when results are often available synchronously (cache hits). Await it only once and never block on it.

**Q55. `lock` vs `SemaphoreSlim`?**
`lock` gives one thread exclusive access and cannot contain an `await` (a compile error). `SemaphoreSlim` allows N concurrent holders and offers `WaitAsync`, so it is the tool for limiting concurrency in async code.

**Q56. What does a `using` statement compile to?**
A `try/finally` that calls `Dispose()` in the `finally` block (after a null check), so the resource is released even when an exception is thrown.

**Q57. Why does the GC use generations?**
Most objects die young, so frequently collecting just gen 0 reclaims a lot of memory cheaply, while long-lived gen 2 objects are scanned rarely. Large objects (85,000 bytes or more) live on the large object heap and are collected with gen 2.

**Q58. Should you call `GC.Collect()`?**
Almost never. The GC tunes itself, and forced collections promote live objects into older generations and hurt performance. Fix the leak or reduce allocations instead.

**Q59. Workstation GC vs server GC?**
Workstation GC targets client apps and low latency. Server GC uses a heap and a collector thread per core for throughput, and it is the default for ASP.NET Core apps.

## Generics, patterns and modern C# (see [C# advanced](02-advanced.md))

**Q60. Why use generics instead of `object`?**
Compile-time type safety, no casts, and no boxing for value types: `List<int>` stores raw ints, while `ArrayList` boxes each one.

**Q61. What does `where T : new()` allow?**
Calling `new T()` inside the generic code; any type argument must have a public parameterless constructor.

**Q62. What are covariance and contravariance?**
Covariance (`out T`) lets a producer of a derived type stand in for a producer of its base type: an `IEnumerable<string>` is usable as an `IEnumerable<object>`. Contravariance (`in T`) lets a consumer of a base type stand in for a consumer of a derived type: an `Action<object>` is usable as an `Action<string>`.

**Q63. What is a switch expression?**
An expression form, `value switch { pattern => result, _ => fallback }`, that returns a value, supports patterns and `when` guards, and gets a compiler warning when it may not cover every input.

**Q64. What are property and relational patterns?**
`order is { Status: "Paid", Total: > 100m }` checks property values and compares with `<`, `>`, `<=` and `>=`; combine patterns with `and`, `or` and `not`. `x is not null` is the idiomatic null check.

**Q65. What are `init` setters and `required` members?**
An `init` accessor (C# 9) allows setting a property only during construction or in an object initializer; `required` (C# 11) forces callers to set it there. Together they give immutable objects without long constructors.

**Q66. What are primary constructors in C# 12?**
Parameters declared on the class itself, as in `class OrderService(IOrderRepository repo)`, that are in scope throughout the class body; they are popular for DI. Unlike positional records, classes do not get public properties generated from them.

**Q67. Record class vs record struct vs class?**
A `record` (class) is a reference type with value-based equality; a `record struct` is a value type with value-based equality; an ordinary class compares by reference unless you override equality. Records are designed for immutable data.

## What's the output?

**Q68. What does this print?**
```csharp
var s1 = new PointS { X = 1 }; var s2 = s1; s2.X = 99;
var c1 = new PointC { X = 1 }; var c2 = c1; c2.X = 99;
Console.WriteLine($"{s1.X} {c1.X}");

struct PointS { public int X; }
class PointC { public int X; }
```
`1 99`: the struct was copied, while both class variables reference one object.

**Q69. What does this print?**
```csharp
Animal a = new Dog();
Console.WriteLine($"{a.Speak()} {a.Kind()}");

class Animal { public virtual string Speak() => "..."; public string Kind() => "animal"; }
class Dog : Animal { public override string Speak() => "Woof"; public new string Kind() => "dog"; }
```
`Woof animal`: `override` dispatches on the runtime type; `new` only hides, so the compile-time type `Animal` wins.

**Q70. What does this print?**
```csharp
var actions = new List<Action>();
for (int i = 0; i < 3; i++) actions.Add(() => Console.Write(i));
foreach (var act in actions) act();
```
`333`: all three lambdas share the one loop variable, which is 3 when they run. With `foreach` over `{ 0, 1, 2 }` it would print `012`.

**Q71. What does this print?**
```csharp
string s = "abc";
s.ToUpper();
Console.WriteLine(s);
```
`abc`: strings are immutable, and the uppercase result was discarded. Write `s = s.ToUpper();`.

**Q72. What does this print?**
```csharp
object a = 5, b = 5;
Console.WriteLine($"{a == b} {a.Equals(b)}");
```
`False True`: `==` on `object` compares references of two separate boxes, while `Equals` compares the boxed values.

**Q73. What does this print?**
```csharp
static int F()
{
    try { return 1; }
    finally { Console.Write("finally "); }
}
Console.WriteLine(F());
```
`finally 1`: the return value is computed, `finally` runs, then the method returns.

**Q74. What does this print?**
```csharp
int x = int.MaxValue;
x++;
Console.WriteLine(x);
```
`-2147483648`: integer arithmetic is unchecked by default and wraps around. Inside a `checked` block (or with the `CheckForOverflowUnderflow` project setting) it throws `OverflowException` instead.

**Q75. What does this print?**
```csharp
Console.WriteLine(new P(1) == new P(1));
Console.WriteLine(new C(1) == new C(1));

record P(int X);
class C { public int X; public C(int x) => X = x; }
```
`True`, then `False`: records compare by value, classes by reference.

**Q76. What does this print?**
```csharp
var nums = new List<int> { 1, 2, 3 };
var evens = nums.Where(n => n % 2 == 0);
nums.Add(4);
Console.WriteLine(evens.Count());
```
`2`: the query is deferred, so it runs at `Count()` and sees the 4 added afterwards.

## Spot the bug

**Q77. What is wrong here?**
```csharp
public async void SaveAsync() => await _db.SaveChangesAsync();
```
`async void` means callers cannot await completion or catch failures, and an exception can crash the process. Return `Task`: `public async Task SaveAsync()`.

**Q78. What is wrong here?**
```csharp
public IActionResult Index()
{
    var orders = _service.GetOrdersAsync().Result;
    return View(orders);
}
```
It blocks a thread on async work: a deadlock risk wherever a synchronization context exists and thread-pool starvation in ASP.NET Core. Make the action `async Task<IActionResult>` and `await` the call.

**Q79. What is wrong here?**
```csharp
foreach (var item in cart.Items)
    if (item.Quantity == 0) cart.Items.Remove(item);
```
Modifying a `List<T>` while enumerating it throws `InvalidOperationException`. Use `cart.Items.RemoveAll(i => i.Quantity == 0);` or iterate over a copy.

**Q80. What is wrong here?**
```csharp
var client = new HttpClient();    // inside a method called on every request
var json = await client.GetStringAsync(url);
```
Creating an `HttpClient` per call can exhaust sockets under load, because closed connections linger. Use `IHttpClientFactory` (typed or named clients) or a long-lived shared instance.
