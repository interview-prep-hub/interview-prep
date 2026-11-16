# Django ORM Performance

> **Tier:** T1 (know cold) · **Related:** [Models and ORM](02-models-and-orm.md), [Django REST Framework](04-django-rest-framework.md), [Advanced Django](06-django-advanced.md), [Django rapid-fire questions](99-interview-questions.md), [Indexing and query optimization](../12-databases/05-indexing-and-query-optimization.md), [Transactions and concurrency](../12-databases/06-transactions-and-concurrency.md), [PostgreSQL](../12-databases/04-postgresql.md), [ActiveRecord N+1](../05-ruby-on-rails/05-activerecord-querying-n-plus-1.md)
> **On your resume:** The role-based dashboards in the PureLogics LMS (instructor analytics, student progress) are aggregation-heavy, and DRF list endpoints are exactly where N+1 queries appear. The same instincts carry over to the Rails and PostgreSQL work you do at Devntech.

## Quick revision
- **N+1**: one query for a list plus one per row for a relation. Detect it with django-debug-toolbar, SQL logging, `connection.queries` or `assertNumQueries` in tests.
- `select_related` adds SQL JOINs for single-valued relations (forward FK, one-to-one). `prefetch_related` runs one extra `WHERE id IN (...)` query per relation and stitches results in Python, for multi-valued relations (reverse FK, M2M).
- `Prefetch("reviews", queryset=..., to_attr=...)` customizes the prefetched query (filter, order, nested `select_related`).
- Calling `.filter()` or `.order_by()` on a prefetched relation ignores the prefetch cache and queries again.
- `only()`/`defer()` limit columns, but touching a deferred field costs one query per object. `values()`/`values_list()` skip model instances entirely.
- `annotate()` adds a computed column per row; `aggregate()` returns one dict for the whole QuerySet. Aggregating over two multi-valued relations multiplies rows: use `distinct=True` or subqueries.
- `F("field")` refers to a column inside the database: `update(count=F("count") + 1)` is atomic and avoids lost updates.
- `Q()` objects build OR, NOT and dynamic conditions: `Q(a) | Q(b)`, `~Q(c)`.
- `Subquery` with `OuterRef` for correlated subqueries, `Exists` for cheap yes/no checks, `Case`/`When` for conditional values, `Count(..., filter=Q(...))` for conditional aggregation.
- Django runs in autocommit mode. `transaction.atomic()` makes a block all-or-nothing; a nested `atomic()` is a savepoint.
- `transaction.on_commit(fn)` runs side effects (emails, Celery tasks) only after the transaction commits, and never if it rolls back.
- `select_for_update()` locks the selected rows until the transaction ends and must run inside `atomic()`; `skip_locked=True` builds work queues, `nowait=True` fails fast.
- `bulk_create`/`bulk_update` with `batch_size` for many writes; `iterator(chunk_size=...)` streams large result sets without caching them.
- Raw SQL: `Model.objects.raw(sql, params)` or `cursor.execute(sql, params)`. Always pass parameters separately, never f-strings.
- `QuerySet.explain()` (with `analyze=True` on PostgreSQL) shows the plan; add `Meta.indexes` for your real filter and sort patterns.
- `CONN_MAX_AGE` reuses connections across requests, PgBouncer pools across processes, and Django 5.1 added optional native pooling for PostgreSQL with psycopg 3 (check the docs for your version).

## 1. Finding the problem: N+1 in Django

```python
# courses/views.py: looks innocent
def course_list(request):
    courses = Course.objects.filter(status="published")[:20]
    return render(request, "courses/list.html", {"courses": courses})
```

```html
{% for course in courses %}
  {{ course.title }} by {{ course.instructor.username }} ({{ course.reviews.count }} reviews)
{% endfor %}
```

