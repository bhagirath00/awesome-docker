# Non-Root User in Production Dockerfiles

## Introduction

The `USER` instruction was covered conceptually in the Advanced Dockerfile chapter. This file goes deeper — showing complete, real-world production Dockerfile patterns for multiple languages and frameworks, including the nuances of file permissions, volume compatibility, and the non-root considerations that catch teams by surprise in production.

Non-root is not just a security recommendation. In modern cloud environments, it is a requirement. AWS ECR's enhanced scanning flags containers running as root. Kubernetes PodSecurityAdmission in `restricted` mode rejects containers that run as root. ECS task definitions on Fargate can enforce non-root via `user` field. If your containers run as root, you have technical debt.

---

## The Non-Root Pattern for Python Applications

```dockerfile
FROM python:3.11-slim

# Install runtime dependencies (as root — required for apt/pip)
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        curl \
        libpq5 \
    && rm -rf /var/lib/apt/lists/*

# Create non-root user
RUN groupadd --system --gid 1001 appuser && \
    useradd --system --uid 1001 --gid 1001 --no-create-home --shell /bin/false appuser

WORKDIR /app

# Install Python dependencies as root, but make them readable by appuser
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy source code with correct ownership
COPY --chown=appuser:appuser src/ .

# Create writable directories the app needs (as root, then chown)
RUN mkdir -p /app/logs /app/tmp && \
    chown -R appuser:appuser /app

# Switch to non-root user for all subsequent instructions
USER appuser

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
    CMD curl -f http://localhost:8080/health || exit 1

CMD ["gunicorn", "--workers", "4", "--bind", "0.0.0.0:8080", \
     "--log-level", "info", "--access-logfile", "-", \
     "--error-logfile", "-", "app:app"]
```

---

## The Non-Root Pattern for Node.js Applications

Node.js images come with a built-in `node` user (UID 1000). Use it:

```dockerfile
FROM node:20-alpine

# The node user already exists (UID 1000, GID 1000)

WORKDIR /app

# Copy package files first (dependency caching layer)
COPY package.json package-lock.json ./

# Install production dependencies as root
RUN npm ci --only=production && \
    npm cache clean --force

# Copy source with correct ownership
COPY --chown=node:node src/ ./src/

# Change ownership of node_modules too
RUN chown -R node:node /app

# Switch to the node user
USER node

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=10s --start-period=20s --retries=3 \
    CMD wget -qO- --tries=1 http://localhost:3000/health || exit 1

CMD ["node", "src/server.js"]
```

---

## The Non-Root Pattern for Go Applications

Go compiles to a static binary, enabling the smallest possible image:

```dockerfile
# Stage 1: Build
FROM golang:1.22-alpine AS builder

WORKDIR /build

COPY go.mod go.sum ./
RUN go mod download

COPY . .

RUN CGO_ENABLED=0 GOOS=linux go build \
    -ldflags="-w -s" \
    -o /build/api \
    ./cmd/api

# Stage 2: Production image (distroless — no shell, no package manager)
FROM gcr.io/distroless/static-debian12:nonroot

# nonroot tag = already set to run as user 65532 (nonroot)
# No need to create a user — it is part of the distroless image

COPY --from=builder /build/api /api

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=10s --start-period=10s --retries=3 \
    CMD ["/api", "health"]

ENTRYPOINT ["/api"]
CMD ["serve", "--port", "8080"]
```

Distroless images (`gcr.io/distroless/`) contain only the application and its runtime dependencies — no shell, no package manager, no utilities. The `:nonroot` tag runs as UID 65532. This is the most hardened production image possible.

---

## The Non-Root Pattern for Java/Spring Boot

Java applications have longer startup times — configure `start-period` accordingly:

```dockerfile
# Stage 1: Build
FROM eclipse-temurin:21-jdk-alpine AS builder

WORKDIR /build

COPY mvnw .
COPY .mvn .mvn
COPY pom.xml .

# Download dependencies first (cached layer)
RUN ./mvnw dependency:go-offline -q

COPY src/ src/
RUN ./mvnw package -DskipTests -q

# Extract the layered jar for optimized caching
RUN java -Djarmode=layertools -jar target/*.jar extract --destination /extracted

# Stage 2: Production image
FROM eclipse-temurin:21-jre-alpine

RUN addgroup --system --gid 1001 appuser && \
    adduser --system --uid 1001 --ingroup appuser --no-create-home appuser

WORKDIR /app

# Copy Spring Boot layered jar (in order from least to most frequently changed)
COPY --from=builder --chown=appuser:appuser /extracted/dependencies/ ./
COPY --from=builder --chown=appuser:appuser /extracted/spring-boot-loader/ ./
COPY --from=builder --chown=appuser:appuser /extracted/snapshot-dependencies/ ./
COPY --from=builder --chown=appuser:appuser /extracted/application/ ./

USER appuser

EXPOSE 8080

HEALTHCHECK \
    --interval=30s \
    --timeout=10s \
    --start-period=90s \     # JVM startup takes 30-90 seconds
    --retries=3 \
    CMD wget -qO- --tries=1 http://localhost:8080/actuator/health | grep -q '"status":"UP"' || exit 1

ENTRYPOINT ["java", \
    "-XX:+UseContainerSupport", \
    "-XX:MaxRAMPercentage=75.0", \
    "-jar", \
    "app.jar"]
```

