# Django REST Framework

> **Tier:** T1 (know cold) · **Related:** [DRF auth, permissions and advanced topics](05-drf-auth-permissions-advanced.md), [ORM performance](03-orm-performance.md), [Models and ORM](02-models-and-orm.md), [LMS project patterns](07-lms-project-patterns.md), [Django rapid-fire questions](99-interview-questions.md), [REST API design](../04-web-fundamentals/04-rest-api-design.md), [Rails API development](../05-ruby-on-rails/10-api-development.md), [Data fetching with TanStack Query](../06-react/04-data-fetching-tanstack-query.md)
> **On your resume:** At PureLogics the LMS backend served a React frontend through Django REST Framework: serializers, viewsets and routers for courses, enrollments, progress and the role-based dashboards.

## Quick revision
- DRF adds to Django: a `Request`/`Response` pair with content negotiation, serializers (conversion plus validation), API views, viewsets and routers, authentication, permissions, throttling, pagination, filtering, the browsable API and schema generation.
- `request.data` is the parsed body (JSON, form or multipart) for any method; `request.query_params` is the query string. `Response(data, status=...)` is rendered by the negotiated renderer (JSON or the browsable API).
- A serializer converts model instances to primitive data and validates incoming data. `ModelSerializer` generates fields and validators from the model and implements `create()` and `update()`.
- Input flow: `S(data=request.data)`, `is_valid(raise_exception=True)`, `save(owner=request.user)`. Output: `S(instance).data`, and `many=True` for lists.
- `read_only=True` fields only appear in output; `write_only=True` fields only in input (passwords).
- Validation layers: field arguments and `validators=`, `validate_<field>(self, value)`, `validate(self, attrs)` for cross-field rules, and model-derived unique validators. DRF does not call the model's `full_clean()`.
- Nested serializers are read-only unless you write `create()`/`update()` yourself.
- `SerializerMethodField` calls `get_<name>(self, obj)` per object, which makes it a classic N+1 source. Load data in the view's `get_queryset()` with `select_related`, `prefetch_related` and annotations.
- View layers: `@api_view` functions, `APIView`, generic views (`ListCreateAPIView`, `RetrieveUpdateDestroyAPIView`) built from mixins on `GenericAPIView`, and viewsets (`ModelViewSet`) wired by routers.
- Key hooks: `get_queryset()`, `get_serializer_class()`, `perform_create()`, `get_permissions()`; `self.action` names the current viewset action.
- `@action(detail=True, methods=["post"])` adds custom routes such as `POST /courses/{slug}/enroll/`.
- `DefaultRouter` generates list and detail routes plus an API root; pass `basename` when the viewset has no `queryset` attribute.
- Status codes: 200, 201 (created), 204 (deleted), 400 (validation), 401 (not authenticated), 403 (not allowed), 404, 405, 409 (conflict), 429 (throttled).
- A custom `EXCEPTION_HANDLER` gives every error the same JSON shape.
- Test with `APITestCase` and `APIClient`: `force_authenticate()`, `format="json"`, then assert the status code and body.

## 1. Why DRF

You can return JSON from plain Django with `JsonResponse`, but every endpoint would hand-roll parsing, validation, error formats, authentication for tokens, permission checks, pagination and documentation. DRF standardizes all of that:

| Concern | Plain Django | DRF |
|---|---|---|
| Parsing the body | `json.loads(request.body)` by hand | `request.data` for JSON, form and multipart |
| Validation and conversion | Forms (HTML-oriented) or manual code | Serializers with structured error responses |
| Auth for APIs | Sessions only | Session, token and JWT (via simplejwt) authentication classes |
| Access control | Decorators per view | Permission classes, object-level permissions, throttling |
| CRUD boilerplate | Write each view | Generic views, viewsets and routers |
| Lists | Manual slicing | Pagination, filtering, search and ordering backends |
| Docs and exploration | None | Browsable API, OpenAPI schemas (drf-spectacular) |

## 2. Request, Response, parsers and renderers

