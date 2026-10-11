# .NET

Practical reference notes for building ASP.NET Core back ends with Entity Framework Core. Each file is a self-contained reference meant to be kept open next to the code while writing it: what a thing is, how to set it up, and the mistakes to avoid.

Examples target **.NET 10**. The database examples use **PostgreSQL** (Npgsql) in the setup note and SQL Server in the cheat sheet, with the differences called out where they matter.

---

## Topics

| Note | Covers |
|---|---|
| [Configuration & Environments](configuration-env.md) | Configuration sources and precedence, `appsettings.{Environment}.json`, `ASPNETCORE_ENVIRONMENT`, environment variables, User Secrets (including nested keys), logging levels per environment |
| [Options Pattern](options-pattern.md) | Strongly-typed configuration, binding a class to a section, fail-fast validation with `ValidateOnStart`, `IOptions` vs `IOptionsSnapshot` vs `IOptionsMonitor`, connection strings |
| [Database Setup & AppDbContext](database-setup-efcore.md) | EF Core packages, folder layout, `AppDbContext`, registration, DI lifetimes (captive dependency), migrations, repository pattern, `IQueryable` pitfall |
| [EF Core and Fluent API Cheat Sheet](EF%20Core%20and%20Fluent%20API%20Cheat%20Sheet.md) | CRUD, queries, loading related data, tracking, Fluent API model configuration, transactions, concurrency, raw SQL, diagnostics, gotchas |

## Suggested reading order

Setting up a new project from scratch:

1. [Configuration & Environments](configuration-env.md): how settings and secrets are loaded.
2. [Options Pattern](options-pattern.md): how to read them safely in code.
3. [Database Setup & AppDbContext](database-setup-efcore.md): connect the app to the database.
4. [EF Core and Fluent API Cheat Sheet](EF%20Core%20and%20Fluent%20API%20Cheat%20Sheet.md): day-to-day querying and model configuration.

## Quick lookup

| I need to... | Go to |
|---|---|
| Know which setting wins when the same key is defined twice | [Configuration sources and precedence](configuration-env.md#configuration-sources-and-precedence) |
| Set the environment (Development / Production) | [ASPNETCORE_ENVIRONMENT](configuration-env.md#setting-the-environment-aspnetcore_environment) |
| Keep a password out of the repo | [Managing secrets](configuration-env.md#managing-secrets) |
| Override a nested setting from an environment variable | [Overriding settings with environment variables](configuration-env.md#overriding-settings-with-environment-variables) |
| Make the app fail at startup on bad configuration | [Registration and fail-fast validation](options-pattern.md#registration-and-fail-fast-validation) |
| Register `DbContext` and the connection string | [Registering the context](database-setup-efcore.md#registering-the-context) |
| Fix "Cannot consume scoped service from singleton" | [DbContext lifetime](database-setup-efcore.md#dbcontext-lifetime-and-captive-dependencies) |
| Add or roll back a migration | [Migrations](database-setup-efcore.md#migrations) |
| Configure relationships and delete behavior | [Fluent API configuration](EF%20Core%20and%20Fluent%20API%20Cheat%20Sheet.md#fluent-api-configuration) |
| Page, filter or project a query | [Everyday query patterns](EF%20Core%20and%20Fluent%20API%20Cheat%20Sheet.md#everyday-query-patterns) |
| Fix slow queries or N+1 problems | [Common gotchas](EF%20Core%20and%20Fluent%20API%20Cheat%20Sheet.md#common-gotchas) |
