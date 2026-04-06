# Apify Plugin Tool Reference

This file documents how to use the `apify` tool provided by the OpenClaw Apify plugin (`@apify/apify-openclaw-plugin`). Use this tool for all scraping operations in Steps 1 and 2 of the pipeline.

The plugin handles auth, polling, and result fetching. You do NOT need to make raw API calls or use curl.

---

## Tool: `apify`

The plugin registers a single tool called `apify` with three actions: `discover`, `start`, and `collect`.

### Workflow

```
start (fire off runs) → do other work → collect (get results)
```

For this pipeline, you already know the actor IDs and input schemas (documented in `scraper_configs.md`), so skip `discover` and go straight to `start` → `collect`.

---

## Action: `start`

Launches an Apify Actor run. Returns immediately with run metadata — does NOT wait for the run to finish.

**Call:**
```json
{
  "action": "start",
  "actorId": "compass~crawler-google-places",
  "input": {
    "searchStringsArray": ["tax advisory firm"],
    "countryCode": "us",
    "languageCode": "en",
    "maxCrawledPlaces": 5,
    "proxyConfig": {"useApifyProxy": true}
  }
}
```

**Response:**
```json
{
  "runs": [
    {
      "runId": "ZWNLDbfsa0R3syEgL",
      "actorId": "compass~crawler-google-places",
      "datasetId": "TVc4fv8mx1Ewfyi1S",
      "status": "RUNNING"
    }
  ]
}
```

Save the entire `runs` array — you pass it to `collect` later.

---

## Action: `collect`

Polls run status and returns results for completed runs. Pass the `runs` array from `start`.

**Call:**
```json
{
  "action": "collect",
  "runs": [
    {
      "runId": "ZWNLDbfsa0R3syEgL",
      "actorId": "compass~crawler-google-places",
      "datasetId": "TVc4fv8mx1Ewfyi1S"
    }
  ]
}
```

**Response:**
```json
{
  "completed": [
    {
      "runId": "ZWNLDbfsa0R3syEgL",
      "actorId": "compass~crawler-google-places",
      "status": "SUCCEEDED",
      "items": [ ...dataset results... ]
    }
  ],
  "pending": []
}
```

If `pending` is non-empty, wait 30 seconds and call `collect` again with the pending runs. Repeat until all runs are in `completed` or have failed.

---

## Action: `discover`

Use `discover` only if you need to look up an unknown actor or check an actor's input schema. For this pipeline, the actors and schemas are already documented in `scraper_configs.md`, so you should not need `discover` during normal runs.

**Search for actors:**
```json
{
  "action": "discover",
  "query": "google maps scraper"
}
```

**Get an actor's input schema:**
```json
{
  "action": "discover",
  "actorId": "compass~crawler-google-places"
}
```

---

## Pipeline Execution Pattern

### Step 1: Start all 3 scrapers in parallel

Fire off all three `start` calls. Each returns a `runs` array. Combine them into one array for collecting.

**Google Maps:**
```json
{
  "action": "start",
  "actorId": "compass~crawler-google-places",
  "input": {
    "searchStringsArray": [
      "tax advisory firm",
      "tax strategy consultant",
      "CPA firm business advisory",
      "estate planning attorney",
      "wealth management firm",
      "fractional CFO services",
      "business law firm small business",
      "business funding consultant"
    ],
    "countryCode": "us",
    "languageCode": "en",
    "maxCrawledPlaces": 200,
    "proxyConfig": {"useApifyProxy": true}
  }
}
```

**Meta Ad Library:**
```json
{
  "action": "start",
  "actorId": "curious_coder~facebook-ads-library-scraper",
  "input": {
    "urls": [
      {"url": "https://www.facebook.com/ads/library/?active_status=active&ad_type=all&country=US&q=tax+strategy"},
      {"url": "https://www.facebook.com/ads/library/?active_status=active&ad_type=all&country=US&q=real+estate+coaching"},
      {"url": "https://www.facebook.com/ads/library/?active_status=active&ad_type=all&country=US&q=business+funding"},
      {"url": "https://www.facebook.com/ads/library/?active_status=active&ad_type=all&country=US&q=wealth+management"},
      {"url": "https://www.facebook.com/ads/library/?active_status=active&ad_type=all&country=US&q=short+term+rental+coaching"},
      {"url": "https://www.facebook.com/ads/library/?active_status=active&ad_type=all&country=US&q=business+coaching+high+ticket"},
      {"url": "https://www.facebook.com/ads/library/?active_status=active&ad_type=all&country=US&q=CPA+firm"}
    ],
    "maxAds": 100
  }
}
```

**Instagram Profiles:**
```json
{
  "action": "start",
  "actorId": "apify~instagram-profile-scraper",
  "input": {
    "usernames": [
      "beardybrandon", "creditwithcolin", "amanda_han_cpa",
      "samfasterfreedom", "austinrutherfordofficial", "realkingkhang",
      "thejustincolby", "ajosborne", "mikebuontempo", "wadehouston",
      "adamerhart", "nicholaskusmich", "grantmitt", "charlietpham",
      "avamistruzzi"
    ]
  }
}
```

### Step 1b: Collect all results

Combine all `runs` arrays from the three `start` calls into one array, then call `collect`:

```json
{
  "action": "collect",
  "runs": [
    {"runId": "...", "actorId": "compass~crawler-google-places", "datasetId": "..."},
    {"runId": "...", "actorId": "curious_coder~facebook-ads-library-scraper", "datasetId": "..."},
    {"runId": "...", "actorId": "apify~instagram-profile-scraper", "datasetId": "..."}
  ]
}
```

If any runs are still `pending`, wait 30 seconds and call `collect` again. Repeat until all are complete.

### Step 2: Run website contact scraper

After Step 1 completes, collect all website URLs from the results, deduplicate by domain, and start the contact scraper:

```json
{
  "action": "start",
  "actorId": "vdrmota~contact-info-scraper",
  "input": {
    "startUrls": [
      {"url": "https://www.example1.com"},
      {"url": "https://www.example2.com"}
    ],
    "maxDepth": 2,
    "maxPagesPerDomain": 5,
    "sameDomain": true,
    "proxy": {"useApifyProxy": true}
  }
}
```

Batch at 50 URLs per `start` call. For larger URL lists, start multiple runs and collect them all together.

---

## Actor IDs Quick Reference

| Scraper | Actor ID | Typical runtime |
|---|---|---|
| Google Maps | `compass~crawler-google-places` | 2-10 min |
| Meta Ad Library | `curious_coder~facebook-ads-library-scraper` | 3-15 min |
| Instagram Profiles | `apify~instagram-profile-scraper` | 5-30 sec per profile |
| Website Contact Info | `vdrmota~contact-info-scraper` | 5-30 sec per batch |

---

## Common Errors

| Error | Cause | Fix |
|---|---|---|
| `invalid-input: Items in input.urls do not contain valid URLs` | Meta Ads `urls` passed as strings instead of objects | Use `[{"url": "..."}]` format |
| `platform-feature-disabled: Monthly usage hard limit exceeded` | Apify billing limit hit | Increase limit in Apify console or wait for reset |
| Run status `ABORTED` with no error | Usually billing or memory limit | Check status message, increase memory or reduce `maxCrawledPlaces` |
