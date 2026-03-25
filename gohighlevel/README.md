# GHL Operations Skill

Manage GoHighLevel (GHL) sub-accounts via REST API for Netrevenue's client accounts.

## Overview

This skill enables AI agents to perform admin and operational tasks across multiple client GHL accounts. Each client account is a GHL "sub-account" (also called a "location") with its own Private Integration Token (PIT) for API access.

## Capabilities

- **User Management**: Add, remove, update users; manage roles and permissions
- **Calendar Management**: Create, update, delete calendars; assign/remove team members
- **Phone Number Management**: Search available numbers, purchase, assign to users
- **Pipeline & Opportunity Operations**: List pipelines, search opportunities, audit pipeline health
- **Account Configuration**: Manage sub-account settings, custom fields, location details

## Files

- **[SKILL.md](SKILL.md)**: Main skill file with routing logic, API fundamentals, and workflow patterns
- **[references/users.md](references/users.md)**: User management API endpoints
- **[references/calendars.md](references/calendars.md)**: Calendar management API endpoints
- **[references/phone-numbers.md](references/phone-numbers.md)**: Phone system API endpoints
- **[references/pipelines.md](references/pipelines.md)**: Pipelines and opportunities API endpoints
- **[references/account-config.md](references/account-config.md)**: Sub-account configuration and custom fields API endpoints

## Authentication

Uses Sub-Account level Private Integration Tokens (PIT). Each client account requires:
- **PIT token**: For API authentication
- **locationId**: The GHL sub-account ID
- **companyId**: Required for user operations (obtained from location endpoint)

## API Details

- **Base URL**: `https://services.leadconnectorhq.com`
- **Required Headers**: `Authorization: Bearer {pit_token}`, `Version: 2021-07-28`, `Content-Type: application/json`, `Accept: application/json`
- **Rate Limits**: 100 requests per 10 seconds per location, 200,000 per day per location

## Verification Status

All API endpoints have been verified against a live GHL test account. Request/response examples use real data from actual API calls.

## Common Workflows

**User Onboarding:**
1. Create user (references/users.md)
2. Assign to calendar(s) (references/calendars.md)
3. Purchase and assign phone number (references/phone-numbers.md)

**User Offboarding:**
1. Remove from calendar(s) (references/calendars.md)
2. Unassign phone number (references/phone-numbers.md)
3. Delete or deactivate user (references/users.md)

**Pipeline Auditing:**
1. List pipelines and stages (references/pipelines.md)
2. Search opportunities by status/stage/rep
3. Generate health reports (distribution, stale opps, unassigned)

## Known API Quirks

- Users list uses `GET /users/?locationId={id}` (query param)
- Users search requires both `companyId` and `locationId`
- Opportunities search uses `location_id` (snake_case), NOT `locationId`
- Phone number paths: `/phone-system/numbers/location/{locationId}/...`
- Phone assignment via `lcPhone` field in user object (not phone API)
- Delete user response field is `succeded` (GHL's typo)
- Calendar `teamMembers.isZoomAdded` is string `"true"`/`"false"`, not boolean
