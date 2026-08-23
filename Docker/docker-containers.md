# Docker Containers

**Status: Covered — Phase 1**

---

## Overview

A container is a running (or stopped) instance of an image, with its own writable layer. Managing containers day to day comes down to: start it, watch it, get inside it, stop it, remove it.

---

## Commands

```bash
docker run -d -p 3000:3000 --name myapp myimage   # run detached, named, port-mapped
docker ps                                          # running containers
docker ps -a                                       # all, incl. stopped
docker logs <container>                            # view logs
docker logs -f <container>                         # follow live
docker exec -it <container> sh                     # shell into a RUNNING container
docker stop <container>                             # graceful (SIGTERM)
docker rm <container>                               # remove a stopped container
docker rm -f <container>                             # force-kill + remove (SIGKILL, unclean)
docker inspect <container>                           # full JSON metadata
docker inspect <container> --format='{{.Config.Entrypoint}}'
```

---

## Notes

- **`docker exec` requires a running container.** Can't exec into something stopped/exited — spin up a temp one instead: `docker run -it --rm --entrypoint sh <image>` to poke around a broken image's filesystem.
- **`docker attach` vs `docker run` vs `docker exec`** — three genuinely different things, not interchangeable:
  - `attach` reconnects to an existing container's *original* stdin/stdout stream — fails for anything that needed input at startup, since that window already passed.
  - `run` (with `-it`) starts a **fresh** container with stdin wired in from the very beginning.
  - `exec` jumps into an **already-running** container to run a *new* process alongside the main one.
- **`docker rm -f`** sends SIGKILL, not a graceful shutdown — for stateful services like databases this means an unclean shutdown and recovery-on-next-start (proven with Postgres — see docker-volumes.md).
- Testing non-root protection: `docker exec -it <container> whoami` should return your app user, not `root`; `docker exec -it <container> sh -c "touch /root/test.txt"` should fail with `Permission denied` if the non-root user is actually enforced.

---

[[README|← Back to Docker index]]
