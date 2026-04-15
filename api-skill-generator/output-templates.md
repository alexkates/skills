# Output Templates

Templates for generating each file in the skill directory. Adapt these to the specific API —
don't use them verbatim if they don't fit.

## Generated SKILL.md Template

```markdown
---
name: <api-name>
description: >
  Interact with the <API Display Name> API. <One sentence about what the API does>.
  Use this skill when the user wants to <primary use cases>. Also trigger on mentions
  of <API name>, <key resource names>, or related actions like <common verbs>.
compatibility: Requires Python 3.10+ and uv
---

# <API Display Name>

## Setup

### 1. Get your API key

Go to [<Dashboard Name>](<dashboard-url>) and create a new API key.
<Any scope or permission notes.>

### 2. Set the environment variable

\`\`\`bash
export <API_NAME>_API_KEY="your-key-here"
\`\`\`

### 3. Verify

\`\`\`bash
uv run scripts/request.py GET <simplest-endpoint>
\`\`\`

## Authentication

All requests require <auth type>. The `scripts/request.py` helper adds this automatically.

<If header auth>: Sent as `<Header-Name>: <format>` header.
<If query auth>: Sent as `?<param_name>=<key>` query parameter.

## Quick Start

### <Most common operation 1>

\`\`\`bash
uv run scripts/request.py GET /v1/<resource>
\`\`\`

### <Most common operation 2>

\`\`\`bash
uv run scripts/request.py POST /v1/<resource> --body '{"field": "value"}'
\`\`\`

### <Most common operation 3>

\`\`\`bash
uv run scripts/request.py GET /v1/<resource>/{id}
\`\`\`

## Available Operations

<Grouped by resource. Keep this section brief — just method + path + one-line description.
Full details go in references/endpoints.md.>

### <Resource 1>

- `GET /v1/<resource>` — List all <resources>
- `POST /v1/<resource>` — Create a <resource>
- `GET /v1/<resource>/{id}` — Get a <resource>
- `PATCH /v1/<resource>/{id}` — Update a <resource>
- `DELETE /v1/<resource>/{id}` — Delete a <resource>

### <Resource 2>

...

For full parameter details, read `references/endpoints.md`.

## Rate Limits

<Global limit, e.g. "1000 requests per minute.">
<Per-endpoint limits if any.>

The `request.py` script automatically retries on HTTP 429 with exponential backoff.
Rate limit headers: `<header names>`.

## Gotchas

- <Non-obvious behavior 1>
- <Non-obvious behavior 2>
- <Deprecation warnings>
- <Common mistakes from the docs>
- <Field naming inconsistencies>
- <Pagination quirks>

## Error Handling

Errors return HTTP <status codes> with this shape:

\`\`\`json
<example error response>
\`\`\`

Common errors:
- `<code>` — <meaning and fix>
- `<code>` — <meaning and fix>

For the full list, read `references/error-codes.md`.
```

## Generated request.py Template

