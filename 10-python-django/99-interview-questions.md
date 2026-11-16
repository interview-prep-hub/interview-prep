# Django and DRF: Rapid-Fire Interview Questions

> How to use: cover the answer, say yours out loud, then check. Deeper explanations live in the topic files linked under each section.

Questions run from basics to advanced. The last section is "spot the bug": read the snippet, find the problem, then name the fix.

## Fundamentals (see [01-django-fundamentals.md](01-django-fundamentals.md))

**Q1. What is Django?**
A batteries-included Python web framework: ORM and migrations, URL routing, templates, forms, authentication, the admin, sessions, caching and security middleware are built in and designed to work together.

**Q2. Explain MTV and how it maps to MVC.**
Model is data and rules, Template is presentation, View handles the request (the MVC controller role), and the URL dispatcher routes requests to views. A Django view decides what to show; the template decides how it looks.

**Q3. Describe the request lifecycle.**
The WSGI or ASGI server passes the request to Django's handler, which builds an `HttpRequest`, runs middleware top to bottom, resolves the URL to a view, calls it, then runs middleware bottom to top on the `HttpResponse` before the server sends it.

**Q4. WSGI vs ASGI?**
WSGI is the synchronous server interface (Gunicorn). ASGI adds async and long-lived connections (Uvicorn, Daphne), needed for async views to pay off and for WebSockets.

**Q5. Project vs app?**
The project holds settings, the root URLconf and the WSGI/ASGI entry points. An app is a feature package (models, views, URLs, migrations, admin) activated by listing it in `INSTALLED_APPS`.

**Q6. What does `DEBUG = True` do, and why is it dangerous in production?**
It shows detailed error pages with settings, SQL and local variables, and stores every query in memory. In production that leaks secrets and grows memory.

**Q7. What is `ALLOWED_HOSTS` for?**
It lists the host names Django will serve; other `Host` headers get a 400. It blocks host header attacks such as poisoned password reset links, and it must be set when `DEBUG = False`.

**Q8. What is `SECRET_KEY` used for?**
Cryptographic signing: password reset tokens, signed cookies and messages, CSRF secrets and anything built on `django.core.signing`. Keep it out of git and rotate it if it leaks.

**Q9. How do you handle settings for different environments?**
Read environment variables (with `os.environ` or django-environ) in one settings module, or split `base`, `dev` and `prod` modules chosen by `DJANGO_SETTINGS_MODULE`. Secrets never live in the repository.

**Q10. What are URL namespaces and `reverse()`?**
`app_name` in an app's `urls.py` namespaces its route names (`courses:detail`). `reverse()` in Python and `{% url %}` in templates build URLs from names, so changing a path does not break links.

**Q11. Function-based or class-based views?**
FBVs are explicit and suit custom flows; CBVs remove boilerplate for standard CRUD through generic views and mixins, at the cost of spreading logic across methods.

**Q12. What does `as_view()` do?**
It returns a function that, for each request, instantiates the view class, calls `setup()` and then `dispatch()`, which routes to `get()`, `post()` and so on (405 if the method is missing).

**Q13. Why must `LoginRequiredMixin` be the leftmost base class?**
Python resolves methods left to right through the MRO, so the mixin's `dispatch()` must come first to reject anonymous users before the view runs.

**Q14. What is template autoescaping?**
Every `{{ }}` output has HTML special characters escaped, which prevents XSS. The `safe` filter, `mark_safe()` and `{% autoescape off %}` disable it, so use them only for trusted HTML.

**Q15. How does form validation work?**
`is_valid()` runs field conversion and validators, then each `clean_<field>()`, then `clean()` for cross-field rules. ModelForms also validate the model, including unique checks. Results land in `cleaned_data` or `errors`.

**Q16. What does `form.save(commit=False)` give you?**
An unsaved model instance, so you can set server-side fields (owner, status) before calling `save()`.

