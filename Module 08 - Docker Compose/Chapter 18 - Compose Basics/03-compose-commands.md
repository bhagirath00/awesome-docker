# Compose Commands

## Introduction

The `docker compose` CLI is how you interact with your multi-container application. Knowing all the available commands — not just `up` and `down` — gives you precise control over your application stack. You can start individual services, watch logs from specific containers, scale services, run one-off tasks, and clean up completely.

This file covers every significant `docker compose` command with real usage patterns.

---

## docker compose up — Start the Stack

`docker compose up` creates and starts all containers defined in `docker-compose.yml`.

```bash
# Start all services in the foreground (logs stream to terminal)
docker compose up

# Start in the background (detached)
docker compose up -d

# Build images before starting (forces rebuild)
docker compose up -d --build

# Start specific services only
docker compose up -d api redis

# Pull latest images before starting
docker compose up -d --pull always

# Scale a service to multiple instances on startup
docker compose up -d --scale api=3

# Recreate containers even if config has not changed
docker compose up -d --force-recreate
```

### What `--build` Does

Without `--build`, Compose uses a locally cached image if available. With `--build`, Compose runs `docker build` before starting the container. Always use `--build` after changing your Dockerfile or source code.

```bash
# Typical development flow: rebuild and restart
docker compose up -d --build api
```

---

## docker compose down — Stop and Remove

`docker compose down` stops all running containers and removes them, their networks, and optionally their volumes and images.

```bash
# Stop containers and remove them (preserves volumes and images)
docker compose down

# Also remove named volumes declared in the volumes: section
docker compose down -v

# Also remove images built from the Compose file
docker compose down --rmi all

# Remove only locally built images (not pulled ones)
docker compose down --rmi local

# Remove orphan containers (containers for services no longer in the file)
docker compose down --remove-orphans
```

**Production rule:** Never run `docker compose down -v` on a production host without explicitly understanding which volumes it will delete. It removes all named volumes — including your database.

---

## docker compose start / stop / restart

These commands operate on already-created containers without removing or recreating them:

```bash
# Start all stopped services
docker compose start

# Start a specific service
docker compose start api

# Stop all running services (sends SIGTERM, waits for graceful shutdown)
docker compose stop

# Stop a specific service
docker compose stop db

# Restart all services
docker compose restart

# Restart a specific service with a stop timeout
docker compose restart -t 30 api
```

Use these when you want to stop or start services without changing their configuration. If you change `docker-compose.yml`, you need `docker compose up -d` to apply the changes.

---

## docker compose ps — List Running Services

```bash
# List all services and their status
docker compose ps

# Output:
NAME            IMAGE        COMMAND             SERVICE   STATUS        PORTS
myapp_api_1     myapp:1.4.2  "python app.py"     api       Up (healthy)  0.0.0.0:8080->8080/tcp
myapp_db_1      postgres:16  "docker-entrypoint"  db        Up (healthy)  127.0.0.1:5432->5432/tcp
myapp_redis_1   redis:7...   "docker-entrypoint"  redis     Up            6379/tcp

# Show all containers including stopped ones
docker compose ps -a

# Show only running service names
docker compose ps --services

# Filter by status
docker compose ps --status running
docker compose ps --status exited
```

---

## docker compose logs — View Output

```bash
# Show logs from all services
docker compose logs

# Follow logs in real time
docker compose logs -f

# Show logs from specific services
docker compose logs -f api db

# Show last 100 lines
docker compose logs --tail 100

# Show last 50 lines from api and follow
docker compose logs --tail 50 -f api

# Add timestamps
docker compose logs -t

# Since a specific time
docker compose logs --since 30m api
```

---

## docker compose exec — Run Commands in Containers

```bash
# Open a bash shell in the api container
docker compose exec api bash

# Run a specific command
docker compose exec api python manage.py migrate

# Run as root
docker compose exec -u root api bash

# Set environment variables for the exec session
docker compose exec -e DEBUG=true api python debug.py

# Run in a specific container when service is scaled
docker compose exec --index 2 api bash
```

---

## docker compose run — One-Off Task Containers

`docker compose run` starts a new container for a service and runs a specific command. Unlike `exec` (which runs inside an existing container), `run` creates a fresh container.

```bash
# Run a database migration
docker compose run --rm api python manage.py migrate

# Open a shell in a fresh container
docker compose run --rm api bash

# Run tests
docker compose run --rm api pytest tests/

# Run with a different command than the service default
docker compose run --rm api python scripts/seed_data.py

# Run without publishing ports (avoids port conflicts)
docker compose run --rm --no-deps api python manage.py check
```

