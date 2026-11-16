# Django Fundamentals

> **Tier:** T1 (know cold) · **Related:** [Models and ORM](02-models-and-orm.md), [Django REST Framework](04-django-rest-framework.md), [Advanced Django](06-django-advanced.md), [LMS project patterns](07-lms-project-patterns.md), [Django rapid-fire questions](99-interview-questions.md), [Python fundamentals](../03-languages/python/01-fundamentals.md), [How the web works](../04-web-fundamentals/01-how-the-web-works.md), [Rails fundamentals](../05-ruby-on-rails/01-rails-fundamentals.md)
> **On your resume:** PureLogics, Python (Django) Full-Stack Intern: you proposed and built an LMS (Coursera clone) with subscriptions and role-based dashboards using DRF and React. Even behind a React frontend, the project layout, settings, URLconf, admin, auth and custom user model are core Django.

## Quick revision
- Django is a "batteries-included" Python web framework: ORM, migrations, admin, auth, forms, templates, sessions, caching and security middleware ship in the box.
- **MTV**: Model (data), Template (presentation), View (request handling, the role of an MVC controller). The URL dispatcher and framework glue act as the controller.
- Request flow: WSGI/ASGI server, handler builds an `HttpRequest`, middleware (top to bottom), URL resolver, view, `HttpResponse`, middleware again (bottom to top), server.
- **Project** = settings, root URLconf and WSGI/ASGI entry points. **App** = a package for one feature area (models, views, urls, admin, migrations) listed in `INSTALLED_APPS`.
- Never run production with `DEBUG = True`. With `DEBUG = False`, `ALLOWED_HOSTS` must list your domains or every request gets a 400.
- Secrets and per-environment values come from environment variables, never from a committed `settings.py`.
- `manage.py` is `django-admin` bound to your settings: `runserver`, `startapp`, `makemigrations`, `migrate`, `createsuperuser`, `shell`, `test`, `collectstatic`, `check --deploy`.
- URLs: `path("<slug:slug>/", view, name="detail")`, one `include()` per app, `app_name` namespaces, and `reverse()` or `{% url %}` instead of hardcoded paths.
- FBVs are explicit functions; CBVs reuse behavior through generic views and mixins. Mixins such as `LoginRequiredMixin` go leftmost in the bases.
- Templates: inheritance via `{% extends %}` and `{% block %}`, autoescaping on by default; `|safe` and `mark_safe()` turn it off (XSS risk).
- Forms validate and clean input: `is_valid()`, `cleaned_data`, `clean_<field>()`, `clean()`. `ModelForm` builds a form from a model; `save(commit=False)` lets you set server-side fields.
- CSRF: `CsrfViewMiddleware` plus `{% csrf_token %}` in every POST form; JavaScript sends the token in an `X-CSRFToken` header.
- The admin is generated from your models and tuned with `ModelAdmin` (`list_display`, `list_filter`, `search_fields`, `list_select_related`, inlines, actions). It is a staff tool, not a customer UI.
- Static files are your CSS/JS (`collectstatic` gathers them into `STATIC_ROOT`); media files are user uploads (`MEDIA_ROOT` or a storage backend such as S3).
- Sessions keep per-visitor data server-side behind a cookie; the messages framework shows one-time notices after redirects.
- Set `AUTH_USER_MODEL` to a custom user model before the first migration; changing it later is painful. Reference it with `settings.AUTH_USER_MODEL` in fields and `get_user_model()` in code.

## 1. What Django is and why it is built this way

Django was built in a newsroom to ship database-backed sites fast, which explains its philosophy:
- **Batteries included**: the common needs of a web app (ORM, auth, admin, forms, security) are built in and designed to work together, so teams do not assemble and maintain a stack of small libraries.
- **DRY and explicit configuration**: models are the single source of truth for schema, admin, forms and serializers; settings and URL patterns are explicit Python rather than conventions resolved by magic.
- **Loose coupling**: apps are self-contained packages, and layers (URLs, views, templates, models) can be swapped (Jinja2 templates, DRF instead of templates, another database).
- **Secure by default**: CSRF protection, template autoescaping, parameterized SQL, clickjacking headers and password hashing are on without extra work.

| MVC term | Django (MTV) | What it does |
|---|---|---|
| Model | Model | Data, business rules, persistence through the ORM |
| View | Template | Presentation: HTML (or JSON through DRF serializers) |
| Controller | View (plus the URL dispatcher) | Takes the request, talks to models, picks a template or response |

