# Crawl Strategy for API Documentation

How to systematically extract structured API knowledge from a documentation URL.

## Phase 1: Reconnaissance

Start by fetching the root URL. Identify what kind of documentation site it is:

### Common doc site patterns

| Pattern | How to navigate |
|---------|----------------|
| **Single-page API reference** (e.g. a long scrollable page) | Fetch the whole page; extract sections by heading |
| **Multi-page docs** (sidebar nav with links) | Extract nav links from the sidebar/TOC, then fetch each page |
| **OpenAPI/Swagger UI** | Look for a link to the raw spec (usually `/openapi.json`, `/swagger.json`, `/api-docs`) — if found, fetch it and skip most crawling |
| **Readme.com / Mintlify / GitBook / Slate** | These have consistent structures; extract nav from sidebar |
| **GitHub README** | Usually a single page; may link to more detailed docs |

### What to look for on the landing page

1. **Navigation structure** — sidebar links, table of contents, "API Reference" link
2. **Authentication section** — often linked prominently or in a "Getting Started" guide
3. **OpenAPI/Swagger spec link** — check for links containing `openapi`, `swagger`, `api-docs`, `spec`
4. **Base URL** — often shown in code examples on the landing page
5. **Rate limit info** — sometimes on the landing page, sometimes buried in a dedicated page

## Phase 2: Prioritized Crawling

Don't crawl everything blindly. Prioritize pages in this order:

1. **Authentication / Getting Started** — always crawl first
2. **API Reference index** — the page that lists all endpoints
3. **Individual endpoint pages** — grouped by resource
4. **Rate Limits / Usage Limits page** — if a dedicated page exists
5. **Error Handling / Error Codes page** — if a dedicated page exists
6. **Changelog / Migration guides** — only if you need to understand deprecations

### Crawling budget

- For small APIs (< 20 endpoints): crawl everything
- For medium APIs (20-100 endpoints): crawl auth + rate limits + the 10 most important
  resources fully, summarize the rest from the index page
- For large APIs (100+ endpoints): crawl auth + rate limits + the index, then ask the user
  which resources to focus on. Generate references for the focused resources and a lightweight
  index for the rest.

## Phase 3: Extraction

For each page, extract structured information. Here's what to look for by section type:

### Authentication pages

Extract:
- Auth mechanism (API key, Bearer token, OAuth2, Basic Auth, custom)
- Where the credential goes (header name, query parameter name, request body field)
- How to obtain credentials (dashboard URL, CLI command, OAuth flow)
- Scopes or permissions (if OAuth2)
- Token expiry and refresh behavior (if applicable)
- Example authenticated request (copy the code sample if present)

### Endpoint pages

Extract per endpoint:
- HTTP method + URL path (e.g. `GET /v1/users/{id}`)
- Description (one sentence)
- Path parameters (name, type, description)
- Query parameters (name, type, required/optional, default, description)
- Request body schema (field names, types, required/optional, constraints)
- Response schema (field names, types, envelope structure)
- Pagination info (if a list endpoint — cursor field, page size param, max page size)
- Error codes specific to this endpoint
- Code examples (useful for understanding request format)

### Rate limit pages

Extract:
- Global rate limit (requests per second/minute/hour)
- Per-endpoint limits (if any)
- Rate limit headers (e.g. `X-RateLimit-Remaining`, `Retry-After`)
- What happens when rate limited (HTTP 429? specific error body?)
- Retry guidance (backoff strategy, retry-after header)
- Burst allowance (if mentioned)

### Error pages

Extract:
- Error response format (JSON shape, fields like `error.code`, `error.message`)
- List of error codes/types with meanings
- HTTP status code mapping
- Retry guidance per error type (which are retryable?)

## Phase 4: OpenAPI Shortcut

If you find an OpenAPI spec (v2 or v3), it accelerates everything:

1. Fetch the raw JSON/YAML spec
2. Extract auth from `securityDefinitions` / `components.securitySchemes`
3. Extract all paths, methods, parameters, request bodies, responses
4. Still crawl the human-readable docs for: rate limits, gotchas, getting-started guides,
   and any information not captured in the spec
5. Save the spec as `assets/openapi.json` in the generated skill

The spec is a machine-readable source of truth for schemas and endpoints, but human docs
almost always contain critical context (gotchas, rate limits, auth setup) that the spec lacks.

## Tips

- **Follow redirects** — many doc sites redirect from old to new URLs.
- **Check for versioned docs** — look for `/v1/`, `/v2/` in URLs. Default to the latest
  stable version unless the user specifies otherwise.
- **Watch for "beta" or "deprecated" labels** — note these in the generated skill's gotchas.
- **Code examples are gold** — they reveal the actual request/response shapes more reliably
  than prose descriptions.
- **Look for SDKs** — if the API has an official SDK, note it in the generated skill but
  still generate the raw HTTP scripts (the skill should work without SDK dependencies).