```python
# /// script
# dependencies = [
#   "httpx>=0.27,<1",
# ]
# ///

"""
Authenticated HTTP client for the <API Name> API.

Usage:
    uv run scripts/request.py METHOD /path [OPTIONS]

Examples:
    uv run scripts/request.py GET /v1/users
    uv run scripts/request.py POST /v1/users --body '{"name": "Alice"}'
    uv run scripts/request.py GET /v1/users --query 'limit=10&offset=0'
"""

import argparse
import json
import os
import sys
import time

import httpx

BASE_URL = "<base-url>"
ENV_VAR = "<API_NAME>_API_KEY"
MAX_RETRIES = 3
RETRY_BACKOFF = 1.0  # seconds, doubles each retry


def get_api_key() -> str:
    key = os.environ.get(ENV_VAR)
    if not key:
        print(f"Error: {ENV_VAR} environment variable is not set.", file=sys.stderr)
        print(f"Get your API key at: <dashboard-url>", file=sys.stderr)
        sys.exit(2)
    return key


def build_headers(api_key: str) -> dict:
    # Adapt this to the API's auth mechanism
    return {
        "Authorization": f"Bearer {api_key}",
        "Content-Type": "application/json",
        "Accept": "application/json",
    }


def parse_query(query_str: str | None) -> dict:
    if not query_str:
        return {}
    params = {}
    for pair in query_str.split("&"):
        if "=" in pair:
            k, v = pair.split("=", 1)
            params[k] = v
    return params


def make_request(
    method: str,
    path: str,
    headers: dict,
    params: dict | None = None,
    body: str | None = None,
) -> httpx.Response:
    url = f"{BASE_URL}{path}"
    kwargs: dict = {"headers": headers, "params": params, "timeout": 30.0}
    if body:
        kwargs["content"] = body

    for attempt in range(MAX_RETRIES + 1):
        response = httpx.request(method.upper(), url, **kwargs)

        if response.status_code == 429:
            if attempt == MAX_RETRIES:
                print("Error: Rate limit exhausted after retries.", file=sys.stderr)
                sys.exit(3)
            retry_after = response.headers.get("Retry-After")
            wait = float(retry_after) if retry_after else RETRY_BACKOFF * (2 ** attempt)
            print(f"Rate limited. Retrying in {wait:.1f}s...", file=sys.stderr)
            time.sleep(wait)
            continue

        return response

    # Unreachable, but satisfies type checkers
    sys.exit(3)


def main():
    parser = argparse.ArgumentParser(
        description="Authenticated HTTP client for the <API Name> API.",
        formatter_class=argparse.RawDescriptionHelpFormatter,
        epilog="""
Examples:
  uv run scripts/request.py GET /v1/users
  uv run scripts/request.py POST /v1/users --body '{"name": "Alice"}'
  uv run scripts/request.py GET /v1/users --query 'limit=10&offset=0'
        """,
    )
    parser.add_argument("method", help="HTTP method (GET, POST, PUT, PATCH, DELETE)")
    parser.add_argument("path", help="API path (e.g. /v1/users)")
    parser.add_argument("--body", help="JSON request body", default=None)
    parser.add_argument("--query", help="Query string (key=val&key2=val2)", default=None)
    args = parser.parse_args()

    api_key = get_api_key()
    headers = build_headers(api_key)
    params = parse_query(args.query)

    response = make_request(args.method, args.path, headers, params, args.body)

    # Exit codes by status class
    if response.status_code == 401 or response.status_code == 403:
        print(f"Auth error ({response.status_code}):", file=sys.stderr)
        try:
            print(json.dumps(response.json(), indent=2), file=sys.stderr)
        except Exception:
            print(response.text, file=sys.stderr)
        sys.exit(2)
    elif 400 <= response.status_code < 500:
        try:
            print(json.dumps(response.json(), indent=2))
        except Exception:
            print(response.text)
        sys.exit(1)
    elif response.status_code >= 500:
        print(f"Server error ({response.status_code}):", file=sys.stderr)
        try:
            print(json.dumps(response.json(), indent=2), file=sys.stderr)
        except Exception:
            print(response.text, file=sys.stderr)
        sys.exit(4)

    # Success
    try:
        print(json.dumps(response.json(), indent=2))
    except Exception:
        print(response.text)


if __name__ == "__main__":
    main()
```

## Generated paginate.py Template

Only generate this if the API uses pagination.

