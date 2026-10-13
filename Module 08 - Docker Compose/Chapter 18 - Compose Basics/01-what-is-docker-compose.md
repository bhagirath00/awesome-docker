# What is Docker Compose

## Introduction

Running a single container is straightforward. Running a real application is not. A production web application typically involves a web server, an API, a database, a cache, a message queue, a background worker, and monitoring agents. Starting each container individually — with the right flags, the right network, the right volumes, the right environment variables, in the right order — is tedious, error-prone, and impossible to reproduce reliably.

Docker Compose solves this. It is a tool that lets you define and run multi-container applications using a single YAML configuration file. You describe every service, network, and volume your application needs in that file, and Docker Compose handles the rest.

---

## The Problem Compose Solves

Without Compose, running a Python web app with PostgreSQL and Redis requires:

```bash
# Create the network
docker network create myapp-net

# Create volumes
docker volume create pgdata
docker volume create redis-data

# Start PostgreSQL
docker run -d \
  --name db \
  --network myapp-net \
  --restart unless-stopped \
  -v pgdata:/var/lib/postgresql/data \
  -e POSTGRES_USER=apiuser \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_DB=myapp \
  postgres:16

# Start Redis
docker run -d \
  --name redis \
  --network myapp-net \
  --restart unless-stopped \
  -v redis-data:/data \
  redis:7-alpine \
  redis-server --appendonly yes

# Start the API (after db and redis are ready)
docker run -d \
  --name api \
  --network myapp-net \
  --restart unless-stopped \
  -p 8080:8080 \
  -e DATABASE_URL=postgresql://apiuser:secret@db:5432/myapp \
  -e REDIS_URL=redis://redis:6379 \
  -e APP_ENV=production \
  myapp:1.4.2
```

You need to run this exact sequence every time. If you change a port, a password, or add a service, you have to manually update multiple commands. If a colleague wants to run this locally, they need all these commands too.

With Compose, all of this becomes a single file:

```yaml
# docker-compose.yml
version: "3.8"

services:
  db:
    image: postgres:16
    restart: unless-stopped
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      POSTGRES_USER: apiuser
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: myapp

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    volumes:
      - redis-data:/data
    command: redis-server --appendonly yes

  api:
    image: myapp:1.4.2
    restart: unless-stopped
    ports:
      - "8080:8080"
    environment:
      DATABASE_URL: postgresql://apiuser:secret@db:5432/myapp
      REDIS_URL: redis://redis:6379
      APP_ENV: production
    depends_on:
      - db
      - redis

volumes:
  pgdata:
  redis-data:
```

And a single command to start everything:

```bash
docker compose up -d
```

---

## What Compose Does Automatically

When you run `docker compose up`, Compose:

1. **Reads** `docker-compose.yml` in the current directory
2. **Creates a project network** — a custom bridge network named `<project>_default`, all services join it automatically
3. **Creates declared volumes** — named volumes defined in the `volumes:` block are created if they do not exist
4. **Starts services in dependency order** — if service B `depends_on` service A, Compose starts A first
5. **Pulls missing images** — if an image is not cached locally, Compose pulls it
6. **Names containers** — containers are named `<project>_<service>_<index>` (e.g., `myapp_db_1`)

All services on the project's network can resolve each other by service name. The `api` service connects to the database using the hostname `db` — Compose's DNS handles the rest.

---

## Compose vs docker run — When to Use Each

| Use `docker run` | Use `docker compose` |
|---|---|
| Quick one-off container tests | Multi-service applications |
| Running a single utility container | Development environments |
| CI/CD pipeline individual steps | Local dev with databases, caches |
| Debugging a container issue | Production single-host deployments |

**Rule of thumb:** Any time your application involves more than one container that needs to work together, use Docker Compose.

---

## The Compose File Location and Naming

By default, `docker compose` looks for a file named `docker-compose.yml` or `docker-compose.yaml` in the current directory. You can specify a different file with `-f`:

```bash
# Use the default docker-compose.yml
docker compose up -d

# Use a specific file
docker compose -f docker-compose.prod.yml up -d

# Use multiple files (merged together)
docker compose -f docker-compose.yml -f docker-compose.override.yml up -d
```

---

## Compose v1 vs Compose v2 (Modern Compose)

Docker Compose has two versions to be aware of:

**Compose v1** (`docker-compose` — note the hyphen)
- Written in Python, installed as a separate binary
- Command: `docker-compose up`
- Being phased out — EOL since July 2023

**Compose v2** (`docker compose` — note the space)
- Written in Go, integrated directly into the Docker CLI as a plugin
- Command: `docker compose up`
- Current standard — use this

```bash
# Check which version you have
docker compose version
# Docker Compose version v2.24.0

# v1 check (deprecated)
docker-compose --version
```

All examples in this module use Compose v2 (`docker compose`).

---

## Compose in Production

Docker Compose is most commonly used for development and single-host production deployments. For multi-host, highly-available production deployments, orchestrators like AWS ECS, Kubernetes (EKS), or Docker Swarm handle container scheduling, scaling, and health management.

That said, many small-to-medium scale production deployments run Docker Compose on a single EC2 instance and that is entirely legitimate. A well-configured Compose file with `restart: unless-stopped` on all services provides solid reliability for applications that do not need horizontal scaling.

```bash
# On an EC2 instance in production
git clone https://github.com/myorg/myapp.git
cd myapp

# Configure secrets via .env file (not committed to git)
cp .env.example .env
vim .env  # fill in production credentials

# Start the application
docker compose -f docker-compose.prod.yml up -d
```

---

## Summary

- Docker Compose is a tool for defining and running multi-container applications using a single YAML file (`docker-compose.yml`)
- It automates network creation, volume creation, dependency ordering, and container naming — everything you would do manually with multiple `docker run` commands
- `docker compose up -d` starts the entire application stack in the background; `docker compose down` stops and removes everything
- All services defined in the same Compose file automatically join a shared custom bridge network and can resolve each other by service name
- Use Compose v2 (`docker compose` with a space) — Compose v1 (`docker-compose` with a hyphen) is deprecated and end-of-life
- Compose is appropriate for development environments, single-host production, and any application that needs multiple cooperating containers

In the next file, we cover the structure of the Compose file in depth — every top-level key and what belongs inside each section.
