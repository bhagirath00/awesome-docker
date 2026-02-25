# Secrets in Docker Builds

## Introduction

Build processes frequently need credentials: to pull packages from a private PyPI or npm registry, to clone private Git repositories during build, to authenticate against internal APIs. The naive approach — passing secrets as `ARG` or `ENV` values — permanently embeds them in the image's layer history, visible to anyone who can pull the image.

This file covers the correct approaches for handling secrets during Docker builds: BuildKit's secret mounts, SSH forwarding, and multi-stage build patterns. These are not theoretical — credential leakage in Docker images is a real and common vulnerability.

---

## Why Build Arguments Are Not Safe for Secrets

```dockerfile
# DANGEROUS — password is baked into the layer history
ARG PYPI_TOKEN=default
RUN pip install --index-url https://oauth2:${PYPI_TOKEN}@mypypi.example.com/simple mypackage
```

Even if you later unset the `ARG` or use a different `RUN` command, the value is captured in `docker history`:

```bash
docker history myapp:1.0 --no-trunc

# Output shows the full command including the token:
# sha256:... RUN pip install --index-url https://oauth2:pypi-Token123abc@mypypi... mypackage
```

This history is stored in the image manifest and accessible on ECR, Docker Hub, and any registry. It cannot be removed without rebuilding the image.

---

## Solution 1: BuildKit Secret Mounts (Recommended)

BuildKit's `--mount=type=secret` provides a secure mechanism to pass secrets to a `RUN` command. The secret is mounted as a file during the build, available only for that single `RUN` instruction, and is **never stored in any image layer**.

### Enabling BuildKit

BuildKit is the default in Docker 23+ and Docker Desktop. For older versions:

```bash
# Enable BuildKit for a single build
DOCKER_BUILDKIT=1 docker build -t myapp .

# Enable globally in Docker daemon config
# /etc/docker/daemon.json
{
  "features": {
    "buildkit": true
  }
}
```

### Syntax in the Dockerfile

```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .

# Secret is mounted at /run/secrets/pypi_token during this RUN only
# It is NOT stored in any layer
RUN --mount=type=secret,id=pypi_token \
    pip install \
      --index-url https://oauth2:$(cat /run/secrets/pypi_token)@mypypi.example.com/simple \
      --no-cache-dir \
      -r requirements.txt

COPY src/ .
CMD ["python", "app.py"]
```

### Passing the Secret at Build Time

```bash
# Pass secret from a file
docker build --secret id=pypi_token,src=./secrets/pypi_token.txt -t myapp .

# Pass secret from an environment variable
docker build --secret id=pypi_token,env=PYPI_TOKEN -t myapp .

# Pass secret from a file with the standard secret path
export PYPI_TOKEN=pypi-Token123abc
docker build --secret id=pypi_token,env=PYPI_TOKEN -t myapp .
```

The secret is only available during the `RUN --mount=type=secret,...` instruction. After the instruction completes, the secret is unmounted. `docker history` and `docker inspect` show nothing about the secret value.

---

## Solution 2: SSH Agent Forwarding

When the build needs to clone private Git repositories (for go modules, npm packages, pip installs from git URLs), SSH agent forwarding provides secure access without embedding private keys.

```dockerfile
# syntax=docker/dockerfile:1
FROM golang:1.22-alpine AS builder

WORKDIR /build

# Mount the SSH agent socket for this RUN instruction only
RUN --mount=type=ssh \
    go env -w GOPRIVATE=github.com/myorg && \
    go mod download

COPY . .
RUN go build -o /build/api ./cmd/api
```

```bash
# Add your SSH key to the agent first
eval $(ssh-agent)
ssh-add ~/.ssh/id_rsa

# Build with SSH agent forwarding
docker build --ssh default -t myapp .
```

The SSH key is never embedded in the image. It is forwarded through the Docker socket only for the duration of the SSH-mounted `RUN` instruction.

### Configuring SSH Known Hosts

Before using SSH forwarding, the build needs to trust the git host:

```dockerfile
# Add GitHub to known hosts during the build
RUN --mount=type=ssh \
    mkdir -p ~/.ssh && \
    ssh-keyscan github.com >> ~/.ssh/known_hosts && \
    git clone git@github.com:myorg/private-lib.git /tmp/private-lib
```

---

## Solution 3: Multiple Secrets in One Build

BuildKit supports multiple secret mounts in a single `RUN`:

```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt requirements-private.txt ./

RUN --mount=type=secret,id=pypi_token \
    --mount=type=secret,id=gemfury_token \
    pip install \
      --index-url https://oauth2:$(cat /run/secrets/pypi_token)@pypi.example.com/simple \
      --extra-index-url https://$(cat /run/secrets/gemfury_token):@pypi.fury.io/myorg/ \
      --no-cache-dir \
      -r requirements.txt -r requirements-private.txt
```

