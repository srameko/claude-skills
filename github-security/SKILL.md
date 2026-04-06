---
name: github-security
description: >
  GitHub security hardening — Actions, supply chain, CI/CD pipelines. Use this skill whenever the user works on
  GitHub Actions workflows, adds or updates actions, reviews CI/CD pipelines,
  or implements supply chain security improvements. Trigger for SHA pinning,
  Harden-Runner, zizmor static analysis, Dependabot/Renovate config, OpenSSF
  Scorecard, or any supply chain security topic. Always use this skill when
  touching .github/workflows/ files — even for small changes.
---

# GitHub Security Skill

> For general GitHub workflow patterns, devcontainer, repo structure and branch protection — see the **github** skill.

## Context
Ondřej experienced a supply chain attack via unpinned action tag (trivy-action).
All hardening recommendations below are grounded in that firsthand experience.

---

## Core Principle: SHA Pinning

**Always pin actions to a full commit SHA, never to a tag or branch.**

```yaml
# ❌ VULNERABLE — tag can be moved
- uses: actions/checkout@v4
- uses: aquasecurity/trivy-action@v0.35.0

# ✅ SAFE — SHA is immutable
- uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2
- uses: aquasecurity/trivy-action@57a97c7e7821a5776cebc9bb87c984fa69cba8f1  # v0.35.0
```

**How to get the SHA:**
```bash
# From GitHub UI: commit history of the action repo → copy full SHA
# Or via API:
gh api repos/actions/checkout/git/ref/tags/v4 --jq '.object.sha'
```

Always include the tag as a comment so humans know what version it is.

---

## StepSecurity Harden-Runner

Adds runtime security monitoring to jobs — detects unexpected network calls,
file writes, and process executions.

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: step-security/harden-runner@v2
        with:
          egress-policy: audit        # start with audit, move to block after baseline
          # egress-policy: block      # block unexpected outbound connections
          # allowed-endpoints: >
          #   github.com:443
          #   pypi.org:443
```

**Workflow:**
1. Start with `egress-policy: audit`
2. Review baseline in StepSecurity dashboard
3. Move to `egress-policy: block` with explicit `allowed-endpoints`

---

## zizmor — Static Analysis

Scans workflows for security issues (injection, excessive permissions, etc.)

```bash
# Install
pip install zizmor

# Scan all workflows
zizmor .github/workflows/

# Scan specific file
zizmor .github/workflows/deploy.yml
```

Run zizmor before committing any workflow change.

---

## Permissions — Least Privilege

Always set minimal permissions at job level:

```yaml
jobs:
  build:
    permissions:
      contents: read        # default: read only
      # Add only what's needed:
      pages: write          # only for Pages deploy jobs
      id-token: write       # only for OIDC
      packages: write       # only for container publish
```

Set `permissions: {}` at workflow level to deny all by default, then grant per job.

---

## Dependabot / Renovate

**Dependabot** (already configured in docker repo):
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
```

Add to every repo. Dependabot creates PRs for action updates — review and merge
to keep SHAs current.

**Renovate** — alternative with more flexibility, can auto-merge patch updates.

---

## Trivy in CI

Scan for secrets and misconfigs (as in docker repo):
```yaml
- name: Scan for secrets and misconfigs
  uses: aquasecurity/trivy-action@<SHA>  # v0.35.0
  with:
    scan-type: fs
    scan-ref: .
    scanners: secret,misconfig
    severity: CRITICAL,HIGH
    exit-code: 1
```

Also scan container images before push:
```yaml
- name: Scan image
  uses: aquasecurity/trivy-action@<SHA>
  with:
    scan-type: image
    image-ref: ${{ env.IMAGE }}
    severity: CRITICAL,HIGH
    exit-code: 1
```

---

## OpenSSF Scorecard

Automated security scoring for repos:
```yaml
# .github/workflows/scorecard.yml
- uses: ossf/scorecard-action@<SHA>
  with:
    results_file: results.sarif
    results_format: sarif
    publish_results: true
```

Key checks: Branch protection, dependency pinning, CI tests, code review.

---

## Hardening Checklist (per workflow)

- [ ] All actions pinned to full commit SHA with tag comment
- [ ] `permissions:` set to minimum required per job
- [ ] Harden-Runner added (at least in audit mode)
- [ ] No secrets in env vars — use `${{ secrets.NAME }}`
- [ ] No `pull_request_target` with untrusted code checkout
- [ ] zizmor passes with no HIGH/CRITICAL findings
- [ ] Dependabot enabled for github-actions ecosystem
- [ ] Trivy scan in CI for secrets and misconfigs

---

## Supply Chain Lessons Learned

- **Tag compromise (trivy-action incident):** Attacker moved a tag to malicious
  commit. SHA pinning would have prevented execution of the malicious code entirely.
- **Transitive dependency theft:** Socket Firewall blocks known-malicious package
  installs but does NOT catch credential theft via unpinned transitive deps.
  SHA pinning is the correct mitigation — not Socket Firewall alone.
- **Pip cache bypass:** Always purge pip cache before `sfw pip install` in CI —
  cached packages bypass Socket Firewall network interception.

### npm Supply Chain (axios, March 2026)

Compromised maintainer account → malicious version published → hidden `postinstall`
dependency dropped a RAT. Projects using `^` ranges pulled it automatically.

**Lessons:**
- Use `npm ci` in CI — enforces lockfile, prevents version drift
- Pin exact versions in `package.json` — remove `^` and `~` for critical packages
- Use `--ignore-scripts` in CI to block `postinstall` hooks entirely
- Migrate to OIDC Trusted Publishing — long-lived npm tokens are a liability
- `npm config set min-release-age 3` — 72h quarantine on newly published versions
- Harden-Runner egress monitoring detects unexpected C2 callbacks in CI

---

## Repos and Current State

| Repo | SHA pinning | Harden-Runner | zizmor | Dependabot |
|---|---|---|---|---|
| docker.home.ondrejsramek.cz | ✅ (checkout, trivy) | ❌ | ❌ | ✅ (actions) |
| infrastructure.sramkovi.family | ❌ (tags only) | ❌ | ❌ | ❌ |
| skoleni-kyberneticke-bezpecnosti | unknown | ❌ | ❌ | ❌ |