```text
SELECT ... FROM courses_course WHERE status = 'published' ORDER BY created_at DESC LIMIT 20
SELECT ... FROM accounts_user WHERE id = 7
SELECT COUNT(*) FROM courses_review WHERE course_id = 101
SELECT ... FROM accounts_user WHERE id = 7          (the same instructor, fetched again)
SELECT COUNT(*) FROM courses_review WHERE course_id = 102
... 41 queries for 20 rows
```

Each `course.instructor` access lazily loads one user, and each `course.reviews.count` runs a `COUNT`. The fix moves both into the main query:

```python
from django.db.models import Count

courses = (
    Course.objects.filter(status="published")
    .select_related("instructor")                   # JOIN: instructor arrives with the course
    .annotate(review_count=Count("reviews"))        # COUNT computed in the same query
)[:20]
# template: {{ course.review_count }} instead of {{ course.reviews.count }}
```

**How to see queries:**

```python
from django.db import connection, reset_queries

reset_queries()
list(Course.objects.select_related("instructor")[:20])
print(len(connection.queries))                         # populated only when DEBUG = True
print(Course.objects.filter(status="published").query) # the SQL Django will send
```

```python
# settings.py (development): print every SQL statement (only works with DEBUG = True)
LOGGING = {
    "version": 1,
    "handlers": {"console": {"class": "logging.StreamHandler"}},
    "loggers": {"django.db.backends": {"handlers": ["console"], "level": "DEBUG"}},
}
```

- **django-debug-toolbar**: add it to `INSTALLED_APPS`, `MIDDLEWARE` and the URLconf, set `INTERNAL_IPS`, and its SQL panel lists every query with timings, highlights similar and duplicated queries, and shows the line of code that triggered each one. It also renders on DRF's browsable API pages.
- **Tests that lock it in**: `with self.assertNumQueries(3): self.client.get(url)` in Django's `TestCase`, or the `django_assert_num_queries` fixture in pytest-django. Seed several rows first, so an N+1 would show up as a larger count.

## 2. `select_related` vs `prefetch_related`

| | `select_related` | `prefetch_related` |
|---|---|---|
| SQL | one query with JOINs | one extra query per relation: `WHERE fk IN (...)` |
| Relations | forward FK and one-to-one (single object) | reverse FK and M2M (sets of objects); also works for FKs and generic relations |
| Where the join happens | database | Python, matching by primary key |
| Watch out for | many wide JOINs on big tables | huge `IN` lists; later `.filter()` calls bypass the cache |

```python
courses = (
    Course.objects.filter(status="published")
    .select_related("instructor", "category")       # single-valued: JOIN
    .prefetch_related("modules__lessons")           # multi-valued: two extra queries
)
for course in courses:                              # 3 queries in total, however many rows
    for module in course.modules.all():
        for lesson in module.lessons.all():
            print(course.instructor.username, module.title, lesson.title)
```

**`Prefetch` objects** customize the prefetched query:

```python
from django.db.models import Prefetch

visible_reviews = Review.objects.filter(is_hidden=False).select_related("student").order_by("-created_at")
courses = Course.objects.prefetch_related(
    Prefetch("reviews", queryset=visible_reviews, to_attr="visible_reviews")
)
for course in courses:                       # 2 queries in total
    for review in course.visible_reviews:    # a plain list: filtered, ordered, students joined
        print(review.student.username, review.rating)
```

**The prefetch cache trap:**

```python
courses = Course.objects.prefetch_related("reviews")
for course in courses:
    course.reviews.all()               # served from the prefetch cache
    course.reviews.filter(rating=5)    # a NEW query per course: filter() ignores the cache
```

Either prefetch exactly what you need with `Prefetch(..., to_attr=...)` or filter the cached list in Python (`[r for r in course.reviews.all() if r.rating == 5]`).

## 3. Loading less data

