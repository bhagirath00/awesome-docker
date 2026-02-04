# Docker CLI Cheatsheet

## Introduction

This file is your quick reference for Docker CLI commands. Bookmark it and come back whenever you need to find the right command fast.

Everything here is organized by task so you can scan to the section you need without reading through the entire file. Each command includes a brief explanation so you understand what it does, not just how to write it.

---

## Container Lifecycle Commands

```bash
# Create and start a container
docker run IMAGE

# Create and start in background (detached)
docker run -d IMAGE

# Create and start with interactive shell
docker run -it IMAGE bash

# Start a stopped container
docker start CONTAINER

# Stop a container (graceful — sends SIGTERM, waits 10s)
docker stop CONTAINER

# Stop immediately (sends SIGKILL)
docker kill CONTAINER

# Restart a container
docker restart CONTAINER

# Pause a container (freezes processes)
docker pause CONTAINER

# Unpause a container
docker unpause CONTAINER

# Remove a stopped container
docker rm CONTAINER

# Force remove a running container
docker rm -f CONTAINER

# Remove all stopped containers
docker container prune -f
```

---

## Running Containers — Common Flags

```bash
# --name         Give the container a name
docker run --name webserver nginx

# -d             Detached mode (background)
docker run -d nginx

# -it            Interactive + TTY (for shell access)
docker run -it ubuntu bash

# -p             Port mapping  HOST:CONTAINER
docker run -p 8080:80 nginx
docker run -p 8080:80 -p 8443:443 nginx

# -e             Environment variable
docker run -e APP_ENV=production myapp:1.0

# --env-file     Load environment variables from a file
docker run --env-file .env myapp:1.0

# -v             Volume mount  HOST_PATH:CONTAINER_PATH
docker run -v /data:/app/data myapp:1.0

# --rm           Remove container automatically when it exits
docker run --rm alpine echo "hello"

# --restart      Restart policy
docker run --restart always nginx         # Always restart
docker run --restart unless-stopped nginx  # Restart unless manually stopped
docker run --restart on-failure nginx      # Restart only on non-zero exit

# --network      Connect to a specific Docker network
docker run --network mynetwork nginx

# -m             Memory limit
docker run -m 512m nginx

# --cpus         CPU limit
docker run --cpus 1.5 nginx

# -w             Set working directory inside container
docker run -w /app -v $(pwd):/app python:3.11 python app.py

# -u             Run as specific user
docker run -u 1000:1000 myapp:1.0

# --entrypoint   Override the image entrypoint
docker run --entrypoint bash myapp:1.0
```

---

## Listing and Inspecting Containers

```bash
# List running containers
docker ps

# List all containers (running + stopped)
docker ps -a

# List only container IDs
docker ps -q

# List all container IDs (running + stopped)
docker ps -aq

# Custom table format
docker ps --format "table {{.ID}}\t{{.Names}}\t{{.Status}}\t{{.Image}}"

# Inspect full container details (JSON)
docker inspect CONTAINER

# Extract specific field
docker inspect --format '{{.State.Status}}' CONTAINER
docker inspect --format '{{.NetworkSettings.IPAddress}}' CONTAINER

# Show running processes in a container
docker top CONTAINER

# Show live resource usage
docker stats

# Show resource usage once (no live update)
docker stats --no-stream

# Show resource usage for specific containers
docker stats CONTAINER1 CONTAINER2
```

---

## Container Logs

```bash
# View all logs
docker logs CONTAINER

# Follow logs in real time
docker logs -f CONTAINER

# Show last N lines
docker logs --tail 100 CONTAINER

# Show logs with timestamps
docker logs -t CONTAINER

# Show logs since a specific time
docker logs --since 2024-01-15T09:00:00 CONTAINER

# Combine flags — last 50 lines with timestamps, live
docker logs --tail 50 -t -f CONTAINER
```

---

## Executing Commands in Containers

```bash
# Run a command in a running container
docker exec CONTAINER ls /app

# Open an interactive shell in a running container
docker exec -it CONTAINER bash

# Use sh if bash is not available (Alpine images)
docker exec -it CONTAINER sh

# Run as root inside the container
docker exec -it -u root CONTAINER bash

# Set environment variable for the exec session
docker exec -it -e DEBUG=true CONTAINER bash
```

---

## Copying Files

```bash
# Copy file from container to host
docker cp CONTAINER:/path/in/container ./local/path

# Copy file from host to container
docker cp ./local/file CONTAINER:/path/in/container

# Copy directory from container
docker cp CONTAINER:/app/logs ./local-logs/
```

---

## Image Commands

```bash
# List all local images
docker images

# List images matching a name
docker images nginx

# List all image IDs
docker images -q

# Pull an image
docker pull IMAGE:TAG

# Pull latest
docker pull nginx

# Pull specific version
docker pull nginx:1.25.3

# Push an image
docker push yourusername/myapp:1.0

# Tag an image
docker tag SOURCE_IMAGE:TAG TARGET_IMAGE:TAG

# Inspect image details
docker inspect IMAGE

# Show image build history (layers)
docker history IMAGE

# Remove an image
docker rmi IMAGE

# Force remove
docker rmi -f IMAGE

# Remove all unused images
docker image prune -a
```

---

## Building Images

```bash
# Build from Dockerfile in current directory
docker build -t myapp:1.0 .

# Build from a specific Dockerfile
docker build -f Dockerfile.prod -t myapp:prod .

# Build with build arguments
docker build --build-arg VERSION=1.0 -t myapp:1.0 .

# Build without cache
docker build --no-cache -t myapp:1.0 .

# Build and show full build output
docker build --progress=plain -t myapp:1.0 .

# Build for a specific platform
docker build --platform linux/amd64 -t myapp:1.0 .

# Multi-platform build with buildx
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t myapp:1.0 \
  --push \
  .
```

