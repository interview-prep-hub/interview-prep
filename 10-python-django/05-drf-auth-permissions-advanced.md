# DRF Authentication, Permissions and Advanced Topics

> **Tier:** T1 (know cold) · **Related:** [Django REST Framework](04-django-rest-framework.md), [Advanced Django](06-django-advanced.md), [LMS project patterns](07-lms-project-patterns.md), [Django rapid-fire questions](99-interview-questions.md), [Authentication and authorization](../04-web-fundamentals/06-authentication-authorization.md), [Web security and OWASP](../04-web-fundamentals/07-web-security-owasp.md), [React state management](../06-react/03-state-management.md), [Data fetching with TanStack Query](../06-react/04-data-fetching-tanstack-query.md)
> **On your resume:** The PureLogics LMS had role-based dashboards for students, instructors and admins, a React frontend calling DRF, and paid subscriptions. That is authentication, role-based permissions, per-user querysets, CORS and token handling in one project.

## Quick revision
- **Authentication** identifies the caller (`request.user`, `request.auth`), **permissions** decide whether the request may proceed, **throttling** limits its rate. They run in that order before the handler; object permissions run later, in `get_object()`.
- `SessionAuthentication` uses Django's session cookie and enforces CSRF on unsafe methods: ideal for a same-site SPA and the browsable API.
- `TokenAuthentication` (`rest_framework.authtoken`): one opaque database token per user, sent as `Authorization: Token <key>`, with no built-in expiry.
- JWT through `djangorestframework-simplejwt`: a short-lived access token sent as `Authorization: Bearer <token>` plus a longer-lived refresh token. `ROTATE_REFRESH_TOKENS` and `BLACKLIST_AFTER_ROTATION` (with the `token_blacklist` app) enable rotation and revocation.
- JWTs are signed, not encrypted: anyone can read the payload. Access tokens cannot be revoked before they expire, so keep them short.
- Built-in permissions: `AllowAny`, `IsAuthenticated`, `IsAdminUser` (`is_staff`), `IsAuthenticatedOrReadOnly`, `DjangoModelPermissions`, `DjangoObjectPermissions`.
- Custom permissions subclass `BasePermission` and implement `has_permission(request, view)` and/or `has_object_permission(request, view, obj)`. Compose with `&`, `|` and `~`.
- `has_object_permission` never runs for list endpoints, so lists must be filtered in `get_queryset()`.
- **IDOR prevention**: scope every queryset to the caller, set owners server-side, validate related IDs against what the user may access, and return 404 for other people's objects.
- Throttles (`AnonRateThrottle`, `UserRateThrottle`, `ScopedRateThrottle`) count requests in Django's cache, so use a shared cache such as Redis across processes. Exceeding a rate returns 429.
- Pagination: `PageNumberPagination` (`?page=`), `LimitOffsetPagination` (`?limit=&offset=`), `CursorPagination` (opaque cursor, stable ordering, no total count, fast on huge tables).
- Filtering: django-filter's `DjangoFilterBackend`, `SearchFilter` (`?search=`) and `OrderingFilter` (`?ordering=-price`) with an explicit `ordering_fields` list.
- Versioning: `URLPathVersioning` (`/api/v1/`), `NamespaceVersioning`, `AcceptHeaderVersioning`, `QueryParameterVersioning`; read `request.version`.
- CORS for a React app: django-cors-headers with `CorsMiddleware` near the top, `CORS_ALLOWED_ORIGINS`, and `CORS_ALLOW_CREDENTIALS` only when cookies are needed. CORS is not authentication.
- OpenAPI docs with drf-spectacular (`SpectacularAPIView`, Swagger UI, Redoc, `@extend_schema`); generate TypeScript types for the frontend from the schema.
- Uploads: `MultiPartParser` with `FileField`/`ImageField`, size and type validation; large videos go straight to object storage through presigned URLs.
- React token storage: access token in memory and refresh token in an httpOnly, Secure, SameSite cookie is the common recommendation; localStorage is readable by any XSS payload.

## 1. How DRF runs authentication, permissions and throttling

For every request, `APIView.initial()` runs three checks before your handler:
1. **Authentication**: each class in `authentication_classes` is tried in order. The first one that returns `(user, auth)` wins; returning `None` passes to the next; raising `AuthenticationFailed` stops the request. If none succeeds, `request.user` is an `AnonymousUser`.
2. **Permissions**: every class in `permission_classes` must return `True` from `has_permission()`. A failure becomes 401 when the user is not authenticated and the auth scheme can send a challenge, otherwise 403.
3. **Throttles**: every class in `throttle_classes` must allow the request, or the response is 429.

