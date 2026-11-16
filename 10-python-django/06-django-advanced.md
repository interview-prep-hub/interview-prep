# Advanced Django

> **Tier:** T1 (know cold) · **Related:** [Django fundamentals](01-django-fundamentals.md), [ORM performance](03-orm-performance.md), [DRF auth and permissions](05-drf-auth-permissions-advanced.md), [LMS project patterns](07-lms-project-patterns.md), [Django rapid-fire questions](99-interview-questions.md), [Background processing patterns](../13-architecture-and-practices/06-background-processing-patterns.md), [Redis](../12-databases/10-redis.md), [Sidekiq](../05-ruby-on-rails/09-background-jobs-sidekiq.md), [Docker](../14-devops-and-tools/03-docker.md)
> **On your resume:** The production side of the PureLogics LMS (emails, certificates and payment webhooks handled in the background, cached catalog data, tests, secure settings, deployment), plus the background job experience you bring from Sidekiq on Colochousing and the sync jobs on CrystalCommerce at Devntech.

## Quick revision
- Middleware wraps `get_response`: code before the call runs on the way in (top to bottom), code after runs on the way out (bottom to top). Optional hooks: `process_view`, `process_exception`, `process_template_response`.
- Order matters: `SecurityMiddleware` first, `SessionMiddleware` before `AuthenticationMiddleware`, `CsrfViewMiddleware` before views that rely on it, CORS near the top, WhiteNoise right after `SecurityMiddleware`.
- Signals (`post_save`, `pre_delete`, `m2m_changed`, `user_logged_in`) decouple senders from receivers. Connect them in `AppConfig.ready()`, and avoid them for core business flow: they are hidden, skipped by bulk operations and run inside the caller's transaction.
- Caching levels: per-site middleware, per-view `@cache_page`, template fragments `{% cache %}`, and the low-level API (`cache.get`, `set`, `get_or_set`, `delete`). A Redis backend is built in since Django 4.0.
- Invalidate with TTLs, explicit deletes on writes or versioned keys, and put the user or locale in the key for personalized data.
- Celery: a broker (Redis, RabbitMQ) holds task messages, workers execute them, a result backend is optional. `@shared_task`, `.delay()`, `.apply_async(countdown=...)`, `autoretry_for` with `retry_backoff`, and celery beat for schedules.
- Tasks must be idempotent because delivery is at least once and retries happen: pass IDs, re-read state, guard with status checks and unique constraints.
- Enqueue from inside transactions with `transaction.on_commit(lambda: task.delay(obj.pk))`.
- Django 6.0 added a built-in tasks framework; its bundled backends target development and tests, so check the release notes before choosing it over Celery.
- Async views (3.1+) and async ORM methods (`aget`, `acount`, `async for`, 4.1+) help with I/O fan-out under ASGI; transactions and much of the ecosystem (including DRF) stay synchronous.
- Channels adds WebSockets on ASGI: consumers, routing and Redis-backed channel layers for groups.
- `TestCase` wraps each test in a rolled-back transaction (fast); `TransactionTestCase` truncates tables and is needed when code must really commit. Use pytest-django, factory_boy, and mock HTTP at the boundary.
- Secrets live in environment variables (django-environ). Run `manage.py check --deploy` before going live.
- Built-in protections: CSRF, autoescaping, parameterized SQL, clickjacking headers, password hashing, `ALLOWED_HOSTS`. You configure HTTPS settings (`SECURE_SSL_REDIRECT`, secure cookies, HSTS, `SECURE_PROXY_SSL_HEADER`).
- Deploy: Gunicorn (WSGI) or Uvicorn (ASGI) behind Nginx, WhiteNoise or a CDN after `collectstatic`, Docker images, migrations as a release step, logs to stdout.
- Custom management commands live in `app/management/commands/<name>.py` and subclass `BaseCommand`.

## 1. Custom middleware and ordering

```python
# core/middleware.py
import logging
import time
import uuid

logger = logging.getLogger(__name__)

class RequestTimingMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response              # runs once, when the server starts

    def __call__(self, request):
        request.request_id = request.headers.get("X-Request-ID", uuid.uuid4().hex)   # before the view
        start = time.perf_counter()
        response = self.get_response(request)         # the next middleware, and eventually the view
        duration_ms = (time.perf_counter() - start) * 1000
        response["X-Request-ID"] = request.request_id                                # after the view
        logger.info("%s %s %s %.1fms", request.method, request.path, response.status_code, duration_ms)
        return response

    def process_exception(self, request, exception):  # optional hook: the view raised
        logger.error("unhandled error on %s", request.path, exc_info=exception)
        return None                                   # None lets Django's normal 500 handling continue
```

