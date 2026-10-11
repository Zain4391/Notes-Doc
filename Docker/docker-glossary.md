# Docker Glossary

Plain-English definitions of the terms used across these notes, in alphabetical order.

| Term | Definition |
|---|---|
| **Base image** | The starting image a Dockerfile builds on, set with `FROM` (e.g. `FROM node:20-alpine`) |
| **Bind mount** | Maps a specific host path into a container. Unlike a volume, Docker does not manage it |
| **Bridge network** | The default network type. Lets containers on the same host talk to each other privately. A *user-defined* bridge adds name resolution |
| **Build context** | The directory sent to the daemon by `docker build`; what `COPY` can see. Trim it with `.dockerignore` |
| **cgroups** | Linux kernel feature that limits the resources (CPU, memory) a process can use |
| **CMD** | Dockerfile instruction giving the default command or arguments run when a container starts. Overridable at `docker run` |
| **Container** | A running (or stopped) instance of an image. An isolated process with its own filesystem, network and process space, sharing the host kernel |
| **Dangling image** | An untagged image no longer referenced by any tag, typically left over from rebuilds. Removed by `docker image prune` |
| **Docker Compose** | Tool for defining and running multi-container applications from a single YAML file |
| **Docker daemon (`dockerd`)** | Background service that builds, runs and manages containers. The CLI talks to it through the Docker API |
| **Dockerfile** | Text file of instructions (`FROM`, `COPY`, `RUN`, `CMD`, ...) describing how to build an image |
| **Embedded DNS** | Resolver at `127.0.0.11` on user-defined networks that lets containers find each other by name |
| **ENTRYPOINT** | Dockerfile instruction defining the fixed executable a container runs. `CMD` then supplies default arguments to it |
| **Healthcheck** | A command Docker runs periodically to decide whether a container's application is actually ready |
| **Image** | Read-only template with everything needed to run an app (code, runtime, libraries, config). Built from a Dockerfile as stacked layers |
| **Layer** | A cached, reusable filesystem change created by a Dockerfile instruction. Layers stack to form an image |
| **Multi-stage build** | Dockerfile technique using multiple `FROM` stages so only the build output, not the toolchain, ends up in the final image |
| **Named volume** | Docker-managed persistent storage that lives outside a container's writable layer and survives container removal |
| **Namespaces** | Linux kernel feature that limits what a process can see (its own processes, network, filesystem) |
| **Non-root user** | An unprivileged user a container runs as (set with `USER`), instead of the default `root` |
| **Port publishing** | Making a container port reachable from the host with `-p hostPort:containerPort`. `EXPOSE` alone only documents it |
| **Registry** | Server that stores and distributes images (Docker Hub, GitHub Container Registry, private registries) |
| **Repository** | A collection of related images in a registry sharing a name and differing by tag (e.g. `nginx:1.25`, `nginx:latest`) |
| **Tag** | A label pointing at a specific image version (`latest`, `1.0`, `alpine`) |
| **Volume** | Docker-managed persistent storage outside a container's writable layer |
| **Writable layer** | The thin read-write layer on top of an image that belongs to one container. Destroyed by `docker rm` |
| **WSL2** | Windows Subsystem for Linux 2. The lightweight Linux VM in which Docker Desktop runs containers on Windows |

---

[← Back to Docker index](README.md)