The naming confuses people: a Django "view" decides *what* data to show, and the template decides *how* it looks.

## 2. The request/response cycle

```text
Browser
  -> Nginx (TLS termination, static files)
  -> Gunicorn worker (WSGI) or Uvicorn worker (ASGI)
  -> Django handler builds an HttpRequest
  -> middleware, request phase, in MIDDLEWARE order (security, session, CSRF, auth, ...)
  -> URL resolver matches request.path against ROOT_URLCONF and picks a view plus kwargs
  -> view runs: queries models, renders a template or builds JSON
  -> middleware, response phase, in reverse order
  -> HttpResponse goes back through the server to the browser
```

Details interviewers like:
- `AuthenticationMiddleware` sets `request.user` (an `AnonymousUser` when nobody is logged in), which is why it must come after `SessionMiddleware`.
- If no URL matches, Django raises `Http404`; an unhandled exception becomes a 500 (with a debug page only when `DEBUG = True`). Middleware can intercept both through `process_exception`.
- WSGI is the synchronous Python server interface; ASGI adds async support (async views, websockets through Channels). The same project ships both `wsgi.py` and `asgi.py`.

## 3. Projects and apps

```bash
django-admin startproject config .        # the project: settings, root urls, wsgi.py, asgi.py
python manage.py startapp courses         # an app: one feature area
```

```text
lms/
  manage.py
  config/
    settings.py          # or a settings/ package: base.py, dev.py, prod.py
    urls.py              # ROOT_URLCONF
    wsgi.py, asgi.py     # server entry points
  accounts/              # custom user model, auth views
  courses/
    models.py  views.py  urls.py  admin.py  apps.py  forms.py  tests.py
    migrations/
    templates/courses/   # namespaced template folder
```

Split apps by domain (accounts, courses, enrollments, billing), not by technical layer. An app does nothing until it is added to `INSTALLED_APPS`: Django then discovers its models (and migrations), admin registrations, templates, static files and management commands. `apps.py` holds the `AppConfig`, whose `ready()` method is the place to connect signals.

## 4. Settings

```python
# config/settings.py (excerpt)
import os
from pathlib import Path

BASE_DIR = Path(__file__).resolve().parent.parent
SECRET_KEY = os.environ["DJANGO_SECRET_KEY"]                       # fail fast if missing
DEBUG = os.environ.get("DJANGO_DEBUG", "false").lower() == "true"
ALLOWED_HOSTS = os.environ.get("DJANGO_ALLOWED_HOSTS", "localhost").split(",")

INSTALLED_APPS = [
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",
    "rest_framework",
    "accounts",
    "courses",
]

DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.postgresql",
        "NAME": os.environ.get("DB_NAME", "lms"),
        "USER": os.environ.get("DB_USER", "lms"),
        "PASSWORD": os.environ.get("DB_PASSWORD", ""),
        "HOST": os.environ.get("DB_HOST", "localhost"),
        "PORT": os.environ.get("DB_PORT", "5432"),
    }
}

AUTH_USER_MODEL = "accounts.User"
TIME_ZONE = "Asia/Karachi"
USE_TZ = True                          # store UTC in the database, convert for display
STATIC_URL = "static/"
STATIC_ROOT = BASE_DIR / "staticfiles"
MEDIA_URL = "media/"
MEDIA_ROOT = BASE_DIR / "media"
```

| Setting | Purpose | Gotcha |
|---|---|---|
| `DEBUG` | Detailed error pages, static serving by `runserver` | On in production it leaks settings, SQL and stack traces |
| `ALLOWED_HOSTS` | Allowed `Host` header values | Empty with `DEBUG = False` means every request returns 400 |
| `SECRET_KEY` | Signs sessions, password reset tokens and other signed data | Leaking it lets attackers forge signed values |
| `INSTALLED_APPS` | Activates apps | A missing app means no migrations, admin or templates for it |
| `MIDDLEWARE` | Ordered request and response hooks | Order matters (session before auth) |
| `DATABASES` | Connections, options, `CONN_MAX_AGE` | Credentials belong in environment variables |
| `TEMPLATES` | Engines, template dirs, context processors | `APP_DIRS: True` finds `app/templates/` |
| `AUTH_USER_MODEL` | Swappable user model | Set it before the first `migrate` |

**Per-environment settings.** Two common patterns: a single `settings.py` that reads everything from environment variables (12-factor style, often with django-environ and a git-ignored `.env` for local development), or a `settings/` package with `base.py` plus `dev.py` and `prod.py` that start with `from .base import *`, selected through the `DJANGO_SETTINGS_MODULE` environment variable. Either way, secrets never live in git.

