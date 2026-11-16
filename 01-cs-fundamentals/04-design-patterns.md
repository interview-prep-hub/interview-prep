# Design Patterns

> **Tier:** T2 (working knowledge) · **Related:** [SOLID and design principles](03-solid-and-design-principles.md), [OOP](02-oop.md), [Rails architecture patterns](../05-ruby-on-rails/12-architecture-patterns.md), [React patterns and architecture](../06-react/07-patterns-and-architecture.md), [Background processing patterns](../13-architecture-and-practices/06-background-processing-patterns.md), [Express](../09-nodejs-backend/03-express.md)
> **On your resume:** Service objects, background jobs, API client wrappers and middleware across Rails, Node (Hono, oRPC) and React at Devntech, including the PriceCharting integration for CrystalCommerce.

## Quick revision
- A design pattern is a named, reusable solution to a recurring design problem. Its value is shared vocabulary ("wrap it in an adapter"), not code to paste.
- **Creational** (how objects are made): Singleton, Factory, Builder. **Structural** (how objects are composed): Adapter, Decorator, Facade, Proxy. **Behavioral** (how objects collaborate): Strategy, Observer, Command, Template Method, State.
- Adapter converts an interface; Decorator adds behavior behind the same interface; Facade simplifies a subsystem behind a new interface; Proxy controls access behind the same interface.
- Strategy: the client picks an interchangeable algorithm. State: the object switches its own behavior as its state changes. Template Method fixes the skeleton through inheritance; Strategy swaps steps through composition.
- Observer: subscribers register with a subject. Pub/Sub adds a broker so publishers and subscribers never know each other.
- Command turns a request into an object you can queue, retry, log or undo: background jobs are commands.
- Singleton is often an anti-pattern (global state, hidden dependencies, hard to test). Web patterns: Repository, Service Object, Unit of Work, Dependency Injection, Middleware (Chain of Responsibility), Pub/Sub.

## 1. Categories

| Category | Question it answers | Patterns here |
|---|---|---|
| Creational | How do we create objects without hard-coding concrete classes or messy constructors? | Singleton, Simple Factory, Factory Method, Builder |
| Structural | How do we combine objects into larger structures and adapt interfaces? | Adapter, Decorator, Facade, Proxy |
| Behavioral | How do objects divide work and communicate? | Strategy, Observer, Command, Template Method, State |

The C++ sketches below assume `#include <algorithm>`, `<functional>`, `<iostream>`, `<memory>`, `<queue>`, `<stdexcept>`, `<string>`, `<unordered_map>`, `<utility>` and `<vector>`, plus `using namespace std;`. Types such as `SalesChannel`, `Writer` and `Order` are declared elsewhere; the focus is the pattern's shape.

## 2. Creational patterns

### Singleton

**Problem:** exactly one instance must exist (configuration, a logger) with a global access point.

```cpp
class Config {
public:
    static Config& instance() {
        static Config inst;              // created on first use; thread-safe since C++11
        return inst;
    }
    Config(const Config&) = delete;      // no copies
    Config& operator=(const Config&) = delete;
    string get(const string& key) const { auto it = values_.find(key); return it == values_.end() ? "" : it->second; }
private:
    Config() = default;                  // nobody else can construct it
    unordered_map<string, string> values_;
};
```

**Why it is often an anti-pattern:** it is global mutable state in disguise, it hides dependencies (a function silently reaches for `Config::instance()`), it is hard to replace in tests, and its lifetime and initialization order can bite. Prefer creating one instance at startup and passing it to whoever needs it.

> **In your web stack:** `Rails.application`, `Rails.logger` and `Rails.cache` are process-wide objects; Ruby's standard library offers `include Singleton` (`Klass.instance`). Node caches modules after the first import, so a module-level database or Redis client behaves like a singleton. React context gives one instance per component subtree instead of a true global.

### Simple Factory and Factory Method

**Problem:** callers should not know which concrete class to build; the choice depends on input or configuration.

```cpp
// Simple factory: one function decides the concrete type
unique_ptr<SalesChannel> makeChannel(const string& type) {
    if (type == "storefront") return make_unique<StorefrontChannel>();
    if (type == "marketplace") return make_unique<MarketplaceChannel>();
    throw invalid_argument("unknown channel: " + type);
}

// Factory Method: a base class defers creating a collaborator to its subclasses
class ReportExporter {
public:
    virtual ~ReportExporter() = default;
    void exportReport(const string& data) { createWriter()->write(data); }   // uses the product
protected:
    virtual unique_ptr<Writer> createWriter() = 0;                           // the factory method
};
// class CsvExporter : public ReportExporter { createWriter() returns make_unique<CsvWriter>() }
```

