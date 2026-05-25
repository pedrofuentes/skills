# Contributing

Thank you for your interest in contributing to this skills collection! This guide covers everything you need to add a new skill or improve an existing one.

## Adding a New Skill

### 1. Set Up Your Skill Folder

```bash
# Copy the template
cp -r template skills/my-skill-name

# Edit the skill
# your-editor skills/my-skill-name/SKILL.md
```

### 2. Write the SKILL.md

Every skill requires a `SKILL.md` file with two parts:

**YAML Frontmatter** (between `---` markers):

```yaml
---
name: my-skill-name
description: 'Clear description of what the skill does and when to use it. Include trigger keywords like "build dashboard", "create report". Do NOT trigger for unrelated tasks like X, Y, Z.'
---
```

**Instruction Body** (Markdown after frontmatter):

Write the instructions that the AI agent will follow when this skill is activated. This is your skill's "brain" — be specific, structured, and actionable.

### 3. Frontmatter Requirements

| Field | Required | Rules |
|-------|----------|-------|
| `name` | ✅ | Lowercase `a-z`, `0-9`, single hyphens; no leading/trailing/consecutive hyphens; max 64 chars. **Must match folder name.** |
| `description` | ✅ | 1–1024 chars per spec (repo minimum: 10). Explain what it does, when to trigger, when NOT to trigger. |
| `license` | ❌ | If your skill has a different license than the repo |
| `compatibility` | ❌ | Environment requirements (e.g., "Requires Python 3.8+") |
| `metadata` | ❌ | Version, author, tags, etc. |

### 4. Description Tips

The `description` is the most important field — it determines when agents activate your skill.

**Do:**
- Include specific trigger phrases users would say
- Include exclusions (when NOT to trigger)
- Use 50–200 words for complex skills

**Don't:**
- Write vague one-liners ("Helps with stuff")
- Skip exclusions (causes false activations)

### 5. Optional Resources

You can bundle additional files with your skill:

| Folder | Purpose | Notes |
|--------|---------|-------|
| `scripts/` | Executable code | Python, Bash, Node.js scripts the skill references |
| `references/` | Supplemental docs | Loaded on-demand by the agent — keep focused |
| `assets/` | Templates, data | < 5MB per file |

### 6. Update the README

Add a row to the Skills Catalog table in `README.md`:

```markdown
| [my-skill-name](./skills/my-skill-name/) | Brief description of what it does. |
```

## Quality Checklist

Before submitting:

- [ ] Folder name matches `name` field exactly
- [ ] `name` is lowercase, hyphens only, ≤ 64 characters
- [ ] `description` is 1–1024 characters (≥ 10 recommended) with trigger keywords
- [ ] SKILL.md body is < 500 lines
- [ ] All asset files are < 5MB
- [ ] All bundled assets are referenced in the instructions
- [ ] README.md Skills Catalog table is updated
- [ ] Skill follows the [Agent Skills spec](https://agentskills.io/specification)

## Pull Request Guidelines

- **One skill per PR** — keep changes focused
- **Branch from `main`** — create a feature branch (e.g., `add-skill/my-skill-name`)
- **Descriptive PR title** — e.g., "Add my-skill-name skill"
- **Test your skill** — install it locally and verify it activates on the right prompts

## Modifying Existing Skills

When improving an existing skill:

- Keep backward compatibility in mind — don't break trigger patterns
- Update the `description` if you change when the skill should activate
- Update `metadata.version` if the skill uses versioning
- Update the README catalog if the description changed significantly

## Questions?

Open an issue if you have questions about contributing or need help with the skill format.
