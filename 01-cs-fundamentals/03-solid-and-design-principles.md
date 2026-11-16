# SOLID and Design Principles

> **Tier:** T1 (know cold) · **Related:** [OOP](02-oop.md), [Design patterns](04-design-patterns.md), [Rails architecture patterns](../05-ruby-on-rails/12-architecture-patterns.md), [Clean architecture](../13-architecture-and-practices/02-clean-architecture.md), [Third-party API integration](../13-architecture-and-practices/05-third-party-api-integration.md), [Code quality and agile](../13-architecture-and-practices/10-code-quality-and-agile.md)
> **On your resume:** Clean Architecture in your skills; service objects, background jobs and third-party integrations (PriceCharting as a sales channel and price source) in the Rails monoliths at Devntech; typed procedures and org-scoped access control in the TypeScript platform migration.

## Quick revision
- **S**ingle Responsibility: a class has one reason to change (one responsibility, one group of people asking for changes).
- **O**pen/Closed: add new behavior by adding code (a new class), not by editing working code; design extension points where change is likely.
- **L**iskov Substitution: a subtype must work anywhere its base type is expected, with no surprises: no stronger preconditions, weaker postconditions, broken invariants or new exceptions.
- **I**nterface Segregation: many small, client-specific interfaces beat one fat interface that forces stub implementations.
- **D**ependency Inversion: high-level policy depends on abstractions, and low-level details implement them; dependency injection is the usual technique.
- **DRY:** each piece of knowledge has one authoritative home. Similar-looking code that changes for different reasons is not duplication.
- **KISS:** prefer the simplest design that solves today's problem.
- **YAGNI:** do not build features or flexibility for imagined future requirements.
- **Law of Demeter:** talk to direct collaborators only: `order.customer_city`, not `order.customer.address.city`.
- **Tell, don't ask:** tell an object to do the work (`account.withdraw(amount)`) instead of pulling its data out and deciding for it.
- **Separation of concerns:** keep presentation, domain logic, persistence and integrations apart.
- **High cohesion, low coupling** is the goal behind all of these.
- Over-applying them hurts: one-implementation interfaces everywhere, layers of indirection, abstractions invented before the second use case. The wrong abstraction costs more than some duplication.

## 1. Why principles matter

Code is read and changed far more often than it is written. These principles are heuristics for keeping the cost of change low: each one reduces how far a change ripples or how hard a piece is to test. They are not laws; good engineers know when to bend them.

## 2. S: Single Responsibility Principle

**In plain words:** a class should have one job, so it has one reason to change. "Reason" means an axis of change, often tied to a group of people: pricing rules (business), email wording (marketing), the database schema (engineering).

**Violation:**

```cpp
#include <stdexcept>
#include <string>
#include <vector>
using namespace std;

struct Item { string sku; int quantity; long long priceCents; };
struct Order { string customerEmail; vector<Item> items; };

class OrderService {                      // changes whenever ANY of these concerns changes
public:
    void placeOrder(const Order& order) {
        if (order.items.empty()) throw invalid_argument("empty order");   // validation rules
        long long total = 0;
        for (const auto& i : order.items) total += i.priceCents * i.quantity;
        total = total * 117 / 100;                                      // tax rules
        // INSERT INTO orders ...                                       // persistence
        // POST to the card gateway ...                                 // payments
        // build and send the confirmation email ...                    // notifications
    }
};
```

**Refactor:** one class per concern and a thin coordinator (bodies omitted).

```cpp
class OrderValidator  { public: void validate(const Order& o) const; };
class PriceCalculator { public: long long totalCents(const Order& o) const; };
class OrderRepository { public: virtual ~OrderRepository() = default; virtual void save(const Order& o, long long total) = 0; };
class PaymentGateway  { public: virtual ~PaymentGateway() = default; virtual bool charge(const string& email, long long cents) = 0; };
class Mailer          { public: virtual ~Mailer() = default; virtual void sendConfirmation(const Order& o) = 0; };

class CheckoutService {                   // orchestrates; each collaborator owns one concern
public:
    CheckoutService(OrderValidator& v, PriceCalculator& p, OrderRepository& r, PaymentGateway& g, Mailer& m)
        : validator_(v), prices_(p), repo_(r), gateway_(g), mailer_(m) {}

    void placeOrder(const Order& order) {
        validator_.validate(order);
        long long total = prices_.totalCents(order);
        if (!gateway_.charge(order.customerEmail, total)) throw runtime_error("payment declined");
        repo_.save(order, total);
        mailer_.sendConfirmation(order);
    }
private:
    OrderValidator& validator_;
    PriceCalculator& prices_;
    OrderRepository& repo_;
    PaymentGateway& gateway_;
    Mailer& mailer_;
};
```

