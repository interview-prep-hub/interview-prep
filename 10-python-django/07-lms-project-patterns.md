# LMS Project Patterns: a Coursera-Style Platform in Django, DRF and React

> **Tier:** T1 (know cold) · **Related:** [LMS (Coursera clone) deep dive](../17-projects-deep-dive/05-lms-coursera-clone.md), [Models and ORM](02-models-and-orm.md), [ORM performance](03-orm-performance.md), [DRF auth and permissions](05-drf-auth-permissions-advanced.md), [Advanced Django](06-django-advanced.md), [Payments and subscriptions](../13-architecture-and-practices/08-payments-and-subscriptions.md), [Authentication and authorization](../04-web-fundamentals/06-authentication-authorization.md), [React patterns and architecture](../06-react/07-patterns-and-architecture.md)
> **On your resume:** PureLogics, Python (Django) Full-Stack Intern (Jul 2025 to Sep 2025): you proposed and built an LMS (Coursera clone) with subscriptions and role-based dashboards using DRF and React. **Everything below is an illustrative design to adapt, not your actual code.** Replace each part with what you really built using the Fill in prompts.

## Quick revision
- Domain: users with a role (student, instructor, admin); courses made of modules and lessons; enrollments; lesson progress; quizzes and attempts; certificates; reviews; plans and subscriptions.
- Put invariants in the database: one enrollment per student and course, one progress row per lesson, one review per student and course, ratings from 1 to 5, at most one live subscription per user.
- Role-based dashboards = a `role` field + permission classes per role + querysets scoped per role.
- Student progress = completed lessons divided by total lessons, computed with annotations, not a query per lesson.
- Instructor analytics = annotations (students, completions, average rating) on `Course.objects.filter(instructor=request.user)`.
- Admin moderation = a course status workflow (draft, pending, published, rejected) with admin-only transitions done as conditional updates.
- Payments (illustrated with Stripe): Checkout in subscription mode collects payment, Billing charges every period, and **webhooks are the source of truth** for subscription status.
- Webhooks: verify the signature on the raw body, record the event ID under a unique constraint so each event is processed once, re-fetch current state because events can arrive out of order, and return 2xx quickly.
- Access gate: free or preview lesson, or an enrollment plus a subscription whose status is active or trialing and whose period has not ended. Enforce it in a DRF permission, not only in React.
- Failed payment: `invoice.payment_failed` moves the subscription to `past_due`, the user is told to fix the card, the provider retries, and access ends when the subscription becomes `canceled` or `unpaid`.
- Video: private storage, transcoding to HLS, and short-lived signed URLs or signed cookies issued only after an access check.
- Search: `SearchFilter` plus django-filter for a small catalog; PostgreSQL full-text search or a search engine as it grows.
- Test a role-by-endpoint permission matrix, webhook idempotency, progress math and query counts.

## 1. Scope and requirements

Functional: students browse a catalog, preview lessons, subscribe, enroll, watch lessons, track progress, take quizzes, earn certificates and review courses. Instructors create courses (modules, lessons, quizzes), submit them for review and see analytics. Admins moderate courses and reviews and handle support. Non-functional: video must not be publicly accessible, payments must never double-charge or grant access without payment, dashboards must stay fast as data grows, and every rule must be enforced by the API because the React app is untrusted.

> **Fill in:** the scope you actually proposed at PureLogics, who approved it, how long you had, and whether you built it alone or with other interns.

## 2. Data model

