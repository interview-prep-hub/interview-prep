# Advanced Python Features

> **Tier:** T1 (know cold) · **Related:** [Python fundamentals](01-fundamentals.md), [Python OOP](02-oop.md), [Python rapid-fire questions](99-interview-questions.md), [Operating systems: threads and processes](../../01-cs-fundamentals/05-operating-systems.md), [JavaScript event loop](../javascript/04-async-event-loop-promises.md), [Advanced Django: Celery and async](../../10-python-django/06-django-advanced.md)
> **On your resume:** Python and Django REST Framework (PureLogics LMS), plus the Python side of your final-year project. Decorators (`@login_required`, `@action`), context managers (`transaction.atomic()`), generators (`QuerySet.iterator()`) and the concurrency model (why Celery runs worker processes) appear constantly in Django work.

## Quick revision
- **Iterable**: has `__iter__` returning an iterator. **Iterator**: has `__next__` (raises `StopIteration` when done) and an `__iter__` that returns itself. Iterators are single-use.
- `for x in obj` calls `iter(obj)` once, then `next()` until `StopIteration`.
- A function containing `yield` returns a generator object; its body runs lazily, pausing at each `yield` with its local state intact.
- Generators need O(1) memory for any length, and chained generators process a stream item by item. `yield from sub` delegates to a sub-iterator.
- A decorator is a callable that takes a function and returns a replacement: `@d` above `def f` means `f = d(f)`. Always apply `functools.wraps` to the wrapper.
- Decorators with arguments have three levels: factory(options) returns decorator(func), which returns wrapper(*args, **kwargs).
- Stacked decorators are applied bottom-up and run top-down at call time.
- Context managers implement `__enter__` and `__exit__`; `@contextlib.contextmanager` builds one from a generator with a single `yield` inside `try/finally`. A truthy return from `__exit__` suppresses the exception.
- Closures capture variables, not values: lambdas created in a loop all see the last value (late binding). Fix with a default argument (`lambda i=i: i`) or `functools.partial`.
- `functools`: `lru_cache`/`cache` memoize on hashable arguments, `partial` pre-fills arguments, `reduce` folds, `cached_property` caches per instance.
- `collections`: `defaultdict`, `Counter`, `deque` (O(1) at both ends), `namedtuple`. `itertools`: `chain`, `islice`, `groupby` (sort first), `product`, `combinations`, `accumulate`, `pairwise` (3.10+), `batched` (3.12+).
- Type hints are not enforced at runtime. Modern syntax: `list[int]` (3.9+), `int | None` (3.10+), `def first[T](xs: list[T]) -> T` (3.12+). Check with mypy or pyright.
- The **GIL** lets one thread at a time execute Python bytecode in a CPython process. Threads still help with I/O (the GIL is released while waiting) but not with CPU-bound pure Python.
- I/O-bound with blocking libraries: threads. CPU-bound: processes. Many concurrent network calls with async libraries: `asyncio`.
- `async def` defines a coroutine, `await` hands control back to the event loop, `asyncio.gather` and `TaskGroup` (3.11+) run coroutines concurrently. Never call blocking functions inside a coroutine.
- Memory: reference counting frees most objects immediately; a cyclic garbage collector reclaims reference cycles.
- `match`/`case` (3.10+) destructures sequences, mappings and objects, with guards. A bare name in a `case` captures; it does not compare.

## 1. The iterator protocol

```python
nums = [1, 2]
it = iter(nums)        # a list is iterable: iter() returns a fresh list_iterator
next(it)               # 1
next(it)               # 2
next(it)               # raises StopIteration

class Countdown:
    """An iterator: __iter__ returns self, __next__ produces the values."""
    def __init__(self, start):
        self.current = start

    def __iter__(self):
        return self

    def __next__(self):
        if self.current <= 0:
            raise StopIteration
        self.current -= 1
        return self.current + 1

list(Countdown(3))     # [3, 2, 1]
```

A `for` loop is exactly this protocol: it calls `iter()` once, then `next()` repeatedly, and stops quietly when `StopIteration` is raised.

