# depends_on in Docker Compose

## Introduction

In a multi-container application, services have dependencies. Your API cannot serve requests if the database is not ready. Your worker cannot consume jobs if the message queue is not running. Docker Compose provides `depends_on` to express these relationships and control startup order.

However, `depends_on` is widely misused. The simple form only waits for the container to *start* — it does not wait for the service *inside* the container to be *ready*. This file explains the correct way to use `depends_on`, the difference between container start and service readiness, and the patterns that actually work in production.

---

## The Simple Form — Container Start

In its basic form, `depends_on` lists the services that must be *started* before the current service:

```yaml
services:
  api:
    image: myapp:1.0
    depends_on:
      - db
      - redis
```

This tells Compose: start `db` and `redis` before starting `api`. But "starting" means the container has launched — not that PostgreSQL is accepting connections or Redis is ready to handle requests.

PostgreSQL typically takes 2-5 seconds to initialize after the container starts. If `api` connects to the database in its startup code, it may attempt to connect before PostgreSQL is ready and fail.

The simple `depends_on` form is sufficient only when your application has proper retry logic (exponential backoff on connection failure) or when you know the dependency starts fast enough.

---

## The Condition Form — Service Readiness

The full `depends_on` syntax with conditions lets you wait for actual readiness, not just container start:

```yaml
services:
  api:
    image: myapp:1.0
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started    # start is enough for redis
```

### Three Conditions

| Condition | Meaning |
|---|---|
| `service_started` | Container has started (same as simple form) |
| `service_healthy` | Container's healthcheck is passing |
| `service_completed_successfully` | Container has exited with code 0 |

---

## service_healthy — The Production Pattern

`service_healthy` waits until the dependency's health check reports healthy. This is the correct way to wait for a database to be ready.

It requires the dependency to have a `healthcheck` defined:

```yaml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_USER: apiuser
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: myapp
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U apiuser -d myapp"]
      interval: 5s       # check every 5 seconds
      timeout: 5s        # check must complete in 5 seconds
      retries: 5         # mark unhealthy after 5 consecutive failures
      start_period: 10s  # wait 10 seconds before starting checks

  redis:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

  api:
    image: myapp:1.0
    depends_on:
      db:
        condition: service_healthy   # waits until pg_isready succeeds
      redis:
        condition: service_healthy   # waits until redis-cli ping succeeds
```

With `condition: service_healthy`:
1. Compose starts `db` and `redis` containers
2. Compose watches their health check status
3. Once both report `healthy`, Compose starts `api`

PostgreSQL will be accepting connections by the time `api` starts. No retry logic required in the application — though adding it is still good practice for resilience.

### Common Healthcheck Commands

**PostgreSQL:**
```yaml
healthcheck:
  test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-postgres} -d ${POSTGRES_DB:-postgres}"]
  interval: 5s
  timeout: 5s
  retries: 5
  start_period: 10s
```

**MySQL/MariaDB:**
```yaml
healthcheck:
  test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-p${MYSQL_ROOT_PASSWORD}"]
  interval: 5s
  timeout: 5s
  retries: 5
  start_period: 15s
```

**Redis:**
```yaml
healthcheck:
  test: ["CMD", "redis-cli", "ping"]
  interval: 5s
  timeout: 3s
  retries: 5
```

**RabbitMQ:**
```yaml
healthcheck:
  test: ["CMD", "rabbitmq-diagnostics", "check_port_connectivity"]
  interval: 10s
  timeout: 10s
  retries: 5
  start_period: 30s
```

**Generic HTTP:**
```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
  interval: 30s
  timeout: 10s
  retries: 3
  start_period: 40s
```

---

## service_completed_successfully — Database Migrations

The `service_completed_successfully` condition waits for a service to exit with code 0. This is perfect for running database migrations before starting the application:

```yaml
services:
  db:
    image: postgres:16
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U apiuser -d myapp"]
      interval: 5s
      timeout: 5s
      retries: 5

  migration:
    image: myapp:1.0
    command: python manage.py migrate
    depends_on:
      db:
        condition: service_healthy   # wait for DB to be ready first
    restart: "no"    # never restart — run once and exit

  api:
    image: myapp:1.0
    depends_on:
      db:
        condition: service_healthy
      migration:
        condition: service_completed_successfully   # migrations must finish first
    restart: unless-stopped
```

