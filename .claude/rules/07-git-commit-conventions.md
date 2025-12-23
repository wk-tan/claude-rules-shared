# Git Commit Message and Semantic Release Conventions

**When to use this file:** Reference this for conventional commit format, semantic versioning rules, branch naming conventions, PR templates, and semantic-release workflow.

**Related documentation:**
- For GitHub Actions workflows that enforce these conventions, see [05-github-actions.md](05-github-actions.md)
- For pre-commit hook configuration, see workspace root `pyproject.toml`

These guidelines establish the standards for git commit messages, branch naming, and pull request workflows across all projects in our organization. We use [Conventional Commits](https://www.conventionalcommits.org/) to enable automated semantic versioning and changelog generation through `python-semantic-release`.

## 1. Commit Message Format

All commit messages MUST follow the Conventional Commits specification.

### Structure
```
<type>(<scope>): <subject>

[optional body]

[optional footer(s)]
```

### Required Components

#### Type (Required)
The commit type determines the version bump in semantic versioning:

- **feat:** A new feature (triggers MINOR version bump: 1.0.0 → 1.1.0)
- **fix:** A bug fix (triggers PATCH version bump: 1.0.0 → 1.0.1)
- **docs:** Documentation only changes (no version bump)
- **style:** Changes that don't affect code meaning (formatting, whitespace) (no version bump)
- **refactor:** Code changes that neither fix bugs nor add features (no version bump)
- **perf:** Performance improvements (triggers PATCH version bump: 1.0.0 → 1.0.1)
- **test:** Adding or updating tests (no version bump)
- **build:** Changes to build system or dependencies (no version bump)
- **ci:** Changes to CI/CD configuration files and scripts (no version bump)
- **chore:** Other changes that don't modify src or test files (no version bump)

#### Scope (Optional but Recommended)
The scope indicates what part of the codebase is affected:

**Common scopes:**
- `pipeline`: Data pipeline logic
- `cdc`: Change Data Capture functionality
- `gcp`: GCP-related components
- `settings`: Configuration and settings
- `logging`: Logging infrastructure
- `ci`: CI/CD workflows
- `cdk`: AWS CDK infrastructure
- `docker`: Dockerfile changes

**Examples:**
```
feat(pipeline): add event deduplication
fix(logging): correct sentry integration
docs(readme): update installation steps
ci(deploy): migrate workflow to uv
```

#### Subject (Required)
The subject is a brief description of the change:

- **Use imperative mood:** "add feature" not "added feature" or "adds feature"
- **Don't capitalize first letter:** "add feature" not "Add feature"
- **No period at the end:** "add feature" not "add feature."
- **Keep it concise:** Maximum 72 characters
- **Be descriptive:** Explain what changed, not just which file

**Good examples:**
```
feat(pipeline): add CDC event deduplication
fix(cdc): handle null values in replication slot
docs: update pyproject.toml configuration guide
```

**Bad examples:**
```
feat: Update files  # Too vague
Fix bug  # Missing type parentheses, not descriptive
feat(pipeline): Added new feature.  # Wrong tense, capitalized, has period
```

### Optional Components

#### Body (Optional)
Provide additional context about the change:

- Separate from subject with a blank line
- Wrap at 72 characters
- Explain what and why, not how
- Use imperative mood

**Example:**
```
feat(pipeline): add CDC event deduplication

Implement deduplication logic to prevent duplicate CDC events from
being published to Pub/Sub. Uses in-memory cache with LRU eviction
to track recently processed LSN positions.
```

#### Footer (Optional)
Reference issues, breaking changes, or co-authors:

**Breaking changes:**
```
feat(settings)!: change environment variable naming

BREAKING CHANGE: All environment variables now use SCREAMING_SNAKE_CASE
instead of PascalCase. Update your .env files accordingly.
```

**Issue references:**
```
fix(cdc): handle connection timeouts

Closes #123
Fixes #456
```

**Co-authoring:**
```
feat(pipeline): implement new data transformation

Co-authored-by: John Doe <john@example.com>
```

## 2. Breaking Changes

Breaking changes trigger a MAJOR version bump (1.0.0 → 2.0.0).

### Indicating Breaking Changes

**Method 1: Add `!` after type/scope:**
```
feat(api)!: remove deprecated endpoints
```

**Method 2: Use `BREAKING CHANGE:` footer:**
```
feat(settings): change configuration format

BREAKING CHANGE: Configuration now uses YAML instead of JSON.
All existing config.json files must be migrated to config.yaml.
```

### When to Use Breaking Changes
- Removing public APIs or features
- Changing function signatures
- Renaming environment variables
- Changing configuration formats
- Modifying database schemas (for libraries)

## 3. Complete Examples

### Feature Addition
```
feat(pipeline): add retry logic for failed CDC events

Implement exponential backoff retry mechanism for failed Pub/Sub
publish operations. Retries up to 3 times with 2s, 4s, 8s delays.

Closes #142
```

### Bug Fix
```
fix(logging): correct sentry DSN configuration

Previously used wrong DSN for production environment, causing errors
to be sent to staging Sentry project. Now correctly reads from
SENTRY_DSN environment variable.

Fixes #156
```

### Documentation
```
docs: add comprehensive Google-style docstrings to core modules

Add complete docstrings following Google style guide to all core.py files
in bases and components directories. Includes detailed descriptions for:
- Function purposes and behavior
- All parameters with types and descriptions
- Return values and types
- Raised exceptions
- Important notes and caveats
```

### CI/CD Changes
```
ci: migrate deploy workflow to uv

Replace Poetry with uv for faster dependency installation.
Update all workflow steps to use uv sync --frozen.
```

### Dependency Updates
```
build: upgrade google-cloud-pubsub to 2.33.0

Update to latest version for improved performance and bug fixes.
```

### Breaking Change
```
feat(settings)!: migrate to pydantic v2 settings

BREAKING CHANGE: Settings class now uses Pydantic v2 API.
Update all code using `Config` inner class to use `model_config` instead.

Migration guide:
- Replace `class Config:` with `model_config = SettingsConfigDict()`
- Update `env_prefix` to use new format
```

## 4. Branch Naming Conventions

### Standard Format
```
<type>/<ticket-id>-<brief-description>
```

**Examples:**
```
feat/DA-687-migrate-to-uv
fix/DA-701-sentry-integration
docs/DA-688-update-readme
chore/DA-702-upgrade-dependencies
```

### Branch Types
- **feat/**: New features
- **fix/**: Bug fixes
- **docs/**: Documentation updates
- **refactor/**: Code refactoring
- **test/**: Test additions or updates
- **chore/**: Maintenance tasks

### Guidelines
- Always include ticket ID (e.g., DA-687, JIRA-123)
- Use lowercase with hyphens
- Keep description brief but descriptive
- Maximum 50 characters total

## 5. Pull Request Conventions

### PR Title Format
PR titles SHOULD follow the same format as commit messages:

```
<type>(<scope>): <description>
```

**Examples:**
```
feat(pipeline): add CDC event deduplication
fix(logging): correct sentry integration
docs: update installation guide
```

### PR Description Template
```markdown
## Summary
Brief description of what this PR does.

## Changes
- Bullet point list of changes
- Another change
- Yet another change

## Testing
- [ ] Unit tests added/updated
- [ ] Integration tests pass
- [ ] Manual testing completed

## Related Issues
Closes #123
Relates to #456

## Breaking Changes
(If applicable) Describe breaking changes and migration steps.

## Screenshots
(If applicable) Add screenshots for UI changes.
```

## 6. Semantic Release Workflow

### How Semantic Release Works
1. **Analyze commits:** Scans commit messages since last release
2. **Determine version bump:**
   - `feat:` → MINOR bump (1.0.0 → 1.1.0)
   - `fix:` or `perf:` → PATCH bump (1.0.0 → 1.0.1)
   - `BREAKING CHANGE:` or `!` → MAJOR bump (1.0.0 → 2.0.0)
   - Other types → No version bump
3. **Generate changelog:** Creates CHANGELOG.md from commit messages
4. **Create git tag:** Tags the release (e.g., v1.2.3)
5. **Publish release:** Creates GitHub release with notes

### Configuration
Our `python-semantic-release` configuration is in [pyproject.toml:252](pyproject.toml#L252):

```toml
[tool.semantic_release]
version_toml = ["pyproject.toml:project.version"]
branch = "main"
build_command = "echo 'No build command'"
```

### Triggering Releases
Releases are triggered automatically when:
1. Commits are pushed to `main` branch
2. Semantic release workflow runs
3. Commits since last release determine version bump
4. New version is tagged and published

### Manual Release (Rare)
```bash
# Dry run to see what would happen
semantic-release --noop version --print

# Actually perform the release
semantic-release version
semantic-release publish
```

## 7. Best Practices

### Writing Good Commit Messages

**DO:**
- ✅ Write clear, descriptive subjects
- ✅ Use conventional commit format consistently
- ✅ Include scope when relevant
- ✅ Reference issue numbers in body or footer
- ✅ Explain why a change was made, not just what changed
- ✅ Keep commits focused on a single concern

**DON'T:**
- ❌ Use vague messages like "fix bug" or "update code"
- ❌ Combine unrelated changes in one commit
- ❌ Forget the type prefix (feat, fix, etc.)
- ❌ Use past tense ("added" instead of "add")
- ❌ Make commits too large (prefer smaller, focused commits)

### Commit Frequency

**Commit often:**
- After completing a logical unit of work
- Before switching tasks
- Before refactoring
- When tests pass

**Each commit should:**
- Be atomic (single logical change)
- Pass tests (if applicable)
- Build successfully
- Be potentially revertible

### Squashing vs. Preserving History

**When to squash (combine commits):**
- Multiple "WIP" or "fix typo" commits
- Commits that fix issues in earlier commits in same PR
- Cleaning up before merging to main

**When to preserve commits:**
- Each commit represents a distinct logical change
- Commits have valuable individual context
- History aids in debugging or understanding changes

## 8. Common Mistakes and Fixes

### Mistake 1: Wrong Commit Type
```
# Wrong
update: add new feature

# Correct
feat: add new feature
```

### Mistake 2: Missing Type
```
# Wrong
added CDC event deduplication

# Correct
feat(pipeline): add CDC event deduplication
```

### Mistake 3: Past Tense
```
# Wrong
feat(pipeline): added event deduplication

# Correct
feat(pipeline): add event deduplication
```

### Mistake 4: Capitalized Subject
```
# Wrong
feat(pipeline): Add event deduplication

# Correct
feat(pipeline): add event deduplication
```

### Mistake 5: Period at End
```
# Wrong
feat(pipeline): add event deduplication.

# Correct
feat(pipeline): add event deduplication
```

### Mistake 6: Too Vague
```
# Wrong
fix: update code

# Correct
fix(logging): correct sentry DSN configuration for production
```

## 9. Git Workflow Integration

### Pre-commit Hooks
Pre-commit hooks automatically run before each commit:

1. **ruff-format:** Formats Python code
2. **ruff-linter:** Lints Python code
3. **pytest:** Runs test suite

If hooks modify files (e.g., formatting), the commit will fail. Stage the changes and retry.

### Commit Message Validation
Consider adding `commitlint` to validate commit messages:

```yaml
# .pre-commit-config.yaml
- repo: https://github.com/compilerla/conventional-pre-commit
  rev: v3.0.0
  hooks:
    - id: conventional-pre-commit
      stages: [commit-msg]
```

## 10. Tools and Resources

### Helpful Commands
```bash
# View commit history with conventional format
git log --oneline --decorate

# View commits since last tag
git log $(git describe --tags --abbrev=0)..HEAD --oneline

# Amend last commit message
git commit --amend

# Interactive rebase to fix commit messages
git rebase -i HEAD~3
```

### Conventional Commits Tools
- **commitizen:** CLI tool for writing conventional commits
- **semantic-release:** Automated version management
- **conventional-changelog:** Generate changelogs from commits

### Resources
- [Conventional Commits Specification](https://www.conventionalcommits.org/)
- [Semantic Versioning](https://semver.org/)
- [Python Semantic Release Docs](https://python-semantic-release.readthedocs.io/)

## 11. Troubleshooting

### Issue: Semantic Release Doesn't Bump Version
**Cause:** No commits with `feat:` or `fix:` since last release.
**Solution:** Ensure commits use correct types that trigger version bumps.

### Issue: Wrong Version Bump
**Cause:** Incorrect commit type (e.g., used `fix:` instead of `feat:`).
**Solution:** Use interactive rebase to fix commit messages before merging to main.

### Issue: Release Failed
**Cause:** Commit message format error or missing permissions.
**Solution:** Check commit messages match conventional format; verify `GH_TOKEN` has correct permissions.

### Issue: Merge Conflicts in Changelog
**Cause:** Multiple PRs merged simultaneously updating CHANGELOG.md.
**Solution:** Let semantic-release regenerate the changelog; don't manually resolve conflicts.