```python
# accounts/models.py
class User(AbstractUser):
    class Role(models.TextChoices):
        STUDENT = "student", "Student"
        INSTRUCTOR = "instructor", "Instructor"
        ADMIN = "admin", "Admin"

    email = models.EmailField(unique=True)
    role = models.CharField(max_length=20, choices=Role.choices, default=Role.STUDENT)
    stripe_customer_id = models.CharField(max_length=64, blank=True)

# courses/models.py (TimeStampedModel adds created_at and updated_at)
class Course(TimeStampedModel):
    class Status(models.TextChoices):
        DRAFT = "draft", "Draft"
        PENDING = "pending", "Pending review"
        PUBLISHED = "published", "Published"
        REJECTED = "rejected", "Rejected"

    instructor = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.PROTECT, related_name="courses_taught")
    category = models.ForeignKey("Category", on_delete=models.SET_NULL, null=True, blank=True, related_name="courses")
    title = models.CharField(max_length=200)
    slug = models.SlugField(max_length=220, unique=True)
    description = models.TextField(blank=True)
    level = models.CharField(max_length=20, default="beginner")
    status = models.CharField(max_length=20, choices=Status.choices, default=Status.DRAFT, db_index=True)
    is_free = models.BooleanField(default=False)
    published_at = models.DateTimeField(null=True, blank=True)

class Module(models.Model):
    course = models.ForeignKey(Course, on_delete=models.CASCADE, related_name="modules")
    title = models.CharField(max_length=200)
    position = models.PositiveIntegerField()

    class Meta:
        ordering = ["position"]
        constraints = [models.UniqueConstraint(fields=["course", "position"], name="unique_module_position")]

class Lesson(models.Model):
    module = models.ForeignKey(Module, on_delete=models.CASCADE, related_name="lessons")
    title = models.CharField(max_length=200)
    position = models.PositiveIntegerField()
    video_key = models.CharField(max_length=255, blank=True)     # storage key of the HLS manifest
    duration_seconds = models.PositiveIntegerField(default=0)
    is_preview = models.BooleanField(default=False)              # watchable without a subscription

    class Meta:
        ordering = ["position"]
        constraints = [models.UniqueConstraint(fields=["module", "position"], name="unique_lesson_position")]

# learning/models.py
class Enrollment(TimeStampedModel):
    student = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name="enrollments")
    course = models.ForeignKey(Course, on_delete=models.CASCADE, related_name="enrollments")
    completed_at = models.DateTimeField(null=True, blank=True)

    class Meta:
        constraints = [models.UniqueConstraint(fields=["student", "course"], name="unique_enrollment")]

class LessonProgress(models.Model):
    enrollment = models.ForeignKey(Enrollment, on_delete=models.CASCADE, related_name="lesson_progress")
    lesson = models.ForeignKey(Lesson, on_delete=models.CASCADE, related_name="+")
    completed_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        constraints = [models.UniqueConstraint(fields=["enrollment", "lesson"], name="unique_lesson_progress")]

class Quiz(models.Model):                                        # Question and Choice(is_correct) hang off it
    lesson = models.OneToOneField(Lesson, on_delete=models.CASCADE, related_name="quiz")
    pass_mark = models.PositiveSmallIntegerField(default=70)     # percent

class QuizAttempt(TimeStampedModel):
    enrollment = models.ForeignKey(Enrollment, on_delete=models.CASCADE, related_name="quiz_attempts")
    quiz = models.ForeignKey(Quiz, on_delete=models.CASCADE, related_name="attempts")
    score = models.PositiveSmallIntegerField()                   # percent, graded on the server
    passed = models.BooleanField()

class Certificate(models.Model):
    enrollment = models.OneToOneField(Enrollment, on_delete=models.PROTECT, related_name="certificate")
    code = models.UUIDField(default=uuid.uuid4, unique=True, editable=False)   # public verification code
    issued_at = models.DateTimeField(null=True, blank=True)
    pdf_url = models.URLField(blank=True)

class Review(TimeStampedModel):
    course = models.ForeignKey(Course, on_delete=models.CASCADE, related_name="reviews")
    student = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name="reviews")
    rating = models.PositiveSmallIntegerField()
    body = models.TextField(blank=True)
    is_hidden = models.BooleanField(default=False)               # set by admin moderation

    class Meta:
        constraints = [
            models.UniqueConstraint(fields=["course", "student"], name="one_review_per_student"),
            models.CheckConstraint(condition=Q(rating__gte=1) & Q(rating__lte=5), name="rating_1_to_5"),
        ]

# billing/models.py
class Plan(models.Model):
    code = models.SlugField(unique=True)                         # "monthly", "yearly"
    stripe_price_id = models.CharField(max_length=64, unique=True)
    price_cents = models.PositiveIntegerField()

class Subscription(TimeStampedModel):
    class Status(models.TextChoices):                            # mirrors the provider's statuses
        TRIALING = "trialing"
        ACTIVE = "active"
        PAST_DUE = "past_due"
        CANCELED = "canceled"
        UNPAID = "unpaid"
        INCOMPLETE = "incomplete"
        INCOMPLETE_EXPIRED = "incomplete_expired"
        PAUSED = "paused"

    user = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.PROTECT, related_name="subscriptions")
    plan = models.ForeignKey(Plan, on_delete=models.PROTECT)
    stripe_subscription_id = models.CharField(max_length=64, unique=True)
    status = models.CharField(max_length=20, choices=Status.choices)
    current_period_end = models.DateTimeField()
    cancel_at_period_end = models.BooleanField(default=False)

    class Meta:
        constraints = [
            models.UniqueConstraint(
                fields=["user"],
                condition=Q(status__in=["trialing", "active", "past_due"]),
                name="one_live_subscription_per_user",       # partial unique index (PostgreSQL)
            )
        ]

class StripeEvent(models.Model):                                 # webhook idempotency log
    event_id = models.CharField(max_length=255, unique=True)
    type = models.CharField(max_length=100)
    received_at = models.DateTimeField(auto_now_add=True)
```