```python
MIDDLEWARE = [
    "core.middleware.RequestTimingMiddleware",                  # first, so it times the whole stack
    "corsheaders.middleware.CorsMiddleware",                    # before anything that can return early
    "django.middleware.security.SecurityMiddleware",            # HTTPS redirect, HSTS, security headers
    "whitenoise.middleware.WhiteNoiseMiddleware",               # serve static files early and cheaply
    "django.contrib.sessions.middleware.SessionMiddleware",
    "django.middleware.common.CommonMiddleware",
    "django.middleware.csrf.CsrfViewMiddleware",
    "django.contrib.auth.middleware.AuthenticationMiddleware",  # needs the session
    "django.contrib.messages.middleware.MessageMiddleware",
    "django.middleware.clickjacking.XFrameOptionsMiddleware",
]
```

On the way in, middleware runs top to bottom, and any of them can return a response early (a redirect, a 403), which skips everything below it. On the way out, the response passes back bottom to top. Use middleware for cross-cutting concerns that apply to every request (request IDs, timing, security headers, tenant resolution, maintenance mode), and keep it fast because it runs for every request. Per-view rules belong in decorators, mixins or DRF permissions. Under ASGI, sync-only middleware forces thread switches, so prefer middleware that supports async.

## 2. Signals, and when to avoid them

```python
# accounts/signals.py
from django.conf import settings
from django.db import transaction
from django.db.models.signals import post_save
from django.dispatch import receiver

from .models import Profile
from .tasks import send_welcome_email

@receiver(post_save, sender=settings.AUTH_USER_MODEL)
def on_user_created(sender, instance, created, **kwargs):
    if not created:
        return
    Profile.objects.create(user=instance)
    transaction.on_commit(lambda: send_welcome_email.delay(instance.pk))

# accounts/apps.py
from django.apps import AppConfig

class AccountsConfig(AppConfig):
    name = "accounts"

    def ready(self):
        from . import signals  # noqa: F401  (importing the module registers the receivers)
```

| Pros | Cons |
|---|---|
| Decouple apps: react to models you do not own (the auth `User`, third-party apps) | Hidden control flow: reading the model does not reveal the side effects |
| Several independent receivers for one event | Not sent by `update()`, `bulk_create()`, `bulk_update()` or raw SQL |
| Framework hooks such as `user_logged_in`, `request_finished`, `post_migrate` | Receivers run synchronously inside the caller's transaction; a slow or failing receiver slows or breaks the save |
| | Hard to test and reason about; a receiver that saves the same model can recurse |

Rule of thumb: core business flows (enrolling, paying, issuing a certificate) go in explicit service functions that you can read top to bottom and test directly; signals are for cross-app notifications and integrations. Pass `dispatch_uid` to `connect()` if a module might be imported twice.

## 3. Caching

```python
# config/settings.py
CACHES = {
    "default": {
        "BACKEND": "django.core.cache.backends.redis.RedisCache",    # built in since Django 4.0
        "LOCATION": os.environ.get("REDIS_URL", "redis://127.0.0.1:6379/1"),
        "TIMEOUT": 300,                                              # default TTL in seconds
    }
}
```

```python
from django.core.cache import cache
from django.utils.decorators import method_decorator
from django.views.decorators.cache import cache_page

@cache_page(60 * 15)                        # whole response for 15 minutes, keyed by URL
def course_catalog(request): ...

class CategoryViewSet(viewsets.ReadOnlyModelViewSet):
    @method_decorator(cache_page(60 * 60))  # public, identical for everyone: safe to cache
    def list(self, request, *args, **kwargs):
        return super().list(request, *args, **kwargs)

def course_stats(course_id):                # low-level cache-aside
    key = f"course:{course_id}:stats:v1"
    return cache.get_or_set(key, lambda: compute_course_stats(course_id), timeout=600)

def invalidate_course_stats(course_id):     # call it whenever reviews or enrollments change
    cache.delete(f"course:{course_id}:stats:v1")
```

```html
{% load cache %}
{% cache 600 course_curriculum course.pk %}
  ... expensive curriculum markup ...
{% endcache %}
```

- **Levels**: the per-site cache middleware caches every anonymous GET; `@cache_page` caches one view's response; `{% cache %}` caches part of a template; the low-level API caches any computed value.
- **Invalidation**: short TTLs where staleness is acceptable, explicit `delete()` on writes for data that must be fresh, and a version segment in the key (`v1`) to invalidate everything at once after a format change.
- **Personalized data**: `cache_page` keys by URL (plus headers listed in `Vary`), so never cache per-user responses without `vary_on_cookie`/`vary_on_headers("Authorization")`, or simply put the user ID in low-level keys.
- **Stampedes**: when a hot key expires, many requests recompute it at once; mitigate with a lock (`cache.add()` as a mutex), staggered TTLs or background refresh.
- The local-memory backend is per process and fine for development; production needs a shared backend (Redis or Memcached). HTTP caching (`Cache-Control`, ETags via `ConditionalGetMiddleware`) and a CDN can remove load before it reaches Django.