Later, `get_object()` calls `check_object_permissions()`, which runs `has_object_permission()` on the fetched object. If you fetch objects yourself in an `APIView`, call `self.check_object_permissions(request, obj)` explicitly.

## 2. Authentication classes

| | Session | DRF Token | JWT (simplejwt) |
|---|---|---|---|
| Credential | `sessionid` cookie | opaque key stored in the database | signed token (header.payload.signature) |
| Sent as | cookie, automatically | `Authorization: Token <key>` | `Authorization: Bearer <access>` |
| Server state | session row or cache entry | token row | none for access tokens (a blacklist table for refresh tokens) |
| Expiry | session age | none built in | short access, longer refresh |
| Revocation | delete the session | delete the token | wait for expiry; blacklist refresh tokens |
| CSRF | required for unsafe methods | not needed (header) | not needed when sent in a header |
| Good for | same-site web apps, admin, browsable API | simple internal or server-to-server clients | SPAs and mobile apps, several services |

```python
# config/settings.py
from datetime import timedelta

INSTALLED_APPS += [
    "rest_framework",
    "rest_framework_simplejwt.token_blacklist",     # only needed for blacklisting (run migrate)
]

REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": [
        "rest_framework_simplejwt.authentication.JWTAuthentication",   # React and mobile clients
        "rest_framework.authentication.SessionAuthentication",          # staff in the browsable API
    ],
    "DEFAULT_PERMISSION_CLASSES": ["rest_framework.permissions.IsAuthenticated"],
}

SIMPLE_JWT = {
    "ACCESS_TOKEN_LIFETIME": timedelta(minutes=15),
    "REFRESH_TOKEN_LIFETIME": timedelta(days=7),
    "ROTATE_REFRESH_TOKENS": True,          # each refresh returns a new refresh token
    "BLACKLIST_AFTER_ROTATION": True,       # and blacklists the old one
    "AUTH_HEADER_TYPES": ("Bearer",),
}
```

```python
# accounts/auth.py
from rest_framework import status
from rest_framework.response import Response
from rest_framework.views import APIView
from rest_framework_simplejwt.exceptions import TokenError
from rest_framework_simplejwt.serializers import TokenObtainPairSerializer
from rest_framework_simplejwt.tokens import RefreshToken
from rest_framework_simplejwt.views import TokenObtainPairView

class LmsTokenObtainPairSerializer(TokenObtainPairSerializer):
    @classmethod
    def get_token(cls, user):
        token = super().get_token(user)
        token["role"] = user.role          # readable by the client: a UI hint, never the source of truth
        return token

class LmsTokenObtainPairView(TokenObtainPairView):
    serializer_class = LmsTokenObtainPairSerializer

class LogoutView(APIView):
    def post(self, request):
        try:
            RefreshToken(request.data["refresh"]).blacklist()   # requires the token_blacklist app
        except (KeyError, TokenError):
            return Response(status=status.HTTP_400_BAD_REQUEST)
        return Response(status=status.HTTP_205_RESET_CONTENT)

# config/urls.py
from rest_framework_simplejwt.views import TokenRefreshView

urlpatterns = [
    path("api/auth/token/", LmsTokenObtainPairView.as_view(), name="token_obtain_pair"),  # credentials -> access + refresh
    path("api/auth/token/refresh/", TokenRefreshView.as_view(), name="token_refresh"),    # refresh -> new access (+ refresh)
    path("api/auth/logout/", LogoutView.as_view(), name="logout"),
]
```

How JWT works: the token is three base64url parts, `header.payload.signature`. simplejwt signs it with `SECRET_KEY` by default (HS256); on each request `JWTAuthentication` verifies the signature and the `exp` claim, then loads the user by the `user_id` claim. Because the user comes from the database on every request, permission checks on `request.user.role` see the current role, while the `role` claim inside an old token can be stale. The payload is readable by anyone, so never put secrets in it. Short access lifetimes limit the damage of a stolen token; rotation plus blacklisting lets you revoke refresh tokens at logout or when a password changes.

## 3. Permissions, including role-based access

| Class | Allows |
|---|---|
| `AllowAny` | everyone |
| `IsAuthenticated` | any logged-in user |
| `IsAdminUser` | users with `is_staff` |
| `IsAuthenticatedOrReadOnly` | anyone for safe methods, logged-in users for writes |
| `DjangoModelPermissions` | users holding the model's add, change or delete permission (needs a `queryset`) |
| `DjangoObjectPermissions` | per-object permissions through a backend such as django-guardian |

