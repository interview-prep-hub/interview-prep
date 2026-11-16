# .NET Web Stack: Rapid-Fire Interview Questions

> How to use: cover the answer, say yours out loud, then check. Deeper explanations live in the topic files linked under each section. For C# language questions, see [C# rapid-fire](../03-languages/csharp/99-interview-questions.md).

## ASP.NET Core fundamentals and middleware (see [ASP.NET Core MVC](01-aspnet-core-mvc.md))

**Q1. What is ASP.NET Core, and how does it differ from classic ASP.NET?**
A cross-platform, open-source, high-performance rewrite that runs on modern .NET (Windows, Linux, macOS, containers) with built-in DI, configuration and logging. Classic ASP.NET (`System.Web`) is Windows- and IIS-only and tied to .NET Framework.

**Q2. What is Kestrel, and do you still need IIS or Nginx?**
Kestrel is the built-in cross-platform web server that every ASP.NET Core app runs on. It can face the internet directly, but production setups often put a reverse proxy or load balancer (IIS, Nginx, a cloud load balancer) in front for TLS termination, routing and extra protection.

**Q3. What does `WebApplication.CreateBuilder(args)` set up?**
Configuration (JSON files, environment variables, user secrets in Development, command line), logging providers, the DI container and Kestrel. You register services on `builder.Services`, then `builder.Build()` creates the app on which you add middleware and endpoints.

**Q4. What is middleware?**
A component in the request pipeline that receives the `HttpContext` and a `next` delegate. It can act before and after calling the rest of the pipeline, or short-circuit by writing the response itself.

**Q5. What is the difference between `app.Use`, `app.Run` and `app.Map`?**
`Use` adds middleware that can call the next component; `Run` adds terminal middleware that never calls `next`; `Map` branches the pipeline for a path prefix. `MapGet`, `MapControllerRoute` and friends map endpoints rather than middleware.

**Q6. Why must `UseAuthentication` come before `UseAuthorization`?**
Authentication turns the cookie or token into `HttpContext.User`; authorization then checks that user against the endpoint's requirements. Reversed, authorization sees an anonymous user and rejects every protected request.

**Q7. Where does `UseRouting` belong?**
Before any middleware that needs to know the selected endpoint, such as CORS and authorization, which read endpoint metadata like `[Authorize]`. `WebApplication` adds routing automatically if you omit it, but calling it explicitly controls its position.

**Q8. How do you handle exceptions globally?**
Use the exception handler middleware (`app.UseExceptionHandler("/Error")`) in production and the developer exception page in Development. .NET 8 added `IExceptionHandler` for custom handlers, and `AddProblemDetails()` produces standard JSON error bodies for APIs.

**Q9. What is `HttpContext`?**
The per-request object holding `Request`, `Response`, `User`, `Items`, `RequestServices` and `RequestAborted`. Controllers expose it as `HttpContext`; elsewhere `IHttpContextAccessor` provides it, but passing data explicitly is cleaner.

## Dependency injection and configuration (see [ASP.NET Core MVC](01-aspnet-core-mvc.md))

**Q10. What are the three service lifetimes?**
Transient: a new instance every time it is resolved. Scoped: one instance per scope, which in a web app means per request. Singleton: one instance for the application's lifetime, so it must be thread-safe.

**Q11. What is a captive dependency, and how does ASP.NET Core catch it?**
A longer-lived service holding a shorter-lived one, typically a singleton capturing a scoped `DbContext`, which then leaks across requests and threads. In Development the container validates scopes and throws when a singleton depends on a scoped service.

**Q12. How do you use a scoped service from a singleton or a background service?**
Inject `IServiceScopeFactory`, create a scope per unit of work with `using var scope = factory.CreateScope();`, and resolve the service from `scope.ServiceProvider`. Hosted services are singletons, so this is the standard pattern for background jobs.

