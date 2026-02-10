# Reducing Image Size

## Introduction

A smaller Docker image is better in almost every way. It pulls faster, deploys faster, uses less storage, has fewer vulnerabilities, and reduces your AWS ECR storage costs. The difference between a 1GB image and a 100MB image is real — in a CI/CD pipeline that deploys dozens of times per day, the cumulative time savings are significant.

This file covers every practical technique for reducing image size, how to measure the impact of your changes, and what results you can realistically expect for different types of applications.

---

## Why Image Size Matters

Before optimizing, understand what you are optimizing for:

**Pull time** — When ECS launches a new task, it pulls the image if it is not cached on the instance. A 1GB image takes 30-60 seconds to pull. A 100MB image takes 3-5 seconds. During auto-scaling events or deployments, this difference compounds across many instances.

**Storage cost** — AWS ECR charges for storage. If you have 50 images in a repository and each is 1GB, you are storing 50GB. At $0.10 per GB per month, that is $5/month per repository. With many repositories and teams, this adds up.

**Security** — Every package in your image is a potential vulnerability. Smaller images have fewer packages, therefore fewer CVEs.

**CI/CD speed** — If your CI runner does not have the image cached, it pulls it on every run. Smaller images mean faster pipelines.

---

## Technique 1: Choose the Right Base Image

The base image is the largest lever you have for image size. We covered this in depth in the base image selection file, but here is a quick reminder of the impact:

```bash
# Full Python image
FROM python:3.11            # ~922MB base

# Slim variant
FROM python:3.11-slim       # ~128MB base  ← 7x smaller

# Alpine variant
FROM python:3.11-alpine     # ~48MB base   ← 19x smaller

# After adding your app and dependencies, the final sizes are:
# Full:    ~1.1GB
# Slim:    ~300MB
# Alpine:  ~120MB
```

This single choice reduces your image by 70-90% before you do anything else.

---

## Technique 2: Use Multi-Stage Builds

Multi-stage builds remove build tools from the final image entirely. A Go application goes from 800MB to 10MB. A Node.js frontend goes from 300MB to 30MB. Covered in depth in the previous file — always use multi-stage builds for compiled languages.

---

## Technique 3: Combine RUN Instructions and Clean Up

Each `RUN` creates a layer. Data in a layer persists even if deleted in a later layer.

```dockerfile
# BAD — three layers, cleanup in a different layer does not help
RUN apt-get update
RUN apt-get install -y gcc libpq-dev
RUN rm -rf /var/lib/apt/lists/*

# GOOD — one layer, cleanup happens before layer is committed
RUN apt-get update && \
    apt-get install -y --no-install-recommends gcc libpq-dev && \
    rm -rf /var/lib/apt/lists/*
```

Always clean up in the same `RUN` instruction:

```bash
# For apt:
rm -rf /var/lib/apt/lists/*

# For pip:
pip install --no-cache-dir -r requirements.txt
# --no-cache-dir prevents caching downloaded packages

# For npm:
npm ci && npm cache clean --force

# For apk (Alpine):
apk add --no-cache package-name
# --no-cache means no package cache file is written at all
```

---

## Technique 4: Install Only What You Need

`apt-get install` pulls in recommended packages by default. Use `--no-install-recommends` to stop it:

```dockerfile
# Installs gcc AND many recommended packages you probably don't need
RUN apt-get install -y gcc

# Installs only gcc and its true dependencies
RUN apt-get install -y --no-install-recommends gcc
```

The difference can be dozens of MB for a single package.

Similarly for pip:

```dockerfile
# Install production dependencies only
RUN pip install --no-cache-dir -r requirements.txt

# NOT requirements-dev.txt (which includes pytest, black, flake8, etc.)
```

For npm:

```dockerfile
# Install production dependencies only
RUN npm ci --only=production
```

---

## Technique 5: Use .dockerignore

The `.dockerignore` file prevents unnecessary files from being copied into the image. A comprehensive `.dockerignore` can reduce image size significantly when using `COPY . .`:

```
.git
__pycache__
*.pyc
.pytest_cache
tests/
docs/
*.log
.env
.venv
node_modules
dist/
build/
coverage/
.terraform
*.tfstate
```

Without `.dockerignore`, your image might contain test fixtures, documentation, log files, and development configuration — none of which belong in a production image.

---

## Technique 6: Copy Only What the Application Needs

Instead of `COPY . .` which copies everything, copy only the files your application actually needs to run:

```dockerfile
# Copies everything — tests, docs, CI configs, etc.
COPY . .

# Copies only what the app needs to run
COPY src/ ./src/
COPY config/ ./config/
COPY static/ ./static/
```

This requires knowing your application structure well, but it keeps the image clean.

---

## Technique 7: Remove Temporary Files