## 4. Background work with Celery

Anything slow, unreliable or not needed for the response (emails, PDFs, video processing, webhooks to third parties, report generation) should leave the request cycle. Celery is the standard tool: your code publishes a message to a **broker** (Redis, RabbitMQ, SQS), **worker** processes consume and execute it, and an optional **result backend** stores return values.

```python
# config/celery.py
import os
from celery import Celery

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "config.settings")
app = Celery("config")
app.config_from_object("django.conf:settings", namespace="CELERY")   # reads CELERY_* settings
app.autodiscover_tasks()                                              # finds tasks.py in installed apps

# config/__init__.py
from .celery import app as celery_app
__all__ = ("celery_app",)

# config/settings.py
CELERY_BROKER_URL = os.environ.get("CELERY_BROKER_URL", "redis://127.0.0.1:6379/0")
CELERY_TASK_ACKS_LATE = True        # acknowledge after the task finishes, so a crashed worker's task is redelivered
CELERY_TASK_TIME_LIMIT = 300        # hard limit in seconds
```

```python
# certificates/tasks.py
import requests
from celery import shared_task
from django.utils import timezone

@shared_task(
    bind=True,                                    # gives access to self.request and self.retry()
    autoretry_for=(requests.RequestException,),   # retry automatically on network errors
    retry_backoff=True,                           # exponential backoff (1s, 2s, 4s...) with jitter
    retry_kwargs={"max_retries": 5},
)
def issue_certificate(self, enrollment_id):
    enrollment = Enrollment.objects.select_related("student", "course").get(pk=enrollment_id)
    if enrollment.completed_at is None:
        return "not completed"                    # state changed since enqueue: do nothing
    certificate, _ = Certificate.objects.get_or_create(enrollment=enrollment)   # unique per enrollment
    if certificate.pdf_url:
        return "already issued"                   # a retry or duplicate delivery is harmless
    pdf_url = render_and_upload_pdf(certificate)  # external PDF and storage calls
    Certificate.objects.filter(pk=certificate.pk).update(pdf_url=pdf_url, issued_at=timezone.now())
    return "issued"

# enrollments/services.py
def complete_course(enrollment):
    with transaction.atomic():
        enrollment.completed_at = timezone.now()
        enrollment.save(update_fields=["completed_at", "updated_at"])
        transaction.on_commit(lambda: issue_certificate.delay(enrollment.pk))   # only after COMMIT
```

```python
# config/settings.py: periodic tasks with celery beat
from celery.schedules import crontab

CELERY_BEAT_SCHEDULE = {
    "sync-subscription-status-nightly": {
        "task": "billing.tasks.reconcile_subscriptions",
        "schedule": crontab(hour=2, minute=0),
    },
}
```

```bash
celery -A config worker -l info --concurrency 4
celery -A config beat -l info        # run exactly ONE beat process, or schedules fire twice
```

**Idempotency** is the rule that matters most. Delivery is at least once: with `acks_late` a task whose worker crashed is redelivered, Redis and SQS redeliver tasks that outlive the visibility timeout, and retries rerun the whole function. So a task must be safe to run twice:
- Pass IDs, not model instances, and re-read current state at the start (the row may have changed or been deleted).
- Guard with state checks (`if certificate.pdf_url: return`), unique constraints with `get_or_create`, or conditional updates (`filter(status="pending").update(status="sent")` returns 1 for only one worker).
- Send idempotency keys to external APIs that support them (Stripe accepts an `idempotency_key`), and record processed message or event IDs.

Other essentials: retry only transient errors with backoff and a maximum; set time limits; route slow tasks to their own queue so they cannot starve quick ones; keep task arguments JSON-serializable; monitor with Flower or your APM. If you know Sidekiq, the model is the same (at-least-once, idempotent jobs, retries), with Celery's prefork workers using processes where Sidekiq uses threads.

**Django's own tasks framework.** Django 6.0 introduced a built-in tasks API for defining tasks and enqueuing them through a configurable backend. The backends shipped with Django are meant for development and testing, so real background execution needs a production backend and worker, for example from a third-party package. It is new: read the 6.0 release notes and docs before choosing it, and expect Celery to stay common in existing codebases.

## 5. Async views, ASGI and Channels

