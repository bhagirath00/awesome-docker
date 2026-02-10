# Dockerfile Overview

## Introduction

A Dockerfile is a plain text file that contains all the instructions needed to build a Docker image. It is the blueprint — you write it once, and Docker uses it to reproduce the exact same image every time it is built, on any machine, in any environment.

Mastering Dockerfiles is one of the most important skills you can develop as a DevOps engineer. Nearly every application you deploy to AWS ECS, EKS, or Fargate starts with a Dockerfile. The quality of your Dockerfile directly affects your image size, build speed, security posture, and application reliability.

---

## What a Dockerfile Does

A Dockerfile describes a series of steps to build an image. Docker reads the file top to bottom, executes each instruction, and creates a layer for each instruction that modifies the filesystem. The final stack of layers is your image.

A Dockerfile answers the questions:
- What base environment should this image start from?
- What files need to be added to the image?
- What commands need to run to set up the application?
- What should happen when a container starts from this image?

---

## Your First Dockerfile

Here is a minimal but complete Dockerfile for a Python Flask web application:

```dockerfile
# Start from an official Python image
FROM python:3.11-slim

# Set the working directory inside the container
WORKDIR /app

# Copy dependency list first
COPY requirements.txt .

# Install dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Copy the application code
COPY . .

# Tell Docker what port the app listens on
EXPOSE 8080

# The command to run when the container starts
CMD ["python", "app.py"]
```

Build this into an image:

```bash
docker build -t myflaskapp:1.0 .
```

Run a container from it:

```bash
docker run -d -p 8080:8080 --name flask-api myflaskapp:1.0
```

That is the complete cycle — write the Dockerfile, build the image, run the container.

---

## The Dockerfile Format

Every line in a Dockerfile is either:
- An **instruction** — a keyword in uppercase followed by its arguments
- A **comment** — a line starting with `#`
- A **blank line** — ignored by Docker

```dockerfile
# This is a comment
FROM ubuntu:22.04          # instruction: keyword + argument

# Blank lines are ignored

RUN apt-get update         # another instruction
```

Instructions are case-insensitive — `FROM` and `from` both work. However, the convention is to write instructions in uppercase so they stand out clearly from the arguments.

---

## The Most Important Dockerfile Instructions

Here is a quick overview of every instruction you will use. Each one gets a deep explanation in the next file.

### FROM

Every Dockerfile must start with `FROM`. It sets the base image — the starting point for your build.

```dockerfile
FROM python:3.11-slim
FROM node:20-alpine
FROM ubuntu:22.04
FROM scratch   # empty base — for static binaries
```

### WORKDIR

Sets the working directory for all subsequent instructions. If the directory does not exist, Docker creates it.

```dockerfile
WORKDIR /app
```

### COPY

Copies files from the build context (your local machine) into the image.

```dockerfile
COPY requirements.txt .
COPY src/ /app/src/
COPY . .
```

### RUN

Executes a command during the build. The result becomes a new image layer.

```dockerfile
RUN pip install -r requirements.txt
RUN apt-get update && apt-get install -y curl
RUN npm install
```

### ENV

Sets environment variables that are available at build time and at runtime.

```dockerfile
ENV APP_ENV=production
ENV PORT=8080
```

### ARG

Defines build-time variables — passed with `--build-arg` during `docker build`. Not available at runtime.

```dockerfile
ARG APP_VERSION=1.0
ARG BUILD_ENV=production
```

### EXPOSE

Documents the port the container listens on. Does not actually open or publish the port.

```dockerfile
EXPOSE 8080
EXPOSE 80 443
```

### CMD

Sets the default command that runs when the container starts. Can be overridden by the user.

```dockerfile
CMD ["python", "app.py"]
CMD ["nginx", "-g", "daemon off;"]
```

### ENTRYPOINT

Sets the main executable that always runs when the container starts. CMD provides default arguments to it.

```dockerfile
ENTRYPOINT ["python"]
CMD ["app.py"]    # default argument — can be overridden
```

### USER

Sets the user for subsequent instructions and the final container process. Critical for security — do not run as root.

```dockerfile
USER appuser
```

### LABEL

