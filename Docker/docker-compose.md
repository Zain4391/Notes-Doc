# Docker Compose

**Status: Covered — Phase 4**

---

## Overview

Compose takes what you'd otherwise run as several manual `docker network create` / `docker volume create` / `docker run` commands and describes the same end state declaratively in one YAML file. Nothing new under the hood — same primitives, one command (`docker compose up`) to reconcile reality against the spec.

- A `service:` block is a **spec** for how to run a container, not the container itself.
- The moment you run `docker compose up`, Compose auto-creates an implicit bridge network (`<folder>_default`) and attaches every service to it — same embedded-DNS name resolution as a manual user-defined network, zero explicit `networks:` block required for the simple case.

---

## Example — Postgres + client, hardened

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
    command: ["tail", "-f", "/dev/null"]   # idle container, exec in on demand

volumes:
  pgvol:
```

`.env` (gitignored):
```
POSTGRES_PASSWORD=postgres
```

---

## Commands

```bash
docker compose up -d              # start all services, detached
docker compose ps                 # list this project's services + health status
docker compose logs -f            # follow logs, all services
docker compose down               # stop + remove containers/network
docker compose down -v            # also remove volumes — true clean reset
docker compose exec client sh     # jump into an already-running service
docker compose run --rm client psql -h db -U postgres   # fresh one-off interactive container
docker compose config             # render final YAML with .env substitution resolved
```

---

## Notes — real bugs hit and fixed

- **`depends_on` without a condition only waits for "container started," not "app ready."** Hit this for real: `client` raced `db`'s startup and got connection-refused, exiting immediately (`docker compose ps` showed `Exited (2)`, not "Up," since Compose doesn't restart one-shot commands that fail). Fixed with `healthcheck` (`pg_isready`) + `depends_on: db: condition: service_healthy` — Compose now polls until `db` reports healthy before starting `client`.
- **Stale-volume/baked-in-password gotcha** — hit twice. `initdb` only runs once, on a fresh volume. Changing `POSTGRES_PASSWORD` in YAML (or migrating it into `.env`) has **zero effect** on an already-initialized volume — the old password is what's on disk. Only `docker compose down -v` + fresh `up` re-triggers `initdb` with the new value.
- **`attach` vs `run --rm` vs `exec`** have genuinely different failure modes, not just definitions — `attach` fails for startup-time prompts (missed window), `run --rm` gives a fresh interactive container with stdin wired from the start, `exec` jumps into a container already running (paired with a `tail -f /dev/null` idle-service pattern for on-demand tool access — the real pattern used for "toolbox" containers in production).
- `.env` is **not** real secrets management (see docker-security.md) — it's just "stop hardcoding a password in a file you'd commit to git." Compose auto-loads `.env` from the same folder as `docker-compose.yml`, no flag needed, and substitutes `${VAR}` references before validating the YAML.

---

[[README|← Back to Docker index]]
