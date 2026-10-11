# Docker Volumes

Persisting data beyond the life of a container: named volumes, bind mounts, and what actually survives what.

Related: [Containers](docker-containers.md) · [Compose](docker-compose.md) · [Fundamentals](docker-fundamentals.md)

**Contents**

1. [Overview](#overview)
2. [Storage options](#storage-options)
3. [Commands](#commands)
4. [Worked example: Postgres with a named volume](#worked-example-postgres-with-a-named-volume)

---

## Overview

A container's **writable layer is destroyed the moment you `docker rm` it**. Demonstration: write a file inside a container, remove the container, and the data is gone.

A **named volume** decouples data lifetime from container lifetime: the volume survives full container removal and recreation.

| Action | Data in the writable layer |
|---|---|
| `docker stop` / `docker start` | Kept |
| `docker rm` | **Lost** |
| `docker rm` with a named volume mounted | Volume data **kept** |

Volumes only matter once removal, intentional or accidental, enters the picture.

On Windows, volume data lives inside the WSL2 backing VM. The `Mountpoint` from `docker volume inspect` points there, not at a path you can browse from File Explorer.

## Storage options

| Type | Managed by | Syntax | Use for |
|---|---|---|---|
| **Named volume** | Docker | `-v pgvol:/var/lib/postgresql/data` | Persistent application data (databases) |
| **Bind mount** | You (a host path) | `-v $(pwd):/path` | Sharing source code or config from the host |

Bind mount notes:

- With `-v`, a **missing host path is silently created as an empty folder** instead of causing an error. This is easy to mistake for a broken mount. The `--mount` flag errors instead.
- `$(pwd)` is Linux / macOS / Git Bash syntax. In PowerShell use `${PWD}`; in cmd use `%cd%`.

## Commands

```bash
docker volume create pgvol            # create a named volume
docker volume ls                      # list volumes
docker volume inspect pgvol           # Mountpoint, driver, etc.
docker volume rm pgvol                # remove a volume
docker run -v pgvol:/var/lib/postgresql/data <image>   # mount a named volume
docker run -v $(pwd):/path <image>    # bind mount (host path, not Docker-managed)
```

## Worked example: Postgres with a named volume

**Setup.** A Postgres container with a named volume on a custom network, plus a `psql` client container connecting **by container name**. Create a table, insert a row, then run `docker rm -f` on the database container (`SIGKILL`, an unclean shutdown) and recreate it pointing at the **same volume**.

**Result.** The logs showed Postgres **skipped `initdb`**:

```
Skipping initialization — database directory appears to contain a database
```

and instead ran **WAL recovery** to replay uncommitted state left by the unclean shutdown. The row was still there: **the data survived, the container did not.**

**Why it matters.** Production databases should be stopped gracefully (`docker stop`, with proper `SIGTERM` handling) rather than with `-f` / `SIGKILL`. Data is not lost either way, but an unclean shutdown forces a recovery replay on every subsequent start.

---

[← Back to Docker index](README.md)
