# Sharing Volumes

## Introduction

A Docker volume can be mounted by more than one container simultaneously. When two or more containers share the same volume, they read and write to the same underlying files in real time. This enables powerful architectural patterns: sidecar processes that enrich or ship logs, data pipelines where one container produces data and another consumes it, and shared configuration between cooperating services.

This file covers the mechanics of shared volumes, the real-world patterns that use them, and the critical data consistency issues you must handle.

---

## How Volume Sharing Works

Sharing a volume requires no special configuration on the volume itself. You simply mount the same volume name into multiple containers.

```bash
# Create a shared volume
docker volume create shared-logs

# Container 1: the API, writes logs to /app/logs
docker run -d \
  --name api \
  -v shared-logs:/app/logs \
  myapp:1.0

# Container 2: a log shipper, reads from the same /logs path
docker run -d \
  --name log-shipper \
  -v shared-logs:/logs \
  fluent/fluent-bit:latest
```

Both containers now see the same files. The API writes `access.log` to `/app/logs/`. The log shipper reads from `/logs/` and ships those same log files to CloudWatch, Elasticsearch, or Splunk.

The volume mount path does not need to be the same in both containers. The API mounts it at `/app/logs` and the log shipper mounts it at `/logs` — both point to the same volume data.

---

## Pattern 1: Sidecar Log Shipping

The most common volume sharing pattern in production. The main application writes logs to a file (not to stdout). A sidecar container reads those log files and ships them to a centralized logging system.

```bash
docker run -d \
  --name nginx \
  -v nginx-logs:/var/log/nginx \
  -p 80:80 \
  nginx:1.25.3

docker run -d \
  --name fluentbit \
  -v nginx-logs:/var/log/nginx:ro \   # read-only — log shipper does not need write access
  -v $(pwd)/fluent-bit.conf:/fluent-bit/etc/fluent-bit.conf:ro \
  fluent/fluent-bit:latest
```

The `:ro` suffix on the log shipper's mount is important — a log shipper should never modify the log files it reads. Read-only mounts prevent accidental writes and add a layer of safety.

```conf
# fluent-bit.conf
[INPUT]
    Name    tail
    Path    /var/log/nginx/access.log
    DB      /tmp/flb_nginx.db
    Tag     nginx.access

[OUTPUT]
    Name               cloudwatch_logs
    Match              nginx.*
    region             us-east-1
    log_group_name     /myapp/nginx
    log_stream_prefix  nginx-
    auto_create_group  true
```

---

## Pattern 2: Data Processing Pipeline

A pipeline where one container generates data and another processes it:

```bash
docker volume create pipeline-data

# Stage 1: Download and decompress raw data
docker run -d \
  --name downloader \
  -v pipeline-data:/data \
  data-downloader:1.0 \
  download --output /data/raw.csv

# Stage 2: Process the raw data (starts after downloader completes)
docker run -d \
  --name processor \
  -v pipeline-data:/data \
  data-processor:1.0 \
  process --input /data/raw.csv --output /data/processed.parquet

# Stage 3: Load the processed data into a data warehouse
docker run -d \
  --name loader \
  -v pipeline-data:/data:ro \
  data-loader:1.0 \
  load --input /data/processed.parquet
```

Each stage reads from and writes to the shared `/data` volume. The pipeline is orchestrated by timing or by watching for file creation (e.g., using `inotifywait`).

---

## Pattern 3: Shared Configuration

Containers that share a configuration volume can pick up configuration changes without rebuilding images:

```bash
docker volume create app-config

# Configuration manager: fetches config from AWS Parameter Store and writes to volume
docker run -d \
  --name config-manager \
  -v app-config:/config \
  config-manager:1.0 \
  sync --param-path /myapp/production --output /config/settings.json --interval 60

# Application: reads configuration from the shared volume
docker run -d \
  --name api \
  -v app-config:/etc/myapp:ro \
  myapp:1.0
```

The config manager fetches settings from AWS Parameter Store every 60 seconds and writes `settings.json` to the volume. The API reads from `/etc/myapp/settings.json`. When the config manager updates the file, the API's next read gets the new configuration.

---

## The Critical Issue: Concurrent Write Safety