Lists, tuples, dicts, sets, strings and `range` are **iterables**: each `iter()` call gives a fresh iterator, so you can loop over them many times. File objects, generators, `map`, `zip` and `enumerate` are **iterators**: one pass and they are exhausted. That difference explains the classic bug where a second loop over a generator silently does nothing.

## 2. Generators

```python
def countdown(start):
    while start > 0:
        yield start          # pause here and hand a value to the caller
        start -= 1           # resume here on the next next()

gen = countdown(3)           # nothing has run yet
next(gen), next(gen)         # (3, 2)
list(gen)                    # [1]: the rest
list(gen)                    # []: exhausted
```

A generator function returns a generator object that implements the iterator protocol for you. Its frame (local variables and the position in the code) is suspended at each `yield`, which is why it can produce an unbounded or huge sequence in constant memory:

```python
import sys

squares_list = [n * n for n in range(1_000_000)]
squares_gen = (n * n for n in range(1_000_000))
sys.getsizeof(squares_list)  # about 8 MB for the list's pointer array alone
sys.getsizeof(squares_gen)   # a couple of hundred bytes, whatever the length
```

**Pipelines.** Chained generators process one item through every stage before reading the next, so memory stays flat even for a multi-gigabyte file:

```python
def read_lines(path):
    with open(path, encoding="utf-8") as f:
        for line in f:
            yield line.rstrip("\n")

def parse(lines):
    for line in lines:
        user_id, action, amount = line.split(",")
        yield {"user_id": user_id, "action": action, "amount": int(amount)}

def purchases(events):
    return (e for e in events if e["action"] == "purchase")

total = sum(e["amount"] for e in purchases(parse(read_lines("events.csv"))))
```

**`yield from`** delegates to another iterable (and, for generators, forwards `send()`, `throw()` and the return value):

```python
def all_lessons(course):
    for module in course["modules"]:
        yield from module["lessons"]      # same as: for lesson in ...: yield lesson
```

In Django you meet the same idea in `QuerySet.iterator()` (stream rows in chunks instead of caching them all) and in `StreamingHttpResponse`, which accepts a generator for large CSV exports.

### Going deeper
`gen.send(value)` resumes a generator and makes the paused `yield` expression evaluate to `value`; `gen.throw(exc)` raises inside it; `gen.close()` raises `GeneratorExit` at the pause point so `finally` blocks run. Generator-based coroutines built on these methods were the foundation that `async`/`await` later formalized.

## 3. Decorators

A decorator is a function that takes a function and returns a new one that usually wraps the original. Because functions are objects, `@timed` above `def build_report` is just `build_report = timed(build_report)`.

```python
import functools
import time

def timed(func):
    @functools.wraps(func)                # copy __name__, __doc__, __module__ and set __wrapped__
    def wrapper(*args, **kwargs):         # accept anything and forward it
        start = time.perf_counter()
        try:
            return func(*args, **kwargs)
        finally:
            print(f"{func.__name__} took {time.perf_counter() - start:.3f}s")
    return wrapper

@timed
def build_report(course_id):
    time.sleep(0.2)
    return {"course_id": course_id}

build_report(7)            # prints "build_report took 0.200s" and returns the dict
build_report.__name__      # 'build_report' (it would be 'wrapper' without wraps)
```

Without `functools.wraps`, every decorated function reports itself as `wrapper`, which breaks logs, tracebacks, documentation tools and anything that registers functions by name (Celery task names, URL introspection).

**Decorators with arguments** need one more level: the outer call receives the options and returns the real decorator.

```python
import functools
import random
import time

def retry(times=3, exceptions=(Exception,), base_delay=0.5):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(1, times + 1):
                try:
                    return func(*args, **kwargs)
                except exceptions:
                    if attempt == times:
                        raise                                    # out of attempts: propagate
                    delay = base_delay * 2 ** (attempt - 1)      # exponential backoff
                    time.sleep(delay + random.uniform(0, delay / 2))   # plus jitter
        return wrapper
    return decorator

@retry(times=5, exceptions=(ConnectionError, TimeoutError))
def fetch_exchange_rates():
    ...
```

`@retry(times=5, ...)` first calls `retry(...)`, which returns `decorator`, which is then applied to the function. Writing `@retry` without parentheses passes the function in as `times`, a classic bug. Retry only idempotent operations, and only on errors that are worth retrying.

