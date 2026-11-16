# C# Fundamentals

> **Tier:** T3 (essentials) · **Related:** [C# advanced](02-advanced.md), [ASP.NET Core MVC](../../11-dotnet/01-aspnet-core-mvc.md), [Java essentials](../java/01-java-essentials.md), [C++ OOP](../c-cpp/03-cpp-oop.md), [OOP concepts](../../01-cs-fundamentals/02-oop.md)
> **On your resume:** C# in your skills list, the language behind your ASP.NET Core MVC, Blazor, Entity Framework and Dapper work at university and on the .NET side of your final-year project.

## Quick revision
- C# compiles to **IL** inside an assembly (`.dll`); the **CLR** JIT-compiles IL to native code at run time and garbage-collects memory. The **BCL** is the standard library.
- Modern .NET (5 and later, formerly .NET Core) is cross-platform; .NET Framework 4.x is the Windows-only legacy. .NET 8 (Nov 2023) and .NET 10 (Nov 2025) are LTS releases.
- **Value types** (`int`, `bool`, `decimal`, `DateTime`, enums, structs) hold their data and copy on assignment. **Reference types** (classes, `string`, arrays, delegates, records) hold a reference to a heap object. **Boxing** copies a value type into a heap object; **unboxing** casts it back to the exact type.
- `int?` is `Nullable<int>`. Nullable reference types (`string?`) are compile-time warnings, enabled with `<Nullable>enable</Nullable>` (on in new templates).
- Properties wrap fields with `get`/`set`; `init` (C# 9) allows setting only during initialization; `required` (C# 11) makes it mandatory.
- Access: `public`, `private` (default for members), `protected`, `internal` (same assembly; default for top-level types), `protected internal`, `private protected`.
- One base class, many interfaces. Methods are non-virtual by default: `virtual` + `override` for polymorphism, `new` hides, `sealed` stops inheritance or overriding.
- `string` is immutable: build strings in loops with `StringBuilder`. `Dictionary` and `HashSet` are hash tables; prefer `TryGetValue` (the indexer throws `KeyNotFoundException`).
- `throw;` rethrows and keeps the stack trace; `throw ex;` resets it. C# has no checked exceptions.
- `record` (C# 9): a class with value-based equality, a readable `ToString` and `with` copies; `record struct` (C# 10) is the value-type version.

## 1. .NET, CLR, IL, JIT and the BCL

| Term | Meaning |
|---|---|
| .NET | cross-platform runtime, SDK and libraries (`dotnet new`, `dotnet run`, `dotnet add package`) |
| CLR | the runtime: JIT compilation, garbage collection, type safety, exceptions, threading |
| IL (CIL) | CPU-independent bytecode in assemblies, which lets C#, F# and VB.NET interoperate |
| JIT | compiles each method to machine code on first call; ahead-of-time options exist (ReadyToRun, Native AOT) |
| BCL | base class library: `System.Collections.Generic`, `System.IO`, `System.Linq`, `System.Net.Http` |

NuGet is the package manager (the RubyGems or npm of .NET). Java follows the same design (bytecode plus a JIT-compiling JVM). New projects use top-level statements and implicit usings, so `Program.cs` can be just `Console.WriteLine("Hello");`.

## 2. Value types vs reference types, boxing

```csharp
var s1 = new PointS { X = 1 };
var s2 = s1; s2.X = 99;     // copies the whole value: s1.X is still 1
var c1 = new PointC { X = 1 };
var c2 = c1; c2.X = 99;     // copies the reference (one object, two variables): c1.X is now 99

struct PointS { public int X; }
class PointC { public int X; }
```

| | Value types | Reference types |
|---|---|---|
| Examples | `int`, `double`, `decimal`, `bool`, `char`, `DateTime`, enums, structs, tuples | classes, `string`, arrays, delegates, interfaces, records |
| Variable holds | the data itself | a reference to an object on the managed heap |
| Assignment | copies the value | copies the reference |
| Default | zeroed (`0`, `false`) | `null` |

"Value types live on the stack" is a simplification: a struct field inside a class lives on the heap with that object. Use structs for small immutable values (`Point`, `Money`) and classes for things with identity.

Everything is passed **by value** by default (for reference types, the reference is copied); `ref`, `out` and `in` pass by reference:

```csharp
void Reassign(PointC p) { p = new PointC { X = 5 }; }   // caller's variable unchanged
void Mutate(PointC p) { p.X = 5; }                      // caller sees X == 5
void Swap(ref int a, ref int b) => (a, b) = (b, a);
bool ok = int.TryParse("42", out int n);                // out: the method must assign n

int num = 42;
object o = num;          // boxing: allocates a heap object holding a copy
int back = (int)o;       // unboxing: type check plus copy
long bad = (long)o;      // InvalidCastException: unbox to the exact type
```

Boxing costs an allocation and GC pressure. It hides in non-generic collections (`ArrayList` boxes every `int`, `List<int>` does not), `object` parameters, and structs used through interfaces.

## 3. Nullable value types and nullable reference types

```csharp
int? maybe = null;                     // Nullable<int>
int value = maybe ?? 0;                // null-coalescing
maybe ??= 5;                           // assign only if null (C# 8)
string? nickname = null;               // with <Nullable>enable</Nullable>
int len = nickname?.Length ?? 0;       // ?. yields null instead of throwing
string name = null;                    // compiler warning: null into a non-nullable reference
```

Nullable reference types change nothing at run time (`string?` and `string` are the same type); flow analysis warns about possible null dereferences. The `!` operator silences a warning but still throws if the value is null. Same idea as TypeScript's `strictNullChecks`.

## 4. Properties

```csharp
public class Product
{
    private decimal _price;
    public int Id { get; set; }                       // auto-property
    public string Name { get; init; } = "";           // init-only: set in object initializers only
    public decimal Price                              // full property with validation
    {
        get => _price;
        set => _price = value >= 0 ? value : throw new ArgumentOutOfRangeException(nameof(value));
    }
    public bool IsFree => Price == 0;                 // computed, read-only
}

var p = new Product { Id = 1, Name = "Mouse", Price = 25m };
// p.Name = "Keyboard";                               // error: init-only
```

Properties compile to `get_X`/`set_X` methods, so validation can be added later without breaking callers, and model binding, EF Core and JSON serializers all work with public properties. `{ get; private set; }` gives public read, private write.

## 5. Classes, interfaces, abstract classes and overriding

```csharp
public abstract class Shape
{
    protected Shape(string name) => Name = name;      // abstract classes can have constructors and state
    public string Name { get; }
    public abstract double Area();                    // must be overridden
    public virtual string Describe() => $"{Name}: {Area():F2}";   // may be overridden
}

public sealed class Circle : Shape                    // sealed: no further inheritance
{
    private readonly double _r;
    public Circle(double r) : base("circle") => _r = r;   // base(...) runs the parent constructor
    public override double Area() => Math.PI * _r * _r;
}

class Animal { public virtual string Speak() => "..."; public string Kind() => "animal"; }
class Dog : Animal
{
    public override string Speak() => "Woof";         // runtime polymorphism
    public new string Kind() => "dog";                // hides Animal.Kind: chosen by compile-time type
}

Animal a = new Dog();
Console.WriteLine(a.Speak());   // Woof
Console.WriteLine(a.Kind());    // animal
```

| | Abstract class | Interface |
|---|---|---|
| Per class | one base class | any number |
| Instance fields and constructors | yes | no |
| Member access | any modifier | public by default |
| Default implementations | yes | yes, since C# 8 |
| Use for | shared code and state in an is-a hierarchy | capabilities and contracts, DI, mocking in tests |

Unlike Java (virtual by default), C# methods are non-virtual unless marked `virtual`, as in C++; forgetting `override` gives warning CS0114 and silently hides the base method.

## 6. Access modifiers, static and constructors

| Modifier | Visible to |
|---|---|
| `public` / `private` | everyone / the containing type (default for members) |
| `protected` | the type and derived types |
| `internal` | code in the same assembly (project); default for top-level types |
| `protected internal` / `private protected` | same assembly **or** derived types / derived types **in** the same assembly |

- `static` members belong to the type; a `static class` (the home of extension methods) cannot be instantiated; a static constructor runs once before first use. `const` is a compile-time constant; `readonly` fields are assigned once, in the declaration or a constructor.
- Without a declared constructor the compiler adds a parameterless one. Chain with `this(...)` and `base(...)`; C# 12 adds primary constructors (`class Service(ILogger log)`). Object initializers run after the constructor.

## 7. Collections

```csharp
var nums = new List<int> { 3, 1, 2 };
nums.Add(4); nums.Sort();                       // 1 2 3 4
var stock = new Dictionary<string, int> { ["apple"] = 3 };
if (stock.TryGetValue("kiwi", out var qty))     // one lookup, no exception
    Console.WriteLine(qty);
var seen = new HashSet<int>();
bool added = seen.Add(5);                       // false if already present
```

- `List<T>`: O(1) index, amortized O(1) `Add`, O(n) `Insert`/`Remove`/`Contains`. `Dictionary` and `HashSet`: O(1) average. Also `Queue<T>`, `Stack<T>`, `SortedDictionary<K, V>` (ordered keys) and `PriorityQueue<TElement, TPriority>` (.NET 6).
- Modifying a `List` inside a `foreach` over it throws `InvalidOperationException`. Custom key types need `Equals` and `GetHashCode` (records generate both). Expose the narrowest interface: `IEnumerable<T>`, `IReadOnlyList<T>`.

## 8. Strings and StringBuilder

```csharp
string s = "hello";
string up = s.ToUpper();                       // returns a new string; s is unchanged
bool same = "abc" == "ABC".ToLower();          // true: string == compares contents
var sb = new System.Text.StringBuilder();
for (int i = 0; i < 1000; i++) sb.Append(i).Append(',');   // one growing buffer: O(n) overall
string csv = $"{sb}";                          // interpolation (ToString under the hood)
```

Immutability gives thread safety, safe dictionary keys and interning of literals; the price is that `s += x` in a loop copies everything each time (O(n^2)). Compare case-insensitively with `string.Equals(a, b, StringComparison.OrdinalIgnoreCase)`.

## 9. Exceptions

```csharp
try { int count = int.Parse(File.ReadAllText(path)); }
catch (FileNotFoundException ex) { logger.LogWarning(ex, "Missing file {Path}", path); }
catch (FormatException) when (path.EndsWith(".tmp")) { }   // exception filter: catch only when true
catch (Exception) { throw; }       // rethrow with the original stack trace (never "throw ex;")
finally { /* always runs: cleanup */ }
```

All C# exceptions are unchecked: no `throws` clause as in Java. Prefer `TryParse`-style APIs for expected failures, and derive custom exceptions from `Exception`.

## 10. Namespaces, using and records

```csharp
// OrderDto.cs
using System.Text.Json;                         // import a namespace
using static System.Math;                       // Sqrt(2) instead of Math.Sqrt(2)
namespace Shop.Orders;                          // file-scoped namespace (C# 10)
public record OrderDto(int Id, string Customer, decimal Total);   // positional record

// elsewhere
var a = new OrderDto(1, "Sara", 50m);
var b = a with { Total = 75m };                 // non-destructive copy
Console.WriteLine(a == new OrderDto(1, "Sara", 50m));   // True: value-based equality
Console.WriteLine(a);                           // OrderDto { Id = 1, Customer = Sara, Total = 50 }
```

Positional records generate init-only properties, a constructor, `Deconstruct`, `Equals`/`GetHashCode` and `ToString`: ideal for DTOs, API responses and messages. Keep classes for entities with identity and mutable state, such as EF Core entities.

## Common pitfalls
- Expecting a copied struct to share changes (`list[0].X = 5` on a `List<PointS>` does not even compile).
- Boxing in hot paths; string concatenation in loops; `double` for money instead of `decimal`.
- `throw ex;` (loses the stack trace) and empty `catch` blocks that swallow errors.
- Forgetting `override`, which hides instead of overriding.
- The dictionary indexer on a missing key; modifying a collection during `foreach`.
- Sprinkling `!` to silence nullable warnings instead of handling the null.

## How to talk about this in an interview
- Frame it honestly: C# comes from university work and the .NET side of your final-year project, while your daily work is Rails and TypeScript. The ideas transfer: C# and TypeScript share a lead designer (Anders Hejlsberg), properties resemble Ruby's `attr_accessor`, and LINQ mirrors Ruby's `Enumerable`.
- For a .NET-heavy role, name what you would refresh first: modern C# features (records, pattern matching, nullable reference types) and async patterns.
- > **Fill in:** which C# features and .NET version your final-year project used (MVC controllers, EF Core entities, Blazor components), and what you built with them.

## Interview questions

**Q1. What is the difference between value types and reference types?**
A value-type variable holds the data and assignment copies it; a reference-type variable holds a reference to a heap object and assignment copies the reference, so two variables can share one object. Value types default to zero values and cannot inherit; reference types default to `null`.
*Follow-up:* Are value types always on the stack? No: a struct field of a class lives on the heap inside that object.

**Q2. What are boxing and unboxing, and why do they matter?**
Boxing wraps a value type in a heap object when it is treated as `object` or an interface; unboxing casts it back and must use the exact type. Each box is an allocation that adds GC pressure, which is why generic collections (`List<int>`) replaced `ArrayList`.

**Q3. Abstract class vs interface: when do you use each?**
An abstract class shares code and state in an is-a hierarchy: fields, constructors, any access level, but only one base class. An interface defines a capability that any class can implement, many at once, with default implementations (C# 8) but no instance state. ASP.NET Core favors interfaces because they pair with DI and make mocking easy.

**Q4. What is the difference between `override` and `new` on a method?**
`override` replaces a `virtual` base implementation, so the runtime type decides which version runs, even through a base reference. `new` hides the base member, so the compile-time type decides: a `Dog` stored in an `Animal` variable runs the `Animal` version.

**Q5. Why are strings immutable, and when do you use `StringBuilder`?**
Immutability makes strings thread-safe, safe as dictionary keys and shareable through interning. Every modification creates a new string, so repeated concatenation in a loop is O(n^2); `StringBuilder` appends into a growable buffer and builds the string once.

**Q6. What do the CLR, IL and the JIT do?**
The compiler emits IL and metadata into an assembly. At run time the CLR loads and verifies it, and the JIT compiles each method to native code on first call (tiered compilation can recompile hot methods with more optimization). The CLR also provides garbage collection, exception handling and type safety.

**Q7. What are nullable reference types?**
A C# 8 compiler feature that treats `string` as non-nullable and `string?` as nullable, warning when code might dereference null or store null in a non-nullable reference. It is purely compile-time and is enabled per project with `<Nullable>enable</Nullable>`.

**Q8. When would you use a record instead of a class?**
For data-centric, usually immutable types where instances with equal values should be equal: DTOs, API responses, value objects, messages. Records generate value equality, `ToString`, deconstruction and `with` copies. Classes suit entities with identity and lifecycle, like EF Core entities tracked by key.
