# uv Package Management and pyproject.toml Configuration Rules

**When to use this file:** This is the most comprehensive guide for dependency management, workspace vs project configuration, the critical subset rule, deployment patterns (Cloud Functions copy.sh, Docker COPY), and pyproject.toml standards.

**Related documentation:**
- For high-level architecture concepts, see [01-polylith-architecture-and-uv.md](01-polylith-architecture-and-uv.md)
- For CI/CD workflows using uv, see [05-github-actions.md](05-github-actions.md)

These guidelines establish the standards for managing Python dependencies with `uv` and configuring `pyproject.toml` files across all projects in our organization. Following these rules ensures consistent, reproducible builds and maintainable project configurations that work seamlessly with Polylith architecture.

## Table of Contents

1. [Project Metadata Standards](#1-project-metadata-standards)
2. [Workspace vs Project Configuration](#2-workspace-vs-project-configuration)
3. [Dependency Management with uv](#3-dependency-management-with-uv)
4. [Build System Configuration](#4-build-system-configuration)
5. [Polylith Brick Configuration](#5-polylith-brick-configuration)
6. [uv Workspace Configuration](#6-uv-workspace-configuration)
7. [Deployment Configuration for Serverless and Containers](#7-deployment-configuration-for-serverless-and-containers)
8. [Complete Configuration Examples](#8-complete-configuration-examples)
9. [Common Operations](#9-common-operations)
10. [Version Synchronization](#10-version-synchronization)
11. [Prohibited Patterns](#11-prohibited-patterns)
12. [Troubleshooting Common Issues](#12-troubleshooting-common-issues)
13. [Best Practices](#13-best-practices)

## 1. Project Metadata Standards

All `pyproject.toml` files MUST follow PEP 621 standards for project metadata.

### Required Fields
```toml
[project]
name = "app-data-pipeline"
version = "2.5.0"
description = "Data pipeline for CDC replication"
readme = "README.md"
requires-python = "~=3.13.0"
authors = [
    { name = "Data Engineering Team", email = "data_eng@pulsifi.me" }
]
```

### Field Requirements
- **name:** Use lowercase with hyphens (kebab-case) for project names.
- **version:** Follow semantic versioning (MAJOR.MINOR.PATCH).
- **requires-python:** Always use tilde operator `~=` for minor version pinning (e.g., `~=3.13.0` allows `>=3.13.0, <3.14.0`).
- **authors:** Use consistent team attribution across all repositories:
  - `{ name = "Data Engineering Team", email = "data_eng@pulsifi.me" }`

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

[tool.ruff.lint]
select = ["E", "F", "I", "N", "W", "UP"]

[tool.pyright]
pythonVersion = "3.13"
typeCheckingMode = "standard"
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

#### [tool.hatch.build] - When Using copy.sh Deployment
```toml
# ❌ DON'T add this to project files if using copy.sh for deployment
[tool.hatch.build]
dev-mode-dirs = ["../../components", "../../bases", "../../development", "../.."]

[tool.hatch.build.targets.wheel]
packages = []

[tool.hatch.build.hooks.polylith-bricks]
enabled = true

[tool.hatch.metadata]
allow-direct-references = true
```

**Why omit:** When deploying Cloud Functions with `copy.sh` scripts, you're manually copying bricks into the project directory. The hatch build configuration is only needed for local development (workspace root) or when building Python packages for distribution.

**When to include:** Only add these sections to project files if you're building distributable packages (wheels) from individual projects.

#### [tool.uv] default-groups
```toml
# ❌ DON'T add this to project files
[tool.uv]
default-groups = "all"
```

**Why omit:** Projects inherit this setting from workspace root. Adding it to project files is redundant unless you need to override with a different value.

### Quick Reference: Workspace vs Project Sections

| Section | Workspace Root | Project Files | Notes |
|---------|---------------|---------------|-------|
| `[project]` | ✅ Required | ✅ Required | Different content in each |
| `[build-system]` | ✅ Required | ✅ Required | Identical in both |
| `[dependency-groups]` | ✅ Required | ❌ Omit | Inherited by projects |
| `[tool.uv.workspace]` | ✅ Required | ❌ Never | Defines workspace |
| `[tool.uv]` default-groups | ✅ Required | ❌ Omit | Inherited by projects |
| `[tool.ruff]` | ✅ Required | ❌ Never | Workspace-wide config |
| `[tool.pyright]` | ✅ Required | ❌ Never | Workspace-wide config |
| `[tool.polylith.bricks]` | ✅ Required | ✅ Required | Different paths in each |
| `[tool.hatch.build]` | ✅ Required | ⚠️ Optional | Omit if using copy.sh |

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

## 7. Deployment Configuration for Serverless and Containers

Both Cloud Functions and Docker containers require special configuration to deploy Polylith bricks correctly.

### Common Pattern: Copying Polylith Bricks

Both deployment strategies require manually copying bricks from `bases/` and `components/` directories because:
- Deployment environments don't support editable installs (`-e .`)
- Need flat directory structure for runtime
- Want to include only required bricks (minimize deployment size)

### Cloud Functions Deployment

When deploying to Google Cloud Functions, use the `copy.sh` script approach.

#### copy.sh Script Pattern

```bash
#!/bin/bash

# Copy Polylith bricks to project directory for Cloud Functions deployment

# Define the namespace
namespace="asset"

# Define the project name
project="data_transformation"

# Create namespace directory structure
mkdir -p ${namespace}

# Copy only the specific base this project needs
echo "Copying base: ${project}..."
cp -r ../../bases/${namespace}/${project} ${namespace}/

# Copy only the components this project needs
echo "Copying component: logging..."
cp -r ../../components/${namespace}/logging ${namespace}/

echo "Copying component: settings..."
cp -r ../../components/${namespace}/settings ${namespace}/

echo "Polylith bricks copied successfully to projects/${project}/"
```

**Key principles:**
- Copy only bricks listed in project's `[tool.polylith.bricks]`
- Create namespace directory structure (e.g., `asset/`)
- Execute before `uv export` in CI/CD workflows

#### Requirements.txt Generation

Use `uv export` with specific flags for Cloud Functions compatibility:

```bash
uv export --format requirements-txt --no-hashes --no-emit-project -o requirements.txt
```

**Flag explanations:**
- `--format requirements-txt`: Generate pip-compatible format
- `--no-hashes`: Cloud Functions doesn't require hash verification
- `--no-emit-project`: Exclude `-e .` editable install reference
- `-o requirements.txt`: Output file name

#### .gitignore Configuration

Add generated files to `.gitignore` in project directories:

```gitignore
# Cloud Functions deployment artifacts
asset/
requirements.txt
```

**Why:** These files are generated during CI/CD and should not be committed.

#### Deployment Dependencies

Cloud Functions require `functions-framework`:

```toml
[project]
dependencies = [
    "functions-framework>=3.10.0",  # Required for Cloud Functions HTTP server
    # ... other dependencies
]
```

#### Entry Point Shim Pattern

Cloud Functions expects to import your function directly from `main.py`, but Polylith bases live in `{namespace}/{base_name}/core.py`. Use a shim to bridge this gap.

**Pattern:**
```python
# projects/{project_name}/main.py
"""Cloud Function entry point for {service description}."""

from {namespace}.{base_name}.core import {function_name}

__all__ = ["{function_name}"]
```

**Example:**
```python
# projects/data_transformation/main.py
"""Cloud Function entry point for IP geolocation service."""

from asset.data_transformation.core import get_ip_geo_info

__all__ = ["get_ip_geo_info"]
```

**Key points:**
- Shim file must be named `main.py` (Cloud Functions convention)
- Import the actual function from the base's `core.py`
- Export via `__all__` for clarity
- Include docstring describing the service
- Shim is committed to git (unlike copied bricks)

**Why this pattern:**
- Cloud Functions runtime expects `main.{function_name}` import path
- Polylith base implements function in `{namespace}/{base_name}/core.py`
- Shim keeps Cloud Functions happy while maintaining Polylith structure
- Allows base to remain reusable and testable independently

### Docker Container Deployment

When deploying containerized applications, use Dockerfile COPY commands to include Polylith bricks.

#### Dockerfile Pattern

```dockerfile
FROM python:3.13-slim@sha256:a93b51c5acbd72f9d28fd811a230c67ed59bcd727ac08774dee4bbf64b7630c7

RUN apt-get update && \
    apt-get upgrade -y && \
    apt-get install -y --no-install-recommends ca-certificates curl

# Install uv with specific version for reproducibility
ADD https://astral.sh/uv/0.7.8/install.sh /uv-installer.sh
RUN sh /uv-installer.sh && rm /uv-installer.sh

ENV PATH="/root/.local/bin/:$PATH"

WORKDIR /app

# Copy project configuration and lock file
COPY projects/app_cdc_ecs/pyproject.toml ./
COPY uv.lock ./

# Install dependencies without installing the project itself
RUN uv sync --frozen --no-default-groups --no-install-project

# Copy only the Polylith bricks this project needs
COPY components/app_data_pipeline/gcp app_data_pipeline/gcp
COPY components/app_data_pipeline/logging app_data_pipeline/logging
COPY components/app_data_pipeline/settings app_data_pipeline/settings
COPY components/app_data_pipeline/pipeline app_data_pipeline/pipeline
COPY bases/app_data_pipeline/app_cdc_ecs app_data_pipeline/app_cdc_ecs

# Move base entry point to main.py if needed
RUN mv app_data_pipeline/app_cdc_ecs/core.py main.py

# Ensure virtual environment is in PATH
ENV PATH="/app/.venv/bin:$PATH"
CMD ["python", "main.py"]
```

**Key principles:**
- Install uv with pinned version (0.7.8) for reproducibility
- Copy `pyproject.toml` and `uv.lock` first (Docker layer caching)
- Use `uv sync --frozen --no-default-groups --no-install-project` for minimal install
- Copy only bricks listed in project's `[tool.polylith.bricks]`
- Create namespace directory structure during COPY (e.g., `app_data_pipeline/`)
- Move entry point to `main.py` if application expects it

#### Why --no-install-project?

The `--no-install-project` flag tells uv to:
- Install all dependencies from `pyproject.toml`
- Skip installing the project itself as an editable package
- This is necessary because we're manually copying bricks instead

#### Multi-stage Build Pattern (Optional)

For smaller production images, use multi-stage builds:

```dockerfile
# Build stage
FROM python:3.13-slim AS builder

ADD https://astral.sh/uv/0.7.8/install.sh /uv-installer.sh
RUN sh /uv-installer.sh && rm /uv-installer.sh

ENV PATH="/root/.local/bin/:$PATH"

WORKDIR /app

COPY projects/app_cdc_ecs/pyproject.toml ./
COPY uv.lock ./

RUN uv sync --frozen --no-default-groups --no-install-project

# Runtime stage
FROM python:3.13-slim

WORKDIR /app

# Copy virtual environment from builder
COPY --from=builder /app/.venv /app/.venv

# Copy Polylith bricks
COPY components/app_data_pipeline/gcp app_data_pipeline/gcp
COPY components/app_data_pipeline/logging app_data_pipeline/logging
COPY components/app_data_pipeline/settings app_data_pipeline/settings
COPY components/app_data_pipeline/pipeline app_data_pipeline/pipeline
COPY bases/app_data_pipeline/app_cdc_ecs app_data_pipeline/app_cdc_ecs

RUN mv app_data_pipeline/app_cdc_ecs/core.py main.py

ENV PATH="/app/.venv/bin:$PATH"
CMD ["python", "main.py"]
```

### Deployment Strategy Comparison

| Aspect | Cloud Functions | Docker Container |
|--------|----------------|------------------|
| Brick copying | `copy.sh` script | Dockerfile COPY commands |
| Dependency install | `uv export` → requirements.txt | `uv sync --frozen --no-install-project` |
| When to copy | During CI/CD before deployment | During Docker image build |
| Generated files | `asset/`, `requirements.txt` | Baked into image layers |
| .gitignore | Yes (exclude generated files) | Not applicable (in image only) |

### Best Practices for Both

1. **Only copy required bricks:** Check project's `[tool.polylith.bricks]` section
2. **Maintain namespace structure:** Preserve the namespace package layout
3. **Pin uv version:** Use exact version (0.7.8) for reproducibility
4. **Use --frozen flag:** Ensure deterministic builds from lock file
5. **Layer optimization:** For Docker, copy dependencies before source code for better caching

## 8. Complete Configuration Examples

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
    { name = "Data Engineering Team", email = "data_eng@pulsifi.me" }
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

[tool.ruff.lint]
select = ["E", "F", "I", "N", "W", "UP"]

[tool.pyright]
pythonVersion = "3.13"
typeCheckingMode = "standard"
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
readme = "README.md"
requires-python = "~=3.13.0"
authors = [
    { name = "Data Engineering Team", email = "data_eng@pulsifi.me" }
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
- No `[dependency-groups]` - inherited from workspace
- No `[tool.hatch.build]` - using copy.sh deployment
- No `[tool.uv]` - inherited from workspace
- No `[tool.ruff]` or `[tool.pyright]` - workspace-wide config
- Only bricks actually used by this project

## 9. Common Operations

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

## 10. Version Synchronization

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

## 11. Prohibited Patterns

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
    { name = "Data Engineering Team", email = "data_eng@pulsifi.me" }
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
dependencies = [
    "Flask>=3.1.2",
    "google-cloud-logging>=3.13.0",
    "ipinfo>=5.3.0",
    # Only includes dependencies this project needs
    # If new dependency needed, add to workspace root FIRST
]
```

## 12. Troubleshooting Common Issues

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
**Solution:** Install correct Python version using pyenv or conda; verify with `python --version`.

**Fix with pyenv:**
```bash
pyenv install 3.13
pyenv local 3.13
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

## 13. Best Practices

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
