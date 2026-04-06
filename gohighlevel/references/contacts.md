# GHL Contacts API Reference

All endpoints use base URL `https://services.leadconnectorhq.com` with Sub-Account PIT auth.

Required headers on every request:
```
Authorization: Bearer {pit_token}
Version: 2021-07-28
Content-Type: application/json
Accept: application/json
```

**Scope note:** Full CRUD on contacts, plus advanced search with filtering, sorting, and pagination. Tags set via create/update/upsert will **overwrite** all existing tags on the contact — use the dedicated Add Tag / Remove Tag endpoints in `account-config.md` for incremental tag changes.

---

## Get Contact

Get a single contact by ID.

```
GET /contacts/{contactId}
```

**Path Parameters:**
- `contactId` (required): Contact ID

**Response (200):**
```json
{
  "contact": {
    "id": "abc123DEFghiJKL456",
    "firstName": "Jane",
    "firstNameLowerCase": "jane",
    "lastName": "Smith",
    "lastNameLowerCase": "smith",
    "fullNameLowerCase": "jane smith",
    "email": "jane.smith@example.com",
    "emailLowerCase": "jane.smith@example.com",
    "phone": "+14155551234",
    "additionalPhones": [],
    "additionalEmails": [],
    "type": "lead",
    "locationId": "ve9EPM428h8vShlRW1KT",
    "assignedTo": "userId123abc",
    "tags": ["prospect"],
    "country": "US",
    "timezone": "America/New_York",
    "dnd": false,
    "customFields": [],
    "followers": [],
    "dateAdded": "2025-11-24T16:58:56.678Z",
    "dateUpdated": "2026-03-06T20:27:20.348Z",
    "attributionSource": {
      "sessionSource": "CRM UI",
      "medium": "manual",
      "mediumId": null
    },
    "createdBy": {
      "source": "WEB_USER",
      "channel": "APP",
      "sourceId": "userId123abc",
      "sourceName": "Jane Smith",
      "timestamp": "2025-11-24T16:58:56.677Z"
    }
  },
  "traceId": "aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee"
}
```

**Key field notes:**
- `createdBy` includes the source (WEB_USER, INTEGRATION, etc.), channel, and who created the record.
- `customFields` is an array of `{id, value}` objects. Resolve IDs to names via `GET /locations/{locationId}/customFields` (see `account-config.md`).
- `assignedTo` is a userId. Look up the user name via `GET /users/{userId}` if needed.

---

## Create Contact

Create a new contact in a sub-account.

```
POST /contacts/
```

**Request Body:**
```json
{
  "firstName": "Rosan",
  "lastName": "Deo",
  "name": "Rosan Deo",
  "email": "rosan@deos.com",
  "locationId": "ve9EPM428h8vShlRW1KT",
  "gender": "male",
  "phone": "+1 888-888-8888",
  "address1": "3535 1st St N",
  "city": "Dolomite",
  "state": "AL",
  "postalCode": "35061",
  "website": "https://www.tesla.com",
  "timezone": "America/Chihuahua",
  "dnd": true,
  "dndSettings": {},
  "inboundDndSettings": {},
  "tags": ["prospect", "inbound"],
  "customFields": [
    { "id": "customFieldId123", "field_value": "some value" }
  ],
  "source": "public api",
  "dateOfBirth": "1990-09-25",
  "country": "US",
  "companyName": "DGS VolMAX",
  "assignedTo": "y0BeYjuRIlDwsDcOHOJo"
}
```

**Required fields:** `locationId`

