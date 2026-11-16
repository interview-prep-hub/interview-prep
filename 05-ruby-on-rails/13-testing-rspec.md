# Testing Rails with RSpec

> **Tier:** T1 (know cold) · **Related:** [Testing fundamentals](../13-architecture-and-practices/09-testing-fundamentals.md), [Background jobs](09-background-jobs-sidekiq.md), [API development](10-api-development.md), [Authentication and authorization](11-authentication-authorization.md), [Third-party API integration](../13-architecture-and-practices/05-third-party-api-integration.md), [Legacy codebases and upgrades](17-legacy-codebases-and-upgrades.md), [React testing](../06-react/10-testing.md)
> **On your resume:** changes to legacy Rails monoliths at Devntech (where tests are the safety net), the PriceCharting integration in CrystalCommerce (stubbing external APIs), and the Colochousing Rails 7 app and API.

## Quick revision
- Test behavior through public interfaces, mostly at the model/service and request level, with a few system specs for critical journeys.
- `describe` names the thing, `context` the situation ("when overdue"), `it` one behavior. `subject` is the object under test; `described_class` is the class in `describe`.
- `let` is lazy and memoized per example; `let!` runs before each example. A lazy `let` that is never referenced never creates its record.
- `before` runs per example inside the test transaction; `before(:all)` runs once outside it, so its data leaks into other tests.
- Shared examples (`shared_examples` + `it_behaves_like`) reuse behavior specs; shared contexts reuse setup.
- FactoryBot: factories, traits, sequences, associations, transient attributes. `build` (memory), `create` (saved), `build_stubbed` (fake ID, cannot touch the DB), `attributes_for` (hash).
- Fixtures: YAML loaded once and shared (fast, Rails default with Minitest); factories: data per test (explicit, slower when overused).
- Request specs (full stack through routing and middleware) replaced controller specs as the recommendation since Rails 5 and RSpec 3.5.
- System specs drive a browser with Capybara: `rack_test` (fast, no JS) or headless Chrome via Selenium or Cuprite for JS. Capybara matchers wait; `sleep` does not belong in tests.
- **Stub** = canned answer (`allow ... and_return`); **mock** = expectation set first (`expect ... to receive`); **spy** = assert afterwards (`have_received`). Use verified doubles (`instance_double`) so fakes match the real API.
- External HTTP: WebMock blocks real requests and stubs responses (including 429s and timeouts); VCR records real responses into cassettes and replays them; filter secrets with `filter_sensitive_data`.
- `travel_to`, `freeze_time` and `travel` (ActiveSupport::Testing::TimeHelpers) control time; never depend on today's date.
- `use_transactional_fixtures = true` wraps each example in a rolled-back transaction; since Rails 5.1 system tests share the connection, so DatabaseCleaner is mostly legacy.
- Speed: fewer `create`s, `build_stubbed`, trimmed `let!`, test-prof (`let_it_be`), `rspec --profile`, parallel_tests.
- SimpleCov measures coverage (start it before the app loads); coverage shows what is untested, not what is tested well.
- Minitest: Rails default, plain Ruby classes, `assert_*`, fixtures, built-in parallelization.
- Legacy code: write characterization tests that pin current behavior before refactoring.

## 1. Why test, and what to test

Tests let you change code without fear. In a legacy monolith they are also the best documentation of what the code really does.

| Level | What it covers | How many |
|---|---|---|
| Model / service / unit | Business rules, edge cases, calculations, state transitions | Many: fast and precise |
| Request (integration) | Routing, auth, params, status codes, JSON shape, authorization | One or more per endpoint |
| System (end-to-end) | Critical user journeys in a real browser | Few: slow and more brittle |

Test what could break and would hurt: business rules, edge cases (rounding, empty lists, time zones), authorization and tenant isolation, failure paths of external calls. Do not test Rails itself (that `has_many` works) or private methods directly; test them through the public behavior.

