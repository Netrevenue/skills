# GHL Phone System API Reference

All endpoints use base URL `https://services.leadconnectorhq.com` with Sub-Account PIT auth.

Required headers on every request:
```
Authorization: Bearer {pit_token}
Version: 2021-07-28
Content-Type: application/json
Accept: application/json
```

---

## List Active Phone Numbers

Retrieve all phone numbers currently active in a sub-account.

```
GET /phone-system/numbers/location/{locationId}
```

**Path Parameters:**
- `locationId` (required): Sub-account ID

**Response (200):**
```json
{
  "numbers": [
    {
      "phoneNumber": "+14387032919",
      "friendlyName": "John Dear's Number",
      "sid": "{SID}",
      "countryCode": "CA",
      "capabilities": {
        "fax": false,
        "mms": true,
        "sms": true,
        "voice": true
      },
      "type": "local",
      "isDefaultNumber": false,
      "linkedUser": "WYrr6bdoI44Wgp46nHtb",
      "linkedRingAllUsers": [],
      "forwardingNumber": "",
      "isGroupConversationEnabled": true,
      "addressSid": null,
      "bundleSid": null,
      "dateAdded": "Mon, 23 Feb 2026 22:00:52 +0000",
      "dateUpdated": "Tue, 24 Feb 2026 10:15:26 +0000",
      "origin": "twilio"
    }
  ],
  "isUnderGhl": true,
  "pageSize": 1000,
  "page": 0,
  "accountStatus": "active",
  "rcsSenderIds": []
}
```

**Key field notes:**
- `linkedUser` contains the userId of the user this number is assigned to. Empty string if unassigned.
- `linkedRingAllUsers` is an array of userIds who should ring when this number is called (for group/ring-all scenarios).
- `sid` is the Twilio SID for the phone number resource.
- `type` values: `"local"`, `"tollFree"`, `"mobile"`
- `isDefaultNumber` indicates if this is the default number for the location.

---

## Search Available Phone Numbers

Search for phone numbers available for purchase.

```
GET /phone-system/numbers/location/{locationId}/available
```

**Path Parameters:**
- `locationId` (required): Sub-account ID

**Query Parameters:**
- `countryCode` (required): ISO country code (e.g., `"US"`, `"CA"`)
- `numberTypes` (required): Type of number to search for (e.g., `"local"`, `"tollFree"`, `"mobile"`)
- `firstPart` (optional): Filter by the first part of the phone number (typically area code, e.g., `"415"`, `"212"`, `"438"`)
- `contains` (optional): Filter numbers containing this digit sequence
- `limit` (optional): Max results to return

**Response (200):**
```json
{
  "fingerprintId": "1773775197629",
  "numbers": [
    {
      "phoneNumber": "+14155551234",
      "friendlyName": "(415) 555-1234",
      "locality": "San Francisco",
      "region": "CA",
      "isoCountry": "US",
      "capabilities": {
        "voice": true,
        "sms": true,
        "mms": true
      }
    }
  ]
}
```

**Note:** The response returns an empty `numbers` array if no numbers are available matching the criteria. The `fingerprintId` is **CRITICAL** - it must be included in the purchase request to successfully buy a number from this search result set.

**Area Code Selection Strategy:**
1. Get the sub-account's location/region via GET /locations/{locationId}
2. Use the primary area code for that city/region as the `firstPart` parameter
3. If no numbers are available, try these fallback strategies:
   - Neighboring area codes in the same metropolitan area
   - Other area codes in the same state
   - Ask the user for an alternative area code preference
4. Common area code groupings for major metros (use as fallback reference):
   - San Francisco: 415, 628, 510, 650
   - Los Angeles: 213, 323, 310, 424, 818, 747
   - New York: 212, 646, 332, 917, 718, 347, 929
   - Chicago: 312, 872, 773
   - Dallas: 214, 469, 972, 945
   - Miami: 305, 786, 954
   - If the target metro is not listed above, search online for "area codes near {city}" to find neighboring codes

---

## Purchase Phone Number

Purchase a phone number for a sub-account.

```
POST /phone-system/numbers/location/{locationId}/purchase
```

**Path Parameters:**
- `locationId` (required): Sub-account ID

