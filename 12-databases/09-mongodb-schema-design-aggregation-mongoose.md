# MongoDB Schema Design, Aggregation and Mongoose

> **Tier:** T1 (know cold) · **Related:** [MongoDB fundamentals](08-mongodb-fundamentals.md), [DBMS fundamentals: normalization and denormalization](01-dbms-fundamentals.md), [Next.js backend APIs and MongoDB](../07-nextjs-and-tanstack/03-nextjs-backend-apis-and-mongodb.md), [RemoteDev project deep dive](../17-projects-deep-dive/06-remotedev-nextjs-mongodb.md), [LMS project deep dive](../17-projects-deep-dive/05-lms-coursera-clone.md)
> **On your resume:** RemoteDev internship (Next.js with TypeScript and MongoDB APIs) and the LMS (Coursera clone) you proposed and built at PureLogics. The LMS used DRF; modeling it in MongoDB below is a design exercise that makes a great interview comparison.

## Quick revision
- Golden rule: **data that is read together is stored together**. Design around queries, not entities.
- **Embed** when the child is owned by the parent, bounded, small and read with it (one-to-few: addresses). **Reference** when it is unbounded, shared, accessed on its own or updated independently.
- Cardinality: **one-to-few** embed; **one-to-many** (up to thousands) array of ids in the parent or parent id in each child; **one-to-squillions** (unbounded) parent id in each child, indexed.
- Patterns: **subset** (embed the latest N, rest elsewhere), **extended reference** (copy a few fields of the referenced doc), **bucket** (group time-series points per period), **computed** (store precomputed totals), **outlier** (special-case the few huge documents), **polymorphic** (several shapes in one collection with a `type` field).
- 16 MB document limit; unbounded arrays slow every write long before reaching it.
- Every duplicated field needs a consistency decision: snapshot on purpose, update everywhere, or eventually reconcile.
- Aggregation pipeline: `$match` -> `$group` -> `$sort` -> `$limit` -> `$lookup` -> `$project`; also `$unwind`, `$addFields`, `$facet`, `$merge`. Put `$match` (and `$sort` + `$limit`) first so indexes apply; each stage has a memory limit.
- Mongoose: schemas with types, validation, defaults, virtuals, methods and statics, middleware, `populate`, `lean()`, `timestamps`, schema indexes, `InferSchemaType` for TypeScript.
- `populate` runs one extra `$in` query per populated path, not per document; populating inside a loop is the real N+1.
- Update queries (`updateOne`, `findOneAndUpdate`) skip `save` middleware and run validators only with `runValidators: true`.
- `unique: true` builds an index; it is not a validator.
- In Next.js and serverless, cache the Mongoose connection promise on `globalThis` and reuse compiled models.

## 1. Embed vs reference

| Question | Points to embedding | Points to referencing |
|---|---|---|
| Is the child read with the parent almost every time? | Yes | No, it is often read alone |
| How many children? | A few, bounded | Hundreds, thousands, unbounded |
| Who owns the child's lifecycle? | The parent (delete together) | The child lives independently or is shared |
| How often does the child change, and by whom? | Rarely, with the parent | Frequently, by many writers |
| Must parent and child change atomically? | Yes (single-document atomicity) | No, or a transaction is acceptable |
| Is the child large but rarely needed? | No | Yes (keep the parent small) |

By cardinality:
- **One-to-few**: embed. `user.addresses: [{ label, line1, city }]`.
- **One-to-many** (dozens to a few thousand): keep an array of ids in the parent when you load children from the parent (`product.partIds`), or a parent id in each child when you mostly query children (`variant.productId`).
- **One-to-squillions** (unbounded: log lines per device, messages per chat): always store the parent id in each child with an index on it. Never an array in the parent.
- **Many-to-many**: arrays of ids on one or both sides if both are small and bounded; otherwise a separate collection for the relationship, especially when the relationship has its own data (an enrollment with progress).

## 2. Schema design patterns

