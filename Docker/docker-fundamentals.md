# Docker Fundamentals

**Status: Covered — Phase 1**

---

## Overview

Docker containers share the host's kernel and isolate processes via Linux namespaces + cgroups — unlike a VM, which virtualizes an entire OS. That's why containers start in milliseconds and are lightweight.

On Windows 11 + Docker Desktop, everything actually runs Linux end-to-end: Docker Desktop spins up a lightweight Linux VM via **WSL2**, and every container executes inside that VM. Windows itself never runs the app directly — this is why commands run inside a container (`ls`, `sh`, `whoami`) are Linux commands, not PowerShell, even though the host is Windows.

**Image vs container vs layer:**
- **Image** = read-only template (built once, stacked layers)
- **Container** = a running (or stopped) instance of an image, with its own writable layer on top
- **Layer** = each Dockerfile instruction that touches the filesystem creates a cached layer

---

## Commands

```bash
docker run <image>              # Create + start a container
docker run -it <image> sh       # Interactive shell inside a container
docker ps                       # Running containers
docker ps -a                    # All containers, incl. stopped
docker exec -it <container> sh  # Shell into an already-running container
docker logs <container>         # View logs
docker stop <container>         # Graceful stop (SIGTERM)
docker rm <container>           # Remove a stopped container
docker images                   # List local images
docker rmi <image>              # Remove an image
```

---

## Notes

- **Bind mount gotcha:** if the host path in a bind mount doesn't exist, Docker silently creates an empty folder instead of erroring — easy to miss and think the mount is broken.
- On Windows, volume data physically lives inside the WSL2 backing VM, not a Windows-browsable path — you won't find it by digging through File Explorer.
- `docker stop`/`docker start` never lose data. Only `docker rm` destroys a container's writable layer — that distinction matters a lot once you're using volumes.

---

[[README|← Back to Docker index]]