```python
# config/settings.py
REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": [
        "rest_framework_simplejwt.authentication.JWTAuthentication",
        "rest_framework.authentication.SessionAuthentication",
    ],
    "DEFAULT_PERMISSION_CLASSES": ["rest_framework.permissions.IsAuthenticated"],
    "DEFAULT_PAGINATION_CLASS": "rest_framework.pagination.PageNumberPagination",
    "PAGE_SIZE": 20,
    "EXCEPTION_HANDLER": "core.exceptions.api_exception_handler",
}
```

```python
from rest_framework import status
from rest_framework.decorators import api_view, permission_classes
from rest_framework.permissions import IsAuthenticated
from rest_framework.response import Response

@api_view(["GET", "POST"])                  # other methods get a 405
@permission_classes([IsAuthenticated])
def my_notes(request):
    if request.method == "GET":
        notes = Note.objects.filter(owner=request.user)
        return Response(NoteSerializer(notes, many=True).data)
    serializer = NoteSerializer(data=request.data)
    serializer.is_valid(raise_exception=True)          # 400 with field errors if invalid
    serializer.save(owner=request.user)
    return Response(serializer.data, status=status.HTTP_201_CREATED)
```

- DRF's `Request` wraps Django's: `request.data` (parsed body, chosen by the `Content-Type` and the parser classes `JSONParser`, `FormParser`, `MultiPartParser`), `request.query_params` (the query string), `request.user` and `request.auth` (set by authentication classes).
- `Response` takes native Python data; content negotiation picks a renderer from the `Accept` header (or `?format=json`), usually `JSONRenderer` or `BrowsableAPIRenderer`.
- Without explicit settings DRF allows everyone (`AllowAny`) and authenticates with session and basic auth, so set project-wide defaults deliberately.

## 3. Serializers

```python
from rest_framework import serializers
from .models import Course

class CourseSerializer(serializers.ModelSerializer):
    instructor_name = serializers.CharField(source="instructor.get_full_name", read_only=True)
    student_count = serializers.IntegerField(read_only=True)      # filled by annotate() in the view

    class Meta:
        model = Course
        fields = ["id", "title", "slug", "description", "price", "level", "status",
                  "instructor_name", "student_count", "created_at"]
        read_only_fields = ["slug", "status", "created_at"]

class EnrollRequestSerializer(serializers.Serializer):              # plain Serializer: input that is not a model
    course_id = serializers.IntegerField()
    coupon_code = serializers.CharField(required=False, allow_blank=True, max_length=32)
```

| | `Serializer` | `ModelSerializer` |
|---|---|---|
| Fields | declared by hand | generated from the model, overridable |
| Validators | yours | plus model field validators and unique checks |
| `create()` / `update()` | you implement them | default implementations |
| Use for | non-model input (login, search, action payloads), computed output | CRUD on a model |

```python
serializer = CourseSerializer(data=request.data)              # input: deserialize and validate
serializer.is_valid(raise_exception=True)
course = serializer.save(instructor=request.user)             # no instance, so create(); kwargs join validated_data

serializer = CourseSerializer(course, data=request.data, partial=True)   # instance given, so update(); PATCH semantics
serializer.is_valid(raise_exception=True)
serializer.save()

CourseSerializer(course).data                                 # output: a dict-like structure
CourseSerializer(Course.objects.all(), many=True).data        # a list
```

`save()` calls `create()` when the serializer was built without an instance and `update()` when it was given one. Keyword arguments to `save()` are merged into `validated_data`, which is how server-controlled values such as the owner get in without trusting the client.

```python
class RegisterSerializer(serializers.ModelSerializer):
    password = serializers.CharField(write_only=True, min_length=8)    # accepted, never returned

    class Meta:
        model = User
        fields = ["id", "username", "email", "password"]

    def create(self, validated_data):
        return User.objects.create_user(**validated_data)              # hashes the password
```

`read_only=True` fields are ignored on input (ids, timestamps, computed values); `write_only=True` fields never appear in output. `HiddenField(default=serializers.CurrentUserDefault())` fills a field from the request user without exposing it, which also lets unique-together validators include the user.

## 4. Validation