```python
import asyncio
import httpx
from django.http import JsonResponse

async def instructor_overview(request):
    user = await request.auser()                  # 5.0+; plain request.user would query synchronously
    async with httpx.AsyncClient(timeout=5) as client:
        payouts, analytics = await asyncio.gather(  # two slow external calls run concurrently
            client.get(f"{PAYOUTS_URL}/instructors/{user.pk}"),
            client.get(f"{ANALYTICS_URL}/instructors/{user.pk}"),
        )
    course_count = await Course.objects.filter(instructor=user).acount()   # async ORM method (4.1+)
    return JsonResponse({"payouts": payouts.json(), "analytics": analytics.json(), "courses": course_count})
```

- `async def` views work since Django 3.1, but they only improve concurrency when served by an ASGI server (Uvicorn, Daphne, Hypercorn). Under WSGI they run, yet each request still occupies a worker.
- Since 4.1 the ORM has async methods (`aget`, `acreate`, `afirst`, `acount`, `aexists`, `adelete`, `async for` over a QuerySet). Internally they still use synchronous drivers in a thread, and transactions are not supported in async code; wrap such code in `asgiref.sync.sync_to_async`.
- Calling the synchronous ORM directly inside an event loop raises `SynchronousOnlyOperation`.
- DRF views are synchronous, so async gains in a DRF project usually come from separate async views or background jobs. Use async for I/O fan-out, streaming and long-lived connections, not for CPU work or ordinary CRUD.

**Channels** extends Django to WebSockets and other long-lived protocols on ASGI:

```python
# notifications/consumers.py
from channels.generic.websocket import AsyncJsonWebsocketConsumer

class NotificationConsumer(AsyncJsonWebsocketConsumer):
    async def connect(self):
        user = self.scope["user"]                     # filled in by AuthMiddlewareStack
        if not user.is_authenticated:
            await self.close()
            return
        self.group = f"user_{user.pk}"
        await self.channel_layer.group_add(self.group, self.channel_name)
        await self.accept()

    async def disconnect(self, code):
        if hasattr(self, "group"):
            await self.channel_layer.group_discard(self.group, self.channel_name)

    async def notify(self, event):                    # handles group messages with {"type": "notify"}
        await self.send_json(event["payload"])
```

The ASGI app routes `"websocket"` traffic through `AuthMiddlewareStack(URLRouter(...))` inside a `ProtocolTypeRouter`, and a Redis channel layer (`channels_redis`) lets any process, including a Celery task, push to a group with `async_to_sync(get_channel_layer().group_send)(group, {"type": "notify", "payload": {...}})`. Typical LMS uses: live notifications, progress updates while a video is processed, or chat in a live class. For simple one-way updates, server-sent events or polling may be enough (see [realtime and WebSockets](../04-web-fundamentals/09-realtime-websockets.md)).

## 6. Testing

| | `SimpleTestCase` | `TestCase` | `TransactionTestCase` | `LiveServerTestCase` |
|---|---|---|---|---|
| Database | not allowed | yes | yes | yes |
| Isolation | none needed | each test runs in a transaction that is rolled back (fast) | tables truncated after each test (slow) | like `TransactionTestCase` plus a running server |
| Use for | pure logic, utilities | almost everything | code that needs real commits: `on_commit` without capturing, `select_for_update` across connections, transaction behavior | browser tests (Selenium, Playwright) |

```python
# conftest.py (pytest-django plus factory_boy)
import factory
import pytest
from rest_framework.test import APIClient

class UserFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = "accounts.User"

    username = factory.Sequence(lambda n: f"user{n}")
    email = factory.LazyAttribute(lambda o: f"{o.username}@example.com")
    role = "student"

class CourseFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = "courses.Course"

    instructor = factory.SubFactory(UserFactory, role="instructor")
    title = factory.Faker("sentence", nb_words=3)
    slug = factory.Sequence(lambda n: f"course-{n}")
    status = "published"

@pytest.fixture
def api_client():
    return APIClient()
```

```python
# tests/test_billing.py
from types import SimpleNamespace
from unittest.mock import patch

import pytest
from django.urls import reverse

@pytest.mark.django_db
def test_student_sees_only_own_enrollments(api_client):
    me, other = UserFactory(), UserFactory()
    course = CourseFactory()
    Enrollment.objects.create(student=me, course=course)
    Enrollment.objects.create(student=other, course=course)

    api_client.force_authenticate(user=me)
    response = api_client.get(reverse("my-enrollment-list"))

    assert response.status_code == 200
    assert len(response.data["results"]) == 1

@pytest.mark.django_db
@patch("billing.services.stripe.checkout.Session.create")      # patch where the code looks it up
def test_checkout_returns_redirect_url(mock_create, api_client):
    mock_create.return_value = SimpleNamespace(id="cs_test_123", url="https://checkout.stripe.com/c/pay/cs_test_123")
    api_client.force_authenticate(user=UserFactory())
    response = api_client.post(reverse("billing-checkout"), {"plan": "monthly"}, format="json")
    assert response.status_code == 201
    mock_create.assert_called_once()
```

