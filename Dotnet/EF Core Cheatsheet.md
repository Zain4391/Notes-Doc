# EF Core Cheatsheet

## DbContext Setup

```csharp
public class AppDbContext : DbContext
{
    public DbSet<Author> Authors => Set<Author>();
    public DbSet<Book> Books => Set<Book>();

    protected override void OnConfiguring(DbContextOptionsBuilder options)
        => options.UseSqlServer("connection-string");

    protected override void OnModelCreating(ModelBuilder modelBuilder)
        => modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
}

// DI registration
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString));

builder.Services.AddDbContextFactory<AppDbContext>(options => ...); // for factory pattern
builder.Services.AddDbContextPool<AppDbContext>(options => ...);    // pooled instances
```

## CRUD Operations

```csharp
// Create
context.Books.Add(book);
context.Books.AddRange(book1, book2);
await context.Books.AddAsync(book);           // rarely needed unless using value generators
await context.SaveChangesAsync();

// Read
var book = await context.Books.FindAsync(id);                    // checks local cache first
var book = await context.Books.FirstOrDefaultAsync(b => b.Id == id);
var book = await context.Books.SingleOrDefaultAsync(b => b.Id == id);
var books = await context.Books.ToListAsync();

// Update
context.Books.Update(book);            // marks all properties modified
context.Entry(book).State = EntityState.Modified;
context.Entry(book).Property(b => b.Title).IsModified = true;    // single property
await context.SaveChangesAsync();

// Delete
context.Books.Remove(book);
context.Books.RemoveRange(books);
await context.SaveChangesAsync();

// Bulk operations (EF Core 7+, no SaveChanges needed, executes immediately)
await context.Books.Where(b => b.OutOfPrint).ExecuteDeleteAsync();
await context.Books.Where(b => b.Year < 2000).ExecuteUpdateAsync(s =>
    s.SetProperty(b => b.Archived, true));
```

## Querying (LINQ)

```csharp
// Filtering / projection
context.Books.Where(b => b.Price > 10);
context.Books.Select(b => new { b.Title, b.Price });
context.Books.OrderBy(b => b.Title).ThenByDescending(b => b.Year);
context.Books.Skip(10).Take(20);                      // paging
context.Books.Distinct();
context.Books.GroupBy(b => b.AuthorId);

// Aggregates
await context.Books.CountAsync();
await context.Books.AnyAsync(b => b.Price > 100);
await context.Books.AllAsync(b => b.Price > 0);
await context.Books.SumAsync(b => b.Price);
await context.Books.AverageAsync(b => b.Price);
await context.Books.MaxAsync(b => b.Price);
await context.Books.MinAsync(b => b.Price);

// First/Single variants
FirstAsync / FirstOrDefaultAsync
SingleAsync / SingleOrDefaultAsync   // throws if more than one match
LastAsync / LastOrDefaultAsync       // needs OrderBy, not supported by all providers

// Async streaming (no buffering entire result)
await foreach (var book in context.Books.AsAsyncEnumerable()) { ... }
```

## Loading Related Data

```csharp
// Eager loading
context.Books.Include(b => b.Author);
context.Books.Include(b => b.Author).ThenInclude(a => a.Publisher);
context.Books.Include(b => b.Reviews.Where(r => r.Rating > 3)); // filtered include (EF7+)

// Explicit loading
context.Entry(book).Reference(b => b.Author).Load();
context.Entry(author).Collection(a => a.Books).Load();
await context.Entry(author).Collection(a => a.Books).Query().Where(b => b.Year > 2000).LoadAsync();

// Lazy loading (needs UseLazyLoadingProxies + virtual navigation props)
options.UseLazyLoadingProxies();
public virtual Author Author { get; set; }

// Split queries (avoids cartesian explosion with multiple Includes)
context.Books.Include(b => b.Reviews).AsSplitQuery();
options.UseSqlServer(cs, o => o.UseQuerySplittingBehavior(QuerySplittingBehavior.SplitQuery)); // global default
```

## Tracking Behavior

```csharp
context.Books.AsNoTracking();               // read-only queries, better perf
context.Books.AsNoTrackingWithIdentityResolution();
context.ChangeTracker.QueryTrackingBehavior = QueryTrackingBehavior.NoTracking; // context-wide

context.ChangeTracker.Clear();               // detach all entities (EF7+)
context.Entry(entity).State = EntityState.Detached;
context.ChangeTracker.DetectChanges();
context.ChangeTracker.HasChanges();
```