| Pattern | Problem it solves | Example |
|---|---|---|
| **Subset** | A large related list makes the parent heavy, but pages show only a few | `course.recentReviews` holds the newest 10 (bounded with `$push` + `$slice`); all reviews live in `reviews` |
| **Extended reference** | Listing pages need a couple of fields from another document, and `$lookup` on every request is wasteful | `order.customer = { _id, name, city }`; the full customer stays in `customers` |
| **Bucket** | Millions of tiny time-series documents bloat indexes | Smartwatch readings grouped per device per hour: `{ deviceId, hourStart, count, readings: [...] }`; MongoDB 5.0+ time series collections do this natively |
| **Computed** | Expensive aggregates recomputed on every read | `course.stats = { enrolledCount, ratingCount, ratingSum }` updated with `$inc` on each write |
| **Outlier** | A few documents are huge (a celebrity with millions of followers) and break the common design | Normal documents embed; outliers set `hasOverflow: true` and spill into extra documents |
| **Polymorphic** | Related things with different shapes are queried together | One `notifications` collection whose `type` is `rent_due`, `event_invite` or `bill_split`, each with its own fields (Mongoose discriminators) |

A related habit: add a `schemaVersion` field when document shapes change, so code can handle old and new shapes while a background job migrates them.

## 3. The 16 MB limit and unbounded arrays

A document can be at most 16 MB, but trouble starts much earlier:
- Every update to a big document rewrites it; every read without a projection sends it over the network.
- Multikey indexes create one entry per array element, so a growing array inflates the index and slows writes.
- `$push` without a bound (`$slice`) grows forever; the arrays look harmless in development with five items.

Fixes: subset (bounded embed plus a full collection), bucket, or child references (`parentId` on each child).

## 4. Denormalization and consistency

Duplicated data must be classified field by field:

| Kind | Example | Strategy |
|---|---|---|
| Snapshot (must not change) | Price, product name and address inside an order | Copy at write time and never update: it is history |
| Display copy that should follow the source | Author name on posts, course title in enrollments | `updateMany` when the source changes (same request or a job), possibly in a transaction |
| Derived counters | `enrolledCount`, `ratingSum` | `$inc` atomically with the write; a periodic job recomputes to fix drift |
| Read models for dashboards | Monthly revenue per store | Rebuild on a schedule with `$merge`, or update from change streams |

Prefer duplicating fields that rarely change (names, slugs) over volatile ones (stock, prices). Change streams (a replica-set feature) let a worker react to source changes and update copies.

## 5. The aggregation pipeline

An aggregation is an array of stages; documents flow through them in order.

| Stage | Does | SQL analogy |
|---|---|---|
| `$match` | Filter documents | WHERE |
| `$project` / `$unset` | Reshape: include, exclude, compute fields | SELECT list |
| `$addFields` / `$set` | Add computed fields, keep the rest | SELECT *, expr |
| `$group` | Group by `_id` with accumulators (`$sum`, `$avg`, `$min`, `$max`, `$push`, `$addToSet`, `$first`, `$last`) | GROUP BY |
| `$sort`, `$skip`, `$limit` | Order and page | ORDER BY, OFFSET, LIMIT |
| `$unwind` | One output document per array element | Joining a child table |
| `$lookup` | Left outer join to another collection | LEFT JOIN |
| `$facet` | Several sub-pipelines over the same input | Several queries at once |
| `$count`, `$bucket` | Count, histogram buckets | COUNT, CASE ranges |
| `$merge` / `$out` | Write results into a collection | INSERT ... SELECT / materialized view |

