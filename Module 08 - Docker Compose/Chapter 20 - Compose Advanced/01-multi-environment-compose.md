# Multi-Environment Compose

## Introduction

Real applications run in multiple environments — local development, CI testing, staging, and production. Each environment needs the same services but different configuration: development mounts source code and enables debug tools; production uses pinned image versions, production secrets, and ships logs to CloudWatch.

This file presents a complete, practical multi-environment Compose architecture that scales from a single developer's machine to production deployments on AWS EC2.

---

## The Architecture

A well-structured multi-environment Compose project uses three layers:

```
Layer 1 (Base):    docker-compose.yml         Shared structure — service names, volumes, networks, health checks
Layer 2 (Override): docker-compose.<env>.yml  Environment-specific config — images, ports, env vars, resources
Layer 3 (Secrets):  .env                      Runtime secrets — never committed to version control
```

This is the file structure:

```
project/
├── docker-compose.yml           # base — shared structure
├── docker-compose.override.yml  # dev — auto-applied, gitignored
├── docker-compose.staging.yml   # staging
├── docker-compose.prod.yml      # production
├── docker-compose.test.yml      # CI test runs
├── docker-compose.ci.yml        # CI build and scan
├── .env.example                 # committed — placeholder values only
├── .env                         # NOT committed — real dev values (in .gitignore)
├── .env.staging                 # NOT committed — staging secrets
└── .env.production              # NOT committed — production secrets
```

---

## Layer 1: The Base File

Contains everything shared across all environments. No secrets, no image versions, no ports.

```yaml
# docker-compose.yml
version: "3.8"

services:
  api:
    restart: unless-stopped
    networks:
      - app-net
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started

  db:
    image: postgres:16
    restart: always
    networks:
      - app-net
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $$POSTGRES_USER -d $$POSTGRES_DB"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 20s

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    networks:
      - app-net
    volumes:
      - redis-data:/data
    command: redis-server --appendonly yes

volumes:
  pgdata:
  redis-data:

networks:
  app-net:
    driver: bridge
```

---

## Layer 2a: Development Override (docker-compose.override.yml)

Auto-applied in development. Adds source code mounts, debug tooling, local ports.

```yaml
# docker-compose.override.yml — auto-applied when no -f is specified
version: "3.8"

services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
      target: development
    command: python -m debugpy --listen 0.0.0.0:5678 app.py
    ports:
      - "8080:8080"
      - "5678:5678"   # debugger attach port
    volumes:
      - .:/app
      - /app/__pycache__
    environment:
      APP_ENV: development
      LOG_LEVEL: debug
      DATABASE_URL: postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@db:5432/${POSTGRES_DB}
      REDIS_URL: redis://redis:6379

  db:
    ports:
      - "127.0.0.1:5432:5432"   # expose for local DB tools
    environment:
      POSTGRES_USER: devuser
      POSTGRES_PASSWORD: devpassword
      POSTGRES_DB: myapp_dev
```

```bash
# Developer workflow
cp .env.example .env
# Edit .env with any personal overrides

docker compose up -d --build
docker compose logs -f api
```

---

## Layer 2b: Staging Override

```yaml
# docker-compose.staging.yml
version: "3.8"

services:
  api:
    image: ${ECR_REGISTRY}/myapp:${APP_VERSION}
    ports:
      - "8080:8080"
    environment:
      APP_ENV: staging
      LOG_LEVEL: info
      DATABASE_URL: postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@db:5432/${POSTGRES_DB}
      REDIS_URL: redis://redis:6379
    mem_limit: 512m
    cpus: 0.5
    logging:
      driver: json-file
      options:
        max-size: "50m"
        max-file: "3"

  db:
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    mem_limit: 512m

  redis:
    mem_limit: 128m
```

```bash
# staging/.env (not committed)
ECR_REGISTRY=123456789012.dkr.ecr.us-east-1.amazonaws.com
APP_VERSION=1.4.2
POSTGRES_USER=staginguser
POSTGRES_PASSWORD=<fetched-from-secrets-manager>
POSTGRES_DB=myapp_staging
```

```bash
# Staging deploy script
export $(cat .env.staging | xargs)
docker compose -f docker-compose.yml -f docker-compose.staging.yml up -d
```

---

## Layer 2c: Production Override