## Raw SQL

```csharp
// Returning entities
context.Books.FromSqlRaw("SELECT * FROM Books WHERE Price > {0}", 10);
context.Books.FromSqlInterpolated($"SELECT * FROM Books WHERE Price > {minPrice}"); // parameterized, safe

// Composing further (FromSql + LINQ)
context.Books.FromSqlInterpolated($"SELECT * FROM Books").Where(b => b.Year > 2000);

// Non-query SQL
await context.Database.ExecuteSqlInterpolatedAsync($"UPDATE Books SET Price = Price * 1.1");

// Stored procedures
context.Books.FromSqlRaw("EXEC dbo.GetBooksByAuthor @AuthorId = {0}", authorId);
```

## Transactions

```csharp
using var tx = await context.Database.BeginTransactionAsync();
try
{
    await context.SaveChangesAsync();
    await tx.CommitAsync();
}
catch { await tx.RollbackAsync(); throw; }

// Or use built-in execution strategy for retry-safe transactions
var strategy = context.Database.CreateExecutionStrategy();
await strategy.ExecuteAsync(async () =>
{
    using var tx = await context.Database.BeginTransactionAsync();
    await context.SaveChangesAsync();
    await tx.CommitAsync();
});
```

## Migrations (CLI - dotnet ef / Package Manager Console equivalents)

```bash
dotnet ef migrations add InitialCreate
dotnet ef migrations remove
dotnet ef migrations list
dotnet ef database update
dotnet ef database update PreviousMigrationName   # rollback target
dotnet ef migrations script                        # generate SQL script
dotnet ef migrations script FromMigration ToMigration
dotnet ef database drop
dotnet ef dbcontext info
dotnet ef dbcontext scaffold "connection-string" Microsoft.EntityFrameworkCore.SqlServer  # DB-first
```

```csharp
// Applying migrations at runtime
await context.Database.MigrateAsync();
await context.Database.EnsureCreatedAsync();   // dev/testing only, no migration history
await context.Database.EnsureDeletedAsync();
bool pending = (await context.Database.GetPendingMigrationsAsync()).Any();
```

## Model Configuration (Fluent API essentials)

```csharp
modelBuilder.Entity<Book>(entity =>
{
    entity.HasKey(b => b.Id);
    entity.Property(b => b.Title).IsRequired().HasMaxLength(200);
    entity.HasIndex(b => b.Isbn).IsUnique();
    entity.HasOne(b => b.Author).WithMany(a => a.Books).HasForeignKey(b => b.AuthorId);
    entity.HasMany(b => b.Reviews).WithOne(r => r.Book);
    entity.ToTable("Books");
    entity.HasQueryFilter(b => !b.IsDeleted);          // global query filter (soft delete)
    entity.Property(b => b.Price).HasPrecision(10, 2);
    entity.Property(b => b.CreatedAt).HasDefaultValueSql("GETUTCDATE()");
});

// Ignoring global filters per query
context.Books.IgnoreQueryFilters();
```

## Concurrency

```csharp
entity.Property(b => b.RowVersion).IsRowVersion();   // or [Timestamp] attribute

try { await context.SaveChangesAsync(); }
catch (DbUpdateConcurrencyException ex)
{
    var entry = ex.Entries.Single();
    entry.OriginalValues.SetValues(await entry.GetDatabaseValuesAsync());
}
```

## Useful Diagnostics

```csharp
options.LogTo(Console.WriteLine, LogLevel.Information);
options.EnableSensitiveDataLogging();     // dev only — logs parameter values
context.Books.ToQueryString();            // prints generated SQL without executing
```

## Common Extension Method Cheat Table

|Purpose|Method|
|---|---|
|Filter|`.Where()`|
|Project|`.Select()`, `.SelectMany()`|
|Sort|`.OrderBy()`, `.OrderByDescending()`, `.ThenBy()`|
|Page|`.Skip()`, `.Take()`|
|Load related|`.Include()`, `.ThenInclude()`|
|Read-only perf|`.AsNoTracking()`|
|Split query|`.AsSplitQuery()` / `.AsSingleQuery()`|
|Existence check|`.Any()`, `.All()`|
|Single result|`.First()/FirstOrDefault()`, `.Single()/SingleOrDefault()`|
|Bulk write (no tracking)|`.ExecuteUpdateAsync()`, `.ExecuteDeleteAsync()`|
|Debug SQL|`.ToQueryString()`|