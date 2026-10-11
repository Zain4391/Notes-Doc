# Docker Cheatsheet

The "I forgot the command" reference, grouped by category. For explanations see the topic notes linked from the [index](README.md).

**Contents**

[Setup & Info](#setup--info) · [Images](#images) · [Containers](#containers) · [Debugging](#debugging) · [Volumes](#volumes) · [Networks](#networks) · [Docker Compose](#docker-compose) · [Cleanup](#cleanup)

---

## Setup & Info

```bash
docker --version              # Installed version
docker info                   # System-wide info (containers, images, storage driver)
docker version                # Client and server (daemon) version details
```

## Images

```bash
docker images                    # List local images
docker pull <image>:<tag>        # Download an image from a registry
docker rmi <image>               # Remove an image
docker build -t <name>:<tag> .   # Build an image from the Dockerfile in the current dir
docker build --no-cache -t <name> .   # Build ignoring cached layers
docker tag <src> <target>        # Tag an image (e.g. for pushing to a registry)
docker push <image>              # Push an image to a registry
```

More: [Images](docker-images.md), [Dockerfile Basics](dockerfile-basics.md)

## Containers

```bash
docker run <image>                    # Create and start a container
docker run -d <image>                 # Run detached (background)
docker run -it <image> bash           # Run interactively with a shell
docker run -p 8080:80 <image>         # Map host:container port
docker run --name myapp <image>       # Give the container a name
docker run --rm <image>               # Remove the container when it exits
docker ps                             # List running containers
docker ps -a                          # List all containers (incl. stopped)
docker stop <container>               # Gracefully stop (SIGTERM)
docker start <container>              # Start a stopped container
docker restart <container>            # Restart
docker rm <container>                 # Remove a stopped container
docker rm -f <container>              # Force-kill and remove (SIGKILL)
docker logs <container>               # View container logs
docker logs -f <container>            # Follow logs live
docker exec -it <container> bash      # Shell into a running container
docker inspect <container>            # Full JSON metadata
```

More: [Containers](docker-containers.md)

## Debugging

```bash
docker inspect <container> --format='{{.State.ExitCode}}'     # Exit code
docker inspect <image> --format='{{.Config.Entrypoint}}'      # What got baked in
docker run -it --rm --entrypoint sh <image>                   # Shell into a broken image
```

More: [Troubleshooting](docker-troubleshooting.md)

## Volumes

```bash
docker volume ls                      # List volumes
docker volume create <name>           # Create a named volume
docker volume inspect <name>          # Inspect a volume
docker volume rm <name>               # Remove a volume
docker run -v <name>:/path <image>    # Mount a named volume
docker run -v $(pwd):/path <image>    # Bind mount current dir (Linux/macOS/Git Bash)
```

On Windows PowerShell use `${PWD}` instead of `$(pwd)`; in cmd use `%cd%`.

More: [Volumes](docker-volumes.md)

## Networks

```bash
docker network ls                       # List networks
docker network create <name>            # Create a custom network
docker network inspect <name>           # Inspect a network
docker network connect <net> <ctr>      # Connect a running container to a network
docker run --network <net> <image>      # Start a container on a network
```

More: [Networking](docker-networking.md)

## Docker Compose

```bash
docker compose up                     # Start services defined in the Compose file
docker compose up -d                  # Start detached
docker compose down                   # Stop and remove containers and networks
docker compose down -v                # Also remove volumes (clean reset)
docker compose build                  # Build or rebuild services
docker compose logs -f                # Follow logs for all services
docker compose ps                     # List services in this project
docker compose exec <service> sh      # Shell into a running service
docker compose run --rm <service> sh  # Fresh one-off container
docker compose config                 # Render final YAML with variables resolved
```

More: [Compose](docker-compose.md)

## Cleanup

```bash
docker system df                      # Disk usage summary
docker system prune                   # Remove unused containers, networks and dangling images
docker container prune                # Remove stopped containers
docker image prune                    # Remove dangling images
docker volume prune                   # Remove unused volumes (deletes data)
```

---

[← Back to Docker index](README.md)