Now a tax change touches only `PriceCalculator`, and each piece can be unit tested alone.

> **In your web stack:** the classic Rails SRP move is extracting a service object (`Orders::Checkout.call(order)`) from a fat controller or model: the model keeps associations, validations and scopes, mailers own emails, and jobs own asynchronous work. In React, a custom hook owns data fetching while the component only renders.

## 3. O: Open/Closed Principle

**In plain words:** modules should be open for extension and closed for modification. When a new variant appears, you add a class instead of editing a growing `if/else` that already works.

**Violation:**

```cpp
// Every new channel means editing (and re-testing) this function
void publishListing(const string& channel, const Listing& l) {
    if (channel == "storefront") { /* render the product page */ }
    else if (channel == "marketplace") { /* POST to the marketplace API */ }
    else if (channel == "price_site") { /* another integration, another branch */ }
}
```

**Refactor:** a strategy interface plus a registry.

```cpp
#include <functional>
#include <map>
#include <memory>
#include <stdexcept>
#include <string>
using namespace std;

struct Listing { string sku; int quantity; };

class SalesChannel {
public:
    virtual ~SalesChannel() = default;
    virtual void publish(const Listing& l) = 0;
};

using ChannelFactory = function<unique_ptr<SalesChannel>()>;

map<string, ChannelFactory>& channelRegistry() {
    static map<string, ChannelFactory> registry;
    return registry;
}

void publishListing(const string& channel, const Listing& l) {
    auto it = channelRegistry().find(channel);
    if (it == channelRegistry().end()) throw invalid_argument("unknown channel: " + channel);
    it->second()->publish(l);              // no branching on type
}

// A new channel is a new class plus one registration line; publishListing never changes:
// channelRegistry()["marketplace"] = [] { return make_unique<MarketplaceChannel>(); };
```

OCP does not mean "never edit code". It means putting extension points where variation is likely (sales channels, payment providers, export formats) and not everywhere.

> **In your web stack:** in Ruby the registry is often just a frozen Hash from a channel name to a class: `CHANNELS.fetch(listing.channel).new.publish(listing)`. Adding a marketplace integration then means adding one class and one entry, which keeps the existing, tested channels untouched.

## 4. L: Liskov Substitution Principle

**In plain words:** if code works with a base type, it must keep working when handed any subtype. Subclasses may extend behavior but must not break the base class's promises.

**Violation (the classic):**

```cpp
#include <cassert>

class Rectangle {
public:
    virtual ~Rectangle() = default;
    virtual void setWidth(int w) { width_ = w; }
    virtual void setHeight(int h) { height_ = h; }
    int area() const { return width_ * height_; }
protected:
    int width_ = 0, height_ = 0;
};

class Square : public Rectangle {         // "a square is a rectangle"... until you mutate it
public:
    void setWidth(int w) override { width_ = height_ = w; }
    void setHeight(int h) override { width_ = height_ = h; }
};

void resize(Rectangle& r) {
    r.setWidth(5);
    r.setHeight(4);
    assert(r.area() == 20);               // fails for a Square: its area is 16
}
```

`Rectangle` implicitly promises that width and height change independently; `Square` breaks that promise, so code written for rectangles misbehaves.

**Refactor:** remove the broken promise. Shapes become immutable values behind a read-only interface, so no subtype has to honor a setter it cannot support.

```cpp
class Shape {
public:
    virtual ~Shape() = default;
    virtual int area() const = 0;            // the only promise: you can ask for the area
};

class Rectangle : public Shape {
public:
    Rectangle(int w, int h) : width_(w), height_(h) {}
    int area() const override { return width_ * height_; }
private:
    int width_, height_;
};

class Square : public Shape {                // a sibling, not a child, of Rectangle
public:
    explicit Square(int side) : side_(side) {}
    int area() const override { return side_ * side_; }
private:
    int side_;
};
// "Resizing" now means constructing a new value: Rectangle(5, 4) or Square(4).
```