**Q17. What is CSRF, and how does Django stop it?**
A malicious site makes the victim's browser send a state-changing request with the victim's cookies. `CsrfViewMiddleware` requires a secret token (`{% csrf_token %}` or the `X-CSRFToken` header) on unsafe methods and checks `Origin`/`Referer` on HTTPS.

**Q18. Static files vs media files?**
Static files ship with the code (CSS, JS) and are gathered by `collectstatic`; media files are user uploads saved through a storage backend, typically S3 in production.

**Q19. What is the Django admin for?**
An auto-generated CRUD interface for staff, customized with `ModelAdmin` (`list_display`, `list_filter`, `search_fields`, `list_select_related`, inlines, actions). It is an internal tool, not a customer UI.

**Q20. Why create a custom user model at the start of a project?**
Foreign keys and initial migrations across apps point at `AUTH_USER_MODEL`, so swapping it later means hand-written migrations. Subclass `AbstractUser` from day one, reference it with `settings.AUTH_USER_MODEL` in fields and `get_user_model()` in code.

**Q21. `AbstractUser` vs `AbstractBaseUser`?**
`AbstractUser` keeps Django's fields (username, email, names, permissions) and lets you add more. `AbstractBaseUser` plus `PermissionsMixin` lets you define everything, such as email-only login, and needs a custom manager.

**Q22. How do sessions work?**
`SessionMiddleware` loads `request.session` using the `sessionid` cookie from a backend (database by default). Data is JSON-serialized, `login()` rotates the session key, and `logout()` flushes it.

## Models and migrations (see [02-models-and-orm.md](02-models-and-orm.md))

**Q23. `null=True` vs `blank=True`?**
`null` is the database (the column accepts NULL); `blank` is validation (forms accept empty). Optional text fields use `blank=True` with `default=""`.

**Q24. Why `default=dict` and not `default={}`?**
A literal mutable default would be one object shared by every instance; Django requires a callable for mutable defaults.

**Q25. Name the `on_delete` options.**
`CASCADE`, `PROTECT`, `RESTRICT`, `SET_NULL` (needs `null=True`), `SET_DEFAULT`, `SET(...)` and `DO_NOTHING`.

**Q26. When do you need a `through` model on a ManyToManyField?**
When the link has its own data, such as an enrollment's date, status or progress: `students = ManyToManyField(User, through="Enrollment")`.

**Q27. What does `related_name` do?**
It names the reverse accessor (`course.enrollments.all()`) and the reverse lookup; the default is `<model>_set`. Two foreign keys to the same model need distinct names.

**Q28. `OneToOneField` vs `ForeignKey(unique=True)`?**
The database schema is the same, but a one-to-one's reverse accessor returns a single object instead of a manager, which is why Django recommends `OneToOneField`.

**Q29. `UniqueConstraint` vs `unique_together`?**
Both enforce multi-column uniqueness. `UniqueConstraint` is the modern option and supports conditions (partial uniqueness) and expressions.

**Q30. What is a `CheckConstraint`?**
A row rule enforced by the database, such as a rating between 1 and 5. Django 5.1 renamed its `check=` argument to `condition=`.

**Q31. How do migrations work?**
`makemigrations` rebuilds the historical model state from existing migration files, diffs it against your models and writes new operations; `migrate` applies unapplied migrations in dependency order and records them in `django_migrations`. `sqlmigrate` prints the SQL.

**Q32. How do you roll back a migration?**
`python manage.py migrate app 0004` runs the reverse operations of every later migration, which works only if those operations are reversible.

**Q33. What is a data migration, and why use `apps.get_model()`?**
A migration with `RunPython` or `RunSQL` that changes rows. `apps.get_model()` returns the historical model at that point in the graph, so the migration keeps working after later schema changes. Always add a reverse function.

**Q34. Two branches both added migration `0007`. What now?**
Run `makemigrations --merge`, or before merging, delete your unshared migration and regenerate it on top of the other branch.

