# Security Best Practices for Dockerfiles

## Introduction

A Dockerfile is a security boundary. Every decision you make — which base image, which user, which packages, which files you copy — determines how large your attack surface is. Most container security incidents are not zero-day exploits; they are predictable failures to apply basic hardening that was available and well-documented.

This file is a complete security hardening guide for production Dockerfiles — covering every major control from base image selection to runtime constraints.

---

## 1. Use Minimal, Pinned Base Images

### Use Minimal Base Images

Fewer packages = fewer vulnerabilities. Each package in your base image is a potential attack surface.

```dockerfile
# BAD — Ubuntu has hundreds of packages, most unneeded
FROM ubuntu:22.04

# BETTER — Debian slim removes most unnecessary packages
FROM python:3.11-slim

# BEST — Distroless has no shell, no package manager, minimal attack surface
FROM gcr.io/distroless/python3-debian12
```

| Base Image | Size | Shell | Package Manager | Attack Surface |
|---|---|---|---|---|
| ubuntu:22.04 | ~77MB | Yes | Yes | Large |
| python:3.11-slim | ~125MB | Yes | No (apt removed) | Medium |
| python:3.11-alpine | ~50MB | Yes | apk | Small |
| gcr.io/distroless/python3 | ~50MB | No | No | Minimal |

### Pin Base Images to Exact Digests

Tags like `python:3.11-slim` can change at any moment when the image maintainer pushes an update. This means your build is not reproducible and a compromised upstream image can silently affect your builds.

```dockerfile
# BAD — tag can change
FROM python:3.11-slim

# GOOD — digest is immutable, build is fully reproducible
FROM python:3.11-slim@sha256:a1b2c3d4e5f6789012345678901234567890abcdef1234567890abcdef123456
```

```bash
# Get the digest of an image
docker pull python:3.11-slim
docker inspect python:3.11-slim --format '{{index .RepoDigests 0}}'
# python@sha256:a1b2c3d4...
```

In CI/CD, use tools like Renovate or Dependabot to automatically update pinned digests when security patches are released.

---

## 2. Run as a Non-Root User

Always switch to a non-root user before the final CMD/ENTRYPOINT:

```dockerfile
RUN groupadd --system --gid 1001 appuser && \
    useradd --system --uid 1001 --gid 1001 --no-create-home --shell /bin/false appuser

# ... setup ...

COPY --chown=appuser:appuser . /app

USER 1001
```

Verify in CI:

```bash
# Fail the build if the final USER is root
docker inspect myapp:1.0 --format '{{.Config.User}}' | grep -q "0\|root" && \
  (echo "ERROR: Container runs as root" && exit 1) || echo "OK: Non-root user"
```

---

## 3. Never Store Secrets in the Image

```dockerfile
# FORBIDDEN — credentials visible in docker history
ENV AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
ARG PYPI_PASSWORD=supersecret
RUN curl -H "Authorization: Bearer ${API_KEY}" https://api.example.com/

# CORRECT — use BuildKit secrets for build-time credentials
RUN --mount=type=secret,id=pypi_token \
    pip install --index-url https://user:$(cat /run/secrets/pypi_token)@mypypi.com/simple pkg

# CORRECT — inject secrets at runtime via environment variables
# (fetched from AWS Secrets Manager, never hardcoded)
```

Verify after build:

```bash
# Scan image history for common secret patterns
docker history myapp:1.0 --no-trunc | grep -iE "password|secret|token|key|api_key"
```

---

## 4. Keep Images Minimal — Remove Build Artifacts

Build tools installed to compile native extensions should not exist in the final image:

```dockerfile
FROM python:3.11-slim

# Install build tools only temporarily
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        gcc \
        libpq-dev \
    && pip install --no-cache-dir psycopg2==2.9.9 \
    && apt-get purge -y gcc libpq-dev \
    && apt-get autoremove -y \
    && rm -rf /var/lib/apt/lists/*

# Now psycopg2 is installed but gcc is gone
```

Or better — use multi-stage builds:

```dockerfile
FROM python:3.11-slim AS builder

RUN apt-get update && \
    apt-get install -y --no-install-recommends gcc libpq-dev && \
    pip install --user --no-cache-dir -r requirements.txt

FROM python:3.11-slim

COPY --from=builder /root/.local /root/.local
ENV PATH=/root/.local/bin:$PATH

# gcc is never in the final image
```

---

## 5. Use .dockerignore

Without `.dockerignore`, `COPY . /app` copies everything — including `.git`, local `.env` files, test data, CI configuration, and developer credentials.

```dockerignore
# .dockerignore
.git
.gitignore
.env
.env.*
*.env
node_modules
__pycache__
*.pyc
*.pyo
.venv
venv
*.log
*.tar
*.zip
dist
build
coverage
.coverage
htmlcov
.pytest_cache
.mypy_cache
Dockerfile*
docker-compose*
.dockerignore
README.md
docs
tests
.github
.gitlab-ci.yml
.circleci
```

---

## 6. Scan Images for Vulnerabilities

Integrate vulnerability scanning into the build pipeline. Never ship images with known Critical or High severity CVEs.

### Trivy (Most Common Open-Source Scanner)

