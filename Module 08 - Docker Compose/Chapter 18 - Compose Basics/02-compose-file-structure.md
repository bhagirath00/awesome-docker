# Compose File Structure

## Introduction

The `docker-compose.yml` file is the single source of truth for your multi-container application. Understanding its structure — what goes at the top level, what goes inside a service definition, and how sections relate to each other — is the foundation of working effectively with Docker Compose.

This file walks through the complete structure of a Compose file, every major top-level section, and the most important fields within each.

---

## The Top-Level Structure

A Compose file has four main top-level sections:

```yaml
version: "3.8"       # Compose file format version

services:             # Container definitions (required)
  service-name:
    ...

volumes:              # Named volume declarations (optional)
  volume-name:
    ...

networks:             # Custom network declarations (optional)
  network-name:
    ...
```

Every Compose file must have `services`. The `volumes` and `networks` sections are optional — Compose creates a default network automatically if you do not define one.

---

## The version Field

The `version` field declares which version of the Compose file format you are using. Different versions support different features.

```yaml
version: "3.8"
```

In practice, most teams use `3.8` (the latest stable schema). Some newer Compose features do not require a version at all — the schema is now versionless and Compose infers the format. However, explicitly declaring `version: "3.8"` is widely used and safe.

---

## The services Section

The `services` block is where all containers are defined. Each key under `services` is the service name — this becomes the DNS hostname other services use to communicate with it.

```yaml
services:
  web:       # accessible as "web" via DNS
    ...
  api:       # accessible as "api" via DNS
    ...
  db:        # accessible as "db" via DNS
    ...
```

### Core Service Fields

#### image — Use a Pre-Built Image

```yaml
services:
  db:
    image: postgres:16
```

Pulls and runs the `postgres:16` image. The image must be available in a registry (Docker Hub, ECR, etc.) or already cached locally.

#### build — Build from a Dockerfile

```yaml
services:
  api:
    build:
      context: .             # directory containing the Dockerfile
      dockerfile: Dockerfile # default, can be customized
      args:
        APP_VERSION: "1.4.2"
        GIT_COMMIT: ${GIT_SHA}
```

Short form (context only):

```yaml
services:
  api:
    build: .
```

You can use `image` and `build` together — Compose builds the image from the Dockerfile and names it as specified in `image`:

```yaml
services:
  api:
    build: .
    image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:1.4.2
```

#### ports — Port Mapping

```yaml
services:
  api:
    ports:
      - "8080:8080"           # host:container
      - "127.0.0.1:5432:5432" # bind to specific interface
      - "9090:9090/udp"       # UDP
```

#### environment — Environment Variables

```yaml
services:
  api:
    environment:
      APP_ENV: production      # key: value format
      LOG_LEVEL: info
      DATABASE_URL: postgresql://apiuser:secret@db:5432/myapp
```

Or list format:

```yaml
    environment:
      - APP_ENV=production
      - LOG_LEVEL=info
```

