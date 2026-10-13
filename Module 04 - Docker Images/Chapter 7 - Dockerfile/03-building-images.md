# Building Images

## Introduction

Writing a Dockerfile is only half the work. The other half is understanding how `docker build` works — what options you have, how to use them effectively, how to debug build failures, and how to make builds fast and reproducible in CI/CD pipelines.

This file covers the complete build process from the command you run to the image that comes out the other end.

---

## The docker build Command

The basic syntax:

```
docker build [OPTIONS] PATH | URL | -
```

- `PATH` — the build context directory (usually `.` for current directory)
- `URL` — a Git repository URL to use as the build context
- `-` — read the Dockerfile from stdin

```bash
# Most common form — build from current directory
docker build -t myapp:1.0 .

# Specify a different Dockerfile
docker build -f Dockerfile.prod -t myapp:prod .

# Build from a Git repository
docker build -t myapp:1.0 https://github.com/myorg/myapp.git

# Build from a specific branch
docker build -t myapp:1.0 https://github.com/myorg/myapp.git#main

# Build from stdin (Dockerfile piped in)
cat Dockerfile | docker build -t myapp:1.0 -
```

---

## What Happens During a Build

When you run `docker build -t myapp:1.0 .`, Docker does the following:

**Step 1: Package the build context**

Docker packages all files in the build context directory (`.`) and sends them to the Docker daemon. Files listed in `.dockerignore` are excluded before packaging.

```
Sending build context to Docker daemon  24.58kB
```

If this number is large (megabytes or gigabytes), you have a poorly configured `.dockerignore`. You do not want to send your `.git` directory, `node_modules`, or test data to the daemon.

**Step 2: Parse the Dockerfile**

The daemon reads the Dockerfile and processes each instruction in order.

**Step 3: Execute each instruction**

For each instruction:
- Docker checks the build cache for a matching layer
- If found: `---> Using cache`
- If not found: executes the instruction and creates a new layer

**Step 4: Output the final image**

After all instructions are processed, Docker creates the final image and applies the tag you specified with `-t`.

```
Successfully built a1b2c3d4ef56
Successfully tagged myapp:1.0
```

---

## Build Options

### -t / --tag — Naming the Image

```bash
# Single tag
docker build -t myapp:1.0 .

# Multiple tags in one build
docker build \
  -t myapp:1.4.2 \
  -t myapp:1.4 \
  -t myapp:latest \
  -t 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:1.4.2 \
  .
```

Building with multiple tags is more efficient than building once and then running `docker tag` multiple times — the image is built only once.

### -f / --file — Specify Dockerfile

```bash
docker build -f Dockerfile.prod -t myapp:prod .
docker build -f docker/Dockerfile.worker -t worker:1.0 .
```

### --build-arg — Pass Build Arguments

Build arguments let you pass dynamic values into the build without hardcoding them in the Dockerfile:

```bash
docker build \
  --build-arg APP_VERSION=1.4.2 \
  --build-arg BUILD_DATE=$(date -u +'%Y-%m-%dT%H:%M:%SZ') \
  --build-arg GIT_COMMIT=$(git rev-parse HEAD) \
  --build-arg GIT_BRANCH=$(git rev-parse --abbrev-ref HEAD) \
  -t myapp:1.4.2 .
```

In the Dockerfile:

```dockerfile
ARG APP_VERSION=unknown
ARG BUILD_DATE=unknown
ARG GIT_COMMIT=unknown

LABEL version="${APP_VERSION}" \
      build-date="${BUILD_DATE}" \
      git-commit="${GIT_COMMIT}"
```

### --no-cache — Force Full Rebuild

```bash
# Ignore all cached layers and rebuild from scratch
docker build --no-cache -t myapp:1.0 .
```

Use this when:
- You suspect a cached layer contains stale data (e.g., `apt-get update` cached an old package list)
- You want to ensure the final image has the latest security patches
- Debugging a build that worked before but is now failing despite no Dockerfile changes

### --target — Build a Specific Stage

In multi-stage Dockerfiles, build only up to a specific stage:

```dockerfile
FROM python:3.11-slim AS base
RUN apt-get update && apt-get install -y --no-install-recommends gcc

FROM base AS development
RUN pip install -r requirements-dev.txt
CMD ["python", "-m", "debugpy", "--listen", "0.0.0.0:5678", "app.py"]

FROM base AS production
RUN pip install -r requirements.txt
CMD ["python", "app.py"]
```

```bash
# Build only the development stage
docker build --target development -t myapp:dev .

# Build the full production stage
docker build --target production -t myapp:prod .
```

### --platform — Build for a Specific Architecture

```bash
# Build for Intel/AMD (most common EC2 instances)
docker build --platform linux/amd64 -t myapp:1.0 .

# Build for ARM64 (AWS Graviton — cheaper and faster for many workloads)
docker build --platform linux/arm64 -t myapp:1.0 .

# Build for both (requires buildx)
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:1.0 \
  --push \
  .
```

### --progress — Control Build Output

```bash
# Default — shows progress bar and status
docker build -t myapp:1.0 .

# Full output — shows every command and its output
docker build --progress=plain -t myapp:1.0 .

# Quiet — only shows the final image ID
docker build -q -t myapp:1.0 .
```

