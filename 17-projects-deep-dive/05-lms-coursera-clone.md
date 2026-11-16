# LMS (Coursera Clone): Django REST Framework and React at PureLogics

> **Tier:** T1 (know cold) · **Related:** [LMS project patterns](../10-python-django/07-lms-project-patterns.md), [Django REST Framework](../10-python-django/04-django-rest-framework.md), [DRF auth, permissions and advanced topics](../10-python-django/05-drf-auth-permissions-advanced.md), [ORM performance](../10-python-django/03-orm-performance.md), [Payments and subscriptions](../13-architecture-and-practices/08-payments-and-subscriptions.md), [Authentication and authorization](../04-web-fundamentals/06-authentication-authorization.md), [System design from your projects](../15-system-design/03-case-studies-from-your-projects.md)
> **On your resume:** PureLogics, Python (Django) Full-Stack Intern (Jul 2025 to Sep 2025, onsite): proposed and built an LMS (Coursera clone) with subscriptions and role-based dashboards using DRF and React.

## Quick revision

| Item | What to remember |
|---|---|
| Where | PureLogics, Python (Django) Full-Stack Intern, onsite, Jul to Sep 2025 (about three months) |
| What | Proposed and built a learning management system modeled on Coursera |
| Stack on your resume | Django REST Framework (API) and React (frontend) |
| Features on your resume | Subscriptions and role-based dashboards |
| Two verbs to defend | "Proposed" (initiative) and "built" (ownership) |

- **The behavioral angle is the strongest part:** you proposed the project. Prepare why you proposed it, how you pitched it, and what scope was agreed.
- **Domain (typical LMS):** courses contain modules, modules contain lessons; users enroll; progress is tracked per lesson; typical roles are student, instructor and admin.
- **Subscriptions (typical design; compare with yours):** plans; one subscription per user with a `status` and a `current_period_end`; access means "status allows it and the period has not ended"; the payment provider's webhooks drive the state; the client is never trusted.
- **Access gating belongs on the backend** (DRF permission classes plus queryset filtering). The React UI only hides or shows things.
- **RBAC options:** a role field on the user, Django groups and permissions, or a per-course membership table with roles. Object-level checks (`has_object_permission`) cover "instructors edit only their own courses".
- **DRF gotcha:** `has_object_permission` runs only when a view calls `get_object()`, so list endpoints must filter their querysets.
- **Role-based dashboards (typical design):** role-specific endpoints (or one endpoint with role-specific payloads), guarded routes in React, data scoped on the server, aggregates with `annotate` to avoid N+1 queries.
- **Protecting videos (options):** no public URLs for paid content; issue short-lived signed URLs after an access check; HLS streaming and DRM are heavier options; screen recording cannot be fully prevented.
- **DRF auth choices:** session auth with CSRF, DRF's token auth (`rest_framework.authtoken`), or JWT through the third-party `djangorestframework-simplejwt`.
- **Honest scope:** an internship project. Do not imply paying users or production traffic unless that really happened.

> **Fill in:** the database, the auth method, whether payments were real, test mode or simulated, where it was deployed (if anywhere), and whether you built it alone or with others.

## The 30-second pitch

> "During my internship at PureLogics I proposed and built a learning management system modeled on Coursera, with Django REST Framework for the API and React for the frontend. It has subscriptions that gate access to course content, and role-based dashboards, so each type of user sees the tools and data for their role."

> **Fill in:** one sentence on why you proposed it and one on the part you are proudest of (for example the access-gating logic or the instructor dashboard).

## Domain primer

### LMS building blocks