**Revenue per store** (orders shaped as in [08](08-mongodb-fundamentals.md)):
```js
db.orders.aggregate([
  { $match: {                                         // first: can use an index on (status, createdAt)
      status: { $in: ["paid", "shipped"] },
      createdAt: { $gte: ISODate("2025-06-01"), $lt: ISODate("2025-07-01") }
  } },
  { $group: {
      _id: "$storeId",
      revenueCents: { $sum: "$totalCents" },
      orders: { $sum: 1 },
      avgOrderCents: { $avg: "$totalCents" }
  } },
  { $sort: { revenueCents: -1 } },
  { $limit: 10 },                                     // limit before the join: only 10 lookups
  { $lookup: { from: "stores", localField: "_id", foreignField: "_id", as: "store" } },
  { $unwind: "$store" },
  { $project: {
      _id: 0, storeId: "$_id", storeName: "$store.name", revenueCents: 1, orders: 1,
      avgOrderCents: { $round: ["$avgOrderCents", 0] }
  } }
]);
```

**Top products** (unwinding the embedded items):
```js
db.orders.aggregate([
  { $match: { status: { $in: ["paid", "shipped"] } } },
  { $unwind: "$items" },                              // one document per order line
  { $group: {
      _id: "$items.sku",
      name: { $first: "$items.name" },
      unitsSold: { $sum: "$items.qty" },
      revenueCents: { $sum: { $multiply: ["$items.qty", "$items.priceCents"] } }
  } },
  { $sort: { unitsSold: -1 } },
  { $limit: 5 }
]);
```

**Course enrollment stats** for an instructor dashboard (enrollments as in section 8):
```js
db.enrollments.aggregate([
  { $match: { enrolledAt: { $gte: ISODate("2025-01-01") } } },
  { $group: {
      _id: "$courseId",
      enrolled: { $sum: 1 },
      completed: { $sum: { $cond: [{ $eq: ["$status", "completed"] }, 1, 0] } },
      avgProgress: { $avg: "$progress.percent" }
  } },
  { $addFields: {
      completionRate: { $round: [{ $multiply: [{ $divide: ["$completed", "$enrolled"] }, 100] }, 1] }
  } },
  { $lookup: {                                        // localField + pipeline together needs MongoDB 5.0+
      from: "courses", localField: "_id", foreignField: "_id", as: "course",
      pipeline: [{ $project: { title: 1, instructorId: 1 } }]
  } },
  { $unwind: "$course" },
  { $sort: { enrolled: -1 } }
]);
```

**`$facet`** for a search page (results, total and filter counts in one round trip):
```js
db.products.aggregate([
  { $match: { tags: "accessories", priceCents: { $lte: 10000 } } },
  { $facet: {
      results: [{ $sort: { priceCents: 1 } }, { $limit: 20 }, { $project: { name: 1, priceCents: 1 } }],
      total:   [{ $count: "count" }],
      byBrand: [{ $group: { _id: "$brand", count: { $sum: 1 } } }, { $sort: { count: -1 } }]
  } }
]);
```

**Performance rules:**
- Only stages at the start of the pipeline can use indexes: begin with `$match`, and `$sort` (plus `$limit`) on indexed fields.
- `$sort` followed by `$limit` becomes a cheap top-N sort.
- Shrink early: filter before `$unwind` (which multiplies documents) and before `$lookup` (which runs per input document; index the `foreignField`).
- Each stage has a memory limit (100 MB); large sorts and groups need to spill to disk (`allowDiskUse`; newer versions allow disk use by default, so check yours).
- Inspect with `db.orders.explain("executionStats").aggregate([...])`.
- For dashboards hit often, precompute into a summary collection with `$merge` on a schedule instead of aggregating per request.

## 6. Mongoose

Mongoose is an ODM (object document mapper) for Node.js: schemas, validation, middleware and a query builder on top of the official driver.

```ts
import mongoose, { Schema, model, models, type InferSchemaType, type HydratedDocument } from "mongoose";

await mongoose.connect(process.env.MONGODB_URI!, { maxPoolSize: 10 });
```

