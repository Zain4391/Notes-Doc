# Dockerizing an ASP.NET Core App

How to package an ASP.NET Core app as a Docker image: a line-by-line walkthrough of the `DemoApi` Dockerfile (a .NET 10 minimal API), how to build and run it, and the improvements worth making before using the pattern in a real project.

**Contents**

1. [The Dockerfile](#the-dockerfile)
2. [How it works](#how-it-works)
3. [Build and run](#build-and-run)
4. [Things that behave differently in a container](#things-that-behave-differently-in-a-container)
5. [Improvements](#improvements)
6. [Improved Dockerfile](#improved-dockerfile)
7. [Running with a database (Compose)](#running-with-a-database-compose)
8. [Troubleshooting](#troubleshooting)

Related: [Docker: Images](../Docker/docker-images.md#net-sdk-vs-runtime) (general multi-stage pattern) · [Docker: Compose](../Docker/docker-compose.md) · [Configuration & Environments](configuration-env.md) · [Database Setup & AppDbContext](database-setup-efcore.md)

---

## The Dockerfile

From `DemoApi` (project file `DemoApi.csproj`, `net10.0`):

```dockerfile
# Stage 1: build/publish stage
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src

COPY *.csproj .
RUN dotnet restore

COPY . .
RUN dotnet publish -c Release -o /app/publish

# Stage 2: runtime stage
FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS runtime
WORKDIR /app
COPY --from=build --chown=app:app /app/publish .
USER app

ENTRYPOINT ["dotnet", "DemoApi.dll"]
```

## How it works

It is a [multi-stage build](../Docker/docker-images.md#multi-stage-builds): the large **SDK** image compiles and publishes the app, and only the published output is copied into the much smaller **ASP.NET runtime** image.

### Stage 1: build

| Line | Purpose |
|---|---|
| `FROM ...dotnet/sdk:10.0 AS build` | Image with the full .NET SDK (compiler, `dotnet publish`). Named `build` so stage 2 can copy from it |
| `WORKDIR /src` | Working directory for the build |
| `COPY *.csproj .` then `RUN dotnet restore` | Copy **only** the project file and restore NuGet packages. This layer is re-run only when dependencies change |
| `COPY . .` | Now copy the rest of the source. Code edits invalidate layers from here down, but not the restore layer above |
| `RUN dotnet publish -c Release -o /app/publish` | Compile in Release mode and write the deployable output to `/app/publish` |

The `.csproj`-first order is the same [layer caching](../Docker/dockerfile-basics.md#layer-caching-order) idea as copying `package.json` first in Node.

### Stage 2: runtime

| Line | Purpose |
|---|---|
| `FROM ...dotnet/aspnet:10.0 AS runtime` | Slim image with only the ASP.NET runtime. No SDK, no compiler |
| `WORKDIR /app` | Where the app lives in the final image |
| `COPY --from=build --chown=app:app /app/publish .` | Copy the published output from stage 1, owned by the `app` user |
| `USER app` | Drop root privileges. Must come **after** the `COPY --chown`. The `app` user (UID 1654) is built into Microsoft's images |
| `ENTRYPOINT ["dotnet", "DemoApi.dll"]` | The command the container runs |

### The `ENTRYPOINT` line

`ENTRYPOINT ["dotnet", "DemoApi.dll"]` is the .NET equivalent of `java -jar app.jar`: the `dotnet` host runs the compiled assembly.

- The DLL name is the **assembly name**, which defaults to the `.csproj` file name (`DemoApi.csproj` gives `DemoApi.dll`). If you set `<AssemblyName>` or rename the project, update this line.
- Linux file names are **case-sensitive**: `DemoAPI.dll` fails in the container even though it works on Windows.
- It is in exec form (a JSON array), so the app receives `SIGTERM` directly and can shut down gracefully.

## Build and run

```bash
docker build -t demoapi:1.0 .
docker run --rm -p 8080:8080 demoapi:1.0
```

```bash
curl http://localhost:8080/weatherforecast
```

Run these from the folder containing the `Dockerfile` (the build context).

Since .NET 8 the official images listen on **port 8080** (via `ASPNETCORE_HTTP_PORTS=8080`), not 80, so that a non-root user can bind to it (ports below 1024 require root). Map it with `-p hostPort:8080`.

## Things that behave differently in a container

| Topic | In the container |
|---|---|
| **Environment** | `launchSettings.json` is not used, and the image defaults to `Production`. `DemoApi` only maps OpenAPI when `app.Environment.IsDevelopment()`, so `/openapi/v1.json` returns 404 unless you run with `-e ASPNETCORE_ENVIRONMENT=Development`. See [Setting the environment](configuration-env.md#setting-the-environment-aspnetcore_environment) |
| **Configuration** | `appsettings.json` is baked in. Override values with environment variables at run time (`-e Section__Key=value`), never by rebuilding. See [Overriding settings with environment variables](configuration-env.md#overriding-settings-with-environment-variables) |
| **Secrets** | User Secrets do not exist in the container. Inject secrets as environment variables or from a secret store at run time. Never `COPY` them into the image |
| **HTTPS** | The container only listens on HTTP. `app.UseHttpsRedirection()` finds no HTTPS port, logs a warning, and does not redirect. Terminate TLS at a reverse proxy or load balancer, or remove the redirect when running in containers |
| **File writes** | The app runs as non-root, so it cannot write to most of the filesystem. Write to `/tmp` or a mounted [volume](../Docker/docker-volumes.md) |

## Improvements

What can be changed or added, in rough order of value.

### 1. Add a `.dockerignore`

The `DemoApi` folder contains `bin/` and `obj/`, and without a `.dockerignore`, `COPY . .` copies them into the build. The host's `obj/` holds restore output with Windows paths, which can conflict with the container's own restore, and everything extra slows the build context upload.

```
**/bin/
**/obj/
**/.vs/
**/.vscode/
**/*.user
.git
.gitignore
.dockerignore
Dockerfile*
**/.env
**/secrets.json
```

### 2. Tighten the publish step

```dockerfile
RUN dotnet publish -c Release -o /app/publish --no-restore -p:UseAppHost=false
```

| Flag | Effect |
|---|---|
| `--no-restore` | Skip a second restore; the packages were already restored in the cached layer |
| `-p:UseAppHost=false` | Do not generate the native launcher executable. It is not needed, because `ENTRYPOINT` uses `dotnet DemoApi.dll` |

### 3. Make the configuration a build argument

```dockerfile
ARG BUILD_CONFIGURATION=Release
RUN dotnet publish -c ${BUILD_CONFIGURATION} -o /app/publish --no-restore -p:UseAppHost=false
```

Lets you build `Debug` images on demand: `docker build --build-arg BUILD_CONFIGURATION=Debug .`.

### 4. Use `./` as the `COPY` destination

`COPY *.csproj ./` instead of `COPY *.csproj .`. A wildcard source should be paired with a destination that is clearly a directory.

For a solution with several projects, copy **each** `.csproj` into its own folder before restoring, so the restore layer stays cached:

```dockerfile
COPY MyApp.sln ./
COPY src/Api/Api.csproj src/Api/
COPY src/Core/Core.csproj src/Core/
RUN dotnet restore
COPY . .
```

### 5. Cache NuGet packages between builds (optional)

With BuildKit (the default in Docker Desktop), a cache mount keeps downloaded packages across builds, even when the restore layer is invalidated:

```dockerfile
RUN --mount=type=cache,target=/root/.nuget/packages dotnet restore
```

If you use this, add the same mount to the `publish` step too when it runs `--no-restore`.

### 6. Pin the image version

`sdk:10.0` and `aspnet:10.0` float to the latest patch release. That is fine for learning; for reproducible builds pin a patch version (for example `10.0.x`) or an image digest, and bump it deliberately.

### 7. Consider a smaller runtime image

| Variant | Trade-off |
|---|---|
| `aspnet:10.0` (Debian) | Default. Has a shell, easy to debug |
| `aspnet:10.0-alpine` | Smaller; uses musl libc, so test native dependencies |
| Chiseled (Ubuntu "distroless-style") | Smallest attack surface; **no shell or package manager**, so `docker exec ... sh` does not work |

Check the exact tag names on the Microsoft Artifact Registry listing for the .NET images, since the OS suffix changes between releases.

### 8. Add a health endpoint

```csharp
builder.Services.AddHealthChecks();
// ...
app.MapHealthChecks("/healthz");
```

Useful for orchestrators and load balancers. The default `aspnet` image has **no `curl` or `wget`**, so a Dockerfile / Compose `healthcheck` that shells out to them will not work. Either check from outside the container (the orchestrator or a sidecar), or add the tool in a custom layer.

### 9. Keep configuration out of the image

Nothing environment-specific belongs in the image. The same image should run in every environment; only the environment variables change. See [Configuration & Environments](configuration-env.md).

### 10. Clean up

Remove the leftover learning comment (`# What's the equivalent of "java -jar app.jar" here?`) from the `Dockerfile`.

## Improved Dockerfile

Items 1 to 4 combined. (Not yet built or tested; try it against `DemoApi`.)

```dockerfile
ARG DOTNET_VERSION=10.0

# Stage 1: build and publish
FROM mcr.microsoft.com/dotnet/sdk:${DOTNET_VERSION} AS build
ARG BUILD_CONFIGURATION=Release
WORKDIR /src

COPY *.csproj ./
RUN dotnet restore

COPY . .
RUN dotnet publish -c ${BUILD_CONFIGURATION} -o /app/publish --no-restore -p:UseAppHost=false

# Stage 2: runtime
FROM mcr.microsoft.com/dotnet/aspnet:${DOTNET_VERSION} AS runtime
WORKDIR /app
COPY --from=build --chown=app:app /app/publish ./
USER app

ENTRYPOINT ["dotnet", "DemoApi.dll"]
```

Plus the `.dockerignore` from item 1.

## Running with a database (Compose)

An API and PostgreSQL together. The API reaches the database **by service name** (`db`), and the connection string is supplied as an environment variable using the `__` nesting syntax (see [Overriding settings with environment variables](configuration-env.md#overriding-settings-with-environment-variables)).

```yaml
services:
  api:
    build: .
    ports:
      - "8080:8080"
    environment:
      ASPNETCORE_ENVIRONMENT: Production
      ConnectionStrings__Default: Host=db;Port=5432;Database=shop;Username=app;Password=${DB_PASSWORD}
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:16
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: shop
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d shop"]
      interval: 5s
      timeout: 3s
      retries: 5

volumes:
  pgdata:
```

Put `DB_PASSWORD` in a git-ignored `.env` file next to the Compose file. `depends_on ... service_healthy` makes the API wait until Postgres is ready ([why](../Docker/docker-compose.md#pitfalls)).

### Applying EF Core migrations

The runtime image has **no SDK**, so `dotnet ef` is not available inside it. Options:

| Approach | Use for |
|---|---|
| `context.Database.MigrateAsync()` at startup | Development and small apps only (see [Migrations](EF%20Core%20and%20Fluent%20API%20Cheat%20Sheet.md#migrations)) |
| `dotnet ef migrations script --idempotent` and run the SQL in your pipeline | Production |
| `dotnet ef migrations bundle` to produce a self-contained executable that applies migrations | Production, as a one-off job or init step |

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| Container exits immediately with "Could not execute because the specified command or file was not found" | Wrong DLL name or case in `ENTRYPOINT`, or the `COPY --from=build` path is wrong |
| Connection refused on `localhost:8080` | Port not published (`-p 8080:8080`), or the app was started with `ASPNETCORE_URLS` pointing at a different port |
| `/openapi/v1.json` returns 404 | Running as `Production`; OpenAPI is only mapped in Development |
| Warning: "Failed to determine the https port for redirect" | `UseHttpsRedirection()` with only an HTTP endpoint in the container (harmless; see above) |
| Build fails restoring packages that worked locally | Host `bin/` / `obj/` copied into the build; add a `.dockerignore` |
| "Permission denied" writing a file | Non-root `app` user; write to `/tmp` or a volume with suitable ownership |
| Works locally, fails in the container only | Case-sensitive file names on Linux, a missing environment variable, or `launchSettings.json` assumptions. See [Docker: Troubleshooting](../Docker/docker-troubleshooting.md) |