| Concept | Meaning | Typical model |
|---|---|---|
| Course | A unit of learning with a title, description and instructor | `Course(instructor, title, is_published)` |
| Module | An ordered section of a course | `Module(course, title, position)` |
| Lesson | Video, reading or quiz inside a module | `Lesson(module, kind, video_key, position)` |
| Enrollment | A learner joined a course | `Enrollment(user, course, enrolled_at, completed_at)` |
| Progress | Which lessons a learner finished | `LessonProgress(user, lesson, completed_at)` |
| Assessment | Quizzes or assignments with scores | `Quiz`, `Attempt` |
| Certificate | Proof of completion | Generated when all required lessons are done |

### Coursera-style access
Platforms like Coursera have mixed free access, single-course purchases and subscriptions, and their models have changed over time. Your clone only needs one clear rule: describe the one you implemented (for example "browsing is free, lessons need an active subscription").

> **Fill in:** the exact access rule in your LMS: what is free, what needs a subscription, and whether instructors or admins bypass it.

### Subscription vocabulary
Plan, billing interval, trial, renewal, grace period, past due (a renewal payment failed), cancel at period end, proration, webhook, idempotency key. The essential idea: the payment provider is the source of truth for billing, and your database keeps a copy that webhooks update.

### Roles and permissions vocabulary
Authentication answers "who are you?"; authorization answers "what may you do?". RBAC (role-based access control) grants permissions to roles and roles to users. Object-level permissions answer "may you do this to this particular object?". Least privilege: give each role only what it needs.

## Typical architecture

> A typical setup for a DRF and React LMS, to adapt to what you actually built.

```text
React SPA (student, instructor and admin dashboards)
   |  JSON over HTTPS (session cookie + CSRF token, or JWT)
   v
Django + DRF
   accounts : User (role), login and registration endpoints
   catalog  : Course, Module, Lesson (viewsets + serializers)
   learning : Enrollment, LessonProgress
   billing  : Plan, Subscription, webhook endpoint
   perms    : IsInstructor, IsCourseOwnerOrReadOnly, HasActiveSubscription
   |                         |
   v                         v
 Relational database      Media storage (local media folder, S3-compatible bucket or a video host)
                            ^ short-lived URLs issued after an access check
Payment provider --webhook--> /api/billing/webhook/
```

> **Fill in:** what your architecture really looked like: database, media storage, payment provider (or simulation), and how the React app was served.

## Bullet-by-bullet

### "Proposed ..."
- **Signals:** initiative and product thinking, rare in an intern.
- **They will ask:** why this project, how you pitched it, how the scope was agreed, what feedback you got.
- **Answer frame:** the gap or goal you saw, the pitch (who you spoke to, what you promised), the scope cut to fit the internship, the outcome.

> **Fill in:** why you proposed an LMS (learning goals, company interest, a real need), who approved it, and what was agreed.

### "... and built an LMS (Coursera clone)"
- **They will ask:** the main features, your design, what was hardest, what was cut.
- **Answer frame:** one sentence of scope, then the two headline features (subscriptions, dashboards), then one hard problem.

### "... with subscriptions"
- **They will ask:** the lifecycle, failed payments, cancellations, webhooks, how access is enforced. See Part A.

> **Fill in:** whether a real payment provider was integrated (and in test mode or live) or subscriptions were simulated.

### "... and role-based dashboards"
- **They will ask:** which roles, how roles are stored, how the API enforces them, what each dashboard shows. See Parts B and C.

> **Fill in:** the roles and what each dashboard shows.

### "... using DRF and React"
- **They will ask:** serializers vs model forms, viewsets and routers, pagination, how React called the API, auth between the SPA and the API, CORS and CSRF.
- **Study:** [Django REST Framework](../10-python-django/04-django-rest-framework.md), [DRF auth and permissions](../10-python-django/05-drf-auth-permissions-advanced.md), [LMS project patterns](../10-python-django/07-lms-project-patterns.md).

## Design deep-dive

> Typical designs to adapt. Where your real project differs, describe what you actually built and why.

### Part A: subscription lifecycle and access gating

