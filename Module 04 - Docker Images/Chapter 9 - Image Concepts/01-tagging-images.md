# Tagging Images

## Introduction

Image tagging is how you label Docker images with meaningful names and version identifiers. A well-designed tagging strategy makes your deployment pipeline clear, traceable, and safe. A poorly designed one creates confusion, prevents reliable rollbacks, and makes debugging harder.

This file covers the complete picture of Docker image tagging — the mechanics, the patterns, and how to implement a production-grade tagging workflow in a real CI/CD pipeline.

---

## The Mechanics of Tags

A Docker tag is a label attached to an image in a registry. It is not permanently bound to a specific image — it is a pointer that can be updated to point to a new image at any time.

```
Registry: 123456789012.dkr.ecr.us-east-1.amazonaws.com
Repository: myapp
Tags:
  1.4.2   → sha256:abc123 (permanent — always this image)
  1.4     → sha256:abc123 (same image right now)
  latest  → sha256:abc123 (will move when next version ships)
```

When you push `myapp:1.5.0`, the `latest` tag moves:

```
  1.4.2   → sha256:abc123 (unchanged)
  1.5.0   → sha256:def456 (new)
  latest  → sha256:def456 (moved to new image)
```

A tag like `1.4.2` that is never overwritten becomes effectively immutable. This is the foundation of reliable deployments — deploy by a specific version tag, not by `latest`.

---

## The docker tag Command

`docker tag` creates a new tag that references the same image data. No image data is copied — just a new pointer.

```bash
# Basic syntax
docker tag SOURCE_IMAGE[:TAG] TARGET_IMAGE[:TAG]

# Create a new tag for a locally built image
docker tag myapp:build myapp:1.4.2

# Tag for pushing to ECR
docker tag myapp:1.4.2 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:1.4.2

# Copy a tag
docker tag myapp:1.4.2 myapp:latest

# Tag an image by ID
docker tag a1b2c3d4ef56 myapp:1.4.2
```

After tagging, both names refer to the same image data. Pushing either tag uploads the same layers.

---

## Standard Tagging Patterns

### Pattern 1: Semantic Version Tags

Semantic versioning is the most human-readable tagging strategy.

```bash
VERSION="1.4.2"
ECR="123456789012.dkr.ecr.us-east-1.amazonaws.com"

docker build -t myapp:build .

# Full semantic version
docker tag myapp:build ${ECR}/myapp:${VERSION}

# Minor version (points to latest patch of this minor)
docker tag myapp:build ${ECR}/myapp:1.4

# Major version (points to latest minor of this major)
docker tag myapp:build ${ECR}/myapp:1

# Latest (points to latest overall)
docker tag myapp:build ${ECR}/myapp:latest

docker push ${ECR}/myapp:${VERSION}
docker push ${ECR}/myapp:1.4
docker push ${ECR}/myapp:1
docker push ${ECR}/myapp:latest
```

Users who want stability pull the exact version. Users who want updates pull the minor or major version.

### Pattern 2: Git Commit SHA Tags

```bash
# Full SHA (40 characters) — maximum uniqueness
GIT_SHA=$(git rev-parse HEAD)

# Short SHA (7-8 characters) — readable and still unique enough
GIT_SHORT=$(git rev-parse --short HEAD)

docker tag myapp:build ${ECR}/myapp:${GIT_SHORT}
docker push ${ECR}/myapp:${GIT_SHORT}
```

### Pattern 3: Environment Tags

Tags that represent the deployment target:

```bash
# Tag by environment
docker tag myapp:build ${ECR}/myapp:staging
docker tag myapp:build ${ECR}/myapp:production

docker push ${ECR}/myapp:staging
```

**Important:** Environment tags should point to the same image as the version tag — never build a separate image per environment. Configuration differences between environments should come from environment variables, not from different images.

```bash
# Wrong — different images for different environments
docker build --build-arg ENV=staging -t myapp:staging .
docker build --build-arg ENV=production -t myapp:production .

# Right — same image, environment configured at runtime
docker build -t myapp:1.4.2 .
docker tag myapp:1.4.2 myapp:staging  # when promoting to staging
docker tag myapp:1.4.2 myapp:production  # when promoting to production
```

### Pattern 4: Branch and PR Tags

In development workflows, tagging by branch name is useful for deploying specific branches to review environments:

```bash
# Clean the branch name for use in a tag (replace / with -)
BRANCH=$(git rev-parse --abbrev-ref HEAD | sed 's/[^a-zA-Z0-9._-]/-/g')

docker tag myapp:build ${ECR}/myapp:branch-${BRANCH}
docker push ${ECR}/myapp:branch-${BRANCH}
```

This lets you deploy `feature/user-auth` as `myapp:branch-feature-user-auth` to a temporary review environment.

---

## Production Tagging Strategy

Here is a complete, production-grade tagging strategy that balances traceability, readability, and reliability:

### One Image, Many Tags

Every release produces one image with multiple tags:

```
myapp:1.4.2           ← exact version (never changes after push)
myapp:a1b2c3d         ← git commit SHA (never changes after push)
myapp:latest          ← moves to latest release
```

