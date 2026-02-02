# Docker Registries

## Introduction

Every time you run `docker pull nginx` or `docker push myapp:1.0`, you are interacting with a **Docker registry**. Registries are the storage and distribution system for Docker images — they are to Docker what GitHub is to code.

Understanding registries is essential for real DevOps work. You need to know how to pull images, push your own images, authenticate with private registries, and choose the right registry for your organization. This is especially important on AWS, where most production images are stored in **Amazon ECR (Elastic Container Registry)**.

---

## What is a Docker Registry?

A **Docker registry** is a storage system for Docker images. It organizes images into **repositories** — one repository per application — and each repository can hold multiple **tagged versions** of that application's image.

The hierarchy works like this:

```
Registry
└── Repository (one per application)
    ├── Image Tag: myapp:1.0
    ├── Image Tag: myapp:1.1
    ├── Image Tag: myapp:2.0
    └── Image Tag: myapp:latest
```

Think of the registry as a library. Repositories are the bookshelves. Image tags are the individual books (specific editions of your application).

---

## Types of Docker Registries

### Public Registries

Public registries allow anyone to pull images. They are the source of the official base images you use in your Dockerfiles.

**Docker Hub** (`docker.io`) is the most popular public registry. It is the default registry Docker uses when you do not specify one.

```bash
# These two commands are identical — Docker Hub is the default
docker pull nginx
docker pull docker.io/library/nginx:latest
```

Docker Hub provides:
- **Official images** — maintained by Docker and verified software vendors (nginx, postgres, python, node, redis, etc.)
- **Verified publisher images** — from companies like AWS, HashiCorp, Microsoft
- **Community images** — from individual developers and organizations

```bash
# Pull official images
docker pull python:3.11-slim
docker pull postgres:16
docker pull redis:7-alpine
docker pull node:20-alpine

# Pull a vendor image
docker pull hashicorp/terraform:1.6.6
docker pull amazon/aws-cli:latest
```

### Private Registries

Private registries restrict access — only authenticated users or services can push or pull images. Organizations use private registries to:

- Keep proprietary application images secure
- Control access to who can deploy what
- Keep images close to where they are deployed (reducing latency and data transfer costs)

The major private registry options are:

| Registry | Provider | Best for |
|---|---|---|
| **AWS ECR** | Amazon Web Services | Teams using AWS for deployment |
| **Google Artifact Registry** | Google Cloud | Teams using GCP |
| **Azure Container Registry** | Microsoft Azure | Teams using Azure |
| **Docker Hub (private)** | Docker | Small teams, simple setup |
| **GitHub Container Registry** | GitHub | Teams using GitHub Actions |
| **GitLab Container Registry** | GitLab | Teams using GitLab CI |
| **Harbor** | Open source | Self-hosted, on-premises |

---

## Docker Hub in Practice

Docker Hub is where you start when learning Docker. Even in production, you will always be pulling base images from Docker Hub.

### Understanding Image Names on Docker Hub

Docker Hub image names follow this format:

```
[registry/][username/]repository[:tag]
```

- `nginx` → Docker Hub, official image, latest tag → `docker.io/library/nginx:latest`
- `python:3.11-slim` → Docker Hub, official image, tag 3.11-slim → `docker.io/library/python:3.11-slim`
- `myusername/myapp:1.0` → Docker Hub, user image, tag 1.0 → `docker.io/myusername/myapp:1.0`

### Pulling Images from Docker Hub

```bash
# Pull the latest official nginx image
docker pull nginx

# Pull a specific version (always do this in production)
docker pull nginx:1.25.3

# Pull a specific architecture variant
docker pull nginx:1.25.3-alpine

# Pull and verify the image exists locally
docker images nginx
```

### Pushing Images to Docker Hub

```bash
# Step 1: Log in to Docker Hub
docker login
# Enter your Docker Hub username and password when prompted

# Step 2: Tag your image with your Docker Hub username
docker tag myapp:1.0 yourusername/myapp:1.0

# Step 3: Push the image
docker push yourusername/myapp:1.0

# Step 4: Verify it is accessible
docker pull yourusername/myapp:1.0
```

---

## AWS ECR — The Production Standard on AWS

If you are deploying to AWS (ECS, EKS, or Fargate), **Amazon ECR (Elastic Container Registry)** is the standard place to store your images.

ECR is fully integrated with AWS:
- AWS IAM controls access — no separate user management
- Images stay in your AWS region — no data transfer costs when deploying to ECS or EKS in the same region
- ECR is highly available and scales automatically
- Images are encrypted at rest using AWS KMS
- Lifecycle policies automatically clean up old images to reduce storage costs

### Creating an ECR Repository

```bash
# Using AWS CLI
aws ecr create-repository \
  --repository-name myapp \
  --region us-east-1 \
  --image-scanning-configuration scanOnPush=true \
  --encryption-configuration encryptionType=AES256

# Output:
{
    "repository": {
        "repositoryArn": "arn:aws:ecr:us-east-1:123456789012:repository/myapp",
        "registryId": "123456789012",
        "repositoryName": "myapp",
        "repositoryUri": "123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp",
        "createdAt": "2024-01-15T09:23:41.000Z"
    }
}
```

