# Merge, Dedup & Filter Rules

Read this file when executing Step 3 of the pipeline. It contains the exact logic for combining scraper outputs into unified lead records.

## Table of Contents
- [Unified Lead Schema](#unified-lead-schema)
- [Domain Extraction](#domain-extraction)
- [Name Normalization](#name-normalization)
- [Merge Order and Rules](#merge-order-and-rules)
- [NR Client Exclusion](#nr-client-exclusion)
- [Hard Disqualifier Filter](#hard-disqualifier-filter)

---

## Unified Lead Schema

Every lead record produced by the merge step should have this structure:

```json
{
  "lead_id": "uuid",
  "name": "",
  "email": "",
  "phone": "",
  "website": "",
  "linkedin_url": "",
  "ig_handle": "",
  "bio": "",
  "follower_count": null,
  "review_count": null,
  "rating": null,
  "category": "",
  "has_booking_link": false,
  "is_running_ads": false,
  "sources": [],
  "scraped_date": "YYYY-MM-DD"
}
```

---

## Domain Extraction

Used for matching leads across sources. Given a URL:
1. Parse with standard URL parser (add `https://` if no scheme)
2. Extract the hostname
3. Lowercase it
4. Strip `www.` prefix
5. Remove port number if present

Example: `https://www.PinnacleAdvisors.com:443/about` → `pinnacleadvisors.com`

---

## Name Normalization

Used for fuzzy matching business names:
1. Lowercase
2. Remove all non-alphanumeric characters (except spaces)
3. Remove common business suffixes: `llc`, `inc`, `corp`, `ltd`, `pllc`, `pc`, `pa`, `llp`
4. Collapse extra whitespace

Example: `"Pinnacle Tax Advisors, LLC"` → `"pinnacle tax advisors"`

---

## Merge Order and Rules

Process sources in this order. Earlier sources take priority for field values — later sources backfill empty fields.

### 1. Google Maps (primary source)
- Create initial lead record with all GM fields
- Link to email scraper data via domain matching
- Set `is_running_ads: false`
- Set `sources: ["google_maps"]`
- Index by domain AND normalized name

### 2. Meta Ads (merge or create)
- Extract domain from `website` field (fallback to `page_url`)
- If domain matches existing lead:
  - Set `is_running_ads: true`
  - Append `"meta_ad_library"` to sources array
- If no match: create new lead with Meta Ads data, set `is_running_ads: true`

### 3. IG Profiles (merge or create)
- Match by: `ig_handle` first, then domain of `external_url`
- If match found:
  - Backfill `ig_handle`, `bio`, `follower_count` (only if existing value is empty/null)
  - Append `"ig_followers"` to sources
- If no match: create new lead from IG profile data

### 4. Email Scraper Data (backfill only)
- Match to leads by domain
- Backfill into existing leads: `email` (first from emails_found), `phone` (first from phones_found), `linkedin_url` (first from linkedin_urls_found), `ig_handle` (first from instagram_handles_found)
- Set `has_booking_link: true` if `booking_link_found` is non-null

---

## NR Client Exclusion

After merging, remove any lead whose `ig_handle` matches an existing NR client (case-insensitive comparison). The full list of 61 handles is in the `nr-icp-classifier` skill's reference file: `nr-icp-classifier/references/nr_client_handles.md`.

Log how many leads were removed and which handles matched.

---

## Hard Disqualifier Filter

After NR client exclusion, scan each lead's `bio`, `category`, and `name` fields (lowercased, concatenated) for these keywords:

```
ecommerce, e-commerce, dropshipping, crypto, cryptocurrency,
forex, day trading, daytrading, realtor, real estate agent,
mlm, network marketing, affiliate, fitness coach, personal trainer,
weight loss, bodybuilding, keto, supplement
```

Any keyword match → remove the lead. Log which keyword triggered each removal.

This filter catches the most obvious disqualifiers early, before expensive API calls (Apollo, Claude). The `nr-icp-classifier` skill does more nuanced classification later — this is just the first pass to reduce the dataset.
