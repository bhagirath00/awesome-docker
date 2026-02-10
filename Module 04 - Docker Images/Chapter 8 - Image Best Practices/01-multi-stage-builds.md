# Multi-Stage Builds

## Introduction

Multi-stage builds are the single most impactful technique for creating small, secure, production-ready Docker images. They solve a fundamental problem: the tools you need to build an application are not the tools you need to run it.

Before multi-stage builds existed, engineers had to choose between:
- A large image that included build tools (simple but bloated and insecure)
- Complex shell scripts that built the app outside Docker and then packaged it manually

Multi-stage builds give you the best of both worlds: build in a full environment with all the tools you need, then copy only the output to a minimal runtime image. The build tools never appear in the final image.

---

## The Problem Multi-Stage Builds Solve

Take a Go application. To build it, you need the Go compiler, the standard library, and various build utilities. At runtime, Go produces a single static binary — nothing else is needed.

**Without multi-stage builds:**

```dockerfile
FROM golang:1.21
WORKDIR /app
COPY . .
RUN go build -o server .
CMD ["./server"]
```

The resulting image:
- Includes the entire Go compiler and standard library
- Contains the Go toolchain used to build the binary
- Typically 800MB+ for a simple Go app
- Has a large attack surface (everything in the golang image)

The running container only needs the `server` binary — about 10MB. Everything else is waste.

**With multi-stage builds:**

```dockerfile
FROM golang:1.21-alpine AS builder
WORKDIR /build
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o server .

FROM scratch
COPY --from=builder /build/server /server
CMD ["/server"]
```

The resulting image:
- Contains only the `server` binary
- Approximately 10-15MB
- No compiler, no shell, no package manager
- Minimal attack surface

Same functionality. 50x smaller image.

---

## How Multi-Stage Builds Work

A Dockerfile with multiple `FROM` instructions creates multiple build stages. Each stage:
- Starts fresh from its own base image
- Can copy files from previous stages using `COPY --from=STAGE_NAME`
- Only the final stage becomes the output image

```dockerfile
# Stage 1: named "builder"
FROM python:3.11 AS builder
# ... install tools, compile, build

# Stage 2: named "production"
FROM python:3.11-slim AS production
# COPY --from=builder copies from stage 1
COPY --from=builder /app/compiled /app/
```

Intermediate stages exist only during the build. They are not part of the final image. Docker may cache them locally for future builds but they are not pushed to registries.

---

## Multi-Stage Build for Python

Python applications do not compile to a binary, but multi-stage builds are still useful for separating build-time dependencies from runtime.

```dockerfile
# Stage 1: Build — install all dependencies including dev tools
FROM python:3.11-slim AS builder

# Install build dependencies (gcc, libpq-dev for psycopg2)
RUN apt-get update && \
    apt-get install -y --no-install-recommends gcc libpq-dev && \
    rm -rf /var/lib/apt/lists/*

WORKDIR /build

COPY requirements.txt .

# Install packages to a user-local directory
RUN pip install --no-cache-dir --user -r requirements.txt

# Stage 2: Production — minimal runtime image, no build tools
FROM python:3.11-slim AS production

WORKDIR /app

# Copy only the installed packages from builder
COPY --from=builder /root/.local /root/.local

# Copy application code
COPY src/ ./src/
COPY config/ ./config/

# Create non-root user
RUN groupadd --gid 1001 appgroup && \
    useradd --uid 1001 --gid appgroup --no-create-home appuser

USER appuser

ENV PATH=/root/.local/bin:$PATH

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1

CMD ["python", "src/app.py"]
```

The production stage does not have `gcc` or `libpq-dev` — they were only needed to compile the Python C extensions. The final image is smaller and has fewer vulnerabilities.

---

## Multi-Stage Build for Node.js

React and Vue frontends are compiled during build. The output is static files that can be served by nginx — no Node.js runtime needed in production.

```dockerfile
# Stage 1: Build the React/Vue application
FROM node:20-alpine AS builder

WORKDIR /build

# Install dependencies
COPY package.json package-lock.json ./
RUN npm ci

# Copy source and build
COPY . .
RUN npm run build

# Stage 2: Serve with nginx
FROM nginx:1.25-alpine AS production

# Remove default nginx page
RUN rm -rf /usr/share/nginx/html/*

# Copy built assets from builder
COPY --from=builder /build/dist /usr/share/nginx/html

# Copy nginx configuration
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD curl -f http://localhost/ || exit 1

CMD ["nginx", "-g", "daemon off;"]
```

The production image is `nginx:alpine` with your static files — approximately 30MB. Compared to a Node.js image serving the same files (~200MB), it is a dramatic improvement.

---

## Multi-Stage Build for Java

