# Setup: Polylith Architecture & Tool Configuration

**When to use this file:** Start here when setting up a new project, understanding Polylith architecture concepts, or checking tool versions.

**Related documentation:**
- For dependency management details, see [03-dependencies.md](03-dependencies.md)
- For Python coding standards, see [02-development.md](02-development.md)
- For deployment patterns, see [05-deployment.md](05-deployment.md)

---

## Table of Contents

1. [Infrastructure and Application Separation](#1-infrastructure-and-application-separation)
2. [Understanding Polylith Core Concepts](#2-understanding-polylith-core-concepts)
3. [Monorepo Structure and Organization](#3-monorepo-structure-and-organization)
4. [Code Sharing and Reusability](#4-code-sharing-and-reusability)
5. [Testing Strategy](#5-testing-strategy)
6. [Tool Versions](#6-tool-versions)
7. [Developer Workflow](#7-developer-workflow)

---

## 1. Infrastructure and Application Separation

A fundamental architectural principle in Polylith repositories is the **separation of infrastructure definitions from application code**. This separation is critical for managing different change frequencies and deployment lifecycles.

Infrastructure changes rarely (weeks to months) while application code changes frequently (multiple times per day). Keeping them separate enables:
- Fast application deployments (2-3 minutes vs 15-20 minutes)
- Independent rollbacks and version control
- Clear ownership boundaries (platform team vs application team)
- Reduced risk when deploying application changes

**For comprehensive documentation on infrastructure patterns, see [07-infrastructure.md](07-infrastructure.md):**
- [Infrastructure and Application Separation](07-infrastructure.md#1-infrastructure-and-application-separation) - Core principles and directory structure
- [Two-Stack Pattern](07-infrastructure.md#2-two-stack-pattern) - Stack 1 (Pulumi) and Stack 2 (Application) details
- [Pulumi IaC Patterns](07-infrastructure.md#5-pulumi-iac-patterns) - Configuration, naming conventions, resource patterns
- [Kustomize for Cloud Run](07-infrastructure.md#6-kustomize-for-cloud-run) - Services, Jobs, and common patterns

---

## 2. Understanding Polylith Core Concepts

Polylith provides a structured approach to building software by organizing code into reusable, independent "bricks" within a monorepo. This architecture helps manage the complexities of both monolithic and microservice approaches by promoting code sharing and clear separation of concerns.

### Core Concepts

**Workspace**
- The top-level directory of your monorepo
- Houses all Polylith building blocks and configuration
- Defined in `[tool.polylith]` within `workspace.toml` at the workspace root

**Bricks (Components & Bases)**
- Fundamental units of code in Polylith, acting like LEGO bricks
- Essentially Python namespace packages

  **Python Namespace Packages and `__init__.py`:**

  Polylith bricks use Python's namespace package feature (PEP 420, Python 3.3+):

  - **Namespace level** (e.g., `model_context/`): Does NOT need `__init__.py`
    - Python automatically treats directories without `__init__.py` as namespace packages
    - Allows splitting a namespace across multiple directories (bases/, components/)
  - **Brick level** (e.g., `model_context/settings/`): DOES need `__init__.py`
    - Polylith CLI automatically creates `__init__.py` when creating bricks
    - Makes the brick a regular Python package that can be imported

  **In Dockerfiles:**
  ```dockerfile
  # ✅ Correct - just create the namespace directory
  RUN mkdir model_context

  # ❌ Wrong - don't manually add __init__.py at namespace level
  RUN mkdir model_context && touch model_context/__init__.py
  ```

  **Why this matters:**
  - Namespace packages allow `from model_context.settings.core import Settings` to work
  - The `settings` brick has `__init__.py` (created by Polylith CLI)
  - The `model_context` namespace directory doesn't need one
  - Adding `__init__.py` at namespace level won't break anything, but it's unnecessary

  **Components:**
  - High-level, reusable building blocks encapsulating specific functionalities or domain logic
  - Designed for single responsibility, statelessness, and purity
  - Hide implementation details and expose only their signatures
  - Form the core reusable logic across projects
  - Examples: `logging`, `settings`, `pubsub`, `bigquery`

  **Bases:**
  - Special bricks that serve as entry points for applications or services
  - Examples: FastAPI routes, CLI commands, serverless functions
  - Act as thin layers that compose components to deliver specific functionality
  - Examples: `data_transformation`, `api_server`, `cli_tool`

**Projects:**
- Define deployable units (e.g., microservice, web application, CLI tool, library)
- Specify which components and bases are required for their functionality
- A monorepo can contain multiple projects
- Each project has its own `pyproject.toml` with specific runtime dependencies

### Polylith Benefits

1. **Code Reusability:** Share components across multiple projects
2. **Clear Boundaries:** Well-defined responsibilities and interfaces
3. **Independent Testing:** Test components and bases in isolation
4. **Flexible Deployment:** Mix and match bricks for different deployments

---

## 3. Monorepo Structure and Organization

Maintaining a strict and consistent monorepo structure is paramount. This consistency enables automated code generation, linting, testing, and deployment scripts.

### Standard Directory Structure

Based on [`python-polylith-example-uv`](https://github.com/DavidVujic/python-polylith-example-uv/tree/main):

```
workspace-root/
├── pyproject.toml          # Workspace-level config and all dependencies
├── uv.lock                 # Locked dependency versions
├── workspace.toml          # Polylith workspace metadata
├── infrastructure/         # Infrastructure definitions (separate from app code)
│   ├── pyproject.toml      # Infrastructure dependencies (Pulumi, providers)
│   ├── cloudrun/           # Cloud Run Kustomize manifests (base + overlays)
│   ├── cloudfunction/      # Cloud Function deployment configs
│   └── pulumi/             # Pulumi IaC stacks (service accounts, storage, IAM)
├── bases/
│   └── {workspace_name}/   # Namespace directory
│       ├── api_server/     # Base for API service
│       ├── cli_tool/       # Base for CLI application
│       └── data_processor/ # Base for data pipeline
├── components/
│   └── {workspace_name}/   # Namespace directory
│       ├── logging/        # Logging component
│       ├── settings/       # Configuration component
│       ├── gcp/            # Optional grouping by cloud provider
│       │   ├── pubsub/
│       │   └── bigquery/
│       └── aws/            # Optional grouping
│           └── s3/
├── development/            # REPL-driven development, notebooks, scratch code
│   └── {workspace_name}/   # Development utilities and experimental code
├── projects/
│   ├── api_service/        # Deployable API project
│   │   ├── pyproject.toml
│   │   └── main.py
│   └── data_pipeline/      # Deployable pipeline project
│       ├── pyproject.toml
│       └── main.py
└── test/                   # Tests at workspace root
    ├── components/         # Component tests mirror components/ structure
    │   └── {workspace_name}/
    └── bases/              # Base tests mirror bases/ structure
        └── {workspace_name}/
```

### Directory Rules

**Top-level `pyproject.toml`:**
- Defines global development dependencies (linters, formatters, test runners)
- Contains all runtime dependencies used by any project
- Configures `uv` for the entire monorepo

**`infrastructure/` directory:**
- Contains infrastructure definitions separate from application code
- Organized by deployment target (cloudrun, cloudfunction, pulumi)
- Manages service configurations, IAM policies, networking, storage
- Changes infrequently and requires deliberate deployment
- Examples:
  - `cloudrun/`: Kustomize manifests for Cloud Run services
  - `cloudfunction/`: Deployment configurations for Cloud Functions
  - `pulumi/`: Infrastructure as Code for GCP resources (service accounts, buckets, IAM)

**`bases/{workspace_name}/` directory:**
- Contains all base bricks
- Each base resides in a distinct directory within workspace namespace
- Example: If workspace is `example`, bases at `bases/example/my_api_base`

**`components/{workspace_name}/` directory:**
- Contains all component bricks
- Each component in a distinct directory within workspace namespace
- Optional additional layer for logical grouping (cloud provider, domain area, service type)
- Examples:
  - Flat: `components/example/user_management`, `components/example/product_catalog`
  - Grouped: `components/example/gcp/logging`, `components/example/aws/s3`

**`development/` directory:**
- Workspace for REPL-driven development and experimentation
- Contains Jupyter notebooks, scratch code, and development utilities
- Has access to all components, bases, and dependencies
- Configured in `dev-mode-dirs` for interactive development
- Not deployed to production - development only
- Examples: `development/notebooks/`, `development/experiments/`

**`projects/` directory:**
- Contains all project definitions
- Each project has its own `pyproject.toml`
- Specifies unique runtime dependencies and consumed bricks
- Deployment configuration files (per deployment target):
  - `Dockerfile` - For containerized deployments (Docker, Kubernetes, Cloud Run, ECS)
  - `copy.sh` - For Cloud Functions deployments
  - `main.py` - Entry point shim (Cloud Functions or standalone apps)
- Examples: `projects/api_service`, `projects/data_processor`

**`test/` directory:**
- Located at workspace root (NOT inside component/base directories)
- Mirrors the structure of `components/` and `bases/`
- Created automatically by Polylith CLI when creating bricks
- Contains `test_core.py` files for each component and base
- Used for unit tests (components) and integration tests (bases)
- Shared test infrastructure placed at `test/` root level:
  - `conftest.py` - Workspace-wide pytest fixtures
  - `fixtures/` - Shared test data and mock responses
  - `emulators/` - Emulator setup scripts (Pub/Sub, BigQuery, etc.)
- Note: pytest configuration (`[tool.pytest.ini_options]`) lives in workspace root `pyproject.toml`, not in test folder

### Why Structure Matters

**This consistency is not merely for readability; it is a fundamental enabler for:**
- Automated code generation
- Linting and type checking
- Testing scripts
- Deployment automation
- IDE navigation and autocomplete
- Clear separation between infrastructure and application concerns

---

## 4. Code Sharing and Reusability

The primary benefit of Polylith is fostering code reuse.

### Best Practices

**Component-first Thinking**
- Prioritize creating new functionalities as components
- Design components to be independent and potentially reusable
- Think: "Will another project need this functionality?"

**Avoid Duplication**
- Actively seek to reuse existing components
- Don't copy-paste logic between projects
- Extract common patterns into shared components

**Clear Boundaries**
- Components should have well-defined responsibilities
- Minimize implicit dependencies between components
- Use explicit interfaces (function signatures, type hints)

### Example: Component Design

```python
# Good: Reusable logging component
# components/example/logging/core.py
import logging
from typing import Any

def log_event(event: str, payload: dict[str, Any] | None = None) -> None:
    """Log structured event with optional payload.

    Args:
        event: Past-tense description of what happened
        payload: Additional context data
    """
    logging.info(msg={"event": event, "payload": payload or {}})
```

This component can be used by multiple bases and projects.

---

## 5. Testing Strategy

Polylith's modularity naturally supports robust testing.

### Test Levels

**Component-level Testing (Unit Tests)**
- Write comprehensive unit tests for each component
- Ensure individual functionality is correct and isolated
- Mock external dependencies
- Fast execution, high coverage

**Base-level Testing (Integration Tests)**
- Test bases to ensure they correctly compose components
- Verify entry-point logic (API request/response handling, CLI argument parsing)
- Test component integration

**Project-level Testing (End-to-End Tests)**
- Implement integration or E2E tests for each deployable project
- Verify the assembled system functions as expected
- Test against real or simulated external services

### Testing Tools

**Consistent Tooling:**
- Use workspace-level testing tools (e.g., `pytest` installed at root via `uv`)
- Ensures consistent test execution and reporting across the monorepo
- Shared test configuration in workspace `pyproject.toml`

**Test Isolation:**
- Test components and bases independently before integrating into projects
- Use fixtures and mocks to isolate dependencies
- Fast feedback loops during development

**Test Infrastructure:**
- **Workspace-wide fixtures:** `test/conftest.py` - Shared across all tests
- **Test data:** `test/fixtures/` - Reusable test data and mock responses
- **Emulators:** `test/emulators/` - Setup scripts for Pub/Sub, BigQuery, etc.
- **Component-specific fixtures:** `test/components/{namespace}/conftest.py`
- **Base-specific fixtures:** `test/bases/{namespace}/conftest.py`
- **pytest configuration:** Defined in workspace root `pyproject.toml` under `[tool.pytest.ini_options]`
  - Standard options: `addopts = "--tb=short -v"`, `testpaths = ["test"]`, `required_plugins = ["pytest-cov", "pytest-env"]`
  - See [04-testing.md](04-testing.md#pytest-configuration) for complete configuration details

---

## 6. Tool Versions

Current versions used across the project (source of truth).

### Standard Versions

- **Python:** `3.13` (defined in `pyproject.toml` `requires-python` field)
- **uv:** `0.7.8` (see [06-automation.md](06-automation.md) for CI/CD usage)
- **hatchling:** Latest compatible (defined in `[build-system]`)
- **hatch-polylith-bricks:** Latest compatible (defined in `[build-system]`)

### Version Upgrade Checklist

When upgrading versions:

1. **Update `pyproject.toml`** for Python version
   ```toml
   [project]
   requires-python = "~=3.13.0"
   ```

2. **Update CI/CD workflows** for uv version (see [06-automation.md](06-automation.md))
   ```yaml
   - uses: astral-sh/setup-uv@v5
     with:
       version: "0.7.8"
   ```

3. **Regenerate lock file**
   ```bash
   uv lock --upgrade
   ```

4. **Test thoroughly** before committing
   - Run full test suite
   - Test local builds
   - Verify deployment pipelines

5. **Update documentation** if needed

---

## 7. Developer Workflow

### Daily Development

**Single Virtual Environment**
- Benefit from a single virtual environment for the entire monorepo
- All code runs with same dependency versions, linters, and type checkers
- Create once at workspace root: `uv venv`
- Sync dependencies: `uv sync`

**Consistent Tooling**
- Centralize configuration for linting (`ruff`), type checking (`pyright`), formatting
- Configuration in workspace root `pyproject.toml`
- Ensures consistency across all bricks and projects

**Running Commands**
- Use `uv run` for executing commands within virtual environment context
- Examples:
  ```bash
  uv run pytest test/
  uv run ruff check .
  uv run pyright
  ```

**IDE Configuration**
- Configure IDEs (VSCode, PyCharm) to recognize workspace virtual environment
- Enable Polylith structure awareness for seamless navigation
- Set up auto-completion and debugging

### Development Principles

**Workspace-first Approach**
- Always add dependencies to workspace root first
- Then add to projects as subsets
- See [03-dependencies.md](03-dependencies.md) for details

**Test in Isolation**
- Test components and bases independently before integrating into projects
- Run component tests frequently during development
- Catch issues early

**Incremental Development**
- Build components incrementally
- Test each component before moving to next
- Compose into bases only after component testing

---

## Quick Start Checklist

Setting up a new Polylith project:

- [ ] Initialize workspace with `pyproject.toml` and `workspace.toml`
- [ ] Create virtual environment: `uv venv`
- [ ] Set up directory structure: `infrastructure/`, `bases/`, `components/`, `development/`, `projects/`, `test/`
- [ ] Configure workspace-level tools (ruff, pyright) in `pyproject.toml`
- [ ] Create `infrastructure/` directory for infrastructure definitions
- [ ] Create first component with tests
- [ ] Create first base that uses the component
- [ ] Create first project that combines base and component
- [ ] Set up CI/CD workflows (see [06-automation.md](06-automation.md))
- [ ] Document component APIs and project purposes

---

## Examples

### Example: Workspace Structure with Infrastructure

```
data-engineering-workspace/
├── pyproject.toml
├── uv.lock
├── workspace.toml
├── infrastructure/              # Infrastructure separate from app code
│   ├── pyproject.toml          # Infrastructure dependencies (Pulumi, providers)
│   ├── cloudrun/                # Cloud Run deployment configs (Kustomize)
│   │   ├── base/
│   │   │   ├── service.yaml
│   │   │   └── kustomization.yaml
│   │   └── overlays/
│   │       ├── sandbox/
│   │       │   └── kustomization.yaml
│   │       └── production/
│   │           └── kustomization.yaml
│   ├── cloudfunction/           # Cloud Function deployment configs (optional)
│   └── pulumi/                  # Infrastructure as Code (GCP resources)
│       ├── __main__.py
│       ├── helpers/
│       │   ├── naming.py
│       │   └── locations.py
│       └── Pulumi.{stack}.yaml
├── bases/
│   └── pipeline/
│       ├── cdc_processor/
│       └── etl_runner/
├── components/
│   └── pipeline/
│       ├── logging/
│       ├── settings/
│       ├── gcp/
│       │   ├── pubsub/
│       │   └── bigquery/
│       └── transformations/
├── development/
│   └── pipeline/
│       ├── notebooks/           # Jupyter notebooks for exploration
│       └── experiments/         # Experimental code and prototypes
├── projects/
│   ├── cdc_pipeline/
│   └── batch_etl/
└── test/
    ├── conftest.py          # Workspace-wide fixtures
    ├── fixtures/            # Shared test data
    ├── emulators/           # Emulator setup scripts
    ├── components/
    │   └── pipeline/
    └── bases/
        └── pipeline/
```

### Example: Component Directory

```
components/pipeline/gcp/pubsub/
├── __init__.py
└── core.py          # Main implementation
```

### Example: Base Directory

```
bases/pipeline/cdc_processor/
├── __init__.py
└── core.py          # Entry point
```

### Example: Test Directory (Workspace Root)

```
test/
├── conftest.py              # Workspace-wide pytest fixtures
├── fixtures/                # Shared test data
│   ├── sample_data.json
│   └── mock_responses.py
├── emulators/               # Emulator setup scripts
│   ├── pubsub_emulator.py
│   └── bigquery_emulator.py
├── components/
│   └── pipeline/
│       ├── conftest.py      # Component-specific fixtures
│       └── gcp/
│           └── pubsub/
│               └── test_core.py  # Component unit tests
└── bases/
    └── pipeline/
        ├── conftest.py      # Base-specific fixtures
        └── cdc_processor/
            └── test_core.py      # Base integration tests
```

### Example: Project Directory

```
projects/cdc_pipeline/
├── pyproject.toml   # Project dependencies and brick references
├── main.py          # Entry point shim (optional)
├── copy.sh          # Deployment script (if Cloud Functions)
└── Dockerfile       # Docker configuration (if containerized deployment)
```

**Deployment files vary by target platform:**
- **Cloud Functions:** `copy.sh` and `main.py` (entry point shim)
- **Docker/Kubernetes/Cloud Run/ECS:** `Dockerfile`
- **Both:** `pyproject.toml` (always required)
