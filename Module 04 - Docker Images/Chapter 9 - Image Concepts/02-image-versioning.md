# Image Versioning

## Introduction

Image versioning is how you track, manage, and communicate changes to your Docker images over time. Done well, versioning gives you complete traceability — you can always answer the questions: what is running in production right now, what changed between versions, and how do I roll back?

Done poorly, you lose visibility into what is deployed, you cannot roll back reliably, and debugging production issues becomes significantly harder.

---

## Why Versioning Matters in DevOps

In a DevOps workflow, Docker images flow through a pipeline:

```
Code commit
    ↓
Build image
    ↓
Run automated tests
    ↓
Push to registry (ECR)
    ↓
Deploy to staging
    ↓
Deploy to production on AWS ECS/EKS
```

At every step, you need to know exactly which image you are working with. If images are not versioned properly:

- You cannot tell which code version is running in production
- You cannot selectively roll back to a previous good version
- You cannot reproduce a bug that exists in production but not locally
- You cannot audit what changed between deployments

Proper versioning makes all of these possible.

---

## The Problems with `latest`

The `latest` tag is the most common versioning mistake in Docker. Beginners use it because it is the default.

The problems:

**It is mutable** — `latest` can point to a different image after every push. If your deployment always pulls `latest`, you do not know which code version is running.

**It hides what changed** — If something broke in production after the last deployment, you cannot tell exactly what changed because you do not know which image was deployed before or after.

**It makes rollbacks impossible** — If `latest` always points to the newest image, you cannot redeploy "the previous version" — you do not have a stable reference to it.

**It breaks reproducibility** — A Dockerfile that says `FROM python:3.11-slim` might build a different image today versus 3 months from now, because `latest` was updated.

```bash
# Today's deployment
docker pull 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:latest
# Image: sha256:abc123 — version 1.4.2

# Something breaks. Roll back.
docker pull 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:latest
# Image: sha256:abc123 — STILL the broken version. You cannot go back.
```

Never use `latest` as your primary production deployment tag.

---

## Versioning Strategies

### Strategy 1: Semantic Versioning

Semantic versioning (SemVer) uses a three-number format: `MAJOR.MINOR.PATCH`

- **MAJOR** — breaking changes (API changes, configuration changes that require updates)
- **MINOR** — new features, backward compatible
- **PATCH** — bug fixes, security patches, backward compatible

```bash
# Tag with full semantic version
docker tag myapp:build 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:1.4.2

# Tag parent versions (optional but useful)
docker tag myapp:build 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:1.4
docker tag myapp:build 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:1

# Update latest (optional, not for production deployments)
docker tag myapp:build 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:latest
```

With semantic versioning:
- Production always deploys a specific patch version: `myapp:1.4.2`
- You know exactly what changed between `myapp:1.4.1` and `myapp:1.4.2`
- Rollback means redeploying `myapp:1.4.1`

```bash
# Deploy
aws ecs update-service --cluster prod --service myapp \
  --task-definition myapp:1.4.2 --force-new-deployment

# Rollback — simple and reliable
aws ecs update-service --cluster prod --service myapp \
  --task-definition myapp:1.4.1 --force-new-deployment
```

### Strategy 2: Git Commit SHA

Every Git commit produces a unique SHA hash. Tagging images with the commit SHA creates a direct link between your code and your image.

```bash
# In CI/CD pipeline
GIT_SHA=$(git rev-parse HEAD)           # full SHA: a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0
GIT_SHORT=$(git rev-parse --short HEAD) # short SHA: a1b2c3d

docker build -t myapp:${GIT_SHORT} .
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:${GIT_SHORT}
```

With commit SHA tags:
- You can look at any running container's tag and immediately know the exact commit in Git
- You can find the exact diff between two deployed versions with `git diff a1b2c3d def456a`
- You can trace a production bug back to the commit that introduced it

```bash
# What is running in production right now?
aws ecs describe-tasks --cluster prod --tasks $(aws ecs list-tasks --cluster prod --service-name myapp --query 'taskArns[0]' --output text) \
  --query 'tasks[0].containers[0].image' --output text
# Output: 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:a1b2c3d

# What code is in that version?
git log a1b2c3d -1 --oneline
# a1b2c3d feat: add user authentication endpoint
```

### Strategy 3: Combined (Recommended)

The best production strategy combines semantic versioning with Git SHA. Tag images with both:

```bash
GIT_SHA=$(git rev-parse --short HEAD)
VERSION=$(cat VERSION)  # or from package.json, pyproject.toml, etc.

ECR=123456789012.dkr.ecr.us-east-1.amazonaws.com

# Build once
docker build -t myapp:build .

# Tag with both semantic version and commit SHA
docker tag myapp:build ${ECR}/myapp:${VERSION}
docker tag myapp:build ${ECR}/myapp:${GIT_SHA}
docker tag myapp:build ${ECR}/myapp:latest

# Push all tags (only one image is stored — just multiple references)
docker push ${ECR}/myapp:${VERSION}
docker push ${ECR}/myapp:${GIT_SHA}
docker push ${ECR}/myapp:latest
```

Deploy with semantic version (human-friendly, easy to communicate):

