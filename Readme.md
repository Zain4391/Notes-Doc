# Dev Notes

Personal developer notes: practical, quick-lookup references for the frameworks, tools and shells I use. Each folder covers one technology; each file inside covers one topic, with short explanations, working examples and the pitfalls worth remembering.

The vault is plain Markdown, so it reads the same on GitHub and in [Obsidian](https://obsidian.md) (open this folder as a vault).

---

## Contents

### Frameworks & libraries

| Folder | Covers | Start with |
|---|---|---|
| [Angular](Angular/) | Setup, core concepts, routing, forms, services and HTTP | [basic-setup.md](Angular/basic-setup.md) |
| [Dotnet](Dotnet/) | ASP.NET Core configuration, options pattern, EF Core (setup and Fluent API cheat sheet), Docker | [README.md](Dotnet/README.md) |
| [Nest](Nest/) | Modules, controllers, services, custom exceptions, JWT authentication and RBAC, TypeORM, RabbitMQ | [README.md](Nest/README.md) |
| [Next](Next/) | Overview and routing, data fetching, Axios client, React Query hooks, custom hooks, NextAuth, Zod + React Hook Form, TypeScript patterns | [next-overview.md](Next/next-overview.md) |
| [React](React/) | Project setup with Vite and Tailwind CSS | [React Setup - Vite with TailwindCSS.md](React/React%20Setup%20-%20Vite%20with%20TailwindCSS.md) |
| [Zustand](Zustand/) | Client state management | [overview.md](Zustand/overview.md) |

### Tools & shells

| Folder | Covers | Start with |
|---|---|---|
| [Docker](Docker/) | Fundamentals, Dockerfile, images, containers, Compose, networking, volumes, security, troubleshooting, cheatsheet, glossary | [README.md](Docker/README.md) |
| [Powershell](Powershell/) | Command reference with every common flag explained | [README.md](Powershell/README.md) |
| [linux-manjaro-commands](linux-manjaro-commands/) | Manjaro / Arch package management | [update-packages.md](linux-manjaro-commands/update-packages.md) |

### Architecture

| Folder | Covers | Start with |
|---|---|---|
| [Microservices](Microservices/) | Monolith vs microservices, service boundaries, API gateway, service discovery, configuration, a full reference architecture, all with Mermaid diagrams | [README.md](Microservices/README.md) |

## How the notes are organised

- One folder per technology; one Markdown file per topic.
- Folders with several files have (or should get) a `README.md` that acts as an index, with a reading order and a "I need to..." lookup table. [Docker](Docker/README.md), [Dotnet](Dotnet/README.md), [Nest](Nest/README.md) and [Powershell](Powershell/README.md) are the model to follow.
- Notes are written as documentation, not throwaway code blocks: explain *what* and *why*, then show the command or code.
- Cross-link related notes with relative links, e.g. `[Dockerizing an ASP.NET Core App](../Dotnet/dotnet-docker.md)`.

## Adding a new note

1. **New technology:** create a folder named after it, then add a `README.md` index once it has more than two or three files.
2. **New topic:** add `topic-name.md` inside the matching folder. Name files after the concept, in lowercase `kebab-case`.
3. Open with a one or two sentence summary of what the note covers and when to use it.
4. Use fenced code blocks with a language tag, and put the explanation next to the code. For commands, document each flag in a table.
5. Link the new file from the folder's `README.md` and, if it is a new folder, from the table above.

## Conventions

| Item | Convention |
|---|---|
| New folders | Technology name, e.g. `Powershell/`. Existing folders mix capitalised names and `kebab-case` (`linux-manjaro-commands`); keep an existing name rather than rename it, since renames break links. |
| New files | Lowercase `kebab-case`, e.g. `docker-networking.md`. A few older files use spaces and capitals; those are linked with `%20` in URLs. |
| Folder index | `README.md` |
| Diagrams | Mermaid in ```` ```mermaid ```` blocks (renders in Obsidian and on GitHub); colors and types are explained in [Microservices/README.md](Microservices/README.md) |
| Code blocks | Always include the language (`bash`, `powershell`, `ts`, `csharp`, `yaml`, …) |
| Headings | One `#` title per file, then `##` sections. Don't skip levels. |
| `.obsidian/` | Git-ignored; editor settings stay local. |

## Status

Some folders are still single-topic or lack an index, and are candidates for the same treatment the Docker, Dotnet, Nest and Powershell notes received: [Angular](Angular/), [Next](Next/), [React](React/), [Zustand](Zustand/) and [linux-manjaro-commands](linux-manjaro-commands/).