**Q35. What does `squashmigrations` do?**
Replaces a range of migrations with one equivalent migration to speed up setup; the originals can be removed once every environment has applied the squash.

**Q36. How do you add a NOT NULL column to a large table safely?**
Add it nullable or with a database default, backfill in batches, then add the constraint in a later migration. Build indexes concurrently on PostgreSQL in a non-atomic migration.

**Q37. Abstract base vs multi-table vs proxy inheritance?**
Abstract base: shared fields, no table of its own. Multi-table: a table per class joined one-to-one. Proxy: the same table with different Python behavior (manager, ordering, admin).

**Q38. Does `save()` call `full_clean()`?**
No. ModelForms and the admin call it; DRF serializers, scripts and the shell do not. Database constraints are the real guarantee.

**Q39. Override `save()` or use a signal?**
`save()` for logic about the model itself (slugs, derived fields); signals for reacting to models you do not own or decoupling apps. Neither runs for `update()` or `bulk_create()`.

**Q40. What is a custom QuerySet method?**
A chainable, named filter (`published()`, `taught_by(user)`) on a `QuerySet` subclass exposed with `as_manager()`, similar to Rails scopes.

## QuerySets and ORM performance (see [02-models-and-orm.md](02-models-and-orm.md) and [03-orm-performance.md](03-orm-performance.md))

**Q41. What does it mean that QuerySets are lazy?**
Building and chaining only composes SQL; iteration, `list()`, `len()`, `bool()`, `repr()` and stepped slicing run it. An evaluated QuerySet caches its rows.

**Q42. `get()` vs `filter()` vs `first()`?**
`get()` returns exactly one object or raises `DoesNotExist`/`MultipleObjectsReturned`; `filter()` returns a QuerySet; `first()` returns an object or `None`.

**Q43. `exists()` vs `count()` vs `len()`?**
`exists()` answers yes/no with `LIMIT 1`, `count()` runs `COUNT(*)`, and `len()` loads every row, which is only right if you will iterate them anyway.

**Q44. QuerySet `update()` vs calling `save()` in a loop?**
`update()` runs one `UPDATE` and skips `save()` and signals; a `save()` loop runs model logic but costs one query per row.

**Q45. What does `bulk_create()` skip?**
`save()`, `pre_save`/`post_save` signals and per-object validation. It batches inserts (`batch_size`) and can ignore conflicts or upsert (`update_conflicts`, 4.1+).

**Q46. What is the N+1 problem in Django?**
One query loads a list, then accessing a relation on each object (`course.instructor`) runs one more query per row.

**Q47. `select_related` vs `prefetch_related`?**
`select_related` follows forward FK and one-to-one relations with SQL JOINs; `prefetch_related` runs one extra `IN (...)` query per relation for reverse FK and M2M and joins in Python.

**Q48. What is a `Prefetch` object?**
It customizes the prefetched queryset (filter, order, nested `select_related`) and can store results in a list attribute with `to_attr`.

**Q49. Why does `course.reviews.filter(rating=5)` re-query after `prefetch_related("reviews")`?**
The prefetch cache only serves `.all()`; any new filter builds a new query. Prefetch the filtered set with `Prefetch(..., to_attr=...)` instead.

**Q50. How do you find N+1 queries?**
django-debug-toolbar's SQL panel, SQL logging on `django.db.backends` with `DEBUG = True`, `connection.queries`, and `assertNumQueries` in tests.

**Q51. `only()` vs `defer()`?**
`only()` loads just the listed columns and `defer()` loads all but the listed ones. Touching a deferred field later costs one query per object.

**Q52. `annotate()` vs `aggregate()`?**
`annotate()` adds a computed value to each row and returns a QuerySet; `aggregate()` collapses the whole QuerySet into one dict.

**Q53. Why can two `Count` annotations return inflated numbers?**
Each multi-valued relation adds a JOIN and the rows multiply. Use `distinct=True` or compute aggregates in subqueries.

