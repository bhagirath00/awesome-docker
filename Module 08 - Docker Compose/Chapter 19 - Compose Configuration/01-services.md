# Services in Docker Compose

## Introduction

Every container in a Docker Compose application is defined as a service. A service definition is the complete specification for how a container should be created and run — its image, configuration, resources, networking, and behavior. Understanding every field available in a service definition lets you move beyond basic Compose files to precisely configured, production-ready application stacks.

This file is a deep-dive into the service definition — the most important section of any Compose file.

---

## The image Field

The `image` field specifies which Docker image to use for the service. It accepts any valid Docker image reference.

```yaml
services:
  # From Docker Hub (official image)
  db:
    image: postgres:16

  # Specific digest for exact reproducibility
  db-pinned:
    image: postgres@sha256:a1b2c3d4e5f6...

  # From AWS ECR
  api:
    image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:1.4.2

  # From a private registry
  internal-tool:
    image: registry.mycompany.com/tools/reporter:latest
```

---

## The build Field

When your service needs to be built from a Dockerfile, use `build` instead of (or alongside) `image`.

```yaml
services:
  api:
    # Simple form — build from Dockerfile in the current directory
    build: .

  api-advanced:
    build:
      context: ./api                  # directory sent to Docker daemon
      dockerfile: Dockerfile.prod     # specific Dockerfile filename
      args:                           # --build-arg values
        APP_VERSION: "1.4.2"
        GIT_COMMIT: ${GIT_SHA}
        BUILD_ENV: production
      target: production              # --target for multi-stage builds
      cache_from:                     # images to use as layer cache sources
        - 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:latest
      labels:
        app: myapp
        version: "1.4.2"
```

When you specify both `build` and `image`, Compose builds the image and names it with the value of `image`:

```yaml
services:
  api:
    build: .
    image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:1.4.2
```

Running `docker compose build api` produces the image, and `docker compose push api` pushes it to ECR.

---

## The command and entrypoint Fields

Override the CMD and ENTRYPOINT from the image:

```yaml
services:
  # Override command only (entrypoint from image is kept)
  worker:
    image: myapp:1.0
    command: python worker.py --concurrency 4

  # String form
  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes --maxmemory 256mb

  # List form (avoids shell interpretation)
  api:
    image: myapp:1.0
    command: ["gunicorn", "--workers", "4", "--bind", "0.0.0.0:8080", "app:app"]

  # Override both
  debug-server:
    image: myapp:1.0
    entrypoint: python
    command: -m debugpy --listen 0.0.0.0:5678 app.py
```

---

## The ports Field

```yaml
services:
  api:
    ports:
      - "8080:8080"                 # host_port:container_port
      - "127.0.0.1:5432:5432"      # bind to specific interface (security)
      - "53:53/udp"                 # UDP port
      - "8080"                      # random host port mapped to container 8080

  # Long form (more explicit)
  web:
    ports:
      - target: 80
        host_ip: "127.0.0.1"
        published: 8080
        protocol: tcp
        mode: host
```

---

## The expose Field

Documents ports the container uses internally without publishing them to the host. Used for inter-service communication where the port does not need to be accessible from outside Docker.

```yaml
services:
  api:
    expose:
      - "8080"    # other services can connect, host cannot (without -p)
```

---

## The volumes Field

```yaml
services:
  api:
    volumes:
      # Named volume (declared in top-level volumes:)
      - uploads:/app/uploads

      # Bind mount — host path to container path
      - ./config/app.yml:/app/config.yml:ro

      # Source code bind mount (development)
      - .:/app

      # Anonymous volume (shields /app/node_modules from bind mount above)
      - /app/node_modules

      # Long form
      - type: volume
        source: pgdata
        target: /var/lib/postgresql/data
        read_only: false

      - type: bind
        source: ./nginx.conf
        target: /etc/nginx/nginx.conf
        read_only: true
```

---

## The environment and env_file Fields

```yaml
services:
  api:
    environment:
      # Inline values
      APP_ENV: production
      PORT: "8080"
      # Reference from host shell (no value = use host's variable)
      AWS_REGION:
      # Compose variable substitution from .env file
      DATABASE_URL: postgresql://user:${DB_PASSWORD}@db:5432/myapp

    env_file:
      - .env              # base config
      - .env.production   # environment-specific overrides
```

---

## The networks Field

```yaml
services:
  api:
    networks:
      - frontend
      - backend

  # With network alias
  legacy-api:
    image: myapp:0.9
    networks:
      backend:
        aliases:
          - old-api    # other containers can reach this service as "legacy-api" OR "old-api"
```

---

