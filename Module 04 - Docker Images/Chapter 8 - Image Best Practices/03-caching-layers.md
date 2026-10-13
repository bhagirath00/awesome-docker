# Caching Layers

## Introduction

Docker's build cache is one of the most powerful features for making development and CI/CD fast. When used well, builds that take 3 minutes on first run finish in under 10 seconds on subsequent runs. When used poorly, every code change triggers a full rebuild from scratch.

Understanding how the cache works and structuring your Dockerfile to take maximum advantage of it is one of the highest-leverage skills in Docker development.

---

## How the Build Cache Works

Docker caches each image layer. When you run `docker build`, Docker checks each instruction against its cache. If nothing has changed that would affect that instruction's output, Docker reuses the cached layer instead of re-executing the instruction.

The cache key for each instruction is based on:
1. The instruction itself (the exact text)
2. The parent layer's cache key
3. For `COPY` and `ADD`: the content of the files being copied (checksum)
4. For `RUN`: only the instruction text (not the output of external commands like `apt-get update`)

**Critical rule:** Once Docker finds a cache miss — any instruction that cannot use the cache — **all subsequent instructions are also rebuilt**, even if they individually have not changed.

This is why instruction order matters enormously.

---

## Visualizing Cache Hits and Misses

```bash
docker build -t myapp:1.0 .

# Output with cache hits:
Step 1/6 : FROM python:3.11-slim
 ---> e4c58958181a    ← cached
Step 2/6 : WORKDIR /app
 ---> Using cache     ← cached
 ---> a1b2c3d4ef56
Step 3/6 : COPY requirements.txt .
 ---> Using cache     ← cached (file not changed)
 ---> b2c3d4e5f678
Step 4/6 : RUN pip install -r requirements.txt
 ---> Using cache     ← cached (requirements.txt not changed)
 ---> c3d4e5f6a789
Step 5/6 : COPY . .
 ---> d4e5f6a7b890    ← cache miss (code changed)
Step 6/6 : CMD ["python", "app.py"]
 ---> e5f6a7b8c901    ← rebuilt (comes after cache miss)
```

Steps 1-4 used the cache. Steps 5-6 were rebuilt. The pip install (slow) was cached because `requirements.txt` did not change.

---

## The Most Important Caching Pattern

The most impactful thing you can do for build performance is to copy slow-changing files before fast-changing files.

In a typical application:
- **Base image** — changes rarely (security patches, major version upgrades)
- **System packages** — changes rarely
- **Application dependencies** — changes occasionally (when you add/remove packages)
- **Application code** — changes constantly (every commit)

Structure your Dockerfile to match this order — slowest-changing at the top, fastest-changing at the bottom:

```dockerfile
# Slowest to change — stays cached the longest
FROM python:3.11-slim

RUN apt-get update && \
    apt-get install -y --no-install-recommends libpq-dev && \
    rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Changes occasionally — copy before code
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Changes constantly — copy last
COPY . .

CMD ["python", "app.py"]
```

Now:
- Changing `app.py` → only the `COPY . .` layer rebuilds (~1 second)
- Changing `requirements.txt` → pip install and everything after rebuilds (~30-60 seconds)
- Changing the base image → everything rebuilds (rare)

Without this ordering, changing `app.py` would trigger a full `pip install` every time — a 30-60 second wait on every code change.

---

## Language-Specific Caching Patterns

### Python

```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Copy just the dependency definition first
COPY requirements.txt .

# Install — only re-runs when requirements.txt changes
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code last
COPY src/ ./src/
COPY config/ ./config/

CMD ["python", "src/app.py"]
```

### Node.js

```dockerfile
FROM node:20-alpine

WORKDIR /app

# Copy package files before source code
COPY package.json package-lock.json ./

# Install — only re-runs when package files change
RUN npm ci --only=production && npm cache clean --force

# Copy source code last
COPY src/ ./src/

CMD ["node", "src/index.js"]
```

### Java with Maven

```dockerfile
FROM eclipse-temurin:21-jdk-alpine AS builder

WORKDIR /build

# Copy POM file first — dependencies defined here
COPY pom.xml .

# Download dependencies separately — cached until pom.xml changes
RUN mvn dependency:go-offline -B

# Now copy source and build
COPY src/ ./src/
RUN mvn package -DskipTests -B

FROM eclipse-temurin:21-jre-alpine
COPY --from=builder /build/target/app.jar /app.jar
CMD ["java", "-jar", "/app.jar"]
```

### Go

```dockerfile
FROM golang:1.21-alpine AS builder

WORKDIR /build

# Copy go.mod and go.sum first — module dependencies
COPY go.mod go.sum ./

# Download modules — cached until go.mod/go.sum change
RUN go mod download

# Copy source and build
COPY . .
RUN CGO_ENABLED=0 go build -o app .

FROM scratch
COPY --from=builder /build/app /app
CMD ["/app"]
```

---

## Cache Busting Strategies

Sometimes you want to force certain layers to rebuild even though their inputs have not changed. This is called **cache busting**.

### Force apt-get to Get Fresh Packages

The `RUN apt-get update` instruction is cached based on its text — Docker does not know whether new packages have been released since the last build. To force it to get fresh data:

```dockerfile
# Bust the apt cache by including a date ARG
ARG CACHE_DATE=unknown
RUN apt-get update && apt-get upgrade -y && ...
```

