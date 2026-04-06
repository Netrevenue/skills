---
name: nr-acquisition-pipeline
description: Runs the full Net Revenue client acquisition pipeline — scraping leads from Google Maps, Meta Ads, and Instagram via Apify, extracting contact info from websites, merging and deduplicating across sources, enriching with Apollo.io company data, classifying and scoring leads using the nr-icp-classifier and nr-setter-scorecard skills, and pushing qualified leads to GoHighLevel CRM. Use this skill whenever the user asks to "run the pipeline", "scrape leads", "find new leads", "run acquisition", "generate leads", "run the lead gen", or any variation of executing the NR client acquisition workflow. Also use when the user wants to run individual pipeline stages like "just scrape Google Maps" or "enrich and score the leads we already have" — this skill knows how to run partial pipelines too.
metadata: {"openclaw":{"emoji":"🦞"}}
---

# NR Client Acquisition Pipeline

This is the master orchestrator for Net Revenue's automated lead generation pipeline. It coordinates 7 steps that take raw search queries and turn them into scored, assigned leads in the GoHighLevel CRM.

## Pipeline overview

```
Step 1: Scrape (Google Maps + Meta Ads + IG Profiles — parallel)
Step 2: Extract contact info from websites
Step 3: Merge, deduplicate, and filter
Step 4: Enrich with Apollo.io company data
Step 5: Classify leads (nr-icp-classifier skill)
Step 6: Score and assign leads (nr-setter-scorecard skill)
Step 7: Push qualified leads to GoHighLevel CRM
```

Each step builds on the output of the previous one. Steps can be skipped if the user asks to start partway through (e.g., "skip scraping, use the leads we already have").

## When to use partial runs

The user may not always want the full pipeline. Common patterns:

- **"Run the full pipeline"** — Execute all 7 steps
- **"Run a dry run"** — All 7 steps but skip the actual GHL push (log what would be pushed)
- **"Just scrape"** — Steps 1-2 only
- **"Skip scraping, enrich what we have"** — Steps 3-7, using previously scraped data
- **"Skip Apollo"** — Run everything except Step 4
- **"Re-score the leads"** — Steps 5-6 only, on previously enriched data
- **"Test with 5 leads"** — Full pipeline but limit scraper results (set max_results to 5)

Always confirm what the user wants before starting, especially for live GHL pushes.

## Step 1: Scrape leads from 3 sources

Run these three scrapers in parallel using Apify. Read `references/scraper_configs.md` for the exact actor IDs, search queries, and parameters.

**Google Maps** — `compass~crawler-google-places`
- 8 search queries targeting professional service firms (tax, CPA, legal, wealth, CFO, funding)
- 200 results per query, US only
- Key signals: reviews, rating, website, category

**Meta Ad Library** — `curious_coder~facebook-ads-library-scraper`
- 7 search queries targeting active advertisers in NR ICP verticals
- 100 results per query, US only
- Key signal: is_running_ads = true

**Instagram Profiles** — `apify~instagram-profile-scraper`
- 15 seed accounts (NR client handles), 500 followers each
- Residential proxy required
- Key signals: bio, follower_count, external_url

After each scraper completes, normalize the output fields according to the mappings in the reference file. Each record should include a `source` field identifying where it came from.

## Step 2: Extract contact info from websites

Collect all website URLs discovered in Step 1 (from Google Maps website fields, Meta Ads page URLs, and IG bio external_urls). Deduplicate URLs, then scrape them for contact info.

**Actor:** `vdrmota~contact-info-scraper`
- Batch at 50 URLs per run
- maxDepth 2, maxPagesPerDomain 5
- Extract: emails, phones, LinkedIn URLs, Instagram handles, booking links

Booking links are a strong ICP signal — check for keywords like `calendly.com`, `acuityscheduling.com`, `hubspot.com/meetings`, `book-a-call`, `consultation`, `schedule`.

## Step 3: Merge, deduplicate, and filter

This is where the 4 data streams combine into unified lead records. Read `references/merge_dedup_rules.md` for the full logic.

The short version:
1. **Merge** Google Maps leads first (primary), then overlay Meta Ads (adds is_running_ads flag), then IG Profiles (backfills social data), then email scraper data (backfills contact info)
2. **Match** leads across sources by domain (extracted from URLs) and ig_handle
3. **Exclude** leads whose ig_handle matches an existing NR client (61 handles — see `nr-icp-classifier/references/nr_client_handles.md`)
4. **Filter** leads matching hard disqualifier keywords (crypto, forex, MLM, fitness coach, etc.)

Each merged lead gets a UUID `lead_id` and a `sources` array tracking which scrapers contributed data.

Report: how many raw leads → how many after merge → how many after dedup → how many after DQ filter.

