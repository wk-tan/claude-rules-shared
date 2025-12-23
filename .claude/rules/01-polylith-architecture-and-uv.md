# Polylith Architecture & `uv` Guidelines for Python Development

**When to use this file:** Start here for understanding monorepo architecture, Polylith concepts (workspace, bricks, components, bases), deployment strategy decisions, and tool version references.

These guidelines establish the principles and practices for developing Python projects using the Polylith architecture and `uv` as our primary dependency management tool. The goal is to foster a modular, maintainable, and efficient development workflow within our monorepo, leveraging a consistent folder structure to facilitate automated tooling and code generation.

## 1. Understanding Polylith Core Concepts
Polylith provides a structured approach to building software by organizing code into reusable, independent "bricks" within a monorepo. This architecture helps manage the complexities of both monolithic and microservice approaches by promoting code sharing and clear separation of concerns.

*   **Workspace:** The top-level directory of our monorepo, housing all Polylith building blocks and configuration. The workspace name is defined in `[tool.polylith]` within `workspace.toml` at the workspace root.
*   **Bricks (Components & Bases):** The fundamental units of code in Polylith, acting like LEGO bricks. They are essentially Python namespace packages.
    *   **Components:** These are high-level, reusable building blocks encapsulating specific functionalities or domain logic. They should be designed for single responsibility, statelessness, and purity, hiding their implementation details and exposing only their signatures. Components form the core reusable logic across our projects.
    *   **Bases:** These are special bricks that serve as entry points for our applications or services (e.g., FastAPI routes, CLI commands, serverless functions). They act as thin layers that compose components to deliver specific functionality.
*   **Projects:** Define deployable units (e.g., a microservice, a web application, a CLI tool, or a library) by specifying which components and bases are required for their functionality. A monorepo can contain multiple projects.

