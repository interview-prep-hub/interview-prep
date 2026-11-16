# Python OOP

> **Tier:** T1 (know cold) · **Related:** [Python fundamentals](01-fundamentals.md), [Advanced Python features](03-advanced-features.md), [Python rapid-fire questions](99-interview-questions.md), [OOP concepts](../../01-cs-fundamentals/02-oop.md), [SOLID and design principles](../../01-cs-fundamentals/03-solid-and-design-principles.md), [Ruby OOP, modules and mixins](../ruby/03-oop-modules-mixins.md), [Django models and ORM](../../10-python-django/02-models-and-orm.md)
> **On your resume:** Python and Django REST Framework (the PureLogics LMS). Every Django model, form, view, serializer and permission class you wrote is a Python class, and DRF's generic views are assembled from mixins ordered by the MRO.

## Quick revision
- A class is an object that creates instances. `__new__` creates the instance, `__init__` initializes it. `self` is the explicit instance parameter: `obj.method(x)` runs `type(obj).method(obj, x)`.
- Attribute lookup: the instance `__dict__`, then the class, then base classes along the MRO (data descriptors such as properties take priority over the instance dict).
- Class attributes are shared by all instances. A mutable class attribute (`items = []`) is shared state; `self.x = ...` creates an instance attribute that shadows the class one.
- Instance methods get `self`; `@classmethod` gets `cls` (alternative constructors that respect subclasses); `@staticmethod` gets nothing (a function namespaced in the class).
- `@property` turns a method into a computed attribute, with an optional validating setter, without changing the public API.
- `super()` returns a proxy for the **next class in the MRO** of the instance's type, which is not always the direct parent.
- The MRO uses C3 linearization: subclasses before bases, the base order you wrote is kept, each class appears once. Inspect it with `Class.__mro__`.
- ABCs (`abc.ABC` plus `@abstractmethod`) cannot be instantiated until every abstract method is implemented.
- `__repr__` is for developers (unambiguous), `__str__` for users. `print` uses `__str__` and falls back to `__repr__`; containers show their items' `__repr__`.
- Defining `__eq__` without `__hash__` makes instances unhashable. Equal objects must have equal hashes, so hash only immutable fields.
- `functools.total_ordering` derives the remaining comparisons from `__eq__` plus one of `__lt__`, `__le__`, `__gt__`, `__ge__`.
- Protocols by dunder: `__len__`, `__getitem__`, `__iter__`, `__contains__` (containers), `__enter__`/`__exit__` (`with`), `__call__` (callable instances).
- Binary dunders return `NotImplemented` (not `False`) for unsupported operand types.
- `_name` means internal by convention; `__name` is name-mangled to `_Class__name` to avoid subclass clashes. Neither is security.
- Duck typing: behavior matters, not declared type. `typing.Protocol` adds static, structural checking of it.
- `@dataclass` generates `__init__`, `__repr__` and `__eq__`. Use `field(default_factory=list)` for mutable defaults, `frozen=True` for immutability, `slots=True` (3.10+) to save memory.
- Prefer composition over deep inheritance; use small mixins for reusable, orthogonal behavior.
- `__slots__` removes the per-instance `__dict__`: less memory, slightly faster access, no ad hoc attributes.
- Classes are instances of a metaclass (`type` by default). Django's `ModelBase` metaclass turns field declarations into `_meta`, managers and per-model exceptions; the inner `class Meta` is only an options holder.

## 1. Classes, `__init__` and `self`

```python
class Course:
    platform = "LMS"                      # class attribute: shared by every instance

    def __init__(self, title, price_cents=0):
        self.title = title                # instance attributes live in self.__dict__
        self.price_cents = price_cents
        self.lessons = []                 # mutable state belongs here, per instance

    def add_lesson(self, title):          # instance method: self is the instance
        self.lessons.append(title)
        return self                       # returning self allows chaining

    def __repr__(self):
        return f"Course(title={self.title!r}, price_cents={self.price_cents})"

c = Course("Django 101", 4999)
c.add_lesson("Models").add_lesson("Views")
Course.add_lesson(c, "Admin")             # exactly what c.add_lesson("Admin") does
print(c)                                  # Course(title='Django 101', price_cents=4999)
print(c.lessons)                          # ['Models', 'Views', 'Admin']
```

