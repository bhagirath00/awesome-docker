# Docker in CI/CD Overview

## Introduction

Continuous Integration and Continuous Delivery (CI/CD) pipelines are where Docker's benefits are most fully realized. Every commit is automatically built into a tested, scanned, and versioned image. Every deployment is a predictable, reproducible container swap. The environment that ran tests is identical to the environment that runs in production.

This file covers the architecture of a Docker-based CI/CD pipeline — what the stages are, what guarantees each stage provides, and how Docker images flow from a developer's commit to a production deployment.

---

## The Core Principle: Immutable Artifacts

The most important principle in Docker-based CI/CD is that the container image is the deployment artifact.

In traditional pipelines, the artifact is code (a tarball, a JAR, a zip) that is then assembled into a running environment on the target server. The environment on the target server may differ from the environment where tests ran. "It works on my machine" becomes "it works in staging."

In a Docker-based pipeline:
1. **One image is built once** from the commit
2. **The same image** runs in all environments: test, staging, production
3. The image is **immutable** — its SHA256 digest never changes
4. The image is **versioned** by a git commit SHA or semantic version tag

This means if the image passes tests, you have high confidence the exact same image will work in production — because it is the same image.

---

## The Standard Docker CI/CD Pipeline

```
Developer Commits
       │
       ▼
┌──────────────────┐
│ 1. Build         │  docker build → image:commit-sha
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ 2. Lint          │  hadolint Dockerfile
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ 3. Test          │  docker run → pytest / jest / go test
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ 4. Scan          │  trivy image → exit 1 on CRITICAL/HIGH
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ 5. Push          │  docker push → ECR/Docker Hub
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ 6. Deploy        │  ECS service update / docker compose pull + up
└──────────────────┘
```

Each stage is a gate. If a stage fails, the pipeline stops. Code with failing tests, known vulnerabilities, or Dockerfile linting errors never reaches the registry.

---

## Stage 1: Build

The build stage creates the image from the Dockerfile using the current commit's source code.

```bash
# Tag the image with the git commit SHA for traceability
GIT_SHA=$(git rev-parse --short HEAD)
IMAGE_TAG="${ECR_REGISTRY}/myapp:${GIT_SHA}"

docker build \
  --build-arg GIT_COMMIT=${GIT_SHA} \
  --build-arg BUILD_DATE=$(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --build-arg APP_VERSION=${SEMVER_TAG:-dev} \
  --cache-from ${ECR_REGISTRY}/myapp:latest \
  -t ${IMAGE_TAG} \
  -t ${ECR_REGISTRY}/myapp:latest \
  .
```

Key practices:
- Tag with git SHA for immutable, traceable images
- Use `--cache-from` to pull previous layers for fast rebuilds
- Pass build metadata (version, commit, date) as `ARG` values

---

## Stage 2: Lint (Static Analysis)

Lint the Dockerfile before running any containers — fast feedback, no compute wasted:

```bash
# Hadolint for Dockerfile linting
hadolint --failure-threshold warning Dockerfile

# Trivy for Dockerfile misconfiguration (before building)
trivy config --severity HIGH,CRITICAL Dockerfile
```

---

## Stage 3: Test

Run the test suite inside the built image, using the exact same environment that will run in production:

```bash
# Run tests in the built image
docker run --rm \
  --network test-net \
  -e DATABASE_URL=postgresql://test:test@db:5432/testdb \
  ${IMAGE_TAG} \
  pytest tests/ -v --junitxml=/tmp/test-results.xml

# Copy results out for CI reporting
docker run --rm \
  -v $(pwd)/test-results:/results \
  ${IMAGE_TAG} \
  sh -c "pytest tests/ --junitxml=/results/test-results.xml && cp coverage.xml /results/"
```

Using Docker Compose for integration tests:

```bash
docker compose -f docker-compose.test.yml up \
  --abort-on-container-exit \
  --exit-code-from test-runner

docker compose -f docker-compose.test.yml down
```

---

## Stage 4: Security Scan

Scan the built image for known CVEs before pushing:

```bash
# Fail the pipeline if Critical or High CVEs are found
trivy image \
  --exit-code 1 \
  --severity CRITICAL,HIGH \
  --format table \
  ${IMAGE_TAG}
```

