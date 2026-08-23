# Docker Troubleshooting

**Status: Covered — ongoing, real bugs from every phase**

---

## Overview

The most valuable debugging habit used throughout: don't guess, gather evidence. `docker logs`, `docker inspect --format`, rebuilding with `--no-cache` to rule out stale cache, and dropping into a broken image with a temp shell (`docker run -it --rm --entrypoint sh <image>`) to look around, rather than reasoning purely from symptoms.

---

## Commands for diagnosing a broken container

```bash
docker logs <container>                                       # first stop, always
docker logs -f <container>                                     # follow live
docker ps -a                                                    # check exit code / status
docker inspect <container> --format='{{.State.ExitCode}}'
docker inspect <image> --format='{{.Config.Entrypoint}}'        # confirm what actually got baked in
docker build --no-cache -t <name> .                             # rule out stale cached layers
docker run -it --rm --entrypoint sh <image>                     # poke around a broken image's filesystem
docker exec -it <container> sh -c "find /app -name '*.js'"      # locate where output actually landed
```

---

## Real bugs debugged (log of actual fixes)

- **TS output path mismatch** — `CMD` pointed at the wrong compiled file; root cause was not checking `tsconfig.json`'s `rootDir`/`outDir` before writing the path.
- **3-issue broken Dockerfile audit** — no `package.json`-first caching order, no multi-stage separation (shipped the full TypeScript compiler + devDependencies in the runtime image), and no non-root `USER` (container ran as root by default).
- **Docker Desktop BuildKit/attestation engine bug** — a genuine Docker Desktop bug, not an app/Dockerfile issue; confirmed via reproducible evidence gathering rather than assuming user error.
- **`JarLauncher` classpath misconfiguration** — `ENTRYPOINT` needed `-cp spring-boot-loader` explicitly; the bare class alone couldn't find its classpath.
- **Manifest-resolution failure** — `layertools extract` was pointed at a jar path that didn't exist yet at that point in the build; fixed by making sure the actual `mvn package` output path was used, not an assumed one.
- **Case-sensitivity bug** — `DemoAPI.dll` vs `DemoApi.dll` worked fine on the Windows host (NTFS is case-insensitive) but failed inside the Linux container (ext4/overlay is case-sensitive) — a real "works on my machine" class of bug.
- **Compose `depends_on` race condition** — `client` started before `db` was actually ready to accept connections, not just "running." Fixed with `healthcheck` + `condition: service_healthy`.
- **Stale-volume/baked-in-password** — changing a password's *source* (hardcoded → `.env`) doesn't touch an already-initialized volume; `initdb` only runs once. Required `docker compose down -v` to actually reproduce/fix.

---

[[README|← Back to Docker index]]
