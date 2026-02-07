# Image Commands

## Introduction

Managing Docker images effectively is a core skill for every DevOps engineer. You need to know how to inspect images for debugging, tag images for deployment, push to registries, search for the right base image, and keep your local image cache clean.

This file covers all the image management commands in depth, with real-world patterns for how and when to use each one.

---

## docker images — Listing Local Images

The `docker images` command shows all images stored locally on your machine.

```bash
# List all local images
docker images

# Output:
REPOSITORY     TAG         IMAGE ID       CREATED        SIZE
myapp          2.0         a1b2c3d4ef56   2 hours ago    245MB
myapp          1.0         bc7d8e9f0a12   2 days ago     240MB
nginx          1.25.3      cd3e4f5a6b78   5 days ago     187MB
python         3.11-slim   ef9a0b1c2d34   8 days ago     128MB
postgres       16          12ab3c4d5e67   2 weeks ago    425MB
redis          7-alpine    89fe0a1b2c34   3 weeks ago    41MB
```

### Filtering Results

```bash
# List images for a specific repository
docker images nginx

# List images with a specific tag
docker images nginx:1.25.3

# List all images including intermediary (layers without tags)
docker images -a

# List only image IDs
docker images -q

# Filter by label
docker images --filter label=environment=production

# Filter by dangling (none:none images)
docker images --filter dangling=true

# Filter images created before a specific image
docker images --filter before=myapp:2.0

# Filter images created since a specific image
docker images --filter since=myapp:1.0

# Custom output format
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}\t{{.CreatedSince}}"
```

---

## docker pull — Downloading Images

```bash
# Pull the latest version
docker pull nginx

# Pull a specific version (always do this in production)
docker pull nginx:1.25.3

# Pull from AWS ECR
docker pull 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:1.0

# Pull for a specific platform
docker pull --platform linux/amd64 python:3.11-slim
docker pull --platform linux/arm64 python:3.11-slim

# Pull by digest (maximum reproducibility)
docker pull nginx@sha256:32da30332506740a2f7c34d5dc70467b7f14ec67d6b2660a57db8e2e55c3ef0f

# Pull all tags of a repository
docker pull --all-tags nginx
```

---

## docker build — Building Images from a Dockerfile

```bash
# Basic build — tag as myapp:1.0, use current directory as context
docker build -t myapp:1.0 .

# Build from a specific Dockerfile
docker build -f Dockerfile.prod -t myapp:prod .

# Build from a different directory context
docker build -t myapp:1.0 /path/to/project/

# Build with build arguments
docker build \
  --build-arg APP_VERSION=1.0 \
  --build-arg BUILD_ENV=production \
  -t myapp:1.0 .

# Build without using cache (full rebuild)
docker build --no-cache -t myapp:1.0 .

# Build and print full output (useful for debugging)
docker build --progress=plain -t myapp:1.0 .

# Build for a specific platform
docker build --platform linux/amd64 -t myapp:1.0 .

# Build with multiple tags at once
docker build \
  -t myapp:1.4.2 \
  -t myapp:1.4 \
  -t myapp:latest \
  .
```

### Build Context

The `.` at the end of `docker build -t myapp:1.0 .` is the **build context** — the directory Docker sends to the daemon to execute the build. Every file in that directory is sent, which is why `.dockerignore` matters.

```bash
# What is sent to the daemon (before .dockerignore filtering)
ls -la .

# After .dockerignore is applied, only the necessary files are sent
# Creating a .dockerignore reduces build time and prevents secrets from leaking
cat .dockerignore
```

```
.git
.env
__pycache__
*.pyc
*.log
tests/
docs/
node_modules
.terraform
```

---

## docker tag — Tagging Images

Tagging creates a new name (alias) for an existing image. The original image is not modified.

```bash
# Tag with a new name and version
docker tag myapp:1.0 myapp:latest

# Tag for AWS ECR (add the registry URL)
docker tag myapp:1.0 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:1.0

# Tag with multiple versions (common in release workflows)
docker tag myapp:1.4.2 myapp:1.4
docker tag myapp:1.4.2 myapp:1
docker tag myapp:1.4.2 myapp:latest

# Tag using Git commit hash (for traceability in CI/CD)
GIT_SHA=$(git rev-parse --short HEAD)
docker tag myapp:build myapp:${GIT_SHA}
docker tag myapp:build 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:${GIT_SHA}
```