```python
Course.objects.only("id", "title", "slug")            # SELECT id, title, slug
Course.objects.defer("description", "metadata")       # every column except the heavy ones

for course in Course.objects.only("id", "title"):
    course.description         # one extra query PER course: a deferred-field N+1

Course.objects.values("id", "title", "instructor__username")   # dicts with a JOIN, no model objects
Course.objects.values_list("id", flat=True)                    # [1, 2, 3]
```

Use `values()`/`values_list()` for exports, dropdowns, charts and collecting IDs to hand to a background task, where model behavior is not needed and instantiating thousands of objects is wasted CPU. Use `only()`/`defer()` when you still want model instances but some columns are large (long text, JSON). And let the database answer questions it can answer: `exists()` instead of `if qs:`, `count()` instead of `len(qs)`.

## 4. `annotate()` and `aggregate()`

```python
from django.db.models import Avg, Count, Q

courses = (
    Course.objects.filter(instructor=request.user)
    .annotate(
        student_count=Count("enrollments", distinct=True),
        avg_rating=Avg("reviews__rating"),
    )
    .order_by("-student_count")
)
courses[0].student_count, courses[0].avg_rating      # extra attributes on each object

Enrollment.objects.filter(course__instructor=request.user).aggregate(
    total=Count("id"),
    completed=Count("id", filter=Q(completed_at__isnull=False)),
)
# one dict, for example {'total': 412, 'completed': 97}

Course.objects.values("level").annotate(n=Count("id"))   # GROUP BY level: [{'level': 'beginner', 'n': 12}, ...]
```