```python
# accounts/permissions.py
from rest_framework.permissions import SAFE_METHODS, BasePermission

class IsStudent(BasePermission):
    def has_permission(self, request, view):
        return request.user.is_authenticated and request.user.role == "student"

class IsInstructor(BasePermission):
    message = "Only instructors can do this."

    def has_permission(self, request, view):
        return request.user.is_authenticated and request.user.role == "instructor"

class IsPlatformAdmin(BasePermission):
    def has_permission(self, request, view):
        return request.user.is_authenticated and (request.user.role == "admin" or request.user.is_staff)

class IsCourseOwnerOrReadOnly(BasePermission):
    def has_object_permission(self, request, view, obj):
        if request.method in SAFE_METHODS:          # GET, HEAD, OPTIONS
            return True
        return obj.instructor_id == request.user.id
```

```python
from rest_framework.permissions import AllowAny, IsAuthenticated

class CourseViewSet(viewsets.ModelViewSet):
    def get_permissions(self):                      # different rules per action
        if self.action in ("list", "retrieve"):
            return [AllowAny()]
        if self.action == "create":
            return [(IsInstructor | IsPlatformAdmin)()]
        if self.action == "enroll":
            return [IsStudent()]
        return [IsAuthenticated(), IsCourseOwnerOrReadOnly()]   # update, partial_update, destroy
```

- `has_permission()` runs on every request, before any object is loaded: use it for role checks ("is this an instructor?").
- `has_object_permission()` runs only when `get_object()` is called (retrieve, update, destroy and detail actions): use it for ownership rules.
- Composition: `permission_classes = [IsAuthenticated & (IsInstructor | IsPlatformAdmin)]`, and `~` negates. Every class in the list must pass, so a plain list is already an AND.
- Role checks belong on the server. The React app can hide buttons by role, but the API must enforce every rule, because anyone can call it directly.

## 4. Per-user querysets: preventing IDOR

An **IDOR** (insecure direct object reference) happens when changing an ID in a URL or body exposes someone else's data: `GET /api/enrollments/42/` returns another student's enrollment because the view only checked "is logged in".

```python
# Vulnerable: any logged-in user can read any enrollment by guessing ids
class EnrollmentViewSet(viewsets.ReadOnlyModelViewSet):
    queryset = Enrollment.objects.all()
    serializer_class = EnrollmentSerializer

# Fixed: each role sees only its own slice; other ids return 404
class EnrollmentViewSet(viewsets.ReadOnlyModelViewSet):
    serializer_class = EnrollmentSerializer

    def get_queryset(self):
        user = self.request.user
        qs = Enrollment.objects.select_related("course", "student")
        if user.role == "admin":
            return qs
        if user.role == "instructor":
            return qs.filter(course__instructor=user)      # enrollments in my courses
        return qs.filter(student=user)                     # my own enrollments

class ReviewSerializer(serializers.ModelSerializer):
    class Meta:
        model = Review
        fields = ["id", "course", "rating", "body"]

    def validate_course(self, course):                     # related IDs in the body need checks too
        user = self.context["request"].user
        if not Enrollment.objects.filter(student=user, course=course).exists():
            raise serializers.ValidationError("You can only review courses you are enrolled in.")
        return course
```

Checklist: scope `get_queryset()` for every action (a 404 also avoids revealing that the object exists); set ownership fields in `perform_create()`; validate every related ID in input against what the caller may access; add object permissions for finer rules; and write a test where user A requests user B's object. Random UUIDs make IDs harder to guess but are not authorization.

## 5. Throttling

```python
REST_FRAMEWORK = {
    "DEFAULT_THROTTLE_CLASSES": [
        "rest_framework.throttling.AnonRateThrottle",
        "rest_framework.throttling.UserRateThrottle",
    ],
    "DEFAULT_THROTTLE_RATES": {
        "anon": "60/minute",
        "user": "1000/hour",
        "login": "5/minute",                 # for views with throttle_scope = "login"
    },
}

from rest_framework.throttling import ScopedRateThrottle

class LmsTokenObtainPairView(TokenObtainPairView):
    serializer_class = LmsTokenObtainPairSerializer
    throttle_classes = [ScopedRateThrottle]
    throttle_scope = "login"                 # slows down password guessing
```

