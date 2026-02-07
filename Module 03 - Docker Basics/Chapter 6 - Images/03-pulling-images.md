# Pulling Images

## Introduction

Pulling images is one of the most frequent operations you do with Docker. Every time you `docker run` an image that is not in your local cache, Docker pulls it automatically. But there is much more to understand about pulling — from controlling exactly what you pull to managing your local image cache to authenticating with private registries.

This file covers everything about pulling images, managing them locally, and working with Docker Hub and AWS ECR in a production context.

---

## How Image Pulling Works

When you run `docker pull nginx`, here is what happens behind the scenes:

1. Docker reads the image name and determines the registry (`docker.io` by default)
2. Docker contacts the registry and requests the image manifest — a JSON document listing all the layers in the image
3. Docker checks which layers it already has locally (from previous pulls)
4. Docker downloads only the layers it does not have
5. Docker stores the layers in its local cache at the configured storage location

This is why pulling the same base image for the second time is almost instant — the layers are already cached.

```bash
# First pull — all layers downloaded
docker pull python:3.11-slim

# Output:
3.11-slim: Pulling from library/python
a2abf6c4d29d: Pull complete      ← layer downloaded
a9edb18cadd1: Pull complete      ← layer downloaded
589b7251471a: Pull complete      ← layer downloaded
186b1aaa4aa6: Pull complete      ← layer downloaded
b4df32aa5a72: Pull complete      ← layer downloaded
Digest: sha256:abc123...
Status: Downloaded newer image for python:3.11-slim

# Second pull of the same image (no changes) — instant
docker pull python:3.11-slim

# Output:
3.11-slim: Pulling from library/python
Status: Image is up to date for python:3.11-slim
```

Now pull a different image that shares the base layer:

```bash
# Pull python:3.11 — shares some layers with 3.11-slim
docker pull python:3.11

# Output:
a2abf6c4d29d: Already exists    ← shared layer, not downloaded again
a9edb18cadd1: Already exists    ← shared layer
589b7251471a: Pull complete     ← new layer (unique to full python:3.11)
...
```

Docker is smart enough to share layers between different images. You save bandwidth and disk space automatically.

---

## Pulling Specific Versions

Always specify the exact version you want when working in production or CI/CD:

```bash
# Pull latest (ambiguous — avoid in production)
docker pull nginx

# Pull a specific version
docker pull nginx:1.25.3

# Pull a specific minor version (gets latest patch)
docker pull nginx:1.25

# Pull a specific variant
docker pull nginx:1.25.3-alpine

# Pull from a non-default registry
docker pull public.ecr.aws/nginx/nginx:1.25.3
```

### Understanding Tag Mutability

Tags in Docker registries can be updated to point to a newer image:

```bash
# Today's pull of nginx:latest might be version 1.25.3
docker pull nginx:latest  # gets image ABC123

# Three months later, nginx:latest might be 1.27.0
docker pull nginx:latest  # gets image DEF456 — different image, same tag
```

This is why pinning to specific versions is essential. If you pull `python:3.11` in your Dockerfile today and the same Dockerfile is built in 6 months, you might get a different Python patch version with different behavior.

### Pulling by Digest (Absolute Version Pinning)

For the highest level of reproducibility and security, pull images by their digest:

```bash
# Get the digest of an image
docker pull nginx:1.25.3
docker inspect --format '{{index .RepoDigests 0}}' nginx:1.25.3
# nginx@sha256:32da30332506740a2f7c34d5dc70467b7f14ec67d6b2660a57db8e2e55c3ef0f

# Pull by digest — guaranteed to be exactly this image
docker pull nginx@sha256:32da30332506740a2f7c34d5dc70467b7f14ec67d6b2660a57db8e2e55c3ef0f

# Use in Dockerfile for maximum reproducibility
FROM nginx@sha256:32da30332506740a2f7c34d5dc70467b7f14ec67d6b2660a57db8e2e55c3ef0f
```

This is the most secure approach — the digest is based on the image content, so it cannot be modified without changing the digest.

---

## Pulling for Specific Platforms

Modern Docker supports multi-platform images. The same tag can contain images for different CPU architectures.

```bash
# Pull for current platform (default)
docker pull python:3.11-slim

# Pull specifically for AMD64 (Intel/AMD)
docker pull --platform linux/amd64 python:3.11-slim

# Pull specifically for ARM64 (Apple Silicon, AWS Graviton)
docker pull --platform linux/arm64 python:3.11-slim

# Check what platform the local image is for
docker inspect --format '{{.Architecture}}' python:3.11-slim
```

On Apple Silicon Macs, Docker Desktop automatically selects the ARM64 image when available. For deployment to AWS, you need to know whether your target servers are AMD64 (regular EC2) or ARM64 (Graviton instances).

---

## Managing Your Local Image Cache

Every image you pull or build is stored locally. Over time, this cache grows and needs management.

### Listing Local Images

```bash
# List all local images
docker images

# Output:
REPOSITORY     TAG         IMAGE ID       CREATED       SIZE
myapp          1.0         a1b2c3d4       2 hours ago   245MB
nginx          1.25.3      e5f6a7b8       5 days ago    187MB
python         3.11-slim   c9d0e1f2       8 days ago    128MB
postgres       16          b3c4d5e6       2 weeks ago   425MB

# List images for a specific repository
docker images nginx

# List all images including intermediary
docker images -a

# Show only image IDs
docker images -q

# Filter by before or since a specific image
docker images --filter before=nginx:1.25.3
docker images --filter since=nginx:1.25
```