**Q54. What is an F expression?**
A reference to a column resolved in SQL. `update(stock=F("stock") - 1)` is atomic and prevents lost updates; F also compares two columns in a filter.

**Q55. What is a Q object?**
A wrapper for conditions that combine with `|` (OR), `&` (AND) and `~` (NOT), useful for dynamic filters built from request parameters.

**Q56. What are `Subquery` and `OuterRef`?**
Correlated subqueries: `OuterRef("pk")` points at the outer row, for example annotating each course with its latest enrollment date.

**Q57. When do you use `Exists`?**
For yes/no annotations or filters such as `is_enrolled` on a course list; it stops at the first match and never duplicates rows.

**Q58. How do you do conditional aggregation?**
Pass a filter to the aggregate: `Count("enrollments", filter=Q(enrollments__completed_at__isnull=False))`.

**Q59. What is `iterator()` for?**
Streaming a large QuerySet in chunks without caching every row; with `prefetch_related` it needs `chunk_size` (4.1+).

**Q60. How do you run raw SQL safely?**
`Model.objects.raw(sql, params)` or `cursor.execute(sql, params)` with `%s` placeholders. Never build SQL with f-strings.

**Q61. How do you inspect a slow query?**
Print `queryset.query` to see the SQL and run `queryset.explain()` (with `analyze=True` on PostgreSQL) to read the plan, then add indexes that match real filters.

**Q62. What do `CONN_MAX_AGE` and pooling do?**
`CONN_MAX_AGE` reuses a worker's connection across requests (0 closes it each time). PgBouncer, or the native PostgreSQL pooling added in Django 5.1, shares connections across many processes.

## Transactions and concurrency (see [03-orm-performance.md](03-orm-performance.md))

**Q63. What is Django's default transaction behavior?**
Autocommit: each query commits on its own unless it runs inside `transaction.atomic()` (or `ATOMIC_REQUESTS = True` wraps the whole view).

**Q64. What does `transaction.atomic()` do?**
It wraps a block in a transaction that commits on success and rolls back when an exception escapes. A nested `atomic()` creates a savepoint.

**Q65. Why do you get `TransactionManagementError` after catching an `IntegrityError`?**
The database error left the transaction unusable. Wrap the risky statement in a nested `atomic()` (a savepoint) and catch the exception outside that inner block.

**Q66. What is `transaction.on_commit()`?**
It registers a callback that runs after the outermost transaction commits and is discarded on rollback, the right place to enqueue tasks and send emails.

**Q67. What does `select_for_update()` do?**
It adds `FOR UPDATE`, locking the selected rows until the transaction ends. It must run inside `atomic()`; `nowait`, `skip_locked` and `of` adjust the behavior.

**Q68. What is `skip_locked=True` for?**
Work queues: each worker selects rows that nobody else has locked, so several workers drain a table without blocking each other.

**Q69. How do you stop overselling seats without explicit locks?**
A conditional update: `filter(pk=pk, seats_taken__lt=F("capacity")).update(seats_taken=F("seats_taken") + 1)`; a row count of 0 means the course is full.

## Django REST Framework (see [04-django-rest-framework.md](04-django-rest-framework.md))

**Q70. Why use DRF?**
It standardizes API work: serializers with validation, authentication, permissions and throttling, generic views and viewsets, routers, pagination, filtering, the browsable API and OpenAPI support.

**Q71. `Serializer` vs `ModelSerializer`?**
`Serializer` declares fields and `create()`/`update()` by hand; `ModelSerializer` generates fields, validators and default `create()`/`update()` from a model.

**Q72. Where does validation go in a serializer?**
Field arguments and `validators=` for single values, `validate_<field>()` for one field's rules, `validate()` for cross-field rules, plus model-derived unique validators.

**Q73. Does DRF run the model's `clean()`?**
No. Put rules in the serializer, and keep database constraints as the final guarantee.

**Q74. `read_only` vs `write_only` fields?**
Read-only fields appear only in responses (ids, timestamps, computed values); write-only fields are accepted only as input (passwords).

