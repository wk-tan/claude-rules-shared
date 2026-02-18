# Project Context

Python monorepo using Polylith architecture. See Core Rules for concepts and conventions.

## Rules & Skills

- **Rules** (`.claude/rules/00-core.md`) — Always-loaded conventions
- **Skills** (`.claude/skills/`) — On-demand procedures

When a task matches a skill trigger below, read the skill's `SKILL.md` before proceeding. Load linked reference files only when needed for detail.

### Available Skills

| Skill                      | Trigger                                                                       |
| -------------------------- | ----------------------------------------------------------------------------- |
| `polylith-new-brick`       | Creating components, bases, or projects                                       |
| `dependency-management`    | Adding/updating dependencies                                                  |
| `deploy-cloudrun`          | Deploying Cloud Run Jobs, Services, or Functions                              |
| `github-actions-cicd`      | Creating/modifying workflows                                                  |
| `pulumi-infrastructure`    | Infrastructure provisioning                                                   |
| `semantic-release`         | Release workflows, versioning                                                 |
| `pre-commit-hooks`         | Setting up git hooks                                                          |
| `python-code-standards`    | Writing/reviewing Python code: style, type hints, docstrings, naming, logging |
| `planning-session`         | Planning tasks, designing solutions, architecture discussions                 |

### Available Commands

| Command        | Purpose                                                       |
| -------------- | ------------------------------------------------------------- |
| `/plan`        | Start a structured planning session using the plan template   |
| `/implement`   | Execute an approved plan phase by phase with stop-gates       |
| `/investigate` | Diagnose and debug an issue without making changes            |
| `/review`      | Review code or architecture against project conventions       |
| `/prototype`   | Rapid prototyping and experimentation with relaxed guardrails |
