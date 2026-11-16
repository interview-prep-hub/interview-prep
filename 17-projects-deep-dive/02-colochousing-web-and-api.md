# Colochousing: Rails 7 Web App and REST API

> **Tier:** T1 (know cold) · **Related:** [React on Rails](../05-ruby-on-rails/16-react-on-rails.md), [Background jobs and Sidekiq](../05-ruby-on-rails/09-background-jobs-sidekiq.md), [API development](../05-ruby-on-rails/10-api-development.md), [Rails authentication and authorization](../05-ruby-on-rails/11-authentication-authorization.md), [REST API design](../04-web-fundamentals/04-rest-api-design.md), [Coloc resident app](03-coloc-resident-app.md), [Platform migration](04-multi-tenant-platform-migration.md), [System design from your projects](../15-system-design/03-case-studies-from-your-projects.md)
> **On your resume:** Project Colochousing (Devntech client; Ruby on Rails, React on Rails, React Native (Expo), TypeScript, PostgreSQL): worked on the Rails 7 web app and REST API (React on Rails, ERB, Sidekiq) of a Belgian co-living platform with 16K+ users, 65 houses in 8 cities and 24K+ rental applications.

## Quick revision

| Item | What to remember |
|---|---|
| Where | A Devntech client project, during your Devntech role (Nov 2025 to present) |
| What | The Rails 7 web app and REST API of a Belgian co-living platform |
| Web stack | Rails 7, ERB views, React components mounted with React on Rails, Sidekiq for background jobs, PostgreSQL |
| Likely API consumer | The Coloc resident app (React Native, Expo, TypeScript), if it talks to this API (confirm); see [the app deep-dive](03-coloc-resident-app.md) |
| Platform scale | 16K+ users, 65 houses in 8 cities, 24K+ rental applications: the platform's numbers, not yours |
| Sibling bullets | The resident app and the multi-tenant platform migration have their own deep-dives |

- **Say it like this:** "I work on the Rails 7 web app and REST API of Colochousing, a Belgian co-living platform with 16K+ users, 65 houses in 8 cities and 24K+ rental applications."
- **React on Rails + ERB:** the usual split is server-rendered ERB pages with React components mounted where interactivity pays off, through the `react_component` view helper, which passes Rails data in as JSON props. Optional SSR (`prerender: true`) renders the component on the server, then React hydrates it in the browser.
- **Sidekiq:** jobs live in Redis and run on threads in a separate process with at-least-once delivery, so jobs must be idempotent and should be enqueued after commit.
- **API design for mobile clients (typical; know what the real API does):** token auth (not cookies), versioned routes (`/api/v1`), additive changes only, because old app versions stay installed for weeks after a release.
- **Domain core (typical for co-living):** houses, rooms, rental applications (a status workflow) and residents (leases with dates).
- **Data protection:** a Belgian platform holds personal data of people in the EU, so GDPR applies: minimize, restrict access, filter logs, support export and deletion.
- **i18n:** Belgium has three official languages (Dutch, French, German) and co-living attracts international residents. Know whether the app was multilingual instead of assuming.

> **Fill in:** the parts of the web app and API you personally built or changed (pages, endpoints, jobs), and whether you also worked on admin screens.

## The 30-second pitch

> "On Colochousing, a Belgian co-living platform with 16K+ users, 65 houses in 8 cities and 24K+ rental applications, I work on the Rails 7 web app and its REST API. The web side is ERB with React components mounted through React on Rails, and Sidekiq runs the background work. I also build features for Coloc, the platform's resident mobile app."

> **Fill in:** confirm whether the Coloc app calls this REST API. If it does, say so in the pitch ("the same Rails app serves the API behind the resident app"), because it ties your web, API and mobile work into one story.

> **Fill in:** one more sentence on a feature you shipped end to end (UI, endpoint, job) and why it mattered to residents or the operations team.

## Domain primer

### What co-living is
Co-living is a form of shared housing: each resident rents a private bedroom (sometimes with a private bathroom) in a house with a shared kitchen, living spaces and often services. Typical traits (check which apply to Colochousing):
- **Furnished, all-in rent:** utilities, internet and cleaning of common areas are often bundled into one monthly price.
- **Community:** operators organize events and help house life run smoothly (chores, shared costs, house rules), which is why a resident app exists at all.
- **Turnover:** students, young professionals and newcomers to a city move in and out often, so the application funnel never stops.

### The main entities