Adds metadata key-value pairs to the image.

```dockerfile
LABEL version="1.0"
LABEL maintainer="team@example.com"
LABEL environment="production"
```

### VOLUME

Declares a mount point for external volumes. Signals that this path should be managed as a volume.

```dockerfile
VOLUME ["/app/data"]
```

### ADD

Like COPY but also supports URLs and automatically extracts tar archives.

```dockerfile
ADD https://example.com/file.tar.gz /tmp/
ADD myapp.tar.gz /app/      # extracts automatically
```

In most cases, prefer `COPY` over `ADD` — it is more explicit and predictable. Use `ADD` only when you need the automatic extraction feature.

### HEALTHCHECK

Defines a command Docker uses to check if the container is healthy.

```dockerfile
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1
```

---

## The .dockerignore File

Before Docker sends your build context to the daemon, it filters out files listed in `.dockerignore`. This is placed in the same directory as your Dockerfile.

```
# .dockerignore
.git
.env
__pycache__
*.pyc
*.pyo
.pytest_cache
tests/
docs/
*.log
node_modules
.terraform
terraform.tfstate
*.tfvars
```

Without `.dockerignore`:
- Your `.git` directory is sent to the daemon (slow, large)
- Your `.env` file with secrets is sent (security risk — secrets can end up in the image)
- All your test files are included in the image (larger image)

With `.dockerignore`, only what the build actually needs is sent. This makes builds faster and images smaller.

---

## Dockerfile Location and Naming

By default, Docker looks for a file named `Dockerfile` in the build context directory:

```bash
# Uses Dockerfile in current directory
docker build -t myapp:1.0 .
```

You can specify a different filename with `-f`:

```bash
# Use a different Dockerfile
docker build -f Dockerfile.prod -t myapp:prod .
docker build -f Dockerfile.dev -t myapp:dev .
```

Common naming patterns:

```
project/
├── Dockerfile          # default production Dockerfile
├── Dockerfile.dev      # development Dockerfile
├── Dockerfile.prod     # explicit production Dockerfile
├── Dockerfile.test     # testing Dockerfile
└── .dockerignore
```

---

## A Complete Real-World Example

Here is a production-quality Dockerfile for a Node.js API:

```dockerfile
# Use specific version — never use latest in production Dockerfiles
FROM node:20-alpine AS base

# Install security updates
RUN apk update && apk upgrade --no-cache

# Create non-root user for security
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# Set working directory
WORKDIR /app

# Copy package files first to leverage layer caching
COPY package.json package-lock.json ./

# Install production dependencies only
RUN npm ci --only=production && npm cache clean --force

# Copy application source code
COPY src/ ./src/
COPY config/ ./config/

# Set environment
ENV NODE_ENV=production
ENV PORT=3000

# Document the port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
  CMD node -e "require('http').get('http://localhost:3000/health', (r) => r.statusCode === 200 ? process.exit(0) : process.exit(1))"

# Switch to non-root user
USER appuser

# Start the application
CMD ["node", "src/index.js"]
```

This Dockerfile demonstrates:
- Specific version pinning (`node:20-alpine`)
- Security updates applied in base
- Non-root user created and used
- Dependency file copied before app code (cache optimization)
- Production-only dependencies (`npm ci --only=production`)
- Health check configured
- Port documented
- Minimal and clean structure

---

## Summary

- A Dockerfile is a plain text file with instructions that Docker reads top to bottom to build an image
- Every filesystem-modifying instruction creates a new image layer
- Key instructions: `FROM`, `WORKDIR`, `COPY`, `RUN`, `ENV`, `EXPOSE`, `CMD`, `ENTRYPOINT`, `USER`, `LABEL`, `HEALTHCHECK`
- Always create a `.dockerignore` file to exclude unnecessary files from the build context — prevents secrets from leaking into images and speeds up builds
- Use specific image version tags in `FROM` instructions — never `FROM python:latest` in production
- A well-written Dockerfile produces a small, fast-building, secure image that behaves identically everywhere it runs

In the next file, we go deep into every Dockerfile instruction — exactly how each one works, the correct syntax, and the common mistakes to avoid.
