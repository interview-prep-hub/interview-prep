# Django Models and the ORM

> **Tier:** T1 (know cold) · **Related:** [Django fundamentals](01-django-fundamentals.md), [ORM performance](03-orm-performance.md), [LMS project patterns](07-lms-project-patterns.md), [Django rapid-fire questions](99-interview-questions.md), [DBMS fundamentals](../12-databases/01-dbms-fundamentals.md), [SQL from zero](../12-databases/02-sql-from-zero.md), [ORMs compared](../12-databases/12-orms-compared.md), [ActiveRecord associations](../05-ruby-on-rails/04-activerecord-associations.md)
> **On your resume:** The PureLogics LMS data model (users with roles, courses, lessons, enrollments, progress, reviews and subscriptions) lived in Django models and migrations. You also use PostgreSQL and ActiveRecord daily at Devntech, so you can compare the two ORMs from experience.

## Quick revision
- A model is a class mapped to a table and each field is a column. Django adds an auto-incrementing `id` (type set by `DEFAULT_AUTO_FIELD`, usually `BigAutoField`).
- `null=True` is about the database (the column allows NULL); `blank=True` is about validation (forms and the admin accept empty). For text fields use `blank=True` with `default=""`, not `null=True`.
- Mutable defaults must be callables: `default=dict`, never `default={}`. `db_default` (5.0+) sets a database-level default.
- `choices` with `TextChoices`/`IntegerChoices` gives named enums and `get_<field>_display()`.
- Every `ForeignKey` needs `on_delete`: `CASCADE`, `PROTECT`, `RESTRICT`, `SET_NULL` (needs `null=True`), `SET_DEFAULT`, `SET(...)`, `DO_NOTHING`.
- `OneToOneField` is a unique foreign key (profile pattern). `ManyToManyField` creates a join table; use `through=` a model when the link carries data (an `Enrollment` with dates).
- `related_name` names the reverse accessor (`course.enrollments.all()`); the default is `<model>_set`.
- `Meta`: `ordering`, `indexes`, `constraints` (`UniqueConstraint`, `CheckConstraint`), `db_table`, `abstract`, `proxy`.
- `makemigrations` writes migration files from model changes; `migrate` applies them and records them in `django_migrations`; `sqlmigrate` prints the SQL; `RunPython` runs data migrations with `apps.get_model()`.
- Two branches adding migrations with the same parent conflict: fix with `makemigrations --merge` or by renumbering before merging.
- QuerySets are lazy: chaining builds SQL, and iteration, `list()`, `len()`, `bool()`, `repr()` or a stepped slice runs it. An evaluated QuerySet caches its rows.
- `get()` raises `DoesNotExist` or `MultipleObjectsReturned`; `first()` returns `None`. Prefer `exists()` and `count()` over loading rows.
- `get_or_create()` and `update_or_create()` return `(obj, created)` and are only race-safe with a unique constraint behind them.
- QuerySet `update()`, `bulk_create()` and `bulk_update()` skip `save()` and save signals; QuerySet `delete()` skips each model's `delete()` method but still sends delete signals.
- Lookups use double underscores and can cross relations: `price__gte=10`, `title__icontains="django"`, `enrollments__student=user`.
- Custom QuerySet methods exposed with `as_manager()` give chainable, named filters: `Course.objects.published().taught_by(user)`.
- Override `save()` for invariants of the model itself; use signals for decoupled reactions. Neither runs for `update()` or `bulk_create()`, and `save()` does not call `full_clean()`.
- Inheritance: abstract base (shared fields, no table), multi-table (a table per class joined one-to-one), proxy (same table, different Python behavior).

## 1. Models and fields