## 5. manage.py commands

| Command | What it does |
|---|---|
| `runserver` | Development server with auto-reload (never for production) |
| `startapp name` | Scaffold a new app |
| `makemigrations [app]` | Generate migration files from model changes |
| `migrate [app] [target]` | Apply (or roll back to) migrations |
| `showmigrations`, `sqlmigrate app 0003` | List applied migrations; print a migration's SQL |
| `createsuperuser` | Create an admin user |
| `shell`, `dbshell` | Python shell with Django loaded; database client |
| `test` | Run the test suite (or `pytest` with pytest-django) |
| `collectstatic` | Copy static files into `STATIC_ROOT` for deployment |
| `check --deploy` | Audit settings for production security problems |
| `dumpdata`, `loaddata` | Export and import fixtures |

## 6. URLconf: path, include, namespaces and reverse

```python
# config/urls.py
from django.contrib import admin
from django.urls import include, path

urlpatterns = [
    path("admin/", admin.site.urls),
    path("courses/", include("courses.urls")),
    path("api/", include("api.urls")),
]

# courses/urls.py
from django.urls import path
from . import views

app_name = "courses"                     # namespace: "courses:detail"
urlpatterns = [
    path("", views.CourseListView.as_view(), name="list"),
    path("new/", views.course_create, name="create"),
    path("<slug:slug>/", views.course_detail, name="detail"),
    path("<slug:slug>/edit/", views.CourseUpdateView.as_view(), name="edit"),
    path("<slug:slug>/enroll/", views.enroll, name="enroll"),
]
```

- Path converters: `str` (default), `int`, `slug`, `uuid`, `path`; `re_path()` takes a regex. Captured values arrive as keyword arguments to the view.
- Patterns are tried in order and the first match wins, so `"new/"` must come before `"<slug:slug>/"`.
- Never hardcode URLs. In Python use `reverse("courses:detail", kwargs={"slug": course.slug})` or `redirect("courses:detail", slug=course.slug)`; in templates use `{% url 'courses:detail' slug=course.slug %}`; use `reverse_lazy` in class attributes evaluated at import time (such as `success_url`). Renaming a path then changes one line.
- Defining `get_absolute_url()` on a model gives templates, the admin and redirects one canonical link.

## 7. Function-based vs class-based views

```python
# courses/views.py
from django.contrib import messages
from django.contrib.auth.decorators import login_required
from django.contrib.auth.mixins import LoginRequiredMixin
from django.shortcuts import get_object_or_404, redirect, render
from django.urls import reverse
from django.views.decorators.http import require_POST
from django.views.generic import ListView, UpdateView

from .forms import CourseForm
from .models import Course, Enrollment

def course_detail(request, slug):
    course = get_object_or_404(Course.objects.select_related("instructor"), slug=slug, status="published")
    return render(request, "courses/detail.html", {"course": course})

@login_required
@require_POST                                   # 405 for anything but POST
def enroll(request, slug):
    course = get_object_or_404(Course, slug=slug, status="published")
    Enrollment.objects.get_or_create(student=request.user, course=course)
    messages.success(request, f"You are enrolled in {course.title}.")
    return redirect("courses:detail", slug=slug)   # Post/Redirect/Get

class CourseListView(ListView):
    template_name = "courses/list.html"
    context_object_name = "courses"
    paginate_by = 20

    def get_queryset(self):
        return Course.objects.filter(status="published").select_related("instructor")

class CourseUpdateView(LoginRequiredMixin, UpdateView):   # mixin first
    model = Course
    form_class = CourseForm
    template_name = "courses/form.html"

    def get_queryset(self):                   # other instructors' courses become a 404
        return Course.objects.filter(instructor=self.request.user)

    def get_success_url(self):
        return reverse("courses:detail", kwargs={"slug": self.object.slug})
```

How a CBV runs: `as_view()` returns a plain function; on each request it creates a fresh instance, calls `setup()` and then `dispatch()`, which calls `get()`, `post()` and so on according to the HTTP method (405 if the method is not defined). Generic views fill in those methods and expose hooks: `get_queryset()`, `get_object()`, `get_context_data()`, `form_valid()`, `get_success_url()`.