```python
class CourseSerializer(serializers.ModelSerializer):
    class Meta:
        model = Course
        fields = ["id", "title", "price", "level"]

    def validate_title(self, value):                  # one field, after its built-in checks
        if "test" in value.lower():
            raise serializers.ValidationError("Choose a real course title.")
        return value.strip()                          # the returned value is what gets saved

    def validate(self, attrs):                        # cross-field rules
        level = attrs.get("level", getattr(self.instance, "level", None))   # PATCH may omit fields
        price = attrs.get("price", getattr(self.instance, "price", 0))
        if level == "advanced" and price == 0:
            raise serializers.ValidationError({"price": "Advanced courses cannot be free."})
        return attrs
```

Order inside `is_valid()`: each field converts and validates its value (type, `max_length`, `required`, `validators=[...]`), then `validate_<field>()`, then `validate()`, then serializer-level validators such as `UniqueTogetherValidator`. Errors come back as `{"price": ["..."]}`, with cross-field errors raised as a plain string landing under `non_field_errors`.

- `ModelSerializer` adds validators from the model: `unique=True` becomes a `UniqueValidator`, `unique_together` a `UniqueTogetherValidator` (recent DRF versions also derive them from `UniqueConstraint`).
- DRF does **not** call the model's `clean()` or `full_clean()`. Database constraints still protect you, but a violation raises `IntegrityError`, a 500 unless you catch it and return a 400 or 409.
- Validation that needs the current user reads `self.context["request"]`, which generic views pass automatically.
- Keep serializers about shape and input rules; put multi-model workflows (enroll, charge, notify) in a service function called from the view.

## 5. Relations and nested serializers

| Field | Output | Input |
|---|---|---|
| `PrimaryKeyRelatedField(queryset=...)` | `7` | an id, validated against the queryset |
| `SlugRelatedField(slug_field="slug", queryset=...)` | `"python"` | a slug |
| `StringRelatedField()` | `str(obj)` | read-only |
| `HyperlinkedRelatedField` | a URL | a URL |
| A nested serializer | the full object | read-only unless you write `create()`/`update()` |

```python
class LessonSerializer(serializers.ModelSerializer):
    class Meta:
        model = Lesson
        fields = ["id", "title", "position", "duration_seconds", "is_preview"]

class ModuleSerializer(serializers.ModelSerializer):
    lessons = LessonSerializer(many=True, read_only=True)

    class Meta:
        model = Module
        fields = ["id", "title", "position", "lessons"]

class CourseDetailSerializer(serializers.ModelSerializer):
    modules = ModuleSerializer(many=True, read_only=True)
    category = serializers.SlugRelatedField(slug_field="slug", queryset=Category.objects.all())

    class Meta:
        model = Course
        fields = ["id", "title", "slug", "category", "modules"]
```

**Writable nested serializers** need explicit code, because DRF cannot guess whether children should be created, updated, replaced or deleted:

```python
from django.db import transaction

class ModuleWriteSerializer(serializers.ModelSerializer):
    lessons = LessonSerializer(many=True)

    class Meta:
        model = Module
        fields = ["id", "title", "position", "lessons"]

    def create(self, validated_data):
        lessons_data = validated_data.pop("lessons")
        with transaction.atomic():                                  # module and lessons together or not at all
            module = Module.objects.create(**validated_data)        # course arrives via save(course=...)
            Lesson.objects.bulk_create([Lesson(module=module, **data) for data in lessons_data])
        return module
```

For updates you must decide the semantics (replace every child, or match children by id) and test them. Many teams avoid writable nesting entirely and expose child endpoints such as `POST /modules/{id}/lessons/`, which are simpler to validate and authorize.

## 6. `SerializerMethodField` and the serializer N+1

```python
class CourseListSerializer(serializers.ModelSerializer):
    instructor = serializers.CharField(source="instructor.username", read_only=True)
    student_count = serializers.IntegerField(read_only=True)
    is_enrolled = serializers.SerializerMethodField()

    class Meta:
        model = Course
        fields = ["id", "title", "slug", "price", "instructor", "student_count", "is_enrolled"]

    def get_is_enrolled(self, obj):
        # BAD: Enrollment.objects.filter(course=obj, student=user).exists() would run once per course
        return getattr(obj, "is_enrolled", False)       # GOOD: precomputed by the view's queryset
```

