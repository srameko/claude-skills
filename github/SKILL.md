---
name: github
description: >
  GitHub best practices for repository structure, workflows, devcontainer, branch
  protection, and conventions. Use this skill whenever the user works on GitHub
  repo setup, creates or edits GitHub Actions workflows, sets up a devcontainer,
  configures branch protection, manages Dependabot, or organizes a new repository.
  Also trigger for questions about repo conventions, README structure, .gitignore,
  or general CI/CD patterns. For security hardening of workflows and supply chain
  protection, see the github-security skill.
---

# GitHub Skill

> For security hardening (SHA pinning, Harden-Runner, zizmor, supply chain) — see the **github-security** skill.

---

## Repository Structure

```
.github/
  workflows/          ← CI/CD workflows
  dependabot.yml      ← automated dependency updates
CLAUDE.md             ← context for Claude Code (always add to new repos)
README.md             ← project description, links, setup instructions
.gitignore            ← always commit, use language-appropriate template
.devcontainer/
  devcontainer.json   ← reproducible dev environment
```

---

## CLAUDE.md

Every repo should have a `CLAUDE.md` at the root. It gives Claude Code context
about the project without needing to re-explain every session.

**What to include:**
- What the repo is and what it does
- Directory structure overview
- Key conventions (naming, patterns, do/don't)
- How to run, test, deploy
- Any gotchas or non-obvious decisions

Keep it concise — it's loaded into context on every session.

---

## Devcontainer

Devcontainer ensures consistent dev environment across machines and Codespaces.

```json
// .devcontainer/devcontainer.json
{
  "name": "project-name",
  "image": "mcr.microsoft.com/devcontainers/base:ubuntu-22.04",
  "features": {
    "ghcr.io/devcontainers/features/node:1": { "version": "22" },
    "ghcr.io/devcontainers/features/python:1": { "version": "3.11" },
    "ghcr.io/devcontainers/features/docker-in-docker:2": {}
  },
  "postCreateCommand": "npm install",
  "remoteUser": "vscode"
}
```

**Patterns:**
- Use `postCreateCommand` for one-time setup (install deps, sync skills)
- Pin feature versions — avoid `latest`
- `remoteUser: vscode` is standard for devcontainers/base images

---

## GitHub Actions — Workflow Patterns

### Basic workflow structure
```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  contents: read          # always set minimum permissions

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@<SHA>  # v4 — always pin to SHA
      - name: Setup Node
        uses: actions/setup-node@<SHA>  # v4
        with:
          node-version: 22
          cache: npm
```

### Path-based triggers (only run when relevant files change)
```yaml
on:
  push:
    paths:
      - "src/**"
      - ".github/workflows/ci.yml"
  pull_request:
    paths:
      - "src/**"
```

### Reusable workflow (call from other workflows)
```yaml
# .github/workflows/reusable-lint.yml
on:
  workflow_call:
    inputs:
      node-version:
        type: string
        default: "22"
```

### Manual trigger
```yaml
on:
  workflow_dispatch:    # adds "Run workflow" button in GitHub UI
```

### Concurrency — cancel in-progress runs on new push
```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

---

## Dependabot

Add to every repo — keeps actions and dependencies updated:

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: github-actions
    directory: /
    schedule:
      interval: weekly
    groups:
      actions:
        patterns: ["*"]

  - package-ecosystem: npm       # add relevant ecosystem
    directory: /
    schedule:
      interval: weekly
```

Dependabot PRs for actions include the new SHA — review and merge to keep pinning current.

---

## Repository Validation & CI Testing

### Philosophy: TDD for Infrastructure and Content

**Tests first, content second.** Before creating or modifying content in a repo,
ask what makes it valid and write CI checks for it. This ensures the repo stays
correct, parseable, and deployable over time.

### Step 1: Ask Before Writing Tests

When working on a new repo or adding CI validation, always ask first:

1. **What type of content is in this repo?**
   (code, config, IaC, docs, presentations, data...)
2. **What does "valid" mean for this content?**
   (syntax correct, schema valid, builds successfully, passes lint...)
3. **What does "working" mean?**
   (deploys, tests pass, renders correctly, idempotent...)
4. **What are the failure modes we want to catch early?**
   (broken links, missing required fields, secrets accidentally committed...)

Then propose a test suite — user confirms — then write the tests — then write the content.

### Step 2: Match Tests to Content Type

| Content type | Suggested tests |
|---|---|
| Ansible roles | Molecule (converge + idempotence + verify) |
| Docker Compose | `docker compose config --quiet --no-interpolate` |
| Terraform/OpenTofu | `terraform validate`, `tflint` |
| Node.js / npm | `npm ci`, `npm test`, `npm run build` |
| Python | `pytest`, `ruff`, `mypy` |
| Markdown/docs | `markdownlint`, broken link checker |
| YAML config files | `yamllint`, schema validation |
| Slidev presentations | `npm run build` |
| Shell scripts | `shellcheck` |
| Secrets (any repo) | Trivy `--scanners secret` or `gitleaks` |
| Misconfigs (any repo) | Trivy `--scanners misconfig` |

### Step 3: CI Workflow Pattern

```yaml
name: Validate

on: [push, pull_request]

jobs:
  validate:
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@<SHA>  # v4

      # Add steps based on content type — see table above
      # Always include secret scanning as a baseline:
      - name: Scan for secrets
        uses: aquasecurity/trivy-action@<SHA>
        with:
          scan-type: fs
          scanners: secret
          severity: CRITICAL,HIGH
          exit-code: 1
```

### Step 4: Required Status Checks

Once CI is green, add the validate job to branch protection required checks:
Settings → Branches → Ruleset → Require status checks → add job name.

This ensures nothing merges without passing validation.

### Key Principle

> Never add content to a repo without a CI check that would catch it being broken.
> If you can't test it automatically, document why and what the manual check is.

Recommended settings for `main`:

- ✅ Require pull request before merging
- ✅ Require status checks to pass (add CI jobs here)
- ✅ Require branches to be up to date before merging
- ✅ Do not allow bypassing the above settings
- ✅ Restrict force pushes

Set via: Settings → Branches → Add branch ruleset

---

## README Structure

```markdown
# Project Name

One paragraph describing what this is.

## Setup

Steps to get running locally.

## Usage

How to use it.

## Deploy

How to deploy / CI/CD overview.
```

For course/presentation repos (Czechitas), follow the template in the czechitas-slidev skill.

---

## .gitignore Essentials

Always include:
```
# Secrets and generated files
*.env
*.yml.secret
host_vars/*.yml    # for Ansible (generated from 1Password)

# OS
.DS_Store
Thumbs.db

# Editor
.vscode/
.idea/
*.swp
```

Use `gitignore.io` or GitHub's template for language-specific ignores.

---

## Commit Conventions

Use conventional commits for clarity:
```
feat: add new scenario
fix: correct translation in Czech
chore: update dependencies
docs: update README
ci: add SHA pinning to workflows
```

---

## Common Pitfalls

- **Secrets in workflows** — never use `env:` for secrets, always `${{ secrets.NAME }}`
- **`npm install` in CI** — use `npm ci` to enforce lockfile (see github-security skill)
- **Missing `cache:` in setup-node/setup-python** — slows down CI unnecessarily
- **No path filters** — without `paths:`, every push triggers every workflow
- **`latest` in devcontainer features** — pin versions for reproducibility
