# Secrets and Authentication Guide

How the generated skill should handle API keys, tokens, and other credentials.

## Core Principle

**Never store secrets in the skill directory.** Not in SKILL.md, not in scripts, not in
config files, not in comments. Secrets are provided at runtime via environment variables.

## Environment Variable Naming

Use a consistent, predictable naming convention:

```
<API_NAME>_API_KEY        — for simple API key auth
<API_NAME>_ACCESS_TOKEN   — for OAuth2 access tokens
<API_NAME>_SECRET_KEY     — for secret keys (when there's also a publishable key)
<API_NAME>_CLIENT_ID      — for OAuth2 client credentials
<API_NAME>_CLIENT_SECRET  — for OAuth2 client credentials
```

Examples:
- `STRIPE_API_KEY`
- `OPENAI_API_KEY`
- `GITHUB_TOKEN`
- `SHOPIFY_ACCESS_TOKEN`
- `TWILIO_ACCOUNT_SID` + `TWILIO_AUTH_TOKEN`

Use the name that the API's own docs use if they suggest one. Otherwise, follow the pattern above.

## Auth Type Handling

### API Key in Header

The most common pattern. The generated `request.py` should:

```python
import os

api_key = os.environ.get("EXAMPLE_API_KEY")
if not api_key:
    print("Error: EXAMPLE_API_KEY environment variable is not set.", file=sys.stderr)
    print("Get your API key at: https://example.com/dashboard/api-keys", file=sys.stderr)
    sys.exit(2)

headers = {
    "Authorization": f"Bearer {api_key}",  # or whatever the API expects
    # Some APIs use custom headers like "X-API-Key"
}
```

### API Key in Query Parameter

Less common but some APIs require it:

```python
params = {
    "api_key": api_key,
    **user_params,
}
```

### OAuth2 (Client Credentials)

For machine-to-machine APIs:

```python
client_id = os.environ.get("EXAMPLE_CLIENT_ID")
client_secret = os.environ.get("EXAMPLE_CLIENT_SECRET")

# Token exchange
token_response = httpx.post(
    "https://api.example.com/oauth/token",
    data={
        "grant_type": "client_credentials",
        "client_id": client_id,
        "client_secret": client_secret,
    },
)
access_token = token_response.json()["access_token"]
```

The generated skill should document the full token exchange in its Setup section, and the
`request.py` script should handle token refresh if the API supports it.

### OAuth2 (Authorization Code)

For user-facing APIs, the skill can't fully automate the browser-based auth flow. Instead:

1. Document the OAuth2 flow in the skill's Setup section
2. Tell the user to obtain an access token manually (or via a separate tool)
3. Have the script read the token from `<API_NAME>_ACCESS_TOKEN`
4. If the API supports refresh tokens, accept `<API_NAME>_REFRESH_TOKEN` and handle
   automatic refresh in `request.py`

### Basic Auth

```python
import base64
username = os.environ.get("EXAMPLE_USERNAME")
password = os.environ.get("EXAMPLE_PASSWORD")
credentials = base64.b64encode(f"{username}:{password}".encode()).decode()
headers = {"Authorization": f"Basic {credentials}"}
```

### Multiple Credentials

Some APIs need multiple values (e.g. Twilio needs Account SID + Auth Token). Document
all required env vars in the Setup section and validate all of them at script startup.

## Setup Section Template

The generated skill's SKILL.md should include a Setup section like this:

```markdown
## Setup

### 1. Get your API key

Go to [Example Dashboard](https://example.com/settings/api-keys) and create a new API key.

### 2. Set the environment variable

```bash
export EXAMPLE_API_KEY="your-key-here"
```

For persistent configuration, add this to your shell profile (`~/.bashrc`, `~/.zshrc`)
or use a `.env` file with your agent's env loading mechanism.

### 3. Verify access

```bash
uv run scripts/request.py GET /v1/me
```

You should see your account details in the response.
```

Adapt this template to the specific API — include the exact dashboard URL where keys are
created, any scope or permission requirements, and a simple verification command.

## Security Notes to Include

The generated skill should include these notes in its Setup section or Gotchas:

- "Keep your API key secret. Do not commit it to version control."
- "Use environment variables or a secrets manager. Never paste keys into SKILL.md."
- If the API has both test and production keys: "Use the test/sandbox key during development.
  Set `EXAMPLE_API_KEY` to your test key (usually prefixed with `test_` or `sk_test_`)."
- If the API supports key rotation: "Rotate your API key periodically via the dashboard."
