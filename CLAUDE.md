# Project Context

Python monorepo using Polylith architecture. See Core Rules for concepts and conventions.

## Rules & Skills

- **Rules** (`.claude/rules/00-core.md`) — Always-loaded conventions
- **Skills** (`.claude/skills/`) — On-demand procedures

When a task matches a skill trigger below, read the skill's `SKILL.md` before proceeding. Load linked reference files only when needed for detail.

### Available Skills

| Skill | Trigger |
|-------|---------|
| `polylith-new-brick` | Creating components, bases, or projects |
| `dependency-management` | Adding/updating dependencies |
| `deploy-cloudrun-functions` | Deploying Cloud Run Functions |
| `deploy-cloudrun-service` | Deploying Cloud Run Services |
| `deploy-cloudrun-job` | Deploying Cloud Run Jobs |
| `github-actions-cicd` | Creating/modifying workflows |
| `pulumi-infrastructure` | Infrastructure provisioning |
| `semantic-release` | Release workflows, versioning |
| `pre-commit-hooks` | Setting up git hooks |
| `python-code-standards` | Writing/reviewing Python code: style, type hints, docstrings, naming, logging |
