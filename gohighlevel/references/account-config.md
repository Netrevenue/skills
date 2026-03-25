# GHL Account Configuration API Reference

All endpoints use base URL `https://services.leadconnectorhq.com` with Sub-Account PIT auth.

Required headers on every request:
```
Authorization: Bearer {pit_token}
Version: 2021-07-28
Content-Type: application/json
Accept: application/json
```

---

## Get Sub-Account / Location

Retrieve the sub-account's details including address, settings, and configuration.

```
GET /locations/{locationId}
```

**Path Parameters:**
- `locationId` (required): Sub-account ID

**Response (200):**
```json
{
  "location": {
    "id": "ox6oaZ5j2iG0qxjc9Pfc",
    "brandId": "19SGOnV3i6Eqnt0ouXR1",
    "companyId": "VBW5vxWDhCP6mOEKBupq",
    "name": "Testing Account",
    "address": "Test",
    "city": "test",
    "state": "test",
    "country": "CA",
    "postalCode": "15478",
    "website": "",
    "timezone": "America/New_York",
    "firstName": "Shiloh",
    "lastName": "Rodrigues",
    "email": "shiloh@netrevenue.io",
    "phone": "+15147738698",
    "logoUrl": "",
    "automaticMobileAppInvite": false,
    "business": {
      "name": "Testing Account",
      "address": "Test",
      "city": "test",
      "state": "test",
      "country": "CA",
      "postalCode": "15478",
      "timezone": "America/New_York"
    },
    "social": {
      "facebookUrl": "",
      "googlePlus": "",
      "linkedIn": "",
      "foursquare": "",
      "twitter": "",
      "yelp": "",
      "instagram": "",
      "youtube": "",
      "pinterest": "",
      "blogRss": "",
      "googlePlacesId": ""
    },
    "settings": {
      "allowDuplicateContact": false,
      "allowDuplicateOpportunity": false,
      "allowFacebookNameMerge": false,
      "disableContactTimezone": false,
      "contactUniqueIdentifiers": ["email", "phone"],
      "crmSettings": {
        "deSyncOwners": false,
        "syncFollowers": {
          "contact": false,
          "opportunity": false
        }
      },
      "saasSettings": {
        "saasMode": "not_activated",
        "twilioRebilling": {
          "markup": 10,
          "enabled": false
        }
      }
    },
    "dateAdded": "2024-06-28T16:00:51.806Z",
    "domain": "",
    "currency": "",
    "isAgencySubAccount": {},
    "defaultEmailService": "",
    "permissions": {
      "dashboardStatsEnabled": true,
      "funnelsEnabled": true,
      "phoneCallEnabled": true,
      "formsEnabled": true,
      "textToPayEnabled": true,
      "gmbMessagingEnabled": true,
      "htmlBuilderEnabled": true,
      "contactsEnabled": true,
      "tagsEnabled": true,
      "botServiceEnabled": true,
      "websitesEnabled": true,
      "appointmentsEnabled": true,
      "proposalsEnabled": true,
      "webChatEnabled": true,
      "facebookMessengerEnabled": true,
      "affiliateManagerEnabled": true,
      "gmbCallTrackingEnabled": true,
      "marketingEnabled": true,
      "emailBuilderEnabled": true,
      "attributionsReportingEnabled": true,
      "triggerLinksEnabled": true,
      "membershipEnabled": true,
      "settingsEnabled": true,
      "surveysEnabled": true,
      "opportunitiesEnabled": true,
      "reviewsEnabled": true,
      "smsEmailTemplatesEnabled": true,
      "facebookAdsReportingEnabled": true,
      "adManagerEnabled": true,
      "bloggingEnabled": true,
      "workflowsEnabled": true,
      "campaignsEnabled": true,
      "conversationsEnabled": true,
      "adwordsReportingEnabled": true,
      "bulkRequestsEnabled": true,
      "agentReportingEnabled": true,
      "triggersEnabled": true
    },
    "snapshotId": ""
  },
  "traceId": "27862a40-ecd8-46da-ab53-e87e244c3fa2"
}
```

**Key field notes:**
- `brandId`: The agency brand this location belongs to.
- `companyId`: The agency company ID. Required for user creation and search.
- `business`: Duplicate of primary location address fields (for business listings).
- `social`: Social media profile URLs for the business.
- `settings.contactUniqueIdentifiers`: Array of fields used to deduplicate contacts (typically `["email", "phone"]`).
- `settings.crmSettings`: CRM-level configuration.
- `settings.saasSettings`: SaaS mode configuration (for white-label resellers).
- `permissions`: Location-level feature flags that control what's available to users in this sub-account.
- `isAgencySubAccount`: Empty object in observed data. Likely a boolean in other contexts.