Design notes: the unique constraints turn double clicks and retried requests into harmless no-ops (`get_or_create` returns the existing row); `PROTECT` keeps certificates and paid history from vanishing when someone deletes a course; `is_correct` on choices must never reach a student serializer; and `Subscription` is a local mirror of the provider's state, updated only from webhooks and reconciliation jobs. Access is subscription-based (a Coursera Plus-style model); per-course purchases would add an `Order` model instead.

> **Fill in:** your real models: which of these existed, what you named them, and anything different (per-course purchases instead of subscriptions, assignments, discussion forums).

## 3. Roles, permissions and role-based dashboards

| Capability | Student | Instructor | Admin |
|---|---|---|---|
| Browse the catalog, watch preview lessons | yes | yes | yes |
| Enroll, watch, track progress, take quizzes | yes (free course or live subscription) | no | read-only support access |
| Create and edit own courses, submit for review | no | yes | yes |
| Analytics | own progress | own courses | platform-wide |
| Approve courses, hide reviews, handle refunds | no | no | yes |

Permission classes such as `IsStudent`, `IsInstructor` and `IsPlatformAdmin` (see [DRF auth and permissions](05-drf-auth-permissions-advanced.md)) guard each endpoint, and every queryset is scoped to the caller. Each dashboard is a separate endpoint with its own permission and query, which keeps the rules readable.

**Student dashboard: progress without N+1**

```python
class MyLearningView(generics.ListAPIView):
    serializer_class = MyEnrollmentSerializer
    permission_classes = [IsStudent]

    def get_queryset(self):
        return (
            Enrollment.objects.filter(student=self.request.user)
            .select_related("course")
            .annotate(
                total_lessons=Count("course__modules__lessons", distinct=True),
                done_lessons=Count("lesson_progress", distinct=True),
            )
            .order_by("-updated_at")
        )

class MyEnrollmentSerializer(serializers.ModelSerializer):
    course_title = serializers.CharField(source="course.title", read_only=True)
    progress_percent = serializers.SerializerMethodField()

    class Meta:
        model = Enrollment
        fields = ["id", "course_title", "completed_at", "progress_percent"]

    def get_progress_percent(self, obj):        # annotated values: no extra queries
        if not obj.total_lessons:
            return 0
        return round(100 * obj.done_lessons / obj.total_lessons)
```

**Instructor dashboard: analytics scoped to their own courses**

```python
class InstructorDashboardView(APIView):
    permission_classes = [IsInstructor]

    def get(self, request):
        courses = (
            Course.objects.filter(instructor=request.user)          # scope first
            .annotate(
                students=Count("enrollments", distinct=True),
                completions=Count("enrollments", filter=Q(enrollments__completed_at__isnull=False), distinct=True),
                avg_rating=Avg("reviews__rating"),
            )
            .values("id", "title", "status", "students", "completions", "avg_rating")
        )
        rows = [
            {**c, "completion_rate": round(100 * c["completions"] / c["students"]) if c["students"] else 0}
            for c in courses
        ]
        return Response({"courses": rows})
```

`distinct=True` keeps the counts correct even though enrollments and reviews are joined in one query. For large courses, move the average into a `Subquery`, cache the dashboard for a few minutes, or maintain a summary table updated by background jobs.

**Admin moderation: state transitions as conditional updates**

```python
class ModerationViewSet(mixins.ListModelMixin, viewsets.GenericViewSet):
    serializer_class = CourseSerializer
    permission_classes = [IsPlatformAdmin]

    def get_queryset(self):
        return Course.objects.filter(status=Course.Status.PENDING).select_related("instructor")

    @action(detail=True, methods=["post"])
    def approve(self, request, pk=None):
        updated = Course.objects.filter(pk=pk, status=Course.Status.PENDING).update(
            status=Course.Status.PUBLISHED, published_at=timezone.now()
        )
        if not updated:                         # someone else already acted, or it was never pending
            return Response({"detail": "Course is not pending review."}, status=status.HTTP_409_CONFLICT)
        return Response({"status": "published"})
```