```bash
# Force fresh apt update on today's build
docker build --build-arg CACHE_DATE=$(date +%Y-%m-%d) -t myapp:1.0 .
```

Any change in `CACHE_DATE` invalidates the `RUN apt-get update` layer and all layers after it.

### Always Get Latest Dependencies

If you want a `RUN` that always re-executes (e.g., always install the latest security patch):

```dockerfile
ARG CACHEBUST=1
RUN pip install --upgrade pip
```

```bash
docker build --build-arg CACHEBUST=$(date +%s) -t myapp:1.0 .
```

This is an escape hatch. Use it only when necessary — its value is that it forces a rebuild of something that would otherwise be cached.

---

## Analyzing Your Cache Efficiency

Check where cache misses are happening and why:

```bash
# Build with verbose output to see cache behavior
docker build --progress=plain -t myapp:1.0 . 2>&1 | grep -E "(Step|cache|CACHED)"
```

Use the `dive` tool to see which layers are the largest and inspect what changed:

```bash
# Install dive
brew install dive  # macOS
# or download from https://github.com/wagoodman/dive

# Analyze an image
dive myapp:1.0
```

`dive` shows you each layer and what files it contains. You can immediately see if a layer is larger than expected or contains files it should not.

---

## The --no-cache Flag

Sometimes the cache becomes stale and you want a clean build:

```bash
# Ignore all cached layers
docker build --no-cache -t myapp:1.0 .
```

Use this when:
- `apt-get update` is cached with stale package lists
- You want to verify the image builds cleanly without any cache
- You are debugging an issue that might be caused by a stale cached layer
- Running periodic security builds that must always use the latest base image and packages

Never use `--no-cache` by default in development — you lose all the build speed benefits.

---

## Cache in CI/CD

In CI/CD environments, each pipeline run starts fresh — there is no build cache from the previous run unless you explicitly configure it.

### GitHub Actions — Layer Caching

Using `docker/build-push-action` with GitHub Actions cache:

```yaml
- name: Set up Docker Buildx
  uses: docker/setup-buildx-action@v3

- name: Build and push
  uses: docker/build-push-action@v5
  with:
    context: .
    push: true
    tags: |
      ${{ env.ECR_REGISTRY }}/myapp:${{ github.sha }}
    cache-from: type=gha         # restore cache from GitHub
    cache-to: type=gha,mode=max  # save cache to GitHub
```

With this configuration:
- On the first run, all layers are built and cached
- On subsequent runs with the same base layers, those layers are restored from cache
- Only the layers that changed are rebuilt

### ECR Cache with Inline Cache

You can also use the registry itself as a cache source:

```yaml
- name: Build and push with ECR cache
  uses: docker/build-push-action@v5
  with:
    context: .
    push: true
    tags: ${{ env.ECR_REGISTRY }}/myapp:${{ github.sha }}
    cache-from: type=registry,ref=${{ env.ECR_REGISTRY }}/myapp:cache
    cache-to: type=registry,ref=${{ env.ECR_REGISTRY }}/myapp:cache,mode=max
```

This stores the build cache as a special image in ECR. Any runner can restore the cache by pulling from ECR — useful when GitHub Actions cache is too small or you use multiple CI systems.

---

## Common Caching Mistakes

### Mistake 1: COPY . . before installing dependencies

```dockerfile
# WRONG — invalidates dependency cache on every code change
FROM python:3.11-slim
WORKDIR /app
COPY . .                        # ← code changes here invalidate everything below
RUN pip install -r requirements.txt  # ← always re-runs

# RIGHT
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .         # ← only invalidates cache when requirements change
RUN pip install -r requirements.txt  # ← cached until requirements change
COPY . .                        # ← code changes here only
```

### Mistake 2: Running apt-get update alone

```dockerfile
# WRONG — update cache goes stale, apt-get install later may fail
RUN apt-get update
RUN apt-get install -y curl     # uses stale package list if cached

# RIGHT — always update and install in the same RUN
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl && \
    rm -rf /var/lib/apt/lists/*
```

### Mistake 3: Invalidating cache with unnecessary file changes

If files that are not needed in the Docker image frequently change and are in the COPY path, they cause unnecessary cache misses.

```dockerfile
# If tests/ changes with every PR, this invalidates the code layer unnecessarily
COPY . .   # copies tests/ too
```

```dockerfile
# Better — copy only what the application needs
COPY src/ ./src/
COPY config/ ./config/
# tests/ is not copied — changes to tests don't invalidate this layer
```

---

## Summary

- The build cache reuses layers from previous builds — a cache miss at any instruction invalidates all subsequent layers
- Structure Dockerfiles with the slowest-changing instructions first and fastest-changing instructions last
- Always copy dependency definition files (`requirements.txt`, `package.json`) before copying application code
- Use cache-busting ARGs when you need to force a rebuild of a specific layer (like `apt-get update`)
- `--no-cache` forces a full rebuild — use it for security builds and debugging, not routine development
- In CI/CD, configure layer caching explicitly using GitHub Actions cache, ECR registry cache, or another persistent cache store
- Never run `apt-get update` alone — always combine it with `apt-get install` in the same `RUN` instruction

In the next file, we cover multi-stage builds — the most powerful technique for creating small, secure production images while keeping development builds convenient.