| | Function-based views | Class-based views |
|---|---|---|
| Readability | explicit, reads top to bottom | logic spread across methods and mixins |
| Reuse | decorators and helper functions | inheritance, mixins, generic views |
| HTTP methods | `if request.method == "POST":` | separate `get()` and `post()` methods |
| Best for | one-off flows, webhooks, unusual logic | standard list, detail and CRUD screens, shared behavior |
| Learning curve | low | you must know the generic view method flow |

Common generic views: `TemplateView`, `RedirectView`, `ListView`, `DetailView`, `CreateView`, `UpdateView`, `DeleteView`, `FormView`. Access mixins: `LoginRequiredMixin`, `PermissionRequiredMixin`, `UserPassesTestMixin`. Scoping `get_queryset()` to the current user, as above, is the simplest defense against editing someone else's object.

## 8. Templates

```html
<!-- templates/base.html -->
{% load static %}
<!doctype html>
<html>
<head>
  <title>{% block title %}LMS{% endblock %}</title>
  <link rel="stylesheet" href="{% static 'css/app.css' %}">
</head>
<body>
  {% for message in messages %}<div class="alert {{ message.tags }}">{{ message }}</div>{% endfor %}
  {% block content %}{% endblock %}
</body>
</html>

<!-- templates/courses/detail.html -->
{% extends "base.html" %}
{% block title %}{{ course.title }}{% endblock %}
{% block content %}
  <h1>{{ course.title }}</h1>
  <p>By {{ course.instructor.get_full_name|default:"Unknown" }}, {{ course.created_at|date:"M j, Y" }}</p>
  <p>{{ course.description|truncatewords:50 }}</p>
  {% for module in course.modules.all %}
    <h2>{{ forloop.counter }}. {{ module.title }}</h2>
  {% empty %}
    <p>No modules yet.</p>
  {% endfor %}
  <form method="post" action="{% url 'courses:enroll' slug=course.slug %}">
    {% csrf_token %}
    <button type="submit">Enroll</button>
  </form>
{% endblock %}
```

- `{{ }}` outputs a variable, `{% %}` runs a tag (`if`, `for`, `url`, `include`, `block`), and `|` applies a filter (`date`, `default`, `length`, `truncatewords`, `linebreaks`).
- **Autoescaping**: every `{{ }}` output escapes `<`, `>`, `&`, `'` and `"`, so a comment containing `<script>` renders as text. `|safe`, `{% autoescape off %}` and `mark_safe()` disable it; use them only for HTML you control, and build HTML in Python with `format_html()`, which escapes its arguments.
- **Context processors** are functions that add variables to every template rendered with a request (`request`, `user`, `messages` come from the defaults), registered in `TEMPLATES[...]["OPTIONS"]["context_processors"]`.
- The template language is deliberately limited (no arbitrary Python), but it can call zero-argument methods such as `course.modules.all`, which is where hidden N+1 queries come from. Prepare data in the view with `prefetch_related` (see [ORM performance](03-orm-performance.md)).
- Custom filters and tags live in an app's `templatetags/` package and register on a `template.Library()` instance.

## 9. Forms, ModelForms and CSRF

```python
# courses/forms.py
from django import forms
from .models import Course

class CourseForm(forms.ModelForm):
    class Meta:
        model = Course
        fields = ["title", "description", "price", "level"]     # explicit, never "__all__"

    def clean_title(self):                                      # one field
        title = self.cleaned_data["title"].strip()
        duplicate = Course.objects.filter(title__iexact=title).exclude(pk=self.instance.pk)
        if duplicate.exists():
            raise forms.ValidationError("A course with this title already exists.")
        return title

    def clean(self):                                            # rules across fields
        cleaned = super().clean()
        if cleaned.get("level") == "advanced" and cleaned.get("price") == 0:
            self.add_error("price", "Advanced courses cannot be free.")
        return cleaned

# courses/views.py
@login_required
def course_create(request):
    if request.method == "POST":
        form = CourseForm(request.POST, request.FILES)
        if form.is_valid():
            course = form.save(commit=False)     # build the instance without saving
            course.instructor = request.user     # server-side field the user must not control
            course.save()
            return redirect("courses:detail", slug=course.slug)
    else:
        form = CourseForm()
    return render(request, "courses/form.html", {"form": form})   # invalid forms re-render with errors
```

Validation order inside `is_valid()`: each field converts and validates its raw value (plus any `validators=`), then `clean_<field>()` runs for each field, then `clean()` for cross-field rules; a `ModelForm` then validates the model instance too (model `clean()` and unique checks). Results land in `form.cleaned_data` or `form.errors`. Listing `fields` explicitly prevents mass assignment: with `"__all__"` a crafted POST could set `instructor` or `is_featured`.

