# Docker Troubleshooting

How to diagnose a broken container or image, plus a log of real problems and their fixes.

Related: [Containers](docker-containers.md) · [Images](docker-images.md) · [Compose](docker-compose.md) · [Cheatsheet](docker-cheatsheet.md)

---

## Approach

Gather evidence instead of guessing. The habits that pay off most:

1. Read the logs (`docker logs`).
2. Inspect what actually got baked in (`docker inspect --format`).
3. Rebuild with `--no-cache` to rule out stale layers.
4. Drop into the broken image with a temporary shell and look around.

## Diagnostic commands

```bash
docker logs <container>                                       # first stop, always
docker logs -f <container>                                    # follow live
docker ps -a                                                  # check exit code / status
docker inspect <container> --format='{{.State.ExitCode}}'
docker inspect <image> --format='{{.Config.Entrypoint}}'      # confirm what actually got baked in
docker build --no-cache -t <name> .                           # rule out stale cached layers
docker run -it --rm --entrypoint sh <image>                   # poke around a broken image's filesystem
docker exec -it <container> sh -c "find /app -name '*.js'"    # locate where output actually landed
```

## Problem log

| Problem | Cause | Fix / lesson |
|---|---|---|
| **TypeScript output path mismatch** | `CMD` pointed at the wrong compiled file | Check `tsconfig.json`'s `rootDir` / `outDir` before writing the path |
| **Broken Dockerfile (three issues)** | No `package.json`-first caching order; no multi-stage separation (the full TypeScript compiler and devDependencies shipped in the runtime image); no non-root `USER` | Apply the [layer caching order](dockerfile-basics.md#layer-caching-order), [multi-stage builds](docker-images.md#multi-stage-builds) and a [non-root user](docker-security.md) |
| **Docker Desktop BuildKit / attestation engine error** | A genuine Docker Desktop bug, not the app or Dockerfile | Confirm with reproducible evidence rather than assuming user error |
| **`JarLauncher` classpath error** | `ENTRYPOINT` needed `-cp spring-boot-loader`; the bare class could not find its classpath | Add `-cp spring-boot-loader` ([Images](docker-images.md#spring-boot-layered-jars)) |
| **Manifest-resolution failure** | `layertools extract` pointed at a jar path that did not exist yet at that point in the build | Use the real output path of `mvn package`, not an assumed one |
| **Case-sensitivity bug** | `DemoAPI.dll` vs `DemoApi.dll` worked on the Windows host (NTFS is case-insensitive) but failed in the Linux container (case-sensitive) | Match file names exactly; a classic "works on my machine" bug |
| **Routing bug that looked Docker-related** | Unregistered `/` route plus the `ASPNETCORE_ENVIRONMENT` setting, not Docker | Test the app outside the container first to separate app bugs from container bugs |
| **Compose `depends_on` race** | `client` started before `db` could accept connections (running is not ready) | `healthcheck` plus `condition: service_healthy` ([Compose](docker-compose.md#pitfalls)) |
| **Stale volume / baked-in password** | Changing a password's *source* (hardcoded to `.env`) does not touch an already-initialized volume; `initdb` runs once | `docker compose down -v`, then `up` ([Compose](docker-compose.md#pitfalls)) |

---

[← Back to Docker index](README.md)
