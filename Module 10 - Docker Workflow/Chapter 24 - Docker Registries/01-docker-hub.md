# Docker Hub

## Introduction

Docker Hub is the world's largest public container registry. It is the default registry Docker uses — when you run `docker pull nginx`, Docker fetches the image from Docker Hub's `library/nginx` repository without you specifying a registry URL.

Understanding Docker Hub is essential even if your organization uses a private registry. Most base images you build on (Python, Node, Alpine, Ubuntu, Postgres, Redis) live on Docker Hub. Understanding pull limits, versioning, and best practices directly impacts your CI/CD pipeline reliability.

---

## How Docker Hub Works

Docker Hub organizes images into repositories:

```
docker.io / NAMESPACE / REPOSITORY : TAG
   │            │           │          │
 registry     owner      image      version

Examples:
  docker.io/library/nginx:1.25.3       # official image (library namespace)
  docker.io/mycompany/myapp:1.4.2      # organization image
  docker.io/username/myapp:latest      # personal image

# docker.io is omitted in most commands — it is the default
docker pull nginx:1.25.3              # same as docker.io/library/nginx:1.25.3
docker pull mycompany/myapp:1.4.2
```

---

## Account Types and Repository Visibility

| Account Type | Free Plan | Paid Plan |
|---|---|---|
| Personal free | 1 private repo | Up to 50,000 private repos |
| Organization | 1 private repo | Up to 50,000 private repos |
| Private repos | Very limited | Based on plan |

For production workloads at companies, Docker Hub's private repositories and pull limits make it less attractive than ECR or a private registry. Most enterprise teams use Docker Hub only for pulling public base images.

---

## Pull Rate Limits

Docker Hub enforces pull rate limits for unauthenticated users and free accounts:

| Authentication | Pull Limit |
|---|---|
| Anonymous (no login) | 100 pulls per 6 hours per IP |
| Authenticated free account | 200 pulls per 6 hours per account |
| Docker Hub paid subscription | Unlimited |

In CI/CD environments with many runners sharing an IP, anonymous pulls can hit the 100-pull limit within hours. This breaks builds with:

```
Error response from daemon: toomanyrequests: You have reached your pull rate limit.
You may increase the limit by authenticating and upgrading: https://www.docker.com/increase-rate-limit
```

### Solution: Authenticate in CI

```bash
# Log in to Docker Hub in CI to get 200 pulls/6h instead of 100
echo "${DOCKERHUB_TOKEN}" | docker login --username "${DOCKERHUB_USERNAME}" --password-stdin
```

```yaml
# GitHub Actions
- name: Login to Docker Hub
  uses: docker/login-action@v3
  with:
    username: ${{ secrets.DOCKERHUB_USERNAME }}
    password: ${{ secrets.DOCKERHUB_TOKEN }}
```

Always use a Docker Hub **Access Token** (not your password) for CI authentication. Create one at: https://hub.docker.com/settings/security

### Solution: Mirror Official Images to ECR

The production-grade approach: mirror all base images you use to your own ECR registry. Your CI/CD and ECS tasks pull from ECR, never directly from Docker Hub.

```bash
# One-time: pull and push to ECR
docker pull python:3.11-slim@sha256:a1b2c3d4...
docker tag python:3.11-slim@sha256:a1b2c3d4... \
  123456789012.dkr.ecr.us-east-1.amazonaws.com/base-images/python:3.11-slim

aws ecr get-login-password | docker login --username AWS --password-stdin \
  123456789012.dkr.ecr.us-east-1.amazonaws.com

docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/base-images/python:3.11-slim
```

```dockerfile
# Dockerfile using the ECR mirror — no Docker Hub dependency
FROM 123456789012.dkr.ecr.us-east-1.amazonaws.com/base-images/python:3.11-slim
```

This also protects you from the Docker Hub outage scenario — your builds continue even if Docker Hub is down.

---

## Pushing to Docker Hub