- `TestCase` never commits, so `on_commit` callbacks do not fire; wrap the code in `self.captureOnCommitCallbacks(execute=True)` (or use the `django_capture_on_commit_callbacks` fixture in pytest-django).
- Mock external services at your boundary (`unittest.mock.patch`, or the `responses` and `respx` libraries for requests and httpx); never call real payment or email APIs from the test suite. Emails sent through Django land in `django.core.mail.outbox` during tests.
- `setUpTestData` builds class-level data once per test class; factories keep tests readable; `override_settings` changes settings per test; `assertNumQueries` guards against N+1 regressions.
- For Celery, call the task function directly in unit tests; `CELERY_TASK_ALWAYS_EAGER` runs tasks inline but hides serialization and transaction problems, so use it sparingly.

## 7. Settings and secrets

```python
# config/settings.py with django-environ
import environ

env = environ.Env(DEBUG=(bool, False))
environ.Env.read_env(BASE_DIR / ".env")           # local development only; .env is git-ignored

DEBUG = env("DEBUG")
SECRET_KEY = env("SECRET_KEY")
ALLOWED_HOSTS = env.list("ALLOWED_HOSTS", default=["localhost"])
DATABASES = {"default": env.db("DATABASE_URL")}   # postgres://user:password@host:5432/lms
STRIPE_SECRET_KEY = env("STRIPE_SECRET_KEY")
```

Configuration that varies per environment comes from environment variables (12-factor style). Commit a `.env.example` with dummy values, never the real `.env`; in production inject variables from the platform's secret store. Use separate keys per environment (Stripe test keys locally, live keys only in production), rotate anything that leaks, and fail fast at startup when a required variable is missing.

## 8. Security features

| Threat | What Django gives you | What you must do |
|---|---|---|
| CSRF | `CsrfViewMiddleware` and the token | keep it on; `@csrf_exempt` only for signature-verified webhooks |
| XSS | template autoescaping | never apply the `safe` filter or `mark_safe()` to user input; no `dangerouslySetInnerHTML` with user content in React; consider a Content Security Policy |
| SQL injection | parameterized ORM queries | pass params to `raw()` and `cursor.execute()` |
| Clickjacking | `XFrameOptionsMiddleware` (`X_FRAME_OPTIONS = "DENY"` by default) | exempt only pages meant to be embedded |
| Host header attacks | `ALLOWED_HOSTS` validation | list your real domains |
| Session theft | httpOnly session cookie, key rotation on login | HTTPS plus `SESSION_COOKIE_SECURE` |
| Weak passwords | PBKDF2 hashing (Argon2 optional), password validators | keep `AUTH_PASSWORD_VALIDATORS` |
| Mass assignment | explicit form and serializer fields | never `"__all__"` on writable serializers |
| IDOR | nothing automatic | scope querysets per user |

```python
# production settings
SECURE_SSL_REDIRECT = True
SECURE_PROXY_SSL_HEADER = ("HTTP_X_FORWARDED_PROTO", "https")   # only if your proxy sets this header
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
SECURE_HSTS_SECONDS = 31536000          # start with a small value, raise once HTTPS is solid everywhere
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True
```

`python manage.py check --deploy` audits these settings and warns about anything missing. Django 6.0 added built-in Content Security Policy support; older projects use the django-csp package (check the docs for your version). See [web security and OWASP](../04-web-fundamentals/07-web-security-owasp.md) for the underlying attacks.

## 9. Deployment

```text
Browser -> CDN (optional) -> Nginx (TLS, static and media files, buffering, gzip)
        -> Gunicorn workers (WSGI) or Uvicorn workers (ASGI) running Django
        -> PostgreSQL, Redis (cache and Celery broker), object storage (S3)
        -> Celery workers plus exactly one celery beat
```

```bash
python manage.py collectstatic --noinput
python manage.py migrate --noinput                     # a release step, before new code takes traffic
gunicorn config.wsgi:application --workers 3 --bind 0.0.0.0:8000 --timeout 30
uvicorn config.asgi:application --host 0.0.0.0 --port 8000 --workers 3    # ASGI alternative
```

```text
# Dockerfile
FROM python:3.12-slim
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
RUN python manage.py collectstatic --noinput
USER nobody
CMD ["gunicorn", "config.wsgi:application", "--bind", "0.0.0.0:8000", "--workers", "3"]
```

