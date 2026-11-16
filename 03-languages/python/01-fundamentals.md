# Python Fundamentals

> **Tier:** T1 (know cold) · **Related:** [Python OOP](02-oop.md), [Advanced Python features](03-advanced-features.md), [Python rapid-fire questions](99-interview-questions.md), [Programming fundamentals](../../01-cs-fundamentals/01-programming-fundamentals.md), [Ruby basics](../ruby/01-basics.md), [Django fundamentals](../../10-python-django/01-django-fundamentals.md)
> **On your resume:** Python is in your skills list. You used it with Django REST Framework to build the LMS (Coursera clone) at PureLogics, and on the Python side of your final-year project (human activity recognition with fine-tuned LLMs).

## Quick revision
- CPython compiles source to bytecode (cached as `.pyc` files in `__pycache__`) and runs it on a stack-based virtual machine. "Interpreted" really means "compiled to bytecode, then interpreted".
- Dynamic (types live on values and are checked at runtime) and strong (`"1" + 1` raises `TypeError`).
- Variables are names bound to objects. Assignment never copies; it binds another name to the same object.
- `/` always returns a float, `//` floors (`-7 // 2 == -4`), `%` takes the sign of the divisor. `int` has arbitrary precision. `round(2.5) == 2` (round half to even).
- Floats are binary approximations: `0.1 + 0.2 != 0.3`. Use `Decimal("19.99")` or integer cents for money.
- Strings are immutable Unicode. Build them with f-strings and `"".join(parts)`, not `+=` in a loop.
- Falsy: `None`, `False`, `0`, `0.0`, `""`, `[]`, `()`, `{}`, `set()`, `range(0)`. Everything else is truthy unless it defines `__bool__` or `__len__`.
- Compare with `None` using `is None`, never `== None`.
- Mutable: `list`, `dict`, `set`, most custom objects. Immutable: `int`, `float`, `str`, `tuple`, `frozenset`, `bytes`. Only hashable (normally immutable) values can be dict keys or set members.
- `dict` keeps insertion order (a language guarantee since 3.7). `dict` and `set` lookups are O(1) on average; `x in list` is O(n).
- `is` checks identity (same object); `==` checks value (`__eq__`).
- `copy.copy` is shallow (new outer container, shared inner objects); `copy.deepcopy` copies recursively.
- Default arguments are evaluated once, when `def` runs. Use `None` as the default for mutable values.
- Comprehensions build collections eagerly; generator expressions `(x for x in xs)` are lazy and single-use.
- Parameter order: `def f(pos_only, /, normal, *args, kw_only, **kwargs)`.
- Name lookup is LEGB: Local, Enclosing, Global, Built-in. Assigning to a name anywhere in a function makes it local unless you declare `global` or `nonlocal`.
- `try/except/else/finally`: `else` runs only when nothing was raised, `finally` always runs. `raise NewError(...) from err` chains exceptions.
- Idiomatic Python is EAFP (try it and handle the exception) more than LBYL (check first).
- `if __name__ == "__main__":` runs only when the file is executed directly, not when it is imported.
- One virtual environment per project (`python -m venv .venv`). `requirements.txt` lists pinned packages; `pyproject.toml` is the standard project file used by pip, Poetry and uv.
- Open files with `with open(path, encoding="utf-8") as f:` so they close even on errors.
- `sorted()` returns a new list; `list.sort()` sorts in place and returns `None`. Both are stable and accept `key=` and `reverse=`.
- `map`, `filter`, `zip`, `enumerate` and `range` are lazy. `any([])` is `False`, `all([])` is `True`.

## 1. How Python runs: CPython, bytecode and the interpreter

"Python" is a language specification; **CPython** (written in C) is the reference implementation from python.org that almost everyone runs. Others exist (PyPy has a JIT, MicroPython targets microcontrollers), but interview questions assume CPython.

What happens when you run `python app.py`:
1. The source is parsed into an abstract syntax tree and compiled to **bytecode**: compact instructions for a stack-based virtual machine.
2. Bytecode for imported modules is cached as `.pyc` files in `__pycache__/`, so the next import skips compilation if the source is unchanged. The main script is not cached.
3. The interpreter loop executes the bytecode instruction by instruction.

```python
import dis

def add(a, b):
    return a + b

dis.dis(add)   # prints instructions such as LOAD_FAST and BINARY_OP (names vary by version)
```