**Rules a subtype must follow:**
- Do not strengthen preconditions (accept at least what the base accepts).
- Do not weaken postconditions (deliver at least what the base promises).
- Preserve invariants of the base class.
- Do not throw new kinds of exceptions for operations the base supports. A `ReadOnlyChannel` whose `syncStock` throws "not supported" breaks every caller that loops over channels.

> **In your web stack:** Rails Single Table Inheritance is where LSP bites: a subclass that raises `NotImplementedError` for an inherited method, or adds validations that make shared code paths fail, cannot stand in for its parent. If a `GuestUser` must behave like a `User`, the LSP-friendly answer is a Null Object implementing the same methods with safe defaults rather than a subclass that throws.

## 5. I: Interface Segregation Principle

**In plain words:** clients should not depend on methods they do not use. Split fat interfaces into small role interfaces.

**Violation:** one integration interface that every channel must implement in full.

```cpp
class ChannelIntegration {
public:
    virtual ~ChannelIntegration() = default;
    virtual void publish(const Listing& l) = 0;
    virtual void syncStock(const string& sku, int qty) = 0;
    virtual vector<string> importOrderIds() = 0;
    virtual long long marketPriceCents(const string& sku) = 0;   // most channels have no market data
};
// A storefront is forced to implement marketPriceCents by throwing "not supported".
```

**Refactor:** role interfaces that classes combine as needed.

```cpp
#include <string>
#include <vector>
using namespace std;

struct Listing { string sku; int quantity; };

class ListingPublisher { public: virtual ~ListingPublisher() = default; virtual void publish(const Listing& l) = 0; };
class OrderSource      { public: virtual ~OrderSource() = default; virtual vector<string> importOrderIds() = 0; };
class PriceFeed        { public: virtual ~PriceFeed() = default; virtual long long marketPriceCents(const string& sku) = 0; };

class Storefront : public ListingPublisher {                    // implements only what it supports
public:
    void publish(const Listing&) override { /* render */ }
};

class MarketplaceIntegration : public ListingPublisher, public OrderSource, public PriceFeed {
public:
    void publish(const Listing&) override { /* POST listing */ }
    vector<string> importOrderIds() override { return {}; }
    long long marketPriceCents(const string&) override { return 0; }
};

long long suggestedPriceCents(PriceFeed& feed, const string& sku) {   // depends on one small role
    return feed.marketPriceCents(sku) * 95 / 100;
}
```

An integration that is both a sales channel and a source of market prices implements several roles, while a repricing job depends only on `PriceFeed` and can be tested with a three-line fake.

> **In your web stack:** TypeScript makes small interfaces cheap: `interface PriceFeed { marketPriceCents(sku: string): Promise<number> }`. In React, the same idea is passing components only the props they use (`<PriceTag cents={listing.priceCents} />` rather than the whole listing object), which keeps them reusable and re-render friendly. Ruby gets ISP almost for free through duck typing: a method needs only the messages it actually sends.

## 6. D: Dependency Inversion Principle

**In plain words:** business logic should not depend on concrete infrastructure (HTTP clients, databases, SDKs). Both should depend on an abstraction owned by the business side, and the concrete implementation is passed in.

**Violation:**

```cpp
#include <string>
using namespace std;

class PriceApiHttpClient {                          // talks to a real third-party API
public:
    long long fetchPriceCents(const string& sku) { /* HTTP call */ return 0; }
};

class RepricingService {
    PriceApiHttpClient client_;                     // hard-wired: cannot test without the network
public:
    long long suggestedPrice(const string& sku) { return client_.fetchPriceCents(sku) * 95 / 100; }
};
```

**Refactor:**

```cpp
#include <string>
using namespace std;

class PriceSource {                                 // the abstraction, owned by the business logic
public:
    virtual ~PriceSource() = default;
    virtual long long priceCents(const string& sku) = 0;
};

class RepricingService {
public:
    explicit RepricingService(PriceSource& source) : source_(source) {}   // injected
    long long suggestedPrice(const string& sku) { return source_.priceCents(sku) * 95 / 100; }
private:
    PriceSource& source_;
};

class HttpPriceSource : public PriceSource {        // detail: implements the abstraction
public:
    long long priceCents(const string& sku) override { /* HTTP call */ return 0; }
};

class FakePriceSource : public PriceSource {        // used in tests: fast and deterministic
public:
    long long priceCents(const string&) override { return 1000; }
};

// RepricingService prod(httpSource);   RepricingService test(fakeSource);  test.suggestedPrice("A1") == 950
```

