# C# Advanced: LINQ, Delegates, Async and More

> **Tier:** T3 (essentials) · **Related:** [C# fundamentals](01-fundamentals.md), [ASP.NET Core MVC](../../11-dotnet/01-aspnet-core-mvc.md), [EF Core and Dapper](../../11-dotnet/02-entity-framework-and-dapper.md), [JavaScript async and the event loop](../javascript/04-async-event-loop-promises.md)
> **On your resume:** C#, ASP.NET Core MVC and Entity Framework from university work and the .NET side of your final-year project. Most ideas here mirror tools you use daily: ActiveRecord relations, JavaScript promises, TypeScript generics.

## Quick revision
- LINQ query syntax (`from ... where ... select`) compiles to method calls (`Where`, `Select`, `OrderBy`, `GroupBy`); method syntax is more common.
- **Deferred execution**: most operators only build a query, which runs when enumerated (`foreach`, `ToList`, `Count`, `First`) and runs again on every enumeration.
- `IEnumerable<T>` runs in memory with delegates; `IQueryable<T>` builds expression trees that a provider (EF Core) translates to SQL and runs in the database.
- Delegates are type-safe function references: `Func<..., TResult>` returns a value, `Action<...>` returns `void`, `Predicate<T>` returns `bool`. An `event` exposes a delegate for `+=`/`-=` only; just the owner can raise it.
- `await` frees the thread while the awaited task is incomplete; the method resumes later. Async methods return `Task`, `Task<T>` or `ValueTask<T>`.
- `async void` is for event handlers only: callers cannot await it, and its exceptions crash the process.
- Blocking with `.Result` or `.Wait()` can deadlock where a `SynchronizationContext` exists (WinForms, WPF, classic ASP.NET) and starves the thread pool in ASP.NET Core. Go async all the way.
- `ConfigureAwait(false)` skips resuming on the captured context: useful in libraries, unnecessary in ASP.NET Core app code. `IDisposable` + `using` give deterministic cleanup; `using var x = ...;` (C# 8) disposes at the end of the scope.
- The GC is generational (gen 0, 1, 2, plus a large object heap for objects of 85,000 bytes or more). Finalizers run late and unpredictably: implement `IDisposable` instead.
- Generic constraints: `where T : class`, `struct`, `new()`, `notnull`, a base class, an interface. .NET generics are reified (kept at run time), unlike Java's type erasure.
- Extension methods are static methods whose first parameter has `this` (LINQ is built from them). Pattern matching: type, property, relational (`>= 100`), logical (`and`, `or`, `not`) and list patterns.

## 1. LINQ

```csharp
var orders = new List<Order>
{
    new(1, "Sara", 120m, "Lahore"),
    new(2, "Ali", 40m, "Karachi"),
    new(3, "Sara", 75m, "Lahore"),
};
var big = orders.Where(o => o.Total > 50)               // method syntax
                .OrderByDescending(o => o.Total)
                .Select(o => new { o.Id, o.Customer });
var same = from o in orders                             // query syntax: compiles to the same calls
           where o.Total > 50
           orderby o.Total descending
           select new { o.Id, o.Customer };
var totals = orders.GroupBy(o => o.Customer)            // Sara 195, Ali 40
                   .Select(g => new { Customer = g.Key, Total = g.Sum(o => o.Total) });

record Order(int Id, string Customer, decimal Total, string City);
```

| Category | Operators |
|---|---|
| Filter, project, sort | `Where`, `OfType<T>`, `Select`, `SelectMany` (flatten), `OrderBy`, `ThenBy` |
| Group, join, aggregate | `GroupBy`, `Join`, `ToDictionary`, `Count`, `Sum`, `Min`, `Max`, `Average` |
| Element, quantifiers | `First`, `FirstOrDefault`, `Single`, `SingleOrDefault`, `Any`, `All`, `Contains` |
| Sets, paging, materialize | `Distinct`, `DistinctBy` (.NET 6), `Union`, `Except`, `Skip`, `Take`, `ToList`, `ToArray` |

`First` throws on an empty sequence, `Single` throws on zero or several matches, and the `OrDefault` versions return `default` instead of throwing when nothing matches.

```csharp
var nums = new List<int> { 1, 2, 3 };
var evens = nums.Where(n => n % 2 == 0);   // deferred: nothing runs yet
nums.Add(4);
Console.WriteLine(evens.Count());          // 2: runs now and sees the 4 (ToList() would snapshot it)
```

| | `IEnumerable<T>` | `IQueryable<T>` |
|---|---|---|
| Lambdas become | delegates (`Func<T, bool>`) | expression trees (`Expression<Func<T, bool>>`) |
| Runs | in memory (LINQ to Objects) | translated by a provider (EF Core to SQL), run in the database |
| Good for | collections already in memory | composing database queries step by step |
| Watch out for | an early `ToList()` pulls whole tables into memory | only translatable expressions work; arbitrary C# methods fail at run time |

```csharp
// IQueryable: the filter and paging become SQL (WHERE, ORDER BY, OFFSET/LIMIT)
var page = await db.Orders.Where(o => o.Total > 50).OrderBy(o => o.Id).Skip(20).Take(10).ToListAsync();
// ToList() first: every order is loaded, then filtered in memory
var slow = db.Orders.ToList().Where(o => o.Total > 50).ToList();
```

In your web stack: an ActiveRecord relation behaves like `IQueryable` (lazy, builds SQL, runs on `to_a` or `each`), while Ruby's `Enumerable#select` on a loaded array is like `IEnumerable`.

## 2. Delegates, lambdas and events

```csharp
Func<int, int, int> add = (a, b) => a + b;          // the last type argument is the return type
Action<string> log = msg => Console.WriteLine(msg);  // returns void
Predicate<int> isEven = n => n % 2 == 0;             // returns bool
DiscountRule tenPercent = total => total * 0.9m;     // custom delegate type
log += msg => File.AppendAllText("app.log", msg);    // multicast: both handlers run

var service = new OrderService();
service.OrderPlaced += (sender, id) => Console.WriteLine($"Email for order {id}");
service.Place(42);                                   // Email for order 42

public delegate decimal DiscountRule(decimal total);

public class OrderService
{
    public event EventHandler<int>? OrderPlaced;     // outsiders can only += or -=
    public void Place(int orderId)
    {
        OrderPlaced?.Invoke(this, orderId);          // after saving: only the owner can raise it
    }
}
```

Lambdas capture variables, not values, exactly like JavaScript closures. Compared with a public delegate field, an `event` cannot be invoked or overwritten (`=`) from outside. A long-lived publisher keeps every subscriber alive, so unsubscribe with `-=` when the subscriber goes away.

## 3. async/await and Task

```csharp
public async Task<string> GetProfileAsync(int id, CancellationToken ct = default)
{
    string json = await _http.GetStringAsync($"users/{id}", ct);   // thread is free while waiting
    return json;                                                  // wrapped in a Task<string>
}

Task<string> a = GetProfileAsync(1);              // start both...
Task<string> b = GetProfileAsync(2);
string[] both = await Task.WhenAll(a, b);         // ...then wait for both together
```

The method runs synchronously until the first `await` on an incomplete task, then returns an incomplete `Task` to its caller; a compiler-generated state machine resumes the rest when the work finishes, so no thread sits blocked on I/O. Use real async APIs for I/O and `Task.Run` only to move CPU-bound work off a UI thread. Unlike JavaScript's single event-loop thread, continuations run on thread-pool threads, so real parallelism (and data races) are possible.

| Pitfall | Why it hurts | Fix |
|---|---|---|
| `async void` | callers cannot await or catch it; an exception crashes the process | return `Task` (keep `async void` for event handlers) |
| `.Result` / `.Wait()` | under a UI or classic ASP.NET context, the continuation needs the very thread you are blocking: deadlock; in ASP.NET Core, thread-pool starvation | `await` all the way up the call chain |
| Missing `await` | fire-and-forget: exceptions vanish (warning CS4014) | await it, or hand it to a background queue |
| Awaiting independent calls one by one | latencies add up | `Task.WhenAll` |
| Ignoring `CancellationToken` | abandoned requests keep working | accept tokens and pass them down |

`ConfigureAwait(false)` tells `await` not to resume on the captured context. Libraries use it so they behave under any caller; ASP.NET Core has no `SynchronizationContext`, so application code there can skip it.

## 4. IDisposable, using, garbage collection and finalizers

```csharp
using (var conn = new SqlConnection(connectionString))   // Microsoft.Data.SqlClient
{
    conn.Open();
}                                                       // conn.Dispose() runs here, even after an exception

using var reader = new StreamReader("data.txt");        // using declaration: disposed at end of scope
await using var stream = File.OpenRead("big.bin");      // IAsyncDisposable
```

- **GC basics:** objects unreachable from roots (locals, statics, registers) are garbage. New objects start in gen 0, which is collected often and cheaply; survivors are promoted to gen 1 and then gen 2 (long-lived, collected rarely and expensively). Objects of 85,000 bytes or more go to the large object heap, collected with gen 2. The GC compacts memory to fight fragmentation.
- **Dispose vs finalizer:** `Dispose` releases resources deterministically, while memory is still reclaimed later by the GC. A finalizer (`~MyType()`) runs on a finalizer thread at an unknown time and makes the object survive an extra collection; you only need one when directly owning an unmanaged handle (prefer `SafeHandle`). Non-sealed classes follow the dispose pattern: `protected virtual void Dispose(bool disposing)` plus `GC.SuppressFinalize(this)`. Leaks still happen: static collections that only grow, event handlers never removed, unbounded caches.

## 5. Generics and constraints

```csharp
public interface IEntity { int Id { get; } }

public class Repository<T> where T : class, IEntity, new()
{
    private readonly Dictionary<int, T> _items = new();
    public void Add(T item) => _items[item.Id] = item;                 // IEntity guarantees Id
    public T? Find(int id) => _items.TryGetValue(id, out var item) ? item : null;
    public T CreateBlank() => new T();                                 // allowed by new()
}

public static T Max<T>(T a, T b) where T : IComparable<T> => a.CompareTo(b) >= 0 ? a : b;
```

Constraints (`struct`, `class`, `notnull`, `unmanaged`, `new()`, a base class, an interface) tell the compiler which members of `T` are usable. .NET generics are **reified**: `typeof(T)` works at run time, and `List<int>` stores raw ints without boxing because the JIT specializes code for value types. Going deeper: `IEnumerable<out T>` is covariant (an `IEnumerable<string>` can be used as an `IEnumerable<object>`), and `Action<in T>` is contravariant.

## 6. Extension methods

```csharp
public static class StringExtensions
{
    public static string Truncate(this string s, int max) => s.Length <= max ? s : s[..max] + "...";
}
"Hello, world".Truncate(5);   // "Hello..."
```

They are resolved at compile time from imported namespaces, and a real instance method with the same signature always wins. LINQ (`Where` and `Select` on every `IEnumerable<T>`) and ASP.NET Core setup (`builder.Services.AddDbContext(...)`) are extension methods. Think ActiveSupport core extensions, but scoped and checked at compile time.

## 7. Pattern matching and switch expressions

```csharp
static string Describe(object? value) => value switch
{
    null                  => "nothing",
    int n when n < 0      => "negative int",
    int n                 => $"int {n}",
    string { Length: 0 }  => "empty string",        // property pattern
    string s              => $"string '{s}'",
    IEnumerable<int> list => $"{list.Count()} numbers",
    _                     => "something else",      // discard: the default arm
};

static decimal Shipping(Order o) => o switch
{
    { City: "Lahore" }          => 0m,
    { Total: >= 100m }          => 0m,              // relational pattern (C# 9)
    { Total: > 50m and < 100m } => 5m,              // logical patterns: and, or, not
    _                           => 10m,
};

if (input is string { Length: > 3 } text) Console.WriteLine(text);   // test and bind in one step
```

Arms are tried top to bottom (the compiler rejects arms that earlier ones make unreachable) and it warns when a switch expression may not cover every input; an unmatched value throws `SwitchExpressionException` at run time. Compare Ruby's `case/in` and TypeScript's narrowing of discriminated unions.

## 8. Dependency injection basics

```csharp
public interface IEmailSender { Task SendAsync(string to, string body); }
builder.Services.AddScoped<IEmailSender, SmtpEmailSender>();   // Program.cs: register an implementation

public class SignupService(IEmailSender email)                 // constructor injection (C# 12 primary constructor)
{
    public Task WelcomeAsync(string to) => email.SendAsync(to, "Welcome!");
}
```

Classes depend on abstractions, the container builds the object graph and manages lifetimes, and tests pass fakes. Lifetimes (transient, scoped, singleton) and the captive-dependency pitfall are covered in [ASP.NET Core MVC](../../11-dotnet/01-aspnet-core-mvc.md).

## Common pitfalls
- Enumerating an `IEnumerable` several times, re-running an expensive query or database call each time.
- Calling `ToList()` or `AsEnumerable()` on an EF query before filtering.
- `async void`, `.Result`/`.Wait()`, and forgotten `await`s.
- Creating a new `HttpClient` per request (socket exhaustion): use `IHttpClientFactory` or a shared instance.
- Event subscriptions never removed; relying on finalizers for cleanup; `First()` on a possibly empty sequence.

## How to talk about this in an interview
- Lean on mappings you know well: LINQ over EF is like ActiveRecord relations (lazy until enumerated, translated to SQL), and C# `async`/`await` uses the same mental model as TypeScript promises, except continuations run on a thread pool.
- Be honest about depth: you used these features in university and final-year-project code; for a .NET role you would brush up on advanced async (cancellation, `ValueTask`, channels) and GC tuning.
- > **Fill in:** did your final-year project's .NET side call the Python model service over HTTP? Which async APIs did it use, and did you use LINQ with EF Core for reporting queries?

## Interview questions

**Q1. What is the difference between `IEnumerable<T>` and `IQueryable<T>`?**
`IEnumerable<T>` executes in memory: lambdas compile to delegates and every element flows through your process. `IQueryable<T>` keeps lambdas as expression trees, so a provider such as EF Core translates the whole chain into SQL and the database does the filtering, sorting and paging. Keep queries `IQueryable` until the final `ToListAsync()`.
*Follow-up:* What happens if you call a custom C# method inside an EF `Where`? Translation fails at run time, unless it is in the final projection.

**Q2. What is deferred execution?**
Most LINQ operators return a query object instead of results; the work happens when you enumerate it, and it happens again on every enumeration, seeing the source's current contents. Materialize with `ToList()` when you need a snapshot or will enumerate more than once. Aggregates such as `Count` and `First` execute immediately.

**Q3. How does `async`/`await` work, and how can it deadlock?**
An async method runs until it awaits an incomplete task, returns that task to its caller, and resumes later via a compiler-generated state machine, so threads are not blocked during I/O. If code blocks on the task with `.Result` or `.Wait()` while a single-threaded `SynchronizationContext` is captured (UI apps, classic ASP.NET), the continuation needs the blocked thread and both wait forever. Fix: await all the way up; in libraries, add `ConfigureAwait(false)`.
*Follow-up:* Does ASP.NET Core deadlock the same way? No context exists there, but blocking still wastes thread-pool threads and can starve the server under load.

**Q4. Why should you avoid `async void`?**
The caller gets no `Task`, so it cannot await completion or catch exceptions; an exception thrown from an `async void` method is raised on the synchronization context and typically crashes the process. Use it only for event handlers, whose signatures require `void`.

**Q5. What are `Func`, `Action` and `Predicate`, and how is an event different from a delegate?**
They are built-in generic delegate types: `Func` returns a value (its last type argument), `Action` returns `void`, `Predicate<T>` returns `bool`. An `event` wraps a delegate so outside code can only subscribe or unsubscribe; only the declaring class can raise it or replace the handler list.

**Q6. What does `IDisposable` do, and how is `Dispose` different from a finalizer?**
`IDisposable.Dispose` releases resources such as connections and file handles deterministically, usually through `using`, which calls it even when exceptions occur. A finalizer runs at an unpredictable time on the GC's finalizer thread and delays collection, so it is only a safety net for unmanaged handles.

**Q7. How does the .NET garbage collector work?**
It is a generational, compacting, tracing collector: objects reachable from roots survive and the rest are reclaimed. New objects live in gen 0, collected frequently; survivors move to gen 1 and gen 2, collected less often because long-lived objects rarely die. Large objects (85,000 bytes or more) go to the large object heap, collected with gen 2.

**Q8. What are extension methods, and what are generic constraints for?**
Extension methods are static methods in a static class whose first parameter is marked `this`, so they can be called like instance methods; LINQ is built this way. Generic constraints (`where T : IComparable<T>, new()`) limit which types can be used for `T`, which lets the compiler allow calls like `a.CompareTo(b)` or `new T()`.