The `repositoryUri` is the address you use to push and pull images.

### Authenticating with ECR

ECR uses AWS IAM authentication. You get a temporary password token that lasts 12 hours.

```bash
# Authenticate Docker to your ECR registry
aws ecr get-login-password --region us-east-1 | \
  docker login \
  --username AWS \
  --password-stdin \
  123456789012.dkr.ecr.us-east-1.amazonaws.com

# Output:
Login Succeeded
```

### Pushing an Image to ECR

```bash
# Step 1: Build your image
docker build -t myapp:1.0 .

# Step 2: Tag it with the full ECR URI
docker tag myapp:1.0 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:1.0

# Step 3: Push to ECR
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:1.0

# Output:
The push refers to repository [123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp]
4a5e7c3d: Pushed
b2f3e1a4: Pushed
1.0: digest: sha256:abc123... size: 1234
```

### Pulling an Image from ECR

```bash
# Authenticate first (if not already done)
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789012.dkr.ecr.us-east-1.amazonaws.com

# Pull the image
docker pull 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:1.0
```

### Listing Images in ECR

```bash
# List all images in a repository
aws ecr list-images \
  --repository-name myapp \
  --region us-east-1

# Describe images with details
aws ecr describe-images \
  --repository-name myapp \
  --region us-east-1
```

---

## ECR Lifecycle Policies

Without lifecycle policies, every image you push stays in ECR forever, increasing storage costs. Lifecycle policies automatically remove old images.

A practical policy: keep only the last 30 images:

```json
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Keep only the last 30 images",
      "selection": {
        "tagStatus": "any",
        "countType": "imageCountMoreThan",
        "countNumber": 30
      },
      "action": {
        "type": "expire"
      }
    }
  ]
}
```

Apply the policy:

```bash
aws ecr put-lifecycle-policy \
  --repository-name myapp \
  --lifecycle-policy-text file://lifecycle-policy.json \
  --region us-east-1
```

---

## ECR in a CI/CD Pipeline

Here is a complete example of using ECR in a GitHub Actions pipeline:

```yaml
name: Build and Push to ECR

on:
  push:
    branches: [main]

env:
  AWS_REGION: us-east-1
  ECR_REPOSITORY: myapp
  ECR_REGISTRY: 123456789012.dkr.ecr.us-east-1.amazonaws.com

jobs:
  build-and-push:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build, tag, and push image to Amazon ECR
        env:
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
          docker tag $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG $ECR_REGISTRY/$ECR_REPOSITORY:latest
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:latest
          echo "Image pushed: $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG"
```

This pipeline:
1. Builds a Docker image on every push to `main`
2. Tags it with both the Git commit SHA and `latest`
3. Pushes both tags to ECR

The commit SHA tag means you can always trace exactly which code version is running in production.

---

## Image Tags — Best Practices

Tags are how you version your images. Bad tagging practices cause serious problems in production.

### Never Rely on `latest` in Production

The `latest` tag is mutable — it can point to a different image every time you push. If you deploy using `latest`, you do not know exactly what is running.

```bash
# Bad — which version is actually running?
docker run 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:latest

# Good — you know exactly what is running
docker run 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:a1b2c3d4
```

### Good Tagging Strategies

```bash
# Use Git commit SHA for exact traceability
docker tag myapp:build 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:${GIT_COMMIT}

# Use semantic versioning for releases
docker tag myapp:build 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:1.4.2

# Use build number from CI system
docker tag myapp:build 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:build-${BUILD_NUMBER}

# Tag multiple versions for the same image
docker tag myapp:build 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:1.4.2
docker tag myapp:build 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:1.4
docker tag myapp:build 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:latest
```

---

## Running a Private Registry Locally

For development or air-gapped environments, you can run your own private registry using Docker itself:

```bash
# Start a local registry on port 5000
docker run -d \
  --name registry \
  --restart always \
  -p 5000:5000 \
  -v registry-data:/var/lib/registry \
  registry:2

# Push an image to the local registry
docker tag myapp:1.0 localhost:5000/myapp:1.0
docker push localhost:5000/myapp:1.0

# Pull from the local registry
docker pull localhost:5000/myapp:1.0
```

This is useful for:
- Local testing of your push/pull workflow
- Development environments without internet access
- Caching frequently used base images to reduce Docker Hub pull rate limits

---

## Summary

- A Docker registry stores and distributes Docker images, organized into repositories with tagged versions
- Docker Hub is the default public registry — it is the source of official base images like `nginx`, `python`, and `postgres`
- AWS ECR is the standard private registry for AWS-based deployments — it integrates with IAM, ECS, EKS, and Fargate
- To push to ECR, authenticate with `aws ecr get-login-password`, tag your image with the full ECR URI, then push
- Always apply ECR lifecycle policies to automatically remove old images and control storage costs
- In CI/CD pipelines, tag images with the Git commit SHA for full traceability — never depend on `latest` in production
- You can run your own private registry locally using the official `registry:2` Docker image

In the next module, we move from architecture to practice — installing Docker on Linux, macOS, and Windows so you can start running real containers.
