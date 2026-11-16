# Rails Architecture Patterns

> **Tier:** T1 (know cold) · **Related:** [MVC and layered architecture](../13-architecture-and-practices/01-mvc-and-layered-architecture.md), [Clean architecture](../13-architecture-and-practices/02-clean-architecture.md), [Design patterns](../01-cs-fundamentals/04-design-patterns.md), [SOLID](../01-cs-fundamentals/03-solid-and-design-principles.md), [ActiveRecord basics](03-activerecord-basics.md), [Legacy codebases and upgrades](17-legacy-codebases-and-upgrades.md), [Authentication and authorization](11-authentication-authorization.md)
> **On your resume:** working across legacy Rails monoliths at Devntech (CrystalCommerce, Colochousing), MVC from the Visnext training, and Clean Architecture in your skills list.

## Quick revision
- **Fat model, skinny controller:** controllers translate HTTP; models hold domain logic. It breaks down when models become god objects whose callbacks send emails and call APIs.
- **Concern** (`ActiveSupport::Concern`): a module with `included do ... end` and `class_methods do ... end`. Good for a cohesive trait shared by several models (Archivable); bad when it just splits one fat model into files.
- **Service object:** one use case per class (`RentalApplications::Approve.call(...)`), one public method, a result object for expected failures, DB writes in a transaction, emails and jobs after commit.
- **Form object:** `ActiveModel::Model` + `ActiveModel::Attributes` for a form that spans several models or has validations that belong to the use case, not the table.
- **Query object:** a class that builds and returns a relation from many filters, so it still chains, paginates and preloads.
- **Presenter / decorator:** wraps a model with display logic (`price`, `status_badge`); Draper or a `SimpleDelegator` PORO.
- **Value object:** immutable, compared by value (`Money`, `DateRange`); Ruby 3.2's `Data.define` is perfect for them.
- **Policy object:** a business rule answered in one place (eligibility, refundability), separate from authorization policies like Pundit's.
- **Interactor / command:** an object per step with a shared context and an organizer chaining steps (interactor gem); explicit service objects are usually simpler.
- **State machine:** AASM gives states, events, guards, callbacks, `may_approve?`, `approve!` and scopes; a plain enum plus a transitions table works for small lifecycles. Lock the row when transitions can race.
- **Events:** `ActiveSupport::Notifications.instrument` / `subscribe` (synchronous, in-process), `after_commit` hooks, Wisper. Decoupling costs traceability.
- **Large monolith:** namespace by domain, then engines or Packwerk packages to enforce boundaries before reaching for microservices.
- Add a pattern when it removes real pain (hard to test, read or change); never as decoration around a one-line `create`.

## 1. Fat model, skinny controller, and its limits

The Rails default is good: controllers parse params, call the domain and pick a response; models own validations, associations, scopes and business methods. The trouble starts when everything lands in the model:

```ruby
class RentalApplication < ApplicationRecord
  after_update :create_lease, if: -> { saved_change_to_status?(to: "approved") }
  after_update :email_applicant, :notify_house_manager, :sync_crm, :mark_room_taken
  # ...800 more lines: scoring, pricing, PDF generation, reporting
end
```

Symptoms: a 1,000-line model, callbacks firing during imports, console fixes and tests that only wanted a record, side effects nobody can find from the controller, a slow suite because every `create` sends emails and calls APIs. The fix is not "move everything to services" either; it is to keep data rules in the model and move orchestration of side effects into explicit objects.

| Keep in the model | Move out |
|---|---|
| Validations, associations, scopes | Multi-model workflows (approve, checkout, pay rent) |
| Derived values and normalization (`normalizes`, `before_validation`) | Emails, push notifications, third-party API calls |
| Small domain methods (`overdue?`, `total_cents`) | Long processes, imports, report generation |
| Data-integrity callbacks | Anything that needs request context (current user, params) |

## 2. Concerns

