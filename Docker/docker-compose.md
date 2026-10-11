# Docker Compose

Describing a multi-container application in one YAML file and running it with a single command.

Related: [Networking](docker-networking.md) · [Volumes](docker-volumes.md) · [Security](docker-security.md) · [Containers](docker-containers.md)

**Contents**

1. [Overview](#overview)
2. [Example: Postgres and a client](#example-postgres-and-a-client)
3. [Key concepts](#key-concepts)
4. [Commands](#commands)
5. [Pitfalls](#pitfalls)

---

## Overview

Compose takes what you would otherwise run as several manual `docker network create`, `docker volume create` and `docker run` commands and describes the same end state declaratively in one file. It adds nothing new under the hood: the same primitives, with one command (`docker compose up`) to bring reality in line with the spec.

- A `services:` entry is a **spec** for how to run a container, not the container itself.
- On `docker compose up`, Compose automatically creates a bridge network named `<folder>_default` and attaches every service to it. Services reach each other **by service name** through Docker's embedded DNS, so the simple case needs no `networks:` block (see [Networking](docker-networking.md)).
- The file is conventionally `compose.yaml` or `docker-compose.yml`; both are supported.

## Example: Postgres and a client

```yaml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - pgvol:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 3s
      retries: 5

  client:
    image: postgres:16
    depends_on:
      db:
        condition: service_healthy
    command: ["tail", "-f", "/dev/null"]   # idle container; exec in on demand

volumes:
  pgvol:
```

`.env` (add to `.gitignore`):

```
POSTGRES_PASSWORD=postgres
```

What each part does:

| Part | Purpose |
|---|---|
| `volumes: pgvol` | Named volume so the database survives container removal ([Volumes](docker-volumes.md)) |
| `healthcheck` | Defines how Compose decides `db` is *ready*, not just started |
| `depends_on ... service_healthy` | `client` starts only after `db` reports healthy |
| `command: tail -f /dev/null` | Keeps `client` running idle so you can `exec` into it. This is the "toolbox container" pattern used in production for on-demand tooling |

## Key concepts

### `depends_on` conditions

| Condition | Waits until |
|---|---|
| *(none)* / `service_started` | The container has **started**. It says nothing about whether the app inside is ready |
| `service_healthy` | The dependency's `healthcheck` reports healthy |
| `service_completed_successfully` | The dependency has exited with code 0 (for one-shot jobs such as migrations) |

### `.env` and variable substitution

Compose automatically loads `.env` from the same folder as the Compose file (no flag needed) and substitutes `${VAR}` references before validating the YAML. Use `docker compose config` to see the final rendered result.

> `.env` is **not** secrets management (see [Security](docker-security.md)). It only stops you hardcoding a password in a file you would commit to git.

## Commands

```bash
docker compose up -d              # start all services, detached
docker compose ps                 # list this project's services and health status
docker compose logs -f            # follow logs, all services
docker compose down               # stop and remove containers and network
docker compose down -v            # also remove volumes: a true clean reset
docker compose exec client sh     # shell into an already-running service
docker compose run --rm client psql -h db -U postgres   # fresh one-off interactive container
docker compose config             # render final YAML with .env substitution resolved
```

`exec` joins a running service; `run --rm` starts a new one-off container. For how these differ from `attach`, see [Containers](docker-containers.md#attach-vs-run-vs-exec).

## Pitfalls

| Problem | What happens | Fix |
|---|---|---|
| **`depends_on` without a condition** | `client` starts as soon as `db`'s container starts, races its startup, gets *connection refused* and exits (`docker compose ps` shows `Exited (2)`). Compose does not restart one-shot commands that fail | Add a `healthcheck` (`pg_isready`) and `depends_on: db: condition: service_healthy` |
| **Stale volume / baked-in password** | `initdb` runs **only once, on a fresh volume**. Changing `POSTGRES_PASSWORD` in the YAML, or moving it into `.env`, has **no effect** on an already-initialized volume; the old password is what is on disk | `docker compose down -v`, then `up` again so `initdb` runs with the new value (this deletes the data) |

---

[← Back to Docker index](README.md)
