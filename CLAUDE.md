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
| `planning-session`         | Planning tasks, designing solutions, architecture discussions. Hard gate: no implementation until plan approved |
| `executing-plans`          | Executing an approved implementation plan phase by phase with verification gates |
| `verification-before-completion` | About to claim work is complete, a phase is done, or a deployment has succeeded |

### Workflow Chain

```
/plan (planning-session)
  → approved plan saved to plans/YYYY-MM-DD-<topic>.md
    → /implement (executing-plans)
      → phase-scoped verification (verification-before-completion) at each boundary
```

Each skill knows what comes next. Planning hands off to execution. Execution uses verification at every boundary.

### Available Commands

| Command        | Purpose                                                       |
| -------------- | ------------------------------------------------------------- |
| `/plan`        | Start a planning session. With argument: uses it as objective. Without: asks for objective |
| `/implement`   | Execute an approved plan. With argument: uses it as plan path. Without: asks for plan location |
| `/investigate` | Diagnose and debug an issue without making changes            |
| `/review`      | Review code or architecture against project conventions       |
| `/prototype`   | Rapid prototyping and experimentation with relaxed guardrails |