```ruby
# app/models/concerns/archivable.rb
module Archivable
  extend ActiveSupport::Concern

  included do                                  # runs in the including class's context
    scope :archived, -> { where.not(archived_at: nil) }
    scope :active,   -> { where(archived_at: nil) }
  end

  class_methods do
    def archive_all!
      update_all(archived_at: Time.current, updated_at: Time.current)
    end
  end

  def archive!
    update!(archived_at: Time.current)
  end

  def archived?
    archived_at.present?
  end
end

class Listing < ApplicationRecord
  include Archivable
end
```

`ActiveSupport::Concern` adds the `included` and `class_methods` blocks and resolves dependencies when one concern includes another. Controller concerns work the same way (an `Authentication` or `StoreScoped` module with before_actions).

| Concerns help when | Concerns hurt when |
|---|---|
| The trait is cohesive and named after a capability (Archivable, Sluggable, Taggable) | A god model is split into `User::Billing`, `User::Notifications`... that all read each other's state |
| Several models share it | Only one model includes it, so it just hides code in another file |
| It has few dependencies on the host class | It assumes columns, callbacks or methods the host must happen to have |

Some well-known Rails codebases use many small concerns as roles inside models very successfully; the problem is not concerns but incoherent ones.

## 3. Service objects

A service object represents one use case, named as a verb, with one public entry point:

```ruby
# app/services/rental_applications/approve.rb
module RentalApplications
  class Approve
    Error = Class.new(StandardError)
    Result = Data.define(:success, :application, :error) do   # Ruby 3.2+; a Struct works on older Rubies
      def success? = success
    end

    def self.call(**kwargs)
      new(**kwargs).call
    end

    def initialize(application:, approved_by:)
      @application = application
      @approved_by = approved_by
    end

    def call
      lease = ActiveRecord::Base.transaction do                     # returns the block's last value
        room = @application.room.lock!                              # SELECT ... FOR UPDATE
        raise Error, "Room is no longer available" unless room.available?
        @application.approved_by = @approved_by
        @application.approve!                                       # state machine event (section 10)
        room.update!(available: false)
        Lease.create!(resident: @application.applicant, room: room, starts_on: @application.move_in_on)
      end
      LeasePdfJob.perform_later(lease.id)                           # side effect after the commit
      Result.new(success: true, application: @application, error: nil)
    rescue Error, ActiveRecord::RecordInvalid, AASM::InvalidTransition => e
      Result.new(success: false, application: @application, error: e.message)
    end
  end
end

# Controller
result = RentalApplications::Approve.call(application: @application, approved_by: current_user)
if result.success?
  redirect_to result.application, notice: "Approved."
else
  redirect_to result.application, alert: result.error
end
```

Conventions worth stating in an interview:
- **Location and naming:** `app/services/<domain>/<verb>.rb` (autoloaded like anything under `app/`), `RentalApplications::Approve` or `ApproveRentalApplication`; pick one style per codebase.
- **One public method** (`call`), keyword arguments, records or IDs in, a result out.
- **Expected failures** (validation, unavailable room, declined card) return a failed result; **unexpected ones** (bugs, timeouts you cannot handle) raise and reach the error tracker. Libraries like dry-monads formalize this with `Success`/`Failure`.
- **Transactions** wrap only the database writes. HTTP calls stay outside the transaction, and emails or jobs go after it. If a caller wraps your service in an outer transaction, "after the block" is no longer after commit; Rails 7.2+ can defer job enqueueing to after commit, but the default changed across versions, so check yours. Avoid `return` inside a transaction block (its behavior changed across Rails versions); raise instead, as above.
- **Small and composable:** a service may call other services; a 300-line service is a fat model in a new folder.

## 4. Form objects

When one form creates several records, or validations belong to the use case (terms accepted, password confirmation, invite code), a form object keeps that logic out of the models:

```ruby
# app/forms/house_signup_form.rb: creates a user, an organization and an owner membership
class HouseSignupForm
  include ActiveModel::Model
  include ActiveModel::Attributes

  attribute :name, :string
  attribute :email, :string
  attribute :password, :string
  attribute :house_name, :string
  attribute :terms, :boolean

  validates :name, :email, :house_name, presence: true
  validates :password, length: { minimum: 12 }
  validates :terms, acceptance: true

  attr_reader :user

  def save
    return false unless valid?

    ActiveRecord::Base.transaction do
      @user = User.create!(name: name, email: email, password: password)
      organization = Organization.create!(name: house_name)
      Membership.create!(user: @user, organization: organization, role: :owner)
    end
    true
  rescue ActiveRecord::RecordInvalid => e
    errors.add(:base, e.record.errors.full_messages.to_sentence)
    false
  end
end
```

```ruby
def create
  @form = HouseSignupForm.new(params.expect(house_signup_form: %i[name email password house_name terms]))  # Rails 8
  if @form.save
    redirect_to dashboard_path
  else
    render :new, status: :unprocessable_entity
  end
end
```

The view uses `form_with model: @form, url: signups_path`; params arrive under `house_signup_form` because `ActiveModel::Model` provides `model_name`. Form objects are often cleaner than `accepts_nested_attributes_for` chains and let admin and self-service flows validate differently.

## 5. Query objects

```ruby
# app/queries/listings/search_query.rb
module Listings
  class SearchQuery
    SORTS = { "newest" => { created_at: :desc }, "price_asc" => { price_cents: :asc } }.freeze

    def initialize(relation = Listing.all)
      @relation = relation
    end

    def call(params)
      scope = @relation
      scope = scope.where(condition: params[:condition]) if params[:condition].present?
      scope = scope.where(price_cents: ..params[:max_price].to_i) if params[:max_price].present?
      scope = scope.where("listings.title ILIKE ?", "%#{Listing.sanitize_sql_like(params[:q])}%") if params[:q].present?
      scope.order(SORTS.fetch(params[:sort], SORTS["newest"]))
    end
  end
end

# Still a relation: chain, preload and paginate as usual
Listings::SearchQuery.new(current_store.listings).call(params).includes(:product).page(params[:page])
```

Use scopes for small reusable filters (`Listing.active`); move to a query object when filtering combines many params, joins or raw SQL, or when the same search serves the web page, the API and an export. Taking the starting relation as an argument keeps tenant scoping in the caller's hands.

## 6. Presenters and decorators

```ruby
# Plain Ruby presenter
class ListingPresenter < SimpleDelegator               # unknown methods go to the listing
  include ActionView::Helpers::NumberHelper

  def price = number_to_currency(price_cents / 100.0)
  def status_label = archived? ? "Archived" : "Live"
end

@listing = ListingPresenter.new(Listing.find(params[:id]))   # view: @listing.title, @listing.price

# Draper gem
class ListingDecorator < Draper::Decorator
  delegate_all
  def price = h.number_to_currency(object.price_cents / 100.0)   # h = view helpers
end
@listing = Listing.find(params[:id]).decorate
```