```python
# courses/models.py
from django.conf import settings
from django.core.validators import MinValueValidator
from django.db import models
from django.db.models import Q

class TimeStampedModel(models.Model):
    created_at = models.DateTimeField(auto_now_add=True)   # set once on insert
    updated_at = models.DateTimeField(auto_now=True)       # set on every save()

    class Meta:
        abstract = True                                    # no table; fields are copied into children

class Category(models.Model):
    name = models.CharField(max_length=80, unique=True)
    slug = models.SlugField(unique=True)

    def __str__(self):
        return self.name

class Course(TimeStampedModel):
    class Status(models.TextChoices):
        DRAFT = "draft", "Draft"
        PENDING = "pending", "Pending review"
        PUBLISHED = "published", "Published"

    class Level(models.TextChoices):
        BEGINNER = "beginner", "Beginner"
        ADVANCED = "advanced", "Advanced"

    instructor = models.ForeignKey(
        settings.AUTH_USER_MODEL, on_delete=models.PROTECT, related_name="courses_taught"
    )
    category = models.ForeignKey(
        Category, on_delete=models.SET_NULL, null=True, blank=True, related_name="courses"
    )
    students = models.ManyToManyField(
        settings.AUTH_USER_MODEL, through="Enrollment", related_name="enrolled_courses"
    )
    title = models.CharField(max_length=200)
    slug = models.SlugField(max_length=220, unique=True)
    description = models.TextField(blank=True)
    price = models.DecimalField(max_digits=8, decimal_places=2, default=0, validators=[MinValueValidator(0)])
    level = models.CharField(max_length=20, choices=Level.choices, default=Level.BEGINNER)
    status = models.CharField(max_length=20, choices=Status.choices, default=Status.DRAFT, db_index=True)
    published_at = models.DateTimeField(null=True, blank=True)
    metadata = models.JSONField(default=dict, blank=True)    # callable default, never {}

    class Meta:
        ordering = ["-created_at"]
        indexes = [models.Index(fields=["status", "-published_at"], name="course_status_pub_idx")]

    def __str__(self):
        return self.title
```

Common field types: `CharField` (requires `max_length`), `TextField`, `IntegerField`, `PositiveIntegerField`, `DecimalField` (money: `max_digits`, `decimal_places`), `BooleanField`, `DateField`, `DateTimeField`, `EmailField`, `SlugField`, `URLField`, `UUIDField`, `JSONField`, `FileField` and `ImageField` (needs Pillow), plus the relationship fields below.

## 2. Field options

| Option | Layer | Meaning |
|---|---|---|
| `null=True` | database | Column allows NULL |
| `blank=True` | validation | Forms, the admin and `full_clean()` accept an empty value |
| `default=` | Python | Value (or callable) used when creating an object without one |
| `db_default=` (5.0+) | database | Default computed by the database, applied to inserts from any client |
| `unique=True` | database | Unique index; violations raise `IntegrityError` |
| `db_index=True` | database | Plain index on the column |
| `choices=` | validation and display | Allowed values plus labels, `get_<field>_display()` |
| `validators=[...]` | validation | Extra checks run by `full_clean()` (forms, admin, serializers for model fields) |
| `editable=False` | forms | Hidden from ModelForms and the admin |
| `auto_now_add`, `auto_now` | Python | Timestamps set on insert, or on every `save()` |

`null` vs `blank` combinations: an optional date is `null=True, blank=True`; an optional text field is `blank=True, default=""` (otherwise "empty" has two representations, `NULL` and `""`); a required field uses neither. The exception is a text field that is both optional and unique, which needs `null=True` because many empty strings would collide on the unique index.

`TextChoices` stores the first value (`"published"`) and exposes the label (`"Published"`); since Django 5.0 you can also pass the enum class directly as `choices=Status`.

## 3. Relationships

```python
class Module(models.Model):
    course = models.ForeignKey(Course, on_delete=models.CASCADE, related_name="modules")
    title = models.CharField(max_length=200)
    position = models.PositiveIntegerField()

    class Meta:
        ordering = ["position"]
        constraints = [models.UniqueConstraint(fields=["course", "position"], name="unique_module_position")]

class Enrollment(TimeStampedModel):                  # the "through" model of Course.students
    student = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name="enrollments")
    course = models.ForeignKey(Course, on_delete=models.CASCADE, related_name="enrollments")
    completed_at = models.DateTimeField(null=True, blank=True)

    class Meta:
        constraints = [models.UniqueConstraint(fields=["student", "course"], name="unique_enrollment")]

class InstructorProfile(models.Model):
    user = models.OneToOneField(settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name="instructor_profile")
    bio = models.TextField(blank=True)
```

```python
course.instructor                      # forward FK: one object (a query unless select_related)
course.modules.all()                   # reverse FK via related_name
user.courses_taught.filter(status="published")
user.enrolled_courses.all()            # M2M through Enrollment
course.students.count()
user.instructor_profile                # reverse one-to-one: raises RelatedObjectDoesNotExist if missing
course.students.add(user)              # fine with a through model (2.2+); pass through_defaults={...} for required extra fields
```