Any temporary files created during build should be deleted in the same `RUN` instruction:

```dockerfile
# Download a tool, use it, remove the downloaded file
RUN curl -Lo /tmp/tool.tar.gz https://example.com/tool.tar.gz && \
    tar -xzf /tmp/tool.tar.gz -C /usr/local/bin/ && \
    rm /tmp/tool.tar.gz && \
    chmod +x /usr/local/bin/tool

# Compile something and clean up sources
RUN cd /tmp && \
    git clone --depth=1 https://github.com/example/lib.git && \
    cd lib && make && make install && \
    cd / && rm -rf /tmp/lib
```

---

## Technique 8: Flatten Layers (Last Resort)

For images that have grown large due to many layers, you can flatten them into a single layer:

```bash
# Export and import to squash all layers into one
docker run myapp:1.0 | \    # run a container
docker export | \            # export its filesystem (single layer)
docker import - myapp:flattened  # import as a flat image

# More practical with buildx squash (experimental)
docker build --squash -t myapp:1.0 .
```

**Warning:** Squashing removes the layer sharing benefit. If multiple images share base layers, squashing means each image stores a separate copy. Use this only as a last resort.

---

## Measuring Image Size

Always measure before and after optimizations:

```bash
# Basic size check
docker images myapp

# History — see which layers are largest
docker history myapp:1.0

# Sort layers by size
docker history --format "{{.Size}}\t{{.CreatedBy}}" myapp:1.0 | sort -rh

# Detailed analysis with dive
dive myapp:1.0
```

### Interpreting dive Output

`dive` shows you each layer and what files it added or changed. Look for:
- Layers with large sizes from files you did not expect
- Files in the wrong place (test data, build artifacts, secrets)
- Files that were deleted in later layers but are still in earlier ones

---

## Real-World Before and After Examples

### Python Flask API

**Before optimization:**
```dockerfile
FROM python:3.11
COPY . .
RUN pip install -r requirements.txt
CMD ["python", "app.py"]
```
Image size: **1.1GB**

**After optimization:**
```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY src/ ./src/

RUN groupadd --gid 1001 appgroup && \
    useradd --uid 1001 --gid appgroup --no-create-home appuser

USER appuser

CMD ["python", "src/app.py"]
```
Image size: **185MB** — **83% smaller**

### Node.js React Frontend

**Before optimization:**
```dockerfile
FROM node:20
COPY . .
RUN npm install && npm run build
CMD ["node", "server.js"]
```
Image size: **980MB**

**After optimization (multi-stage):**
```dockerfile
FROM node:20-alpine AS builder
WORKDIR /build
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:1.25-alpine
COPY --from=builder /build/dist /usr/share/nginx/html
```
Image size: **28MB** — **97% smaller**

### Go REST API

**Before optimization:**
```dockerfile
FROM golang:1.21
COPY . .
RUN go build -o server .
CMD ["./server"]
```
Image size: **850MB**

**After optimization (multi-stage with scratch):**
```dockerfile
FROM golang:1.21-alpine AS builder
WORKDIR /build
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o server .

FROM scratch
COPY --from=builder /build/server /server
CMD ["/server"]
```
Image size: **12MB** — **98.6% smaller**

---

## When Good Enough is Enough

Optimization has diminishing returns. Prioritize:

1. **Always do** — choose slim/alpine base image (huge impact, minimal effort)
2. **Always do** — use multi-stage builds for compiled languages (huge impact)
3. **Always do** — combine RUN instructions and clean up (medium impact)
4. **Always do** — `.dockerignore` with comprehensive exclusions (medium impact)
5. **Consider** — copy only needed files instead of `COPY . .` (small impact)
6. **Rarely needed** — layer flattening (marginal impact, adds complexity)

Getting from 1GB to 200MB is easy and well worth the effort. Getting from 200MB to 180MB requires significant work for minimal gain.

---

## Summary

- The most impactful size reduction is choosing the right base image — `python:3.11-slim` instead of `python:3.11` cuts 800MB immediately
- Multi-stage builds eliminate build tools from production images — the most powerful technique for compiled languages
- Always combine related `RUN` instructions and clean up (apt cache, pip cache, temp files) in the same layer
- Use `--no-install-recommends` with `apt-get install` and `--no-cache-dir` with `pip install` to avoid storing unnecessary files
- A comprehensive `.dockerignore` prevents tests, docs, logs, and secrets from entering the image through `COPY . .`
- Always measure before and after — use `docker history`, `docker images`, and `dive` to find where size is coming from
- Focus optimization effort where it matters: the difference between 1GB and 200MB is worth optimizing; from 200MB to 180MB is not

In the next chapter, we cover image concepts — versioning, tagging strategies, and security scanning.