## 2. Monorepo Structure and Organization
Maintaining a strict and consistent monorepo structure, [as demonstrated in the `python-polylith-example-uv`](https://github.com/DavidVujic/python-polylith-example-uv/tree/main), is paramount. All Python code, including components, bases, and projects, **must** adhere to this predefined structure. **This consistency is not merely for readability; it is a fundamental enabler for automated code generation, linting, testing, and deployment scripts.** Code generation tools and scripts will rely on these predictable paths.

*   **Top-level `pyproject.toml`:** This file at the workspace root will define global development dependencies (e.g., linters, formatters, test runners) and `uv` specific configurations for the entire monorepo.
*   **`bases/{workspace_name}/` directory:** Contains all base bricks. Each base resides in a distinct directory within a subdirectory named after your workspace.
    *   *Example based on `python-polylith-example-uv`*: If your workspace is named `example`, then bases would be located at `bases/example/my_api_base`, `bases/example/my_cli_base`.
*   **`components/{workspace_name}/` directory:** Contains all component bricks. Each component resides in a distinct directory within a subdirectory named after your workspace. An *optional* additional layer of directories is permitted for logical grouping (e.g., by cloud provider, domain area, or service type).
    *   *Example based on `python-polylith-example-uv`*: If your workspace is named `example`, then components could be located at `components/example/user_management`, `components/example/product_catalog`.
    *   *Example with grouping*: `components/example/gcp/logging`, `components/example/gcp/pubsub`, `components/example/aws/s3`, `components/example/settings`.
*   **`projects/` directory:** Contains all project definitions. Each project will have its own `pyproject.toml` specifying its unique runtime dependencies and the components/bases it consumes.
    *   *Example:* `projects/api_service`, `projects/data_processor`

## 3. Dependency Management with `uv`
`uv` will be used for all dependency management operations, ensuring speed and consistency.

*   **Workspace-level Virtual Environment:** Create and manage a single virtual environment at the monorepo root using `uv`. All development tools (linters, formatters, test frameworks) will be installed here.
    *   **Command:** `uv venv` (to create), `uv sync` (to install/update dependencies from the root `pyproject.toml`).
*   **Project-specific Dependencies:** Each project within the `projects/` directory will have its own `pyproject.toml` that declares its direct runtime dependencies. `uv` will manage these dependencies efficiently without creating separate virtual environments for each project.
    *   `uv` supports relative includes for components and bases through the `[tool.polylith.bricks]` configuration in `pyproject.toml`, simplifying packaging.
*   **Subset Rule (CRITICAL):** Project dependencies MUST be a subset of workspace root dependencies. Never add dependencies to project `pyproject.toml` files that aren't already in the workspace root. This ensures:
    *   Consistent versions across all projects
    *   Single source of truth for dependency versions
    *   No version conflicts between projects
*   **Adding/Removing Dependencies:** Always use `uv add` and `uv remove` within the context of the relevant `pyproject.toml`.
    *   Example: To add a development dependency to the workspace: `uv add <package_name> --group dev`.
    *   Example: To add a runtime dependency (workspace-first approach):
        1. First add to workspace root: `uv add <package_name>`
        2. Then add to project if needed: `cd projects/my_service && uv add <package_name>`
        3. This ensures the dependency version is controlled at workspace level

## 4. Code Sharing and Reusability
The primary benefit of Polylith is fostering code reuse.

*   **Component-first Thinking:** Prioritize creating new functionalities as components that are independent and potentially reusable by multiple bases or projects.
*   **Avoid Duplication:** Actively seek to reuse existing components rather than duplicating logic.
*   **Clear Boundaries:** Components should have well-defined responsibilities and interfaces, minimizing implicit dependencies.

## 5. Testing Strategy
Polylith's modularity naturally supports robust testing.

*   **Component-level Testing:** Write comprehensive unit tests for each component, ensuring its individual functionality is correct and isolated.
*   **Base-level Testing:** Test bases to ensure they correctly compose components and handle their specific entry-point logic (e.g., API request/response handling).
*   **Project-level Integration/End-to-End Testing:** For each deployable project, implement integration or end-to-end tests to verify the assembled system functions as expected.
*   **Consistent Tooling:** Utilize workspace-level testing tools (e.g., `pytest` installed at the root via `uv`) to ensure consistent test execution and reporting across the monorepo.

## 6. Packaging and Deployment
Projects are the deployable artifacts. The deployment strategy varies based on the target platform.

### Deployment Strategy Decision Tree

**Use hatchling auto-assembly when:**
*   Deploying to environments that support editable installs
*   Building Python packages for PyPI distribution
*   Running in containerized services with full build control
*   Local development and testing

**Use copy.sh scripts when:**
*   Deploying to Google Cloud Functions
*   Need flat directory structure with requirements.txt
*   Want explicit control over which bricks are deployed
*   Minimizing deployment package size is critical

**Use Dockerfile COPY commands when:**
*   Deploying containerized applications to Cloud Run, ECS, Kubernetes
*   Building Docker images for production
*   Want layer caching benefits for faster builds
*   Need reproducible production images

### Cloud Functions and Docker Deployment

For detailed deployment patterns, see **`06-uv-and-pyproject.md` Section 7** which covers:
*   Cloud Functions `copy.sh` script patterns
*   Docker Dockerfile COPY commands
*   Requirements.txt generation with `uv export`
*   Multi-stage Docker builds
*   Deployment comparison table
*   Best practices for both strategies

### Python Package Distribution

For distributing libraries or packages:

*   **Building:** Use hatchling with hatch-polylith-bricks to build wheel files
*   **Command:** `uv build` or `python -m build`
*   **Publishing:** `uv publish` to PyPI or private package index
*   The `pyproject.toml` within each project directory dictates what gets packaged

## 7. Tool Versions

Current versions used across the project (source of truth):

*   **Python:** 3.13 (defined in `pyproject.toml` `requires-python` field)
*   **uv:** 0.7.8 (see `05-github-actions.md` for CI/CD usage)
*   **hatchling:** Latest compatible (defined in `[build-system]`)
*   **hatch-polylith-bricks:** Latest compatible (defined in `[build-system]`)

When upgrading versions:
1. Update `pyproject.toml` for Python version
2. Update `05-github-actions.md` for uv version
3. Run `uv lock --upgrade` to regenerate lock file
4. Test thoroughly before committing

## 8. Developer Workflow and Tooling
*   **Single Virtual Environment:** Benefit from a single virtual environment for the entire monorepo, ensuring all code runs with the same dependency versions, linters, and type checkers.
*   **Consistent Tooling:** Centralize the configuration for linting (`ruff`), type checking (`pyright`), and formatting at the workspace root to ensure consistency across all bricks and projects.
*   **`uv run` for Scripts:** Utilize `uv run` for executing commands within the virtual environment context.
*   **IDE Configuration:** Configure IDEs (e.g., VSCode) to recognize the workspace's virtual environment and the Polylith structure for seamless navigation, auto-completion, and debugging.
*   **Workspace-first approach:** Always add dependencies to workspace root first, then to projects as subsets.
*   **Test in isolation:** Test components and bases independently before integrating into projects.