Or pass through from the host shell (no value = use host's value):

```yaml
    environment:
      - AWS_ACCESS_KEY_ID     # inherits from host shell
      - AWS_SECRET_ACCESS_KEY
```

#### env_file — Load Environment from File

```yaml
services:
  api:
    env_file:
      - .env              # loaded first
      - .env.production   # overrides values from .env
```

#### volumes — Volume Mounts

```yaml
services:
  db:
    volumes:
      - pgdata:/var/lib/postgresql/data       # named volume
      - ./config/postgres.conf:/etc/postgresql/postgresql.conf:ro  # bind mount, read-only
  api:
    volumes:
      - uploads:/app/uploads                  # named volume
      - .:/app                                # bind mount source code (dev only)
```

#### command — Override the Default CMD

```yaml
services:
  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes    # override default redis command

  worker:
    image: myapp:1.0
    command: python worker.py --concurrency 4
```

#### entrypoint — Override the Default ENTRYPOINT

```yaml
services:
  migration:
    image: myapp:1.0
    entrypoint: python
    command: manage.py migrate
```

#### restart — Restart Policy

```yaml
services:
  api:
    restart: unless-stopped   # most services
  db:
    restart: always           # infrastructure
  migration:
    restart: "no"             # one-time tasks
```

#### depends_on — Startup Ordering

```yaml
services:
  api:
    depends_on:
      - db
      - redis
```

Basic `depends_on` only waits for the container to *start*, not for the service inside it to be *ready*. The advanced form with health check conditions is covered in the Compose Advanced chapter.

#### container_name — Fixed Container Name

```yaml
services:
  db:
    container_name: production-db   # overrides auto-generated name
```

Use sparingly — a fixed container name prevents running multiple instances of the same service.

#### working_dir — Override WORKDIR

```yaml
services:
  api:
    image: python:3.11-slim
    working_dir: /app
    command: python app.py
```

#### user — Run as Specific User

```yaml
services:
  api:
    user: "1001:1001"
```

#### deploy — Resource Limits (Compose v3 with Swarm)

```yaml
services:
  api:
    deploy:
      resources:
        limits:
          cpus: "1.5"
          memory: 512M
        reservations:
          memory: 256M
```

For standalone Compose (not Swarm), use `mem_limit` and `cpus` directly:

```yaml
services:
  api:
    mem_limit: 512m
    mem_reservation: 256m
    cpus: 1.5
```

#### logging — Log Driver Configuration

```yaml
services:
  api:
    logging:
      driver: json-file
      options:
        max-size: "50m"
        max-file: "3"
```

For AWS CloudWatch:

```yaml
services:
  api:
    logging:
      driver: awslogs
      options:
        awslogs-group: /myapp/production
        awslogs-region: us-east-1
        awslogs-stream-prefix: api
```

#### healthcheck — Container Health Checks

```yaml
services:
  api:
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
```

---

## The volumes Section

Volumes declared at the top level are created by Compose if they do not exist:

```yaml
volumes:
  pgdata:           # minimal — uses local driver with defaults
  redis-data:
    labels:
      app: myapp
  uploads:
    driver: local
    driver_opts:
      type: nfs
      o: "addr=10.0.0.50,rw"
      device: ":/exports/uploads"
```

If you want Compose to use an existing volume that it should not manage, declare it as external:

```yaml
volumes:
  pgdata:
    external: true    # Compose uses existing "pgdata" volume, does not create/delete it
```

---

## The networks Section

By default, Compose creates one network (`<project>_default`) and attaches all services to it. Define custom networks when you need isolation or special configuration:

```yaml
networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true    # no outbound internet access
  monitoring:
    driver: bridge
```

Attach services to specific networks:

```yaml
services:
  web:
    networks:
      - frontend

  api:
    networks:
      - frontend
      - backend

  db:
    networks:
      - backend
```

---

## A Complete, Real-World Compose File

```yaml
version: "3.8"

services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
      args:
        APP_VERSION: "1.4.2"
    image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:1.4.2
    restart: unless-stopped
    ports:
      - "8080:8080"
    environment:
      APP_ENV: production
      LOG_LEVEL: info
      DATABASE_URL: postgresql://apiuser:${DB_PASSWORD}@db:5432/myapp
      REDIS_URL: redis://redis:6379
    env_file:
      - .env
    volumes:
      - uploads:/app/uploads
    depends_on:
      - db
      - redis
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
    networks:
      - frontend
      - backend
    mem_limit: 512m
    cpus: 1.0

  db:
    image: postgres:16
    restart: always
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      POSTGRES_USER: apiuser
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: myapp
    ports:
      - "127.0.0.1:5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U apiuser -d myapp"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - backend
    mem_limit: 1g

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    volumes:
      - redis-data:/data
    command: redis-server --appendonly yes
    networks:
      - backend
    mem_limit: 256m

volumes:
  pgdata:
  redis-data:
  uploads:

networks:
  frontend:
  backend:
    internal: true
```

---

## Summary

- A Compose file has four top-level sections: `version`, `services` (required), `volumes`, and `networks`
- Each key under `services` is a service name that also becomes the DNS hostname for inter-container communication
- Use `image` to run a pre-built image or `build` to build from a Dockerfile — combine both to build and tag in one step
- `environment`, `env_file`, `volumes`, `ports`, `restart`, `depends_on`, and `healthcheck` are the core service fields you configure on almost every service
- Declare volumes in the top-level `volumes` section to make them named and persistent; use `external: true` to reference volumes managed outside Compose
- Declare networks in the top-level `networks` section to create isolated tiers; attach services with per-service `networks:` lists

In the next file, we cover all the Compose CLI commands — how to start, stop, scale, inspect, and debug multi-container applications from the terminal.
