# Image Scanning

## Introduction

Every Docker image you build contains software — an OS, runtimes, libraries, and your application code. Any of this software can have known security vulnerabilities (CVEs — Common Vulnerabilities and Exposures). Image scanning is the process of analyzing your image's contents against a database of known CVEs and reporting what is found.

In a production DevOps workflow, image scanning is not optional. Running unscanned images in production means you are potentially running software with known exploits. This file covers how image scanning works, how to use the main tools, and how to integrate scanning into your CI/CD pipeline on AWS.

---

## What Image Scanning Finds

An image scanner:

1. Extracts the list of all software packages in the image — OS packages (`dpkg`, `rpm`, `apk`), language packages (`pip`, `npm`, `gem`, `cargo`), and binaries
2. Looks up each package and version against a CVE database (NVD, Red Hat, Ubuntu, Debian, GitHub Advisory DB, etc.)
3. Reports findings by severity: CRITICAL, HIGH, MEDIUM, LOW, INFORMATIONAL

A finding looks like this:

```
CVE-2023-29491
Package:   ncurses
Version:   6.3-2
Severity:  HIGH
Fix:       Upgrade to 6.4-1
Description: Local users can trigger security-relevant memory corruption
             via malformed data in a terminfo database file found in $HOME/.terminfo
```

The scanner tells you:
- What package has the vulnerability
- How severe it is
- Whether a fixed version is available

---

## When to Scan

**In the CI/CD pipeline** — scan every image before it is pushed to the registry. Block deployments of images with CRITICAL or HIGH vulnerabilities that have a fix available.

**In the registry** — AWS ECR can scan images when they are pushed and continuously re-scan as new CVEs are discovered. A vulnerability found today might not have existed when you built the image 2 months ago.

**Before deployments** — before promoting an image from staging to production, confirm its scan results are clean.

---

## Tool 1: AWS ECR Image Scanning

AWS ECR has built-in image scanning at two levels:

### Basic Scanning (Free)

Uses the open-source Clair scanner. Scans OS packages.

```bash
# Enable basic scanning on push when creating a repository
aws ecr create-repository \
  --repository-name myapp \
  --image-scanning-configuration scanOnPush=true \
  --region us-east-1

# Enable scanning on an existing repository
aws ecr put-image-scanning-configuration \
  --repository-name myapp \
  --image-scanning-configuration scanOnPush=true \
  --region us-east-1
```

### Enhanced Scanning (AWS Inspector — Paid)

Uses AWS Inspector with a continuously updated database. Scans OS packages AND application packages (pip, npm, gem, etc.). Re-scans images automatically when new CVEs are discovered.

Enable via the ECR console: **Private registry → Scanning → Enhanced scanning**.

### Viewing Scan Results

```bash
# Get scan findings for a specific image
aws ecr describe-image-scan-findings \
  --repository-name myapp \
  --image-id imageTag=1.4.2 \
  --region us-east-1

# Get a summary of findings severity
aws ecr describe-image-scan-findings \
  --repository-name myapp \
  --image-id imageTag=1.4.2 \
  --region us-east-1 \
  --query 'imageScanFindings.findingSeverityCounts'

# Output:
{
    "CRITICAL": 0,
    "HIGH": 2,
    "MEDIUM": 15,
    "LOW": 45,
    "INFORMATIONAL": 12
}
```

### Triggering a Manual Scan

```bash
# Trigger a scan of a specific image
aws ecr start-image-scan \
  --repository-name myapp \
  --image-id imageTag=1.4.2 \
  --region us-east-1
```

---

## Tool 2: Trivy (Open Source)

Trivy is the most widely used open-source container security scanner. It is fast, accurate, and easy to integrate.

```bash
# Install on Linux
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin

# Install on macOS
brew install trivy

# Scan a local image
trivy image myapp:1.4.2

# Scan a remote image from ECR
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com
trivy image 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:1.4.2

# Scan and show only HIGH and CRITICAL
trivy image --severity HIGH,CRITICAL myapp:1.4.2

# Scan and fail the command if CRITICAL vulnerabilities found (for CI/CD)
trivy image --severity CRITICAL --exit-code 1 myapp:1.4.2
```

### Trivy Output

```
myapp:1.4.2 (debian 12.4)

Total: 8 (HIGH: 3, CRITICAL: 2, MEDIUM: 3)

┌──────────────────────┬────────────────┬──────────┬──────────────────┬──────────────────┬─────────────────────────────────────┐
│       Library        │ Vulnerability  │ Severity │ Installed Version│  Fixed Version   │                Title                │
├──────────────────────┼────────────────┼──────────┼──────────────────┼──────────────────┼─────────────────────────────────────┤
│ openssl              │ CVE-2023-5678  │ CRITICAL │ 3.0.11-1         │ 3.0.12-1         │ OpenSSL: excessive time spent...    │
│ libcurl4             │ CVE-2023-46218 │ HIGH     │ 7.88.1-10        │ 7.88.1-10+deb12u5│ curl: information disclosure...     │
└──────────────────────┴────────────────┴──────────┴──────────────────┴──────────────────┴─────────────────────────────────────┘
```

### Scanning the Filesystem

Trivy can also scan your application dependencies before building the image:

```bash
# Scan Python dependencies
trivy fs --security-checks vuln requirements.txt

# Scan Node.js dependencies
trivy fs --security-checks vuln package-lock.json

# Scan the current directory (all supported files)
trivy fs --security-checks vuln .
```