**Q13. What is `IHttpClientFactory`, and why use it?**
It manages pooled `HttpMessageHandler` instances so you avoid the socket exhaustion caused by creating `HttpClient` per request and the stale DNS of one static client forever. It also supports named and typed clients and plugs in resilience handlers such as retries.

**Q14. How do you register several implementations of one interface?**
Register each one: injecting `IEnumerable<INotifier>` gives all of them, while injecting a single `INotifier` gives the last registration. .NET 8 added keyed services (`AddKeyedScoped<INotifier, SmsNotifier>("sms")` with `[FromKeyedServices("sms")]`).

**Q15. `IOptions<T>` vs `IOptionsSnapshot<T>` vs `IOptionsMonitor<T>`?**
`IOptions<T>` is a singleton read once at startup. `IOptionsSnapshot<T>` is scoped and re-reads configuration per request. `IOptionsMonitor<T>` is a singleton that always returns current values and raises change notifications.

**Q16. In what order are configuration sources applied?**
`appsettings.json`, then `appsettings.{Environment}.json`, then user secrets (Development only), then environment variables, then command-line arguments; later sources override earlier ones. Nested keys use `:` in code and `__` in environment variable names (`Smtp__Host`).

**Q17. How do you keep secrets out of source control?**
`dotnet user-secrets` during development (stored outside the project folder), and environment variables or a secret manager such as Azure Key Vault in production. Commit only non-secret defaults in `appsettings.json`.

**Q18. How is the environment chosen, and what changes in Development?**
From `ASPNETCORE_ENVIRONMENT` (or `DOTNET_ENVIRONMENT`), with Production as the default. Development enables the developer exception page, user secrets and DI scope validation; code checks it with `app.Environment.IsDevelopment()`.

## MVC, Razor, routing, validation and filters (see [ASP.NET Core MVC](01-aspnet-core-mvc.md))

**Q19. How does MVC work in ASP.NET Core?**
Routing selects a controller action, model binding builds its parameters, the action runs business logic (usually through injected services) and returns a result: a view rendered from a model, JSON, a redirect or a status code. Same pattern as Rails, with Razor instead of ERB.

**Q20. `Controller` vs `ControllerBase`?**
`ControllerBase` provides API essentials (`Ok`, `NotFound`, `ModelState`, `User`). `Controller` inherits from it and adds view support (`View`, `PartialView`, `ViewBag`, `TempData`). API controllers derive from `ControllerBase`.

**Q21. `IActionResult` vs `ActionResult<T>`?**
`IActionResult` can be any result but tells tools nothing about the success payload. `ActionResult<T>` lets an action return either a `T` (serialized with 200) or another result such as `NotFound()`, and it documents the response type for OpenAPI.

**Q22. What does `[ApiController]` do?**
It requires attribute routing, returns a 400 `ProblemDetails` response automatically when `ModelState` is invalid, infers binding sources (`[FromBody]` for complex types, route and query for simple ones), and formats error status codes as problem details.

**Q23. Without binding attributes, where does MVC look for values?**
In form fields, then route values, then the query string (plus uploaded files). With `[ApiController]`, complex types come from the body. Explicit attributes (`[FromQuery]`, `[FromRoute]`, `[FromBody]`, `[FromHeader]`) remove ambiguity.

**Q24. What is overposting, and how do you prevent it?**
A client posting extra fields (such as `IsAdmin=true`) that bind onto an entity. Prevent it by binding to view models or DTOs that contain only editable fields and mapping them to entities yourself, the equivalent of Rails strong parameters.

**Q25. How does validation work, and how do you add custom rules?**
Data annotations on the model are checked during binding and fill `ModelState`; MVC actions test `ModelState.IsValid`, API controllers get automatic 400s. For custom rules write a `ValidationAttribute`, implement `IValidatableObject`, or use a library such as FluentValidation.

**Q26. Tag helpers vs HTML helpers?**
Tag helpers are server-side attributes on normal HTML elements (`<input asp-for="Email" />`), so markup stays readable and designer-friendly. HTML helpers are C# method calls (`@Html.TextBoxFor(m => m.Email)`). Both generate names, ids, values and validation attributes from the model.