Java applications need the JDK to compile and the JRE to run. Never include the full JDK in your production image.

```dockerfile
# Stage 1: Compile with full JDK
FROM eclipse-temurin:21-jdk-alpine AS builder

WORKDIR /build

# Copy Maven wrapper and POM
COPY mvnw pom.xml ./
COPY .mvn/ .mvn/

# Download dependencies (cached separately)
RUN ./mvnw dependency:go-offline -B

# Copy source and build
COPY src/ ./src/
RUN ./mvnw package -DskipTests -B

# Stage 2: Run with minimal JRE
FROM eclipse-temurin:21-jre-alpine AS production

WORKDIR /app

# Create non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# Copy only the compiled jar
COPY --from=builder /build/target/app.jar ./app.jar

USER appuser

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
  CMD curl -f http://localhost:8080/actuator/health || exit 1

CMD ["java", "-jar", "app.jar"]
```

JDK is ~400MB. JRE is ~200MB. Your production image contains only the JRE and your jar — not the full build toolchain.

---

## Targeting Specific Stages

Multi-stage builds are excellent for creating different images from the same Dockerfile — development, testing, and production.

```dockerfile
FROM python:3.11-slim AS base
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

FROM base AS development
# Add development tools — debugger, test runner, linting
RUN pip install --no-cache-dir \
  debugpy \
  pytest \
  pytest-cov \
  flake8 \
  black
COPY . .
CMD ["python", "-m", "debugpy", "--listen", "0.0.0.0:5678", "--wait-for-client", "app.py"]

FROM base AS test
COPY requirements-test.txt .
RUN pip install --no-cache-dir -r requirements-test.txt
COPY . .
CMD ["python", "-m", "pytest", "tests/", "-v"]

FROM base AS production
# Non-root user for security
RUN groupadd --gid 1001 appgroup && \
    useradd --uid 1001 --gid appgroup --no-create-home appuser
COPY --chown=appuser:appgroup . .
USER appuser
EXPOSE 8080
CMD ["python", "app.py"]
```

Build each stage separately:

```bash
# Development image — includes debugger
docker build --target development -t myapp:dev .

# Test image — includes test runner
docker build --target test -t myapp:test .

# Run tests
docker run --rm myapp:test

# Production image — minimal, secure
docker build --target production -t myapp:1.0 .
```

In your CI/CD pipeline:

```bash
# Build test image and run tests
docker build --target test -t myapp:test .
docker run --rm myapp:test

# If tests pass, build production image
docker build --target production -t myapp:1.0 .
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:1.0
```

---

## Copying from External Images

`COPY --from` is not limited to stages in the same Dockerfile. You can copy from any image:

```dockerfile
FROM python:3.11-slim

# Copy the AWS CLI binary from the official AWS CLI image
COPY --from=amazon/aws-cli:latest /usr/local/bin/aws /usr/local/bin/aws
COPY --from=amazon/aws-cli:latest /usr/local/lib/aws-cli /usr/local/lib/aws-cli

# Copy the kubectl binary from bitnami image
COPY --from=bitnami/kubectl:1.28 /opt/bitnami/kubectl/bin/kubectl /usr/local/bin/kubectl

COPY . /app
WORKDIR /app
CMD ["python", "app.py"]
```

This is how you include specific tools without installing their entire dependency chain.

---

## Multi-Stage Build for Go with Distroless

The most minimal possible image — a static Go binary in a distroless container:

```dockerfile
FROM golang:1.21-alpine AS builder

WORKDIR /build

COPY go.mod go.sum ./
RUN go mod download

COPY . .

RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build \
    -ldflags="-w -s" \
    -o server .

FROM gcr.io/distroless/static-debian12

COPY --from=builder /build/server /server

USER nonroot:nonroot

EXPOSE 8080

CMD ["/server"]
```

The final image is approximately 10MB and contains:
- The Go binary
- SSL certificates (from distroless)
- Nothing else — no shell, no package manager, no utilities

---

## Summary

- Multi-stage builds let you use heavy build tools in one stage and copy only the output to a minimal runtime image — the build tools never appear in the final image
- Use `FROM ... AS stage_name` to name stages and `COPY --from=stage_name` to copy files between stages
- For Go: build in the golang image, copy the static binary to scratch or distroless
- For Node.js frontends: build with node, serve static files with nginx — dramatic size reduction
- For Java: compile with JDK, run with JRE — avoids shipping the compiler with your app
- For Python: use one stage to compile C extensions, copy installed packages to a clean slim image without build tools
- Use `--target STAGE_NAME` to build specific stages — enables different development, test, and production images from a single Dockerfile

In the next file, we cover additional techniques for reducing image size — removing unnecessary files, using lighter alternatives, and measuring the impact of your changes.
