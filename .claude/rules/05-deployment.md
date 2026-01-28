# Deployment: Cloud Functions & Docker Patterns

**When to use this file:** Reference this for deploying Polylith projects to Google Cloud Functions, Docker containers, or other platforms.

> **IMPORTANT:** This file covers **Stack 2: Application Deployment** patterns (Cloud Functions, Docker, Cloud Run). For **Stack 1: Infrastructure Foundation** (Pulumi provisioning), see [01-setup.md](01-setup.md#1-infrastructure-and-application-separation) and [06-automation.md](06-automation.md#example-4-infrastructure-provision-workflow).

**Related documentation:**
- For Polylith architecture, see [01-setup.md](01-setup.md)
- For dependency management, see [03-dependencies.md](03-dependencies.md)
- For CI/CD workflows, see [06-automation.md](06-automation.md)
- For infrastructure/application separation, see [01-setup.md](01-setup.md#1-infrastructure-and-application-separation)
- For Kustomize patterns (Cloud Run Services, Jobs), see [07-infrastructure.md](07-infrastructure.md#6-kustomize-for-cloud-run)

---

## Table of Contents

1. [Deployment Strategy Decision Tree](#1-deployment-strategy-decision-tree)
2. [Cloud Functions Deployment](#2-cloud-functions-deployment)
3. [Docker Container Deployment](#3-docker-container-deployment)
4. [Cloud Run Configuration (Kustomize)](#4-cloud-run-configuration-kustomize)
5. [Deployment Strategy Comparison](#5-deployment-strategy-comparison)
6. [Best Practices](#6-best-practices)

---

## 1. Deployment Strategy Decision Tree

Projects are the deployable artifacts. The deployment strategy varies based on the target platform.

### When to Use Each Strategy

**Use hatchling auto-assembly when:**
- Deploying to environments that support editable installs
- Building Python packages for PyPI distribution
- Running in containerized services with full build control
- Local development and testing

**Use copy.sh scripts when:**
- Deploying to Google Cloud Functions
- Need flat directory structure with requirements.txt
- Want explicit control over which bricks are deployed
- Minimizing deployment package size is critical

**Use Dockerfile COPY commands when:**
- Deploying containerized applications to Cloud Run, ECS, Kubernetes
- Building Docker images for production
- Want layer caching benefits for faster builds
- Need reproducible production images

### Python Package Distribution

For distributing libraries or packages:

- **Building:** Use hatchling with hatch-polylith-bricks to build wheel files
- **Command:** `uv build` or `python -m build`
- **Publishing:** `uv publish` to PyPI or private package index
- The `pyproject.toml` within each project directory dictates what gets packaged

---

## 2. Cloud Functions Deployment

When deploying to Google Cloud Functions, use the `copy.sh` script approach.

### Common Pattern: Copying Polylith Bricks

Both deployment strategies require manually copying bricks from `bases/` and `components/` directories because:
- Deployment environments don't support editable installs (`-e .`)
- Need flat directory structure for runtime
- Want to include only required bricks (minimize deployment size)

### copy.sh Script Pattern

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

### Requirements.txt Generation

Use `uv export` with specific flags for Cloud Functions compatibility:

```bash
uv export --format requirements-txt --no-hashes --no-emit-project -o requirements.txt
```

**Flag explanations:**
- `--format requirements-txt`: Generate pip-compatible format
- `--no-hashes`: Cloud Functions doesn't require hash verification
- `--no-emit-project`: Exclude `-e .` editable install reference
- `-o requirements.txt`: Output file name

### .gitignore Configuration

Add generated files to `.gitignore` in project directories:

```gitignore
# Cloud Functions deployment artifacts
asset/
requirements.txt
```

**Why:** These files are generated during CI/CD and should not be committed.

### Deployment Dependencies

Cloud Functions require `functions-framework`:

```toml
[project]
dependencies = [
    "functions-framework>=3.10.0",  # Required for Cloud Functions HTTP server
    # ... other dependencies
]
```

### Entry Point Shim Pattern

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

---

## 3. Docker Container Deployment

> **Note:** This section covers **application deployment mechanics** (building Docker images and deploying containers). For the broader infrastructure/application separation pattern and how this fits into the 2-stack architecture, see [01-setup.md](01-setup.md#1-infrastructure-and-application-separation).

When deploying containerized applications, the Dockerfile should be placed in the project directory: `projects/{project_name}/Dockerfile`

### Docker Build Context

The Docker build is executed from the **workspace root** to access all bricks:

```bash
# From workspace root
docker build -f projects/app_cdc_ecs/Dockerfile -t app-cdc-ecs:latest .
```

### Two Approaches for Including Bricks

There are two ways to include Polylith bricks in Docker images:

1. **COPY commands** - Manually copy brick directories (current pattern)
2. **Build wheels** - Use hatchling to build Python wheels and install them

**Note:** This documentation currently shows the COPY approach. The wheel-based approach will be documented in future updates.

### Dockerfile Pattern (Using COPY)

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

### Why --no-install-project?

The `--no-install-project` flag tells uv to:
- Install all dependencies from `pyproject.toml`
- Skip installing the project itself as an editable package
- This is necessary because we're manually copying bricks instead

### Multi-stage Build Pattern (Optional)

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

---

## 4. Cloud Run Configuration (Kustomize)

Cloud Run deployments use Kustomize to manage environment-specific configurations separately from application code. This applies to both:
- **Cloud Run Services** - 24/7 running web servers and APIs
- **Cloud Run Jobs** - Batch or scheduled tasks

**For comprehensive Kustomize patterns, see [07-infrastructure.md](07-infrastructure.md#6-kustomize-for-cloud-run):**
- Cloud Run Services: [07-infrastructure.md#61-cloud-run-services](07-infrastructure.md#61-cloud-run-services)
- Cloud Run Jobs: [07-infrastructure.md#62-cloud-run-jobs](07-infrastructure.md#62-cloud-run-jobs)
- Common patterns (JSON patch escaping, placeholders): [07-infrastructure.md#64-common-patterns](07-infrastructure.md#64-common-patterns)

---

## 5. Deployment Strategy Comparison

| Aspect | Cloud Functions | Docker Container |
|--------|----------------|------------------|
| Brick copying | `copy.sh` script | Dockerfile COPY commands |
| Dependency install | `uv export` → requirements.txt | `uv sync --frozen --no-install-project` |
| When to copy | During CI/CD before deployment | During Docker image build |
| Generated files | `asset/`, `requirements.txt` | Baked into image layers |
| .gitignore | Yes (exclude generated files) | Not applicable (in image only) |

---

## 6. Best Practices

### For Both Strategies

1. **Only copy required bricks:** Check project's `[tool.polylith.bricks]` section
2. **Maintain namespace structure:** Preserve the namespace package layout
3. **Pin uv version:** Use exact version (0.7.8) for reproducibility
4. **Use --frozen flag:** Ensure deterministic builds from lock file
5. **Layer optimization:** For Docker, copy dependencies before source code for better caching
6. **Follow subset rule:** Project dependencies must be a subset of workspace root dependencies - see [03-dependencies.md](03-dependencies.md#2-workspace-vs-project-configuration) for details

### Cloud Functions Specific

1. **Run copy.sh before uv export:** Ensure bricks are in place before generating requirements.txt
2. **Use --no-emit-project flag:** Prevent editable install references in requirements.txt
3. **Keep entry point shim simple:** Just import and re-export, no logic
4. **Add generated files to .gitignore:** Don't commit `asset/` or `requirements.txt`

### Docker Specific

1. **Use multi-stage builds:** Separate build and runtime for smaller images
2. **Copy pyproject.toml first:** Leverage layer caching for dependencies
3. **Use --no-default-groups:** Minimize production image size
4. **Pin base image with digest:** Ensure reproducible builds with exact image versions
5. **Set PATH explicitly:** Make virtual environment binaries available

### CI/CD Integration

**Application deployment** is handled through GitHub Actions workflows. For the complete 2-stack infrastructure and application separation pattern, see [01-setup.md](01-setup.md#1-infrastructure-and-application-separation).

**Infrastructure provisioning** (Stack 1):
- Uses Pulumi to provision foundational GCP resources (Cloud Storage, Service Accounts, Artifact Registry, Cloud Run service shells)
- Triggered by workflow_dispatch (production) or PR labels (sandbox)
- See [06-automation.md](06-automation.md#example-4-infrastructure-provision-workflow) for complete workflow example

**Application deployment** (Stack 2):
- Cloud Functions: Uses `copy.sh` to copy bricks, `uv export` to generate requirements.txt, then `gcloud functions deploy`
- Cloud Run: Uses Docker build with Kustomize manifests, builds image, pushes to Artifact Registry, deploys with `gcloud run services replace`
- See [06-automation.md](06-automation.md#5-complete-workflow-examples) for workflow patterns

**Key principle:** Infrastructure changes are rare and deliberate; application deployments are frequent and automated.

### Troubleshooting

**Issue: "Module not found" in Cloud Functions**
- Ensure `copy.sh` was executed before deployment
- Verify namespace directory structure matches imports
- Check that all required bricks are copied

**Issue: "Module not found" in Docker**
- Verify COPY commands match project's brick references
- Check namespace directory structure in COPY commands
- Ensure COPY commands execute from workspace root context

**Issue: Large deployment size**
- Only copy bricks actually used by the project
- Use `--no-default-groups` to exclude dev/test dependencies
- For Docker, use multi-stage builds to exclude build tools

**Issue: "Container failed startup probe" in Cloud Run**
- Check startup probe configuration - settings may be too aggressive
- Review Cloud Run logs for application errors during startup
- Common causes:
  - `failureThreshold: 1` - increase to 3 or more
  - `initialDelaySeconds: 0` - increase to 5 or more
  - Missing environment variables causing application crash
  - Health check endpoint not implemented or returning non-200 status
- Recommended settings: see [Startup Probe Configuration](#startup-probe-configuration-important)
