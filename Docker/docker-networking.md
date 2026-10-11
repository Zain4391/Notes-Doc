# Docker Networking

How containers talk to each other and to the outside world: bridge networks, name resolution, and publishing ports.

Related: [Compose](docker-compose.md) · [Containers](docker-containers.md) · [Dockerfile Basics](dockerfile-basics.md)

---

## Overview

Docker's **default bridge network** is legacy: containers on it can reach each other only by IP address, with no name resolution.

A **user-defined bridge network** gets Docker's embedded DNS resolver (`127.0.0.11`, visible in `/etc/resolv.conf` inside any container on that network). Containers can then reach each other **by container or service name**, with no IP addresses to hardcode.

| | Default bridge | User-defined bridge |
|---|---|---|
| Resolve containers by name | No (IP only) | **Yes** |
| Embedded DNS (`127.0.0.11`) | No | Yes |
| Created by | Docker, automatically | You (`docker network create`) or Compose |

## Network drivers

| Driver | Purpose |
|---|---|
| `bridge` | Private network on a single host. The default, and the one used for almost everything here |
| `host` | Container shares the host's network stack (no isolation, no port mapping) |
| `none` | No networking |
| `overlay` | Spans multiple hosts (Swarm / multi-host setups) |

## Commands

```bash
docker network create mynet                    # create a user-defined bridge network
docker network ls                              # list networks
docker network inspect mynet                   # connected containers, subnet, etc.
docker network connect mynet <container>       # attach a running container to a network
docker run --network mynet --name db <image>   # run a container attached to a custom network
```

## Name resolution

Containers on the **same user-defined network** resolve each other by the `--name` you gave them, or by the Compose service name. Example: a `psql` client container connects to `db` by name, not by IP.

**How to verify it yourself:** compare the resolver config of two containers.

```bash
docker exec <container-on-default-bridge> cat /etc/resolv.conf   # no 127.0.0.11; points at an external DNS server (marked "(legacy)")
docker exec <container-on-custom-network> cat /etc/resolv.conf   # nameserver 127.0.0.11
```

## Publishing ports

`EXPOSE` in a Dockerfile only **documents** a port. It does not publish it. Publishing to the host requires `-p` at run time:

```bash
docker run -p 8080:80 <image>     # host port 8080 -> container port 80
```

Containers on the same network do not need published ports to reach each other; publishing is only for access from the host or outside.

## Naming rules and conflicts

- Container names must start with a letter or digit and be **at least two characters** (`[a-zA-Z0-9][a-zA-Z0-9_.-]`). A one-character name such as `A` is rejected.
- A name stays reserved even by a **stopped** container. Reusing it fails with a name-conflict error until you `docker rm` the old container.

---

[← Back to Docker index](README.md)