### Tagging Strategy for Production

A solid tagging strategy makes deployments traceable and rollbacks easy:

```bash
# In your CI/CD pipeline
GIT_SHA=$(git rev-parse HEAD)
GIT_SHORT=$(git rev-parse --short HEAD)
VERSION=$(cat VERSION)  # if you have a VERSION file

# Tag with multiple identifiers
docker build -t myapp:build .

docker tag myapp:build myapp:${VERSION}          # semantic version: 1.4.2
docker tag myapp:build myapp:${GIT_SHORT}        # short sha: a1b2c3d
docker tag myapp:build myapp:latest              # latest tag

# Push all tags to ECR
ECR=123456789012.dkr.ecr.us-east-1.amazonaws.com

docker tag myapp:build ${ECR}/myapp:${VERSION}
docker tag myapp:build ${ECR}/myapp:${GIT_SHA}
docker tag myapp:build ${ECR}/myapp:latest

docker push ${ECR}/myapp:${VERSION}
docker push ${ECR}/myapp:${GIT_SHA}
docker push ${ECR}/myapp:latest
```

---

## docker push — Uploading Images to a Registry

```bash
# Push to Docker Hub (must be logged in)
docker push yourusername/myapp:1.0

# Push to AWS ECR (must authenticate first)
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789012.dkr.ecr.us-east-1.amazonaws.com

docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:1.0

# Push all tags of a repository
docker push --all-tags 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp
```

Docker pushes only the layers that are not already in the registry. If you built on a cached base layer, only your new layer is uploaded.

---

## docker inspect — Deep Image Inspection

```bash
# Full JSON details of an image
docker inspect nginx:1.25.3

# Get the default command
docker inspect --format '{{json .Config.Cmd}}' nginx:1.25.3
# ["nginx","-g","daemon off;"]

# Get the entrypoint
docker inspect --format '{{json .Config.Entrypoint}}' nginx:1.25.3

# Get environment variables set in the image
docker inspect --format '{{range .Config.Env}}{{println .}}{{end}}' nginx:1.25.3

# Get exposed ports
docker inspect --format '{{json .Config.ExposedPorts}}' nginx:1.25.3
# {"80/tcp":{}}

# Get the working directory
docker inspect --format '{{.Config.WorkingDir}}' myapp:1.0

# Get all image labels
docker inspect --format '{{json .Config.Labels}}' myapp:1.0

# Get all layer IDs
docker inspect --format '{{json .RootFS.Layers}}' myapp:1.0

# Get the content digest
docker inspect --format '{{index .RepoDigests 0}}' nginx:1.25.3

# Get the image creation time
docker inspect --format '{{.Created}}' nginx:1.25.3
```

---

## docker history — Layer Breakdown

`docker history` shows how an image was built — what commands created each layer and how much space each layer uses.

```bash
docker history myapp:1.0

# Output:
IMAGE         CREATED        CREATED BY                                      SIZE
a1b2c3d4      2 hours ago    CMD ["python" "app.py"]                         0B
<missing>     2 hours ago    COPY . . # buildkit                             24.5kB
<missing>     2 hours ago    RUN pip install -r requirements.txt             89.3MB
<missing>     2 hours ago    COPY requirements.txt .                         342B
<missing>     2 hours ago    WORKDIR /app                                    0B
<missing>     8 days ago     CMD ["python3"]                                 0B
<missing>     8 days ago     FROM python:3.11-slim                           74.8MB
```

Look for layers with large SIZE values. Those are your optimization targets.

```bash
# Show full command (not truncated)
docker history --no-trunc myapp:1.0

# Show only sizes
docker history --format "{{.Size}}\t{{.CreatedBy}}" myapp:1.0 | sort -rh
```

---

## docker rmi — Removing Images

```bash
# Remove a specific image
docker rmi nginx:1.25.3

# Remove multiple images
docker rmi nginx:1.25.3 nginx:1.25.2 nginx:1.25.1

# Force remove (even if containers reference it)
docker rmi -f nginx:1.25.3

# Remove by image ID
docker rmi a1b2c3d4ef56

# Remove all images
docker rmi $(docker images -q)

# Remove all images for a repository
docker rmi $(docker images nginx -q)

# Remove dangling images (none:none)
docker image prune

# Remove all unused images
docker image prune -a

# Remove unused images older than 48 hours
docker image prune -a --filter "until=48h"
```