```python
from django.db.models import Count, Exists, OuterRef
from rest_framework import viewsets

class CatalogViewSet(viewsets.ReadOnlyModelViewSet):
    serializer_class = CourseListSerializer

    def get_queryset(self):
        qs = (
            Course.objects.filter(status=Course.Status.PUBLISHED)
            .select_related("instructor")                                   # for instructor.username
            .annotate(student_count=Count("enrollments", distinct=True))    # for student_count
        )
        user = self.request.user
        if user.is_authenticated:
            qs = qs.annotate(is_enrolled=Exists(
                Enrollment.objects.filter(course=OuterRef("pk"), student=user)
            ))
        return qs
```

A serializer declares the shape; the view's `get_queryset()` must load everything that shape touches. Dotted `source` paths need `select_related`, nested serializers need `prefetch_related` (or a `Prefetch` with the right ordering), and counts or flags need `annotate`. Lock the result in with `assertNumQueries` on the endpoint (see [ORM performance](03-orm-performance.md)).

## 7. Views: `APIView`, generic views and mixins

```python
from django.shortcuts import get_object_or_404
from rest_framework import generics, status
from rest_framework.permissions import IsAuthenticated
from rest_framework.response import Response
from rest_framework.views import APIView

class CompleteLessonView(APIView):                       # custom flow: full control
    permission_classes = [IsAuthenticated]

    def post(self, request, lesson_id):
        lesson = get_object_or_404(Lesson.objects.select_related("module__course"), pk=lesson_id)
        enrollment = get_object_or_404(Enrollment, student=request.user, course=lesson.module.course)
        _, created = LessonProgress.objects.get_or_create(enrollment=enrollment, lesson=lesson)
        return Response({"lesson": lesson.id, "completed": True},
                        status=status.HTTP_201_CREATED if created else status.HTTP_200_OK)

class MyEnrollmentList(generics.ListAPIView):            # standard pattern: generic view
    serializer_class = EnrollmentSerializer
    permission_classes = [IsAuthenticated]

    def get_queryset(self):
        return Enrollment.objects.filter(student=self.request.user).select_related("course")
```

| Generic view | Built from | HTTP methods |
|---|---|---|
| `ListAPIView` | `ListModelMixin` | GET (collection) |
| `CreateAPIView` | `CreateModelMixin` | POST |
| `RetrieveAPIView` | `RetrieveModelMixin` | GET (one object) |
| `ListCreateAPIView` | list and create mixins | GET, POST |
| `RetrieveUpdateDestroyAPIView` | retrieve, update and destroy mixins | GET, PUT, PATCH, DELETE |

All of them extend `GenericAPIView`, which supplies `queryset`/`get_queryset()`, `serializer_class`/`get_serializer_class()`, `get_object()` (lookup, filtering and object permission checks), `get_serializer()` (passes the request in the context), pagination and filter backends. Override `perform_create()`, `perform_update()` and `perform_destroy()` to add side effects or server-set fields without rewriting the whole action. Prefer `get_queryset()` over a `queryset` class attribute whenever results depend on the request: the class attribute is built once at import time and cannot see `self.request` (DRF re-runs it with `.all()` per request so results are not cached between requests).

## 8. ViewSets, `@action` and routers

```python
from django.db.models import Count
from rest_framework import status, viewsets
from rest_framework.decorators import action
from rest_framework.response import Response

class CourseViewSet(viewsets.ModelViewSet):
    lookup_field = "slug"

    def get_queryset(self):
        qs = Course.objects.select_related("instructor").annotate(
            student_count=Count("enrollments", distinct=True)
        )
        if self.action in ("list", "retrieve", "enroll"):
            return qs.filter(status=Course.Status.PUBLISHED)
        return qs.filter(instructor=self.request.user)         # writes: only your own courses (404 otherwise)

    def get_serializer_class(self):
        return CourseDetailSerializer if self.action == "retrieve" else CourseSerializer

    def perform_create(self, serializer):
        serializer.save(instructor=self.request.user)          # never accept the owner from the client

    @action(detail=True, methods=["post"])
    def enroll(self, request, slug=None):                      # POST /courses/{slug}/enroll/
        course = self.get_object()
        _, created = Enrollment.objects.get_or_create(student=request.user, course=course)
        return Response({"enrolled": True},
                        status=status.HTTP_201_CREATED if created else status.HTTP_200_OK)
```

