# Project Context
This is a Python project using Polylith architecture.

## Rules

See the detailed rules in `.claude/rules/` for comprehensive guidelines. Below is a quick reference:

| File | Purpose | Key Topics |
|------|---------|-----------|
| [01-setup.md](.claude/rules/01-setup.md) | Monorepo architecture & tool configuration | Polylith concepts (workspace, bricks, components, bases), infrastructure/application separation (2-stack pattern), tool versions (Python 3.13, uv 0.7.8), developer workflow |
| [02-development.md](.claude/rules/02-development.md) | Python coding standards | PEP 8 compliance, ruff formatting, naming conventions, type hints, import ordering, docstrings, logging |
| [03-dependencies.md](.claude/rules/03-dependencies.md) | Dependency & deployment configuration | Workspace vs project config, subset rule (CRITICAL), Cloud Functions copy.sh, Docker COPY patterns, dependency groups |
| [04-testing.md](.claude/rules/04-testing.md) | Testing standards 🚧 | Placeholder for pytest patterns, coverage requirements, data testing (to be expanded) |
| [05-deployment.md](.claude/rules/05-deployment.md) | Deployment patterns | Cloud Functions deployment, Docker containers, deployment strategy decision tree, infrastructure vs application deployment |
| [06-automation.md](.claude/rules/06-automation.md) | CI/CD workflows & version control | GitHub Actions setup, infrastructure provisioning workflow, uv installation (0.7.8), `uv sync --frozen`, conventional commits, semantic versioning, PR conventions |
| [07-infrastructure.md](.claude/rules/07-infrastructure.md) | Infrastructure patterns | Pulumi IaC patterns (config, YAML anchors, resource loops), Kustomize for Cloud Run (Services, Jobs), supplementary file loading (BigQuery schemas, monitoring dashboards) |

### Quick Navigation

**For architecture questions**: Start with `01-setup.md`
**For code style questions**: See `02-development.md` (comprehensive guide covering code, docstrings, and logging)
**For dependency management**: See `03-dependencies.md` (most comprehensive)
**For testing standards**: See `04-testing.md` (🚧 placeholder - to be expanded)
**For deployment**: See `05-deployment.md`
**For CI/CD & git workflow**: See `06-automation.md`
**For infrastructure patterns**: See `07-infrastructure.md` (Pulumi IaC, Kustomize for Cloud Run)

### Task Reference

| When working on... | Read this file |
|-------------------|----------------|
| Creating a new project or brick | [01-setup.md](.claude/rules/01-setup.md) |
| Writing or modifying Python code | [02-development.md](.claude/rules/02-development.md) |
| Adding/updating dependencies | [03-dependencies.md](.claude/rules/03-dependencies.md) |
| Creating/modifying pyproject.toml | [03-dependencies.md](.claude/rules/03-dependencies.md) |
| Writing tests | [04-testing.md](.claude/rules/04-testing.md) |
| Deploying to Cloud Functions | [05-deployment.md](.claude/rules/05-deployment.md) |
| Deploying to Cloud Run (Docker) | [05-deployment.md](.claude/rules/05-deployment.md) |
| Creating GitHub Actions workflows | [06-automation.md](.claude/rules/06-automation.md) |
| Writing commit messages | [06-automation.md](.claude/rules/06-automation.md) |
| Creating pull requests | [06-automation.md](.claude/rules/06-automation.md) |
| Writing Pulumi IaC code | [07-infrastructure.md](.claude/rules/07-infrastructure.md) |
| Configuring Cloud Run Jobs | [07-infrastructure.md](.claude/rules/07-infrastructure.md) |
| Writing Kustomize overlays | [07-infrastructure.md](.claude/rules/07-infrastructure.md) |

---

## Glossary & Key Concepts

Quick definitions of Polylith and architecture-specific terms used throughout the documentation.

### Polylith Architecture

- **Workspace** - The top-level monorepo directory containing all Polylith bricks, projects, and configuration files
- **Brick** - Fundamental unit of code in Polylith (either a component or base), acting like LEGO bricks you can compose
- **Component** - Reusable, stateless building block encapsulating specific functionality (e.g., `logging`, `settings`, `pubsub`)
- **Base** - Entry point for applications or services that compose components (e.g., `api_server`, `data_transformation`, `cli_tool`)
- **Project** - Deployable unit that specifies which components and bases to include (e.g., microservice, web app, CLI tool)
- **Namespace** - Python package namespace grouping related bricks (e.g., `my_app`, `data_pipeline`, `shared`)

### Infrastructure & Deployment

- **2-Stack Pattern** - Architectural pattern separating infrastructure foundation (Stack 1) from application deployment (Stack 2)
- **Stack 1: Infrastructure Foundation** - Rarely-changing foundational resources provisioned by Pulumi (service accounts, buckets, IAM, Artifact Registry)
- **Stack 2: Application Deployment** - Frequently-changing application code and deployment configs (Docker images, Cloud Run manifests, Cloud Functions)
- **Kustomize** - Kubernetes-native configuration management tool used for environment-specific Cloud Run manifests
- **Overlay** - Environment-specific configuration patches in Kustomize (e.g., `sandbox`, `production`)
- **Lifecycle-Based Separation** - Organizing code/config by change frequency (infrastructure: rare, application: frequent)

### Tools & Workflows

- **uv** - Fast Python package manager (v0.7.8) replacing pip/poetry for dependency management
- **Pulumi** - Infrastructure as Code tool for provisioning GCP resources declaratively
- **Polylith CLI** - Command-line tool for creating and managing Polylith bricks (`poly create component`)
- **Conventional Commits** - Standardized commit message format enabling automated semantic versioning

### Polylith Directory Structure

- **workspace root** - Contains `pyproject.toml`, `uv.lock`, `workspace.toml`, and all Polylith directories
- **bases/** - Entry points for applications (e.g., `bases/{namespace}/api_server/`)
- **components/** - Reusable building blocks (e.g., `components/{namespace}/logging/`)
- **projects/** - Deployable units with their own `pyproject.toml` (e.g., `projects/my_service/`)
- **infrastructure/** - Infrastructure definitions separate from application code (pulumi, cloudrun, cloudfunction)
- **test/** - Tests mirroring bases/ and components/ structure
