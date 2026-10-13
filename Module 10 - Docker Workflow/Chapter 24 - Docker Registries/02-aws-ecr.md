# AWS ECR (Elastic Container Registry)

## Introduction

AWS Elastic Container Registry (ECR) is the container registry for AWS workloads. It is tightly integrated with ECS, EKS, Fargate, Lambda, CodeBuild, and IAM — making it the natural choice for any team deploying containers on AWS.

ECR eliminates the operational overhead of running your own registry while providing enterprise-grade features: IAM-based authentication, lifecycle policies for automated image cleanup, cross-region replication, image scanning, and private endpoints that keep registry traffic within the AWS network.

---

## ECR Core Concepts

**Registry** — Each AWS account has one private ECR registry per region:
```
ACCOUNT_ID.dkr.ecr.REGION.amazonaws.com
123456789012.dkr.ecr.us-east-1.amazonaws.com
```

**Repository** — A named collection of related images (equivalent to a Docker Hub repository):
```
123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp
123456789012.dkr.ecr.us-east-1.amazonaws.com/base-images/python
```

**Image** — A specific version of an image within a repository, identified by tag or SHA256 digest:
```
123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:1.4.2
123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp@sha256:a1b2c3d4...
```

---

## Creating ECR Repositories

```bash
# Create a private repository
aws ecr create-repository \
  --repository-name myapp \
  --region us-east-1

# Create with image scanning enabled on push
aws ecr create-repository \
  --repository-name myapp \
  --image-scanning-configuration scanOnPush=true \
  --image-tag-mutability IMMUTABLE \
  --region us-east-1

# Create with encryption (KMS)
aws ecr create-repository \
  --repository-name myapp \
  --encryption-configuration encryptionType=KMS,kmsKey=arn:aws:kms:us-east-1:123456789012:key/mrk-... \
  --region us-east-1
```

### Tag Mutability

```bash
# MUTABLE (default) — tags like "latest" can be overwritten
--image-tag-mutability MUTABLE

# IMMUTABLE — tags cannot be overwritten once set
# Attempting to push the same tag a second time returns an error
# Strongly recommended for production — prevents tag hijacking
--image-tag-mutability IMMUTABLE
```

With `IMMUTABLE` enabled, `latest` cannot be pushed twice — you must always use unique tags (git SHA, semantic version). This is a good forcing function for proper image versioning.

---

## Authentication

ECR uses temporary authorization tokens (valid for 12 hours). Unlike Docker Hub, there is no username/password — authentication is entirely IAM-based.

```bash
# Get an auth token and login (standard approach)
aws ecr get-login-password --region us-east-1 | \
    docker login --username AWS --password-stdin \
    123456789012.dkr.ecr.us-east-1.amazonaws.com

# Verify login worked
docker pull 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:latest
```

### Automatic Re-authentication

The token expires after 12 hours. In long-running CI environments:

```bash
# Wrapper function that re-authenticates before each operation
ecr_login() {
    aws ecr get-login-password --region "${AWS_REGION}" | \
        docker login --username AWS --password-stdin "${ECR_REGISTRY}"
}

# Call before push operations
ecr_login
docker push "${IMAGE_TAG}"
```

---

## IAM Permissions for ECR

ECR access is controlled by IAM policies. The minimum permissions for pushing images:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "ECRPushPull",
            "Effect": "Allow",
            "Action": [
                "ecr:GetAuthorizationToken"
            ],
            "Resource": "*"
        },
        {
            "Sid": "ECRPush",
            "Effect": "Allow",
            "Action": [
                "ecr:BatchCheckLayerAvailability",
                "ecr:GetDownloadUrlForLayer",
                "ecr:BatchGetImage",
                "ecr:InitiateLayerUpload",
                "ecr:UploadLayerPart",
                "ecr:CompleteLayerUpload",
                "ecr:PutImage"
            ],
            "Resource": "arn:aws:ecr:us-east-1:123456789012:repository/myapp"
        }
    ]
}
```

For ECS tasks pulling images (task execution role):

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ecr:GetAuthorizationToken",
                "ecr:BatchCheckLayerAvailability",
                "ecr:GetDownloadUrlForLayer",
                "ecr:BatchGetImage"
            ],
            "Resource": "*"
        }
    ]
}
```

This is included in the AWS-managed `AmazonECSTaskExecutionRolePolicy`.

---

## Lifecycle Policies — Automated Image Cleanup

Without lifecycle policies, ECR repositories grow indefinitely. Old images cost storage fees and make the repository hard to navigate. Lifecycle policies automatically delete images based on rules.

```bash
# Apply a lifecycle policy
aws ecr put-lifecycle-policy \
  --repository-name myapp \
  --lifecycle-policy-text file://ecr-lifecycle-policy.json
```

```json
// ecr-lifecycle-policy.json
{
    "rules": [
        {
            "rulePriority": 1,
            "description": "Keep only the last 10 tagged release images",
            "selection": {
                "tagStatus": "tagged",
                "tagPrefixList": ["v"],
                "countType": "imageCountMoreThan",
                "countNumber": 10
            },
            "action": {
                "type": "expire"
            }
        },
        {
            "rulePriority": 2,
            "description": "Delete untagged images older than 1 day",
            "selection": {
                "tagStatus": "untagged",
                "countType": "sinceImagePushed",
                "countUnit": "days",
                "countNumber": 1
            },
            "action": {
                "type": "expire"
            }
        },
        {
            "rulePriority": 3,
            "description": "Keep only last 30 any-tagged images",
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

Rules are evaluated in priority order (lowest number first). An image matching any rule is deleted.

---

## Image Scanning

ECR provides two scanning modes:

**Basic Scanning** — Free, uses Clair, scans on push. Detects CVEs from common vulnerability databases.

**Enhanced Scanning** — Uses AWS Inspector, continuous scanning (re-scans existing images as new CVEs are discovered), more accurate.

```bash
# Enable enhanced scanning on a repository
aws ecr put-image-scanning-configuration \
  --repository-name myapp \
  --image-scanning-configuration scanOnPush=true