Terms people mix up:
- **Dependency Inversion (DIP):** the principle about the direction of dependencies (policy depends on abstractions).
- **Dependency Injection (DI):** the technique of passing dependencies in (constructor, parameter, setter) instead of creating them inside.
- **Inversion of Control (IoC):** the broader idea that a framework calls your code, and DI containers can wire dependencies for you.

> **In your web stack:** in Ruby, a default argument keeps call sites simple while allowing injection in specs:
```ruby
class RepricingService
  def initialize(price_source: PriceApi::Client.new)   # real client by default
    @price_source = price_source
  end
end
RepricingService.new(price_source: FakePriceSource.new) # in a spec: no HTTP
```
In TypeScript, pass clients as function parameters or through request context (oRPC handlers receive a typed context, a natural place to inject the database and API clients).

## 7. DRY, KISS and YAGNI

**DRY (Don't Repeat Yourself)** is about knowledge, not keystrokes. A business rule (how tax is calculated, which statuses count as "active") should live in one place, such as a model method or a scope. Two snippets that look alike but change for different reasons are not duplicates, and merging them creates coupling. A practical rule of three: tolerate the second copy, abstract on the third, once you can see what actually varies.

**KISS (Keep It Simple)** favors the design a teammate can understand quickly: a scheduled job that runs a nightly sync before an event-streaming platform; a plain Ruby object before a framework; a SQL query before a caching layer, until measurements say otherwise.

**YAGNI (You Aren't Gonna Need It)** says not to build for hypothetical requirements: no plugin system for two sales channels, no multi-currency support until a market requires it. YAGNI does not forbid clean code: keeping code easy to change is how you stay ready for the future without paying for it today.

## 8. Law of Demeter and tell, don't ask

```cpp
// Law of Demeter violation: the caller depends on Order, Customer AND Address internals
string city = order.customer().address().city();

// Ask: the caller pulls data out and makes the decision
if (account.balance() >= amount) account.setBalance(account.balance() - amount);

// Tell: the object that owns the data owns the rule
account.withdraw(amount);                  // throws or returns false when funds are insufficient
```

The Law of Demeter says a method should call methods only on itself, its parameters, objects it creates and its direct fields, not on objects returned by those. The goal is fewer hidden dependencies, so a change in `Address` does not break code that only knows about `Order`. It is not a "count the dots" rule: fluent APIs that return the same kind of object (`Order.where(...).order(...).limit(5)`) are fine.

**Tell, don't ask** keeps behavior next to the data it needs: an object that exposes `withdraw` can enforce its invariant; one that exposes `setBalance` cannot.

> **In your web stack:** Rails `delegate` is the idiomatic Demeter fix:
```ruby
class Order < ApplicationRecord
  belongs_to :customer
  delegate :city, to: :customer, prefix: true, allow_nil: true   # order.customer_city
end
```
`Customer` can in turn delegate `city` to its address, so views call `order.customer_city` and never learn the structure behind it.

## 9. Separation of concerns, cohesion and coupling

Keep code that changes for different reasons in different places:

| Concern | Rails home | React / TypeScript home |
|---|---|---|
| HTTP and request handling | controllers | route handlers, server functions, oRPC procedures |
| Domain rules | models, plain Ruby objects, service objects | domain modules, server-side services |
| Persistence | ActiveRecord models and scopes | ORM layer (Drizzle schema and queries) |
| Presentation | views, partials, serializers | components |
| Async work | jobs (Sidekiq, ActiveJob) | queues, background workers |
| Third-party integrations | API client classes behind an interface | client modules injected into services |
| Server state and caching | caching layer | TanStack Query |

High cohesion (each module does one coherent thing) and low coupling (modules depend on small, stable interfaces) are the outcomes; SOLID and the principles above are the techniques. See [MVC and layered architecture](../13-architecture-and-practices/01-mvc-and-layered-architecture.md) and [Clean architecture](../13-architecture-and-practices/02-clean-architecture.md).

## 10. When over-applying these principles hurts

- An interface for every class even when there will only ever be one implementation, which adds indirection with no benefit (in Ruby and TypeScript, duck and structural typing make most of these unnecessary).
- A service object for every one-line operation, or a "Manager", "Handler" and "Processor" for each feature, so reading one flow means opening ten files.
- Abstractions created before the second use case, which usually guess the wrong axis of change; later requirements then fight the abstraction.
- DRYing tests until each test is unreadable without its shared helpers.
- Splitting a cohesive class into many tiny ones in the name of SRP, increasing coupling between them.

A balanced stance for interviews: apply the principles where change is likely or testability demands it, and refactor toward them when the second or third case actually appears.

## Common pitfalls
- Reciting definitions without an example; interviewers ask "show me" immediately.
- Treating SRP as "a class should have one method".
- Thinking OCP means never editing existing code.
- Explaining LSP only with Rectangle and Square and not knowing a real-world case (STI subclasses that throw, channels that do not support an operation).
- Confusing Dependency Inversion with Dependency Injection.
- Treating DRY as removing all similar-looking code, coupling unrelated features.
- Applying every principle everywhere, producing indirection that makes simple changes slow.

## How to talk about this in an interview
- Have one concrete example ready per principle, ideally from your work: a service object extracted from a fat model (SRP), sales channels behind a common interface (OCP and LSP), a price source injected so jobs can be tested without HTTP (DIP), small typed interfaces in TypeScript (ISP).
- Show judgment: mention when you would not apply a principle (YAGNI, avoiding one-implementation interfaces in Ruby).
- As an OOP lab TA, you can explain these with a small diagram or code sketch on a whiteboard; keep the C++ version short and translate it to Ruby or TypeScript to show you use the ideas daily.
- > **Fill in:** how the PriceCharting integration was structured in the CrystalCommerce codebase (for example a client class, sync jobs and a channel abstraction), described in principle terms without inventing details.
- > **Fill in:** one refactor you did or reviewed at Devntech that moved logic out of a controller, model or React component, and what it made easier (testing, reuse, readability).

## Interview questions

**Q1. Explain the Single Responsibility Principle with an example.**
A class should have one reason to change. An `OrderService` that validates, calculates tax, saves, charges and emails changes whenever any of those rules change; splitting it into a validator, a price calculator, a repository, a gateway and a mailer, coordinated by a thin checkout service, keeps each change local. In Rails this is the move from a fat controller or model to a service object plus a mailer and a job.
*Follow-up:* How do you identify responsibilities? Ask who would request a change to each part: finance, marketing, operations or engineering.

**Q2. Explain the Open/Closed Principle with an example.**
Code should be open for extension but closed for modification. Instead of an `if/else` on channel type in `publishListing`, define a `SalesChannel` interface and register one class per channel; adding a marketplace means adding a class and a registry entry while existing, tested code stays untouched.
*Follow-up:* Does that mean you never edit old code? No; you add extension points where variation is expected and refactor when a new kind of variation appears.

**Q3. Explain the Liskov Substitution Principle with an example.**
Any subtype must be usable through the base type without breaking callers. A `Square` that extends a mutable `Rectangle` breaks code that sets width and height independently, and a channel subclass that throws "not supported" from `syncStock` breaks loops over all channels. Fix by redesigning the hierarchy (immutable shapes, smaller interfaces) or using composition.
*Follow-up:* What rules does a subtype follow? No stronger preconditions, no weaker postconditions, preserved invariants, no new exceptions.

**Q4. Explain the Interface Segregation Principle with an example.**
Clients should not be forced to depend on methods they do not use. A fat `ChannelIntegration` with publishing, order import and market prices forces a storefront to stub out market prices; splitting it into `ListingPublisher`, `OrderSource` and `PriceFeed` lets each class implement only its roles and lets a repricing job depend only on `PriceFeed`.
*Follow-up:* How does this show up in React? Components receive only the props they use.

**Q5. Explain the Dependency Inversion Principle with an example.**
High-level logic should depend on abstractions rather than concrete infrastructure. A `RepricingService` that creates its own HTTP client cannot be tested without the network; depending on a `PriceSource` interface and receiving the implementation through the constructor allows a real HTTP source in production and a fake in tests.
*Follow-up:* How do you do this in Ruby without interfaces? Inject the collaborator with a keyword argument that defaults to the real client.

**Q6. Which SOLID principle do you find most useful in day-to-day Rails or TypeScript work?**
Usually SRP and DIP. SRP keeps models and controllers from turning into thousand-line files, and DIP makes third-party integrations testable by injecting clients. A good answer names a concrete case and the benefit (faster tests, easier changes).
*Follow-up:* Which is least relevant in Ruby? ISP, because duck typing means a method depends only on the messages it sends.

**Q7. What is the difference between SRP and ISP?**
SRP is about the implementation side: a class should have one reason to change. ISP is about the client side: an interface should not force clients to depend on methods they do not need. A class can satisfy SRP and still expose a fat interface, and vice versa.
*Follow-up:* Can one refactor serve both? Yes, splitting a god class often produces both smaller classes and smaller interfaces.

**Q8. Dependency Inversion vs Dependency Injection vs Inversion of Control?**
Dependency Inversion is the principle: policy depends on abstractions, details implement them. Dependency Injection is a technique: pass dependencies in instead of constructing them. Inversion of Control is the broader pattern where a framework calls your code and may wire dependencies for you (DI containers).
*Follow-up:* Does Rails use a DI container? No; Rails relies on conventions, and injection is usually done manually with constructor or keyword arguments.

**Q9. Is DRY always good? When is duplication acceptable?**
DRY targets duplicated knowledge, such as a business rule implemented in two places. Code that looks alike but changes for different reasons should stay separate, because merging it couples unrelated features. Waiting until the third occurrence before abstracting usually reveals the right abstraction.
*Follow-up:* What is worse than duplication? The wrong abstraction, which every later change has to work around.

**Q10. How do you balance YAGNI with designing for extensibility?**
Build only what current requirements need, but keep the code easy to change: small classes, clear boundaries, tests. Add explicit extension points where variation has already appeared or is near certain, such as a second payment provider or sales channel.
*Follow-up:* Give a YAGNI violation. A plugin architecture for two integrations that never gained a third.

**Q11. What is the Law of Demeter, and how do you apply it in Rails?**
A method should talk only to its direct collaborators, not reach through chains of returned objects. `order.customer.address.city` couples the caller to three classes; `delegate :city, to: :customer, prefix: true` gives `order.customer_city` and hides the structure. Chains on the same type, such as ActiveRecord query builders, are fine.
*Follow-up:* What does `allow_nil: true` do on `delegate`? Returns `nil` instead of raising when the target is `nil`.

**Q12. What does "tell, don't ask" mean?**
Instead of querying an object's state and deciding on its behalf, tell it what to do and let it enforce its own rules: `account.withdraw(amount)` rather than checking `balance` and calling `setBalance`. It keeps invariants inside the object and reduces coupling.
*Follow-up:* How does it relate to encapsulation? It is encapsulation applied to behavior: data and the rules about it stay together.

**Q13. How does Single Table Inheritance in Rails relate to LSP?**
STI creates real subclasses that share one table and are often used interchangeably, so each subclass must honor the parent's contract. Subclasses that raise for inherited methods, or whose validations make shared code fail, violate LSP; such cases are better modeled with composition, roles or separate tables.
*Follow-up:* When is STI a good fit? When subclasses share nearly all columns and differ mainly in behavior.

**Q14. How do the SOLID principles make code easier to test?**
SRP gives small units with focused tests, ISP means fakes implement only small interfaces, and DIP lets you inject fakes for databases, APIs and clocks. The result is fast, deterministic unit tests plus a few integration tests for the real adapters.
*Follow-up:* What is a seam? A place where you can substitute behavior without editing the code under test, such as an injected dependency.

**Q15. Give an example of the Open/Closed Principle from a real system.**
Sales channels in an e-commerce platform: the core sync logic iterates over channel objects that share an interface, so integrating a new marketplace means writing a new channel class (and its background jobs) rather than adding branches to existing sync code. Payment providers and export formats follow the same pattern.
*Follow-up:* Which pattern implements OCP here? Strategy, often combined with a factory or registry.

**Q16. When does applying SOLID become over-engineering?**
When abstractions exist without real variation: interfaces with a single implementation, factories for objects with one constructor, a service object per trivial action, or layers that only forward calls. Signs include changes that touch many files and new teammates needing a map to follow one request.
*Follow-up:* How do you avoid it? Start simple, write tests, and refactor toward principles when duplication or change pressure appears.

**Q17. What does separation of concerns look like in a React application?**
Components render UI, custom hooks own state and effects, a data layer (for example TanStack Query plus an API client) owns server state and caching, and utilities hold pure logic. Each layer can be changed and tested independently.
*Follow-up:* What about server code in a TypeScript backend? Route handlers or procedures validate input and call services, which call repositories or ORM queries.

## Further reading
- [Rails guides: Active Support core extensions (delegate)](https://guides.rubyonrails.org/active_support_core_extensions.html)
- [TypeScript handbook: interfaces and object types](https://www.typescriptlang.org/docs/handbook/2/objects.html)
