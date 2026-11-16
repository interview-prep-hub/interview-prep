# Entity Framework Core and Dapper

> **Tier:** T3 (essentials) · **Related:** [ASP.NET Core MVC](01-aspnet-core-mvc.md), [C# advanced (LINQ)](../03-languages/csharp/02-advanced.md), [ORMs compared](../12-databases/12-orms-compared.md), [N+1 in ActiveRecord](../05-ruby-on-rails/05-activerecord-querying-n-plus-1.md), [Transactions and concurrency](../12-databases/06-transactions-and-concurrency.md)
> **On your resume:** Entity Framework and Dapper (skills list), from university work and the .NET side of your final-year project. The trade-offs mirror ActiveRecord and Drizzle, which you use in production.

## Quick revision
- EF Core is a full ORM: `DbContext` = session + unit of work + change tracker; `DbSet<T>` = a table you query with LINQ, translated to SQL.
- Mapping: conventions (`Id` is the key, `BlogId` the foreign key) < data annotations (`[Required]`, `[MaxLength]`) < Fluent API in `OnModelCreating`, which wins over both.
- Migrations: `dotnet ef migrations add Name`, then `dotnet ef database update` (needs the `dotnet-ef` tool and the `Microsoft.EntityFrameworkCore.Design` package).
- Queried entities are **tracked**: `SaveChanges` detects changes and writes INSERT/UPDATE/DELETE in one transaction. Use `AsNoTracking()` for read-only queries.
- Related data: eager (`Include`/`ThenInclude`), explicit (`Entry(x).Collection(...).LoadAsync()`), lazy (opt-in proxies, the classic N+1 trap). `AsSplitQuery()` avoids cartesian explosions from several collection includes.
- Raw SQL: `FromSql($"... {value}")` turns interpolated values into parameters (EF Core 7+); never concatenate input into SQL.
- Optimistic concurrency: a concurrency token (`[Timestamp]`, `IsConcurrencyToken()`) makes conflicting updates throw `DbUpdateConcurrencyException`.
- `ExecuteUpdate`/`ExecuteDelete` (EF Core 7) run set-based SQL without loading entities or touching the change tracker.
- `DbContext` is not thread-safe and is registered as **scoped** (one per request); background work creates its own scope or uses `IDbContextFactory<T>`.
- Dapper is a micro-ORM: extension methods on `IDbConnection` (`Query<T>`, `QueryFirstOrDefault<T>`, `Execute`, `QueryMultiple`) that map rows to objects. You write the SQL; anonymous objects supply parameters, which prevents SQL injection.
- EF Core for CRUD, domain logic and migrations; Dapper for hot read paths, reports and hand-tuned SQL. Both can share one connection and transaction.

## 1. DbContext, DbSet and the model

```csharp
public class Blog
{
    public int Id { get; set; }                                     // convention: primary key
    [Required, MaxLength(200)] public string Title { get; set; } = "";
    public List<Post> Posts { get; set; } = new();                  // navigation: one-to-many
}

public class Post
{
    public int Id { get; set; }
    public string Body { get; set; } = "";
    public int BlogId { get; set; }                                 // convention: foreign key to Blog
    public Blog Blog { get; set; } = null!;
    public List<Comment> Comments { get; set; } = new();
    [Timestamp] public byte[] RowVersion { get; set; } = null!;     // concurrency token (SQL Server rowversion)
}

public class Comment { public int Id { get; set; } public int PostId { get; set; } public string Text { get; set; } = ""; }

public class BlogContext(DbContextOptions<BlogContext> options) : DbContext(options)
{
    public DbSet<Blog> Blogs => Set<Blog>();
    public DbSet<Post> Posts => Set<Post>();

    protected override void OnModelCreating(ModelBuilder b)
    {
        b.Entity<Blog>().HasIndex(x => x.Title).IsUnique();                      // Fluent API
        b.Entity<Post>().HasOne(p => p.Blog).WithMany(bl => bl.Posts)
                        .HasForeignKey(p => p.BlogId).OnDelete(DeleteBehavior.Cascade);
    }
}

// Program.cs (provider package: Microsoft.EntityFrameworkCore.SqlServer; PostgreSQL uses UseNpgsql)
builder.Services.AddDbContext<BlogContext>(o => o.UseSqlServer(builder.Configuration.GetConnectionString("Default")));
```

| Approach | Examples | Notes |
|---|---|---|
| Conventions | `Id` or `BlogId` key, `BlogId` foreign key, nullable reference types decide required columns | zero configuration |
| Data annotations | `[Key]`, `[Required]`, `[MaxLength]`, `[Column("name")]`, `[Timestamp]` | quick, but mixes persistence into entities |
| Fluent API | `HasIndex`, `HasOne().WithMany()`, `HasKey(x => new { x.A, x.B })`, `HasConversion` | most powerful; overrides annotations |

How concurrency tokens map depends on the provider (`[Timestamp]` is SQL Server's `rowversion`), so check your provider's docs.

## 2. Migrations

```bash
dotnet tool install --global dotnet-ef                   # once per machine
dotnet add package Microsoft.EntityFrameworkCore.Design
dotnet ef migrations add AddPostComments                 # generates Up/Down code plus a model snapshot
dotnet ef database update                                # applies pending migrations
dotnet ef migrations remove                              # deletes the last migration if it is not applied
dotnet ef migrations script --idempotent -o deploy.sql   # SQL script for production deploys
```

Applied migrations are recorded in the `__EFMigrationsHistory` table, much like Rails' `schema_migrations`. Always review generated code: a renamed property can become a drop plus an add, which loses data. In production, prefer reviewed SQL scripts or migration bundles (`dotnet ef migrations bundle`) over calling `Database.Migrate()` at startup from several instances at once.

## 3. Querying, change tracking and SaveChanges

```csharp
var recent = await db.Posts                                // LINQ to Entities: becomes parameterized SQL
    .Where(p => p.Blog.Title.StartsWith("Tech"))
    .OrderByDescending(p => p.Id)
    .Select(p => new { p.Id, p.Body, Blog = p.Blog.Title })   // projection: only these columns, not tracked
    .Take(10)
    .ToListAsync();

var blog = await db.Blogs.FirstAsync(b => b.Id == id);    // tracked: EF keeps a snapshot
blog.Title = "New title";                                  // state becomes Modified
db.Posts.Add(new Post { Body = "Hello", BlogId = id });    // state: Added
await db.SaveChangesAsync();                               // UPDATE + INSERT in one transaction

var list = await db.Blogs.AsNoTracking().ToListAsync();   // read-only: no snapshots, faster
```

- Entity states are Detached, Unchanged, Added, Modified and Deleted (`db.Entry(blog).State`). On `SaveChanges`, the change tracker compares current values with the snapshots and generates only the needed SQL.
- The tracker also does identity resolution (one instance per key per context), and `Find`/`FindAsync` check it before querying.
- Since EF Core 3, a query that cannot be translated to SQL throws instead of silently running in memory (only the final `Select` may evaluate on the client).

## 4. Loading related data and N+1

```csharp
foreach (var b in await db.Blogs.ToListAsync())
    Console.WriteLine(b.Posts.Count);    // lazy loading on: one query per blog (N+1); off: silently 0

var blogs = await db.Blogs
    .Include(b => b.Posts)               // eager loading
        .ThenInclude(p => p.Comments)    // next level down
    .AsSplitQuery()                      // one SQL query per collection instead of one giant JOIN
    .ToListAsync();

var first = await db.Blogs.FirstAsync();
await db.Entry(first).Collection(b => b.Posts).LoadAsync();   // explicit loading, on demand
```

| Strategy | How | When |
|---|---|---|
| Eager | `Include` / `ThenInclude` | you know you need the related data |
| Explicit | `Entry(x).Collection(...)` or `.Reference(...)`, then `LoadAsync()` | only sometimes needed |
| Lazy | `UseLazyLoadingProxies()` plus `virtual` navigations (Proxies package) | convenient, but hides N+1; off by default |
| Projection | `Select` into a DTO | lists and APIs: usually the best option |

Including two collections in one query multiplies rows (blogs x posts x tags): a **cartesian explosion**. `AsSplitQuery()` (EF Core 5) sends separate queries instead, at the cost of more round trips and no single consistent snapshot unless you wrap them in a transaction. These are the same trade-offs as Rails' `includes`, `preload` and `eager_load`.

## 5. Raw SQL, transactions, concurrency and bulk operations

```csharp
var posts = await db.Posts
    .FromSql($"SELECT * FROM Posts WHERE BlogId = {blogId}")   // interpolated value becomes a parameter
    .Where(p => p.Body != "")                                  // still composable with LINQ
    .ToListAsync();
// NEVER: db.Posts.FromSqlRaw("SELECT * FROM Posts WHERE Body = '" + input + "'")

await using var tx = await db.Database.BeginTransactionAsync();   // only for multiple SaveChanges calls
db.Blogs.Add(new Blog { Title = "A" });
await db.SaveChangesAsync();
db.Posts.Add(new Post { Body = "B", BlogId = 1 });
await db.SaveChangesAsync();
await tx.CommitAsync();                                     // disposing without commit rolls back

try { await db.SaveChangesAsync(); }
catch (DbUpdateConcurrencyException) { /* reload, merge, or tell the user someone else changed it */ }

await db.Posts.Where(p => p.BlogId == oldId)                // one UPDATE statement, nothing loaded (EF Core 7+)
    .ExecuteUpdateAsync(s => s.SetProperty(p => p.BlogId, newId));
await db.Posts.Where(p => p.Body == "").ExecuteDeleteAsync();
```

- A single `SaveChanges` is already transactional; explicit transactions are for several `SaveChanges` calls or for mixing in raw SQL or Dapper.
- With a concurrency token, EF adds `WHERE RowVersion = @original` to UPDATE and DELETE; zero affected rows means someone changed the row first, so it throws. That is optimistic concurrency; pessimistic locking (`SELECT ... FOR UPDATE`) needs raw SQL.
- `ExecuteUpdate`/`ExecuteDelete` bypass the change tracker: already-loaded entities in the context go stale, and no `SaveChanges` call is involved.

## 6. DbContext lifetime

`AddDbContext` registers the context as scoped: one instance per request, acting as that request's unit of work. It is not thread-safe, so running parallel queries on one context (for example `Task.WhenAll` over two queries) throws "A second operation was started on this context instance before a previous operation completed." Long-lived contexts accumulate tracked entities and slow down, so background jobs create a scope per unit of work or use `AddDbContextFactory` with `IDbContextFactory<T>` (also the recommended approach in Blazor Server). `AddDbContextPool` reuses context instances to reduce allocation cost.

## 7. Dapper

```csharp
using Dapper;
using Microsoft.Data.SqlClient;

await using var conn = new SqlConnection(connectionString);     // Dapper opens and closes it as needed

var posts = (await conn.QueryAsync<Post>(
    "SELECT Id, Body, BlogId FROM Posts WHERE BlogId = @BlogId AND Body LIKE @Term",
    new { BlogId = 5, Term = "%dotnet%" })).ToList();             // anonymous object -> SQL parameters

var post = await conn.QueryFirstOrDefaultAsync<Post>("SELECT * FROM Posts WHERE Id = @id", new { id = 7 });
int rows = await conn.ExecuteAsync("UPDATE Posts SET Body = @body WHERE Id = @id", new { body = "edited", id = 7 });
var some = await conn.QueryAsync<Post>("SELECT * FROM Posts WHERE Id IN @ids", new { ids = new[] { 1, 2, 3 } });

var withBlogs = await conn.QueryAsync<Post, Blog, Post>(          // multi-mapping: one row -> Post + Blog
    "SELECT p.Id, p.Body, p.BlogId, b.Id, b.Title FROM Posts p JOIN Blogs b ON b.Id = p.BlogId",
    (p, b) => { p.Blog = b; return p; },
    splitOn: "Id");                                               // Blog's columns start at the second Id

using var grid = await conn.QueryMultipleAsync(                   // several result sets, one round trip
    "SELECT * FROM Blogs WHERE Id = @id; SELECT * FROM Posts WHERE BlogId = @id;", new { id = 5 });
var blog = await grid.ReadSingleAsync<Blog>();
var blogPosts = (await grid.ReadAsync<Post>()).ToList();

await conn.OpenAsync();
await using var tx = await conn.BeginTransactionAsync();
await conn.ExecuteAsync("UPDATE Accounts SET Balance = Balance - @amt WHERE Id = @fromId", new { amt, fromId }, tx);
await conn.ExecuteAsync("UPDATE Accounts SET Balance = Balance + @amt WHERE Id = @toId", new { amt, toId }, tx);
await tx.CommitAsync();
```

Dapper is fast because it does very little: no change tracking and no LINQ translation, and it caches the generated mapping code per query, so performance is close to hand-written ADO.NET. It never builds SQL for you, and it has no migrations (pair it with EF migrations or a tool such as DbUp). Parameters are sent separately from the SQL text, so `@BlogId` is injection-safe while string interpolation into the SQL is not.

## 8. EF Core vs Dapper, and the repository debate

| | EF Core | Dapper |
|---|---|---|
| Type | full ORM | micro-ORM (a row-to-object mapper) |
| SQL | generated from LINQ | written by you |
| Change tracking | yes, `SaveChanges` | no; explicit `Execute` calls |
| Migrations | built in | none |
| Relationships | navigations, `Include` | multi-mapping by hand |
| Performance | good; tracking and translation add overhead (`AsNoTracking` and compiled queries help) | close to raw ADO.NET |
| Risks | N+1, inefficient generated SQL, over-fetching | SQL strings unchecked at compile time, manual mapping |

**Using both** is common: EF Core for writes, domain logic and migrations, Dapper for dashboards, reports and hot read endpoints. Share the connection with `db.Database.GetDbConnection()` and enlist in the current transaction via `db.Database.CurrentTransaction?.GetDbTransaction()`.

**Repository and unit of work:** a classic layered app wraps data access in `IRepository<T>` plus `IUnitOfWork.Commit()`. With EF Core, `DbSet<T>` already is a repository and `DbContext` already is a unit of work (`SaveChanges` commits everything in one transaction), so generic wrappers often just hide useful features (`Include`, projections, `AsNoTracking`) or leak `IQueryable` anyway. Specific, intention-revealing query services (`IOrderQueries.GetDashboardAsync()`) are a reasonable middle ground, especially with Clean Architecture; test data access against a real database (a container or SQLite) rather than mocking `DbContext`.

## Common pitfalls
- N+1 queries through lazy loading, or missing `Include` leaving navigations silently empty.
- Calling `ToList()` before `Where`, which loads the whole table; tracking large read-only result sets.
- A singleton or shared `DbContext` (concurrency exceptions, stale data, memory growth), or parallel queries on one context.
- `FromSqlRaw` with concatenated input, or interpolating values into Dapper SQL strings: SQL injection.
- Multiple collection `Include`s without `AsSplitQuery`; trusting generated migrations without reading them.

## How to talk about this in an interview
- Map it to what you use daily: `DbContext` and migrations resemble ActiveRecord's connection and Rails migrations, `Include` is `includes`, `ExecuteUpdate` is `update_all`, and Dapper sits close to `find_by_sql` or Drizzle's SQL-first style.
- Show judgment rather than tribalism: EF Core for productivity and consistency, Dapper where you need exact SQL or maximum speed, and measure before switching.
- > **Fill in:** which of EF Core and Dapper your final-year project used, with which database (SQL Server, PostgreSQL), and what data it stored (users, sensor readings, activity labels).

## Interview questions

**Q1. EF Core vs Dapper: when would you use each?**
EF Core is a full ORM: LINQ queries, change tracking, relationships and migrations, which makes CRUD and domain logic fast to build and keeps the schema in code. Dapper maps the results of SQL you write, with almost no overhead, ideal for complex reporting queries, hot read paths or existing hand-tuned SQL. Many teams use EF Core for writes and migrations and Dapper for performance-critical reads.
*Follow-up:* Can they share a transaction? Yes: get the connection from `db.Database.GetDbConnection()` and pass the current `DbTransaction` to Dapper.

**Q2. What is change tracking, and when do you use `AsNoTracking`?**
When EF loads entities it stores snapshots; on `SaveChanges` it compares current values with the snapshots and writes only the changes, all in one transaction. Tracking costs memory and CPU, so read-only queries (lists, API responses, reports) should use `AsNoTracking()` or project into DTOs.

**Q3. Eager vs lazy vs explicit loading: how do you avoid N+1?**
Eager loading (`Include`/`ThenInclude`) fetches related data in the same query; explicit loading fetches it later on purpose; lazy loading fetches it automatically on first access, which inside a loop becomes one query per row (N+1). Avoid N+1 with `Include` or, better, a `Select` projection of exactly the needed fields, and watch the SQL logs.

**Q4. What problem does `AsSplitQuery` solve?**
Including several collections in one query produces a JOIN whose row count is the product of the collections' sizes (cartesian explosion), duplicating data and slowing everything down. `AsSplitQuery()` loads each collection with its own query and stitches the results together, trading one large query for a few small ones.

**Q5. How do migrations work in EF Core?**
`dotnet ef migrations add Name` compares the current model with the last snapshot and generates a migration class with `Up` and `Down` methods. `dotnet ef database update` applies pending migrations and records them in `__EFMigrationsHistory`. For production, generate idempotent SQL scripts or migration bundles and review them.

**Q6. How do you prevent SQL injection with EF Core and Dapper?**
LINQ queries are always parameterized. With raw SQL, use `FromSql`/`ExecuteSql` with interpolated strings (EF turns the holes into parameters) or `FromSqlRaw` with placeholders, never concatenation. In Dapper, write `@name` parameters and pass an anonymous object; never interpolate values into the SQL string.

**Q7. How does optimistic concurrency work in EF Core?**
Mark a concurrency token (`[Timestamp]` row version or `IsConcurrencyToken()`). EF includes the original token value in the WHERE clause of UPDATE and DELETE statements; if another user changed the row, zero rows match and EF throws `DbUpdateConcurrencyException`. You then reload the entity, merge or ask the user, and retry.

**Q8. Why is `DbContext` scoped, and should you wrap it in a repository?**
A context is a short-lived unit of work that is not thread-safe, so one per request is the natural fit; a singleton context would be shared across threads and grow forever. It already implements the repository and unit-of-work patterns, so wrap it only when an abstraction adds clear value, such as intention-revealing query methods or isolating the domain layer.