### 6.1 Schemas, types, validation and defaults
```ts
const lessonSchema = new Schema({
  title: { type: String, required: true, trim: true },
  durationMin: { type: Number, min: 1 },
  kind: { type: String, enum: ["video", "article", "quiz"], default: "video" },
});

const courseSchema = new Schema(
  {
    title: { type: String, required: [true, "Title is required"], trim: true, maxlength: 120 },
    slug: {
      type: String, required: true, unique: true, lowercase: true,      // unique builds an index
      validate: { validator: (v: string) => /^[a-z0-9-]+$/.test(v), message: "Invalid slug" },
    },
    instructor: { type: Schema.Types.ObjectId, ref: "User", required: true, index: true },
    level: { type: String, enum: ["beginner", "intermediate", "advanced"], default: "beginner" },
    priceCents: { type: Number, min: 0, default: 0 },
    tags: { type: [String], default: [] },
    lessons: { type: [lessonSchema], default: [] },                       // embedded, bounded
    ratingCount: { type: Number, default: 0 },
    ratingSum: { type: Number, default: 0 },
    published: { type: Boolean, default: false },
    deletedAt: { type: Date, default: null },
  },
  {
    timestamps: true,                                                     // createdAt and updatedAt
    methods: {                                                            // typed instance methods
      addRating(stars: number) {
        this.ratingCount += 1;
        this.ratingSum += stars;
        return this.save();
      },
    },
    statics: {                                                            // typed model-level helpers
      findPublished() {
        return this.find({ published: true, deletedAt: null }).sort({ createdAt: -1 });
      },
    },
  }
);

courseSchema.index({ instructor: 1, createdAt: -1 });                     // compound index

courseSchema.virtual("averageRating").get(function () {                   // computed, not stored
  return this.ratingCount ? this.ratingSum / this.ratingCount : null;
});
courseSchema.set("toJSON", { virtuals: true });
```

- Validators (`required`, `min`, `max`, `enum`, `match`, `maxlength`, custom `validate`) run on `save()` and `create()`. Query updates (`updateOne`, `findOneAndUpdate`) validate only with `{ runValidators: true }`, and even then with limits.
- `unique: true` is not a validator: it asks MongoDB to build a unique index. A duplicate surfaces as a duplicate-key error (code 11000) from the server, and only if the index actually exists.
- `autoIndex` builds schema indexes when the app starts, which is handy in development; in production, disable it and create indexes deliberately (a script or `Model.syncIndexes()` during deploys).

### 6.2 Middleware (hooks)
```ts
courseSchema.pre("validate", async function () {         // document middleware: `this` is the document
  if (this.isModified("title") && !this.slug) {          // runs before validation, so `required` sees the slug
    this.slug = this.title.toLowerCase().replace(/[^a-z0-9]+/g, "-");
  }
});

courseSchema.post("save", function (doc) {
  console.log(`Course ${doc._id} saved`);
});

courseSchema.pre("find", function () {                   // query middleware: `this` is the query
  this.where({ deletedAt: null });                       // hide soft-deleted courses by default
});
```

Classic gotcha: `findOneAndUpdate`, `updateOne` and `updateMany` do **not** run `save` hooks, so logic such as password hashing in `pre("save")` is silently skipped when another code path updates with a query. Either always use `doc.save()` for those fields or add matching query middleware.

### 6.3 populate and its N+1 risk
```ts
const enrollments = await EnrollmentModel.find({ user: userId })
  .populate("course", "title slug")        // one extra query: courses where _id in [...]
  .lean();
```
`populate` runs the main query, collects the referenced ids, then runs one `$in` query per populated path, and stitches results together in Node. That is 1 + 1 queries, not N + 1, but:
- Populating inside a loop (`for (const e of list) await e.populate("course")`) is a real N+1.
- Nested and multiple populates add a round trip each; deep chains get slow.
- For heavy read paths, use an aggregation with `$lookup` (one server-side query) or denormalize the few needed fields (extended reference).

### 6.4 lean()
```ts
const courses = await CourseModel.find({ published: true }).select("title slug priceCents").lean();
```
`lean()` returns plain JavaScript objects instead of Mongoose documents: much less memory and CPU, ideal for read-only API responses. Trade-offs: no `save()`, no instance methods, and no virtuals or getters unless you add plugins or transform the output yourself.

