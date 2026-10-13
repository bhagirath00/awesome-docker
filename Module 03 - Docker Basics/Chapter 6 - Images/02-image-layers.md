# Image Layers

## Introduction

Image layers are the foundation of how Docker images work. They are what makes Docker images fast to build, efficient to store, and quick to transfer. Understanding layers is also what separates engineers who write mediocre Dockerfiles from those who write optimized, production-grade ones.

This file explains how layers are created, how they are stored and shared, how the build cache uses layers to speed up builds, and how to make the right decisions to keep your images small and your builds fast.

---

## How Layers are Created

Every instruction in a Dockerfile that modifies the filesystem creates a new layer. Instructions that do not modify the filesystem (ENV, CMD, ENTRYPOINT, EXPOSE, LABEL) add metadata but do not create file layers.

Layer-creating instructions:
- `FROM` — starts a new layer from a base image
- `RUN` — executes a command; the resulting filesystem changes become a layer
- `COPY` — copies files from the build context into the image as a new layer
- `ADD` — like COPY but also supports URLs and auto-extracting archives

Non-layer instructions (metadata only):
- `ENV` — sets environment variables
- `CMD` — sets the default command
- `ENTRYPOINT` — sets the container entrypoint
- `EXPOSE` — documents a port (does not actually open it)
- `LABEL` — adds metadata
- `WORKDIR` — sets the working directory (creates the dir if needed, but no file layer)
- `USER` — sets the user for subsequent instructions

---

## Visualizing Layers

Take this Dockerfile:

```dockerfile
FROM python:3.11-slim          # Layer 1: base image (74.8MB)
WORKDIR /app                   # Metadata only
COPY requirements.txt .        # Layer 2: requirements file added (tiny)
RUN pip install -r requirements.txt  # Layer 3: Python packages installed (large)
COPY . .                       # Layer 4: application code copied (varies)
ENV APP_ENV=production          # Metadata only
CMD ["python", "app.py"]       # Metadata only
```

The resulting image has 4 file layers stacked on each other. You can see them:

```bash
docker build -t myapp:1.0 .
docker history myapp:1.0
```

Output:

```
IMAGE          CREATED         CREATED BY                              SIZE
a1b2c3d4       2 minutes ago   CMD ["python" "app.py"]                0B
<missing>      2 minutes ago   ENV APP_ENV=production                 0B
<missing>      2 minutes ago   COPY . . # buildkit                    24.5kB
<missing>      2 minutes ago   RUN pip install -r requirements.txt    89.3MB
<missing>      2 minutes ago   COPY requirements.txt .                342B
<missing>      2 minutes ago   WORKDIR /app                           0B
<missing>      3 days ago      CMD ["python3"]                        0B
<missing>      3 days ago      FROM python:3.11-slim                  74.8MB
```

Each layer is identified by a content hash. If the content of a layer has not changed, its hash remains the same.

---

## Layer Sharing and Storage Efficiency

Docker stores layers in a content-addressed cache. Each unique layer is stored only once, even if multiple images use it.

```bash
# You build three versions of your app
docker build -t myapp:1.0 .
docker build -t myapp:1.1 .   # Changed app code only
docker build -t myapp:2.0 .   # Changed requirements too

# Check disk usage
docker system df

# Even with 3 images, the base python:3.11-slim layer
# is stored only once. The pip install layer for 1.0 and 1.1 are the same.
# Only the truly different layers take extra space.
```

This sharing is why pulling a new version of your application image is fast — Docker only downloads the layers that changed. If your base layer and dependencies did not change, only the code layer needs to be downloaded.

---

## The Build Cache

The build cache is Docker's system for reusing layers from previous builds. This is what makes iterative Docker development fast.

When you build an image:
1. Docker reads each Dockerfile instruction in order
2. For each instruction, Docker checks if there is a cached layer with the same content hash
3. If yes — it reuses the cached layer (fast, no work needed)
4. If no — it executes the instruction and creates a new layer
5. **Once a cache miss occurs, all subsequent layers are also rebuilt** — even if they have not changed

This last point is critical. It explains why the order of instructions in a Dockerfile matters enormously.

### Cache Miss in Action

```bash
# First build — no cache, everything is built fresh
docker build -t myapp:1.0 .

# Output:
# Step 1/5 : FROM python:3.11-slim
#  ---> 74.8MB layer downloaded
# Step 2/5 : WORKDIR /app
#  ---> Running in abc123
# Step 3/5 : COPY requirements.txt .
#  ---> Layer created
# Step 4/5 : RUN pip install -r requirements.txt
#  ---> Running in def456 (takes 30 seconds)
# Step 5/5 : COPY . .
#  ---> Layer created

# Second build — no changes made
docker build -t myapp:1.0 .

# Output:
# Step 1/5 : FROM python:3.11-slim
#  ---> Using cache
# Step 2/5 : WORKDIR /app
#  ---> Using cache
# Step 3/5 : COPY requirements.txt .
#  ---> Using cache
# Step 4/5 : RUN pip install -r requirements.txt
#  ---> Using cache
# Step 5/5 : COPY . .
#  ---> Using cache
# Built in 0.1 seconds
```

The second build uses all cached layers and finishes instantly.

Now change your application code (not requirements.txt):

```bash
# Third build — app code changed
docker build -t myapp:1.1 .

# Step 1/5 : FROM python:3.11-slim
#  ---> Using cache
# Step 2/5 : WORKDIR /app
#  ---> Using cache
# Step 3/5 : COPY requirements.txt .
#  ---> Using cache
# Step 4/5 : RUN pip install -r requirements.txt
#  ---> Using cache  ← pip install is NOT re-run
# Step 5/5 : COPY . .
#  ---> Layer created  ← only this changes
# Built in 1.2 seconds
```

