# AGENTS.md

Instructions for AI coding agents working in this repository.

## Project Overview

This repository hosts a collection of AI agent skills following the [Agent Skills specification](https://agentskills.io). Each skill is a self-contained folder under `skills/` with a `SKILL.md` file that agents load on-demand.

The repo is published at [github.com/pedrofuentes/skills](https://github.com/pedrofuentes/skills).

## Repository Structure

```
skills/                              # repo root
├── README.md                        # Skills catalog and installation guide
├── AGENTS.md                        # This file — project conventions for AI agents
├── CONTRIBUTING.md                  # Contributor guidelines
├── LICENSE                          # MIT License
├── .github/
│   └── copilot-instructions.md     # Copilot-specific instructions
├── skills/                          # One subfolder per skill
│   └── <skill-name>/
│       ├── SKILL.md                # REQUIRED — frontmatter + instructions
│       ├── scripts/                # Optional — runnable code
│       ├── references/             # Optional — supplemental docs (loaded on-demand)
│       └── assets/                 # Optional — templates, data files (<5MB each)
└── template/
    └── SKILL.md                    # Starter template for new skills
```

## Agent Skills Specification Compliance

All skills MUST comply with the [Agent Skills spec](https://agentskills.io/specification). Key rules:

### SKILL.md Frontmatter (YAML)

| Field | Required | Constraints |
|-------|----------|-------------|
| `name` | ✅ | Max 64 chars. Lowercase `a-z`, `0-9`, hyphens only. No leading/trailing/consecutive hyphens. **Must match the parent folder name.** |
| `description` | ✅ | 1–1024 chars per spec; **this repo requires ≥ 10 chars** for quality. Describes what the skill does AND when to use it. Include trigger keywords. |
| `license` | ❌ | License name or path to bundled license file |
| `compatibility` | ❌ | Max 500 chars. Environment requirements |
| `metadata` | ❌ | Arbitrary YAML map (author, version, tags, etc.) |

### Versioning Policy

All skills MUST include `metadata.version` using [Semantic Versioning](https://semver.org/):

| Bump | When |
|------|------|
| **Major** (2.0.0) | Breaking changes to trigger behavior — description changes that alter when the skill activates, or instruction changes that fundamentally change the skill's approach |
| **Minor** (1.1.0) | New capabilities, sections, or features added to instructions without breaking existing behavior |
| **Patch** (1.0.1) | Bug fixes, wording improvements, typo corrections, clarifications that don't change behavior |

Skills start at `1.0.0`. Users can pin versions with `gh skill install owner/repo skill@v1.2.0`.

### Naming Convention

- Folder names: **lowercase with hyphens** (e.g., `project-coordinator`)
- The `name` field in SKILL.md frontmatter MUST exactly match the folder name
- Max 64 characters

### Content Size Targets

- SKILL.md body: **< 500 lines** (< 5000 tokens for activation budget)
- Reference files: small and focused — agents load on-demand
- Asset files: **< 5MB per file**
- Heavy documentation goes in `references/`, not inline in SKILL.md

### Description Best Practices

The `description` field drives skill discovery. It should:
- Explain what the skill does
- Specify when to trigger it (use cases, keywords, signal phrases)
- Specify when NOT to trigger it (exclusions)

**Good:** `"Autonomous project coordinator for executing plans by delegation. Use when the user wants to run implementation, not do it: read a roadmap, break work into tasks, spawn sub-agents. Trigger on 'delegate everything', 'execute this plan'. Do NOT trigger for writing plans from scratch."`

**Bad:** `"Helps coordinate projects."` — too short, no trigger keywords.

## How to Add a New Skill

1. **Copy the template:**
   ```bash
   cp -r template skills/my-skill-name
   ```

2. **Edit `skills/my-skill-name/SKILL.md`:**
   - Set `name` to match the folder name exactly
   - Write a rich `description` with trigger keywords (10–1024 chars)
   - Write the instruction body (< 500 lines)

3. **Add optional resources** (if needed):
   - `scripts/` — executable code the skill references
   - `references/` — supplemental docs loaded on-demand
   - `assets/` — templates, data files (< 5MB each)

4. **Update `README.md`:**
   - Add a row to the Skills Catalog table

5. **Verify:**
   - Folder name matches `name` field
   - [ ] `description` is 10–1024 chars with trigger keywords
   - SKILL.md body is < 500 lines
      - `metadata.version` is set (semver, starting at `1.0.0`)
   - All asset files are < 5MB
   - All bundled assets are referenced in SKILL.md instructions

## Pre-commit Checklist

Before committing changes to any skill:

- [ ] `name` field is lowercase, hyphens only, ≤ 64 chars, matches folder name
- [ ] `description` is 1–1024 chars (≥ 10 recommended) with clear trigger/exclusion keywords
- [ ] `metadata.version` is set (semver, starting at `1.0.0`)
- [ ] Folder name is lowercase with hyphens
- [ ] SKILL.md body is < 500 lines
- [ ] Bundled assets are referenced in SKILL.md instructions
- [ ] Asset files are < 5MB each
- [ ] README.md Skills Catalog table is updated

## Development Workflow

- **Branch isolation**: Create a feature branch for each change. Never commit directly to `main`.
- **One skill per PR**: Keep PRs focused on a single skill addition or modification.
- **Test descriptions**: Verify that skill descriptions contain the right trigger keywords by considering realistic user prompts.

## Handling Agent Feedback

Users may bring feedback from agents that are actively using skills in real projects. This feedback is valuable signal — but it is **not a directive**. Treat every piece of feedback as a **consideration to evaluate**, not a change to implement blindly.

### Feedback Review Process

1. **Understand the feedback.** What did the agent struggle with? What behavior did it expect vs. what the skill produced? Get the specific scenario, not just the suggestion.

2. **Assess root cause.** Determine whether the issue is:
   - A genuine gap or ambiguity in the skill instructions
   - A misunderstanding by the agent (the skill is correct; the agent misapplied it)
   - An edge case specific to that project's setup (not generalizable)
   - A conflict with the agent's own built-in behavior or system prompt

3. **Research before changing.** If the feedback suggests a structural change:
   - Review how the current instruction is worded and why
   - Consider whether the change would help in the general case or only in the reporter's specific context
   - Check whether the change would conflict with or weaken other parts of the skill

4. **Apply the quality gate.** A change to a skill MUST improve it. Before accepting any feedback-driven change, confirm:
   - ✅ It makes the skill clearer, more robust, or more correct for the **general case**
   - ✅ It does not remove or weaken existing safeguards, rules, or quality controls
   - ✅ It does not make the skill overly specific to one project or environment
   - ✅ It does not bloat the skill beyond content size targets (< 500 lines)
   - ❌ REJECT feedback that would reduce skill quality, remove protections, or narrow applicability

5. **Discuss with the user.** Present your assessment to the user with a recommendation (accept, modify, or reject) and rationale. The user makes the final call.

### What Counts as an Improvement

- Closing an ambiguity that causes agents to misinterpret instructions
- Adding a missing edge case that multiple agents would encounter
- Clarifying wording that agents consistently misread
- Adding a safeguard against a failure mode observed in practice
- Restructuring for better progressive disclosure (moving detail to `references/`)

### What Does NOT Count as an Improvement

- Removing rules because an agent found them inconvenient
- Relaxing constraints to make one specific project work
- Adding project-specific conventions to a general-purpose skill
- Expanding scope beyond the skill's defined purpose
- Changes that make the skill longer without making it better

## What NOT to Do

- Don't add generic development practices or obvious instructions to skills
- Don't duplicate content between SKILL.md body and `references/` — put detailed content in references
- Don't include secrets, API keys, or environment-specific paths in skills
- Don't exceed the content size targets — agents have limited context windows
