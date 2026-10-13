# Docker Build and Push Pipeline

## Introduction

Every CI platform — GitHub Actions, Jenkins, GitLab CI, CircleCI, AWS CodeBuild — runs the same core Docker operations. Understanding these operations as a set of shell commands that work anywhere makes you platform-independent. You can implement this pipeline in any environment, troubleshoot it without CI-specific knowledge, and run the exact same steps locally to reproduce CI failures.

This file presents the canonical Docker build-push pipeline as a set of reusable shell scripts, then shows how to plug them into CI platforms.

---

## The Core Script: build-push.sh

```bash
#!/usr/bin/env bash
# build-push.sh — Standard Docker build, test, scan, and push pipeline
# Usage: ./build-push.sh [push|no-push]

set -euo pipefail

# ─── Configuration ───────────────────────────────────────────────────────────

AWS_REGION="${AWS_REGION:-us-east-1}"
ECR_REGISTRY="${ECR_REGISTRY:-123456789012.dkr.ecr.us-east-1.amazonaws.com}"
IMAGE_NAME="${IMAGE_NAME:-myapp}"
GIT_SHA="${GIT_SHA:-$(git rev-parse --short HEAD)}"
BUILD_DATE=$(date -u +%Y-%m-%dT%H:%M:%SZ)
PUSH="${1:-no-push}"

IMAGE_BASE="${ECR_REGISTRY}/${IMAGE_NAME}"
IMAGE_SHA="${IMAGE_BASE}:${GIT_SHA}"
IMAGE_LATEST="${IMAGE_BASE}:latest"

# ─── Functions ────────────────────────────────────────────────────────────────

log() {
    echo "──── $* ────"
}

ecr_login() {
    log "Authenticating to ECR"
    aws ecr get-login-password --region "$AWS_REGION" | \
        docker login --username AWS --password-stdin "$ECR_REGISTRY"
}

pull_cache() {
    log "Pulling cache image"
    docker pull "${IMAGE_LATEST}" 2>/dev/null || true
}

build_image() {
    log "Building image: ${IMAGE_SHA}"
    docker build \
        --cache-from "${IMAGE_LATEST}" \
        --build-arg GIT_COMMIT="${GIT_SHA}" \
        --build-arg BUILD_DATE="${BUILD_DATE}" \
        --build-arg APP_VERSION="${APP_VERSION:-dev}" \
        -t "${IMAGE_SHA}" \
        -t "${IMAGE_LATEST}" \
        .
}

run_tests() {
    log "Running tests"
    docker run --rm \
        -v "$(pwd)/test-results:/results" \
        --network host \
        "${IMAGE_SHA}" \
        pytest tests/ -v --junitxml=/results/test-results.xml
}

scan_image() {
    log "Scanning image for vulnerabilities"
    docker run --rm \
        -v /var/run/docker.sock:/var/run/docker.sock \
        aquasec/trivy:latest \
        image \
        --exit-code 1 \
        --severity CRITICAL,HIGH \
        --format table \
        "${IMAGE_SHA}"
}

push_image() {
    if [[ "${PUSH}" == "push" ]]; then
        log "Pushing to ECR"
        docker push "${IMAGE_SHA}"
        docker push "${IMAGE_LATEST}"
        log "Pushed: ${IMAGE_SHA}"
    else
        log "Skipping push (PUSH=${PUSH})"
    fi
}

cleanup() {
    log "Cleaning up"
    docker rmi "${IMAGE_SHA}" "${IMAGE_LATEST}" 2>/dev/null || true
    docker image prune -f
}

# ─── Main ─────────────────────────────────────────────────────────────────────

main() {
    log "Starting pipeline for ${IMAGE_NAME}:${GIT_SHA}"

    ecr_login
    pull_cache
    build_image
    run_tests
    scan_image
    push_image

    log "Pipeline complete: ${IMAGE_SHA}"
}

trap cleanup EXIT

main
```

---

## Tagging Strategy

The tagging strategy determines how images are identified across environments.

### Commit SHA Tags