- `self` is only a naming convention, but always use it. Forgetting it in a method definition gives `TypeError: add_lesson() takes 1 positional argument but 2 were given`.
- `__init__` does not create the object; `__new__` does, and you override it only for rare cases (subclassing immutable types such as `str`, or caching instances). `__init__` must return `None`.
- There are no access modifiers: every attribute is public, and conventions signal intent (section 9).

## 2. Instance attributes vs class attributes

```python
class Cart:
    items = []                 # BUG: one list shared by every Cart
    discount = 0               # immutable class attribute: fine as a default

    def add(self, item):
        self.items.append(item)     # finds the class list and mutates it

a, b = Cart(), Cart()
a.add("book")
print(b.items)                 # ['book']: b sees a's item

a.discount = 10                # assignment creates an instance attribute on a only
print(a.discount, b.discount, Cart.discount)   # 10 0 0
```

Reading `self.items` searches the instance first, then the class, so all instances find the same list; *assigning* `self.discount` always writes to the instance. The fix is to create mutable state in `__init__` (`self.items = []`). Class attributes are right for constants, defaults and declarative configuration that is never mutated: Django model fields, DRF's `queryset`, `serializer_class` and `permission_classes` are all class attributes read by the framework.

## 3. Instance methods, `@classmethod` and `@staticmethod`

```python
from datetime import date

class Enrollment:
    def __init__(self, student, course, started_on):
        self.student = student
        self.course = course
        self.started_on = started_on

    def days_active(self, today):             # instance method: needs the instance
        return (today - self.started_on).days

    @classmethod
    def from_dict(cls, data):                 # alternative constructor; cls is the class it was called on
        return cls(data["student"], data["course"], date.fromisoformat(data["started_on"]))

    @staticmethod
    def is_valid_code(code):                  # needs neither self nor cls
        return code.isalnum() and len(code) == 8

e = Enrollment.from_dict({"student": "sara", "course": "drf", "started_on": "2025-07-01"})
e.days_active(date(2025, 7, 31))              # 30
Enrollment.is_valid_code("ABCD1234")          # True
```

| | Instance method | `@classmethod` | `@staticmethod` |
|---|---|---|---|
| First argument | `self` (the instance) | `cls` (the class) | none |
| Reads or writes instance state | yes | no | no |
| Creates instances | via `type(self)(...)` | yes, and subclass-aware | only by hardcoding the class name |
| Typical use | behavior of one object | alternative constructors (`from_dict`, `dict.fromkeys`, `datetime.fromisoformat`), factories | helpers that belong in the class namespace |

Why `cls` matters: if `class TrialEnrollment(Enrollment)` calls `TrialEnrollment.from_dict(...)`, it gets a `TrialEnrollment`, because `cls` is the subclass. A static method that hardcodes `Enrollment(...)` would silently return the parent type.

## 4. `@property`: computed and validated attributes

```python
class Subscription:
    def __init__(self, price_cents):
        self.price_cents = price_cents          # goes through the setter below

    @property
    def price_cents(self):
        return self._price_cents

    @price_cents.setter
    def price_cents(self, value):
        if value < 0:
            raise ValueError("price cannot be negative")
        self._price_cents = value

    @property
    def price_display(self):                    # read-only computed attribute
        return f"${self._price_cents / 100:.2f}"

s = Subscription(1999)
s.price_display          # '$19.99' (accessed without parentheses)
s.price_cents = -5       # ValueError
s.price_display = "x"    # AttributeError: the property has no setter
```

Start with plain public attributes; if you later need validation or computation, switch to a property without changing a single caller (no Java-style getters up front). Keep properties cheap: a property that runs a database query looks like a field access and quietly creates N+1 queries when a Django template or serializer loops over objects. `functools.cached_property` computes a value once per instance and stores it.

## 5. Inheritance and `super()`

