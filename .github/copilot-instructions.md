# Project Guidelines

## Overview

This is a collection of agent skills installable via `npx skills@latest add alexkates/skills/<skill-name>`. Each skill is a self-contained directory with a `SKILL.md` and optional supporting files (scripts, references, assets).

## Skill Structure

Every skill directory must contain a `SKILL.md` with YAML frontmatter (`name`, `description`) and follow the [Agent Skills specification](https://agentskills.io).

```
<skill-name>/
├── SKILL.md          # Main skill instructions with YAML frontmatter
├── scripts/          # Executable helpers (optional)
├── references/       # Supporting docs for progressive disclosure (optional)
└── assets/           # Static files like schemas or templates (optional)
```

## Conventions

- Skill directory names use kebab-case
- `SKILL.md` frontmatter must include `name` and `description`
- The `description` field is the discovery surface — include trigger phrases so agents know when to invoke the skill
- Reference files go in `references/`, scripts in `scripts/`, static assets in `assets/`
- Keep `SKILL.md` focused on workflow steps; push detailed reference material into `references/`

## Writing Style

- Be direct and actionable — skills are instructions for agents, not documentation for humans
- Use numbered steps for sequential workflows
- Use progressive disclosure — summarize in SKILL.md, detail in references
