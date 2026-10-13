# Contributing to Docker for DevOps

Thank you for your interest in contributing to **awesome docker**! We welcome contributions from the community.

## How to Contribute

### Reporting Content Issues

If you find a bug in the code examples or have a suggestion for the content:

1. Check if the issue already exists
2. Open a new issue with a clear title and description
3. Include steps to reproduce (for code bugs)

### Making Changes

1. **Fork** the repository
2. **Clone** your fork locally:
   ```bash
   git clone https://github.com/bhagirath00/awesome-docker.git
   ```
3. Create a **branch** for your changes:
   ```bash
   git checkout -b feature/your-feature-name
   ```
4. Make your changes following the style guide below
5. **Commit** your changes:
   ```bash
   git commit -m "Add your feature description"
   ```
6. **Push** to your fork:
   ```bash
   git push origin feature/your-feature-name
   ```
7. Open a **Pull Request**

## Style Guide 

### Code & Configuration

- Use official base images (e.g., `python:slim` instead of full OS)
- Follow Dockerfile best practices (layers, cache, etc.)
- Use clear naming conventions for containers and volumes
- Provide `docker-compose.yml` for multi-container setups

### Documentation

- Use clear, concise language
- Include working command examples
- Add links to official Docker documentation
- Follow the existing module format:
  - Learning Objectives
  - Concepts with examples (Diagrams preferred)
  - Summary

### Commit Messages

Use conventional commit format:

- `feat:` - New feature
- `fix:` - Bug fix
- `docs:` - Documentation changes
- `refactor:` - Code refactoring

## Questions?

Open an issue or reach out to the maintainers.

---

Thank you for contributing!
