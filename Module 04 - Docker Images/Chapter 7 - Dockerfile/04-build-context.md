# Build Context

## Introduction

Every time you run `docker build`, you see this line at the start:

```
Sending build context to Docker daemon  24.58kB
```

That number — the build context size — has a bigger impact on your Docker workflow than most engineers realize. An oversized build context slows down every single build, can accidentally include secrets in your images, and wastes bandwidth in CI/CD pipelines.

This file explains exactly what the build context is, what goes into it, and how to control it precisely.

---

## What is the Build Context?

The **build context** is the set of files that Docker makes available during the build process. When you run:

```bash
docker build -t myapp:1.0 .
```

The `.` is the build context — it tells Docker to use the current directory as the context. Docker packages up everything in that directory and sends it to the Docker daemon before any instructions in the Dockerfile are executed.

This is important: **the Docker daemon does not have access to your filesystem**. It only sees the files you explicitly send in the build context. This is why `COPY` instructions only work with files that are inside the build context — you cannot `COPY` a file from outside the build context directory.

```dockerfile
# This works — file is inside the build context
COPY requirements.txt .

# This fails — cannot access files outside the build context
COPY /home/user/other-project/config.yml .
```

---

## The Build Context Process

Here is what happens step by step:

1. You run `docker build -t myapp:1.0 .`
2. The Docker CLI scans the build context directory (`.`)
3. It applies the rules in `.dockerignore` to filter out excluded files
4. It packages the remaining files into a tar archive
5. It sends the tar archive to the Docker daemon over the Unix socket (or network)
6. The daemon extracts the archive to a temporary directory
7. The daemon reads the Dockerfile and executes each instruction, with access to the extracted build context

The daemon never touches your local filesystem again after receiving the context. From the daemon's perspective, the build context IS all the files in the world.

---

## Why Context Size Matters

A large build context causes real problems.

### Slow Builds

Packaging and sending the context happens before any build instruction runs — including those with cached layers. Even if every layer is cached and the build would take 0.1 seconds, a large context can take 10+ seconds to package and send.

```bash
# Large context — slow even with full cache
docker build -t myapp:1.0 .
# Sending build context to Docker daemon  985.7MB  ← takes 30+ seconds

# Small context — fast
docker build -t myapp:1.0 .
# Sending build context to Docker daemon  24.58kB  ← nearly instant
```

### Secrets in Images

If your build context includes sensitive files and your Dockerfile uses `COPY . .`, those secrets end up in the image. Even if you later delete them with a `RUN rm` command, they are still in the COPY layer.

```dockerfile
# If .env is in the build context and your .dockerignore is wrong
COPY . .   # .env and its secrets are now in the image
RUN rm .env  # too late — still in the previous layer
```

This is a critical security issue. Anyone who pulls the image can extract the secrets from the COPY layer using `docker save`.

### CI/CD Bandwidth

In CI/CD pipelines where the Docker daemon is remote (Docker-in-Docker or a remote build server), the build context is sent over the network. A 1GB build context means 1GB of data transfer on every build.

---

## Measuring Your Build Context

Before fixing the problem, measure it:

```bash
# See how much data is being sent
docker build -t myapp:1.0 . 2>&1 | head -1
# Sending build context to Docker daemon  985.7MB

# Find large files and directories
du -sh * .[^.]* 2>/dev/null | sort -rh | head -20
```

Common culprits:
- `node_modules/` — can be hundreds of MB
- `.git/` — grows with repository history
- `*.log` files — can grow large in development
- Build artifacts (`dist/`, `build/`, `__pycache__/`)
- Test data and fixtures
- Documentation (`docs/`)
- Virtual environments (`.venv/`, `venv/`)
- Infrastructure code that is not needed by the app (`terraform/`, `ansible/`)

---

## The .dockerignore File

`.dockerignore` works exactly like `.gitignore` — it tells Docker which files to exclude from the build context.

Create `.dockerignore` in the same directory as your Dockerfile:

```
# .dockerignore for a Python project

# Version control
.git
.gitignore

# Python artifacts
__pycache__
*.py[cod]
*.pyo
.pytest_cache
.mypy_cache
htmlcov/
.coverage
.coverage.*

# Virtual environments
.venv
venv
env
.env

# Secrets and config
*.env
.env.*
secrets/
*.pem
*.key

# Tests and docs (not needed in production image)
tests/
docs/
*.md
!README.md

# Build artifacts
dist/
build/

# IDE files
.vscode/
.idea/
*.swp
*.swo

# OS files
.DS_Store
Thumbs.db

# Logs
*.log
logs/

# Docker files (no need to include these in the image)
Dockerfile*
docker-compose*
.dockerignore
```

### .dockerignore for Node.js

```
.git
.gitignore
node_modules
npm-debug.log*
.npm
.env
.env.*
coverage/
.nyc_output
dist/
.vscode/
*.md
!README.md
Dockerfile*
docker-compose*
.dockerignore
tests/
__tests__/
*.test.js
*.spec.js
.DS_Store
```

