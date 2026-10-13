# Container Naming

## Introduction

Container names seem like a minor detail, but they affect how you manage, monitor, and script your Docker environment. A well-named container is immediately identifiable in logs, monitoring dashboards, and operational scripts. A randomly named container (`focused_turing`, `quirky_kapitsa`) creates confusion when you are trying to debug a production incident at 2am.

This file covers container naming, labels, and how these two tools work together to organize containers at scale.

---

## Why Container Names Matter

Every container has an ID (a 64-character hex string) and a name. Docker uses both interchangeably in commands:

```bash
# Both of these work for the same container
docker stop a1b2c3d4ef56
docker stop webserver
```

The ID is guaranteed unique. The name is human-readable. Always use names in production because:

- Commands like `docker logs webserver` are instantly understandable
- Scripts that reference containers by name are self-documenting
- Monitoring dashboards show names in alerts and metrics
- Debugging during incidents is much faster with meaningful names

---

## How Docker Names Work

### Auto-Generated Names

Without `--name`, Docker generates a random two-word name combining an adjective and a famous scientist's name:

```bash
docker run -d nginx
# Container name: focused_turing
# or: quirky_kapitsa
# or: sleepy_hopper
```

These names are random, change every time you recreate the container, and make automation difficult.

### Custom Names with --name

```bash
# Give the container a specific name
docker run -d --name webserver nginx
docker run -d --name api-server myapp:1.0
docker run -d --name postgres-primary postgres:16
docker run -d --name redis-cache redis:7-alpine
```

Custom name rules:
- Must be unique on the host — only one container can have a given name at a time
- Can contain letters, numbers, hyphens (`-`), and underscores (`_`)
- Cannot start with a hyphen
- Case-sensitive

### Naming Conventions

Establish a consistent naming convention for your organization. Common patterns:

```bash
# Pattern: service-role
docker run --name api-primary myapp:1.0
docker run --name api-secondary myapp:1.0
docker run --name db-primary postgres:16
docker run --name db-replica postgres:16

# Pattern: environment-service
docker run --name prod-api myapp:1.0
docker run --name staging-api myapp:1.0

# Pattern: service-number (for multiple instances)
docker run --name api-1 myapp:1.0
docker run --name api-2 myapp:1.0
docker run --name api-3 myapp:1.0

# Pattern: descriptive-with-context
docker run --name payment-service-primary myapp:1.0
docker run --name user-auth-api myapp:1.0
```

---

## Name Conflicts

A container name must be unique on the host. If you try to reuse a name that exists (even for a stopped container), Docker returns an error:

```bash
docker run --name webserver nginx
# Runs successfully

docker run --name webserver nginx
# Error: Conflict. The container name "/webserver" is already in use by container "a1b2c3d4..."
# You have to remove (or rename) that container to be able to reuse that name.
```

### Handling Name Conflicts

```bash
# Remove the old container first
docker rm webserver

# Or force-remove if running
docker rm -f webserver

# Then recreate
docker run -d --name webserver nginx

# Or rename the existing container
docker rename webserver webserver-old
docker run -d --name webserver nginx
```

In CI/CD scripts, use a cleanup pattern to avoid conflicts:

```bash
# Stop and remove if exists, then start fresh
docker stop webserver 2>/dev/null || true
docker rm webserver 2>/dev/null || true
docker run -d --name webserver nginx
```

---

## Container Labels

Labels are key-value metadata that you attach to containers (and images). They serve as a flexible tagging system for filtering, querying, and organizing containers.

```bash
# Add labels at runtime
docker run -d \
  --name api \
  --label app=myapp \
  --label version=1.4.2 \
  --label environment=production \
  --label team=backend \
  --label service=api \
  myapp:1.4.2
```

Labels are defined as `key=value` pairs. The key can be namespaced with dots to avoid collisions:

```bash
docker run -d \
  --name api \
  --label com.mycompany.app=myapp \
  --label com.mycompany.version=1.4.2 \
  --label com.mycompany.environment=production \
  myapp:1.4.2
```

### Adding Labels in the Dockerfile

