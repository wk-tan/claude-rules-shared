# Dependencies: uv Package Management & pyproject.toml Configuration

**When to use this file:** Reference this for dependency management, workspace vs project configuration, the critical subset rule, and pyproject.toml standards.

**Related documentation:**
- For Polylith architecture concepts, see [01-setup.md](01-setup.md)
- For deployment patterns, see [05-deployment.md](05-deployment.md)
- For CI/CD workflows using uv, see [06-automation.md](06-automation.md)

These guidelines establish the standards for managing Python dependencies with `uv` and configuring `pyproject.toml` files. Following these rules ensures consistent, reproducible builds and maintainable project configurations that work seamlessly with Polylith architecture.

---

## Table of Contents

1. [Project Metadata Standards](#1-project-metadata-standards)
2. [Workspace vs Project Configuration](#2-workspace-vs-project-configuration)
3. [Dependency Management with uv](#3-dependency-management-with-uv)
4. [Build System Configuration](#4-build-system-configuration)
5. [Polylith Brick Configuration](#5-polylith-brick-configuration)
6. [uv Workspace Configuration](#6-uv-workspace-configuration)
7. [Complete Configuration Examples](#7-complete-configuration-examples)
8. [Common Operations](#8-common-operations)
9. [Version Synchronization](#9-version-synchronization)
10. [Prohibited Patterns](#10-prohibited-patterns)
11. [Troubleshooting Common Issues](#11-troubleshooting-common-issues)
12. [Best Practices](#12-best-practices)

---

## 1. Project Metadata Standards

All `pyproject.toml` files MUST follow PEP 621 standards for project metadata.

### Required Fields

**Workspace root pyproject.toml:**
```toml
[project]
name = "app-data-pipeline"
version = "2.5.0"
description = "Data pipeline for CDC replication"
readme = "README.md"
requires-python = "~=3.13.0"
authors = [
    { name = "Data Engineering Team", email = "de-team@pulsifi.me" }
]
```

**Project-level pyproject.toml:**
```toml
[project]
name = "app-data-pipeline"
version = "2.5.0"
description = "Data pipeline for CDC replication"
requires-python = "~=3.13.0"
authors = [
    { name = "Data Engineering Team", email = "de-team@pulsifi.me" }
]
```

### Field Requirements
- **name:** Use lowercase with hyphens (kebab-case) for project names.
- **version:** Follow semantic versioning (MAJOR.MINOR.PATCH).
- **description:** Brief description of the project's purpose.
- **readme:** Path to README file (typically `"README.md"`). **ONLY** include in workspace root `pyproject.toml`. **NEVER** include in project-level `pyproject.toml` files.
- **requires-python:** Always use tilde operator `~=` for minor version pinning (e.g., `~=3.13.0` allows `>=3.13.0, <3.14.0`).
- **authors:** Use consistent team attribution across all repositories:
  - `{ name = "Data Engineering Team", email = "de-team@pulsifi.me" }`

## 2. Workspace vs Project Configuration

Understanding what belongs in workspace root vs project-level `pyproject.toml` files is critical for maintainability.

### Workspace-Only Configuration

These sections should **ONLY** appear in workspace root `pyproject.toml`:

#### [tool.uv.workspace]
```toml
[tool.uv.workspace]
members = ["projects/*"]
```
- Defines which directories are workspace members
- **Never** add this to project files

#### [dependency-groups] (Shared Groups)
```toml
[dependency-groups]
dev = [
    "polylith-cli>=1.40.0",
    "pre-commit>=4.5.0",
    "pyright>=1.1.407",
    "ruff>=0.14.8",
]
test = [
    "pytest>=9.0.2",
    "pytest-cov>=7.0.0",
    "pytest-env>=1.2.0",
]
release = [
    "pulumi>=3.213.0",
    "pulumi-gcp>=9.6.0",
    "python-semantic-release==10.4.0",
]
```
- Development, test, and release tools are **workspace-level only**
- All projects inherit these groups automatically
- **Never** duplicate these in project files

**Why workspace-only:** These tools (linters, test runners, deployment tools) are used across all projects and should have consistent versions workspace-wide.

#### [tool.ruff] and [tool.pyright]
```toml
[tool.ruff]
line-length = 88
target-version = "py313"

[tool.ruff.format]
quote-style = "double"

[tool.ruff.lint]
select = ["E", "F", "I", "N", "W", "UP"]

[tool.ruff.lint.pydocstyle]
convention = "google"

[tool.pyright]
pythonVersion = "3.13"
typeCheckingMode = "standard"
include = [
    "bases",
    "components",
    "development",
    "projects",
    "update_version.py",
]
```
- Code quality tool configurations are workspace-wide
- **Never** add these to project files

**Why workspace-only:** Ensures consistent linting and type checking across all code.

#### [tool.uv] default-groups
```toml
[tool.uv]
default-groups = "all"
```
- Sets default groups for local development (`uv sync`)
- Projects **inherit** this setting automatically
- **Never** duplicate in project files

**Why workspace-only:** Projects that don't define their own dependency groups inherit workspace settings, making this redundant in project files.

### Project-Specific Configuration

These sections appear in **both** workspace and project `pyproject.toml` files:

#### [project]
```toml
# Workspace root
[project]
name = "bigquery-asset-mgmt"
version = "3.151.0"
description = "BigQuery asset management workspace"
readme = "README.md"
dependencies = [
    "Flask>=3.1.2",
    "functions-framework>=3.10.0",
    "google-cloud-aiplatform>=1.132.0",
    "google-cloud-logging>=3.13.0",
    "ipinfo>=5.3.0",
    "pydantic-settings>=2.12.0",
    # ... all runtime dependencies used by any project
]

# Project file
[project]
name = "data-transformation"
version = "3.151.0"
description = "IP geolocation transformation function"
requires-python = "~=3.13.0"
authors = [
    { name = "Data Engineering Team", email = "de-team@pulsifi.me" }
]
dependencies = [
    "Flask>=3.1.2",
    "functions-framework>=3.10.0",
    "google-cloud-logging>=3.13.0",
    "ipinfo>=5.3.0",
    "pydantic-settings>=2.12.0",
    # ... only dependencies this project needs (subset of workspace)
]
```

**Key differences:**
- Workspace root lists **all** runtime dependencies used anywhere
- Workspace root includes `readme` field; project files **never** include `readme`
- Project files list **only** dependencies that specific project needs
- **CRITICAL:** Project dependencies MUST be a **subset** of workspace root dependencies
- Project files should never introduce dependencies not in workspace root
- This ensures consistent versions across all projects

#### [build-system]
```toml
[build-system]
requires = ["hatchling", "hatch-polylith-bricks"]
build-backend = "hatchling.build"
```
- **Must** appear in both workspace and project files
- Identical in both locations
- Required for building packages

#### [tool.polylith.bricks]
```toml
# Workspace root - paths relative to workspace root
[tool.polylith.bricks]
"bases/asset/data_transformation" = "asset/data_transformation"
"components/asset/logging" = "asset/logging"

# Project file - paths relative to project directory
[tool.polylith.bricks]
"../../bases/asset/data_transformation" = "asset/data_transformation"
"../../components/asset/logging" = "asset/logging"
```

**Key differences:**
- Workspace root defines **all** bricks
- Project files define **only** bricks they use
- Paths are relative to their respective locations

### Configuration NOT Needed in Projects

These sections should be **omitted** from project `pyproject.toml` files:

#### [tool.hatch.build]
```toml
# ❌ DON'T add this to project files
[tool.hatch.build]
dev-mode-dirs = ["../../components", "../../bases", "../../development", "../.."]

[tool.hatch.build.targets.wheel]
packages = []

[tool.hatch.build.hooks.polylith-bricks]
enabled = true

[tool.hatch.metadata]
allow-direct-references = true
```

**Why omit:** The hatch build configuration is only needed in workspace root for local development. Projects don't need this because bricks are either copied via `copy.sh` (Cloud Functions) or `COPY` commands (Docker).

#### [tool.uv] default-groups
```toml
# ❌ DON'T add this to project files
[tool.uv]
default-groups = "all"
```

**Why omit:** Projects inherit this setting from workspace root. Adding it to project files is redundant unless you need to override with a different value.

### Quick Reference: Workspace vs Project Sections

| Section                    | Workspace Root | Project Files | Notes                     |
| -------------------------- | -------------- | ------------- | ------------------------- |
| `[project]`                | ✅ Required     | ✅ Required    | Different content in each |
| `[build-system]`           | ✅ Required     | ✅ Required    | Identical in both         |
| `[dependency-groups]`      | ✅ Required     | ❌ Omit        | Inherited by projects     |
| `[tool.uv.workspace]`      | ✅ Required     | ❌ Never       | Defines workspace         |
| `[tool.uv]` default-groups | ✅ Required     | ❌ Omit        | Inherited by projects     |
| `[tool.ruff]`              | ✅ Required     | ❌ Never       | Workspace-wide config     |
| `[tool.pyright]`           | ✅ Required     | ❌ Never       | Workspace-wide config     |
| `[tool.polylith.bricks]`   | ✅ Required     | ✅ Required    | Different paths in each   |
| `[tool.hatch.build]`       | ✅ Required     | ❌ Omit        | Workspace-only config     |

### Infrastructure Configuration (Special Case)

The `infrastructure/pyproject.toml` file is separate from both workspace and project configurations. It has its own isolated dependency management with Pulumi and GCP provider packages, and is NOT included in workspace members.

**Key points:**
- Located at `infrastructure/pyproject.toml` (same level as `pulumi/`)
- Contains infrastructure-specific dependencies (Pulumi, providers)
- Updated separately via `infrastructure-provision.yaml` workflow
- No `readme` field - documentation is in Claude rules

**For comprehensive documentation, see [07-infrastructure.md](07-infrastructure.md#4-infrastructure-dependencies).**

## 3. Dependency Management with uv

### Dependency Groups (Not Optional Dependencies)
- **Use `[dependency-groups]`:** This is uv's standard format, NOT `[project.optional-dependencies]`.
- **Define only in workspace root:** All projects automatically inherit these groups.

### Standard Dependency Groups

#### dev - Development Tools
```toml
dev = [
    "polylith-cli>=1.40.0",    # Polylith architecture tools
    "pre-commit>=4.5.0",       # Git hooks framework
    "pyright>=1.1.407",        # Type checker
    "ruff>=0.14.8",            # Linter and formatter
]
```
**Used for:** Local development, code quality checks

#### test - Testing Tools
```toml
test = [
    "pytest>=9.0.2",           # Test framework
    "pytest-cov>=7.0.0",       # Coverage plugin
    "pytest-env>=1.2.0",       # Environment variables for tests
    "docker>=7.1.0",           # Container testing
]
```
**Used for:** Running test suites

#### release - Deployment Tools
```toml
release = [
    "pulumi>=3.213.0",              # Infrastructure as Code
    "pulumi-gcp>=9.6.0",            # GCP provider
    "python-semantic-release==10.4.0",  # Automated versioning
]
```
**Used for:** Deploying infrastructure and releasing versions

### Which Group Should I Use?

Use this decision tree when adding a new dependency:

```
Is it needed at runtime in production?
├── YES → Add to [project] dependencies (not a group)
└── NO → Continue below...

Is it for running tests?
├── YES → Add to `test` group
│         Examples: pytest, pytest-cov, faker, responses
└── NO → Continue below...

Is it for deploying/releasing?
├── YES → Add to `release` group
│         Examples: pulumi, semantic-release, twine
└── NO → Continue below...

Is it for local development only?
└── YES → Add to `dev` group
          Examples: ruff, pyright, pre-commit, polylith-cli
```

**Quick reference:**

| Package Type | Group | Examples |
|-------------|-------|----------|
| Runtime libraries | `[project] dependencies` | Flask, pydantic, google-cloud-* |
| Linters, formatters, type checkers | `dev` | ruff, pyright, mypy |
| Test frameworks and helpers | `test` | pytest, pytest-cov, faker |
| CI/CD and deployment tools | `release` | pulumi, semantic-release |

### Version Constraints
- **Use `>=` with locked versions:** All dependencies MUST specify minimum versions that match the locked versions in `uv.lock`.
- **Example:**
  - If `uv.lock` contains `google-cloud-pubsub==2.33.0`
  - Then `pyproject.toml` should have `google-cloud-pubsub>=2.33.0`
- **Rationale:** This allows patch and minor updates while ensuring the tested version is the minimum requirement.

### Lock File Management
- **Always commit `uv.lock`:** The lock file MUST be committed to version control.
- **Use `--frozen` in CI/CD:** Never allow lock file updates during CI/CD builds.
- **Update lock file deliberately:** Run `uv lock` or `uv lock --upgrade` when intentionally updating dependencies.
- **Commands:**
  ```bash
  # Update lock file only (no installation)
  uv lock

  # Update lock file with latest compatible versions
  uv lock --upgrade

  # Update specific package only
  uv lock --upgrade-package <package_name>

  # Install from existing lock file (CI/CD)
  uv sync --frozen
  ```

## 4. Build System Configuration

### Standard Build Backend
All projects MUST use `hatchling` with `hatch-polylith-bricks` for Polylith support.

```toml
[build-system]
requires = ["hatchling", "hatch-polylith-bricks"]
build-backend = "hatchling.build"
```

**Placement:** This section MUST appear in both workspace root and project `pyproject.toml` files.

**Important:** Always place `[build-system]` at the **top** of project `pyproject.toml` files for better readability.

### Hatch Configuration for Polylith (Workspace Only)

```toml
[tool.hatch.build]
dev-mode-dirs = ["components", "bases", "development", "."]

[tool.hatch.build.targets.wheel]
packages = []

[tool.hatch.build.hooks.polylith-bricks]
enabled = true

[tool.hatch.metadata]
allow-direct-references = true
```

**Key points:**
- **dev-mode-dirs:** Allows editable installs of Polylith bricks during development.
- **packages = []:** Prevents hatch from auto-discovering packages; bricks are managed by the plugin.
- **allow-direct-references:** Enables git or URL-based dependencies if needed.
- **Workspace root only:** Projects using copy.sh deployment don't need these sections.

## 5. Polylith Brick Configuration

### Workspace Root pyproject.toml
The root `pyproject.toml` MUST define all bricks in the workspace.

```toml
[tool.polylith.bricks]
"bases/asset/data_transformation" = "asset/data_transformation"
"bases/asset/gemini_text_generation" = "asset/gemini_text_generation"
"components/asset/logging" = "asset/logging"
"components/asset/settings" = "asset/settings"
```

**Format:**
- **Key:** Relative path from workspace root to the brick directory.
- **Value:** Python package name (namespace path).

### Project-level pyproject.toml
Projects reference bricks using relative paths from the project directory.

```toml
[tool.polylith.bricks]
"../../bases/asset/data_transformation" = "asset/data_transformation"
"../../components/asset/logging" = "asset/logging"
"../../components/asset/settings" = "asset/settings"
```

**Important:** Only include bricks that the project actually uses. Don't copy all bricks from workspace root.

## 6. uv Workspace Configuration

### Workspace Root Configuration
```toml
[tool.uv]
default-groups = "all"

[tool.uv.workspace]
members = ["projects/*"]
```

**Key points:**
- **default-groups = "all":** Local development includes all dependency groups (dev, test, release).
- **members:** Glob pattern for workspace projects (typically `projects/*`).

### Project-level Configuration
Projects inherit uv settings from the workspace root.

```toml
# ❌ DON'T add [tool.uv] to project files
# Projects inherit workspace settings automatically
```

**Only override if needed:** Add `[tool.uv]` to project files only if you need different settings than workspace root.

## 7. Complete Configuration Examples

### Example 1: Workspace Root pyproject.toml
```toml
[build-system]
requires = ["hatchling", "hatch-polylith-bricks"]
build-backend = "hatchling.build"

[project]
name = "bigquery-asset-mgmt"
version = "3.151.0"
description = "BigQuery asset management workspace"
readme = "README.md"
requires-python = "~=3.13.0"
authors = [
    { name = "Data Engineering Team", email = "de-team@pulsifi.me" }
]
dependencies = [
    "Flask>=3.1.2",
    "functions-framework>=3.10.0",
    "google-cloud-aiplatform>=1.132.0",
    "google-cloud-logging>=3.13.0",
    "ipinfo>=5.3.0",
    "pydantic-settings>=2.12.0",
]

[dependency-groups]
dev = [
    "polylith-cli>=1.40.0",
    "pre-commit>=4.5.0",
    "pyright>=1.1.407",
    "ruff>=0.14.8",
]
test = [
    "pytest>=9.0.2",
    "pytest-cov>=7.0.0",
    "pytest-env>=1.2.0",
]
release = [
    "pulumi>=3.213.0",
    "pulumi-gcp>=9.6.0",
    "pulumi-google-native>=0.32.0",
    "python-semantic-release==10.4.0",
    "setuptools>=80.9.0",
]

[tool.hatch.build]
dev-mode-dirs = ["components", "bases", "development", "."]

[tool.hatch.build.targets.wheel]
packages = []

[tool.hatch.build.hooks.polylith-bricks]
enabled = true

[tool.hatch.metadata]
allow-direct-references = true

[tool.polylith.bricks]
"bases/asset/data_transformation" = "asset/data_transformation"
"bases/asset/gemini_text_generation" = "asset/gemini_text_generation"
"components/asset/logging" = "asset/logging"
"components/asset/settings" = "asset/settings"

[tool.uv]
default-groups = "all"

[tool.uv.workspace]
members = ["projects/*"]

[tool.ruff]
line-length = 88
target-version = "py313"

[tool.ruff.format]
quote-style = "double"

[tool.ruff.lint]
select = ["E", "F", "I", "N", "W", "UP"]

[tool.ruff.lint.pydocstyle]
convention = "google"

[tool.pyright]
pythonVersion = "3.13"
typeCheckingMode = "standard"
include = [
    "bases",
    "components",
    "development",
    "projects",
    "update_version.py",
]

[tool.pytest.ini_options]
addopts = "--tb=short -v"
testpaths = ["test"]
required_plugins = ["pytest-cov", "pytest-env"]
```

### Example 2: Cloud Function Project pyproject.toml
```toml
[build-system]
requires = ["hatchling", "hatch-polylith-bricks"]
build-backend = "hatchling.build"

[project]
name = "data-transformation"
version = "3.151.0"
description = "Retrieves geographical information for a list of IP addresses."
requires-python = "~=3.13.0"
authors = [
    { name = "Data Engineering Team", email = "de-team@pulsifi.me" }
]
dependencies = [
    "Flask>=3.1.2",
    "functions-framework>=3.10.0",
    "google-cloud-logging>=3.13.0",
    "ipinfo>=5.3.0",
    "pydantic-settings>=2.12.0",
]

[tool.polylith.bricks]
"../../bases/asset/data_transformation" = "asset/data_transformation"
"../../components/asset/logging" = "asset/logging"
"../../components/asset/settings" = "asset/settings"
```

**Key observations:**
- `[build-system]` at the top for readability
- No `readme` field - **ONLY** in workspace root
- No `[dependency-groups]` - inherited from workspace
- No `[tool.hatch.build]` - using copy.sh deployment
- No `[tool.uv]` - inherited from workspace
- No `[tool.ruff]` or `[tool.pyright]` - workspace-wide config
- Only bricks actually used by this project

## 8. Common Operations

### Adding Dependencies

```bash
# Add runtime dependency to workspace root
uv add <package_name>

# Add dependency to specific project
cd projects/my_service && uv add <package_name>

# Add development dependency to workspace
uv add <package_name> --group dev

# Add test dependency
uv add <package_name> --group test

# Add release dependency
uv add <package_name> --group release
```

**Important:** Runtime dependencies can be added to either workspace root or project files. Workspace root dependencies are available to all projects. Project-specific dependencies are only available to that project.

### Removing Dependencies

```bash
# Remove runtime dependency from workspace
uv remove <package_name>

# Remove from specific project
cd projects/my_service && uv remove <package_name>

# Remove from dependency group
uv remove <package_name> --group dev
```

### Updating Dependencies

```bash
# Update all dependencies to latest compatible versions
uv lock --upgrade

# Update specific package
uv lock --upgrade-package <package_name>

# Install updated dependencies
uv sync
```

**Workflow:**
1. Run `uv lock --upgrade` to update `uv.lock`
2. Check the resolved versions in `uv.lock`
3. Update version constraints in `pyproject.toml` to match (optional but recommended)
4. Run `uv sync` to install updated dependencies
5. Test thoroughly
6. Commit both `pyproject.toml` and `uv.lock` together

### Syncing Dependencies

```bash
# Local development (all groups)
uv sync

# CI/CD (frozen, deterministic)
uv sync --frozen

# Specific group only
uv sync --frozen --group release

# No default groups (minimal install)
uv sync --frozen --no-default-groups
```

**When to use `--frozen`:**
- Always in CI/CD pipelines
- When you want to ensure exact versions from lock file
- To prevent accidental lock file updates

**When NOT to use `--frozen`:**
- Local development when you want to update dependencies
- After adding/removing dependencies with `uv add/remove`

## 9. Version Synchronization

### Keeping pyproject.toml and uv.lock in Sync

When `uv.lock` is updated, dependency versions in `pyproject.toml` SHOULD be updated to reflect the locked versions.

**Process:**
1. Run `uv lock` or `uv lock --upgrade` to update the lock file.
2. Inspect `uv.lock` to see the resolved versions.
3. Update `pyproject.toml` dependency constraints to use `>=` with the locked version.

**Example:**
```bash
# After running uv lock, if google-cloud-pubsub resolves to 2.33.0
# Update pyproject.toml from:
google-cloud-pubsub>=2.30.0

# To:
google-cloud-pubsub>=2.33.0
```

**Why synchronize:**
- Documents the tested version
- Ensures new installations get at least the tested version
- Makes version constraints meaningful (not just arbitrary minimums)

**Automation tip:** This process can be automated with a script or git hook to ensure consistency.

## 10. Prohibited Patterns

### ❌ DO NOT DO THIS:

```toml
# ❌ Don't use project.optional-dependencies (old format)
[project.optional-dependencies]
dev = ["ruff>=0.14.8"]

# ❌ Don't use loose version constraints without minimum
dependencies = [
    "requests",  # Missing version constraint
    "pandas>1.0",  # Only lower bound, no tested version
]

# ❌ Don't use poetry as build backend
[build-system]
requires = ["poetry-core"]
build-backend = "poetry.core.masonry.api"

# ❌ Don't hardcode Python version (too strict)
requires-python = "3.13.0"

# ❌ Don't use inconsistent author information
authors = [
    { name = "John Doe", email = "john@example.com" }
]

# ❌ Don't duplicate workspace config in projects
[tool.uv]
default-groups = "all"  # Redundant in project files

# ❌ Don't duplicate dependency groups in projects
[dependency-groups]
dev = ["ruff>=0.14.8"]  # Should only be in workspace root

# ❌ Don't include all bricks in project files
[tool.polylith.bricks]
"../../bases/asset/data_transformation" = "asset/data_transformation"
"../../bases/asset/gemini_text_generation" = "asset/gemini_text_generation"
# Project uses data_transformation but lists both bases

# ❌ Don't add dependencies to projects that aren't in workspace root
# Workspace root dependencies
[project]
dependencies = [
    "Flask>=3.1.2",
    "google-cloud-logging>=3.13.0",
]

# Project dependencies - WRONG! Adds new dependency
[project]
dependencies = [
    "Flask>=3.1.2",
    "google-cloud-logging>=3.13.0",
    "requests>=2.31.0",  # NOT in workspace root - inconsistent versions!
]

# ❌ Don't include readme in project-level pyproject.toml
# Project pyproject.toml - WRONG!
[project]
name = "my-project"
version = "1.0.0"
description = "My project description"
readme = "README.md"  # WRONG! Should only be in workspace root
requires-python = "~=3.13.0"
```

### ✅ DO THIS INSTEAD:

```toml
# ✅ Use dependency-groups (modern format)
[dependency-groups]
dev = ["ruff>=0.14.8"]

# ✅ Use >= with tested versions from uv.lock
dependencies = [
    "requests>=2.31.0",
    "pandas>=2.2.0",
]

# ✅ Use hatchling with polylith support
[build-system]
requires = ["hatchling", "hatch-polylith-bricks"]
build-backend = "hatchling.build"

# ✅ Use tilde operator for minor version pinning
requires-python = "~=3.13.0"

# ✅ Use consistent team attribution
authors = [
    { name = "Data Engineering Team", email = "de-team@pulsifi.me" }
]

# ✅ Omit workspace config from project files
# Projects inherit from workspace root automatically

# ✅ Keep dependency groups only in workspace root
# All projects inherit them

# ✅ Only include bricks the project uses
[tool.polylith.bricks]
"../../bases/asset/data_transformation" = "asset/data_transformation"
"../../components/asset/logging" = "asset/logging"
"../../components/asset/settings" = "asset/settings"

# ✅ Project dependencies must be subset of workspace root
# Workspace root dependencies (includes ALL dependencies)
[project]
dependencies = [
    "Flask>=3.1.2",
    "google-cloud-logging>=3.13.0",
    "requests>=2.31.0",
    "ipinfo>=5.3.0",
]

# Project dependencies (subset of workspace root)
[project]
name = "my-project"
version = "1.0.0"
description = "My project description"
# No readme field - ONLY in workspace root
requires-python = "~=3.13.0"
authors = [
    { name = "Data Engineering Team", email = "de-team@pulsifi.me" }
]
dependencies = [
    "Flask>=3.1.2",
    "google-cloud-logging>=3.13.0",
    "ipinfo>=5.3.0",
    # Only includes dependencies this project needs
    # If new dependency needed, add to workspace root FIRST
]
```

## 11. Troubleshooting Common Issues

### Issue: "Package not found" during build
**Cause:** Missing brick configuration or incorrect paths.
**Solution:** Verify `[tool.polylith.bricks]` paths match actual directory structure.

**Debug steps:**
```bash
# Check if brick paths exist
ls bases/asset/data_transformation
ls components/asset/logging

# Verify relative paths from project directory
cd projects/data_transformation
ls ../../bases/asset/data_transformation
```

### Issue: Dependencies installed but imports fail
**Cause:** Virtual environment not activated or wrong Python interpreter.
**Solution:** Ensure `.venv/bin/python` is being used; check IDE interpreter settings.

**Debug steps:**
```bash
# Verify virtual environment
which python  # Should show .venv/bin/python

# Check installed packages
uv pip list

# Reinstall if needed
uv sync --frozen
```

### Issue: Lock file conflicts in git
**Cause:** Multiple developers running `uv lock` with different package indices or settings.
**Solution:** Use `uv sync --frozen` in development; only update lock file deliberately.

**Best practice:**
- Designate one person to update lock file
- Use `uv sync --frozen` for daily work
- Only run `uv lock` when adding/updating dependencies

### Issue: "Requires python ~=3.13.0 but..." error
**Cause:** System Python version doesn't match project requirements.
**Solution:** Install correct Python version using uv; verify with `python --version`.

**Fix with uv:**
```bash
# Install Python 3.13
uv python install 3.13

# Pin Python version for the project (creates .python-version file)
uv python pin 3.13

# Create virtual environment and sync dependencies
uv venv
uv sync --frozen

# Verify
python --version  # Should show Python 3.13.x
```

### Issue: Cloud Functions deployment fails with "editable requirement" error
**Cause:** `uv export` included `-e .` in requirements.txt
**Solution:** Use `--no-emit-project` flag

**Correct command:**
```bash
uv export --format requirements-txt --no-hashes --no-emit-project -o requirements.txt
```

### Issue: Project can't import workspace dependencies
**Cause:** Project `pyproject.toml` missing dependencies that are only in workspace root
**Solution:** Add required dependencies to project's `[project] dependencies` section

**Why this happens:** Projects don't automatically inherit workspace runtime dependencies, only dependency groups (dev, test, release).

## 12. Best Practices

### Workspace Organization
- **Single source of truth:** Keep all shared dependencies in workspace root `pyproject.toml`.
- **Subset rule (CRITICAL):** Project dependencies MUST be a subset of workspace root dependencies. Never add dependencies to project files that aren't in workspace root.
- **Add to workspace first:** If a project needs a new dependency, add it to workspace root `pyproject.toml` first, then reference it in project file.
- **Consistent versions:** All projects use the same versions from workspace root, preventing version conflicts.
- **Minimal project files:** Keep project `pyproject.toml` files as simple as possible by leveraging workspace inheritance.

### Dependency Hygiene
- **Regular updates:** Schedule monthly dependency updates with `uv lock --upgrade`.
- **Security audits:** Run security scanners (e.g., `pip-audit`) on dependencies regularly.
- **Minimize dependencies:** Only add dependencies that provide significant value; avoid dependency bloat.
- **Document constraints:** Add inline comments explaining why specific versions are pinned.

### Configuration Management
- **Workspace-first approach:** Configure once in workspace root, inherit in projects
- **Override sparingly:** Only add project-level config when truly different from workspace
- **Clean project files:** Remove redundant sections that are inherited from workspace
- **Consistent ordering:** Place `[build-system]` at top of project files for readability

### Documentation
- **Comment complex dependencies:** Add inline comments explaining why specific versions are pinned.
- **Track constraints:** Document version constraints imposed by external services or APIs.

**Example:**
```toml
dependencies = [
    "google-cloud-pubsub>=2.33.0",  # Required for Cloud Functions Gen 2
    "functions-framework>=3.10.0",  # Minimum version supporting Python 3.13
    "pydantic-settings>=2.12.0",    # Pydantic v2 API required
]
```

### Git Workflow
- **Always commit lock file:** Never add `uv.lock` to `.gitignore`.
- **Review lock file changes:** Check `uv.lock` diffs in PRs to catch unexpected version changes.
- **Atomic updates:** Commit `pyproject.toml` and `uv.lock` changes together.
- **Synchronized versions:** Update `pyproject.toml` constraints to match `uv.lock` resolved versions.

### CI/CD Integration
- **Always use `--frozen`:** Ensures deterministic builds in pipelines
- **Separate deployment requirements:** Generate requirements.txt for Cloud Functions separately
- **Validate before commit:** Run `uv sync --frozen` locally to ensure lock file is valid
