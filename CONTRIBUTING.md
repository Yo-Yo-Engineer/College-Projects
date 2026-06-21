# Contributing

Thank you for your interest in contributing! We welcome all contributions — bug fixes, features, documentation, and ideas.

## Table of Contents

- [Code of Conduct](#code-of-conduct)
- [Getting Started](#getting-started)
- [How to Contribute](#how-to-contribute)
- [Development Workflow](#development-workflow)
- [Commit Message Standards](#commit-message-standards)
- [Pull Request Process](#pull-request-process)
- [Reporting Issues](#reporting-issues)
- [Security Vulnerabilities](#security-vulnerabilities)
- [Code Style & Standards](#code-style--standards)
- [License](#license)

## Code of Conduct

All contributors are expected to treat each other with respect and professionalism. Harassment, discrimination, and disruptive behavior will not be tolerated. By participating, you agree to uphold a welcoming and inclusive environment.

## Getting Started

### Prerequisites

- [Git](https://git-scm.com/)
- [Docker](https://www.docker.com/) (optional, for containerized development)
- [VS Code](https://code.visualstudio.com/) (recommended, with GitHub Copilot)

### Setup

1. Fork the repository and clone your fork:

    ```bash
    git clone https://github.com/<your-username>/<repo-name>.git
    cd <repo-name>
    ```

2. Add the upstream remote:

    ```bash
    git remote add upstream https://github.com/<org>/<repo-name>.git
    ```

3. Set up environment variables:

    ```bash
    cp example.env .env
    # Edit .env with your configuration
    ```

4. (Optional) Start the local development environment:

    ```bash
    docker compose -f docker-compose.local.yml up
    ```

## How to Contribute

We accept the following types of contributions:

| Type               | Examples                                            |
| ------------------ | --------------------------------------------------- |
| **Bug fixes**      | Resolve open issues, fix regressions                |
| **Features**       | New capabilities, enhancements to existing behavior |
| **Documentation**  | Improve guides, fix typos, add examples             |
| **Refactoring**    | Code clarity, performance, structure improvements   |
| **Infrastructure** | CI/CD, Docker, environment configuration            |
| **Tests**          | New test coverage, fix flaky tests                  |

> **First time contributing?** Look for issues labeled `good first issue` or `help wanted`.

## Development Workflow

### 1. Sync with upstream

```bash
git fetch upstream
git checkout main
git merge upstream/main
```

### 2. Create a feature branch

Use a descriptive branch name:

```
<type>/<short-description>

# Examples
feat/add-user-auth
fix/null-pointer-on-login
docs/update-contributing-guide
refactor/extract-validation-logic
```

### 3. Make your changes

- Keep changes focused — one concern per branch.
- Write or update tests for any new or changed behavior.
- Update documentation if your changes affect public-facing behavior.

### 4. Test locally

Run the full test suite before pushing. Verify that existing tests still pass and no functionality is broken.

### 5. Push and open a PR

```bash
git push origin <your-branch>
```

Then open a Pull Request against `main`. Fill out the [Pull Request Template](PULL_REQUEST_TEMPLATE.md) completely.

## Commit Message Standards

Follow the [Conventional Commits](https://www.conventionalcommits.org/) specification:

```
<type>(<scope>): <subject>

[optional body]

[optional footer(s)]
```

### Types

| Type       | Purpose                                |
| ---------- | -------------------------------------- |
| `feat`     | New feature                            |
| `fix`      | Bug fix                                |
| `docs`     | Documentation only                     |
| `style`    | Formatting, no logic change            |
| `refactor` | Code restructuring, no behavior change |
| `perf`     | Performance improvement                |
| `test`     | Adding or updating tests               |
| `chore`    | Build, CI, tooling, dependencies       |

### Rules

- **Subject line:** Imperative mood, lowercase, no period, max 72 characters.
- **Body:** Explain _what_ and _why_, not _how_. Wrap at 80 characters.
- **Footer:** Reference issues (`Fixes #123`, `Closes #456`).

### Examples

```
feat(auth): add OAuth2 login flow

Implement OAuth2 authorization code flow with PKCE for
secure browser-based authentication.

Closes #42
```

```
fix(api): prevent null response on empty query

Return an empty array instead of null when the search
query matches no results.

Fixes #87
```

## Pull Request Process

1. **Fill out the template** — Use the [Pull Request Template](PULL_REQUEST_TEMPLATE.md). Incomplete PRs will be sent back.
2. **Link related issues** — Reference issues using `Fixes #123` or `Closes #456`.
3. **Keep PRs small** — Smaller PRs get faster, better reviews. Split large changes into stacked PRs when possible.
4. **All checks must pass** — Tests, linting, and CI must be green before review.
5. **Request review** — Assign at least one reviewer. PRs require approval before merging.
6. **Address feedback promptly** — Respond to review comments. Push follow-up commits; do not force-push during review.
7. **Squash on merge** — Keep the commit history clean. The final merge commit message should follow [commit standards](#commit-message-standards).

## Reporting Issues

When reporting a bug or requesting a feature:

1. **Search first** — Check existing issues to avoid duplicates.
2. **Use a clear title** — Summarize the problem or request in one line.
3. **Provide details:**
    - **Bug reports:** Steps to reproduce, expected vs. actual behavior, environment info (OS, runtime version), and relevant logs or screenshots.
    - **Feature requests:** Problem statement, proposed solution, and any alternatives considered.

## Security Vulnerabilities

Do **not** open a public issue for security vulnerabilities. Instead, report them privately to the maintainers. Include:

- Description of the vulnerability
- Steps to reproduce
- Potential impact
- Suggested fix (if any)

## Code Style & Standards

- Follow the existing code patterns and conventions in the repository.
- Refer to the [best practice guides](templates/references/best_practices/) for detailed standards on security, performance, testing, code quality, and more.
- Write self-documenting code. Add comments only when the _why_ isn't obvious.
- Keep dependencies minimal and justified.

## License

This project is licensed under the [MIT License](LICENSE). By contributing, you agree that your contributions will be licensed under the same terms.