## The restart Field

```yaml
services:
  api:
    restart: unless-stopped    # restart unless manually stopped
  db:
    restart: always            # always restart (survives daemon restart)
  migration:
    restart: "no"              # never restart
  worker:
    restart: on-failure        # restart only on non-zero exit
  batch-job:
    restart: on-failure:5      # restart up to 5 times on failure
```

---

## Resource Limits

For standalone Compose (without Swarm), use these top-level service fields:

```yaml
services:
  api:
    mem_limit: 512m          # hard memory limit
    mem_reservation: 256m   # soft memory reservation
    cpus: 1.5                # CPU limit (1.5 = 1.5 cores)
    mem_swappiness: 0        # disable swap

  # Equivalent using the deploy key (Compose v3 syntax, primarily for Swarm)
  api-swarm:
    deploy:
      resources:
        limits:
          cpus: "1.5"
          memory: 512M
        reservations:
          cpus: "0.5"
          memory: 256M
```

---

## The logging Field

```yaml
services:
  api:
    logging:
      driver: json-file
      options:
        max-size: "50m"
        max-file: "3"

  # CloudWatch Logs
  api-cloud:
    logging:
      driver: awslogs
      options:
        awslogs-group: /myapp/production
        awslogs-region: us-east-1
        awslogs-stream-prefix: api
        awslogs-create-group: "true"
```

---

## The labels Field

```yaml
services:
  api:
    labels:
      app: myapp
      version: "1.4.2"
      environment: production
      team: backend
      # Namespaced labels avoid key collisions
      com.mycompany.maintainer: platform-team@example.com
```

Labels appear on the running container and are queryable via `docker ps --filter label=`.

---

## The healthcheck Field

```yaml
services:
  api:
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s       # how often to check
      timeout: 10s        # max time for the check to complete
      retries: 3          # failures before marking unhealthy
      start_period: 40s   # grace period after startup before counting failures

  db:
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $$POSTGRES_USER -d $$POSTGRES_DB"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Disable health check from image
  redis:
    healthcheck:
      disable: true
```

---

## The depends_on Field

```yaml
services:
  api:
    depends_on:
      # Basic — wait for container to start
      db:
        condition: service_started

      # Wait for container to pass health check
      db:
        condition: service_healthy

      # Wait for container to complete (useful for migrations)
      migration:
        condition: service_completed_successfully
```

The `service_healthy` condition requires the dependency to have a `healthcheck` defined.

---

## The profiles Field

Profiles let you optionally include services — only started when the profile is activated:

```yaml
services:
  api:
    image: myapp:1.0

  debug-tools:
    image: debug-toolkit:1.0
    profiles:
      - debug     # only started when "debug" profile is active

  pgadmin:
    image: dpage/pgadmin4
    profiles:
      - tools     # only started when "tools" profile is active
```

```bash
# Start api only (no profiles activated)
docker compose up -d api

# Start api + debug-tools
docker compose --profile debug up -d

# Start api + pgadmin
docker compose --profile tools up -d
```

---

## A Complete Service Definition Example

```yaml
version: "3.8"

services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
      target: production
      args:
        APP_VERSION: "1.4.2"
    image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:1.4.2
    command: ["gunicorn", "--workers", "4", "--bind", "0.0.0.0:8080", "app:app"]
    restart: unless-stopped
    ports:
      - "8080:8080"
    environment:
      APP_ENV: production
      PORT: "8080"
      DATABASE_URL: postgresql://apiuser:${DB_PASSWORD}@db:5432/myapp
    env_file:
      - .env
    volumes:
      - uploads:/app/uploads
    networks:
      - frontend
      - backend
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    logging:
      driver: json-file
      options:
        max-size: "50m"
        max-file: "3"
    mem_limit: 512m
    cpus: 1.5
    labels:
      app: myapp
      version: "1.4.2"
      environment: production
```

---

## Summary

- `image` or `build` (or both) is the only truly required service field — everything else is optional
- `build` supports `context`, `dockerfile`, `args`, `target`, and `cache_from` for precise control over the build process
- `command` and `entrypoint` override the image's CMD and ENTRYPOINT — use list form to avoid shell interpretation issues
- Resource limits (`mem_limit`, `cpus`) belong on the service directly in standalone Compose; use the `deploy.resources` key for Swarm
- `depends_on` with `condition: service_healthy` is the correct way to wait for a dependency to be ready — it requires the dependency to have a `healthcheck`
- `profiles` let you include optional services that are only started when the matching profile is activated

In the next file, we cover environment variables in Compose in depth — the `.env` file, variable substitution, and secure handling of secrets.