**Key uses:**
- Get the sub-account's `state` and `city` to determine the correct area code for phone number purchasing
- Get the `timezone` for scheduling and audit context
- Get the `companyId` for user creation and search operations
- Verify account configuration during audits

---

## Update Sub-Account / Location

Update sub-account settings.

```
PUT /locations/{locationId}
```

**Request Body (partial update):**
```json
{
  "name": "Acme Corp - Updated",
  "address": "456 Market St",
  "city": "San Francisco",
  "state": "CA",
  "postalCode": "94105",
  "timezone": "America/Los_Angeles"
}
```

**CAUTION:** Changing location settings affects the entire sub-account. Confirm with the user before modifying.

**Response (200):** Returns the updated location object.

---

## Custom Fields (V2 API)

Manage custom fields for contacts, opportunities, and other objects.

### List Custom Fields

```
GET /locations/{locationId}/customFields
```

**Query Parameters:**
- `model` (optional): Filter by object type: `"contact"`, `"opportunity"`, etc.

**Response (200):**
```json
{
  "customFields": [
    {
      "id": "OSdgsWb8ojmYn19KJrlL",
      "name": "SDR",
      "model": "contact",
      "fieldKey": "contact.sdr",
      "placeholder": "Enter name exactly as in Slack",
      "dataType": "TEXT",
      "position": 50,
      "documentType": "field",
      "parentId": "P74K0s2IP5IWKZxgZIhY",
      "locationId": "ox6oaZ5j2iG0qxjc9Pfc",
      "dateAdded": "2024-07-02T15:58:09.590Z",
      "standard": false
    },
    {
      "id": "SnIDv0ZmOr6AbJGqpVgJ",
      "name": "Call Disposition",
      "model": "contact",
      "fieldKey": "contact.call_disposition",
      "placeholder": "",
      "dataType": "SINGLE_OPTIONS",
      "position": 350,
      "documentType": "field",
      "parentId": "P74K0s2IP5IWKZxgZIhY",
      "locationId": "ox6oaZ5j2iG0qxjc9Pfc",
      "dateAdded": "2025-11-14T16:40:40.024Z",
      "standard": false,
      "picklistOptions": [
        "No Answer",
        "Engaged - Follow Up",
        "Unqualified",
        "Not Interested",
        "Closing Call Booked"
      ]
    }
  ],
  "traceId": "574aea84-4ca3-4180-a1fa-a4f411b8c2b3"
}
```

**Key field notes:**
- `fieldKey`: The API key used to reference this field (e.g., in opportunity/contact objects).
- `dataType`: See list below for all types.
- `picklistOptions`: Array of options for `SINGLE_OPTIONS` and `MULTIPLE_OPTIONS` types.
- `standard`: `false` for custom fields, `true` for GHL built-in fields.
- `position`: Display order in the UI.

### Create Custom Field

```
POST /locations/{locationId}/customFields
```

**Request Body:**
```json
{
  "name": "Rep Assignment Date",
  "dataType": "DATE",
  "model": "contact",
  "placeholder": "Date rep was assigned"
}
```

**Data types:** `"TEXT"`, `"LARGE_TEXT"`, `"NUMERICAL"`, `"PHONE"`, `"MONETORY"` (note: GHL uses this spelling, not "MONETARY"), `"CHECKBOX"`, `"SINGLE_OPTIONS"`, `"MULTIPLE_OPTIONS"`, `"FLOAT"`, `"DATE"`, `"TEXTBOX_LIST"`, `"FILE_UPLOAD"`, `"SIGNATURE"`

For `SINGLE_OPTIONS` and `MULTIPLE_OPTIONS`, include the options:
```json
{
  "name": "Lead Quality",
  "dataType": "SINGLE_OPTIONS",
  "model": "contact",
  "options": [
    { "value": "hot", "name": "Hot" },
    { "value": "warm", "name": "Warm" },
    { "value": "cold", "name": "Cold" }
  ]
}
```

**Response (201):** Returns the created custom field object.

### Update Custom Field

```
PUT /locations/{locationId}/customFields/{customFieldId}
```

**Request Body:** Include only the fields to update.

**Response (200):** Returns the updated custom field object.

### Delete Custom Field

```
DELETE /locations/{locationId}/customFields/{customFieldId}
```

**CAUTION:** Deleting a custom field removes its data from all records. This is irreversible.

**Response (200):** Success response (shape not verified).

---

## Location Tags

Manage tags available in the sub-account. Tags are used on contacts for segmentation.

**Note:** Tag management endpoints may be nested under contacts or locations. The primary way to work with tags is through the contacts API:

**Add tags to a contact:**
```
POST /contacts/{contactId}/tags
```
```json
{
  "tags": ["new-lead", "vip"]
}
```

**Remove tag from a contact:**
```
DELETE /contacts/{contactId}/tags
```
```json
{
  "tags": ["old-tag"]
}
```
