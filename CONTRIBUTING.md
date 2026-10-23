# Contributing to Awesome Docker

Thank you for your interest in contributing to **Awesome Docker**! We welcome contributions, bug fixes, enhancements, and architectural best practices from the community.

---

## How to Contribute

### Reporting Issues
If you identify a bug in any Docker command, syntax error in Dockerfiles, broken links, or have suggestions for new workflows:
1. Check existing issues to verify if the topic has already been reported.
2. Open a new issue with a clear, descriptive title and explanation.
3. Include your Docker version (`docker version`), host OS, and steps to reproduce.

### Making Changes

1. **Fork** the repository to your GitHub account.
2. **Clone** your fork locally:
   ```bash
   git clone https://github.com/bhagirath00/awesome-docker.git
   cd awesome-docker
   ```
3. Create a descriptive feature **branch**:
   ```bash
   git checkout -b feature/your-feature-name
   ```
4. Make your changes adhering to the style guide below.
5. **Commit** your changes:
   ```bash
   git commit -m "feat(module-09): add buildkit cache mount examples"
   ```
6. **Push** to your fork:
   ```bash
   git push origin feature/your-feature-name
   ```
7. Open a **Pull Request** against the `main` branch.

---

## Style Guide

### Dockerfiles & Configurations
- **Best Practices**: Use multi-stage builds, pin base image tags (e.g., `alpine:3.19` or `golang:1.22-bookworm`), and avoid `:latest`.
- **Security**: Run containers as non-root users (`USER nonroot` or `USER 10001`).
- **Validation**: Lint Dockerfiles locally using Hadolint (`hadolint Dockerfile`).
- **Compose Files**: Use modern Compose specification (omit deprecated `version:` top-level attribute).

### Documentation & Guides
- Use clear, professional, technical English.
- Include working CLI command snippets and sample expected outputs.
- Include ASCII architectural flow diagrams or container lifecycle transitions where helpful.
- Reference official Docker documentation ([docs.docker.com](https://docs.docker.com/)).
- Maintain sequential chapter numbering (`01-`, `02-`, etc.) to preserve directory sorting.

### Commit Messages
We follow the [Conventional Commits](https://www.conventionalcommits.org/) specification:
- `feat:` – New chapter, guide, or feature addition
- `fix:` – Bug fix, Dockerfile correction, or typo fix
- `docs:` – Documentation updates or formatting adjustments
- `refactor:` – Reorganizing chapters, updating naming, or streamlining configurations

---

## Questions & Support

Feel free to open a GitHub Discussion or Issue for any questions.

Thank you for contributing to **Awesome Docker**!