- Anonymous throttles key on the client IP and user throttles on the user ID. Behind a load balancer, set `NUM_PROXIES` so DRF reads the right address from `X-Forwarded-For`.
- Request history lives in Django's cache. The default local-memory cache is per process, so with several Gunicorn workers each one counts separately; use Redis.
- A throttled request gets 429 with a `Retry-After` header. Throttling is abuse and fairness control, not a security boundary: add rate limits at Nginx or the gateway too, and lock accounts or add CAPTCHAs in the auth flow for credential stuffing.

## 6. Pagination

```python
from rest_framework.pagination import CursorPagination, PageNumberPagination

class StandardPagination(PageNumberPagination):
    page_size = 20
    page_size_query_param = "page_size"      # clients may ask for ?page_size=50
    max_page_size = 100                      # but never more than this

class ActivityFeedPagination(CursorPagination):
    page_size = 50
    ordering = "-created_at"                 # needs a stable ordering, ideally indexed

class CourseViewSet(viewsets.ModelViewSet):
    pagination_class = StandardPagination
```

| | PageNumber | LimitOffset | Cursor |
|---|---|---|---|
| Query | `?page=3` | `?limit=20&offset=40` | `?cursor=<opaque>` |
| Response | `count`, `next`, `previous`, `results` | same | `next`, `previous`, `results` (no count) |
| Jump to page N | yes | yes | no |
| Deep pages on big tables | slow (`OFFSET` plus `COUNT(*)`) | same | fast (indexed `WHERE` on the ordering field) |
| Rows inserted while paging | items shift or repeat | same | stable |
| Best for | numbered pages, admin-style tables | clients choosing arbitrary windows | infinite scroll, feeds, very large tables |

Every paginated queryset needs a deterministic `order_by()`; otherwise rows can repeat or vanish between pages (Django warns with `UnorderedObjectListWarning`).

## 7. Filtering, search and ordering

```python
# settings: "DEFAULT_FILTER_BACKENDS": [
#     "django_filters.rest_framework.DjangoFilterBackend",
#     "rest_framework.filters.SearchFilter",
#     "rest_framework.filters.OrderingFilter",
# ]   (and "django_filters" in INSTALLED_APPS)
import django_filters

class CourseFilter(django_filters.FilterSet):
    min_price = django_filters.NumberFilter(field_name="price", lookup_expr="gte")
    max_price = django_filters.NumberFilter(field_name="price", lookup_expr="lte")
    category = django_filters.CharFilter(field_name="category__slug")

    class Meta:
        model = Course
        fields = ["level"]                    # declared filters above are included automatically

class CatalogViewSet(viewsets.ReadOnlyModelViewSet):
    serializer_class = CourseListSerializer
    filterset_class = CourseFilter            # ?level=beginner&category=python&max_price=20
    search_fields = ["title", "description", "instructor__username"]   # ?search=django
    ordering_fields = ["price", "published_at", "title"]               # ?ordering=-price
    ordering = ["-published_at"]                                       # default order

    def get_queryset(self):
        return Course.objects.filter(status="published").select_related("instructor")
```

Filter backends refine whatever `get_queryset()` returns, so they never widen access beyond the user's scope. Always whitelist `ordering_fields` (sorting by any field can expose information and hit unindexed columns). `SearchFilter` runs `icontains` across the listed fields, which is fine for small catalogs; for larger ones use PostgreSQL full-text search or a search engine (see [LMS project patterns](07-lms-project-patterns.md)).

## 8. Versioning

```python
REST_FRAMEWORK = {
    "DEFAULT_VERSIONING_CLASS": "rest_framework.versioning.URLPathVersioning",
    "DEFAULT_VERSION": "v1",
    "ALLOWED_VERSIONS": ["v1", "v2"],
}

# api/urls.py
urlpatterns = [path("api/<str:version>/", include(router.urls))]

class CourseViewSet(viewsets.ModelViewSet):
    def get_serializer_class(self):
        return CourseSerializerV2 if self.request.version == "v2" else CourseSerializer
```

URL path versioning (`/api/v1/`) is the most visible and cache-friendly; `AcceptHeaderVersioning` (`Accept: application/json; version=2`), `QueryParameterVersioning`, `NamespaceVersioning` and `HostNameVersioning` also exist. Version only for breaking changes: adding optional fields is backward compatible. Publish a deprecation timeline before removing an old version, and remember that mobile apps in the wild keep calling old versions for months.

## 9. CORS for a React frontend