Startup order:
1. `db` starts, health check begins
2. Once `db` is healthy, `migration` starts
3. `migration` runs `python manage.py migrate` and exits with code 0
4. Once `migration` completes successfully, `api` starts
5. `api` is guaranteed to find a ready, up-to-date database

If the migration fails (exit code ≠ 0), `api` never starts — Compose waits indefinitely. You will see the failure in `docker compose logs migration`.

---

## restart: on

The `restart` field combines with `depends_on` to control failure behavior:

```yaml
services:
  migration:
    image: myapp:1.0
    command: python manage.py migrate
    restart: on-failure:3    # retry up to 3 times if migration fails
    depends_on:
      db:
        condition: service_healthy
```

---

## depends_on Does Not Guarantee Readiness Without healthcheck

A common mistake: setting `condition: service_healthy` without defining a healthcheck on the dependency.

```yaml
services:
  db:
    image: postgres:16
    # NO healthcheck defined

  api:
    depends_on:
      db:
        condition: service_healthy   # this is wrong — db has no healthcheck
```

Without a healthcheck, Compose cannot determine if `db` is healthy. It will wait indefinitely and your deployment will hang.

Always define a healthcheck on any service used as a `service_healthy` dependency.

---

## Checking Startup Order in Practice

```bash
# Watch all service events as they start
docker compose up --no-start   # create containers without starting
docker compose start            # start in dependency order

# Or just watch the logs during startup
docker compose up 2>&1 | head -100

# Output shows:
# ✔ Container myapp-db-1        Created
# ✔ Container myapp-redis-1     Created
# ✔ Container myapp-migration-1 Created
# ✔ Container myapp-api-1       Created
# Attaching to db-1, redis-1, migration-1, api-1
# db-1 | database system is ready to accept connections
# db-1 | database system is ready to accept connections (the healthcheck passes)
# migration-1 | Running migrations...
# migration-1 | Successfully applied 4 migrations.
# migration-1 exited with code 0
# api-1 | Server started on 0.0.0.0:8080
```

---

## When depends_on is Not Enough

`depends_on` with health conditions handles most startup ordering needs. But it does not help with:

**Runtime failures** — if the database crashes after the API starts, `depends_on` does nothing. Your application must handle reconnection independently.

**Partial availability** — in a scaled deployment where `api` has 3 replicas, `depends_on` does not coordinate between API instances.

**Production orchestration** — in AWS ECS, startup ordering is handled differently. ECS does not have `depends_on`. Instead:
- Use `startPeriod` in health checks to give containers time to initialize
- Design applications to retry connections at startup
- Use `dependsOn` in ECS task definitions (for essential container ordering within a task)

The ECS pattern is application-level retry:

```python
# Python: retry database connection at startup with exponential backoff
import time
import psycopg2

def connect_with_retry(dsn, max_retries=10, base_delay=1):
    for attempt in range(max_retries):
        try:
            conn = psycopg2.connect(dsn)
            print(f"Connected to database on attempt {attempt + 1}")
            return conn
        except psycopg2.OperationalError as e:
            if attempt < max_retries - 1:
                delay = base_delay * (2 ** attempt)
                print(f"Connection failed (attempt {attempt + 1}): {e}. Retrying in {delay}s...")
                time.sleep(delay)
            else:
                raise

db = connect_with_retry(os.environ["DATABASE_URL"])
```

This pattern works with or without Compose's `depends_on` and is essential for ECS and Kubernetes deployments.

---

## Summary

- Basic `depends_on` (list form) only waits for the container to start — not for the service inside it to be ready
- Use `condition: service_healthy` with a `healthcheck` defined on the dependency to wait for actual service readiness
- Use `condition: service_completed_successfully` to wait for a one-time task like database migrations to finish before starting the main application
- The migration → api startup pattern is a clean, reliable way to guarantee schema consistency at startup
- Always define a `healthcheck` on any service used as a `service_healthy` dependency — without it, Compose waits indefinitely
- In AWS ECS, `depends_on` does not exist — design applications to retry connections at startup with exponential backoff as the primary resilience mechanism

In the next chapter, we cover advanced Compose features — override files, profiles, and multi-environment configuration patterns.