```python
class Notification:
    def __init__(self, recipient):
        self.recipient = recipient

    def render(self):
        return f"Hello {self.recipient}"

    def send(self):
        print(self.render())           # subclasses customize render(), send() stays the same

class EnrollmentNotification(Notification):
    def __init__(self, recipient, course):
        super().__init__(recipient)    # let the parent initialize its part
        self.course = course

    def render(self):                  # override and extend
        return f"{super().render()}, you are enrolled in {self.course}"

EnrollmentNotification("Sara", "DRF Basics").send()
# Hello Sara, you are enrolled in DRF Basics
isinstance(EnrollmentNotification("a", "b"), Notification)   # True
issubclass(EnrollmentNotification, Notification)             # True
```

- Every class ultimately inherits from `object`. Forgetting `super().__init__(...)` leaves the parent's attributes unset, and the `AttributeError` appears far from the cause.
- Python has no overloading by signature: a second `def render` replaces the first. Use default arguments, keyword-only parameters or `functools.singledispatchmethod`.
- Prefer `isinstance(x, Base)` over `type(x) is Base`, because it respects subclasses.

## 6. Multiple inheritance and the MRO

```python
class A:
    def hello(self):
        return "A"

class B(A):
    def hello(self):
        return "B -> " + super().hello()

class C(A):
    def hello(self):
        return "C -> " + super().hello()

class D(B, C):
    def hello(self):
        return "D -> " + super().hello()

print(D().hello())                         # D -> B -> C -> A
print([k.__name__ for k in D.__mro__])     # ['D', 'B', 'C', 'A', 'object']
```

This is the **diamond**: `B` and `C` both inherit from `A`. Python computes one linear **method resolution order** with the C3 algorithm, which guarantees that a class appears before its bases, that the left-to-right order of bases in each class statement is kept, and that every class appears exactly once. So `A.hello` runs once, after both `B` and `C`.

The key insight: `super()` inside `B` does not mean "B's parent". It means "the class after `B` in the MRO of the object's actual type". For a `D` instance that is `C`, B's sibling. This makes **cooperative multiple inheritance** work, provided every class in the chain calls `super()` and passes along arguments it does not use (usually via `**kwargs`).

If no consistent order exists, the class statement itself fails:

```python
class X(A, B): ...
class Y(B, A): ...
class Z(X, Y): ...   # TypeError: Cannot create a consistent method resolution order (MRO)
```

**Mixins in practice.** A mixin is a small class that adds one behavior and expects to be combined with others. Django's class-based views rely on this: `class CourseUpdateView(LoginRequiredMixin, UserPassesTestMixin, UpdateView)` puts the mixins on the left so their `dispatch()` runs first and then calls `super().dispatch()`. DRF's `ModelViewSet` is literally `CreateModelMixin`, `RetrieveModelMixin`, `UpdateModelMixin`, `DestroyModelMixin`, `ListModelMixin` and `GenericViewSet` combined. Ruby solves the same problem with single inheritance plus modules; Python allows true multiple inheritance and orders it with the MRO.

## 7. Abstract base classes

```python
from abc import ABC, abstractmethod

class PaymentGateway(ABC):
    @abstractmethod
    def charge(self, customer_id: str, amount_cents: int) -> str:
        """Charge the customer and return a transaction id."""

    @abstractmethod
    def refund(self, charge_id: str) -> None: ...

    def refund_all(self, charge_ids):           # concrete methods are allowed
        for charge_id in charge_ids:
            self.refund(charge_id)

class StripeGateway(PaymentGateway):
    def charge(self, customer_id, amount_cents):
        return "ch_123"

PaymentGateway()    # TypeError: can't instantiate an abstract class
StripeGateway()     # TypeError as well: refund is still abstract
```

An ABC defines an interface and enforces it when you instantiate, not when you define the subclass. `collections.abc` provides ready-made ABCs with free mixin methods: subclass `collections.abc.Mapping`, implement `__getitem__`, `__iter__` and `__len__`, and you get `keys()`, `items()`, `get()` and `in` for free. You can also test capabilities with `isinstance(obj, collections.abc.Iterable)`.

## 8. Dunder (magic) methods

Dunder methods ("double underscore") are how your objects plug into Python syntax and built-ins.