**Field notes:**
| Field | Type | Notes |
|---|---|---|
| `firstName` | string, nullable | |
| `lastName` | string, nullable | |
| `name` | string, nullable | Full name. If provided alongside firstName/lastName, GHL uses this for display. |
| `email` | string, nullable | Used as a duplicate-matching key by default. |
| `locationId` | string, **required** | Sub-account ID. |
| `gender` | string | e.g., `"male"`, `"female"` |
| `phone` | string, nullable | Include country code. e.g., `"+1 888-888-8888"` |
| `address1` | string, nullable | Street address |
| `city` | string, nullable | |
| `state` | string, nullable | |
| `postalCode` | string | |
| `website` | string, nullable | |
| `timezone` | string, nullable | IANA timezone. e.g., `"America/Chihuahua"` |
| `dnd` | boolean | Do Not Disturb master toggle |
| `dndSettings` | object | Per-channel DND (email, SMS, calls, etc.) |
| `inboundDndSettings` | object | Inbound-specific DND settings |
| `tags` | string[] | **Overwrites** all existing tags. Use Add/Remove Tag endpoints for incremental changes. |
| `customFields` | object[] | Array of `{id, field_value}` pairs. Get field IDs from `GET /locations/{locationId}/customFields`. |
| `source` | string | e.g., `"public api"`, `"website"`, `"referral"` |
| `dateOfBirth` | string, nullable | Formats: `YYYY-MM-DD`, `MM-DD-YYYY`, `YYYY/MM/DD`, `MM/DD/YYYY`, `YYYY.MM.DD`, `MM.DD.YYYY` |
| `country` | string | e.g., `"US"`, `"CA"` |
| `companyName` | string, nullable | |
| `assignedTo` | string | User ID to assign the contact to. |

**Response (200):**
```json
{
  "contact": {
    "id": "newContact789xyz",
    "dateAdded": "2026-04-06T03:54:28.835Z",
    "dateUpdated": "2026-04-06T03:54:28.835Z",
    "deleted": false,
    "tags": ["prospect", "inbound"],
    "type": "lead",
    "customFields": [],
    "locationId": "ve9EPM428h8vShlRW1KT",
    "firstName": "Rosan",
    "firstNameLowerCase": "rosan",
    "fullNameLowerCase": "rosan deo",
    "lastName": "Deo",
    "lastNameLowerCase": "deo",
    "email": "rosan@deos.com",
    "emailLowerCase": "rosan@deos.com",
    "phone": "+1 888-888-8888",
    "country": "US",
    "source": "public api",
    "createdBy": {
      "source": "INTEGRATION",
      "channel": "OAUTH",
      "sourceId": "integrationId123",
      "timestamp": "2026-04-06T03:54:28.835Z"
    }
  },
  "traceId": "aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee"
}
```

---

## Update Contact

Update an existing contact. Supports partial updates — only send the fields you want to change.

```
PUT /contacts/{contactId}
```

**Path Parameters:**
- `contactId` (required): Contact ID

**Request Body:** Same fields as Create (all optional except `contactId` in path). Only include fields you want to change.

```json
{
  "firstName": "UpdatedName",
  "city": "Toronto",
  "tags": ["prospect", "updated"]
}
```

**Response (200):**
```json
{
  "succeded": true,
  "contact": {
    "id": "abc123DEFghiJKL456",
    "dateAdded": "2026-04-06T03:54:28.835Z",
    "dateUpdated": "2026-04-06T03:55:02.586Z",
    "tags": ["prospect", "updated"],
    "type": "lead",
    "locationId": "ve9EPM428h8vShlRW1KT",
    "firstName": "UpdatedName",
    "firstNameLowerCase": "updatedname",
    "fullNameLowerCase": "updatedname deo",
    "lastName": "Deo",
    "lastNameLowerCase": "deo",
    "email": "rosan@deos.com",
    "emailLowerCase": "rosan@deos.com",
    "phone": "+1 888-888-8888",
    "country": "US",
    "city": "Toronto",
    "source": "public api",
    "followers": [],
    "customFields": [],
    "additionalEmails": [],
    "additionalPhones": []
  },
  "traceId": "aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee"
}
```

**IMPORTANT quirks:**
- Response field is `succeded` (GHL typo, not `succeeded`). Same typo as DELETE user.
- `tags` in the request body **overwrites** all tags. To add/remove individual tags, use the dedicated tag endpoints in `account-config.md`.

---

## Upsert Contact

Create a new contact or update an existing one. GHL matches on the location's configured unique identifiers (typically `email` and/or `phone`). If a match is found, updates the existing contact; otherwise creates a new one.

```
POST /contacts/upsert
```

**Request Body:** Same fields as Create, plus one additional field:

| Field | Type | Notes |
|---|---|---|
| `createNewIfDuplicateAllowed` | boolean | Default: `false`. When `true` AND the location allows duplicates, creates a new contact without checking for existing matches. When `true` but duplicates are NOT allowed, this field is ignored and normal upsert behavior applies. When `false` or omitted, normal upsert applies regardless of location settings. |

