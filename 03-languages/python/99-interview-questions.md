# Python: Rapid-Fire Interview Questions

> How to use: cover the answer, say yours out loud, then check. Deeper explanations live in the topic files linked under each section.

Snippets assume CPython 3.10 or newer. The "What's the output?" section at the end collects the classic trick questions.

## Basics and data types (see [01-fundamentals.md](01-fundamentals.md))

**Q1. What kind of language is Python?**
Dynamically and strongly typed, multi-paradigm (procedural, OOP, functional). CPython compiles source to bytecode and runs it on a virtual machine, and everything is an object.

**Q2. What does "names are bound to objects" mean?**
Variables are labels that point at objects; assignment binds a name and never copies. Two names can share one object, which matters when the object is mutable.

**Q3. Which values are falsy?**
`None`, `False`, numeric zeros, and empty containers (`""`, `[]`, `()`, `{}`, `set()`, `range(0)`), plus objects whose `__bool__` or `__len__` says so. Everything else is truthy, including `"0"` and `[0]`.

**Q4. `/` vs `//` vs `%`?**
`/` is true division and always returns a float. `//` floors toward negative infinity (`-7 // 2 == -4`). `%` gives a remainder with the sign of the divisor (`-7 % 2 == 1`).

**Q5. Why is `0.1 + 0.2 != 0.3`, and what do you use for money?**
Binary floating point cannot represent most decimal fractions exactly. Use `decimal.Decimal` built from strings, or integer cents, and compare floats with `math.isclose`.

**Q6. Are strings mutable?**
No. Every "modification" returns a new string and `s[0] = "x"` raises `TypeError`. Build strings with f-strings or `"".join(parts)`.

**Q7. What are f-strings? Name a few format specs.**
String literals prefixed with `f` that evaluate expressions in braces: `f"{price:.2f}"`, `f"{n:,}"`, `f"{name!r}"`, and `f"{x=}"` (3.8+) for quick debugging.

**Q8. `str` vs `bytes`?**
`str` is Unicode text, `bytes` is raw binary data. Convert with `.encode("utf-8")` and `.decode("utf-8")` at the I/O boundary (files, sockets, HTTP).

**Q9. How does slicing work?**
`seq[start:stop:step]`: start inclusive, stop exclusive, negative indexes count from the end, and out-of-range slices return empty results instead of raising. `s[::-1]` reverses.

**Q10. What is `None`, and how do you test for it?**
The singleton "no value" object of type `NoneType`; functions without `return` return it. Test with `is None`, never `== None`.

**Q11. `is` vs `==`?**
`==` compares values through `__eq__`; `is` compares identity (the same object). Use `is` only for singletons such as `None`.

**Q12. What is the walrus operator?**
`:=` (3.8+) assigns inside an expression: `if (n := len(items)) > 10: ...`. Handy in `while` loops and comprehensions; easy to overuse.

**Q13. What does `for ... else` do?**
The `else` block runs only if the loop completed without `break`, which tidies up "search, then handle not found" loops.

**Q14. What do `enumerate` and `zip` do?**
`enumerate(xs, start=1)` yields `(index, item)` pairs; `zip(a, b)` pairs items and stops at the shortest input (`strict=True`, 3.10+, raises on a length mismatch).

**Q15. When should you not use a comprehension?**
When you need side effects, exception handling or several nested conditions. A plain loop is clearer, and comprehensions should only build collections.

**Q16. What does `if __name__ == "__main__":` do?**
Runs the block only when the file is executed directly, not when it is imported, so one module can double as a script.

## Collections and complexity (see [01-fundamentals.md](01-fundamentals.md))

**Q17. List vs tuple?**
Lists are mutable and suit sequences that change. Tuples are immutable, hashable when their items are, slightly lighter, and used for fixed records, multiple return values and dict keys.

**Q18. List vs set for membership tests?**
`x in list` scans, O(n); `x in set` hashes, O(1) on average. Convert to a set when you test membership repeatedly.

**Q19. What can be a dict key?**
Any hashable object: numbers, strings, tuples of hashables, frozensets and objects with a consistent `__hash__` and `__eq__`. Lists, dicts and sets cannot.

**Q20. Is a dict ordered?**
Yes: insertion order is guaranteed since Python 3.7. Updating an existing key keeps its position.

**Q21. `d[key]` vs `d.get(key)`?**
`d[key]` raises `KeyError` when the key is missing; `d.get(key, default)` returns the default (`None` if omitted).