**Q75. What does `serializer.save()` do?**
Calls `create()` when no instance was passed and `update()` when one was, merging extra keyword arguments such as `save(owner=request.user)` into `validated_data`.

**Q76. How do you make a nested serializer writable?**
Override `create()` and `update()`, pop the nested data and write the children yourself inside a transaction, or expose separate child endpoints instead.

**Q77. What is the main risk of `SerializerMethodField`?**
It runs once per object, so a query inside it becomes an N+1. Precompute values with annotations or `Exists` in `get_queryset()`.

**Q78. `APIView` vs generic views vs viewsets?**
`APIView` has explicit `get()`/`post()` methods; generic views combine mixins with `GenericAPIView` for one URL pattern; viewsets group all actions of a resource and are wired by routers.

**Q79. What does `ModelViewSet` provide?**
`list`, `create`, `retrieve`, `update`, `partial_update` and `destroy`. Use `ReadOnlyModelViewSet` or `GenericViewSet` plus chosen mixins for fewer actions.

**Q80. What does `@action` do?**
Adds a custom route to a viewset: `detail=True` gives `/courses/{pk}/enroll/`, `detail=False` gives a collection route such as `/courses/featured/`.

**Q81. What do routers do?**
Generate URL patterns mapping HTTP methods to viewset actions and `@action` methods. `DefaultRouter` adds an API root, and `basename` is required when the viewset has no `queryset` attribute.

**Q82. Why override `perform_create()`?**
To set server-controlled fields (`serializer.save(instructor=self.request.user)`) or trigger side effects while DRF still handles validation and the 201 response.

**Q83. Why prefer `get_queryset()` over a `queryset` attribute?**
It runs per request, so it can scope rows to the current user or action and add `select_related`/`prefetch_related`.

**Q84. 401 vs 403?**
401: not authenticated (missing or invalid credentials). 403: authenticated but not allowed, and also returned for anonymous users under session auth, which has no challenge header.

**Q85. How do you give all API errors the same shape?**
Point `EXCEPTION_HANDLER` at a function that calls DRF's default `exception_handler` and reshapes `response.data`; raise custom `APIException` subclasses for domain errors.

**Q86. `request.data` vs `request.query_params`?**
`request.data` is the parsed body (JSON, form or multipart); `request.query_params` is the query string. Django's `request.POST` is empty for JSON bodies.

**Q87. How do you test DRF endpoints?**
`APITestCase` or pytest with `APIClient`, `force_authenticate()`, `format="json"`, and assertions on status codes and `response.data`, covering every role per endpoint.

## DRF authentication, permissions and advanced topics (see [05-drf-auth-permissions-advanced.md](05-drf-auth-permissions-advanced.md))

**Q88. Authentication vs permissions in DRF?**
Authentication classes identify the caller and set `request.user`; permission classes decide whether that caller may proceed; throttles then limit the rate.

**Q89. Session vs Token vs JWT authentication?**
Session: cookie plus CSRF, best for same-site apps. Token: one opaque database token per user with no expiry. JWT via simplejwt: short-lived signed access tokens plus refresh tokens, stateless.

**Q90. How does JWT authentication work with simplejwt?**
`TokenObtainPairView` exchanges credentials for an access and a refresh token; clients send `Authorization: Bearer <access>`; `TokenRefreshView` trades the refresh token for a new access token, with optional rotation and blacklisting.

**Q91. How do you revoke JWTs?**
Keep access tokens short-lived, blacklist refresh tokens with the `token_blacklist` app, and rotate refresh tokens on every use.

**Q92. Is a JWT's payload secret?**
No. It is base64url-encoded and signed, not encrypted, so never put sensitive data in it.

**Q93. `has_permission` vs `has_object_permission`?**
`has_permission` runs on every request; `has_object_permission` runs only when `get_object()` loads a single object, never on list endpoints.

