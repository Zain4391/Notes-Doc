# Docker Images

**Status: Covered — Phase 2 (Node, Spring Boot, .NET)**

---

## Multi-stage builds — the general pattern

Build stage compiles/installs everything needed to produce the artifact; runtime stage copies **only** the final artifact into a slim base image. Nothing from the build toolchain (compilers, dev dependencies, source files) ends up in the shipped image.

```dockerfile
# Stage 1: builder
FROM node:20 AS builder
WORKDIR /app
COPY package*.json .
RUN npm install
COPY . .
RUN npm run build

# Stage 2: runtime
FROM node:20-alpine
WORKDIR /app
COPY package*.json .
RUN npm install --omit=dev
COPY --from=builder /app/dist ./dist
EXPOSE 3000
CMD ["node", "dist/app.js"]
```

Real result on my Node/TS project: **1.59GB → 240MB** after splitting build vs runtime stages.

**Gotcha I hit:** `CMD` pointed at the wrong compiled path — check `tsconfig.json`'s `rootDir`/`outDir` to know exactly where `tsc` actually puts output before writing `CMD`.

---

## Spring Boot — layered JARs

A Spring Boot JAR is just a zip of everything mashed together. `layertools` splits it into independently-cacheable layers so a code change doesn't force Docker to re-copy your dependencies.

Enable in `pom.xml`:
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

**Why this COPY order:** `dependencies/` is the biggest and almost never changes — put it first so it stays cached. `application/` is your compiled code, changes every build — put it last.

**Proved myself:** `springboot-full-v5` (single COPY) and `springboot-layered-v1` (layered) were both **324MB** — identical disk size. Layering isn't about shrinking the image, it's entirely about rebuild/push speed via cache reuse.

**Real bugs debugged:** a Docker Desktop BuildKit/attestation engine bug, `JarLauncher` needing `-cp spring-boot-loader` in the entrypoint (not just the bare class), and a manifest-resolution failure caused by copying the jar before it existed on disk (`mvn package` had already run, so `layertools extract` needed the real built jar path — not an assumed one).

---

## .NET — SDK vs runtime split

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY *.csproj .
RUN dotnet restore
COPY . .
RUN dotnet publish -c Release -o /app/publish

FROM mcr.microsoft.com/dotnet/aspnet:8.0
WORKDIR /app
COPY --from=build --chown=app:app /app/publish .
USER app
ENTRYPOINT ["dotnet", "DemoApi.dll"]
```

- `.csproj`-first `COPY` before full source, same caching reasoning as Node/`package.json`.
- `dotnet publish` (not just `build`) — produces the actual deployable output.
- Microsoft's ASP.NET runtime base image ships a **pre-built non-root `app` user (UID 1654)** — no need to create your own.
- `--chown=app:app` on the `COPY` (while still root) + `USER app` **after** — copy first as root, then drop privileges.

**Real bugs debugged:** a live routing bug that looked Docker-related but wasn't (unregistered `/` route + `ASPNETCORE_ENVIRONMENT`), and a deliberate `DemoAPI.dll` vs `DemoApi.dll` typo proving **Linux filesystems are case-sensitive** while NTFS (Windows) isn't — a mismatch that works fine on your Windows host but fails inside the Linux container.

---

## General tagging & registry commands

```bash
docker build -t myapp:1.0 .
docker tag myapp:1.0 myregistry/myapp:1.0
docker push myregistry/myapp:1.0
docker images                 # compare sizes across build iterations
docker inspect <image> --format='{{.Config.Entrypoint}}'   # verify what actually got baked in
```

---

[[README|← Back to Docker index]]