### Checking Image Disk Usage

```bash
# Breakdown of Docker disk usage
docker system df

# Output:
TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
Images          8         5         2.1GB     890MB (42%)
Containers      3         2         45MB      12MB (26%)
Local Volumes   4         2         890MB     400MB (44%)
Build Cache     15        0         450MB     450MB

# Detailed breakdown per image
docker system df -v
```

### Removing Local Images

```bash
# Remove a specific image
docker rmi nginx:1.25.3

# Force remove (even if a container references it)
docker rmi -f nginx:1.25.3

# Remove multiple images
docker rmi nginx:1.25 nginx:1.24 nginx:1.23

# Remove all images by ID
docker rmi $(docker images -q)

# Remove dangling images (none:none)
docker image prune

# Remove all unused images (not referenced by any container)
docker image prune -a

# Remove images older than 48 hours
docker image prune -a --filter "until=48h"
```

### When to Remove Images

- Remove old versions of your application image after deploying new ones
- Remove images from completed one-off builds
- Clean up before storage runs low on a CI runner
- Remove dangling images after repeated builds with the same tag

---

## Authenticating with Private Registries

Public images on Docker Hub can be pulled without authentication. Private registries require login.

### Docker Hub Authentication

```bash
# Interactive login
docker login
# Enter username and password when prompted

# Non-interactive (for scripts and CI/CD)
echo "$DOCKER_HUB_PASSWORD" | docker login \
  --username "$DOCKER_HUB_USERNAME" \
  --password-stdin

# Log out
docker logout
```

Credentials are stored in `~/.docker/config.json`. In CI/CD, use secrets management to provide credentials without hardcoding them.

### AWS ECR Authentication

ECR uses temporary tokens that expire after 12 hours. You must re-authenticate regularly.

```bash
# Standard ECR login (uses AWS CLI credentials)
aws ecr get-login-password --region us-east-1 | \
  docker login \
  --username AWS \
  --password-stdin \
  123456789012.dkr.ecr.us-east-1.amazonaws.com

# Pull from ECR
docker pull 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:1.0
```

### ECR Authentication in CI/CD (GitHub Actions)

```yaml
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    aws-region: us-east-1

- name: Login to Amazon ECR
  id: login-ecr
  uses: aws-actions/amazon-ecr-login@v2

- name: Pull application image
  run: |
    docker pull ${{ steps.login-ecr.outputs.registry }}/myapp:${{ github.sha }}
```

### ECR Authentication in Production (IAM Role — Best Practice)

On AWS infrastructure (EC2, ECS, EKS), the best practice is to use IAM roles instead of static credentials. When your EC2 instance or ECS task has the right IAM role, it can pull from ECR automatically without any credentials in your code.

Attach this IAM policy to your ECS task role:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "ecr:GetAuthorizationToken"
      ],
      "Resource": "*"
    }
  ]
}
```

With this role, ECS pulls your images from ECR automatically. No credentials to manage, rotate, or accidentally expose.

---

## Caching Images in CI/CD

In CI/CD pipelines, pulling base images on every build wastes time. Use caching strategies to speed this up.

### GitHub Actions Cache

```yaml
- name: Set up Docker Buildx
  uses: docker/setup-buildx-action@v3

- name: Build and push
  uses: docker/build-push-action@v5
  with:
    context: .
    push: true
    tags: 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:${{ github.sha }}
    cache-from: type=gha          # Pull build cache from GitHub Actions cache
    cache-to: type=gha,mode=max   # Push build cache to GitHub Actions cache
```

This caches all Docker build layers in GitHub's cache. Subsequent builds reuse cached layers and only rebuild what changed.

---

## Checking for Image Updates

To check if a newer version of an image is available:

```bash
# Pull and let Docker tell you if it is up to date
docker pull nginx:1.25.3

# If there is a newer image with the same tag:
# Status: Downloaded newer image for nginx:1.25.3

# If you already have the latest:
# Status: Image is up to date for nginx:1.25.3

# Check the image digest to detect changes
docker inspect --format '{{index .RepoDigests 0}}' nginx:1.25.3
```

In production, you should have a process that automatically checks for updated base images and triggers a rebuild when security patches are released. AWS ECR's image scanning can alert you when vulnerabilities are found in your images.

---

## Summary

- Docker pulls only the layers that are not already cached locally — pulling an image with shared base layers is fast
- Always specify exact version tags when pulling for production or CI/CD — never rely on `latest`
- Pull by digest for maximum reproducibility and security — the digest is content-addressed and cannot change
- Use `docker system df` to monitor image disk usage and `docker image prune -a` to clean up unused images
- For Docker Hub, authenticate with `docker login` using secrets — never hardcode credentials
- For AWS ECR, use `aws ecr get-login-password` to get a temporary token — in production use IAM roles instead of static credentials
- Cache Docker layers in CI/CD pipelines (GitHub Actions cache, ECR pull-through cache) to dramatically speed up build times

In the next file, we cover the full set of image management commands — building, tagging, pushing, inspecting, and removing images efficiently.
