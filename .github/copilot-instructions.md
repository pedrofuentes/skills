# Copilot Instructions

This repository hosts AI agent skills following the [Agent Skills specification](https://agentskills.io). See [AGENTS.md](../AGENTS.md) at the project root for comprehensive context.

## Key Rules

When working in this repository, follow these conventions:

### Skill Structure
- Each skill lives in its own folder under `skills/`
- Every skill folder MUST contain a `SKILL.md` with YAML frontmatter (`name` + `description`)
- The `name` field MUST match the parent folder name exactly
- Names: lowercase `a-z`, `0-9`, single hyphens; no leading/trailing/consecutive hyphens; max 64 chars

### Content Limits
- SKILL.md body: < 500 lines (< 5000 tokens)
- Asset files: < 5MB each
- Detailed docs go in `references/` subfolder, not inline

### Description Quality
- `description` must be 1–1024 characters per spec (repo minimum: 10 chars)
- Include trigger keywords and signal phrases
- Include exclusions (when NOT to trigger)
- This field drives skill discovery — invest time in it

### Adding a New Skill
1. Copy `template/SKILL.md` to `skills/<skill-name>/SKILL.md`
2. Set `name` to match the folder name
3. Write description with trigger keywords
4. Write instruction body
5. Add a row to the Skills Catalog table in `README.md`

### Code Review Focus
When reviewing skill PRs, verify:
- Frontmatter compliance (name matches folder, description length)
- Description quality (trigger keywords, exclusions present)
- Body size (< 500 lines)
- README catalog updated