```ruby
# spec/rails_helper.rb (excerpt)
RSpec.configure do |config|
  config.use_transactional_fixtures = true                        # roll back after each example
  config.include FactoryBot::Syntax::Methods                       # create(:user) instead of FactoryBot.create
  config.include ActiveSupport::Testing::TimeHelpers               # travel_to, freeze_time
  config.include Devise::Test::IntegrationHelpers, type: :request  # sign_in (Devise apps)
  config.infer_spec_type_from_file_location!                       # spec/requests -> type: :request
end
```

## 2. RSpec structure

```ruby
RSpec.describe RentPayment, type: :model do
  describe "#overdue?" do
    subject(:payment) { build(:rent_payment, due_on: due_on, paid_at: nil) }

    context "when the due date has passed" do
      let(:due_on) { Date.current - 1 }
      it { is_expected.to be_overdue }               # predicate matcher: calls overdue?
    end

    context "when it is due today" do
      let(:due_on) { Date.current }
      it { is_expected.not_to be_overdue }
    end
  end
end
```

**`let` vs `let!`:**

```ruby
describe ".published" do
  let!(:published) { create(:post, published: true) }   # created before every example
  let(:draft) { create(:post, published: false) }       # created only when referenced

  it "excludes drafts" do
    expect(Post.published).to eq([published])           # passes even with a broken scope: draft never existed
  end

  it "excludes drafts (for real)" do
    draft                                               # reference it (or use let!) so it exists
    expect(Post.published).to contain_exactly(published)
  end
end
```

- `let` defines a memoized method: evaluated on first use, cached for that example, rebuilt for the next one. Prefer it; use `let!` only for records that must exist without being referenced.
- Hooks: `before` / `after` run around each example (`before(:each)` is the default). `around` wraps an example (`Time.use_zone("Europe/Brussels") { example.run }`). `before(:all)` / `before(:context)` run once and are not rolled back; avoid them for database records.
- `subject` and `is_expected` keep one-liners readable; name it (`subject(:payment)`) when examples refer to it.

**Shared examples** reuse a behavior across groups:

```ruby
RSpec.shared_examples "an authenticated endpoint" do |verb, path|
  it "returns 401 without a token" do
    public_send(verb, path)
    expect(response).to have_http_status(:unauthorized)
  end
end

RSpec.describe "Listings API", type: :request do
  it_behaves_like "an authenticated endpoint", :get, "/api/v1/listings"
  it_behaves_like "an authenticated endpoint", :post, "/api/v1/listings"
end
```

## 3. Matchers

| Kind | Examples |
|---|---|
| Equality and identity | `eq` (`==`), `eql`, `equal` / `be` (same object) |
| Truthiness | `be true`, `be_truthy`, `be_nil` |
| Predicates and attributes | `be_valid`, `be_overdue`, `have_attributes(status: "paid", amount_cents: 500)` |
| Collections | `include(x)`, `contain_exactly(a, b)` / `match_array([a, b])` (any order), `be_empty`, `all(be_positive)` |
| Errors | `expect { pay! }.to raise_error(PaymentError, /declined/)` |
| Changes | `expect { create_bill }.to change(Bill, :count).by(1)`; `change { payment.reload.status }.from("pending").to("paid")` |
| Rails (rspec-rails) | `have_http_status(:created)`, `redirect_to(bill_path(bill))`, `have_enqueued_job(SyncJob).with(store.id)`, `have_enqueued_mail(RentMailer, :reminder)` |
| Composable | `a_hash_including(id: 1)`, `an_instance_of(Bill)`, combining with `.and` / `.or` |

shoulda-matchers turns common model checks into one-liners: `it { is_expected.to belong_to(:house) }`, `it { is_expected.to validate_presence_of(:amount_cents) }`, `it { is_expected.to validate_numericality_of(:amount_cents).is_greater_than(0) }`. Tag an example with `:aggregate_failures` to see every failed expectation instead of only the first.

## 4. FactoryBot vs fixtures

