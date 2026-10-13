# Docker CLI Overview

## Introduction

The Docker CLI is your primary tool for working with Docker. Every time you type a command starting with `docker`, you are using the CLI. Understanding it deeply — not just memorizing commands — makes you far more effective and helps you troubleshoot problems quickly.

This file covers the structure of the Docker CLI, how commands are organized, the most important flags that apply everywhere, and how to read Docker's output so you always know exactly what is happening.

---

## The Structure of Docker Commands

Docker CLI commands follow a consistent pattern:

```
docker [OPTIONS] COMMAND [ARGUMENTS]
```

For example:

```bash
docker run -d -p 8080:80 --name webserver nginx
#      ^^^  ^  ^^^^^^^^^  ^^^^^^^^^^^^^^^^  ^^^^^
#       |   |      |             |           image
#      cmd  |      |           name flag
#           |    port mapping
#       detach flag
```

Starting from Docker 17.09, commands are organized into **management commands** and **subcommands**:

```
docker MANAGEMENT-COMMAND SUBCOMMAND [OPTIONS] [ARGUMENTS]
```

For example:

```bash
docker container run     # management command: container, subcommand: run
docker image build       # management command: image, subcommand: build
docker volume ls         # management command: volume, subcommand: ls
```

The older short-form commands still work:

```bash
docker run    # same as docker container run
docker build  # same as docker image build
docker ps     # same as docker container ls
```

Both forms are valid. You will see both in real-world codebases. This course uses the short forms for speed, but knowing the management command structure helps you discover commands with `docker --help`.

---

## Getting Help

The Docker CLI has built-in help for every command. This is the most important thing to know — you never need to guess syntax.

```bash
# General help — lists all top-level commands
docker --help

# Help for a specific management command
docker container --help

# Help for a specific subcommand
docker container run --help
docker image build --help
docker volume ls --help
```

The help output shows you every available flag, its type (string, int, bool), and its description. When you are unsure about a flag, `--help` is always the fastest answer.

```bash
# Example: find out how to run a container with resource limits
docker container run --help | grep -i memory

# Output:
  -m, --memory bytes                    Memory limit
      --memory-reservation bytes        Memory soft limit
      --memory-swap bytes               Swap limit equal to memory plus swap: '-1' to enable unlimited swap
```

---

## Docker CLI Management Commands

Docker groups related commands under management categories:

| Management Command | What it manages |
|---|---|
| `docker container` | Containers |
| `docker image` | Images |
| `docker volume` | Volumes |
| `docker network` | Networks |
| `docker system` | Docker system resources |
| `docker compose` | Multi-container applications |
| `docker buildx` | Extended image building |
| `docker context` | Docker daemon connections |

Each management command has subcommands. You can discover them all with `docker COMMAND --help`.

---

## Working with Containers

Containers are the core of Docker. These are the commands you will use most frequently.

### Running Containers

```bash
# Run a container (pulls image if not local)
docker run nginx

# Run in detached mode (background) — you get your terminal back
docker run -d nginx

# Run with a custom name
docker run -d --name webserver nginx

# Run with port mapping (host:container)
docker run -d -p 8080:80 --name webserver nginx

# Run and remove automatically when it stops
docker run --rm alpine echo "Hello, Docker!"

# Run interactively with a shell
docker run -it ubuntu bash

# Run with environment variables
docker run -d \
  -e DATABASE_URL=postgresql://user:pass@db:5432/myapp \
  -e APP_ENV=production \
  --name api myapp:1.0
```

### Listing Containers

```bash
# List running containers
docker ps

# List all containers (running + stopped)
docker ps -a

# List only container IDs (useful in scripts)
docker ps -q

# List all container IDs (running + stopped)
docker ps -aq

# Custom output format
docker ps --format "table {{.ID}}\t{{.Names}}\t{{.Status}}\t{{.Ports}}"
```

### Stopping and Starting Containers

```bash
# Stop a container gracefully (sends SIGTERM, waits 10s, then SIGKILL)
docker stop webserver

# Stop immediately (sends SIGKILL)
docker kill webserver

# Start a stopped container
docker start webserver

# Restart a container
docker restart webserver

# Stop all running containers
docker stop $(docker ps -q)
```

### Removing Containers

```bash
# Remove a stopped container
docker rm webserver

# Force remove a running container
docker rm -f webserver

# Remove all stopped containers
docker container prune

# Remove all stopped containers (without confirmation prompt)
docker container prune -f

# Remove specific containers by ID
docker rm abc123 def456
```

---

## Working with Images

### Listing and Inspecting Images

```bash
# List all local images
docker images

# List images with a specific name
docker images nginx

# List only image IDs
docker images -q

# Show full image details
docker inspect nginx:latest

# Show image history (layers)
docker history nginx:latest
```

### Pulling and Pushing Images

```bash
# Pull an image from Docker Hub
docker pull nginx

# Pull a specific version
docker pull nginx:1.25.3

# Pull from AWS ECR
docker pull 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:1.0

# Push to Docker Hub
docker push yourusername/myapp:1.0

# Push to AWS ECR
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:1.0
```

### Tagging Images