```python
# api/urls.py
from rest_framework.routers import DefaultRouter

router = DefaultRouter()
router.register(r"courses", CourseViewSet, basename="course")    # basename needed: no queryset attribute
router.register(r"me/enrollments", MyEnrollmentViewSet, basename="my-enrollment")
urlpatterns = router.urls
```

| URL | Methods | Viewset actions | URL name |
|---|---|---|---|
| `/courses/` | GET, POST | `list`, `create` | `course-list` |
| `/courses/{slug}/` | GET, PUT, PATCH, DELETE | `retrieve`, `update`, `partial_update`, `destroy` | `course-detail` |
| `/courses/{slug}/enroll/` | POST | `enroll` (`@action(detail=True)`) | `course-enroll` |

- A viewset groups related actions in one class; the router maps HTTP methods and URLs onto them, so `ModelViewSet` gives you five endpoints from a few lines.
- `ReadOnlyModelViewSet` offers only list and retrieve; `GenericViewSet` plus chosen mixins gives any subset (for example list, retrieve and create but no delete).
- `@action(detail=False)` adds collection routes (`/courses/featured/`), and `url_path` customizes the segment.
- `DefaultRouter` also creates an API root view and format suffixes; `SimpleRouter` does not. Nested URLs such as `/courses/{slug}/reviews/` need explicit `path()` entries or a third-party package such as drf-nested-routers.

## 9. Status codes and error handling

| Code | DRF source | Typical cause |
|---|---|---|
| 200 OK | default | GET, PUT, PATCH succeeded |
| 201 Created | `CreateModelMixin` | POST created a resource |
| 204 No Content | `DestroyModelMixin` | DELETE succeeded |
| 400 Bad Request | `ValidationError`, `ParseError` | invalid input |
| 401 Unauthorized | `NotAuthenticated`, `AuthenticationFailed` | missing or invalid credentials (token and JWT schemes send `WWW-Authenticate`) |
| 403 Forbidden | `PermissionDenied` | authenticated but not allowed; also unauthenticated requests under session auth, which has no `WWW-Authenticate` challenge |
| 404 Not Found | `Http404`, `NotFound` | missing, or filtered out of the user's queryset |
| 405 Method Not Allowed | `MethodNotAllowed` | wrong verb |
| 409 Conflict | your own exception | duplicate or state conflict |
| 429 Too Many Requests | `Throttled` | rate limit exceeded |

```python
# core/exceptions.py
from rest_framework.exceptions import APIException
from rest_framework.views import exception_handler

def api_exception_handler(exc, context):
    response = exception_handler(exc, context)    # DRF default: APIException, Http404, PermissionDenied
    if response is not None:
        response.data = {
            "error": {
                "status": response.status_code,
                "code": getattr(exc, "default_code", "error"),
                "detail": response.data,
            }
        }
    return response                               # None means unhandled: Django returns a 500

class SubscriptionRequired(APIException):
    status_code = 402                             # some teams prefer 403 here
    default_detail = "An active subscription is required for this course."
    default_code = "subscription_required"
```

A consistent error envelope lets the React client handle errors in one place (show field errors under inputs, redirect on 401, show an upgrade prompt on `subscription_required`).

## 10. The browsable API

`BrowsableAPIRenderer` renders every endpoint as an HTML page with the response, the allowed methods and forms for POST and PUT, using session login. It is excellent for development, demos and manual QA. In production many teams keep only `JSONRenderer` in `DEFAULT_RENDERER_CLASSES`, or restrict the browsable renderer to staff, and rely on OpenAPI docs instead (see [DRF advanced topics](05-drf-auth-permissions-advanced.md)).

## 11. Testing with `APIClient` and `APITestCase`

