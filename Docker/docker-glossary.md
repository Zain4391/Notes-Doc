# Docker Glossary

Plain-English definitions, built up as new terms come up in lessons.

---

**Image** — A read-only template with everything needed to run an app (code, runtime, libraries, config). Built from a Dockerfile, made of stacked layers.

**Container** — A running (or stopped) instance of an image. Isolated process with its own filesystem, network, and process space, but shares the host kernel.

**Dockerfile** — A text file of instructions (`FROM`, `COPY`, `RUN`, `CMD`, etc.) describing how to build an image.

**Layer** — Each instruction in a Dockerfile that changes the filesystem creates a cached, reusable layer. Layers stack to form the final image.

**Registry** — A server that stores and distributes images (e.g. Docker Hub, GitHub Container Registry, private registries).

**Repository (in a registry)** — A collection of related images with the same name, differentiated by tags (e.g. `nginx:1.25`, `nginx:latest`).

**Tag** — A label on an image pointing to a specific version (e.g. `latest`, `1.0`, `alpine`).

**Volume** — Docker-managed persistent storage that lives outside a container's writable layer, survives container removal.

**Bind Mount** — Maps a specific host path into a container, unlike a volume which Docker manages internally.

**Bridge Network** — Default Docker network type; lets containers on the same host talk to each other privately.

**Docker Daemon (dockerd)** — Background service that builds, runs, and manages containers. The Docker CLI talks to it via the Docker API.

**Docker Compose** — Tool for defining and running multi-container applications via a single `docker-compose.yml` file.

**Base Image** — The starting image a Dockerfile builds on top of, defined via `FROM` (e.g. `FROM node:20-alpine`).

**Multi-stage Build** — A Dockerfile technique using multiple `FROM` stages to keep the final image small (e.g. build in one stage, copy only the output into a slim runtime stage).

**Entrypoint vs CMD** — `ENTRYPOINT` defines the fixed executable a container runs; `CMD` supplies default arguments (both can be overridden differently at `docker run`).