**Q22. How do you merge two dicts?**
`{**a, **b}` or `a | b` (3.9+) build a new dict; `a.update(b)` and `a |= b` merge in place. The right-hand side wins on duplicate keys.

**Q23. What is the complexity of common list operations?**
Indexing and `append` are O(1) (append is amortized), `pop()` is O(1), `insert(0, x)` and `pop(0)` are O(n), `in` is O(n) and `sort` is O(n log n).

**Q24. How do you remove duplicates but keep the order?**
`list(dict.fromkeys(items))`, because dict keys are unique and ordered. `list(set(items))` loses the order.

**Q25. What is `collections.deque` for?**
A double-ended queue with O(1) appends and pops at both ends: queues, BFS, and bounded "last N items" buffers with `maxlen`.

**Q26. `defaultdict` vs `dict.setdefault`?**
`defaultdict(list)` creates missing values automatically on access; `setdefault(key, [])` does it one call at a time. `defaultdict` reads better in grouping loops.

**Q27. What does `Counter` do?**
Counts hashable items, for example `Counter(words).most_common(3)`. Missing keys read as 0, and counters support `+`, `-`, `&` and `|`.

**Q28. `list.sort()` vs `sorted()`?**
`list.sort()` sorts in place and returns `None`; `sorted()` accepts any iterable and returns a new list. Both are stable and accept `key` and `reverse`.

**Q29. How do you sort by several keys?**
Use a tuple key, `sorted(users, key=lambda u: (u.city, -u.age))`, or sort several times from the least to the most significant key and rely on stability.

## Functions, scope and exceptions (see [01-fundamentals.md](01-fundamentals.md))

**Q30. What are `*args` and `**kwargs`?**
`*args` collects extra positional arguments into a tuple and `**kwargs` collects extra keyword arguments into a dict. At a call site, `*` and `**` unpack a sequence and a mapping.

**Q31. What are keyword-only and positional-only parameters?**
Parameters after `*` (or after `*args`) must be passed by keyword; parameters before `/` (3.8+) must be passed by position.

**Q32. Why is a mutable default argument dangerous?**
Defaults are evaluated once, at definition time, so a default list or dict is shared by every call. Use `None` as the default and create the object inside.

**Q33. Is Python pass-by-value or pass-by-reference?**
Pass by object reference: the function receives a new name for the caller's object. Mutations are visible to the caller; rebinding the parameter is not.

**Q34. What is a lambda?**
An anonymous, single-expression function, mostly used for `key=` arguments and short callbacks. Use `def` for anything that deserves a name or needs statements.

**Q35. Explain LEGB.**
The name lookup order: Local, Enclosing function, Global (module), Built-in.

**Q36. When do you get `UnboundLocalError`?**
When a function assigns to a name, making it local for the whole function, and reads it before that assignment, as in `count += 1` on a module variable without `global`.

**Q37. `global` vs `nonlocal`?**
`global` rebinds a module-level name; `nonlocal` rebinds a name in the nearest enclosing function scope.

**Q38. What do `else` and `finally` do in a `try` statement?**
`else` runs only if the `try` block raised nothing; `finally` always runs, even after `return`, and holds cleanup code.

**Q39. What does `raise ... from ...` do?**
Chains exceptions explicitly: the original becomes `__cause__`, so the traceback shows both. `raise ... from None` hides the original context.

**Q40. Why avoid a bare `except:`?**
It catches `BaseException`, including `KeyboardInterrupt` and `SystemExit`, and hides bugs. Catch specific exceptions, or `Exception` at a boundary with logging.

**Q41. How do you define a custom exception?**
Subclass `Exception` or your app's base exception and optionally add attributes: `class CardDeclined(PaymentError): ...`. Callers can then catch the whole family.

**Q42. EAFP vs LBYL?**
EAFP (try, then handle the exception) is idiomatic and avoids check-then-act races; LBYL checks preconditions first. Python favors EAFP.

## OOP (see [02-oop.md](02-oop.md))

**Q43. What is `self`?**
The instance a method was called on, passed explicitly as the first parameter: `obj.m(x)` runs `type(obj).m(obj, x)`.

**Q44. `__init__` vs `__new__`?**
`__new__` creates the instance and `__init__` initializes it. Override `__new__` only for immutable subclasses, instance caching or metaclasses.

**Q45. Instance attributes vs class attributes?**
Instance attributes live in each object's `__dict__`; class attributes are shared by every instance. A mutable class attribute quietly becomes shared state.