- `runserver` is for development only. Gunicorn manages several worker processes (a common starting point is two per CPU core plus one); Uvicorn serves ASGI for async views and Channels.
- Nginx terminates TLS, buffers slow clients and serves files, so Python workers only run Python. Without Nginx, WhiteNoise serves static files efficiently from the app (`STORAGES["staticfiles"]` set to `whitenoise.storage.CompressedManifestStaticFilesStorage`).
- Building `collectstatic` into the image means settings must import without production secrets; otherwise run it at release time.
- Run migrations once per release (not in every container on startup, where replicas race), keep them backward compatible for rolling deploys, add health checks, send logs to stdout, and track errors with a service such as Sentry. See [Docker](../14-devops-and-tools/03-docker.md) and [CI/CD and deployment](../14-devops-and-tools/04-ci-cd-and-deployment.md).

## 10. Logging

```python
LOGGING = {
    "version": 1,
    "disable_existing_loggers": False,
    "formatters": {"simple": {"format": "{asctime} {levelname} {name} {message}", "style": "{"}},
    "handlers": {"console": {"class": "logging.StreamHandler", "formatter": "simple"}},
    "root": {"handlers": ["console"], "level": "INFO"},
    "loggers": {
        "django.request": {"handlers": ["console"], "level": "ERROR", "propagate": False},   # 5xx with tracebacks
    },
}

logger = logging.getLogger(__name__)
logger.info("enrolled user=%s course=%s", user.pk, course.pk)   # %s args are formatted lazily
```

Log to stdout in containers and let the platform collect it; use structured (JSON) logs and request IDs in production so you can follow one request across services; never log passwords, tokens, card data or unnecessary personal data. Exceptions with full context belong in an error tracker. The `django.db.backends` logger prints SQL in development (with `DEBUG = True`).

## 11. Custom management commands

```python
# marketing/management/commands/deactivate_expired_coupons.py
# (management/ and commands/ both need an __init__.py)
from django.core.management.base import BaseCommand
from django.utils import timezone

from marketing.models import Coupon

class Command(BaseCommand):
    help = "Deactivate coupons whose expiry date has passed."

    def add_arguments(self, parser):
        parser.add_argument("--dry-run", action="store_true", help="Only report what would change.")

    def handle(self, *args, **options):
        expired = Coupon.objects.filter(is_active=True, expires_at__lt=timezone.now())
        if options["dry_run"]:
            self.stdout.write(f"Would deactivate {expired.count()} coupons.")
            return
        updated = expired.update(is_active=False)
        self.stdout.write(self.style.SUCCESS(f"Deactivated {updated} coupons."))
```

Run it with `python manage.py deactivate_expired_coupons --dry-run`, schedule it with cron or a platform scheduler, and test it with `call_command("deactivate_expired_coupons", dry_run=True)`. Commands suit one-off data fixes, backfills, imports and scheduled maintenance; keep the real logic in a service function so a Celery beat task can call the same code.

## Common pitfalls
- Database queries or slow calls in middleware, which then run on every request; `AuthenticationMiddleware` placed before `SessionMiddleware`.
- Core business rules hidden in signals, expecting signals from `update()` or `bulk_create()`, or a `post_save` receiver that saves the same instance and recurses.
- Caching per-user responses under a shared key, caching with no invalidation plan, or relying on the per-process local-memory cache with several workers.
- Enqueueing Celery tasks inside a transaction instead of `transaction.on_commit()`, so workers run before the data exists.
- Passing model instances to tasks, non-idempotent tasks combined with retries or `acks_late`, and running two beat processes.
- Long tasks that outlive the broker's visibility timeout and get delivered twice.
- Using the synchronous ORM or `request.user` inside async views (`SynchronousOnlyOperation`), or expecting async views to help under WSGI.
- Testing `on_commit` behavior inside `TestCase` without capturing callbacks, then wondering why the task never ran.
- Patching the wrong import path in tests, so the real HTTP call still happens.
- `SECURE_SSL_REDIRECT = True` behind a TLS-terminating proxy without `SECURE_PROXY_SSL_HEADER`, causing an infinite redirect loop.
- `runserver` in production, migrations run concurrently by every container at startup, or secrets baked into images.
- Logging tokens, passwords or full request bodies.

## How to talk about this in an interview
- Background jobs are your strongest bridge: at Devntech you work in production codebases that rely on them (Sidekiq on Colochousing, the PriceCharting sync jobs on CrystalCommerce), and the rules transfer directly to Celery: idempotent jobs, retries with backoff, enqueue after commit, pass IDs, monitor queues. Then say what the LMS needed in the background.
- For security, walk through `check --deploy`, HTTPS settings behind a proxy, secrets in environment variables, CSRF for cookie-authenticated endpoints and scoped querysets for IDOR.
- For testing, describe a permission matrix per role, webhook tests with mocked provider calls, and query-count assertions on dashboards.
- > **Fill in:** what ran in the background in the LMS (emails, certificates, webhook processing, video processing) and whether you used Celery, a simpler approach, or synchronous code.
- > **Fill in:** how the LMS was deployed (VPS with Nginx and Gunicorn, Docker, a PaaS) or, if it never reached production, how you ran it for demos.
- > **Fill in:** the testing setup you used (Django `TestCase` or pytest-django, factories, coverage of permissions).

