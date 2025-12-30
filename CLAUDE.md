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

### Quick Navigation

**For architecture questions**: Start with `01-setup.md`
**For code style questions**: See `02-development.md` (comprehensive guide covering code, docstrings, and logging)
**For dependency management**: See `03-dependencies.md` (most comprehensive)
**For testing standards**: See `04-testing.md` (🚧 placeholder - to be expanded)
**For deployment**: See `05-deployment.md`
**For CI/CD & git workflow**: See `06-automation.md`

---

## Glossary & Key Concepts

Quick definitions of Polylith and architecture-specific terms used throughout the documentation.

### Polylith Architecture

- **Workspace** - The top-level monorepo directory containing all Polylith bricks, projects, and configuration files
- **Brick** - Fundamental unit of code in Polylith (either a component or base), acting like LEGO bricks you can compose
- **Component** - Reusable, stateless building block encapsulating specific functionality (e.g., `logging`, `settings`, `pubsub`)
- **Base** - Entry point for applications or services that compose components (e.g., `api_server`, `data_transformation`, `cli_tool`)
- **Project** - Deployable unit that specifies which components and bases to include (e.g., microservice, web app, CLI tool)
- **Namespace** - Python package namespace grouping related bricks (e.g., `de_backoffice`, `asset`, `pipeline`)

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

### Repository Specifics

- **de_backoffice** - The actual namespace used in this repository (documentation examples may use `asset` or `example`)
- **console-cr** - Cloud Run service name for the Streamlit backoffice console application
- **infrastructure/** - Directory containing all infrastructure definitions (pulumi, cloudrun, cloudfunction)
- **projects/** - Directory containing deployable applications with their own `pyproject.toml`