`annotate()` works per row (`GROUP BY` the model's primary key), `aggregate()` collapses the whole QuerySet, and `values(...)` before `annotate(...)` changes the grouping to those fields. Annotated values can be filtered (`filter(student_count__gte=10)`, which becomes `HAVING`) and ordered.

**Join multiplication.** Each multi-valued relation in an aggregate adds a JOIN, and two of them multiply rows: a course with 30 enrollments and 10 reviews produces 300 joined rows, so `Count("enrollments")` returns 300. `distinct=True` fixes `Count`; for `Sum` or more complex cases, compute each aggregate in its own subquery (section 7). Averages over uniformly duplicated rows happen to stay correct, which is why this bug often hides until someone adds a `Sum`.

## 5. F expressions: let the database do the arithmetic

```python
from django.db.models import F

# Lost update: two requests both read 10 and both write 11
course = Course.objects.get(pk=pk)
course.view_count += 1                     # assume Course has view_count and capacity fields
course.save()

# Atomic: UPDATE ... SET view_count = view_count + 1
Course.objects.filter(pk=pk).update(view_count=F("view_count") + 1)

# On an instance
course.view_count = F("view_count") + 1
course.save(update_fields=["view_count"])
course.refresh_from_db(fields=["view_count"])   # the attribute holds an expression until refreshed

# Compare two columns of the same row
Course.objects.filter(seats_taken__lt=F("capacity"))
```

An `F()` object is a reference to a column that is resolved in SQL, so the read and the write happen in one statement and no other request can sneak in between. It is the Django equivalent of Rails' `update_counters`. Use it for counters, balances, stock and any read-modify-write on hot rows.

## 6. Q objects: OR, NOT and dynamic filters

```python
from django.db.models import Q

Course.objects.filter(Q(title__icontains=q) | Q(description__icontains=q))   # OR
Course.objects.filter(~Q(status="draft"), price__lte=50)                      # NOT; Q arguments go before kwargs

def search_courses(params):
    query = Q(status="published")
    if level := params.get("level"):
        query &= Q(level=level)
    if params.get("free") == "true":
        query &= Q(price=0)
    return Course.objects.filter(query)
```

Keyword arguments in `filter()` are always ANDed; `Q` objects combine with `|`, `&` and `~`, and can be built up step by step from request parameters.

## 7. Subquery, OuterRef, Exists and Case/When

```python
from datetime import timedelta
from django.db.models import Case, CharField, Count, Exists, OuterRef, Q, Subquery, Value, When
from django.utils import timezone

latest = Enrollment.objects.filter(course=OuterRef("pk")).order_by("-created_at")
courses = Course.objects.annotate(
    last_enrolled_at=Subquery(latest.values("created_at")[:1]),        # correlated subquery, one value
    is_enrolled=Exists(Enrollment.objects.filter(course=OuterRef("pk"), student=request.user)),
)

Course.objects.filter(Exists(Review.objects.filter(course=OuterRef("pk"), rating=5)))  # Exists in filter (3.0+)

week_ago = timezone.now() - timedelta(days=7)
Enrollment.objects.annotate(
    stage=Case(
        When(completed_at__isnull=False, then=Value("completed")),
        When(created_at__gte=week_ago, then=Value("new")),
        default=Value("in progress"),
        output_field=CharField(),
    )
)

Course.objects.annotate(                                               # conditional aggregation
    completed=Count("enrollments", filter=Q(enrollments__completed_at__isnull=False)),
    in_progress=Count("enrollments", filter=Q(enrollments__completed_at__isnull=True)),
)
```

- `OuterRef("pk")` points from the inner query to the outer row; `Subquery` must return one column (and one row, hence `[:1]`).
- `Exists` lets the database stop at the first match and avoids the duplicates a JOIN would create. It is ideal for "is the current user enrolled?" flags on a course list.
- `Case`/`When` is SQL `CASE WHEN`, useful for computed labels, custom sort orders and conditional updates.
- Several filtered `Count`s over the same relation share one JOIN, so one query returns a whole dashboard row.

## 8. Transactions: `atomic`, savepoints and `on_commit`

```python
from django.db import IntegrityError, transaction

def enroll_after_payment(user, course, payment_ref):
    with transaction.atomic():                         # all or nothing
        enrollment = Enrollment.objects.create(student=user, course=course)
        Payment.objects.create(user=user, course=course, provider_ref=payment_ref)
        try:
            with transaction.atomic():                 # nested block = savepoint
                Badge.objects.create(user=user, kind="first_enrollment")
        except IntegrityError:
            pass                                       # badge already exists; only the savepoint rolls back
        transaction.on_commit(lambda: send_receipt_email.delay(enrollment.pk))   # runs after COMMIT
    return enrollment
```

- By default Django is in **autocommit** mode: every query commits on its own. `atomic()` (as a decorator or context manager) opens a transaction, commits when the block exits normally and rolls back if an exception escapes.
- A nested `atomic()` creates a **savepoint**. Catching a database error inside an `atomic()` block *without* a nested savepoint leaves the transaction broken, and the next query raises `TransactionManagementError`.
- `transaction.on_commit()` callbacks run after the outermost transaction commits and are dropped on rollback. Enqueue Celery tasks there: otherwise a fast worker can start before the commit and fail with `DoesNotExist`, or act on rows that were later rolled back.
- `ATOMIC_REQUESTS = True` (per database) wraps every view in a transaction. It is simple, but it holds the transaction open for the whole request, including slow external calls.
- Keep transactions short and never call external APIs (Stripe, email) while holding locks. PostgreSQL defaults to Read Committed isolation; see [transactions and concurrency](../12-databases/06-transactions-and-concurrency.md).

## 9. `select_for_update` and lock-free alternatives

```python
def reserve_seat(course_id, user):
    with transaction.atomic():
        course = Course.objects.select_for_update().get(pk=course_id)   # SELECT ... FOR UPDATE
        if course.seats_taken >= course.capacity:
            raise CourseFull(course_id)
        course.seats_taken += 1           # safe: competing transactions wait for this row lock
        course.save(update_fields=["seats_taken"])
        Enrollment.objects.create(student=user, course=course)

# Lock-free version: one conditional UPDATE, check the affected row count
updated = Course.objects.filter(pk=course_id, seats_taken__lt=F("capacity")).update(seats_taken=F("seats_taken") + 1)
if updated == 0:
    raise CourseFull(course_id)

# Work queue: each worker grabs rows nobody else has locked
with transaction.atomic():
    batch = list(
        PayoutRequest.objects.select_for_update(skip_locked=True)
        .filter(status="pending").order_by("created_at")[:50]
    )
    for payout in batch:
        process(payout)
```

- The lock lasts until the transaction ends, so `select_for_update()` must be evaluated inside `atomic()` (in autocommit mode Django raises `TransactionManagementError`).
- `nowait=True` raises immediately instead of waiting; `skip_locked=True` silently skips locked rows; `of=("self",)` limits the lock to the main table when you also `select_related`. SQLite ignores the clause.
- Lock rows in a consistent order to avoid deadlocks, and keep the locked section tiny. When a single conditional `UPDATE` can express the rule, prefer it: it needs no explicit lock.

## 10. Bulk operations and `iterator()`

```python
Lesson.objects.bulk_create(lessons, batch_size=1000)                       # a few INSERTs, not 10,000
Lesson.objects.bulk_update(changed_lessons, ["position", "title"], batch_size=1000)
LessonProgress.objects.bulk_create(rows, ignore_conflicts=True)           # skip rows hitting a unique constraint
CourseStat.objects.bulk_create(                                           # upsert (4.1+)
    stats, update_conflicts=True, unique_fields=["course"], update_fields=["student_count", "avg_rating"]
)

for enrollment in Enrollment.objects.select_related("student").iterator(chunk_size=2000):
    write_csv_row(enrollment)        # streams rows; nothing is cached on the QuerySet
```

`iterator()` fetches rows in chunks (server-side cursors on PostgreSQL) instead of loading and caching the whole result, so memory stays flat for exports and batch jobs. Since Django 4.1 it can also apply `prefetch_related` when you pass `chunk_size`. For updates across millions of rows, process primary key ranges in batches (often in a background job) rather than one enormous statement that locks the table.

## 11. Raw SQL, safely

```python
from django.db import connection

courses = Course.objects.raw(
    "SELECT * FROM courses_course WHERE level = %s AND price < %s", ["beginner", 20]
)

with connection.cursor() as cursor:
    cursor.execute(
        "SELECT course_id, COUNT(*) FROM courses_enrollment WHERE created_at >= %s GROUP BY course_id",
        [since],
    )
    rows = cursor.fetchall()

# NEVER build SQL with f-strings or % formatting on user input: SQL injection
Course.objects.raw(f"SELECT * FROM courses_course WHERE title = '{q}'")
```

Pass parameters as the second argument and let the driver escape them; Django uses `%s` placeholders on every backend, and you must not wrap them in quotes. Reach for raw SQL only when the ORM cannot express a query (window functions in complex reports, recursive CTEs) or for measured hot paths, and keep it in one well-tested place. `RawSQL()` expressions exist for embedding a fragment; `extra()` is legacy, so avoid it.

## 12. Indexes and `.explain()`

```python
print(Course.objects.filter(status="published").order_by("-published_at").explain())
print(Enrollment.objects.filter(student=user).explain(analyze=True))   # PostgreSQL: actually runs it
```

Read the plan for sequential scans on large tables, big sorts and row estimates that are far off, then add an index that matches the real filter and sort pattern through `Meta.indexes` (composite and partial indexes are supported, and `django.contrib.postgres.indexes` adds GIN and other PostgreSQL types). Django already indexes every `ForeignKey` column. Each index slows writes and uses space, so index for measured queries, not for every column. The theory is in [indexing and query optimization](../12-databases/05-indexing-and-query-optimization.md).

## 13. Connections and pooling

- By default `CONN_MAX_AGE = 0`, so each request opens and closes its own database connection. Setting it (for example to 60 seconds) lets a worker reuse its connection across requests; `CONN_HEALTH_CHECKS = True` (4.1+) discards broken persistent connections.
- Every worker process or thread holds its own connection, so total connections are roughly workers times threads times instances; keep that below PostgreSQL's `max_connections`.
- PgBouncer in transaction pooling mode multiplexes many app connections onto a few database connections. With it, set `DISABLE_SERVER_SIDE_CURSORS = True` (server-side cursors used by `iterator()` do not survive transaction pooling).
- Django 5.1 added optional native connection pooling for PostgreSQL through psycopg 3 (`"OPTIONS": {"pool": True}`). It is newer, so check the docs for your version, including how it interacts with `CONN_MAX_AGE`.

## Common pitfalls
- Optimizing blind: measure first with the debug toolbar or SQL logs, then fix the biggest offenders.
- `select_related` on a many-to-many or reverse foreign key (it raises an error), or `prefetch_related` where a simple JOIN would do.
- `.filter()`, `.order_by()` or `.count()` on a prefetched relation inside a loop, silently re-querying per row.
- `only()` followed by access to a deferred field in a loop.
- A `SerializerMethodField` or model property that queries the database for every object in a list.
- Two `Count` annotations over different relations without `distinct=True`, producing inflated numbers.
- Read-modify-write in Python (`obj.count += 1; obj.save()`) on hot rows instead of an `F()` update.
- Catching `IntegrityError` inside `atomic()` without a nested savepoint, then hitting `TransactionManagementError`.
- Enqueueing Celery tasks or sending emails inside a transaction instead of in `transaction.on_commit()`.
- `select_for_update()` outside `atomic()`, or calling external APIs while holding row locks.
- Loading a million rows with `list(qs)` or plain iteration instead of `iterator()` and batching.
- Raw SQL with f-strings.
- `DEBUG = True` in a long-running script, which keeps every query in `connection.queries` and leaks memory.

## How to talk about this in an interview
- Tell the N+1 story concretely, using your real example from the prompt below: a list endpoint, the debug toolbar showing dozens of near-identical queries, `select_related` for the instructor, `prefetch_related` or a `Prefetch` for nested modules, `annotate` for counts, and an `assertNumQueries` test so it cannot regress. You know the same story from ActiveRecord (`includes`, `preload`, `eager_load`), which shows the concept, not just the API.
- For dashboards, explain that aggregates belong in the database (`annotate`, conditional `Count`, `Subquery`), not in Python loops over every enrollment.
- For concurrency, give the ladder: an `F()` or conditional update first, `select_for_update` when you must read then decide, unique constraints as the final guard, and `on_commit` for side effects.
- > **Fill in:** an N+1 or slow query you found in the LMS (which endpoint, what it cost, how you fixed it).
- > **Fill in:** how the instructor analytics were computed (annotations, cached counters, a nightly job?) and how long the main dashboard query took.
- > **Fill in:** whether enrollment or payment code used `transaction.atomic()` and `on_commit()`, and why.

## Interview questions

**Q1. What is the N+1 problem in Django, and how do you detect it?**
Loading a list with one query and then touching a relation on each object (`course.instructor`, `course.reviews.count`) fires one more query per row. Detect it with django-debug-toolbar's SQL panel (duplicated queries), SQL logging of `django.db.backends`, `connection.queries` in a shell, or `assertNumQueries` in tests seeded with several rows.
*Follow-up:* Why do templates make it worse? They can call zero-argument methods such as `course.modules.all`, which hides queries from the view author.

**Q2. `select_related` vs `prefetch_related`?**
`select_related` follows single-valued relations (forward FK, one-to-one) with SQL JOINs in the same query. `prefetch_related` handles multi-valued relations (reverse FK, M2M) with one extra query per relation using `IN (...)` and joins the results in Python. Combine them: `select_related("instructor").prefetch_related("modules__lessons")`.
*Follow-up:* Can `prefetch_related` follow a foreign key? Yes, but `select_related` is usually cheaper for single objects.

**Q3. What is a `Prefetch` object for?**
It controls the queryset used for a prefetch: filtering (only visible reviews), ordering (lessons by position), nested `select_related`, and `to_attr` to store the result as a plain list attribute. It avoids both over-fetching and the re-query that happens when you filter a prefetched relation later.
*Follow-up:* What does `to_attr` change? The results land on a list attribute instead of the related manager's cache, which makes it explicit that they are pre-filtered.

**Q4. You used `prefetch_related("reviews")`, yet `course.reviews.filter(rating=5)` still queries per course. Why?**
The prefetch cache only serves `.all()`. Any new QuerySet method (`filter`, `order_by`, `exclude`) builds a new query. Prefetch exactly what you need with `Prefetch("reviews", queryset=Review.objects.filter(rating=5), to_attr="top_reviews")`, or filter the cached list in Python.
*Follow-up:* What about `.count()` on a prefetched relation? Django uses the cache for `count()` after a prefetch, but annotating the count in the main query is cleaner.

**Q5. `only()`/`defer()` vs `values()`/`values_list()`?**
`only()` and `defer()` still return model instances but load fewer columns; accessing a deferred column later triggers one query per object. `values()` returns dicts and `values_list()` tuples (or flat values), skipping model instantiation entirely, which is faster for exports and read-only data but loses model methods.
*Follow-up:* When is `values_list("id", flat=True)` handy? Collecting IDs to pass to a background task or an `__in` filter.

**Q6. `annotate()` vs `aggregate()`?**
`annotate()` adds a computed value to each row (`Course.objects.annotate(student_count=Count("enrollments"))`), so you still get a QuerySet of objects. `aggregate()` reduces the whole QuerySet to one dict (`Enrollment.objects.aggregate(total=Count("id"))`). `values("level").annotate(...)` groups by the listed fields.
*Follow-up:* How do you filter on an annotation? Chain `.filter(student_count__gte=10)`, which Django turns into a `HAVING` clause.

**Q7. Why can two `Count` annotations return inflated numbers?**
Each multi-valued relation adds a JOIN, and two JOINs multiply rows: 30 enrollments times 10 reviews gives 300 rows per course, so both counts are wrong. Use `Count(..., distinct=True)`, or compute each aggregate in a `Subquery` when you need `Sum` or more complex expressions.
*Follow-up:* Several counts over the same relation? Conditional aggregation (`Count("enrollments", filter=Q(...))`) shares a single JOIN and stays correct.

**Q8. What are F expressions, and how do they prevent race conditions?**
`F("field")` references a column value inside the database, so `update(view_count=F("view_count") + 1)` performs the read and the write in one SQL statement. Two concurrent requests can no longer read the same old value and overwrite each other. `F` also compares columns in filters (`seats_taken__lt=F("capacity")`).
*Follow-up:* After `obj.count = F("count") + 1; obj.save()`, what is `obj.count`? Still an expression object until you call `refresh_from_db()`.

**Q9. What are Q objects?**
Objects that wrap filter conditions so they can be combined with `|` (OR), `&` (AND) and `~` (NOT) and built dynamically from request parameters. Plain keyword arguments to `filter()` can only express AND.
*Follow-up:* Can you mix `Q` objects and keyword arguments? Yes, but the `Q` arguments must come first.

**Q10. When would you use `Subquery`, `OuterRef` and `Exists`?**
For values that depend on each outer row without a JOIN explosion: the latest enrollment date per course (`Subquery` of an ordered, sliced queryset referencing `OuterRef("pk")`), or an `is_enrolled` flag for the current user on a course list (`Exists`). `Exists` stops at the first match and never duplicates rows.
*Follow-up:* Why `[:1]` on a Subquery? A subquery used as a value must return a single row and column.

**Q11. How do transactions work in Django? What does a nested `atomic()` do?**
Django runs in autocommit mode, committing each query immediately. `transaction.atomic()` groups statements into a transaction that commits on success and rolls back if an exception escapes. A nested `atomic()` creates a savepoint, so a failure inside it rolls back only that part, provided you catch the exception outside the inner block.
*Follow-up:* What is `ATOMIC_REQUESTS`? A per-database setting that wraps every view in a transaction; convenient, but long requests then hold transactions open.

**Q12. Why use `transaction.on_commit()`?**
Side effects such as sending email or enqueueing a Celery task must happen only if the data was really committed. Inside a transaction, a worker might pick up the task before the commit (and not find the row) or act on data that gets rolled back. `on_commit` defers the callback until after the outermost commit and drops it on rollback.
*Follow-up:* How do you test it with `TestCase`? `TestCase` never commits, so wrap the call in `self.captureOnCommitCallbacks(execute=True)`.

**Q13. What does `select_for_update()` do, and what are the alternatives?**
It adds `FOR UPDATE` to the SELECT, locking the rows until the transaction ends, so a read-check-write sequence (seat reservation, wallet balance) cannot interleave with another transaction. It must run inside `atomic()`; `skip_locked=True` lets several workers pull from a queue table without blocking each other, and `nowait=True` fails fast. Often a conditional `UPDATE ... WHERE seats_taken < capacity` with an `F()` expression does the job without an explicit lock.
*Follow-up:* What is the risk? Deadlocks and lock waits, so lock in a consistent order and keep the section short.

**Q14. How do you process millions of rows with the ORM?**
Stream with `iterator(chunk_size=...)` so the QuerySet does not cache everything, write in batches with `bulk_create` or `bulk_update` and a `batch_size`, or push the whole operation into one set-based `update()`. Run long jobs in the background, batch by primary key ranges, and commit per batch so a failure does not undo hours of work.
*Follow-up:* What breaks with PgBouncer in transaction mode? Server-side cursors, so set `DISABLE_SERVER_SIDE_CURSORS = True`.

**Q15. How do you run raw SQL safely in Django?**
Use `Model.objects.raw(sql, params)` to get model instances, or `connection.cursor()` with `cursor.execute(sql, params)` for arbitrary results, always passing user values through the params list with `%s` placeholders. Never interpolate values with f-strings or `%`, which reopens SQL injection. Keep raw SQL rare, isolated and tested.
*Follow-up:* Is the ORM immune to injection? Its parameterized queries are, but `raw()`, `RawSQL` and `extra()` with interpolated strings are not.

**Q16. A Django page is slow. How do you investigate?**
Reproduce it with the debug toolbar or an APM tool and look at the query count and the slowest queries. Fix N+1 patterns with `select_related`, `prefetch_related` and annotations, run `.explain(analyze=True)` on slow queries to spot sequential scans and missing indexes, and only then consider caching. Confirm the improvement with numbers and lock it in with `assertNumQueries`.
*Follow-up:* What if the time is not in SQL? Profile Python (serialization, template rendering) and external calls, which often belong in a background job.

**Q17. What do `CONN_MAX_AGE` and connection pooling do?**
`CONN_MAX_AGE` keeps a worker's database connection open for reuse across requests instead of reconnecting every time (`0` closes it after each request). A pooler such as PgBouncer shares a small number of database connections among many app processes, which matters when workers times threads would exceed `max_connections`. Django 5.1 added optional native pooling for PostgreSQL via psycopg 3.
*Follow-up:* What does `CONN_HEALTH_CHECKS` add? It verifies a persistent connection before reusing it, so a dropped connection does not fail a request.

## Further reading
- [Database access optimization](https://docs.djangoproject.com/en/stable/topics/db/optimization/)
- [Aggregation](https://docs.djangoproject.com/en/stable/topics/db/aggregation/)
- [Query expressions (F, Subquery, Exists)](https://docs.djangoproject.com/en/stable/ref/models/expressions/)
- [Database transactions](https://docs.djangoproject.com/en/stable/topics/db/transactions/)
- [Performing raw SQL queries](https://docs.djangoproject.com/en/stable/topics/db/sql/)
