---
name: nr-acquisition-pipeline
description: Runs the full Net Revenue client acquisition pipeline — scraping leads from Google Maps, Meta Ads, and Instagram via Apify, extracting contact info from websites, merging and deduplicating across sources, enriching with Apollo.io company data, classifying and scoring leads using the nr-icp-classifier and nr-setter-scorecard skills, and pushing qualified leads to GoHighLevel CRM. Use this skill whenever the user asks to "run the pipeline", "scrape leads", "find new leads", "run acquisition", "generate leads", "run the lead gen", or any variation of executing the NR client acquisition workflow. Also use when the user wants to run individual pipeline stages like "just scrape Google Maps" or "enrich and score the leads we already have" — this skill knows how to run partial pipelines too.
metadata: {"openclaw":{"emoji":"🦞"}}
---

# NR Client Acquisition Pipeline

This is the master orchestrator for Net Revenue's automated lead generation pipeline. It coordinates 8 steps that take raw search queries and turn them into scored, assigned leads in the GoHighLevel CRM.

**You are the orchestrator.** Your job is to manage data flow, spawn sub-agents for heavy lifting, and report progress. You should NOT do the scraping, enrichment, classification, scoring, or CRM pushing yourself — delegate each of those to a focused sub-agent that handles one job and returns results.

## Pipeline overview

```
Step 1-2: Scrape + extract contacts       → sub-agent: scraper
Step 3:   Merge, dedup, filter             → orchestrator (you)
Step 4:   Enrich with Apollo.io            → sub-agent: enricher
Step 5:   Classify leads                   → sub-agents: classifiers (batched)
Step 6:   Score and assign leads           → sub-agents: scorers (batched)
Step 7:   Push to GoHighLevel CRM          → sub-agent: crm-pusher
Step 8:   Update lead ledger               → orchestrator (you)
```

## Data flow between steps

Each step writes its output to a JSON file. Store all run data in `data/runs/YYYY-MM-DD/`:

```
data/runs/2026-04-06/
├── step1_raw_leads.json            # Combined normalized scraper output
├── step2_contact_info.json         # Website contact scraper output
├── step3_merged_leads.json         # Unified leads after merge + dedup + filter
├── step4_enriched_leads.json       # After Apollo enrichment
├── step5_classified_leads.json     # After ICP classification
├── step6_scored_leads.json         # After scoring + assignment (final dataset)
├── step7_push_report.json          # GHL push results (or dry run log)
└── run_summary.txt                 # Human-readable summary
```

Write each file immediately after completing the step. If a step fails, you have the previous step's output to retry from. Partial runs can resume from any step by reading the last completed file.

## When to use partial runs

The user may not always want the full pipeline. Common patterns:

- **"Run the full pipeline"** — Execute all 8 steps
- **"Run a dry run"** — All 8 steps but skip the actual GHL push (log what would be pushed)
- **"Just scrape"** — Steps 1-2 only
- **"Skip scraping, enrich what we have"** — Steps 3-8, using previously scraped data
- **"Skip Apollo"** — Run everything except Step 4
- **"Re-score the leads"** — Steps 5-6 only, on previously enriched data
- **"Test with 5 leads"** — Full pipeline but limit scraper results (use dry run config from scraper_configs.md)

Always confirm what the user wants before starting, especially for live GHL pushes.

---

## Sub-agent architecture

The pipeline uses sub-agents to keep each task focused and prevent the orchestrator from getting bogged down in execution details. Each sub-agent gets a clear, bounded task with specific inputs and expected outputs.

**Why sub-agents matter here:**
- Scraping involves waiting for Apify runs — a sub-agent handles the start/poll/collect loop without blocking your context
- Classification and scoring of hundreds of leads would overwhelm a single context — splitting into batches across parallel sub-agents is faster and more reliable
- Each sub-agent focuses on ONE job with ONE skill's reference docs, so it doesn't get lost reading irrelevant material

**How to spawn sub-agents:**
- Give each sub-agent a clear task description, the file paths it needs to read, and the file path where it should save its output
- For steps that use a skill (classification, scoring, GHL push), tell the sub-agent which skill to use
- Sub-agents should save results to the data directory so you can read them back
- Launch independent sub-agents in parallel when possible (e.g., all classification batches at once)

---

## Step 1-2: Scrape and extract contacts

Spawn a single sub-agent to handle all scraping. This sub-agent manages the Apify start/collect lifecycle and normalizes the output.