When multiple containers write to the same volume simultaneously, data corruption is a real risk. Volume sharing does not provide any automatic locking or coordination between containers.

### Example of a Dangerous Pattern

```bash
# Two API workers writing to the same log file simultaneously
docker run -d --name api-1 -v logs:/app/logs myapp:1.0
docker run -d --name api-2 -v logs:/app/logs myapp:1.0
```

If both workers write to `/app/logs/access.log` at the same time, log lines from each worker will interleave. In the worst case, partial writes create malformed log lines.

### Safe Concurrent Write Patterns

**Pattern 1:** Write to separate files per container.

```bash
# Each container writes to its own named log file
docker run -d --name api-1 -v logs:/app/logs -e LOG_FILE=api-1.log myapp:1.0
docker run -d --name api-2 -v logs:/app/logs -e LOG_FILE=api-2.log myapp:1.0
```

**Pattern 2:** Use a daemon designed for concurrent access (like a database) and mount the data directory into only one writer container at a time.

**Pattern 3:** Write to stdout/stderr instead of files. Docker handles log shipping for each container independently.

---

## Read-Only Volume Mounts

For containers that should read from a shared volume but never write, always use `:ro`:

```bash
# Writer — read-write access
docker run -d --name producer -v shared:/data myapp-producer:1.0

# Readers — read-only access
docker run -d --name consumer-1 -v shared:/data:ro myapp-consumer:1.0
docker run -d --name consumer-2 -v shared:/data:ro myapp-consumer:1.0
```

Read-only mounts:
- Prevent accidental writes from consumer containers
- Protect against a compromised consumer container corrupting shared data
- Make the architectural intent explicit — who writes and who reads

```bash
# Using --mount syntax
docker run -d \
  --name consumer \
  --mount type=volume,source=shared,target=/data,readonly \
  myapp-consumer:1.0
```

---

## Volume Sharing in Docker Compose

Docker Compose makes volume sharing between services clean and declarative:

```yaml
version: "3.8"

services:
  api:
    image: myapp:1.0
    volumes:
      - app-logs:/app/logs   # read-write access
      - app-config:/etc/myapp:ro

  log-shipper:
    image: fluent/fluent-bit:latest
    volumes:
      - app-logs:/logs:ro    # read-only — same volume, different mount path
      - ./fluent-bit.conf:/fluent-bit/etc/fluent-bit.conf:ro

  config-manager:
    image: config-manager:1.0
    volumes:
      - app-config:/config   # read-write — writes config here

volumes:
  app-logs:
  app-config:
```

---

## Volume Sharing for Initialization

A common pattern in container orchestration: an init container writes configuration or data into a volume, and the main application container starts after and reads from that volume.

```yaml
version: "3.8"

services:
  # Init container: runs migrations and seeds the database
  init:
    image: myapp:1.0
    command: ["python", "manage.py", "migrate"]
    volumes:
      - init-flag:/init
    environment:
      - DATABASE_URL=postgresql://user:pass@db:5432/myapp
    depends_on:
      - db

  api:
    image: myapp:1.0
    volumes:
      - init-flag:/init:ro
    environment:
      - DATABASE_URL=postgresql://user:pass@db:5432/myapp
    depends_on:
      - init
    # Only starts after init service has completed

volumes:
  init-flag:
```

---

## Summary

- Multiple containers can mount the same named volume simultaneously — no special configuration required on the volume itself
- Containers that only need to read from a shared volume should always use `:ro` (read-only) mounts to prevent accidental writes
- The sidecar log shipping pattern is the most common production use: the main app writes log files to a volume, a sidecar container reads and ships them to CloudWatch or Elasticsearch
- Concurrent writes to the same file from multiple containers cause data corruption — prevent this by having each container write to its own uniquely named file, or by routing all writes through a single authoritative writer
- Docker Compose makes shared volumes clean and explicit — declare the volume once in the top-level `volumes:` block and mount it in as many services as needed
- The init container pattern uses a shared volume to signal completion: the init container writes to the volume, the main application container sees the signal and starts

In the next file, we cover volume backup and restore — the critical operational skill for protecting persistent data and enabling disaster recovery.
