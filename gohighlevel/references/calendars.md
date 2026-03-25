# GHL Calendars API Reference

All endpoints use base URL `https://services.leadconnectorhq.com` with Sub-Account PIT auth.

Required headers on every request:
```
Authorization: Bearer {pit_token}
Version: 2021-07-28
Content-Type: application/json
Accept: application/json
```

---

## List Calendars

Get all calendars in a sub-account.

```
GET /calendars/
```

**Query Parameters:**
- `locationId` (required): Sub-account ID

**Response (200):**
```json
{
  "calendars": [
    {
      "id": "wOgSbXECifHbsctUQ4ZZ",
      "locationId": "ox6oaZ5j2iG0qxjc9Pfc",
      "groupId": "8d2dhTPqTgxC8Dh7Fg2P",
      "teamMembers": [
        {
          "priority": 0.5,
          "selected": true,
          "userId": "6X2oYxq1RoF0BwIMTg7c",
          "isZoomAdded": "true",
          "zoomOauthId": "w2y58nIs9Zs8HnuilUA5",
          "meetingLocation": "",
          "locationConfigurations": [
            {
              "location": "",
              "position": 0,
              "kind": "zoom_conference",
              "zoomOauthId": "w2y58nIs9Zs8HnuilUA5",
              "meetingId": "zoom_conference_0"
            }
          ],
          "isPrimary": true
        },
        {
          "priority": 0.5,
          "selected": true,
          "userId": "LtsEY7DZOE20t09TqfaG",
          "isZoomAdded": "false",
          "zoomOauthId": "",
          "meetingLocation": "",
          "locationConfigurations": [
            {
              "location": "",
              "position": 0,
              "kind": "custom",
              "zoomOauthId": "",
              "meetingId": "custom_0"
            }
          ]
        }
      ],
      "eventType": "RoundRobin_OptimizeForAvailability",
      "name": "Salesvue Testing",
      "description": "This is a 30 - 45 minute zoom call with our VP of enrollment.",
      "widgetSlug": "cal-81f4184d-dea1-4b6a-bdb1-c401a402c8da",
      "calendarType": "round_robin",
      "widgetType": "default",
      "eventTitle": "{{contact.name}} and **PROGRAM NAME**",
      "eventColor": "#3F51B5",
      "slotDuration": 45,
      "slotDurationUnit": "mins",
      "slotInterval": 15,
      "slotIntervalUnit": "mins",
      "slotBufferUnit": "mins",
      "appoinmentPerSlot": 1,
      "appoinmentPerDay": "",
      "openHours": [
        {
          "daysOfTheWeek": [1, 2, 3, 4, 5],
          "hours": [
            { "openHour": 9, "openMinute": 0, "closeHour": 17, "closeMinute": 0 }
          ]
        }
      ],
      "enableRecurring": false,
      "autoConfirm": true,
      "googleInvitationEmails": true,
      "allowReschedule": true,
      "allowCancellation": true,
      "formSubmitType": "ThankYouMessage",
      "isActive": true
    }
  ],
  "traceId": "ecec47c5-4320-48bf-94fe-6e69c0a7bded"
}
```

**Key field notes:**

**teamMembers structure:**
- `priority`: Round-robin weight (0-1). Higher values mean more appointments assigned to this user.
- `selected`: Boolean indicating if this user is currently active on the calendar.
- `userId`: The GHL user ID.
- `isZoomAdded`: String `"true"` or `"false"` (not boolean).
- `zoomOauthId`: OAuth connection ID for Zoom integration (empty string if not connected).
- `meetingLocation`: Custom meeting location string (empty if using locationConfigurations).
- `locationConfigurations`: Array of meeting location configs. Common `kind` values: `"zoom_conference"`, `"google_meet"`, `"custom"`, `"phone_call"`.
- `isPrimary`: Boolean indicating the primary host (appears on some but not all team members).

**Time and duration fields:**
- `slotDuration`, `slotInterval`: Numeric values.
- `slotDurationUnit`, `slotIntervalUnit`, `slotBufferUnit`: Always `"mins"` in observed data.
- `appoinmentPerSlot`: Max bookings per time slot.
- `appoinmentPerDay`: Max bookings per day (empty string for unlimited).

**openHours structure:**
- `daysOfTheWeek`: Array of day numbers (0=Sunday through 6=Saturday).
- `hours`: Array of time windows for that day. `0, 0, 0, 0` means closed/unavailable.

**Other fields:**
- `groupId`: Calendar group this calendar belongs to (optional).
- `eventType`: Common values: `"RoundRobin_OptimizeForAvailability"`, `"RoundRobin_OptimizeForEqualDistribution"`.
- `calendarType`: `"round_robin"`, `"personal"`, `"class_booking"`, `"collective"`, `"service_booking"`, `"event"`.
- `widgetType`: Usually `"default"`.
- `enableRecurring`: Boolean for recurring event support.
- `recurring`: Configuration for recurring events (usually not used for booking calendars).
- `stickyContact`: Boolean. If true, contact remains assigned to the same team member for future bookings.
- `autoConfirm`: Boolean. Auto-confirm bookings without manual approval.
- `googleInvitationEmails`: Boolean. Send Google Calendar invites.
- `allowReschedule`, `allowCancellation`: Boolean. Allow contacts to reschedule/cancel.
- `shouldAssignContactToTeamMember`: Boolean. Assign contact to the booked team member in CRM.
- `shouldSkipAssigningContactForExisting`: Boolean. Skip assignment if contact already has an owner.
- `formSubmitType`: `"ThankYouMessage"` or `"RedirectURL"`.
- `guestType`: `"count_only"` or `"collect_detail"`.
- `lookBusyConfig`: Settings to simulate partial availability.
- `allowBookingAfter`, `allowBookingAfterUnit`: Minimum lead time before a slot can be booked.
- `allowBookingFor`, `allowBookingForUnit`: How far in advance bookings are allowed.
- `isActive`: Boolean. Calendar is published and accepting bookings.