> **In your web stack:** ActiveRecord acts as a factory when it instantiates the right STI subclass from the `type` column; FactoryBot (`create(:listing)`) builds test data; `React.createElement` builds elements from a type and props; a hash from channel names to classes is a simple factory in Ruby.

### Builder

**Problem:** an object has many optional parts; telescoping constructors become unreadable and invalid combinations slip through.

```cpp
struct HttpRequest { string method = "GET", url, body; unordered_map<string, string> headers; };
class RequestBuilder {
public:
    explicit RequestBuilder(string url) { req_.url = move(url); }
    RequestBuilder& method(string m) { req_.method = move(m); return *this; }   // fluent: returns *this
    RequestBuilder& header(const string& k, const string& v) { req_.headers[k] = v; return *this; }
    RequestBuilder& body(string b) { req_.body = move(b); return *this; }
    HttpRequest build() const {
        if (req_.method == "GET" && !req_.body.empty()) throw logic_error("GET with a body");
        return req_;                                                            // validated product
    }
private:
    HttpRequest req_;
};
// auto req = RequestBuilder("https://api.example.com/listings").method("POST")
//                .header("Content-Type", "application/json").body(R"({"sku":"A1"})").build();
```

> **In your web stack:** the ActiveRecord query interface is a builder: `Listing.where(store_id: 7).order(:price_cents).limit(10)` accumulates a Relation and runs SQL only when results are needed. Drizzle's `db.select().from(users).where(eq(users.id, id))` follows the same idea in TypeScript.

## 3. Structural patterns

### Adapter

**Problem:** a third-party API has an interface you cannot change, and your code wants its own.

```cpp
class ThirdPartyPriceApi {                     // vendor SDK: product ids and raw JSON
public:
    string getProductJson(int productId);
};
class PriceSource {                            // the interface your code depends on
public:
    virtual ~PriceSource() = default;
    virtual long long priceCents(const string& sku) = 0;
};
class PriceApiAdapter : public PriceSource {   // translates between the two
public:
    explicit PriceApiAdapter(ThirdPartyPriceApi& api) : api_(api) {}
    long long priceCents(const string& sku) override {
        string json = api_.getProductJson(lookupProductId(sku));  // our key to theirs
        return parsePriceCents(json);                             // their format to ours
    }
private:
    ThirdPartyPriceApi& api_;
    int lookupProductId(const string& sku);
    long long parsePriceCents(const string& json);
};
```

> **In your web stack:** wrapping an external API client in your own class keeps vendor details in one file, so the rest of the app (and its tests) depends on your interface. Rails itself is full of adapters: database adapters behind ActiveRecord, storage services behind Active Storage, queue backends behind ActiveJob's `perform_later`.

### Decorator

**Problem:** add responsibilities to an object at runtime without a subclass for every combination, keeping the same interface.

```cpp
class Notifier { public: virtual ~Notifier() = default; virtual void send(const string& msg) = 0; };
class EmailNotifier : public Notifier { public: void send(const string& msg) override { cout << "email: " << msg << '\n'; } };
class LoggingNotifier : public Notifier {      // wraps any Notifier and adds logging
public:
    explicit LoggingNotifier(unique_ptr<Notifier> inner) : inner_(move(inner)) {}
    void send(const string& msg) override {
        cout << "log: sending\n";
        inner_->send(msg);                     // delegate to the wrapped object
    }
private:
    unique_ptr<Notifier> inner_;
};
// auto n = make_unique<LoggingNotifier>(make_unique<EmailNotifier>());  // decorators stack
```

> **In your web stack:** presenters built on Ruby's `SimpleDelegator` add view helpers around a model; `Module#prepend` wraps methods; React higher-order components (`withAuth(Component)`) wrap components; TypeScript decorators annotate classes and methods.

### Facade

**Problem:** a subsystem has many parts and steps; callers need one simple entry point.

```cpp
class CheckoutFacade {
public:
    CheckoutFacade(Inventory& inv, PaymentGateway& pay, Shipping& ship, Mailer& mail)
        : inv_(inv), pay_(pay), ship_(ship), mail_(mail) {}
    void placeOrder(const Order& o) {          // one call hides four subsystems
        inv_.reserve(o);
        pay_.charge(o);
        ship_.schedule(o);
        mail_.confirm(o);
    }
private:
    Inventory& inv_; PaymentGateway& pay_; Shipping& ship_; Mailer& mail_;
};
```

> **In your web stack:** a service object such as `Orders::Checkout.call(order)` is a facade over models, gateways and mailers; `Rails.cache` and Active Storage present simple APIs over complex backends; a custom React hook can be a facade over several queries and mutations.