### .dockerignore for a Go Project

```
.git
.gitignore
*.md
!README.md
tests/
docs/
.vscode/
.env
.env.*
Dockerfile*
docker-compose*
.dockerignore
*.exe
*.dll
*.so
*.dylib
tmp/
```

---

## .dockerignore Syntax

The syntax closely follows `.gitignore` rules:

```
# Comment

# Ignore everything in a directory
node_modules

# Ignore all files with extension
*.log
*.pyc

# Ignore a specific file
.env

# Ignore files matching a pattern in any subdirectory
**/*.pyc

# Exception — include this even if the parent was excluded
!README.md

# Ignore a file in a specific directory only
tests/*.log
```

### Exception Patterns

You can use `!` to include specific files even when the parent directory is excluded:

```
# Exclude all markdown files
*.md

# But keep README.md
!README.md
```

```
# Exclude the docs directory
docs/

# But keep one specific file from docs
!docs/api-reference.md
```

---

## Alternative Build Contexts

### Building from a Subdirectory

You can use a subdirectory as the build context while specifying a Dockerfile elsewhere:

```bash
# Build using ./backend as the context, Dockerfile is at project root
docker build -f Dockerfile -t backend:1.0 ./backend

# This is useful in monorepos
project/
├── Dockerfile.backend
├── Dockerfile.frontend
├── backend/
│   ├── app.py
│   └── requirements.txt
└── frontend/
    ├── index.html
    └── package.json

docker build -f Dockerfile.backend -t backend:1.0 ./backend
docker build -f Dockerfile.frontend -t frontend:1.0 ./frontend
```

### Building from a Git Repository

Docker can use a Git repository as the build context:

```bash
# Build from a public GitHub repository
docker build -t myapp:1.0 https://github.com/myorg/myapp.git

# Build from a specific branch
docker build -t myapp:1.0 https://github.com/myorg/myapp.git#main

# Build from a specific commit
docker build -t myapp:1.0 https://github.com/myorg/myapp.git#a1b2c3d4

# Build from a subdirectory of the repo
docker build -t myapp:1.0 https://github.com/myorg/myapp.git#main:backend
```

Docker clones the repository and uses it as the build context. This is useful for automated builds that do not have a local copy of the code.

### Building from stdin

You can pipe a Dockerfile directly to `docker build`:

```bash
# Pipe Dockerfile from stdin, use current directory as context
docker build -t myapp:1.0 -f - . <<'EOF'
FROM python:3.11-slim
WORKDIR /app
COPY . .
CMD ["python", "app.py"]
EOF

# Build with no build context (standalone Dockerfile that does not COPY anything)
docker build -t utility-image:1.0 - <<'EOF'
FROM ubuntu:22.04
RUN apt-get update && apt-get install -y curl wget jq
CMD ["bash"]
EOF
```

---

## Context in Multi-Stage Builds

In multi-stage builds, all stages share the same build context. Every file you sent to the daemon is available to every `COPY` instruction in every stage:

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /build
COPY package.json package-lock.json ./   # from build context
RUN npm ci
COPY src/ ./src/                          # from build context
RUN npm run build

FROM nginx:1.25-alpine AS production
COPY --from=builder /build/dist /usr/share/nginx/html   # from builder stage
COPY nginx.conf /etc/nginx/nginx.conf                   # from build context
```

The `COPY nginx.conf` in the final stage works because `nginx.conf` is in the build context, even though the base image is `nginx`, not `node`.

---

## Verifying Your .dockerignore is Working

After creating `.dockerignore`, verify it is doing its job:

```bash
# Build and check the context size
docker build -t myapp:1.0 . 2>&1 | head -1

# List exactly what files are included in the context
# (requires Docker 23.0+ with BuildKit)
docker build --progress=plain -t myapp:1.0 . 2>&1 | grep "COPY"

# Alternative: use a dummy Dockerfile to list context contents
cat > /tmp/test.dockerfile << 'EOF'
FROM busybox
COPY . /context
RUN find /context -type f | sort
EOF

docker build -f /tmp/test.dockerfile -t context-test . 2>&1
```

---

## Summary

- The build context is the set of files Docker packages and sends to the daemon before executing any Dockerfile instructions — the daemon cannot see your local filesystem, only what you send
- Large build contexts slow every build (even fully cached ones), can leak secrets into images, and waste CI/CD bandwidth
- `.dockerignore` filters files from the build context — it works like `.gitignore` and should always exist in your project
- Always exclude `.git`, `.env`, `node_modules`, `__pycache__`, build artifacts, tests, and documentation from the build context
- You can use a subdirectory, a Git repository URL, or stdin as the build context instead of the current directory
- In multi-stage builds, all stages share the same build context — all files sent to the daemon are available to all `COPY` instructions

In the next chapter, we move to image best practices — how to choose the right base image, write optimal layer ordering, build multi-stage images, and minimize image size for production use.
