# Compose Override Files

## Introduction

A single `docker-compose.yml` file cannot cleanly serve all environments. Development needs source code bind mounts and debug ports. Staging needs specific resource limits and external service connections. Production needs exact image versions, logging to CloudWatch, and no debug tools.

Override files are Docker Compose's solution. You maintain a base `docker-compose.yml` with shared configuration, and environment-specific override files extend or modify it for each deployment context. Compose merges them at runtime.

---

## How Override Files Work

Compose merges multiple files together when you specify them with `-f`:

```bash
# Base + override merged together
docker compose -f docker-compose.yml -f docker-compose.override.yml up -d
```

The merge rules:
- Values in later files **override** values in earlier files
- Lists (like `ports`, `volumes`, `environment`) are **appended** — not replaced
- Mappings (like individual `environment` key-value pairs) are merged by key

---

## The Automatic docker-compose.override.yml

If a file named `docker-compose.override.yml` exists in the same directory as `docker-compose.yml`, Compose automatically merges it **without needing `-f`**. This is the conventional pattern for development overrides.

```bash
docker compose up -d   # automatically merges docker-compose.yml + docker-compose.override.yml
```

This makes development seamless: developers run `docker compose up -d` and get the development configuration automatically. Production deployments explicitly specify the production override file and skip the development defaults.

---

## A Real Example: Base + Dev + Prod

### Base File: docker-compose.yml

Contains configuration shared across all environments — service definitions, network structure, volume declarations:

```yaml
# docker-compose.yml — shared base configuration
version: "3.8"

services:
  api:
    image: myapp:${APP_VERSION:-latest}
    restart: unless-stopped
    depends_on:
      db:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    networks:
      - app-net

  db:
    image: postgres:16
    restart: always
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $$POSTGRES_USER -d $$POSTGRES_DB"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 15s
    networks:
      - app-net

volumes:
  pgdata:

networks:
  app-net:
```

Notice: no ports, no environment variables with secrets, no source code mounts. Those are all environment-specific.

### Development Override: docker-compose.override.yml

Auto-applied in development. Adds debug tools, source code mounts, and port exposure:

```yaml
# docker-compose.override.yml — development additions (auto-applied)
version: "3.8"

services:
  api:
    build:
      context: .
      target: development
    command: python -m debugpy --listen 0.0.0.0:5678 --wait-for-client app.py
    ports:
      - "8080:8080"
      - "5678:5678"   # debugger port
    volumes:
      - .:/app        # source code bind mount for live edits
      - /app/__pycache__
    environment:
      APP_ENV: development
      FLASK_DEBUG: "1"
      LOG_LEVEL: debug
      DATABASE_URL: postgresql://apiuser:devpassword@db:5432/myapp_dev

  db:
    ports:
      - "127.0.0.1:5432:5432"  # expose for local DB client
    environment:
      POSTGRES_USER: apiuser
      POSTGRES_PASSWORD: devpassword
      POSTGRES_DB: myapp_dev
```

```bash
# Development (auto-merges override)
docker compose up -d --build
```

### Production Override: docker-compose.prod.yml

Explicitly specified. No debug tools, CloudWatch logging, production secrets from environment:

```yaml
# docker-compose.prod.yml — production configuration
version: "3.8"

services:
  api:
    image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:${APP_VERSION}
    ports:
      - "8080:8080"
    environment:
      APP_ENV: production
      LOG_LEVEL: info
      DATABASE_URL: postgresql://apiuser:${DB_PASSWORD}@db:5432/myapp
    mem_limit: 512m
    cpus: 1.5
    logging:
      driver: awslogs
      options:
        awslogs-group: /myapp/production
        awslogs-region: us-east-1
        awslogs-stream-prefix: api
        awslogs-create-group: "true"

  db:
    environment:
      POSTGRES_USER: apiuser
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: myapp
    mem_limit: 2g
    logging:
      driver: awslogs
      options:
        awslogs-group: /myapp/production
        awslogs-region: us-east-1
        awslogs-stream-prefix: db
        awslogs-create-group: "true"
```

```bash
# Production deployment
export APP_VERSION=1.4.2
export DB_PASSWORD=$(aws secretsmanager get-secret-value ...)

docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

---

## The Merge Behavior in Detail

### Scalars and Mappings — Last Value Wins

```yaml
# docker-compose.yml
services:
  api:
    image: myapp:latest
    environment:
      LOG_LEVEL: info

# docker-compose.override.yml
services:
  api:
    image: myapp:dev      # overrides myapp:latest
    environment:
      LOG_LEVEL: debug    # overrides info
```

Result: `image: myapp:dev`, `LOG_LEVEL: debug`

### Lists — Appended

```yaml
# docker-compose.yml
services:
  api:
    ports:
      - "8080:8080"
    volumes:
      - uploads:/app/uploads

# docker-compose.override.yml
services:
  api:
    ports:
      - "5678:5678"      # appended, not replacing
    volumes:
      - .:/app           # appended
```

Result: `ports: ["8080:8080", "5678:5678"]`, `volumes: ["uploads:/app/uploads", ".:/app"]`

This list-append behavior is why adding ports in an override works — the base file's ports are preserved.

---

## Viewing the Merged Configuration

Before applying, you can see exactly what the merged Compose file looks like:

```bash
# Preview the merged configuration
docker compose config

# Preview production merge
docker compose -f docker-compose.yml -f docker-compose.prod.yml config

# Validate without running
docker compose -f docker-compose.yml -f docker-compose.prod.yml config --quiet
echo $?  # 0 = valid, 1 = error
```

Always run `docker compose config` to verify the merge result before deploying — especially for production.

---

## Typical Override File Structure in a Real Project

```
project/
├── docker-compose.yml          # base — shared across all environments
├── docker-compose.override.yml # dev — auto-applied, gitignored if sensitive
├── docker-compose.dev.yml      # alternate dev (when override is too generic)
├── docker-compose.staging.yml  # staging-specific config
├── docker-compose.prod.yml     # production config
├── docker-compose.test.yml     # CI test config
└── .env.example                # committed placeholder values
```

---

## Using Override Files in CI/CD

```bash
# CI: run tests with the test override
docker compose -f docker-compose.yml -f docker-compose.test.yml up -d
docker compose -f docker-compose.yml -f docker-compose.test.yml run --rm api pytest tests/

# Production deploy
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d

# Validate before deploy (runs in CI before deployment step)
docker compose -f docker-compose.yml -f docker-compose.prod.yml config --quiet \
  && echo "Config valid" \
  || (echo "Config invalid" && exit 1)
```

---

## Summary

- Override files allow you to maintain a shared base configuration and layer environment-specific additions or overrides on top
- `docker-compose.override.yml` is automatically merged when you run `docker compose up` — use this for development additions (source code mounts, debug ports)
- For staging, production, and CI, specify override files explicitly: `docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d`
- Scalar values are replaced by the last file's value; list values are appended — adding ports in an override adds to the base file's ports, not replaces them
- Always run `docker compose config` to preview the merged result before deploying — particularly important for production
- A typical project has: `docker-compose.yml` (base), `docker-compose.override.yml` (dev, auto-applied), `docker-compose.prod.yml` (production, explicit)

In the next file, we cover Compose profiles — a mechanism for optionally including services based on the active profile.
