# Docker Fundamentals

What Docker is, how it isolates applications, how it differs from a virtual machine, and the core vocabulary and commands everything else builds on.

Related: [Containers](docker-containers.md) · [Images](docker-images.md) · [Volumes](docker-volumes.md) · [Glossary](docker-glossary.md) · [Cheatsheet](docker-cheatsheet.md)

---

## What Docker does

Docker packages an application together with everything it needs to run (runtime, libraries, configuration) into an **image**, and runs that image as an isolated process called a **container**. The same image behaves the same way on a laptop, a CI server or a production host.

## Containers vs virtual machines

| | Container | Virtual machine |
|---|---|---|
| What is virtualized | Nothing: processes are isolated | Hardware, running a full guest OS |
| Kernel | **Shared** with the host | Its own kernel per VM |
| Startup time | Milliseconds | Seconds to minutes |
| Size | Lightweight (an app and its dependencies) | Heavy (a whole OS) |

Containers start in milliseconds and stay small because there is no guest OS to boot.

### How isolation works

Docker uses two Linux kernel features:

| Feature | Role |
|---|---|
| **Namespaces** | Limit what a process can *see*: its own process tree, network, filesystem and hostname |
| **cgroups** | Limit what a process can *use*: CPU, memory and so on |

## Architecture

| Component | Role |
|---|---|
| **Docker CLI** (`docker`) | The command you type. Sends requests to the daemon through the Docker API |
| **Docker daemon** (`dockerd`) | Background service that builds images and runs and manages containers |
| **Registry** | Server that stores and distributes images (Docker Hub, GitHub Container Registry, private registries) |

## Docker on Windows (WSL2)

On Windows 11, Docker Desktop runs everything on Linux end to end. It starts a lightweight Linux VM through **WSL2**, and every container executes inside that VM. Windows itself never runs the application directly.

Consequences:

- Commands run *inside* a container (`ls`, `sh`, `whoami`) are Linux commands, not PowerShell, even though the host is Windows.
- Volume data lives inside the WSL2 backing VM, not at a path you can browse in File Explorer (see [Volumes](docker-volumes.md)).
- Linux file names are case-sensitive while NTFS is not (see [Troubleshooting](docker-troubleshooting.md)).

## Images, containers and layers

| Term | Meaning |
|---|---|
| **Image** | Read-only template, built once from a Dockerfile as a stack of layers |
| **Container** | A running (or stopped) instance of an image, with its own **writable layer** on top |
| **Layer** | Each Dockerfile instruction that touches the filesystem creates a cached, reusable layer |

One image can start many containers. Each container's changes go into its own writable layer and never alter the image.

## Core commands

```bash
docker run <image>              # Create and start a container
docker run -it <image> sh       # Interactive shell inside a new container
docker ps                       # Running containers
docker ps -a                    # All containers, including stopped
docker exec -it <container> sh  # Shell into an already-running container
docker logs <container>         # View logs
docker stop <container>         # Graceful stop (SIGTERM)
docker rm <container>           # Remove a stopped container
docker images                   # List local images
docker rmi <image>              # Remove an image
```

See [Containers](docker-containers.md) for these in depth and the [Cheatsheet](docker-cheatsheet.md) for the full list.

## Things that surprise people

| Behavior | What to know |
|---|---|
| Bind mount to a missing host path | With `-v`, Docker silently creates an **empty folder** instead of erroring. Easy to mistake for a broken mount |
| `docker stop` / `docker start` | Never lose data |
| `docker rm` | Destroys the container's writable layer. This is the distinction that matters once you use [volumes](docker-volumes.md) |

---

[← Back to Docker index](README.md)
