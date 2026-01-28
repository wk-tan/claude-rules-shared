# Automation: CI/CD Workflows & Version Control

**When to use this file:** Reference this for GitHub Actions CI/CD workflows, git commit conventions, and semantic versioning.

**Related documentation:**
- For Python setup, see [02-development.md](02-development.md)
- For dependency management with uv, see [03-dependencies.md](03-dependencies.md)
- For deployment patterns, see [05-deployment.md](05-deployment.md)

---

## Table of Contents

### Part 1: CI/CD Workflows
1. [Python Setup Standards](#1-python-setup-standards)
2. [Dependency Installation](#2-dependency-installation)
3. [Virtual Environment Activation](#3-virtual-environment-activation)
4. [Caching Policy](#4-caching-policy)
5. [Complete Workflow Examples](#5-complete-workflow-examples)
6. [Common Patterns and Best Practices](#6-common-patterns-and-best-practices)
7. [Version Upgrade Checklist](#7-version-upgrade-checklist)
8. [Prohibited Patterns](#8-prohibited-patterns)
9. [Workflow Maintenance](#9-workflow-maintenance)

### Part 2: Version Control & Releases
10. [Commit Message Format](#10-commit-message-format)
11. [Breaking Changes](#11-breaking-changes)
12. [Complete Commit Examples](#12-complete-commit-examples)
13. [Branch Naming Conventions](#13-branch-naming-conventions)
14. [Pull Request Conventions](#14-pull-request-conventions)
15. [Semantic Release Workflow](#15-semantic-release-workflow)
16. [Best Practices for Commit Messages](#16-best-practices-for-commit-messages)
17. [Common Mistakes and Fixes](#17-common-mistakes-and-fixes)
18. [Git Workflow Integration](#18-git-workflow-integration)
19. [Tools and Resources](#19-tools-and-resources)
20. [Troubleshooting](#20-troubleshooting)

---

# Part 1: CI/CD Workflows

These guidelines establish the standards and best practices for GitHub Actions workflows across all our Python projects. Following these rules ensures consistent, reliable, and efficient CI/CD pipelines that leverage `uv` for fast, deterministic builds.

## 1. Python Setup Standards

All workflows that require Python MUST follow these setup patterns:

### Python Version Configuration
- **Always use `python-version-file`:** Never hardcode Python versions in workflow files. Instead, reference the version from `pyproject.toml`.
- **Command:**
  ```yaml
  - name: "Set up Python"
    uses: actions/setup-python@v5
    with:
      python-version-file: "pyproject.toml"
  ```
- **Rationale:** This ensures the CI/CD environment matches the project's declared Python version, preventing version drift between local development and CI/CD.

### uv Installation
- **Always hardcode uv version:** Use an exact version (currently `0.7.8`) to ensure reproducible builds.
- **No version ranges:** Never use `latest` or version ranges.
- **Command:**
  ```yaml
  - name: "Set up uv"
    uses: astral-sh/setup-uv@v5
    with:
      version: "0.7.8"
  ```
- **Rationale:** Pinning the exact version prevents unexpected breaking changes and ensures all builds use the same dependency resolution logic.

## 2. Dependency Installation

### Standard Installation Pattern
- **Always use `uv sync --frozen`:** This ensures the CI/CD environment matches the exact dependencies from `uv.lock`.
- **Command:**
  ```yaml
  - name: "Install dependencies"
    run: |
      uv sync --frozen
  ```
- **Never use `uv sync` without `--frozen`:** The `--frozen` flag prevents uv from updating the lock file, ensuring deterministic builds.

### Group-Specific Installation
- **For targeted dependency groups:** Use `--group <name>` to install only required dependencies for specific jobs.
- **Examples:**
  ```yaml
  # For release jobs
  - name: "Install release dependencies"
    run: |
      uv sync --frozen --group release

  # For test jobs (if not using default groups)
  - name: "Install test dependencies"
    run: |
      uv sync --frozen --group test
  ```

## 3. Virtual Environment Activation

### Activation Pattern
- **Use absolute path activation:** Add the virtualenv to `$GITHUB_PATH` using an absolute path.
- **Command:**
  ```yaml
  - name: "Activate virtualenv"
    run: echo "$PWD/.venv/bin" >> $GITHUB_PATH
  ```
- **Rationale:** This approach works reliably across all subsequent steps without requiring manual activation in each step.

### Alternative: Direct Execution
- **For single commands:** You can optionally use `uv run` for one-off commands.
- **Example:**
  ```yaml
  - name: "Run tests"
    run: uv run pytest test/
  ```

## 4. Caching Policy

### No Caching for uv
- **NEVER cache uv dependencies:** uv is optimized for speed and caching adds unnecessary complexity without meaningful performance benefits.
- **Remove all cache-related steps:** Do not use `actions/cache` for Python dependencies when using uv.
- **Rationale:** uv's dependency resolution and installation is already extremely fast (<5 seconds in most cases), making caching overhead not worthwhile.

## 5. Complete Workflow Examples

### Example 1: Type Check Workflow
```yaml
name: Type Check

on:
  pull_request:
    branches: [main]

jobs:
  type-check:
    runs-on: ubuntu-latest
    steps:
      - name: "Check out GitHub repository"
        uses: actions/checkout@v4

      - name: "Set up Python"
        uses: actions/setup-python@v5
        with:
          python-version-file: "pyproject.toml"

      - name: "Set up uv"
        uses: astral-sh/setup-uv@v5
        with:
          version: "0.7.8"

      - name: "Install dependencies"
        run: |
          uv sync --frozen

      - name: "Activate virtualenv"
        run: echo "$PWD/.venv/bin" >> $GITHUB_PATH

      - name: "Execute type check via pyright"
        uses: jakebailey/pyright-action@v2
```

### Example 2: Test Workflow
```yaml
name: Test

on:
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: "Check out GitHub repository"
        uses: actions/checkout@v4

      - name: "Set up Python"
        uses: actions/setup-python@v5
        with:
          python-version-file: "pyproject.toml"

      - name: "Set up uv"
        uses: astral-sh/setup-uv@v5
        with:
          version: "0.7.8"

      - name: "Install dependencies"
        run: |
          uv sync --frozen

      - name: "Activate virtualenv"
        run: echo "$PWD/.venv/bin" >> $GITHUB_PATH

      - name: "Execute pytest"
        run: |
          uv run pytest test/             \
          --junitxml=pytest.xml                \
          --cov-report=xml:coverage.xml        \
          --cov=components                     \
          --cov=bases                          \
          | tee pytest-coverage.txt
```

### Example 3: Release Workflow with Group
```yaml
name: Release

on:
  push:
    branches: [main]

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - name: "Check out GitHub repository"
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: "Set up Python"
        uses: actions/setup-python@v5
        with:
          python-version-file: "pyproject.toml"

      - name: "Set up uv"
        uses: astral-sh/setup-uv@v5
        with:
          version: "0.7.8"

      - name: "Install release dependencies"
        run: |
          uv sync --frozen --group release

      - name: "Activate virtualenv"
        run: echo "$PWD/.venv/bin" >> $GITHUB_PATH

      - name: "Run semantic release"
        run: semantic-release publish
```

### Example 4: Infrastructure Provision Workflow
```yaml
name: Infrastructure Provision

on:
  pull_request:
    types: [labeled]      # Sandbox: add "provision infrastructure" label
  workflow_dispatch:      # Production: manual trigger from GitHub UI

jobs:
  provision-infrastructure:
    if: |
      github.event_name == 'workflow_dispatch' ||
      (github.event_name == 'pull_request' && github.event.label.name == 'provision infrastructure')
    runs-on: ubuntu-latest
    environment: ${{ github.event_name == 'workflow_dispatch' && 'production' || 'sandbox' }}
    permissions:
      id-token: write
      contents: read
      pull-requests: write
    steps:
      - name: "Check out GitHub repository"
        uses: actions/checkout@v4

      - name: "Determine stack and version"
        id: determine-stack-version
        run: |
          if [[ "${{ github.event_name }}" == "workflow_dispatch" ]]; then
            STACK="production"
            VERSION=$(echo "${{ github.ref }}" | sed 's|refs/tags/v||')
          else
            STACK="sandbox"
            TIMESTAMP=$(date -u +%Y%m%d-%H%M%S)
            BRANCH=${GITHUB_REF_NAME//[\/\-]/_}
            VERSION=${BRANCH}-${TIMESTAMP}
          fi
          echo "STACK=${STACK}" >> $GITHUB_OUTPUT
          echo "VERSION=${VERSION}" >> $GITHUB_OUTPUT

      - name: "Set up Python"
        uses: actions/setup-python@v5
        with:
          python-version-file: "pyproject.toml"

      - name: "Set up uv"
        uses: astral-sh/setup-uv@v5
        with:
          version: "0.7.8"

      - name: "Install dependencies"
        run: |
          uv sync --frozen --group release

      - name: "Activate virtualenv"
        run: echo "$PWD/.venv/bin" >> $GITHUB_PATH

      - name: "Authenticate to Google Cloud"
        uses: google-github-actions/auth@v3
        with:
          token_format: "access_token"
          workload_identity_provider: "projects/${{ secrets.GCP_PROJECT_NUMBER }}/locations/global/workloadIdentityPools/gha-${{ vars.ENVIRONMENT }}/providers/pulsifi-github-${{ vars.ENVIRONMENT }}"
          service_account: "github-actions@${{ vars.GCP_PROJECT_ID }}.iam.gserviceaccount.com"

      - name: "Extract Pulumi version from lock file"
        id: extract-pulumi-version
        run: |
          # Extract exact Pulumi version from uv.lock
          PULUMI_VERSION=$(grep -A 1 '^name = "pulumi"$' uv.lock | grep '^version = ' | head -1 | sed 's/version = "\(.*\)"/\1/')
          echo "PULUMI_VERSION=${PULUMI_VERSION}" >> $GITHUB_OUTPUT
          echo "Using Pulumi version: ${PULUMI_VERSION}"

      - name: "Update Pulumi"
        uses: pulumi/actions@v6
        with:
          pulumi-version: ${{ steps.extract-pulumi-version.outputs.PULUMI_VERSION }}

      - name: "Initialize Pulumi dependencies"
        working-directory: infrastructure/pulumi
        run: |
          pulumi install

      - name: "Provision infrastructure with Pulumi"
        uses: pulumi/actions@v6
        env:
          PULUMI_CONFIG_PASSPHRASE: ${{ secrets.PULUMI_CONFIG_PASSPHRASE }}
          VERSION: ${{ steps.determine-stack-version.outputs.VERSION }}
        with:
          command: up
          cloud-url: gs://${{ vars.PULUMI_BUCKET }}
          work-dir: infrastructure/pulumi
          stack-name: ${{ steps.determine-stack-version.outputs.STACK }}
          upsert: true
          comment-on-pr: true
          edit-pr-comment: false
          refresh: true
          config-map: |
            {
              input:version: { value: ${{ env.VERSION }}, secret: false },
            }
```

**Key differences from application deployment workflows:**
- **Trigger mechanisms:** Uses `workflow_dispatch` for production and PR labels for sandbox
- **Environment selection:** Dynamically determines environment based on trigger type
- **Dependency group:** Uses `--group release` for Pulumi and IaC tools
- **Working directory:** Operates in `infrastructure/pulumi/` directory
- **Pulumi-specific steps:** Extracts Pulumi version from lock file, initializes dependencies, runs `pulumi up`
- **Stack management:** Automatically selects correct Pulumi stack (production or sandbox)
- **Version handling:** Generates version from git tag (production) or branch+timestamp (sandbox)

**When to use this workflow:**
- Provisioning foundational GCP resources (service accounts, buckets, IAM, Artifact Registry)
- Rare infrastructure changes (weeks to months)
- Changes that affect multiple applications or services
- Infrastructure updates that don't require application redeployment

**For Pulumi IaC patterns and configuration details, see [07-infrastructure.md](07-infrastructure.md#5-pulumi-iac-patterns).**

## 6. Common Patterns and Best Practices

### Multi-Job Workflows
- **Repeat setup steps in each job:** GitHub Actions jobs run in separate environments, so each job needs its own setup.
- **Keep jobs independent:** Avoid dependencies between jobs unless absolutely necessary.

### Error Handling
- **Use `set -e` implicitly:** GitHub Actions runs with `set -e` by default, so commands will fail fast on errors.
- **Explicit error checks:** For complex steps, add explicit error checking:
  ```yaml
  - name: "Run command with error check"
    run: |
      if ! uv run mycommand; then
        echo "Command failed" >&2
        exit 1
      fi
  ```

### Testing Workflows Locally
- **Use `act`:** Install and use [act](https://github.com/nektos/act) to test workflows locally before pushing.
- **Command:** `act -j <job-name>`

### Docker Build Best Practices

**Docker Buildx Setup:**
- Always use `docker/setup-buildx-action@v3` before building Docker images
- Buildx enables advanced features like multi-platform builds and improved caching

```yaml
- name: "Set up Docker Buildx"
  uses: docker/setup-buildx-action@v3
```

**GitHub Actions Layer Caching:**
- Use `cache-from: type=gha` and `cache-to: type=gha,mode=max` for faster builds in development/sandbox environments
- **Do NOT use caching in production** to ensure fresh, reproducible builds

```yaml
# Sandbox deployment - WITH caching for faster iteration
- name: "Build and push Docker image"
  uses: docker/build-push-action@v5
  with:
    context: .
    file: ./projects/console/Dockerfile
    push: true
    tags: |
      asia-southeast1-docker.pkg.dev/${{ vars.GCP_PROJECT_ID }}/de-backoffice/console-cr:${{ steps.configure_version_tag.outputs.VERSION }}
      asia-southeast1-docker.pkg.dev/${{ vars.GCP_PROJECT_ID }}/de-backoffice/console-cr:latest
    cache-from: type=gha
    cache-to: type=gha,mode=max

# Production deployment - WITHOUT caching for reproducibility
- name: "Build and push Docker image"
  uses: docker/build-push-action@v5
  with:
    context: .
    file: ./projects/console/Dockerfile
    push: true
    tags: |
      asia-southeast1-docker.pkg.dev/${{ vars.GCP_PROJECT_ID }}/de-backoffice/console-cr:${{ steps.set_version.outputs.VERSION }}
      asia-southeast1-docker.pkg.dev/${{ vars.GCP_PROJECT_ID }}/de-backoffice/console-cr:latest
    # No cache-from or cache-to for production
```

**Why different caching strategies:**
- **Sandbox:** Fast iteration is valuable; caching speeds up development feedback loops
- **Production:** Reproducibility and security are paramount; build from scratch every time

### Step Naming and ID Conventions

**Step ID naming convention:**
- Use `id:` field when the step sets output variables that other steps will reference
- Format: `verb-subject` in kebab-case (lowercase with hyphens)
- Examples: `determine-stack-version`, `extract-pulumi-version`, `configure-version-tag`, `get-secrets`

```yaml
# Good - descriptive verb-subject format
- name: "Determine stack and version"
  id: determine-stack-version
  run: |
    echo "STACK=${STACK}" >> $GITHUB_OUTPUT
    echo "VERSION=${VERSION}" >> $GITHUB_OUTPUT

- name: "Extract Pulumi version from lock file"
  id: extract-pulumi-version
  run: |
    # Extract exact Pulumi version from uv.lock
    PULUMI_VERSION=$(grep -A 1 '^name = "pulumi"$' uv.lock | grep '^version = ' | head -1 | sed 's/version = "\(.*\)"/\1/')
    echo "PULUMI_VERSION=${PULUMI_VERSION}" >> $GITHUB_OUTPUT
    echo "Using Pulumi version: ${PULUMI_VERSION}"

# Bad - not verb-subject format
- name: "Stack and version"
  id: stack_version  # Wrong: uses underscores, not descriptive

- name: "Get Pulumi version"
  id: pulumi  # Wrong: missing verb, too terse
```

**When to add `id` to steps:**
- Step sets `GITHUB_OUTPUT` variables
- Step outputs need to be referenced by later steps (e.g., `${{ steps.configure-version-tag.outputs.VERSION }}`)
- Step results are used in conditionals or other workflow logic

**When NOT to add `id`:**
- Step has no outputs
- Step is self-contained and doesn't share data with other steps

### Descriptive Logging in Workflows

**Echo statement best practices:**
- Use descriptive echo statements to make workflow logs readable and debuggable
- Include context about what's happening and the values being used
- Echo before setting output variables to document the values

```yaml
# Good - descriptive logging
- name: "Configure version tag"
  id: configure_version_tag
  run: |
    TIMESTAMP=$(date -u +%Y%m%d-%H%M%S)
    BRANCH=${GITHUB_REF_NAME//[\/-]/_}
    VERSION=${BRANCH}-${TIMESTAMP}

    echo "Generated sandbox version: ${VERSION}"
    echo "VERSION=${VERSION}" >> $GITHUB_OUTPUT

- name: "Determine stack and version"
  id: determine-stack-version
  run: |
    if [[ "${{ github.event_name }}" == "workflow_dispatch" ]]; then
      STACK="production"
      VERSION=$(echo "${{ github.ref }}" | sed 's|refs/tags/v||')
      echo "Using version from selected tag: ${VERSION}"
    else
      STACK="sandbox"
      TIMESTAMP=$(date -u +%Y%m%d-%H%M%S)
      BRANCH=${GITHUB_REF_NAME//[\/-]/_}
      VERSION=${BRANCH}-${TIMESTAMP}
      echo "Generated sandbox version: ${VERSION}"
    fi

    echo "Deploying to stack: ${STACK} with version: ${VERSION}"
    echo "STACK=${STACK}" >> $GITHUB_OUTPUT
    echo "VERSION=${VERSION}" >> $GITHUB_OUTPUT

# Bad - no logging, hard to debug
- name: "Configure version tag"
  id: configure_version_tag
  run: |
    VERSION=${GITHUB_REF_NAME//[\/-]/_}-$(date -u +%Y%m%d-%H%M%S)
    echo "VERSION=${VERSION}" >> $GITHUB_OUTPUT
```

**Benefits of descriptive logging:**
- Easier debugging when workflows fail
- Better visibility into what's happening during execution
- Helps reviewers understand workflow behavior
- Documents expected values and transformations

### Workflow Naming Conventions

**All workflows MUST have both `name` and `run-name` fields:**

```yaml
name: Infrastructure Provision
run-name: Provision ${{ github.event_name == 'workflow_dispatch' && 'production' || 'sandbox' }} infrastructure
```

**`name` field:**
- Static, descriptive name of the workflow
- Shows in GitHub Actions tab and workflow file list
- Use title case (e.g., "Infrastructure Provision", "Sandbox Deploy")

**`run-name` field:**
- Dynamic, context-specific description shown in GitHub Actions run list
- Use expressions to include environment, version, or other runtime context
- Makes it easy to identify specific runs in the UI

**Examples:**

```yaml
# Infrastructure provision - shows environment
name: Infrastructure Provision
run-name: Provision ${{ github.event_name == 'workflow_dispatch' && 'production' || 'sandbox' }} infrastructure

# Application deployment - shows target environment
name: Deploy
run-name: Deploy to ${{ github.event.inputs.environment }}

# Sandbox deployment - static since always sandbox
name: Sandbox Deploy
run-name: Deploy to sandbox

# Test workflow - shows PR number
name: Test
run-name: Test PR #${{ github.event.pull_request.number }}
```

**When to use dynamic `run-name`:**
- Workflow can run in multiple environments (use variable to show which)
- Workflow has user inputs that affect behavior (show the inputs)
- Workflow behavior changes based on trigger (show what triggered it)

**When NOT to use dynamic `run-name`:**
- Workflow only runs in one context (static description is sufficient)
- For simple workflows like linting or type checking (static name is clear enough)
- When hardcoded values are acceptable (e.g., test, semantic release, code quality checks)

## 7. Version Upgrade Checklist

When upgrading uv or Python versions:

1. **Update all workflow files:** Search for and update ALL occurrences of the version number.
2. **Update `pyproject.toml`:** Ensure `requires-python` is updated if changing Python version.
3. **Regenerate `uv.lock`:** Run `uv lock --upgrade` after version changes.
4. **Test locally:** Verify the new version works with your codebase before pushing.
5. **Update Dockerfiles:** If using Docker, update base images and uv installation scripts.

## 8. Prohibited Patterns

### ❌ DO NOT DO THIS:
```yaml
# Don't hardcode Python version
- uses: actions/setup-python@v5
  with:
    python-version: "3.13"

# Don't use version ranges for uv
- uses: astral-sh/setup-uv@v5
  with:
    version: "latest"

# Don't use uv sync without --frozen in CI/CD
- run: uv sync

# Don't add caching for uv
- uses: actions/cache@v4
  with:
    path: ~/.cache/uv
    key: uv-${{ hashFiles('uv.lock') }}
```

### ✅ DO THIS INSTEAD:
```yaml
# Use python-version-file
- uses: actions/setup-python@v5
  with:
    python-version-file: "pyproject.toml"

# Pin exact uv version
- uses: astral-sh/setup-uv@v5
  with:
    version: "0.7.8"

# Always use --frozen in CI/CD
- run: uv sync --frozen

# No caching needed - uv is fast enough
```

## 9. Workflow Maintenance

### Regular Updates
- **Monthly review:** Check for updates to GitHub Actions used in workflows.
- **Security patches:** Apply security updates promptly when Actions vulnerabilities are announced.
- **Version consistency:** Ensure all repositories use the same uv and Python versions.

### Documentation
- **Comment complex steps:** Add inline comments for non-obvious workflow logic.
- **Keep this document updated:** When patterns change, update this rule file immediately.

---

# Part 2: Version Control & Releases

These guidelines establish the standards for git commit messages, branch naming, and pull request workflows across all projects in our organization. We use [Conventional Commits](https://www.conventionalcommits.org/) to enable automated semantic versioning and changelog generation through `python-semantic-release`.

## 10. Commit Message Format

All commit messages MUST follow the Conventional Commits specification.

### Structure
```
<type>(<scope>): <subject>

[optional body]

[optional footer(s)]
```

### Required Components

#### Type (Required)
The commit type determines the version bump in semantic versioning:

- **feat:** A new feature (triggers MINOR version bump: 1.0.0 → 1.1.0)
- **fix:** A bug fix (triggers PATCH version bump: 1.0.0 → 1.0.1)
- **docs:** Documentation only changes (no version bump)
- **style:** Changes that don't affect code meaning (formatting, whitespace) (no version bump)
- **refactor:** Code changes that neither fix bugs nor add features (no version bump)
- **perf:** Performance improvements (triggers PATCH version bump: 1.0.0 → 1.0.1)
- **test:** Adding or updating tests (no version bump)
- **build:** Changes to build system or dependencies (no version bump)
- **ci:** Changes to CI/CD configuration files and scripts (no version bump)
- **chore:** Other changes that don't modify src or test files (no version bump)

#### Scope (Optional but Recommended)
The scope indicates what part of the codebase is affected:

**Common scopes:**
- `pipeline`: Data pipeline logic
- `cdc`: Change Data Capture functionality
- `gcp`: GCP-related components
- `settings`: Configuration and settings
- `logging`: Logging infrastructure
- `ci`: CI/CD workflows
- `cdk`: AWS CDK infrastructure
- `docker`: Dockerfile changes

**Examples:**
```
feat(pipeline): add event deduplication
fix(logging): correct sentry integration
docs(readme): update installation steps
ci(deploy): migrate workflow to uv
```

#### Subject (Required)
The subject is a brief description of the change:

- **Use imperative mood:** "add feature" not "added feature" or "adds feature"
- **Don't capitalize first letter:** "add feature" not "Add feature"
- **No period at the end:** "add feature" not "add feature."
- **Keep it concise:** Maximum 72 characters
- **Be descriptive:** Explain what changed, not just which file

**Good examples:**
```
feat(pipeline): add CDC event deduplication
fix(cdc): handle null values in replication slot
docs: update pyproject.toml configuration guide
```

**Bad examples:**
```
feat: Update files  # Too vague
Fix bug  # Missing type parentheses, not descriptive
feat(pipeline): Added new feature.  # Wrong tense, capitalized, has period
```

### Optional Components

#### Body (Optional)
Provide additional context about the change:

- Separate from subject with a blank line
- Wrap at 72 characters
- Explain what and why, not how
- Use imperative mood

**Example:**
```
feat(pipeline): add CDC event deduplication

Implement deduplication logic to prevent duplicate CDC events from
being published to Pub/Sub. Uses in-memory cache with LRU eviction
to track recently processed LSN positions.
```

#### Footer (Optional)
Reference issues, breaking changes, or co-authors:

**Breaking changes:**
```
feat(settings)!: change environment variable naming

BREAKING CHANGE: All environment variables now use SCREAMING_SNAKE_CASE
instead of PascalCase. Update your .env files accordingly.
```

**Issue references:**
```
fix(cdc): handle connection timeouts

Closes #123
Fixes #456
```

**Co-authoring:**
```
feat(pipeline): implement new data transformation

Co-authored-by: John Doe <john@example.com>
```

## 11. Breaking Changes

Breaking changes trigger a MAJOR version bump (1.0.0 → 2.0.0).

### Indicating Breaking Changes

**Method 1: Add `!` after type/scope:**
```
feat(api)!: remove deprecated endpoints
```

**Method 2: Use `BREAKING CHANGE:` footer:**
```
feat(settings): change configuration format

BREAKING CHANGE: Configuration now uses YAML instead of JSON.
All existing config.json files must be migrated to config.yaml.
```

### When to Use Breaking Changes
- Removing public APIs or features
- Changing function signatures
- Renaming environment variables
- Changing configuration formats
- Modifying database schemas (for libraries)

## 12. Complete Commit Examples

### Feature Addition
```
feat(pipeline): add retry logic for failed CDC events

Implement exponential backoff retry mechanism for failed Pub/Sub
publish operations. Retries up to 3 times with 2s, 4s, 8s delays.

Closes #142
```

### Bug Fix
```
fix(logging): correct sentry DSN configuration

Previously used wrong DSN for production environment, causing errors
to be sent to staging Sentry project. Now correctly reads from
SENTRY_DSN environment variable.

Fixes #156
```

### Documentation
```
docs: add comprehensive Google-style docstrings to core modules

Add complete docstrings following Google style guide to all core.py files
in bases and components directories. Includes detailed descriptions for:
- Function purposes and behavior
- All parameters with types and descriptions
- Return values and types
- Raised exceptions
- Important notes and caveats
```

### CI/CD Changes
```
ci: migrate deploy workflow to uv

Replace Poetry with uv for faster dependency installation.
Update all workflow steps to use uv sync --frozen.
```

### Dependency Updates
```
build: upgrade google-cloud-pubsub to 2.33.0

Update to latest version for improved performance and bug fixes.
```

### Breaking Change
```
feat(settings)!: migrate to pydantic v2 settings

BREAKING CHANGE: Settings class now uses Pydantic v2 API.
Update all code using `Config` inner class to use `model_config` instead.

Migration guide:
- Replace `class Config:` with `model_config = SettingsConfigDict()`
- Update `env_prefix` to use new format
```

## 13. Branch Naming Conventions

### Standard Format
```
<type>/<ticket-id>-<brief-description>
```

**Examples:**
```
feat/DA-687-migrate-to-uv
fix/DA-701-sentry-integration
docs/DA-688-update-readme
chore/DA-702-upgrade-dependencies
```

### Branch Types
- **feat/**: New features
- **fix/**: Bug fixes
- **docs/**: Documentation updates
- **refactor/**: Code refactoring
- **test/**: Test additions or updates
- **chore/**: Maintenance tasks

### Guidelines
- Always include ticket ID (e.g., DA-687, JIRA-123)
- Use lowercase with hyphens
- Keep description brief but descriptive
- Maximum 50 characters total

## 14. Pull Request Conventions

### PR Title Format
PR titles SHOULD follow the same format as commit messages:

```
<type>(<scope>): <description>
```

**Examples:**
```
feat(pipeline): add CDC event deduplication
fix(logging): correct sentry integration
docs: update installation guide
```

### PR Description Template
```markdown
## Summary
Brief description of what this PR does.

## Changes
- Bullet point list of changes
- Another change
- Yet another change

## Testing
- [ ] Unit tests added/updated
- [ ] Integration tests pass
- [ ] Manual testing completed

## Related Issues
Closes #123
Relates to #456

## Breaking Changes
(If applicable) Describe breaking changes and migration steps.

## Screenshots
(If applicable) Add screenshots for UI changes.
```

## 15. Semantic Release Workflow

### How Semantic Release Works
1. **Analyze commits:** Scans commit messages since last release
2. **Determine version bump:**
   - `feat:` → MINOR bump (1.0.0 → 1.1.0)
   - `fix:` or `perf:` → PATCH bump (1.0.0 → 1.0.1)
   - `BREAKING CHANGE:` or `!` → MAJOR bump (1.0.0 → 2.0.0)
   - Other types → No version bump
3. **Generate changelog:** Creates CHANGELOG.md from commit messages
4. **Create git tag:** Tags the release (e.g., v1.2.3)
5. **Publish release:** Creates GitHub release with notes

### Configuration
Our `python-semantic-release` configuration is in workspace root `pyproject.toml`:

```toml
[tool.semantic_release]
version_variable = ["pyproject.toml:project.version"]
version_toml = ["pyproject.toml:project.version"]
branch = "main"
upload_to_pypi = false
upload_to_release = true
```

**Configuration fields:**
- `version_variable`: Updates the version in pyproject.toml using variable replacement
- `version_toml`: Updates the version in pyproject.toml using TOML parsing
- `branch`: Branch to create releases from (typically "main")
- `upload_to_pypi`: Disabled - we don't publish to PyPI
- `upload_to_release`: Enabled - creates GitHub releases with release notes

### Triggering Releases
Releases are triggered automatically when:
1. Commits are pushed to `main` branch
2. Semantic release workflow runs
3. Commits since last release determine version bump
4. New version is tagged and published

### Manual Release (Rare)
```bash
# Dry run to see what would happen
semantic-release --noop version --print

# Actually perform the release
semantic-release version
semantic-release publish
```

## 16. Best Practices for Commit Messages

### Writing Good Commit Messages

**DO:**
- ✅ Write clear, descriptive subjects
- ✅ Use conventional commit format consistently
- ✅ Include scope when relevant
- ✅ Reference issue numbers in body or footer
- ✅ Explain why a change was made, not just what changed
- ✅ Keep commits focused on a single concern

**DON'T:**
- ❌ Use vague messages like "fix bug" or "update code"
- ❌ Combine unrelated changes in one commit
- ❌ Forget the type prefix (feat, fix, etc.)
- ❌ Use past tense ("added" instead of "add")
- ❌ Make commits too large (prefer smaller, focused commits)

### Commit Frequency

**Commit often:**
- After completing a logical unit of work
- Before switching tasks
- Before refactoring
- When tests pass

**Each commit should:**
- Be atomic (single logical change)
- Pass tests (if applicable)
- Build successfully
- Be potentially revertible

### Squashing vs. Preserving History

**When to squash (combine commits):**
- Multiple "WIP" or "fix typo" commits
- Commits that fix issues in earlier commits in same PR
- Cleaning up before merging to main

**When to preserve commits:**
- Each commit represents a distinct logical change
- Commits have valuable individual context
- History aids in debugging or understanding changes

## 17. Common Mistakes and Fixes

### Mistake 1: Wrong Commit Type
```
# Wrong
update: add new feature

# Correct
feat: add new feature
```

### Mistake 2: Missing Type
```
# Wrong
added CDC event deduplication

# Correct
feat(pipeline): add CDC event deduplication
```

### Mistake 3: Past Tense
```
# Wrong
feat(pipeline): added event deduplication

# Correct
feat(pipeline): add event deduplication
```

### Mistake 4: Capitalized Subject
```
# Wrong
feat(pipeline): Add event deduplication

# Correct
feat(pipeline): add event deduplication
```

### Mistake 5: Period at End
```
# Wrong
feat(pipeline): add event deduplication.

# Correct
feat(pipeline): add event deduplication
```

### Mistake 6: Too Vague
```
# Wrong
fix: update code

# Correct
fix(logging): correct sentry DSN configuration for production
```

## 18. Git Workflow Integration

### Pre-commit Hooks

Pre-commit hooks automatically run before each commit to ensure code quality. All repositories MUST use this standard configuration.

#### Standard Configuration

Create `.pre-commit-config.yaml` at the workspace root:

```yaml
repos:
  - repo: local
    hooks:
      - id: ruff-format
        name: ruff-format
        entry: "uv run ruff format bases components projects update_version.py"
        language: system
        always_run: true
        types: [python]

  - repo: local
    hooks:
      - id: ruff-linter
        name: ruff-linter
        entry: "uv run ruff check --fix bases components projects update_version.py"
        language: system
        always_run: true
        types: [python]

  - repo: local
    hooks:
      - id: pytest
        name: pytest
        entry: "uv run pytest"
        pass_filenames: false
        language: system
        always_run: true
        types: [python]
```

#### Hook Descriptions

| Hook | Purpose | Behavior |
|------|---------|----------|
| **ruff-format** | Formats Python code | Auto-fixes formatting issues |
| **ruff-linter** | Lints and fixes code | Auto-fixes linting issues with `--fix` |
| **pytest** | Runs test suite | Ensures tests pass before commit |

#### Setup Instructions

```bash
# Install pre-commit (included in dev dependency group)
uv sync

# Install the git hooks
uv run pre-commit install

# Run hooks manually on all files (optional)
uv run pre-commit run --all-files
```

#### Handling Hook Failures

If hooks modify files (e.g., formatting), the commit will fail. Stage the changes and retry:

```bash
# After hooks auto-fix files
git add -u
git commit -m "your commit message"
```

### Commit Message Validation

Optionally add `commitlint` to validate commit messages follow conventional format:

```yaml
# Add to .pre-commit-config.yaml
  - repo: https://github.com/compilerla/conventional-pre-commit
    rev: v3.0.0
    hooks:
      - id: conventional-pre-commit
        stages: [commit-msg]
```

## 19. Tools and Resources

### Helpful Commands
```bash
# View commit history with conventional format
git log --oneline --decorate

# View commits since last tag
git log $(git describe --tags --abbrev=0)..HEAD --oneline

# Amend last commit message
git commit --amend

# Interactive rebase to fix commit messages
git rebase -i HEAD~3
```

### Conventional Commits Tools
- **commitizen:** CLI tool for writing conventional commits
- **semantic-release:** Automated version management
- **conventional-changelog:** Generate changelogs from commits

### Resources
- [Conventional Commits Specification](https://www.conventionalcommits.org/)
- [Semantic Versioning](https://semver.org/)
- [Python Semantic Release Docs](https://python-semantic-release.readthedocs.io/)

## 20. Troubleshooting

### Issue: Semantic Release Doesn't Bump Version
**Cause:** No commits with `feat:` or `fix:` since last release.
**Solution:** Ensure commits use correct types that trigger version bumps.

### Issue: Wrong Version Bump
**Cause:** Incorrect commit type (e.g., used `fix:` instead of `feat:`).
**Solution:** Use interactive rebase to fix commit messages before merging to main.

### Issue: Release Failed
**Cause:** Commit message format error or missing permissions.
**Solution:** Check commit messages match conventional format; verify `GH_TOKEN` has correct permissions.

### Issue: Merge Conflicts in Changelog
**Cause:** Multiple PRs merged simultaneously updating CHANGELOG.md.
**Solution:** Let semantic-release regenerate the changelog; don't manually resolve conflicts.
