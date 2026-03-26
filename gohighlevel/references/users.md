# GHL Users API Reference

All endpoints use base URL `https://services.leadconnectorhq.com` with Sub-Account PIT auth.

Required headers on every request:
```
Authorization: Bearer {pit_token}
Version: 2021-07-28
Content-Type: application/json
Accept: application/json
```

---

## List Users by Location

Retrieve all users in a GHL sub-account.

```
GET /users/?locationId={locationId}
```

**Query Parameters:**
- `locationId` (required): The GHL sub-account/location ID

**Response (200):**
```json
{
  "users": [
    {
      "id": "WYrr6bdoI44Wgp46nHtb",
      "name": "John Dear",
      "firstName": "John",
      "lastName": "Dear",
      "email": "john+delete@netrevenue.io",
      "deleted": false,
      "roles": {
        "type": "account",
        "role": "user",
        "locationIds": ["ox6oaZ5j2iG0qxjc9Pfc"],
        "restrictSubAccount": false
      },
      "scopes": ["contacts.write", "opportunities.write"],
      "scopesAssignedToOnly": [],
      "lcPhone": {
        "ox6oaZ5j2iG0qxjc9Pfc": "+14387032919"
      }
    }
  ],
  "traceId": "bb1d3bad-bf73-42b2-917f-e2bbafe86528"
}
```

**Key field notes:**
- `roles` is a nested object containing `type`, `role`, `locationIds`, and `restrictSubAccount`. This differs from the GET single user and search endpoints.
- `lcPhone` maps locationId to the user's assigned phone number for that location. Empty object `{}` if no phone assigned.
- `scopes` lists the user's API-level permission scopes. `scopesAssignedToOnly` lists scopes that restrict the user to only assigned data.
- The list endpoint does NOT return the expanded `permissions` object. Use GET single user or search for that.
- Agency-type users (type `"agency"`) may appear if they have access to the location. They have broader `locationIds` arrays.

---

## Get User

Retrieve a single user by ID.

```
GET /users/{userId}
```

**Path Parameters:**
- `userId` (required): The GHL user ID

**Response (200):**
```json
{
  "id": "WYrr6bdoI44Wgp46nHtb",
  "name": "John Dear",
  "firstName": "John",
  "lastName": "Dear",
  "email": "john+delete@netrevenue.io",
  "permissions": {
    "campaignsEnabled": true,
    "campaignsReadOnly": false,
    "workflowsEnabled": true,
    "workflowsReadOnly": false,
    "contactsEnabled": true,
    "triggersEnabled": true,
    "opportunitiesEnabled": true,
    "settingsEnabled": true,
    "tagsEnabled": true,
    "leadValueEnabled": true,
    "dashboardStatsEnabled": true,
    "bulkRequestsEnabled": true,
    "opportunitiesBulkActionsEnabled": true,
    "appointmentsEnabled": true,
    "reviewsEnabled": true,
    "onlineListingsEnabled": true,
    "phoneCallEnabled": true,
    "conversationsEnabled": true,
    "assignedDataOnly": false,
    "funnelsEnabled": true,
    "websitesEnabled": true,
    "marketingEnabled": true,
    "adwordsReportingEnabled": true,
    "facebookAdsReportingEnabled": true,
    "attributionsReportingEnabled": true,
    "membershipEnabled": true,
    "botService": false,
    "agentReportingEnabled": true,
    "socialPlanner": true,
    "bloggingEnabled": true,
    "invoiceEnabled": true,
    "affiliateManagerEnabled": true,
    "contentAiEnabled": true,
    "refundsEnabled": true,
    "recordPaymentEnabled": true,
    "cancelSubscriptionEnabled": true,
    "paymentsEnabled": true,
    "communitiesEnabled": true,
    "exportPaymentsEnabled": true,
    "certificatesEnabled": true,
    "mediaStorageEnabled": true,
    "reportingEnabled": true,
    "adPublishingEnabled": true,
    "adPublishingReadOnly": true,
    "wordpressEnabled": true,
    "customMenuLinkReadOnly": true,
    "customMenuLinkWrite": false,
    "gokollabEnabled": true
  },
  "deleted": false,
  "roles": {
    "type": "account",
    "role": "user",
    "locationIds": ["ox6oaZ5j2iG0qxjc9Pfc"],
    "restrictSubAccount": false
  },
  "scopes": [],
  "scopesAssignedToOnly": [],
  "totpEnabled": false,
  "lcPhone": {
    "ox6oaZ5j2iG0qxjc9Pfc": "+14387032919"
  },
  "dateAdded": "2026-02-23T21:52:27.279Z",
  "dateUpdated": "2026-02-23T22:03:59.020Z",
  "platformLanguage": "en_US",
  "traceId": "300766a2-d0c1-4dd2-b338-93184600e47e"
}
```

**Note:** The GET single user response includes the full `permissions` object and `dateAdded`/`dateUpdated` timestamps, which are not present in the list endpoint.

