# GHL Conversations API Reference

Search conversations and retrieve message history for contacts.

---

## Search Conversations by Contact

```
GET /conversations/search
```

**Query params:**
| Param | Required | Description |
|---|---|---|
| `locationId` | Yes | GHL location ID |
| `contactId` | No | Filter to a specific contact |
| `limit` | No | Max results (default 20) |

**Response:**
```json
{
  "conversations": [
    {
      "id": "ro3gtFVHmBeuXHttPe2T",
      "contactId": "xNOerPpgA18MStNTRvrc",
      "locationId": "...",
      "lastMessageDate": "2026-04-22T21:04:07.014Z",
      "type": "TYPE_PHONE",
      "unreadCount": 0
    }
  ]
}
```

---

## Get Messages in a Conversation

```
GET /conversations/{conversationId}/messages
```

**Query params:**
| Param | Required | Description |
|---|---|---|
| `limit` | No | Max messages per page (max 100) |
| `lastMessageId` | No | Pagination cursor — pass the `lastMessageId` from the previous response to get the next page |

**Response structure:**
```json
{
  "messages": {
    "lastMessageId": "msg_abc",
    "nextPage": true,
    "messages": [ ... ]
  }
}
```

> **Note:** The response nests messages inside `messages.messages`. Pagination uses `messages.lastMessageId` as the cursor and `messages.nextPage` to determine if more pages exist.

---

## Message Object Fields

```json
{
  "id": "msg_abc",
  "type": 2,
  "messageType": "TYPE_SMS",
  "direction": "outbound",
  "userId": "UdBJYEGU0xoSFcavXGcP",
  "contactId": "xNOerPpgA18MStNTRvrc",
  "conversationId": "ro3gtFVHmBeuXHttPe2T",
  "dateAdded": "2026-04-09T14:23:00.000Z",
  "source": "app",
  "body": "Hey, just confirming our call tomorrow...",
  "from": "+14155551234",
  "to": "+14155556789"
}
```

---

## Message Type Codes

The `type` field is an **integer**. The `messageType` field is a string label.

| `type` (int) | `messageType` (string) | Category | Description |
|---|---|---|---|
| 1 | `TYPE_CALL` | Call | Phone call (inbound or outbound) |
| 2 | `TYPE_SMS` | SMS | Standard SMS message |
| 3 | `TYPE_EMAIL` | Email | Email message |
| 6 | `TYPE_LIVE_CHAT` | Chat | Live chat / widget message |
| 20 | `TYPE_WORKFLOW_SMS` | Automated | SMS sent by a workflow |
| 22 | `TYPE_CUSTOM_PROVIDER_SMS` | SMS | SMS via custom provider (e.g. LeadConnector) |
| 23 | `TYPE_CUSTOM_PROVIDER_CALL` | Call | Call via custom provider |
| 25 | `TYPE_ACTIVITY_CONTACT` | Activity | Contact activity log (e.g. DnD enabled) |
| 28 | `TYPE_ACTIVITY_OPPORTUNITY` | Activity | Opportunity activity (e.g. status changed) |
| 31 | `TYPE_ACTIVITY_APPOINTMENT` | Activity | Appointment activity log |

**For filtering manual outbound SMS/Call messages, use these types:**
- SMS types: `2` (TYPE_SMS), `22` (TYPE_CUSTOM_PROVIDER_SMS)
- Call types: `1` (TYPE_CALL), `23` (TYPE_CUSTOM_PROVIDER_CALL)

**Do NOT count these as manual contacts:**
- Type `3` (Email)
- Type `20` (Workflow SMS — automated)
- Types `25`, `28`, `31` (Activity logs)
- Any type not listed above

---

## Distinguishing Manual vs Automated Messages

- `userId` is populated → sent by a real user. Cross-check against the account's user list to confirm they're a known team member.
- `userId` is null or absent → sent by an automation or workflow. **Discard — does not count as a manual contact attempt.**
- When in doubt, if the `userId` does not appear in the users list, treat the message as automated.

---

## GHLClient Usage

The `GHLClient` class provides a convenience method for conversation search:

```python
convs = ghl.get_conversations(contact_id="abc123")
```

For retrieving messages, use the raw HTTP call:
```python
import requests
url = f"{ghl.base_url}/conversations/{conv_id}/messages"
r = requests.get(url, headers=ghl.headers, params={"limit": 100})
data = r.json()
messages = data["messages"]["messages"]
last_id = data["messages"]["lastMessageId"]
has_more = data["messages"]["nextPage"]
```
