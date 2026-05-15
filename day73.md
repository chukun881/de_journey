# 📅 Day 73 — Saturday, 26 July 2026
# 🏢 Professional Repo Setup: Branch Protection, PR Templates, Contributing Guide

---

## 🎯 Today's Goal

Your repos work. Your CI passes. Today you make them look like professional team projects — the kind of repos that signal to a hiring manager "this person knows how to work on a real engineering team." Branch protection, PR templates, contributing guides, pre-commit hooks.

**Philosophy:** At companies like Grab, Shopee, or Singtel, you can't push directly to main. Everything goes through a Pull Request. CI must pass. At least one person reviews your code. Today you set up the same workflow for your own projects — not because you need to, but because it shows employers you know how.

---

## ☀️ Morning Block (2 hours): Branch Protection + PR Templates

### Branch Protection Rules

On GitHub: Settings → Branches → Add rule → Branch name pattern: `main`

Enable these rules:

| Rule | Why |
|------|-----|
| Require a pull request before merging | Forces code review |
| Require approvals (1) | Someone else must review |
| Require status checks to pass | CI must be green |
| Require branches to be up to date | Must have latest main |
| Do not allow force pushes | Protect history |
| Do not allow deletions | Protect the branch |

### How It Works in Practice

```
WITHOUT branch protection:
  You edit code → push to main → done (risky!)

WITH branch protection:
  1. Create feature branch
  2. Make changes + commit
  3. Push branch to GitHub
  4. Open Pull Request
  5. CI runs (automated tests)
  6. Teammate reviews code
  7. Address feedback
  8. CI ✅ + Review ✅ → Merge button unlocks
  9. Merge → main updated
```

### Pull Request Template

Create `.github/pull_request_template.md`:

```markdown
## Description
<!-- What does this PR do? -->

## Type of Change
- [ ] 🐛 Bug fix
- [ ] ✨ New feature
- [ ] ♻️ Refactor
- [ ] 📝 Documentation
- [ ] 🧪 Test

## Related Issue
Closes #

## Changes Made
- 
- 

## Testing
- [ ] Existing tests pass
- [ ] New tests added
- [ ] Manual testing done

## Checklist
- [ ] Code follows project conventions
- [ ] No hardcoded credentials
- [ ] SQL queries are optimized
- [ ] Documentation updated

## Screenshots (if applicable)
<!-- Add screenshots of query results or dashboard changes -->
```

### Exercise 1: Set Up Branch Protection

1. Go to your most polished repo on GitHub
2. Settings → Branches → Add rule for `main`
3. Enable: require PR, require status checks, no force push
4. Try pushing directly to main — it should be rejected!

<details>
<summary>🔑 Expected behavior</summary>

```bash
git checkout main
echo "test" > test.txt
git add . && git commit -m "test direct push"
git push origin main
```

You'll get an error:
```
remote: error: GH006: Protected branch update failed for refs/heads/main.
remote: error: Changes must be made through a pull request.
```

This is branch protection working! Now create a branch, open a PR, and watch it require CI to pass.
</details>

### CODEOWNERS File

Create `CODEOWNERS` in repo root:

```
# These people MUST review changes to these areas

# SQL models — reviewed by data team lead
/models/           @yourusername

# Python ETL scripts
/src/              @yourusername

# Documentation
*.md               @yourusername

# CI/CD configuration
/.github/          @yourusername
```

Even though you're the only owner, this shows employers you know the concept.

---

## 🌤️ Afternoon Block (2 hours): Contributing Guide + Pre-commit Hooks

### CONTRIBUTING.md

Create `CONTRIBUTING.md` in repo root:

```markdown
# Contributing to [Project Name]

Thank you for your interest! This guide explains how to contribute.

## Getting Started

1. Fork this repository
2. Clone your fork: `git clone https://github.com/YOUR_USERNAME/REPO.git`
3. Create a branch: `git checkout -b feature/your-feature-name`
4. Make changes and commit: `git commit -m "feat: description"`
5. Push: `git push origin feature/your-feature-name`
6. Open a Pull Request

## Development Setup

```bash
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate
pip install -r requirements.txt
pip install -r requirements-dev.txt  # dev dependencies
```

## Code Style

- Python: Follow PEP 8 (enforced by `black`)
- SQL: Follow SQLFluff rules (`.sqlfluff`)
- Commit messages: Conventional Commits format

## Running Tests

```bash
pytest tests/ -v
sqlfluff lint models/
black --check src/
```

## Pull Request Process