**Q27. What are `_Layout.cshtml`, `_ViewStart.cshtml` and `_ViewImports.cshtml`?**
The layout is the shared page shell that renders each view through `@RenderBody()`. `_ViewStart` runs before every view and usually sets the layout. `_ViewImports` holds directives shared by all views in its folder, such as `@using` and `@addTagHelper`.

**Q28. `ViewData` vs `ViewBag` vs `TempData`?**
`ViewData` is a string-keyed dictionary and `ViewBag` a dynamic wrapper over the same data, both living for one request. `TempData` survives a redirect (backed by a cookie or session), which makes it right for flash messages after Post/Redirect/Get.

**Q29. Partial view vs view component?**
A partial view is reusable markup rendered with data you pass in. A view component is a class with its own logic and dependencies plus a view, suited to self-contained widgets such as a cart summary or a navigation menu.

**Q30. What are route constraints?**
Rules inside route templates that a segment must satisfy, such as `{id:int}`, `{slug:alpha}`, `{id:min(1)}` or `{id?}` for optional. They disambiguate routes, and when a constraint fails the route simply does not match (often ending in a 404).

**Q31. What filter types exist, and in what order do they run?**
Authorization filters first, then resource filters, then model binding, then action filters around the action, then result filters around result execution; exception filters handle unhandled exceptions from actions and other filters. Within a type, global filters wrap controller-level filters, which wrap action-level filters.

**Q32. Filters vs middleware?**
Middleware runs for every request, including static files, and knows nothing about MVC. Filters run only around controller actions and can see the chosen action, its arguments, `ModelState` and the result. Cross-cutting HTTP concerns go in middleware; MVC-specific concerns go in filters.

**Q33. How does anti-forgery (CSRF) protection work?**
The form tag helper emits a hidden token, a matching value goes in a cookie, and `[ValidateAntiForgeryToken]` (or the global `AutoValidateAntiforgeryToken` filter) rejects posts where they do not match. APIs authenticated with bearer tokens do not need it, since browsers do not attach those automatically.

## Web APIs, authentication and logging (see [ASP.NET Core MVC](01-aspnet-core-mvc.md))

**Q34. Minimal APIs vs controllers?**
Minimal APIs map lambdas to routes in `Program.cs` (organized with `MapGroup`, endpoint filters and `TypedResults`), with less ceremony for small services. Controllers group endpoints into classes with filters and conventions, which scale well for large APIs. Both share routing, DI, middleware and auth.