The `WHERE status = 'pending'` in the update makes the transition atomic: two admins clicking at once cannot both publish, and a draft cannot jump straight to published. The instructor's `submit` action uses the same pattern from `draft` or `rejected` to `pending`. Internal tasks (refunds, data fixes, hiding abusive reviews) can live in the Django admin with custom actions.

> **Fill in:** how your dashboards were organized (one per role, endpoints you built, charts on the React side) and which metrics the instructor dashboard showed.

## 4. Subscriptions and payments

The design below uses Stripe because its hosted Checkout and Billing are the common choice; the same shape works with other providers.

1. React calls `POST /api/billing/checkout-session/` with a plan code.
2. Django creates (or reuses) a provider customer for the user and a Checkout Session in `subscription` mode, then returns its URL.
3. The browser goes to the hosted payment page. After payment the user returns to `success_url`, which only shows "processing": a redirect proves nothing.
4. The provider sends webhooks (`checkout.session.completed`, `customer.subscription.created`, `customer.subscription.updated`, `customer.subscription.deleted`, `invoice.paid`, `invoice.payment_failed`), and Django updates its `Subscription` mirror.
5. Every access check reads the local `Subscription` row.

```python
# billing/services.py
import stripe
from django.conf import settings

stripe.api_key = settings.STRIPE_SECRET_KEY

def create_checkout_session(user, plan):
    if not user.stripe_customer_id:
        customer = stripe.Customer.create(email=user.email, metadata={"user_id": str(user.pk)})
        user.stripe_customer_id = customer.id
        user.save(update_fields=["stripe_customer_id"])
    session = stripe.checkout.Session.create(
        mode="subscription",
        customer=user.stripe_customer_id,
        line_items=[{"price": plan.stripe_price_id, "quantity": 1}],
        client_reference_id=str(user.pk),
        success_url=f"{settings.FRONTEND_URL}/billing/success?session_id={{CHECKOUT_SESSION_ID}}",
        cancel_url=f"{settings.FRONTEND_URL}/pricing",
    )
    return session.url
```

```python
# billing/webhooks.py (imports: stripe, settings, csrf_exempt, require_POST, HttpResponse,
# transaction, and datetime plus timezone as dt_timezone from the standard library)
@csrf_exempt                    # the provider cannot send a CSRF token; the signature authenticates it
@require_POST
def stripe_webhook(request):
    try:
        event = stripe.Webhook.construct_event(          # verify against the RAW body
            request.body, request.headers.get("Stripe-Signature", ""), settings.STRIPE_WEBHOOK_SECRET
        )
    except Exception:          # malformed payload (ValueError) or a signature verification error
        return HttpResponse(status=400)

    with transaction.atomic():
        _, created = StripeEvent.objects.get_or_create(event_id=event["id"], defaults={"type": event["type"]})
        if not created:
            return HttpResponse(status=200)              # duplicate delivery: already processed
        handle_event(event)                              # an exception rolls back the event row, so a retry reprocesses
    return HttpResponse(status=200)

def handle_event(event):
    obj = event["data"]["object"]
    if event["type"].startswith("customer.subscription."):
        sync_subscription(obj["id"])
    elif event["type"] == "checkout.session.completed" and obj["subscription"]:   # None for one-time payments
        sync_subscription(obj["subscription"])
    elif event["type"] == "invoice.payment_failed":
        transaction.on_commit(lambda: send_payment_failed_email.delay(obj["customer"]))

def sync_subscription(subscription_id):
    data = stripe.Subscription.retrieve(subscription_id)       # current state, robust to out-of-order events
    item = data["items"]["data"][0]
    Subscription.objects.update_or_create(
        stripe_subscription_id=data["id"],
        defaults={
            "user": User.objects.get(stripe_customer_id=data["customer"]),
            "plan": Plan.objects.get(stripe_price_id=item["price"]["id"]),
            "status": data["status"],
            "current_period_end": datetime.fromtimestamp(period_end_ts(data, item), tz=dt_timezone.utc),
            "cancel_at_period_end": data["cancel_at_period_end"],
        },
    )
```

`period_end_ts()` is a small helper of your own: older Stripe API versions expose `current_period_end` on the subscription, while newer ones moved the billing period onto each subscription item, so read whichever your pinned API version provides (check the Stripe changelog).

