# Skills

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

A curated collection of AI agent skills for [GitHub Copilot](https://github.com/features/copilot), [Claude Code](https://docs.anthropic.com/en/docs/agents-and-tools/claude-code), and any agent that supports the [Agent Skills specification](https://agentskills.io).

## What Are Agent Skills?

Agent Skills are reusable instruction sets that extend what AI coding agents can do. Each skill is a folder containing a `SKILL.md` file with structured instructions that agents load on-demand when a task matches the skill's description.

Skills follow the open [Agent Skills specification](https://agentskills.io/specification):
- **Discovery** — only the skill name and description are loaded at startup (~100 tokens)
- **Activation** — the full instruction body is injected when the agent decides the task matches
- **Execution** — bundled scripts, references, and assets are loaded as needed

## Skills Catalog

| Skill | Version | Description |
|-------|---------|-------------|
| [sentinel-project-coordinator](./skills/sentinel-project-coordinator/) | 2.0.0 | Autonomous project coordinator for **agents-template / Sentinel** projects — delegates all implementation to sub-agents and drives the SENTINEL.md TDD-and-review workflow: breaks work into tasks, spawns sub-agents, parallelizes, invokes Sentinel, verifies merges, tracks progress. |

## Installation

### Via GitHub CLI (recommended)

```bash
# Install a specific skill (latest)
gh skill install pedrofuentes/skills sentinel-project-coordinator

# Install a pinned version
gh skill install pedrofuentes/skills sentinel-project-coordinator@v2.0.0

# Install to personal scope (available across all projects)
gh skill install pedrofuentes/skills sentinel-project-coordinator --scope user
```

> Requires [GitHub CLI](https://cli.github.com/) ≥ 2.90.0 with `gh skill` support.

### Manual Installation

Copy the skill folder to one of these locations:

| Scope | Path |
|-------|------|
| **Project** (repo-specific) | `.github/skills/<skill-name>/` or `.claude/skills/<skill-name>/` |
| **Personal** (all projects) | `~/.copilot/skills/<skill-name>/` or `~/.claude/skills/<skill-name>/` |

```bash
# Example: install sentinel-project-coordinator for personal use (Copilot)
cp -r skills/sentinel-project-coordinator ~/.copilot/skills/

# Example: install for Claude Code
cp -r skills/sentinel-project-coordinator ~/.claude/skills/
```

## Usage

Once installed, skills are automatically discovered by your AI agent. Just describe what you want to do — the agent will activate the matching skill based on its description and trigger keywords.

For example, with `sentinel-project-coordinator` installed, simply say:
> "Execute this implementation plan" or "Coordinate the project and delegate tasks"

## Creating a New Skill

Use the [template](./template/SKILL.md) as a starting point:

```bash
cp -r template skills/my-new-skill
# Edit skills/my-new-skill/SKILL.md
```

See [CONTRIBUTING.md](CONTRIBUTING.md) for full guidelines.

## Contributing

We welcome contributions! Please read [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines on adding new skills, naming conventions, and the PR process.

## License

This project is licensed under the MIT License — see [LICENSE](LICENSE) for details.