```python
INSTALLED_APPS += ["corsheaders"]
MIDDLEWARE = [
    "corsheaders.middleware.CorsMiddleware",            # as high as possible, before CommonMiddleware
    "django.middleware.security.SecurityMiddleware",
    # ... the rest
]
CORS_ALLOWED_ORIGINS = ["http://localhost:5173", "https://app.example.com"]
CORS_ALLOW_CREDENTIALS = True                            # only if the browser must send cookies
CSRF_TRUSTED_ORIGINS = ["https://app.example.com"]       # cross-origin unsafe requests that use the session cookie
```

Browsers stop JavaScript on one origin from reading responses from another unless the server opts in with `Access-Control-Allow-Origin`, and they send a preflight `OPTIONS` request for non-simple requests (JSON bodies, an `Authorization` header). CORS protects users' browsers, not your API: curl and server-side code ignore it, so it is never a replacement for authentication. Avoid `CORS_ALLOW_ALL_ORIGINS = True` in production, especially with credentials. In development, a Vite dev-server proxy can serve the API under the same origin and avoid CORS entirely.

## 10. OpenAPI documentation with drf-spectacular

```python
INSTALLED_APPS += ["drf_spectacular"]
REST_FRAMEWORK = {"DEFAULT_SCHEMA_CLASS": "drf_spectacular.openapi.AutoSchema"}

from drf_spectacular.utils import extend_schema
from drf_spectacular.views import SpectacularAPIView, SpectacularRedocView, SpectacularSwaggerView

urlpatterns += [
    path("api/schema/", SpectacularAPIView.as_view(), name="schema"),
    path("api/docs/", SpectacularSwaggerView.as_view(url_name="schema"), name="swagger-ui"),
    path("api/redoc/", SpectacularRedocView.as_view(url_name="schema"), name="redoc"),
]

class CheckoutView(APIView):
    @extend_schema(request=CheckoutRequestSerializer, responses={201: CheckoutResponseSerializer})
    def post(self, request): ...
```

drf-spectacular introspects serializers and views to produce an OpenAPI 3 schema; `@extend_schema` fills gaps for `APIView`s and custom actions. Running `python manage.py spectacular --file schema.yml` in CI and generating TypeScript types from the schema (for example with openapi-typescript) keeps the React client's types in sync with the backend.

## 11. File uploads

```python
from rest_framework import generics, serializers
from rest_framework.parsers import FormParser, MultiPartParser

class AvatarSerializer(serializers.ModelSerializer):
    class Meta:
        model = Profile
        fields = ["avatar"]                       # an ImageField, stored through django-storages (S3)

    def validate_avatar(self, file):
        if file.size > 2 * 1024 * 1024:
            raise serializers.ValidationError("Maximum size is 2 MB.")
        if file.content_type not in {"image/jpeg", "image/png", "image/webp"}:
            raise serializers.ValidationError("Use JPEG, PNG or WebP.")
        return file

class AvatarView(generics.UpdateAPIView):        # PUT/PATCH /api/me/avatar/ with multipart form data
    serializer_class = AvatarSerializer
    parser_classes = [MultiPartParser, FormParser]

    def get_object(self):
        return self.request.user.profile          # "me" endpoints need no ID, so no IDOR
```

The `content_type` comes from the client, so treat it as a hint; `ImageField` also checks that Pillow can open the file. Large files such as course videos should not stream through Django workers at all: an endpoint returns a short-lived presigned upload URL for object storage, the browser uploads directly, and a follow-up call (or a storage event) records the file and starts processing.

## 12. React and DRF together

| Token storage option | XSS exposure | CSRF exposure | Notes |
|---|---|---|---|
| Access token in memory, refresh token in an httpOnly, Secure, SameSite cookie | access token only while the page is open | the cookie-based refresh endpoint needs SameSite and CSRF care | the common SPA recommendation; refresh silently on page load |
| Both tokens in `localStorage` | high: any injected script can read them | none | simplest, common in tutorials, risky |
| Django session cookie (`SessionAuthentication`) | the cookie is httpOnly | send the CSRF token as `X-CSRFToken` | great when the SPA and API share a site |