Webhook rules worth saying out loud:
- **Verify the signature** on the raw request body with the endpoint's secret. That is why this is a plain Django view: parsing and re-serializing JSON would break verification.
- **Process each event once**: the unique `event_id` plus a transaction means duplicates return 200 without side effects, and handlers such as `update_or_create` are safe to repeat anyway.
- **Do not trust ordering**: events can arrive out of order or late, so re-fetch the subscription (or compare timestamps) instead of applying each event blindly.
- **Answer fast**: return 2xx quickly and push slow work (emails, PDFs) to Celery through `on_commit`. Stripe retries failed deliveries with backoff for up to three days in live mode, and a nightly reconciliation job catches anything missed.
- Develop locally with test-mode keys and the Stripe CLI (`stripe listen --forward-to localhost:8000/webhooks/stripe/`), with a separate webhook secret per environment.

**Gating access**

```python
LIVE_STATUSES = ["trialing", "active"]

def has_active_subscription(user):
    return Subscription.objects.filter(
        user=user, status__in=LIVE_STATUSES, current_period_end__gt=timezone.now()
    ).exists()

class CanWatchLesson(BasePermission):
    message = "Subscribe to watch this lesson."

    def has_object_permission(self, request, view, lesson):
        if lesson.is_preview:
            return True
        course = lesson.module.course
        enrolled = Enrollment.objects.filter(student=request.user, course=course).exists()
        return enrolled and (course.is_free or has_active_subscription(request.user))
```

**Failed payments and cancellations**: `invoice.payment_failed` moves the subscription to `past_due`; email the user a link to the provider's customer portal (`stripe.billing_portal.Session.create(customer=..., return_url=...)`) to update the card. The provider retries according to its dunning settings; if every retry fails, the subscription becomes `canceled` or `unpaid` and the next `customer.subscription.updated` or `deleted` webhook removes access. Whether `past_due` users keep access during the retry window is a product decision; make it explicit. A cancellation with `cancel_at_period_end=True` keeps access until the paid period ends.

> **Fill in:** which payment provider you used, hosted checkout or embedded forms, which webhook events you handled, and how you tested payments (test mode, CLI forwarding).

## 5. Protecting video content

1. An instructor asks the API for an upload URL; Django returns a short-lived presigned URL for a **private** raw-uploads bucket, and the browser uploads directly (Django never holds the file).
2. When the upload lands, a Celery task submits a transcoding job (a cloud service such as AWS Elemental MediaConvert, or a video platform such as Mux or Cloudflare Stream) that produces HLS renditions at several bitrates in a private output bucket.
3. A completion event or webhook stores `video_key` and `duration_seconds` on the lesson.
4. Playback: `GET /api/lessons/{id}/playback/` runs `CanWatchLesson`, then returns a URL that expires in minutes.

```python
import boto3

s3 = boto3.client("s3")

def presigned_upload_url(key):
    return s3.generate_presigned_url(
        "put_object",
        Params={"Bucket": settings.RAW_VIDEO_BUCKET, "Key": key, "ContentType": "video/mp4"},
        ExpiresIn=900,                                  # 15 minutes to start the upload
    )

class LessonPlaybackView(generics.RetrieveAPIView):
    permission_classes = [IsAuthenticated, CanWatchLesson]
    queryset = Lesson.objects.select_related("module__course")

    def retrieve(self, request, *args, **kwargs):
        lesson = self.get_object()                      # runs CanWatchLesson.has_object_permission
        return Response({"url": signed_playback_url(lesson.video_key, expires_in=300)})
```

`signed_playback_url()` wraps your CDN or video provider. An HLS manifest references many segment files, so per-file presigned S3 URLs do not scale; use CDN signed cookies (for example CloudFront) or the provider's signed playback tokens. Signed URLs stop link sharing and hotlinking but not screen recording; for premium content, DRM (Widevine, FairPlay) through a video platform and visible watermarks with the user's ID are the next steps.

> **Fill in:** where your videos were stored and how they were served (YouTube unlisted, S3, Cloudinary, a video platform), and whether you protected them with signed URLs.

## 6. Course search and filtering

For a small catalog, DRF's `SearchFilter` plus a django-filter `FilterSet` (category, level, free or paid, minimum rating through an annotation) and `OrderingFilter` (newest, most popular, best rated) are enough. As the catalog grows, use PostgreSQL full-text search:

```python
from django.contrib.postgres.search import SearchQuery, SearchRank, SearchVector

def search_courses(q):
    vector = SearchVector("title", weight="A") + SearchVector("description", weight="B")
    query = SearchQuery(q, search_type="websearch")    # supports "quoted phrases" and -exclusions
    return (
        Course.objects.filter(status=Course.Status.PUBLISHED)
        .annotate(search=vector, rank=SearchRank(vector, query))
        .filter(search=query)
        .order_by("-rank")
    )
```