```bash
# Always unique — maps 1:1 with a git commit
GIT_SHA=$(git rev-parse --short HEAD)   # Short: a1b2c3d
GIT_SHA_FULL=$(git rev-parse HEAD)       # Full: a1b2c3d4e5f6789...

IMAGE="${ECR_REGISTRY}/myapp:${GIT_SHA}"
```

SHA tags are immutable. If you know the SHA, you know exactly which commit the image was built from and which code is running.

### Semantic Version Tags

```bash
# On a git tag like "v1.4.2"
GIT_TAG=$(git describe --tags --exact-match 2>/dev/null || echo "")

if [[ -n "${GIT_TAG}" && "${GIT_TAG}" =~ ^v[0-9]+\.[0-9]+\.[0-9]+$ ]]; then
    SEMVER="${GIT_TAG#v}"   # Remove the "v" prefix → "1.4.2"
    MAJOR="${SEMVER%%.*}"   # "1"
    MINOR="${SEMVER%.*}"    # "1.4"

    docker tag myapp:${GIT_SHA} ${ECR_REGISTRY}/myapp:${SEMVER}   # 1.4.2
    docker tag myapp:${GIT_SHA} ${ECR_REGISTRY}/myapp:${MINOR}    # 1.4
    docker tag myapp:${GIT_SHA} ${ECR_REGISTRY}/myapp:${MAJOR}    # 1
    docker tag myapp:${GIT_SHA} ${ECR_REGISTRY}/myapp:latest
fi
```

### Branch Tags

```bash
# For preview/staging environments — tag with branch name
BRANCH=$(git rev-parse --abbrev-ref HEAD | sed 's|/|-|g')   # feature/login → feature-login

docker tag myapp:${GIT_SHA} ${ECR_REGISTRY}/myapp:branch-${BRANCH}
```

---

## Multi-Architecture Builds (ARM + AMD64)

AWS Graviton EC2 instances use ARM (aarch64). If your team uses M-series Macs, the image must run on ARM locally. Build for both architectures in CI:

```bash
# Enable BuildKit with Buildx
docker buildx create --use --name multiarch-builder
docker buildx inspect --bootstrap

# Build for linux/amd64 and linux/arm64, push directly to registry
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --push \
  -t ${ECR_REGISTRY}/myapp:${GIT_SHA} \
  -t ${ECR_REGISTRY}/myapp:latest \
  .
```

A multi-architecture image is stored as an image index (manifest list) in ECR. When you `docker pull` it, Docker automatically selects the correct architecture for the current machine.

---

## Image Provenance — Signing and Attestation

For regulated environments, you may need to prove an image was built from a specific commit by a specific pipeline:

```bash
# Generate and attach SBOM (Software Bill of Materials) to the image
docker buildx build \
  --sbom=true \
  --provenance=true \
  --push \
  -t ${ECR_REGISTRY}/myapp:${GIT_SHA} \
  .
```

```bash
# Sign the image with cosign (keyless signing via OIDC in CI)
cosign sign \
  --yes \
  ${ECR_REGISTRY}/myapp:${GIT_SHA}@${IMAGE_DIGEST}
```

AWS ECR supports storing and verifying both SBOM attestations and cosign signatures.

---

## Using AWS CodeBuild

AWS CodeBuild is a managed build service that integrates natively with ECR, CodePipeline, and IAM roles:

```yaml
# buildspec.yml — AWS CodeBuild native format
version: 0.2

env:
  variables:
    AWS_REGION: us-east-1
    ECR_REGISTRY: 123456789012.dkr.ecr.us-east-1.amazonaws.com
    IMAGE_NAME: myapp
  parameter-store:
    APP_VERSION: /myapp/build/version

phases:
  pre_build:
    commands:
      - echo "Authenticating to ECR..."
      - aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REGISTRY
      - export GIT_SHA=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c1-8)
      - export IMAGE_TAG=$ECR_REGISTRY/$IMAGE_NAME:$GIT_SHA
      - echo "Building image $IMAGE_TAG"

  build:
    commands:
      - echo "Building Docker image..."
      - docker pull $ECR_REGISTRY/$IMAGE_NAME:latest || true
      - |
        docker build \
          --cache-from $ECR_REGISTRY/$IMAGE_NAME:latest \
          --build-arg GIT_COMMIT=$GIT_SHA \
          --build-arg BUILD_DATE=$(date -u +%Y-%m-%dT%H:%M:%SZ) \
          -t $IMAGE_TAG \
          -t $ECR_REGISTRY/$IMAGE_NAME:latest \
          .

  post_build:
    commands:
      - echo "Running tests..."
      - docker run --rm $IMAGE_TAG pytest tests/ -v

      - echo "Scanning for vulnerabilities..."
      - |
        docker run --rm \
          -v /var/run/docker.sock:/var/run/docker.sock \
          aquasec/trivy image --exit-code 1 --severity CRITICAL,HIGH $IMAGE_TAG

      - echo "Pushing to ECR..."
      - docker push $IMAGE_TAG
      - docker push $ECR_REGISTRY/$IMAGE_NAME:latest

      - echo "Generating image definition for ECS deploy..."
      - printf '[{"name":"api","imageUri":"%s"}]' $IMAGE_TAG > imagedefinitions.json

artifacts:
  files:
    - imagedefinitions.json
```

The `imagedefinitions.json` file is consumed by CodePipeline's ECS Deploy action to update the service with the new image.

---

## The Rollback Script

When a deployment fails, you need to rollback to the previous working image:

```bash
#!/bin/bash
# rollback.sh — Roll back ECS service to the previous task definition revision
set -euo pipefail

CLUSTER="${1:?Usage: ./rollback.sh CLUSTER SERVICE}"
SERVICE="${2:?Usage: ./rollback.sh CLUSTER SERVICE}"
AWS_REGION="${AWS_REGION:-us-east-1}"

# Get the current task definition ARN
CURRENT_TASK_DEF=$(aws ecs describe-services \
  --cluster "$CLUSTER" \
  --services "$SERVICE" \
  --region "$AWS_REGION" \
  --query 'services[0].taskDefinition' \
  --output text)

echo "Current task definition: $CURRENT_TASK_DEF"

# Get the previous revision number
CURRENT_REVISION=$(echo "$CURRENT_TASK_DEF" | grep -oP ':\d+$' | tr -d ':')
PREVIOUS_REVISION=$((CURRENT_REVISION - 1))
TASK_DEF_FAMILY=$(echo "$CURRENT_TASK_DEF" | grep -oP '[^/]+(?=:\d+$)')
PREVIOUS_TASK_DEF="${TASK_DEF_FAMILY}:${PREVIOUS_REVISION}"

echo "Rolling back to: $PREVIOUS_TASK_DEF"

aws ecs update-service \
  --cluster "$CLUSTER" \
  --service "$SERVICE" \
  --task-definition "$PREVIOUS_TASK_DEF" \
  --region "$AWS_REGION"

aws ecs wait services-stable \
  --cluster "$CLUSTER" \
  --services "$SERVICE" \
  --region "$AWS_REGION"

echo "Rollback complete."
```

---

## Summary

- The build-push pipeline is a sequence of shell commands: authenticate → pull cache → build → test → scan → push — these work identically on any CI platform
- Use git SHA tags for immutable, traceable image identification — `myapp:a1b2c3d` maps directly to the exact commit
- Use `--cache-from` with the `latest` tag to reuse unchanged layers — critical for keeping build times fast in CI
- Multi-architecture builds with `docker buildx` produce a single image that runs on both AMD64 and ARM64 — essential for AWS Graviton instances and Apple Silicon developer machines
- AWS CodeBuild uses `buildspec.yml` natively and automatically has IAM role credentials — no access key management needed
- Always have a rollback script that reverts the ECS service to the previous task definition revision — rollback must be a one-command operation during an incident

In the next chapter, we cover Docker registries — Docker Hub, AWS ECR, and managing authentication and lifecycle policies.
