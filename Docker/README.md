# Docker

Practical reference notes covering Docker from fundamentals through Compose and security. Each file is a quick-lookup reference: commands, short explanations of *why*, and the pitfalls worth remembering.

Environment used for the examples: Windows 11 with Docker Desktop (WSL2 backend). Commands are Linux commands because containers run on Linux; host-side differences (such as PowerShell path syntax) are noted where they matter.

---

## Topics

| Note | Covers |
|---|---|
| [Fundamentals](docker-fundamentals.md) | What Docker is, containers vs VMs, architecture, Docker on Windows (WSL2), core commands |
| [Dockerfile Basics](dockerfile-basics.md) | Instructions, build time vs run time, layer caching order, `.dockerignore`, non-root `USER` |
| [Images](docker-images.md) | Multi-stage builds for Node, Spring Boot (layered JARs) and .NET; tagging and registries |
| [Containers](docker-containers.md) | Running, managing and inspecting containers; `attach` vs `run` vs `exec` |
| [Compose](docker-compose.md) | Multi-container apps with a Compose file, healthchecks, `depends_on` conditions, `.env` |
| [Networking](docker-networking.md) | Bridge networks, name resolution, drivers, port publishing |
| [Volumes](docker-volumes.md) | Persistent data, named volumes vs bind mounts, a Postgres recovery example |
| [Security](docker-security.md) | Non-root users and least privilege; topics still to cover |
| [Troubleshooting](docker-troubleshooting.md) | Diagnostic commands and a log of real problems and fixes |

## Reference

| Note | Use for |
|---|---|
| [Cheatsheet](docker-cheatsheet.md) | Every command, grouped by category |
| [Glossary](docker-glossary.md) | Terminology in plain English |

## Suggested reading order

1. [Fundamentals](docker-fundamentals.md)
2. [Dockerfile Basics](dockerfile-basics.md) and [Images](docker-images.md)
3. [Containers](docker-containers.md)
4. [Volumes](docker-volumes.md) and [Networking](docker-networking.md)
5. [Compose](docker-compose.md)
6. [Security](docker-security.md)
7. [Troubleshooting](docker-troubleshooting.md), at any point something breaks

## Quick lookup

| I need to... | Go to |
|---|---|
| Speed up rebuilds | [Layer caching order](dockerfile-basics.md#layer-caching-order) |
| Shrink an image | [Multi-stage builds](docker-images.md#multi-stage-builds) |
| Run as a non-root user | [Non-root USER](dockerfile-basics.md#non-root-user), [Security](docker-security.md) |
| Get a shell in a container | [`attach` vs `run` vs `exec`](docker-containers.md#attach-vs-run-vs-exec) |
| Keep database data after removing a container | [Volumes](docker-volumes.md) |
| Let containers reach each other by name | [Networking](docker-networking.md#overview) |
| Start a service only after its database is ready | [`depends_on` conditions](docker-compose.md#depends_on-conditions) |
| Fix a container that exits immediately | [Troubleshooting](docker-troubleshooting.md) |
| Run a .NET API in a container | [.NET: SDK vs runtime](docker-images.md#net-sdk-vs-runtime), and [Configuration & Environments](../Dotnet/configuration-env.md) |

---

Related notes: [.NET](../Dotnet/README.md) · [Dockerizing an ASP.NET Core App](../Dotnet/dotnet-docker.md)