## Step 4: Enrich with Apollo.io

Read `references/apollo_enrichment.md` for the full API details.

For each unique domain across all leads:
1. Call Apollo's organization enrichment endpoint to get headcount, revenue, industry, keywords, founded year, technologies, funding
2. Optionally call person matching endpoint to get contact email and title (skip on small test runs)
3. Cache results persistently so domains aren't re-queried on future runs
4. Merge Apollo data back into lead records, set `apollo_enriched: true/false`

Rate limiting is critical — stay under 180 calls/hour, pause 60 seconds between batches of 50, sleep 1.5 seconds between requests.

If the user doesn't have an Apollo API key or wants to skip this step, the pipeline continues without enrichment. Classification and scoring can work without Apollo data — they just produce less precise results.

## Step 5: Classify leads

For each lead, invoke the **`nr-icp-classifier`** skill. Pass the lead data as a JSON object and receive back:

```json
{
  "vertical": "tax_strategy",
  "offer_type": "service",
  "estimated_ticket": "10k_25k",
  "audience_tier": "unknown",
  "demand_signals": ["high_reviews", "has_team", "booking_calls"],
  "disqualify_reason": null
}
```

Merge classification fields into each lead record. Leads classified as `disqualified` stay in the dataset (they get scored as 0 in the next step) — don't remove them here.

Process up to 10 leads concurrently to stay within API rate limits.

## Step 6: Score and assign leads

For each classified lead, invoke the **`nr-setter-scorecard`** skill. Pass both the classification output and the lead data, and receive back:

```json
{
  "setter_score": 76,
  "score_breakdown": {"social_proof": 16, "review_volume": 16, "audience_size": 12, "offer_fit": 20, "demand_indicator": 12},
  "score_tier": "standard",
  "notes": "CPA firm with 65+ reviews...",
  "assigned_account": "Britt",
  "secondary_account": "Wade",
  "primary_channel": "linkedin",
  "is_partnership_track": false
}
```

Merge scoring and assignment fields into each lead. Sort leads by score descending.

Report tier distribution: how many priority_a, standard, nurture, dq.

## Step 7: Push to GoHighLevel CRM

Use the **`gohighlevel` skill** for all GHL API operations — contact creation/upsert, search, opportunity management, custom fields, and tags. Read `references/ghl_push.md` for the NR-specific field mapping, tag conventions, and pipeline routing rules.

**This step defaults to dry run.** Live pushes require the user to explicitly confirm (say "push to GHL", "go live", or "run --live"). This safeguard exists because pushing creates real contacts in the CRM that setters will act on.

Before pushing each lead:
1. Skip if score < 50 or tier is dq
2. Skip if no contact method (email, phone, or ig_handle)
3. Search GHL for existing contact by email — skip duplicates

For leads that pass all filters:
- Resolve custom field IDs via gohighlevel skill (list custom fields, match by name)
- Upsert GHL contact with tags (tier, vertical, account, channel) and custom fields (score, notes, Apollo data)
- Create an opportunity linking the contact to the correct pipeline stage: LeadGen for priority_a and standard, Nurture stage for nurture tier

Report: how many pushed, how many skipped (by reason), any errors.

## Cost awareness

The pipeline spends money on API calls. For context:
- **Apify:** Billed by compute units per actor run
- **Apollo.io:** 180 free calls/hour, cached across runs
- **Anthropic (Claude):** ~$0.003/1K input + $0.015/1K output for classification + scoring. Alert if a single run exceeds $30.
- **GHL:** Included in subscription

For testing, use small datasets (max_results 5-10) and dry run mode. This keeps costs near zero while validating the pipeline works end to end.

## Run summary

At the end of every run (full or partial), produce a summary:

```
Pipeline Run Summary — YYYY-MM-DD
──────────────────────────────────
Raw leads scraped:     X (GM: X, Meta: X, IG: X)
After merge:           X
After NR client dedup: X (removed X)
After DQ filter:       X (removed X)
Apollo enriched:       X / X
Classified:            X
Score distribution:    Priority A: X | Standard: X | Nurture: X | DQ: X
Pushed to GHL:         X (dry run: Y/N)
  Skipped (dedup):     X
  Skipped (no contact):X
  Errors:              X
Estimated API cost:    $X.XX
Duration:              Xm Xs
```

## Guardrails

- Confirm with the user before any live GHL push. "Dry run" is the safe default.
- Don't modify scraper search queries without the user's approval — they're tuned to NR's ICPs.
- If a step fails, stop and report the error. Don't silently skip it — the user needs to know.
- If Apify returns empty results for a scraper, warn but continue with the other sources. One failing scraper shouldn't block the whole pipeline.
- Keep Apollo rate limits conservative. Getting rate-limited mid-run wastes time and cache progress.