```python
# /// script
# dependencies = [
#   "httpx>=0.27,<1",
# ]
# ///

"""
Auto-paginating client for the <API Name> API.

Wraps request.py to iterate through paginated results.

Usage:
    uv run scripts/paginate.py /path [OPTIONS]

Examples:
    uv run scripts/paginate.py /v1/users --max-pages 5
    uv run scripts/paginate.py /v1/orders --query 'status=active'
"""

import argparse
import json
import os
import subprocess
import sys


SKILL_DIR = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))
REQUEST_SCRIPT = os.path.join(SKILL_DIR, "scripts", "request.py")

# -- Adapt these to the API's pagination style --

# Cursor-based:    next cursor is in response body at this key path
# Offset-based:    use offset + limit query params
# Page-based:      use page query param
PAGINATION_STYLE = "cursor"  # "cursor" | "offset" | "page"

# For cursor-based: JSON key path to the next cursor value (e.g. "meta.next_cursor")
CURSOR_KEY = "meta.next_cursor"
# For cursor-based: query param name to pass the cursor
CURSOR_PARAM = "starting_after"

# For offset-based:
OFFSET_PARAM = "offset"
LIMIT_PARAM = "limit"
DEFAULT_LIMIT = 100

# For page-based:
PAGE_PARAM = "page"

# JSON key path to the array of results (e.g. "data" or "results")
DATA_KEY = "data"


def get_nested(obj, key_path):
    """Get a nested value from a dict using dot notation."""
    keys = key_path.split(".")
    for k in keys:
        if isinstance(obj, dict) and k in obj:
            obj = obj[k]
        else:
            return None
    return obj


def call_request(path, extra_query=""):
    """Call request.py and return parsed JSON."""
    cmd = ["uv", "run", REQUEST_SCRIPT, "GET", path]
    if extra_query:
        cmd += ["--query", extra_query]

    result = subprocess.run(cmd, capture_output=True, text=True)
    if result.returncode != 0:
        print(result.stderr, file=sys.stderr)
        sys.exit(result.returncode)

    return json.loads(result.stdout)


def main():
    parser = argparse.ArgumentParser(
        description="Auto-paginating client for the <API Name> API."
    )
    parser.add_argument("path", help="API path to paginate (e.g. /v1/users)")
    parser.add_argument("--query", help="Additional query params", default="")
    parser.add_argument(
        "--max-pages", type=int, default=100, help="Max pages to fetch (default: 100)"
    )
    parser.add_argument(
        "--stream",
        action="store_true",
        help="Print each page's items as they arrive (JSONL)",
    )
    args = parser.parse_args()

    all_items = []

    if PAGINATION_STYLE == "cursor":
        cursor = None
        for page_num in range(args.max_pages):
            query = args.query
            if cursor:
                query = f"{query}&{CURSOR_PARAM}={cursor}" if query else f"{CURSOR_PARAM}={cursor}"
            resp = call_request(args.path, query)
            items = get_nested(resp, DATA_KEY) or []

            if args.stream:
                for item in items:
                    print(json.dumps(item))
            else:
                all_items.extend(items)

            cursor = get_nested(resp, CURSOR_KEY)
            if not cursor or not items:
                break

    elif PAGINATION_STYLE == "offset":
        offset = 0
        for page_num in range(args.max_pages):
            query = f"{LIMIT_PARAM}={DEFAULT_LIMIT}&{OFFSET_PARAM}={offset}"
            if args.query:
                query = f"{args.query}&{query}"
            resp = call_request(args.path, query)
            items = get_nested(resp, DATA_KEY) or []

            if args.stream:
                for item in items:
                    print(json.dumps(item))
            else:
                all_items.extend(items)

            if len(items) < DEFAULT_LIMIT:
                break
            offset += len(items)

    elif PAGINATION_STYLE == "page":
        for page_num in range(1, args.max_pages + 1):
            query = f"{PAGE_PARAM}={page_num}"
            if args.query:
                query = f"{args.query}&{query}"
            resp = call_request(args.path, query)
            items = get_nested(resp, DATA_KEY) or []

            if args.stream:
                for item in items:
                    print(json.dumps(item))
            else:
                all_items.extend(items)

            if not items:
                break

    if not args.stream:
        print(json.dumps(all_items, indent=2))


if __name__ == "__main__":
    main()
```

## Generated endpoints.md Template

```markdown
# Endpoint Reference

Full parameter details for all <API Name> endpoints.

## <Resource 1>

### GET /v1/<resource>

List all <resources>.

**Query parameters:**

| Param | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| limit | integer | No | 20 | Max items per page (max 100) |
| offset | integer | No | 0 | Pagination offset |
| ...   | ...  | ...      | ...     | ...         |

**Response:**

\`\`\`json
{
  "data": [...],
  "meta": { "total": 142, "next_cursor": "abc123" }
}
\`\`\`

### POST /v1/<resource>

Create a <resource>.

**Request body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| name  | string | Yes | ... |
| ...   | ...  | ...      | ...         |

**Response:** Returns the created <resource> object.

... (repeat for each endpoint)
```

## Generated error-codes.md Template

```markdown
# Error Codes

## Error Response Format

All errors return a JSON body with this structure:

\`\`\`json
{
  "<error-key>": {
    "code": "<error-code>",
    "message": "Human-readable description"
  }
}
\`\`\`

## Common Error Codes

| HTTP Status | Code | Meaning | Retryable |
|-------------|------|---------|-----------|
| 400 | bad_request | Malformed request | No |
| 401 | unauthorized | Invalid or missing API key | No |
| 403 | forbidden | Insufficient permissions | No |
| 404 | not_found | Resource doesn't exist | No |
| 409 | conflict | Resource already exists | No |
| 422 | unprocessable | Validation error | No — fix request |
| 429 | rate_limited | Too many requests | Yes — wait and retry |
| 500 | server_error | Internal server error | Yes — retry with backoff |
| 503 | unavailable | Service temporarily unavailable | Yes — retry with backoff |

## API-Specific Error Codes

| Code | Meaning | How to Fix |
|------|---------|------------|
| ... | ... | ... |
```

## Customization Notes

These templates are starting points. When generating the actual skill, adapt them:

- Remove sections that don't apply (e.g. no paginate.py if the API doesn't paginate)
- Add sections for API-specific concepts (e.g. webhooks, batch operations, file uploads)
- Adjust the auth mechanism in request.py to match the actual API
- Use the API's actual field names, not placeholders
- Match the API's URL structure and versioning scheme