## Interview questions

**Q1. What is middleware in Django, and how do you write one?**
A middleware is a callable built with `get_response` that wraps every request: code before `self.get_response(request)` runs on the way in, code after it runs on the way out, and it can short-circuit by returning a response early. Write a class with `__init__(self, get_response)` and `__call__(self, request)`, optionally adding `process_view`, `process_exception` or `process_template_response`, and register its path in `MIDDLEWARE`.
*Follow-up:* When is middleware the wrong tool? For rules that apply to some views only; use decorators, mixins or DRF permissions instead.

**Q2. Why does middleware order matter? Give examples.**
Requests pass down the list and responses pass back up, and later middleware depends on earlier work. `AuthenticationMiddleware` needs the session, so it must follow `SessionMiddleware`; `SecurityMiddleware` goes first so redirects and headers apply to everything; CORS goes near the top so preflight responses get headers even when another middleware returns early; WhiteNoise sits right after `SecurityMiddleware` to serve static files cheaply.
*Follow-up:* What happens when a middleware returns a response early? The middleware below it and the view never run, but the ones above still process the response.

**Q3. What are signals, and when should you avoid them?**
Signals let receivers react to events (`post_save`, `pre_delete`, `m2m_changed`, `user_logged_in`) without the sender knowing about them, which is useful for decoupled apps and models you do not own. Avoid them for core business flow: the side effects are invisible when reading the code, they are skipped by bulk operations and raw SQL, they run synchronously inside the caller's transaction, and they are harder to test. Prefer explicit service functions.
*Follow-up:* Where do you connect receivers? In `AppConfig.ready()`, by importing the signals module.

**Q4. What caching options does Django provide?**
A cache framework with pluggable backends (Redis built in since 4.0, Memcached, database, local memory) used at several levels: the per-site cache middleware, `@cache_page` per view, `{% cache %}` for template fragments, and the low-level API (`get`, `set`, `get_or_set`, `delete`, `incr`) for any computed value. HTTP caching headers and a CDN add another layer in front of Django.
*Follow-up:* Why not use the local-memory cache in production? Each process has its own copy, so workers disagree and memory is duplicated.

**Q5. How do you keep cached data correct?**
Choose per piece of data: short TTLs where slight staleness is acceptable, explicit deletes or updates when the source data changes, versioned key prefixes to invalidate a whole family at once, and keys that include the user or locale for personalized content. Protect hot keys from stampedes with a lock or early refresh.
*Follow-up:* What is the hardest part? Knowing every write path that should invalidate a key, which is why short TTLs are a useful safety net.

**Q6. How does Celery work with Django?**
The Django process publishes a task message (function name plus JSON arguments) to a broker such as Redis or RabbitMQ; separate worker processes consume messages and run the task code with full Django access; an optional result backend stores results. `config/celery.py` creates the app, reads `CELERY_*` settings and autodiscovers `tasks.py` modules; you call `task.delay(...)` or `apply_async(...)`.
*Follow-up:* Why pass IDs instead of model instances? Arguments are serialized, and the row may change before the worker runs, so the task should re-read fresh state.

**Q7. What makes a task idempotent, and why does it matter?**
Running it twice has the same effect as running it once. Celery delivers at least once (retries, `acks_late` redelivery after a crash, visibility timeouts), so duplicates happen. Achieve idempotency with state checks at the start, unique constraints with `get_or_create`, conditional updates whose row count tells you whether you won, idempotency keys for external APIs, and records of processed events.
*Follow-up:* An example of a non-idempotent task? "Charge the customer" or "add 10 credits" without a guard; a retry charges or credits twice.

**Q8. How do you handle retries and failures in Celery?**
Retry only transient errors (timeouts, 5xx responses) with exponential backoff and jitter and a maximum number of attempts, using `autoretry_for`, `retry_backoff` and `retry_kwargs`, or `self.retry(exc=..., countdown=...)` with `bind=True`. Set time limits, log and alert on final failures, and keep the task idempotent so retries are safe.
*Follow-up:* What should never be retried automatically? Permanent errors such as validation failures or a 404 from a deleted resource.

**Q9. How do you run periodic tasks?**
Celery beat publishes tasks on a schedule (`CELERY_BEAT_SCHEDULE` with `crontab()` or intervals, or database-stored schedules through django-celery-beat), and normal workers execute them. Run exactly one beat process, or every task fires once per beat instance. Simple setups can use cron or a platform scheduler to run management commands.
*Follow-up:* How do you make a nightly job safe to overlap? A lock (database or cache) or a design where a second run finds nothing to do.