### Proxy

**Problem:** control access to an object (lazy loading, caching, permission checks, remote calls) without changing its interface.

```cpp
class CachingPriceSource : public PriceSource {     // same interface as the real source
public:
    explicit CachingPriceSource(PriceSource& real) : real_(real) {}
    long long priceCents(const string& sku) override {
        auto it = cache_.find(sku);
        if (it != cache_.end()) return it->second;  // served without calling the real object
        return cache_[sku] = real_.priceCents(sku);
    }
private:
    PriceSource& real_;
    unordered_map<string, long long> cache_;
};
```

> **In your web stack:** ActiveRecord association proxies (`store.listings` returns a collection proxy that loads lazily and adds query methods); JavaScript's built-in `Proxy` object intercepts property access; at the network level, Nginx acts as a reverse proxy in front of app servers.

| | Interface seen by the client | Main purpose |
|---|---|---|
| Adapter | a different one (yours) | make an incompatible interface fit |
| Decorator | the same | add behavior, stackable |
| Facade | a new, simpler one | hide a complex subsystem |
| Proxy | the same | control access: lazy, cached, protected or remote |

## 4. Behavioral patterns

### Strategy

**Problem:** several interchangeable algorithms, chosen at runtime, without `if/else` chains.

```cpp
using PricingStrategy = function<long long(long long marketCents)>;
long long priceFor(long long marketCents, const PricingStrategy& strategy) { return strategy(marketCents); }

PricingStrategy matchMarket = [](long long m) { return m; };
PricingStrategy undercut5   = [](long long m) { return m * 95 / 100; };
PricingStrategy withFloor   = [](long long m) { return max(m * 90 / 100, 500LL); };   // never below 5.00
// priceFor(2000, undercut5) == 1900
```

> **In your web stack:** sales channels and payment providers behind one interface, Warden and Passport.js authentication strategies, a comparator passed to `Array.prototype.sort`, or a formatting function passed as a React prop. Letting each store choose how to price inventory from market data is a natural strategy.

### Observer (and Pub/Sub)

**Problem:** when something happens, several parts of the system must react, without the subject knowing who they are.

```cpp
class EventBus {
public:
    using Handler = function<void(const string& payload)>;
    void subscribe(const string& event, Handler h) { handlers_[event].push_back(move(h)); }
    void publish(const string& event, const string& payload) {
        for (auto& h : handlers_[event]) h(payload);    // the publisher never names its subscribers
    }
private:
    unordered_map<string, vector<Handler>> handlers_;
};
```

> **In your web stack:** `ActiveSupport::Notifications` is built in:

```ruby
ActiveSupport::Notifications.subscribe("listing.synced") do |event|
  Rails.logger.info("synced #{event.payload[:sku]} in #{event.duration.round(1)}ms")
end
ActiveSupport::Notifications.instrument("listing.synced", sku: listing.sku) { channel.publish(listing) }
```

Model callbacks (`after_commit`) are a lightweight observer, DOM `addEventListener` and Node's `EventEmitter` are observers, and React stores notify subscribed components. Pub/Sub inserts a broker (Redis pub/sub, Action Cable, a message queue) so producers and consumers can live in different processes. Heavy callback use hides control flow, so keep side effects explicit where possible.

### Command

**Problem:** treat a request as an object so it can be queued, scheduled, retried, logged or undone.

```cpp
class Command { public: virtual ~Command() = default; virtual void execute() = 0; };

class SyncListingCommand : public Command {
public:
    explicit SyncListingCommand(string sku) : sku_(move(sku)) {}
    void execute() override { cout << "syncing " << sku_ << '\n'; }
private:
    string sku_;
};
// queue<unique_ptr<Command>> jobs; jobs.push(...); a worker pops, executes, retries and logs later
```

> **In your web stack:** a background job is a command: `SyncListingJob.perform_later(listing.id)` serializes the job class and its arguments into a queue so a worker can run, retry and log it later. Pass ids rather than objects and make jobs idempotent, because queues such as Sidekiq deliver at least once. Redux actions, undo stacks and Rails migrations (`up`/`down`) are commands too.

### Template Method

**Problem:** an algorithm has a fixed sequence of steps, but some steps differ per variant.

```cpp
class Importer {
public:
    virtual ~Importer() = default;
    void run() {                                      // the fixed skeleton
        for (const auto& row : fetchRows()) if (isValid(row)) save(row);
        report();
    }
protected:
    virtual vector<string> fetchRows() = 0;           // required steps
    virtual void save(const string& row) = 0;
    virtual bool isValid(const string& row) { return !row.empty(); }   // hooks with defaults
    virtual void report() { cout << "import finished\n"; }
};
```