```bash
# Login
docker login

# Tag with your namespace
docker tag myapp:1.4.2 myusername/myapp:1.4.2
docker tag myapp:1.4.2 myusername/myapp:latest

# Push
docker push myusername/myapp:1.4.2
docker push myusername/myapp:latest
```

For organizations:

```bash
# Tag with the organization name
docker tag myapp:1.4.2 mycompany/myapp:1.4.2
docker push mycompany/myapp:1.4.2
```

---

## Docker Hub Official Images — What They Are

Images like `nginx`, `postgres`, `redis`, `python`, and `node` are Docker Hub "Official Images." They are:

- Maintained by trusted parties (the software vendors or the Docker team)
- Regularly scanned for vulnerabilities
- Published under the `library` namespace (can be pulled without a namespace prefix)
- Documented with clear tagging conventions

### Understanding Official Image Tag Conventions

```
nginx:latest          # latest stable (changes over time — not reproducible)
nginx:1.25            # stable minor version (receives patch updates)
nginx:1.25.3          # exact patch version (reproducible until the tag is updated)
nginx:1.25.3-alpine   # alpine variant (minimal, smaller, musl libc)
nginx:1.25.3-slim     # slim variant (reduced packages)
nginx:mainline        # upstream mainline branch (newer, may be less stable)
```

Best practice: always pin to at least a minor version (`nginx:1.25`) and prefer an exact patch version (`nginx:1.25.3`) for production Dockerfiles.

---

## Automated Builds (Docker Hub Build Integration)

Docker Hub can build images automatically from a GitHub repository when you push commits. However, this feature has significant limitations for production use:

- Slow builds (shared infrastructure)
- Limited build configuration
- No custom build secrets support
- Docker Hub build agents can have outdated Docker versions

**Recommendation:** Use GitHub Actions, Jenkins, or AWS CodeBuild for building images. Use Docker Hub only as a registry (storage), not as a build platform.

---

## Docker Hub Webhooks

Trigger downstream systems when a new image is pushed:

```
Docker Hub Repository → Settings → Webhooks → Add Webhook
```

When an image is pushed, Docker Hub sends a POST request to your webhook URL with the image metadata:

```json
{
  "push_data": {
    "pushed_at": 1736942621,
    "tag": "1.4.2"
  },
  "repository": {
    "name": "myapp",
    "namespace": "mycompany",
    "repo_name": "mycompany/myapp"
  }
}
```

This can trigger a deployment pipeline, update a dashboard, or send a notification.

---

## Docker Hub Image Lifecycle

For public repositories, images remain indefinitely. For free-tier Docker Hub accounts, images in inactive repositories (not pushed or pulled for 6+ months) may be subject to deletion under Docker Hub's retention policies.

For paid accounts, images are retained indefinitely.

**Best practice for production:** Use ECR instead of Docker Hub for your application images. Docker Hub is suitable for public open-source projects or as a secondary mirror, but should not be the primary registry for enterprise workloads.

---

## Summary

- Docker Hub is the default registry — `docker pull nginx` pulls from `docker.io/library/nginx` without needing to specify a registry URL
- Pull rate limits (100/6h anonymous, 200/6h authenticated) can break CI/CD pipelines — always authenticate Docker Hub in CI using an Access Token
- Mirror frequently used base images from Docker Hub to your own ECR repository — eliminates Docker Hub rate limits, network dependency, and potential Docker Hub outages in your builds
- For pushing images, always use namespaced tags (`mycompany/myapp:1.4.2`) — the namespace must match your Docker Hub username or organization
- Official images (`nginx`, `python`, `postgres`) follow a consistent tagging convention — use exact patch versions (`python:3.11.8-slim`) for reproducible builds, minor version (`python:3.11-slim`) for automatic security patch updates
- For production workloads, use AWS ECR instead of Docker Hub for private images — better integration with IAM, no rate limits, and natively available in the AWS network

In the next file, we cover AWS ECR — the recommended container registry for AWS workloads.