---

## Search Users

Search users by name or email. Returns the same shape as GET single user (with permissions).

```
GET /users/search
```

**Query Parameters:**
- `companyId` (required): The agency company ID. Get this from GET /locations/{locationId} response field `companyId`.
- `locationId` (required): Sub-account ID
- `query` (optional): Search term (matches against name, email)
- `limit` (optional): Max results (default 25)
- `skip` (optional): Offset for pagination

**Response (200):**
```json
{
  "users": [
    {
      "id": "WYrr6bdoI44Wgp46nHtb",
      "deleted": false,
      "email": "john+delete@netrevenue.io",
      "firstName": "John",
      "lastName": "Dear",
      "name": "John Dear",
      "permissions": { ... },
      "roles": {
        "type": "account",
        "role": "user",
        "locationIds": ["ox6oaZ5j2iG0qxjc9Pfc"],
        "restrictSubAccount": false
      },
      "dateAdded": "2026-02-23T21:52:27.279Z",
      "dateUpdated": "2026-02-23T22:03:59.020Z",
      "platformLanguage": "en_US"
    }
  ],
  "count": 1,
  "traceId": "a02d756b-7def-41f2-8de7-ff62c7c6473c"
}
```

---

## Create User

Add a new user to a GHL sub-account.

```
POST /users/
```

**Request Body:**
```json
{
  "companyId": "VBW5vxWDhCP6mOEKBupq",
  "firstName": "Jane",
  "lastName": "Doe",
  "email": "jane.doe@example.com",
  "password": "",
  "type": "account",
  "role": "user",
  "locationIds": ["ox6oaZ5j2iG0qxjc9Pfc"],
  "permissions": {
    "contactsEnabled": true,
    "opportunitiesEnabled": true,
    "dashboardStatsEnabled": true,
    "appointmentsEnabled": true,
    "conversationsEnabled": true
  }
}
```

**Field Notes:**
- `companyId` (required): The agency company ID. Get from GET /locations/{locationId}.
- `firstName`, `lastName`, `email`: Required.
- `password`: If empty string, GHL sends a setup email. If provided, must meet GHL password requirements (must include uppercase, lowercase, numbers, and special chars).
- `type`: Use `"account"` for sub-account level users. Do NOT use `"agency"`.
- `role`: Values are `"admin"` or `"user"`. Admins have full access; users are restricted by permissions.
- `locationIds`: Array of location IDs the user should have access to.
- `permissions`: Only the fields you send are used as inputs. GHL auto-expands to a full permissions object on the response, setting many fields to `true` by default for `"user"` role.

**IMPORTANT:** When creating a user with role `"user"`, GHL fills in many permission defaults as `true`. If you want a restricted rep, you must explicitly set unwanted permissions to `false` in a follow-up PUT call after verifying the created user's permissions.

**Response (201):** Returns the full user object (same shape as GET single user), including the auto-expanded permissions and assigned scopes.

**Error (400) - User Already Exists:**
```json
{
  "message": "A user with this email already exists"
}
```

When you receive this error, the user exists in the agency but may not have access to the target sub-account. Follow the "Handling Duplicate Users" workflow below to add them to the sub-account.

---

## Update User

Modify an existing user's profile, role, or permissions.

```
PUT /users/{userId}
```

**Path Parameters:**
- `userId` (required): The GHL user ID

**Request Body:** Include only the fields you want to update:
```json
{
  "firstName": "Jane",
  "permissions": {
    "workflowsEnabled": true
  }
}
```

**IMPORTANT:** When updating permissions, GET the user first to see the current full permissions object, modify the fields you need, then PUT the entire permissions object back. Sending a partial permissions object may reset unlisted fields to defaults.

**Response (200):** Returns the updated user object.

---

## Handling Duplicate Users

When creating a user, GHL returns a "user already exists" error if the email is already registered in the agency. This does NOT mean the user has access to your target sub-account—it only means they exist somewhere in the agency. This case cannot be solved through the api with your access, it requires agency access. Inform the user that an agency admin will have to go into the agency team settings to assign the location id to the user.

---

## Delete User

Remove a user from a GHL sub-account.

```
DELETE /users/{userId}
```

**Path Parameters:**
- `userId` (required): The GHL user ID

**CAUTION:** This permanently removes the user. There is no undo. Before deleting:
1. Confirm with the requesting user.
2. Verify the user ID is correct (GET the user first to confirm name/email).
3. Remove the user from calendars first (to avoid orphaned calendar assignments).

**Response (200):**
```json
{
  "succeded": true,
  "message": "Queued deleting user with e-mail jane.doe@example.com and name Jane Doe. Will take effect in a few minutes.",
  "traceId": "622c026e-54ec-4f69-872c-4b13cd5aabc5"
}
```

**Note:** The field is `succeded` (GHL's typo, not `succeeded`). Deletion is queued and takes effect after a few minutes, not immediately.
