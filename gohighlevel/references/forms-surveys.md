# GHL Forms & Surveys API Reference

All endpoints use base URL `https://services.leadconnectorhq.com` with Sub-Account PIT auth.

Required headers on every request:
```
Authorization: Bearer {pit_token}
Version: 2021-07-28
Content-Type: application/json
Accept: application/json
```

---

## Understanding Forms vs Surveys in GHL

Forms and surveys serve similar purposes in GHL — both capture lead data via web-embedded questionnaires. Users often refer to them interchangeably as "applications", "lead forms", or "qualification forms". The key differences:

- **Forms** (`/forms/`) — simpler structure, single-page. Submissions appear under `/forms/submissions`.
- **Surveys** (`/surveys/`) — multi-step, can include conditional logic. Submissions appear under `/surveys/submissions`.

When a user asks about "application data", "lead data", or "qualification data", check **both** forms and surveys — the account may use either or both.

**Important: 3rd-party form builders.** Some GHL accounts do NOT use GHL's internal forms/surveys for lead capture. They use external form builders (Typeform, JotForm, etc.) integrated via webhook or Zapier to create the contact directly. In those cases:
- There will be **no submissions** in the forms/surveys endpoints.
- The captured data is still stored as **custom fields on the contact record** (see `references/account-config.md` for custom fields).
- If forms/surveys submissions come back empty but the account clearly has leads, check the contact's custom fields instead via the Contacts API.
- Ask the user: "This account may be using an external form builder. Would you like me to check the contact custom fields instead?"

---

## List Forms

Retrieve all forms in a GHL sub-account.

```
GET /forms/?locationId={locationId}
```

**Query Parameters:**
- `locationId` (required): The GHL sub-account/location ID

**Response (200):**
```json
{
  "forms": [
    {
      "id": "Vn5xBZJsQyADy5QlGmtt",
      "locationId": "5PyQjfcGFRFxYjFyxEBy",
      "name": "02-26 Application Form Masterclass"
    },
    {
      "id": "ykicAwjYjgaFRjKlwhkW",
      "locationId": "5PyQjfcGFRFxYjFyxEBy",
      "name": "Application Form Bootcamp"
    },
    {
      "id": "6EC9APYFXEsAboIV9RUn",
      "locationId": "5PyQjfcGFRFxYjFyxEBy",
      "name": "Cancel Form"
    }
  ],
  "total": 22,
  "traceId": "c7ba6bde-c7ee-4eda-a034-b377bd3ea8be"
}
```

**Key field notes:**
- `total` is returned as a number (despite the official docs showing it as a string — verified as number in practice).
- The list endpoint returns **all** forms with no pagination parameters. The full list is returned in a single response.
- Each form object is minimal: only `id`, `locationId`, and `name`. There is no detail about form fields or structure from this endpoint.

---

## Get Form Submissions

Retrieve submissions for forms in a GHL sub-account. Supports filtering by form, contact, date range, and search.

```
GET /forms/submissions?locationId={locationId}
```

**Query Parameters:**
- `locationId` (required): The GHL sub-account/location ID
- `formId` (optional): Filter submissions to a specific form ID
- `page` (optional): Page number. Default: `1`
- `limit` (optional): Results per page. Default: `20`, max: `100`
- `q` (optional): Search by contactId, name, email, or phone number
- `startAt` (optional): Start date filter, format `YYYY-MM-DD`. Default: one month ago
- `endAt` (optional): End date filter, format `YYYY-MM-DD`. Default: today

