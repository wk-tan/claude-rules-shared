# Core Rules: Universal Conventions

These rules apply to ALL tasks. For detailed procedures and examples, use the appropriate skill.

---

## 1. Polylith Architecture

### Core Concepts

| Term | Definition |
|------|------------|
| **Workspace** | Top-level monorepo directory with `pyproject.toml`, `uv.lock`, `workspace.toml` |
| **Component** | Reusable brick in `components/{namespace}/` — stateless, single responsibility |
| **Base** | Entry point brick in `bases/{namespace}/` — composes components, thin layer |
| **Project** | Deployable unit in `projects/` — specifies which bricks to include |

### Directory Structure

```
workspace-root/
├── pyproject.toml       # Workspace config + ALL dependencies
├── uv.lock              # Locked versions (always commit)
├── workspace.toml       # Polylith metadata
├── infrastructure/      # IaC separate from app code (Pulumi, Kustomize)
├── bases/{namespace}/   # Entry points
├── components/{namespace}/  # Reusable logic
├── projects/            # Deployable units (each has pyproject.toml)
└── test/                # Tests mirror bases/ and components/ structure
```

### Namespace Package Rule

- **Namespace level** (`{namespace}/`): NO `__init__.py` — Python auto-treats as namespace package
- **Brick level** (`{namespace}/logging/`): HAS `__init__.py` — created by Polylith CLI

---

## 2. Tool Versions

| Tool | Version | Where Defined |
|------|---------|---------------|
| Python | 3.13 | `pyproject.toml` `requires-python` |
| uv | 0.7.8 | CI/CD workflows, Dockerfiles |
| Build backend | hatchling + hatch-polylith-bricks | `[build-system]` |

These versions are intentionally pinned. Do not upgrade unless explicitly requested.

---

## 3. Python Code Standards

Code quality enforced via `ruff` and `pyright`. For detailed conventions (docstrings, naming, logging), load the `python-code-standards` skill.

---

## 4. Dependency Management

### CRITICAL: Subset Rule

> Project dependencies MUST be a subset of workspace root dependencies.
> Add new dependencies to workspace root FIRST, then reference in project.

### Workspace vs Project Configuration

| Section | Workspace Root | Project Files |
|---------|----------------|---------------|
| `[project]` dependencies | ALL deps | Subset only |
| `[dependency-groups]` | Define here | NEVER duplicate |
| `[tool.ruff]`, `[tool.pyright]` | Define here | NEVER add |
| `[tool.pytest.ini_options]` | Define here | NEVER add |
| `[tool.polylith.bricks]` | All bricks | Only used bricks |
| `readme` field | YES | NEVER include |

### Lock File Commands

```bash
uv sync --frozen      # CI/CD (deterministic)
uv sync               # Local dev
uv lock --upgrade     # Update dependencies
```

---

## 5. Infrastructure Separation

Infrastructure and application code have different lifecycles — keep them separate.

| Aspect | Infrastructure (Pulumi) | Application (Docker/Kustomize) |
|--------|------------------------|-------------------------------|
| Change frequency | Rare (weeks/months) | Frequent (daily) |
| Location | `infrastructure/pulumi/` | `projects/` + `infrastructure/cloudrun*/` |
| Deployment | Manual, deliberate | Automated on merge |

---

## 6. Testing

- **Test location**: `test/` at workspace root (mirrors `bases/` and `components/`)
- **Run tests**: `uv run pytest`
- **Test levels**: Unit (components) → Integration (bases) → E2E (projects)

---

## 7. Version Control

### Conventional Commits

```
<type>(<scope>): <subject>
```

| Type | Version Bump | Example |
|------|--------------|---------|
| `feat` | MINOR | `feat(pipeline): add deduplication` |
| `fix` | PATCH | `fix(logging): correct DSN config` |
| `docs`, `style`, `refactor`, `test`, `chore`, `ci`, `build` | None | `docs: update README` |
| `!` or `BREAKING CHANGE:` | MAJOR | `feat(api)!: remove endpoint` |

**Subject rules**: imperative mood, lowercase, no period, max 72 chars

### Branch Naming

```
<type>/<ticket-id>-<brief-description>
```

Example: `feat/DA-687-migrate-to-uv`

---

## Quick Reference

### Do This

- Add deps to workspace root first, then project subset
- Use `uv sync --frozen` in CI/CD
- Commit `uv.lock` always
- Keep infrastructure in `infrastructure/` directory

### Never Do This

- Add project deps not in workspace root
- Use `uv sync` without `--frozen` in CI/CD
- Hardcode Python version in workflows (use `python-version-file`)
- Mix infrastructure with application code in `projects/`