```text
(checkout completed) -> trialing or active -> (renewal paid) -> active
active -> (renewal failed) -> past_due -> (retry paid) -> active
                                       -> (retries exhausted) -> canceled
active -> (user cancels) -> active with cancel_at_period_end -> (period ends) -> canceled
```

```python
from django.conf import settings
from django.db import models
from django.utils import timezone


class SubscriptionQuerySet(models.QuerySet):
    def granting_access(self):
        # past_due keeps access until the period ends: a grace-period policy you choose.
        return self.filter(
            status__in=["trialing", "active", "past_due"],
            current_period_end__gt=timezone.now(),
        )


class Subscription(models.Model):
    class Status(models.TextChoices):
        TRIALING = "trialing"
        ACTIVE = "active"
        PAST_DUE = "past_due"
        CANCELED = "canceled"

    user = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name="subscriptions")
    plan = models.ForeignKey("billing.Plan", on_delete=models.PROTECT)
    status = models.CharField(max_length=20, choices=Status.choices)
    current_period_end = models.DateTimeField()
    cancel_at_period_end = models.BooleanField(default=False)
    provider_subscription_id = models.CharField(max_length=255, unique=True, null=True, blank=True)

    objects = SubscriptionQuerySet.as_manager()
```

```python
from rest_framework.permissions import BasePermission


class HasActiveSubscription(BasePermission):
    message = "An active subscription is required."

    def has_permission(self, request, view):
        user = request.user
        return bool(user and user.is_authenticated and user.subscriptions.granting_access().exists())
```

- Put `HasActiveSubscription` on lesson-content endpoints (and video URL endpoints), not on the catalog, so visitors can still browse.
- Webhooks are the only thing that changes billing state. Verify the provider's signature, store the event ID, and process each event once:

```python
from django.db import transaction
from django.http import HttpResponse
from django.views.decorators.csrf import csrf_exempt
from django.views.decorators.http import require_POST


@csrf_exempt              # the provider cannot send a CSRF token; the signature check protects this view
@require_POST
def billing_webhook(request):
    event = verify_and_parse(request)    # signature check with the provider's SDK (illustrative helper)
    with transaction.atomic():
        _, created = ProcessedWebhookEvent.objects.get_or_create(event_id=event["id"])  # unique event_id
        if created:
            apply_subscription_event(event)   # update status and current_period_end
    return HttpResponse(status=200)          # duplicates are acknowledged and ignored
```

- **Out-of-order events:** either compare event timestamps before applying, or treat each webhook as "something changed" and re-fetch the subscription from the provider's API, which always gives the latest state.
- **Never trust the redirect:** the success page after checkout only says the user finished the flow; access starts when the webhook (or a server-side fetch) confirms payment.
- If you used Stripe, signature verification is `stripe.Webhook.construct_event(payload, signature_header, endpoint_secret)`; other providers have an equivalent.

> **Fill in:** the real lifecycle you implemented, which events you handled, and what a failed payment did to access.

### Part B: the RBAC schema

| Approach | How | Good for | Limits |
|---|---|---|---|
| Role field on the user | `user.role == "instructor"` | A few fixed, global roles | One role per user, no per-course roles |
| Django groups and permissions | `user.groups`, `user.has_perm("courses.change_course")` | Admin integration, configurable permission sets | Model-level, not per object, out of the box |
| Membership table | `CourseMembership(user, course, role)` | Per-course roles (instructor of one course, assistant in another) | More joins and more code |
| Object-permission library | for example django-guardian | Many per-object grants | An extra dependency |

```python
from django.contrib.auth.models import AbstractUser
from django.db import models
from rest_framework.permissions import SAFE_METHODS, BasePermission


class User(AbstractUser):
    class Role(models.TextChoices):
        STUDENT = "student"
        INSTRUCTOR = "instructor"
        ADMIN = "admin"

    role = models.CharField(max_length=20, choices=Role.choices, default=Role.STUDENT)


class IsInstructor(BasePermission):
    def has_permission(self, request, view):
        return request.user.is_authenticated and request.user.role == User.Role.INSTRUCTOR


class IsCourseOwnerOrReadOnly(BasePermission):
    def has_object_permission(self, request, view, obj):     # runs only via get_object()
        return request.method in SAFE_METHODS or obj.instructor_id == request.user.id
```

