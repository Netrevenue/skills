# GHL Pipelines & Opportunities API Reference

All endpoints use base URL `https://services.leadconnectorhq.com` with Sub-Account PIT auth.

Required headers on every request:
```
Authorization: Bearer {pit_token}
Version: 2021-07-28
Content-Type: application/json
Accept: application/json
```

**Scope note:** Pipeline structure (creating/editing pipelines and stages) is NOT available via API. The API provides read-only access to pipeline definitions and full CRUD on opportunities (deals) within pipelines. If pipeline structure changes are needed, use the GHL frontend.

---

## List Pipelines

Get all pipelines and their stages for a sub-account.

```
GET /opportunities/pipelines
```

**Query Parameters:**
- `locationId` (required): Sub-account ID

**Response (200):**
```json
{
  "pipelines": [
    {
      "id": "reZzabRl5XS2NMOLkE3s",
      "name": "NetRev Sales",
      "originId": "vORwC4366y7IQLTTrR2S",
      "dateAdded": "2025-04-10T13:42:59.516Z",
      "dateUpdated": "2025-04-10T13:42:59.517Z",
      "stages": [
        {
          "id": "2f8862a1-8480-4b60-a8e2-f0b8fbbb3a07",
          "name": "Booked Call",
          "originId": "MTIzYTZiNTQtN2MyNi00NTZhLTgxYzUtZWY0NTE1OGZhMGI4",
          "position": 0,
          "showInFunnel": true,
          "showInPieChart": true
        },
        {
          "id": "b83be732-96b8-46a6-83ab-56b91d3d93b9",
          "name": "Rescheduled",
          "originId": "NTY5YmRlOTMtODAyNi00MDFhLWE1NzMtNmJkOTNjODg3YmYy",
          "position": 1,
          "showInFunnel": true,
          "showInPieChart": true
        }
      ]
    }
  ],
  "traceId": "152c94e8-4c53-46c7-b321-42d63ff7e1a7"
}
```

**Key field notes:**
- `originId` appears on both pipelines and stages. This may be used for tracking imported/copied pipelines from templates.
- Some pipelines also include `showInFunnel` and `showInPieChart` at the pipeline level (not shown in example above but observed in data).
- `position` determines the display order of stages (0-indexed).

Useful for auditing: verify that expected stages exist and are in the correct order.

---

## Search Opportunities

Search for deals/opportunities with filtering.

```
GET /opportunities/search
```

**Query Parameters:**
- `location_id` (required): Sub-account ID. **IMPORTANT:** Use snake_case `location_id`, NOT camelCase `locationId`.
- `pipelineId` (optional): Filter to a specific pipeline
- `pipelineStageId` (optional): Filter to a specific stage
- `status` (optional): `"open"`, `"won"`, `"lost"`, `"abandoned"`
- `assignedTo` (optional): User ID to filter by assigned rep
- `contactId` (optional): Filter by contact
- `query` (optional): Search term (matches against opportunity name)
- `limit` (optional): Results per page (default 20, max 100)
- `startAfter` (optional): Pagination cursor (Unix timestamp in milliseconds)
- `startAfterId` (optional): Pagination cursor ID
- `order` (optional): Sort order, `"added_asc"` or `"added_desc"`

**Response (200):**
```json
{
  "opportunities": [
    {
      "id": "9nIPBAugRGo7ZIAq6ULU",
      "name": "Nick French | March 6, 2026",
      "monetaryValue": 0,
      "pipelineId": "jlFOTGazbXfm8NcJCUJJ",
      "pipelineStageId": "bfd80cda-f727-4c59-af57-12361a5d3779",
      "pipelineStageUId": "bfd80cda-f727-4c59-af57-12361a5d3779",
      "assignedTo": "UdBJYEGU0xoSFcavXGcP",
      "status": "open",
      "source": null,
      "lastStatusChangeAt": "2026-03-06T20:27:20.238Z",
      "lastStageChangeAt": "2026-03-06T20:27:19.570Z",
      "createdAt": "2025-12-01T16:06:30.468Z",
      "updatedAt": "2026-03-06T20:27:20.238Z",
      "indexVersion": 3,
      "contactId": "neNUnHGMxxytPXZcvxcU",
      "locationId": "ox6oaZ5j2iG0qxjc9Pfc",
      "customFields": [],
      "lostReasonId": null,
      "followers": [],
      "relations": [
        {
          "associationId": "OPPORTUNITIES_CONTACTS_ASSOCIATION",
          "relationId": "9nIPBAugRGo7ZIAq6ULU",
          "primary": true,
          "objectKey": "contact",
          "recordId": "neNUnHGMxxytPXZcvxcU",
          "fullName": "Nick French",
          "contactName": "Nick French",
          "companyName": null,
          "email": "nick.french@netrevenue.io",
          "phone": "+14384665023",
          "tags": ["booked call"],
          "attributed": true
        }
      ],
      "contact": {
        "id": "neNUnHGMxxytPXZcvxcU",
        "name": "Nick French",
        "companyName": null,
        "email": "nick.french@netrevenue.io",
        "phone": "+14384665023",
        "tags": ["booked call"],
        "score": []
      },
      "sort": [1764605190468, "neNUnHGMxxytPXZcvxcU"],
      "attributions": [
        {
          "utmSessionSource": "CRM UI",
          "isFirst": true,
          "medium": "manual"
        }
      ]
    }
  ],
  "meta": {
    "total": 23,
    "nextPageUrl": "https://services.leadconnectorhq.com/opportunities/search?location_id=ox6oaZ5j2iG0qxjc9Pfc&limit=2&startAfter=1764004064616&startAfterId=neNUnHGMxxytPXZcvxcU",
    "startAfterId": "neNUnHGMxxytPXZcvxcU",
    "startAfter": 1764004064616,
    "currentPage": 1,
    "nextPage": 2,
    "prevPage": null
  },
  "traceId": "84e6fed4-1a7b-4f27-ac30-262b4f1672ad"
}
```

