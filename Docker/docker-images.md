# Docker Images

Building small, cache-friendly images: the multi-stage pattern and how it applies to Node, Spring Boot and .NET, plus tagging and registry commands.

Related: [Dockerfile Basics](dockerfile-basics.md) · [Security](docker-security.md) · [Troubleshooting](docker-troubleshooting.md)

**Contents**

1. [Multi-stage builds](#multi-stage-builds)
2. [Node / TypeScript](#node--typescript)
3. [Spring Boot: layered JARs](#spring-boot-layered-jars)
4. [.NET: SDK vs runtime](#net-sdk-vs-runtime)
5. [Tagging and registry commands](#tagging-and-registry-commands)
6. [Pitfalls](#pitfalls)

---

## Multi-stage builds

A multi-stage build uses one stage to compile or install everything needed to produce the artifact, and a second stage that copies **only the final artifact** into a slim base image. Nothing from the build toolchain (compilers, dev dependencies, source files) ends up in the shipped image.

| Stage | Contains | Shipped? |
|---|---|---|
| Build | SDK or compiler, dev dependencies, source code | No |
| Runtime | Slim base image and the built artifact | Yes |

## Node / TypeScript

```dockerfile
# Stage 1: builder
FROM node:20 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

# Stage 2: runtime
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install --omit=dev
COPY --from=builder /app/dist ./dist
EXPOSE 3000
CMD ["node", "dist/app.js"]
```

**Measured result** on a Node/TypeScript project: **1.59 GB → 240 MB** after splitting build and runtime stages.

> **Check the output path.** Look at `tsconfig.json`'s `rootDir` / `outDir` to see exactly where `tsc` puts compiled output *before* writing `CMD`. A wrong path is a common cause of a container that exits immediately.

## Spring Boot: layered JARs

A Spring Boot JAR is a single zip of everything. The `layertools` mode splits it into independently cacheable layers, so a code change does not force Docker to re-copy your dependencies.

Enable layers in `pom.xml`:

```xml
<plugin>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-maven-plugin</artifactId>
  <configuration>
    <layers><enabled>true</enabled></layers>
  </configuration>
</plugin>
```

```dockerfile
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn clean package -DskipTests
RUN java -Djarmode=layertools -jar target/app.jar extract

FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
# Order = least likely to change -> most likely to change
COPY --from=build /app/dependencies/ dependencies/
COPY --from=build /app/spring-boot-loader/ spring-boot-loader/
COPY --from=build /app/snapshot-dependencies/ snapshot-dependencies/
COPY --from=build /app/application/ application/
EXPOSE 8080
ENTRYPOINT ["java", "-cp", "spring-boot-loader", "org.springframework.boot.loader.launch.JarLauncher"]
```

**Why this `COPY` order:** `dependencies/` is the largest layer and almost never changes, so it goes first and stays cached. `application/` is your compiled code and changes on every build, so it goes last.

**Layering does not shrink the image.** A single-`COPY` build (`springboot-full-v5`) and a layered build (`springboot-layered-v1`) were both **324 MB**. The benefit is entirely faster rebuilds and pushes through cache reuse.

Points to remember:

- `ENTRYPOINT` needs `-cp spring-boot-loader` as well as the `JarLauncher` class name.
- `layertools extract` must point at the jar that `mvn package` actually produced, so run it after the package step.

## .NET: SDK vs runtime

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY *.csproj ./
RUN dotnet restore
COPY . .
RUN dotnet publish -c Release -o /app/publish

FROM mcr.microsoft.com/dotnet/aspnet:8.0
WORKDIR /app
COPY --from=build --chown=app:app /app/publish .
USER app
ENTRYPOINT ["dotnet", "DemoApi.dll"]
```

| Detail | Why |
|---|---|
| `COPY *.csproj` before the full source | Same caching reasoning as `package.json` in Node: restore is only re-run when dependencies change |
| `dotnet publish`, not just `build` | Produces the actual deployable output |
| Pre-built `app` user (UID 1654) | Microsoft's ASP.NET runtime image already includes a non-root user; no need to create one |
| `--chown=app:app` on `COPY`, then `USER app` **after** | Copy first as root, then drop privileges |
| `ENTRYPOINT` DLL name | Linux file names are **case-sensitive**: `DemoAPI.dll` and `DemoApi.dll` are different files |

A full walkthrough with improvements, port and environment behavior, and a Compose setup with a database is in [Dockerizing an ASP.NET Core App](../Dotnet/dotnet-docker.md). For configuring the app inside the container (environment, secrets), see [Configuration & Environments](../Dotnet/configuration-env.md).

## Tagging and registry commands

```bash
docker build -t myapp:1.0 .
docker tag myapp:1.0 myregistry/myapp:1.0
docker push myregistry/myapp:1.0
docker images                                              # compare sizes across build iterations
docker inspect <image> --format='{{.Config.Entrypoint}}'   # verify what actually got baked in
```

## Pitfalls

| Symptom | Cause | Where it is explained |
|---|---|---|
| Container exits immediately, file not found | `CMD` points at the wrong compiled path | [Troubleshooting](docker-troubleshooting.md) |
| `JarLauncher` cannot find its classes | Missing `-cp spring-boot-loader` | [Troubleshooting](docker-troubleshooting.md) |
| `layertools extract` fails | Wrong or not-yet-existing jar path | [Troubleshooting](docker-troubleshooting.md) |
| `dll` not found only in the container | Case mismatch that NTFS tolerates but Linux does not | [Troubleshooting](docker-troubleshooting.md) |
| Odd build errors on Docker Desktop | A BuildKit / attestation engine bug, not the Dockerfile | [Troubleshooting](docker-troubleshooting.md) |

---

[← Back to Docker index](README.md)
