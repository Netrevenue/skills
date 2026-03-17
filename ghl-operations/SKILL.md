---
name: ghl-operations
description: >
  Manage GoHighLevel (GHL) sub-accounts via API for Netrevenue clients. Use this skill whenever the user asks about
  GHL account operations including: adding, removing, or updating users in a GHL account; managing calendar assignments
  and placements; purchasing or assigning phone numbers; listing or searching users, calendars, or phone numbers;
  viewing pipeline data or opportunity details; managing custom fields; or auditing account configuration.
  Also use this skill when the user mentions "onboarding" or "offboarding" a rep (these are multi-step workflows
  that include GHL operations). Trigger on mentions of "GHL", "GoHighLevel", "HighLevel", "sub-account", "location",
  or any client account name when combined with admin/operational actions. If an action is NOT covered by the API
  endpoints in this skill, use browser automation as a fallback -- ask the user for exact frontend steps and offer
  to remember the steps for future use.
---

# GHL Operations Skill

Manage GoHighLevel sub-accounts via API for Netrevenue's client accounts. This skill covers user management, calendar management, phone number management, pipeline/opportunity reads, and account configuration.

## Important: Read Before Any Action

Before performing ANY GHL API action, you MUST:

1. **Identify the target GHL account.** Ask the user which client account this action targets if not obvious from context.
2. **Obtain the API token and locationId** for that account. Currently using Private Integration Tokens (PIT) stored per account. Ask the user for both the token and locationId, or retrieve them from the configured credential source.
3. **Obtain the companyId** if creating or searching users. Get this from GET /locations/{locationId} response field `companyId`.
4. **Confirm destructive actions.** Before deleting users, releasing numbers, or modifying calendars, confirm with the user: "I'm about to [action] on [account]. Proceed?"

## API Fundamentals

**Base URL:** `https://services.leadconnectorhq.com`

**Required headers on EVERY request:**
```
Authorization: Bearer {token}
Version: 2021-07-28
Content-Type: application/json
Accept: application/json
```

**Authentication:** All actions use Sub-Account level Private Integration Tokens (PIT). We do NOT use Agency-level tokens. Each client GHL account has its own PIT.

**Rate limits:** 100 requests per 10 seconds per app per location. 200,000 requests per day per app per location. If you receive a 429 response, wait 10 seconds and retry.

**Location ID:** All endpoints require a `locationId` parameter (as query param, path param, or in request body). The PIT token is scoped to a specific location, but you must still pass the locationId explicitly. The user will provide both the PIT token and locationId together.

**IMPORTANT API Quirks:**
- Users list: `GET /users/?locationId={id}` (query param)
- Users search: Requires both `companyId` and `locationId` (query params)
- Opportunities search: Uses `location_id` (snake_case), NOT `locationId` (camelCase)
- Phone numbers: Prefix is `/phone-system/numbers/location/{locationId}/...`
- User phone assignment: Via `lcPhone` field in user object (maps locationId to phone number)
- Delete user response: Field is `succeded` (GHL's typo, not `succeeded`)
- teamMembers `isZoomAdded`: String `"true"`/`"false"`, NOT boolean

**Error handling:** If an API call returns 401 Unauthorized, the token may be expired or invalid. Ask the user to verify the token. If 422 Unprocessable Entity, the request body is malformed -- check required fields. If 400 Bad Request, review the parameters against the reference docs.

## Action Categories

Read the appropriate reference file for detailed endpoint documentation before making API calls:

| Category | Reference File | Key Actions |
|---|---|---|
| User Management | `references/users.md` | Add, remove, update, list, search users |
| Calendar Management | `references/calendars.md` | Create, update, delete calendars; assign/remove reps |
| Phone Numbers | `references/phone-numbers.md` | Search available, purchase, list active, assign to users |
| Pipelines & Opportunities | `references/pipelines.md` | List pipelines, search/read opportunities, audit pipeline health |
| Account Configuration | `references/account-config.md` | Location settings, custom fields |

## Phone Number Purchasing Logic

When purchasing a phone number for a rep, follow this specific sequence:

1. **Determine the area code.** The area code should match the region where the sub-account is based. Get the sub-account's address via GET /locations/{locationId} and extract the state/city/country.
2. **Search available numbers** using GET /phone-system/numbers/location/{locationId}/available with `type=local`, `countryCode` (e.g., `US` or `CA`), and `areaCode` (e.g., `415`).
3. **If no numbers available** for the primary area code, try neighboring area codes for the same region. For example, if 415 has no availability, try 628 (also San Francisco), 510 (East Bay), or 650 (Peninsula).
4. **Purchase the number** using POST /phone-system/numbers/location/{locationId}/purchase with the phone number and type.
5. **Assign to the user** by updating the user's `lcPhone` field via PUT /users/{userId}. Set `lcPhone[locationId] = "+14155551234"`.

Read `references/phone-numbers.md` for exact endpoint details and full workflow.

## Browser Fallback for Unsupported Actions

If a user requests an action that is NOT available via the GHL API (examples: enabling/disabling workflows, creating/editing pipeline stages, or any action not documented in the reference files), proceed as follows:

1. Inform the user: "This action is not available via the GHL API. I can perform it through the browser if you walk me through the exact steps."
2. Ask the user for the step-by-step frontend navigation: which URL to visit, what to click, what values to enter, and how to confirm the action.
3. Execute via browser automation following the user's instructions.
4. After completing the action, ask: "Should I remember these browser steps for future use? If so, I'll save them so I can perform this action again without needing a walkthrough."
5. If the user says yes, save the steps as a note associated with that action type so you can reference them in future conversations.

## Multi-Step Workflows

This skill provides the atomic GHL actions used in larger workflows. When a user asks to "onboard a rep" or "offboard a rep," the agent should compose these atomic actions with actions from other platforms (Slack, Salesvue). The workflow skill defines the sequence; this skill provides the GHL steps.

**Typical GHL steps in rep onboarding:**
1. Create user (references/users.md)
2. Assign to calendar(s) (references/calendars.md)
3. Purchase and assign phone number (references/phone-numbers.md)

**Typical GHL steps in rep offboarding:**
1. Remove from calendar(s) (references/calendars.md)
2. Unassign phone number (references/phone-numbers.md)
3. Delete or deactivate user (references/users.md)