```bash
# Scan locally
trivy image myapp:1.0

# Fail CI if critical or high vulnerabilities found
trivy image --exit-code 1 --severity CRITICAL,HIGH myapp:1.0

# Output as SARIF for GitHub Security tab
trivy image --format sarif --output results.sarif myapp:1.0

# Scan a dockerfile (static analysis)
trivy config --severity HIGH,CRITICAL Dockerfile
```

### AWS ECR Enhanced Scanning

```bash
# Push to ECR — Enhanced Scanning runs automatically
aws ecr put-image-scanning-configuration \
  --repository-name myapp \
  --image-scanning-configuration scanOnPush=true

# Get scan results
aws ecr describe-image-scan-findings \
  --repository-name myapp \
  --image-id imageTag=1.4.2
```

### In CI/CD (GitHub Actions)

```yaml
- name: Build image
  run: docker build -t myapp:${{ github.sha }} .

- name: Scan with Trivy
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: myapp:${{ github.sha }}
    format: table
    exit-code: 1
    severity: CRITICAL,HIGH
```

---

## 7. Apply Filesystem Permissions Correctly

Files in production images should be owned by the appropriate user and have restrictive permissions:

```dockerfile
# Copy with correct ownership
COPY --chown=appuser:appuser src/ /app/

# Config files — readable by app, not writable
COPY --chown=root:appuser config/app.yml /app/config.yml
RUN chmod 640 /app/config.yml   # owner (root) can write, group (appuser) can read

# Executable scripts — executable by app user
COPY --chown=appuser:appuser scripts/entrypoint.sh /usr/local/bin/
RUN chmod 550 /usr/local/bin/entrypoint.sh  # owner and group can execute, others cannot
```

---

## 8. Use Read-Only Root Filesystem

Prevent the application from modifying its own filesystem — limits damage if compromised:

```dockerfile
# In the Dockerfile, document writable paths needed at runtime
# (VOLUME creates anonymous volumes for these paths)
VOLUME ["/tmp", "/app/logs"]
```

```bash
# Enable at runtime
docker run --read-only myapp:1.0

# Or in Compose
services:
  api:
    read_only: true
    tmpfs:
      - /tmp:size=256m
```

Test which paths need to be writable:

```bash
docker run --read-only myapp:1.0
# PermissionError: [Errno 30] Read-only file system: '/tmp/gunicorn.pid'
# → Add --tmpfs /tmp or -v tmp:/tmp
```

---

## 9. Drop Unnecessary Capabilities

By default, Docker grants a set of Linux capabilities that most applications do not need. Drop all and add back only what is required:

```bash
# Drop all capabilities, add back only what is needed
docker run --cap-drop ALL --cap-add NET_BIND_SERVICE myapp:1.0
```

In Compose:

```yaml
services:
  api:
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE   # only if you actually need to bind to port < 1024
```

Most web applications and APIs need zero Linux capabilities. Use high ports (8080) and drop `NET_BIND_SERVICE` too.

---

## 10. Do Not Install curl or wget in Production

These tools are often used in health checks but make the container more dangerous — an attacker can use them to download additional payloads.

Alternatives:

```dockerfile
# Instead of curl for health checks, use a minimal binary
COPY --from=healthcheck-builder /healthcheck /usr/local/bin/healthcheck
HEALTHCHECK CMD /usr/local/bin/healthcheck

# Or install nc (netcat) for TCP health checks without full HTTP
# nc is much smaller than curl and cannot download content
```

For Alpine-based images, use `wget` from `busybox` (already present) instead of installing `curl`.

---

## 11. The Complete Production Dockerfile Checklist

Apply this checklist to every production Dockerfile:

```
[ ] Base image is minimal (slim, alpine, or distroless)
[ ] Base image is pinned to a SHA256 digest
[ ] Build tools are removed or in a separate build stage
[ ] .dockerignore prevents sensitive files from entering the build context
[ ] No secrets in ENV, ARG, or RUN commands
[ ] BuildKit secrets used for build-time credentials
[ ] Non-root user created and switched to before CMD
[ ] COPY --chown used for correct file ownership
[ ] HEALTHCHECK defined with appropriate start_period
[ ] Vulnerability scan passes with no Critical/High findings
[ ] Image size reviewed (docker images — flag if unexpectedly large)
[ ] docker history reviewed — no credentials visible
[ ] OCI labels set (version, revision, source, maintainer)
```

---

## Summary

- Pin base images to SHA256 digests for reproducible, tamper-proof builds — use Renovate or Dependabot to auto-update digests
- Never store secrets in `ENV`, `ARG`, or `RUN` commands — use BuildKit `--mount=type=secret` for build-time credentials and runtime environment injection for application secrets
- Use minimal base images (`slim`, `alpine`, `distroless`) — each package is a potential vulnerability surface
- Remove build tools from the final image with multi-stage builds — the compiler should not be in production
- Always switch to a non-root user (`USER 1001`) before `CMD` — verify this in CI with `docker inspect`
- Enable read-only root filesystem at runtime and use `--tmpfs` for writable paths — this limits what an attacker can do even with arbitrary code execution
- Integrate Trivy or ECR Enhanced Scanning into CI/CD — fail builds that introduce Critical or High severity CVEs
- Drop all Linux capabilities (`--cap-drop ALL`) unless your application specifically requires one

In the next file, we cover Dockerfile linting — automatically catching style violations, security issues, and best practice violations before they reach production.
