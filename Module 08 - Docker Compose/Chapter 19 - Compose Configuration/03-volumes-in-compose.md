# Volumes in Docker Compose

## Introduction

Docker Compose has a two-part model for volumes: service-level mount definitions that specify what gets mounted where inside a container, and the top-level `volumes:` block that declares and configures the named volumes themselves. Understanding both parts — and how they interact — lets you build production-grade persistent storage configurations declaratively.

---

## The Two Parts of Compose Volume Configuration

```yaml
services:
  db:
    volumes:
      - pgdata:/var/lib/postgresql/data   # Part 1: mount definition (service level)

volumes:
  pgdata:                                  # Part 2: volume declaration (top level)
```

The service-level entry says "mount the volume named `pgdata` at `/var/lib/postgresql/data` inside this container." The top-level entry says "ensure a volume named `pgdata` exists; create it if it does not."

If you reference a volume in a service but do not declare it at the top level, Compose still works — it creates the volume automatically with default settings. However, the explicit top-level declaration is required when you need to configure the volume (driver, labels, external flag).

---

## Service-Level Volume Mounts

### Short Syntax

The most common form: `SOURCE:DESTINATION[:OPTIONS]`

```yaml
services:
  api:
    volumes:
      # Named volume
      - pgdata:/var/lib/postgresql/data

      # Bind mount — absolute host path
      - /etc/ssl/certs:/etc/ssl/certs:ro

      # Bind mount — relative host path
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./config:/app/config:ro

      # Anonymous volume (no name)
      - /app/node_modules

      # Named volume with options
      - cache:/app/.cache:rw
```

Options after the second colon:
- `ro` — read-only
- `rw` — read-write (default, explicit)
- `z` — shared SELinux label (multi-container access)
- `Z` — private SELinux label (single-container access)

### Long Syntax

More explicit, better for documentation in production Compose files:

```yaml
services:
  api:
    volumes:
      # Named volume
      - type: volume
        source: pgdata
        target: /var/lib/postgresql/data
        read_only: false

      # Bind mount
      - type: bind
        source: ./config/app.yml
        target: /app/config.yml
        read_only: true

      # tmpfs mount
      - type: tmpfs
        target: /tmp
        tmpfs:
          size: 268435456   # 256MB in bytes

      # Anonymous volume
      - type: volume
        target: /app/node_modules
```

---

## Top-Level volumes: Block

### Minimal Declaration

```yaml
volumes:
  pgdata:         # creates a local named volume with defaults
  redis-data:
  uploads:
```

Compose prefixes these with the project name: `myapp_pgdata`, `myapp_redis-data`, `myapp_uploads`.

### With Labels

```yaml
volumes:
  pgdata:
    labels:
      app: myapp
      component: database
      environment: production
      team: backend
```

Labels appear in `docker volume ls` output and help identify volumes when multiple applications share a host.

### With Driver Options

```yaml
volumes:
  # NFS-backed named volume
  shared-uploads:
    driver: local
    driver_opts:
      type: nfs
      o: "addr=10.0.0.50,rw,noatime"
      device: ":/exports/uploads"

  # tmpfs as a named volume
  fast-scratch:
    driver: local
    driver_opts:
      type: tmpfs
      device: tmpfs
      o: "size=512m"

  # Specific host path as a named volume
  legacy-data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /mnt/legacy-storage/myapp
```

---

## External Volumes

An external volume is one that Compose should use but not manage. Compose will not create it on `docker compose up` and will not remove it on `docker compose down -v`.

Use external volumes when:
- The volume is created by a separate process (a provisioning script, another Compose project)
- The data must never be accidentally deleted by Compose
- Multiple Compose projects share the same volume

```yaml
volumes:
  pgdata:
    external: true    # Compose uses the volume named "pgdata" exactly — no prefix
```

```bash
# You must create this volume manually before running docker compose up
docker volume create pgdata
```

If the volume does not exist and is declared external, Compose fails with a clear error:

```
Error response from daemon: Get volume: No such volume: pgdata
```

### External Volume with a Different Name

If the external volume has a different name than what you want to reference inside the Compose file:

```yaml
volumes:
  db-data:
    external:
      name: production-postgres-pgdata   # the actual volume name on the host
```

Inside the Compose file, you reference it as `db-data`. Compose mounts the actual volume `production-postgres-pgdata`.

---

## Development vs Production Volume Patterns

### Development — Bind Mount for Code

```yaml
# docker-compose.dev.yml
version: "3.8"

services:
  api:
    build: .
    command: python -m debugpy --listen 0.0.0.0:5678 app.py
    ports:
      - "8080:8080"
      - "5678:5678"   # debugger port
    volumes:
      - .:/app         # bind mount source code for live edits
      - /app/__pycache__  # anonymous volume shields pycache from bind mount
    environment:
      FLASK_DEBUG: "1"

  db:
    image: postgres:16
    volumes:
      - pgdata:/var/lib/postgresql/data   # named volume keeps data between restarts

volumes:
  pgdata:
```

### Production — Named Volumes Only, No Code Mounts

```yaml
# docker-compose.prod.yml
version: "3.8"

services:
  api:
    image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:${APP_VERSION}
    restart: unless-stopped
    ports:
      - "8080:8080"
    volumes:
      - uploads:/app/uploads   # named volume for user-generated content
    # NO source code bind mount — code is baked into the image

  db:
    image: postgres:16
    restart: always
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
    labels:
      environment: production
  uploads:
    labels:
      environment: production
```

---

## Volumes in Compose Down Behavior

Understanding what `docker compose down` does to volumes is critical in production:

```bash
# Stops and removes containers and networks — volumes PRESERVED
docker compose down

# Stops and removes containers, networks, AND named volumes declared in volumes:
docker compose down -v

# Remove orphan volumes (volumes no longer referenced by any service)
docker compose down --remove-orphans
```

**Production rule:** Never use `docker compose down -v` on a production host without knowing which volumes it will remove. The volumes section in your Compose file becomes the list of volumes that will be deleted.

---

## Checking Volume Mounts in Running Services

```bash
# See all mounts for a specific service's container
docker compose exec api mount | grep data

# Inspect via docker inspect
docker inspect myapp_api_1 --format '{{json .Mounts}}' | python3 -m json.tool

# List volumes used by the Compose project
docker compose config --volumes
```

---

## Complete Volumes Example

```yaml
version: "3.8"

services:
  api:
    image: myapp:1.4.2
    volumes:
      # Named volume for user uploads
      - uploads:/app/uploads
      # Read-only config bind mount
      - ./config/app.prod.yml:/app/config.yml:ro
      # tmpfs for fast temporary processing
      - type: tmpfs
        target: /tmp
        tmpfs:
          size: 268435456   # 256MB

  db:
    image: postgres:16
    volumes:
      - pgdata:/var/lib/postgresql/data
      # Custom postgres config
      - ./config/postgres.conf:/etc/postgresql/postgresql.conf:ro

  redis:
    image: redis:7-alpine
    volumes:
      - redis-data:/data
    command: redis-server --appendonly yes

  backup:
    image: alpine
    volumes:
      - pgdata:/source:ro    # reads from DB volume
      - /mnt/backups:/dest   # writes to host backup directory

volumes:
  pgdata:
    labels:
      app: myapp
      component: database
  redis-data:
    labels:
      app: myapp
      component: cache
  uploads:
    labels:
      app: myapp
      component: storage
```

---

## Summary

- Compose volumes have two parts: service-level mount definitions (what gets mounted where) and top-level volume declarations (how the volume is configured)
- Use short syntax (`- pgdata:/var/lib/postgresql/data`) for readability and long syntax (`type: volume, source: pgdata, target: ...`) for explicit production documentation
- The `external: true` flag tells Compose to use an existing volume it does not manage — essential for critical production data that must survive `docker compose down -v`
- Development patterns use bind mounts for source code (live edits) combined with anonymous volumes to shield container-internal directories from being overridden
- Production patterns use named volumes only — no source code bind mounts, code is baked into the image
- `docker compose down -v` removes all named volumes declared in the `volumes:` block — never run this in production without confirming which volumes it will delete

In the next file, we cover networking in Compose — defining custom networks, isolating service tiers, and using external networks.