```bash
# Create a new tag for an existing image
docker tag myapp:1.0 myapp:latest
docker tag myapp:1.0 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:1.0

# Tag with the Git commit hash (great for traceability)
docker tag myapp:build myapp:$(git rev-parse --short HEAD)
```

### Removing Images

```bash
# Remove an image
docker rmi nginx

# Force remove (even if containers reference it)
docker rmi -f nginx

# Remove all unused images (not referenced by any container)
docker image prune

# Remove ALL images including referenced ones
docker image prune -a
```

### Building Images

```bash
# Build from Dockerfile in current directory
docker build -t myapp:1.0 .

# Build from a specific Dockerfile
docker build -f Dockerfile.prod -t myapp:prod .

# Build with build arguments
docker build \
  --build-arg APP_VERSION=1.0 \
  --build-arg ENV=production \
  -t myapp:1.0 .

# Build without using cache (full rebuild)
docker build --no-cache -t myapp:1.0 .
```

---

## Interacting with Running Containers

### Viewing Logs

```bash
# View container logs
docker logs webserver

# Follow logs in real time (like tail -f)
docker logs -f webserver

# Show last 100 lines
docker logs --tail 100 webserver

# Show logs with timestamps
docker logs -t webserver

# Follow logs since a specific time
docker logs --since 2024-01-15T09:00:00 -f webserver
```

### Running Commands Inside Containers

```bash
# Execute a command in a running container
docker exec webserver ls /etc/nginx

# Open an interactive shell in a running container
docker exec -it webserver bash

# Open a shell in an Alpine-based container (no bash, use sh)
docker exec -it myalpinecontainer sh

# Run a command as a specific user
docker exec -it --user root webserver bash
```

### Copying Files

```bash
# Copy a file from container to host
docker cp webserver:/etc/nginx/nginx.conf ./nginx.conf

# Copy a file from host to container
docker cp ./nginx.conf webserver:/etc/nginx/nginx.conf
```

---

## Inspecting and Monitoring

### Inspect Details

```bash
# Full JSON details of a container
docker inspect webserver

# Extract a specific field using Go template
docker inspect --format '{{.State.Status}}' webserver
docker inspect --format '{{.NetworkSettings.IPAddress}}' webserver
docker inspect --format '{{.HostConfig.Memory}}' webserver
```

### Monitor Resource Usage

```bash
# Show live resource usage for all containers
docker stats

# Show resource usage once (no live update)
docker stats --no-stream

# Show usage for specific containers
docker stats webserver api-server db
```

Output:

```
CONTAINER ID   NAME        CPU %     MEM USAGE / LIMIT   MEM %     NET I/O       BLOCK I/O
7d3c8e1f4a2b   webserver   0.01%     8.5MiB / 7.77GiB    0.11%     1.2kB / 648B  0B / 0B
```

### View Running Processes Inside a Container

```bash
# Show processes running inside a container
docker top webserver
```

---

## System Management

### Disk Usage

```bash
# Show overall Docker disk usage
docker system df

# Show detailed disk usage
docker system df -v
```

Output:

```
TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
Images          8         5         2.1GB     890MB (42%)
Containers      3         2         45MB      12MB (26%)
Local Volumes   4         2         890MB     400MB (44%)
Build Cache     15        0         450MB     450MB
```

### Cleaning Up

```bash
# Remove all stopped containers, unused networks, dangling images, and build cache
docker system prune

# Same but also remove unused images (not just dangling)
docker system prune -a

# Skip confirmation prompt
docker system prune -af

# Remove only build cache
docker builder prune
```

---

## Practical Patterns

### Find the IP of a Container

```bash
docker inspect --format '{{.NetworkSettings.IPAddress}}' webserver
```

### Wait for a Container to Be Healthy Before Proceeding

```bash
# Start a database container
docker run -d --name db \
  -e POSTGRES_PASSWORD=pass \
  postgres:16

# Wait until it is accepting connections
until docker exec db pg_isready -U postgres; do
  echo "Waiting for database..."
  sleep 2
done

echo "Database is ready"
```

### Remove All Containers (Clean Slate)

```bash
# Stop all running containers first
docker stop $(docker ps -q)

# Remove all containers
docker rm $(docker ps -aq)
```

### Debug a Failed Container

```bash
# If a container exits immediately, check its logs
docker logs myapp

# Override the entrypoint to get a shell instead
docker run -it --entrypoint bash myapp:1.0

# Run the failed container again with interactive shell
docker run -it --rm myapp:1.0 bash
```

---

## Summary

- Docker CLI commands follow the pattern `docker [OPTIONS] COMMAND [ARGUMENTS]`
- Use `docker --help` and `docker COMMAND --help` to discover commands and flags — never guess syntax
- The most used container commands are: `run`, `ps`, `stop`, `start`, `rm`, `logs`, `exec`, and `inspect`
- The most used image commands are: `pull`, `push`, `build`, `tag`, `images`, and `rmi`
- `docker system df` shows disk usage and `docker system prune` cleans up unused resources
- `docker stats` and `docker top` help you monitor and debug resource usage inside containers

In the next file, we provide a complete Docker CLI cheatsheet — a quick reference you can return to whenever you need to find the right command fast.