At scale, store the vector in a `SearchVectorField` with a GIN index instead of computing it per query, add `pg_trgm` trigram similarity for typo tolerance, and move to a dedicated engine (Elasticsearch or OpenSearch, Meilisearch, Typesense) when you need facets and relevance tuning, syncing documents through `on_commit` tasks. Cache popular catalog pages and always paginate.

## 7. The API surface for the React frontend

| Method and path | Who | Purpose |
|---|---|---|
| `POST /api/auth/token/`, `/token/refresh/`, `/logout/` | anyone, then logged in | JWT login, refresh, logout |
| `GET /api/me/` | logged in | current user, role and subscription status (drives the UI) |
| `GET /api/courses/?search=&category=&level=&ordering=` | public | paginated catalog |
| `GET /api/courses/{slug}/` | public | details, curriculum titles, preview flags, rating |
| `POST /api/courses/{slug}/enroll/` | student | enroll (free course or live subscription) |
| `GET /api/me/enrollments/` | student | my learning with progress percentages |
| `POST /api/lessons/{id}/complete/` | enrolled student | mark a lesson complete (idempotent) |
| `GET /api/lessons/{id}/playback/` | enrolled student | short-lived signed video URL |
| `POST /api/quizzes/{id}/attempts/` | enrolled student | submit answers; graded on the server |
| `POST /api/courses/{slug}/reviews/` | enrolled student | one review per course |
| `GET, POST /api/instructor/courses/`, `POST .../{slug}/submit/` | instructor | manage own courses, submit for review |
| `GET /api/instructor/dashboard/` | instructor | analytics for own courses |
| `GET /api/admin/moderation/courses/`, `POST .../{id}/approve/` | admin | review queue |
| `POST /api/billing/checkout-session/`, `/portal-session/` | logged in | hosted checkout and customer portal URLs |
| `POST /webhooks/stripe/` | the provider (signed) | subscription lifecycle events |

```tsx
import type { ReactNode } from "react";
import { useQuery } from "@tanstack/react-query";
import { Navigate } from "react-router-dom";

type Role = "student" | "instructor" | "admin";

// UX only: the API enforces every rule again
export function RequireRole({ roles, children }: { roles: Role[]; children: ReactNode }) {
  const { data: me, isPending } = useQuery({ queryKey: ["me"], queryFn: fetchMe });
  if (isPending) return <Spinner />;
  if (!me) return <Navigate to="/login" replace />;
  if (!roles.includes(me.role)) return <Navigate to="/" replace />;
  return <>{children}</>;
}
```

Server state (catalog, enrollments, dashboards) fits TanStack Query (see [data fetching](../06-react/04-data-fetching-tanstack-query.md)); invalidate `["me"]` after checkout returns so the UI picks up the new subscription once the webhook has landed.

> **Fill in:** how your React app was structured (routing, state management, how it called the API and stored tokens) and which screens you built yourself.

## 8. Testing and deployment

```python
@pytest.mark.parametrize("role, expected", [("student", 403), ("instructor", 200), ("admin", 403)])
@pytest.mark.django_db
def test_instructor_dashboard_access(api_client, role, expected):
    api_client.force_authenticate(user=UserFactory(role=role))
    assert api_client.get(reverse("instructor-dashboard")).status_code == expected

@pytest.mark.django_db
@patch("billing.webhooks.handle_event")
@patch("billing.webhooks.stripe.Webhook.construct_event")
def test_duplicate_webhook_is_processed_once(mock_construct, mock_handle, client):
    mock_construct.return_value = {"id": "evt_123", "type": "invoice.paid", "data": {"object": {}}}
    url = reverse("stripe-webhook")
    for _ in range(2):
        assert client.post(url, data=b"{}", content_type="application/json").status_code == 200
    assert mock_handle.call_count == 1
```

Also test progress percentages, the access gate (preview, free course, active, past due, canceled), quiz grading without leaking answers, and query counts on the catalog and dashboards. Deployment follows [Advanced Django](06-django-advanced.md): Gunicorn web containers, Celery workers plus one beat, PostgreSQL, Redis, object storage with a CDN, the React build on a CDN or Nginx, provider keys and the webhook secret per environment, `CORS_ALLOWED_ORIGINS` and `CSRF_TRUSTED_ORIGINS` set to the frontend's domain, and `check --deploy` before launch.

