# Docker with GitHub Actions

## Introduction

GitHub Actions is the most widely adopted CI/CD platform for teams using GitHub. Its tight integration with repositories, large marketplace of pre-built actions, and free tier make it the default choice for open-source and many enterprise projects.

This file covers building a production-grade Docker CI/CD pipeline with GitHub Actions — from a first commit to an image deployed on AWS ECS.

---

## How GitHub Actions Works with Docker

GitHub Actions runners (the machines that execute workflows) come pre-installed with Docker. When your workflow runs `docker build`, it uses the Docker daemon on the runner VM directly — no Docker-in-Docker complexity.

```yaml
runs-on: ubuntu-latest   # Ubuntu 22.04 with Docker 24+ pre-installed
```

Key GitHub Actions concepts for Docker:
- **Workflow** — the YAML file defining your pipeline (`.github/workflows/`)
- **Job** — a group of steps that run on the same runner
- **Step** — a single command or action
- **Action** — a reusable unit of work (from the GitHub Marketplace or your repo)
- **Secret** — encrypted repository variable (used for credentials)

---

## Basic Build and Push to ECR

```yaml
# .github/workflows/docker.yml
name: Docker Build and Push

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  AWS_REGION: us-east-1
  ECR_REPOSITORY: myapp
  REGISTRY: 123456789012.dkr.ecr.us-east-1.amazonaws.com

jobs:
  build-and-push:
    name: Build, Test, and Push
    runs-on: ubuntu-latest

    steps:
      # Step 1: Check out code
      - name: Checkout
        uses: actions/checkout@v4

      # Step 2: Configure AWS credentials
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      # Step 3: Log in to Amazon ECR
      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      # Step 4: Set image metadata
      - name: Set image tag
        id: meta
        run: |
          echo "sha_short=$(git rev-parse --short HEAD)" >> $GITHUB_OUTPUT
          echo "image=${{ env.REGISTRY }}/${{ env.ECR_REPOSITORY }}" >> $GITHUB_OUTPUT

      # Step 5: Build the Docker image
      - name: Build Docker image
        run: |
          docker build \
            --build-arg GIT_COMMIT=${{ steps.meta.outputs.sha_short }} \
            --build-arg BUILD_DATE=$(date -u +%Y-%m-%dT%H:%M:%SZ) \
            --build-arg APP_VERSION=${{ github.ref_name }} \
            -t ${{ steps.meta.outputs.image }}:${{ steps.meta.outputs.sha_short }} \
            -t ${{ steps.meta.outputs.image }}:latest \
            .

      # Step 6: Run tests
      - name: Run tests
        run: |
          docker run --rm \
            ${{ steps.meta.outputs.image }}:${{ steps.meta.outputs.sha_short }} \
            pytest tests/ -v

      # Step 7: Scan for vulnerabilities
      - name: Scan image with Trivy
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ steps.meta.outputs.image }}:${{ steps.meta.outputs.sha_short }}
          format: table
          exit-code: 1
          severity: CRITICAL,HIGH

      # Step 8: Push to ECR (only on main branch, not PRs)
      - name: Push to ECR
        if: github.ref == 'refs/heads/main' && github.event_name == 'push'
        run: |
          docker push ${{ steps.meta.outputs.image }}:${{ steps.meta.outputs.sha_short }}
          docker push ${{ steps.meta.outputs.image }}:latest
```

---

## Using docker/build-push-action (Recommended)

The Docker team maintains official GitHub Actions that integrate with BuildKit and provide advanced caching:

```yaml
name: Docker Build and Push

on:
  push:
    branches: [main]
  pull_request:

env:
  REGISTRY: 123456789012.dkr.ecr.us-east-1.amazonaws.com
  IMAGE_NAME: myapp

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      # Set up Docker Buildx (enables BuildKit)
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      # Configure AWS and log into ECR
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1

      - name: Login to Amazon ECR
        uses: aws-actions/amazon-ecr-login@v2

      # Generate image metadata (tags, labels)
      - name: Extract Docker metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix=,suffix=,format=short
            type=ref,event=branch
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=raw,value=latest,enable={{is_default_branch}}

      # Build and push with cache
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=registry,ref=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:cache
          cache-to: type=registry,ref=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:cache,mode=max
          build-args: |
            GIT_COMMIT=${{ github.sha }}
            BUILD_DATE=${{ github.event.head_commit.timestamp }}
```

The `docker/metadata-action` automatically generates appropriate tags based on the event type:
- On push to `main`: `latest`, `sha-a1b2c3d`
- On a tag `v1.4.2`: `1.4.2`, `1.4`, `latest`
- On a PR: only builds, does not push

---

## Complete Production Pipeline with ECS Deploy

