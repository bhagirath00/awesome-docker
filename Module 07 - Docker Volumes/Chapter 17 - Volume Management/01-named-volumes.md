# Named Volumes

## Introduction

Named volumes are the foundation of persistent, production-ready storage in Docker. They are the recommended way to persist all stateful data — databases, user uploads, application caches, and anything else that must survive container restarts, deployments, and host reboots.

This file covers the named volume workflow end-to-end: creating, using, inspecting, migrating, and operating named volumes in production with real application examples.

---

## What Makes a Volume "Named"

A named volume is simply a volume you give an explicit name to, rather than letting Docker generate a random UUID. The name makes the volume identifiable, referenceable, and persistent by design.

```bash
# Named volume — you choose "pgdata"
docker volume create pgdata
docker run -v pgdata:/var/lib/postgresql/data postgres:16

# Anonymous volume — Docker generates "3f4a5b6c7d8e..."
docker run -v /var/lib/postgresql/data postgres:16
```

The named volume `pgdata` persists until you explicitly run `docker volume rm pgdata`. You can mount it into any container at any time, on any image version. Anonymous volumes are automatically removed when you run `docker container prune` or `docker system prune --volumes` — they are designed to be temporary.

Always use named volumes for data that matters.

---

## The Named Volume Lifecycle in Production

Here is the complete lifecycle of a named volume for a PostgreSQL database:

### Phase 1: Initial Setup

```bash
# Create the volume (or let docker run create it on first use)
docker volume create myapp-postgres-prod

# Start PostgreSQL with the volume
docker run -d \
  --name db \
  --restart unless-stopped \
  -v myapp-postgres-prod:/var/lib/postgresql/data \
  -e POSTGRES_USER=apiuser \
  -e POSTGRES_PASSWORD=supersecretpassword \
  -e POSTGRES_DB=myapp \
  --memory 2g \
  --memory-swap 2g \
  -p 127.0.0.1:5432:5432 \
  postgres:16
```

On first run, Docker sees the volume is empty and seeds it with PostgreSQL's initial data directory. PostgreSQL then initializes its database files, creates the `myapp` database, and starts accepting connections. All of this happens inside `myapp-postgres-prod`.

### Phase 2: Normal Operation

Applications connect to the database. Records are written. The data lives in the named volume, completely outside the container's writable layer.

```bash
# Verify the volume is mounted and in use
docker inspect db --format '{{range .Mounts}}Name: {{.Name}}, Path: {{.Destination}}{{end}}'
# Name: myapp-postgres-prod, Path: /var/lib/postgresql/data
```

### Phase 3: Container Restart (Data Preserved)

```bash
# Stop and remove the container — volume is untouched
docker stop db && docker rm db

# Confirm volume still exists with all its data
docker volume inspect myapp-postgres-prod
# Volume is present, CreatedAt is the original creation date

# Start a new container with the same volume — all data is there
docker run -d \
  --name db \
  --restart unless-stopped \
  -v myapp-postgres-prod:/var/lib/postgresql/data \
  -e POSTGRES_USER=apiuser \
  -e POSTGRES_PASSWORD=supersecretpassword \
  postgres:16
```

PostgreSQL starts and finds its fully initialized database at `/var/lib/postgresql/data`. All tables, records, and indexes are intact. The container was replaced; the data was not.

### Phase 4: Upgrading PostgreSQL Version

```bash
# Stop the current container
docker stop db && docker rm db

# Check the PostgreSQL version in the data directory (important for major upgrades)
MOUNT=$(docker volume inspect --format '{{.Mountpoint}}' myapp-postgres-prod)
cat $MOUNT/PG_VERSION
# 16

# For minor version upgrades (16.1 → 16.2), directly remount is safe
docker run -d \
  --name db \
  --restart unless-stopped \
  -v myapp-postgres-prod:/var/lib/postgresql/data \
  -e POSTGRES_USER=apiuser \
  -e POSTGRES_PASSWORD=supersecretpassword \
  postgres:16.2       # Updated minor version, same data format
```

**Major version upgrades** (e.g., 15 → 16) require running `pg_upgrade` because the data directory format changes. Always backup before major upgrades.

---

## Named Volumes for Different Applications

### Redis Persistence

```bash
# Redis with AOF (Append Only File) persistence
docker run -d \
  --name redis \
  --restart unless-stopped \
  -v myapp-redis-prod:/data \
  -p 127.0.0.1:6379:6379 \
  redis:7-alpine \
  redis-server --appendonly yes

# Redis data directory is /data — every AOF log and RDB snapshot lives in the volume
```

