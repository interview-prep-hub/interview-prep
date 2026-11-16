# ASP.NET Core MVC

> **Tier:** T3 (essentials) · **Related:** [C# fundamentals](../03-languages/csharp/01-fundamentals.md), [C# advanced](../03-languages/csharp/02-advanced.md), [EF Core and Dapper](02-entity-framework-and-dapper.md), [Blazor](03-blazor.md), [Rails routing and controllers](../05-ruby-on-rails/02-routing-and-controllers.md), [Authentication and authorization](../04-web-fundamentals/06-authentication-authorization.md)
> **On your resume:** ASP.NET Core MVC, from university work and the .NET side of your final-year project (Recognition and Demonstrations of Human Activities Leveraging LLMs).

## Quick revision
- ASP.NET Core is cross-platform and open source; apps run on the built-in **Kestrel** server, often behind IIS, Nginx or a cloud load balancer. One framework covers MVC, Razor Pages, Web APIs, minimal APIs, Blazor and SignalR.
- `Program.cs` (minimal hosting, .NET 6+): `WebApplication.CreateBuilder` -> register services -> `builder.Build()` -> add middleware -> map endpoints -> `app.Run()`.
- **Middleware** form a pipeline: each piece acts before and after calling `next`, or short-circuits. Order matters: exception handler, HTTPS, static files, routing, CORS, authentication, authorization, endpoints.
- Built-in DI lifetimes: **Transient** (new per resolution), **Scoped** (one per request), **Singleton** (one per app). A singleton must never capture a scoped service (captive dependency).
- Controllers return `IActionResult` (`View()`, `Ok()`, `NotFound()`, `RedirectToAction()`) or `ActionResult<T>` in APIs. Model binding reads route, query, form and body (`[FromBody]` for JSON); data annotations fill `ModelState`.
- Razor views (`.cshtml`) use `@model`, layouts (`@RenderBody()`), partials and tag helpers (`asp-for`, `asp-action`); output is HTML-encoded by default. Routing: conventional (`{controller=Home}/{action=Index}/{id?}`) for MVC sites, attribute routing (`[Route]`, `[HttpGet("{id}")]`) for APIs. Filters wrap MVC actions; middleware wraps every request.
- Configuration layers (later wins): `appsettings.json`, `appsettings.{Environment}.json`, user secrets (Development), environment variables, command line. Bind sections with `IOptions<T>`.
- Auth: cookies for server-rendered apps, JWT bearer tokens for APIs and mobile clients; `[Authorize]` with roles or policies. 401 = not authenticated, 403 = authenticated but forbidden.

## 1. Program.cs and the middleware pipeline

```csharp
var builder = WebApplication.CreateBuilder(args);       // config, logging, DI container, Kestrel
builder.Services.AddControllersWithViews();              // 1. register services
builder.Services.AddDbContext<ShopDbContext>(o =>
    o.UseNpgsql(builder.Configuration.GetConnectionString("Default")));   // Npgsql EF Core provider package
builder.Services.AddScoped<IOrderService, OrderService>();

var app = builder.Build();
if (!app.Environment.IsDevelopment())                    // 2. middleware, in order
{
    app.UseExceptionHandler("/Home/Error");              // first, so it catches everything after it
    app.UseHsts();
}
app.UseHttpsRedirection();
app.UseStaticFiles();                                    // serves wwwroot files and short-circuits
app.UseRouting();                                        // selects the endpoint
app.UseAuthentication();                                 // who are you?
app.UseAuthorization();                                  // may you? (reads the endpoint's [Authorize])

app.MapControllerRoute("default", "{controller=Home}/{action=Index}/{id?}");   // 3. endpoints
app.Run();
```

Each middleware gets the `HttpContext` and a `next` delegate; it can run code before and after `await next(context)` or answer directly, and responses travel back through the same components in reverse, just like Rack middleware in Rails. Order is behavior: the exception handler must wrap everything, static files come early so they skip auth and MVC, `UseRouting` must precede authorization so `[Authorize]` metadata is known, and authentication must precede authorization (swap them and every protected request looks anonymous, returning 401). `UseCors` sits after routing and before authorization. Recent templates may call `app.MapStaticAssets()` instead of `UseStaticFiles()`. Custom middleware can be inline, as below, or a reusable class with a constructor taking `RequestDelegate next` and an `InvokeAsync(HttpContext context)` method, registered with `app.UseMiddleware<RequestIdMiddleware>()`.

```csharp
app.Use(async (context, next) =>                         // inline custom middleware: time every request
{
    var sw = System.Diagnostics.Stopwatch.StartNew();
    await next(context);                                 // run the rest of the pipeline
    app.Logger.LogInformation("{Path} took {Ms} ms", context.Request.Path, sw.ElapsedMilliseconds);
});
```

## 2. Dependency injection and lifetimes

```csharp
builder.Services.AddTransient<IPriceFormatter, PriceFormatter>();   // new instance per resolution
builder.Services.AddScoped<IOrderService, OrderService>();          // one per HTTP request
builder.Services.AddSingleton<IClock, UtcClock>();                  // one for the app's lifetime
public class OrdersController(IOrderService orders) : Controller    // constructor injection
{
    public async Task<IActionResult> Index() => View(await orders.ListAsync());
}
```

| Lifetime | Instance | Typical use |
|---|---|---|
| Transient | new on every resolution | lightweight, stateless helpers |
| Scoped | one per request (scope) | `DbContext`, unit of work, current-user context |
| Singleton | one for the whole app | caches, configuration, `IHttpClientFactory`; must be thread-safe |

**Captive dependency:** a singleton that takes a scoped service in its constructor keeps that first instance forever, for example one `DbContext` shared by every request and thread, causing stale data and concurrency exceptions. In Development the host validates scopes and fails at startup. Fix it by making the consumer scoped, or by injecting `IServiceScopeFactory` and calling `CreateScope()` per unit of work. Background services (`BackgroundService`) are singletons, so they must use this pattern to get a fresh `DbContext` on each run.

## 3. Controllers, actions, model binding and validation

```csharp
public class ProductsController(ShopDbContext db) : Controller
{
    public async Task<IActionResult> Index() => View(await db.Products.OrderBy(p => p.Name).ToListAsync());

    [HttpPost, ValidateAntiForgeryToken]
    public async Task<IActionResult> Create(ProductForm form)        // bound from posted form fields
    {
        if (!ModelState.IsValid) return View(form);                   // redisplay with errors
        db.Products.Add(new Product { Name = form.Name, Price = form.Price });
        await db.SaveChangesAsync();
        return RedirectToAction(nameof(Index));                       // Post/Redirect/Get
    }
}

public class ProductForm                                              // view model: only bindable fields
{
    [Required, StringLength(100)] public string Name { get; set; } = "";
    [Range(0.01, 100000)] public decimal Price { get; set; }
}
```

| Helper | Response |
|---|---|
| `View(model)`, `PartialView()` | 200 with HTML |
| `Ok(obj)`, `Json(obj)` | 200 with JSON |
| `CreatedAtAction(nameof(Get), new { id }, obj)` | 201 with a `Location` header |
| `NoContent()`, `RedirectToAction("Index")` | 204, 302 |
| `BadRequest(errors)`, `NotFound()`, `Unauthorized()`, `Forbid()` | 400, 404, 401, 403 |

- **Binding sources:** `[FromRoute]`, `[FromQuery]`, `[FromForm]`, `[FromBody]` (JSON, at most one per action), `[FromHeader]`, `[FromServices]`. Without attributes MVC looks in form fields, route values, then the query string; `[ApiController]` infers `[FromBody]` for complex types.
- **Overposting:** bind to view models or DTOs, never directly to entities, so a client cannot set fields like `IsAdmin` (ASP.NET's version of Rails strong parameters). **Validation:** annotations (`[Required]`, `[StringLength]`, `[Range]`, `[EmailAddress]`) run during binding and fill `ModelState`. MVC actions must check `ModelState.IsValid`; `[ApiController]` returns a 400 `ProblemDetails` response automatically.

## 4. Razor views, layouts, partials and tag helpers

```html
@* Views/Products/Create.cshtml *@
@model ProductForm
<form asp-action="Create" method="post">         @* the form tag helper adds the anti-forgery token *@
    <input asp-for="Name" />                      @* generates name, id, value and validation attributes *@
    <span asp-validation-for="Name"></span>
    <input asp-for="Price" />
    <button type="submit">Save</button>
</form>
<a asp-action="Index">Back to list</a>
```

- `@expr` output is HTML-encoded (XSS protection); `@Html.Raw` opts out. `@{ }` holds C# code, and `@if`/`@foreach` mix logic with markup.
- `Views/Shared/_Layout.cshtml` wraps every page through `@RenderBody()` (plus optional `@RenderSection("Scripts", required: false)`); `_ViewStart.cshtml` chooses the layout and `_ViewImports.cshtml` holds shared `@using` and `@addTagHelper` lines. Partials (`<partial name="_Row" model="p" />`) reuse markup; view components add their own logic. `TempData` survives one redirect (flash messages); `ViewData`/`ViewBag` pass loose values to a view.

## 5. Routing, Web API controllers and minimal APIs

```csharp
[ApiController]
[Route("api/[controller]")]                                   // attribute routing: api/orders
public class OrdersController(IOrderService orders) : ControllerBase
{
    [HttpGet("{id:int}")]                                     // GET api/orders/5, int constraint
    public async Task<ActionResult<OrderDto>> Get(int id)
    {
        var order = await orders.FindAsync(id);
        return order is null ? NotFound() : order;            // ActionResult<T> accepts either
    }
    [HttpPost]
    public async Task<IActionResult> Create(CreateOrderDto dto)   // bound from the JSON body
    {
        var created = await orders.CreateAsync(dto);
        return CreatedAtAction(nameof(Get), new { id = created.Id }, created);
    }
}

// the same read endpoint as a minimal API (.NET 6+)
app.MapGet("/api/orders/{id:int}", async (int id, IOrderService orders) =>
    await orders.FindAsync(id) is OrderDto o ? Results.Ok(o) : Results.NotFound());
```

- **Conventional routing** maps URLs to controllers by pattern (`/Products/Edit/5`) and suits HTML apps. **Attribute routing** puts routes on controllers and actions, with tokens (`[controller]`), constraints (`{id:int}`) and optional segments (`{id?}`); it is the norm for APIs. Rails' `resources :orders` has no one-line equivalent, so API verbs are spelled out with attributes.
- **Controllers vs minimal APIs:** both share routing, DI, middleware and auth. Controllers bring classes, filters and `[ApiController]` conventions, which suit large APIs and MVC-minded teams. Minimal APIs map lambdas in `Program.cs` (grouped with `MapGroup`, with endpoint filters and `TypedResults`) for small services with less ceremony. Built-in validation for minimal APIs arrived only in recent versions, so check the docs for yours.

## 6. Filters

| Filter | Runs | Typical use |
|---|---|---|
| Authorization | first, before model binding | `[Authorize]`, custom access checks |
| Resource | around model binding | caching, short-circuiting |
| Action | before and after the action method (sees arguments) | logging, argument checks |
| Exception | when an action or filter throws | turning exceptions into error responses |
| Result | around executing the result | headers, response wrapping |

A filter is a class implementing an interface such as `IActionFilter` (with `OnActionExecuting` and `OnActionExecuted`), registered globally with `AddControllersWithViews(o => o.Filters.Add<LogActionFilter>())` or per controller with attributes. Filters run only for MVC and API actions and can see arguments and results; middleware runs for every request but knows nothing about MVC. Rails equivalents: `before_action`, `around_action`, `rescue_from`.

## 7. Configuration, options and logging

```csharp
// appsettings.json: { "Smtp": { "Host": "smtp.example.com", "Port": 587 } }
public class SmtpOptions { public string Host { get; set; } = ""; public int Port { get; set; } }
builder.Services.Configure<SmtpOptions>(builder.Configuration.GetSection("Smtp"));

public class EmailSender(IOptions<SmtpOptions> options, ILogger<EmailSender> logger)
{
    private readonly SmtpOptions _smtp = options.Value;              // strongly typed settings
    public void Send(int orderId) =>
        logger.LogInformation("Mail for order {OrderId} via {Host}", orderId, _smtp.Host);   // structured fields
}
```

- Sources override in order: `appsettings.json`, `appsettings.{Environment}.json`, user secrets (Development only), environment variables (`Smtp__Host` for nested keys), command-line arguments. `ASPNETCORE_ENVIRONMENT` selects Development, Staging or Production.
- `IOptions<T>` reads once; `IOptionsSnapshot<T>` re-reads per request; `IOptionsMonitor<T>` reports changes. Keep secrets out of `appsettings.json`: `dotnet user-secrets` locally, environment variables or a vault in production. Log with message templates, not interpolation: values stay searchable fields and formatting is skipped when the level is off. Levels (Trace to Critical) are set per category under `Logging:LogLevel`; Serilog or NLog can plug in as providers.

## 8. Authentication and authorization

```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)   // JwtBearer NuGet package
    .AddJwtBearer(o => { o.Authority = "https://login.example.com"; o.Audience = "shop-api"; });   // issuer, audience
builder.Services.AddAuthorization(o =>
    o.AddPolicy("CanRefund", p => p.RequireRole("Admin").RequireClaim("department", "billing")));

[Authorize]                                             // any signed-in user
public class AccountController : Controller
{
    [AllowAnonymous] public IActionResult Login() => View();
    [Authorize(Roles = "Admin")] public IActionResult Users() => View();
    [Authorize(Policy = "CanRefund")] public IActionResult Refund(int id) => View();
}
```

Server-rendered MVC apps usually use cookie authentication (`AddCookie()`, then `HttpContext.SignInAsync(principal)`), and ASP.NET Core Identity adds users, password hashing and login UI. APIs for SPAs and mobile apps use JWT bearer tokens. Identity is claims-based (`User.IsInRole("Admin")`, `User.FindFirst(ClaimTypes.NameIdentifier)`); policies combine requirements, and resource-based checks ("may this user edit this order?") go through `IAuthorizationService`.

## 9. Rails to ASP.NET Core mapping

| Rails | ASP.NET Core |
|---|---|
| `config/routes.rb`, `resources :orders` | conventional routes in `Program.cs`, attribute routes |
| `ApplicationController` actions, `render`, `redirect_to` | `Controller` actions, `View()`, `RedirectToAction()` |
| strong parameters, model validations | view models or DTOs, data annotations and `ModelState` |
| ERB views, layouts with `yield`, partials | Razor `.cshtml`, `_Layout` with `@RenderBody()`, partials, tag helpers |
| `before_action`, `rescue_from`, Rack middleware | filters, exception-handling middleware, ASP.NET Core middleware |
| ActiveRecord, `rails db:migrate` | Entity Framework Core, `dotnet ef database update` |
| credentials and `ENV` | `appsettings.json`, user secrets, environment variables |
| Devise or `has_secure_password` | ASP.NET Core Identity |
| Sidekiq jobs; Bundler and RubyGems; Puma | hosted services (`BackgroundService`) or Hangfire; NuGet; Kestrel |

## Common pitfalls
- Middleware in the wrong order (authorization before authentication, auth after the endpoints).
- Captive dependencies: scoped services, especially `DbContext`, injected into singletons or hosted services.
- Binding entities directly (overposting), or forgetting `ModelState.IsValid` in non-API controllers.
- Blocking on async code with `.Result`; creating an `HttpClient` per request instead of using `IHttpClientFactory`.
- Secrets committed in `appsettings.json`; returning EF entities with navigation cycles straight from APIs.

## How to talk about this in an interview
- Your daily framework, Rails, is also MVC with middleware, so lead with the mapping: you can explain pipelines, DI lifetimes and model binding by comparison, then show the ASP.NET specifics.
- Be honest about depth: ASP.NET Core comes from university work and your final-year project, not production. For a .NET role you would ramp up on hosting (IIS, Azure, containers), observability and the current version's new features.
- > **Fill in:** what the .NET side of your final-year project did (MVC web UI, API endpoints receiving smartwatch, phone and glasses data, calls to the Python model service), which .NET version and database it used, and how authentication worked.

## Interview questions

**Q1. What is the middleware pipeline, and why does order matter?**
Middleware are components chained around every request: each can inspect or modify the request, call the next component, then act on the response, or short-circuit. Order defines behavior: the exception handler must come first to catch everything, static files early to skip unnecessary work, routing before authorization so endpoint metadata is available, and authentication before authorization.
*Follow-up:* Middleware vs filters? Middleware sees all requests and no MVC details; filters run only around controller actions and can see arguments and results.

**Q2. Explain transient, scoped and singleton lifetimes. What is a captive dependency?**
Transient creates a new instance per resolution, scoped one per request, singleton one for the app's lifetime. A captive dependency is a longer-lived service holding a shorter-lived one, typically a singleton keeping a scoped `DbContext` alive across requests and threads. Avoid it by matching lifetimes or creating scopes with `IServiceScopeFactory`.

**Q3. How does model binding work? `[FromBody]` vs `[FromQuery]`?**
Model binding builds action parameters from request data using value providers: form fields, route values, the query string, headers and the body. `[FromQuery]` reads `?page=2`-style values; `[FromBody]` deserializes the request body (usually JSON) with an input formatter, and only one parameter can use it. `[ApiController]` infers these sources for you.

**Q4. How do you validate input in ASP.NET Core MVC?**
Put data annotations on a view model or DTO (`[Required]`, `[StringLength]`, `[Range]`), let binding populate `ModelState`, check `ModelState.IsValid` and redisplay the view with errors; tag helpers render the messages and client-side validation attributes. API controllers with `[ApiController]` return 400 automatically. Complex rules can implement `IValidatableObject` or use a library such as FluentValidation.

**Q5. Conventional vs attribute routing?**
Conventional routing defines URL patterns centrally (`{controller}/{action}/{id?}`), which suits server-rendered sites with predictable URLs. Attribute routing declares routes on controllers and actions (`[Route("api/orders")]`, `[HttpGet("{id:int}")]`), giving precise control over REST APIs. Apps can mix both.

**Q6. How do authentication and authorization work? Roles vs policies?**
Authentication middleware establishes the user's identity (a `ClaimsPrincipal`) from a cookie or a bearer token; authorization middleware then evaluates `[Authorize]` requirements for the selected endpoint. Roles are simple membership checks; policies are named sets of requirements (roles, claims, custom handlers) registered once and reused, which keeps rules out of controllers.

**Q7. How does configuration work, and what is the options pattern?**
Configuration merges providers in order (JSON files, environment-specific JSON, user secrets in Development, environment variables, command line), with later sources overriding earlier ones. The options pattern binds a section to a class (`services.Configure<SmtpOptions>(...)`) and injects `IOptions<SmtpOptions>`, giving typed, testable settings instead of magic strings.