**Sub-agent prompt:**
```
You are the scraping sub-agent for the NR acquisition pipeline.

Your job:
1. Read references/apify_api.md for the apify tool actions (start, collect)
2. Read references/scraper_configs.md for the exact input payloads and output field mappings

Execute:
1. Start all 3 scrapers in parallel using the apify tool:
   - Google Maps: compass~crawler-google-places
   - Meta Ad Library: curious_coder~facebook-ads-library-scraper
   - Instagram Profiles: apify~instagram-profile-scraper
   Copy the verified input payloads from scraper_configs.md exactly.

2. Collect results from all 3 runs. If any are pending, wait 30 seconds and collect again.

3. Normalize output fields using the mapping tables in scraper_configs.md.
   Add a "source" field to each record.

4. Collect all website URLs from results, deduplicate by domain, and run
   the contact info scraper (vdrmota~contact-info-scraper) in batches of 50.

5. Save results:
   - {run_dir}/step1_raw_leads.json (normalized leads from all 3 scrapers)
   - {run_dir}/step2_contact_info.json (contact scraper results)

Report: how many results per scraper, any failures, total URLs scraped for contacts.
```

Replace `{run_dir}` with the actual path (e.g., `data/runs/2026-04-06/`).

Wait for this sub-agent to complete, then read `step1_raw_leads.json` and `step2_contact_info.json` to proceed.

## Step 3: Merge, deduplicate, and filter

**Execute this step yourself** — it's data manipulation that needs context from the reference files.

Read `references/merge_dedup_rules.md` for the full logic. The short version:

1. **Merge** Google Maps leads first (primary), then overlay Meta Ads (adds is_running_ads flag), then IG Profiles (backfills social data), then email scraper data (backfills contact info)
2. **Match** leads across sources by domain (extracted from URLs) and ig_handle
3. **Cross-run dedup** — Load `data/processed_leads.jsonl` (the lead ledger) and remove any leads already processed in a previous run. Match by domain, email, ig_handle, or place_id. Entries older than 90 days are ignored for non-CRM outcomes. See `references/merge_dedup_rules.md` for full ledger logic.
4. **Exclude** leads whose ig_handle matches an existing NR client (61 handles — see `nr-icp-classifier/references/nr_client_handles.md`)
5. **Filter** leads matching hard disqualifier keywords (crypto, forex, MLM, fitness coach, etc.)

Each merged lead gets a UUID `lead_id` and a `sources` array tracking which scrapers contributed data.

Save to `step3_merged_leads.json`.

Report: how many raw leads → after merge → after ledger dedup → after NR client exclusion → after DQ filter.

## Step 4: Enrich with Apollo.io

Spawn a sub-agent to handle Apollo API calls. This isolates the rate-limiting logic.

**Sub-agent prompt:**
```
You are the Apollo enrichment sub-agent for the NR acquisition pipeline.

Your job:
1. Read references/apollo_enrichment.md for the API endpoints, rate limits, and field mapping
2. Read {run_dir}/step3_merged_leads.json to get the leads

Execute:
1. Extract unique domains from all leads
2. Check the persistent cache (data/apollo_cache.json) — skip domains already cached
3. For each uncached domain, call Apollo's organization enrichment endpoint
   - Stay under 180 calls/hour
   - Pause 60 seconds between batches of 50
   - Sleep 1.5 seconds between individual requests
4. Update the cache with new results (including empty results for domains with no data)
5. Merge Apollo data back into each lead record, set apollo_enriched: true/false
6. Save enriched leads to {run_dir}/step4_enriched_leads.json

Report: how many domains enriched, how many from cache, any rate limit issues.
```

If the user wants to skip Apollo, copy `step3_merged_leads.json` to `step4_enriched_leads.json` and continue.

## Step 5: Classify leads (batched sub-agents)

Split the leads into batches and spawn parallel sub-agents, each classifying its batch using the `nr-icp-classifier` skill.

**Batch size:** ~25 leads per sub-agent. For 200 leads, spawn 8 sub-agents. For 50 leads, spawn 2. Minimum 1, maximum 10.

**Sub-agent prompt (one per batch):**
```
You are a classification sub-agent for the NR acquisition pipeline.
Use the nr-icp-classifier skill to classify each lead.

Your batch: leads {start_idx} through {end_idx}
Read your leads from: {run_dir}/step4_enriched_leads.json (indices {start_idx} to {end_idx})

For each lead, classify it and capture the response JSON containing:
vertical, offer_type, estimated_ticket, audience_tier, demand_signals, disqualify_reason

Save your batch results to: {run_dir}/step5_batch_{batch_num}.json
Format: JSON array of objects, each with the original lead data plus classification fields merged in.

Process leads one at a time. Do not skip any leads. Do not try to classify
leads yourself — invoke the nr-icp-classifier skill for each one.
```

