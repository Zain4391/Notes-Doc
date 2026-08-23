# Docker Cheatsheet

Living reference. Updated every time we learn a new command. Grouped by category, not by lesson order.

---

## Setup & Info

```bash
docker --version              # Check installed version
docker info                   # System-wide info (containers, images, storage driver)
docker version                # Client + server (daemon) version details
```

---

## Images

```bash
docker images                 # List local images
docker pull <image>:<tag>     # Download an image from a registry
docker rmi <image>            # Remove an image
docker build -t <name>:<tag> .   # Build an image from a Dockerfile in current dir
docker tag <src> <target>     # Tag an image (e.g. for pushing to a registry)
docker push <image>           # Push an image to a registry
```

---

## Containers

```bash
docker run <image>                    # Create + start a container
docker run -d <image>                 # Run detached (background)
docker run -it <image> bash           # Run interactively with a shell
docker run -p 8080:80 <image>         # Map host:container port
docker run --name myapp <image>       # Give the container a name
docker ps                             # List running containers
docker ps -a                          # List all containers (incl. stopped)
docker stop <container>               # Gracefully stop
docker start <container>              # Start a stopped container
docker restart <container>            # Restart
docker rm <container>                 # Remove a stopped container
docker logs <container>               # View container logs
docker logs -f <container>            # Follow logs live
docker exec -it <container> bash      # Shell into a running container
docker inspect <container>            # Full JSON metadata
```

---

## Volumes

```bash
docker volume ls                      # List volumes
docker volume create <name>           # Create a named volume
docker volume inspect <name>          # Inspect a volume
docker volume rm <name>               # Remove a volume
docker run -v <name>:/path <image>    # Mount a named volume
docker run -v $(pwd):/path <image>    # Bind mount current dir (Linux/macOS syntax)
```

---

## Networks

```bash
docker network ls                     # List networks
docker network create <name>          # Create a custom network
docker network inspect <name>         # Inspect a network
docker network connect <net> <ctr>    # Connect a running container to a network
```

---

## Docker Compose

```bash
docker compose up                     # Start services defined in docker-compose.yml
docker compose up -d                  # Start detached
docker compose down                   # Stop and remove containers/networks
docker compose build                  # Build/rebuild services
docker compose logs -f                # Follow logs for all services
docker compose ps                     # List services in this project
```

---

## Cleanup

```bash
docker system df                      # Disk usage summary
docker system prune                   # Remove unused containers/networks/images
docker container prune                # Remove stopped containers
docker image prune                    # Remove dangling images
docker volume prune                   # Remove unused volumes
```