### 6.5 TypeScript with InferSchemaType
```ts
export type Course = InferSchemaType<typeof courseSchema>;   // plain shape inferred from the schema
export type CourseDocument = HydratedDocument<Course>;       // a full Mongoose document

const createCourseModel = () => model("Course", courseSchema);
type CourseModelType = ReturnType<typeof createCourseModel>;

// Reuse the compiled model when Next.js hot reload re-runs this module (avoids OverwriteModelError)
export const CourseModel = (models.Course as CourseModelType | undefined) ?? createCourseModel();
```
With methods and statics declared in the schema options, `CourseModel.findPublished()` and `doc.addRating(5)` are typed. Whether `timestamps` fields appear in the inferred type depends on your Mongoose version, so check before relying on it.

## 7. Connection caching in serverless and Next.js

In Next.js development, hot reload re-runs modules; in serverless deployments, each warm instance keeps module state between invocations but cold starts begin fresh. Calling `mongoose.connect` per request opens new pools and quickly hits Atlas connection limits. Cache the connection promise on `globalThis`:

```ts
// lib/db.ts
import mongoose from "mongoose";

type Cache = { conn: typeof mongoose | null; promise: Promise<typeof mongoose> | null };
const globalForMongoose = globalThis as typeof globalThis & { mongooseCache?: Cache };
const cache: Cache = (globalForMongoose.mongooseCache ??= { conn: null, promise: null });

export async function connectDB() {
  if (cache.conn) return cache.conn;
  cache.promise ??= mongoose.connect(process.env.MONGODB_URI!, { bufferCommands: false, maxPoolSize: 10 });
  try {
    cache.conn = await cache.promise;
  } catch (err) {
    cache.promise = null;              // let the next request retry instead of reusing a failed promise
    throw err;
  }
  return cache.conn;
}
```

Call `await connectDB()` at the top of each route handler or server action that touches the database. `bufferCommands: false` makes queries fail fast instead of queueing silently when the connection is down. Keep `maxPoolSize` small, since many instances each open their own pool. More context: [Next.js backend APIs and MongoDB](../07-nextjs-and-tanstack/03-nextjs-backend-apis-and-mongodb.md).

## 8. Modeling an LMS in MongoDB

A Coursera-style LMS with students, instructors, courses, lessons, enrollments with progress, reviews and subscriptions:

| Collection | Shape (abridged) | Why |
|---|---|---|
| `users` | `{ name, email (unique), role, subscription: { plan, status, currentPeriodEnd } }` | Subscription is one-to-one and read with the user: embed |
| `courses` | `{ title, slug (unique), instructorId, instructorName, level, priceCents, published, sections: [{ title, lessons: [{ _id, title, durationMin, kind }] }], stats: { enrolledCount, ratingCount, ratingSum }, recentReviews: [...] }` | The outline is bounded and always shown with the course (embed); instructor name is an extended reference; stats are computed; reviews use the subset pattern |
| `lessonContents` | `{ _id: lessonId, courseId, body, videoUrl, attachments }` | Large and loaded only when a lesson opens: reference |
| `enrollments` | `{ userId, courseId, courseTitle, status, progress: { completedLessonIds, percent }, enrolledAt, completedAt }` | Many-to-many with its own data: a separate collection; unique index `{ userId: 1, courseId: 1 }` |
| `reviews` | `{ courseId, userId, rating, text, createdAt }` | Unbounded per course; unique `{ courseId: 1, userId: 1 }` |
| `quizAttempts` | `{ userId, courseId, quizId, score, answers, submittedAt }` | Unbounded per user: child references |

Key operations:
```js
// Course page: one document read, no joins
db.courses.findOne({ slug: "intro-to-sql", published: true });

// Mark a lesson complete: atomic within one enrollment document, idempotent thanks to $addToSet
db.enrollments.updateOne(
  { userId: uid, courseId: cid },
  { $addToSet: { "progress.completedLessonIds": lessonId } }
);

// "My learning": enrollments by user, newest first (index { userId: 1, enrolledAt: -1 })
db.enrollments.find({ userId: uid }).sort({ enrolledAt: -1 }).limit(20);
```

