# Compose Profiles

## Introduction

As applications grow, the `docker-compose.yml` accumulates more services — some needed for every developer, some needed only by specific teams, some needed only for specific tasks. Running everything every time wastes resources and slows startup.

Compose profiles let you tag services as optional. You start the core application with `docker compose up` and activate additional services only when you need them. This keeps the development environment lean while still making everything available.

---

## Defining Profiles on Services

Assign a service to a profile by adding the `profiles:` key:

```yaml
services:
  # Core services — always started (no profiles tag)
  api:
    image: myapp:1.0

  db:
    image: postgres:16

  # Optional services — only started when their profile is activated

  pgadmin:
    image: dpage/pgadmin4:latest
    profiles:
      - tools         # started when "tools" profile is active
    ports:
      - "5050:80"
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@example.com
      PGADMIN_DEFAULT_PASSWORD: admin

  redis-commander:
    image: rediscommander/redis-commander:latest
    profiles:
      - tools
    ports:
      - "8081:8081"

  debug-api:
    image: myapp:1.0
    profiles:
      - debug
    command: python -m debugpy --listen 0.0.0.0:5678 app.py
    ports:
      - "5678:5678"

  k6-load-test:
    image: grafana/k6:latest
    profiles:
      - testing
    volumes:
      - ./tests/load:/scripts
    command: run /scripts/api-load-test.js
```

---

## Running Profiles

```bash
# Start only core services (no profiles activated)
docker compose up -d
# Starts: api, db

# Start core + tools profile
docker compose --profile tools up -d
# Starts: api, db, pgadmin, redis-commander

# Start core + debug profile
docker compose --profile debug up -d
# Starts: api, db, debug-api

# Start multiple profiles simultaneously
docker compose --profile tools --profile debug up -d
# Starts: api, db, pgadmin, redis-commander, debug-api

# Using environment variable
COMPOSE_PROFILES=tools,debug docker compose up -d
```

---

## Profile Inheritance for Core Services

Services without a `profiles:` tag are always started regardless of active profiles. This is the correct pattern for core services.

If a core service depends on a profiled service, the profiled service is automatically started when the core service needs it:

```yaml
services:
  api:
    image: myapp:1.0
    depends_on:
      - db

  worker:
    image: myapp:1.0
    command: python worker.py
    profiles:
      - workers     # only start when "workers" profile is active
    depends_on:
      - db
      - redis

  db:
    image: postgres:16

  redis:
    image: redis:7-alpine
    profiles:
      - workers     # redis only needed when workers profile is active
```

```bash
# Core: api + db
docker compose up -d

# Workers stack: api + db + worker + redis
docker compose --profile workers up -d
```

---

## Real-World Profile Scenarios

### Monitoring Stack

```yaml
services:
  api:
    image: myapp:1.0

  prometheus:
    image: prom/prometheus:v2.48.0
    profiles:
      - monitoring
    ports:
      - "9090:9090"
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml:ro

  grafana:
    image: grafana/grafana:10.2.0
    profiles:
      - monitoring
    ports:
      - "3000:3000"
    volumes:
      - grafana-data:/var/lib/grafana
      - ./monitoring/dashboards:/etc/grafana/provisioning/dashboards:ro
    depends_on:
      - prometheus

  node-exporter:
    image: prom/node-exporter:v1.7.0
    profiles:
      - monitoring
    network_mode: host

volumes:
  grafana-data:
```

```bash
# Standard development
docker compose up -d

# Development with full monitoring
docker compose --profile monitoring up -d
```

### Database Management Tools

```yaml
services:
  db:
    image: postgres:16

  pgadmin:
    image: dpage/pgadmin4:8.2
    profiles:
      - db-tools
    ports:
      - "5050:80"
    environment:
      PGADMIN_DEFAULT_EMAIL: ${PGADMIN_EMAIL:-admin@dev.local}
      PGADMIN_DEFAULT_PASSWORD: ${PGADMIN_PASSWORD:-admin}
    volumes:
      - pgadmin-data:/var/lib/pgadmin
    depends_on:
      - db

  adminer:
    image: adminer:4.8.1
    profiles:
      - db-tools
    ports:
      - "8080:8080"

volumes:
  pgadmin-data:
```

### CI-Specific Services

```yaml
services:
  api:
    image: myapp:1.0

  test-runner:
    image: myapp:1.0
    profiles:
      - ci
    command: pytest tests/ -v --junitxml=/results/test-report.xml
    volumes:
      - ./test-results:/results
    environment:
      DATABASE_URL: postgresql://test:test@db:5432/myapp_test
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:16
    environment:
      POSTGRES_USER: test
      POSTGRES_PASSWORD: test
      POSTGRES_DB: myapp_test
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U test -d myapp_test"]
      interval: 5s
      timeout: 5s
      retries: 5
```

```bash
# CI pipeline
docker compose --profile ci run --rm test-runner
```

---

## Profiles and docker compose Commands

Profile activation applies to all compose commands:

```bash
# Stop the monitoring stack
docker compose --profile monitoring stop

# Remove monitoring containers
docker compose --profile monitoring down

# Check status with monitoring profile
docker compose --profile monitoring ps

# Exec into a profiled service
docker compose --profile tools exec pgadmin bash

# Logs from profiled service
docker compose --profile monitoring logs -f grafana
```

---

## Explicitly Starting a Profiled Service

You can start a specific profiled service by name without activating the profile globally:

```bash
# Start pgadmin without activating the full "tools" profile
docker compose up -d pgadmin
```

This is useful when you want just one optional service rather than the full profile.

---

## Profiles vs Override Files — When to Use Each

| Situation | Use Profiles | Use Override Files |
|---|---|---|
| Optional services (tools, monitoring) | ✅ | ❌ |
| Different service configuration per environment | ❌ | ✅ |
| Same service, different image or command | ❌ | ✅ |
| Adding ports for development only | ❌ | ✅ |
| Services only needed by specific team members | ✅ | ❌ |
| Development vs production resource limits | ❌ | ✅ |

Use profiles when you are **adding or removing services** based on context.
Use override files when you are **changing service configuration** based on environment.

They complement each other:

```bash
# Development: override file for dev config + profiles for optional tools
docker compose -f docker-compose.yml -f docker-compose.override.yml --profile tools up -d

# Production: prod override file, no profiles
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

---

## Listing Available Profiles

Compose does not have a built-in command to list all profiles in a file. You can find them with:

```bash
# Extract all profile names from the compose file
docker compose config | grep -A 999 "profiles:" | grep "^      - " | sort -u

# Or with yq (YAML processor)
yq '.services[].profiles[]' docker-compose.yml | sort -u
```

---

## Summary

- Profiles tag services as optional — services without a `profiles:` key always start; profiled services only start when their profile is activated
- Activate profiles with `--profile NAME` flag or the `COMPOSE_PROFILES` environment variable
- Use profiles for optional functionality: GUI database tools, monitoring stacks, load test runners, debug servers, CI-only services
- Multiple profiles can be active simultaneously — `--profile tools --profile monitoring` activates both
- Override files change service configuration per environment; profiles add or remove services — use both together for full flexibility
- A profiled service can still be started individually by name with `docker compose up -d SERVICE_NAME` without activating the whole profile

In the next file, we cover multi-environment Compose patterns — the complete approach to managing development, staging, and production from a single project.