**Request Body:**
```json
{
  "phoneNumber": "+14388394450",
  "countryCode": "CA",
  "numberType": "local",
  "fingerprintId": "1774462989812"
}
```

**Field Notes:**
- `phoneNumber` (required): The full E.164 formatted number from the search results (e.g., `"+14388394450"`)
- `countryCode` (required): ISO country code matching the number (e.g., `"US"`, `"CA"`)
- `numberType` (required): Must match the type from search results (`"local"`, `"tollFree"`, `"mobile"`)
- `fingerprintId` (required): The `fingerprintId` value from the search response - this ties the purchase to the specific search results

**Response (201):**
```json
{
  "id": "bETS543iEjDlndeY9rWK",
  "account_sid": "{SID}",
  "under_ghl_account": true,
  "validate_sms": true,
  "location_id": "ox6oaZ5j2iG0qxjc9Pfc",
  "migration_numbers": [],
  "assigned_to_numbers": {
    "+14386065685": "dlhRBzdJhuW0Z5X22AXD"
  },
  "numbers": {
    "+14388394450": "conversation"
  },
  "number_name": {
    "+14388394450": "(438) 839-4450"
  },
  "new_account_sid": ""
}
```

Returns updated phone system state showing the newly purchased number in the `numbers` object.

**CAUTION:** Purchasing a number incurs a recurring cost. Always confirm with the user before purchasing.

---

## Assign Phone Number to User

There is NO dedicated phone number update endpoint. Phone number assignment is managed through the **Users API**.

To assign a phone number to a user, update the user's `lcPhone` field:

```
PUT /users/{userId}
```

**Request Body:**
```json
{
  "lcPhone": {
    "ox6oaZ5j2iG0qxjc9Pfc": "+14387032919"
  }
}
```

**Process:**
1. GET the user to retrieve their current `lcPhone` object
2. Add or update the locationId → phone number mapping
3. PUT the entire user object back (or at minimum the `lcPhone` field)

**To unassign a phone number from a user:**
Set the phone number value to an empty string for that location:
```json
{
  "lcPhone": {
    "ox6oaZ5j2iG0qxjc9Pfc": ""
  }
}
```

**Note:** The phone number resource itself has a `linkedUser` field that reflects this assignment, but it is read-only from the phone system API perspective. Assignment is controlled via the users API.

---

## Phone Number Management Workflow

### Purchasing and assigning a number for a new rep:

```
Step 1: Get the sub-account location to determine region
  GET /locations/{locationId}
  -> Extract address/state/city to determine area code

Step 2: Search for available numbers
  GET /phone-system/numbers/location/{locationId}/available?countryCode=US&numberTypes=local&firstPart={areaCode}
  -> Save the fingerprintId from the response
  -> If empty, try neighboring area codes (see area code strategy above)

Step 3: Confirm number selection with user
  Present the available numbers and ask which one to purchase.
  If only one result, confirm: "Found (415) 555-1234. Purchase this number?"

Step 4: Purchase the number
  POST /phone-system/numbers/location/{locationId}/purchase
  Body: {
    phoneNumber: "+14155551234",
    countryCode: "US",
    numberType: "local",
    fingerprintId: "1774462989812"
  }

Step 5: Assign to the user
  GET /users/{userId} to retrieve current lcPhone
  PUT /users/{userId} with updated lcPhone object:
  { "lcPhone": { "locationId": "+14155551234" } }

Step 6: Confirm completion
  Report: "Purchased +14155551234 and assigned to John Smith on the Acme Corp account."
```

### Checking which users have/don't have numbers (for auditing):

```
Step 1: List all users
  GET /users/?locationId={locationId}

Step 2: List all active numbers
  GET /phone-system/numbers/location/{locationId}

Step 3: Cross-reference
  For each user, check if their userId appears in any number's linkedUser field.
  Alternatively, check if their lcPhone object has an entry for the location.
  Report users without an assigned number.
```

### Unassigning a number (during offboarding):

```
Step 1: GET the user to retrieve their current lcPhone
  GET /users/{userId}

Step 2: Update their lcPhone to set the location's number to empty string
  PUT /users/{userId}
  Body: { "lcPhone": { "locationId": "" } }

```