**Response (200):**
```json
{
  "submissions": [
    {
      "id": "69c4b7cc2d98452bb4230eb1",
      "contactId": "I9mabTv8G9s7NYaWN9cO",
      "formId": "KrOKPfnKuA75kGeZrw8G",
      "name": "Z K",
      "email": "email@example.com",
      "others": {
        "er2CKhmOFeBd1Xjnv77M": ["Wholesaling"],
        "WAHcAJ5YRBGgfwp9xRso": ["Not sure where to start"],
        "U9GMof6lCh6CH7MZZYm4": ["Missing my window to build wealth"],
        "phone": "+212658015873",
        "email": "zakaria.tin@gmail.com",
        "SrJvjpBhx9swA9vWMXk8": "I'm brand new to real estate and ready to close my first deal",
        "CIDGQrOf19W3mf9L8TA6": "None yet",
        "o8WzOyMWUlI7wJ6AEPpK": "Less than $5,000",
        "ZIUfoCzpcaanJ3jTPTCL": "1-3 months",
        "DVJQSueiEp5aPMKKsK0I": "No I don't",
        "formId": "KrOKPfnKuA75kGeZrw8G",
        "location_id": "5PyQjfcGFRFxYjFyxEBy",
        "first_name": "Z",
        "last_name": "K",
        "Timezone": "Europe/Paris",
        "eventData": {
          "source": "Social media",
          "referrer": "https://l.instagram.com",
          "page": {
            "url": "https://www.url.com"
          },
          "parentName": "Free Resource Booking Calendar",
          "medium": "calendar",
          "mediumId": "S3muLxClN8WGgBZ9fKGO"
        },
        "fieldsOriSequance": [
          "first_name", "last_name", "phone", "email",
          "SrJvjpBhx9swA9vWMXk8", "er2CKhmOFeBd1Xjnv77M"
        ]
      },
      "createdAt": "2026-03-26T04:36:28.553Z",
      "external": false
    }
  ],
  "meta": {
    "total": 14,
    "currentPage": 1,
    "nextPage": 2,
    "prevPage": null
  },
  "traceId": "5052dfe5-aa53-4aad-9dd0-0a81fefdf901"
}
```