**Q46. `@classmethod` vs `@staticmethod`?**
A class method receives the class and suits alternative constructors that must respect subclasses; a static method receives nothing and is a helper namespaced in the class.

**Q47. What is `@property` for?**
Exposing a computed or validated value with attribute syntax, so a plain attribute can later gain logic without breaking callers.

**Q48. What does `super()` do?**
It delegates to the next class in the instance's MRO, which may be a sibling class under multiple inheritance. That is what makes cooperative mixins work.

**Q49. What is the MRO?**
The method resolution order, computed by C3 linearization: subclasses before bases, base order preserved, each class once. Inspect it with `Class.__mro__`.

**Q50. How does Python handle the diamond problem?**
C3 places the shared base after all of its subclasses (`D, B, C, A, object`), so the base runs once and each class that calls `super()` hands off to the next.

**Q51. What is an abstract base class?**
A class inheriting from `abc.ABC` with `@abstractmethod` methods. It cannot be instantiated until a subclass implements every abstract method.

**Q52. `__str__` vs `__repr__`?**
`__repr__` is the unambiguous developer view used by the REPL, logs and containers; `__str__` is the readable user view used by `print`. `__str__` falls back to `__repr__`.

**Q53. What happens if you define `__eq__` but not `__hash__`?**
`__hash__` is set to `None` and instances become unhashable. Define both from the same immutable fields.

**Q54. What is duck typing?**
Using an object by what it can do (its methods) rather than by its declared type. `typing.Protocol` lets type checkers verify that structural compatibility.

**Q55. What does name mangling do?**
Inside class `C`, `self.__x` becomes `self._C__x`, which prevents accidental clashes with subclass attributes. It is not access control.

**Q56. What does `@dataclass` generate?**
`__init__`, `__repr__` and `__eq__` from the annotated fields, plus ordering with `order=True`, immutability and hashing with `frozen=True`, and `__slots__` with `slots=True` (3.10+).

**Q57. What are `__slots__`?**
A fixed list of attribute names that removes the per-instance `__dict__`, saving memory and blocking ad hoc attributes.

**Q58. What is a metaclass?**
The class of a class (`type` by default), which controls how classes are created. Django's `ModelBase` uses one to build `_meta`, the default manager and per-model exceptions.

**Q59. How do you make an instance callable?**
Define `__call__`. Callable instances are useful for stateful functions and for decorators implemented as classes.

**Q60. What does `functools.total_ordering` do?**
Given `__eq__` and one ordering method such as `__lt__`, it generates the remaining comparison methods.

## Iterators, generators, decorators and context managers (see [03-advanced-features.md](03-advanced-features.md))

**Q61. Iterable vs iterator?**
An iterable returns an iterator from `__iter__`; an iterator returns successive values from `__next__` and raises `StopIteration` at the end. Iterators are single-use.

**Q62. What does `yield` do?**
Turns a function into a generator: each `yield` produces a value and pauses the function with its local state intact until the next `next()` call.

**Q63. Generator expression vs list comprehension?**
A generator expression is lazy, uses constant memory and can be consumed once; a list comprehension builds the whole list immediately.

**Q64. What does `yield from` do?**
Delegates iteration (and `send()`/`throw()`) to a sub-iterator and evaluates to the sub-generator's return value.

**Q65. What is a decorator?**
A callable that takes a function and returns a replacement, usually a wrapper that adds behavior around the call. `@d` above `def f` means `f = d(f)`.

**Q66. Why use `functools.wraps` in a decorator?**
It copies the wrapped function's name, docstring and metadata onto the wrapper, keeping logs, tracebacks and introspection correct.

**Q67. How does a decorator with arguments work?**
The outer function takes the options and returns the real decorator, which takes the function and returns the wrapper: three nested levels.

**Q68. What is a context manager?**
An object with `__enter__` and `__exit__` used with `with` to guarantee cleanup even when the block raises. `contextlib.contextmanager` builds one from a generator.

**Q69. What is a closure?**
An inner function that keeps access to variables of its enclosing scope after that scope has returned.

**Q70. What does `functools.partial` do?**
Creates a new callable with some arguments pre-filled, for example `partial(send_email, priority="high")`.

**Q71. What does `lru_cache` need, and what does it give you?**
Hashable arguments. It memoizes results with least-recently-used eviction and exposes `cache_info()` and `cache_clear()`.

