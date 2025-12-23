# Project Context
This is a Python project using Polylith architecture.

## Rules

See the detailed rules in `.claude/rules/` for comprehensive guidelines. Below is a quick reference:

| File | Purpose | Key Topics |
|------|---------|-----------|
| [01-polylith-architecture-and-uv.md](.claude/rules/01-polylith-architecture-and-uv.md) | Monorepo architecture & dependency management | Polylith concepts (workspace, bricks, components, bases), deployment decision tree, tool versions (Python 3.13, uv 0.7.8), subset rule |
| [02-python-code.md](.claude/rules/02-python-code.md) | Code style & formatting | PEP 8 compliance, ruff formatting, naming conventions, type hints, import ordering |
| [03-python-docstring.md](.claude/rules/03-python-docstring.md) | Documentation standards | Google-style docstrings, args/returns/raises sections, examples |
| [04-python-logging.md](.claude/rules/04-python-logging.md) | Logging patterns | Structured logging format, event/payload structure, past-tense verbs |
| [05-github-actions.md](.claude/rules/05-github-actions.md) | CI/CD workflows | Python setup, uv installation (0.7.8), `uv sync --frozen`, no caching policy |
| [06-uv-and-pyproject.md](.claude/rules/06-uv-and-pyproject.md) | Dependency & deployment configuration | Workspace vs project config, subset rule (CRITICAL), Cloud Functions copy.sh, Docker COPY patterns, dependency groups |
| [07-git-commit-conventions.md](.claude/rules/07-git-commit-conventions.md) | Version control & releases | Conventional commits, semantic versioning, branch naming, PR conventions |

### Quick Navigation

**For architecture questions**: Start with `01-polylith-architecture-and-uv.md`
**For code style questions**: See `02-python-code.md`, `03-python-docstring.md`, `04-python-logging.md`
**For dependency management**: See `06-uv-and-pyproject.md` (most comprehensive)
**For CI/CD setup**: See `05-github-actions.md`
**For git workflow**: See `07-git-commit-conventions.md`
