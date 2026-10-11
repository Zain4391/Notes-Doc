# Options Pattern

How to read configuration through strongly-typed classes instead of raw `IConfiguration`: binding a class to a section, validating it at startup, choosing between `IOptions<T>`, `IOptionsSnapshot<T>` and `IOptionsMonitor<T>`, and how connection strings fit in.

**Contents**

1. [Why not raw IConfiguration?](#why-not-raw-iconfiguration)
2. [Defining and binding an options class](#defining-and-binding-an-options-class)
3. [Registration and fail-fast validation](#registration-and-fail-fast-validation)
4. [IOptions vs IOptionsSnapshot vs IOptionsMonitor](#ioptions-vs-ioptionssnapshot-vs-ioptionsmonitor)
5. [Consuming options](#consuming-options)
6. [Connection strings](#connection-strings)
7. [Section design](#section-design)

Related: [Configuration & Environments](configuration-env.md) (where the values come from, and how to override them) · [Database Setup & AppDbContext](database-setup-efcore.md) (connection string usage)

---

## Why not raw IConfiguration?

Reading values straight from `IConfiguration` is **stringly-typed**. A typo in the key compiles fine and only fails (or returns `null`) at runtime:

```csharp
var key = configuration["BlobStorage:AzureStorageKey"];   // typo in the key → silent null
```

`GetConnectionString` is only a shortcut for `GetSection("ConnectionStrings")[name]`:

```csharp
var cs = configuration.GetConnectionString("Default");
```

The **options pattern** binds a strongly-typed class to a configuration section. Keys are then properties, so a typo becomes a compile error instead of a runtime surprise, and the class is a single place that documents what a feature needs.

## Defining and binding an options class

**The options class**

```csharp
public class BlobStorageOptions
{
    public const string SectionName = "BlobStorage";

    [Required(AllowEmptyStrings = false)]
    public string AzureStorageKey { get; set; } = string.Empty;

    [Required(AllowEmptyStrings = false)]
    public string ContainerName { get; set; } = string.Empty;
}
```

**The matching configuration**

```json
{
  "BlobStorage": {
    "AzureStorageKey": "...",
    "ContainerName": "product-images"
  }
}
```

How names are matched:

| Item | Rule |
|---|---|
| Section name | Must match the string passed to `BindConfiguration("BlobStorage")`. Keeping it in a `SectionName` constant avoids repeating the literal |
| Class name | Does **not** matter. Call it anything |
| Property names | Must match the keys inside that section (matching is case-insensitive) |
| Properties | Need a public setter so the binder can populate them |

The values can come from any configuration source. The secret key above would normally come from User Secrets or an environment variable (`BlobStorage__AzureStorageKey`); see [Managing secrets](configuration-env.md#managing-secrets).

`[Required]` comes from `System.ComponentModel.DataAnnotations`.

## Registration and fail-fast validation

```csharp
builder.Services.AddOptions<BlobStorageOptions>()
    .BindConfiguration(BlobStorageOptions.SectionName)
    .ValidateDataAnnotations()
    .ValidateOnStart();
```

| Call | Effect |
|---|---|
| `AddOptions<T>()` | Registers the options type with the DI container |
| `BindConfiguration(section)` | Binds the named configuration section to `T` |
| `ValidateDataAnnotations()` | Checks the validation attributes (`[Required]`, `[Range]`, etc.) on `T` |
| `ValidateOnStart()` | Runs validation when the app starts, instead of on first use |

Why `ValidateOnStart()` matters:

| Without `.ValidateOnStart()` | With `.ValidateOnStart()` |
|---|---|
| Validation runs the first time `IOptions<T>` is resolved, which could be in the middle of a request | Validation runs at application startup |
| A misconfiguration surfaces as a failure on some unlucky user's request in production | The app refuses to start, before serving anything |

> **Limit:** validation checks that a value is present and well-formed, not that it is *correct*. A non-empty but misspelled value passes. See [Common mistakes](configuration-env.md#common-mistakes).

## IOptions vs IOptionsSnapshot vs IOptionsMonitor

| Type | Lifetime | When it reads config | Use for |
|---|---|---|---|
| `IOptions<T>` | Singleton | Once, at startup | Static values that never change while the app runs (container names, endpoints) |
| `IOptionsSnapshot<T>` | Scoped | Once per request (scope) | Values that may change between requests when the config source is reloadable |
| `IOptionsMonitor<T>` | Singleton | Live, whenever the source changes | Hot reload and change callbacks (`OnChange`) |

Choosing:

- Default to **`IOptions<T>`**. Most settings do not change at runtime.
- Use **`IOptionsSnapshot<T>`** when each request should see the latest values. It is Scoped, so it cannot be injected into a Singleton service.
- Use **`IOptionsMonitor<T>`** when a long-lived (Singleton) service must react to changes while running.

## Consuming options

Inject the wrapper and read `.Value`:

```csharp
public class FileStorageService(IOptions<BlobStorageOptions> options)
{
    private readonly BlobStorageOptions _options = options.Value;
}
```

The class depends on `BlobStorageOptions` only, not on `IConfiguration`, so it is easy to test: pass `Options.Create(new BlobStorageOptions { ... })` in a unit test.

## Connection strings

Connection strings have their own convention and do not need a custom options class by default.

```json
{ "ConnectionStrings": { "Default": "Host=localhost;Database=..." } }
```

```csharp
var cs = configuration.GetConnectionString("Default");
```

`ConnectionStrings:{name}` is the idiomatic location, and tooling follows it too (for example `dotnet ef dbcontext scaffold "Name=ConnectionStrings:Default" ...`).

| Scenario | Approach |
|---|---|
| Just a connection string | Use `GetConnectionString()`. The options pattern adds little |
| Connection string plus extra settings (retry count, timeout, `EnableSensitiveDataLogging`) | Put the *extra settings* in a `DbOptions` class, and keep reading the string with `GetConnectionString()` |

See [Connection string](database-setup-efcore.md#connection-string) for how it is registered with `AddDbContext`.

## Section design

Give each concern its own section. Do not put unrelated settings into `ConnectionStrings` just because they are "storage-related".

```json
// ❌ Unrelated settings dumped into ConnectionStrings
{ "ConnectionStrings": { "BlobKey": "...", "ContainerName": "..." } }
```

```json
// ✅ Its own section, bound to BlobStorageOptions
{ "BlobStorage": { "AzureStorageKey": "...", "ContainerName": "..." } }
```

Reasons: `ConnectionStrings` is reserved for connection strings (tools and hosts treat it specially), and a dedicated section maps cleanly onto one options class.