```ruby
# spec/factories/orders.rb
FactoryBot.define do
  factory :user do
    sequence(:email) { |n| "user#{n}@example.com" }     # unique per call
    password { "long-password-1" }
    trait(:admin) { role { :admin } }
  end

  factory :order do
    user                                               # association: uses the :user factory
    status { :pending }

    trait :paid do
      status { :paid }
      paid_at { Time.current }
    end

    trait :with_items do
      transient { items_count { 2 } }                  # input that is not an attribute
      after(:create) do |order, evaluator|
        create_list(:order_item, evaluator.items_count, order: order)
      end
    end
  end
end

build(:order)                      # in memory; its user is built too (parent strategy)
create(:order, :paid)              # saved, with a saved user
create(:order, :with_items, items_count: 3)
build_stubbed(:order)              # fake id, persisted? is true, any DB access raises
attributes_for(:order)             # plain hash, handy for request params
```

| | FactoryBot | Rails fixtures |
|---|---|---|
| Defined in | Ruby (`spec/factories`) | YAML (`test/fixtures/*.yml`, ERB allowed) |
| When data exists | Created per test, only what you ask for | All loaded once into the test DB and shared |
| Speed | Slower when tests `create` heavily | Fast: tests read pre-inserted rows |
| Variations | Traits and overrides in the test | One named fixture per variation (`users(:admin)`) |
| Readability | Setup visible in the test | Data lives far from the test |
| Typical with | RSpec | Minitest and the Rails default |

Keep factories minimal and valid (only required attributes); put optional details in traits. A factory that creates ten associated records by default makes every test slow.

## 5. Model specs

```ruby
RSpec.describe Bill, type: :model do
  it { is_expected.to belong_to(:house) }
  it { is_expected.to validate_numericality_of(:amount_cents).is_greater_than(0) }

  describe ".unpaid" do
    it "returns bills with at least one unpaid share" do
      unpaid = create(:bill, :with_unpaid_share)
      create(:bill, :fully_paid)
      expect(Bill.unpaid).to contain_exactly(unpaid)
    end
  end

  describe "#split_evenly!" do
    it "gives the leftover cent to the first shares" do
      bill = create(:bill, amount_cents: 1000, residents: create_list(:resident, 3))
      bill.split_evenly!
      expect(bill.shares.order(:id).pluck(:amount_cents)).to eq([334, 333, 333])
    end
  end
end
```

Test scopes with records that should and should not match, and test callbacks through their visible effect.

## 6. Request specs (preferred over controller specs)

| | Request specs | Controller specs |
|---|---|---|
| What runs | Router, middleware, controller, views or serializers | One controller action in isolation |
| Catches | Routing, auth filters, params, status codes, response shape | Mostly action logic |
| Status | Recommended by Rails and RSpec since Rails 5 / RSpec 3.5 | Discouraged; `assigns` and `assert_template` moved to the rails-controller-testing gem |

```ruby
RSpec.describe "Bills", type: :request do
  let(:resident) { create(:resident) }
  before { sign_in resident }

  it "re-renders the form with 422 when invalid" do
    post bills_path, params: { bill: { title: "", amount_cents: 0 } }
    expect(response).to have_http_status(:unprocessable_entity)
    expect(response.body).to include("can&#39;t be blank")
  end

  it "does not show another house's bill" do
    get bill_path(create(:bill))                     # another house: the scoped find raises RecordNotFound
    expect(response).to have_http_status(:not_found) # rendered as 404 (rescue_from, or Rails 7.1+ test defaults)
  end
end
```

JSON API specs (headers, `as: :json`, `response.parsed_body`) are in [API development](10-api-development.md).

## 7. System specs with Capybara