```json
{
  "firstName": "Rosan",
  "lastName": "Deo",
  "email": "rosan@deos.com",
  "locationId": "ve9EPM428h8vShlRW1KT",
  "phone": "+1 888-888-8888",
  "tags": ["prospect"],
  "source": "public api"
}
```

**Response (200) — Updated existing contact:**
```json
{
  "new": false,
  "succeded": true,
  "contact": {
    "id": "abc123DEFghiJKL456",
    "dateAdded": "2026-04-06T03:54:28.835Z",
    "dateUpdated": "2026-04-06T03:55:04.906Z",
    "firstName": "Rosan",
    "firstNameLowerCase": "rosan",
    "lastName": "Deo",
    "lastNameLowerCase": "deo",
    "email": "rosan@deos.com",
    "emailLowerCase": "rosan@deos.com",
    "phone": "+1 888-888-8888",
    "tags": ["prospect"],
    "source": "public api",
    "locationId": "ve9EPM428h8vShlRW1KT",
    "type": "lead",
    "customFields": [],
    "followers": [],
    "additionalEmails": [],
    "additionalPhones": []
  },
  "traceId": "aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee"
}
```

**Key field notes:**
- `new`: `true` if a new contact was created, `false` if an existing contact was updated.
- `succeded`: GHL typo (not `succeeded`).
- Matching is based on the location's `contactUniqueIdentifiers` setting (usually email and/or phone). Check via `GET /locations/{locationId}` → `settings.contactUniqueIdentifiers`.

---

## Delete Contact

Permanently delete a contact.

```
DELETE /contacts/{contactId}
```

**Path Parameters:**
- `contactId` (required): Contact ID

**CAUTION:** Permanent deletion. Confirm with user first.

**Response (200):**
```json
{
  "succeded": true,
  "traceId": "aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee"
}
```

---

## Search Contacts

Advanced contact search with filtering, sorting, and pagination.

```
POST /contacts/search
```

**Request Body:**
```json
{
  "locationId": "ve9EPM428h8vShlRW1KT",
  "page": 1,
  "pageLimit": 20,
  "filters": [
    {
      "field": "firstNameLowerCase",
      "operator": "eq",
      "value": "john"
    }
  ],
  "sort": [
    {
      "field": "dateAdded",
      "direction": "desc"
    }
  ],
  "query": "search term"
}
```

**Parameters:**

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `locationId` | string | **Yes** | Sub-account ID |
| `page` | number | No | Page number (standard pagination). Do NOT use with `searchAfter`. |
| `pageLimit` | number | **Yes** | Results per page (max 500) |
| `searchAfter` | array | No | Cursor-based pagination value from previous response. Required for results beyond 10,000 records. Do NOT use with `page`. |
| `filters` | array | No | Array of filter objects or nested filter groups. Default logical grouping is `AND`. |
| `sort` | array | No | Array of sort objects. Max 2 sort fields. |
| `query` | string | No | Free-text search across configured searchable fields. |

### Pagination

Two modes — choose one, do not mix:

**Standard pagination** (`page` + `pageLimit`):
- Max 10,000 total records accessible
- Max 500 per page
- Example: `page=20, pageLimit=500` retrieves the last page of 10,000 records

**Cursor-based pagination** (`searchAfter` + `pageLimit`):
- Required for datasets exceeding 10,000 records
- Use the `searchAfter` value from each contact in the response as the cursor for the next page
- Do NOT include `page` when using `searchAfter`

