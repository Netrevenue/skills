# GHL Authentication API Reference

All GHL API actions require credentials (locationId + PIT token). This skill uses an n8n webhook to automatically resolve credentials from account names.

**Base URL:** n8n webhook endpoint (configured via environment variable)

**Required environment variables:**
```
GHL_AUTH_WEBHOOK_URL - The n8n webhook URL
GHL_AUTH_API_KEY - API key for webhook authentication
```

---

## Lookup Credentials by Account Name

Resolve GHL credentials from an account name using LLM-powered fuzzy matching.

```
POST $GHL_AUTH_WEBHOOK_URL
```

**Headers:**
```
Content-Type: application/json
X-API-Key: {GHL_AUTH_API_KEY}
```

**Request Body:**
```json
{
  "accountName": "Testing Account"
}
```

**Field Notes:**
- `accountName` (required): The account name to search for. Supports fuzzy matching (acronyms, aliases, typos, partial matches).
- API key can be sent via `X-API-Key` header.

**Response (200):**
```json
{
  "success": true,
  "searchTerm": "Testing Account",
  "match": {
    "name": "Testing Account",
    "locationId": "ox6oaZ5j2iG0qxjc9Pfc",
    "token": "pit-1b6efa02-331d-4a0b-8523-bcdfdf7ba039ce",
    "confidence": 1.0,
    "matchReason": "exact match",
    "agencyId": "VBW5vxWDhCP6mOEKBupq",
    "status": "active",
    "aliases": "test,testing,ta"
  },
  "llmAnalysis": {
    "matchFound": true,
    "bestMatch": {...},
    "alternativeMatches": []
  }
}
```

**Response (200) - No Match:**
```json
{
  "success": false,
  "error": "No account found matching 'xyz'",
  "searchTerm": "xyz",
  "suggestions": ["Testing Account", "Acme Corp", "Self Storage Income"]
}
```

**Response (200) - Ambiguous:**
```json
{
  "success": false,
  "error": "Multiple accounts match. Please clarify.",
  "searchTerm": "ac",
  "matches": [
    {
      "name": "Acme Corp",
      "confidence": "75%",
      "reason": "alias match"
    },
    {
      "name": "ABC Industries",
      "confidence": "70%",
      "reason": "acronym"
    }
  ]
}
```

**Response (401) - Unauthorized:**
```json
{
  "success": false,
  "error": "Unauthorized: Invalid or missing API key",
  "message": "Please provide a valid API key in the request body or X-API-Key header"
}
```

## Credential Lookup Workflow

When the user mentions an account name in a request, follow this workflow:

```
Step 1: Extract the account name from user request
  User: "List users in the Testing account"
  Extract: "Testing"

Step 2: Call the credential lookup webhook
  POST $GHL_AUTH_WEBHOOK_URL
  Headers: X-API-Key: $GHL_AUTH_API_KEY
  Body: {"accountName": "Testing"}

Step 3: Handle the response
  If success=true:
    → Use match.locationId and match.token for GHL API calls

  If success=false (no match):
    → Show user the suggestions
    → Ask: "Did you mean: Testing Account, Acme Corp, or ABC Corp?"

  If success=false (ambiguous):
    → Show user the matches with confidence scores
    → Ask: "Which account: Acme Corp (75%) or ABC Corp (70%)?"

  If 401 Unauthorized:
    → Check GHL_AUTH_API_KEY environment variable is set correctly
    → Fall back to asking user for credentials manually

Step 4: Proceed with GHL API call using resolved credentials
```

---

## Manual Override

If the user explicitly provides credentials, bypass the webhook and use them directly:

**Examples:**
- "Use locationId abc123 and token pit-xyz for this request"
- "Here's the token: pit-abc123..."

In these cases, do NOT call the webhook. Use the provided credentials.

---

## Example Usage

**Simple lookup:**
```bash
curl -X POST $GHL_AUTH_WEBHOOK_URL \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $GHL_AUTH_API_KEY" \
  -d '{"accountName": "test"}'

# Returns credentials for "Testing Account"
```

**Acronym lookup:**
```bash
curl -X POST $GHL_AUTH_WEBHOOK_URL \
  -H "X-API-Key: $GHL_AUTH_API_KEY" \
  -d '{"accountName": "SSI"}'

# Returns credentials for "Self Storage Income" via acronym match
```

**No match:**
```bash
curl -X POST $GHL_AUTH_WEBHOOK_URL \
  -H "X-API-Key: $GHL_AUTH_API_KEY" \
  -d '{"accountName": "nonexistent"}'

# Returns 200 with suggestions
```