---

## Get Calendar

Get a specific calendar by ID.

```
GET /calendars/{calendarId}
```

**Path Parameters:**
- `calendarId` (required): Calendar ID

**Response (200):** Same shape as a single calendar object above.

---

## Create Calendar

Create a new calendar in a sub-account.

```
POST /calendars/
```

**Request Body (minimal example for round-robin):**
```json
{
  "locationId": "ox6oaZ5j2iG0qxjc9Pfc",
  "name": "Setter Calendar",
  "description": "Round-robin for appointment setters",
  "calendarType": "round_robin",
  "slotDuration": 30,
  "slotInterval": 30,
  "appoinmentPerSlot": 1,
  "openHours": [
    {
      "daysOfTheWeek": [1, 2, 3, 4, 5],
      "hours": [
        { "openHour": 9, "openMinute": 0, "closeHour": 17, "closeMinute": 0 }
      ]
    }
  ],
  "teamMembers": [
    {
      "userId": "user_abc123",
      "priority": 0.5,
      "selected": true,
      "isZoomAdded": "false",
      "locationConfigurations": [
        {
          "kind": "custom",
          "location": "Zoom",
          "position": 0
        }
      ]
    }
  ]
}
```

**Field Notes:**
- `locationId`: Required. The sub-account this calendar belongs to.
- `name`: Required. Display name of the calendar.
- `calendarType`: Required. Common values: `"round_robin"`, `"event"`, `"class_booking"`, `"collective"`, `"personal"`, `"service_booking"`.
- `slotDuration`: Required. Appointment length in minutes.
- `slotInterval`: Required. Interval between available start times in minutes.
- `teamMembers`: Required for round-robin calendars. Array of users assigned to this calendar.
  - Each entry must have `userId`, `priority` (0-1), `selected` (bool), `isZoomAdded` (string), and `locationConfigurations` array.
- `openHours`: Required. Availability windows. `daysOfTheWeek` uses 0=Sunday through 6=Saturday.

**Response (201):** Returns the created calendar object.

---

## Update Calendar

Modify a calendar's settings, team members, or configuration.

```
PUT /calendars/{calendarId}
```

**Path Parameters:**
- `calendarId` (required): Calendar ID

**Request Body:** Include only the fields to update. Most commonly used for managing team members:

**Adding a rep to a calendar:**
1. GET the calendar to retrieve the current `teamMembers` array.
2. Append the new user to the array.
3. PUT the calendar with the updated `teamMembers`.

```json
{
  "teamMembers": [
    {
      "userId": "existing_user_1",
      "priority": 0.5,
      "selected": true,
      "isZoomAdded": "false",
      "locationConfigurations": [
        {
          "kind": "custom",
          "location": "Zoom",
          "position": 0
        }
      ]
    },
    {
      "userId": "new_user_to_add",
      "priority": 0.5,
      "selected": true,
      "isZoomAdded": "false",
      "locationConfigurations": [
        {
          "kind": "custom",
          "location": "Zoom",
          "position": 0
        }
      ]
    }
  ]
}
```

**Removing a rep from a calendar:**
1. GET the calendar to retrieve current `teamMembers`.
2. Remove the target user from the array.
3. PUT the calendar with the updated `teamMembers` (without the removed user).

**IMPORTANT:** Always GET the calendar first and modify the full teamMembers array. Do NOT send a partial array -- GHL will replace the entire team with what you send. If you send only the new user, all existing users will be removed.

**Response (200):** Returns the updated calendar object.

---

## Delete Calendar

Remove a calendar.

```
DELETE /calendars/{calendarId}
```

**Path Parameters:**
- `calendarId` (required): Calendar ID

**CAUTION:** Deleting a calendar removes all associated appointments and settings. Confirm with the user before proceeding.

**Response (200):**
```json
{
  "succeeded": true
}
```

---

## Get Free Slots

Check available booking slots for a calendar.

```
GET /calendars/{calendarId}/free-slots
```

**Query Parameters:**
- `startDate` (required): Start of date range (Unix timestamp in milliseconds)
- `endDate` (required): End of date range (Unix timestamp in milliseconds)
- `timezone` (optional): IANA timezone string (e.g., "America/New_York")
- `userId` (optional): Filter slots for a specific team member

**Response (200):**
```json
{
  "slots": {
    "2025-03-17": [
      { "slot": "2025-03-17T09:00:00-04:00" },
      { "slot": "2025-03-17T09:30:00-04:00" },
      { "slot": "2025-03-17T10:00:00-04:00" }
    ]
  }
}
```

Useful for auditing: check if a calendar has any available slots in the next week to verify it's properly configured.

---

## Other Calendar Endpoints

**Calendar Groups:** Organize multiple calendars. GET/POST `/calendars/groups` with `locationId` param.

**Calendar Events:** Book/manage appointments. GET/POST/PUT/DELETE `/calendars/events` and `/calendars/events/{eventId}`. Event management is typically handled by booking flows, not manual operations.

**User Availability:** Override calendar hours per user. GET/POST/PUT/DELETE `/calendars/availability` and `/calendars/availability/{scheduleId}`. Use for reps with custom availability (e.g., 10am-6pm instead of default 9am-5pm).