# Check scan results
aws ecr describe-image-scan-findings \
  --repository-name myapp \
  --image-id imageTag=1.4.2 \
  --region us-east-1

# Get critical/high findings count
aws ecr describe-image-scan-findings \
  --repository-name myapp \
  --image-id imageTag=1.4.2 \
  --query 'imageScanFindings.findingSeverityCounts' \
  --output table
```

---

## Cross-Region Replication

For multi-region deployments, replicate images across regions automatically:

```bash
aws ecr put-replication-configuration \
  --replication-configuration '{
    "rules": [
      {
        "destinations": [
          {"region": "us-west-2", "registryId": "123456789012"},
          {"region": "eu-west-1", "registryId": "123456789012"}
        ],
        "repositoryFilters": [
          {"filter": "myapp", "filterType": "PREFIX_MATCH"}
        ]
      }
    ]
  }'
```

After pushing to `us-east-1`, the image is automatically replicated to `us-west-2` and `eu-west-1`. ECS in those regions can pull locally without cross-region data transfer.

---

## ECR Pull Through Cache

ECR Pull Through Cache proxies public registries (Docker Hub, Quay, GitHub GHCR) through ECR. This eliminates Docker Hub rate limits for your ECS tasks — they pull from ECR instead of Docker Hub directly.

```bash
# Create a pull-through cache rule for Docker Hub
aws ecr create-pull-through-cache-rule \
  --ecr-repository-prefix docker-hub \
  --upstream-registry-url registry-1.docker.io

# Now pull Docker Hub images via ECR
docker pull 123456789012.dkr.ecr.us-east-1.amazonaws.com/docker-hub/library/nginx:1.25
# ECR fetches from Docker Hub, caches it, and serves subsequent requests from ECR
```

---

## ECR Public Gallery

ECR Public is a public registry at `public.ecr.aws` — Amazon's alternative to Docker Hub for public images:

```bash
# Pull from ECR Public (no authentication needed)
docker pull public.ecr.aws/nginx/nginx:1.25.3

# Push to ECR Public (requires authentication to ECR Public)
aws ecr-public get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin public.ecr.aws
docker push public.ecr.aws/mycompany/myapp:1.4.2
```

ECR Public has unlimited storage and bandwidth for public images — a viable alternative to Docker Hub for open-source projects.

---

## Practical: Full ECR Setup Script

```bash
#!/bin/bash
# setup-ecr.sh — Create and configure ECR repositories for a new service

set -euo pipefail

SERVICE_NAME="${1:?Usage: ./setup-ecr.sh SERVICE_NAME}"
AWS_REGION="${AWS_REGION:-us-east-1}"
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

echo "Setting up ECR for service: ${SERVICE_NAME}"

# Create the repository
aws ecr create-repository \
  --repository-name "${SERVICE_NAME}" \
  --image-scanning-configuration scanOnPush=true \
  --image-tag-mutability IMMUTABLE \
  --region "${AWS_REGION}" 2>/dev/null || echo "Repository already exists"

# Apply lifecycle policy
cat > /tmp/lifecycle-policy.json << 'EOF'
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Keep last 20 tagged images",
      "selection": {
        "tagStatus": "tagged",
        "tagPrefixList": ["v", "sha"],
        "countType": "imageCountMoreThan",
        "countNumber": 20
      },
      "action": {"type": "expire"}
    },
    {
      "rulePriority": 2,
      "description": "Delete untagged images after 1 day",
      "selection": {
        "tagStatus": "untagged",
        "countType": "sinceImagePushed",
        "countUnit": "days",
        "countNumber": 1
      },
      "action": {"type": "expire"}
    }
  ]
}
EOF

aws ecr put-lifecycle-policy \
  --repository-name "${SERVICE_NAME}" \
  --lifecycle-policy-text file:///tmp/lifecycle-policy.json \
  --region "${AWS_REGION}"

echo "ECR repository ready:"
echo "  ${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${SERVICE_NAME}"
```

---

## Summary

- ECR is the AWS-native container registry — IAM-based authentication (no usernames/passwords), tight integration with ECS, Fargate, and CodeBuild
- Authenticate with `aws ecr get-login-password | docker login` — the token is valid for 12 hours; re-authenticate in long-running pipelines
- Use `IMMUTABLE` tag mutability for production repositories — prevents tag hijacking and forces unique image versioning
- Set lifecycle policies to automatically delete old images — without them, repositories grow indefinitely and incur unnecessary storage costs
- Enhanced Scanning (AWS Inspector) provides continuous CVE monitoring — re-scans existing images when new CVEs are published, not just on push
- ECR Pull Through Cache proxies Docker Hub through ECR — eliminates rate limits for ECS tasks and CI/CD pipelines pulling public images

In the next file, we cover private registries — self-hosted options for organizations with strict data residency requirements.