---

## Volume Commands

```bash
# List volumes
docker volume ls

# Create a named volume
docker volume create mydata

# Inspect a volume
docker volume inspect mydata

# Remove a volume
docker volume rm mydata

# Remove all unused volumes
docker volume prune -f

# Run container with a named volume
docker run -v mydata:/app/data myapp:1.0

# Run container with a bind mount (host directory)
docker run -v /host/path:/container/path myapp:1.0

# Read-only volume mount
docker run -v /host/path:/container/path:ro myapp:1.0
```

---

## Network Commands

```bash
# List networks
docker network ls

# Create a network
docker network create mynetwork

# Create a network with subnet
docker network create \
  --driver bridge \
  --subnet 192.168.1.0/24 \
  mynetwork

# Connect a running container to a network
docker network connect mynetwork CONTAINER

# Disconnect a container from a network
docker network disconnect mynetwork CONTAINER

# Inspect network details
docker network inspect mynetwork

# Remove a network
docker network rm mynetwork

# Remove all unused networks
docker network prune -f

# Run container on a specific network
docker run --network mynetwork myapp:1.0
```

---

## System and Cleanup Commands

```bash
# Show Docker system information
docker info

# Show Docker version
docker version

# Show disk usage
docker system df

# Show detailed disk usage
docker system df -v

# Remove all unused resources (stopped containers, unused networks, dangling images, build cache)
docker system prune

# Remove all unused resources including unused images
docker system prune -a

# Remove without confirmation
docker system prune -af

# Remove only build cache
docker builder prune -f
```

---

## Docker Compose Commands

```bash
# Start all services defined in docker-compose.yml
docker compose up

# Start in background
docker compose up -d

# Build images before starting
docker compose up --build

# Stop all services
docker compose down

# Stop and remove volumes
docker compose down -v

# View logs for all services
docker compose logs

# Follow logs
docker compose logs -f

# Logs for a specific service
docker compose logs -f api

# List running services
docker compose ps

# Execute command in a service container
docker compose exec api bash

# Scale a service (run N instances)
docker compose up -d --scale api=3

# Restart a specific service
docker compose restart api

# Pull latest images for all services
docker compose pull

# Build images without starting
docker compose build
```

---

## Registry Authentication

```bash
# Log in to Docker Hub
docker login

# Log in with credentials (non-interactive, useful in scripts)
echo "$DOCKER_PASSWORD" | docker login --username "$DOCKER_USERNAME" --password-stdin

# Log in to AWS ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789012.dkr.ecr.us-east-1.amazonaws.com

# Log out
docker logout
docker logout 123456789012.dkr.ecr.us-east-1.amazonaws.com
```

---

## Useful One-Liners

```bash
# Stop all running containers
docker stop $(docker ps -q)

# Remove all stopped containers
docker rm $(docker ps -aq)

# Remove all images
docker rmi $(docker images -q)

# Get the IP address of a container
docker inspect --format '{{.NetworkSettings.IPAddress}}' CONTAINER

# Get the container ID by name
docker ps -aqf "name=webserver"

# Follow logs of the most recently started container
docker logs -f $(docker ps -lq)

# Count running containers
docker ps -q | wc -l

# List containers with their image names and status
docker ps -a --format "table {{.Names}}\t{{.Image}}\t{{.Status}}"

# Find all containers using a specific image
docker ps -a --filter ancestor=nginx

# Remove containers that exited more than 1 hour ago
docker container prune --filter "until=1h"

# Show container resource usage sorted by CPU
docker stats --no-stream --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}" | sort -k2 -rh
```

---

## Flags Quick Reference

| Flag | Short | Description | Example |
|---|---|---|---|
| `--detach` | `-d` | Run in background | `docker run -d nginx` |
| `--interactive` | `-i` | Keep stdin open | `docker run -i ubuntu` |
| `--tty` | `-t` | Allocate a TTY | `docker run -it ubuntu bash` |
| `--name` | | Assign a name | `docker run --name web nginx` |
| `--publish` | `-p` | Map ports | `docker run -p 8080:80 nginx` |
| `--env` | `-e` | Set env variable | `docker run -e ENV=prod app` |
| `--volume` | `-v` | Mount volume | `docker run -v /data:/data app` |
| `--rm` | | Auto-remove on exit | `docker run --rm alpine sh` |
| `--network` | | Set network | `docker run --network mynet app` |
| `--memory` | `-m` | Memory limit | `docker run -m 512m app` |
| `--cpus` | | CPU limit | `docker run --cpus 1.5 app` |
| `--restart` | | Restart policy | `docker run --restart always app` |
| `--user` | `-u` | Run as user | `docker run -u 1000 app` |
| `--workdir` | `-w` | Working directory | `docker run -w /app app` |
| `--entrypoint` | | Override entrypoint | `docker run --entrypoint sh app` |

---

## Summary

- The Docker CLI follows a consistent pattern: `docker [OPTIONS] COMMAND [ARGUMENTS]`
- Container lifecycle commands: `run`, `start`, `stop`, `kill`, `restart`, `rm`, `pause`
- Inspection commands: `ps`, `logs`, `inspect`, `top`, `stats`, `exec`
- Image commands: `pull`, `push`, `build`, `tag`, `images`, `rmi`, `history`
- Management commands: `volume`, `network`, `system`, `compose`, `context`
- `docker system prune -a` is your cleanup command — use it regularly to reclaim disk space
- Always use `docker COMMAND --help` when you need to find a specific flag

In the next module, we move from setup into practice — working with containers and images in depth.