```ruby
# spec/rails_helper.rb
config.before(:each, type: :system) { driven_by :rack_test }                               # no JavaScript, fast
config.before(:each, type: :system, js: true) { driven_by :selenium, using: :headless_chrome }

# spec/system/split_bill_spec.rb
RSpec.describe "Splitting a bill", type: :system do
  it "creates a bill shared by the house", js: true do
    sign_in create(:resident)                       # Devise helpers included for type: :system
    visit new_bill_path
    fill_in "Title", with: "Groceries"
    fill_in "Amount", with: "60"
    click_on "Save bill"
    expect(page).to have_content("Bill created")    # retries until it appears or the wait time ends
    expect(page).to have_css("li.share", count: 3)
  end
end
```

- Capybara's finders and matchers wait and retry (up to `Capybara.default_max_wait_time`), which handles Turbo and JavaScript timing. `expect(page.text).to include(...)` does not wait; neither does `sleep` fix anything reliably.
- `not_to have_content` waits for the text to disappear; use Capybara matchers for negative checks too.
- Cuprite (Chrome DevTools Protocol, no Selenium) is a popular faster driver. Rails saves a screenshot when a system test fails.
- Keep system specs for journeys that matter (sign-in, checkout, paying rent); cover variations in request and model specs.

## 8. Job and mailer specs

Job testing (`perform` directly, `have_enqueued_job`, `Sidekiq::Testing.fake!`, idempotency by running twice) is in [background jobs](09-background-jobs-sidekiq.md). Active Job matchers need the `:test` queue adapter in the test environment.

```ruby
RSpec.describe RentMailer, type: :mailer do
  it "sends the reminder to the resident" do
    payment = create(:rent_payment)
    mail = RentMailer.with(payment: payment).reminder
    expect(mail.to).to eq([payment.resident.email])
    expect(mail.body.encoded).to include(payment.resident.first_name)
  end
end

it "enqueues the reminder email" do
  expect { RentReminders.send_for(payment) }.to have_enqueued_mail(RentMailer, :reminder)
end
```

`perform_enqueued_jobs { ... }` (from `ActiveJob::TestHelper`) runs enqueued jobs inline and fills `ActionMailer::Base.deliveries` for mail; use it for end-to-end checks only.

## 9. Mocks, stubs, spies and verified doubles

| Term | Meaning | RSpec |
|---|---|---|
| Stub | Replace a method with a canned answer; the test does not care if it is called | `allow(obj).to receive(:fetch).and_return(data)` |
| Mock | An expectation declared before acting: this call must happen | `expect(obj).to receive(:charge).with(500).once` |
| Spy | Record calls, assert after acting (arrange, act, assert order) | `allow(...)` or `spy`, then `expect(obj).to have_received(:charge)` |
| Verified double | A fake that only accepts methods the real class defines, with matching arity | `instance_double(PaymentGateway)`, `class_double`, `instance_spy` |

```ruby
it "charges the gateway once with the rent amount" do
  gateway = instance_double(PaymentGateway)
  allow(gateway).to receive(:charge).and_return(instance_double(ChargeResult, success?: true, id: "ch_1"))

  RentPayments::Pay.call(payment: payment, gateway: gateway)        # dependency injected

  expect(gateway).to have_received(:charge).with(65_000, idempotency_key: payment.idempotency_key)
end
```

- Verified doubles fail when the real method is renamed or its arguments change; a plain `double` keeps passing while production breaks. Keep `verify_partial_doubles` on (the generated spec_helper does).
- Do not mock the object under test, and avoid `allow_any_instance_of` (a sign the design needs injection).
- "Don't mock what you don't own": wrap a third-party SDK in your own adapter, mock the adapter in most tests, and test the adapter itself against stubbed HTTP.
- Over-mocking tests the implementation: refactors break tests even when behavior is unchanged.

## 10. Testing external APIs: WebMock and VCR

