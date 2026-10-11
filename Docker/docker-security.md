# Docker Security

Hardening containers and images. This note currently covers non-root users; further topics are listed at the end.

Related: [Dockerfile Basics](dockerfile-basics.md#non-root-user) · [Images](docker-images.md) · [Compose](docker-compose.md) · [Containers](docker-containers.md)

---

## Non-root users

By default a container's process runs as `root`. Switch to an unprivileged user:

```dockerfile
# Alpine
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
```

```dockerfile
# Debian / Ubuntu
RUN useradd -m appuser
USER appuser
```

Rules:

- `USER` goes **after** any root-requiring `RUN` or `COPY --chown`, because everything after it, including `CMD` / `ENTRYPOINT`, runs as that user.
- Prefer a base image's pre-built non-root user when one exists: Microsoft's ASP.NET image ships `app` (UID 1654), and the official `node` image ships `USER node`.

### Why it matters

Containers isolate processes and filesystems, not privilege. A process running as root inside a container that gets compromised is still root. Combined with a misconfigured bind mount or a container-escape CVE (runc vulnerabilities are a real category), root-in-container can become root-on-host, or at minimum full read/write over everything the container touches, including secrets loaded as environment variables.

### Verify it

```bash
docker exec -it <container> whoami                          # non-root user
docker exec -it <container> sh -c "touch /root/test.txt"    # Permission denied
```

### Limits of `USER`

A non-root user can still do damage within its own writable paths. Linux capabilities and `--privileged` are a separate axis from "which user am I", and are not covered here yet.

## Not yet covered

| Topic | What it will cover |
|---|---|
| Minimal base images | Alpine vs distroless vs slim: attack surface, glibc vs musl, debuggability, not just raw size |
| Secrets management | Docker secrets and BuildKit `--secret`, beyond the `.env` convenience approach in [Compose](docker-compose.md) |
| Image scanning | `docker scout` or Trivy for finding real CVEs in built images |
| Resource limits | `--memory` and `--cpus`, and why unbounded containers are a production risk |
| Capabilities | Dropping Linux capabilities, avoiding `--privileged` |

---

[← Back to Docker index](README.md)