## Common pitfalls
- Granting access on the checkout success redirect instead of waiting for the webhook.
- Skipping webhook signature verification, or verifying parsed JSON instead of the raw body.
- Processing a duplicated webhook twice (double emails, double credits), or assuming events arrive in order.
- Returning `is_correct` in quiz serializers, or grading quizzes in the browser.
- Public object-storage URLs for paid videos, or signed URLs that live for days.
- Progress or analytics computed with a query per lesson or per enrollment.
- Instructor endpoints that forget `instructor=request.user`, or role checks that exist only in React.
- Missing unique constraints, so a double click creates two enrollments or two progress rows.
- Cutting access immediately on cancellation when the user already paid until the period end.
- One webhook secret or live keys shared across environments.

## How to talk about this in an interview
- Pitch it in 30 seconds from the resume line: you proposed an LMS similar to Coursera at PureLogics and built it with DRF and React, with subscriptions and role-based dashboards. Then offer depth on one of three threads: role-based access, the payment and webhook flow, or keeping dashboards fast.
- Be explicit about what was real and what you would do in production. Interviewers respect "we used X; in production I would add Y because Z" far more than an inflated story. The [project deep dive](../17-projects-deep-dive/05-lms-coursera-clone.md) has the full checklist.
- Connect it to your Devntech work: org-scoped access control in the multi-tenant platform migration and Sidekiq jobs on Colochousing apply the same patterns.
- > **Fill in:** which roles your dashboards actually served and what each role could see and do.
- > **Fill in:** what you proposed and why the team accepted it, and what you built personally versus with others.
- > **Fill in:** the hardest bug or design decision (for example webhook ordering, progress calculation, permissions) and how you solved it.
- > **Fill in:** what state the project reached (demo, internal use, deployed) and what you would change if you rebuilt it.

## Interview questions

Model answers describe the illustrative design above; swap in your real details where they differ.

**Q1. Walk me through the data model of your LMS.**
Users carry a role; instructors own courses; courses contain ordered modules and lessons; students link to courses through an `Enrollment` with a unique (student, course) constraint; `LessonProgress` records completed lessons per enrollment; quizzes hang off lessons with attempts graded on the server; certificates attach one-to-one to completed enrollments; reviews are unique per student and course with a 1 to 5 check constraint; subscriptions mirror the payment provider. Constraints make retries and double clicks harmless.
*Follow-up:* Why `PROTECT` on certificates? Issued certificates and payment history must survive course or user cleanup.

**Q2. How did you implement role-based access for students, instructors and admins?**
A `role` field on the custom user model, small permission classes per role (`IsStudent`, `IsInstructor`, `IsPlatformAdmin`) chosen per action, object permissions for ownership (instructors edit only their courses), and querysets scoped per role so each dashboard sees only its slice. React hides UI by role, but the API enforces everything.
*Follow-up:* What would you change for many more roles? Move to Django groups and permissions, or a policy layer, instead of hardcoded role strings.

**Q3. How do you compute a student's progress efficiently?**
Annotate each enrollment with `Count("course__modules__lessons", distinct=True)` and `Count("lesson_progress", distinct=True)` in one query, then compute the percentage in the serializer from those attributes. That stays at a constant number of queries no matter how many courses the student has.
*Follow-up:* What if courses have thousands of lessons? Store `lesson_count` on the course (updated when lessons change) and a completed count on the enrollment.

**Q4. How would you build the instructor analytics dashboard?**
Start from `Course.objects.filter(instructor=request.user)` and annotate students, completions (a filtered `Count`) and average rating, returning plain values. Use `distinct=True` because two relations are joined, and move heavy aggregates into subqueries, a cache or a summary table refreshed by background jobs as data grows.
*Follow-up:* Why scope first? It is both the security boundary and the cheapest filter.

**Q5. How does course moderation work, and how do you avoid race conditions?**
Courses move through draft, pending, published and rejected. Each transition is a conditional update (`filter(pk=pk, status="pending").update(status="published")`), and the affected row count tells you whether it happened, so two admins cannot both act and invalid jumps are impossible. Notify the instructor through `on_commit` after the transition.
*Follow-up:* Why not read, check and save? Two requests can read the same status before either saves.

**Q6. How do subscriptions work end to end?**
The React app asks the API for a checkout session; Django creates or reuses the provider customer and a subscription-mode Checkout Session and returns its URL; the user pays on the hosted page; webhooks then create and update the local `Subscription` row, and access checks read that row. Renewals, failures and cancellations arrive as further webhooks.
*Follow-up:* Why hosted checkout? Card data never touches your servers, which keeps PCI scope minimal.

