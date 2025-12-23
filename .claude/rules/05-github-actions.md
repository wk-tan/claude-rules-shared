# GitHub Actions CI/CD Workflow Rules

**When to use this file:** Reference this when creating or maintaining GitHub Actions workflows, setting up Python/uv in CI/CD, or troubleshooting workflow issues.

**Related documentation:**
- For dependency management details, see [06-uv-and-pyproject.md](06-uv-and-pyproject.md)
- For tool versions (Python, uv), see [01-polylith-architecture-and-uv.md](01-polylith-architecture-and-uv.md#7-tool-versions)
- For commit conventions in workflows, see [07-git-commit-conventions.md](07-git-commit-conventions.md)

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
          --cov=components/                    \
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