```python
from functools import total_ordering

@total_ordering
class Money:                            # treat instances as immutable: the hash depends on the fields
    def __init__(self, cents, currency="USD"):
        self.cents = cents
        self.currency = currency

    def __repr__(self):                 # developers: unambiguous, ideally looks like code
        return f"Money({self.cents!r}, {self.currency!r})"

    def __str__(self):                  # users: print(), str(), f-strings
        return f"{self.cents / 100:.2f} {self.currency}"

    def __eq__(self, other):
        if not isinstance(other, Money):
            return NotImplemented       # let Python try the other operand instead
        return (self.cents, self.currency) == (other.cents, other.currency)

    def __hash__(self):                 # equal objects must have equal hashes
        return hash((self.cents, self.currency))

    def __lt__(self, other):
        if not isinstance(other, Money) or other.currency != self.currency:
            return NotImplemented
        return self.cents < other.cents

    def __add__(self, other):
        if not isinstance(other, Money) or other.currency != self.currency:
            return NotImplemented
        return Money(self.cents + other.cents, self.currency)

    def __bool__(self):
        return self.cents != 0

a, b = Money(500), Money(250)
print(a + b)             # 7.50 USD
print([a, b])            # [Money(500, 'USD'), Money(250, 'USD')]: containers use repr
sorted([a, b])           # [Money(250, 'USD'), Money(500, 'USD')]
a >= b                   # True: __ge__ was generated by total_ordering
len({Money(1), Money(1)})   # 1: equal and same hash
bool(Money(0))           # False
```

Why `NotImplemented` and not `False`: for `a == x`, Python then tries `x.__eq__(a)`, and only if both return `NotImplemented` does it fall back to identity. For `a + 5` it tries `int.__radd__` and finally raises a clear `TypeError`. Returning `False` or raising yourself breaks that negotiation.

**Container protocol**

```python
class Playlist:
    def __init__(self, lessons):
        self._lessons = list(lessons)

    def __len__(self):
        return len(self._lessons)

    def __getitem__(self, index):     # p[0], p[-1], slicing
        return self._lessons[index]

    def __iter__(self):               # for lesson in p
        return iter(self._lessons)

    def __contains__(self, item):     # "views" in p
        return item in self._lessons

p = Playlist(["intro", "models", "views"])
len(p), p[0], p[-1], "models" in p    # (3, 'intro', 'views', True)
```

**Context managers and callables**

```python
import time

class Timer:
    def __enter__(self):
        self.start = time.perf_counter()
        return self                      # bound to the name after "as"

    def __exit__(self, exc_type, exc, tb):
        self.elapsed = time.perf_counter() - self.start
        return False                     # False (or None) lets exceptions propagate

with Timer() as t:
    sum(range(1_000_000))
print(f"{t.elapsed:.3f}s")

class RateLimiter:
    def __init__(self, limit):
        self.limit = limit
        self.calls = 0

    def __call__(self):                  # instances become callable like functions
        self.calls += 1
        return self.calls <= self.limit

allow = RateLimiter(2)
allow(), allow(), allow()                # (True, True, False)
```

| Method | Triggered by |
|---|---|
| `__new__`, `__init__` | `Class(...)` |
| `__repr__`, `__str__`, `__format__` | `repr()`, `str()` and `print()`, f-string format specs |
| `__eq__`, `__lt__` (and friends), `__hash__` | `==`, `<`, `hash()`, dict keys and set members |
| `__len__`, `__bool__` | `len()`, truthiness |
| `__getitem__`, `__setitem__`, `__delitem__`, `__contains__` | `obj[k]`, `obj[k] = v`, `del obj[k]`, `in` |
| `__iter__`, `__next__` | `for`, `iter()`, `next()` |
| `__enter__`, `__exit__` | `with` |
| `__call__` | `obj(...)` |
| `__add__`, `__radd__`, `__iadd__` | `a + b`, `b + a` when `b` does not handle it, `a += b` |
| `__getattr__` | attribute access that failed the normal lookup |

## 9. Encapsulation by convention