**An access-control decorator** in the style of Django's own `login_required`:

```python
import functools
from django.contrib.auth.views import redirect_to_login
from django.http import HttpResponseForbidden

def role_required(*roles):
    def decorator(view):
        @functools.wraps(view)
        def wrapper(request, *args, **kwargs):
            if not request.user.is_authenticated:
                return redirect_to_login(request.get_full_path())
            if request.user.role not in roles:
                return HttpResponseForbidden("Not allowed")
            return view(request, *args, **kwargs)
        return wrapper
    return decorator

@role_required("instructor", "admin")
def instructor_dashboard(request):
    ...
```

**Stacking** applies decorators bottom-up: `@a` over `@b` over `def f` means `f = a(b(f))`, so `a`'s wrapper runs first at call time. Put authentication outermost so nothing else runs for anonymous users.

**Class decorators** take a class and return it (often after registering it). `@dataclass` and Django's `@admin.register(Course)` are examples:

```python
EVENT_HANDLERS = {}

def handles(event_type):
    def register(cls):
        EVENT_HANDLERS[event_type] = cls   # register, then return the class unchanged
        return cls
    return register

@handles("invoice.paid")
class InvoicePaidHandler:
    def handle(self, payload): ...
```

A decorator can also be a class with `__call__`, which is handy when the decorator needs state (a call counter, a cache).

## 4. Context managers

A context manager guarantees setup and teardown around a block. `with` calls `__enter__` on entry (its return value is bound by `as`) and `__exit__(exc_type, exc, tb)` on exit, whether the block finished normally or raised.

```python
import os
import time
from contextlib import contextmanager

@contextmanager
def timer(label):
    start = time.perf_counter()
    try:
        yield                              # the body of the with-block runs here
    finally:                               # without finally, an exception would skip cleanup
        print(f"{label}: {time.perf_counter() - start:.3f}s")

class TempEnv:
    """Temporarily set an environment variable (class-based version)."""
    def __init__(self, key, value):
        self.key, self.value = key, value

    def __enter__(self):
        self.old = os.environ.get(self.key)
        os.environ[self.key] = self.value
        return self

    def __exit__(self, exc_type, exc, tb):
        if self.old is None:
            os.environ.pop(self.key, None)
        else:
            os.environ[self.key] = self.old
        return False                       # do not swallow exceptions

with timer("import"), TempEnv("FEATURE_X", "on"):
    run_import()
```

- Returning a truthy value from `__exit__` suppresses the exception. Do that only deliberately; `contextlib.suppress(FileNotFoundError)` says it explicitly.
- In a `@contextmanager` generator, an exception from the block is re-raised at the `yield`, which is why `try/finally` is mandatory there.
- You use context managers daily: `open()`, `transaction.atomic()`, `threading.Lock()`, `unittest.mock.patch(...)`, Django's `override_settings`, and `contextlib.ExitStack` for a dynamic number of them. The async versions are `async with`, `__aenter__`/`__aexit__` and `contextlib.asynccontextmanager`.

## 5. Closures and late binding

```python
def make_multiplier(factor):
    def multiply(x):
        return x * factor        # factor is captured from the enclosing scope
    return multiply

double = make_multiplier(2)
double(21)                       # 42, even though make_multiplier has returned

funcs = [lambda: i for i in range(3)]
[f() for f in funcs]             # [2, 2, 2]: each lambda looks i up when it is CALLED

funcs = [lambda i=i: i for i in range(3)]    # fix: bind the current value as a default
[f() for f in funcs]             # [0, 1, 2]
```

A closure keeps a reference to the variable (a "cell"), not a snapshot of its value. The real-world version of this bug: registering callbacks in a loop, such as `for user in users: transaction.on_commit(lambda: notify(user.id))`, notifies the last user several times. Use `functools.partial(notify, user.id)` or `lambda uid=user.id: notify(uid)`.

## 6. functools