> **In your web stack:** frameworks are template methods at scale: Rails runs callbacks around your controller action or job's `perform`, test frameworks call your `setup` and `teardown`, and an `ApplicationService` base class may define `call` while subclasses implement the steps. Template Method relies on inheritance; Strategy achieves similar flexibility through composition.

### State

**Problem:** behavior depends on the current state, and `switch (status)` statements spread across the code.

```cpp
class OrderState {
public:
    virtual ~OrderState() = default;
    virtual unique_ptr<OrderState> pay()  { throw logic_error("cannot pay in state " + name()); }
    virtual unique_ptr<OrderState> ship() { throw logic_error("cannot ship in state " + name()); }
    virtual string name() const = 0;
};
class Shipped : public OrderState { public: string name() const override { return "shipped"; } };
class Paid : public OrderState {
public:
    unique_ptr<OrderState> ship() override { return make_unique<Shipped>(); }
    string name() const override { return "paid"; }
};
class Pending : public OrderState {
public:
    unique_ptr<OrderState> pay() override { return make_unique<Paid>(); }
    string name() const override { return "pending"; }
};
class Order {
public:
    void pay()  { state_ = state_->pay(); }           // the current state decides the next one
    void ship() { state_ = state_->ship(); }
    string status() const { return state_->name(); }
private:
    unique_ptr<OrderState> state_ = make_unique<Pending>();
};
```

> **In your web stack:** order, payment and application workflows. Rails apps typically combine `enum :status` with guard methods, or use a state machine gem such as AASM that declares events and allowed transitions; JavaScript has XState. Unlike Strategy, the client does not pick the behavior: transitions change it.

## 5. Web and architectural patterns

| Pattern | Problem it solves | In Rails | In Node, TypeScript and React |
|---|---|---|---|
| Repository | hide persistence behind a collection-like interface | ActiveRecord follows the Active Record pattern instead (a model is both row and persistence); repositories appear for complex queries or external stores | repository modules over Drizzle or Mongoose; in-memory versions in tests |
| Service Object | one business operation in one place | `app/services/orders/checkout.rb` exposing `.call` | service functions called by route handlers or oRPC procedures |
| Unit of Work | commit a group of changes atomically | `ActiveRecord::Base.transaction do ... end` | `db.transaction(async (tx) => { ... })` in Drizzle |
| Dependency Injection | supply collaborators from outside | keyword arguments with defaults; `Current` attributes for request context | function parameters, React context, typed request context |
| Middleware (Chain of Responsibility) | pass a request through handlers that can act, modify or stop it | the Rack middleware stack; controller callbacks | Express `(req, res, next)`, Hono `app.use(async (c, next) => { await next() })` |
| Pub/Sub | decouple producers from consumers through a broker | Action Cable, Redis pub/sub, `ActiveSupport::Notifications` | WebSockets, Redis pub/sub, event emitters |

A Rack middleware is a class whose `initialize(app)` stores the next link and whose `call(env)` can act before and after calling `@app.call(env)`, returning `[status, headers, body]`; you register it with `config.middleware.use`.

## Common pitfalls
- Starting from a pattern and forcing the problem to fit it; patterns should emerge from real needs.
- Singletons as convenient globals, making tests order-dependent and hiding dependencies.
- Confusing Decorator, Proxy and Adapter: ask whether the interface stays the same and what the wrapper is for.
- Deep Template Method hierarchies that are hard to follow; prefer Strategy when variants multiply.
- Observer chains (especially model callbacks) that trigger hidden side effects in surprising order.
- Commands (jobs) that are not idempotent, so a retry charges twice or sends two emails.

## How to talk about this in an interview
- Map each pattern to code you have touched: jobs as Commands, an API client wrapper as an Adapter, service objects as Facades, sales channels or pricing rules as Strategies, `ActiveSupport::Notifications` and callbacks as Observers, Rack and Hono middleware as Chain of Responsibility, ActiveRecord relations as Builders.
- Lead with the problem the pattern solves, then the structure, then a real example; that order sounds like experience rather than memorization.
- > **Fill in:** which of these patterns appear in the CrystalCommerce or Colochousing codebases you worked on (for example how the PriceCharting client is wrapped, how sync jobs are structured, or how status changes are modeled).
- > **Fill in:** one place in the TypeScript platform migration where middleware or a typed context handled cross-cutting concerns such as org-scoped access control.

## Interview questions

