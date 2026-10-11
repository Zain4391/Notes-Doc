# Database Setup & AppDbContext

How to wire Entity Framework Core to a PostgreSQL database in an ASP.NET Core (.NET 10) project: packages, project layout, the `DbContext`, dependency-injection lifetimes, migrations, and the repository pattern.

**Contents**

1. [Packages](#packages)
2. [Folder structure](#folder-structure)
3. [Connection string](#connection-string)
4. [AppDbContext](#appdbcontext)
5. [Registering the context](#registering-the-context)
6. [DbContext lifetime and captive dependencies](#dbcontext-lifetime-and-captive-dependencies)
7. [Migrations](#migrations)
8. [Repository pattern](#repository-pattern)
9. [IQueryable pitfall](#iqueryable-pitfall)
10. [Common bugs](#common-bugs)

Related: [EF Core and Fluent API Cheat Sheet](EF%20Core%20and%20Fluent%20API%20Cheat%20Sheet.md) (querying, day-to-day usage and entity configuration) · [Configuration & Environments](configuration-env.md) (where the connection string and its password live) · [Options Pattern](options-pattern.md)

---

## Packages

```bash
dotnet add package Microsoft.EntityFrameworkCore
dotnet add package Npgsql.EntityFrameworkCore.PostgreSQL   # PostgreSQL provider (not Microsoft.EntityFrameworkCore.SqlServer)
dotnet add package Microsoft.EntityFrameworkCore.Tools     # design-time only: migrations and scaffolding
```

| Package | Role |
|---|---|
| `Microsoft.EntityFrameworkCore` | Core EF library (`DbContext`, `DbSet`, LINQ provider) |
| `Npgsql.EntityFrameworkCore.PostgreSQL` | The PostgreSQL provider; supplies `UseNpgsql(...)`. Use the provider that matches your database |
| `Microsoft.EntityFrameworkCore.Tools` | Powers `dotnet ef` commands. Needed at design time only |

Because `Tools` is only needed during development, mark it `PrivateAssets=all` so it does not flow into the published app or the production image:

```xml
<PackageReference Include="Microsoft.EntityFrameworkCore.Tools" Version="10.0.11">
  <PrivateAssets>all</PrivateAssets>
</PackageReference>
```

The `dotnet ef` command itself is a separate global tool. Install it once per machine:

```bash
dotnet tool install --global dotnet-ef
```

## Folder structure

```
Data/
 ├─ AppDbContext.cs
 └─ Configs/
     ├─ ProductConfig.cs
     └─ ProductImageConfig.cs
Models/          ← domain entities only (Product, ProductImage)
```

| Location | Contains |
|---|---|
| `Data/` | Persistence infrastructure: the `DbContext` |
| `Data/Configs/` | One entity configuration class per entity (picked up automatically, see [AppDbContext](#appdbcontext); how to write them: [Fluent API configuration](EF%20Core%20and%20Fluent%20API%20Cheat%20Sheet.md#fluent-api-configuration)) |
| `Models/` | Domain entities only |

`DbContext` is infrastructure, not a domain model, so it does not belong under `Models/`.

## Connection string

The connection string lives in configuration under `ConnectionStrings`, not in code.

**`appsettings.json`** (committed, with no password)

```json
{
  "ConnectionStrings": {
    "Default": "Host=localhost;Port=5432;Database=shop;Username=app"
  }
}
```

The password is a secret. For local development, supply it through User Secrets, using the full nested key path:

```bash
dotnet user-secrets set "ConnectionStrings:Default" "Host=localhost;Port=5432;Database=shop;Username=app;Password=<password>"
```

In deployed environments set it as an environment variable: `ConnectionStrings__Default=...`. See [Managing secrets](configuration-env.md#managing-secrets) for how nested keys map across sources.

## AppDbContext

```csharp
public class AppDbContext(DbContextOptions<AppDbContext> options) : DbContext(options)
{
    public DbSet<Product> Products => Set<Product>();
    public DbSet<ProductImage> ProductImages => Set<ProductImage>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
        => modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
}
```

| Member | Purpose |
|---|---|
| Primary constructor taking `DbContextOptions<AppDbContext>` | Receives the provider and connection string from dependency injection |
| `DbSet<T>` properties | One per entity you want to query or save. Written as `=> Set<T>()` so the property is never `null` |
| `OnModelCreating` | Customises the model. `ApplyConfigurationsFromAssembly` finds every `IEntityTypeConfiguration<T>` class in the assembly (the files in `Data/Configs/`) and applies it, so new configs need no extra registration |

**There is no `OnConfiguring` override.** The connection string is supplied through DI at registration time, so nothing is hardcoded in the context.

## Registering the context

```csharp
// Program.cs
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseNpgsql(builder.Configuration.GetConnectionString("Default")));
```

`GetConnectionString("Default")` reads the `ConnectionStrings:Default` key, so it picks up whichever configuration source has the highest precedence (see [Configuration sources and precedence](configuration-env.md#configuration-sources-and-precedence)).

## DbContext lifetime and captive dependencies

`AddDbContext` registers `AppDbContext` as **Scoped**: one instance per HTTP request. A service with a longer lifetime must never depend on it.

```csharp
// ❌ Wrong: a Singleton captures one DbContext and holds it for the life of the app
builder.Services.AddSingleton<IProductRepository, ProductRepository>();

// ✅ Right: Scoped, matching the DbContext
builder.Services.AddScoped<IProductRepository, ProductRepository>();
```

| | Detail |
|---|---|
| Symptom | `InvalidOperationException` from scope validation (on by default in Development), or in other environments a stale or already-disposed `DbContext` |
| Cause | A `DbContext` is one unit of work. Sharing a single instance across concurrent requests corrupts change tracking and is not thread-safe |
| Rule | A service may depend only on services with the **same or longer** lifetime. Anything that uses the `DbContext` must be Scoped or Transient |

## Migrations

```bash
dotnet ef migrations add InitialCreate        # generate a migration from model changes
dotnet ef database update                     # apply all pending migrations
dotnet ef database update PreviousMigration   # roll back to a named migration
dotnet ef migrations remove                   # delete the latest migration (only if not yet applied)
```

| Command | Notes |
|---|---|
| `migrations add <Name>` | Compares the model to the last snapshot and writes `Up()` and `Down()` methods |
| `database update` | Applies every pending migration in order |
| `database update <Name>` | Moves the database to that migration, running `Down()` for later ones. `database update 0` reverts everything |
| `migrations remove` | Removes the latest migration file. Revert it first if it was already applied |

Good habits:

- **Read the generated `Up()` and `Down()`** before applying a migration. Do not trust it blindly.
- **Test a rollback early.** Run `update <Previous>`, confirm the tables are dropped, then `update` forward again. Do this before you are ten migrations deep on a shared database.

## Repository pattern

```csharp
public interface IProductRepository
{
    Task<Product?> GetByIdAsync(int id);
    Task<List<Product>> GetAllAsync();
    Task AddAsync(Product product);
}
```

A repository wraps data access behind an interface so services depend on the interface, not on EF Core. Whether it is worth the extra layer is debated:

| Argument | Verdict |
|---|---|
| "It lets me mock `DbContext` / `DbSet` in tests" | Weak. Async LINQ on `DbSet` does not mock cleanly |
| "The service layer can mock `IProductRepository` easily" | The real benefit, and the actual testability win |
| "The repository itself needs tests" | Use integration tests against a real or in-memory database, not pure unit tests |

**Accepted trade-off:** `SaveChangesAsync()` is called inside the repository methods rather than by a service-layer Unit of Work. This means there are no atomic transactions spanning several repositories. That is fine at small scope, but it is a known limitation to revisit if the project grows.

## IQueryable pitfall

Do not return `IQueryable` from a repository. It is not a result; it is a query that has not run yet.

```csharp
// ❌ Leaks deferred execution up the stack: the caller's .Where() may run
//    against a DbContext whose scope has already ended
public IQueryable<Product> GetAll() => _context.Products;

// ✅ Materialize inside the repository
public async Task<List<Product>> GetAllAsync() => await _context.Products.ToListAsync();
```

Materialize with `ToListAsync()`, `FirstOrDefaultAsync()` and similar methods before the result leaves the repository.

## Common bugs

Two real bugs caught in code review. Both compile and run without errors.

**1. Staged but never persisted, yet the endpoint returned `201`**

```csharp
await _context.Products.AddAsync(product);
// missing: await _context.SaveChangesAsync();
```

**2. Ignored its own parameter and returned every row**

```csharp
public async Task<List<ProductImage>> GetAllImages(int id)
    => await _context.ProductImages.ToListAsync();   // should filter by id
```

Fix:

```csharp
public async Task<List<ProductImage>> GetAllImages(int id)
    => await _context.ProductImages.Where(i => i.ProductId == id).ToListAsync();
```

**Takeaway:** `Add` / `AddAsync` only *stage* a change, and `SaveChangesAsync` *persists* it. They are separate steps. Verify that the write actually landed, and do not trust the status code alone.