```ruby
# spec/support/webmock.rb
require "webmock/rspec"
WebMock.disable_net_connect!(allow_localhost: true)      # any real HTTP call fails the test

it "parses market prices" do
  stub_request(:get, "https://api.example-prices.test/v1/products/123")
    .with(query: hash_including("token" => "test-token"))
    .to_return(status: 200, body: { id: "123", loose_price: 4599 }.to_json,
               headers: { "Content-Type" => "application/json" })
  expect(PriceClient.new(token: "test-token").fetch("123").loose_price_cents).to eq(4599)
end

it "raises RateLimited on 429 so the job can retry later" do
  stub_request(:get, %r{/v1/products/123}).to_return(status: 429, headers: { "Retry-After" => "30" })
  expect { PriceClient.new(token: "t").fetch("123") }.to raise_error(PriceClient::RateLimited)
end

it "handles timeouts" do
  stub_request(:get, %r{/v1/products/}).to_timeout
  expect { PriceClient.new(token: "t").fetch("1") }.to raise_error(PriceClient::Unavailable)
end
```

```ruby
# VCR: record a real response once, replay it afterwards
VCR.configure do |c|
  c.cassette_library_dir = "spec/cassettes"
  c.hook_into :webmock
  c.configure_rspec_metadata!                                    # `vcr: true` on an example
  c.filter_sensitive_data("<PRICE_API_TOKEN>") { ENV.fetch("PRICE_API_TOKEN", "test-token") }
  c.default_cassette_options = { record: :once }                 # never re-record silently
end

it "imports a product", vcr: true do
  expect(PriceImport.call("123").title).to eq("Super Mario 64")
end
```

| | WebMock stubs | VCR cassettes |
|---|---|---|
| Data | Written by hand | Recorded from the real API |
| Strength | Precise; easy to simulate errors, 429s and timeouts | Realistic payloads with little effort |
| Risk | Stubs drift from the real API | Stale cassettes; leaked secrets if not filtered |

Combine them: VCR for the happy path of the adapter, WebMock for every failure mode, and re-record cassettes when the provider changes. Integration design (retries, rate limits) is in [third-party API integration](../13-architecture-and-practices/05-third-party-api-integration.md).

## 11. Time helpers

```ruby
it "marks rent overdue the day after the due date" do
  payment = create(:rent_payment, due_on: Date.new(2025, 3, 1))
  travel_to Time.zone.local(2025, 3, 2, 9, 0) do
    expect(payment.overdue?).to be(true)
  end
end

freeze_time do                      # Time.current stops moving inside the block
  expect(token.expires_at).to eq(15.minutes.from_now)
end
```

`travel_to` stubs `Time.now`, `Date.today` and friends until the block ends (or `travel_back`); `travel 2.days` moves relative to now. Older apps use the Timecop gem for the same job.

## 12. Database state between tests

- `config.use_transactional_fixtures = true` (despite the name, it applies to factory data too) opens a transaction before each example and rolls it back after: fast and isolated.
- Since Rails 5.1, system tests share the database connection between the test thread and the app server thread, so transactional tests work with JavaScript drivers too.
- `after_commit` callbacks still fire inside transactional tests (since Rails 5).
- DatabaseCleaner (truncation or deletion strategies) is mostly legacy now; you may still need it for tests that spawn other processes or open separate connections.
- Data created in `before(:all)` or at file load time is outside the transaction and leaks into other tests, causing order-dependent failures. Run with `--order random` and use `--seed` to reproduce.

## 13. Test speed

- `build` or `build_stubbed` when the test never needs the database; `create` only when queries are involved.
- Trim `let!` and factory associations that create records nobody uses.
- `rspec --profile 10` lists the slowest examples; test-prof's `let_it_be` / `before_all` create shared data once per group inside a transaction, and FactoryProf shows factory cascades.
- Run in parallel with the parallel_tests gem (`bundle exec parallel_rspec`), one database per process via `TEST_ENV_NUMBER`. Rails' built-in `parallelize` is for Minitest.
- Password hashing: `has_secure_password` uses bcrypt's minimum cost in the test environment, and Devise's initializer lowers `config.stretches` in tests.
- Prefer request specs over JavaScript system specs for anything that does not need a browser.

## 14. Coverage with SimpleCov

