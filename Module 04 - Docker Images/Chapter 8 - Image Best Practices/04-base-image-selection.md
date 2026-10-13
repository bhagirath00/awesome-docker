# Base Image Selection

## Introduction

The base image you choose with `FROM` is the foundation of everything else in your Docker image. It affects your image size, security posture, build time, runtime performance, and compatibility. A poor choice here creates problems that are difficult to fix later.

This file gives you a framework for making the right base image decision for every type of application you work with.

---

## What Choosing a Base Image Means

When you write `FROM python:3.11-slim`, you are inheriting:

- An operating system environment (Debian Bookworm in this case)
- All files in the base image's layers
- All security vulnerabilities currently in those layers
- The package manager available (`apt`)
- The C library used (`glibc` for Debian/Ubuntu, `musl` for Alpine)
- All environment variables and configurations from the base image

Every choice you make in `FROM` cascades through your entire image. This is why base image selection deserves careful thought.

---

## The Main Base Image Families

### Debian and Ubuntu

Debian-based images are the most widely compatible. They use `glibc` (the standard C library), which almost all Linux software is compiled against.

```dockerfile
FROM ubuntu:22.04        # Full Ubuntu — largest
FROM debian:bookworm     # Full Debian — slightly smaller than Ubuntu
FROM debian:bookworm-slim  # Slim Debian — many tools removed
```

| Variant | Size | Use case |
|---|---|---|
| `ubuntu:22.04` | ~78MB | When you need Ubuntu-specific tools |
| `debian:bookworm` | ~117MB | General-purpose base |
| `debian:bookworm-slim` | ~75MB | Production when full Debian compatibility needed |

### Alpine Linux

Alpine is built around `musl libc` and `busybox`. It is extremely small but less compatible than Debian.

```dockerfile
FROM alpine:3.18    # ~8MB base image
```

**Pros:**
- Tiny image size
- Minimal attack surface
- Fast to pull and deploy

**Cons:**
- Uses `musl libc` instead of `glibc` — some software compiled for `glibc` does not work or behaves differently
- `apk` instead of `apt` — different package names and availability
- Some Python packages with C extensions fail to build (need compilation)
- Debugging is harder — fewer tools available

### Distroless

Distroless images from Google contain only the runtime needed to run your application — no shell, no package manager, no utilities.

```dockerfile
FROM gcr.io/distroless/python3-debian12    # Python only, no shell
FROM gcr.io/distroless/java21-debian12     # Java only
FROM gcr.io/distroless/nodejs20-debian12   # Node.js only
FROM gcr.io/distroless/static-debian12     # For static binaries
```

**Pros:**
- Smallest possible attack surface — no shell means attackers cannot run shell commands even if they exploit your app
- Passes most security scanners with minimal findings
- Forces good security practices

**Cons:**
- Cannot exec into a running container (`docker exec -it bash` does not work)
- Cannot install additional tools inside the container
- Debugging requires running a debug variant or using sidecar containers

### scratch

The empty base image — used for statically compiled binaries:

```dockerfile
FROM scratch
```

The resulting image contains only what you explicitly copy into it. Common for Go applications compiled with `CGO_ENABLED=0`.

---

## Language-Specific Official Images

For most applications, the right choice is the official image for your language runtime. These are maintained by the language communities and updated regularly with security patches.

### Python

```dockerfile
FROM python:3.11-slim       # Recommended for most Python apps
FROM python:3.11-alpine     # Smaller, but watch for musl compatibility issues
FROM python:3.11            # Full Debian — only when you need all Debian packages
```

For AWS Lambda:
```dockerfile
FROM public.ecr.aws/lambda/python:3.11   # AWS Lambda base image for Python
```

### Node.js

```dockerfile
FROM node:20-alpine         # Recommended — small and fast
FROM node:20-slim           # Slim Debian variant
FROM node:20                # Full Debian — rarely needed
```

For AWS Lambda:
```dockerfile
FROM public.ecr.aws/lambda/nodejs:20
```

### Java

```dockerfile
FROM eclipse-temurin:21-jre-alpine    # JRE only (smaller) — for running apps
FROM eclipse-temurin:21-jdk-alpine    # JDK (larger) — for building apps
FROM amazoncorretto:21-alpine         # AWS-optimized JDK
```

Amazon Corretto is optimized for AWS workloads and is what AWS Lambda and ECS use internally.

### Go

```dockerfile
# Build stage — needs the full Go toolchain
FROM golang:1.21-alpine AS builder

# Final stage — no Go runtime needed (statically compiled)
FROM scratch
# OR
FROM alpine:3.18   # if you need a shell for debugging
```

Go compiles to static binaries. Your final image does not need Go installed.

### Nginx (Frontend / Reverse Proxy)

```dockerfile
FROM nginx:1.25.3-alpine    # Recommended — small and maintained
FROM nginx:1.25.3           # Full Debian if you need more compatibility
```

---

## The Decision Framework

Use this decision tree when selecting a base image:

**Step 1: Is there an official image for your language/framework?**

- Yes → use it (Python, Node, Java, Go, Nginx, etc.)
- No → continue to Step 2