```python
class Account:
    def __init__(self, owner):
        self.owner = owner          # public
        self._balance = 0           # internal by convention; nothing stops access
        self.__pin = "1234"         # name-mangled to self._Account__pin

acct = Account("Ali")
acct._balance                       # 0 (works, but reviewers and linters will object)
acct.__pin                          # AttributeError
acct._Account__pin                  # '1234': mangling is not security
```

Python has no `private` keyword; the culture is "we are all consenting adults". A single leading underscore marks internal API (and `from module import *` skips such names). Two leading underscores trigger **name mangling** so that a subclass defining its own `__pin` does not overwrite the parent's; it prevents accidents, not access. Names with double underscores on both sides (`__init__`) are reserved for Python's protocols, so never invent new ones. Real encapsulation comes from design: expose methods and read-only properties rather than letting callers mutate internals.

## 10. Duck typing and `typing.Protocol`

"If it walks like a duck and quacks like a duck, it is a duck": Python code relies on what an object can do, not what it inherits from. Anything iterable works in a `for` loop; anything with a `read()` method can stand in for a file.

```python
from typing import Protocol

class Notifier(Protocol):
    def send(self, to: str, message: str) -> None: ...

class EmailNotifier:                     # does not inherit from Notifier
    def send(self, to: str, message: str) -> None:
        print(f"email to {to}: {message}")

class SmsNotifier:
    def send(self, to: str, message: str) -> None:
        print(f"sms to {to}: {message}")

def notify_all(notifiers: list[Notifier], to: str, message: str) -> None:
    for n in notifiers:
        n.send(to, message)

notify_all([EmailNotifier(), SmsNotifier()], "sara@example.com", "Course published")
```

`Protocol` (3.8+) lets mypy or pyright verify the structural match statically, much like a TypeScript interface. Compare the three approaches:

| Approach | Checked | Requires inheritance | Typical use |
|---|---|---|---|
| Plain duck typing | at runtime, when the method is called | no | most everyday Python |
| `typing.Protocol` | statically, by a type checker | no (structural) | typed interfaces for collaborators and test fakes |
| `abc.ABC` | at instantiation time | yes (nominal) | framework base classes that must enforce an interface |

`@runtime_checkable` allows `isinstance(obj, Notifier)`, but it only checks that the methods exist, not their signatures.

## 11. Dataclasses

```python
from dataclasses import asdict, dataclass, field

@dataclass
class LessonDTO:
    title: str
    duration_seconds: int = 0
    tags: list[str] = field(default_factory=list)   # "tags: list[str] = []" raises ValueError

@dataclass(frozen=True, slots=True)                 # slots=True needs 3.10+
class Price:
    amount_cents: int
    currency: str = "USD"

    def __post_init__(self):                        # runs after the generated __init__
        if self.amount_cents < 0:
            raise ValueError("negative price")

lesson = LessonDTO("Intro", 300)
print(lesson)                        # LessonDTO(title='Intro', duration_seconds=300, tags=[])
lesson == LessonDTO("Intro", 300)    # True: generated field-by-field __eq__
asdict(lesson)                       # {'title': 'Intro', 'duration_seconds': 300, 'tags': []}

p = Price(1999)
p.amount_cents = 0                   # dataclasses.FrozenInstanceError
prices = {p: "basic"}                # frozen plus eq makes instances hashable
```

`@dataclass` writes `__init__`, `__repr__` and `__eq__` from the annotated fields; `order=True` adds comparisons and `kw_only=True` (3.10+) forces keyword arguments. A normal (non-frozen) dataclass sets `__hash__` to `None`, so it is unhashable. Use dataclasses for DTOs, config objects and value objects. They do not validate types at runtime; pydantic (third party) does, and in a DRF codebase serializers already play that validation role at the API boundary.

## 12. Composition over inheritance

```python
class PaymentReceiptService:
    def __init__(self, sender, pdf_renderer):     # has-a: collaborators are injected
        self.sender = sender
        self.pdf_renderer = pdf_renderer

    def deliver(self, payment):
        attachment = self.pdf_renderer.render(payment)
        self.sender.send(payment.email, "Your receipt", attachment)

service = PaymentReceiptService(EmailSender(), ReceiptPdf())     # production
test_service = PaymentReceiptService(FakeSender(), ReceiptPdf()) # tests swap one part
```

