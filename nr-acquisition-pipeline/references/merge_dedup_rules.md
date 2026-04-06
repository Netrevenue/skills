# Merge, Dedup & Filter Rules

Read this file when executing Step 3 of the pipeline. It contains the exact logic for combining scraper outputs into unified lead records.

## Table of Contents
- [Unified Lead Schema](#unified-lead-schema)
- [Domain Extraction](#domain-extraction)
- [Name Normalization](#name-normalization)
- [Merge Order and Rules](#merge-order-and-rules)
- [Cross-Run Dedup (Lead Ledger)](#cross-run-dedup-lead-ledger)
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
- Match to leads by `domain` field from scraper output (matches against extracted domain of lead's `website`)
- Backfill into existing leads: `email` (first non-generic from `emails` array), `phone` (first from `phones` array, then `phonesUncertain`), `linkedin_url` (first from `linkedIns` array), `ig_handle` (first from `instagrams` array)
- Set `has_booking_link: true` if any URL in the `scrapedUrls` array matches booking keywords (calendly.com, acuityscheduling.com, hubspot.com/meetings, book-a-call, consultation, schedule, etc.)

---

## Cross-Run Dedup (Lead Ledger)

The pipeline maintains a persistent file called `processed_leads.jsonl` that tracks every lead processed across all runs. This prevents re-scraping, re-enriching, and re-scoring leads that were already handled in previous weeks.

### Ledger location

Store the ledger at a stable path relative to the pipeline skill's working directory: `data/processed_leads.jsonl`. Create the `data/` directory and file on the first run if they don't exist.

### Ledger entry schema

Each line in the JSONL file is one lead:

```json
{
  "domain": "pinnacleadvisors.com",
  "email": "info@pinnacleadvisors.com",
  "ig_handle": "pinnacletax",
  "place_id": "ChIJ...",
  "name_normalized": "pinnacle tax advisors",
  "run_date": "2026-04-06",
  "outcome": "pushed",
  "setter_score": 76,
  "score_tier": "standard"
}
```

All fields except `run_date` and `outcome` can be null — a lead only needs one identifier to be matched.

### Outcome values

| Outcome | Meaning |
|---|---|
| `pushed` | Lead was pushed to GHL |
| `dq_icp` | Disqualified by ICP classifier |
| `dq_filter` | Caught by hard disqualifier keyword filter |
| `below_threshold` | Scored below 50 or tier was dq |
| `no_contact` | No email, phone, or ig_handle — couldn't reach them |
| `nurture` | Pushed to GHL nurture stage |
| `exists_in_ghl` | Already existed in GHL at push time |

### When to check (read)

After the cross-source merge (Step 3, after merge order is applied) but BEFORE NR client exclusion and the hard DQ filter. Load the full ledger into memory and build lookup sets:

1. `known_domains` — set of all non-null `domain` values
2. `known_emails` — set of all non-null `email` values (lowercased)
3. `known_ig_handles` — set of all non-null `ig_handle` values (lowercased)
4. `known_place_ids` — set of all non-null `place_id` values

A merged lead is a **ledger match** if ANY of these are true:
- Its extracted domain is in `known_domains`
- Its `email` (lowercased) is in `known_emails`
- Its `ig_handle` (lowercased) is in `known_ig_handles`
- Its `place_id` is in `known_place_ids`

Remove ledger matches from the working set. Log how many were removed.

### When to write (append)

At the END of the pipeline run (after Step 7 completes or after the last executed step in a partial run), append one ledger entry per lead that made it past the merge step — including leads that were later DQ'd, scored low, or skipped at push time. This ensures those leads are recognized on the next run and not re-processed.

Do NOT write ledger entries for leads that were removed by the ledger check itself (that would create duplicates in the file).

### Re-evaluation window

Some leads that scored poorly may improve over time (e.g., they start running ads, gain followers). To allow re-evaluation, ignore ledger entries older than **90 days** when building the lookup sets. This means a lead that was DQ'd or scored low 3+ months ago will flow through the pipeline again and get a fresh classification.

Leads with outcome `pushed` or `nurture` are never re-evaluated regardless of age — they're already in the CRM.

### Ledger maintenance

The JSONL file will grow over time. Periodically (every 10 runs or when the file exceeds 50,000 lines), compact it:
1. Remove entries older than 90 days that have outcome other than `pushed`, `nurture`, or `exists_in_ghl`
2. Deduplicate entries for the same lead (keep the most recent)
3. Write the compacted result back to the same file

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