Compared with a relational design (an enrollments table with foreign keys and a unique `(user_id, course_id)`, plus one `lesson_progress` row per completed lesson), the document version trades database-enforced integrity and flexible reporting for single-read pages and simpler writes. Either works; the interview value is explaining the trade-off.

## Common pitfalls
- Designing collections like SQL tables (one per entity, joined with `populate` everywhere), getting the costs of both worlds.
- Embedding unbounded lists (all reviews, all messages, all log events).
- Duplicating volatile data (stock, prices) and forgetting to update the copies.
- Starting a pipeline with `$unwind` or `$lookup` instead of `$match`, so no index is used and every document flows through.
- Expecting `pre("save")` hooks or validators to run on `updateOne` / `findOneAndUpdate`.
- Treating `unique: true` as validation, or relying on `autoIndex` in production.
- Calling `populate` inside loops; deep populate chains on hot endpoints.
- Returning full Mongoose documents from read-only APIs instead of using `lean()` and `select()`.
- Creating a new connection per request in Next.js or serverless; recompiling models on hot reload (OverwriteModelError).
- Mixing `ObjectId` and string ids in the same field, so equality queries silently miss.

## How to talk about this in an interview
- RemoteDev is your MongoDB story: backend APIs in Next.js (TypeScript) with MongoDB. Explain one design decision in embed-vs-reference terms and one aggregation or index you used.
- The LMS is a strong comparison story: you built it at PureLogics with Django REST Framework and React, so you can contrast its relational model with how you would model it in MongoDB.
- Show judgment: "I'd embed the course outline and reference lesson content and enrollments, because..." beats listing patterns.

> **Fill in:** whether RemoteDev used Mongoose or the native driver, the main collections, and one embed-or-reference decision you made or saw.

> **Fill in:** which database backed the PureLogics LMS and how enrollments and progress were modeled there.

> **Fill in:** an aggregation pipeline you wrote (what it computed and which stages it used), if any.

## Interview questions

**Q1. How do you decide between embedding and referencing in MongoDB?**
Embed when the data is owned by the parent, bounded in size, read together and needs atomic updates with it (order lines, addresses). Reference when it is unbounded, shared between parents, accessed independently or updated by different writers (reviews, enrollments, users). Start from the application's queries: data read together is stored together.
*Follow-up:* Can you mix both? Yes: the subset and extended reference patterns embed a small part and reference the rest.

**Q2. What do one-to-few, one-to-many and one-to-squillions mean for schema design?**
They are cardinality buckets. One-to-few (a handful): embed. One-to-many (up to thousands): an array of ids in the parent or a parent id in each child, depending on query direction. One-to-squillions (unbounded): only a parent id in each child, indexed, because any array would grow without limit.

**Q3. Why are unbounded arrays a problem even below 16 MB?**
Each update rewrites a growing document, reads transfer it all unless projected, and multikey indexes add an entry per element, slowing writes and wasting memory. Eventually the document hits the 16 MB limit and writes fail. Bound arrays with `$slice` or move items to their own collection.

**Q4. Explain the subset, extended reference, bucket and computed patterns.**
Subset: embed only the most relevant N items (latest reviews) and keep the full list elsewhere. Extended reference: copy a few frequently needed fields of a referenced document (customer name in an order). Bucket: group many small time-series points into one document per period. Computed: store precomputed values (counts, sums) updated on write instead of aggregating on every read.

**Q5. How do you keep denormalized data consistent?**
Decide per field. Snapshots (price at purchase) never update. Display copies get updated with `updateMany` when the source changes, in the same request, a background job or a change-stream worker, optionally in a transaction. Counters use atomic `$inc` with a periodic recompute to fix drift. Avoid duplicating volatile fields.

**Q6. Walk me through an aggregation that computes revenue per store.**
`$match` paid orders in the date range (using an index), `$group` by `storeId` summing `totalCents` and counting, `$sort` by revenue descending, `$limit` 10, then `$lookup` the stores collection for names, `$unwind` the joined array and `$project` the final shape. Filtering and limiting before the lookup keeps it cheap.