Use `--progress=plain` when debugging a failing build — you see the complete output of every `RUN` command.

---

## Debugging Build Failures

When a build fails, Docker tells you which step failed. Your job is to understand why.

### Read the Error Carefully

```bash
docker build -t myapp:1.0 .

# Output:
Step 4/7 : RUN pip install -r requirements.txt
 ---> Running in abc123def456
Collecting flask==3.0.0
  Downloading Flask-3.0.0-py3-none-any.whl (92 kB)
ERROR: Could not find a version that satisfies the requirement psycopg2==2.9.99 (from versions: 2.9.1, 2.9.2, 2.9.3, 2.9.4, 2.9.5, 2.9.6, 2.9.7, 2.9.8, 2.9.9)
ERROR: No matching distribution found for psycopg2==2.9.99
```

The error is clear — `psycopg2==2.9.99` does not exist. The version in `requirements.txt` is wrong.

### Debug by Getting a Shell at the Failure Point

If the error is not obvious, build up to the failing step and get a shell:

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt   # this fails
COPY . .
CMD ["python", "app.py"]
```

The layer just before the failing `RUN` is still cached. Start a container from the last successful layer:

```bash
# Build shows failure at step 4
# The last successful step created a layer for COPY requirements.txt

# Get the ID of the cached layer
docker build --progress=plain -t myapp:1.0 . 2>&1 | grep "Running in"
# ---> Running in abc123def456

# Start a container at that layer
docker run -it abc123def456 bash

# Inside the container, manually run the failing command to see the full error
pip install -r requirements.txt
```

This technique is invaluable for debugging complex build failures.

### Check the Build Context Size

```bash
# If the build is slow at "Sending build context"
docker build -t myapp:1.0 .
# Sending build context to Docker daemon  985.7MB  ← too large

# Check what is in the current directory
du -sh * | sort -rh | head -20

# Add large directories to .dockerignore
echo "node_modules" >> .dockerignore
echo ".git" >> .dockerignore
echo "*.tar.gz" >> .dockerignore
```

---

## Building in CI/CD

Efficient Docker builds in CI/CD are critical for fast deployments. Here are production-grade patterns.

### GitHub Actions — Complete Build and Push

```yaml
name: Build and Push

on:
  push:
    branches: [main]

env:
  AWS_REGION: us-east-1
  ECR_REPOSITORY: myapp

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build, test, and push image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          # Build
          docker build \
            --build-arg GIT_COMMIT=${{ github.sha }} \
            --build-arg BUILD_DATE=$(date -u +'%Y-%m-%dT%H:%M:%SZ') \
            -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG \
            -t $ECR_REGISTRY/$ECR_REPOSITORY:latest \
            .
          
          # Test
          docker run --rm \
            -e TEST_ENV=true \
            $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG \
            python -m pytest tests/ -v
          
          # Push
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:latest
```

### Using GitHub Actions Cache for Build Layers

```yaml
- name: Build and push with cache
  uses: docker/build-push-action@v5
  with:
    context: .
    push: true
    tags: |
      ${{ steps.login-ecr.outputs.registry }}/myapp:${{ github.sha }}
      ${{ steps.login-ecr.outputs.registry }}/myapp:latest
    build-args: |
      GIT_COMMIT=${{ github.sha }}
      BUILD_DATE=${{ steps.date.outputs.date }}
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

The `cache-from: type=gha` and `cache-to: type=gha,mode=max` lines tell Docker Buildx to store and retrieve layer cache from GitHub's cache system. Subsequent builds only rebuild layers that changed.

---

## Verifying the Built Image

After building, always verify the image before pushing or deploying:

```bash
# Confirm the image exists with the right tag
docker images myapp:1.0

# Check image size (should be within expected range)
docker images myapp:1.0 --format "{{.Size}}"

# Run the container and verify it starts correctly
docker run -d --name test-run -p 8080:8080 myapp:1.0
sleep 5
curl -s http://localhost:8080/health
docker rm -f test-run

# Check what user the container runs as
docker run --rm myapp:1.0 whoami

# Check environment variables
docker run --rm myapp:1.0 env

# Inspect the image metadata
docker inspect myapp:1.0 | python3 -m json.tool | head -80

# Review layer sizes
docker history myapp:1.0
```

---

## Summary

- `docker build -t NAME:TAG PATH` is the basic build command — the path is the build context, not just the Dockerfile location
- Use `-t` multiple times to apply multiple tags in a single build — more efficient than tagging afterward
- Use `--build-arg` to pass dynamic values like version numbers, Git commits, and build dates without hardcoding them
- Use `--no-cache` when you need a completely clean rebuild — especially useful after updating base image tags
- Use `--target` in multi-stage builds to build only up to a specific stage
- Use `--progress=plain` to see full command output during build — essential for debugging
- In CI/CD, use `docker/build-push-action` with `cache-from` and `cache-to` to dramatically speed up repeated builds
- Always verify the built image — check it starts correctly, runs as a non-root user, and has the expected size

In the next file, we explain the build context in depth — what it is, why its size matters, and how to control it precisely.