**Q10. Why enqueue tasks with `transaction.on_commit()`?**
If you call `delay()` inside a transaction, a fast worker can start before the transaction commits and find no row, or act on data that is later rolled back. `on_commit()` defers the enqueue until after the outermost commit and drops it if the transaction rolls back.
*Follow-up:* What is the remaining gap? A crash between commit and enqueue loses the task; the transactional outbox pattern closes it by storing the message in the same transaction.

**Q11. Does Django support async? What are the limits?**
Yes: async views since 3.1, ASGI deployment, and async ORM methods (`aget`, `acount`, `async for`) since 4.1, which help when a view waits on several slow I/O operations. Limits: real benefits need an ASGI server, the ORM still uses synchronous drivers in threads, transactions are not supported in async code, sync-only middleware adds thread switches, and DRF views are synchronous.
*Follow-up:* What error do you get calling the sync ORM from async code? `SynchronousOnlyOperation`; wrap the call in `sync_to_async`.

**Q12. What is Django Channels for?**
It adds long-lived connections such as WebSockets to Django through ASGI: consumers handle connect, receive and disconnect events, routing maps paths to consumers, and channel layers (usually Redis) let any process broadcast to groups of connections. Typical uses are live notifications, chat and progress updates.
*Follow-up:* How does a Celery task push a message to a browser? Through the channel layer: `async_to_sync(channel_layer.group_send)(group, message)`.

**Q13. `TestCase` vs `TransactionTestCase`?**
`TestCase` wraps each test in a transaction and rolls it back, which is fast and isolates tests, but nothing ever commits. `TransactionTestCase` lets code commit for real and truncates tables after each test, which is slower but needed for testing transaction behavior, `on_commit` without capturing, or locking across connections.
*Follow-up:* How do you test `on_commit` callbacks in `TestCase`? With `self.captureOnCommitCallbacks(execute=True)`.

**Q14. How do you mock external HTTP calls in Django tests?**
Patch the client at the boundary where your code looks it up (`unittest.mock.patch("billing.services.stripe.checkout.Session.create")`), or use libraries that intercept HTTP (`responses` for requests, `respx` for httpx), and assert both your behavior and the outgoing call. Keep real provider calls out of the suite; run them separately against test-mode keys if needed.
*Follow-up:* Why patch where the name is used rather than where it is defined? `from x import y` copies the reference into your module, so patching the original module leaves your copy untouched.

**Q15. What security features does Django provide, and what must you configure?**
Out of the box: CSRF protection, template autoescaping, parameterized SQL, clickjacking protection, password hashing and validators, `ALLOWED_HOSTS` checks and signed cookies. You configure production pieces: `DEBUG = False`, secrets from the environment, HTTPS settings (`SECURE_SSL_REDIRECT`, secure cookies, HSTS, `SECURE_PROXY_SSL_HEADER` behind a proxy), explicit serializer fields, scoped querysets, and you run `manage.py check --deploy`.
*Follow-up:* What does HSTS do? It tells browsers to use HTTPS only for your domain for a period, preventing downgrade attacks.

**Q16. How would you deploy a Django app?**
Build an image or virtualenv with pinned dependencies, run `collectstatic`, apply migrations as a release step, and serve the app with Gunicorn (WSGI) or Uvicorn (ASGI) behind Nginx or a load balancer that terminates TLS. Serve static files with WhiteNoise or a CDN and media from object storage, run Celery workers and one beat, read configuration from environment variables, and add health checks, logging to stdout and error tracking.
*Follow-up:* How many Gunicorn workers? A common starting point is two per CPU core plus one, then tune using memory and latency measurements.

**Q17. When would you write a custom management command?**
For operational tasks that run outside requests: data backfills, imports, cleanup jobs, scheduled maintenance via cron, or one-off fixes you want reviewed and repeatable instead of pasted into a shell. Create `app/management/commands/name.py` with a `Command(BaseCommand)` that defines `add_arguments()` and `handle()`, and keep the logic in a service function so Celery tasks can reuse it.
*Follow-up:* How do you test one? `call_command("name", option=value)` inside a test, asserting on database state and output.

## Further reading
- [Middleware](https://docs.djangoproject.com/en/stable/topics/http/middleware/)
- [Signals](https://docs.djangoproject.com/en/stable/topics/signals/)
- [Cache framework](https://docs.djangoproject.com/en/stable/topics/cache/)
- [Asynchronous support](https://docs.djangoproject.com/en/stable/topics/async/)
- [Testing in Django](https://docs.djangoproject.com/en/stable/topics/testing/)
- [Security in Django](https://docs.djangoproject.com/en/stable/topics/security/)