Labels added in the Dockerfile are baked into the image and appear on every container created from that image:

```dockerfile
LABEL com.mycompany.app="myapp" \
      com.mycompany.team="backend" \
      com.mycompany.maintainer="platform-team@example.com"
```

Runtime labels (`--label` in `docker run`) are applied on top of and can override image labels.

---

## Filtering Containers by Label

Labels become powerful when you use them to filter and select containers:

```bash
# List containers with a specific label
docker ps --filter label=environment=production

# List all production API containers
docker ps --filter label=app=myapp --filter label=environment=production

# List containers from a specific team
docker ps -a --filter label=team=backend

# Get container IDs matching a label (useful in scripts)
docker ps -q --filter label=environment=production

# Stop all containers with a specific label
docker stop $(docker ps -q --filter label=environment=staging)

# Remove all stopped containers for a specific app
docker rm $(docker ps -aq --filter label=app=myapp --filter status=exited)
```

### Inspecting Labels

```bash
# View all labels on a container
docker inspect --format '{{json .Config.Labels}}' webserver | python3 -m json.tool

# View a specific label value
docker inspect --format '{{index .Config.Labels "version"}}' webserver
# 1.4.2

# List all containers with their labels
docker ps --format "table {{.Names}}\t{{.Labels}}"
```

---

## Names in Docker Compose

Docker Compose automatically names containers using the pattern:

```
<project_name>_<service_name>_<instance_number>
```

```yaml
# docker-compose.yml
version: "3.8"
services:
  api:
    image: myapp:1.0
  db:
    image: postgres:16
```

Running in a directory called `myproject`:

```bash
docker compose up -d

docker ps
# NAMES
# myproject_api_1
# myproject_db_1
```

You can override the project name:

```bash
docker compose -p myapp up -d

docker ps
# NAMES
# myapp_api_1
# myapp_db_1
```

Or set a custom container name directly in the compose file:

```yaml
services:
  api:
    image: myapp:1.0
    container_name: api-server   # overrides auto-generated name
```

Note: Custom container names prevent scaling (running multiple instances of the same service) because names must be unique.

### Labels in Docker Compose

```yaml
version: "3.8"
services:
  api:
    image: myapp:1.0
    labels:
      - "app=myapp"
      - "environment=production"
      - "team=backend"
      - "com.mycompany.version=1.4.2"
```

---

## Container Names in Production on AWS

On AWS ECS, you define the container name in the task definition:

```json
{
  "family": "myapp-task",
  "containerDefinitions": [
    {
      "name": "api",
      "image": "123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:1.4.2",
      "dockerLabels": {
        "app": "myapp",
        "version": "1.4.2",
        "team": "backend",
        "environment": "production"
      }
    }
  ]
}
```

The container name in ECS appears in CloudWatch Logs (as the log stream prefix), in ECS console displays, and in CloudWatch Container Insights metrics. A meaningful name like `api` is much easier to work with than a numeric ID.

---

## docker rename — Renaming Containers

You can rename any container (running or stopped) without recreating it:

```bash
# Rename a running container
docker rename old-name new-name

# Rename to fix a mistake
docker rename api-servr api-server

# Rename to free up a name for a new container
docker rename webserver webserver-deprecated
docker run -d --name webserver nginx:1.25.3
```

---

## Summary

- Always name containers with `--name` in production — random auto-generated names (`focused_turing`) make management and debugging much harder
- Container names must be unique on the host — if a name is in use (even by a stopped container), remove or rename the existing container first
- Use a consistent naming convention: `service-role`, `environment-service`, or `service-number` patterns all work well
- Labels are key-value metadata for containers — use them to tag containers with app name, version, environment, and team
- Filter containers by label with `docker ps --filter label=key=value` — enables bulk operations like stopping all staging containers
- In Docker Compose, containers are named automatically as `project_service_number` — you can override this with `container_name` in the compose file
- On AWS ECS, the container name defined in the task definition appears in CloudWatch logs and metrics — use a meaningful name

In the next file, we cover restart policies — the rules that govern what Docker does when a container exits.