Python 3.11 made the interpreter much faster (a specializing adaptive interpreter), and 3.13 shipped an experimental JIT that is off by default. CPU-heavy work still belongs in C-backed libraries (NumPy, pandas) or in separate processes; see [the GIL](03-advanced-features.md).

## 2. Dynamic, strong typing and names bound to objects

**Dynamic**: a variable has no declared type; the object it points to does, and type errors surface when the line runs. **Strong**: Python almost never converts between unrelated types for you.

```python
x = 42
x = "now a string"          # fine: the name is rebound to another object

"Total: " + 5               # TypeError: can only concatenate str (not "int") to str
"Total: " + str(5)          # 'Total: 5'
f"Total: {5}"               # 'Total: 5'
1 + 2.5                     # 3.5: numeric types do mix (int is widened to float)
```

Every value is an object with an identity (`id()`), a type (`type()`) and a value. A variable is a **name bound to an object**, like a label stuck on a box rather than the box itself:

```python
a = [1, 2, 3]
b = a                # no copy: two names, one list
b.append(4)
print(a)             # [1, 2, 3, 4]
print(a is b)        # True
a = [9]              # rebinding a does not touch b
print(b)             # [1, 2, 3, 4]
```

Arguments are passed the same way ("call by sharing"): the function gets a new name bound to the caller's object. Mutating the object is visible to the caller; rebinding the parameter is not. Type hints (`def total(items: list[float]) -> float:`) document intent and power checkers such as mypy and pyright, but the interpreter ignores them at runtime.

## 3. Numbers

```python
7 / 2          # 3.5   true division always returns a float
7 // 2         # 3     floor division
-7 // 2        # -4    floors toward negative infinity, not toward zero
-7 % 2         # 1     the result takes the sign of the divisor
2 ** 100       # 1267650600228229401496703205376 (ints never overflow)
round(2.5)     # 2     round half to even ("banker's rounding")
round(3.5)     # 4
int("42")      # 42;   int("4.2") raises ValueError
int(3.9)       # 3     truncates toward zero
True + True    # 2     bool is a subclass of int
0.1 + 0.2      # 0.30000000000000004
```