Scanning dependencies before the build gives you faster feedback — you find issues during development rather than after pushing to the registry.

---

## Tool 3: Docker Scout

Docker Scout is Docker's built-in scanning tool, available in Docker Desktop and Docker Hub.

```bash
# Scan an image with Docker Scout
docker scout cves myapp:1.4.2

# Show only CRITICAL and HIGH
docker scout cves --only-severity critical,high myapp:1.4.2

# Compare two versions to see what changed
docker scout compare myapp:1.4.1 myapp:1.4.2

# Quickview — summary dashboard
docker scout quickview myapp:1.4.2
```

Docker Scout is particularly useful for understanding the difference in vulnerability exposure between two image versions.

---

## Integrating Scanning into CI/CD

The right place for scanning is in the CI/CD pipeline — between building and pushing.

```yaml
# .github/workflows/build-and-scan.yml
name: Build, Scan, and Push

on:
  push:
    branches: [main]

jobs:
  build-scan-push:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1

      - name: Login to ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build image
        env:
          ECR: ${{ steps.login-ecr.outputs.registry }}
        run: |
          docker build -t $ECR/myapp:${{ github.sha }} .

      - name: Install Trivy
        run: |
          curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin

      - name: Scan image for vulnerabilities
        env:
          ECR: ${{ steps.login-ecr.outputs.registry }}
        run: |
          trivy image \
            --severity CRITICAL,HIGH \
            --exit-code 1 \
            --format table \
            $ECR/myapp:${{ github.sha }}
        continue-on-error: false  # fail the pipeline on critical/high CVEs

      - name: Push image (only if scan passed)
        env:
          ECR: ${{ steps.login-ecr.outputs.registry }}
        run: |
          docker push $ECR/myapp:${{ github.sha }}
          docker tag $ECR/myapp:${{ github.sha }} $ECR/myapp:latest
          docker push $ECR/myapp:latest
```

With this pipeline:
- Images with CRITICAL or HIGH vulnerabilities that have fixes available are blocked from being pushed
- The build fails with a clear message showing which CVEs need to be fixed
- Only clean images make it to the registry and can be deployed

---

## Responding to Scan Findings

When a scan finds vulnerabilities, you have several options:

### Fix 1: Update the Base Image

The most common fix — a newer base image version has the security patch.

```dockerfile
# Old — vulnerable
FROM python:3.11.5-slim

# Updated — vulnerability patched
FROM python:3.11.7-slim
```

Rebuild and rescan to verify the CVE is gone.

### Fix 2: Update System Packages in the Dockerfile

```dockerfile
FROM python:3.11-slim

# Apply available security patches during build
RUN apt-get update && \
    apt-get upgrade -y --no-install-recommends && \
    rm -rf /var/lib/apt/lists/*
```

This applies all available patches from the Debian repos at build time, even if the base image has not been updated yet.

### Fix 3: Update Application Dependencies

If the vulnerability is in a Python, Node.js, or other language package:

```bash
# Update the vulnerable package
pip install --upgrade vulnerable-package==2.0.0

# Update requirements.txt
pip freeze > requirements.txt
```

### Fix 4: Accept the Risk (Intentional Exception)

Sometimes a CVE has no fix yet, or the fix requires a breaking change. If you have reviewed the vulnerability and determined it is not exploitable in your use case, you can configure the scanner to ignore it:

```bash
# Trivy — ignore specific CVEs with a .trivyignore file
cat .trivyignore
# CVE-2023-12345  # Not exploitable in our deployment context

trivy image --ignorefile .trivyignore myapp:1.4.2
```

Document why each exception is acceptable. Never ignore CVEs without understanding and documenting the justification.

---

## Scanning on a Schedule

New CVEs are published daily. An image that was clean last month might have HIGH vulnerabilities today. Set up scheduled scans:

```yaml
# .github/workflows/scheduled-scan.yml
name: Scheduled Security Scan

on:
  schedule:
    - cron: '0 6 * * 1'   # Every Monday at 6am UTC

jobs:
  scan-production-image:
    runs-on: ubuntu-latest
    steps:
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1

      - name: Login to ECR
        uses: aws-actions/amazon-ecr-login@v2

      - name: Install Trivy
        run: curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin

      - name: Scan production image
        run: |
          trivy image \
            --severity CRITICAL,HIGH \
            --exit-code 1 \
            123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:latest
```

If new CVEs are found in the production image, the scheduled scan fails and alerts your team. You then rebuild with updated base images or packages to resolve the findings.

---

## Summary

- Image scanning detects known CVEs (security vulnerabilities) in OS packages and application dependencies within your Docker images
- AWS ECR has built-in scanning — enable `scanOnPush=true` on every repository for automatic scanning after each push
- Trivy is the best open-source scanner — it scans OS packages AND language packages, runs in CI/CD, and exits with code 1 on findings (useful for blocking deployments)
- Integrate Trivy into your CI/CD pipeline between the build and push steps — block images with CRITICAL or HIGH vulnerabilities from reaching the registry
- When a CVE is found, fix it by updating the base image, running `apt-get upgrade`, or updating the vulnerable dependency
- Set up weekly scheduled scans of production images — new CVEs are published daily and a clean image from last month may have new findings today

In the next module, we move into Docker containers in depth — advanced management, runtime configuration, and patterns for production container operations.