```python
from django.urls import reverse
from rest_framework import status
from rest_framework.test import APITestCase

class CourseApiTests(APITestCase):
    @classmethod
    def setUpTestData(cls):
        cls.instructor = User.objects.create_user(username="ins", email="ins@example.com", role="instructor")
        cls.student = User.objects.create_user(username="stu", email="stu@example.com", role="student")
        Course.objects.create(instructor=cls.instructor, title="DRF", slug="drf", status="published")

    def test_list_returns_published_courses(self):
        self.client.force_authenticate(user=self.student)          # skip real auth in unit tests
        response = self.client.get(reverse("course-list"))
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertEqual(response.data["count"], 1)                # paginated envelope

    def test_enroll_is_idempotent(self):
        self.client.force_authenticate(user=self.student)
        url = reverse("course-enroll", kwargs={"slug": "drf"})
        self.assertEqual(self.client.post(url).status_code, status.HTTP_201_CREATED)
        self.assertEqual(self.client.post(url).status_code, status.HTTP_200_OK)

    def test_student_cannot_edit_someone_elses_course(self):
        self.client.force_authenticate(user=self.student)
        url = reverse("course-detail", kwargs={"slug": "drf"})
        response = self.client.patch(url, {"title": "Hacked"}, format="json")
        self.assertEqual(response.status_code, status.HTTP_404_NOT_FOUND)   # filtered out of the queryset
```

`APITestCase` gives each test an `APIClient` and wraps it in a transaction. `force_authenticate()` bypasses authentication so tests focus on permissions and behavior; separate tests should exercise the real login and token endpoints. Pass `format="json"` (or set `TEST_REQUEST_DEFAULT_FORMAT`) so bodies are sent as JSON, and assert on both the status code and `response.data`. With pytest, wrap `APIClient()` in a fixture and use `@pytest.mark.django_db`.