1. Ensure all tests pass
2. Update documentation if needed
3. Add tests for new features
4. Request review from a maintainer

## Commit Message Format

```
type(scope): description

feat(etl): add daily load for MakanExpress orders
fix(sql): handle null values in customer_id
docs(readme): add architecture diagram
```
```

### Pre-commit Hooks

Create `.pre-commit-config.yaml`:

```yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-json
      - id: check-merge-conflict
      - id: detect-private-key  # Prevent committing secrets!

  - repo: https://github.com/psf/black
    rev: 24.1.1
    hooks:
      - id: black
        language_version: python3.11

  - repo: https://github.com/PyCQA/flake8
    rev: 7.0.0
    hooks:
      - id: flake8
        args: ['--max-line-length=120']

  - repo: https://github.com/sqlfluff/sqlfluff
    rev: 3.0.3
    hooks:
      - id: sqlfluff-lint
        args: ['--dialect', 'postgres']
```

Install and activate:
```bash
pip install pre-commit
pre-commit install
```

Now every time you `git commit`, these hooks run automatically:
- Remove trailing whitespace
- Fix end-of-file issues
- Check for merge conflict markers
- Detect accidentally committed private keys (!!!)
- Format Python with black
- Lint Python with flake8
- Lint SQL with sqlfluff

### Exercise 2: Set Up Pre-commit Hooks

1. Install pre-commit: `pip install pre-commit`
2. Create `.pre-commit-config.yaml` in one of your repos
3. Run `pre-commit install`
4. Intentionally commit a file with trailing whitespace
5. Watch pre-commit block the commit and auto-fix it

<details>
<summary>🔑 Expected behavior</summary>

```bash
echo "trailing spaces   " > test.md
git add test.md
git commit -m "test"

# Pre-commit runs and finds issues:
# Trim Trailing Whitespace........................Failed
# - hook id: trailing-whitespace
# - exit code: 1
# - files were modified by this hook

# It auto-fixed the file! Stage and commit again:
git add test.md
git commit -m "test"
# Now it passes!
```

The key insight: pre-commit catches problems BEFORE they reach GitHub. It's like a spell-checker for code.
</details>

### Issue Templates

Create `.github/ISSUE_TEMPLATE/bug_report.md`:

```markdown
---
name: Bug Report
about: Report a bug
title: '[BUG] '
labels: bug
---

## Description
<!-- What went wrong? -->

## Steps to Reproduce
1. 
2. 
3. 

## Expected Behavior
<!-- What should have happened? -->

## Actual Behavior
<!-- What actually happened? -->

## Environment
- Python version:
- OS:
- Database:
```

Create `.github/ISSUE_TEMPLATE/feature_request.md`:

```markdown
---
name: Feature Request
about: Suggest a new feature
title: '[FEATURE] '
labels: enhancement
---

## Problem
<!-- What problem does this feature solve? -->

## Proposed Solution
<!-- How should it work? -->

## Additional Context
<!-- Any other information? -->
```

---

## 🌙 Evening (1 hour): Apply to All Repos + Push

### Consistency Checklist for All 5 Repos

For each repo, ensure these exist:

| File | Purpose |
|------|---------|
| `README.md` | Project overview, badges, setup instructions |
| `CONTRIBUTING.md` | How to contribute |
| `.github/pull_request_template.md` | PR template |
| `.github/ISSUE_TEMPLATE/bug_report.md` | Bug report form |
| `.github/ISSUE_TEMPLATE/feature_request.md` | Feature request form |
| `.github/workflows/test.yml` | CI pipeline |
| `.pre-commit-config.yaml` | Pre-commit hooks |
| `CODEOWNERS` | Review ownership |
| `.gitignore` | Ignore patterns |
| `LICENSE` | MIT License |

### Exercise 3: Apply to All Repos

1. Start with your best repo (probably `food-delivery-warehouse`)
2. Add all files listed above
3. Set up branch protection on GitHub
4. Install pre-commit hooks
5. Push and verify CI runs
6. Repeat for remaining repos (can be lighter — focus on the top 2-3)

### 📝 Today's Checklist

- [ ] Set up branch protection on at least one repo
- [ ] Created PR template
- [ ] Created CONTRIBUTING.md
- [ ] Installed and configured pre-commit hooks
- [ ] Created issue templates (bug report + feature request)
- [ ] Added CODEOWNERS file
- [ ] Applied professional setup to at least 2 repos
- [ ] All changes pushed to GitHub

---

*Day 73 complete! Tomorrow: System design interview practice — "Design a data pipeline."* 🏗️