**Q7. `$lookup` or `populate`?**
`$lookup` joins inside the database in one aggregation, so it suits reporting and heavy read paths, and it can filter and project the joined side. `populate` is Mongoose's application-side join: an extra `$in` query per path, convenient for simple CRUD on documents. For hot endpoints, consider denormalizing instead of either.

**Q8. What does `$unwind` do, and what is the gotcha?**
It outputs one document per element of an array field, so you can group or filter by element. It multiplies document counts (an order with 5 items becomes 5 documents), and by default drops documents whose array is empty or missing; use `preserveNullAndEmptyArrays: true` to keep them. Filter before unwinding.

**Q9. How do you make aggregations fast?**
Start with `$match` (and `$sort`/`$limit`) on indexed fields, because only leading stages use indexes; reduce documents before `$unwind` and `$lookup`; index `$lookup` foreign fields; project away large fields; watch the per-stage memory limit; and precompute frequently viewed results with `$merge`. Verify with `explain`.

**Q10. What is `$facet` used for?**
Running several sub-pipelines on the same filtered input in one request, such as the current page of results, the total count and counts per filter value for a search page. Each facet's output is an array in a single result document.

**Q11. What is `populate`, and does it cause N+1 queries?**
It replaces stored ObjectIds with the referenced documents by running one extra query per populated path using `$in` over all collected ids, so `find().populate()` is two queries, not N+1. It becomes N+1 when you populate per document inside a loop, and deep or multiple populates add round trips.

**Q12. What does `lean()` do, and when should you not use it?**
It returns plain objects instead of hydrated Mongoose documents, which is faster and lighter for read-only responses. Do not use it when you need to modify and `save()` the document, call instance methods, or rely on virtuals, getters or defaults applied on hydration.

**Q13. Why didn't my `pre("save")` hook run when I called `findOneAndUpdate`?**
Save middleware runs only for `doc.save()` and `Model.create()`. Query methods (`updateOne`, `findOneAndUpdate`, `updateMany`) trigger query middleware instead, where `this` is the query. Either update through `save()` or add `pre("findOneAndUpdate")` logic.

**Q14. Why didn't Mongoose validate my `updateOne` call?**
Validators run on save by default. For update queries you must pass `{ runValidators: true }`, and update validators have limits (for example `this` is not the document). Validate inputs at the API boundary (Zod) as well.

**Q15. Is `unique: true` a validator in Mongoose?**
No. It is an index option: Mongoose asks MongoDB to create a unique index, and duplicates are rejected by the server with error code 11000 (if the index exists). Handle that error to return a friendly message, and manage indexes explicitly in production.

**Q16. How do you get good TypeScript types with Mongoose?**
Define the schema once and infer types from it: `InferSchemaType<typeof schema>` for the plain shape and `HydratedDocument<T>` for documents. Declare methods and statics in schema options so they are typed, and avoid duplicating a manual interface that drifts from the schema.

**Q17. How do you manage Mongoose connections in Next.js or serverless?**
Cache the connection promise on `globalThis` so hot reloads and warm invocations reuse one pool, reset the cached promise if connecting fails, keep `maxPoolSize` small, and reuse compiled models (`models.X ?? model(...)`) to avoid OverwriteModelError.

**Q18. How would you model an LMS in MongoDB?**
Courses embed their bounded outline (sections and lesson titles), computed stats and a subset of recent reviews; lesson content lives in its own collection; enrollments are a separate collection with progress and a unique `{ userId, courseId }` index; reviews and quiz attempts reference their course and user. Course pages are one read, progress updates are atomic per enrollment, and dashboards use aggregations.

## Further reading
- MongoDB data modeling: https://www.mongodb.com/docs/manual/data-modeling/
- Aggregation pipeline: https://www.mongodb.com/docs/manual/core/aggregation-pipeline/
- Mongoose documentation: https://mongoosejs.com/docs/guide.html