```ruby
# Very top of spec/spec_helper.rb, before any application code loads
require "simplecov"
SimpleCov.start "rails" do
  enable_coverage :branch          # also report untaken if/else branches
  minimum_coverage 85              # fail the run below this
end
```

Coverage tells you which lines never ran, which is useful for finding untested code before a refactor. High coverage does not mean good tests: a spec can execute a line without asserting anything about it.

## 15. Minitest basics

```ruby
# test/models/rent_payment_test.rb
require "test_helper"

class RentPaymentTest < ActiveSupport::TestCase
  test "overdue after the due date" do
    payment = rent_payments(:march)                   # fixture from test/fixtures/rent_payments.yml
    travel_to payment.due_on + 1.day do
      assert payment.overdue?
    end
  end

  test "amount must be positive" do
    payment = RentPayment.new(amount_cents: 0)
    assert_not payment.valid?
    assert_includes payment.errors[:amount_cents], "must be greater than 0"
  end
end
```

| | RSpec | Minitest |
|---|---|---|
| Style | DSL: `describe`, `it`, `expect(x).to eq(y)` | Plain classes and methods: `test "..."`, `assert_equal y, x` |
| Default in Rails | No (add rspec-rails) | Yes: generators, `bin/rails test`, system tests |
| Data | Usually FactoryBot | Fixtures by default |
| Parallel | parallel_tests gem | Built in (`parallelize(workers: :number_of_processors)`) |
| Ecosystem | Rich matchers, shared examples, metadata | Smaller and faster to boot; less magic |

Both test the same things; follow whatever the codebase uses.

## 16. The TDD flow

Red, green, refactor: write a failing test that states the behavior, write the simplest code that passes, then clean up with the test as a safety net.

```ruby
RSpec.describe BillSplitter do
  it "splits cents evenly and gives the remainder to the first shares" do
    expect(BillSplitter.split(100, 3)).to eq([34, 33, 33])     # red: BillSplitter does not exist yet
  end
end

class BillSplitter                                               # green: simplest correct code
  def self.split(total_cents, count)
    base, remainder = total_cents.divmod(count)
    Array.new(count) { |i| i < remainder ? base + 1 : base }
  end
end
```

Then add the next case (zero residents should raise, negative totals are invalid) and refactor. TDD shines for pure logic and bug fixes (write the test that reproduces the bug first); exploratory UI work often gets its tests right after.

## 17. Characterization tests before refactoring legacy code

A characterization test records what the code does today, right or wrong, so a refactor cannot change behavior unnoticed:

```ruby
# Pins current behavior of LegacyPricing before extracting it into a service.
# The rounding looks odd, but stores rely on it: change only with a product decision.
it "keeps legacy rounding for foreign-currency prices" do
  listing = create(:listing, price_cents: 1999, currency: "CAD")
  expect(LegacyPricing.display_price(listing)).to eq("$20.00 CAD")
end
```

Workflow: pick the seam you will change, write request or service specs around it from real inputs (production-like data, logs), run them against the old code until green, then refactor in small steps. For wide outputs (reports, exports), a "golden master" compares the new output with a saved copy of the old one. See [legacy codebases and upgrades](17-legacy-codebases-and-upgrades.md).

## Common pitfalls
- Lazy `let` records that never get created, making a test pass for the wrong reason.
- `before(:all)` data and global state leaking between tests; order-dependent failures.
- `sleep` in system specs, or non-waiting assertions like `page.text.include?`.
- Mocking everything, including the object under test, so tests pass while production breaks.
- Plain `double`s that drift from the real class; use verified doubles.
- Real HTTP calls from tests (slow, flaky, rate limited, possibly charging real money).
- Unfiltered secrets in VCR cassettes committed to git.
- Tests that depend on the current date, time zone or random order.
- Factories that create large object graphs by default.
- Chasing 100% coverage with assertion-free tests.