**CSRF** (cross-site request forgery): a malicious page makes the victim's browser submit a request to your site, and the browser attaches the victim's cookies. Django's defense: `CsrfViewMiddleware` requires a secret token, from the `csrftoken` cookie, to come back in the form (`{% csrf_token %}`) or an `X-CSRFToken` header on every unsafe request (POST, PUT, PATCH, DELETE), and on HTTPS it also checks the `Origin` or `Referer` header. A failed check returns 403. List cross-origin frontends in `CSRF_TRUSTED_ORIGINS`, and reserve `@csrf_exempt` for endpoints authenticated another way, such as signed webhooks.

## 10. The admin

```python
# courses/admin.py
from django.contrib import admin
from .models import Course, Module

class ModuleInline(admin.TabularInline):
    model = Module
    extra = 0

@admin.register(Course)
class CourseAdmin(admin.ModelAdmin):
    list_display = ["title", "instructor", "status", "price", "created_at"]
    list_filter = ["status", "level"]
    search_fields = ["title", "instructor__email"]
    list_select_related = ["instructor"]          # one JOIN instead of N+1 on the changelist
    prepopulated_fields = {"slug": ["title"]}
    raw_id_fields = ["instructor"]                # no giant dropdown of every user
    inlines = [ModuleInline]
    actions = ["publish"]

    @admin.action(description="Publish selected courses")
    def publish(self, request, queryset):
        updated = queryset.update(status="published")
        self.message_user(request, f"{updated} course(s) published.")
```

The admin reads your models' metadata to build list, search, filter and edit screens, with per-model add, change, delete and view permissions. Users need `is_staff` to log in. Treat it as an internal tool for staff and moderators (reviewing courses, refunds, support lookups); customer-facing dashboards belong in your own views or React frontend. Override `get_queryset()` to scope or optimize what staff see, and use `readonly_fields` for audit data.

## 11. Static and media files

| | Static files | Media files |
|---|---|---|
| What | CSS, JS and images that ship with the code | User uploads (avatars, thumbnails, assignments) |
| Settings | `STATIC_URL`, `STATICFILES_DIRS`, `STATIC_ROOT` | `MEDIA_URL`, `MEDIA_ROOT`, a storage backend |
| Production | `collectstatic`, then WhiteNoise, Nginx or a CDN serves them | Object storage (S3 via django-storages) or a dedicated volume |
| In templates | `{% static 'css/app.css' %}` | `{{ course.thumbnail.url }}` |

In development `runserver` serves static files when `DEBUG = True`; media needs `urlpatterns += static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)`, which is a development-only helper. Since Django 4.2 storage backends are configured through the `STORAGES` setting. Validate uploads (size, type) and never trust the client's file name.

## 12. Sessions and messages

```python
from django.contrib import messages
from django.utils import timezone

def checkout_success(request):
    request.session["last_checkout_at"] = timezone.now().isoformat()  # JSON-serializable values only
    visits = request.session.get("visits", 0)
    request.session["visits"] = visits + 1
    messages.success(request, "Payment received. Welcome aboard!")    # shown once on the next page
    return redirect("dashboard")
```

A session is server-side data keyed by a random ID in the `sessionid` cookie (HttpOnly by default). The default backend is the database; cache, cached_db, file and signed-cookie backends exist. Data is JSON-serialized, so store IDs and strings, not model instances. `login()` rotates the session key to prevent session fixation, and `logout()` flushes the session. The messages framework stores one-time notices (in the session or a cookie) that templates display after a redirect.

## 13. Authentication and the custom user model

```python
# accounts/models.py
from django.contrib.auth.models import AbstractUser
from django.db import models

class User(AbstractUser):
    class Role(models.TextChoices):
        STUDENT = "student", "Student"
        INSTRUCTOR = "instructor", "Instructor"
        ADMIN = "admin", "Admin"

    email = models.EmailField(unique=True)
    role = models.CharField(max_length=20, choices=Role.choices, default=Role.STUDENT)

# config/settings.py
AUTH_USER_MODEL = "accounts.User"

# courses/models.py: refer to the user model indirectly
from django.conf import settings

class Course(models.Model):
    instructor = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.PROTECT, related_name="courses_taught")

# in code: from django.contrib.auth import get_user_model; User = get_user_model()
```