**Q35. How do you document and version an API?**
Generate an OpenAPI document (Swashbuckle, or the built-in OpenAPI support in recent versions; check your version's docs) and serve a UI for exploring it. Version through a URL segment, a header or a query parameter, typically with the API versioning library (the `Asp.Versioning` packages).

**Q36. Cookie authentication vs JWT bearer authentication?**
Cookies suit server-rendered apps on one domain: the browser sends them automatically, they can be `HttpOnly`, and they need CSRF protection. JWT bearer tokens suit APIs consumed by SPAs, mobile apps and other services: sent explicitly in the `Authorization` header and validated without server session state, but hard to revoke before expiry.

**Q37. Roles vs claims vs policies?**
A role is a coarse group (`[Authorize(Roles = "Admin")]`). A claim is any key-value fact about the user (email, department, tenant). A policy is a named rule built from requirements (roles, claims, custom handlers) and applied with `[Authorize(Policy = "CanRefund")]`, keeping authorization logic in one place.

**Q38. When does ASP.NET Core return 401 vs 403?**
401 Unauthorized when no valid identity is present (the scheme issues a challenge: a login redirect for cookies, a `WWW-Authenticate` header for JWT). 403 Forbidden when the user is authenticated but fails the authorization requirements.

**Q39. What is ASP.NET Core Identity?**
A membership system providing user and role storage (usually through EF Core), password hashing, lockout, two-factor authentication, external logins and default UI pages. It plays the role Devise plays in Rails.

**Q40. How do you configure CORS?**
`builder.Services.AddCors(o => o.AddPolicy("web", p => p.WithOrigins("https://app.example.com").AllowAnyHeader().AllowAnyMethod()));` and `app.UseCors("web")` between routing and authorization. Never combine allowing any origin with credentials.

**Q41. How does logging work, and what is structured logging?**
Inject `ILogger<T>` and call `LogInformation`, `LogWarning` and so on; providers (console, debug, Serilog, cloud sinks) decide where entries go, and levels are configured per category. Message templates such as `"Order {OrderId} shipped"` keep values as named fields that can be searched and filtered.

**Q42. How do you return consistent error responses from an API?**
Use problem details (the standard JSON error format from RFC 7807, since updated by RFC 9457): `AddProblemDetails()`, automatic validation responses from `[ApiController]`, and an exception handler that maps known exceptions to status codes.

## Entity Framework Core (see [EF Core and Dapper](02-entity-framework-and-dapper.md))

**Q43. What is a `DbContext`?**
A session with the database: it exposes `DbSet<T>` properties for querying, tracks loaded entities, and saves changes as one unit of work. It is lightweight, not thread-safe, and meant to be short-lived.

**Q44. Code-first vs database-first?**
Code-first: you write entity classes and EF migrations create and evolve the schema. Database-first: you scaffold entities from an existing database with `dotnet ef dbcontext scaffold`. New projects usually go code-first; legacy databases go database-first.

**Q45. Conventions vs data annotations vs Fluent API?**
Conventions infer keys, foreign keys and column types with no configuration. Data annotations (`[Required]`, `[MaxLength]`) configure inline on the entity. The Fluent API in `OnModelCreating` covers everything (composite keys, indexes, relationships, value conversions) and overrides annotations.

**Q46. How do migrations work?**
`dotnet ef migrations add Name` diffs the model against the last snapshot and generates `Up`/`Down` code; `dotnet ef database update` applies pending migrations and records them in `__EFMigrationsHistory`. For production, generate idempotent scripts or migration bundles and review them.

**Q47. What does `SaveChanges` do?**
It runs change detection, orders the needed INSERT, UPDATE and DELETE statements (respecting relationships), executes them in a single transaction, updates generated keys and concurrency tokens, and returns the number of affected rows.

**Q48. Tracking vs no-tracking queries?**
Tracking queries snapshot entities so changes can be saved later, and resolve each key to one instance. `AsNoTracking()` skips the snapshots, which is faster and uses less memory: the right choice for read-only lists, API responses and reports.

**Q49. Eager vs explicit vs lazy loading?**
Eager (`Include`/`ThenInclude`) loads related data in the same query. Explicit (`Entry(x).Collection(...).LoadAsync()`) loads it later on purpose. Lazy (proxies plus `virtual` navigations) loads it on first access, which is convenient but causes N+1 queries in loops.

**Q50. How do you detect N+1 queries in EF Core?**
Log the generated SQL (`optionsBuilder.LogTo(Console.WriteLine)` or the `Microsoft.EntityFrameworkCore.Database.Command` log category) and look for the same query repeated per row; profilers help too. Fix with `Include`, a `Select` projection, or batching.

**Q51. What is a cartesian explosion, and how does `AsSplitQuery` help?**
Including several collections in one query yields a JOIN whose row count is the product of the collection sizes, duplicating data massively. `AsSplitQuery()` loads each collection with its own SQL query and stitches the results, at the cost of extra round trips.

**Q52. What is the danger of calling `ToList()` too early?**
Everything after `ToList()` runs in memory, so filtering, sorting and paging no longer happen in SQL: the whole table is loaded first. Keep the query `IQueryable` until the end, and check the SQL with `ToQueryString()`.

**Q53. How do you run raw SQL safely?**
`FromSql($"... {value}")` and `ExecuteSql($"...")` (EF Core 7+) convert interpolated values into parameters; `FromSqlRaw` with `{0}` placeholders is also parameterized. Never concatenate user input into the SQL string.

**Q54. How do transactions work in EF Core?**
Each `SaveChanges` call is atomic on its own. For several calls, or for mixing in raw SQL or Dapper, use `await using var tx = await db.Database.BeginTransactionAsync();` and `CommitAsync()`; disposing without committing rolls back.

**Q55. How does optimistic concurrency work?**
A concurrency token (`[Timestamp]` row version or `IsConcurrencyToken()`) is added to the WHERE clause of UPDATE and DELETE statements; if the row changed since it was read, no rows match and EF throws `DbUpdateConcurrencyException`, which you resolve by reloading, merging or asking the user.

**Q56. What are `ExecuteUpdate` and `ExecuteDelete`?**
EF Core 7 methods that run a single set-based UPDATE or DELETE from a LINQ filter without loading entities. They are fast for bulk changes, but they bypass the change tracker, so entities already loaded in the context go stale.

**Q57. Why is `DbContext` registered as scoped, and why is it not thread-safe?**
It is a unit of work holding a connection and tracked entities, so one per request fits naturally. It is not designed for concurrent access: two queries running in parallel on one context throw an exception. Background jobs create a scope or use `IDbContextFactory<T>`.

**Q58. How do you speed up slow EF Core queries?**
Project only the needed columns, use `AsNoTracking`, add indexes (`HasIndex`), remove N+1 patterns, paginate, split heavy includes, consider compiled queries for hot paths, and read the generated SQL (`ToQueryString()`, logging) and its execution plan. Drop to raw SQL or Dapper when the LINQ cannot express the query efficiently.

## Dapper and data-access patterns (see [EF Core and Dapper](02-entity-framework-and-dapper.md))

**Q59. What is Dapper?**
A micro-ORM from the Stack Overflow team: a set of extension methods on `IDbConnection` (`Query<T>`, `QueryFirstOrDefault<T>`, `Execute`, `QueryMultiple`) that run your SQL and map rows to objects by column name, with performance close to raw ADO.NET.

**Q60. How do you pass parameters in Dapper, and why does it matter?**
Write `@name` placeholders and pass an anonymous object or `DynamicParameters`: `conn.Query<Post>("... WHERE Id = @id", new { id })`. Values travel as real database parameters, which prevents SQL injection and lets the database reuse query plans. Dapper also expands `IN @ids` from a list.

**Q61. How does multi-mapping work, and what is `splitOn`?**
`Query<Post, Blog, Post>(sql, (p, b) => { p.Blog = b; return p; }, splitOn: "Id")` maps each joined row into several objects. `splitOn` names the column where the next object's columns begin (default `Id`); duplicated parents must be merged by hand, for example with a dictionary.

**Q62. What is `QueryMultiple`?**
It executes a batch containing several SELECT statements in one round trip and returns a grid reader; you call `Read<T>` or `ReadSingle<T>` in order, once per result set. Useful for loading a parent and its children together.

**Q63. How do transactions work with Dapper?**
Open the connection, call `BeginTransaction`, pass the transaction to every `Execute` or `Query` call (`transaction: tx`), then `Commit`. Disposing the transaction without committing rolls it back.

**Q64. EF Core vs Dapper?**
EF Core gives LINQ, change tracking, relationships and migrations, which means high productivity for CRUD and domain logic. Dapper gives full SQL control and minimal overhead, ideal for complex reports and hot read paths. Many apps use EF Core for writes and Dapper for heavy reads.

**Q65. Should you wrap EF Core in a generic repository and unit of work?**
Often not: `DbSet<T>` already is a repository and `DbContext` already is a unit of work, so a generic wrapper adds a layer while hiding features like `Include`, projections and `AsNoTracking`. Specific, intention-revealing query services or repositories are worth it when they isolate the domain layer or centralize complex queries.

**Q66. How would you use EF Core and Dapper together?**
Register the `DbContext` as usual, and for Dapper queries take its connection with `db.Database.GetDbConnection()`. Inside an EF transaction, pass `db.Database.CurrentTransaction?.GetDbTransaction()` to Dapper so both run atomically.

## Blazor (see [Blazor](03-blazor.md))

**Q67. What is Blazor?**
A framework for building interactive web UIs with Razor components written in C# instead of JavaScript. Components can run on the server (updating the browser over SignalR) or in the browser on WebAssembly.

**Q68. Blazor Server vs Blazor WebAssembly?**
Server: tiny download and direct access to server resources, but every interaction is a network round trip, the server holds per-user state, and a dropped connection breaks the UI. WebAssembly: runs in the browser with offline support and static hosting, but a larger first download, code visible to clients, and data access only through APIs.

**Q69. What is a circuit?**
The server-side session of a Blazor Server user: component instances, their state and the SignalR connection. Losing the connection triggers a reconnect attempt; if the circuit expires, the in-memory state is lost and the page must reload.

**Q70. What are the render modes in a .NET 8 Blazor Web App?**
Static SSR (HTML only, the default), Interactive Server, Interactive WebAssembly and Interactive Auto (Server first, then WebAssembly once the runtime is cached). They are set per component, per page, or for the whole app on the router.

**Q71. What is prerendering, and what problems can it cause?**
Interactive components first render as static HTML on the server for fast first paint and SEO, then become interactive. As a result `OnInitializedAsync` can run twice (duplicate API calls) and JS interop is unavailable during the prerender. Persist state with `PersistentComponentState` or disable prerendering for that component.

**Q72. How do Blazor components communicate?**
Parameters (`[Parameter]`) pass data down; `EventCallback<T>` parameters send events up; `RenderFragment` passes markup; cascading values provide data to a whole subtree; scoped state-container services share state across unrelated components.

**Q73. What are the main lifecycle methods?**
`OnInitialized(Async)` runs once (load data), `OnParametersSet(Async)` runs whenever parameters change, `OnAfterRender(Async)(firstRender)` runs after the DOM updates (JS interop), and `Dispose` runs on removal (unsubscribe, stop timers). `ShouldRender` can skip renders.

**Q74. When do you call `StateHasChanged`?**
When state changes outside Blazor's automatic triggers (event handlers, lifecycle methods, new parameters): timers, service events, background results. Call it through `InvokeAsync(StateHasChanged)` if you might be on another thread.

**Q75. How does `@bind` work?**
It creates two-way binding: it renders the field's value and updates the field on the element's `onchange` event (or `oninput` with `@bind:event="oninput"`). Input components use `@bind-Value`, and `@bind:after` runs code after the value is updated.

**Q76. How does JavaScript interop work?**
Inject `IJSRuntime` and call `InvokeAsync<T>("functionName", args)` or `InvokeVoidAsync`; JavaScript calls .NET through `DotNetObjectReference` or static `[JSInvokable]` methods. Call it from `OnAfterRenderAsync`, since the DOM does not exist earlier and interop is unavailable during prerendering.

## Spot the bug

**Q77. What is wrong with this pipeline?**
```csharp
app.UseRouting();
app.UseAuthorization();
app.UseAuthentication();
app.MapControllers();
```
Authorization runs before authentication has set `HttpContext.User`, so every `[Authorize]` endpoint sees an anonymous user and returns 401. Swap the two lines: `UseAuthentication()` then `UseAuthorization()`.

**Q78. What is wrong with this component?**
```csharp
protected override async Task OnInitializedAsync()
{
    var theme = await JS.InvokeAsync<string>("localStorage.getItem", "theme");
}
```
JS interop is not available while the component is prerendering on the server, so this throws (and the DOM is not ready yet anyway). Move the call into `OnAfterRenderAsync(bool firstRender)` and run it only when `firstRender` is true, then call `StateHasChanged()` if the result changes the UI.