```ts
// api.ts: attach the access token, refresh once on 401, then retry
let accessToken: string | null = null;
let refreshing: Promise<string> | null = null;

export function setAccessToken(token: string | null) {
  accessToken = token;
}

async function refreshAccessToken(): Promise<string> {
  const res = await fetch("/api/auth/token/refresh/", {
    method: "POST",
    credentials: "include", // the refresh token travels in an httpOnly cookie
  });
  if (!res.ok) throw new Error("session expired");
  const data: { access: string } = await res.json();
  accessToken = data.access;
  return data.access;
}

export async function apiFetch(url: string, init: RequestInit = {}): Promise<Response> {
  const withAuth = (token: string | null): RequestInit => {
    const headers = new Headers(init.headers);
    if (token) headers.set("Authorization", `Bearer ${token}`);
    return { ...init, headers };
  };

  const res = await fetch(url, withAuth(accessToken));
  if (res.status !== 401) return res;

  if (!refreshing) {
    refreshing = refreshAccessToken().finally(() => {
      refreshing = null; // concurrent 401s share one refresh call
    });
  }
  const token = await refreshing; // if the refresh fails, the error propagates: log the user out
  return fetch(url, withAuth(token));
}
```

- simplejwt's stock `TokenRefreshView` reads the refresh token from the JSON body. To keep it in a cookie, add a small view that reads `request.COOKIES`, calls the refresh serializer, and sets the rotated token with `response.set_cookie(..., httponly=True, secure=True, samesite="Lax")`.
- Tokens sent in the `Authorization` header are never attached automatically by the browser, so those endpoints are not CSRF targets; XSS is the main threat, so avoid `dangerouslySetInnerHTML` with user content and consider a Content Security Policy.
- With session authentication, make sure the `csrftoken` cookie is set (for example through a view decorated with `ensure_csrf_cookie`), then send it as `X-CSRFToken` with `credentials: "include"` on unsafe requests.
- Route guards in React (see [routing](../06-react/08-routing.md)) are UX only; DRF permissions remain the real enforcement.

## Common pitfalls
- Relying on `has_object_permission()` to protect list endpoints; it never runs there.
- `queryset = Model.objects.all()` plus `IsAuthenticated` on user-owned data: a textbook IDOR.
- Trusting the `role` claim in a JWT, or the React UI, as the authorization source.
- Long-lived access tokens with no revocation plan; enabling `BLACKLIST_AFTER_ROTATION` without installing and migrating the blacklist app.
- Tokens in `localStorage` on a site with any XSS exposure.
- `CORS_ALLOW_ALL_ORIGINS = True` in production, or `CorsMiddleware` placed below middleware that returns early responses.
- Believing CORS protects the API from scripts and other servers.
- Throttling backed by the per-process local-memory cache across several workers, or a wrong `NUM_PROXIES` behind a load balancer.
- `page_size_query_param` without `max_page_size`, letting a client request a million rows.
- Unrestricted `ordering_fields`, exposing sorts on sensitive or unindexed columns.
- Uploads with no size or type validation, or large videos streamed through Django workers.
- `PageNumberPagination` on huge, constantly growing tables where deep pages are slow and items shift.

## How to talk about this in an interview
- Explain your LMS access model; if it matches what you built: a `role` on the user (student, instructor, admin), permission classes per role, `get_queryset()` scoped per role so each dashboard only sees its own data, and subscription or enrollment checks before content is served. Say clearly that the React route guards were UX and the API enforced everything.
- Name the IDOR rule out loud ("every queryset is scoped to the caller, and foreign IDs return 404"); interviewers listen for it.
- Bridge to your Rails work: policy libraries such as Pundit follow the same idea, with a policy scope playing the role of `get_queryset()`, and the multi-tenant, org-scoped access control you worked on at Devntech is the same problem one level up.
- > **Fill in:** which authentication the LMS used (simplejwt, DRF tokens or sessions), where the React app stored tokens, and how refresh and logout worked.
- > **Fill in:** how roles were modeled and checked (role field, groups, custom permission classes) and one permission bug you prevented or fixed.
- > **Fill in:** the pagination and filtering you exposed on the course catalog, and your CORS setup for local development versus production.

## Interview questions

**Q1. How do authentication, permissions and throttling fit together in DRF?**
For every request, `initial()` authenticates (the first authentication class that returns a user wins; otherwise the user is anonymous), then checks every permission class's `has_permission()`, then every throttle. Object-level checks happen later when `get_object()` calls `has_object_permission()`. Authentication answers "who are you", permissions "may you do this", throttling "how often".
*Follow-up:* What if authentication fails with a bad token? `AuthenticationFailed` stops the request immediately with 401 (or 403 for schemes without a challenge header).

