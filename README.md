# Shared GitHub configuration

Default community health files and reusable CI workflows for repositories owned by [@baranakgull](https://github.com/baranakgull).

- [`SECURITY.md`](SECURITY.md): the default security policy for every repository that has no policy of its own.
- [`.github/workflows/python-ci.yml`](.github/workflows/python-ci.yml): CI for Python projects managed with uv.
- [`.github/workflows/node-ci.yml`](.github/workflows/node-ci.yml): CI for Node.js projects managed with npm.

## Reusable CI workflows

Each workflow runs these jobs:

| Job | Python | Node.js |
| --- | --- | --- |
| **Tests** | `uv sync --locked`, then the test command | `npm ci --ignore-scripts`, optional lint and build, then the test command |
| **Lint** | Ruff, errors only by default (`E9,F63,F7,F82`) | Uses `lint-command` in the Tests job |
| **Security** | Gitleaks, pip-audit, Bandit | Gitleaks, `npm audit --omit=dev` |

The workflows are hardened in the same way:
- The token can only read.
- Every action is pinned to a commit SHA.
- Checkout does not keep credentials.
- Inputs reach shell commands only through environment variables, never through direct `${{ }}` interpolation.

### Use it in a Python repository

Create `.github/workflows/ci.yml`:

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
  workflow_dispatch:

permissions:
  contents: read
  pull-requests: read

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

jobs:
  ci:
    uses: baranakgull/.github/.github/workflows/python-ci.yml@<commit-sha> # main
    with:
      python-version: "3.12"
      bandit-paths: src
```

### Use it in a Node.js repository

```yaml
jobs:
  ci:
    uses: baranakgull/.github/.github/workflows/node-ci.yml@<commit-sha> # main
    with:
      node-version: "22"
      lint-command: npm run lint
      build-command: npm run build
```

### Inputs

**python-ci.yml**

| Input | Default | Meaning |
| --- | --- | --- |
| `python-version` | `3.12` | Python version for the tests |
| `working-directory` | `.` | Folder with `pyproject.toml` and `uv.lock` |
| `test-command` | `uv run pytest` | Command that runs the tests |
| `lint-select` | `E9,F63,F7,F82` | Ruff rules that fail the build |
| `bandit-paths` | `.` | Paths Bandit scans; empty skips Bandit |
| `bandit-severity` | `high` | Lowest Bandit severity that fails the build |

**node-ci.yml**

| Input | Default | Meaning |
| --- | --- | --- |
| `node-version` | `22` | Node.js version |
| `working-directory` | `.` | Folder with `package.json` and `package-lock.json` |
| `test-command` | `npm test` | Test command; empty skips it |
| `lint-command` | empty | Lint command; empty skips it |
| `build-command` | empty | Build command; empty skips it |
| `audit-level` | `high` | Lowest `npm audit` severity that fails the build |

### Branch rulesets

When a workflow is called from a job named `ci`, its checks are reported as `ci / Tests`, `ci / Lint` and `ci / Security`. Use these names as required status checks.

### Pinning

Callers pin the workflow to a commit SHA rather than `@main`, so a change in this repository reaches a project only when that project updates the SHA. Dependabot's `github-actions` updates do this automatically.