| Entity | What it is | Typical data |
|---|---|---|
| City | Where houses are | Name, slug |
| House | One shared property | Address, city, amenities, photos |
| Room | A rentable unit in a house | Size, price, availability date |
| Rental application | Someone applying for a room or a house | Applicant, preferences, move-in date, status, documents |
| Resident | A person living in a house now | Linked to a room by a lease or contract with start and end dates |
| Event, task, bill | House-life features of the resident app | See [the app deep-dive](03-coloc-resident-app.md) |

### The applicant-to-resident funnel
A typical flow: browse houses and rooms, apply, get screened (documents, possibly a call or a visit), get accepted, sign a contract, pay a deposit or first rent, move in, use the resident app, eventually move out. The scale numbers hint at why this workflow matters: 24K+ applications against 65 houses means applications far outnumber rooms, so filtering, status changes and automated emails are core features, not extras. That is a reasonable inference from the numbers, not a fact about the codebase, so confirm it.

> **Fill in:** the real application statuses and who moves an application between them (applicant, house manager, admin, automation).

### Belgium-specific context
- **Languages:** Dutch, French and German are official; Brussels is officially bilingual (French and Dutch), and many co-living residents are international, so English is common.
- **Time zone:** Belgium uses Central European Time (UTC+1, UTC+2 in summer), 3 to 4 hours behind Pakistan, which shapes meeting overlap with the client.
- **GDPR:** covers personal data of people in the EU, including applicants' documents and residents' details.

> **Fill in:** was the app multilingual? If yes, which languages, and how were translations managed (Rails I18n YAML files, a translation service, translated database content)?

## Typical architecture

> A typical setup for a Rails 7 app like this, to adapt to what you actually saw. It is not a description of the real Colochousing codebase.

```text
Browser                                 Coloc app (React Native, Expo)
  |  HTML (ERB) + React islands            |  JSON over HTTPS, token auth
  v                                        v
+------------------------- Rails 7 monolith -------------------------+
|  Web controllers -> ERB views; react_component(...) mounts React   |
|  Api::V1 controllers -> JSON (serializers or Jbuilder)             |
|  Models: City, House, Room, RentalApplication, Lease, User, ...    |
|  Mailers, Active Storage (documents, photos)                       |
+--------------------------------------------------------------------+
     | perform_async after commit             | SQL
     v                                        v
  Redis <---- Sidekiq process(es)          PostgreSQL
              (emails, push notifications, reminders, imports, syncs)
```