```python
from functools import cache, cached_property, lru_cache, partial, reduce

@lru_cache(maxsize=1024)              # memoize; arguments must be hashable
def fib(n):
    return n if n < 2 else fib(n - 1) + fib(n - 2)

fib(80)                               # instant; the uncached version would take ages
fib.cache_info()                      # CacheInfo(hits=78, misses=81, maxsize=1024, currsize=81)

@cache                                # 3.9+: lru_cache(maxsize=None), unbounded
def plan_limits(plan_code): ...

send_urgent = partial(send_email, priority="high")   # pre-fill arguments
reduce(lambda acc, n: acc * n, [1, 2, 3, 4], 1)      # 24 (math.prod is clearer)

class Report:
    def __init__(self, rows):
        self.rows = rows

    @cached_property                  # computed on first access, then stored on the instance
    def total(self):
        return sum(r["amount"] for r in self.rows)
```

`lru_cache` caveats: the cache lives in one process, so Gunicorn workers do not share it (use Django's cache framework or Redis for that); decorating instance methods keeps every `self` alive; never cache functions whose result depends on time, the database or other side effects. `functools.singledispatch` gives function overloading by argument type.

## 7. itertools and collections

```python
from collections import Counter, defaultdict, deque, namedtuple
from itertools import accumulate, batched, chain, combinations, groupby, islice, pairwise, product

list(chain([1, 2], [3], (4, 5)))           # [1, 2, 3, 4, 5]
list(islice(countdown(100), 3))            # [100, 99, 98]: lazily slice any iterator
list(product("ab", [1, 2]))                # [('a', 1), ('a', 2), ('b', 1), ('b', 2)]
list(combinations("abc", 2))               # [('a', 'b'), ('a', 'c'), ('b', 'c')]
list(accumulate([5, 10, 20]))              # [5, 15, 35]: running totals
list(pairwise([1, 4, 9]))                  # [(1, 4), (4, 9)] (3.10+)
list(batched(range(7), 3))                 # [(0, 1, 2), (3, 4, 5), (6,)] (3.12+)

events = sorted(raw_events, key=lambda e: e["day"])   # groupby only groups ADJACENT items
for day, group in groupby(events, key=lambda e: e["day"]):
    print(day, len(list(group)))

by_course = defaultdict(list)              # missing keys start as list()
for course, student in [("drf", "sara"), ("drf", "ali"), ("react", "hina")]:
    by_course[course].append(student)
dict(by_course)                            # {'drf': ['sara', 'ali'], 'react': ['hina']}

words = Counter("the cat and the hat".split())
words.most_common(1)                       # [('the', 2)]
words["dog"]                               # 0, no KeyError

recent = deque(maxlen=3)                   # bounded buffer: old items fall off the left
for page in ["home", "courses", "drf", "checkout"]:
    recent.append(page)
recent                                     # deque(['courses', 'drf', 'checkout'], maxlen=3)

Point = namedtuple("Point", ["x", "y"])    # immutable tuple with named fields
p = Point(1, 2)
p.x, p[1]                                  # (1, 2)
```

`deque` is the right queue (O(1) `append`, `appendleft`, `pop`, `popleft`; `list.pop(0)` is O(n)) and the standard BFS structure. `typing.NamedTuple` is the class-based, type-hinted form of `namedtuple`. `OrderedDict` is mostly unnecessary now that `dict` is ordered, though its `move_to_end()` is handy for LRU structures.

## 8. Type hints

```python
from collections.abc import Callable, Iterable
from typing import Literal, NotRequired, TypedDict

Role = Literal["student", "instructor", "admin"]

class CoursePayload(TypedDict):          # the shape of a dict, such as parsed JSON
    title: str
    price_cents: int
    tags: NotRequired[list[str]]         # NotRequired: 3.11+

def find_user(user_id: int) -> dict | None:          # X | None: 3.10+ (Optional[X] before)
    ...

def apply_all(items: Iterable[int], fn: Callable[[int], int]) -> list[int]:
    return [fn(x) for x in items]

def first[T](items: list[T]) -> T:       # generic function, 3.12+ syntax
    return items[0]
# Before 3.12: T = TypeVar("T") at module level, then def first(items: list[T]) -> T
```

- Annotations are stored on the function (`__annotations__`) and ignored by the interpreter. Libraries can read them: dataclasses, pydantic and FastAPI do; DRF does not.
- Checkers: **mypy** (the original, configurable strictness) and **pyright** (fast, powers VS Code's Pylance). `django-stubs` and `djangorestframework-stubs` add types for Django and DRF.
- `Protocol` gives structural interfaces (see [Python OOP](02-oop.md)), `Any` switches checking off, `cast()` asserts a type to the checker, and `from __future__ import annotations` postpones evaluation so forward references work on older versions.
- Adopt gradually: type the boundaries first (service functions, DTOs, API clients), then tighten.

## 9. Concurrency: the GIL, threads, processes and asyncio

The **GIL** (global interpreter lock) is a mutex in CPython that lets only one thread execute Python bytecode at a time in a process. It keeps reference counting simple and single-threaded code fast, but it means threads cannot run pure-Python CPU work in parallel. The GIL is released during blocking I/O (sockets, files, `time.sleep`) and by many C extensions (NumPy, hashing large buffers), so threads still overlap their waiting.

| | `threading` | `multiprocessing` | `asyncio` |
|---|---|---|---|
| Unit | OS threads in one process | separate processes, each with its own interpreter and GIL | coroutines (tasks) on one thread |
| Parallel CPU work | no (GIL) | yes | no |
| Switching | preemptive, by the OS | separate processes, no switching inside Python | cooperative, only at `await` |
| Shared memory | yes, so you need locks | no, data is pickled between processes | yes, but single-threaded, so fewer races |
| Best for | I/O-bound work with blocking libraries (`requests`, DB drivers) | CPU-bound work (image processing, ML preprocessing) | thousands of concurrent network operations with async libraries |
| Main cost | race conditions, per-thread memory | startup time, pickling, more RAM | everything on the path must be async; one blocking call stalls every task |

```python
from concurrent.futures import ProcessPoolExecutor, ThreadPoolExecutor
import urllib.request

def download(url):
    with urllib.request.urlopen(url, timeout=10) as resp:    # blocking I/O releases the GIL
        return len(resp.read())

def checksum(n):
    return sum(i * i for i in range(n))                        # pure-Python CPU work

if __name__ == "__main__":                                     # required for process pools on Windows/macOS
    with ThreadPoolExecutor(max_workers=5) as pool:
        sizes = list(pool.map(download, ["https://example.com"] * 5))   # the waits overlap

    with ProcessPoolExecutor() as pool:                        # one worker per CPU core by default
        totals = list(pool.map(checksum, [10_000_000] * 4))    # real parallelism
```

```python
import asyncio
import time

async def fetch(name, delay):
    await asyncio.sleep(delay)            # stands in for a network call; yields to the event loop
    return f"{name} done"

async def main():
    start = time.perf_counter()
    results = await asyncio.gather(fetch("a", 1), fetch("b", 1), fetch("c", 1))
    print(results, f"{time.perf_counter() - start:.1f}s")    # ['a done', 'b done', 'c done'] 1.0s

    async with asyncio.TaskGroup() as tg:  # 3.11+: if one task fails, the others are cancelled
        t1 = tg.create_task(fetch("d", 1))
        t2 = tg.create_task(fetch("e", 2))
    print(t1.result(), t2.result())        # d done e done

asyncio.run(main())                        # creates the event loop and runs main() to completion
```

- Calling `fetch("a", 1)` only creates a coroutine object; nothing runs until it is awaited or wrapped in a task. A forgotten `await` produces a "coroutine was never awaited" warning.
- Never block the event loop: `time.sleep`, `requests.get` or a synchronous ORM query inside `async def` freezes every task. Use async libraries (httpx, aiohttp, asyncpg) or offload with `await asyncio.to_thread(func, arg)` (3.9+).
- `gather` returns results in argument order; `return_exceptions=True` returns exceptions instead of raising the first one. Add timeouts with `asyncio.wait_for()` or `asyncio.timeout()` (3.11+).
- In Django land: Gunicorn scales with worker **processes** (optionally threads), Celery's default prefork pool uses processes, and asyncio appears in async views, Channels websockets and fan-out HTTP calls.

### Going deeper
Python 3.13 shipped an experimental **free-threaded** build (PEP 703, often installed as `python3.13t`) that can run without the GIL. Its status has advanced in recent versions, but C extensions must support it and single-threaded code can be slower, so check the release notes before relying on it.

## 10. Memory management

```python
import gc
import sys

a = []
b = a
sys.getrefcount(a)     # 3: a, b, and the temporary reference held by the call itself
del a                  # deletes the NAME a; the list survives because b still refers to it

node = {}
node["self"] = node    # a reference cycle: its count never drops to zero by itself
del node
gc.collect()           # the cyclic collector finds unreachable cycles and frees them
```

- **Reference counting**: every object counts the references to it, and CPython frees it the moment the count reaches zero. Cleanup is prompt, but still use `with` for files and connections, because other implementations (PyPy) do not promise it.
- **Cyclic garbage collector**: a generational collector (the `gc` module) periodically looks for groups of objects that only reference each other and frees them.
- `weakref` lets caches and back-references point at objects without keeping them alive.
- Freed memory is not always returned to the operating system, so a worker's memory can stay high after a spike. Production setups recycle workers (Gunicorn `--max-requests`, Celery `--max-tasks-per-child`).
- Real leaks in web apps are usually unbounded caches, module-level lists that keep growing, or `DEBUG=True` in a long-running script (Django then records every SQL query in memory). `tracemalloc` shows which lines allocate the most.

## 11. Structural pattern matching (3.10+)

```python
def handle(event: dict) -> str:
    match event:
        case {"type": "invoice.paid", "amount": int(amount)} if amount > 0:
            return f"paid {amount}"
        case {"type": "invoice.payment_failed", "attempt": attempt}:
            return f"failed, attempt {attempt}"
        case {"type": "ping"}:
            return "pong"
        case _:
            return "ignored"

handle({"type": "invoice.paid", "amount": 1999, "currency": "usd"})   # 'paid 1999': extra keys are allowed
```

```python
from dataclasses import dataclass

@dataclass
class Point:
    x: int
    y: int

def describe(value):
    match value:
        case []:
            return "empty sequence"
        case [x]:
            return f"one item: {x}"
        case [x, y, *rest]:
            return f"starts with {x} and {y}, plus {len(rest)} more"
        case Point(x=0, y=0):
            return "origin"
        case Point(x=x, y=0) | Point(x=0, y=x):     # OR pattern: both sides bind x
            return f"on an axis at {x}"
        case str():
            return "text"                          # strings never match sequence patterns
        case _:
            return "something else"
```

`match` is not a C-style `switch`; it destructures. Patterns can be literals, captures, sequences (with `*rest`), mappings (which match a subset of keys), class patterns (dataclasses support positional ones via the generated `__match_args__`), OR patterns and `if` guards. It shines for JSON-like payloads such as webhook events, command tuples and parsed trees. For plain value dispatch, a dict of handlers is often simpler.

The big gotcha: a bare name is a **capture pattern**, not a comparison.

```python
RED = "red"
match color:
    case RED:          # captures ANY value into a new variable named RED
        ...
```

Python raises `SyntaxError` when such a capture makes later cases unreachable, but if it is the last case it silently matches everything. Compare against constants with dotted names (`Color.RED`), literals or a guard (`case c if c == RED`).

## Common pitfalls
- Looping twice over a generator, `map` or file object and getting nothing the second time.
- Forgetting `functools.wraps`, or writing `@retry` instead of `@retry()` for a decorator factory.
- Late-binding lambdas in loops, including `transaction.on_commit` callbacks that all see the last object.
- `lru_cache` on instance methods (keeps every `self` alive), on unhashable arguments (`TypeError`), or expecting it to be shared across worker processes.
- `groupby` on unsorted data, which yields the same key several times.
- Blocking calls inside `async def` (`time.sleep`, `requests`, the synchronous Django ORM, which raises `SynchronousOnlyOperation`), or forgetting `await`.
- Threads for CPU-bound work (no speedup), or a process pool without the `__main__` guard on Windows.
- Treating `counter += 1` as atomic across threads; it is a read, an add and a write, so use a lock or a database-level `F()` update.
- `case SOME_CONSTANT:` capturing instead of comparing.
- Assuming type hints validate data at runtime.

## How to talk about this in an interview
- Anchor features in Django: decorators (`@login_required`, `@api_view`, DRF's `@action`, Celery's `@shared_task`), context managers (`transaction.atomic()`, `mock.patch`, `override_settings`), generators (`QuerySet.iterator()`, streaming CSV exports), `cached_property` on models, and the process-based concurrency of Gunicorn and Celery.
- For the GIL question, state the three-way rule (threads for blocking I/O, processes for CPU, asyncio for massive concurrent I/O), then apply it: in an LMS, video transcoding or certificate PDF generation belongs in background worker processes, not in a request thread.
- If you have written a custom decorator or context manager, describe the problem it solved; if not, explain how Django's own ones work, which shows the same understanding.
- > **Fill in:** any decorator, permission helper or context manager you wrote for the LMS (for example a role check or a timing log).
- > **Fill in:** how the final-year project handled heavy Python work (batch preprocessing, vectorized libraries, GPU training) and whether you used multiprocessing.

## Interview questions

**Q1. What is the difference between an iterable and an iterator?**
An iterable can produce an iterator: it implements `__iter__` (lists, dicts, strings, `range`). An iterator implements `__next__`, which returns the next value or raises `StopIteration`, and an `__iter__` that returns itself. Iterables can be looped many times; iterators are consumed once.
*Follow-up:* Is a file object an iterable or an iterator? An iterator: it yields lines once, then you must reopen or `seek(0)`.

**Q2. What is a generator, and when would you use one?**
A function with `yield` returns a generator, a lazy iterator whose frame pauses at each `yield` and resumes on the next `next()`. Use one for large or infinite sequences, streaming files or query results, and pipelines where each stage processes one item at a time in constant memory.
*Follow-up:* What happens on the second pass over a generator? Nothing: it is exhausted. Recreate it or use a list.

**Q3. What is a decorator? Walk me through writing one.**
A callable that takes a function and returns a replacement, applied with `@` syntax (`f = deco(f)`). Define a `wrapper(*args, **kwargs)` that does work before and after calling the original, decorate the wrapper with `functools.wraps(func)`, and return it. Typical uses: timing, logging, retries, caching, authentication checks.
*Follow-up:* In what order do stacked decorators run? They are applied bottom-up, so the top decorator's wrapper runs first.

**Q4. Why use `functools.wraps`?**
It copies the wrapped function's `__name__`, `__doc__`, `__module__` and other metadata onto the wrapper and sets `__wrapped__`. Without it, every decorated function appears as `wrapper` in tracebacks, logs and documentation, and tools that register or introspect functions by name break.
*Follow-up:* How do you reach the original function in a test? Through `decorated.__wrapped__`.

**Q5. How do decorators that take arguments work?**
`@retry(times=3)` first calls `retry(times=3)`, a factory that returns the actual decorator; Python then applies that decorator to the function. So there are three nested levels: factory (options), decorator (function), wrapper (call arguments).
*Follow-up:* How can one decorator support both `@retry` and `@retry(times=3)`? Check whether the first argument is callable and branch, or accept the function as an optional keyword argument.

**Q6. What is a context manager, and how do you write one?**
An object that guarantees setup and teardown around a `with` block through `__enter__` and `__exit__`; `__exit__` runs even when the block raises. Write a class with those two methods, or decorate a generator with `contextlib.contextmanager` and put cleanup after a `yield` inside `try/finally`.
*Follow-up:* How can a context manager swallow an exception? By returning `True` from `__exit__`; `contextlib.suppress` does this explicitly for listed exception types.

**Q7. What is a closure, and what is the late-binding gotcha?**
A closure is an inner function that remembers variables from its enclosing scope after that scope has returned. It captures the variable, not its value at creation time, so lambdas created in a loop all see the loop variable's final value when they are eventually called. Bind the value with a default argument or `functools.partial`.
*Follow-up:* How does an inner function modify a captured variable? Declare it `nonlocal`.

**Q8. How does `lru_cache` work, and when should you not use it?**
It stores results in a dict keyed by the call arguments (which must be hashable) and evicts the least recently used entry beyond `maxsize`; `cache_info()` shows hits and misses. Avoid it for functions with side effects or results that depend on time or the database, on instance methods (it keeps `self` alive), and when the cache must be shared across processes, where Redis or Django's cache framework fits.
*Follow-up:* `functools.cache` vs `lru_cache`? `cache` (3.9+) is an unbounded `lru_cache(maxsize=None)`.

**Q9. What is the GIL? Can Python do multithreading?**
The global interpreter lock lets only one thread run Python bytecode at a time in a CPython process. Threads exist and are useful because the GIL is released during blocking I/O and in many C extensions, so I/O waits overlap, but CPU-bound pure-Python code gains nothing from extra threads. For CPU parallelism use processes or native libraries.
*Follow-up:* Does the GIL make your code thread-safe? No: it protects the interpreter's internals, but compound operations like `x += 1` can still interleave, so shared state needs locks.

**Q10. When would you choose threading, multiprocessing or asyncio?**
Threads for I/O-bound work that uses blocking libraries (HTTP calls with `requests`, database drivers). Processes for CPU-bound work, since each has its own interpreter and GIL, at the cost of memory and pickling. Asyncio for very high numbers of concurrent network operations when async libraries exist, on one thread with cooperative switching.
*Follow-up:* How do web servers use this? Gunicorn forks worker processes (optionally with threads); ASGI servers such as Uvicorn run an event loop per worker.

**Q11. How do `async` and `await` work? What happens if you call `time.sleep()` inside a coroutine?**
`async def` creates coroutine functions; `await` suspends the current coroutine until the awaited operation completes and lets the event loop run other tasks meanwhile. `asyncio.run()` starts the loop, and `gather` or `TaskGroup` run coroutines concurrently. `time.sleep()` blocks the whole thread, so every task on the loop freezes; use `await asyncio.sleep()` or move blocking work to `asyncio.to_thread()`.
*Follow-up:* Does async make one request faster? No; it improves throughput by overlapping waits, not the latency of a single operation.

**Q12. How does Python manage memory?**
CPython counts references to each object and frees it immediately when the count drops to zero. A generational cyclic garbage collector periodically frees groups of objects that reference each other but are unreachable. Memory comes from private allocators, so freed memory is not always returned to the operating system.
*Follow-up:* How do you hunt a memory leak? Compare `tracemalloc` snapshots, look for unbounded caches or global lists, and recycle long-running workers as a safety net.

**Q13. When would you use `defaultdict`, `Counter` and `deque`?**
`defaultdict(list)` groups items without "if key not in dict" boilerplate. `Counter` counts hashable items and offers `most_common()`, which is perfect for frequency questions. `deque` gives O(1) appends and pops at both ends, so it is the queue for BFS and job buffers, and `maxlen` turns it into a bounded "last N" buffer.
*Follow-up:* Why not `list.pop(0)` for a queue? It shifts every remaining element, which is O(n).

**Q14. Are type hints enforced? How would you use them on a team?**
No. The interpreter stores annotations but does not check them; mypy or pyright in the editor and CI do. Add hints at boundaries first (service functions, DTOs, API clients), use stubs such as `django-stubs`, avoid `Any`, and use `Protocol` for collaborator interfaces. For runtime validation of input, use serializers or pydantic.
*Follow-up:* `Optional[int]` vs `int | None`? The same type; the `|` syntax works from 3.10.

**Q15. What is structural pattern matching good for, and what is its main gotcha?**
`match`/`case` destructures data: mapping patterns pick keys out of dicts, sequence patterns unpack lists, class patterns check type and attributes, and guards add conditions. It suits webhook payloads, parsed commands and tree structures. The gotcha is that a bare name captures any value instead of comparing to a constant; use dotted names or literals.
*Follow-up:* Do mapping patterns require an exact key set? No, extra keys are ignored unless you capture them with `**rest`.

## Further reading
- [Functional programming HOWTO (iterators and generators)](https://docs.python.org/3/howto/functional.html)
- [functools](https://docs.python.org/3/library/functools.html), [itertools](https://docs.python.org/3/library/itertools.html) and [collections](https://docs.python.org/3/library/collections.html)
- [asyncio](https://docs.python.org/3/library/asyncio.html) and [concurrent.futures](https://docs.python.org/3/library/concurrent.futures.html)
- [typing](https://docs.python.org/3/library/typing.html)
- [The match statement](https://docs.python.org/3/tutorial/controlflow.html#match-statements)