### Tag on Merge to main

When code is merged to the `main` branch, your CI/CD pipeline:

1. Builds the image once
2. Runs tests
3. If tests pass, tags with version, SHA, and latest
4. Pushes all tags to ECR

```yaml
# .github/workflows/release.yml
name: Build and Release

on:
  push:
    branches: [main]

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # needed for git log

      - name: Extract version
        id: version
        run: echo "VERSION=$(cat VERSION)" >> $GITHUB_OUTPUT

      - name: Get commit info
        id: commit
        run: |
          echo "SHA=$(git rev-parse HEAD)" >> $GITHUB_OUTPUT
          echo "SHORT=$(git rev-parse --short HEAD)" >> $GITHUB_OUTPUT

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1

      - name: Login to ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build image
        env:
          ECR: ${{ steps.login-ecr.outputs.registry }}
          VERSION: ${{ steps.version.outputs.VERSION }}
          SHA: ${{ steps.commit.outputs.SHORT }}
        run: |
          docker build \
            --build-arg APP_VERSION=$VERSION \
            --build-arg GIT_COMMIT=${{ steps.commit.outputs.SHA }} \
            --build-arg BUILD_DATE=$(date -u +'%Y-%m-%dT%H:%M:%SZ') \
            -t $ECR/myapp:$VERSION \
            -t $ECR/myapp:$SHA \
            -t $ECR/myapp:latest \
            .

      - name: Run tests
        env:
          ECR: ${{ steps.login-ecr.outputs.registry }}
          VERSION: ${{ steps.version.outputs.VERSION }}
        run: |
          docker run --rm \
            -e TEST_ENV=true \
            $ECR/myapp:$VERSION \
            python -m pytest tests/ -v

      - name: Push all tags
        env:
          ECR: ${{ steps.login-ecr.outputs.registry }}
          VERSION: ${{ steps.version.outputs.VERSION }}
          SHA: ${{ steps.commit.outputs.SHORT }}
        run: |
          docker push $ECR/myapp:$VERSION
          docker push $ECR/myapp:$SHA
          docker push $ECR/myapp:latest
```

---

## Tagging for Rollbacks

A good tagging strategy makes rollbacks reliable and fast.

```bash
# Current production is myapp:1.4.2 — it has a bug
# Roll back to the previous version

# Option 1: Redeploy by semantic version
aws ecs update-service \
  --cluster production \
  --service myapp \
  --force-new-deployment \
  --task-definition myapp-task:previous-revision

# Option 2: Update task definition to use old image version
aws ecs register-task-definition \
  --family myapp-task \
  --container-definitions "[{\"image\":\"${ECR}/myapp:1.4.1\"}]" \
  ...

# Option 3: If using Helm on EKS
helm rollback myapp 3  # roll back to revision 3
```

The key is that `myapp:1.4.1` still exists in ECR and is unchanged. You can redeploy it with confidence.

---

## Tagging Conventions for Multi-Arch Images

When you build images for multiple CPU architectures (AMD64 and ARM64), each platform variant needs its own image. Docker handles this through **manifests**.

```bash
# Build for both platforms and push as a multi-arch image
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:1.4.2 \
  --push \
  .
```

The single tag `myapp:1.4.2` now points to a manifest list that contains both the AMD64 and ARM64 images. When you pull `myapp:1.4.2` on an ARM64 machine (like AWS Graviton), Docker automatically pulls the ARM64 image. On an AMD64 machine, it pulls the AMD64 image.

You can inspect the manifest:

```bash
docker buildx imagetools inspect 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:1.4.2
```

---

## Listing and Managing Tags in ECR

```bash
# List all tags for a repository
aws ecr list-images --repository-name myapp --region us-east-1

# List tags with image details
aws ecr describe-images \
  --repository-name myapp \
  --region us-east-1 \
  --query 'sort_by(imageDetails,&imagePushedAt)[-5:].{Tag:imageTags[0],Size:imageSizeInBytes,Pushed:imagePushedAt}' \
  --output table

# Get the latest pushed image
aws ecr describe-images \
  --repository-name myapp \
  --region us-east-1 \
  --query 'sort_by(imageDetails,&imagePushedAt)[-1].imageTags' \
  --output text
```

---

## Summary

- A Docker tag is a mutable pointer to an image — pushing a new image with the same tag moves the pointer, it does not change the old image
- Never use `latest` for production deployments — it moves on every push and makes rollbacks unreliable
- Use semantic version tags (`1.4.2`) for human-readable, releasable versions
- Use Git commit SHA tags (`a1b2c3d`) for precise code traceability
- Apply both tag types to every production image — version for deployment decisions, SHA for debugging
- Environment tags (`staging`, `production`) are valid but should point to the same image as the version tag — never build separate images per environment
- For multi-architecture deployments (AMD64 and ARM64 for AWS Graviton), use `docker buildx` to build and push a manifest list under a single tag

In the next file, we cover image security scanning — how to detect vulnerabilities in your images before they reach production.