**Q72. What does `itertools.groupby` require?**
Input sorted (or at least grouped) by the same key, because it only groups consecutive items.

## Concurrency and memory (see [03-advanced-features.md](03-advanced-features.md))

**Q73. What is the GIL?**
A CPython lock that lets only one thread per process execute Python bytecode at a time. It is released during blocking I/O and by many C extensions.

**Q74. Does the GIL make Python code thread-safe?**
No. It protects interpreter internals, but `counter += 1` is several steps that can interleave between threads, so shared state still needs locks.

**Q75. Threads, processes or asyncio for 500 outbound HTTP calls?**
Asyncio with an async client, or a thread pool with a blocking client. The work is waiting, not computing, so processes add cost for no gain.

**Q76. Threads, processes or asyncio for resizing 10,000 images?**
Processes (a `ProcessPoolExecutor` or several task-queue workers), because CPU-bound pure-Python work cannot run in parallel threads under the GIL.

**Q77. What is the event loop?**
The asyncio scheduler that runs coroutines and switches between them whenever one awaits I/O or a timer.

**Q78. What does `asyncio.gather` do?**
Runs awaitables concurrently and returns their results in argument order; `return_exceptions=True` collects errors instead of raising the first one.

**Q79. Why must you not call `requests.get()` inside `async def`?**
It blocks the event loop's thread, freezing every other task. Use an async client or `await asyncio.to_thread(...)`.

**Q80. How does CPython free memory?**
Reference counting frees an object as soon as its count reaches zero, and a generational garbage collector reclaims reference cycles.

**Q81. What does `del x` do?**
Removes the name `x`. The object is freed only when nothing else references it.

**Q82. What is the free-threaded build?**
An optional CPython build, experimental from 3.13 (PEP 703), that runs without the GIL. Check the release notes for its status in newer versions.

## Typing, tooling and ecosystem

**Q83. Are type hints enforced at runtime?**
No. They are metadata for checkers such as mypy and pyright and for libraries that read annotations, such as dataclasses and pydantic.

**Q84. `Optional[str]` vs `str | None`?**
The same type. The `|` union syntax works from Python 3.10.

**Q85. What is a `TypedDict`?**
A type for dicts with a fixed set of string keys and value types, useful for JSON payloads. At runtime it is a plain dict.

**Q86. What is a virtual environment, and why use one?**
An isolated interpreter and package directory per project (`python -m venv .venv`), so dependency versions never collide across projects or with the system Python.

**Q87. `requirements.txt` vs `pyproject.toml`?**
`requirements.txt` is a flat, ideally pinned, package list for pip. `pyproject.toml` is the standard project file for metadata, dependencies and tool configuration, used by pip, Poetry and uv, usually with a lock file.

**Q88. What are PEP 8, ruff and black?**
PEP 8 is the official style guide; ruff is a fast linter (with a formatter), and black is an opinionated formatter. Teams run them in pre-commit hooks and CI.

**Q89. pytest vs unittest?**
`unittest` is the built-in xUnit-style framework with test classes. pytest uses plain functions with bare `assert`, fixtures and plugins (pytest-django), and is the more common choice.

## What's the output?

**Q90. What does this print?**
```python
def add(item, bucket=[]):
    bucket.append(item)
    return bucket

print(add(1))
print(add(2))
```
`[1]` then `[1, 2]`. The default list is created once, when `def` runs, and shared by both calls.

**Q91. What does this print?**
```python
funcs = [lambda: i for i in range(3)]
print([f() for f in funcs])
```
`[2, 2, 2]`. Each lambda looks up `i` when it is called, after the loop has finished. `lambda i=i: i` captures the value instead.

**Q92. What does this print in CPython?**
```python
print(int("256") is int("256"))
print(int("257") is int("257"))
```
`True` then `False`. CPython caches the integers -5 to 256; larger values created at runtime are separate objects. Never compare numbers with `is`.

**Q93. What does this print?**
```python
grid = [[0] * 3] * 3
grid[0][0] = 1
print(grid)
```
`[[1, 0, 0], [1, 0, 0], [1, 0, 0]]`. The outer `* 3` repeats references to one inner list. Build it with `[[0] * 3 for _ in range(3)]`.

**Q94. What does this print?**
```python
s = "hello"
t = s
s += " world"
print(t)

a = [1]
b = a
a += [2]
print(b)
```
`hello` then `[1, 2]`. Strings are immutable, so `+=` builds a new string and rebinds `s`; lists are mutable, so `+=` extends the shared list in place.