Inheritance models "is-a" and couples a subclass to its parent's internals; a change in the base class can break every child (the fragile base class problem), and deep hierarchies are hard to follow. Composition models "has-a": objects collaborate through small interfaces, and you can swap a collaborator without touching the class. Use inheritance for genuine specialization and framework extension points (`models.Model`, `View`, DRF's `BasePermission`), small mixins for orthogonal behavior, and composition for everything else.

## 13. `__slots__`

```python
class Point:
    __slots__ = ("x", "y")

    def __init__(self, x, y):
        self.x, self.y = x, y

p = Point(1, 2)
p.z = 3        # AttributeError: no __dict__ to store new attributes in
```

Slots replace the per-instance `__dict__` with fixed storage. Benefits: much less memory when you create millions of small objects, slightly faster attribute access, and typos fail loudly. Costs: no dynamic attributes, add `"__weakref__"` to the slots if you need weak references, every subclass must declare `__slots__` too (or it gets a `__dict__` back), and multiple bases with non-empty slots conflict. `@dataclass(slots=True)` generates them for you.

## 14. Metaclasses, briefly

Classes are objects too, and the class of a class is its **metaclass**, `type` by default. A metaclass runs code when a class statement executes.

```python
type(int)                                        # <class 'type'>
Dynamic = type("Dynamic", (), {"greet": lambda self: "hi"})   # build a class at runtime
Dynamic().greet()                                # 'hi'

class Plugin:
    registry = {}

    def __init_subclass__(cls, **kwargs):        # 3.6+: the simple alternative to a metaclass
        super().__init_subclass__(**kwargs)
        Plugin.registry[cls.__name__] = cls

class VideoPlugin(Plugin): ...
list(Plugin.registry)                            # ['VideoPlugin']
```

How Django uses one: `models.Model` is built by the `ModelBase` metaclass. When `class Course(models.Model):` runs, `ModelBase.__new__` collects the `Field` attributes, builds `Course._meta` (fields, table name, ordering, constraints), adds a default `objects` manager if you declared none, creates `Course.DoesNotExist` and `Course.MultipleObjectsReturned`, and registers the model with the app registry. The nested `class Meta:` is not a metaclass; `ModelBase` reads it as configuration. DRF serializers collect declared fields the same way. In application code you rarely need a metaclass: try a decorator, `__init_subclass__` or a descriptor first.

## Common pitfalls
- Mutable class attributes used as per-instance state (`items = []` at class level).
- Forgetting `self` in a method signature, or forgetting `super().__init__()` in a subclass.
- Defining `__eq__` without `__hash__`, then using instances as dict keys or set members (`TypeError: unhashable type`).
- Hashing on mutable fields and then mutating an object that is already inside a set or used as a dict key.
- Returning `False` instead of `NotImplemented` from `__eq__` or `__add__` for unknown types.
- Expensive `__str__` or properties: the Django admin calls `__str__` for every row, so a related lookup inside it causes N+1 queries.
- Putting Django mixins to the right of the base view (`class V(UpdateView, LoginRequiredMixin)`), so their checks never run first.
- A `@staticmethod` that hardcodes the class name where a `@classmethod` was needed, breaking subclasses.
- Deep inheritance hierarchies, or classes where a plain function or module would do.
- Treating `__x` as privacy or security.

## How to talk about this in an interview
- Django is OOP-heavy, so tie answers to it: models inherit from `models.Model` (often through an abstract `TimeStampedModel`), class-based views stack mixins such as `LoginRequiredMixin`, and DRF permissions subclass `BasePermission` with `has_permission` and `has_object_permission`, which is how role-based dashboards for students, instructors and admins are usually enforced.
- Contrast with Ruby, which you use daily: Ruby has single inheritance plus modules, Python has real multiple inheritance ordered by the MRO; Ruby's `to_s` and `inspect` map to `__str__` and `__repr__`; Ruby's `Comparable` maps to `functools.total_ordering`.
- Your PUCIT teaching assistant role covered OOP labs, so you can explain the four pillars, then show how Python expresses each one by convention rather than keywords.
- > **Fill in:** an abstract model, mixin or permission class you wrote for the LMS, and why you chose inheritance or composition there.
- > **Fill in:** whether you used dataclasses or service classes for business logic at PureLogics, or kept logic in models and serializers.