- Scope list querysets by role in `get_queryset()`: instructors see their own courses, students see published courses (and their enrollments), admins see everything.
- Never accept `role` from a signup form; set it server-side or through an admin action.

> **Fill in:** how roles were really stored and enforced, and whether a user could hold more than one role.

### Part C: role-based dashboards
- **API shape:** separate endpoints per role (`/api/dashboard/student/`, `/api/dashboard/instructor/`) are easier to secure and cache than one endpoint that branches on role.
- **React:** a route guard reads the current user's role (from a `/me` endpoint) and redirects, but this is only user experience; the API enforces access.
- **Queries:** compute stats in the database, not in Python loops.

```python
from django.db.models import Count, Q

instructor_courses = Course.objects.filter(instructor=request.user).annotate(
    students=Count("enrollments", distinct=True),
    completions=Count("enrollments", filter=Q(enrollments__completed_at__isnull=False), distinct=True),
)
```

- Student dashboards need progress per course: count completed lessons per enrollment with `annotate`, or keep a cached progress value on the enrollment that updates when a lesson is completed.

> **Fill in:** what each dashboard showed and how its data was computed.

### Part D: protecting course videos

| Option | Protects against | Cost |
|---|---|---|
| Private storage + short-lived signed URLs | Link sharing, guessing URLs | Low |
| Django checks access, the web server streams the file (nginx `X-Accel-Redirect`) | Unauthorized access to files | Low to medium |
| HLS streaming with short-lived tokens | Easy whole-file downloads | Medium (transcoding) |
| DRM (Widevine, FairPlay) | Most casual copying | High (licensing, special players) |
| Visible watermark with the user's identity | Deters leaks and helps trace them | Medium |

```python
import boto3
from django.conf import settings
from rest_framework.exceptions import PermissionDenied

s3 = boto3.client("s3")


def lesson_video_url(user, lesson) -> str:
    if not can_watch(user, lesson):          # enrollment or subscription check first (illustrative helper)
        raise PermissionDenied("Subscribe to watch this lesson.")
    return s3.generate_presigned_url(
        "get_object",
        Params={"Bucket": settings.VIDEO_BUCKET, "Key": lesson.video_key},
        ExpiresIn=300,                       # five minutes: a leaked link dies quickly
    )
```

Say the honest limit: nothing stops a determined screen recorder, so the goal is to make sharing hard and traceable.

> **Fill in:** where videos were stored (local media, a bucket, a video host) and how access was protected.

