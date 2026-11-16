# Java Essentials

> **Tier:** T3 (essentials) · **Related:** [C# fundamentals](../csharp/01-fundamentals.md), [OOP concepts](../../01-cs-fundamentals/02-oop.md), [C++ OOP](../c-cpp/03-cpp-oop.md), [Arrays, strings and hashing](../../02-dsa/01-arrays-strings-hashing.md), [Operating systems](../../01-cs-fundamentals/05-operating-systems.md)
> **On your resume:** Java in your skills list, from academic work. Expect the classic core-Java questions below, and lean on the close parallels with C# and C++.

## Quick revision
- **JVM** runs bytecode; **JRE** = JVM + standard library (Oracle stopped shipping it separately with Java 11); **JDK** = JRE + tools (`javac`, `jar`, debugger). `javac` turns `.java` into `.class` bytecode, which the JVM interprets and JIT-compiles when hot.
- Entry point: `public static void main(String[] args)`. Eight primitives plus wrapper classes; autoboxing converts between them, and unboxing `null` throws `NullPointerException`.
- `==` compares references for objects (values for primitives); `equals` compares content. Override `equals` and `hashCode` together.
- `String` is immutable and literals live in the string pool; build strings in loops with `StringBuilder`.
- One superclass, many interfaces (with `default` and `static` methods since Java 8). Methods are virtual by default; `final` blocks reassignment, overriding or subclassing.
- Checked exceptions (`Exception` minus `RuntimeException`) must be caught or declared; unchecked ones need not be. try-with-resources closes `AutoCloseable` objects.
- `HashMap`: buckets indexed by `(n - 1) & hash`; collision lists become red-black trees past 8 entries (once the table has 64+ buckets, Java 8); doubles above a 0.75 load factor; one `null` key allowed. `Hashtable` is legacy; `ConcurrentHashMap` is the thread-safe map.
- Generics are erased after compilation: no `new T()`, no `List<int>`, no runtime difference between `List<String>` and `List<Integer>`.
- Streams (Java 8) are lazy, single-use pipelines that run on a terminal operation; `Optional<T>` models a possibly absent return value.
- `synchronized` = mutual exclusion plus visibility; `volatile` = visibility only. Prefer `ExecutorService` to raw threads.
- JVM memory: heap (young and old generations), a stack per thread, metaspace for class metadata (replaced PermGen in Java 8). G1 is the default collector since Java 9.

## 1. JDK, JRE, JVM, bytecode and the JIT

```java
public class Main {
    public static void main(String[] args) {     // public, static (no object needed), void, String[]
        System.out.println("Hello, " + (args.length > 0 ? args[0] : "world"));
    }
}
```

`javac Main.java` produces `Main.class`; `java Main Sara` starts a JVM that loads and verifies the bytecode, interprets it, and lets the HotSpot JIT compile hot methods to native code. Since Java 11, `java Main.java` compiles and runs a single file in one step, and recent versions allow simpler entry points for small programs (check your version's release notes). "Write once, run anywhere": the same bytecode runs on any OS with a JVM. C# uses the same model with IL and the CLR.

## 2. Primitives, wrappers and autoboxing

```java
Integer x = 127, y = 127;          // autoboxing calls Integer.valueOf, which caches -128..127
Integer p = 128, q = 128;
System.out.println(x == y);        // true: the same cached object
System.out.println(p == q);        // false: two different objects
System.out.println(p.equals(q));   // true: compares values
```

The primitives are `byte`, `short`, `int`, `long`, `float`, `double`, `char` and `boolean`. Collections and generics need objects (`List<Integer>`, never `List<int>`), so boxing is everywhere and costs allocations in hot loops. Java has no unsigned integer types (`char` is an unsigned 16-bit UTF-16 unit), and `int` overflow wraps silently.

## 3. Strings: immutability, the pool, StringBuilder

```java
String a = "hello";                    // literal: stored in the string pool
String b = "hello";                    // the same pooled object
String c = new String("hello");        // a new heap object
System.out.println(a == b);            // true
System.out.println(a == c);            // false
System.out.println(a.equals(c));       // true; and a == c.intern() is true
```

Strings are immutable so pooled literals can be shared safely, they are thread-safe without locks, their hash code can be cached (fast `HashMap` keys), and security-sensitive values (class names, paths, URLs) cannot change after validation. The cost: `s += x` in a loop creates a new object every time, so use `StringBuilder` (`sb.append(x)`, then `sb.toString()`). `StringBuffer` is its synchronized legacy twin.

## 4. == vs equals and the equals/hashCode contract

```java
public final class Point {
    private final int x, y;
    public Point(int x, int y) { this.x = x; this.y = y; }
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Point)) return false;
        Point other = (Point) o;
        return x == other.x && y == other.y;
    }
    @Override
    public int hashCode() { return java.util.Objects.hash(x, y); }   // equal objects => equal hash codes
}
```

- `equals` must be reflexive, symmetric, transitive and consistent, with `x.equals(null)` false. If `a.equals(b)`, then `a.hashCode() == b.hashCode()`; the reverse need not hold (collisions are fine).
- Override only `equals` and hash collections break: `set.contains(new Point(1, 2))` returns false, because lookup first goes to the bucket picked by the identity-based hash code. Records (Java 16), such as `record Point(int x, int y) {}`, generate both methods.

## 5. Classes, interfaces, abstract classes and modifiers

```java
interface Notifier {
    void send(String to, String msg);                                    // implicitly public abstract
    default void sendTwice(String to, String msg) { send(to, msg); send(to, msg); }   // default method (Java 8)
}
abstract class Shape {
    protected final String name;
    Shape(String name) { this.name = name; }
    abstract double area();                                     // subclasses must implement
}
final class Circle extends Shape {                              // final: no subclasses
    private final double r;
    Circle(double r) { super("circle"); this.r = r; }          // super(...) calls Shape's constructor
    @Override double area() { return Math.PI * r * r; }
}
```

| Modifier | Same class | Same package | Subclass elsewhere | Everywhere |
|---|---|---|---|---|
| `public` | yes | yes | yes | yes |
| `protected` | yes | yes | yes | no |
| none (package-private) | yes | yes | no | no |
| `private` | yes | no | no | no |

- Single class inheritance avoids the diamond problem for state; if two interfaces supply the same `default` method, the class must override it (and may call `A.super.method()`).
- Every non-static, non-private, non-final instance method is virtual, unlike C# and C++ where you opt in with `virtual`; static methods are hidden, not overridden. `final` on a variable fixes the reference, not the object: a `final List` can still be modified.

## 6. Exceptions: checked vs unchecked

| | Checked | Unchecked |
|---|---|---|
| Hierarchy | `Exception` subclasses outside `RuntimeException` | `RuntimeException` and `Error` subclasses |
| Compiler rule | catch it or declare it with `throws` | no requirement |
| Examples | `IOException`, `SQLException`, `InterruptedException` | `NullPointerException`, `IllegalArgumentException`, `IndexOutOfBoundsException` |
| Meaning | recoverable conditions outside your control | programming bugs (`Error`: JVM trouble such as `OutOfMemoryError`) |

```java
import java.io.*;

class FileUtil {
    static String firstLine(String path) throws IOException {      // checked: must be declared
        try (BufferedReader reader = new BufferedReader(new FileReader(path))) {   // try-with-resources
            return reader.readLine();
        }                                                          // reader.close() runs automatically
    }
}
```

Resources close in reverse order even when exceptions occur, and an exception from `close()` is attached as a suppressed exception instead of hiding the original. `finally` runs even after `return` (not after `System.exit`). C# and Kotlin have no checked exceptions.

## 7. Collections framework

`Collection` splits into `List` (`ArrayList`, `LinkedList`), `Set` (`HashSet`, `LinkedHashSet`, `TreeSet`) and `Queue`/`Deque` (`ArrayDeque`, `PriorityQueue`); `Map` (`HashMap`, `LinkedHashMap`, `TreeMap`) is a separate hierarchy.

| | `ArrayList` | `LinkedList` |
|---|---|---|
| Structure | resizable array (grows about 1.5x) | doubly linked list |
| `get(i)` | O(1) | O(n) |
| Insert or remove in the middle | O(n) shifting | O(1) at a known node, O(n) to reach it |
| Verdict | the default (compact, cache-friendly) | rarely better; use `ArrayDeque` for stacks and queues |

**How `HashMap` works:**
1. It keeps a table of buckets (default capacity 16, always a power of two). `put(k, v)` takes `k.hashCode()`, mixes in the high bits (`h ^ (h >>> 16)`) and picks the bucket with `(n - 1) & hash`.
2. Keys in the same bucket form a linked list searched with `equals`. Since Java 8, a bucket with more than 8 entries becomes a red-black tree (if the table has at least 64 buckets; otherwise the table grows instead), so a crowded bucket costs O(log n) rather than O(n).
3. When the size exceeds capacity x load factor (0.75), the table doubles and entries are redistributed.
4. One `null` key and any number of `null` values are allowed; it is not thread-safe; iteration order is unspecified (`LinkedHashMap` keeps insertion order, `TreeMap` sorts keys).

| | `HashMap` | `Hashtable` | `ConcurrentHashMap` |
|---|---|---|---|
| Thread-safe | no | yes: every method synchronized on one lock (slow under contention) | yes: CAS plus per-bin locking (Java 8), scales with threads |
| Nulls | one null key, null values | none | none |
| Iterators | fail-fast (`ConcurrentModificationException`) | fail-fast | weakly consistent, never throw |
| Status | the default | legacy (Java 1.0) | for maps shared between threads |

## 8. Generics and type erasure

```java
List<String> names = new ArrayList<>();      // diamond operator (Java 7); names.add(42) will not compile
class Util {
    static <T extends Comparable<T>> T max(T a, T b) { return a.compareTo(b) >= 0 ? a : b; }
}
```

Generics exist only at compile time: the compiler checks types, inserts casts, then **erases** the type arguments, so `List<String>` and `List<Integer>` are the same class at run time. Consequences: no `new T()` or `new T[]`, no `instanceof List<String>`, no primitive type arguments (`List<Integer>` boxes), and no overloading `f(List<String>)` against `f(List<Integer>)`. C# generics, by contrast, are reified. Wildcards follow PECS: producer `extends`, consumer `super` (`List<? extends Number>` can be read as numbers).

## 9. Lambdas, streams and Optional

```java
// record Order(String customer, double total, String city) {}   (Java 16 record)
List<String> bigSpenders = orders.stream()
    .filter(o -> o.total() > 50)                              // intermediate operations are lazy
    .sorted(Comparator.comparing(Order::total).reversed())    // method references
    .map(Order::customer)
    .collect(Collectors.toList());                            // terminal operation runs the pipeline

String firstLahore = orders.stream()
    .filter(o -> o.city().equals("Lahore"))
    .findFirst()                                              // Optional<Order>
    .map(Order::customer)
    .orElse("none");
```

A lambda implements any functional interface (one abstract method): `Function<T, R>`, `Predicate<T>`, `Consumer<T>`, `Supplier<T>`, `Runnable`, `Comparator<T>`. Streams are lazy, single-use, and can run in parallel with `parallelStream()` (worth it only for large CPU-bound work); `Collectors.groupingBy` builds maps of groups. Use `Optional` for return values that may be absent, consume it with `map`, `orElse`, `orElseThrow` or `ifPresent` instead of an unchecked `get()`, and avoid it for fields and parameters.

## 10. Threads and concurrency basics

```java
class Counter {
    private int count = 0;
    public synchronized void increment() { count++; }   // one thread at a time per object, plus visibility
    public synchronized int get() { return count; }
}
// inside a method declared "throws InterruptedException"
ExecutorService pool = Executors.newFixedThreadPool(4);
Counter counter = new Counter();
for (int i = 0; i < 1000; i++) pool.submit(counter::increment);   // a Runnable via method reference
pool.shutdown();
pool.awaitTermination(1, TimeUnit.MINUTES);
System.out.println(counter.get());   // 1000; without synchronized, often less
```

- `new Thread(runnable).start()` runs code on a new thread (calling `run()` directly runs it on the current one). `synchronized` gives mutual exclusion on the object's monitor plus a happens-before guarantee; `volatile` only guarantees visibility and ordering, so `count++` on a volatile field still races: use `AtomicInteger`.
- `ExecutorService` reuses pooled threads; `Future` and `CompletableFuture` (Java 8) compose async results; virtual threads (Java 21) let blocking-style code scale to huge numbers of tasks. Deadlocks come from taking locks in inconsistent orders.

## 11. JVM memory and garbage collection

| Area | Holds | Shared? |
|---|---|---|
| Heap | all objects and arrays: young generation (Eden plus survivor spaces) and old generation | by all threads |
| Stack | one per thread: frames with locals, primitives and references | per thread |
| Metaspace | class metadata (replaced PermGen in Java 8), in native memory | by all threads |

Objects become garbage when unreachable from GC roots (stack references, static fields, live threads). Most objects die young, so frequent minor collections sweep the young generation cheaply and survivors are promoted to the old generation, which is collected less often. G1 is the default since Java 9, with ZGC and Shenandoah for very low pauses. Deep recursion throws `StackOverflowError`, an exhausted heap `OutOfMemoryError`. `System.gc()` is only a hint, and `finalize()` is deprecated (use try-with-resources or `Cleaner`).

## Common pitfalls
- `==` on `String` or `Integer` values; overriding `equals` without `hashCode`; mutating an object while it is a `HashMap` key.
- Removing items in a for-each loop (`ConcurrentModificationException`): use `Iterator.remove()` or `removeIf`.
- Auto-unboxing a `null` wrapper; swallowing exceptions with an empty `catch`.
- Calling `run()` instead of `start()`; using `volatile` for compound operations like `count++`.

## How to talk about this in an interview
- Be upfront: Java is academic for you, while production work is Rails and TypeScript. Then show transfer: the JVM model mirrors .NET's CLR, and OOP, collections and concurrency concepts carry over from C# and C++.
- If a role needs Java, name the ramp-up honestly: Spring Boot, Maven or Gradle, and JPA/Hibernate (conceptually close to ActiveRecord and EF Core).
- > **Fill in:** which course or semester project you built in Java (an OOP assignment, a desktop app, data structures), and what you used: Swing or JavaFX, JDBC, collections, threads.

## Interview questions

**Q1. What is the difference between the JDK, JRE and JVM?**
The JVM executes bytecode (interpreting it and JIT-compiling hot code) and manages memory. The JRE is the JVM plus the standard class library needed to run programs. The JDK adds development tools (`javac`, `jar`, `jshell`, debuggers). Since Java 11, Oracle ships only the JDK (some vendors still offer JRE builds), and `jlink` can build a trimmed runtime for an app.

**Q2. What is the difference between `==` and `equals()`?**
For objects, `==` checks whether two references point to the same object, while `equals()` checks logical equality as the class defines it (`String` compares characters). For primitives, `==` compares values. Traps: `new String("a") == "a"` is false, and `Integer` values outside -128 to 127 compare false with `==`.
*Follow-up:* Why override `hashCode()` too? Hash collections pick the bucket by `hashCode()` before calling `equals()`, so equal objects with different hash codes are never found.

**Q3. How does `HashMap` work internally?**
Entries live in an array of buckets. `put` computes the key's hash (spreading the high bits), picks a bucket with `(n - 1) & hash`, and chains colliding entries in a list compared with `equals`. Since Java 8, long chains (more than 8 entries, with at least 64 buckets) become red-black trees. When the size passes capacity x 0.75, the table doubles and entries are redistributed. Average `get` and `put` are O(1).
*Follow-up:* Why is the capacity a power of two? So `(n - 1) & hash` works as a fast modulo.

**Q4. Why is `String` immutable? What is the string pool?**
The pool stores one shared instance per literal, which is only safe because strings cannot change. Immutability also makes strings thread-safe, lets them cache their hash code for fast map lookups, and stops security-sensitive values from being altered after checks. `new String("x")` bypasses the pool; `intern()` returns the pooled copy.

**Q5. What is the difference between checked and unchecked exceptions?**
Checked exceptions (subclasses of `Exception` outside `RuntimeException`, such as `IOException`) must be caught or declared with `throws`, so the compiler makes callers plan for recoverable failures. Unchecked exceptions (`RuntimeException` and `Error`, such as `NullPointerException`) usually signal bugs and need no declaration. Many modern APIs prefer unchecked exceptions to avoid boilerplate.

**Q6. `HashMap` vs `Hashtable` vs `ConcurrentHashMap`?**
`HashMap` is unsynchronized and allows a `null` key. `Hashtable` is the legacy class that synchronizes every method on one lock and rejects nulls. `ConcurrentHashMap` is the modern thread-safe map: lock-free reads, CAS and per-bin locks for writes, weakly consistent iterators, no nulls.

**Q7. What is type erasure?**
The compiler uses generic type arguments for checking and inserts casts, then removes them from the bytecode, so at run time `List<String>` and `List<Integer>` are both plain `List`. That is why you cannot write `new T()`, use primitives as type arguments, or test `instanceof List<String>`. It kept generics backward compatible with pre-Java-5 code.

**Q8. `synchronized` vs `volatile`?**
`synchronized` provides mutual exclusion (one thread at a time per monitor) and visibility of changes. `volatile` provides only visibility and ordering for a single variable, so it suits flags like `running = false` but not compound actions like `count++`, which need locks or atomics.
