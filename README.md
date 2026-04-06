# claude-skills

Personal Claude skill library — context files that guide Claude's behavior for recurring workflows.

## What are skills?

A skill is a `SKILL.md` file with instructions that Claude reads before starting work on a specific topic. Instead of re-explaining context every session, Claude picks up the relevant skill automatically and follows its conventions.

## Skills

| Skill | Description |
|---|---|
| `opsec` | PAP protocol enforcement — tracks data classification and blocks actions that would violate it |
| `malwoverview` | Static malware analysis workflow using malwoverview, with PAP-aware API enrichment |
| `czechitas-slidev` | Creating and converting presentations for Czechitas Digital Academy using the Slidev template |
| `ansible` | Ansible workflow for infrastructure.sramkovi.family — roles, Molecule testing, 1Password secrets |
| `docker` | Docker Swarm stack management for home infrastructure — Traefik, Graylog, compose conventions |
| `system-hardening` | Ubuntu server hardening per NIST/CISA — SSH, sysctl, UFW, fail2ban, OCI specifics |
| `github` | GitHub best practices — workflows, devcontainer, branch protection, repo validation |
| `github-security` | GitHub Actions security — SHA pinning, Harden-Runner, zizmor, supply chain hardening |

## Installation

### Local (macOS / Linux)

```bash
git clone https://github.com/srameko/claude-skills.git ~/.claude/skills
```

### Devcontainer / Codespaces

Add to `.devcontainer/devcontainer.json`:

```json
{
  "postCreateCommand": "git clone https://github.com/srameko/claude-skills.git ~/.claude/skills"
}
```

### CLAUDE.md integration

Add to your global `~/.claude/CLAUDE.md`:

```markdown
## Skills
Read the relevant skill from ~/.claude/skills/ before starting work.
```

## Usage

Skills are picked up automatically by Claude Code based on context. You can also reference them explicitly:

```
Use the opsec skill for this analysis.
Read ~/.claude/skills/github/SKILL.md before we start.
```

## Structure

```
claude-skills/
├── <skill-name>/
│   └── SKILL.md
└── ...
```

Each skill follows the [Claude skill format](https://docs.claude.ai) with YAML frontmatter (`name`, `description`) and markdown instructions.