- One codebase serves three audiences: public visitors and applicants, residents (through the mobile app's API) and internal staff (admin screens).
- React on Rails components are bundled by a JavaScript bundler (usually Shakapacker, the successor of Webpacker) and mounted into ERB pages.
- Sidekiq runs as its own process next to the web server; both use the same PostgreSQL database, and Sidekiq uses Redis as its queue.

> **Fill in:** how the app is hosted and deployed (platform, Docker or not, CI), and how the API builds JSON (Jbuilder, a serializer gem, plain `as_json`).

## Bullet-by-bullet

### "Worked on the Rails 7 web app and REST API (React on Rails, ERB, Sidekiq) of a Belgian co-living platform with 16K+ users, 65 houses in 8 cities, and 24K+ rental applications."

Split the bullet into the four things interviewers probe:

| Part | What they ask | Where to study |
|---|---|---|
| Rails 7 web app | What you built, how controllers and views are organized, N+1 issues you fixed | [Routing and controllers](../05-ruby-on-rails/02-routing-and-controllers.md), [N+1 queries](../05-ruby-on-rails/05-activerecord-querying-n-plus-1.md) |
| React on Rails + ERB | Why both, how props reach React, SSR and hydration | [React on Rails](../05-ruby-on-rails/16-react-on-rails.md), [views and ERB](../05-ruby-on-rails/07-views-erb-forms-assets.md) |
| REST API | Auth for mobile, versioning, pagination, error format | [API development](../05-ruby-on-rails/10-api-development.md), [REST API design](../04-web-fundamentals/04-rest-api-design.md) |
| Sidekiq | Which jobs and why, retries, idempotency | [Background jobs](../05-ruby-on-rails/09-background-jobs-sidekiq.md) |

How to answer honestly:
- "Worked on" means you contributed to an existing app. Say what existed before you and what you added.
- Give two or three concrete features with the layers you touched (ERB page, React component, API endpoint, Sidekiq job, migration).
- Attribute the numbers to the platform.

> **Fill in:** three features or fixes you shipped in the web app or API, each with the layers involved.

> **Fill in:** one Sidekiq job you wrote or changed, what triggers it, and what makes it safe to retry.

The other two Colochousing bullets have their own files: [the Coloc resident app](03-coloc-resident-app.md) and [the multi-tenant platform migration](04-multi-tenant-platform-migration.md).

## Design deep-dive

> Typical designs to adapt. Where the real app works differently, describe the real one and explain why it was built that way.

### Part A: React on Rails, from controller to hydrated component

```ruby
# app/controllers/houses_controller.rb
class HousesController < ApplicationController
  def show
    @house = House.includes(:rooms).find_by!(slug: params[:slug])
    @room_props = @house.rooms.map do |room|
      { id: room.id, name: room.name, priceCents: room.price_cents, availableFrom: room.available_from }
    end
  end
end
```

```erb
<%# app/views/houses/show.html.erb: server-rendered ERB with one React island %>
<h1><%= @house.name %></h1>
<%= react_component("RoomPicker", props: { houseId: @house.id, rooms: @room_props }, prerender: false) %>
```

```tsx
// Client bundle entry: register every component the views mount by name.
import ReactOnRails from "react-on-rails";
import RoomPicker from "../components/RoomPicker";

ReactOnRails.register({ RoomPicker });
```

How it works:
- `react_component` writes a placeholder element and the props, serialized as JSON, into the HTML. On page load, React on Rails finds the placeholder, looks up `RoomPicker` in its registry and renders it with those props.
- With `prerender: true`, the component is rendered to HTML on the server (through ExecJS by default, or a Node renderer offered by React on Rails Pro), so the markup arrives in the first response and the client bundle hydrates it. The server bundle must not touch `window` or `document` during render or at import time.
- Props are public: everything you pass is visible in the page source, so never pass secrets or other users' data, and keep the payload small.
- Ruby uses snake_case and JavaScript uses camelCase; convert at the boundary, in one place.

Why ERB plus React islands instead of a full SPA:

| ERB-first with React islands | Full SPA on a JSON API |
|---|---|
| Fast to build CRUD and content pages with Rails conventions | Every screen needs an endpoint and client-side state |
| Server-rendered HTML helps SEO and first load | Needs SSR or prerendering for SEO |
| React only where interactivity pays off (filters, pickers, multi-step forms) | One consistent client architecture everywhere |
| One deploy and one session cookie; Rails handles web auth | A separate auth story for the SPA |
| Two UI paradigms; state shared between islands is awkward | One paradigm, but a bigger build and runtime |

> **Fill in:** which screens were React and which were ERB, and why the team drew the line there.

### Part B: the rental application workflow
A natural design question for this domain is "Design the rental application flow." A typical shape:

```ruby
class RentalApplication < ApplicationRecord
  TRANSITIONS = {
    "submitted" => %w[in_review withdrawn],
    "in_review" => %w[accepted rejected withdrawn],
    "accepted"  => %w[contract_signed withdrawn]
  }.freeze

  enum :status, { submitted: "submitted", in_review: "in_review", accepted: "accepted",
                  rejected: "rejected", withdrawn: "withdrawn", contract_signed: "contract_signed" }

  belongs_to :applicant, class_name: "User"
  belongs_to :room, optional: true
  has_many :status_changes, dependent: :destroy

  after_update_commit :notify_applicant, if: :saved_change_to_status?

  def transition_to!(new_status, actor:)
    with_lock do                                       # row lock: two staff clicks cannot race
      unless TRANSITIONS.fetch(status, []).include?(new_status.to_s)
        raise ArgumentError, "cannot go from #{status} to #{new_status}"
      end
      update!(status: new_status)
      status_changes.create!(from: status_before_last_save, to: status, actor: actor)
    end
  end

  private

  def notify_applicant
    ApplicationStatusEmailJob.perform_async(id, status)   # after commit: the job sees the new status
  end
end
```

Points to make:
- Explicit allowed transitions (or a state machine gem) instead of status updates scattered across controllers.
- An audit trail (`status_changes`) answers "who rejected this, and when?"
- Side effects (emails, push notifications) go to Sidekiq after commit, and the job re-reads the record, so a late retry does not send a stale message.
- Indexes for the admin list on `status`, `room_id` and `created_at`; paginate, and use `includes` to avoid N+1 queries across 24K+ rows.
- Room double-booking: if leases have date ranges, PostgreSQL can enforce "no overlapping leases for the same room" with an exclusion constraint:

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;  -- lets a GiST index handle room_id equality
ALTER TABLE leases
  ADD CONSTRAINT leases_no_overlap
  EXCLUDE USING gist (room_id WITH =, daterange(starts_on, ends_on, '[)') WITH &&);
```

> **Fill in:** how applications are really modeled (statuses, who reviews, which emails or notifications fire) and whether anything enforces one resident per room per period.

### Part C: a REST API for a mobile client

A mobile app changes how you design an API. If the Coloc app consumes this API (confirm), these rules apply to it directly.

```ruby
# config/routes.rb
namespace :api do
  namespace :v1 do
    resources :events, only: %i[index show]
    resources :bills, only: %i[index create]
    resource :push_token, only: %i[create destroy]
  end
end
```

```ruby
module Api
  module V1
    class BaseController < ActionController::API
      before_action :authenticate_user!

      rescue_from ActiveRecord::RecordNotFound do
        render json: { error: "not_found" }, status: :not_found
      end

      private

      attr_reader :current_user

      def authenticate_user!
        token = request.headers["Authorization"].to_s.delete_prefix("Bearer ")
        @current_user = ApiToken.authenticate(token)   # illustrative: compares a digest of the token
        render json: { error: "unauthorized" }, status: :unauthorized unless @current_user
      end
    end
  end
end
```

Design rules for an API a mobile app depends on:
- **Versioning:** old app versions stay installed for weeks, and native changes need a store release, so never break `/api/v1`. Add fields freely; remove or rename only in a new version, or after traffic from old app versions has dropped. A minimum-supported-version check (the app sends its version; the API answers "please update") lets you retire old versions deliberately.
- **Auth:** tokens in the `Authorization` header, kept in the device's secure storage; short-lived access tokens with refresh tokens, or long-lived revocable tokens. Common Rails choices: Devise with a JWT extension, Doorkeeper (an OAuth 2 provider), or random tokens (Rails has `has_secure_token`, though storing only a digest of the token is safer).
- **Scoping:** every query starts from `current_user` (`current_user.bills`), never from a raw ID in params.
- **Consistency:** one error shape, pagination on every list (cursor, or page plus limit), ISO 8601 timestamps, integer cents for money.

> **Fill in:** how the mobile API authenticates (library, token lifetime, refresh), whether it is versioned, and how the app handles a forced update.

### Part D: Sidekiq jobs you would expect in a platform like this
Typical jobs for this domain, as a checklist to compare against what you actually saw. Claim only the ones that exist in the real app.

| Typical job | Trigger | What makes it safe to retry |
|---|---|---|
| Application status email | Status change, after commit | Re-read the record; send only if the status still matches the one in the job arguments |
| Push notification fan-out | New event, bill or announcement | A per-recipient "sent" record, so a retry skips residents already notified |
| Rent or task reminders | Recurring schedule (cron-style) | Look up "already reminded for this period" before sending |
| Document processing | Upload completed | Processing writes derived data keyed by the document ID, so a rerun overwrites, not duplicates |
| Data sync to another system | Record change or schedule | Upserts keyed by an external or legacy ID (see [the migration deep-dive](04-multi-tenant-platform-migration.md)) |
| CSV export for staff | Button click | Writes one file per export request ID; the user downloads it when ready |

```ruby
class ApplicationStatusEmailJob
  include Sidekiq::Job
  sidekiq_options queue: :mailers, retry: 5

  def perform(application_id, expected_status)
    application = RentalApplication.find_by(id: application_id)
    return unless application && application.status == expected_status   # stale job: skip quietly

    RentalApplicationMailer.with(application: application).status_changed.deliver_now
  end
end
```

Typical bugs in apps like this, and how you would debug them:
- **Job runs before the data exists:** enqueued inside a transaction, so the worker cannot find the row yet. Symptom: sporadic `RecordNotFound` in Sidekiq. Fix: enqueue after commit.
- **Dates shifting by a day:** move-in dates stored as timestamps and rendered in a different time zone. Fix: `date` columns for calendar dates and an explicit app time zone (`config.time_zone`).
- **N+1 queries on listing pages:** a house page that loads rooms, photos and prices row by row. Find it in the logs or with a tool like Bullet; fix with `includes`.
- **Data leaking through props:** a whole model serialized into `react_component` props, exposing fields the page never needed. Fix: build props explicitly.
- **Old app versions breaking:** an API field renamed in place. Fix: additive changes and versioning (Part C).

### Going deeper
- **GDPR in practice:** encrypt sensitive columns (Rails 7 `encrypts`), filter personal data out of logs (`filter_parameters`), protect documents with authorization checks and expiring URLs, define retention for rejected applications, and support export and deletion requests.
- **Caching:** fragment caching for house and room pages, HTTP caching (`stale?`, ETags) for public API lists.
- **Search:** filtering houses by city, price and availability is a classic indexing exercise; build composite indexes that match the common filter combinations.

## Likely interview questions

**Q1. Walk me through the Colochousing platform and your role.**
Platform first (a Belgian co-living platform: 16K+ users, 65 houses in 8 cities, 24K+ rental applications), then the architecture (a Rails 7 monolith with ERB plus React on Rails, Sidekiq, PostgreSQL and a REST API, used by the resident app if that is its consumer), then your work as two or three concrete features.
> **Fill in:** your concrete features.

**Q2. Why React on Rails plus ERB instead of a separate React SPA?**
ERB keeps conventional pages fast to build and server-rendered; React is used where interaction justifies it; one deploy and one session. The cost is two UI paradigms and awkward state sharing between islands. Then say which screens used React in your app.

**Q3. How do props get from Rails into React, and how does SSR work?**
The `react_component` helper serializes props to JSON in the page; the client-side registry mounts the named component with them. With `prerender: true`, the server renders HTML first (ExecJS or a Node renderer) and the client hydrates it. Props are visible in the page source.
*Follow-up:* What breaks SSR? Code that touches `window`, `document` or `localStorage` during render or at import time; guard it or move it into effects.

**Q4. Which Sidekiq jobs did you work with, and why were they jobs?**
Anything slow, unreliable or not needed for the response: emails, push notifications, reminders, document processing, syncs. Explain retries, idempotency and enqueueing after commit.
> **Fill in:** the actual jobs you wrote or changed.

**Q5. How does a mobile client authenticate against the API?**
Describe the real mechanism, then justify it: bearer tokens instead of cookies, secure storage on the device, expiry and refresh, revocation on logout or password change, and server-side scoping by `current_user`.
> **Fill in:** the real library and token lifecycle.

**Q6. How do you change the API without breaking app versions already installed?**
Additive changes within a version, a new version for breaking changes, a minimum-version check with a forced-update screen, and monitoring which app versions still call the API before deleting old behavior.

**Q7. Was the app multilingual? How would you implement it?**
Answer the first part only from what you saw; never guess.
> **Fill in:** the real answer.

If it was not, describe how you would do it: Rails I18n with `t(".key")` and locale YAML files, the locale taken from the user's setting or `Accept-Language`, translated database content (a translations table or a JSONB column per field), and translated emails and push messages.

**Q8. How would you model houses, rooms, applications and residents?**
A city has many houses, a house has many rooms, an application belongs to an applicant and optionally a room, and a lease links a resident to a room for a date range. Mention indexes, the status workflow and an exclusion constraint against overlapping leases.

**Q9. How do you keep an admin list of 24K+ applications fast?**
Pagination, indexes that match the filters (`status`, `created_at`, `room_id`), `includes` against N+1, careful counting (counter caches or estimates), and background jobs for CSV exports.

**Q10. How do you handle applicants' documents securely?**
Private storage (Active Storage on a private bucket), authorization on every download, short-lived signed URLs, file type and size checks on upload, encryption at rest and a retention policy.
> **Fill in:** how documents are actually handled, if you worked in that area.

**Q11. How would you test a feature here end to end?**
Model specs for workflow rules, request specs for API endpoints (auth, scoping, error shapes), job specs (including running a job twice), and a system spec for the critical page. See [testing with RSpec](../05-ruby-on-rails/13-testing-rspec.md).
> **Fill in:** the testing setup the project really has.

**Q12. What does GDPR change for a platform like this?**
A lawful basis and consent where needed, data minimization, access control, retention limits, the rights of access, export and deletion, breach handling, and care with personal data in logs and third-party tools.

**Q13. What was the hardest thing you did on the web app or API?**
Use a STAR story from [the story bank](../18-behavioral-and-hr/02-star-stories-bank.md).
> **Fill in:** the story.

**Q14. Rails 7 includes Hotwire by default. Did this app use it?**
Rails 7 makes Turbo and Stimulus the default for new apps, but an app can use React for interactivity instead. Answer from what you saw, and mention your Hotwire training at Visnext if they want depth.
> **Fill in:** whether Turbo or Stimulus appear anywhere in the app.

## Fill-in checklist
- [ ] The features, endpoints and jobs you personally built or changed.
- [ ] Whether you are still on the project (use the right tense in the pitch).
- [ ] Which screens are React and which are ERB; whether SSR (`prerender`) is used.
- [ ] The real Sidekiq jobs and what makes them idempotent.
- [ ] How the mobile API authenticates, whether it is versioned, and how forced updates work.
- [ ] Whether the app is multilingual and how translations are managed.
- [ ] The real application statuses and who moves them.
- [ ] How documents and personal data are protected.
- [ ] The testing setup and deployment process.
- [ ] Where the scale numbers come from (client-provided, public website, admin dashboard).
- [ ] One STAR story from this project.
