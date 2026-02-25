# Dockerfile Linting

## Introduction

Writing a Dockerfile is easy. Writing a correct, secure, efficient Dockerfile is harder. Common mistakes — wrong instruction order that breaks the cache, using `latest` tags, running as root, not cleaning up package manager caches — are invisible without tooling.

Dockerfile linting automatically checks your Dockerfile against a ruleset of best practices and security policies. It catches issues before they reach production, enforces team standards, and provides immediate educational feedback on why a practice is problematic.

This file covers the primary linting tools, how to integrate them into CI/CD, and how to write linting configurations tailored to your organization's policies.

---

## Hadolint — The Standard Dockerfile Linter

[Hadolint](https://github.com/hadolint/hadolint) is the most widely used Dockerfile linter. It checks for:

- Instruction ordering and cache optimization issues
- Missing `--no-install-recommends` on `apt-get install`
- Using `latest` tag in `FROM` instructions
- Missing version pins in package installs
- Security issues (running as root, secret exposure patterns)
- Syntax errors

### Installation

```bash
# Docker (no installation needed — run from the image)
docker run --rm -i hadolint/hadolint < Dockerfile

# macOS (Homebrew)
brew install hadolint

# Linux (binary)
wget -O /usr/local/bin/hadolint https://github.com/hadolint/hadolint/releases/latest/download/hadolint-Linux-x86_64
chmod +x /usr/local/bin/hadolint

# Windows (Scoop)
scoop install hadolint
```

### Basic Usage

```bash
# Lint a Dockerfile
hadolint Dockerfile

# Lint with JSON output (for CI integration)
hadolint -f json Dockerfile

# Lint multiple files
hadolint Dockerfile Dockerfile.dev Dockerfile.test

# Ignore specific rules
hadolint --ignore DL3008 --ignore DL3009 Dockerfile
```

---

## Understanding Hadolint Rules

Hadolint rules are named with a `DL` (Dockerfile Linting) or `SC` (ShellCheck — for shell commands inside `RUN`) prefix.

### Common Rules You Will Encounter

**DL3007 — Using latest is best avoided**

```dockerfile
# Triggers DL3007
FROM node:latest

# Fix: pin to a specific version
FROM node:20-alpine
```

**DL3008 — Pin versions in apt-get install**

```dockerfile
# Triggers DL3008
RUN apt-get install -y curl git

# Fix: pin versions
RUN apt-get install -y curl=7.81.0-1ubuntu1.14 git=1:2.34.1-1ubuntu1.10
```

In practice, exact version pinning in apt is difficult to maintain. A pragmatic alternative:

```bash
hadolint --ignore DL3008 Dockerfile
```

Or use Alpine which has better version stability, or accept the rule and use `apt-cache madison curl` to find pinnable versions.

**DL3009 — Delete the apt-get lists after installing**

```dockerfile
# Triggers DL3009
RUN apt-get update && apt-get install -y curl

# Fix: clean up to reduce image size
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl && \
    rm -rf /var/lib/apt/lists/*
```

**DL3013 — Pin versions in pip install**

```dockerfile
# Triggers DL3013
RUN pip install flask requests

# Fix: pin versions
RUN pip install flask==3.0.0 requests==2.31.0
# Or use requirements.txt with pinned versions (preferred)
```

**DL3016 — Pin versions in npm install**

```dockerfile
# Triggers DL3016
RUN npm install express

# Fix: use package-lock.json with npm ci
COPY package.json package-lock.json ./
RUN npm ci --only=production
```

**DL3025 — Use arguments JSON notation for CMD and ENTRYPOINT**

```dockerfile
# Triggers DL3025 — shell form
CMD python app.py

# Fix: exec form (JSON array)
CMD ["python", "app.py"]
```

**DL3045 — COPY to a relative destination without WORKDIR**

```dockerfile
# Triggers DL3045 — relative path without WORKDIR set
COPY . app/

# Fix: set WORKDIR first
WORKDIR /app
COPY . .
```

---

## Configuring Hadolint

Create a `.hadolint.yaml` file in your project root to configure linting behavior:

```yaml
# .hadolint.yaml

# Set the minimum failure level (error, warning, info, style, ignore)
failure-threshold: warning

# Globally ignore specific rules
ignore:
  - DL3008    # apt version pinning — too strict for most teams
  - DL3009    # sometimes intentional not to clear apt lists in multi-step layers
  - DL3018    # apk version pinning — often impractical

# Rules that should be errors (fails CI)
# (everything above failure-threshold is an error by default)

# Override rule severity
override:
  error:
    - DL3007   # treat "latest" tag as error, not warning
    - DL3025   # treat shell-form CMD as error
  warning:
    - DL3003   # use WORKDIR instead of RUN cd

# Trust specific base images (skip FROM analysis for internal images)
trustedImages:
  - 123456789012.dkr.ecr.us-east-1.amazonaws.com/base-image

# Label schema validation
label-schema:
  app: text
  maintainer: text
  version: semver
```

---

## CI/CD Integration

### GitHub Actions

```yaml
name: Docker Lint and Build

on: [push, pull_request]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Lint Dockerfile with Hadolint
        uses: hadolint/hadolint-action@v3.1.0
        with:
          dockerfile: Dockerfile
          config: .hadolint.yaml
          failure-threshold: warning

  build:
    needs: lint    # Build only if lint passes
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build image
        run: docker build -t myapp:${{ github.sha }} .
```

### GitLab CI

```yaml
stages:
  - lint
  - build
  - scan

dockerfile-lint:
  stage: lint
  image: hadolint/hadolint:latest-alpine
  script:
    - hadolint Dockerfile
  allow_failure: false   # block pipeline on lint failures

image-build:
  stage: build
  needs: [dockerfile-lint]
  image: docker:24
  script:
    - docker build -t myapp:$CI_COMMIT_SHA .
```

---

## Additional Linting Tools

### docker build --check (Docker Built-in Linting)

Docker 24.0+ includes a basic built-in linting pass that runs before the actual build:

```bash
# Check Dockerfile without building
docker build --check .

# Output:
# [+] Building 0.0s (2/2) FINISHED
# WARN: JSONArgsRecommended
# WARN: LegacyKeyValueFormat
```

### Dockle — Security-Focused Linting

[Dockle](https://github.com/goodwithtech/dockle) focuses on CIS Docker Benchmark compliance:

```bash
# Install
brew install goodwithtech/r/dockle  # macOS
docker run --rm goodwithtech/dockle myapp:1.0  # Docker

# Run
dockle myapp:1.0

# Output example:
# FATAL   - CIS-DI-0001: Create a user for the container
#           * Last user should not be root
# WARN    - CIS-DI-0006: Add HEALTHCHECK instruction to the container image
# INFO    - DKL-DI-0006: Avoid latest tag
```

Dockle checks the built image (not just the Dockerfile), so it catches runtime user settings and missing health checks regardless of how they were configured.

### Trivy (Vulnerability + Misconfiguration)

Trivy scans both for CVEs and misconfiguration:

```bash
# Scan Dockerfile for misconfigurations
trivy config Dockerfile

# Scan built image for CVEs + misconfigurations
trivy image myapp:1.0

# Combined output
trivy image --scanners vuln,config myapp:1.0
```

---

## Automated Fix Suggestions

Hadolint provides fix suggestions in its output:

```bash
hadolint Dockerfile

# Output:
# Dockerfile:4 DL3007 warning: Using latest is best avoided
#              Consider using a specific version tag instead of latest
# Dockerfile:8 DL3008 warning: Pin versions in apt get install
#              Delete the apt-get lists after installing something.
#              Example: RUN apt-get update && apt-get install -y --no-install-recommends curl
#              && rm -rf /var/lib/apt/lists/*
```

---

## IDE Integration

Most IDEs have Hadolint extensions:

- **VS Code**: Install the "hadolint" extension — inline linting as you type
- **JetBrains (IntelliJ, GoLand, PyCharm)**: Docker plugin includes Dockerfile linting

```json
// VS Code settings.json
{
    "hadolint.hadolintPath": "hadolint",
    "hadolint.outputFormat": "column"
}
```

---

## Pre-commit Hook Integration

Prevent developers from committing Dockerfiles with lint errors:

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/hadolint/hadolint
    rev: v2.12.0
    hooks:
      - id: hadolint
        args:
          - --config=.hadolint.yaml
          - --failure-threshold=warning
```

```bash
# Install pre-commit
pip install pre-commit

# Install the hooks
pre-commit install

# Now every git commit runs hadolint automatically
git commit -m "Update Dockerfile"
# Hadolint...................................................................Failed
# - hook id: hadolint
# - exit code: 1
# Dockerfile:3 DL3007 warning: Using latest is best avoided
```

---

## A Complete Linting Pipeline

```yaml
# .github/workflows/docker.yml

name: Docker Quality Gate

on:
  push:
    branches: [main]
  pull_request:

jobs:
  quality-gate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # Step 1: Static Dockerfile lint
      - name: Hadolint - Dockerfile lint
        uses: hadolint/hadolint-action@v3.1.0
        with:
          dockerfile: Dockerfile
          config: .hadolint.yaml

      # Step 2: Build the image
      - name: Build Docker image
        run: docker build -t myapp:${{ github.sha }} .

      # Step 3: Dockle - CIS benchmark check on built image
      - name: Dockle - CIS security check
        run: |
          docker run --rm \
            -v /var/run/docker.sock:/var/run/docker.sock \
            goodwithtech/dockle:latest \
            --exit-code 1 \
            --exit-level warn \
            myapp:${{ github.sha }}

      # Step 4: Trivy - vulnerability scan
      - name: Trivy - vulnerability scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: myapp:${{ github.sha }}
          exit-code: 1
          severity: CRITICAL,HIGH

      # Step 5: Push to ECR (only on main)
      - name: Push to ECR
        if: github.ref == 'refs/heads/main'
        run: |
          aws ecr get-login-password | docker login --username AWS --password-stdin $ECR_REGISTRY
          docker push myapp:${{ github.sha }}
```

---

## Summary

- Hadolint is the standard Dockerfile linter — run `hadolint Dockerfile` locally and in CI to catch instruction ordering, missing version pins, shell-form CMD/ENTRYPOINT, and apt cleanup issues
- Configure per-project linting policy in `.hadolint.yaml` — set `failure-threshold`, `ignore` impractical rules, and promote critical rules to errors
- Run Hadolint in CI before building the image — fail the pipeline on warnings to enforce standards
- Dockle checks the built image against the CIS Docker Benchmark — catches runtime issues that Hadolint cannot see (like missing HEALTHCHECK or running as root)
- Trivy scans for CVEs and misconfigurations — integrate with `--exit-code 1 --severity CRITICAL,HIGH` to fail CI when high-severity vulnerabilities are introduced
- Install the hadolint VS Code extension or JetBrains Docker plugin for inline linting as you write

This completes Module 9 on Advanced Dockerfiles and Production Practices. You now have the full toolkit for writing secure, efficient, well-documented Dockerfiles that meet production standards.