---

## The Non-Root Pattern for Nginx

Standard Nginx requires root to bind to port 80. Use `nginxinc/nginx-unprivileged` for non-root:

```dockerfile
FROM nginxinc/nginx-unprivileged:1.25-alpine

# This image already:
# - Runs as the nginx user (non-root)
# - Listens on port 8080 instead of 80
# - Removes the ability to bind to privileged ports

COPY --chown=nginx:nginx ./dist/ /usr/share/nginx/html/
COPY --chown=nginx:nginx ./nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
    CMD wget -qO- http://localhost:8080/health || exit 1
```

---

## Handling Volumes with Non-Root Users

When a named volume is mounted at container startup, Docker needs to set the right ownership for the non-root user to write to it.

### Problem

```bash
# Volume is owned by root at creation
docker volume create mydata
docker run -v mydata:/app/data myapp:1.0
# PermissionError: [Errno 13] Permission denied: '/app/data/output.json'
```

### Solution 1: Fix Permissions in ENTRYPOINT Script

```bash
# docker-entrypoint.sh
#!/bin/sh
set -e

# Fix volume ownership at startup (runs as root before USER switch)
if [ "$(stat -c '%U' /app/data)" != "appuser" ]; then
    chown -R appuser:appuser /app/data
fi

# Drop to the non-root user and exec the CMD
exec su-exec appuser "$@"
```

```dockerfile
RUN apk add --no-cache su-exec    # alpine's tool to drop privileges

COPY docker-entrypoint.sh /usr/local/bin/
RUN chmod +x /usr/local/bin/docker-entrypoint.sh

# Note: ENTRYPOINT runs as root to fix permissions, then drops to appuser
ENTRYPOINT ["docker-entrypoint.sh"]
CMD ["gunicorn", "--bind", "0.0.0.0:8080", "app:app"]
```

### Solution 2: Use --user at Runtime

```bash
# Mount the volume and run as the host user
docker run \
  --user $(id -u):$(id -g) \
  -v mydata:/app/data \
  myapp:1.0
```

The container runs as your host UID — which matches the volume files created by your host user.

### Solution 3: Initialize Permissions in a One-Off Container

```bash
# Fix permissions before the main container starts
docker run --rm -v mydata:/data alpine chown -R 1001:1001 /data

# Now start the main container
docker run -v mydata:/app/data myapp:1.0
```

---

## Non-Root in Docker Compose

```yaml
version: "3.8"

services:
  api:
    image: myapp:1.0   # USER appuser set in Dockerfile
    # user: "1001:1001"  # You can also enforce at Compose level, but prefer Dockerfile

  # For a service that needs volume permission fixing:
  api-with-volume:
    image: myapp:1.0
    volumes:
      - uploads:/app/uploads
    user: "1001:1001"
    # Ensure the volume is pre-initialized with correct permissions
    # via an init container or chown step

volumes:
  uploads:
```

---

## Summary

- Non-root containers are a production requirement, not just a recommendation — ECR scanning, Kubernetes PSA, and ECS Fargate all have mechanisms to enforce or report on root-running containers
- The core pattern: create a system user with `useradd --system`, copy files with `COPY --chown`, then `USER appuser` before `CMD`
- Node.js images have a built-in `node` user (UID 1000) — use it rather than creating a new one
- For Go services, use `gcr.io/distroless/static-debian12:nonroot` — the most hardened production image, no shell, runs as UID 65532
- Volume ownership is the most common non-root pain point — use an entrypoint script to `chown` the volume directory before dropping privileges with `exec su-exec appuser "$@"`
- Set `start_period` in health checks generously for JVM and Elasticsearch containers — their startup time can be 30-90 seconds, and underestimating causes restart loops

In the next file, we cover secrets in Dockerfile builds — the correct way to use credentials during build without leaking them into image layers.