**Q2. Session, Token or JWT authentication: when would you use each?**
Sessions suit a same-site web app or SPA: an httpOnly cookie, server-side state, easy revocation, but CSRF protection required. DRF's token auth is a simple opaque token per user stored in the database, with no expiry, fine for internal clients. JWT (simplejwt) suits SPAs and mobile apps and multiple services: stateless short-lived access tokens with refresh tokens, at the cost of harder revocation.
*Follow-up:* Which did you use and why? Answer from your LMS (see the fill-in prompt above) and explain the trade-off you accepted.

**Q3. How does JWT authentication work with simplejwt?**
The client posts credentials to `TokenObtainPairView` and receives an access token (minutes) and a refresh token (days). It sends `Authorization: Bearer <access>` on each request; `JWTAuthentication` verifies the signature and expiry and loads the user from the `user_id` claim. When the access token expires, the client posts the refresh token to `TokenRefreshView` for a new access token (and, with rotation, a new refresh token).
*Follow-up:* Is the payload secret? No, it is base64url-encoded and signed, not encrypted; never put sensitive data in it.

**Q4. How do you log a user out or revoke JWTs?**
Access tokens are stateless, so you cannot revoke them individually; keep them short-lived. Revoke refresh tokens by blacklisting them (`RefreshToken(token).blacklist()` with the `token_blacklist` app), enable `ROTATE_REFRESH_TOKENS` and `BLACKLIST_AFTER_ROTATION`, and clear tokens on the client. For "log out everywhere", blacklist all of the user's outstanding tokens or change a per-user value that tokens are checked against.
*Follow-up:* What happens to a stolen access token? It works until it expires, which is why lifetimes are measured in minutes.

**Q5. Where should a React app store tokens?**
A common recommendation is to keep the access token in memory and the refresh token in an httpOnly, Secure, SameSite cookie, refreshing silently on page load. `localStorage` is simple but readable by any injected script. If the SPA and API share a site, Django session authentication with CSRF tokens avoids handling tokens in JavaScript at all.
*Follow-up:* What new risk does the refresh cookie introduce? The refresh endpoint becomes cookie-authenticated, so protect it with SameSite and CSRF measures and rotate refresh tokens.

**Q6. What is the difference between `has_permission` and `has_object_permission`?**
`has_permission(request, view)` runs for every request before any object is loaded, so it suits role and authentication checks. `has_object_permission(request, view, obj)` runs only when `get_object()` fetches a single object (retrieve, update, destroy, detail actions) and suits ownership rules. It never runs on list endpoints, which must be filtered in `get_queryset()`.
*Follow-up:* Does `has_object_permission` run if `has_permission` fails? No, the request is rejected first.

**Q7. How would you implement role-based permissions for students, instructors and admins?**
Store a `role` on the custom user model (or use groups), write small permission classes such as `IsStudent`, `IsInstructor` and `IsPlatformAdmin`, and choose them per action in `get_permissions()`. Combine with object permissions for ownership (instructors edit only their own courses) and scope `get_queryset()` per role so each dashboard sees only its slice. Test a matrix of role by endpoint by expected status.
*Follow-up:* Role field or Django groups? A role field is simple for a few fixed roles; groups and permissions scale better to many fine-grained capabilities managed by admins.

**Q8. How do you combine permissions or vary them per action?**
Permission classes in a list must all pass (AND), and DRF supports `&`, `|` and `~` operators: `[IsAuthenticated & (IsInstructor | IsPlatformAdmin)]`. To vary by action, override `get_permissions()` and branch on `self.action`, returning instances. `@action` also accepts `permission_classes=[...]` for custom routes.
*Follow-up:* Why return instances from `get_permissions()`? DRF calls `has_permission()` on the returned objects, so they must be instantiated.

**Q9. What is IDOR, and how do you prevent it in DRF?**
Insecure direct object reference: a user changes an ID in the URL or body and gets someone else's data because the code checked authentication but not ownership. Prevent it by scoping `get_queryset()` to the caller for every action (others' objects return 404), setting owner fields server-side in `perform_create()`, validating related IDs in serializers against what the user may access, and testing cross-user access.
*Follow-up:* Do UUIDs fix IDOR? They make IDs harder to guess, but leaked or shared IDs still work without real authorization checks.

**Q10. How does throttling work, and what are its limits?**
Throttle classes count requests per key (IP for anonymous users, user ID for authenticated ones, or a named scope) in Django's cache and reject with 429 and `Retry-After` when a rate such as `"5/minute"` is exceeded. With several processes the cache must be shared (Redis), and behind proxies `NUM_PROXIES` must be right. It is a fairness and abuse control, not a security boundary.
*Follow-up:* How do you rate-limit only the login endpoint? `ScopedRateThrottle` with `throttle_scope = "login"` and a `"login"` rate in settings.