`django.contrib.auth` gives you the user model, password hashing (PBKDF2 by default, Argon2, bcrypt and scrypt available), `authenticate()` (returns a user or `None`), `login()` and `logout()`, permissions (`user.has_perm("courses.change_course")`), groups, `is_staff`, `is_superuser`, `is_active`, the `@login_required` decorator and ready-made views such as `LoginView` and `PasswordResetView` (since Django 5.0 `LogoutView` accepts only POST).

**Why a custom user model at the start**: foreign keys and the earliest migrations of every app point at `AUTH_USER_MODEL`. Switching it after tables exist means rewriting migrations and migrating data by hand, so the docs recommend a custom model for every new project even if it adds nothing yet. Choose `AbstractUser` to keep Django's fields and add your own (role, phone), or `AbstractBaseUser` plus `PermissionsMixin` for full control, such as email-only login, which also requires a custom manager with `create_user()` and `create_superuser()`. App-specific data that is not about authentication can live in a one-to-one `Profile` model.

## 14. Django vs Rails mapping

| Concern | Django | Rails |
|---|---|---|
| Structure | Project plus apps | One app (engines for modular pieces) |
| Routing | `urls.py` with `path()` and `include()` | `config/routes.rb` with `resources` |
| Request handling | Views (FBV or CBV) | Controller actions |
| Templates | Django template language (or Jinja2) | ERB views and partials |
| ORM | Models, `QuerySet`, managers | ActiveRecord, relations and scopes |
| Migrations | Generated from model diffs (`makemigrations`) | Hand-written or generated (`bin/rails g migration`), schema in `schema.rb` |
| Input handling | Forms and ModelForms (DRF serializers for APIs) | Strong parameters plus model validations |
| Hooks | Signals, overriding `save()` | Model callbacks |
| Middleware | Django middleware | Rack middleware |
| Admin UI | Built in | Gems (ActiveAdmin, Administrate) |
| Background jobs | Celery (recent Django versions also add a built-in tasks API) | Active Job with Sidekiq or Solid Queue |
| CLI | `manage.py` | `bin/rails` |
| Eager loading | `select_related`, `prefetch_related` | `includes`, `preload`, `eager_load` |
| Dependencies | pip, `requirements.txt` or `pyproject.toml` | Bundler, `Gemfile` |

## Common pitfalls
- `DEBUG = True` in production, or `DEBUG = False` with an empty `ALLOWED_HOSTS` (every request returns 400).
- `SECRET_KEY`, database passwords or a `.env` file committed to git.
- Creating an app but forgetting to add it to `INSTALLED_APPS`, then wondering why `makemigrations` finds nothing.
- Hardcoded URLs instead of `reverse()` and `{% url %}`; overlapping patterns in the wrong order (`<slug>` before `new/`).
- `LoginRequiredMixin` placed after the generic view in the bases, so its check never runs first.
- `fields = "__all__"` in a ModelForm, which lets a crafted POST set fields such as `instructor` or `is_staff`.
- Missing `{% csrf_token %}` (403 on submit), or "fixing" it with `@csrf_exempt`.
- `|safe` or `mark_safe()` on user-provided content, reopening XSS.
- Loops in templates that touch relations (`course.modules.all`) without `prefetch_related`, causing N+1 queries.
- Starting with the default `User` and needing a custom one after the first migrations.
- Rendering a page directly after a successful POST instead of redirecting, so a refresh resubmits the form.
- Naive datetimes (`datetime.now()`) with `USE_TZ = True`; use `django.utils.timezone.now()`.
- Serving media from the app server in production or trusting uploaded file names and types.

## How to talk about this in an interview
- Lead with the PureLogics LMS: a Django and DRF backend with a React frontend. Fundamentals you can speak to from it, where they match what you built: a custom user model with roles, the admin as an internal tool, per-environment settings and namespaced API URLs. Keep the claims to what you actually did.
- Show transferable framework knowledge: you work in Rails daily, so map concepts in one breath (routes.rb to urls.py, controllers to views, ERB to templates, ActiveRecord to the ORM, hand-written migrations to generated ones). Interviewers like candidates who understand the shared web fundamentals rather than one framework's syntax.
- When asked "MTV or MVC?", give the mapping table answer and add that with DRF and React, templates mostly disappear and serializers become the presentation layer.
- > **Fill in:** the Django version you used, and whether the LMS used a custom user model with a `role` field, Django groups, or both.
- > **Fill in:** which admin customizations you built (course review queue, list filters, actions) and who used them.
- > **Fill in:** how settings and secrets were handled (environment variables, django-environ, split settings files).