- `int` is arbitrary precision, `float` is a 64-bit IEEE 754 double, and `complex` exists (`3 + 4j`).
- Compare floats with `math.isclose(a, b)`, never `==`.
- For money use `Decimal("19.99")` (build it from a string; `Decimal(0.1)` captures the float's binary error) or integer cents. Django's `DecimalField` returns `Decimal` values for this reason.
- Underscores make literals readable: `1_000_000`.

## 4. Strings

`str` is an immutable sequence of Unicode code points; `bytes` is an immutable sequence of raw bytes. Convert at the edges with `text.encode("utf-8")` and `data.decode("utf-8")`.

```python
name, price = "Ali", 1234.5
f"{name} owes {price:,.2f}"      # 'Ali owes 1,234.50'
f"{name!r}"                      # "'Ali'" (uses repr)
f"{price=}"                      # 'price=1234.5' (3.8+, handy when debugging)

s = "interview"
s[0], s[-1]          # ('i', 'w')
s[0:5]               # 'inter'   start inclusive, stop exclusive
s[5:]                # 'view'
s[::-1]              # 'weivretni'  reversed
s[::2]               # 'itriw'
s[100:]              # ''  slices never raise IndexError (s[100] does)
s[0] = "I"           # TypeError: 'str' object does not support item assignment
```

Everyday methods (all return new strings): `strip()`, `lower()`, `upper()`, `replace(old, new)`, `split(sep)`, `"-".join(items)`, `startswith()`, `endswith()`, `removeprefix()` (3.9+), `isdigit()`. `find()` returns `-1` when the substring is missing, while `index()` raises `ValueError`. `"a b  c".split()` with no argument splits on runs of whitespace: `['a', 'b', 'c']`.

Because strings are immutable, `result += piece` in a loop can copy the whole string every time (O(n^2) in the worst case). Collect pieces in a list and call `"".join(pieces)` once.

## 5. Truthiness and None

In `if` and `while`, Python calls `bool()` on the value. Falsy values: `None`, `False`, zero of any numeric type (`0`, `0.0`, `Decimal(0)`) and empty containers (`""`, `[]`, `()`, `{}`, `set()`, `range(0)`). Everything else is truthy, including `"0"`, `"False"`, `[0]` and `[[]]`. Your own classes become falsy by defining `__bool__` or `__len__`.

```python
items = []
if not items:                    # idiomatic emptiness check
    print("empty")

user = find_user(1)              # functions return None when nothing is found (or no return runs)
if user is None:                 # identity check: None is a singleton
    print("not found")

limit = 0
page_size = limit or 20          # BUG: 0 is falsy, so you get 20
page_size = 20 if limit is None else limit   # correct when 0 is a valid value
```

`and` and `or` return one of their operands, not necessarily a bool: `"" or "guest"` is `'guest'` and `[1] and "ok"` is `'ok'`. Compared with Ruby, the big difference is that empty strings, empty collections and `0` are falsy in Python (Ruby treats only `nil` and `false` as falsy).

## 6. Built-in collections: list, tuple, dict, set

```python
nums = [3, 1, 2]                 # list: ordered, mutable, duplicates allowed
nums.append(4); nums.extend([5, 6]); nums.insert(0, 0)
nums.pop()                       # removes and returns the last item (6)
nums.remove(3)                   # removes the first 3; ValueError if missing
nums.sort(); nums.reverse()      # both work in place and return None

point = (10, 20)                 # tuple: ordered, immutable
x, y = point                     # unpacking
first, *rest = [1, 2, 3]         # first = 1, rest = [2, 3]
single = (1,)                    # the comma makes the tuple; (1) is just 1

user = {"name": "Sara", "role": "student"}     # dict: key -> value
user["email"] = "sara@example.com"
user.get("phone")                # None instead of KeyError
user.get("phone", "n/a")         # with a default
user.setdefault("tags", []).append("new")      # insert if missing, then use it
for key, value in user.items():
    print(key, value)
merged = user | {"role": "instructor"}         # 3.9+; the right side wins

tags = {"python", "django"}      # set: unordered, unique, fast membership
tags.add("drf"); tags.discard("java")           # discard never raises
tags & {"python", "go"}          # {'python'} intersection
tags | {"go"}                    # union; tags - {"drf"} is the difference
empty = set()                    # {} is an empty dict, not a set
```

| Operation | list | tuple | dict | set |
|---|---|---|---|---|
| Index or key lookup | O(1) `a[i]` | O(1) | O(1) average `d[k]` | n/a |
| Membership `x in c` | O(n) | O(n) | O(1) average (keys) | O(1) average |
| Append or add | O(1) amortized | n/a (immutable) | O(1) average | O(1) average |
| Insert or delete at the front or middle | O(n) | n/a | O(1) average (`del d[k]`) | O(1) average (`remove`) |
| Pop last | O(1) | n/a | O(1) (`popitem()`) | O(1) (`pop()`, arbitrary item) |
| Sort | O(n log n), Timsort, stable | `sorted()` copy | n/a | n/a |
| Copy or iterate | O(n) | O(n) | O(n) | O(n) |

"Average" because many hash collisions can degrade lookups to O(n). Need a queue? `collections.deque` gives O(1) `appendleft` and `popleft`, while `list.pop(0)` is O(n).

When to use which:
- **list** for ordered data you change; **tuple** for fixed records, multiple return values and composite dict keys such as `(lat, lng)`.
- **dict** for lookup by key; **set** for uniqueness and fast membership (turning a "seen" list into a set takes a loop from O(n^2) to O(n)).
- Dict keys and set members must be hashable: `{[1, 2]: "x"}` raises `TypeError: unhashable type: 'list'`. A tuple is hashable only if everything inside it is hashable.

### Going deeper
- A list is a dynamic array of pointers that over-allocates, which is why `append` is amortized O(1).
- A dict is a hash table. Since CPython 3.6 it stores entries in a compact array in insertion order, which became a language guarantee in 3.7.
- `str` and `bytes` hashes are randomized per process (`PYTHONHASHSEED`) to resist hash-flooding attacks, so never persist `hash()` values.

## 7. Mutability, identity and copying

```python
a = [1, 2, 3]
b = [1, 2, 3]
a == b        # True: same value
a is b        # False: two different objects
x = None
x is None     # True: the idiomatic None check
```

Use `is` only for singletons (`None`, `True`, `False`, sentinel objects). CPython caches small integers (-5 to 256) and interns some strings, so `is` can appear to work for them and then fail for other values. That is an implementation detail, and since 3.8 Python warns (`SyntaxWarning`) when you write `x is 5`.

```python
int("256") is int("256")   # True  (cached small int)
int("257") is int("257")   # False (two separate objects)
```

**Shallow vs deep copy**

```python
import copy

grid = [[1, 2], [3, 4]]
shallow = grid.copy()            # same as list(grid), grid[:] or copy.copy(grid)
deep = copy.deepcopy(grid)

grid[0].append(99)
print(shallow)   # [[1, 2, 99], [3, 4]]  the inner lists are shared
print(deep)      # [[1, 2], [3, 4]]      fully independent
```

A shallow copy is a new outer container holding references to the same inner objects. A deep copy recursively copies everything (and handles cycles). Deep copies are slower, so use them only when nested mutable data must be isolated, for example before mutating a config dict you received.

**The mutable default argument trap**

```python
def add_tag(tag, tags=[]):       # the list is created ONCE, when def runs
    tags.append(tag)
    return tags

add_tag("a")   # ['a']
add_tag("b")   # ['a', 'b']  the same list again

def add_tag(tag, tags=None):     # fix: use None as a sentinel and create inside
    if tags is None:
        tags = []
    tags.append(tag)
    return tags
```

The same sharing bug appears in `[[0] * 3] * 3`: the outer `*` repeats references to one inner list, so `grid[0][0] = 1` changes every row. Build grids with `[[0] * 3 for _ in range(3)]`. Django applies the same rule to model fields: `JSONField(default=dict)`, never `default={}`.

## 8. Control flow, enumerate, zip and comprehensions

```python
status = "active" if paid else "trial"          # conditional expression

for i, name in enumerate(["a", "b"], start=1):  # index without range(len(...))
    print(i, name)                              # 1 a, then 2 b

list(zip([1, 2, 3], "ab"))                      # [(1, 'a'), (2, 'b')]: stops at the shortest
# zip(a, b, strict=True) raises ValueError on a length mismatch (3.10+)

for attempt in range(3):
    if try_connect():
        break
else:                                           # runs only if the loop did NOT break
    raise ConnectionError("gave up after 3 attempts")

while (chunk := stream.read(1024)):             # walrus: assign inside an expression (3.8+)
    handle(chunk)
```

```python
squares = [n * n for n in range(5)]                  # [0, 1, 4, 9, 16]
evens = [n for n in range(10) if n % 2 == 0]         # filter
labels = ["even" if n % 2 == 0 else "odd" for n in range(3)]   # transform with if/else
by_id = {u["id"]: u for u in users}                  # dict comprehension
domains = {email.split("@")[1] for email in emails}  # set comprehension
flat = [x for row in matrix for x in row]            # nested: same order as nested for loops
total = sum(n * n for n in range(1_000_000))         # generator expression: lazy, O(1) memory
```

Comprehension variables do not leak into the surrounding scope. A generator expression can be consumed only once. Prefer a plain loop when you need side effects, `try/except`, or several conditions that would make a one-liner hard to read.

## 9. Functions

```python
def create_user(email, /, name, *, role="student", **extra):
    """email is positional-only, name is positional or keyword,
    role is keyword-only, extra collects any other keyword arguments."""
    return {"email": email, "name": name, "role": role, **extra}

create_user("a@x.com", "Ali")                          # role defaults to "student"
create_user("a@x.com", name="Ali", role="admin", team="core")
create_user("a@x.com", "Ali", "admin")                 # TypeError: role is keyword-only
create_user(email="a@x.com", name="Ali")               # TypeError: email must be positional
```

- `*args` collects extra positional arguments into a tuple; `**kwargs` collects extra keyword arguments into a dict. At a call site the same symbols unpack: `f(*items, **options)`.
- A bare `*` makes every later parameter keyword-only, so calls read clearly (`send(user, retry=True)`). A `/` (3.8+) makes earlier parameters positional-only, so you can rename them later without breaking callers.
- Order: positional-only, `/`, normal, `*args` or `*`, keyword-only, `**kwargs`.
- A function without `return` returns `None`. `return a, b` returns a tuple the caller can unpack.

Functions are first-class objects: assign them, store them in dicts, pass them in and return them.

```python
def shout(text):
    return text.upper()

handlers = {"shout": shout, "whisper": lambda text: text.lower()}
handlers["shout"]("hi")                     # 'HI'
sorted(users, key=lambda u: u["age"])       # pass a function as an argument
```

A `lambda` is an anonymous function limited to one expression. Use it for short `key=` callbacks and give anything longer a name with `def` (clearer tracebacks, testable).

## 10. Scope: LEGB, global and nonlocal

Python resolves a name by searching **L**ocal (the current function), **E**nclosing (outer functions), **G**lobal (the module), then **B**uilt-in (`len`, `print`). Scope is decided at compile time: if a function assigns to a name anywhere, that name is local for the whole function.

```python
count = 0

def broken():
    count += 1        # UnboundLocalError: count is local here but not assigned yet

def works():
    global count      # rebind the module-level name
    count += 1

def make_counter():
    total = 0
    def increment():
        nonlocal total    # rebind the enclosing function's name
        total += 1
        return total
    return increment

counter = make_counter()
counter(), counter()      # (1, 2)
```

Reading an outer name needs no declaration; only rebinding does. `global` is a smell in application code (hidden shared state, not thread-safe); pass values in and return them instead. `nonlocal` is normal in closures and decorators. Shadowing built-ins (`list = [...]`, `id = 5`) breaks later calls to them in that scope.

## 11. Exceptions

```python
import json
import logging

logger = logging.getLogger(__name__)

class PaymentError(Exception):
    """Base class for payment failures in this app."""

class CardDeclined(PaymentError):
    def __init__(self, code):
        super().__init__(f"card declined: {code}")
        self.code = code

def load_config(path):
    try:
        with open(path, encoding="utf-8") as f:
            data = json.load(f)
    except FileNotFoundError:
        logger.warning("no config at %s, using defaults", path)
        return {}
    except json.JSONDecodeError as err:
        raise ValueError(f"invalid config in {path}") from err   # original kept as __cause__
    else:
        return data            # runs only if the try block raised nothing
    finally:
        logger.debug("config load attempted")   # always runs, even after a return
```

- Catch the most specific exception you can handle and keep the `try` block small. `except Exception:` belongs at boundaries (a request handler, a job runner) where you log and re-raise or convert.
- A bare `except:` also catches `KeyboardInterrupt` and `SystemExit`, which inherit from `BaseException` rather than `Exception`. Avoid it.
- `raise` alone re-raises the current exception with its traceback. `raise X from err` chains explicitly; `raise X from None` hides the original.
- Custom exceptions subclass `Exception` (or your own base class) so callers can catch a whole family (`except PaymentError`).
- Python 3.11 added exception groups (`ExceptionGroup`, `except*`) for concurrent failures and `add_note()` for extra context.

**EAFP vs LBYL.** "Easier to ask forgiveness than permission" means attempting the operation and handling the failure; "look before you leap" checks first. EAFP is idiomatic and avoids race conditions: a file can disappear between `os.path.exists()` and `open()`.

```python
if "email" in payload:            # LBYL
    email = payload["email"]

try:                              # EAFP
    email = payload["email"]
except KeyError:
    email = None

email = payload.get("email")      # for a simple lookup, better than both
```

## 12. Modules, packages and environments

A **module** is a `.py` file; a **package** is a directory of modules, normally with an `__init__.py` that runs on import. Importing executes a module once and caches it in `sys.modules`; later imports reuse the cached object.

```python
import json                              # json.loads(...)
from pathlib import Path                 # one name
from collections import defaultdict as dd
from .services import charge_card        # relative import inside a package
from ..utils import slugify              # from the parent package
```

```python
# report.py
def main():
    print("generating report")

if __name__ == "__main__":   # True for `python report.py`, False when imported
    main()
```

Running a file directly sets its `__name__` to `"__main__"`; importing it sets `__name__` to the module path (`"report"`). The guard keeps scripts importable and is required for `multiprocessing` on Windows and macOS. Run a module inside a package with `python -m package.module`.

Circular imports (`a` imports `b`, `b` imports `a`) fail with "partially initialized module" errors. Fix them by moving shared code into a third module, importing inside the function that needs it, or, for type hints only, importing under `if TYPE_CHECKING:`.

```bash
python -m venv .venv                 # one isolated environment per project
source .venv/bin/activate            # Windows PowerShell: .venv\Scripts\Activate.ps1
pip install django djangorestframework
pip freeze > requirements.txt        # pin the exact installed versions
pip install -r requirements.txt      # reproduce the environment in CI or Docker
```

| File | Role |
|---|---|
| `requirements.txt` | Flat list of packages, ideally pinned (`Django==5.2.1`). Simple and universal, no metadata. |
| `pyproject.toml` | The standard project file: metadata, dependencies, build backend, and tool config (pytest, ruff, mypy). |
| `poetry.lock`, `uv.lock` | Exact resolved versions of the whole dependency tree for reproducible installs. |

Poetry and uv (a fast installer and project manager written in Rust) manage the virtual environment, resolution and lock file from `pyproject.toml`; pip-tools compiles a `requirements.in` into a pinned `requirements.txt`. Follow whatever the project already uses, and never install packages into the system Python.

## 13. File I/O with `with`

```python
import csv
import json
from pathlib import Path

with open("events.log", "a", encoding="utf-8") as f:   # "r", "w" (truncates), "a", "x" (new only), add "b" for binary
    f.write("started\n")

with open("big.log", encoding="utf-8") as f:
    for line in f:                       # streams line by line; memory stays flat
        if "ERROR" in line:
            print(line.rstrip())

config = json.loads(Path("config.json").read_text(encoding="utf-8"))
Path("out.json").write_text(json.dumps(config, indent=2), encoding="utf-8")

with open("users.csv", newline="", encoding="utf-8") as f:
    for row in csv.DictReader(f):        # each row is a dict keyed by the header
        print(row["email"])
```

`with` calls the file object's `__exit__`, which closes the file even if the block raises. The same protocol powers `transaction.atomic()` in Django (see [context managers](03-advanced-features.md)). Always pass `encoding="utf-8"`: the default depends on the platform and is often not UTF-8 on Windows. `pathlib.Path` is the modern way to build paths, as in Django's `BASE_DIR / "static"`.

## 14. Built-ins you should use fluently

```python
users = [
    {"name": "Sara", "age": 31, "city": "Lahore"},
    {"name": "Ali", "age": 25, "city": "Karachi"},
    {"name": "Hina", "age": 25, "city": "Lahore"},
]

sorted(users, key=lambda u: u["age"])                 # stable: Ali stays before Hina
sorted(users, key=lambda u: (u["city"], -u["age"]))   # city ascending, then age descending
sorted(["b", "A", "c"], key=str.lower)                # ['A', 'b', 'c'] case-insensitive
min(users, key=lambda u: u["age"])["name"]            # 'Ali' (the first of the ties)
max([], default=0)                                    # 0 instead of ValueError
any(u["city"] == "Lahore" for u in users)             # True, stops at the first match
all(u["age"] >= 18 for u in users)                    # True
sum(u["age"] for u in users)                          # 81
list(map(str.upper, ["a", "b"]))                      # ['A', 'B']
list(filter(None, [0, 1, "", "x"]))                   # [1, 'x'] drops falsy values
users.sort(key=lambda u: u["name"], reverse=True)     # in place, returns None
```

`map` and `filter` return lazy iterators; a comprehension is usually clearer. `operator.itemgetter("age")` and `operator.attrgetter("created_at")` are faster alternatives to simple lambdas. Other everyday built-ins: `len`, `isinstance(x, (int, float))`, `reversed`, `range`, `abs`, `divmod`, `print(*items, sep=", ")`.

## Common pitfalls
- Mutable default arguments (`def f(items=[])`) and `[[0] * n] * m` grids: both share one object where you expected many.
- Using `is` to compare numbers or strings; it "works" for small cached values and then fails in production data.
- `value or default` when `0`, `""` or `[]` is a legitimate value. Test `is None` instead.
- Removing items from a list while iterating over it skips elements. Build a new list (`[x for x in items if keep(x)]`) or iterate over a copy.
- `nums = nums.sort()` sets `nums` to `None`, because in-place methods return `None`.
- Money in floats. Use `Decimal` built from strings, or integer cents.
- Shadowing built-ins such as `list`, `dict`, `id`, `type`, `input` or `filter` with variable names.
- `except Exception: pass` hiding real bugs, or a bare `except:` that also traps Ctrl+C.
- Omitting `encoding="utf-8"`: the code works on Linux and breaks on a Windows machine.
- Looping twice over a generator or `map` object; the second loop silently sees nothing.
- Adding or deleting dict keys during iteration (`RuntimeError`), and treating `d.keys()` as a list (it is a live view).
- Installing packages globally or leaving them unpinned, so the project breaks on the next machine or release.

## How to talk about this in an interview
- Frame Python as your second backend language with real project use: you built the LMS backend at PureLogics with Django and DRF, and wrote Python for your final-year project. Your daily languages are Ruby and TypeScript, so make the comparisons explicit: empty collections and `0` are falsy in Python but truthy in Ruby, Python strings are immutable while Ruby strings are mutable, and comprehensions play the role of Ruby's `map` and `select`.
- Answer fundamentals with a definition, a two-line example, then a production consequence ("a mutable default in a helper leaked tags between requests").
- Be honest about depth: internship-level Django plus project scripting. Show that you can read unfamiliar Python quickly and reason about its behavior, which is what most interviews test.
- > **Fill in:** the Python and Django versions you used at PureLogics, and whether dependencies lived in `requirements.txt`, Poetry or uv.
- > **Fill in:** what the Python part of your final-year project did (data preprocessing, fine-tuning scripts, an inference service?) and which libraries it used.
- > **Fill in:** one Python bug you hit and fixed (for example a shared default, a `None` check, or an import cycle) and how you found it.

## Interview questions

**Q1. Is Python compiled or interpreted?**
Both, in CPython: source is compiled to bytecode (cached as `.pyc` files for imported modules), and an interpreter loop executes that bytecode on a stack-based virtual machine. There is no ahead-of-time compilation to machine code, which is one reason pure-Python loops are slower than C. Other implementations differ; PyPy adds a JIT.
*Follow-up:* What is in `__pycache__`? Cached bytecode per module and Python version; it is safe to delete and should be git-ignored.

**Q2. Python is "dynamically and strongly typed". What does that mean?**
Dynamic: names have no declared types; objects do, and type errors appear at runtime. Strong: unrelated types are not silently coerced, so `"1" + 1` raises `TypeError` where JavaScript would return `"11"`. Type hints add optional static checking through mypy or pyright without changing runtime behavior.
*Follow-up:* Are hints checked at runtime? No; only tools and libraries that read annotations (dataclasses, pydantic) use them.

**Q3. What is the difference between a list and a tuple?**
A list is mutable (append, remove, sort in place); a tuple is fixed once created. Tuples are hashable when their contents are, so they can be dict keys or set members; they are slightly smaller and faster, and they signal "fixed record" (coordinates, multiple return values). Lists suit sequences that grow and change.
*Follow-up:* Can a tuple's contents change? Its slots cannot be reassigned, but a mutable object inside it (a list) can still be mutated, and such a tuple is unhashable.

**Q4. Which types are mutable and which are immutable, and why does it matter?**
Immutable: `int`, `float`, `bool`, `str`, `bytes`, `tuple`, `frozenset`, `None`. Mutable: `list`, `dict`, `set`, `bytearray` and most user-defined objects. It matters for aliasing (two names sharing one mutable object), default arguments, hashing (only hashable values can be keys) and thread safety. "Changing" an immutable value creates a new object: `s += "x"` rebinds `s`.
*Follow-up:* Why are strings immutable? Safe hashing and interning, safe sharing between threads, and simpler reasoning.

**Q5. What is the difference between `is` and `==`?**
`==` compares values by calling `__eq__`; `is` compares identity (both names bound to the same object, the same `id()`). Use `is` for `None` and other singletons and `==` for everything else. `is` on ints or strings can appear to work because CPython caches small ints and interns some strings, but that is an implementation detail.
*Follow-up:* Why `x is None` rather than `x == None`? A class can override `__eq__` to return anything; `is` cannot be overridden and is faster.

**Q6. What is the mutable default argument problem?**
Default values are evaluated once, when `def` executes, and stored on the function object. A mutable default (`items=[]`) is therefore shared by every call that omits the argument and accumulates state across calls. Use `None` as the default and create a fresh object inside the function.
*Follow-up:* Is it ever intentional? Occasionally as a quick memo cache, but `functools.lru_cache` states that intent far more clearly.

**Q7. Explain shallow copy vs deep copy.**
A shallow copy (`copy.copy`, `list(x)`, `x[:]`, `d.copy()`) creates a new outer container that references the same inner objects, so mutating a nested list shows up in both. `copy.deepcopy` recursively copies nested objects, using a memo dict to handle cycles. Deep copies cost more time and memory and can fail for objects such as open files or locks.
*Follow-up:* Is `b = a` a copy? No, it binds a second name to the same object.

**Q8. What are `*args` and `**kwargs`, and what do a bare `*` and `/` do in a signature?**
`*args` collects extra positional arguments into a tuple and `**kwargs` collects extra keyword arguments into a dict, which lets wrappers and decorators forward any arguments. A bare `*` makes the following parameters keyword-only; `/` (3.8+) makes the preceding ones positional-only. At a call site, `f(*items, **options)` unpacks a sequence and a mapping.
*Follow-up:* Why keyword-only parameters? Flags become self-documenting (`export(report, include_archived=True)`) and cannot be passed in the wrong position.

**Q9. Is Python pass-by-value or pass-by-reference?**
Neither exactly: it is pass by object reference (call by sharing). The parameter is a new name bound to the object the caller passed. Mutating that object (`items.append(...)`) is visible to the caller; rebinding the parameter (`items = []`) is not.
*Follow-up:* How should a function report several changes? Return new values (often a tuple) instead of mutating its arguments; it is easier to test.

**Q10. Explain LEGB and the difference between `global` and `nonlocal`.**
Names resolve through Local, Enclosing, Global, then Built-in scope. Because any assignment makes a name local for the whole function, `count += 1` on a module variable raises `UnboundLocalError`. `global x` rebinds the module-level name; `nonlocal x` rebinds a name in the nearest enclosing function, which closures such as counters and decorators need.
*Follow-up:* Do you need `global` to read a module variable? No, only to rebind it; mutating a global list needs no declaration either.

**Q11. What is the difference between a list comprehension and a generator expression?**
`[f(x) for x in xs]` builds the whole list in memory immediately. `(f(x) for x in xs)` creates a lazy generator that yields one value at a time, uses constant memory and can be iterated only once. Feed generator expressions straight into consumers such as `sum()`, `any()` or `"".join()`; use a list when you need indexing, `len()` or several passes.
*Follow-up:* What does `any(check(u) for u in users)` gain over a list? It stops at the first `True` without evaluating the rest.

**Q12. How does a Python dict work, and what can be a key?**
It is a hash table: the key's `hash()` picks a slot and `__eq__` resolves collisions, giving O(1) average get, set, delete and membership. Iteration follows insertion order (guaranteed since 3.7). Keys must be hashable, meaning the hash never changes during the object's lifetime, so lists and dicts cannot be keys while tuples of immutable values can.
*Follow-up:* What happens if you add keys while iterating? `RuntimeError: dictionary changed size during iteration`; iterate over `list(d)` instead.

**Q13. Walk me through `try/except/else/finally`.**
`try` holds code that may fail; each `except` handles a specific exception type; `else` runs only when the `try` block raised nothing (follow-up code goes there so its errors are not caught by mistake); `finally` always runs, even after `return` or an unhandled exception, so it holds cleanup. `raise NewError(...) from err` keeps the original exception as the cause in the traceback.
*Follow-up:* What is wrong with `except Exception: pass`? It silently swallows bugs; log the exception at minimum, and catch only what you can handle.

**Q14. What do EAFP and LBYL mean, and which is more Pythonic?**
EAFP (easier to ask forgiveness than permission) attempts the operation and handles the exception; LBYL (look before you leap) checks preconditions first. EAFP is idiomatic because it avoids duplicated checks and time-of-check to time-of-use races, and exceptions are cheap when they are rare. LBYL still fits when failure is common and the check is cheap.
*Follow-up:* An EAFP example from Django? `try: Course.objects.get(pk=pk)` with `except Course.DoesNotExist:`, which `get_object_or_404` wraps for you.

**Q15. What does `if __name__ == "__main__":` do?**
Every module has a `__name__`: it is `"__main__"` when the file runs directly and the import path when the file is imported. The guard lets one file be both an importable module and a script, keeps side effects out of imports and tests, and is required for `multiprocessing` with the spawn start method (the default on Windows and macOS).
*Follow-up:* Other ways to run code as a script? `python -m package.module`, or a console entry point declared in `pyproject.toml`.

**Q16. How do you manage dependencies in a Python project?**
One virtual environment per project (`python -m venv .venv`) so packages do not leak between projects or into the system Python. Pin dependencies in `requirements.txt` (with `pip freeze` or pip-tools) or declare them in `pyproject.toml` with a lock file managed by Poetry or uv. CI and Docker images install exactly the pinned set.
*Follow-up:* Why pin? An unpinned dependency can publish a breaking release and fail a build that passed yesterday.

**Q17. What is the difference between `sorted()` and `list.sort()`?**
`sorted(iterable)` accepts any iterable and returns a new list; `list.sort()` sorts in place and returns `None`, so `nums = nums.sort()` loses the data. Both use Timsort (O(n log n), stable) and accept `key=` and `reverse=`. Stability means you can sort by a secondary key, then by the primary key, or simply use a tuple key.
*Follow-up:* Users by age descending, then name ascending? `sorted(users, key=lambda u: (-u.age, u.name))`.

## Further reading
- [The Python Tutorial](https://docs.python.org/3/tutorial/)
- [Built-in types](https://docs.python.org/3/library/stdtypes.html)
- [Execution model: naming and binding](https://docs.python.org/3/reference/executionmodel.html)
- [Errors and exceptions](https://docs.python.org/3/tutorial/errors.html)
- [Virtual environments and packages](https://docs.python.org/3/tutorial/venv.html)
