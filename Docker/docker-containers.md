# Docker Containers

Day-to-day container management: start it, watch it, get inside it, stop it, remove it, and debug it.

Related: [Fundamentals](docker-fundamentals.md) · [Volumes](docker-volumes.md) · [Troubleshooting](docker-troubleshooting.md) · [Cheatsheet](docker-cheatsheet.md)

---

## Overview

A container is a running (or stopped) instance of an image, with its own writable layer. Managing one comes down to: start it, watch it, get inside it, stop it, remove it.

## Commands

```bash
docker run -d -p 3000:3000 --name myapp myimage   # run detached, named, port-mapped
docker ps                                          # running containers
docker ps -a                                       # all, including stopped
docker logs <container>                            # view logs
docker logs -f <container>                         # follow live
docker exec -it <container> sh                     # shell into a RUNNING container
docker stop <container>                            # graceful (SIGTERM)
docker rm <container>                              # remove a stopped container
docker rm -f <container>                           # force-kill + remove (SIGKILL, unclean)
docker inspect <container>                         # full JSON metadata
docker inspect <container> --format='{{.Config.Entrypoint}}'
```

Common `docker run` flags:

| Flag | Meaning |
|---|---|
| `-d` | Detached: run in the background |
| `-it` | Interactive with a terminal (stdin wired in) |
| `-p host:container` | Publish a container port on the host |
| `--name` | Give the container a name |
| `--rm` | Remove the container automatically when it exits |
| `-v` | Mount a volume or bind mount (see [Volumes](docker-volumes.md)) |
| `--network` | Attach to a network (see [Networking](docker-networking.md)) |

## `attach` vs `run` vs `exec`

Three different things, not interchangeable:

| Command | What it does | Typical failure |
|---|---|---|
| `docker run -it` | Starts a **fresh** container with stdin wired in from the very beginning | None for interactive use |
| `docker exec -it` | Starts a **new process** inside an **already-running** container, alongside the main one | Fails if the container is not running |
| `docker attach` | Reconnects to the container's **original** stdin/stdout stream | Fails for anything that needed input at startup, because that moment has passed |

## Stopping and removing

| Command | Signal | Use for |
|---|---|---|
| `docker stop` | `SIGTERM`, then `SIGKILL` after a timeout | Normal shutdown |
| `docker rm -f` | `SIGKILL` immediately | Last resort |

`docker rm -f` is an unclean shutdown. For stateful services such as databases it means recovery on the next start (demonstrated with Postgres in [Volumes](docker-volumes.md#worked-example-postgres-with-a-named-volume)).

## Debugging

`docker exec` needs a running container. You cannot exec into something that has stopped or exited. To look around a broken image's filesystem, start a temporary container with a shell instead:

```bash
docker run -it --rm --entrypoint sh <image>
```

More techniques are in [Troubleshooting](docker-troubleshooting.md).

## Verifying a non-root user

```bash
docker exec -it <container> whoami                          # should print your app user, not "root"
docker exec -it <container> sh -c "touch /root/test.txt"    # should fail with "Permission denied"
```

If both behave as shown, the non-root user is actually enforced. See [Security](docker-security.md).

---

[← Back to Docker index](README.md)