## How to talk about this in an interview
- In legacy monoliths, tests are how you change code safely: describe writing characterization or request specs around an area before touching it, then making the change.
- CrystalCommerce's PriceCharting integration is a natural place for WebMock and VCR: stubbed responses for listings and prices, simulated 429s and timeouts, and jobs tested by calling `perform` twice.
- For Colochousing, request specs for the API (auth failures, validation errors, other houses' records) and model specs for money logic (split bills, rounding) are good examples.
- Be honest about your testing depth at 1+ year; showing a clear method beats claiming TDD on everything.

> **Fill in:** which test stack each project uses (RSpec or Minitest, FactoryBot or fixtures, system test driver, CI) and roughly how long the suite takes.

> **Fill in:** one test you wrote that caught a real bug, or a bug that a missing test let through and how you covered it afterwards.

## Interview questions

**Q1. Why do you write tests, and what do you test first?**
To change code without breaking it, to document behavior and to catch regressions before users do. I start with business rules and edge cases at the model or service level, then request specs for each endpoint's contract and authorization, and a few system specs for critical journeys.
*Follow-up:* What do you not test? Framework behavior and private methods directly.

**Q2. What is the difference between `let` and `let!`?**
`let` defines a lazily evaluated, memoized helper: it runs the first time an example calls it and is cached for that example only. `let!` runs before every example whether referenced or not. Use `let!` when a record must exist without being referenced, such as data a query should find or exclude.
*Follow-up:* Typical bug? A scope test that passes because the excluded record was a lazy `let` that never got created.

**Q3. Why avoid `before(:all)` for database records?**
It runs once, outside the per-example transaction, so records are never rolled back and leak into other examples and files, causing order-dependent failures. Use `before`, or test-prof's `before_all`/`let_it_be`, which wrap shared setup in a transaction.
*Follow-up:* How do you find an order-dependent failure? Rerun with the same `--seed` and bisect with `rspec --bisect`.

**Q4. What are shared examples for?**
Reusing the same behavior specs across groups, such as "an authenticated endpoint" or "an archivable record", via `shared_examples` and `it_behaves_like`, optionally with parameters. Shared contexts reuse setup instead.
*Follow-up:* Risk? Overused shared examples make failures hard to trace; keep them small.

**Q5. Factories or fixtures?**
Fixtures are YAML rows loaded once and shared, so they are fast but distant from the test and awkward for many variations. Factories build exactly the data each test needs with traits and overrides, which reads clearly but gets slow when tests create large graphs. Many RSpec codebases use FactoryBot; Rails' default with Minitest is fixtures.
*Follow-up:* Can you mix them? Yes, some teams use fixtures for reference data and factories for the rest.

**Q6. `build`, `create` or `build_stubbed`?**
`build` makes an unsaved object (and builds its associations), `create` saves it and its associations, `build_stubbed` fakes a persisted object with an ID and raises on any database access. Use the cheapest one that keeps the test valid: `build_stubbed` for pure logic, `create` when queries are involved.
*Follow-up:* What does `attributes_for` return? A hash of attributes, handy for request params.

**Q7. Request specs or controller specs?**
Request specs go through routing, middleware, filters, the controller and rendering, so they catch auth, params, routing and response-shape bugs. Controller specs test an action in isolation and have been discouraged since Rails 5 and RSpec 3.5, with `assigns` and `assert_template` extracted to a gem.
*Follow-up:* Are request specs slow? Close to controller specs in modern Rails; system specs are the slow ones.

**Q8. How do system specs handle JavaScript, and why do they get flaky?**
They drive a real headless browser (Selenium with Chrome, or Cuprite) instead of `rack_test`. Flakiness comes from timing: assertions that do not wait, `sleep`, animations, and data leaking between tests. Capybara's matchers retry until a timeout, so always assert through them.
*Follow-up:* How do you keep the suite fast? Few system specs for critical journeys; variations in request and model specs.

**Q9. What is the difference between mocks and stubs?**
A stub replaces a method with a canned return value and does not care whether it is called (`allow(...).to receive(...).and_return(...)`). A mock is an expectation that the call must happen, with given arguments or count (`expect(...).to receive(...)`). A spy records calls so you assert after acting (`have_received`).
*Follow-up:* When do you use a mock? When the call itself is the behavior, such as charging a gateway or sending a notification.

**Q10. What are verified doubles, and why use them?**
Doubles (`instance_double`, `class_double`, `instance_spy`) that check stubbed methods exist on the real class with compatible arguments. A plain `double` keeps passing after the real method is renamed; a verified one fails, keeping tests honest.
*Follow-up:* Partial doubles? `allow(RealClass).to receive(...)`; `verify_partial_doubles` checks those too.

**Q11. How do you test code that calls a third-party API?**
Never hit the real API in tests. Disable network access with WebMock and stub responses, including errors, 429s and timeouts; use VCR cassettes for realistic payloads with secrets filtered; wrap the API in an adapter so most tests stub the adapter instead of HTTP.
*Follow-up:* How do you notice the real API changed? Contract checks or re-recording cassettes periodically, plus monitoring in production.

**Q12. How do you test time-dependent code?**
With ActiveSupport time helpers: `travel_to` a fixed moment, `freeze_time`, or `travel 2.days`, inside a block so time resets. Use `Time.current` and `Date.current` in the code, and test boundaries like midnight, month end and DST changes.
*Follow-up:* What did legacy apps use? The Timecop gem.

**Q13. How is the database cleaned between tests?**
Transactional tests wrap each example in a transaction and roll it back. Since Rails 5.1 system tests share the connection with the app server thread, so this works with JavaScript drivers; DatabaseCleaner is mainly for legacy setups or tests using several processes or connections.
*Follow-up:* Do `after_commit` callbacks fire in transactional tests? Yes, since Rails 5.

**Q14. The suite takes 20 minutes. How do you speed it up?**
Profile first (`rspec --profile`, test-prof's FactoryProf), then replace unnecessary `create` with `build` or `build_stubbed`, trim `let!` and heavy factories, share expensive setup with `let_it_be`, move browser tests down to request specs, and run in parallel with parallel_tests.
*Follow-up:* Biggest usual culprit? Factory cascades creating dozens of associated records per example.

**Q15. What does 90% coverage tell you?**
That 90% of lines ran during the suite, not that they were checked: a spec can execute code without asserting on it. Coverage is useful to find untested areas before a refactor; branch coverage gives a sharper picture.
*Follow-up:* How do you set up SimpleCov? Require and start it at the top of the spec helper before the app loads.

**Q16. RSpec or Minitest?**
Minitest is Rails' default: plain Ruby classes, assertions, fixtures, built-in parallel testing and fast boot. RSpec offers a descriptive DSL, rich matchers, shared examples and metadata, and is very common in commercial Rails apps. Both are fine; consistency with the codebase matters more.
*Follow-up:* RSpec equivalent of `assert_difference`? `expect { ... }.to change(Model, :count).by(1)`.

**Q17. Do you practice TDD?**
For pure logic and bug fixes, yes: write a failing test that states the behavior or reproduces the bug, make it pass simply, then refactor. For exploratory UI work I often write tests right after, but before merging.
*Follow-up:* Benefit beyond tests? It forces small, injectable, testable designs.

**Q18. How do you refactor legacy code that has no tests?**
First write characterization tests that pin current behavior through the public interface (request or service specs fed with real inputs), get them green against the old code, then refactor in small, reviewed steps, running the tests each time.
*Follow-up:* What if the current behavior looks wrong? Pin it anyway, flag it, and change it in a separate, deliberate change.

## Further reading
- [Testing Rails Applications](https://guides.rubyonrails.org/testing.html)
- [RSpec documentation](https://rspec.info/documentation/)
- [factory_bot](https://github.com/thoughtbot/factory_bot) and [WebMock](https://github.com/bblimke/webmock)