**Q94. How do you implement role-based permissions?**
`BasePermission` subclasses that check `request.user.role` (`IsStudent`, `IsInstructor`), chosen per action in `get_permissions()`, plus ownership object permissions and querysets scoped per role.

**Q95. How do you combine permission classes?**
A list means all must pass, and the `&`, `|` and `~` operators build expressions such as `[IsAuthenticated & (IsInstructor | IsAdminUser)]`.

**Q96. What is IDOR, and how do you prevent it in DRF?**
Insecure direct object reference: changing an ID exposes someone else's data. Scope `get_queryset()` to the caller, set owners server-side, validate related IDs in serializers, and return 404 for foreign objects.

**Q97. How does throttling work?**
Throttle classes count requests per IP, user or scope in Django's cache and return 429 with `Retry-After` when a rate is exceeded. Use a shared cache such as Redis across processes.

**Q98. PageNumber vs LimitOffset vs Cursor pagination?**
PageNumber and LimitOffset give totals and page jumps but slow down on deep pages; Cursor uses an opaque cursor on an indexed ordering, stays fast and stable, and has no total count.

**Q99. How do you add filtering, search and ordering?**
`DjangoFilterBackend` with `filterset_fields` or a `FilterSet`, `SearchFilter` with `search_fields`, and `OrderingFilter` with a whitelisted `ordering_fields`.

**Q100. How do you version a DRF API?**
Set `DEFAULT_VERSIONING_CLASS` (URL path `/api/v1/` is the most common) and branch on `request.version`; create a new version only for breaking changes.

**Q101. How do you configure CORS for a React frontend, and does it protect the API?**
django-cors-headers with `CorsMiddleware` near the top, explicit `CORS_ALLOWED_ORIGINS`, and `CORS_ALLOW_CREDENTIALS` only for cookies. It does not protect the API: only browsers enforce it.

**Q102. Where should a React SPA keep its tokens?**
Commonly the access token in memory and the refresh token in an httpOnly, Secure, SameSite cookie. `localStorage` is readable by any XSS payload.

**Q103. How do you document a DRF API?**
drf-spectacular generates an OpenAPI schema with Swagger UI and Redoc views; `@extend_schema` fills gaps, and TypeScript types for the frontend can be generated from the schema.

## Advanced Django (see [06-django-advanced.md](06-django-advanced.md) and [07-lms-project-patterns.md](07-lms-project-patterns.md))

**Q104. What is middleware, and what are the ordering rules?**
A wrapper around `get_response` that runs on every request: top to bottom on the way in, bottom to top on the way out. `SecurityMiddleware` goes first, sessions before authentication, CORS near the top.

**Q105. What are the pros and cons of signals?**
Pros: decoupling and reacting to models you do not own. Cons: hidden control flow, skipped by bulk operations, synchronous inside the caller's transaction, harder to test.

**Q106. What caching tools does Django offer?**
Per-site cache middleware, `@cache_page`, `{% cache %}` fragments and the low-level API (`get`, `set`, `get_or_set`, `delete`), with a Redis backend built in since 4.0.

**Q107. How does Celery fit into a Django project?**
Code publishes task messages to a broker (Redis or RabbitMQ); worker processes execute them; celery beat schedules periodic tasks; a result backend is optional.

**Q108. Why must Celery tasks be idempotent?**
Delivery is at least once: retries, redelivery after a worker crash with `acks_late`, and visibility timeouts can run a task twice. Guard with state checks, unique constraints, conditional updates and idempotency keys.

**Q109. Why enqueue Celery tasks with `transaction.on_commit()`?**
A task enqueued inside a transaction can start before the commit (and not find the row) or act on data that is later rolled back.

**Q110. Does Django support async?**
Yes: async views (3.1+), async ORM methods such as `aget` and `acount` (4.1+), and ASGI deployment. Transactions are sync-only, the ORM still uses sync drivers in threads, and DRF views are synchronous.