## Interview questions

**Q1. What is `self`, and why is it explicit in Python?**
`self` is the instance a method was called on. `obj.method(x)` is sugar for `type(obj).method(obj, x)`, so the instance arrives as the first argument. Making it explicit keeps the rule "methods are plain functions stored on the class" simple and makes instance access obvious (`self.total` vs a local `total`).
*Follow-up:* Is `self` a keyword? No, it is a convention; any name works, but always use `self`.

**Q2. What is the difference between instance and class attributes, and what bug do class-level lists cause?**
Instance attributes live in each object's `__dict__`; class attributes live on the class and are shared. Reading `self.items` falls back to the class, so `items = []` at class level means every instance appends to one shared list. Assignment (`self.items = []`) always creates an instance attribute, which is the fix when done in `__init__`.
*Follow-up:* Why is `discount = 0` at class level safe? Integers are immutable; `self.discount = 10` creates an instance attribute instead of changing the shared value.

**Q3. `@staticmethod` vs `@classmethod`: when do you use each?**
A class method receives the class (`cls`), so it suits alternative constructors and factories that must return the right subclass (`User.from_dict(data)`). A static method receives nothing; it is a plain function grouped in the class for organization. Use an instance method whenever the behavior depends on one object's state.
*Follow-up:* A built-in example of a classmethod constructor? `dict.fromkeys()`, `datetime.fromisoformat()` and `int.from_bytes()`.

**Q4. What does `@property` give you, and when would you avoid it?**
It exposes a method as an attribute, so you can add validation (a setter) or computation later without changing callers. Avoid it for expensive work such as database queries or network calls, because attribute access hides the cost and invites N+1 queries in loops. `cached_property` helps when the value is expensive but stable for the object's lifetime.
*Follow-up:* How do you make an attribute read-only? Define only the getter; assigning then raises `AttributeError`.

**Q5. What does `super()` do? Does it always call the parent class?**
It returns a proxy that resolves methods starting from the class after the current one in the MRO of the instance's actual type. With single inheritance that is the parent; with multiple inheritance it can be a sibling class. That is what allows cooperative mixins, as long as every class calls `super()` and forwards arguments it does not use.
*Follow-up:* What breaks if one class in the chain skips `super()`? Every class after it in the MRO is silently skipped.

**Q6. What is the MRO, and how does Python resolve the diamond problem?**
The method resolution order is the linear list of classes searched for an attribute, computed with C3 linearization: subclasses before bases, base order preserved, each class once. In the diamond `D(B, C)` with `B(A)` and `C(A)`, the MRO is `D, B, C, A, object`, so `A` runs once. Check it with `D.__mro__` or `D.mro()`; inconsistent hierarchies raise `TypeError` at class creation.
*Follow-up:* Where do you see this in Django? Class-based views with mixins (`LoginRequiredMixin` first) and DRF's `ModelViewSet` built from five mixins plus `GenericViewSet`.

**Q7. What is the difference between an ABC, a Protocol and plain duck typing?**
Duck typing just calls the method and fails at runtime if it is missing. An ABC is nominal: subclasses must inherit from it, and instantiation fails until every abstract method is implemented. A `Protocol` is structural: any class with matching methods satisfies it, checked statically by mypy or pyright with no inheritance needed.
*Follow-up:* Which suits test doubles? Protocols, because a fake only needs the right methods.

**Q8. What is the difference between `__str__` and `__repr__`?**
`__repr__` is the developer-facing representation used by the REPL, debuggers, logs and containers; it should be unambiguous and ideally look like the constructor call. `__str__` is the user-facing text used by `print()`, `str()` and f-strings, and it falls back to `__repr__` if missing. Implement `__repr__` first. Django models use `__str__` for the admin and shell display.
*Follow-up:* What does `print([obj])` use? The items' `__repr__`, even if `__str__` exists.