| | Helper | Decorator / presenter | ViewComponent |
|---|---|---|---|
| Shape | Global functions available in all views | Object wrapping one model (or a page's data) | Ruby class + template per UI piece |
| Good for | Small generic formatting | Model-specific display logic | Reusable UI with markup and logic |

Pass the real model (not the wrapper) to `form_with` and route helpers if the wrapper confuses polymorphic routing.

## 7. Value objects

A value object has no identity: two `Money(1250, "EUR")` are equal, and it never changes after creation.

```ruby
Money = Data.define(:cents, :currency) do                      # Ruby 3.2+: immutable, value equality
  def +(other)
    raise ArgumentError, "currency mismatch" unless currency == other.currency
    with(cents: cents + other.cents)
  end

  def to_s = format("%.2f %s", cents / 100.0, currency)
end

rent = Money.new(cents: 65_000, currency: "EUR")
rent + Money.new(cents: 2_500, currency: "EUR")      # => #<data Money cents=67500, currency="EUR">
rent == Money.new(cents: 65_000, currency: "EUR")    # => true

class RentPayment < ApplicationRecord
  def amount = Money.new(cents: amount_cents, currency: currency)
end
```

They fix "primitive obsession": currency mismatches, rounding and formatting rules live in one class instead of every caller. Alternatives: the money-rails gem (`monetize :price_cents`), Active Record's older `composed_of`, or a custom attribute type.

## 8. Policy objects

Not every policy is about permissions. A business-rule policy answers one question with reasons, reusable by the controller, the API and a background job:

```ruby
class RentalApplications::EligibilityPolicy
  def initialize(application)
    @application = application
  end

  def eligible? = reasons.empty?

  def reasons
    [].tap do |r|
      r << "Room is not available" unless @application.room.available?
      r << "Move-in date is in the past" if @application.move_in_on.past?
      r << "Applicant already has an active lease" if @application.applicant.leases.active.exists?
    end
  end
end
```

Authorization policies (Pundit, CanCanCan) are covered in [authentication and authorization](11-authentication-authorization.md).

## 9. Interactors and the command pattern

A **command** wraps a request (inputs plus execution) in an object, so it can be queued, logged, retried or composed; an Active Job class is a command. The interactor gem formalizes multi-step commands:

```ruby
class ChargeRent
  include Interactor

  def call
    charge = PaymentGateway.charge(context.payment.amount_cents, idempotency_key: context.payment.idempotency_key)
    context.fail!(error: charge.error_message) unless charge.success?   # stops the organizer
    context.charge_id = charge.id
  end
end

class PayRent
  include Interactor::Organizer
  organize ValidateRentPayment, ChargeRent, MarkRentPaid, NotifyHouse
end

result = PayRent.call(payment: payment)
result.success?   # false if any step called context.fail!
```

The shared `context` is convenient but untyped: any step can read or overwrite any key, which makes flows harder to trace. Many teams prefer explicit service objects that call each other with clear arguments.

## 10. State machines: AASM and enum-based alternatives

Lifecycles such as orders (pending, paid, shipped, refunded) or rental applications (submitted, under review, approved, rejected, withdrawn) need rules about which transitions are legal.

```ruby
class RentalApplication < ApplicationRecord
  include AASM

  aasm column: :status, whiny_persistence: true do   # string column; raise if the save fails
    state :submitted, initial: true
    state :under_review, :approved, :rejected, :withdrawn

    event :start_review do
      transitions from: :submitted, to: :under_review
    end

    event :approve, after_commit: :notify_applicant do
      transitions from: :under_review, to: :approved, guard: :room_available?
    end

    event :reject do
      transitions from: %i[submitted under_review], to: :rejected
    end

    event :withdraw do
      transitions from: %i[submitted under_review], to: :withdrawn
    end
  end

  private

  def room_available? = room.available?
  def notify_applicant = ApplicantMailer.with(application: self).decision.deliver_later
end

app.may_approve?        # => false while still submitted
app.start_review!       # persists; an illegal event raises AASM::InvalidTransition
app.approve!            # checks the guard; after_commit runs after the transaction commits
RentalApplication.under_review   # a scope per state
```

When events fire inside an outer transaction (like the service in section 3), AASM's README recommends the after_commit_everywhere gem so `after_commit` waits for the real commit.

The plain-Rails alternative:

```ruby
class RentalApplication < ApplicationRecord
  enum :status, { submitted: 0, under_review: 1, approved: 2, rejected: 3, withdrawn: 4 }

  TRANSITIONS = {
    "submitted"    => %w[under_review rejected withdrawn],
    "under_review" => %w[approved rejected withdrawn]
  }.freeze
  InvalidTransition = Class.new(StandardError)

  def transition_to!(new_status)
    raise InvalidTransition, "#{status} -> #{new_status}" unless TRANSITIONS.fetch(status, []).include?(new_status.to_s)
    update!(status: new_status)
  end
end
```

| | AASM | Enum + transitions table |
|---|---|---|
| Rules | DSL with events, guards and callbacks | A hash and one method |
| Extras | `may_<event>?`, per-state scopes, whiny transitions, visual docs tools | Enum scopes and predicates |
| Risk | Callback-heavy events become the new god model | Enum's `approved!` jumps from any state unless everyone uses `transition_to!` |
| Fits | Many states and events, several guards | Four or five states and simple rules |

Either way: lock the row (`with_lock`) when two admins can act at once, keep side effects after commit, and consider an audit table (from, to, who, when) for disputed decisions.

## 11. Event-driven decoupling

```ruby
# Publish where the fact happens
ActiveSupport::Notifications.instrument("rent_payment.paid", payment_id: payment.id)

# Subscribe once at boot (config/initializers/subscribers.rb)
ActiveSupport::Notifications.subscribe("rent_payment.paid") do |event|      # an ActiveSupport::Notifications::Event
  RentReceiptJob.perform_later(event.payload[:payment_id])
end
```

- `ActiveSupport::Notifications` is synchronous and in-process: subscribers run in the publisher's thread, and an exception in one surfaces in the publisher's code path. Keep subscribers tiny and hand slow work to jobs.
- `after_commit` hooks (`after_create_commit :publish_created`) are the simplest event source for "this row now exists for real".
- **Wisper** adds publisher objects (`include Wisper::Publisher`, `broadcast(:rent_paid, payment.id)`) and listeners subscribed per call or globally; companion gems run listeners asynchronously.
- Rails 8.1 added a structured event reporter (`Rails.event`), aimed mostly at observability; check the release notes before using it for domain events.
- Trade-off: publishers no longer know who reacts, which decouples code and also hides the flow. Name events in the past tense, pass IDs, make subscribers idempotent, and document who listens.

## 12. Organizing a large monolith

1. **Namespace by domain:** `app/models/billing/invoice.rb` defines `Billing::Invoice`; matching folders for services, jobs and controllers make ownership obvious.
2. **Engines:** a mountable engine (`isolate_namespace Billing`) gets its own models, routes and migrations inside the same app. Boundaries are clear but enforced only by convention, and engines add setup overhead (see [Rails fundamentals](01-rails-fundamentals.md)).
3. **Packwerk (Shopify):** folders become packages with a `package.yml`; `bin/packwerk check` reports references to constants in packages you did not declare as dependencies, and existing violations are recorded in a todo file so you can pay them down gradually. Privacy checking moved to an extension in later versions, so check the docs.

```yaml
# packs/billing/package.yml
enforce_dependencies: true
dependencies:
  - packs/accounts
```

A modular monolith with enforced boundaries is usually the step before (and often instead of) microservices; migration strategies such as the strangler fig are in [platform migration strategies](../13-architecture-and-practices/04-platform-migration-strategies.md).

## 13. When NOT to add a pattern

| Symptom | Reach for |
|---|---|
| Controller action longer than a screen, touching several models | Service object |
| One form, several models or use-case-only validations | Form object |
| Index action assembling ten optional filters | Query object |
| Views full of formatting conditionals | Presenter, decorator or ViewComponent |
| Amounts and currencies passed around as loose integers and strings | Value object |
| Status columns changed from five places with ad-hoc checks | State machine |
| One model reacting to another's changes in callbacks | Explicit service call or event |

Do not add one when:
- The action is plain CRUD: `Listing.create(listing_params)` behind a one-line service adds indirection, not clarity.
- It fights the framework: a repository layer over ActiveRecord duplicates what scopes and models already do.
- The codebase already has a convention. Consistency beats your favorite pattern; propose changes, do not sprinkle a third style.
- You have one example. Extract on the second or third repetition, when the shape is known.

## Common pitfalls
- Callbacks with side effects (emails, API calls, jobs) in `after_save`, running inside the transaction and during every import.
- Concerns used by a single model, or concerns that depend on each other's instance variables.
- Service objects that are just procedural scripts of 300 lines, or that return `true`/`false`/`nil` inconsistently.
- HTTP calls inside `ActiveRecord::Base.transaction`, holding locks and connections while waiting on the network.
- Form objects that skip model validations entirely (`save(validate: false)`), so rules drift apart.
- Query objects that call `to_a` and return arrays, breaking chaining and pagination.
- State changes made with `update_column(:status, ...)` or enum bang methods, bypassing transition rules.
- Event subscribers that do slow work synchronously or are not idempotent.
- Inventing a new pattern per developer, leaving five styles in one app.

## How to talk about this in an interview
- Show judgment rather than pattern-collecting: "I start with Rails defaults and extract when an action or model gets hard to test or change, following the codebase's existing conventions."
- Legacy monoliths at Devntech are where these choices matter: describe reading a model with many callbacks, finding where side effects really happen, and moving one into an explicit service with tests first (link to [legacy codebases](17-legacy-codebases-and-upgrades.md)).
- Lifecycles map naturally to your projects: rental applications in Colochousing and orders in CrystalCommerce. Present state-machine designs as how you would model them unless you actually built them.
- Connect to Clean Architecture from your skills: services and form objects are a pragmatic Rails version of use cases at the boundary, without fighting ActiveRecord.

> **Fill in:** which patterns the Colochousing and CrystalCommerce codebases actually use (service objects, concerns, form objects, a state machine gem) and their naming conventions.

> **Fill in:** one refactor you did or proposed: what was hard before, which pattern you used, and how you kept behavior the same.

## Interview questions

**Q1. What does "fat model, skinny controller" mean, and where does it break down?**
Controllers should only handle HTTP (params, auth, response) while models hold domain logic. It breaks down when models grow into god objects whose callbacks orchestrate emails, APIs and other models, which makes them slow to test and full of hidden side effects.
*Follow-up:* What stays in the model? Validations, associations, scopes, derived values and small domain methods.

**Q2. What is a concern, and when is it a good or bad idea?**
A module extended with `ActiveSupport::Concern`, offering `included` and `class_methods` blocks, mixed into models or controllers. Good for a cohesive capability shared by several classes (Archivable); bad when it only splits one large model into files that still depend on each other.
*Follow-up:* How does `included` differ from a plain module? The block runs in the host class, so you can declare scopes, callbacks and associations there.

**Q3. Concerns, service objects or fat models: how do you decide?**
Model methods for logic about one record's own data; a concern when several models share a coherent trait; a service object for a use case that coordinates several models or side effects (approve an application, pay rent). The deciding questions: who owns this behavior, how many objects does it touch, and can I test it without triggering unrelated side effects?
*Follow-up:* A smell that a concern should be a service? It touches other models, sends emails or needs the current user.

**Q4. What conventions do you follow for service objects?**
One use case per class named as a verb, one public `call`, keyword arguments, a result object for expected failures and exceptions for unexpected ones, database writes in a transaction with external calls and jobs outside it, and a domain folder under `app/services`.
*Follow-up:* Why a class method `.call`? It hides instantiation and keeps call sites short while instances keep internal state private.

**Q5. How should a service object report failure?**
Expected business failures (invalid input, unavailable room, declined card) as a failed result object carrying errors, so callers branch without rescuing; unexpected failures by raising, so they reach the error tracker. Be consistent across the codebase.
*Follow-up:* Libraries for this? dry-monads (`Success`/`Failure`), or a small Struct or `Data` result.

**Q6. Where do transactions and side effects go in a service?**
Wrap only the related database writes in `ActiveRecord::Base.transaction`, lock rows that can race, keep HTTP calls outside, and enqueue emails and jobs after the transaction commits. Raise inside the block to roll back.
*Follow-up:* What if the service is called inside another transaction? "After the block" is no longer after commit; use after-commit hooks or transaction-aware enqueueing.

**Q7. What is a form object, and when do you use one?**
A class including `ActiveModel::Model` (and usually `ActiveModel::Attributes`) that works with `form_with` and validations but is not a table. Use it when one form saves several models, or when validations belong to the use case (terms, invite codes, admin-only rules) rather than to the model.
*Follow-up:* Alternative? `accepts_nested_attributes_for`, which gets awkward beyond simple parent-child forms.

**Q8. What is a query object, and how does it differ from a scope?**
A class that encapsulates a complex query (many filters, joins, raw SQL) and returns a relation. Scopes suit small reusable filters; query objects suit whole searches reused by pages, APIs and exports.
*Follow-up:* Why return a relation? Callers can still chain, preload, paginate and scope by tenant.

**Q9. Helper, presenter or decorator?**
Helpers are global view functions for small formatting; a decorator or presenter wraps a model (or a page's data) and adds display methods like `price` or `status_label`, keeping both models and views clean. ViewComponent fits reusable UI pieces with their own markup.
*Follow-up:* Draper vs a PORO? Draper adds `delegate_all`, `decorates_association` and helper access; a `SimpleDelegator` PORO has no dependency.

**Q10. What is a value object? Give a Rails example.**
An immutable object compared by its values rather than identity, such as `Money(cents, currency)` or `DateRange`. It centralizes rules like currency mismatch and rounding; in Ruby 3.2+ `Data.define` builds one in a line.
*Follow-up:* How do you store it? As columns (`amount_cents`, `currency`) with a reader building the object, money-rails, or a custom attribute type.

**Q11. How would you model a rental application or order lifecycle?**
A status column with explicit states and allowed transitions, enforced in one place (AASM events with guards, or an enum plus a transitions map), transitions run in a transaction with a row lock, side effects after commit, and optionally an audit table of transitions.
*Follow-up:* How do you stop two admins approving at once? Lock the row (`with_lock`) and re-check the state inside the lock.

**Q12. AASM or a plain enum?**
AASM for many states and events with guards and callbacks, plus `may_<event>?` and scopes; a plain enum with a transitions table for a handful of states. Watch AASM callbacks turning into hidden side effects, and enum bang methods skipping transition rules.
*Follow-up:* AASM `after_commit` inside an outer transaction? Use after_commit_everywhere so it waits for the real commit.

**Q13. Callbacks or explicit service calls for side effects?**
Callbacks for data integrity on the record itself (normalizing, defaults, derived columns); explicit calls from a service or controller for side effects such as emails, jobs and API calls. Explicit calls are visible, testable and do not fire during imports or console fixes.
*Follow-up:* If you must use a callback for a side effect? `after_commit`, so it never runs for rolled-back data.

**Q14. How can you decouple parts of a monolith without microservices?**
In-process events (`ActiveSupport::Notifications`, Wisper) or `after_commit` hooks that enqueue jobs, plus enforced module boundaries (namespaces, engines, Packwerk). Keep events past-tense with IDs in the payload and idempotent subscribers.
*Follow-up:* Cost of events? Flow becomes harder to trace, so document publishers and listeners.

**Q15. How would you organize a large Rails monolith?**
Namespace models, services and controllers by domain; extract engines or Packwerk packages with declared dependencies; record existing violations and reduce them over time; extract services only for clear scaling or ownership reasons.
*Follow-up:* What does Packwerk check? References to constants in packages you have not declared as dependencies.

**Q16. When would you avoid adding a pattern?**
Plain CRUD, one-off logic, when it duplicates what Rails already gives (repositories over ActiveRecord), or when it would introduce a second style into a codebase with an established convention. Extract when pain is real and repeated.
*Follow-up:* How do you introduce a new pattern on a team? Propose it with one refactored example and tests, agree on conventions, then apply it gradually.

## Further reading
- [ActiveSupport::Concern](https://api.rubyonrails.org/classes/ActiveSupport/Concern.html)
- [ActiveModel::Model](https://api.rubyonrails.org/classes/ActiveModel/Model.html)
- [ActiveSupport::Notifications](https://api.rubyonrails.org/classes/ActiveSupport/Notifications.html)
- [Data (Ruby)](https://docs.ruby-lang.org/en/3.3/Data.html)