```yaml
name: Production Deploy

on:
  push:
    tags:
      - 'v[0-9]+.[0-9]+.[0-9]+'   # trigger only on semver tags like v1.4.2

env:
  AWS_REGION: us-east-1
  ECR_REGISTRY: 123456789012.dkr.ecr.us-east-1.amazonaws.com
  ECR_REPOSITORY: myapp
  ECS_CLUSTER: production
  ECS_SERVICE: myapp-api
  CONTAINER_NAME: api

jobs:
  # Job 1: Lint
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Dockerfile lint
        uses: hadolint/hadolint-action@v3.1.0
        with:
          dockerfile: Dockerfile
          config: .hadolint.yaml

  # Job 2: Test
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build test image
        run: docker build --target test -t myapp:test .
      - name: Run unit tests
        run: |
          docker run --rm \
            -v $(pwd)/test-results:/results \
            myapp:test \
            pytest tests/unit/ -v --junitxml=/results/unit.xml
      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-results
          path: test-results/

  # Job 3: Build, Scan, Push
  build-push:
    needs: [lint, test]
    runs-on: ubuntu-latest
    outputs:
      image: ${{ steps.build.outputs.image }}

    steps:
      - uses: actions/checkout@v4

      - uses: docker/setup-buildx-action@v3

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to ECR
        uses: aws-actions/amazon-ecr-login@v2

      - name: Set image tag
        id: tag
        run: echo "version=${GITHUB_REF_NAME#v}" >> $GITHUB_OUTPUT

      - name: Build and push
        id: build
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ${{ env.ECR_REGISTRY }}/${{ env.ECR_REPOSITORY }}:${{ steps.tag.outputs.version }}
            ${{ env.ECR_REGISTRY }}/${{ env.ECR_REPOSITORY }}:latest
          cache-from: type=registry,ref=${{ env.ECR_REGISTRY }}/${{ env.ECR_REPOSITORY }}:cache
          cache-to: type=registry,ref=${{ env.ECR_REGISTRY }}/${{ env.ECR_REPOSITORY }}:cache,mode=max

      - name: Scan with Trivy
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.ECR_REGISTRY }}/${{ env.ECR_REPOSITORY }}:${{ steps.tag.outputs.version }}
          format: sarif
          output: trivy-results.sarif
          exit-code: 1
          severity: CRITICAL,HIGH

      - name: Upload SARIF to GitHub Security
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: trivy-results.sarif

  # Job 4: Deploy to ECS
  deploy:
    needs: build-push
    runs-on: ubuntu-latest
    environment: production   # requires approval in GitHub Environments

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Set version
        id: tag
        run: echo "version=${GITHUB_REF_NAME#v}" >> $GITHUB_OUTPUT

      # Update ECS task definition with the new image
      - name: Download current task definition
        run: |
          aws ecs describe-task-definition \
            --task-definition ${{ env.ECS_SERVICE }} \
            --query taskDefinition > task-definition.json

      - name: Update image in task definition
        id: task-def
        uses: aws-actions/amazon-ecs-render-task-definition@v1
        with:
          task-definition: task-definition.json
          container-name: ${{ env.CONTAINER_NAME }}
          image: ${{ env.ECR_REGISTRY }}/${{ env.ECR_REPOSITORY }}:${{ steps.tag.outputs.version }}

      - name: Deploy to ECS
        uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          task-definition: ${{ steps.task-def.outputs.task-definition }}
          service: ${{ env.ECS_SERVICE }}
          cluster: ${{ env.ECS_CLUSTER }}
          wait-for-service-stability: true
```

---

## Using OIDC Instead of Long-Lived Access Keys (Security Best Practice)

Instead of storing long-lived `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` as secrets, use GitHub's OIDC provider to assume an IAM role:

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      id-token: write    # required for OIDC
      contents: read

    steps:
      - name: Configure AWS credentials via OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsDeployRole
          aws-region: us-east-1
          # No access key or secret key needed — OIDC token is used
```

The IAM role has a trust policy that allows GitHub Actions from your specific repository to assume it:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:sub": "repo:myorg/myapp:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

---

## Matrix Builds — Testing Multiple Versions

```yaml
jobs:
  test:
    strategy:
      matrix:
        python-version: ["3.10", "3.11", "3.12"]

    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Build for Python ${{ matrix.python-version }}
        run: |
          docker build \
            --build-arg PYTHON_VERSION=${{ matrix.python-version }} \
            -t myapp:test-py${{ matrix.python-version }} .

      - name: Test Python ${{ matrix.python-version }}
        run: docker run --rm myapp:test-py${{ matrix.python-version }} pytest tests/
```

---

## Summary

- GitHub Actions runners come with Docker pre-installed — no Docker-in-Docker needed
- Use `docker/setup-buildx-action` to enable BuildKit and `docker/build-push-action` for registry caching with `cache-from`/`cache-to`
- `docker/metadata-action` automatically generates appropriate tags based on git event type — SHAs for commits, semver tags for releases, `latest` for main branch pushes
- Use OIDC (`role-to-assume`) instead of long-lived access keys for AWS authentication — eliminates the need to rotate IAM credentials in GitHub Secrets
- Upload Trivy SARIF results to the GitHub Security tab for PR-level vulnerability visibility
- Use `environment: production` with required reviewers for production deployments — adds a manual approval gate before the deploy job runs

In the next file, we build the equivalent pipeline using Jenkins — the most common CI platform in enterprise environments.