```yaml
# docker-compose.prod.yml
version: "3.8"

services:
  api:
    image: ${ECR_REGISTRY}/myapp:${APP_VERSION}
    ports:
      - "8080:8080"
    environment:
      APP_ENV: production
      LOG_LEVEL: info
      DATABASE_URL: postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@db:5432/${POSTGRES_DB}
      REDIS_URL: redis://redis:6379
      AWS_REGION: ${AWS_REGION}
      S3_BUCKET: ${S3_BUCKET}
    mem_limit: 1g
    cpus: 1.5
    logging:
      driver: awslogs
      options:
        awslogs-group: /myapp/production
        awslogs-region: us-east-1
        awslogs-stream-prefix: api
        awslogs-create-group: "true"

  db:
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    mem_limit: 4g
    logging:
      driver: awslogs
      options:
        awslogs-group: /myapp/production
        awslogs-region: us-east-1
        awslogs-stream-prefix: db
        awslogs-create-group: "true"

  redis:
    mem_limit: 512m
    logging:
      driver: awslogs
      options:
        awslogs-group: /myapp/production
        awslogs-region: us-east-1
        awslogs-stream-prefix: redis
        awslogs-create-group: "true"
```

---

## The Production Deployment Script

```bash
#!/bin/bash
# deploy-production.sh

set -euo pipefail

AWS_REGION="us-east-1"
ACCOUNT_ID="123456789012"
ECR_REGISTRY="${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
APP_VERSION="${1:?Usage: ./deploy-production.sh VERSION}"

echo "Deploying version ${APP_VERSION} to production..."

# Authenticate to ECR
aws ecr get-login-password --region $AWS_REGION | \
  docker login --username AWS --password-stdin $ECR_REGISTRY

# Pull the new image to verify it exists
docker pull ${ECR_REGISTRY}/myapp:${APP_VERSION}

# Fetch secrets from AWS Secrets Manager
POSTGRES_PASSWORD=$(aws secretsmanager get-secret-value \
  --secret-id myapp/production/postgres \
  --query 'SecretString' \
  --output text | python3 -c "import sys,json; print(json.load(sys.stdin)['password'])")

# Export all variables for Compose
export ECR_REGISTRY APP_VERSION POSTGRES_PASSWORD
export POSTGRES_USER=apiuser
export POSTGRES_DB=myapp
export AWS_REGION S3_BUCKET=myapp-uploads-production

# Validate the config before applying
echo "Validating compose config..."
docker compose -f docker-compose.yml -f docker-compose.prod.yml config --quiet

# Apply the deployment
echo "Starting deployment..."
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d

# Verify health
echo "Waiting for services to become healthy..."
sleep 10
docker compose -f docker-compose.yml -f docker-compose.prod.yml ps

echo "Deployment complete."
```

```bash
./deploy-production.sh 1.4.2
```

---

## Layer 2d: CI Test Override

```yaml
# docker-compose.test.yml
version: "3.8"

services:
  api:
    build:
      context: .
      target: test
    command: pytest tests/ -v --cov=app --cov-report=xml:/results/coverage.xml --junitxml=/results/test-report.xml
    environment:
      APP_ENV: test
      DATABASE_URL: postgresql://testuser:testpassword@db:5432/myapp_test
    volumes:
      - ./test-results:/results
    depends_on:
      db:
        condition: service_healthy

  db:
    environment:
      POSTGRES_USER: testuser
      POSTGRES_PASSWORD: testpassword
      POSTGRES_DB: myapp_test
    # No volume — throw-away DB for tests
    tmpfs:
      - /var/lib/postgresql/data  # tests run on a tmpfs DB for speed
```

```bash
# CI pipeline (GitHub Actions, GitLab CI, etc.)
docker compose -f docker-compose.yml -f docker-compose.test.yml up -d db
docker compose -f docker-compose.yml -f docker-compose.test.yml run --rm api
docker compose -f docker-compose.yml -f docker-compose.test.yml down
```

---

## Validating All Environments

Add a validation step to your CI pipeline to catch configuration errors before they reach production:

```bash
#!/bin/bash
# validate-compose.sh — run in CI

environments=("override" "staging" "prod" "test")

for env in "${environments[@]}"; do
  file="docker-compose.${env}.yml"
  if [ -f "$file" ]; then
    echo "Validating: docker-compose.yml + $file"
    docker compose -f docker-compose.yml -f "$file" config --quiet \
      && echo "  ✓ Valid" \
      || (echo "  ✗ Invalid" && exit 1)
  fi
done

echo "All environments validated."
```

---

## Summary

- A complete multi-environment architecture uses a base file (shared structure), environment-specific override files (configuration), and runtime secrets (never committed)
- `docker-compose.override.yml` is the development override — auto-applied, contains source code mounts, debug ports, and dev credentials
- Production override files contain exact image versions, resource limits, CloudWatch logging, and reference secrets from environment variables (not `.env` files)
- The deployment script fetches secrets from AWS Secrets Manager at runtime and exports them as shell variables — secrets are never written to disk
- CI uses its own override file with a throwaway database (tmpfs for speed) and the test build target
- Always run `docker compose config --quiet` as a validation step in CI before applying any deployment

This completes Module 8 on Docker Compose. You now have a complete toolkit for defining, running, and managing multi-container applications across all environments.