**Key field notes:**
- `pipelineStageUId` duplicates `pipelineStageId` in the observed data. Purpose unclear.
- `indexVersion` is used for internal search indexing.
- `customFields` is an array of custom field values for this opportunity.
- `followers` is an array of userIds who are following this opportunity (for notifications).
- `relations` includes associated contacts and their details. `primary: true` indicates the main contact.
- `contact` is a convenience object duplicating the primary relation's contact data.
- `sort` is an array used for cursor-based pagination.
- `attributions` tracks the source/medium of how the opportunity was created.

---

## Get Opportunity

Get a single opportunity by ID.

```
GET /opportunities/{opportunityId}
```

**Path Parameters:**
- `opportunityId` (required): Opportunity ID

**Response (200):** Same shape as a single opportunity object from search.

---

## Create Opportunity

Create a new deal/opportunity.

```
POST /opportunities/
```

**Request Body:**
```json
{
  "pipelineId": "pipeline_abc123",
  "locationId": "loc_xyz789",
  "name": "Jane Doe - Premium Package",
  "pipelineStageId": "stage_001",
  "status": "open",
  "contactId": "contact_xyz456",
  "monetaryValue": 3000,
  "assignedTo": "user_abc123",
  "source": "Phone Call"
}
```

**Required fields:** `pipelineId`, `locationId`, `name`, `pipelineStageId`, `status`, `contactId`

**Response (201):** Returns the created opportunity object.

---

## Update Opportunity

Update a deal's stage, value, status, assignment, etc.

```
PUT /opportunities/{opportunityId}
```

**Request Body (partial update):**
```json
{
  "pipelineStageId": "stage_003",
  "monetaryValue": 5500,
  "assignedTo": "user_xyz456"
}
```

**Response (200):** Returns the updated opportunity object.

---

## Update Opportunity Status

Dedicated endpoint for changing opportunity status (open, won, lost, abandoned).

```
PUT /opportunities/{opportunityId}/status
```

**Request Body:**
```json
{
  "status": "won"
}
```

Valid values: `"open"`, `"won"`, `"lost"`, `"abandoned"`

**Response (200):** Returns the updated opportunity object.

---

## Delete Opportunity

```
DELETE /opportunities/{opportunityId}
```

**CAUTION:** Permanent deletion. Confirm with user first.

**Response (200):** Success response (shape not verified).

---

## Upsert Opportunity

Create or update an opportunity. If a matching opportunity exists (by pipeline + contact), it updates; otherwise creates.

```
POST /opportunities/upsert
```

Same body as Create Opportunity. Matching is based on `pipelineId` + `contactId`.

**Response (200 or 201):** Returns the opportunity object.

---

## Pipeline Auditing Patterns

### Stage distribution analysis:
```
1. GET /opportunities/pipelines to get all stages
2. For each pipeline, GET /opportunities/search with status="open" (paginate through all)
3. Count opportunities per stage
4. Calculate percentage distribution
5. Flag stages with disproportionate concentration (e.g., >40% in a single stage)
6. Flag stale opportunities (lastActionDate > 14 days ago, still in early stages)
```

### Rep workload analysis:
```
1. GET /users/?locationId={locationId} to get all reps
2. For each rep, GET /opportunities/search with assignedTo={userId} and status="open"
3. Count open opportunities per rep
4. Report distribution and flag imbalances
```

### Pipeline health summary:
```
Report should include:
- Total open opportunities and total monetary value
- Distribution by stage (count and value)
- Average time in current stage (requires calculating from lastStageChangeAt)
- Stale opportunities (no action in X days)
- Unassigned opportunities (assignedTo is empty)
- Won/lost ratio for the period
```
