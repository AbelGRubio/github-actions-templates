# Continuous Integration and Continuous Deployment (CI/CD) Procedure in GitHub

This document defines the standard procedure for Continuous Integration (CI), Continuous Deployment (CD), branching strategy, code quality control, and automated versioning using **GitHub Actions**, **SonarQube**, **Ruff**, and **Python Semantic Release**.

## 1. Branching Strategy and Versioning

The workflow is based on an adapted **GitFlow** model, optimized for CI/CD automation and Semantic Versioning (`vMAJOR.MINOR.PATCH-prerelease.number`).

```
  master  ----------------●-----------------------● (v1.0.0, v1.0.1)
                   \     ^                 /     ^
  develop  ---------●----+-----●----------●------+ (v1.1.0-rc.1, v1.1.0-rc.2)
             \     /          /          /
  feature/*   ●---●          /          /          (v1.1.0-alpha.1, beta.1)
                            /          /
  hotfix/*  ---------------●----------●            (Urgent patches)

```

### 1.1. Versioning Rules by Branch

| Branch | Release Type | Tag Format | Example |
| ----- | ----- | ----- | ----- |
| `master` | Official / Production Release | `vX.Y.Z` | `v1.0.0`, `v1.0.1` |
| `develop` | Pre-release (Release Candidate) | `vX.Y.Z-rc.N` | `v1.1.0-rc.1` |
| `feature/*` | Alpha / Beta (Experimental) | `vX.Y.Z-alpha.N` / `vX.Y.Z-beta.N` | `v1.1.0-alpha.1` |
| `hotfix/*` | Hotfix / Patch | `vX.Y.Z-rc.N` (Prior to master) | `v1.0.1-rc.1` |

## 2. Workflow and Stages per Branch (CI/CD)

### 2.1. Execution Matrix by Branch and Event

| Stage / Tool | Pull Request (To develop/master) | Push to `feature/*` | Push to `develop` | Push to `master` | Push to `hotfix/*` |
| ----- | ----- | ----- | ----- | ----- | ----- |
| **Linter / Formatter (Ruff)** | Yes | Yes | Yes | Yes | Yes |
| **Unit & Integration Tests** | Yes | Yes | Yes | Yes | Yes |
| **Quality Analysis (SonarQube)** | Yes (PR Scan) | ❌ | Yes | Yes | ❌ |
| **Semantic Release (Versioning)** | ❌ | Yes (alpha/beta) | Yes (rc) | Yes (final) | ❌ |
| **Artifact Publishing** | ❌ | ❌ | Yes (Staging) | Yes (Prod) | ❌ |
| **Automated Deployment** | ❌ | ❌ | Yes (QA Env) | Yes (PROD Env) | ❌ |

## 3. Pull Request (PR) Procedure

### 3.1. What Should Run in a PR and Why?

When a PR is opened or updated towards `develop` or `master`, the **PR Validation** pipeline is executed:

1. **Ruff (Linter & Formatter):** Ensures the code adheres to project style guidelines before human review.

2. **Tests with Coverage:** Verifies that new code does not break existing functionality and generates a coverage report (`coverage.xml`).

3. **SonarQube Quality Gate:** Scans code for vulnerabilities, code smells, duplications, and evaluates test coverage on new changes. If the *Quality Gate* fails, merging is blocked.

> **Important:** PRs **NEVER** publish artifacts or run `semantic-release` to prevent accidental tags or releases from unmerged code.

## 4. Hotfix Management and Rollback

### 4.1. Hotfix Procedure

1. Create a `hotfix/brief-description` branch directly from `master`.

2. Implement and locally validate the fix.

3. Open PRs from `hotfix/*` towards both `master` and `develop`.

4. Once approved and merged into `master`, the `master` pipeline triggers `semantic-release`, creating a **PATCH** version increment (e.g., `v1.0.0` ➔ `v1.0.1`) and deploying it to production.

### 4.2. Rollback Strategy and Procedure in CD

There are two primary strategies to handle a Rollback in the CD environment:

#### Option A: Rollback via Git Revert (Recommended)

This is the safest approach in GitOps:

1. Identify the faulty commit or PR.

2. Run `git revert <commit-id>` on a `hotfix/*` branch or directly towards `master`.

3. Merge to `master`. The CI/CD pipeline compiles the stable prior code as a new patch (`vX.Y.Z+1`) and deploys it.

#### Option B: Manual Rollback via GitHub Actions Workflow

If the production application is completely down and cannot wait for a CI build cycle:

```
[User Triggers Rollback Workflow] 
          │
          ├──> 1. Fetches previous Releases/Tags stored in Registry
          ├──> 2. Downloads Artifact/Image associated with selected Tag
          └──> 3. Executes direct deployment step to the target environment

```

1. Go to the **Actions** tab in GitHub.

2. Select the **CD Deploy / Rollback** workflow.

3. Choose `workflow_dispatch`, enter the target stable version tag (e.g., `v1.0.0`), and select the target environment (`production`).