**Q7. Why are webhooks the source of truth instead of the success redirect?**
The redirect happens in the user's browser: it can be skipped, replayed or forged, and payment confirmation may complete later. Webhooks come from the provider, carry a verifiable signature and cover the whole lifecycle (renewals, failed payments, cancellations), so only they update subscription state.
*Follow-up:* What does the success page do then? Shows "processing" and polls `/api/me/` until the webhook has updated the subscription.

**Q8. How do you make webhook handling idempotent and robust to ordering?**
Verify the signature on the raw body, then inside a transaction insert the event ID into a table with a unique constraint; duplicates return 200 immediately. Handlers re-fetch the current object from the provider (or compare timestamps) instead of trusting event order, use `update_or_create`, and push slow work to Celery with `on_commit`. A nightly reconciliation job repairs anything missed.
*Follow-up:* What if processing fails halfway? The transaction rolls back, including the event row, so the provider's retry processes it again.

**Q9. How do you gate content on an active subscription?**
A DRF permission allows preview lessons for everyone, then requires an enrollment plus either a free course or a subscription whose status is active or trialing and whose `current_period_end` is in the future. It runs on the lesson detail and playback endpoints, so the signed video URL is only issued after the check.
*Follow-up:* What about `past_due` users? A product decision: some platforms keep access during the retry window, others do not; encode it explicitly.

**Q10. What happens when a payment fails?**
The provider marks the invoice failed and the subscription `past_due`, and sends `invoice.payment_failed`; you email the user a customer-portal link to update the card. The provider retries according to its dunning settings; if every retry fails, the subscription becomes `canceled` or `unpaid`, a subscription webhook arrives, and access ends.
*Follow-up:* How do you test this? Test clocks and test cards in the provider's test mode, plus unit tests that feed recorded events to the handler.

**Q11. How do you protect paid video content?**
Keep originals and renditions in private storage, transcode to HLS, and serve through a CDN with short-lived signed URLs or signed cookies (or a video platform's signed playback tokens) issued only after the access check. That stops link sharing; DRM and watermarking address determined copying, and screen recording cannot be fully prevented.
*Follow-up:* Why not presigned S3 URLs per video? An HLS stream has many segment files, so signing at the CDN level is far simpler.

**Q12. How would you implement course search?**
Begin with `SearchFilter` and django-filter for keyword, category, level and price, plus `OrderingFilter`. As the catalog grows, switch to PostgreSQL full-text search (`SearchVector`, `SearchQuery`, `SearchRank`, a stored vector with a GIN index, trigram similarity for typos), and adopt a dedicated search engine when facets and relevance tuning matter.
*Follow-up:* How do you keep a search engine in sync? Index on `on_commit` after writes, and run a periodic full reindex as a safety net.

**Q13. How do you prevent cheating on quizzes through the API?**
Never send correct answers to the client: student serializers omit `is_correct`, grading happens on the server from submitted choice IDs, and validation checks that every choice belongs to the quiz. Rate-limit attempts or cap them per quiz, and record attempts so instructors can review them.
*Follow-up:* Why validate choice ownership? Otherwise a student could submit a choice ID from another quiz that they know is correct.

**Q14. How would you issue certificates?**
When the last lesson is completed (and required quizzes passed), mark the enrollment complete in a transaction and enqueue a certificate task with `on_commit`. The task is idempotent (`get_or_create` on a one-to-one certificate, skip if a PDF already exists), renders the PDF, stores it and emails the link; a public page verifies certificates by their UUID code.
*Follow-up:* Why a UUID code? It is unguessable and safe to print for verification, unlike sequential IDs.

**Q15. What would you change to scale the LMS to many more users?**
Serve video entirely from a CDN or video platform, cache the catalog and course pages, add read replicas or a pooler for the database, precompute dashboard aggregates, move search to a dedicated engine, run more Celery workers with separate queues for slow jobs, and add monitoring and alerts on webhook failures and queue latency.
*Follow-up:* What would you measure first? Slow endpoints and query counts under realistic load, before adding any infrastructure.

## Further reading
- [Django full-text search](https://docs.djangoproject.com/en/stable/ref/contrib/postgres/search/)
- [DRF permissions](https://www.django-rest-framework.org/api-guide/permissions/)
- [Django database transactions](https://docs.djangoproject.com/en/stable/topics/db/transactions/)
