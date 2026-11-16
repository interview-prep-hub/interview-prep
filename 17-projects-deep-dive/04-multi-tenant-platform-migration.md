# Multi-Tenant Platform Migration: oRPC, TanStack Start and Drizzle

> **Tier:** T1 (know cold) · **Related:** [Multi-tenancy](../13-architecture-and-practices/03-multi-tenancy.md), [Platform migration strategies](../13-architecture-and-practices/04-platform-migration-strategies.md), [oRPC and type-safe APIs](../09-nodejs-backend/05-orpc-and-typesafe-apis.md), [TanStack Start](../07-nextjs-and-tanstack/05-tanstack-start.md), [Drizzle ORM](../12-databases/13-drizzle-orm.md), [ORMs compared](../12-databases/12-orms-compared.md), [Colochousing web and API](02-colochousing-web-and-api.md), [System design from your projects](../15-system-design/03-case-studies-from-your-projects.md)
> **On your resume:** (Project Colochousing, Devntech client) contributed to migrating onto a new multi-tenant TypeScript platform (oRPC, TanStack Start, Drizzle): Rails sync APIs, marketplace page migration and org-scoped access control.

## Quick revision

| Item | What to remember |
|---|---|
| Where | Colochousing, a Devntech client, during your Devntech role (Nov 2025 to present) |
| What | Contributed to migrating from the Rails app onto a new multi-tenant TypeScript platform |
| New stack | oRPC (type-safe procedures), TanStack Start (full-stack React framework on TanStack Router and Vite), Drizzle (TypeScript ORM) |
| Your three areas | Rails sync APIs, marketplace page migration, org-scoped access control |
| The key word | "Contributed": a team effort in which you own specific pieces |

- **"I" vs "we":** the rewrite decision, the stack and the overall plan are probably the team's or the client's; the sync APIs, the marketplace page work and the access-control pieces you built are yours. When you explain a decision you did not make, say so and give the reasoning as you understood it.
- **Multi-tenant:** one platform serves many organizations; every tenant-owned row carries an `org_id`; every request runs in exactly one org context.
- **Org-scoped access control:** resolve the org from the request, verify membership and role, filter every query by `org_id`. Defense in depth: one middleware, scoped data helpers, cross-org tests, optionally PostgreSQL row-level security.
- **Rails sync APIs:** while both systems run, data has to move between them. Typical design: Rails exposes changes through an authenticated API with a keyset cursor `(updated_at, id)`, and the new side upserts idempotently, keyed by the Rails ID (`legacy_id`).
- **Deletes and drift:** tombstones or soft deletes so deletions sync, plus a periodic reconciliation.
- **Marketplace migration:** keep URLs (or 301 them), titles, meta descriptions, canonical tags and structured data; server-render so content is in the initial HTML; verify with crawls and search console data.
- **Strangler fig:** move one page or feature at a time behind a routing layer, keep the old system running until the new one has parity, keep a way back.
- **oRPC vs tRPC:** both give end-to-end types; oRPC adds first-class OpenAPI support and optional contract-first development, and has no query/mutation split.
- **TanStack Start vs Next.js:** type-safe routing and search params on Vite with server functions, vs Next.js's mature, Server-Components-first ecosystem. Start reached release-candidate status in 2025, so APIs moved: hedge details.
- **Drizzle vs Prisma:** a TypeScript schema and SQL-shaped queries with no generated client, vs Prisma's schema language and generated client.

> **Fill in:** when you joined relative to the start of the migration, who led it, and the exact pieces you built.

## The 30-second pitch

> "I contributed to Colochousing's migration from its Rails app onto a new multi-tenant TypeScript platform built with oRPC, TanStack Start and Drizzle. My parts were the sync APIs on the Rails side that keep the two systems' data in step during the migration, moving the marketplace page onto the new stack, and org-scoped access control, so that every request is tied to one organization and can only reach that organization's data."

