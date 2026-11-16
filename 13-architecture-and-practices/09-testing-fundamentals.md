# Testing Fundamentals

> **Tier:** T2 (working knowledge) · **Related:** [Testing Rails with RSpec](../05-ruby-on-rails/13-testing-rspec.md), [Testing React](../06-react/10-testing.md), [React Native testing](../08-react-native-expo/09-native-modules-permissions-testing.md), [Clean architecture (fakes)](02-clean-architecture.md), [Third-party API integration (testing integrations)](05-third-party-api-integration.md), [Legacy Rails codebases](../05-ruby-on-rails/17-legacy-codebases-and-upgrades.md), [CI/CD and deployment](../14-devops-and-tools/04-ci-cd-and-deployment.md)
> **On your resume:** feature work in legacy Rails monoliths, REST APIs and background jobs at Devntech (CrystalCommerce, Colochousing), a React Native resident app, a DRF and React LMS, and a Next.js API internship: every one of them needs a testing strategy, and legacy code especially needs tests before refactoring.

## Quick revision
- **Why:** confidence to change code (especially legacy code), regression prevention, executable documentation, design feedback, and the safety net that makes CI/CD possible.
- **Unit:** one unit of logic in isolation, milliseconds. **Integration:** several real parts together (code plus database, an endpoint plus its job). **End-to-end:** the real app through the UI. **Contract:** checks the agreement between a consumer and a provider.
- **Pyramid:** many unit tests, fewer integration, few e2e. **Trophy:** static analysis at the base and most effort in integration tests, which give the most confidence per cost in UI-heavy apps.
- **TDD:** red, green, refactor. **BDD:** describe behavior in domain language (Given, When, Then); RSpec's `describe` and `it` style comes from it.
- **Test doubles:** dummy (passed, unused), stub (canned answers), spy (records calls, assert afterwards), mock (expectations set up front), fake (working lightweight implementation such as an in-memory repository).
- **Mocks vs stubs:** stubs feed inputs and you assert on results (state); mocks verify that an outgoing call happened (behavior). Mock commands to external systems, stub queries.
- **Don't mock what you don't own:** wrap third-party SDKs in adapters and mock or fake the adapter.
- **Test behavior, not implementation:** public results and side effects, not private methods or internal state.
- **Factories** (FactoryBot, factory_boy) are flexible and explicit; **fixtures** are fast, shared and brittle. Prefer `build` and `build_stubbed` when no database is needed.
- **Flaky tests** come from time, randomness, async timing, shared state, unordered queries, real network and parallel runs. Freeze time, seed randomness, await properly, isolate state, stub the network.
- **Coverage** shows what ran, not what was verified. Use it to find untested critical code and as a floor, not as a 100% target.
- **External services:** fakes for use cases, HTTP stubbing for clients, recorded responses, contract checks and sandboxes; no real network in CI.
- **CI:** every pull request runs lint and tests with required checks; parallelize; keep the main branch green.
- **Regression test:** reproduce a bug in a failing test before fixing it. **Characterization test:** pin down what legacy code does today before refactoring it.

## 1. Why test

Tests let you change code without fear. In a legacy Rails monolith, where a model is used from twenty places and nobody remembers why a callback exists, a good test suite is what turns "I hope this works" into "the suite is green". Tests also document intended behavior, give fast feedback compared with clicking through the app, and push design toward smaller, decoupled pieces: code that is hard to test is usually too coupled.

They cost time to write and maintain. Slow, flaky or implementation-coupled tests can cost more than they save, which is why the rest of this file is about choosing the right tests.

## 2. Types of tests

| Type | Scope | Speed | Example |
|---|---|---|---|
| Static analysis | Types and lint rules | Instant | TypeScript, RuboCop, ESLint catch typos and misuse |
| Unit | One function or class, dependencies faked | Milliseconds | `PricingRule#price_for` returns 2299 for a 25.00 market price |
| Integration | Several real parts | Tens to hundreds of ms | A request spec hits `POST /api/bills` and checks the database and the enqueued job |
| End-to-end | The whole app through the UI | Seconds | Playwright logs in, adds a bill and sees it listed; Maestro does the same in the mobile app |
| Contract | Consumer and provider agreement | Fast | The mobile app's expected `/houses` response shape is verified against the API |
| Smoke | A few critical paths after deploy | Seconds | Health check and login work in production |

