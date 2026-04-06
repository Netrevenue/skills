# GoHighLevel CRM Push

Read this file when executing Step 7 of the pipeline.

All GHL API operations — contact creation, search, upsert, opportunity management, custom fields, and tags — are handled by the **`gohighlevel` skill**. This reference only covers the NR-specific business logic for what to push and how to route leads.

---

## Pre-push filters

Apply these in order before touching GHL:

1. **Score threshold** — Skip if `setter_score < 50` or `score_tier == "dq"`. Log reason: "below_threshold".
2. **Contact data required** — The lead needs at least ONE of: `email`, `apollo_contact_email`, `ig_handle`, `phone`. Skip if none exist. Log reason: "no_contact_method".
3. **Dedup** — Use the gohighlevel skill's contact search (`POST /contacts/search`) to check for existing contacts by email. Skip if found. Log reason: "exists_in_ghl".

---

## Contact creation

Use the gohighlevel skill's contact upsert (`POST /contacts/upsert`) for each qualified lead. Upsert avoids duplicates if the dedup check missed an edge case.

**Field mapping:**

| Lead field | GHL contact field |
|---|---|
| name (first word) | firstName |
| name (remaining words) | lastName |
| email OR apollo_contact_email | email |
| phone | phone |
| website | website |
| — | source: "AI Pipeline" |

**Tags to apply** (use upsert's tags field):
- `{score_tier}` (e.g., "priority_a", "standard", "nurture")
- `{vertical}` (e.g., "tax_strategy", "business_coaching")
- `account:{assigned_account}` (e.g., "account:Britt")
- `channel:{primary_channel}` (e.g., "channel:linkedin")

**Custom fields** — Resolve field IDs first using the gohighlevel skill (`GET /locations/{locationId}/customFields?model=contact`), then set:

| Custom field name | Value |
|---|---|
| setter_score | `{setter_score}` |
| score_tier | `{score_tier}` |
| vertical | `{vertical}` |
| assigned_account | `{assigned_account}` |
| primary_channel | `{primary_channel}` |
| lead_source | `{sources joined with ", "}` |
| enrichment_notes | `{notes from scorecard}` |
| ig_handle | `{ig_handle}` |
| linkedin_url | `{linkedin_url OR apollo_contact_linkedin}` |
| apollo_industry | `{apollo_industry}` |
| apollo_headcount | `{apollo_headcount}` |
| apollo_revenue | `{apollo_revenue}` |
| sequence_step | "0" |

Omit empty email/phone fields — sending empty strings can cause GHL validation errors.

---

## Pipeline routing

After creating the contact, use the gohighlevel skill to create an opportunity (`POST /opportunities/`) linking the contact to the correct pipeline stage:

| Tier | Pipeline | Stage |
|------|----------|-------|
| priority_a (80-100) | LeadGen pipeline | New leads stage |
| standard (65-79) | LeadGen pipeline | New leads stage |
| nurture (50-64) | LeadGen pipeline | Nurture stage |

Use the gohighlevel skill to list pipelines first (`GET /opportunities/pipelines`) to resolve the correct pipeline and stage IDs.

---

## Dry run default

This step defaults to dry run — log what would be pushed without making API calls. Live pushes require explicit user confirmation ("push to GHL", "go live"). This matters because pushing creates real contacts that setters will act on.

---

## Push summary

After pushing, report:
- Pushed: count
- Skipped (dedup): count
- Skipped (DQ): count
- Skipped (no contact method): count
- Errors: count and details
