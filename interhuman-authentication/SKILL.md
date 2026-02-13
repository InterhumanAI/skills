---
name: interhuman-authentication
description: Wrapper for Interhuman API POST /v0/auth endpoint. Generates short-lived bearer access tokens using API key credentials. Use when the user needs to authenticate before calling Interhuman API endpoints. Returns the exact JSON response from the API without modification.
---

# Interhuman Authentication

Wrapper for the Interhuman API authentication endpoint that generates short-lived bearer tokens for API access.

## When to Use

Use this skill when:
- You need to obtain an access token before calling Interhuman API endpoints
- The user has API key credentials (key_id and key_secret) from the Interhuman dashboard
- You need to authenticate for either upload or stream endpoints

This skill should be called first, before using `interhuman-post-processing` or `interhuman-stream` skills.

## Required Inputs

1. **key_id** (string): API key ID from the Interhuman dashboard
2. **key_secret** (string): API key secret from the Interhuman dashboard
3. **scopes** (array): List of requested scopes. At least one scope must be provided.
   - `interhumanai.upload` - For `/v0/upload/analyze` endpoint
   - `interhumanai.stream` - For `/v0/stream/analyze` endpoint

## API Call Instructions

### Endpoint Details

- **Base URL**: `https://api.interhuman.ai`
- **Endpoint**: `/v0/auth`
- **Method**: POST
- **Content-Type**: `application/json`
- **Authentication**: None required (uses API key credentials in request body)

### Request Format

Send a JSON request body with `key_id`, `key_secret`, and `scopes` array.

### Example: cURL

```bash
curl -X POST https://api.interhuman.ai/v0/auth \
  -H "Content-Type: application/json" \
  -d '{
    "key_id": "your_key_id",
    "key_secret": "your_key_secret",
    "scopes": ["interhumanai.upload"]
  }'
```

### Example: Python

```python
import requests

response = requests.post(
    "https://api.interhuman.ai/v0/auth",
    json={
        "key_id": "your_key_id",
        "key_secret": "your_key_secret",
        "scopes": ["interhumanai.upload"]
    }
)

# Return the raw JSON response
print(response.json())
```

### Example: JavaScript/Node.js

```javascript
const response = await fetch("https://api.interhuman.ai/v0/auth", {
  method: "POST",
  headers: {
    "Content-Type": "application/json"
  },
  body: JSON.stringify({
    key_id: "your_key_id",
    key_secret: "your_key_secret",
    scopes: ["interhumanai.upload"]
  })
});

const json = await response.json();
console.log(json);
```

## Response Format

The API returns a JSON object with:

- **access_token** (string): The generated access token (use as Bearer token)
- **token_type** (string): Token type, always `Bearer`
- **expires_in** (integer): Time in seconds until expiration (usually 900s or 15 minutes)
- **scope** (string): Space-separated list of granted scopes

### Example Response

```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "Bearer",
  "expires_in": 900,
  "scope": "interhumanai.upload"
}
```

## Error Responses

On error, the API returns JSON with:

- **detail** (string): Error message
- **status_code** (integer): HTTP status code
- **extra** (object): Additional error information

### Status Codes

- `200`: Success
- `400`: Bad request (invalid credentials or parameters)
- `401`: Unauthorized (invalid key_id or key_secret)
- `403`: Forbidden (credentials valid but lack required scope)
- `422`: Validation error (invalid request format)
- `500`: Internal server error

## Using the Access Token

After obtaining an access token, use it in the `Authorization` header for subsequent API calls:

```
Authorization: Bearer <access_token>
```

The token expires after the time specified in `expires_in` (typically 15 minutes). Generate a new token when it expires.

## Output Rules

**CRITICAL**: This skill is a strict wrapper. You MUST:

1. Return the exact JSON response from the API without any modification
2. Do NOT summarize, transform, or rename fields
3. Do NOT extract only the access_token field
4. Do NOT add commentary or interpretation
5. Preserve all fields exactly as received from the API

The response should be the raw JSON object returned by the API, passed through verbatim.