### Part E: what you would change
A strong answer picks two or three real gaps and explains the fix. Common candidates (choose only the true ones):
- More automated tests for permissions and the subscription lifecycle (DRF's `APIClient` or pytest).
- Background tasks (Celery or RQ) for emails and webhook processing, so requests stay fast.
- Per-course memberships instead of a single global role.
- Re-fetching subscription state from the provider on each webhook.
- OpenAPI docs for the API (drf-spectacular is a common choice).
- Video delivery through a CDN, pagination everywhere, Docker and CI.

> **Fill in:** two changes you would really make, and why.

## Likely interview questions

**Q1. Walk me through the LMS.**
Purpose and the access rule in one line, the stack (DRF and React), the data model (courses, modules, lessons, enrollments, progress, subscriptions, roles), then the two headline features and one hard problem.
> **Fill in:** the hard problem.

**Q2. Why did you propose this project, and how did you get it approved?**
Behavioral question in disguise: the motivation, the pitch, the agreed scope, and what you learned about scoping. See [the story bank](../18-behavioral-and-hr/02-star-stories-bank.md).
> **Fill in:** the real story.

**Q3. How did subscriptions and role-based dashboards work in the LMS?**
Subscriptions: a plan, a subscription row with status and period end, webhooks updating it, and a permission class gating lesson endpoints. Dashboards: the user's role decides which dashboard endpoint and route they can use; the server scopes each query by role and computes stats with aggregates.
> **Fill in:** the specifics of your implementation.

**Q4. What happens when a renewal payment fails, or a user cancels?**
Failed renewal: the provider retries and reports `past_due`; you keep or remove access according to your grace policy; after final failure the subscription becomes canceled. Cancellation: usually access continues until `current_period_end`, then ends.

**Q5. How do you stop a non-subscriber from calling the lesson API directly?**
Enforcement is on the server: the permission class runs on every lesson request, querysets are filtered, and video URLs are issued only after the same check. Hiding a button in React protects nothing.

**Q6. How did you model roles, and why that approach?**
Explain your choice against the alternatives in Part B: a role field is simplest for fixed global roles; groups add admin-managed permissions; memberships allow per-course roles.

**Q7. How do you stop an instructor from editing another instructor's course?**
An object-level permission comparing `course.instructor_id` with the user, plus querysets that only include the instructor's courses, so other courses return 404 instead of 403.

**Q8. How did you protect course videos?**
Private storage, an access check, then a short-lived signed URL or a server-controlled stream; stronger options are HLS with tokens, DRM and watermarking; admit that screen recording cannot be fully stopped.

**Q9. How did React authenticate against DRF?**
Session auth works well when the SPA and API share a site (cookies plus CSRF tokens); JWT (simplejwt) suits separate domains or mobile clients, ideally with a short-lived access token and a refresh token in an HttpOnly cookie. Cross-origin setups also need CORS configuration (django-cors-headers is the usual package).
> **Fill in:** what you used and why.

**Q10. How do you track lesson progress efficiently?**
A `LessonProgress(user, lesson, completed_at)` table with a unique constraint on `(user, lesson)`, an idempotent "mark complete" endpoint, and course progress computed with an aggregate or cached on the enrollment.

**Q11. What if the provider sends the same webhook twice, or out of order?**
Store processed event IDs under a unique constraint and skip repeats; for ordering, compare event times or re-fetch the subscription's current state from the provider.

**Q12. How did you test it?**
Permission tests per role, subscription state tests (active, past due, expired), webhook tests with sample payloads, and API tests with an authenticated client.
> **Fill in:** what tests really existed.

**Q13. How did you fit it into three months? What did you cut?**
Name the MVP, what you postponed (for example certificates or quizzes), and how you decided.
> **Fill in:** the real scope decisions.

**Q14. Why DRF rather than plain Django views or FastAPI?**
DRF gives serializers with validation, viewsets and routers, authentication and permission classes, pagination, throttling and a browsable API on top of Django's ORM, admin and auth. FastAPI is excellent for async-first APIs, but you would rebuild much of what Django ships.

**Q15. How would this scale to many concurrent learners?**
Videos from a CDN (the biggest load), cached catalog pages, `select_related` and `prefetch_related` against N+1 queries, indexes on foreign keys and filters, pagination, background tasks, and stateless app servers behind a load balancer.

**Q16. What would you change if you built it again?**
Two or three real items from Part E with the reason for each.

## Fill-in checklist
- [ ] Why you proposed it, who approved it, and the agreed scope.
- [ ] Solo or team; mentor or reviewer involvement.
- [ ] Database, auth method, deployment.
- [ ] Payment provider (and mode) or how subscriptions were simulated.
- [ ] The access rule and the subscription states you handled.
- [ ] Roles, how they were stored, and what each dashboard showed.
- [ ] Where videos lived and how they were protected.
- [ ] Tests that existed.
- [ ] What you cut and what you would change.
- [ ] Any demo, presentation or feedback at the end of the internship.