### MySQL / MariaDB

```bash
docker run -d \
  --name mysql \
  --restart unless-stopped \
  -v myapp-mysql-prod:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=rootpassword \
  -e MYSQL_DATABASE=myapp \
  -e MYSQL_USER=apiuser \
  -e MYSQL_PASSWORD=apipassword \
  -p 127.0.0.1:3306:3306 \
  mysql:8.0
```

### MongoDB

```bash
docker run -d \
  --name mongodb \
  --restart unless-stopped \
  -v myapp-mongo-prod:/data/db \
  -e MONGO_INITDB_ROOT_USERNAME=admin \
  -e MONGO_INITDB_ROOT_PASSWORD=adminpassword \
  -p 127.0.0.1:27017:27017 \
  mongo:7.0
```

### Application File Storage (User Uploads)

```bash
docker run -d \
  --name api \
  --restart unless-stopped \
  -v myapp-uploads-prod:/app/uploads \
  -p 8080:8080 \
  myapp:1.4.2
```

Uploads are stored in `/app/uploads` inside the container — which is actually `myapp-uploads-prod` volume. When you deploy `myapp:1.4.3`, the new container mounts the same volume and all user uploads are available immediately.

---

## Named Volumes in Docker Compose

Docker Compose with named volumes is the most common production pattern for single-host deployments.

```yaml
version: "3.8"

services:
  api:
    image: myapp:1.4.2
    ports:
      - "8080:8080"
    volumes:
      - uploads:/app/uploads
    depends_on:
      - db
      - redis

  db:
    image: postgres:16
    restart: unless-stopped
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      POSTGRES_USER: apiuser
      POSTGRES_PASSWORD: supersecretpassword
      POSTGRES_DB: myapp
    ports:
      - "127.0.0.1:5432:5432"

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    volumes:
      - redis-data:/data
    command: redis-server --appendonly yes

volumes:
  pgdata:
    labels:
      app: myapp
      component: database
      environment: production
  redis-data:
    labels:
      app: myapp
      component: cache
      environment: production
  uploads:
    labels:
      app: myapp
      component: storage
      environment: production
```

When you run `docker compose up`, Compose creates three named volumes:
- `myapp_pgdata`
- `myapp_redis-data`
- `myapp_uploads`

(Prefixed with the Compose project name — the directory name by default.)

When you run `docker compose down`, the containers stop but the volumes are preserved. Running `docker compose up` again mounts the same volumes — all data is intact.

To also delete volumes when stopping:
```bash
# Destroy containers AND volumes (data loss — use intentionally)
docker compose down -v
```

---

## Monitoring Volume Size in Production

Volume size grows over time. An unmonitored database volume can fill the host disk and cause the database to crash.

```bash
# Check size of all volumes
docker system df -v | grep VOLUME -A 50

# Check specific volume size
MOUNT=$(docker volume inspect --format '{{.Mountpoint}}' myapp-postgres-prod)
sudo du -sh $MOUNT
# 15.2G    /var/lib/docker/volumes/myapp-postgres-prod/_data

# Check disk space remaining on the host
df -h /var/lib/docker
# Filesystem      Size  Used Avail Use% Mounted on
# /dev/xvda1      100G   62G   38G  62% /
```

Set up monitoring alerts when the disk usage on `/var/lib/docker` exceeds 80%.

---

## Summary

- Named volumes are the definitive production storage mechanism — name them explicitly, they persist until explicitly removed, and they survive container recreations and upgrades
- The named volume lifecycle: create once → mount in every container deployment → data persists across container restarts, replacements, and image upgrades
- Name volumes clearly with a pattern like `appname-component-environment` (e.g., `myapp-postgres-prod`) so they are identifiable when listed on the host
- In Docker Compose, volumes declared in the top-level `volumes:` block are named and persistent — `docker compose down` preserves them; `docker compose down -v` deletes them
- Use named volumes for every stateful service: PostgreSQL (`/var/lib/postgresql/data`), Redis (`/data`), MySQL (`/var/lib/mysql`), MongoDB (`/data/db`), and application upload directories
- Monitor volume size regularly — growing databases on unmonitored volumes will eventually fill the host disk

In the next file, we cover sharing volumes between multiple containers — a pattern used for sidecar processes, log shipping, and data processing pipelines.
