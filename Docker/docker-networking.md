# Docker Networking

**Status: Covered — Phase 3**

---

## Overview

Docker's **default bridge network** is legacy — containers on it can only reach each other by IP, no name resolution. A **user-defined bridge network** gets Docker's embedded DNS resolver (`127.0.0.11`, visible in `/etc/resolv.conf` inside any container on that network), enabling containers to reach each other **by container/service name**.

Proved this myself with a control experiment: a container on the default bridge had no `127.0.0.11` in its resolver config and went straight to an external DNS server (marked `(legacy)`); a container on a custom network had `127.0.0.11` and could resolve sibling container names directly.

---

## Commands

```bash
docker network create mynet                    # create a user-defined bridge network
docker network ls                               # list networks
docker network inspect mynet                    # inspect (see connected containers, subnet)
docker network connect mynet <container>        # attach a running container to a network
docker run --network mynet --name db <image>    # run a container attached to a custom network
```

---

## Notes

- Containers on the **same user-defined network** resolve each other by the `--name` you gave them (or the Compose service name) — no IP hardcoding needed.
- Container naming has real constraints — hit a rejected name (`A`) for being below the minimum length requirement; also hit a name-conflict error from a leftover *stopped* container still holding that name (had to `docker rm` it first).
- `EXPOSE` in a Dockerfile only documents a port — it does not publish it. Publishing to the host requires `-p hostPort:containerPort` at `docker run` time.

---

[[README|← Back to Docker index]]
