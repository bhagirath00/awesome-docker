# What is a Docker Image?

## Introduction

You have been using Docker images throughout this course — pulling `nginx`, `python:3.11`, and `postgres:16`. But what exactly is an image? What is inside it? How does Docker use it to create containers?

Understanding images deeply makes you better at every Docker task: writing efficient Dockerfiles, debugging build issues, managing disk space, and designing production workflows. This file gives you that deep understanding.

---

## The Definition

A **Docker image** is a read-only, layered package that contains everything needed to run an application:

- The application code
- The runtime (Python, Node.js, Java, etc.)
- Libraries and dependencies
- Configuration files
- Environment variable defaults
- A default command to run

An image is static — it never changes. Once built, an image is immutable. You cannot modify a running image.

When Docker creates a container from an image, it adds a thin writable layer on top of the image layers. The container reads from the image layers (which are shared) and writes only to its own writable layer.

---

## Image vs Container — One More Time

This distinction is so important it bears repeating clearly:

```
Image (read-only blueprint)
    └── Container 1 (writable layer on top)
    └── Container 2 (writable layer on top)
    └── Container 3 (writable layer on top)
```

All three containers share the exact same image layers. Each has its own writable layer for its own changes. This is why Docker is efficient — you can run 10 nginx containers and they all share one copy of the nginx image on disk.

```bash
# See this in action — pull one image, run three containers
docker pull nginx:1.25.3

# All three containers use the same image layers on disk
docker run -d --name web1 -p 8081:80 nginx:1.25.3
docker run -d --name web2 -p 8082:80 nginx:1.25.3
docker run -d --name web3 -p 8083:80 nginx:1.25.3

# Check disk usage — three containers, but one copy of the image
docker system df
# Images: 1 image, ~50MB
# Containers: 3 containers, ~0B (writable layers are tiny until written to)
```

---

## Image Names and Tags

Docker image names follow a specific format:

```
[registry/][namespace/]repository[:tag]
```

Examples:

```bash
nginx                          # Docker Hub, official image, latest tag
nginx:1.25.3                   # Docker Hub, official image, specific version
python:3.11-slim               # Docker Hub, official, slim variant
ubuntu:22.04                   # Docker Hub, official, specific Ubuntu version

myusername/myapp               # Docker Hub, user namespace, latest
myusername/myapp:1.0           # Docker Hub, user namespace, version 1.0

123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:1.0   # AWS ECR private image
ghcr.io/myorg/myapp:main                               # GitHub Container Registry
```

### Tags

A tag is a label that points to a specific version of an image. Without a tag, Docker uses `latest`.

```bash
# These two commands pull the same thing
docker pull nginx
docker pull nginx:latest
```

**Warning about `latest`:** The `latest` tag is mutable. It points to whatever was most recently pushed without a specific tag. If you pull `nginx:latest` today and again in 6 months, you may get a different image. Always use specific version tags in production:

```bash
# Bad — which version of nginx is this?
docker run nginx

# Good — you know exactly what you are running
docker run nginx:1.25.3
```

### Image Digest

Every image also has a content-addressed digest — a SHA256 hash of the image content. This is completely immutable and never changes, even if the tag changes:

```bash
# Pull by digest — absolute guarantee of the exact image
docker pull nginx@sha256:32da30332506740a2f7c34d5dc70467b7f14ec67d6b2660a57db8e2e55c3ef0f

# Get the digest of a local image
docker inspect --format '{{index .RepoDigests 0}}' nginx:1.25.3
```

In security-sensitive environments, pin images by digest instead of tag.

---

## Image Variants

Most official Docker images offer multiple variants for different use cases:

### Base OS Variants

```bash
# Full Ubuntu-based image — largest, most compatible
python:3.11

# Debian Slim — smaller, removes many optional packages
python:3.11-slim

# Alpine Linux — smallest, uses musl libc instead of glibc
python:3.11-alpine

# Distroless — no shell, no package manager (most secure, harder to debug)
gcr.io/distroless/python3
```

### Choosing the Right Variant

| Variant | Size | Compatibility | Security | Best For |
|---|---|---|---|---|
| Full (Ubuntu/Debian) | ~900MB | Highest | Lower | Development, debugging |
| Slim (Debian Slim) | ~100-200MB | High | Medium | Production general use |
| Alpine | ~20-50MB | Medium | Higher | Production where compatibility is confirmed |
| Distroless | ~5-30MB | Lowest | Highest | Production security-critical workloads |

Start with the slim variant for production. Only move to Alpine if you have confirmed all your dependencies compile correctly against musl libc.

```bash
# Compare sizes
docker pull python:3.11
docker pull python:3.11-slim
docker pull python:3.11-alpine

docker images python
# REPOSITORY   TAG         IMAGE ID       SIZE
# python       3.11        abc123         921MB
# python       3.11-slim   def456         128MB
# python       3.11-alpine ghi789         48MB
```

---

## What is Inside an Image?

Every Docker image is a collection of layers stacked on top of each other. Each layer represents a change — a set of files added, modified, or deleted.

The layers are assembled in order from bottom to top:

```
Layer 5: COPY . /app         ← your application code
Layer 4: RUN pip install...  ← Python dependencies installed
Layer 3: COPY requirements.txt .  ← requirements file added
Layer 2: WORKDIR /app        ← working directory set
Layer 1: FROM python:3.11-slim  ← base OS and Python runtime
```

You can see the layers of any image:

```bash
docker history python:3.11-slim

# Output:
IMAGE          CREATED       CREATED BY                                SIZE
abc123f        3 days ago    CMD ["python3"]                           0B
<missing>      3 days ago    ENTRYPOINT ["python3"]                    0B
<missing>      3 days ago    ENV PYTHON_GET_PIP_URL=...               0B
<missing>      3 days ago    RUN /bin/sh -c set -eux; ...             34.4MB
<missing>      3 days ago    RUN /bin/sh -c apt-get update ...        8.2MB
<missing>      3 days ago    FROM debian:bookworm-slim                 74.8MB
```

Each line is a layer. The `SIZE` column shows how much disk space that layer adds.

---

## How Image Layers Work

### Layers are Read-Only and Shared

Once a layer is created, it never changes. If two images share the same base layer (e.g., both use `python:3.11-slim`), Docker stores that layer only once on disk and shares it between images.

```bash
# Build your app image based on python:3.11-slim
docker build -t myapp:1.0 .
docker build -t myapp:1.1 .  # New version with code changes

# myapp:1.0 and myapp:1.1 share all the base layers
# Only the layer with your changed code is different
docker system df
# The shared layers are counted once, not twice
```

### The Union Filesystem

Docker uses a union filesystem (overlayfs on modern Linux) to stack image layers and present them as a single unified filesystem to the container.

When you read a file inside a container:
1. Docker checks the top layer (writable container layer)
2. If not found, checks the next layer down
3. Continues until the file is found

When you write a file:
1. Docker copies the file from whatever layer it is in to the writable container layer
2. Modifies the copy in the writable layer
3. The original in the image layer is untouched

This is called **copy-on-write** — you only copy when you write. Reading is always fast because you read directly from the layer where the file lives.

---

## Image Metadata

Every image contains metadata that Docker uses when creating containers:

```bash
# View image metadata
docker inspect python:3.11-slim

# Key metadata fields:
# - Config.Cmd        — default command to run
# - Config.Entrypoint — entry point command
# - Config.Env        — default environment variables
# - Config.WorkingDir — default working directory
# - Config.ExposedPorts — ports the image expects to be mapped
# - Config.Labels     — key-value metadata
```

Specific fields:

```bash
# What command does this image run by default?
docker inspect --format '{{json .Config.Cmd}}' python:3.11-slim
# ["python3"]

# What environment variables are set?
docker inspect --format '{{range .Config.Env}}{{println .}}{{end}}' python:3.11-slim

# What ports does this image expose?
docker inspect --format '{{json .Config.ExposedPorts}}' nginx:1.25.3
# {"80/tcp":{}}
```

---

## Official Images vs Community Images

### Official Images

Official images are maintained by Docker or verified software publishers. They are:

- Reviewed by Docker's security team
- Updated regularly with security patches
- Well-documented on Docker Hub
- Named without a namespace (e.g., `nginx`, `python`, `postgres`)

Always prefer official images as your base image:

```bash
# Official images — use these
FROM python:3.11-slim
FROM node:20-alpine
FROM nginx:1.25.3
FROM postgres:16
FROM redis:7-alpine
FROM ubuntu:22.04
```

### Community Images

Community images are maintained by individual developers or organizations. They include a namespace:

```bash
# Community image — has a namespace
FROM hashicorp/terraform:1.6.6
FROM bitnami/kafka:latest
```

Only use community images from well-known, trusted organizations. Always check:
- When was the image last updated?
- Does the organization have a reputation for security?
- Is the Dockerfile available to inspect?

### Verified Publisher Images

These are between official and community — companies like AWS, HashiCorp, and Microsoft publish images that are vetted by Docker:

```bash
FROM amazon/aws-cli:latest
FROM hashicorp/vault:1.15
```

---

## Searching for Images

```bash
# Search Docker Hub from the CLI
docker search nginx

# Search with filter — only official images
docker search --filter is-official=true python

# Search with a minimum star rating
docker search --filter stars=1000 redis
```

For more details, browse Docker Hub directly at `https://hub.docker.com`.

---

## Summary

- A Docker image is a read-only, layered package containing everything needed to run an application — code, runtime, dependencies, and configuration
- Images are immutable — once built, they never change
- A container is a running instance of an image — many containers can share one image, each with their own writable layer
- Image names follow the format `[registry/][namespace/]repository[:tag]` — always use specific tags in production, never `latest`
- Images come in variants (full, slim, alpine, distroless) — start with slim for production
- Image layers are read-only and shared between images — Docker only stores each unique layer once on disk
- Docker uses copy-on-write and a union filesystem to make layer stacking fast and efficient

In the next file, we go deeper into image layers — how they are created, how they affect image size, and how to optimize them for production use.
