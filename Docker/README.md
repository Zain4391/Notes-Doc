# Docker

Practical, hands-on notes from my Docker learning path (beginner → advanced). Each file is a quick-lookup reference — commands plus short explanations of *why*, written as we cover the topic. Learning environment: Windows 11 + Docker Desktop, working directory `D:\Docker_Learning`.

---

## How this folder works

- One file per topic, filled in progressively as we cover it in lessons.
- `docker-cheatsheet.md` is the single "I forgot the command" reference — updated continuously.
- `docker-glossary.md` holds terminology in plain English.
- Progress: Fundamentals → Compose all **covered**. Security is **in progress** (non-root users done; base image tradeoffs, secrets, scanning, resource limits still to come).

---

## Topics

- [[docker-fundamentals]] — what Docker is, architecture, how it differs from VMs
- [[dockerfile-basics]] — writing and building Dockerfiles
- [[docker-images]] — image layers, tagging, registries
- [[docker-containers]] — running, managing, inspecting containers
- [[docker-compose]] — multi-container apps with docker-compose.yml
- [[docker-networking]] — bridge/host/overlay networks, container-to-container comms
- [[docker-volumes]] — persistent data, bind mounts vs named volumes
- [[docker-security]] — non-root users, secrets, image scanning, least privilege
- [[docker-troubleshooting]] — debugging broken containers, common errors

## Reference

- [[docker-cheatsheet]]
- [[docker-glossary]]
