# de-claude-rule-shared

This repository contains shared Claude Code rules and configuration for all Python/Polylith projects across the organization.

## What's Included

- `.claude/rules/` - Comprehensive rule files covering:
  - Polylith architecture & uv dependency management
  - Python code style (PEP 8, ruff, type hints)
  - Docstring standards (Google style)
  - Logging patterns (structured logging)
  - GitHub Actions CI/CD workflows
  - pyproject.toml configuration & deployment
  - Git commit conventions (Conventional Commits)
- `CLAUDE.md` - Project context file with rule index

## Using in Your Repository

### Initial Setup

1. **Add as submodule** (from your repository root):
   ```bash
   git submodule add https://github.com/pulsifi/claude-rules-shared.git .claude-shared
   ```

2. **Create symlinks** to make Claude Code find the rules:
   ```bash
   # Create symlink for rules directory
   ln -s .claude-shared/.claude .claude

   # Create symlink for CLAUDE.md
   ln -s .claude-shared/CLAUDE.md CLAUDE.md
   ```

3. **Commit the changes**:
   ```bash
   git add .gitmodules .claude CLAUDE.md
   git commit -m "feat: add shared Claude rules via submodule"
   git push
   ```

### Updating Rules

When rules are updated in the shared repository:

```bash
# Pull latest rules
git submodule update --remote .claude-shared

# Commit the update
git add .claude-shared
git commit -m "chore: update Claude rules to latest version"
git push
```

### Cloning Repository with Submodules

**For new clones:**
```bash
# Clone with submodules initialized
git clone --recurse-submodules <repository-url>
```

**If already cloned without submodules:**
```bash
# Initialize and update submodules
git submodule update --init --recursive
```

## Versioning

This repository uses semantic versioning. Check the [releases page](https://github.com/pulsifi/claude-rules-shared/releases) for version history.

### Pinning to Specific Version (Optional)

If you need stability, pin to a specific tag:

```bash
# In your repository
cd .claude-shared
git checkout v1.0.0
cd ..
git add .claude-shared
git commit -m "chore: pin Claude rules to v1.0.0"
```

## Making Changes

### Proposing Updates

1. Fork this repository
2. Create a feature branch: `git checkout -b feat/add-new-rule`
3. Make your changes
4. Submit a pull request with clear description
5. Once merged, all consuming repositories can update

### Testing Changes Locally

Before proposing changes, test in one repository:

```bash
# In consuming repository
cd .claude-shared
git checkout -b test-new-rule
# Make changes
cd ..

# Test with Claude Code
# If successful, push branch and create PR in claude-rules-shared
```

## Repository Structure

```
claude-rules-shared/
├── .claude/
│   └── rules/
│       ├── 01-polylith-architecture-and-uv.md
│       ├── 02-python-code.md
│       ├── 03-python-docstring.md
│       ├── 04-python-logging.md
│       ├── 05-github-actions.md
│       ├── 06-uv-and-pyproject.md
│       └── 07-git-commit-conventions.md
├── CLAUDE.md
└── README.md (this file)
```

## Current Tool Versions

See [01-polylith-architecture-and-uv.md](.claude/rules/01-polylith-architecture-and-uv.md#7-tool-versions) for current versions:
- Python: 3.13
- uv: 0.7.8
- hatchling: Latest compatible
- hatch-polylith-bricks: Latest compatible

## Troubleshooting

### Submodule shows as modified but no changes

This usually means the submodule pointer needs updating:
```bash
git submodule update --remote .claude-shared
```

### Symlinks not working on Windows

Windows requires admin privileges for symlinks. Alternatives:
1. Use Git Bash with developer mode enabled
2. Copy files instead of symlinking (but requires manual updates)

### Claude Code not finding rules

Ensure symlinks are created correctly:
```bash
ls -la | grep claude
# Should show:
# .claude -> .claude-shared/.claude
# CLAUDE.md -> .claude-shared/CLAUDE.md
```

## Support

For questions or issues:
- Create an issue in this repository
- Contact the Data Engineering team

## License

Internal use only - Pulsifi

