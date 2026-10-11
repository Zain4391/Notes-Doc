# Dockerfile Basics

How a Dockerfile builds an image: the core instructions, the build-time vs run-time distinction, layer caching order, and running as a non-root user.

Related: [Images](docker-images.md) (multi-stage builds) · [Security](docker-security.md) · [Fundamentals](docker-fundamentals.md)

**Contents**

1. [Instruction reference](#instruction-reference)
2. [Build time vs run time](#build-time-vs-run-time)
3. [Layer caching order](#layer-caching-order)
4. [Keeping the build context small](#keeping-the-build-context-small)
5. [Non-root USER](#non-root-user)

---

## Instruction reference

A Dockerfile is a recipe Docker follows, in order, to build an image. Each instruction that changes the filesystem creates a cached layer.

| Instruction | What it does |
|---|---|
| `FROM` | Base image to build on top of |
| `WORKDIR` | Sets the working directory for the instructions that follow |
| `COPY` | Copies files from the host into the image (build time) |
| `RUN` | Executes a command **at build time**; the result is baked into the image |
| `EXPOSE` | Documents the listening port. It does **not** publish it; that is `-p` at run time |
| `CMD` | Default command run **at container start** |
| `ENTRYPOINT` | Fixed executable the container runs; `CMD` supplies default arguments to it |
| `USER` | Switches the user that later instructions (and the final process) run as |

## Build time vs run time

> **Mental model:** `RUN` = build time, baked into the image. `CMD` = container start time, runs on every launch.

`CMD` and `ENTRYPOINT` are best written in **exec form** (a JSON array), so the process receives signals such as `SIGTERM` directly:

```dockerfile
CMD ["node", "app.js"]        # exec form (preferred)
CMD node app.js               # shell form: wrapped in /bin/sh -c
```

## Layer caching order

Order instructions from least likely to change to most likely to change. This is the single most important habit.

```dockerfile
FROM node:20
WORKDIR /app
COPY package.json ./
RUN npm install
COPY . .
EXPOSE 3000
CMD ["node", "app.js"]
```

How it works:

- Docker hashes every file that a `COPY` brings in. If `package.json` is byte-identical to the last build, that layer, and the `RUN npm install` right after it, is a cache hit and is **skipped entirely**.
- Copying `package.json` *before* the rest of the source means editing application code never invalidates the dependency-install layer.
- If you put `COPY . .` before `npm install`, **every** code edit forces a full reinstall.

## Keeping the build context small

`docker build .` sends the whole directory (the *build context*) to the daemon, and `COPY . .` copies all of it. A `.dockerignore` file excludes what should never be sent:

```
node_modules
.git
.env
*.log
```

This speeds up builds, avoids copying host-installed dependencies over the image's own, and keeps secrets such as `.env` out of the image.

## Non-root USER

By default a container runs as root. Switch to an unprivileged user. Security reasoning is in [Security](docker-security.md).

```dockerfile
# Alpine
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
```

```dockerfile
# Debian / Ubuntu
RUN useradd -m appuser
USER appuser
```

Rules:

- `USER` must come **after** any `RUN` that needs root (installing packages, `chown`ing files). Everything after it, including `CMD`, runs as that user.
- Many official images already ship a non-root user. The `node` image has `USER node`, and Microsoft's ASP.NET runtime image has a pre-built `app` user (UID 1654). Prefer the built-in one when available.
- Containers isolate processes and filesystems, not privileges. A root process inside a container that gets compromised is still root, which matters if a misconfigured bind mount or a container-escape CVE comes into play.

---

[← Back to Docker index](README.md)