The `--rm` flag removes the container after the command completes. Always use `--rm` for one-off tasks to avoid accumulating stopped containers.

The `--no-deps` flag skips starting the service's dependencies. Use this when the dependencies are already running.

---

## docker compose build — Build Images

```bash
# Build all services with a build: key
docker compose build

# Build a specific service
docker compose build api

# Build without using cache (forces full rebuild)
docker compose build --no-cache api

# Build and push to registry
docker compose build api
docker compose push api
```

---

## docker compose pull — Pull Images

```bash
# Pull all images referenced in the Compose file
docker compose pull

# Pull a specific service's image
docker compose pull db redis

# Pull in quiet mode (suppress progress output)
docker compose pull -q
```

---

## docker compose config — Validate and View Resolved Config

This command validates your Compose file and prints the fully resolved configuration (with environment variables substituted):

```bash
# Validate and show the resolved config
docker compose config

# Validate only (exit code 0 if valid, 1 if invalid)
docker compose config --quiet

# Show resolved environment variables only
docker compose config --environment

# Show just the service names
docker compose config --services

# Show just the volume names
docker compose config --volumes
```

Use `docker compose config --quiet` in CI/CD pipelines to validate the Compose file before attempting to deploy.

---

## docker compose top — Show Processes

Shows running processes inside all service containers:

```bash
docker compose top

# Output:
myapp_api_1
UID    PID    PPID   CMD
1001   1234   1233   python app.py
1001   1256   1234   gunicorn worker

myapp_db_1
UID    PID    PPID   CMD
999    2345   2344   postgres: checkpointer
999    2346   2344   postgres: background writer
```

---

## docker compose events — Real-Time Events

Stream events for all services in real time:

```bash
docker compose events

# Output (live stream):
2024-01-15T09:23:41.123Z myapp_api_1 start
2024-01-15T09:23:41.456Z myapp_db_1 health_status: healthy
2024-01-15T09:24:12.789Z myapp_api_1 die (exit code=1)
2024-01-15T09:24:13.012Z myapp_api_1 start (restart by policy)
```

---

## docker compose cp — Copy Files

```bash
# Copy a file from the api container to the host
docker compose cp api:/app/logs/error.log ./error.log

# Copy from host to container
docker compose cp ./updated-config.yml api:/app/config.yml
```

---

## docker compose scale — Scale Services

```bash
# Scale the api service to 3 instances
docker compose up -d --scale api=3

# Scale multiple services
docker compose up -d --scale api=3 --scale worker=2
```

When scaling, do not use `container_name` in the service definition — a fixed name prevents running multiple instances.

---

## Specifying the Project Name

The project name prefixes all resource names (containers, networks, volumes). By default it is the current directory name. You can override it:

```bash
# Set project name with -p flag
docker compose -p myapp up -d

# Set project name with environment variable
COMPOSE_PROJECT_NAME=myapp docker compose up -d

# Results in container names like:
# myapp_api_1, myapp_db_1, myapp_redis_1
```

---

## Specifying a Different Compose File

```bash
# Use a specific file
docker compose -f docker-compose.prod.yml up -d

# Merge multiple files
docker compose -f docker-compose.yml -f docker-compose.override.yml up -d

# Set via environment variable
COMPOSE_FILE=docker-compose.prod.yml docker compose up -d
```

---

## Common Workflows

### Development Start/Stop

```bash
# Start all services, rebuilding the app image
docker compose up -d --build

# Watch logs
docker compose logs -f api

# Stop everything, keep data
docker compose stop

# Resume
docker compose start
```

### Deploying an Update

```bash
# Pull new image
docker compose pull api

# Restart just the API with the new image
docker compose up -d api
```

### Complete Cleanup (Development Reset)

```bash
# Remove everything including volumes — fresh start
docker compose down -v --remove-orphans

# Start fresh
docker compose up -d
```

---

## Summary

- `docker compose up -d` starts all services in the background; add `--build` to rebuild images after code changes
- `docker compose down` stops and removes containers; add `-v` to also remove volumes (careful in production) and `--remove-orphans` to clean up stale containers
- `docker compose logs -f SERVICE` follows real-time output from specific services — your primary debugging tool
- `docker compose exec SERVICE bash` opens an interactive shell inside a running container
- `docker compose run --rm SERVICE COMMAND` creates a fresh one-off container for tasks like migrations and tests
- `docker compose config --quiet` validates the Compose file — use this in CI/CD before deploying
- `docker compose ps` shows the status of all services with their health check states and port mappings

In the next chapter, we cover Compose configuration in depth — services, environment variables, volumes, networks, and dependency ordering.