```bash
# Also check for secrets accidentally embedded in the image
trivy image \
  --scanners secret \
  --exit-code 1 \
  ${IMAGE_TAG}
```

---

## Stage 5: Push

After all gates pass, push the image to the registry:

```bash
# Authenticate to ECR
aws ecr get-login-password --region ${AWS_REGION} | \
  docker login --username AWS --password-stdin ${ECR_REGISTRY}

# Push the commit-SHA-tagged image
docker push ${IMAGE_TAG}

# Push the latest tag (optional — for cache-from in subsequent builds)
docker push ${ECR_REGISTRY}/myapp:latest

# If this is a release, push the semantic version tag
if [[ ${GITHUB_REF} =~ ^refs/tags/v([0-9]+\.[0-9]+\.[0-9]+)$ ]]; then
  SEMVER=${BASH_REMATCH[1]}
  docker tag ${IMAGE_TAG} ${ECR_REGISTRY}/myapp:${SEMVER}
  docker push ${ECR_REGISTRY}/myapp:${SEMVER}
fi
```

---

## Stage 6: Deploy

With the image in the registry, trigger the deployment:

```bash
# Update ECS service to use the new image
aws ecs update-service \
  --cluster production \
  --service myapp-api \
  --force-new-deployment

# Wait for the rollout to complete
aws ecs wait services-stable \
  --cluster production \
  --services myapp-api

echo "Deployment complete."
```

Or for single-host Docker Compose deployments:

```bash
# SSH to production host and deploy
ssh deploy@production-host "
  cd /opt/myapp
  export APP_VERSION=${GIT_SHA}
  docker compose -f docker-compose.prod.yml pull
  docker compose -f docker-compose.prod.yml up -d
"
```

---

## The Two Environments Pattern

A key architectural decision: should the CI environment build images using Docker-in-Docker, or should CI runners have Docker available natively?

**Option 1: Docker-in-Docker (DinD)**

The CI runner itself is a container. To run Docker inside it, you need "Docker in Docker." This is complex, has security implications (requires `--privileged`), and has performance overhead.

**Option 2: Docker on the Host Runner (Recommended)**

The CI runner is a VM or a bare-metal machine with Docker installed. Docker commands run directly. No privileged containers, no special configuration. This is how GitHub Actions, most Jenkins setups, and AWS CodeBuild work.

```yaml
# GitHub Actions — runner has Docker pre-installed
runs-on: ubuntu-latest   # Ubuntu runner has Docker 24+ pre-installed
```

---

## Caching Strategy for Fast Builds

Build speed is a developer experience issue. Slow CI is a productivity killer.

### Layer Caching with Registry

```bash
# Pull the latest image to use as cache
docker pull ${ECR_REGISTRY}/myapp:latest || true

# Build using the pulled image as cache
docker build \
  --cache-from ${ECR_REGISTRY}/myapp:latest \
  -t ${IMAGE_TAG} \
  .

# Push both tags so next run can use this as cache
docker push ${IMAGE_TAG}
docker push ${ECR_REGISTRY}/myapp:latest
```

### BuildKit Inline Cache

```bash
# Build with inline cache metadata embedded in the image
DOCKER_BUILDKIT=1 docker build \
  --build-arg BUILDKIT_INLINE_CACHE=1 \
  --cache-from ${ECR_REGISTRY}/myapp:latest \
  -t ${IMAGE_TAG} \
  .
```

---

## Summary

- In Docker-based CI/CD, the container image is the deployment artifact — the same image that passes tests deploys to production
- A standard pipeline has six stages: Build → Lint → Test → Scan → Push → Deploy — each is a gate that can stop the pipeline
- Tag images with the git commit SHA for immutable, traceable artifacts; also tag `latest` for use as a build cache
- Use `--cache-from` to pull previous image layers — dramatically reduces build time by reusing unchanged layers
- Always scan with Trivy before pushing — `--exit-code 1 --severity CRITICAL,HIGH` ensures only secure images reach the registry
- Use Docker-native CI runners (GitHub Actions, AWS CodeBuild) rather than Docker-in-Docker for simplicity, security, and performance

In the next file, we build a complete Docker CI/CD pipeline using GitHub Actions — the most widely used CI platform.