| `on_delete` | When the parent is deleted | Typical use |
|---|---|---|
| `CASCADE` | Delete the children too | Children that cannot exist alone (modules of a course) |
| `PROTECT` | Raise `ProtectedError` | Data you must not lose (an instructor's courses, payments) |
| `RESTRICT` | Like `PROTECT`, but allowed when the parent is also being deleted through a `CASCADE` in the same operation | Complex ownership graphs |
| `SET_NULL` | Set the FK to NULL (requires `null=True`) | Optional links (a deleted category) |
| `SET_DEFAULT`, `SET(...)` | Set a default value or the result of a callable | Reassign to a placeholder ("deleted user") |
| `DO_NOTHING` | Django does nothing; the database constraint decides | You manage referential actions in SQL |

By default Django implements `on_delete` in Python: it collects related rows and issues the deletes or updates itself, sending signals along the way, while the database foreign key has no `ON DELETE` action (recent versions add database-level options; check the release notes). That is why deleting one course can run many queries.

**`related_name`** names the reverse side. Without it you get `course.enrollment_set`, and two foreign keys from one model to the same target (an `author` and a `reviewer` pointing at `User`) clash until you name them. `related_name="+"` disables the reverse accessor; `related_query_name` changes the name used in lookups.

## 4. Meta options: ordering, indexes and constraints

```python
class Review(TimeStampedModel):
    course = models.ForeignKey(Course, on_delete=models.CASCADE, related_name="reviews")
    student = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name="reviews")
    rating = models.PositiveSmallIntegerField()
    body = models.TextField(blank=True)
    is_hidden = models.BooleanField(default=False)

    class Meta:
        ordering = ["-created_at"]
        indexes = [models.Index(fields=["course", "-created_at"], name="review_course_recent_idx")]
        constraints = [
            models.UniqueConstraint(fields=["course", "student"], name="one_review_per_student"),
            models.CheckConstraint(condition=Q(rating__gte=1) & Q(rating__lte=5), name="rating_1_to_5"),
        ]
```

- `ordering` applies to every query that does not call `order_by()`, so keep it cheap (indexed) or leave it out; it also silently affects `distinct()` and aggregation queries.
- `indexes` are created by migrations; put the equality-filtered column first in a composite index (see [indexing](../12-databases/05-indexing-and-query-optimization.md)). Partial indexes take `condition=Q(...)`.
- `UniqueConstraint` supports `condition=` for partial uniqueness (for example one active subscription per user) and replaces the older `unique_together`.
- `CheckConstraint` enforces row rules in the database. Django 5.1 renamed its `check=` argument to `condition=`; older code uses `check=`.
- Constraints are the real guarantee: form and serializer validation runs only in your code paths, while bulk scripts, the shell, race conditions and other services all hit the database rules.

## 5. Migrations

```bash
python manage.py makemigrations courses      # write courses/migrations/0005_....py from model changes
python manage.py sqlmigrate courses 0005     # print the SQL it will run (review it for big tables)
python manage.py migrate                     # apply pending migrations, recorded in django_migrations
python manage.py showmigrations courses      # [X] applied, [ ] pending
python manage.py migrate courses 0004        # roll back to 0004 by running reverse operations
python manage.py makemigrations --merge      # join two conflicting leaf migrations
python manage.py squashmigrations courses 0001 0020   # collapse old history into one migration
```

How it works: every migration file lists `dependencies` and `operations` (`CreateModel`, `AddField`, `AlterField`, `AddConstraint`, `RunPython`, `RunSQL`). Django replays all files to build a historical "migration state" of your models, diffs it against the current `models.py`, and writes the difference as a new migration. `migrate` walks the dependency graph and applies unapplied migrations, each inside a transaction on databases with transactional DDL such as PostgreSQL (MySQL cannot roll back DDL). Migration files are code: commit them, review them, and never edit one that has already run anywhere shared; add a new migration instead.

**Data migrations** change rows rather than schema:

```python
# courses/migrations/0006_backfill_slugs.py
from django.db import migrations
from django.utils.text import slugify

def backfill_slugs(apps, schema_editor):
    Course = apps.get_model("courses", "Course")       # historical model, not a direct import
    for course in Course.objects.filter(slug="").iterator():
        course.slug = f"{slugify(course.title)}-{course.pk}"
        course.save(update_fields=["slug"])

class Migration(migrations.Migration):
    dependencies = [("courses", "0005_course_slug")]
    operations = [migrations.RunPython(backfill_slugs, migrations.RunPython.noop)]
```

`apps.get_model()` returns the model as it existed at this point in history, so the migration keeps working after future field changes; a direct import would use today's model. Historical models have no custom methods or `save()` overrides. Always provide a reverse function (even `RunPython.noop`) so the migration can be rolled back.

**Conflicts and squashing.** If two branches each add `0007_...` on top of `0006`, Django refuses to migrate with two leaf nodes. Run `makemigrations --merge` to create a merge migration, or, before merging, delete your unshared migration and regenerate it on top of the other branch. `squashmigrations` replaces a long history with one equivalent migration to speed up test database creation; once every environment has applied it, the old files can go.

### Going deeper: migrations on large production tables
- Use expand and contract for renames or type changes: add the new nullable column, deploy code that writes both, backfill in batches, switch reads, then drop the old column in a later release. Old and new code run side by side during a rolling deploy.
- Adding a NOT NULL column to a huge table: add it nullable (or with a database default), backfill in batches outside the migration, then add the constraint.
- On PostgreSQL, build indexes without blocking writes using `AddIndexConcurrently` from `django.contrib.postgres.operations` in a migration with `atomic = False`.
- Keep long backfills out of migrations (a management command or batched background job) so deploys stay fast and restartable.

## 6. The QuerySet API

```python
Course.objects.create(instructor=ana, title="DRF Basics", slug="drf-basics")   # INSERT
Course.objects.all()                                         # every row (lazy)
Course.objects.filter(status="published", price__lte=50)     # WHERE ... AND ...
Course.objects.exclude(category__isnull=True)                # WHERE NOT (...)
Course.objects.order_by("-published_at", "title")            # ORDER BY
Course.objects.get(slug="drf-basics")                        # exactly one row, or an exception
Course.objects.filter(slug="drf-basics").first()             # an object or None
Course.objects.filter(status="draft").exists()               # SELECT 1 ... LIMIT 1
Course.objects.filter(status="published").count()            # SELECT COUNT(*)
Course.objects.values("id", "title")                         # dicts: [{'id': 1, 'title': '...'}]
Course.objects.values_list("id", flat=True)                  # flat values: [1, 2, 3]
Course.objects.values_list("level", flat=True).distinct()    # SELECT DISTINCT
Course.objects.filter(pk__in=[1, 2, 3])[:10]                 # LIMIT 10, still lazy
```

```python
try:
    course = Course.objects.get(slug=slug)
except Course.DoesNotExist:                  # each model has its own exception classes
    raise Http404("No such course")
except Course.MultipleObjectsReturned:       # the lookup is not unique: fix the data, add a constraint
    raise
```

In views, `get_object_or_404(Course, slug=slug)` wraps the first case.

```python
enrollment, created = Enrollment.objects.get_or_create(student=user, course=course)

subscription, created = Subscription.objects.update_or_create(
    stripe_subscription_id=sub_id,                                   # lookup
    defaults={"status": status, "current_period_end": period_end},  # values to set
)
# Django 5.0+ also accepts create_defaults= for values used only when creating
```

`get_or_create` tries a `get`, then a `create` inside a savepoint; if a concurrent request inserted the same row first, the unique constraint raises `IntegrityError`, and Django catches it and returns the existing row. Without a unique constraint on the lookup fields, two requests can both create a row.

```python
Course.objects.filter(status="pending", created_at__lt=cutoff).update(status="draft")   # one UPDATE, returns the row count
Enrollment.objects.filter(course=course).delete()     # returns (total, {"courses.Enrollment": n, ...})
Lesson.objects.bulk_create([Lesson(module=m, title=t, position=i) for i, t in enumerate(titles)], batch_size=500)
Lesson.objects.bulk_update(lessons, ["position"], batch_size=500)
```

Bulk operations are fast because they skip per-object work: `update()`, `bulk_create()` and `bulk_update()` do not call `save()`, do not send `pre_save`/`post_save`, and `update()` does not touch `auto_now` fields unless you set them. QuerySet `delete()` does not call each object's `delete()` method, but it still sends `pre_delete`/`post_delete` signals and follows `on_delete` rules.

## 7. Field lookups and spanning relations

| Lookup | SQL idea |
|---|---|
| `exact` (default), `iexact` | `=`, case-insensitive `=` |
| `contains`, `icontains` | `LIKE '%x%'` (`ILIKE` on PostgreSQL for `icontains`) |
| `startswith`, `endswith` (and `i` versions) | `LIKE 'x%'`, `LIKE '%x'` |
| `in` | `IN (...)` |
| `gt`, `gte`, `lt`, `lte`, `range` | comparisons, `BETWEEN` |
| `isnull` | `IS NULL` / `IS NOT NULL` |
| `date`, `year`, `month` | extract part of a date |

```python
Course.objects.filter(instructor__email__iendswith="@example.com")   # forward FK: JOIN users
Course.objects.filter(enrollments__student=user)                     # reverse FK: JOIN enrollments
User.objects.filter(enrolled_courses__category__slug="python").distinct()   # through the M2M
```

Filtering across a reverse foreign key or many-to-many returns one row per matching child, so the same course can appear several times: add `.distinct()` when duplicates matter. Conditions in one `filter()` call must match the **same** related row, while chained `filter()` calls may match different rows:

```python
Course.objects.filter(reviews__rating=5, reviews__created_at__year=2025)          # one review is 5 stars AND from 2025
Course.objects.filter(reviews__rating=5).filter(reviews__created_at__year=2025)   # some 5-star review, some 2025 review
```

## 8. Lazy evaluation and caching

```python
qs = Course.objects.filter(status="published")   # no query yet
qs = qs.order_by("title")                        # still none: each call returns a new QuerySet
for course in qs:                                # the query runs here and rows are cached on qs
    print(course.title)
for course in qs:                                # no new query: served from the cache
    print(course.slug)

Course.objects.all()[0]                          # LIMIT 1 query
Course.objects.all()[0]                          # another query: unevaluated QuerySets do not cache

qs = Course.objects.all()
if qs:                                           # evaluates and caches EVERY row just to test emptiness
    ...
if qs.exists():                                  # a cheap SELECT 1 ... LIMIT 1
    ...
```

A QuerySet runs its SQL when you iterate it, call `list()`, `len()` or `bool()`, `repr()` it in the shell, slice it with a step, or pickle it. Chaining and plain slicing (`qs[:10]`, which adds `LIMIT`) stay lazy. Once evaluated, a QuerySet keeps its results, but any new chained call (`qs.filter(...)`, `qs.all()`) builds a fresh query. Rule of thumb: `exists()` for yes/no, `count()` for numbers you will not iterate, and `len(qs)` only when you are going to loop over `qs` anyway.

## 9. Custom managers and QuerySet methods

```python
class CourseQuerySet(models.QuerySet):
    def published(self):
        return self.filter(status=Course.Status.PUBLISHED)

    def taught_by(self, user):
        return self.filter(instructor=user)

    def with_student_count(self):
        return self.annotate(student_count=models.Count("enrollments", distinct=True))

class Course(TimeStampedModel):
    # ... fields as above ...
    objects = CourseQuerySet.as_manager()

Course.objects.published().taught_by(request.user).with_student_count()   # chainable, reusable
```

Named QuerySet methods keep query logic in one place (the Django version of Rails scopes) and make views read like the business rules. A manager that overrides `get_queryset()` can hide rows by default, for example soft-deleted lessons, but the first manager on a model becomes the default manager used by the admin and by reverse relations, so many teams keep an unfiltered `objects` and add a second, named manager such as `active = ActiveLessonManager()`.

## 10. Model methods, `save()` overrides and signals

```python
from django.urls import reverse
from django.utils import timezone
from django.utils.text import slugify

class Course(TimeStampedModel):
    # ... fields ...
    def save(self, *args, **kwargs):
        if not self.slug:
            self.slug = slugify(self.title)                # the unique index still guards duplicates
        if self.status == self.Status.PUBLISHED and self.published_at is None:
            self.published_at = timezone.now()
        super().save(*args, **kwargs)                      # always call super and pass args through

    def get_absolute_url(self):
        return reverse("courses:detail", kwargs={"slug": self.slug})

class Enrollment(TimeStampedModel):
    # ... fields ...
    @property
    def is_completed(self):
        return self.completed_at is not None

    def mark_completed(self):
        self.completed_at = timezone.now()
        self.save(update_fields=["completed_at", "updated_at"])   # list auto_now fields explicitly
```

Model methods hold behavior that belongs to one row ("fat models, thin views"); logic spanning several models or external services (enrollment plus payment plus email) reads better in a service function. `clean()` holds model-level validation, but `save()` does **not** call `full_clean()`: ModelForms and the admin do, DRF serializers do not, and the shell and scripts never do, which is another reason for database constraints.

| | Override `save()` | Signals (`pre_save`, `post_save`, `post_delete`, `m2m_changed`) |
|---|---|---|
| Visibility | explicit, in the model | hidden, connected in another module |
| Good for | invariants of the model itself (slugs, derived fields) | reacting to models you do not own (the auth `User`, third-party apps), decoupled side effects |
| Skipped by | `update()`, `bulk_create()`, `bulk_update()`, raw SQL | the same operations |
| Transactions | runs inside the caller's transaction | same, so use `transaction.on_commit` for emails and tasks |

## 11. Model inheritance

| | Abstract base class | Multi-table inheritance | Proxy model |
|---|---|---|---|
| Declared with | `class Meta: abstract = True` | subclassing a concrete model | `class Meta: proxy = True` |
| Tables | none for the base; fields copied into each child | one per class; the child links to the parent through an implicit one-to-one (`<parent>_ptr`) | no new table: same rows as the parent |
| Queries | each child is queried on its own | child queries JOIN the parent table | same rows, different Python behavior |
| Use for | shared fields and methods (`TimeStampedModel`) | rarely; true subtypes with extra columns (often composition is clearer) | a different default manager, ordering, methods or admin for the same data |

```python
class PublishedCourseManager(models.Manager):
    def get_queryset(self):
        return super().get_queryset().filter(status="published")

class PublishedCourse(Course):         # same table as Course
    objects = PublishedCourseManager()

    class Meta:
        proxy = True
        ordering = ["-published_at"]
```

## Common pitfalls
- `null=True` on `CharField`/`TextField`, giving two kinds of "empty" (`NULL` and `""`).
- `default={}` or `default=[]` on a `JSONField`; the default must be a callable such as `dict` or `list`.
- `CASCADE` everywhere: deleting an instructor silently deletes their courses, enrollments and payment history. Use `PROTECT` for data you must keep.
- Two foreign keys to the same model without `related_name`, causing a reverse accessor clash at startup.
- Importing models directly in a data migration instead of using `apps.get_model()`.
- Editing or deleting migrations that already ran on shared databases, or running `makemigrations` on a server instead of committing the files.
- Calling `get()` in a view without handling `DoesNotExist`, turning a bad URL into a 500 instead of a 404.
- `if queryset:` or `len(queryset)` just to check for rows or count them; use `exists()` and `count()`.
- Expecting `update()` or `bulk_create()` to run `save()` logic, `auto_now` or signals.
- Duplicate results after filtering across reverse relations without `.distinct()`.
- `get_or_create()` on fields with no unique constraint, which allows duplicates under concurrency.
- Assuming `save()` validates. It does not call `full_clean()`, so only database constraints protect you from scripts and bulk paths.
- Business rules in signals that nobody can find, firing on code paths you did not expect.

## How to talk about this in an interview
- Sketch your real LMS schema in under a minute. A typical shape: users with a role, courses owned by instructors, modules and lessons ordered by position, enrollments as a through model with a unique (student, course) constraint, lesson progress, reviews with a rating check constraint, and subscriptions synced from the payment provider. Stress that invariants live in database constraints, not only in serializers.
- Compare with ActiveRecord, which you use daily: Django declares fields on the model and generates migrations, while Rails migrations define the schema and models infer attributes; `related_name` plays the role of naming a `has_many`; `on_delete` corresponds to `dependent:` plus foreign key actions; custom QuerySet methods are Rails scopes.
- If asked about something you did not do (squashing, zero-downtime migrations), explain the approach and say so honestly.
- > **Fill in:** the main models in your LMS and one constraint or index you added on purpose (and why).
- > **Fill in:** a migration problem you hit (merge conflict, data backfill, a field change on existing data) and how you resolved it.
- > **Fill in:** whether the LMS used PostgreSQL, MySQL or SQLite, locally and in deployment.

## Interview questions

**Q1. What is the difference between `null=True` and `blank=True`?**
`null=True` lets the database column store NULL; `blank=True` lets validation (forms, the admin, `full_clean()`) accept an empty value. An optional date or foreign key usually needs both; an optional text field uses `blank=True` with `default=""` so there is only one empty value. Optional unique text fields are the exception and need `null=True`.
*Follow-up:* Does `blank=True` affect the database? No, it is purely a validation setting.

**Q2. What `on_delete` options exist, and how do you choose?**
`CASCADE` deletes dependents, `PROTECT` blocks the delete with `ProtectedError`, `RESTRICT` blocks it unless the parent is also being cascaded away, `SET_NULL` and `SET_DEFAULT`/`SET()` re-point the foreign key, and `DO_NOTHING` leaves it to the database. Choose by asking whether the child makes sense without the parent: modules of a course cascade, payments and an instructor's courses are protected, an optional category becomes NULL.
*Follow-up:* Where is `CASCADE` executed? In Django's Python code by default, which collects and deletes related rows and sends signals.

**Q3. ForeignKey vs OneToOneField vs ManyToManyField: when do you need `through`?**
A ForeignKey is many-to-one (many enrollments belong to one course); OneToOneField is a unique foreign key (one profile per user); ManyToManyField links many to many through a join table. Use `through=` a model as soon as the relationship has its own data, such as an enrollment date, status or progress, which is almost always true in real systems.
*Follow-up:* Can you still use `add()` with a custom through model? Yes since Django 2.2, passing `through_defaults` for required extra fields.

**Q4. What does `related_name` do?**
It names the reverse accessor and the default reverse lookup: with `related_name="enrollments"` you write `course.enrollments.all()` and `Course.objects.filter(enrollments__student=user)`. Without it Django uses `enrollment_set`, and two foreign keys to the same model clash until you name them.
*Follow-up:* How do you disable the reverse relation? `related_name="+"`.

**Q5. How do Django migrations work?**
`makemigrations` rebuilds the historical model state from existing migration files, diffs it against the current models and writes a new file of operations with dependencies. `migrate` applies unapplied migrations in dependency order (in a transaction where the database supports transactional DDL) and records each one in the `django_migrations` table. `sqlmigrate` shows the SQL, and `migrate app 0004` rolls back.
*Follow-up:* What if a migration's table already exists? `migrate --fake` (or `--fake-initial`) marks it applied without running it; use it carefully.

**Q6. What is a data migration, and why use `apps.get_model()` inside it?**
A migration with `RunPython` (or `RunSQL`) that transforms existing rows, such as backfilling a new column. `apps.get_model()` returns the historical version of the model at that point in the migration graph, so the code keeps working after later schema changes; importing the model directly would use today's fields.
*Follow-up:* What should every `RunPython` include? A reverse function, even `RunPython.noop`, so the migration can be unapplied.

**Q7. How do you resolve conflicting migrations, and what is squashing?**
Conflicts happen when two branches add migrations with the same parent, leaving two leaf nodes. Fix them with `makemigrations --merge`, or regenerate your unshared migration on top of the other branch. Squashing (`squashmigrations app 0001 0020`) replaces a long chain with one equivalent migration to speed up setup; delete the originals only after every environment has applied the squash.
*Follow-up:* How do you avoid conflicts? Merge main often and keep migrations small and feature-scoped.

**Q8. How would you add a required column to a large production table safely?**
Add it as nullable (or with a database default) first, deploy code that writes it, backfill existing rows in batches outside the migration, then make it NOT NULL in a later migration. For indexes on PostgreSQL use `AddIndexConcurrently` in a non-atomic migration so writes are not blocked. Old and new code must both work during a rolling deploy.
*Follow-up:* How do you rename a column with zero downtime? Expand and contract: add the new column, dual-write, backfill, switch reads, drop the old column later.

**Q9. `get()` vs `filter()` vs `first()`?**
`get()` returns exactly one object and raises `DoesNotExist` or `MultipleObjectsReturned`; use it for unique lookups. `filter()` returns a lazy QuerySet of zero or more rows. `first()` returns the first object by ordering, or `None`, and never raises for an empty result.
*Follow-up:* What do you use in a view? `get_object_or_404()`, which turns `DoesNotExist` into a 404.

**Q10. What does it mean that QuerySets are lazy, and when do they hit the database?**
Building and chaining a QuerySet only composes SQL; nothing runs until you iterate it, call `list()`, `len()` or `bool()`, `repr()` it, slice it with a step or pickle it. After evaluation it caches its rows, so iterating again is free, but any new chained call builds a new query. Laziness lets you pass QuerySets around and keep refining them cheaply.
*Follow-up:* Is `qs[:10]` evaluated? No, it adds `LIMIT 10` and stays lazy.

**Q11. `exists()` vs `count()` vs `len()`?**
`exists()` runs a cheap `SELECT 1 ... LIMIT 1` for yes/no questions. `count()` runs `SELECT COUNT(*)` when you need a number but not the rows. `len(qs)` loads every row into memory, which is right only if you will iterate over those same rows anyway (it then reuses the cache).
*Follow-up:* What about `if qs:`? Same as `len()`: it evaluates the whole QuerySet.

**Q12. How does `get_or_create` behave under concurrency?**
It tries `get()`, then `create()` inside a savepoint. If two requests race, both may miss on `get()`, but only one insert succeeds when a unique constraint covers the lookup fields; Django catches the loser's `IntegrityError` and returns the existing row. Without a unique constraint, both inserts succeed and you get duplicates.
*Follow-up:* What is the difference between `defaults` and the lookup kwargs? Lookup kwargs find the row; `defaults` are only used to fill fields when creating (and to update in `update_or_create`).

**Q13. What do `update()`, `bulk_create()` and `bulk_update()` skip?**
They run set-based SQL and skip the model's `save()` method, `pre_save`/`post_save` signals and per-object validation; `update()` also leaves `auto_now` fields alone unless you set them. They are much faster for large batches, so put required logic into the query itself or run it separately.
*Follow-up:* Does `bulk_create` set primary keys on the objects? On PostgreSQL yes (it uses `RETURNING`), and on other backends that support it in recent versions.

**Q14. What is the difference between a custom manager and custom QuerySet methods?**
A manager is the entry point (`Course.objects`) and can change the base QuerySet by overriding `get_queryset()`. Custom QuerySet methods are chainable filters (`published()`, `taught_by(user)`); `CourseQuerySet.as_manager()` exposes them on the manager so they chain in any order. Prefer QuerySet methods for reusable filters and keep the default manager unfiltered.
*Follow-up:* Why is a filtering default manager risky? The admin, related managers and data fixes then silently miss rows.

**Q15. Override `save()` or use a signal?**
Override `save()` for logic that belongs to the model itself (derived fields, slugs, normalizing data): it is explicit and easy to find. Use signals to react to models you do not own (the auth `User`) or to decouple apps. Neither runs for `update()` or `bulk_create()`, and side effects such as emails should run through `transaction.on_commit` or a service function, not inside `save()`.
*Follow-up:* Does `save()` validate the model? No, call `full_clean()` explicitly or rely on forms, serializers and database constraints.

**Q16. Abstract base class vs multi-table inheritance vs proxy model?**
An abstract base shares fields and methods without its own table, so each child table gets the columns (`TimeStampedModel`). Multi-table inheritance gives each class its own table, joined one-to-one, which costs a JOIN on every query and is rarely worth it. A proxy model reuses the parent's table and changes only Python behavior: default manager, ordering, methods or admin.
*Follow-up:* An example of a proxy model? `PublishedCourse` with a manager that filters published courses, registered separately in the admin.

**Q17. If serializers and forms already validate, why add database constraints?**
Application validation only runs on code paths that use it; scripts, the shell, bulk operations, other services and concurrent requests bypass it. `UniqueConstraint` and `CheckConstraint` are enforced by the database for every write, and they close race conditions that a "check then insert" in Python cannot. Keep validation for friendly error messages and constraints for guarantees.
*Follow-up:* What error does a violated constraint raise? `IntegrityError`, which you catch and translate into a 400 or 409 response.

## Further reading
- [Models](https://docs.djangoproject.com/en/stable/topics/db/models/)
- [Making queries](https://docs.djangoproject.com/en/stable/topics/db/queries/)
- [QuerySet API reference](https://docs.djangoproject.com/en/stable/ref/models/querysets/)
- [Migrations](https://docs.djangoproject.com/en/stable/topics/migrations/)
- [Model field reference](https://docs.djangoproject.com/en/stable/ref/models/fields/)