After all classification sub-agents complete, combine the batch files into `step5_classified_leads.json`. Verify lead count matches — no leads should be lost.

## Step 6: Score and assign leads (batched sub-agents)

Same pattern as Step 5, but using the `nr-setter-scorecard` skill.

**Sub-agent prompt (one per batch):**
```
You are a scoring sub-agent for the NR acquisition pipeline.
Use the nr-setter-scorecard skill to score each classified lead.

Your batch: leads {start_idx} through {end_idx}
Read your leads from: {run_dir}/step5_classified_leads.json (indices {start_idx} to {end_idx})

For each lead, score it and capture the response JSON containing:
setter_score, score_breakdown, score_tier, notes, assigned_account,
secondary_account, primary_channel, is_partnership_track

Save your batch results to: {run_dir}/step6_batch_{batch_num}.json
Format: JSON array of objects, each with the original lead data plus scoring fields merged in.

Process leads one at a time. Do not skip any leads. Do not try to score
leads yourself — invoke the nr-setter-scorecard skill for each one.
```

After all scoring sub-agents complete, combine batch files into `step6_scored_leads.json`. Sort by `setter_score` descending.

Report tier distribution: how many priority_a, standard, nurture, dq.

## Step 7: Push to GoHighLevel CRM

Spawn a sub-agent to handle the GHL push using the `gohighlevel` skill.

**This step defaults to dry run.** Live pushes require the user to explicitly confirm (say "push to GHL", "go live", or "run --live"). This safeguard exists because pushing creates real contacts that setters will act on.

**Sub-agent prompt:**
```
You are the CRM push sub-agent for the NR acquisition pipeline.
Use the gohighlevel skill for all GHL API operations.

Read references/ghl_push.md for NR-specific field mapping, tag conventions, and pipeline routing.
Read {run_dir}/step6_scored_leads.json for the scored leads.

Mode: {dry_run | live}

For each lead:
1. Skip if setter_score < 50 or score_tier == "dq" (log: "below_threshold")
2. Skip if no email, phone, or ig_handle (log: "no_contact_method")
3. Search GHL for existing contact by email — skip if found (log: "exists_in_ghl")
4. If live mode: upsert contact with tags and custom fields per ghl_push.md,
   then create opportunity in the correct pipeline stage
5. If dry run: log what would be pushed without making API calls

Save results to: {run_dir}/step7_push_report.json
Format: { "pushed": [...], "skipped": [...], "errors": [...], "mode": "dry_run|live" }

Report: pushed count, skipped count by reason, any errors.
```

## Step 8: Update lead ledger

**Execute this step yourself** — it's a simple file append.

After Step 7 completes (or after the last executed step in a partial run), append all leads that made it past the merge step to `data/processed_leads.jsonl`.

Each entry captures: domain, email, ig_handle, place_id, normalized name, run date, outcome, score, and tier. See `references/merge_dedup_rules.md` for the full schema and maintenance rules.

Set outcome based on what happened to each lead:
- `pushed` — successfully pushed to GHL
- `nurture` — pushed to GHL nurture stage
- `exists_in_ghl` — skipped because already in GHL
- `dq_icp` — disqualified by ICP classifier
- `dq_filter` — caught by hard disqualifier keyword filter
- `below_threshold` — scored below 50
- `no_contact` — no email, phone, or ig_handle

---

## Cost awareness

The pipeline spends money on API calls. For context:
- **Apify:** Billed by compute units per actor run
- **Apollo.io:** 180 free calls/hour, cached across runs
- **Anthropic (Claude):** Classification + scoring sub-agents. Alert if a single run exceeds $30.
- **GHL:** Included in subscription

For testing, use the dry run config from `scraper_configs.md` (1 query per scraper, 5 results max). This keeps costs near zero while validating the pipeline end to end.

## Run summary

At the end of every run (full or partial), produce a summary:

```
Pipeline Run Summary — YYYY-MM-DD
──────────────────────────────────
Raw leads scraped:     X (GM: X, Meta: X, IG: X)
After merge:           X
After ledger dedup:    X (removed X — already processed in prior runs)
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
- If a sub-agent fails, report the error to the user. Don't silently skip the step.
- If Apify returns empty results for a scraper, warn but continue with the other sources.
- Keep Apollo rate limits conservative. Getting rate-limited mid-run wastes time and cache progress.
- After combining batch files (Steps 5-6), always verify the total lead count matches what went in. If leads are missing, report it before continuing.