"Unit" is debated: some teams isolate every collaborator (solitary tests), others let a unit use its real collaborators except slow or external ones (sociable tests). Either is fine if the team is consistent.

## 3. The pyramid and the trophy

| | Testing pyramid | Testing trophy |
|---|---|---|
| Base | Many unit tests | Static analysis (types, linters) |
| Middle | Some integration tests | **Most** tests are integration tests |
| Top | Few e2e tests | Few e2e tests |
| Reasoning | Unit tests are fast and cheap; e2e tests are slow and brittle | Integration tests resemble real use and give the most confidence per cost |
| Fits | Logic-heavy backends, libraries | UI-heavy apps with good tooling (React Testing Library, MSW) |

In practice: a Rails app gets many model and service unit tests, request specs for every endpoint (including authorization and tenant isolation), and a handful of system tests for critical journeys. A React app gets component tests that render real children and mock only the network, plus a few Playwright flows. The anti-pattern is the **ice cream cone**: mostly manual and e2e testing with few fast tests.

## 4. TDD and BDD

**TDD** is a loop: write a failing test (red), write the simplest code that passes (green), clean up with the test as a safety net (refactor). It shines for pure logic (pricing, allocation, date rules) and bug fixes; it is harder for exploratory UI work. Even without strict TDD, writing the test first for a bug fix is a habit interviewers like.

```ruby
RSpec.describe RentPayment do                       # red: overdue? does not exist yet
  it "is overdue the day after its due date when unpaid" do
    payment = RentPayment.new(due_on: Date.new(2026, 11, 5), paid_at: nil)
    expect(payment.overdue?(today: Date.new(2026, 11, 6))).to be(true)
  end
end

class RentPayment < ApplicationRecord               # green: the simplest code that passes
  def overdue?(today: Date.current) = paid_at.nil? && today > due_on
end
# refactor if needed, then the next red test: a payment made on time is never overdue
```

**BDD** describes behavior in the language of the business, often straight from acceptance criteria:

```text
Feature: Split a house bill
  Scenario: Three residents share an electricity bill
    Given a bill of 100.00 EUR in house "Central"
    And residents Ana, Bo and Chen live there
    When Ana splits the bill equally
    Then each resident owes 33.33 EUR and Ana's share absorbs the remaining cent
```

Cucumber runs such Gherkin files; RSpec's `describe`, `context` and `it` bring the same spirit to code.

## 5. Test doubles

| Double | What it does | Verify by |
|---|---|---|
| Dummy | Fills a parameter, never used | Nothing |
| Stub | Returns canned answers to queries | Asserting on the result of the code under test |
| Spy | Records calls for later inspection | Asserting afterwards that a call happened |
| Mock | Has expectations set before acting; fails if they are not met | The expectation itself |
| Fake | A working, simplified implementation (in-memory repository, fake gateway) | Asserting on its state |

```ruby
logger  = double("logger")                                         # dummy: required by the signature
channel = instance_double(MarketplaceChannel, publish: "ext-1")   # stub, verified against the real class's methods
notifier = spy("notifier")                                        # spy: accepts any call and records it
Listings::Publish.new(channel: channel, notifier: notifier, logger: logger).call(listing)
expect(notifier).to have_received(:listing_published).with(listing)

expect(gateway).to receive(:charge).with(amount_cents: 1000).once  # mock: expectation before acting
Payments::Charge.new(gateway: gateway).call(order)
```