**Key field notes:**
- `others` is a flat key-value bag containing ALL submission data. Keys are either custom field IDs (opaque strings like `"er2CKhmOFeBd1Xjnv77M"`) or standard field names (`"phone"`, `"email"`, `"first_name"`, `"last_name"`).
- **Custom field IDs are not human-readable.** To map them to question labels, cross-reference with the account's custom fields (see `references/account-config.md`). The custom field's `name` tells you what the question was.
- `fieldsOriSequance` (GHL's typo — not "Sequence") lists the field IDs in the order they appeared on the form. Useful for reconstructing the form layout.
- `eventData` contains attribution data: `source`, `referrer`, `page.url`, `medium`, and the parent form/calendar/survey name.
- `eventData.medium` indicates how the submission was triggered — `"calendar"` means it came through a booking flow, `"survey"` from an embedded survey, etc.
- `external` — `false` means submitted via GHL's native form. If `true`, it was created via API/integration.
- `createdAt` is the submission timestamp in ISO 8601 format.
- Pagination uses `meta.nextPage` / `meta.prevPage`. When `nextPage` is `null`, you've reached the last page.
- **Default date range:** If `startAt`/`endAt` are omitted, the API returns submissions from the last 30 days only. To get older submissions, you must explicitly set `startAt`.

---

## List Surveys

Retrieve all surveys in a GHL sub-account.

```
GET /surveys/?locationId={locationId}
```

**Query Parameters:**
- `locationId` (required): The GHL sub-account/location ID
- `skip` (optional): Number of records to skip (for pagination). Default: `0`
- `limit` (optional): Results per page. Default: `10`, max: `50`
- `type` (optional): Filter by type (e.g., `"folder"`)

**Response (200):**
```json
{
  "surveys": [
    {
      "id": "7Yw1RmBHov8KMXzFTOvh",
      "locationId": "5PyQjfcGFRFxYjFyxEBy",
      "name": "Application Form - Free Resource Magnet"
    },
    {
      "id": "CVe8M2cM3fc56IyOPoZG",
      "locationId": "5PyQjfcGFRFxYjFyxEBy",
      "name": "Application Form - Bootcamp"
    },
    {
      "id": "9Jp2T6nhDmyDeygK6CfS",
      "locationId": "5PyQjfcGFRFxYjFyxEBy",
      "name": "MM | Application Form - Webinar"
    }
  ],
  "total": 29,
  "traceId": "85fe126f-33b3-4f1e-b918-762ae6b22308"
}
```

**Key field notes:**
- Unlike the forms list endpoint, surveys **do** support pagination via `skip` and `limit`.
- Default `limit` is `10` (not all records). To get all surveys, paginate using `skip` in increments of `limit` until you've collected `total` records.
- Each survey object is minimal: only `id`, `locationId`, and `name`.

---

## Get Survey Submissions

Retrieve submissions for surveys in a GHL sub-account. Supports filtering by survey, contact, date range, and search.

```
GET /surveys/submissions?locationId={locationId}
```

**Query Parameters:**
- `locationId` (required): The GHL sub-account/location ID
- `surveyId` (optional): Filter submissions to a specific survey ID
- `page` (optional): Page number. Default: `1`
- `limit` (optional): Results per page. Default: `20`, max: `100`
- `q` (optional): Search by contactId, name, email, or phone number
- `startAt` (optional): Start date filter, format `YYYY-MM-DD`. Default: one month ago
- `endAt` (optional): End date filter, format `YYYY-MM-DD`. Default: today

**Response (200):**
```json
{
  "submissions": [
    {
      "id": "69c9986f2d32b578d3d4f259",
      "contactId": "bZOSc2NrRsQkJlg56YRk",
      "surveyId": "7Yw1RmBHov8KMXzFTOvh",
      "name": "I B",
      "email": "lead@email.com",
      "others": {
        "PZYHXUi2wUr1kOeFNCRN": "sellerleads",
        "full_name": "I B",
        "email": "lead@example.com",
        "phone": "+15182311101",
        "terms_and_conditions": "I Consent to Receive SMS Notifications...",
        "formId": "7Yw1RmBHov8KMXzFTOvh",
        "location_id": "5PyQjfcGFRFxYjFyxEBy",
        "disqualified": true,
        "SrJvjpBhx9swA9vWMXk8": "I'm brand new to real estate and ready to close my first deal",
        "WAHcAJ5YRBGgfwp9xRso": ["Selling deals and building my buyer list"],
        "U9GMof6lCh6CH7MZZYm4": ["Missing my window to build wealth"],
        "ZIUfoCzpcaanJ3jTPTCL": "ASAP",
        "enjZL6rGTgt1mDQU6EeH": ["Yes"],
        "o8WzOyMWUlI7wJ6AEPpK": "Less than $5,000",
        "Timezone": "America/New_York (GMT-04:00)",
        "eventData": {
          "source": "Referral",
          "referrer": "https://example.com",
          "page": {
            "url": "https://example.com"
          },
          "parentName": "Application Form - Free Resource Magnet",
          "medium": "survey",
          "mediumId": "7Yw1RmBHov8KMXzFTOvh"
        },
        "fieldsOriSequance": [
          "html", "full_name", "email", "phone",
          "terms_and_conditions", "PZYHXUi2wUr1kOeFNCRN"
        ]
      },
      "createdAt": "2026-03-29T21:23:59.031Z"
    }
  ],
  "meta": {
    "total": 19,
    "currentPage": 1,
    "nextPage": 2,
    "prevPage": null
  },
  "traceId": "385070dc-d892-4cf5-9c4f-b7f88d3e8b54"
}
```

**Key field notes:**
- The response shape is identical to form submissions, but uses `surveyId` instead of `formId` at the submission level.
- `others.disqualified` — boolean, present on survey submissions. Indicates whether the respondent was disqualified by the survey's conditional logic. Useful for filtering qualified vs. unqualified leads.
- `others.formId` inside the `others` bag refers to the survey ID (confusing naming — GHL reuses `formId` internally for both forms and surveys).
- All the same notes from form submissions apply: custom field IDs need cross-referencing, `fieldsOriSequance` is a typo, `eventData` has attribution, and the default date range is last 30 days.

---

## IMPORTANT: Resolving Custom Field Names (Required Step)

**You MUST always resolve custom field IDs to human-readable question labels before presenting submission data to the user.** Submission responses contain opaque field IDs (e.g., `"aW4C0qMJbF15uWQ2bgO5"`) that are meaningless on their own. The custom fields endpoint maps these IDs to their actual question text.

### Step 1: Fetch the custom field mapping

Call this endpoint **once per session** (before or in parallel with fetching submissions):

```
GET /locations/{locationId}/customFields
```

This returns all custom fields for the account. Build a lookup map of `id` → `name`:

```json
{
  "customFields": [
    {
      "id": "aW4C0qMJbF15uWQ2bgO5",
      "name": "What best describes your current role or situation?",
      "dataType": "RADIO",
      ...
    },
    {
      "id": "771ALwnV26PJ74La1ZDD",
      "name": "Team Size:",
      "dataType": "TEXT",
      ...
    }
  ]
}
```

### Step 2: Replace IDs in submission `others` with readable names

For each submission, iterate over the `others` key-value pairs. For any key that matches a custom field ID, replace it with the field's `name`. Skip standard keys (`phone`, `email`, `first_name`, `last_name`, `full_name`, `formId`, `location_id`, `Timezone`, `eventData`, `fieldsOriSequance`, `sessionId`, `sessionFingerprint`, `submissionId`, `signatureHash`, `ip`, `terms_and_conditions`, `source`, `dateFieldDetails`, `internalSource`, `calendar_name`, `selected_timezone`, `selected_slot`, `disqualified`, `contact_id`).

**Example — raw submission `others`:**
```json
{
  "aW4C0qMJbF15uWQ2bgO5": "Founder of a business",
  "L1WyG5ycOat0GnYjJTZu": "WXM LEGAL LTD",
  "8oYKsVejNf5Ctb6NBttn": "Consulting / business Influencer",
  "771ALwnV26PJ74La1ZDD": "1",
  "25uz32S95fEn7ZTBjc4v": "Me",
  "yQj1mZwBgjRs8RRnnPeL": "8 figure revenue",
  "NesGqea9QpXrotrwN8SH": "Millionaire status",
  "8g3ZONxBrEtfnuV1A8JX": "Momentum. And now is perfect butterfly effect"
}
```

**After resolving with custom fields mapping:**
```
What best describes your current role or situation? → Founder of a business
Business Name: → WXM LEGAL LTD
Main Offer / Product / Service: → Consulting / business Influencer
Team Size: → 1
#1 Bottleneck Right Now (What's holding you back?) → Me
What's your #1 goal in the next 90 days? → 8 figure revenue
What does growth look like for you this year? → Millionaire status
Why Now? → Momentum. And now is perfect butterfly effect
```

**Always present submission data in this resolved format.** Never show raw field IDs to the user.

### Notes on field ID resolution

- Some keys in `others` won't match any custom field — these are standard GHL fields (email, phone, etc.) or internal metadata. Present those by their key name directly.
- If the custom fields endpoint returns 401, the token may lack the `locations/customFields.readonly` scope. In that case, inform the user that field names can't be resolved and present the raw values with a note that the IDs are unresolved.
- Cache the custom field mapping for the session — it won't change between requests.

---

## Key Gotchas

1. **`fieldsOriSequance` is misspelled.** GHL spells it "Sequance" not "Sequence". Use this exact key when accessing it.

2. **Date range defaults to last 30 days.** If you're looking for older submissions and getting empty results, explicitly set `startAt` to an earlier date.

3. **Forms list has no pagination; surveys list does.** Forms returns all records in one response. Surveys uses `skip`/`limit` and defaults to 10 per page.

4. **Submissions pagination differs from list pagination.** Submissions use `page`/`limit` with `meta.nextPage`. Surveys list uses `skip`/`limit`.

5. **3rd-party forms won't appear here.** If an account uses Typeform, JotForm, or similar, submissions won't show in these endpoints. The data lives on the contact's custom fields instead.

6. **Survey submissions may include `disqualified: true`.** This means the lead failed the survey's qualification logic. Filter these out when analyzing qualified leads.

---

## Workflows

### Finding Application/Lead Data

When a user asks about "applications", "lead data", or "who applied recently":

1. **Fetch the custom field mapping** — `GET /locations/{locationId}/customFields`. Do this in parallel with the next steps.
2. **List both forms and surveys** to identify which ones are application forms (look for names containing "Application", "Qualification", etc.).
3. **Query submissions** for the matching form/survey ID(s) with the relevant date range.
4. **If submissions are empty**, the account may use 3rd-party forms. Check contact custom fields instead.
5. **Resolve all field IDs** using the custom field mapping and present readable question/answer pairs.

### Answering Questions About Specific Form Fields

When a user asks "what did [contact] answer for [question]":

1. **Fetch the custom field mapping** and **search submissions** (using the `q` parameter with the contact's name/email) in parallel.
2. **Match the question** text to its custom field ID using the mapping, then look up the value in the submission's `others` bag.

### Aggregating Submission Data

When a user wants a summary of responses (e.g., "how many leads said ASAP"):

1. **Fetch the custom field mapping** first to identify which field ID corresponds to the question being asked.
2. **Paginate through all submissions** for the target form/survey (set `limit=100`, iterate through `page` until `meta.nextPage` is `null`).
3. **Set `startAt` explicitly** if the desired range exceeds 30 days.
4. **Aggregate answers** using the resolved field names for the summary.