**Response (200):**
```json
{
  "contacts": [
    {
      "id": "abc123DEFghiJKL456",
      "firstName": "John",
      "firstNameLowerCase": "john",
      "lastName": "Doe",
      "lastNameLowerCase": "doe",
      "contactName": "john doe",
      "email": "john.doe@example.com",
      "phone": "+14155551234",
      "additionalEmails": [],
      "additionalPhones": [],
      "type": "lead",
      "locationId": "ve9EPM428h8vShlRW1KT",
      "assignedTo": "userId123abc",
      "tags": ["prospect"],
      "customFields": [],
      "dnd": false,
      "dndSettings": {},
      "inboundDndSettings": {},
      "country": "US",
      "city": "New York",
      "state": "NY",
      "postalCode": "10001",
      "website": null,
      "address": "123 Main St",
      "businessName": null,
      "companyName": "Acme Corp",
      "dateOfBirth": null,
      "timezone": "America/New_York",
      "source": "website",
      "dateAdded": "2025-11-24T16:58:56.678Z",
      "dateUpdated": "2026-03-06T20:27:20.348Z",
      "followers": [],
      "validEmail": null,
      "phoneLabel": null,
      "businessId": null,
      "opportunities": [
        {
          "pipelineId": "pipeline123abc",
          "id": "opp456def",
          "monetaryValue": 5000,
          "pipelineStageId": "stage789ghi",
          "status": "open"
        }
      ],
      "searchAfter": [1764003536678, "abc123DEFghiJKL456"],
      "attributionSource": {
        "sessionSource": "CRM UI",
        "medium": "manual"
      },
      "lastAttributionSource": {}
    }
  ],
  "total": 42,
  "traceId": "aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee"
}
```

**Key differences from GET single contact:**
- Search response includes `opportunities` array (summary of associated deals) — single GET does not.
- Search returns `contactName` (display name) and `searchAfter` (pagination cursor).
- Search response uses `total` at root level (not `meta` like opportunities search).

### Filters

Filters use a `{field, operator, value}` structure. Combine filters with logical groups:

```json
{
  "filters": [
    {
      "field": "dnd",
      "operator": "eq",
      "value": true
    },
    {
      "group": "OR",
      "filters": [
        {
          "field": "firstNameLowerCase",
          "operator": "eq",
          "value": "alex"
        },
        {
          "field": "lastNameLowerCase",
          "operator": "eq",
          "value": "peter"
        }
      ]
    }
  ]
}
```

By default, top-level filters are joined with `AND`. Use `"group": "OR"` (or `"AND"`) to create nested logical groups.

**Supported operators:**

| Operator | Description | Value Type | Char Limit |
|---|---|---|---|
| `eq` | Equals | Number, String, Boolean | 75 |
| `not_eq` | Not equals | Number, String, Boolean | 75 |
| `contains` | Contains substring (no special chars) | String | 75 |
| `not_contains` | Does not contain (no special chars) | String | 75 |
| `exists` | Has a value | No value needed | — |
| `not_exists` | Has no value | No value needed | — |
| `range` | Between two values | `{"low": x, "high": y}` | 75 |

**Supported filter fields:**

