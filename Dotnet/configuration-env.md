# Configuration & Environments

How ASP.NET Core loads settings, how it decides which environment it is running in, and where each kind of value (plain settings, secrets) should live.

**Contents**

1. [Overview](#overview)
2. [Configuration sources and precedence](#configuration-sources-and-precedence)
3. [Environment-specific appsettings files](#environment-specific-appsettings-files)
4. [Setting the environment](#setting-the-environment-aspnetcore_environment)
5. [Overriding settings with environment variables](#overriding-settings-with-environment-variables)
6. [Managing secrets](#managing-secrets)
7. [Logging levels per environment](#logging-levels-per-environment)
8. [Reading configuration in code](#reading-configuration-in-code)
9. [Common mistakes](#common-mistakes)

Related: [Options Pattern](options-pattern.md) (strongly-typed access and validation) · [Database Setup](database-setup-efcore.md) (connection strings)

---

## Overview

`WebApplication.CreateBuilder(args)` builds an `IConfiguration` from several **sources** (providers). Each source is a set of key/value pairs. When two sources define the same key, the one loaded **later wins**.

The practical consequence: put shared defaults in `appsettings.json`, override only what differs per environment, and inject anything sensitive from outside the repo.

## Configuration sources and precedence

Listed from lowest to highest precedence:

| Order | Source                           | Purpose                                                                         |
| ----- | -------------------------------- | ------------------------------------------------------------------------------- |
| 1     | `appsettings.json`               | Base values, common to every environment                                        |
| 2     | `appsettings.{Environment}.json` | Per-environment overrides; loaded after the base file                           |
| 3     | User Secrets                     | **Development only.** Local secrets kept outside the repo                       |
| 4     | Environment variables            | Deploy-time overrides; the standard way to configure containers and cloud hosts |
| 5     | Command-line arguments           | Highest precedence; one-off overrides                                           |

> **Rule of thumb:** the closer a source is to the running process (env vars, CLI), the higher its priority.

## Environment-specific appsettings files

`appsettings.{Environment}.json` only needs to contain the **differences**. Any key it does not define falls through to the base file.

**`appsettings.json`**

```json
{ "TestValue": "FromBase" }
```

**`appsettings.Development.json`**

```json
{ "TestValue": "FromDevelopment" }
```

**Result**

```csharp
Console.WriteLine(app.Configuration["TestValue"]);
// Development → "FromDevelopment"
// Any other environment → "FromBase"
```

The `{Environment}` part of the file name must match the active environment name (`Development`, `Staging`, `Production`, or any custom name). Matching is case-insensitive on Windows but treat it as case-sensitive on Linux hosts.

## Setting the environment (`ASPNETCORE_ENVIRONMENT`)

The active environment is read from the `ASPNETCORE_ENVIRONMENT` setting, which is itself just an OS-level environment variable. If it is not set, ASP.NET Core assumes **`Production`**.

| Where it is set                                                     | Used for                                                                                    |
| ------------------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| `launchSettings.json` (per profile, under `environmentVariables`)   | Local development only. The file is not deployed, so it has no effect in other environments |
| `docker run -e ASPNETCORE_ENVIRONMENT=Production ...`               | Containers                                                                                  |
| systemd unit file (`Environment=ASPNETCORE_ENVIRONMENT=Production`) | Linux service deployments                                                                   |
| Azure App Service → Configuration blade                             | Azure                                                                                       |

Checking the environment in code:

```csharp
if (app.Environment.IsDevelopment())
{
    app.UseDeveloperExceptionPage();
}
```

`IsDevelopment()`, `IsStaging()` and `IsProduction()` are available on `IHostEnvironment` / `IWebHostEnvironment`. Use `IsEnvironment("Name")` for custom environments.

## Overriding settings with environment variables

Configuration keys are hierarchical (`Section:Key`). Because `:` is not valid in environment variable names on every OS, use a **double underscore** (`__`) as the section separator.

```bash
BlobStorage__ContainerName=product-images-prod
```

This maps to:

```json
{
  "BlobStorage": {
    "ContainerName": "product-images-prod"
  }
}
```

This is how secrets and environment-specific values are injected into Docker containers:

```bash
docker run -e BlobStorage__AzureStorageKey=<value> myapi
```

For array items, use the index as a segment: `AllowedHosts__0=example.com`.

## Managing secrets

Never commit real secrets. The approach differs by environment.

| Environment          | Where the secret lives                                                                                         |
| -------------------- | -------------------------------------------------------------------------------------------------------------- |
| Local development    | **User Secrets**, stored in your user profile, outside the repo, so it cannot be committed                     |
| Staging / Production | Environment variables injected at deploy or container start, or a vault (Azure Key Vault, AWS Secrets Manager) |

### User Secrets commands

```bash
dotnet user-secrets init                                    # one-time per project; adds a UserSecretsId to the .csproj
dotnet user-secrets set "BlobStorage:AzureStorageKey" "real-value-here"
dotnet user-secrets list
dotnet user-secrets remove "BlobStorage:AzureStorageKey"
dotnet user-secrets clear
```

Notes:

- Use `:` as the separator in `user-secrets` commands (not `__`).
- Secrets are stored as plain text in `secrets.json` under your user profile (`%APPDATA%\Microsoft\UserSecrets\<id>\` on Windows, `~/.microsoft/usersecrets/<id>/` on Linux/macOS). They keep secrets out of source control; they are not encrypted.
- User Secrets are loaded **only in the Development environment**.

### Commit the shape, not the values

Keep the keys in `appsettings.json` with empty values so the expected configuration is documented, and supply the real values from User Secrets or environment variables:

```json
{
  "BlobStorage": {
    "AzureStorageKey": "",
    "ContainerName": ""
  }
}
```

## Logging levels per environment

Log verbosity is configuration like any other setting, so use the same override mechanism: a conservative default in the base file, and a more verbose level in Development.

**`appsettings.json`** (base, which also serves as Production)

```json
{ "Logging": { "LogLevel": { "Default": "Information" } } }
```

**`appsettings.Development.json`**

```json
{ "Logging": { "LogLevel": { "Default": "Debug" } } }
```

To verify the setup works, log at `Debug` and confirm it appears in Development but not in Production:

```csharp
_logger.LogDebug("Uploading {FileName} to {Container}", file.FileName, containerName);
```

## Reading configuration in code

| Approach              | Example                                  | Use when                                                                                                 |
| --------------------- | ---------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| Indexer               | `config["BlobStorage:ContainerName"]`    | Quick one-off read; always returns `string?`                                                             |
| `GetValue<T>`         | `config.GetValue<int>("Retry:Count", 3)` | Single typed value with a default                                                                        |
| `GetSection`          | `config.GetSection("BlobStorage")`       | Working with a whole section                                                                             |
| `GetConnectionString` | `config.GetConnectionString("Default")`  | Reads `ConnectionStrings:Default`                                                                        |
| Options pattern       | `IOptions<BlobStorageOptions>`           | **Preferred for application code**: strongly typed, validated. See [Options Pattern](options-pattern.md) |

## Common mistakes

| Mistake                                                                          | Why it matters                                                                                                          | Fix                                                                                        |
| -------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| Typo in a value, e.g. `"product-iamges-prod"`                                    | Compiles, and passes `ValidateOnStart()` because the value is non-empty, so the app silently targets the wrong resource | Verify actual values per environment; validate format or allowed values, not just presence |
| Forgetting `__` in env var names                                                 | `BlobStorage:ContainerName=...` may be rejected or ignored by the shell or OS                                           | Use `BlobStorage__ContainerName`                                                           |
| Relying on `launchSettings.json` outside local dev                               | The file is not part of a deployed app, so the environment silently falls back to `Production`                          | Set `ASPNETCORE_ENVIRONMENT` in the host, container or service definition                  |
| Putting real secrets in `appsettings*.json`                                      | They end up in source control                                                                                           | Use User Secrets locally and env vars or a vault elsewhere                                 |
| Leaving test keys (e.g. `TestValue`) and debug `Console.WriteLine` probes behind | Clutters config and leaks noise into logs                                                                               | Delete experiment scaffolding once it has served its purpose                               |