```ts
import { expect, vi } from "vitest";

const publish = vi.fn().mockResolvedValue({ externalId: "ext-1" });    // stub that also records calls
const send = vi.spyOn(mailer, "send").mockResolvedValue(undefined);    // spy on a real object's method
await publishListing({ listings: fakeRepo, channel: { publish, updateStock: vi.fn() } }, input);
expect(publish).toHaveBeenCalledWith(expect.objectContaining({ sku: "SKU1" }));
```

**Mocks vs stubs:** a stub answers questions ("what is the market price?") so you can check the result; a mock checks that your code told another system to do something ("charge 10.00", "send this email", "publish this listing"). Mock outgoing commands that are the behavior; stub incoming queries. Over-mocking (asserting every internal call) couples tests to implementation, so refactors break tests without breaking behavior.

**Don't mock what you don't own:** mocking an SDK's exact call shape breaks when the SDK changes and proves little. Wrap the SDK in your own adapter, fake or mock the adapter in unit tests, and test the adapter itself with HTTP stubs or recorded responses ([clean architecture](02-clean-architecture.md)).

## 6. What to test, and what not to

| Test | Skip |
|---|---|
| Business rules and their edge cases: zero, negative, empty, nil, boundaries, rounding | Framework behavior (that `has_many` works, that React renders) |
| Error paths: timeouts, validation failures, rate limits | Trivial getters and generated code |
| Authorization: who must **not** be able to act, and cross-tenant access | Private methods directly (test them through the public API) |
| Your side of integrations (requests sent, responses handled) | Third-party library internals |
| Every fixed bug (a regression test) | Implementation details: internal state, call order of private helpers |
| Critical user journeys end to end | Pixel-perfect markup via snapshots of everything |

Test **behavior**, not implementation: assert that `price_for` returns 2299, not that it called `nearest_99`; in React, assert what the user sees and can do (React Testing Library queries by role and text), not component state.

## 7. Factories vs fixtures

| | Factories (FactoryBot, factory_boy) | Fixtures (Rails YAML) |
|---|---|---|
| Data defined | Per test, with defaults, traits and sequences | Once, as named records shared by all tests |
| Speed | Slower if every test `create`s deep associations | Fast: loaded once and wrapped in transactions |
| Clarity | The test shows the data it relies on | Data lives elsewhere; many tests depend on the same rows |
| Risk | Slow suites from cascading creates | Brittle when someone edits a shared fixture |

```ruby
FactoryBot.define do
  factory :listing do
    store
    sequence(:sku) { |n| "SKU-#{n}" }
    price_cents { 1999 }
    stock { 3 }
    trait(:sold_out) { stock { 0 } }
  end
end

create(:listing, :sold_out)   # persisted, with a persisted store
build(:listing)               # in memory, not saved
build_stubbed(:listing)       # in memory with a fake id: fastest, no database at all
```

In TypeScript, small factory functions (`makeListing(overrides)`) play the same role.

## 8. Flaky tests

