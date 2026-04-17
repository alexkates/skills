# Agent Skills

A collection of agent skills installable via [`npx skills`](https://agentskills.io).

## Installation

```sh
npx skills@latest add alexkates/skills/<skill-name>
```

## Skills

### API & Integration

#### api-skill-generator

Point this skill at any API's documentation URL and it will crawl the docs, extract everything an agent needs to know (authentication, endpoints, schemas, rate limits, error codes, pagination, gotchas), and produce a complete, ready-to-install skill directory.

The generated output includes:

- **SKILL.md** with setup instructions, auth details, available operations, rate limits, and gotchas
- **`scripts/request.py`** — authenticated request helper with automatic retry and rate-limit backoff
- **`scripts/paginate.py`** — auto-pagination wrapper (cursor, offset, or page-based)
- **`references/`** — full endpoint reference, error codes, and schemas for progressive disclosure

Secrets are handled via environment variables — nothing is ever hardcoded.

```sh
npx skills@latest add alexkates/skills/api-skill-generator
```