## Common pitfalls
- `fields = "__all__"` on serializers: leaks columns you never meant to expose (a user's password hash, internal flags) and accepts writes to them.
- Taking `owner`, `student` or `instructor` IDs from the request body instead of setting them in `perform_create()` or `save(...)`.
- Serializer N+1: `SerializerMethodField` queries, dotted sources and nested serializers without matching `select_related`/`prefetch_related` in `get_queryset()`.
- Cross-field validation that reads only `attrs`, which breaks on PATCH when a field is omitted.
- Assuming DRF runs the model's `clean()`, then seeing constraint violations surface as 500 errors.
- Writable nested serializers without a transaction or clear update semantics.
- Reading `request.body` after `request.data` has been accessed (DRF raises `RawPostDataException`); webhook views that need the raw body should read it first or use a plain Django view.
- A `queryset` class attribute filtered by something request-specific; use `get_queryset()`.
- Returning 200 for creations, 500 for client mistakes, or a different error shape per endpoint.
- Forgetting `basename` when registering a viewset without a `queryset` attribute.
- Heavy business logic inside serializers or views instead of a service layer, making it hard to reuse from Celery tasks and management commands.

## How to talk about this in an interview
- Describe your LMS API in layers; a structure to adapt: public catalog endpoints (list and detail), enrollment as an `@action`, progress endpoints, instructor CRUD restricted to their own courses through `get_queryset()`, and dashboard endpoints returning aggregated data. Viewsets and routers for CRUD, `APIView` for custom flows such as completing a lesson or starting checkout.
- Mention the patterns that show maturity: list and detail serializers, server-set owners in `perform_create()`, query optimization in `get_queryset()`, a consistent error envelope for the React client, and tests per role.
- Compare with your Rails API work: a DRF serializer combines strong parameters, validation and the JSON presenter (Jbuilder or a serializer gem) in one class; viewsets plus routers resemble `resources` routes plus a controller.
- > **Fill in:** the main endpoints of your LMS API and which ones used viewsets versus custom `APIView`s.
- > **Fill in:** one tricky serializer you wrote (nested curriculum, progress percentages, dashboard aggregates) and how you kept it fast.
- > **Fill in:** how the React frontend consumed the API (fetch or axios, TanStack Query or Redux, how errors were displayed).

## Interview questions

**Q1. Why use DRF instead of returning `JsonResponse` from Django views?**
DRF standardizes everything around an API endpoint: parsing (`request.data`), validation and serialization, authentication for tokens and JWTs, permission and throttling classes, pagination, filtering, consistent error responses, generic CRUD views, routers, the browsable API and OpenAPI schemas. With plain views you rebuild all of that by hand for every endpoint.
*Follow-up:* When is plain Django enough? A couple of simple internal JSON endpoints, or a webhook receiver that needs the raw body.

**Q2. `Serializer` vs `ModelSerializer`?**
`Serializer` declares every field and requires your own `create()`/`update()`; use it for inputs that are not models (login, search parameters, action payloads) or computed outputs. `ModelSerializer` introspects the model to generate fields, validators (including unique checks) and default `create()`/`update()` implementations, so it suits CRUD.
*Follow-up:* Can you override generated fields? Yes, declare the field explicitly or tweak it with `extra_kwargs`.

**Q3. Walk me through `is_valid()` and `save()`. How does `save()` know whether to create or update?**
Constructed with `data=`, the serializer validates in `is_valid()`: field conversion and validators, then `validate_<field>()` methods, then `validate()`, filling `validated_data` or `errors`. `save()` calls `update(instance, validated_data)` if the serializer was given an instance and `create(validated_data)` otherwise, merging any keyword arguments passed to `save()`.
*Follow-up:* What does `raise_exception=True` do? Raises `ValidationError` so DRF returns a 400 with the error dict automatically.

**Q4. `read_only` vs `write_only`, and what is `HiddenField` for?**
Read-only fields appear in responses but are ignored on input (ids, timestamps, computed counts); write-only fields are accepted on input but never returned (passwords). `HiddenField(default=CurrentUserDefault())` sets a value from the request without exposing it as input, which also lets uniqueness validators include the user.
*Follow-up:* Where else can you mark fields read-only? `Meta.read_only_fields` or `extra_kwargs = {"field": {"read_only": True}}`.

**Q5. Where do you put validation in DRF?**
Single-field rules go in field arguments, `validators=[...]` or `validate_<field>()`; rules spanning several fields go in `validate()`. Uniqueness comes from model-derived validators backed by database constraints, and anything needing the user reads `self.context["request"]`. Workflow rules that touch several models belong in a service layer.
*Follow-up:* Does DRF call the model's `clean()`? No, so keep important rules in the serializer or the database.

**Q6. How do nested serializers work, and how do you make them writable?**
Using a serializer as a field (`lessons = LessonSerializer(many=True)`) renders related objects inside the parent's output, and it is read-only for writes by default. To accept nested input you override `create()` and `update()`, pop the nested data, create or update children yourself inside a transaction, and define clear semantics for updates.
*Follow-up:* What is the alternative? Separate child endpoints, which keep validation and permissions simple.

**Q7. What is `SerializerMethodField`, and why can it be dangerous?**
A read-only field whose value comes from a `get_<name>(self, obj)` method, useful for computed or request-dependent values. It runs once per object, so any query inside it becomes an N+1 on list endpoints. Precompute values in `get_queryset()` with annotations, `Exists` subqueries or prefetches, and keep the method a cheap attribute read.
*Follow-up:* How do you access the request inside it? Through `self.context["request"]`.

**Q8. `APIView` vs generic views vs viewsets: how do you choose?**
`APIView` gives full control with explicit `get()`/`post()` methods, good for custom flows. Generic views (`ListCreateAPIView` and friends) combine mixins with `GenericAPIView` to remove CRUD boilerplate for one URL pattern. Viewsets group all actions for a resource in one class and let routers generate the URLs, ideal for standard REST resources.
*Follow-up:* What does `GenericAPIView` add over `APIView`? `get_queryset()`, `get_object()`, `get_serializer()`, pagination and filter backends.

**Q9. What does `ModelViewSet` give you, and how do you restrict it?**
`list`, `create`, `retrieve`, `update`, `partial_update` and `destroy` in one class. Restrict it by composing `GenericViewSet` with only the mixins you need, using `ReadOnlyModelViewSet`, setting `http_method_names`, or returning different permissions per `self.action` in `get_permissions()`. Scope `get_queryset()` so users only reach their own objects.
*Follow-up:* How do you use different serializers per action? Override `get_serializer_class()` and branch on `self.action`.

**Q10. What does `@action` do?**
It adds a custom route to a viewset: `@action(detail=True, methods=["post"])` on `enroll` creates `POST /courses/{lookup}/enroll/` named `course-enroll`, and `detail=False` creates a collection-level route. The action gets the same authentication, permission and throttling machinery as built-in actions, and can override them per action.
*Follow-up:* How do you get the object inside a detail action? Call `self.get_object()`, which also runs object permission checks.

**Q11. How do routers work? What is `basename`?**
A router inspects registered viewsets and generates URL patterns that map HTTP methods to actions (`GET /courses/` to `list`, `PATCH /courses/{pk}/` to `partial_update`) plus routes for `@action` methods. `basename` prefixes the URL names (`course-list`, `course-detail`) and is required when the viewset has no `queryset` attribute to derive it from. `DefaultRouter` adds an API root view; `SimpleRouter` does not.
*Follow-up:* How do you reverse a router URL? `reverse("course-detail", kwargs={"slug": "drf"})`.

**Q12. Why override `perform_create()` instead of `create()`?**
`perform_create(serializer)` is the narrow hook that saves the validated serializer, so it is the right place to inject server-side values (`serializer.save(instructor=self.request.user)`) or trigger side effects, while DRF keeps handling validation, headers and the 201 response. Override `create()` only when you need to change the response itself.
*Follow-up:* Which hooks run on update and delete? `perform_update()` and `perform_destroy()`.

**Q13. Which status codes should an API return, and what is the difference between 401 and 403?**
200 for successful reads and updates, 201 for creations, 204 for deletions, 400 for validation errors, 404 for missing or inaccessible objects, 409 for conflicts and 429 for throttling. 401 means "we do not know who you are" (missing or invalid credentials), while 403 means "we know who you are, and you may not do this".
*Follow-up:* Why might DRF return 403 for an anonymous user? With session authentication there is no `WWW-Authenticate` challenge to send, so DRF uses 403 instead of 401.

**Q14. How do you customize error responses?**
Point `EXCEPTION_HANDLER` at a function that calls DRF's default `exception_handler` and reshapes `response.data` into one envelope (status, code, details); unhandled exceptions return `None` and become 500s. Define custom `APIException` subclasses with `status_code`, `default_detail` and `default_code` for domain errors such as "subscription required".
*Follow-up:* What happens to a database `IntegrityError`? It is not an `APIException`, so catch it in the view or service and raise a `ValidationError` or a 409.

**Q15. `request.data` vs `request.query_params` vs Django's `request.POST`?**
`request.data` is DRF's parsed body for POST, PUT and PATCH in any supported format (JSON, form, multipart). `request.query_params` is the query string, a clearer alias for `request.GET`. Django's `request.POST` only contains form-encoded POST data, so it is empty for JSON requests.
*Follow-up:* Where do uploaded files appear? In `request.data` (and `request.FILES`) when `MultiPartParser` handles the request.

**Q16. How do you test DRF APIs?**
Use `APITestCase` (or pytest with an `APIClient` fixture), create data with factories, authenticate with `force_authenticate()` for permission tests, send JSON with `format="json"`, and assert status codes and bodies. Cover each role per endpoint (allowed, forbidden, not found), validation errors, idempotency of actions, and query counts with `assertNumQueries` on list endpoints.
*Follow-up:* When do you test real authentication? In dedicated tests for the login, token and refresh endpoints.

**Q17. What is the browsable API, and should it be enabled in production?**
An HTML renderer that lets you explore endpoints in a browser, see responses and submit forms using session login, which is great for development and QA. It is not a vulnerability by itself because permissions still apply, but many teams disable it in production (JSON renderer only) and publish OpenAPI docs instead.
*Follow-up:* How do you disable it? Set `DEFAULT_RENDERER_CLASSES` to only `JSONRenderer` in production settings.

## Further reading
- [Django REST framework](https://www.django-rest-framework.org/)
- [Serializers](https://www.django-rest-framework.org/api-guide/serializers/)
- [Generic views](https://www.django-rest-framework.org/api-guide/generic-views/) and [ViewSets](https://www.django-rest-framework.org/api-guide/viewsets/)
- [Routers](https://www.django-rest-framework.org/api-guide/routers/)
- [Testing](https://www.django-rest-framework.org/api-guide/testing/)
