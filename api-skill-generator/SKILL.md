---
name: api-skill-generator
description: >
  Generates a complete agent skill directory from API documentation. Given a URL to the root of
  any API's docs, this skill crawls the documentation, extracts authentication, rate limits,
  resources, actions, request/response schemas, error codes, and constraints, then produces a
  ready-to-install skill directory (SKILL.md + scripts + references). Use this skill whenever
  someone asks to "create a skill for an API", "generate a skill from docs", "wrap an API as a
  skill", "make a skill from a URL", or mentions turning API documentation into an agent skill.
  Also trigger when someone pastes an API docs URL and asks to make it usable by an agent.
---

# API Skill Generator

Turns API documentation into a complete, installable agent skill.

## Overview

This skill crawls an API's documentation site, extracts structured knowledge about the API, and
produces a skill directory that any agent can use to interact with that API. The output follows
the Agent Skills specification (agentskills.io).

## Workflow

### Step 1: Crawl and Extract

Given a root URL to API documentation, systematically crawl it to build a complete picture of
the API. Read `references/crawl-strategy.md` for the detailed crawling procedure.

The goal is to extract:

1. **Authentication** — auth type (API key, OAuth2, Bearer token, Basic), where credentials go
   (header, query param, body), header names, token flows, scopes
2. **Base URL(s)** — production, staging, sandbox environments
3. **Rate limits** — requests per second/minute/hour, per-endpoint limits, burst allowances,
   retry-after behavior
4. **Resources and endpoints** — every resource (e.g. Users, Orders, Products), its endpoints,
   HTTP methods, URL patterns
5. **Request schemas** — required/optional parameters, types, validation rules, defaults
6. **Response schemas** — success shapes, pagination patterns (cursor, offset, page), envelope
   structure
7. **Error handling** — error response format, common status codes and meanings, retry guidance
8. **Constraints and gotchas** — max page sizes, field length limits, deprecated endpoints,
   quirks mentioned in docs

### Step 2: Handle Secrets

Almost every API requires a secret key or token. The generated skill must handle this cleanly.

Read `references/secrets-guide.md` for the full approach, but the summary is:

- **Never hardcode secrets** in the skill or its scripts.
- Scripts accept secrets via **environment variables** with a conventional name
  (e.g. `STRIPE_API_KEY`, `OPENAI_API_KEY`).
- The generated SKILL.md includes a **Setup section** that tells the user exactly which env
  vars to set and where to get the key.
- For OAuth2 flows, the skill documents the token exchange steps and stores tokens in a
  conventional location.

### Step 3: Generate the Skill Directory

Produce a complete skill directory with this structure:

```
<api-name>/
├── SKILL.md              # Main instructions
├── scripts/
│   ├── request.py        # Generic authenticated request helper
│   └── paginate.py       # Pagination helper (if API uses pagination)
├── references/
│   ├── endpoints.md      # Full endpoint reference (progressive disclosure)
│   ├── error-codes.md    # Error code reference
│   └── schemas.md        # Key request/response schemas (optional, for complex APIs)
└── assets/
    └── openapi.json      # OpenAPI spec if one was discovered (optional)
```

Read `references/output-templates.md` for the exact templates to follow when generating each file.

#### Generated SKILL.md structure

The generated skill's SKILL.md should follow this structure:

```markdown
---
name: <api-name>
description: <what this API skill does and when to trigger it>
compatibility: Requires Python 3.10+ and uv
---

# <API Name> Skill

## Setup

<How to get and configure the API key / credentials>

## Authentication

<Auth type, where it goes, example header>

## Quick Start

<The 2-3 most common operations with example commands>

## Available Operations

<Grouped by resource — brief summary of each endpoint>
For full endpoint details, read `references/endpoints.md`.

## Rate Limits

<Limits, retry strategy>

## Gotchas

<Non-obvious constraints, quirks, common mistakes>

## Error Handling

<Error format, common codes>
For a full list of error codes, read `references/error-codes.md`.
```

#### Generated scripts

**`scripts/request.py`** — A self-contained Python script (PEP 723 inline deps) that:
- Reads the API key from the appropriate env var
- Accepts method, path, query params, and body as CLI arguments
- Adds authentication headers/params automatically
- Returns JSON to stdout, diagnostics to stderr
- Handles rate limit responses with automatic retry + backoff
- Exits with meaningful codes (0 = success, 1 = client error, 2 = auth error, 3 = rate limit
  exhausted, 4 = server error)

**`scripts/paginate.py`** — If the API uses pagination, a script that:
- Wraps request.py to auto-paginate through result sets
- Supports the API's pagination style (cursor, offset, page number)
- Outputs concatenated results or streams them line-by-line

Both scripts must include `--help` output, use `argparse`, and produce structured JSON output.

### Step 4: Validate

After generating the skill:

1. Check that the SKILL.md frontmatter has valid `name` and `description`
2. Verify all script references in SKILL.md point to files that exist
3. Confirm the `request.py` script handles auth correctly for the API's auth type
4. Ensure rate limit handling matches what the docs specify
5. Sanity check that the endpoint reference covers the major resources found during crawl

### Step 5: Package and Present

If `present_files` is available, copy the generated skill directory to the output directory and
present it. If packaging tools are available (`npx skills` or `python -m scripts.package_skill`),
package it as a `.skill` file.

Tell the user:
- Which env var(s) they need to set
- How to install the skill (`npx skills add ./path/to/skill`)
- A quick smoke test they can try (the simplest GET endpoint)

## Important Principles

- **Progressive disclosure**: Keep the generated SKILL.md under 500 lines. Heavy reference
  material goes in `references/`.
- **Gotchas are gold**: Any non-obvious behavior, undocumented quirk, or common mistake found
  in the docs should go in the Gotchas section. This is the highest-value content.
- **Scripts must be non-interactive**: No prompts, no TTY input. Everything via CLI args and
  env vars.
- **Structured output**: Scripts emit JSON to stdout, diagnostics to stderr.
- **Idempotent operations**: Scripts should be safe to retry.
- **Pin dependencies**: All PEP 723 deps should have version pins.