**Q111. `TestCase` vs `TransactionTestCase`?**
`TestCase` rolls back a transaction after each test (fast, nothing commits); `TransactionTestCase` truncates tables (slow) and allows real commits for transaction-dependent code.

**Q112. What security features does Django provide?**
CSRF protection, template autoescaping, parameterized ORM queries, clickjacking protection, password hashing, `ALLOWED_HOSTS`, secure cookie and HSTS settings, and `manage.py check --deploy` to audit them.

## Spot the bug

**Q113. What is wrong with this viewset?**
```python
class EnrollmentViewSet(viewsets.ModelViewSet):
    queryset = Enrollment.objects.all()
    serializer_class = EnrollmentSerializer
    permission_classes = [IsAuthenticated]
```
An IDOR: any logged-in user can list, read, change or delete everyone's enrollments. Scope `get_queryset()` to `student=self.request.user` (per role), set the student in `perform_create()`, and limit the actions the viewset exposes.

**Q114. What is wrong here?**
```python
def complete_course(enrollment):
    with transaction.atomic():
        enrollment.completed_at = timezone.now()
        enrollment.save()
        issue_certificate.delay(enrollment.pk)
```
The task is enqueued before the transaction commits, so a worker may read the old row or act on a rollback. Use `transaction.on_commit(lambda: issue_certificate.delay(enrollment.pk))`.

**Q115. Why does this lose enrollments under load?**
```python
course = Course.objects.get(pk=pk)
course.enrolled_count += 1
course.save()
```
Two requests read the same value and both write `n + 1` (a lost update), and `save()` also rewrites every other column. Use `Course.objects.filter(pk=pk).update(enrolled_count=F("enrolled_count") + 1)`.

**Q116. How many queries does this run for 50 courses?**
```python
data = [
    {"title": c.title, "instructor": c.instructor.username, "modules": c.modules.count()}
    for c in Course.objects.all()
]
```
101: one for the courses, one per instructor and one count per course. Use `Course.objects.select_related("instructor").annotate(module_count=Count("modules"))` for a single query.

**Q117. What is risky about this serializer?**
```python
class CourseSerializer(serializers.ModelSerializer):
    class Meta:
        model = Course
        fields = "__all__"
```
Every column is exposed and writable, so a client can set `instructor`, `status` or other internal fields (mass assignment). List fields explicitly, mark server-controlled ones read-only and set the owner in `perform_create()`.

**Q118. Every user gets the same email. Why?**
```python
for user in new_users:
    transaction.on_commit(lambda: send_welcome_email.delay(user.id))
```
Late binding: each lambda looks up `user` when it runs after the commit, by which time the loop has finished, so all callbacks use the last user. Bind the value: `lambda uid=user.id: send_welcome_email.delay(uid)` or `functools.partial(send_welcome_email.delay, user.id)`.

**Q119. Lesson detail is protected, yet students see every lesson. Why?**
```python
class LessonViewSet(viewsets.ReadOnlyModelViewSet):
    queryset = Lesson.objects.all()
    serializer_class = LessonSerializer
    permission_classes = [IsAuthenticated, IsEnrolledInCourse]   # only has_object_permission
```
`has_object_permission` runs only for detail actions; the `list` action returns every lesson. Filter `get_queryset()` to lessons of courses the user is enrolled in, and keep the object check for detail views.

**Q120. What can go wrong with this task?**
```python
@shared_task(autoretry_for=(Exception,), max_retries=5)
def charge_monthly_fee(user_id):
    user = User.objects.get(pk=user_id)
    payments.charge(customer_id=user.payment_customer_id, amount_cents=1000)
    Invoice.objects.create(user=user, amount_cents=1000)
```
It is not idempotent: if the charge succeeds and the invoice insert (or the worker) fails, the retry charges again, and retrying on every `Exception` repeats permanent errors too. Record the attempt first with a unique key, pass an idempotency key to the payment provider, retry only transient errors, and skip work that is already done.