**Warning:** You cannot remove an image if a container (even stopped) references it. Remove the containers first or use `-f`.

---

## docker save and docker load — Exporting and Importing Images

Sometimes you need to transfer an image without using a registry — for air-gapped environments, for shipping images on a USB drive, or for backup.

```bash
# Save an image to a tar file
docker save myapp:1.0 -o myapp-1.0.tar

# Save multiple images to one file
docker save myapp:1.0 myapp:2.0 nginx:1.25.3 -o images.tar

# Compress the archive
docker save myapp:1.0 | gzip > myapp-1.0.tar.gz

# Load an image from a tar file
docker load -i myapp-1.0.tar

# Load a compressed image
docker load < myapp-1.0.tar.gz

# Verify the image was loaded
docker images myapp
```

---

## docker search — Searching Docker Hub

```bash
# Search Docker Hub for images
docker search nginx

# Filter to official images only
docker search --filter is-official=true python

# Filter by minimum stars
docker search --filter stars=100 redis

# Limit results
docker search --limit 5 postgres

# Output:
NAME                   DESCRIPTION                       STARS   OFFICIAL
postgres               The PostgreSQL object-relational   13000   [OK]
bitnami/postgresql     Bitnami PostgreSQL Docker Image     500
```

For more details and to browse images with full documentation, use the Docker Hub website at `https://hub.docker.com`.

---

## A Complete Image Workflow — From Build to Production

Putting it all together — a complete workflow for building, testing, tagging, and pushing an image to AWS ECR:

```bash
#!/bin/bash
set -e

# Configuration
ECR_REGISTRY="123456789012.dkr.ecr.us-east-1.amazonaws.com"
REPO_NAME="myapp"
GIT_SHA=$(git rev-parse --short HEAD)
VERSION="1.4.2"

# Step 1: Build the image
echo "Building image..."
docker build \
  --no-cache \
  --build-arg BUILD_DATE=$(date -u +'%Y-%m-%dT%H:%M:%SZ') \
  --build-arg GIT_COMMIT=${GIT_SHA} \
  -t ${REPO_NAME}:build .

# Step 2: Run tests
echo "Running tests..."
docker run --rm \
  -e TEST_ENV=true \
  ${REPO_NAME}:build \
  python -m pytest tests/ -v

# Step 3: Tag the image
echo "Tagging image..."
docker tag ${REPO_NAME}:build ${ECR_REGISTRY}/${REPO_NAME}:${VERSION}
docker tag ${REPO_NAME}:build ${ECR_REGISTRY}/${REPO_NAME}:${GIT_SHA}
docker tag ${REPO_NAME}:build ${ECR_REGISTRY}/${REPO_NAME}:latest

# Step 4: Authenticate with ECR
echo "Authenticating with ECR..."
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin ${ECR_REGISTRY}

# Step 5: Push to ECR
echo "Pushing to ECR..."
docker push ${ECR_REGISTRY}/${REPO_NAME}:${VERSION}
docker push ${ECR_REGISTRY}/${REPO_NAME}:${GIT_SHA}
docker push ${ECR_REGISTRY}/${REPO_NAME}:latest

# Step 6: Clean up local build image
docker rmi ${REPO_NAME}:build

echo "Done! Image ${VERSION} (${GIT_SHA}) pushed to ECR."
```

---

## Summary

- `docker images` lists all local images — use `--filter` and `--format` to get exactly the information you need
- `docker build` creates images from a Dockerfile — always specify a tag (`-t`), use `--no-cache` for clean builds, and exclude unnecessary files with `.dockerignore`
- `docker tag` creates aliases for images — use a multi-version tagging strategy (semantic version + git SHA + latest) for traceable deployments
- `docker push` uploads only the changed layers — authenticate with your registry first
- `docker inspect` and `docker history` are your tools for understanding what is inside an image and how large each layer is
- `docker rmi` and `docker image prune` manage your local image cache — clean up regularly to avoid disk space issues
- `docker save` and `docker load` transfer images without a registry — useful for air-gapped environments

In the next module, we move into more advanced territory — Docker networking and how containers communicate with each other and with the outside world.
