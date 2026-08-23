# Dockerfile Basics

**Status: Covered — Phase 2**

---

## Overview

A Dockerfile is a recipe Docker follows, in order, to build an image. Each instruction creates a cached layer.

| Instruction | What it does |
|---|---|
| `FROM` | Base image to build on top of |
| `WORKDIR` | Sets working directory for subsequent instructions |
| `COPY` | Copies files from host into the image (build time) |
| `RUN` | Executes a command **at build time** — baked into the image |
| `EXPOSE` | Documents the listening port (doesn't publish it — that's `-p` at run time) |
| `CMD` | Default command run **at container start time** |
| `ENTRYPOINT` | Fixed executable a container runs (CMD supplies default args to it) |
| `USER` | Switches the user subsequent instructions (and the final process) run as |

**Key mental model:** `RUN` = build time, baked in. `CMD` = container start time, runs every launch.

---

## Layer caching order (the single most important habit)

```dockerfile
FROM node:20
WORKDIR /app
COPY package.json .
RUN npm install
COPY . .
EXPOSE 3000
CMD ["node", "app.js"]
```

Docker hashes each `COPY`'d file. If `package.json` is byte-identical to the last build, that layer — and `RUN npm install` right after it — is a cache hit and gets **skipped entirely**. Copying `package.json` before the rest of the source means editing app code never invalidates the dependency-install layer. Do `COPY . .` before `npm install` and *every* code edit forces a full reinstall.

---

## Non-root USER (security basics, formalized further in docker-security)

```dockerfile
# Alpine syntax
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
```

```dockerfile
# Debian/Ubuntu syntax
RUN useradd -m appuser
USER appuser
```

- `USER` must come **after** any `RUN` that needs root (installing packages, `chown`ing files) — everything after it, including `CMD`, runs as that user.
- Many official images ship a pre-built non-root user already — e.g. `node` image has `USER node`, Microsoft's ASP.NET runtime image has a pre-built `app` user (UID 1654). Prefer that when available.
- Containers isolate processes/filesystems, not permissions — a root process inside a container that gets RCE'd is still root, which matters a lot if a misconfigured bind mount or a container-escape CVE ever comes into play.

---

[[README|← Back to Docker index]]