| Display Name | Field Name | Operators |
|---|---|---|
| **Contact Information** | | |
| Contact Id | `id` | eq, not_eq |
| Contact Name | `contactName` | eq, not_eq, exists, not_exists |
| First Name | `firstNameLowerCase` | eq, not_eq, contains, not_contains, exists, not_exists |
| Last Name | `lastNameLowerCase` | eq, not_eq, contains, not_contains, exists, not_exists |
| Email | `email` | eq, not_eq, exists, not_exists (no contains/not_contains) |
| Phone | `phone` | eq, not_eq, contains, not_contains, exists, not_exists |
| Address | `address` | eq, not_eq, contains, not_contains, exists, not_exists |
| City | `city` | eq, not_eq, contains, not_contains, exists, not_exists |
| State | `state` | eq, not_eq, contains, not_contains, exists, not_exists |
| Country | `country` | eq, not_eq, exists, not_exists |
| Postal Code | `postalCode` | eq, not_eq, contains, not_contains, exists, not_exists |
| Business Name | `businessName` | eq, not_eq, contains, not_contains, exists, not_exists |
| Company Name | `companyName` | eq, not_eq, contains, not_contains, exists, not_exists |
| Website | `website` | eq, not_eq, exists, not_exists (no contains/not_contains) |
| Source | `source` | eq, not_eq, contains, not_contains, exists, not_exists |
| Assigned To | `assignedTo` | eq, not_eq, exists, not_exists |
| Tags | `tags` | eq, not_eq, contains, not_contains, exists, not_exists |
| Timezone | `timezone` | eq, not_eq, exists, not_exists |
| Contact Type | `type` | eq, not_eq, exists, not_exists |
| DND | `dnd` | eq, not_eq, exists, not_exists |
| Date of Birth | `dateOfBirth` | range, exists, not_exists |
| Created At | `dateAdded` | range, exists, not_exists |
| Updated At | `dateUpdated` | range, exists, not_exists |
| Valid Email | `validEmail` | eq, not_eq, exists, not_exists |
| Valid WhatsApp | `isValidWhatsapp` | eq, not_eq, exists, not_exists |
| Followers | `followers` | eq, not_eq, exists, not_exists |
| **Contact Activity** | | |
| Last Appointment | `lastAppointment` | range, exists, not_exists |
| Active Workflows | `activeWorkflows` | eq, not_eq, exists, not_exists |
| Finished Workflows | `finishedWorkflows` | eq, not_eq, exists, not_exists |
| Last Email Clicked | `lastEmailClickedDate` | range, exists, not_exists |
| Last Email Opened | `lastEmailOpenedDate` | range, exists, not_exists |
| **Opportunity Information** (nested filters) | | |
| Pipeline | `opportunities` → `pipelineId` | eq, not_eq, exists, not_exists |
| Pipeline Stage | `opportunities` → `pipelineStageId` | eq, not_eq, exists, not_exists |
| Pipeline Status | `opportunities` → `status` | eq, not_eq, exists, not_exists |
| **Custom Fields** | | |
| Text, Large Text, Single Options, Radio, Phone | `customFields.{fieldId}` | eq, not_eq, contains, not_contains, exists, not_exists |
| Checkbox, Multiple Options | `customFields.{fieldId}` | eq, not_eq, exists, not_exists |
| Numerical, Monetary | `customFields.{fieldId}` | range, exists, not_exists, eq, not_eq |
| Date | `customFields.{fieldId}` | range, exists, not_exists |
| Textbox List | `customFields.{fieldId}.{optionId}` | eq, not_eq, contains, not_contains, exists, not_exists |

**Nested opportunity filter example:**
```json
{
  "field": "opportunities",
  "operator": "nested",
  "filters": [
    {
      "field": "pipelineId",
      "operator": "eq",
      "value": "pipeline123abc"
    },
    {
      "field": "status",
      "operator": "eq",
      "value": "open"
    }
  ]
}
```

### Sort

Max 2 sort fields. Direction is `"asc"` or `"desc"`.

**Supported sort fields:**

| Field Name | Description |
|---|---|
| `firstNameLowerCase` | First name (alphabetical) |
| `lastNameLowerCase` | Last name (alphabetical) |
| `businessName` | Business name |
| `dateAdded` | Date created |
| `dateUpdated` | Date last updated |
| `email` | Email address |
| `dnd` | DND status |
| `source` | Contact source |

**Example with multiple sorts:**
```json
{
  "sort": [
    { "field": "dateAdded", "direction": "desc" },
    { "field": "firstNameLowerCase", "direction": "asc" }
  ]
}
```

---

## Common Patterns

### Find contact by email
```json
{
  "locationId": "{locationId}",
  "pageLimit": 1,
  "filters": [
    { "field": "email", "operator": "eq", "value": "john@example.com" }
  ]
}
```

### Find contacts with a specific tag
```json
{
  "locationId": "{locationId}",
  "pageLimit": 100,
  "filters": [
    { "field": "tags", "operator": "eq", "value": "booked call" }
  ]
}
```

### Find contacts created in a date range
```json
{
  "locationId": "{locationId}",
  "pageLimit": 100,
  "filters": [
    {
      "field": "dateAdded",
      "operator": "range",
      "value": { "low": "2026-01-01", "high": "2026-03-31" }
    }
  ]
}
```

### Find unassigned contacts
```json
{
  "locationId": "{locationId}",
  "pageLimit": 100,
  "filters": [
    { "field": "assignedTo", "operator": "not_exists" }
  ]
}
```

### Paginate through all contacts (cursor-based)
```
1. First request: POST /contacts/search with page=1, pageLimit=100
2. Get searchAfter from the last contact in response
3. Next request: POST /contacts/search with searchAfter=[...], pageLimit=100 (no page param)
4. Repeat until contacts array is empty
```