> **Fill in:** one sentence on the piece you are proudest of and one concrete detail (for example how the sync avoids duplicates, or how you tested that orgs cannot see each other's data).

## Domain primer

### Multi-tenancy in one table
A tenant here is an organization: a group of users who share data and must never see another group's data. Three common isolation models (depth in [multi-tenancy](../13-architecture-and-practices/03-multi-tenancy.md)):

| Model | How | Pros | Cons |
|---|---|---|---|
| Shared schema, `org_id` column | All tenants in the same tables | Simple, cheap, easy cross-tenant reporting | Every query must filter; one bug can leak data |
| Schema per tenant | Same tables in one schema per tenant | Stronger separation | Migrations run N times; many schemas to manage |
| Database per tenant | One database per tenant | Strongest isolation, per-tenant backups | Expensive, complex routing and operations |

"Org-scoped access control" points to the first model, or at least to org-level checks in the application. Confirm which one the platform uses.

### The org model
Organizations have members; a membership links a user to an org with a role (owner, admin, member, and so on); resources (houses, listings, bookings) belong to an org. Authorization asks two questions: is this user a member of this org, and does their role allow this action?

### Why a company migrates to a platform like this
Typical reasons (do not present them as the real ones unless they are):
- Serve several organizations (operators, brands or partners) from one platform instead of one app per customer.
- One language and shared types across the web app, the API and a React Native app.
- End-to-end type safety to catch contract breaks at compile time.
- Modernize a codebase that has become slow to change.

The costs are real too: a long period running two systems, feature parity work, data migration, and the risk that a rewrite takes far longer than planned.

> **Fill in:** the reasons the team or client actually gave for the migration, and what stayed in Rails.

### Migration vocabulary

| Term | Meaning |
|---|---|
| Strangler fig | Replace a system piece by piece behind a routing layer until the old one can be retired |
| Backfill | A one-time bulk copy of existing data into the new system |
| Incremental sync | Ongoing copying of changes after the backfill |
| Source of truth | The system whose copy wins when the two disagree |
| Legacy ID | The old system's primary key, stored on the new row to map the two |
| Tombstone | A record that says "this was deleted", so deletions can be synced |
| Parity | The new feature behaves like the old one (content, URLs, rules) |
| Cutover | The moment traffic or ownership switches to the new system |

## Typical architecture

> A typical setup for this kind of migration, to adapt to what you actually saw. It is not a description of the real platform.

```text
                       users and crawlers
                              |
               routing layer (per path or per page)
              /                                    \
   Rails app (legacy)                       TanStack Start app (new)
   - pages not migrated yet                 - migrated pages (marketplace, ...)
   - Sync API /api/sync/*   <-- pulls ---   - oRPC router + org middleware
   - Sidekiq, PostgreSQL (legacy)           - sync worker: cursor per resource,
                                              idempotent upserts by legacy_id
                                            - Drizzle -> PostgreSQL (new schema,
                                              org_id on tenant-owned tables)
```

The arrow direction above is only one option: the new platform could pull from Rails, Rails could push to the new platform, or both, depending on which system owns which data at each stage.

> **Fill in:** the real direction of the sync, what triggers it (schedule, events, on demand), and which system owns which data today.

## Bullet-by-bullet

### "Contributed to migrating onto a new multi-tenant TypeScript platform (oRPC, TanStack Start, Drizzle)"
- **Signals:** you can work across Rails and TypeScript and understand migrations, not just greenfield code.
- **They will ask:** why rewrite; why this stack; your exact role.
- **Answer:** the reasons as you understood them, the plan (incremental, strangler fig), then "my parts were the sync APIs, the marketplace page and org-scoped access control."

### "Rails sync APIs"
- **They will ask:** direction, cursor design, ID mapping, idempotency, deletes, authentication between the systems, how big the data is, what happens on failure.
- **Answer:** walk through Part A below with your real choices.

> **Fill in:** the resources the sync covered, the endpoint shapes you built, and how the consumer handled retries.

### "Marketplace page migration"
- **They will ask:** what the page does, how you kept URLs and SEO, how data reaches the page in TanStack Start, how you verified parity, how you rolled it out.
- **Answer:** Part C below.

> **Fill in:** what the marketplace page shows and who uses it (public visitors, applicants, organizations), and what you changed vs ported as is.

### "Org-scoped access control"
- **They will ask:** what it means, where it is enforced, how roles work, how you prevent and test cross-org leaks.
- **Answer:** Part B below; stress layers and tests.

> **Fill in:** where org scoping lives in the real code (middleware, data helpers, row-level security), the roles that exist, and one edge case you handled.

## Design deep-dive

> Typical designs to adapt. Where your real project differs, describe what you actually built and why.

### Part A: Rails sync APIs

Direction options:

| Option | How | Good when | Watch out for |
|---|---|---|---|
| Pull | The new platform polls a Rails sync endpoint with a cursor | The consumer controls pace and retries | Polling latency, cursor correctness |
| Push | Rails jobs send changes (webhooks) to the new platform | Low latency | Retries, ordering, the receiver being down |
| Change data capture | Read the database's change stream (logical replication) | No app code in Rails | Operational complexity |
| Shared database | Both apps read the same tables | Short transitions only | Tight coupling; schema changes break both |

A pull endpoint with a keyset cursor:

```ruby
module Api
  module Sync
    class ListingsController < ActionController::API
      before_action :authenticate_sync_client!
      PAGE_SIZE = 500

      # GET /api/sync/listings?after_updated_at=2026-01-01T10:00:00.123456Z&after_id=42
      def index
        scope = Listing.order(:updated_at, :id).limit(PAGE_SIZE)   # backed by an index on (updated_at, id)
        if params[:after_updated_at].present?
          after = Time.iso8601(params[:after_updated_at])
          scope = scope.where("(updated_at, id) > (?, ?)", after, params[:after_id].to_i)
        end
        rows = scope.to_a

        render json: {
          data: rows.map { |listing| ListingSyncPayload.new(listing).as_json },  # explicit payload, not to_json
          next_cursor: rows.last && { after_updated_at: rows.last.updated_at.iso8601(6), after_id: rows.last.id }
        }
      end

      private

      def authenticate_sync_client!
        token = request.headers["Authorization"].to_s.delete_prefix("Bearer ")
        expected = Rails.application.credentials.dig(:sync_api, :token).to_s
        return if expected.present? && ActiveSupport::SecurityUtils.secure_compare(token, expected)

        head :unauthorized
      end
    end
  end
end
```

Why it is built this way:
- **Keyset cursor, not OFFSET:** offsets skip or repeat rows when data changes between pages and get slower with depth. The `(updated_at, id)` pair is unique and ordered; `id` breaks ties between rows updated in the same microsecond.
- **Microsecond precision:** PostgreSQL timestamps store microseconds, so the cursor must too (`iso8601(6)`), or rows get skipped.
- **An explicit payload class:** the sync contract stays stable when the model gains columns, and private fields never leak.
- **Constant-time token comparison** (`secure_compare`) for the shared secret; rotate it like any credential.

On the new side, map IDs and upsert idempotently:

```sql
-- legacy_id is the Rails primary key; a unique index makes replays harmless.
INSERT INTO listings (id, org_id, legacy_id, title, price_cents, source_updated_at)
VALUES (gen_random_uuid(), $1, $2, $3, $4, $5)
ON CONFLICT (legacy_id) DO UPDATE
SET title = EXCLUDED.title,
    price_cents = EXCLUDED.price_cents,
    source_updated_at = EXCLUDED.source_updated_at
WHERE listings.source_updated_at < EXCLUDED.source_updated_at;   -- never overwrite newer data with older
```

- New rows get their own primary keys (UUIDs here); `legacy_id` keeps the mapping, and related rows look up parents by `legacy_id`. If legacy IDs could repeat across sources, make the unique key `(org_id, legacy_id)` instead.
- Deletes do not appear in an "updated since" feed. Use soft deletes (`deleted_at` bumps `updated_at`), a tombstones table, or a periodic comparison of ID sets.
- Advance the consumer's cursor only after a page is stored, so a crash replays the page instead of skipping it.
- Order matters for foreign keys: sync parents (organizations, houses) before children (listings), or tolerate missing parents and retry.

**Going deeper:** a transaction that commits late can write an `updated_at` earlier than a cursor the consumer already passed, so that row is never read. Mitigate by re-reading a small overlap window behind the cursor (idempotent upserts make it cheap), by using a database sequence or change log instead of timestamps, and by running reconciliation.

> **Fill in:** the real cursor, payload and auth design, and how the consumer stores progress.

### Part B: org-scoped access control

Layers, from most to least important:

| Layer | What it does |
|---|---|
| Request context | Resolve the org (path segment, subdomain or header) and the user from the session |
| Middleware | Check membership and role once; put `orgId` and `role` into a typed context |
| Data access | Every query filters by `context.orgId`; scoped helpers make the safe path the easy path |
| Database | Optional row-level security as a backstop; `org_id` in unique indexes and foreign keys |
| Tests | Cross-org tests for every procedure that takes an ID |

An oRPC middleware plus a scoped Drizzle query (API shape as in the oRPC docs at the time of writing; check your version):

```ts
import { ORPCError, os } from "@orpc/server";
import { z } from "zod";
import { and, eq } from "drizzle-orm";
import { db } from "./db";
import { listings, memberships, organizations } from "./schema";

// Initial context, filled per request by the server adapter (session lookup not shown).
const base = os.$context<{ userId: string | null; orgSlug: string }>();

// Every org-scoped procedure builds on this: membership is checked once, in one place.
export const orgProcedure = base.use(async ({ context, next }) => {
  if (!context.userId) throw new ORPCError("UNAUTHORIZED");

  const [membership] = await db
    .select({ orgId: memberships.orgId, role: memberships.role })
    .from(memberships)
    .innerJoin(organizations, eq(organizations.id, memberships.orgId))
    .where(and(eq(memberships.userId, context.userId), eq(organizations.slug, context.orgSlug)))
    .limit(1);

  if (!membership) throw new ORPCError("NOT_FOUND");           // do not reveal that the org exists
  return next({ context: { orgId: membership.orgId, role: membership.role } });
});

export const getListing = orgProcedure
  .input(z.object({ id: z.string().uuid() }))
  .handler(async ({ input, context }) => {
    const [row] = await db
      .select()
      .from(listings)
      .where(and(eq(listings.id, input.id), eq(listings.orgId, context.orgId)))   // the org filter
      .limit(1);
    if (!row) throw new ORPCError("NOT_FOUND");               // another org's ID looks like a missing ID
    return row;
  });
```

Row-level security as a backstop (PostgreSQL):

```sql
ALTER TABLE listings ENABLE ROW LEVEL SECURITY;
ALTER TABLE listings FORCE ROW LEVEL SECURITY;      -- applies to the table owner too
CREATE POLICY listings_org_isolation ON listings
  USING (org_id = current_setting('app.current_org_id', true)::uuid)
  WITH CHECK (org_id = current_setting('app.current_org_id', true)::uuid);

-- Per request, inside the transaction that runs the queries (true = local to the transaction):
SELECT set_config('app.current_org_id', $1, true);
```

If the setting is missing, `current_setting(..., true)` returns NULL and the policy matches no rows, so it fails closed. Superusers and roles with `BYPASSRLS` skip policies, so the app must connect as an ordinary role. With connection pools, always set the value per transaction, never per session.

Testing for leaks:

```ts
import { describe, expect, it } from "vitest";

describe("org isolation", () => {
  it("does not return another org's listing", async () => {
    const orgA = await createOrg();                  // test factories (illustrative)
    const orgB = await createOrg();
    const userA = await createMember(orgA, "admin");
    const listingB = await createListing(orgB);

    // callAs: a test helper that runs the procedure with userA's context in orgA.
    await expect(callAs(userA, orgA, getListing, { id: listingB.id })).rejects.toMatchObject({
      code: "NOT_FOUND",
    });
  });
});
```

- Write this test for every procedure that accepts an ID, including updates and deletes, and for list endpoints (no foreign rows in results).
- Add role tests: a plain member calling an admin-only procedure gets `FORBIDDEN`.
- Review checklist: any query without an `org_id` filter needs a written reason.

> **Fill in:** where scoping is enforced in the real code, the role list, and what tests exist.

### Part C: migrating the marketplace page without losing SEO
1. **Inventory URLs:** from the sitemap, analytics (top landing pages) and a crawl of the Rails site.
2. **Keep paths identical where you can;** otherwise build a tested 301 redirect map from old to new paths.
3. **Server-render** the page in TanStack Start (a route loader calling a server function), so crawlers and link previews get full HTML.
4. **Port the head tags:** title, meta description, canonical URL, Open Graph tags, structured data, `hreflang` if the site is multilingual.
5. **Keep the sitemap and robots rules** correct for the new pages.
6. **Check parity:** compare old and new HTML for key elements (title, H1, canonical, number of listings), compare screenshots, and compare performance (LCP, INP, CLS).
7. **Roll out gradually:** route one path or a share of traffic first, watch 404s, crawl errors, search traffic and conversions, and keep a quick switch back to Rails.

> **Fill in:** whether URLs changed, how redirects were handled, how parity was checked, and how the cutover happened.

### Part D: why these tools (as you understood the choice)

| | oRPC | tRPC |
|---|---|---|
| End-to-end types | Yes | Yes |
| Procedure kinds | One kind of procedure | Query, mutation, subscription |
| OpenAPI | First-class: generate a spec, expose REST-style routes | Through third-party add-ons |
| Contract-first | Optional: define the contract, implement it separately | Router-first |
| Ecosystem | Younger and smaller | Older and larger |

| | TanStack Start | Next.js (App Router) |
|---|---|---|
| Routing | TanStack Router: file-based, type-safe params and search params | File-based App Router |
| Server code | Server functions, route loaders, SSR | Server Components, Server Functions, route handlers |
| Build tool | Vite | Next's own pipeline (Turbopack in recent versions) |
| Default model | Client-first app with SSR | Server-first with client components |
| Maturity | Release candidate in 2025; APIs changed during beta | Mature, very large ecosystem |

| | Drizzle | Prisma |
|---|---|---|
| Schema | TypeScript (`pgTable`) | Prisma schema language |
| Queries | SQL-shaped builder plus a relational API | High-level client (`findMany`, `include`) |
| Generated code | None: types are inferred from the schema | A generated client |
| Migrations | drizzle-kit (`generate`, `migrate`, `push`) | Prisma Migrate |

These projects move quickly; check their docs before stating version-specific features.

> **Fill in:** the reasons the team gave for picking oRPC, TanStack Start and Drizzle, and anything you found hard when moving from Rails conventions to this stack.

## Likely interview questions

**Q1. Why migrate to a new platform at all instead of improving the Rails app?**
Give the real drivers as you understood them (for example serving multiple organizations, one TypeScript stack across web, API and mobile, type safety), the alternatives (adding tenancy to Rails with scoping or a gem like acts_as_tenant, refactoring in place), and the mitigations for rewrite risk (incremental migration, sync, page-by-page cutover). Say who made the decision.
> **Fill in:** the real reasons and who decided.

**Q2. What exactly did you contribute, and what did the team do?**
Three pieces with one detail each, then the team's context in a sentence. Never present the whole migration as yours.

**Q3. What is org-scoped access control, and where was it enforced?**
Every request runs in one org context; membership and role are checked once in middleware; every query filters by `org_id`; tests prove cross-org access fails; optionally row-level security in the database. Then name where it lived in your code.
> **Fill in:** the real enforcement points.

**Q4. How do you test that one org cannot see another org's data?**
Two orgs, a user in org A, a resource in org B; call every ID-taking procedure as user A with B's IDs and expect not found; check that lists never include foreign rows; role tests for forbidden actions; optionally run the suite with row-level security on.

**Q5. Which direction did the sync go, and why?**
Answer from reality, then justify: pull gives the consumer control over pace and retries; push lowers latency but needs retries and ordering on the sender; change data capture avoids app code but adds operations work.
> **Fill in:** the real direction and the reason.

**Q6. How do you map IDs and keep the sync idempotent?**
Store the Rails ID as `legacy_id` with a unique index, upsert on it, only overwrite when the incoming version is newer, and advance the cursor only after a page is stored. Replays then change nothing.

**Q7. How do deletes and conflicting updates sync?**
Deletes: soft deletes, tombstones or periodic ID-set comparison. Conflicts: one source of truth per field or per resource during each phase; "newer `updated_at` wins" for one-way syncs; avoid two-way writes to the same data whenever possible.

**Q8. oRPC vs tRPC?**
Both give end-to-end type safety for TypeScript clients. oRPC adds first-class OpenAPI and optional contract-first development, and does not split procedures into queries and mutations; tRPC is older with a larger ecosystem. Pick oRPC when non-TypeScript consumers or published API docs matter.

**Q9. TanStack Start vs Next.js?**
Start: fully type-safe routes and search params, Vite, server functions, a client-first model with SSR; younger, with APIs that changed through its release candidates. Next.js: mature, Server Components first, a huge ecosystem. Give the team's reason if you know it.

**Q10. Drizzle vs Prisma, and how does it compare with ActiveRecord?**
Drizzle keeps you close to SQL with types inferred from a TypeScript schema and no generated client; Prisma offers a higher-level client generated from its own schema language. Compared with ActiveRecord: explicit queries and no callbacks or implicit lazy loading, so tenant scoping and eager loading are your job. See [Drizzle ORM](../12-databases/13-drizzle-orm.md).

**Q11. How did you keep the marketplace page's SEO and URLs intact?**
URL inventory, identical paths or tested 301s, server rendering, ported head tags and structured data, sitemap updates, parity checks, gradual rollout and monitoring of crawl errors and traffic.
> **Fill in:** what you actually did.

**Q12. How do you know the new page has parity with the old one?**
A written checklist of behaviors and content, HTML comparisons of key elements, visual comparisons, the same data on both sides thanks to the sync, performance comparisons, and stakeholder sign-off before switching traffic.

**Q13. What is the strangler fig pattern?**
Put a routing layer in front, rebuild one capability at a time in the new system, switch that path over, and retire the old code once nothing routes to it. It lowers risk compared with a big-bang rewrite because each step is small and reversible.

**Q14. Shared tables with `org_id`, schema per tenant or database per tenant?**
Shared tables are simplest and cheapest but rely on discipline and tests; schema per tenant adds separation and migration overhead; database per tenant gives the strongest isolation at the highest cost. Choose by isolation requirements, tenant count and operational capacity.

**Q15. What are the main risks of a migration like this, and what would you do differently?**
Scope creep, parity gaps, sync bugs, two systems to maintain, and a long tail of rarely used features. Mitigate with small steps, measurable parity checklists, reconciliation, and deciding early what will not be migrated.
> **Fill in:** one thing you would genuinely change.

**Q16. How do you roll back if a migrated page breaks in production?**
Keep the routing switch: point the path back to Rails, which still serves it, while the sync keeps data current. That is why the old page stays deployable until the new one has proven itself.

## Fill-in checklist
- [ ] When you joined the migration, who led it and who decided to do it.
- [ ] Your exact pieces vs the team's.
- [ ] Sync direction, trigger, cursor design, payloads, authentication, delete handling.
- [ ] How the consumer stores progress and handles failures.
- [ ] What the marketplace page is, whether URLs changed, how SEO and parity were handled, how cutover happened.
- [ ] Where org scoping is enforced, which roles exist, whether row-level security is used, which tests exist.
- [ ] The team's reasons for oRPC, TanStack Start and Drizzle.
- [ ] One hard bug or edge case from this work, as a STAR story.