**Q11. Compare DRF's pagination styles.**
Page number (`?page=3`) and limit/offset (`?limit=20&offset=40`) return a total count and allow jumping to any page, but deep pages get slow on big tables (`OFFSET` scans and `COUNT(*)`) and items shift when rows are inserted. Cursor pagination returns opaque next/previous links based on an indexed ordering field: fast at any depth and stable under inserts, but no total count and no page jumps, which suits infinite scroll and feeds.
*Follow-up:* What does every paginated queryset need? A deterministic ordering, otherwise rows can repeat or disappear between pages.

**Q12. How do you implement filtering, search and ordering in DRF?**
Add filter backends: django-filter's `DjangoFilterBackend` with `filterset_fields` or a `FilterSet` (ranges such as `min_price`), `SearchFilter` with `search_fields` for `?search=`, and `OrderingFilter` with a whitelisted `ordering_fields` for `?ordering=-price`. Backends refine `get_queryset()`, so they respect the user's scope.
*Follow-up:* When is `SearchFilter` not enough? On large catalogs, where `icontains` scans are slow; use PostgreSQL full-text search or a search engine.

**Q13. How do you version a DRF API?**
Set `DEFAULT_VERSIONING_CLASS` (URL path versioning, `/api/v1/`, is the most common), restrict `ALLOWED_VERSIONS`, and branch on `request.version` to pick serializers or behavior. Only breaking changes need a new version; additive fields do not. Deprecate old versions on a published timeline, especially when mobile clients cannot be force-updated.
*Follow-up:* Header or URL versioning? URLs are explicit, easy to test and cache-friendly; headers keep URLs clean but are easier to forget.

**Q14. What is CORS, and how do you configure it for a React frontend? Does it protect your API?**
CORS is a browser rule: JavaScript from one origin may read responses from another only if the server allows it through `Access-Control-Allow-*` headers, with a preflight `OPTIONS` request for non-simple requests. With django-cors-headers, put `CorsMiddleware` near the top and list exact origins in `CORS_ALLOWED_ORIGINS`, adding `CORS_ALLOW_CREDENTIALS` only for cookies. It does not protect the API: non-browser clients ignore it, so authentication and permissions still do the work.
*Follow-up:* Why does a JSON POST trigger a preflight? `Content-Type: application/json` and an `Authorization` header make it a non-simple request.

**Q15. Do you need CSRF protection with DRF?**
Yes whenever authentication relies on cookies: `SessionAuthentication` enforces CSRF for authenticated unsafe requests, and a cookie-based refresh endpoint needs protection too. Endpoints authenticated only by an `Authorization` header (token or JWT) are not CSRF targets, because browsers never attach that header automatically.
*Follow-up:* How does a React app send the CSRF token? Read the `csrftoken` cookie and send it as `X-CSRFToken` with `credentials: "include"`.

**Q16. How do you document a DRF API?**
Use drf-spectacular to generate an OpenAPI 3 schema from serializers and views, serve Swagger UI and Redoc, and add `@extend_schema` where introspection cannot infer request or response shapes (custom `APIView`s, actions). Generate the schema in CI and derive TypeScript types for the React client so frontend and backend stay in sync.
*Follow-up:* Why not hand-write docs? They drift from the code; a generated schema is checked on every build.

**Q17. How do you handle file uploads, especially large videos?**
For small files, accept multipart data with `MultiPartParser`, use `FileField`/`ImageField` serializer fields, validate size and type, and store files through a storage backend such as S3. For large videos, have the API issue a short-lived presigned upload URL so the browser uploads straight to object storage, then record the upload and trigger transcoding asynchronously. Django workers should never hold multi-gigabyte request bodies.
*Follow-up:* Why distrust `content_type`? The client sets it; verify content server-side (Pillow for images, the transcoder's probe for video).

## Further reading
- [DRF authentication](https://www.django-rest-framework.org/api-guide/authentication/)
- [DRF permissions](https://www.django-rest-framework.org/api-guide/permissions/)
- [DRF throttling](https://www.django-rest-framework.org/api-guide/throttling/) and [pagination](https://www.django-rest-framework.org/api-guide/pagination/)
- [DRF filtering](https://www.django-rest-framework.org/api-guide/filtering/) and [versioning](https://www.django-rest-framework.org/api-guide/versioning/)
- [Django CSRF protection](https://docs.djangoproject.com/en/stable/ref/csrf/)