## 5. Project Configuration (`pyproject.toml`)

Configuration for **Ruff** and **Python Semantic Release** tailored to manage various release channels (master, develop, feature):

```toml
[build-system]
requires = ["poetry-core>=1.0.0"]
build-backend = "poetry.core.masonry.api"

[tool.poetry]
name = "my-project"
version = "0.1.0"
description = "Project with CI/CD Automation"
authors = ["Dev Team <dev@company.com>"]

# --- RUFF CONFIGURATION ---
[tool.ruff]
line-length = 88
target-version = "py311"

[tool.ruff.lint]
select = ["E", "F", "I", "B", "UP"]
ignore = []

# --- PYTHON-SEMANTIC-RELEASE CONFIGURATION ---
[tool.semantic_release]
version_variable = ["pyproject.toml:version"]
branch = "master"
upload_to_pypi = false
upload_to_release = true
build_command = "poetry build"

[tool.semantic_release.branches.master]
match = "master"
prerelease = false

[tool.semantic_release.branches.develop]
match = "develop"
prerelease = true
prerelease_token = "rc"

[tool.semantic_release.branches.features]
match = "(feature|feat)/.*"
prerelease = true
prerelease_token = "alpha"

[tool.semantic_release.branches.hotfix]
match = "hotfix/.*"
prerelease = true
prerelease_token = "rc"

```

## 6. GitHub Actions Workflow Examples

### 6.1. Continuous Integration on PR (`.github/workflows/ci-pr.yml`)

```yaml
name: CI - PR Validation

on:
  pull_request:
    branches: [ master, develop ]

jobs:
  lint-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Install uv
        uses: astral-sh/setup-uv@v3
        with:
          enable-cache: true
          version: "latest"

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install Dependencies
        run: uv sync --all-extras --dev

      - name: Run Ruff (Linter & Formatter Check)
        run: |
          uv run ruff check .
          uv run ruff format --check .

      - name: Run Pytest with Coverage
        run: uv run pytest

      - name: SonarQube Scan
        uses: SonarSource/sonarqube-scan-action@v2.0.2
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
          SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}
```

### 6.2. Main CD & Release Pipeline (`.github/workflows/cd-release.yml`)

```yaml
name: CD - Release & Deploy

on:
  push:
    branches:
      - master
      - develop
      - 'feature/*'

jobs:
  quality-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install uv
        uses: astral-sh/setup-uv@v3
        with:
          enable-cache: true
          version: "latest"

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install Dependencies
        run: uv sync --all-extras --dev

      - name: Lint & Test
        run: |
          uv run ruff check .
          uv run pytest

  semantic-release:
    needs: quality-and-test
    runs-on: ubuntu-latest
    outputs:
      released: ${{ steps.release.outputs.released }}
      version: ${{ steps.release.outputs.version }}
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
          token: ${{ secrets.GITHUB_TOKEN }}

      - name: Install uv
        uses: astral-sh/setup-uv@v3
        with:
          version: "latest"

      - name: Python Semantic Release
        id: release
        uses: python-semantic-release/python-semantic-release@v8.5.1
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}

  build-and-publish-artifact:
    needs: semantic-release
    if: needs.semantic-release.outputs.released == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install uv
        uses: astral-sh/setup-uv@v3
        with:
          version: "latest"

      - name: Build Package with uv
        run: uv build

      - name: Upload Artifact
        uses: actions/upload-artifact@v4
        with:
          name: package-${{ needs.semantic-release.outputs.version }}
          path: dist/

  deploy:
    needs: [semantic-release, build-and-publish-artifact]
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to Environment
        run: |
          if [[ "${{ github.ref }}" == "refs/heads/master" ]]; then
            echo "Deploying official release ${{ needs.semantic-release.outputs.version }} to PRODUCTION..."
          elif [[ "${{ github.ref }}" == "refs/heads/develop" ]]; then
            echo "Deploying pre-release ${{ needs.semantic-release.outputs.version }} to STAGING/QA..."
          fi

```

### 6.3. Manual Rollback Workflow (`.github/workflows/cd-rollback.yml`)

```yaml
name: CD - Manual Rollback

on:
  workflow_dispatch:
    inputs:
      target_version:
        description: 'Tag of the version to restore (e.g., v1.0.0)'
        required: true
        type: string
      environment:
        description: 'Target Environment'
        required: true
        default: 'production'
        type: choice
        options:
          - production
          - staging

jobs:
  rollback:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Specific Tag
        uses: actions/checkout@v4
        with:
          ref: ${{ inputs.target_version }}

      - name: Install uv
        uses: astral-sh/setup-uv@v3
        with:
          version: "latest"

      - name: Execute Rollback Deployment
        run: |
          echo "Starting Rollback to version ${{ inputs.target_version }} in environment ${{ inputs.environment }}"
          # Build artifact for the rollback release target if needed
          uv build
          # Add environment-specific deployment scripts here

```