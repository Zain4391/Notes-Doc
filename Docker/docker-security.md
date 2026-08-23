# Docker Security

**Status: In progress — Phase 5**

---

## Covered so far

**Non-root users** — formalized what was already done ad hoc in the Node and .NET images:

```dockerfile
# Alpine
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
```

```dockerfile
# Debian/Ubuntu
RUN useradd -m appuser
USER appuser
```

- `USER` goes **after** any root-requiring `RUN`/`COPY --chown`, since everything after it — including `CMD`/`ENTRYPOINT` — runs as that user.
- Prefer a base image's pre-built non-root user when one exists (Microsoft's ASP.NET image ships `app`, UID 1654; the official `node` image ships `USER node`) over rolling your own.
- **Why it matters:** containers isolate process/filesystem, not privilege by default. A process running as root inside a container that gets RCE'd is still root — combined with a misconfigured bind mount or a container-escape CVE (runc CVEs are a real category), root-in-container can become root-on-host, or at minimum full read/write over everything the container touches, including any secrets loaded as env vars.
- Proved the boundary hands-on: `docker exec -it <container> whoami` → non-root user; `docker exec -it <container> sh -c "touch /root/test.txt"` → `Permission denied`.
- **Least privilege isn't fully solved by `USER` alone** — a non-root user can still do damage within its own writable paths, and Linux capabilities / `--privileged` are a separate axis from "which user am I," not yet covered in depth.

---

## Planned — not yet covered

- Minimal base images: Alpine vs distroless vs slim — real tradeoffs (attack surface, glibc vs musl, debuggability), not just raw size
- Proper secrets management: Docker secrets / BuildKit `--secret`, beyond the `.env` convenience trick from Compose
- Image scanning: `docker scout` or Trivy — finding real CVEs in images already built in this vault
- Resource limits: `--memory`, `--cpus` constraints, why unbounded containers are a production risk

---

[[README|← Back to Docker index]]
