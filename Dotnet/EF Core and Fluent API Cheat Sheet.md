# EF Core and Fluent API Cheat Sheet

A quick reference for day-to-day Entity Framework Core work: setting up the context, CRUD, queries, loading related data, tracking, **Fluent API model configuration**, migrations, transactions, concurrency, raw SQL and diagnostics.

> **Provider note:** the examples use SQL Server (`UseSqlServer`, `GETUTCDATE()`). The model, query and Fluent API code is provider-neutral. For PostgreSQL, switch to `UseNpgsql(...)` and use PostgreSQL SQL where raw SQL appears (for example `now()` instead of `GETUTCDATE()`).
>
> **Version notes** such as "EF Core 7+" show the release that introduced a feature.

**Contents**

1. [DbContext setup](#dbcontext-setup)
2. [CRUD operations](#crud-operations)
3. [Everyday query patterns](#everyday-query-patterns)
4. [Loading related data](#loading-related-data)
5. [Tracking behavior](#tracking-behavior)
6. [Fluent API configuration](#fluent-api-configuration)
7. [Migrations](#migrations)
8. [Transactions](#transactions)
9. [Concurrency](#concurrency)
10. [Raw SQL](#raw-sql)
11. [Diagnostics](#diagnostics)
12. [Common gotchas](#common-gotchas)
13. [LINQ method reference](#linq-method-reference)

Related: [Database Setup & AppDbContext](database-setup-efcore.md) (packages, folder layout, DI lifetimes, repository pattern) · [Configuration & Environments](configuration-env.md) (connection strings and secrets) · [Options Pattern](options-pattern.md)

---

## DbContext setup

### The context class

```csharp
public class AppDbContext : DbContext
{
    public DbSet<Author> Authors => Set<Author>();
    public DbSet<Book> Books => Set<Book>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
        => modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
}
```

| Member | Purpose |
|---|---|
| `DbSet<T>` | A queryable, saveable collection of entity `T`. Written as `=> Set<T>()` so it is never `null` |
| `OnModelCreating` | Where the model is customised (see [Fluent API configuration](#fluent-api-configuration)) |
| `ApplyConfigurationsFromAssembly` | Finds and applies every `IEntityTypeConfiguration<T>` class in the assembly |

### Registering it (Program.cs)

This is the pattern to use almost every time:

```csharp
var connectionString = builder.Configuration.GetConnectionString("Default");

builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString));
```

```json
// appsettings.json
{
  "ConnectionStrings": {
    "Default": "Server=.;Database=MyApp;Trusted_Connection=True;TrustServerCertificate=True"
  }
}
```

Prefer configuring the connection through DI rather than overriding `OnConfiguring`. It keeps the context testable and keeps secrets out of code. If you do override it, guard with `if (!options.IsConfigured)` so DI-supplied options are not overwritten.

### Registration variants

| Method | Use when |
|---|---|
| `AddDbContext<T>` | Default. One scoped context per request |
| `AddDbContextFactory<T>` | You need to create contexts yourself: Blazor, background workers, anything outside a request scope |
| `AddDbContextPool<T>` | High-throughput APIs; reuses context instances instead of creating new ones |

```csharp
builder.Services.AddDbContextFactory<AppDbContext>(options => /* ... */);
builder.Services.AddDbContextPool<AppDbContext>(options => /* ... */);
```

## CRUD operations

### Create

```csharp
var book = new Book { Title = "Dune", Price = 12.99m, AuthorId = 4 };
context.Books.Add(book);
await context.SaveChangesAsync();
// book.Id is now populated by EF after SaveChanges

context.Books.AddRange(book1, book2);
await context.SaveChangesAsync();
```

`Add` only *stages* the change. Nothing reaches the database until `SaveChangesAsync()`.

### Read

```csharp
var book  = await context.Books.FindAsync(id);                          // primary-key lookup; checks the local cache first
var book  = await context.Books.FirstOrDefaultAsync(b => b.Id == id);   // any predicate; always queries the database
var books = await context.Books.ToListAsync();
```

### Update

**Load, change, save.** This is the typical flow in a service or controller. No `.Update()` call is needed, and EF sends only the changed columns.

```csharp
var book = await context.Books.FirstOrDefaultAsync(b => b.Id == id);
if (book is null) return NotFound();

book.Price = 14.99m;
book.Title = "Dune (Deluxe Edition)";
await context.SaveChangesAsync();
```

**Detached object** (for example, built from an API request body):

```csharp
context.Books.Update(book);     // marks ALL properties as modified
await context.SaveChangesAsync();
```

### Delete

```csharp
var book = await context.Books.FindAsync(id);
if (book is not null)
{
    context.Books.Remove(book);
    await context.SaveChangesAsync();
}
```

### Bulk operations (EF Core 7+)

These run immediately as a single SQL statement. They bypass the change tracker and need no `SaveChanges`.

```csharp
await context.Books.Where(b => b.OutOfPrint).ExecuteDeleteAsync();

await context.Books
    .Where(b => b.Year < 2000)
    .ExecuteUpdateAsync(s => s.SetProperty(b => b.Archived, true));
```

## Everyday query patterns

### Optional filters

Build the query up conditionally. This is extremely common in real endpoints:

```csharp
var query = context.Books.AsQueryable();

if (!string.IsNullOrWhiteSpace(titleFilter))
    query = query.Where(b => b.Title.Contains(titleFilter));

if (minPrice.HasValue)
    query = query.Where(b => b.Price >= minPrice);

if (authorId.HasValue)
    query = query.Where(b => b.AuthorId == authorId);

var results = await query.OrderBy(b => b.Title).ToListAsync();
```

Nothing executes until a terminal method such as `ToListAsync()` runs.

### Pagination with total count

The pattern behind almost every list endpoint:

```csharp
int page = 1, pageSize = 20;

var totalCount = await query.CountAsync();
var pagedItems = await query
    .OrderBy(b => b.Title)
    .Skip((page - 1) * pageSize)
    .Take(pageSize)
    .ToListAsync();
```

Always apply `OrderBy` before `Skip`/`Take`, otherwise page contents are not stable.

### Projection to a DTO

Selecting straight into a DTO avoids over-fetching and keeps EF entities out of your API surface:

```csharp
public record BookSummaryDto(int Id, string Title, string AuthorName, decimal Price);

var summaries = await context.Books
    .Select(b => new BookSummaryDto(b.Id, b.Title, b.Author.Name, b.Price))
    .ToListAsync();
```

A projection does not need `.Include()`: EF joins what it needs.

### Existence checks and aggregates

```csharp
bool exists = await context.Books.AnyAsync(b => b.Isbn == isbn);   // cheaper than loading the entity

var totalValue = await context.Books.SumAsync(b => b.Price);
var avgPrice   = await context.Books.AverageAsync(b => b.Price);
```

### Grouping

```csharp
// Books per author
var counts = await context.Books
    .GroupBy(b => b.AuthorId)
    .Select(g => new { AuthorId = g.Key, Count = g.Count() })
    .ToListAsync();
```

## Loading related data

| Strategy | Use when |
|---|---|
| Eager (`Include`) | **Default choice** for data you will display right away |
| Explicit (`Entry(...).Load`) | You already have the entity and need one more piece |
| Split query (`AsSplitQuery`) | Several collection `Include`s on one query are slow or duplicate rows |
| Lazy | Rarely. Convenient but easy to cause accidental N+1 queries |

### Eager loading

```csharp
var book = await context.Books
    .Include(b => b.Author)
    .FirstOrDefaultAsync(b => b.Id == id);

var books = await context.Books
    .Include(b => b.Author)
    .Include(b => b.Reviews)
    .ToListAsync();

// Multiple levels
context.Books.Include(b => b.Author).ThenInclude(a => a.Publisher);

// Filtered include (EF Core 5+)
context.Books.Include(b => b.Reviews.Where(r => r.Rating > 3));
```

### Explicit loading

```csharp
await context.Entry(author).Collection(a => a.Books).LoadAsync();
```

### Split queries

A query with multiple one-to-many `Include`s produces a join that duplicates rows ("cartesian explosion"). `AsSplitQuery()` runs one SQL query per collection instead:

```csharp
context.Books
    .Include(b => b.Reviews)
    .Include(b => b.Author)
    .AsSplitQuery();
```

### Lazy loading

Needs the `Microsoft.EntityFrameworkCore.Proxies` package, `UseLazyLoadingProxies()`, and `virtual` navigation properties:

```csharp
options.UseLazyLoadingProxies();

public virtual Author Author { get; set; }
```

It is convenient, but it hides database calls inside property access. Prefer eager or explicit loading in APIs.

## Tracking behavior

By default EF **tracks** every entity a query returns so it can detect changes on `SaveChanges`. That costs memory and time on queries that only read.

```csharp
// Use on every read-only query (GET endpoints, reports):
// skips change-tracking overhead and is one of the biggest easy performance wins.
var books = await context.Books.AsNoTracking().ToListAsync();

// No tracking, but the same row is returned as one shared instance
context.Books.AsNoTrackingWithIdentityResolution();

// Context-wide default
context.ChangeTracker.QueryTrackingBehavior = QueryTrackingBehavior.NoTracking;
```

Detaching entities that are already tracked:

```csharp
context.ChangeTracker.Clear();                         // detach everything (EF Core 5+)
context.Entry(entity).State = EntityState.Detached;    // detach one entity
```

## Fluent API configuration

The Fluent API configures the model in code: column rules, keys, indexes, relationships, conversions. It is the most powerful of the three configuration sources.

### Where configuration comes from

EF builds the model from these sources. Later ones override earlier ones:

| Order | Source | Example |
|---|---|---|
| 1 | **Conventions** | A property named `Id` becomes the primary key |
| 2 | **Data annotations** | `[Required]`, `[MaxLength(200)]` on the entity class |
| 3 | **Fluent API** | `entity.Property(b => b.Title).HasMaxLength(200)` |

Use the Fluent API for anything annotations cannot express (indexes, composite keys, precision, delete behavior, query filters) and to keep entity classes free of persistence attributes.

### Where to put it

**Option A: one configuration class per entity (recommended).** Picked up automatically by `ApplyConfigurationsFromAssembly`. See [Folder structure](database-setup-efcore.md#folder-structure).

```csharp
public class BookConfiguration : IEntityTypeConfiguration<Book>
{
    public void Configure(EntityTypeBuilder<Book> entity)
    {
        entity.ToTable("Books");
        entity.HasKey(b => b.Id);

        entity.Property(b => b.Title).IsRequired().HasMaxLength(200);
        entity.Property(b => b.Price).HasPrecision(10, 2);
        entity.Property(b => b.CreatedAt).HasDefaultValueSql("GETUTCDATE()");

        entity.HasIndex(b => b.Isbn).IsUnique();
    }
}
```

**Option B: inline in `OnModelCreating`.** Fine for small models:

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Book>()
        .Property(b => b.Title)
        .IsRequired()
        .HasMaxLength(200);
}
```

### Tables, keys and properties

```csharp
// Table
entity.ToTable("Books");                              // table name
entity.ToTable("Books", schema: "catalog");           // with schema

// Keys
entity.HasKey(b => b.Id);                             // primary key
entity.HasKey(x => new { x.BookId, x.TagId });        // composite key
entity.HasNoKey();                                    // keyless (views, query results)

// Property rules
entity.Property(b => b.Title)
      .IsRequired()                                   // NOT NULL
      .HasMaxLength(200);                             // varchar(200) / nvarchar(200)

entity.Property(b => b.Price).HasPrecision(10, 2);    // decimal(10,2)
entity.Property(b => b.Title).HasColumnName("book_title");
entity.Property(b => b.Notes).HasColumnType("text");

// Defaults and generated values
entity.Property(b => b.CreatedAt).HasDefaultValueSql("GETUTCDATE()");   // SQL expression
entity.Property(b => b.Stock).HasDefaultValue(0);                       // constant
entity.Property(b => b.FullTitle).HasComputedColumnSql("[Title] + ' (' + [Isbn] + ')'");
entity.Property(b => b.Id).ValueGeneratedOnAdd();

// Exclude from the database
entity.Ignore(b => b.DisplayTitle);                   // not-mapped property
```

### Value conversions

Store a property as a different type than it has in code:

```csharp
entity.Property(b => b.Status).HasConversion<string>();   // enum stored as text instead of int
```

### Indexes

```csharp
entity.HasIndex(b => b.Isbn).IsUnique();                          // unique index
entity.HasIndex(b => new { b.AuthorId, b.Title });                // composite index
entity.HasIndex(b => b.Isbn).HasFilter("[IsDeleted] = 0");        // filtered index (SQL is provider-specific)
```

### Relationships

Every relationship is described from one side: `HasOne` / `HasMany`, then the other side with `WithOne` / `WithMany`.

| Type | Configuration |
|---|---|
| One-to-many | `HasOne(...).WithMany(...).HasForeignKey(...)` |
| One-to-one | `HasOne(...).WithOne(...).HasForeignKey<Dependent>(...)` |
| Many-to-many | `HasMany(...).WithMany(...)` |

**One-to-many** (Author → Books), the relationship you will configure most often:

```csharp
modelBuilder.Entity<Book>()
    .HasOne(b => b.Author)
    .WithMany(a => a.Books)
    .HasForeignKey(b => b.AuthorId)
    .OnDelete(DeleteBehavior.Cascade);
```

**One-sided navigation.** Only one entity has a navigation property; leave the other side's `With...()` empty:

```csharp
entity.HasMany(p => p.Images)
      .WithOne()
      .OnDelete(DeleteBehavior.Cascade);   // explicit, not left to convention
```

**One-to-one:**

```csharp
modelBuilder.Entity<Author>()
    .HasOne(a => a.Profile)
    .WithOne(p => p.Author)
    .HasForeignKey<AuthorProfile>(p => p.AuthorId);   // the dependent side holds the FK
```

**Many-to-many** (EF Core 5+, implicit join table):

```csharp
modelBuilder.Entity<Book>()
    .HasMany(b => b.Tags)
    .WithMany(t => t.Books);
```

When the join table needs its own columns (for example `AddedOn`), use an explicit join entity:

```csharp
modelBuilder.Entity<Book>()
    .HasMany(b => b.Tags)
    .WithMany(t => t.Books)
    .UsingEntity<BookTag>();
```

### Delete behavior

Controls what happens to dependents when the principal is deleted:

| `DeleteBehavior` | Effect |
|---|---|
| `Cascade` | Dependents are deleted too. **Default for required relationships** |
| `Restrict` | Delete is blocked if dependents exist |
| `SetNull` | Foreign key is set to `NULL` (needs a nullable FK) |
| `ClientSetNull` | EF nulls the FK on loaded dependents only. **Default for optional relationships** |
| `NoAction` | EF does nothing and the database decides |

State it explicitly with `.OnDelete(...)` rather than relying on convention.

### Owned types

Map a value object into the owner's table, with no separate key or table:

```csharp
modelBuilder.Entity<Author>().OwnsOne(a => a.Address);
// Address columns are stored in the Authors table (Address_Street, Address_City, ...)
```

### Global query filters (soft delete)

```csharp
modelBuilder.Entity<Book>().HasQueryFilter(b => !b.IsDeleted);

// Opt out for admin or reporting queries
var all = await context.Books.IgnoreQueryFilters().ToListAsync();
```

The filter is applied to every query on that entity automatically, which is also why it can silently hide rows (see [Common gotchas](#common-gotchas)).

### Seeding reference data

```csharp
modelBuilder.Entity<Author>().HasData(
    new Author { Id = 1, Name = "Frank Herbert" }
);
```

Seed data must include primary key values. Changes to it are applied through migrations.

### Review what you configure

Fluent configuration can compile and still be wrong. Check for:

- `HasMaxLength(int.MaxValue)` on an `int` column: meaningless, because it applies to strings and byte arrays.
- Duplicate unique constraints, such as an index on a column that is already a key.
- Relationships left to convention that map to the wrong delete behavior.

Read the generated migration (`Up()` / `Down()`) to confirm what was actually configured.

## Migrations

```bash
dotnet ef migrations add AddBookPriceColumn
dotnet ef database update
dotnet ef migrations remove              # undo the last unapplied migration
dotnet ef migrations script              # generate SQL to hand to a DBA or CI pipeline
```

For rollback (`database update <Previous>`) and good habits, see [Migrations](database-setup-efcore.md#migrations).

### Applying migrations at startup

Common for small apps and development environments:

```csharp
using var scope = app.Services.CreateScope();
var context = scope.ServiceProvider.GetRequiredService<AppDbContext>();
await context.Database.MigrateAsync();
```

For production, prefer generating a script (`migrations script`) and applying it in the deployment pipeline.

`EnsureCreated()` / `EnsureDeleted()` are for development and tests only. They create the schema with no migration history, so do not mix them with `Migrate()`.

## Transactions

Most of the time `SaveChangesAsync()` is already atomic. You only need an explicit transaction when several `SaveChanges` calls (or raw SQL) must succeed or fail together.

```csharp
var strategy = context.Database.CreateExecutionStrategy();
await strategy.ExecuteAsync(async () =>
{
    using var tx = await context.Database.BeginTransactionAsync();

    context.Books.Add(newBook);
    await context.SaveChangesAsync();

    context.Authors.Attach(author).Property(a => a.BookCount).CurrentValue += 1;
    await context.SaveChangesAsync();

    await tx.CommitAsync();
});
```

The execution-strategy wrapper is required when retry-on-failure is enabled (`EnableRetryOnFailure()`), because a retry must replay the whole transaction as a unit.

## Concurrency

Optimistic concurrency detects when someone else changed a row between your read and your write.

```csharp
// Configuration (Fluent API); the [Timestamp] attribute does the same
entity.Property(b => b.RowVersion).IsRowVersion();
```

```csharp
try
{
    await context.SaveChangesAsync();
}
catch (DbUpdateConcurrencyException ex)
{
    var entry = ex.Entries.Single();
    var dbValues = await entry.GetDatabaseValuesAsync();

    // Decide: overwrite with dbValues, re-throw to the user, or merge fields
    entry.OriginalValues.SetValues(dbValues);
}
```

`IsRowVersion()` maps to SQL Server's `rowversion`. Other providers use their own mechanism (Npgsql uses the `xmin` system column). For any other property, use `IsConcurrencyToken()`.

## Raw SQL

Use it when LINQ cannot express the query.

```csharp
// Interpolated strings are converted to SQL parameters: safe from SQL injection
var books = await context.Books
    .FromSqlInterpolated($"SELECT * FROM Books WHERE Price > {minPrice}")
    .ToListAsync();

await context.Database.ExecuteSqlInterpolatedAsync(
    $"UPDATE Books SET Price = Price * 1.1 WHERE AuthorId = {authorId}");
```

| Rule | Why |
|---|---|
| Use the `...Interpolated` methods with `{value}` placeholders | Values become parameters, not text pasted into the SQL |
| Never build SQL with `string.Format` or `+` and pass it to `FromSqlRaw` | That is a SQL injection hole |
| `FromSql...` must return all columns of the entity | Otherwise materialisation fails |

## Diagnostics

```csharp
// See the exact SQL EF will generate: invaluable when a query is slow or unexpected
var sql = context.Books.Where(b => b.Price > 10).ToQueryString();

// Log generated SQL to the console during development
options.UseSqlServer(connectionString)
       .LogTo(Console.WriteLine, LogLevel.Information);

options.EnableSensitiveDataLogging();     // development only: logs parameter values
options.EnableDetailedErrors();           // development only: richer error messages
```

Keep these out of production: they can leak data into logs.

## Common gotchas

| Symptom | Cause | Fix |
|---|---|---|
| Extra round-trip queries inside a loop | Lazy loading, or a missing `.Include()` | Eager-load with `.Include()`, or project to a DTO |
| Slow query with several `.Include()`s | Cartesian explosion from multiple one-to-many joins | Add `.AsSplitQuery()` |
| High memory / GC pressure on read APIs | Change tracking enabled on read-only queries | Add `.AsNoTracking()` |
| `InvalidOperationException` on `Update()` / `Attach()` | Another instance with the same key is already being tracked | Update the tracked instance, or detach it first |
| Rows silently missing from results | A global `HasQueryFilter` (for example soft delete) | `.IgnoreQueryFilters()` when you need everything |
| `DbUpdateConcurrencyException` | Another process changed the row (row version mismatch) | Reload values and resolve the conflict before retrying |
| Write "succeeded" but nothing is in the database | `Add` / `Remove` without `SaveChangesAsync()` | Always save, and verify the write landed |
| Query returns every row | A filter parameter was ignored in the `Where` | Check the predicate actually uses the method's parameters |

## LINQ method reference

| Purpose | Method |
|---|---|
| Filter | `.Where()` |
| Project | `.Select()`, `.SelectMany()` |
| Sort | `.OrderBy()`, `.OrderByDescending()`, `.ThenBy()` |
| Page | `.Skip()`, `.Take()` |
| Load related | `.Include()`, `.ThenInclude()` |
| Read-only performance | `.AsNoTracking()` |
| Split query | `.AsSplitQuery()` / `.AsSingleQuery()` |
| Existence check | `.Any()`, `.All()` |
| Single result | `.First()` / `.FirstOrDefault()`, `.Single()` / `.SingleOrDefault()` |
| Bulk write (no tracking) | `.ExecuteUpdateAsync()`, `.ExecuteDeleteAsync()` |
| Debug SQL | `.ToQueryString()` |