```bash
aws ecs update-service --cluster prod --service myapp \
  --task-definition "arn:aws:ecs:us-east-1:123456789:task-definition/myapp:${VERSION}"
```

Trace back to exact code with SHA when debugging:

```bash
# Find the running image version
docker inspect --format '{{index .Config.Labels "git-commit"}}' container_id
# or check the image tag
```

---

## Versioning Dockerfile Base Images

Your Dockerfile's `FROM` instruction also needs version management. Floating base image tags cause non-reproducible builds.

```dockerfile
# Non-reproducible — python:3.11-slim content changes over time
FROM python:3.11-slim

# Reproducible — exact patch version
FROM python:3.11.7-slim

# Maximally reproducible — pinned by content digest
FROM python@sha256:7d9f9f8a0e5e2c7f3b1a2d4e6f8a0b2d4e6f8a0b2d4e6f8a0b2d4e6f8a0b2d4e
```

Pin to at least the patch version. For production-critical systems, pin by digest.

### Updating Base Image Versions

When you pin base image versions, you need a process to update them:

1. **Renovate Bot** — automatically creates PRs to update Dockerfile `FROM` versions when new versions are released
2. **AWS ECR scanning** — alerts when new CVEs are found; triggered by new vulnerability database, not new image pushes
3. **Scheduled rebuilds** — rebuild images on a schedule (weekly/monthly) to pick up security patches

---

## Recording Version Information in the Image

Embed version information directly in the image so you can always inspect a running container to know what it contains:

```dockerfile
ARG APP_VERSION=unknown
ARG GIT_COMMIT=unknown
ARG BUILD_DATE=unknown
ARG GIT_BRANCH=unknown

LABEL app.version="${APP_VERSION}" \
      app.git-commit="${GIT_COMMIT}" \
      app.build-date="${BUILD_DATE}" \
      app.git-branch="${GIT_BRANCH}"

ENV APP_VERSION=${APP_VERSION}
```

Build with arguments:

```bash
docker build \
  --build-arg APP_VERSION=$(cat VERSION) \
  --build-arg GIT_COMMIT=$(git rev-parse HEAD) \
  --build-arg BUILD_DATE=$(date -u +'%Y-%m-%dT%H:%M:%SZ') \
  --build-arg GIT_BRANCH=$(git rev-parse --abbrev-ref HEAD) \
  -t myapp:$(cat VERSION) .
```

Now you can inspect any image or running container:

```bash
# Inspect image labels
docker inspect --format '{{json .Config.Labels}}' myapp:1.4.2 | python3 -m json.tool

# Output:
{
  "app.version": "1.4.2",
  "app.git-commit": "a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0",
  "app.build-date": "2024-01-15T09:23:41Z",
  "app.git-branch": "main"
}

# Or from inside a running container
docker exec mycontainer printenv APP_VERSION
```

---

## Managing Version Files

Keep the authoritative version in a version file tracked by Git:

```
project/
├── VERSION          # contains "1.4.2" — the single source of truth
├── Dockerfile
├── src/
└── ...
```

```bash
# Read version from file
VERSION=$(cat VERSION)

# Bump version (patch)
echo "1.4.3" > VERSION
git commit -am "chore: bump version to 1.4.3"
git tag v1.4.3
```

For Python projects, the version can live in `pyproject.toml` or `setup.py`:

```toml
# pyproject.toml
[tool.poetry]
version = "1.4.2"
```

```bash
VERSION=$(python -c "import tomllib; print(tomllib.load(open('pyproject.toml','rb'))['tool']['poetry']['version'])")
```

For Node.js projects:

```bash
VERSION=$(node -e "console.log(require('./package.json').version)")
```

---

## Image Retention and Lifecycle

As you version and push images over time, old versions accumulate in your registry. Set up lifecycle policies to control this.

```bash
# AWS ECR lifecycle policy — keep only last 20 images per repository
aws ecr put-lifecycle-policy \
  --repository-name myapp \
  --lifecycle-policy-text '{
    "rules": [
      {
        "rulePriority": 1,
        "description": "Keep last 20 images",
        "selection": {
          "tagStatus": "any",
          "countType": "imageCountMoreThan",
          "countNumber": 20
        },
        "action": {
          "type": "expire"
        }
      }
    ]
  }' \
  --region us-east-1
```

Balance retention with rollback capability. Keep at least the last 5-10 versions to enable rollbacks. Keep more if your releases are infrequent.

---

## Summary

- Never use `latest` as your primary deployment tag — it is mutable and makes rollbacks unreliable
- Semantic versioning (`MAJOR.MINOR.PATCH`) communicates the nature of changes and makes rollbacks simple
- Git commit SHA tags create a direct link between your running image and the exact code commit in Git
- Combine both strategies — tag with semantic version for deployments and Git SHA for traceability
- Pin base image versions in your Dockerfile — at minimum to the patch version, ideally to the digest
- Embed version information in image labels and environment variables so you can always inspect a running container to know its version
- Set ECR lifecycle policies to automatically remove old images and control storage costs

In the next file, we cover image tagging strategies in depth — the naming conventions, workflows, and patterns used by real DevOps teams.