Because `requirements.txt` did not change, the `pip install` layer is reused. Only the code layer is rebuilt. This is why copying `requirements.txt` and installing dependencies before copying the app code is the correct pattern.

---

## The Layer Ordering Mistake

The most common Dockerfile mistake that kills build performance is copying all files before installing dependencies:

```dockerfile
# BAD — wrong order
FROM python:3.11-slim
WORKDIR /app
COPY . .                        # ← copies everything including app code
RUN pip install -r requirements.txt  # ← always re-runs, even if only code changed
```

Any change to any file in your application directory invalidates the `COPY . .` layer and triggers a full `pip install`. If pip install takes 2 minutes, you waste 2 minutes on every code change.

```dockerfile
# GOOD — correct order
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .          # ← copy only requirements first
RUN pip install -r requirements.txt  # ← cached unless requirements change
COPY . .                         # ← copy app code last
```

Now pip install only re-runs when requirements.txt changes. Code-only changes skip pip install entirely.

---

## Layer Size and Image Bloat

Every layer adds to the total image size. Several common mistakes cause unnecessarily large images.

### Mistake 1: Installing and Not Cleaning Up in Separate RUN Instructions

```dockerfile
# BAD — each RUN creates a separate layer
RUN apt-get update
RUN apt-get install -y gcc
RUN apt-get install -y libpq-dev
RUN rm -rf /var/lib/apt/lists/*   # ← this cleans up in a NEW layer
# The previous layers still contain the package cache
```

```dockerfile
# GOOD — one RUN instruction, cleanup in the same layer
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
      gcc \
      libpq-dev && \
    rm -rf /var/lib/apt/lists/*   # ← cleanup in the SAME RUN
```

The cleanup must happen in the same `RUN` instruction as the installation. If you put the cleanup in a separate `RUN`, the installation layer still exists with all its size.

### Mistake 2: Leaving Build Tools in the Final Image

```dockerfile
# BAD — build tools end up in production image
FROM python:3.11
RUN apt-get update && apt-get install -y gcc libpq-dev
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
CMD ["python", "app.py"]
# Image includes gcc and libpq-dev even though they're only needed to build
```

The solution is multi-stage builds (covered in the Dockerfiles module), but the key idea is: do not leave build-time tools in your final image.

### Mistake 3: Copying Everything Including Unnecessary Files

```dockerfile
# BAD — copies test files, docs, .git, etc.
COPY . .
```

```dockerfile
# GOOD — use .dockerignore to exclude unnecessary files
# Create .dockerignore in the same directory as Dockerfile:
```

```
# .dockerignore
.git
__pycache__
*.pyc
*.pyo
.pytest_cache
tests/
docs/
.env
*.log
node_modules
```

With `.dockerignore`, Docker excludes those files from the build context entirely. This makes builds faster (less data to send to the daemon) and keeps the image smaller.

---

## Inspecting Layer Content

You can look inside image layers to understand what they contain:

```bash
# Save an image as a tar archive
docker save myapp:1.0 -o myapp.tar

# Extract and inspect
mkdir myapp-layers && cd myapp-layers
tar xf ../myapp.tar

# You will see directories for each layer
ls
# manifest.json
# layer1/
# layer2/
# layer3/

# Each layer directory contains a layer.tar
tar tf layer1/layer.tar | head -20
```

In practice, you rarely need to do this. But knowing that layers are just tar archives of filesystem snapshots helps you understand the mental model.

A faster way to inspect layer contents:

```bash
# Use docker inspect to see layer IDs
docker inspect --format '{{json .RootFS.Layers}}' myapp:1.0

# Dive tool — interactive layer explorer (install separately)
dive myapp:1.0
```

The `dive` tool is excellent for interactively exploring what each layer contains and identifying where image bloat comes from.

---

## Dangling Images

When you rebuild an image with the same tag, the old image layers are no longer referenced by the tag — they become **dangling images** (shown as `<none>:<none>`).

```bash
# Build version 1
docker build -t myapp:latest .

# Rebuild with changes — still tag as latest
docker build -t myapp:latest .

# The old layers are now dangling
docker images
# REPOSITORY   TAG       IMAGE ID       SIZE
# myapp        latest    a1b2c3d4       125MB
# <none>        <none>   e5f6a7b8       124MB  ← dangling
```

Dangling images waste disk space. Clean them up:

```bash
# Remove dangling images only
docker image prune

# Remove all unused images (dangling + not referenced by any container)
docker image prune -a
```

---

## Summary

- Each Dockerfile instruction that modifies the filesystem creates a new layer — layers are stacked in order to form the final image
- Layers are stored once and shared between images — Docker only stores each unique layer once on disk
- The build cache reuses layers from previous builds — once a cache miss occurs, all subsequent layers rebuild
- Always copy dependency files (requirements.txt, package.json) and install dependencies before copying application code — this maximizes cache reuse
- Always clean up (apt cache, temp files) in the same RUN instruction as the installation — not in a separate RUN
- Use `.dockerignore` to exclude unnecessary files from the build context — it reduces build time and image size
- Dangling images (shown as `<none>:<none>`) accumulate when you rebuild images with the same tag — clean them with `docker image prune`

In the next file, we cover pulling images from registries — Docker Hub and AWS ECR — and managing your local image cache.