**Q95. What does this print?**
```python
d = {"b": 1, "a": 2}
d["c"] = 3
d["b"] = 10
print(list(d))
del d["b"]
d["b"] = 0
print(list(d))
```
`['b', 'a', 'c']` then `['a', 'c', 'b']`. Updating a key keeps its position; deleting and re-inserting moves it to the end.

**Q96. What does this print?**
```python
squares = (x * x for x in range(3))
print(sum(squares))
print(sum(squares))
```
`5` then `0`. The generator is exhausted after the first pass.

**Q97. What does this print?**
```python
class Dog:
    tricks = []

    def add_trick(self, trick):
        self.tricks.append(trick)

a, b = Dog(), Dog()
a.add_trick("roll")
print(b.tricks)
```
`['roll']`. `tricks` is a class attribute shared by every instance; create the list in `__init__`.

**Q98. What does this print?**
```python
class Counter:
    count = 0

c1, c2 = Counter(), Counter()
c1.count += 1
print(c1.count, c2.count, Counter.count)
```
`1 0 0`. `c1.count += 1` reads the class value, then assigns a new instance attribute on `c1` only.

**Q99. What does this print?**
```python
t = (1, [2])
try:
    t[1] += [3]
except TypeError:
    print("error")
print(t)
```
`error` then `(1, [2, 3])`. The list is extended in place first; only the assignment back into the tuple slot fails.

**Q100. What does this print?**
```python
print(0.1 + 0.2 == 0.3, round(2.5), round(3.5), -7 // 2)
```
`False 2 4 -4`. Float representation error, round-half-to-even, and floor division toward negative infinity.

**Q101. What does this print?**
```python
print(bool("False"), bool(""), bool([0]), True + True)
```
`True False True 2`. Non-empty strings and lists are truthy, and `bool` is a subclass of `int`.

**Q102. What does this print?**
```python
d = {1: "a", True: "b", 1.0: "c"}
print(d)
```
`{1: 'c'}`. `1`, `True` and `1.0` are equal and hash the same, so they are one key: the first key object is kept and the last value wins.

**Q103. What does this print?**
```python
def f():
    try:
        return "try"
    finally:
        print("finally")

print(f())
```
`finally` then `try`. The `finally` block runs before the function actually returns. A `return` inside `finally` would replace the original return value (and swallow exceptions), which is why recent versions warn about it.

**Q104. What does this print?**
```python
nums = [1, 2, 2, 3]
for n in nums:
    if n == 2:
        nums.remove(n)
print(nums)
```
`[1, 2, 3]`. Removing shifts the remaining items left while the loop index moves on, so the second `2` is skipped. Build a new list instead.

**Q105. What does this print?**
```python
nums = [3, 1, 2]
nums = nums.sort()
print(nums)
```
`None`. `list.sort()` sorts in place and returns `None`; use `sorted(nums)` for a new list.

**Q106. What happens here?**
```python
x = 10

def show():
    print(x)
    x = 5

show()
```
`UnboundLocalError`. The assignment makes `x` local for the whole function, so `print(x)` reads a local that has no value yet.

**Q107. What does this print?**
```python
class A:
    def who(self):
        return "A"

class B(A):
    def who(self):
        return "B" + super().who()

class C(A):
    def who(self):
        return "C" + super().who()

class D(B, C):
    def who(self):
        return "D" + super().who()

print(D().who())
```
`DBCA`. The MRO is `D, B, C, A, object`, so `super()` inside `B` reaches `C`, not `A`.

**Q108. What does this print?**
```python
a = [[1], [2]]
b = a.copy()
b[0].append(9)
b.append([3])
print(a)
```
`[[1, 9], [2]]`. The shallow copy shares the inner lists (so the `9` shows up in `a`) but has its own outer list (so `[3]` does not).

**Q109. What does this print?**
```python
def f(*args, **kwargs):
    print(args, kwargs)

f(1, 2, x=3)
f(*[4, 5], **{"y": 6})
```
`(1, 2) {'x': 3}` then `(4, 5) {'y': 6}`. Extra positionals arrive as a tuple, extra keywords as a dict, and `*`/`**` unpack at the call site.

**Q110. What does this print?**
```python
print(type(()), type((1)), type((1,)))
```
`<class 'tuple'> <class 'int'> <class 'tuple'>`. Parentheses alone do not make a tuple; the comma does (the empty tuple is the exception).
