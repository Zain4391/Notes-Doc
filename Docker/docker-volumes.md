# Docker Volumes

**Status: Covered — Phase 3**

---

## Overview

A container's **writable layer** dies the moment you `docker rm` it — proved this with a real data-loss demo (wrote a file inside a container, removed it, data gone). A **named volume** decouples data lifetime from container lifetime: the volume survives full container removal and recreation.

`docker stop` / `docker start` never touch this — only `docker rm` deletes the writable layer. Volumes only matter once removal (intentional or accidental) enters the picture.

On Windows, volume data physically lives inside the WSL2 backing VM (`Mountpoint` from `docker volume inspect` points there) — not a path you can casually browse from Windows File Explorer.

---

## Commands

```bash
docker volume create pgvol            # create a named volume
docker volume ls                      # list volumes
docker volume inspect pgvol           # see Mountpoint, driver, etc.
docker volume rm pgvol                # remove a volume
docker run -v pgvol:/var/lib/postgresql/data <image>   # mount a named volume
docker run -v $(pwd):/path <image>    # bind mount (host path, not Docker-managed)
```

---

## Real proof — Postgres + named volume + WAL recovery

Ran a real Postgres container with a named volume + custom network, plus a `psql` client container connecting **by container name**. Created a table, inserted a row, then `docker rm -f` on the DB container (SIGKILL — unclean shutdown) and recreated it pointing at the **same volume**.

Result: logs showed Postgres **skipped `initdb`** entirely (`Skipping initialization — database directory appears to contain a database`) and instead ran **WAL recovery** to replay uncommitted state from the unclean shutdown. The row was still there after recreation — **data survived, the container did not.**

This is also why production databases care about graceful shutdown (`docker stop`, proper SIGTERM handling) over `-f`/SIGKILL — not because data is lost, but because unclean shutdowns force a recovery replay on every subsequent start.

---

[[README|← Back to Docker index]]
