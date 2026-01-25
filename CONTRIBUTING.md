# Contributing to Logging Stack Docker

Thank you for your interest in contributing! This document provides guidelines and instructions for contributing.

## Code of Conduct

- Be respectful and inclusive
- Provide constructive feedback
- Focus on issues and improvements, not personal attacks
- Help others learn and grow

## How to Contribute

### Reporting Issues

1. Check if the issue already exists
2. Provide a clear, descriptive title
3. Include steps to reproduce
4. Attach logs if applicable
5. Specify your environment (Docker version, OS, etc.)

### Suggesting Enhancements

1. Check existing issues and discussions
2. Clearly describe the enhancement
3. Explain the use case
4. Provide examples if possible

### Submitting Pull Requests

1. Fork the repository
2. Create a feature branch: `git checkout -b feature/your-feature`
3. Make your changes
4. Add tests if applicable
5. Update documentation
6. Commit with clear messages: `git commit -m "Add feature: description"`
7. Push to your fork
8. Create a Pull Request with a clear description

## Development Setup

```bash
# Clone your fork
git clone https://github.com/YOUR_USERNAME/logging-stack-docker.git

# Create feature branch
git checkout -b feature/your-feature

# Make changes
# Test locally

# Commit and push
git push origin feature/your-feature
```

## Testing Guidelines

- Test changes locally before submitting
- Test with different log volumes
- Verify all services still start correctly
- Test the integration with external services

## Documentation

Update documentation for:
- New configuration options
- Breaking changes
- New features
- Bug fixes that affect usage

## Commit Messages

Use clear, concise commit messages:

```
[type]: Brief description (50 chars max)

Optional detailed explanation (72 chars max per line).
Explain what and why, not how.

Types:
- feat: New feature
- fix: Bug fix
- docs: Documentation
- config: Configuration changes
- security: Security improvements
```

## License

By contributing, you agree that your contributions will be licensed under the MIT License.