| Cause | Symptom | Fix |
|---|---|---|
| Time | Fails at midnight, month end, on daylight-saving days or in another time zone | Freeze time (`travel_to`, `vi.setSystemTime`), use explicit zones |
| Randomness | Fails for some seeds | Deterministic data; record and reuse the seed (`rspec --seed 1234`) |
| Async timing | Element or job not ready yet | Await properly (`findBy*` queries, Capybara's waiting matchers); never fixed `sleep` |
| Shared state | Passes alone, fails in the suite | Transactional tests or cleanup, reset globals and caches, random test order to expose it |
| Unordered results | `first` without `ORDER BY` | Order explicitly or compare as sets |
| Real network | Fails when a provider is slow or down | Stub HTTP (WebMock, MSW); block real connections |
| Parallel runs | Port, file or database collisions | Per-worker databases and temp directories |

```ruby
it "marks rent overdue the day after the due date" do
  travel_to Time.zone.local(2026, 11, 6, 9, 0) do           # frozen clock: no midnight surprises
    expect(build(:rent_payment, due_on: Date.new(2026, 11, 5))).to be_overdue
  end
end
```

```ts
beforeEach(() => { vi.useFakeTimers(); vi.setSystemTime(new Date("2026-11-06T09:00:00Z")); });
afterEach(() => vi.useRealTimers());
```

Process: reproduce with the same seed, run the test in a loop to confirm, quarantine it visibly (tagged and tracked) rather than ignoring it, and fix the root cause. Automatic "retry until green" hides real race conditions.

## 9. Coverage

- **Line coverage** counts executed lines; **branch coverage** counts both sides of each condition, which is stricter.
- Tools: SimpleCov (Ruby), the V8 or Istanbul providers in Vitest and Jest, coverage.py, coverlet (.NET).
- **Useful:** finding untested critical code (payments, sync, authorization), reviewing coverage of the lines a pull request changes, and a floor that stops coverage from silently dropping.
- **Vanity:** chasing 100% leads to tests without meaningful assertions and tests of getters. Coverage shows code ran, not that its result was checked.
- **Mutation testing** (mutant for Ruby, Stryker for JavaScript, TypeScript and .NET) changes the code and checks whether tests fail, measuring how good the assertions are.

## 10. Testing external services

- Use cases: a fake adapter, fast and deterministic.
- The client: HTTP stubs (WebMock in Ruby, MSW or nock in Node, responses in Python) for success, errors, timeouts and rate limits.
- Recorded responses (VCR, VCR.py) from a sandbox, with secrets filtered and re-recorded periodically.
- Contract checks: validate the response shapes you depend on; consumer-driven contract tools such as Pact for services you control on both sides.
- Provider sandboxes and test modes (for example payment test cards) for end-to-end checks outside CI.

Details and examples are in [third-party API integration](05-third-party-api-integration.md).

## 11. Tests in CI

```yaml
# .github/workflows/ci.yml (Rails example)
name: CI
on: [pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: postgres
        ports: ["5432:5432"]
        options: >-
          --health-cmd pg_isready --health-interval 10s --health-timeout 5s --health-retries 5
    env:
      RAILS_ENV: test
      DATABASE_URL: postgres://postgres:postgres@localhost:5432/app_test
    steps:
      - uses: actions/checkout@v4
      - uses: ruby/setup-ruby@v1
        with:
          bundler-cache: true
      - run: bin/rails db:create db:schema:load
      - run: bundle exec rubocop
      - run: bundle exec rspec
```

- Run lint and tests on every pull request and make them **required checks** before merging.
- Keep feedback fast: parallelize, split by timing, cache dependencies, run the slowest suites (full e2e, contract checks against sandboxes) nightly.
- Upload screenshots, videos and logs from failed e2e runs as artifacts.
- A red main branch is everyone's top priority. More in [CI/CD and deployment](../14-devops-and-tools/04-ci-cd-and-deployment.md).

## 12. Tools by stack

| Stack | Unit and integration | Doubles and data | HTTP stubbing | End-to-end | Coverage |
|---|---|---|---|---|---|
| Rails | RSpec, or Minitest (the Rails default) | rspec-mocks, FactoryBot, fixtures | WebMock, VCR | Capybara system tests, Playwright | SimpleCov |
| React, TypeScript | Vitest or Jest with React Testing Library | `vi.fn`, `jest.fn` | MSW | Playwright, Cypress | V8 or Istanbul via the runner |
| Node APIs | Vitest, Jest or `node:test`; supertest for HTTP | `vi.fn`, test data builders | MSW, nock | Playwright API tests | Same as above |
| Django, DRF | pytest with pytest-django, Django `TestCase`, DRF `APIClient` | factory_boy, `unittest.mock` | responses, VCR.py | Playwright | coverage.py |
| .NET | xUnit (also NUnit, MSTest), `WebApplicationFactory` for integration tests | Moq, NSubstitute | WireMock.Net | Playwright | coverlet |
| React Native, Expo | Jest with React Native Testing Library | `jest.fn` | MSW | Maestro (YAML flows), Detox | Jest coverage |

## 13. Regression and characterization tests

**Regression tests:** when a bug is reported, first write a test that reproduces it and fails, then fix the code until it passes. The test stays forever, named after the behavior ("does not double-count refunded items in daily sales"), so the bug cannot quietly return.

**Characterization tests** make legacy code safe to change. Before refactoring code without tests, record what it does **today**, including odd behavior, by running it on many inputs and asserting the outputs. Then refactor with the tests as a net, and change behavior only deliberately, with new tests.

```ruby
# Expected values were recorded by running the current legacy code, not taken from a spec
RSpec.describe LegacyPriceFormatter do
  { 0 => "$0.00", 5 => "$0.05", 1999 => "$19.99", 100_000 => "$1,000.00", -250 => "-$2.50" }.each do |cents, today|
    it("formats #{cents} the way production does") { expect(LegacyPriceFormatter.format(cents)).to eq(today) }
  end
end
```

Legacy code often needs **seams** first: places where you can substitute a dependency (inject a collaborator instead of calling a global, wrap `Time.now` or an HTTP client), so the code becomes testable without changing its behavior. See [legacy Rails codebases](../05-ruby-on-rails/17-legacy-codebases-and-upgrades.md).

## Common pitfalls
- Tests that assert implementation (which private method ran) and break on every refactor.
- Mocking everything, including the code under test's real collaborators, so tests pass while production fails.
- Mocking third-party SDKs directly instead of wrapping them.
- Fixed `sleep` calls to wait for async work.
- Tests that depend on the current date, random data or test order.
- Real HTTP calls in the suite.
- Deep factory chains that make a simple model test create a dozen records.
- Treating coverage percentage as the goal.
- Ignoring or permanently retrying flaky tests instead of fixing them.
- Refactoring legacy code before pinning its behavior with characterization tests.

## How to talk about this in an interview
- Be honest about your testing exposure and specific about practices: which frameworks your Devntech projects use, what you test for each change (models, services, request specs, jobs), and how you handle external APIs in tests.
- Use the legacy angle: in an old monolith you add a characterization or regression test before touching risky code. That is mature, practical testing.
- Show you think in trade-offs: pyramid vs trophy depending on the app, mocks only at boundaries, coverage as a floor rather than a target.
- From your teaching assistant years, you can mention explaining test cases and debugging to students, which builds the habit of thinking in edge cases.

> **Fill in:** which test frameworks the CrystalCommerce and Colochousing codebases use (RSpec or Minitest, FactoryBot or fixtures) and how much test coverage the legacy areas you touched had.

> **Fill in:** one bug you fixed with a regression test, or one risky change you protected with tests first.

> **Fill in:** how tests ran in CI on your projects (GitHub Actions or another tool, required checks, how long the suite took).

## Interview questions

**Q1. Why do you write tests?**
To change code with confidence, catch regressions before users do, document intended behavior and get fast feedback. Tests also improve design, because code that is hard to test is usually too coupled. In legacy code they are what makes refactoring safe.
*Follow-up:* When are tests not worth it? Throwaway spikes and prototypes; anything that will live in production deserves them.

**Q2. What is the difference between unit, integration, end-to-end and contract tests?**
Unit tests check one piece of logic in isolation, in milliseconds. Integration tests run several real parts together, such as an endpoint with the database. End-to-end tests drive the full app through its UI. Contract tests check that a consumer's expectations match what a provider actually returns.
*Follow-up:* Which catches a broken database query? An integration test; a unit test with the database faked cannot.

**Q3. Explain the testing pyramid and the testing trophy.**
The pyramid says to write many fast unit tests, fewer integration tests and very few slow e2e tests. The trophy adds static analysis at the base and puts most effort into integration tests, because they resemble real usage and catch wiring bugs at reasonable cost. Backends with rich logic lean toward the pyramid; UI-heavy apps toward the trophy.
*Follow-up:* What is the ice cream cone? An inverted suite of mostly manual and e2e tests: slow, flaky and expensive.

**Q4. What is TDD, and do you practice it?**
Test-driven development: write a failing test, make it pass with the simplest code, then refactor. It works best for pure logic and bug fixes. Answer honestly about your habits; "I always reproduce a bug with a failing test before fixing it, and use TDD for logic like pricing and allocation" is a strong, credible answer.
*Follow-up:* What does TDD do for design? It forces small, decoupled units with explicit inputs and outputs.

**Q5. What is the difference between a mock and a stub?**
A stub provides canned answers so the code under test can run, and you assert on the result (state verification). A mock carries expectations about calls the code must make, and the test fails if they do not happen (behavior verification). Use stubs for queries and mocks for outgoing commands such as charging a card or sending an email.
*Follow-up:* What are spies and fakes? A spy records calls to assert afterwards; a fake is a working simplified implementation such as an in-memory repository.

**Q6. When should you avoid mocking?**
Avoid mocking types you do not own (wrap SDKs in adapters instead), the internal collaborators of the code under test (that tests implementation, not behavior), and things that are cheap and deterministic to use for real, such as value objects or a test database.
*Follow-up:* How do you make sure a fake matches reality? Run the same contract tests against the fake and the real adapter.

**Q7. What should you test, and what should you not?**
Test business rules and edge cases, error paths, authorization including what users must not access, your side of integrations, fixed bugs and critical journeys. Skip framework behavior, trivial code, private methods directly and library internals. Assert observable behavior, not implementation.
*Follow-up:* Why not test private methods? They are implementation details; test them through the public behavior they support.

**Q8. Factories or fixtures?**
Factories build data per test with defaults and traits, so tests show what they depend on, but deep `create` chains slow suites. Fixtures load shared named records once and are fast, but tests become coupled to shared data. Many teams use factories with `build` and `build_stubbed` wherever the database is not needed.
*Follow-up:* What is `build_stubbed`? An in-memory object with a fake ID that behaves as persisted, without touching the database.

**Q9. How do you deal with flaky tests?**
Find the cause: time, randomness, async timing, shared state, unordered queries, real network or parallel collisions. Reproduce with the same seed, run the test in a loop, freeze time, seed data, await conditions instead of sleeping, isolate state and stub the network. Quarantine visibly while fixing; do not leave automatic retries as the fix.
*Follow-up:* Why are flaky tests so harmful? People start ignoring red builds, and real failures slip through.

**Q10. Is 100% code coverage a good goal?**
No. Coverage shows which code ran, not whether results were checked; chasing 100% produces assertion-free tests of trivial code. Use it to find untested critical paths, as a floor that prevents decline, and on changed lines in pull requests. Mutation testing measures assertion quality better.
*Follow-up:* Line or branch coverage? Branch coverage is stricter because it requires both outcomes of each condition.

**Q11. How do you test code that calls external services?**
Put the service behind an adapter; test use cases with a fake adapter; test the adapter with HTTP stubs for success, errors, timeouts and rate limits, or with recorded sandbox responses; add contract checks on response shapes; and block real network access in CI.
*Follow-up:* What about webhooks? Signed payload fixtures, duplicate deliveries and invalid signatures.

**Q12. How would you add tests to a legacy codebase that has none?**
Start where change happens: before modifying a risky area, write characterization tests that record its current behavior, introducing seams (injected dependencies, wrapped clocks and HTTP clients) where needed. Add a regression test with every bug fix and request specs around endpoints you touch, so coverage grows where it matters.
*Follow-up:* What if current behavior looks wrong? Pin it anyway, then change it deliberately in a separate step with a new test and stakeholder agreement.

**Q13. What is a regression test, and how do you write one?**
A test that reproduces a fixed bug so it cannot return. Write it before the fix, see it fail for the reported reason, fix the code, see it pass, and name it after the behavior with a link to the ticket.
*Follow-up:* At which level? The lowest level that reproduces the bug faithfully, usually a unit or request test rather than e2e.

## Further reading
- Rails testing guide: https://guides.rubyonrails.org/testing.html
- Vitest guide: https://vitest.dev/guide/
- Testing in Django: https://docs.djangoproject.com/en/stable/topics/testing/