**Q9. What happens to `__hash__` when you define `__eq__`, and why?**
Python sets `__hash__` to `None`, making instances unhashable, because the default identity hash would break the rule that equal objects must have equal hashes. Define `__hash__` from the same fields `__eq__` compares, and only from fields that never change while the object is in a set or used as a key.
*Follow-up:* How do dataclasses handle it? `eq=True` with `frozen=False` sets `__hash__` to `None`; `frozen=True` generates a field-based hash.

**Q10. How do you make your own class work with `len()`, indexing, `in` and `for`?**
Implement `__len__`, `__getitem__`, `__contains__` and `__iter__`. `__iter__` should return an iterator (often `iter(self._items)`, or a generator via `yield`). Without `__iter__`, Python falls back to calling `__getitem__` with 0, 1, 2... until `IndexError`, and without `__contains__` the `in` operator iterates.
*Follow-up:* Shortcut for mapping-like classes? Subclass `collections.abc.Mapping` and implement three methods to get the rest.

**Q11. What is `NotImplemented`, and why return it from `__eq__` or `__add__`?**
It is a special singleton that tells Python "this operand type is not supported here, try the other side". Python then calls the reflected method on the other operand (`__radd__`, the other object's `__eq__`) and only then falls back to identity comparison or raises `TypeError`. Returning `False` or raising directly would short-circuit that negotiation.
*Follow-up:* Is it the same as `NotImplementedError`? No, that is an exception for abstract or unfinished methods.

**Q12. What do `_x`, `__x` and `__x__` mean?**
`_x` is a convention for internal use, with no enforcement. `__x` inside a class is name-mangled to `_ClassName__x`, which prevents accidental clashes with subclass attributes but does not hide anything. `__x__` names are reserved for Python's protocols (`__init__`, `__len__`); define the existing ones and never invent new ones.
*Follow-up:* How do you get true privacy? You cannot; you rely on conventions, code review and a small public API.

**Q13. What does `@dataclass` generate, and what are `frozen` and `default_factory` for?**
It generates `__init__`, `__repr__` and `__eq__` from annotated fields, plus ordering methods with `order=True`. `frozen=True` blocks assignment after creation (raising `FrozenInstanceError`) and, together with `eq`, makes instances hashable. `field(default_factory=list)` gives each instance a fresh mutable default; a bare `[]` default is rejected with `ValueError` for exactly the mutable-default reason.
*Follow-up:* Dataclass vs pydantic model? Dataclasses do not validate at runtime; pydantic parses and validates input.

**Q14. What are `__slots__`, and when would you use them?**
`__slots__` declares a fixed set of attributes and removes the per-instance `__dict__`. That saves significant memory when you create very many small objects and makes attribute access slightly faster, but you can no longer add attributes dynamically and inheritance needs care. Use it for hot, numerous objects, not for ordinary application classes.
*Follow-up:* How do you get slots on a dataclass? `@dataclass(slots=True)` on Python 3.10+.

**Q15. What is a metaclass, and how does Django use one?**
A metaclass is the class of a class: it controls how class objects are created, and `type` is the default. Django's `ModelBase` metaclass processes a model's class body when the class is defined: it collects fields, builds `_meta`, adds the default manager and the `DoesNotExist` exception, and registers the model. Most code should prefer decorators or `__init_subclass__`, which are simpler.
*Follow-up:* Is the inner `class Meta` a metaclass? No, it is a plain options holder that `ModelBase` reads.

**Q16. `__new__` vs `__init__`?**
`__new__` is a static method that creates and returns the new instance; `__init__` receives that instance and initializes it, returning `None`. You override `__new__` only to control creation itself: subclassing immutable types (`str`, `tuple`), caching or reusing instances, or in metaclasses.
*Follow-up:* Is `__init__` called if `__new__` returns an instance of another class? No, `__init__` runs only when `__new__` returns an instance of the class being created.

## Further reading
- [Classes tutorial](https://docs.python.org/3/tutorial/classes.html)
- [Data model (special methods)](https://docs.python.org/3/reference/datamodel.html)
- [dataclasses](https://docs.python.org/3/library/dataclasses.html)
- [The Python 2.3 MRO (C3 explained)](https://docs.python.org/3/howto/mro.html)