```bash
docker build \
  --secret id=pypi_token,env=PYPI_TOKEN \
  --secret id=gemfury_token,env=GEMFURY_TOKEN \
  -t myapp .
```

---

## Solution 4: Multi-Stage Build with Credential Layer

If BuildKit is not available (legacy environments), multi-stage builds provide a fallback. Credentials are used in an early stage that is not present in the final image:

```dockerfile
# Stage 1: dependency installation (this stage uses credentials)
FROM python:3.11-slim AS deps

ARG PYPI_TOKEN
WORKDIR /install

# Install packages with credentials
RUN pip install \
    --index-url https://oauth2:${PYPI_TOKEN}@mypypi.example.com/simple \
    --prefix=/install/packages \
    --no-cache-dir \
    mypackage

# Stage 2: production image (DOES NOT inherit the ARG or layer history)
FROM python:3.11-slim

WORKDIR /app

# Copy only the installed packages — not the credentials or the build layer
COPY --from=deps /install/packages /usr/local

COPY src/ .
CMD ["python", "app.py"]
```

**Important caveat:** The `ARG PYPI_TOKEN` and the `RUN pip install ...` instruction that uses it are in the `deps` stage. That stage is NOT included in the final image. However, the `deps` stage image itself (as a build cache artifact) may still contain the credential in its layer history if it is pushed to a registry as a cache source. Use BuildKit secrets whenever possible.

```bash
# Build with the credential as a build arg (fallback approach)
docker build --build-arg PYPI_TOKEN=pypi-Token123abc -t myapp .
```

---

## Protecting AWS Credentials During Build

Building on an EC2 instance or a CI runner with an IAM role? The recommended approach is to use the instance's IAM role credentials via the IMDSv2 endpoint — no hardcoded credentials needed.

```dockerfile
# syntax=docker/dockerfile:1
FROM amazon/aws-cli AS fetcher

# Fetch a config file from S3 using the instance IAM role
# The credentials come from the instance's IMDSv2 endpoint automatically
RUN --mount=type=secret,id=aws_creds,target=/root/.aws/credentials \
    aws s3 cp s3://my-private-bucket/config.json /tmp/config.json

FROM python:3.11-slim

COPY --from=fetcher /tmp/config.json /app/config.json
```

```bash
# Mount credentials from host (from AWS CLI config)
docker build --secret id=aws_creds,src=$HOME/.aws/credentials -t myapp .
```

Or better — run the build on an EC2 instance with an IAM instance profile that grants S3 access. The `aws-cli` image will automatically use the instance credentials from IMDSv2 without any explicit credential mounting.

---

## Verifying No Secrets in Layers

After a build using BuildKit secrets, verify the secret did not leak:

```bash
# Check build history — secret should not appear in any command
docker history myapp:1.0 --no-trunc | grep -i "token\|password\|secret\|key"
# Should output nothing

# Inspect image metadata
docker inspect myapp:1.0 | grep -i "token\|password\|secret\|key"
# Should output nothing

# Check each layer manually (advanced — scan the actual layer tar)
docker save myapp:1.0 | tar x --to-stdout | grep -r "PYPI_TOKEN"
# Should output nothing
```

---

## In CI/CD Pipelines

### GitHub Actions

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx (includes BuildKit)
        uses: docker/setup-buildx-action@v3

      - name: Build with secrets
        run: |
          docker build \
            --secret id=pypi_token,env=PYPI_TOKEN \
            -t myapp:${{ github.sha }} .
        env:
          PYPI_TOKEN: ${{ secrets.PYPI_TOKEN }}
```

### GitLab CI

```yaml
build:
  image: docker:24
  services:
    - docker:dind
  script:
    - docker build
        --secret id=pypi_token,env=PYPI_TOKEN
        -t myapp:$CI_COMMIT_SHA .
  variables:
    PYPI_TOKEN: $PYPI_TOKEN   # from GitLab CI/CD Variables
    DOCKER_BUILDKIT: "1"
```

---

## Summary

- Build arguments (`ARG`) are stored in `docker history` and visible to anyone with image access — never use them for secrets
- BuildKit `--mount=type=secret,id=secret_id` provides secrets during a single `RUN` instruction that are never stored in any layer — this is the recommended approach
- Pass secrets to BuildKit builds with `--secret id=name,env=ENV_VAR` or `--secret id=name,src=./path/to/file`
- SSH agent forwarding (`--mount=type=ssh`) provides secure access to private Git repositories without embedding SSH keys in the image
- The multi-stage fallback (use `ARG` in an intermediate stage, only `COPY` the outputs to the final stage) reduces exposure but is not as secure as BuildKit secrets
- Always verify after build: `docker history myapp:1.0 --no-trunc | grep -i token` should return nothing for secrets handled correctly

In the next file, we cover Dockerfile security best practices — a comprehensive checklist for hardening production images.