## Interview questions

**Q1. What is Django, and what does "batteries included" mean?**
A high-level Python web framework that ships the common pieces of a database-backed web app: ORM and migrations, URL routing, templates, forms, authentication and permissions, the admin, sessions, caching and security middleware. "Batteries included" means these are built in and designed to work together, so you start productive without assembling libraries.
*Follow-up:* What does it not include? A background job system in most versions (teams use Celery) and a REST toolkit (teams use DRF).

**Q2. Explain Django's MTV pattern and how it maps to MVC.**
Model holds data and business rules; Template handles presentation; View receives the request, uses models and returns a response, so it plays the MVC controller's role while the URL dispatcher routes to it. The naming confuses people because a Django "view" decides what to show, not how it looks.
*Follow-up:* What replaces templates in an API backend? DRF serializers and renderers produce JSON for a frontend such as React.

**Q3. Walk me through what happens when a request hits a Django app.**
The WSGI or ASGI server hands the request to Django's handler, which builds an `HttpRequest` and passes it through each middleware in order. The URL resolver matches the path against `ROOT_URLCONF` and calls the view with captured arguments; the view returns an `HttpResponse` (rendering a template or JSON). The response travels back through the middleware in reverse order to the server and client.
*Follow-up:* Where does `request.user` come from? `AuthenticationMiddleware`, which reads the user ID from the session (or DRF's authentication classes for API requests).

**Q4. What is the difference between a project and an app?**
The project is the deployable configuration: settings, the root URLconf and the WSGI/ASGI entry points. An app is a Python package for one feature area (models, views, URLs, admin, migrations), activated by listing it in `INSTALLED_APPS`, and ideally reusable across projects.
*Follow-up:* How do you split apps? By business domain (accounts, courses, billing), keeping dependencies between them one-directional.

**Q5. What does `DEBUG` do, and what changes when you turn it off?**
With `DEBUG = True` Django shows detailed error pages with settings and local variables, serves static files through `runserver`, and records SQL queries in memory. Turning it off hides those pages (use logging and error tracking instead), requires `ALLOWED_HOSTS`, and expects static files to be served by WhiteNoise, Nginx or a CDN after `collectstatic`.
*Follow-up:* What is `ALLOWED_HOSTS` for? It validates the `Host` header to block host header attacks, such as poisoned password reset links.

**Q6. How do you manage settings and secrets across environments?**
Keep secrets and environment-specific values in environment variables (a git-ignored `.env` locally, the platform's secret store in production), read them in settings with `os.environ` or django-environ, and fail fast when a required one is missing. Optionally split settings into `base`, `dev` and `prod` modules selected by `DJANGO_SETTINGS_MODULE`. Run `manage.py check --deploy` against production settings.
*Follow-up:* What if `SECRET_KEY` leaks? Rotate it (Django supports `SECRET_KEY_FALLBACKS` for graceful rotation) and accept that existing sessions and tokens become invalid.

**Q7. How does URL routing work, and why use `reverse()`?**
`urlpatterns` lists `path()` entries tried in order; converters such as `<int:pk>` capture values and pass them to the view as keyword arguments, and `include()` delegates a prefix to an app's URLconf. Naming routes and using `reverse()` or `{% url %}` means links survive URL changes, and `app_name` namespaces prevent name clashes between apps.
*Follow-up:* When do you need `reverse_lazy`? In class attributes or settings evaluated at import time, before the URLconf is loaded.

**Q8. Function-based or class-based views: which do you prefer?**
FBVs are explicit and easy to follow, ideal for unusual flows and webhooks. CBVs remove boilerplate for standard patterns (list, detail, create, update, delete) and share behavior through mixins, at the cost of indirection. I use generic CBVs (or DRF viewsets) for CRUD and FBVs when the flow is custom.
*Follow-up:* How do you protect a CBV? Add `LoginRequiredMixin` or `PermissionRequiredMixin` as the leftmost base, and scope `get_queryset()` to the current user.

**Q9. How does a class-based view work internally?**
`as_view()` returns a function. Per request it instantiates the class, calls `setup()` to store the request and URL arguments, then `dispatch()`, which looks up a method named after the HTTP verb (`get`, `post`) or returns 405. Generic views implement those methods with overridable hooks such as `get_queryset()`, `get_context_data()` and `form_valid()`.
*Follow-up:* Why must mixins come first? Python's MRO searches left to right, so a mixin's `dispatch()` runs before the view's and can stop the request.

**Q10. How do Django templates protect against XSS?**
Autoescaping is on by default: every variable rendered with `{{ }}` has HTML special characters escaped, so injected markup displays as text. The protection disappears with `|safe`, `{% autoescape off %}` or `mark_safe()`, so use those only on trusted HTML and build HTML in Python with `format_html()`.
*Follow-up:* Does autoescaping help inside a `<script>` block? Not reliably; pass data to JavaScript with the `json_script` filter instead.

**Q11. How does form validation work? `clean_<field>()` vs `clean()`?**
`is_valid()` runs each field's conversion and validators, then each `clean_<field>()` method for single-field rules (it returns the cleaned value), then `clean()` for rules that involve several fields (use `add_error()` to attach messages). ModelForms also run the model's validation and unique checks. Valid data ends up in `cleaned_data`; errors in `form.errors`.
*Follow-up:* What does `save(commit=False)` do? Builds the model instance without saving, so you can set server-side fields like the owner before calling `save()`.

**Q12. What is CSRF, and how does Django prevent it?**
Cross-site request forgery tricks a logged-in user's browser into sending a state-changing request to your site with their cookies. `CsrfViewMiddleware` requires a token tied to the `csrftoken` cookie in the form data or the `X-CSRFToken` header for unsafe methods, and checks `Origin`/`Referer` on HTTPS. Templates add the token with `{% csrf_token %}`.
*Follow-up:* Does a token-authenticated API need CSRF protection? Not when credentials travel in an `Authorization` header that browsers never attach automatically; cookie-based sessions do need it.

**Q13. How would you customize the Django admin, and should customers use it?**
Register models with a `ModelAdmin` and configure `list_display`, `list_filter`, `search_fields`, `list_select_related`, `readonly_fields`, inlines for child objects and custom actions for bulk operations. It is an internal tool for staff (support, moderation, data fixes), not a customer-facing UI, because its UX and permission model are built for trusted users.
*Follow-up:* The changelist is slow: why? Usually N+1 queries from foreign keys in `list_display` (fix with `list_select_related`) or a huge FK dropdown on the edit page (fix with `raw_id_fields` or `autocomplete_fields`).

**Q14. What is the difference between static and media files in production?**
Static files are part of your code (CSS, JS, images): `collectstatic` copies them into `STATIC_ROOT`, and WhiteNoise, Nginx or a CDN serves them with long cache headers. Media files are user uploads saved through a storage backend, usually object storage such as S3 in production, and must be validated and never executed.
*Follow-up:* Why not let Django serve them? Python workers are expensive; dedicated servers or CDNs serve files far more efficiently.

**Q15. Why should you set a custom user model at the start of a project?**
`AUTH_USER_MODEL` is referenced by foreign keys and initial migrations across apps, so changing it once tables exist means hand-editing migrations and moving data. Subclassing `AbstractUser` from day one costs nothing and lets you add fields such as `role` later. Use `AbstractBaseUser` plus `PermissionsMixin` when you need a different login field, with a custom manager.
*Follow-up:* How should other models reference the user? `settings.AUTH_USER_MODEL` in foreign keys and `get_user_model()` in runtime code, never a direct import of `User`.

**Q16. How do sessions work in Django?**
`SessionMiddleware` loads a dict-like `request.session` keyed by a random session ID stored in the `sessionid` cookie; the data lives server-side (database by default, or cache, file or signed cookies). Values must be JSON-serializable. `login()` stores the user ID and rotates the key; `logout()` flushes it.
*Follow-up:* When would you use the cache backend? For speed at scale, typically `cached_db` so a cache eviction does not log users out.

**Q17. What are context processors?**
Functions that receive the request and return a dict merged into the context of every template rendered with that request. The defaults add `request`, `user`, `perms` and `messages`; you can add your own for site-wide values such as feature flags or navigation data.
*Follow-up:* What is the downside? They run on every render, so expensive queries inside them slow down every page.

## Further reading
- [Django documentation](https://docs.djangoproject.com/en/stable/)
- [Request and response objects](https://docs.djangoproject.com/en/stable/ref/request-response/)
- [Class-based views](https://docs.djangoproject.com/en/stable/topics/class-based-views/)
- [Customizing authentication](https://docs.djangoproject.com/en/stable/topics/auth/customizing/)
- [Deployment checklist](https://docs.djangoproject.com/en/stable/howto/deployment/checklist/)