**Q1. What is a design pattern, and what are the main categories?**
A named, reusable solution to a recurring design problem, useful mainly as shared vocabulary. Creational patterns deal with object creation (Singleton, Factory, Builder), structural ones with composing objects (Adapter, Decorator, Facade, Proxy), and behavioral ones with collaboration (Strategy, Observer, Command, Template Method, State).
*Follow-up:* Are patterns language-specific? The ideas are general, but dynamic languages make some trivial: a Ruby Strategy can be a lambda.

**Q2. Why is Singleton often called an anti-pattern? How do you write a thread-safe one in C++?**
It introduces global mutable state, hides dependencies, complicates testing and couples code to one concrete instance. In C++11 and later, a function-local static (`static Config inst; return inst;`) is initialized exactly once even with concurrent callers. Prefer creating one instance at startup and injecting it.
*Follow-up:* Where are singletons acceptable? Truly process-wide, mostly read-only services such as a logger or configuration loaded at boot.

**Q3. Factory vs Builder?**
A factory decides which concrete type to create, usually in one call (`makeChannel("marketplace")`). A builder assembles one complex object step by step through a fluent interface and validates it in `build()`. Factories hide the class choice; builders hide construction complexity.
*Follow-up:* Is ActiveRecord's query interface a builder? Yes, it accumulates conditions into a Relation and runs SQL lazily.

**Q4. Compare Adapter, Facade, Decorator and Proxy.**
Adapter converts one interface into the one your code expects. Facade puts a simpler interface in front of a complex subsystem. Decorator wraps an object with the same interface to add behavior and can be stacked. Proxy wraps an object with the same interface to control access through lazy loading, caching, permissions or remote calls.
*Follow-up:* Which would you use around a third-party pricing API? An adapter to your own interface, optionally wrapped by a caching proxy.

**Q5. Strategy vs State?**
Both delegate behavior to interchangeable objects. In Strategy the client chooses the algorithm (a pricing rule per store). In State the object changes its own behavior as it transitions (pending to paid to shipped), and each state knows which transitions are legal.
*Follow-up:* How do Rails apps usually model state? An enum column plus guard methods, or a state machine gem such as AASM.

**Q6. Strategy vs Template Method?**
Template Method uses inheritance: the base class fixes the algorithm skeleton and subclasses override steps. Strategy uses composition: the varying algorithm is a separate object injected at runtime. Strategy is more flexible and easier to test; Template Method suits frameworks with a fixed lifecycle.
*Follow-up:* Which principle favors Strategy? Composition over inheritance.

**Q7. Where does the Observer pattern appear in Rails, and what are the downsides of callbacks?**
`ActiveSupport::Notifications` (instrument and subscribe), model callbacks like `after_commit`, and Action Cable broadcasting. Callbacks hide control flow, run in surprising order, fire in contexts like imports where you did not want them, and make models harder to test. Keep critical side effects explicit in service objects when possible.
*Follow-up:* Why `after_commit` rather than `after_save` for enqueuing jobs? So the job never runs before the transaction commits and sees missing data.

**Q8. How are background jobs an example of the Command pattern?**
A job encapsulates a request (its class plus serialized arguments) as an object that can be queued, scheduled, retried and logged independently of the code that created it. Because delivery is at least once, commands must be idempotent and take ids rather than live objects.
*Follow-up:* What makes a job idempotent? Upserts keyed by external ids, checks of current state before acting, and unique constraints.

**Q9. What is middleware, and how does it relate to Chain of Responsibility?**
Middleware is a chain of handlers that each receive a request, may act on it or stop it, and pass it to the next link. Rack middleware (`call(env)` returning status, headers and body), Express `(req, res, next)` and Hono `app.use` all follow this pattern for logging, authentication, CORS, rate limiting and error handling.
*Follow-up:* What happens if a middleware never calls next? The request stops there, which is how authentication middleware rejects requests.

**Q10. Repository pattern vs the Active Record pattern?**
In Active Record, a model object is both the domain object and its persistence (`listing.save`), which is quick and productive. A repository separates domain objects from storage behind a collection-like interface, which helps when storage is complex or swappable and makes testing with in-memory repositories easy, at the cost of more code.
*Follow-up:* Which does Drizzle encourage? Query-builder functions over schema objects; teams often wrap them in repository or service modules.

**Q11. Observer vs Pub/Sub?**
In Observer, subscribers register directly with the subject, usually in the same process. In Pub/Sub, publishers and subscribers communicate through a broker or channel (Redis, Action Cable, a message queue) and do not know about each other, which enables cross-process and distributed communication.
*Follow-up:* What delivery guarantees does Redis pub/sub give? Fire-and-forget: subscribers that are offline miss messages, unlike durable queues.
