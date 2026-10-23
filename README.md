<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="https://cdn.simpleicons.org/docker/white" />
    <img alt="Docker Logo" src="https://cdn.simpleicons.org/docker/black" width="60" />
  </picture>
</p>

<h1 align="center">Awesome Docker</h1>

<p align="center">
  <a href="https://awesome.re"><picture><source media="(prefers-color-scheme: dark)" srcset="https://shieldcn.dev/badge/awesome.svg?variant=outline&amp;mode=dark&amp;font=geist&amp;logo=awesomelists" /><img alt="awesome" src="https://shieldcn.dev/badge/awesome.svg?variant=outline&amp;mode=light&amp;font=geist&amp;logo=awesomelists" /></picture></a>
  <a href="https://docs.docker.com/"><picture><source media="(prefers-color-scheme: dark)" srcset="https://shieldcn.dev/badge/Docker-26+.svg?variant=outline&amp;mode=dark&amp;font=geist&amp;logo=docker" /><img alt="Docker" src="https://shieldcn.dev/badge/Docker-26+.svg?variant=outline&amp;mode=light&amp;font=geist&amp;logo=docker" /></picture></a>
  <a href="LICENSE"><picture><source media="(prefers-color-scheme: dark)" srcset="https://shieldcn.dev/badge/license-MIT.svg?variant=outline&amp;mode=dark&amp;font=geist" /><img alt="license" src="https://shieldcn.dev/badge/license-MIT.svg?variant=outline&amp;mode=light&amp;font=geist" /></picture></a>
  <a href="https://github.com/bhagirath00/awesome-docker"><picture><source media="(prefers-color-scheme: dark)" srcset="https://shieldcn.dev/views/repo/bhagirath00/awesome-docker.svg?variant=outline&amp;mode=dark&amp;font=geist" /><img alt="views" src="https://shieldcn.dev/views/repo/bhagirath00/awesome-docker.svg?variant=outline&amp;mode=light&amp;font=geist" /></picture></a>
</p>

## Table of Contents

- [Introduction](#introduction)
- [Why Docker for DevOps?](#why-docker-for-devops)
- [Learning Modules](#learning-modules)
  - [Core Containerization](#core-containerization)
  - [Storage, Networking & Optimization](#storage-networking--optimization)
  - [Orchestration & Production Workflows](#orchestration--production-workflows)
- [Resources](#resources)
- [Contributing](#contributing)
- [Security](#security)
- [License](#license)

---

## Introduction

Welcome to **Awesome Docker**! This repository serves as an exhaustive, hands-on architectural roadmap and reference manual designed for site reliability engineers (SREs), DevOps practitioners, and software engineers mastering containerization.

> **"Build once, test anywhere, deploy everywhere."** — The core promise of modern container engineering.

---

## Why Docker for DevOps?

Docker is the foundational runtime standard powering cloud-native software delivery:

| Capability | Importance in DevOps |
| :--- | :--- |
| **Hermetic Environments** | Eliminates "works on my machine" discrepancies between development, CI/CD, and production |
| **Lightweight Isolation** | Sub-second process initialization sharing the host Linux kernel without VM hypervisor overhead |
| **Deterministic Artifacts** | Immutable container images versioned and distributed via OCI registries |
| **Resource Efficiency** | Granular CPU, memory, and I/O cgroup constraints maximizing hardware density |

---

## Learning Modules

Organized into 10 structured modules covering container mechanics, advanced networking, persistent storage, and production workflows.

### Core Containerization

| Module | Description |
| :--- | :--- |
| [**Module 01 - Introduction**](./Module%2001%20-%20Introduction/) | Container primitives, namespaces, cgroups, Docker daemon architecture, and CLI tooling |
| [**Module 02 - Installation**](./Module%2002%20-%20Installation/) | Enterprise engine setup on Ubuntu/Debian, RHEL/Fedora, Windows (WSL2), and macOS |
| [**Module 03 - Docker Basics**](./Module%2003%20-%20Docker%20Basics/) | Container lifecycle commands (`run`, `exec`, `logs`, `inspect`, `stop`), exit codes, and troubleshooting |

### Storage, Networking & Optimization

| Module | Description |
| :--- | :--- |
| [**Module 04 - Docker Images**](./Module%2004%20-%20Docker%20Images/) | Copy-on-Write (CoW) union filesystems, overlay2 driver, layer caching, tagging, and registry pushes |
| [**Module 05 - Docker Containers**](./Module%2005%20-%20Docker%20Containers/) | Resource limits (CPU/Memory cgroups), restart policies, environment variables, and healthchecks |
| [**Module 06 - Docker Networking**](./Module%2006%20-%20Docker%20Networking/) | Bridge networks, host mode, overlay networking, container DNS resolution, and port mapping |
| [**Module 07 - Docker Volumes**](./Module%2007%20-%20Docker%20Volumes/) | Named volumes, bind mounts, tmpfs in-memory storage, volume drivers, and backup workflows |

### Orchestration & Production Workflows

| Module | Description |
| :--- | :--- |
| [**Module 08 - Docker Compose**](./Module%2008%20-%20Docker%20Compose/) | Declarative multi-container orchestration, networking, secrets, depends_on, and environment configs |
| [**Module 09 - Dockerfiles**](./Module%2009%20-%20Dockerfiles/) | Production Dockerfile authoring, multi-stage builds, non-root users, BuildKit, and cache mounts |
| [**Module 10 - Docker Workflow**](./Module%2010%20-%20Docker%20Workflow/) | CI/CD pipeline integration (GitHub Actions, GitLab CI), vulnerability scanning (Trivy), and registries |

---

## Resources

### Official Documentation

| Resource | Link |
| :--- | :--- |
| Docker Documentation | [docs.docker.com](https://docs.docker.com/) |
| Docker Compose Specification | [docs.docker.com/compose](https://docs.docker.com/compose/) |
| Docker Hub Official Images | [hub.docker.com](https://hub.docker.com/) |
| Open Container Initiative (OCI) | [opencontainers.org](https://opencontainers.org/) |

### Recommended Tools

- **Dive**: Terminal UI for exploring container layer contents and discovering image waste.
- **Trivy / Grype**: Comprehensive vulnerability scanners for container images.
- **Hadolint**: Dockerfile linter enforcing best practices and security rules.
- **cAdvisor**: Resource usage and performance metrics analyzer for running containers.

---

## Contributing

Contributions are welcome! Please read our [Contributing Guidelines](CONTRIBUTING.md) before submitting a Pull Request.

---

## Security

Please review our [Security Policy](SECURITY.md) for container security best practices and vulnerability reporting procedures.

---

## License

This educational content is open source and available under the **MIT License**. See [LICENSE](LICENSE) for details.

<p align="center">
  <strong>Happy Containerizing!</strong>
</p>