**Step 2: What OS compatibility do you need?**

- Need maximum compatibility → Debian slim
- Want smallest image and your app works on musl → Alpine
- Security is critical and debugging is acceptable pain → Distroless
- Static binary → scratch

**Step 3: What is the environment?**

- AWS Lambda → use the AWS base image for your language
- AWS ECS/EKS on Graviton → ensure the image supports ARM64
- Standard EC2 → any image works

**Step 4: Do you need system packages?**

- Yes, and they are in Alpine's repos → Alpine is fine
- Yes, and they are Debian-only → use Debian slim
- No system packages needed → Alpine or Distroless

---

## Comparing Variants for a Python App

```dockerfile
# Option 1: python:3.11 (full Debian)
FROM python:3.11
# Final size after your app: ~950MB
# Security vulnerabilities: Many (all Debian packages)
# Compatibility: Highest

# Option 2: python:3.11-slim (Debian slim)
FROM python:3.11-slim
# Final size after your app: ~180MB
# Security vulnerabilities: Fewer
# Compatibility: High

# Option 3: python:3.11-alpine
FROM python:3.11-alpine
# Final size after your app: ~80MB
# Security vulnerabilities: Very few
# Compatibility: Medium (musl issues possible)

# Option 4: Distroless
FROM gcr.io/distroless/python3-debian12
# Final size after your app: ~55MB
# Security vulnerabilities: Minimal
# Compatibility: No shell, harder to debug
```

For most production Python apps, `python:3.11-slim` is the right choice. It is significantly smaller than the full image, has fewer vulnerabilities, and has full glibc compatibility.

---

## Pinning Base Image Versions

Never use floating tags in production Dockerfiles. A floating tag (`python:3.11-slim` without a patch version) can change when Docker Hub updates it.

```dockerfile
# Bad — patch version can change without warning
FROM python:3.11-slim

# Better — explicit minor version
FROM python:3.11.7-slim

# Best — pinned by digest (guaranteed exact image)
FROM python@sha256:7d9f9f8a0e5e2c7f3b1a2d4e6f8a0b2d4e6f8a0b2d4e6f8a0b2d4e6f8a0b2d4e
```

To get the digest:

```bash
docker pull python:3.11.7-slim
docker inspect --format '{{index .RepoDigests 0}}' python:3.11.7-slim
```

### Using Renovate or Dependabot

For large projects, manually tracking base image versions is tedious. Use automated tools:

- **Renovate Bot** — scans your Dockerfiles and opens PRs when base image updates are available
- **AWS ECR Enhanced Scanning** — alerts when new CVEs are found in your images
- **Dependabot** (GitHub) — supports Dockerfile base image version updates

---

## Security Scanning Base Images

Before committing to a base image, scan it for known vulnerabilities:

```bash
# Using Docker Scout (built into Docker Desktop)
docker scout cves python:3.11-slim

# Using Trivy (open source scanner)
trivy image python:3.11-slim

# Sample output:
# python:3.11-slim (debian 12.4)
# 
# Total: 23 (UNKNOWN: 0, LOW: 15, MEDIUM: 5, HIGH: 3, CRITICAL: 0)
```

Compare vulnerability counts between variants:

```bash
# Full image — many vulnerabilities
trivy image --severity HIGH,CRITICAL python:3.11

# Slim — fewer
trivy image --severity HIGH,CRITICAL python:3.11-slim

# Alpine — very few
trivy image --severity HIGH,CRITICAL python:3.11-alpine
```

AWS ECR has built-in image scanning. Enable it when creating repositories:

```bash
aws ecr create-repository \
  --repository-name myapp \
  --image-scanning-configuration scanOnPush=true \
  --region us-east-1
```

ECR will automatically scan every image you push and report findings in the ECR console.

---

## Applying Security Updates to Base Images

Even with a well-chosen base image, new vulnerabilities appear over time. Apply security updates as part of your build:

```dockerfile
FROM python:3.11-slim

# Apply security updates on build
RUN apt-get update && \
    apt-get upgrade -y --no-install-recommends && \
    rm -rf /var/lib/apt/lists/*

WORKDIR /app
...
```

This adds a few seconds to the build but ensures your image has the latest security patches from the Debian repos, not just what was in the base image when it was built.

For Alpine:

```dockerfile
FROM python:3.11-alpine

RUN apk update && apk upgrade --no-cache
```

---

## Summary

- The base image you choose in `FROM` affects image size, security, compatibility, and debugging capability — it deserves careful selection
- For most production applications, use the official image for your language (python, node, java) with the `-slim` or `-alpine` variant
- Debian slim is the safe default — good compatibility, smaller than full, reasonable security
- Alpine is smallest and most secure but uses musl libc which can cause compatibility issues
- Distroless has minimal attack surface but no shell, making debugging harder
- Always pin base image versions explicitly — never use `latest` or a floating tag in production Dockerfiles
- Scan base images for vulnerabilities with Trivy or Docker Scout before adopting them, and enable ECR scanning for ongoing monitoring

In the next file, we cover caching layers strategically — how to structure your Dockerfile to maximize build cache hits and keep CI/CD pipelines fast.
