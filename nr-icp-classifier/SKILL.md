---
name: nr-icp-classifier
description: Classifies B2B leads into Net Revenue's ICP verticals, applies hard disqualifiers, and determines offer type, ticket size, audience tier, and demand signals. Use this skill whenever you have lead data that needs to be qualified before scoring — whether from Apify scraping, Apollo enrichment, CRM imports, or manual entry. Also use it when the user asks to "classify", "qualify", "check fit", or "run ICP" on leads or prospects. This is always run before nr-setter-scorecard.
metadata: {"openclaw":{"emoji":"🎯"}}
---

# NR ICP Classifier

Classify inbound lead data against Net Revenue's Ideal Customer Profiles and return structured JSON. This is step 1 of a two-step process — classification happens here, scoring happens in `nr-setter-scorecard`.

## What it does

Takes raw lead data (name, bio, website, social metrics, Apollo company data) and determines:
- Which vertical the lead belongs to
- Whether they hit a hard disqualifier
- Their offer type, estimated ticket size, audience tier, and demand signals

The output feeds directly into `nr-setter-scorecard` for scoring.

## Target ICP Segments

NR serves two core segments. Understanding the difference matters because they have different qualifying signals — Segment A leads may have zero social presence but still be excellent prospects based on company fundamentals.

### Segment A: Professional Service Firms
Headcount 3-200, deal size $3K-$25K+. These are established businesses that need client acquisition help. They often have strong Google reviews and Apollo data but limited social media.

- **tax_strategy** — Tax strategists, tax advisory firms, tax planning consultants
- **cpa_accounting** — CPA firms, accounting firms with business advisory services
- **legal_services** — Business law, tax law, M&A attorneys, estate planning attorneys
- **wealth_management** — Wealth management firms, financial advisors, family offices
- **financial_services** — Business funding, credit services, lending consultants
- **fractional_cfo** — Fractional CFO services, outsourced finance leadership

### Segment B: High-Ticket Coaches & Consultants
Signals: 20K+ followers, active ads, booking calls. These are personal brands that sell high-ticket programs and need help scaling.

- **real_estate_coaching** — Real estate investing coaches, wholesaling coaches
- **str_coaching** — Short-term rental / Airbnb coaches
- **business_coaching** — Business acquisition coaches, high-ticket business consultants
- **business_funding** — Business credit/funding coaches and consultants
- **career_advancement** — Career coaches, executive placement, leadership development

### Catch-all
- **other** — Doesn't fit Segment A or B but isn't disqualified
- **disqualified** — Matches a hard disqualifier

## Hard Disqualifiers

Check these first before doing any other classification. NR has learned from experience that these verticals have poor close rates and high churn — they're not worth setter time regardless of how good the numbers look.

If the lead's name, bio, category, or industry contains any of these keywords (case-insensitive), classify as `disqualified`:

```
ecommerce, e-commerce, dropshipping, crypto, cryptocurrency,
forex, day trading, daytrading, realtor, real estate agent,
mlm, network marketing, affiliate, fitness coach, personal trainer,
weight loss, bodybuilding, keto, supplement
```

One important nuance: "real estate agent" (someone selling houses) is disqualified, but "real estate coach" (someone teaching investing) is a target ICP. Look at the full context — if the bio says "helping investors" or "coaching program", that's Segment B. If it says "listings" or "your dream home", that's DQ.

## NR Client Exclusion

Before classifying, check if the lead's ig_handle matches an existing NR client. Read `references/nr_client_handles.md` for the full list. Matching leads are existing clients and should be flagged immediately — outreaching to them wastes setter time and looks unprofessional.

## Classification Fields

### vertical
One of: `tax_strategy`, `cpa_accounting`, `legal_services`, `wealth_management`, `financial_services`, `fractional_cfo`, `real_estate_coaching`, `str_coaching`, `business_coaching`, `business_funding`, `career_advancement`, `other`, `disqualified`

Pick the single strongest fit. If a CPA firm also does tax strategy, go with whichever their primary service is based on bio/website/category.

### offer_type
One of: `service`, `coaching`, `course`, `mastermind`, `consulting`, `unknown`

### estimated_ticket
One of: `under_3k`, `3k_5k`, `5k_10k`, `10k_25k`, `25k_plus`, `unknown`

Use Apollo revenue, headcount, offer type, and industry norms to estimate. Legal and wealth management typically run $10K+. Coaching varies widely — large audience with active ads usually means $3K-$10K programs.

### audience_tier
One of: `micro` (under 5K), `small` (5K-20K), `mid` (20K-100K), `large` (100K-250K), `mega` (250K+), `unknown`

Based on follower_count. For Segment A firms without social presence, use `unknown` — it's fine, audience size isn't their qualifying signal.

### demand_signals
Array of any that apply: `running_ads`, `booking_calls`, `hiring`, `active_content`, `high_reviews`, `has_team`, `recent_launch`

Only include signals with clear evidence in the data. An empty array is fine if there's nothing concrete.

## Input format

```json
{
  "name": "Business name or person name",
  "bio": "Instagram bio or business description",
  "website": "https://example.com",
  "category": "Google Maps category if available",
  "follower_count": 15000,
  "review_count": 45,
  "rating": 4.8,
  "has_booking_link": true,
  "is_running_ads": false,
  "apollo_industry": "Accounting",
  "apollo_headcount": 12,
  "apollo_revenue": 2500000,
  "apollo_founded_year": 2018,
  "apollo_keywords": ["tax planning", "business advisory"]
}
```

Any field can be null, empty, or missing. Work with what's available.

## Output format

Return only valid JSON — no markdown wrapping, no explanation text:

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

When the vertical is `disqualified`, `disqualify_reason` explains which keyword or pattern triggered it. Otherwise it's `null`.

## Examples

**Example 1: Clear Segment A fit**
Input:
```json
{"name": "Apex Tax Advisory", "bio": "", "website": "https://apextaxadvisory.com", "category": "Tax preparation service", "follower_count": null, "review_count": 92, "rating": 4.9, "has_booking_link": true, "is_running_ads": false, "apollo_industry": "Accounting", "apollo_headcount": 8, "apollo_revenue": 1800000, "apollo_founded_year": 2015, "apollo_keywords": ["tax strategy", "business tax planning"]}
```
Output:
```json
{"vertical": "tax_strategy", "offer_type": "service", "estimated_ticket": "5k_10k", "audience_tier": "unknown", "demand_signals": ["high_reviews", "has_team", "booking_calls"], "disqualify_reason": null}
```

**Example 2: Hard disqualifier**
Input:
```json
{"name": "CryptoGains Academy", "bio": "Learn to trade crypto like a pro 🚀 Free Discord", "website": "https://cryptogains.io", "category": "", "follower_count": 45000, "review_count": null, "rating": null, "has_booking_link": false, "is_running_ads": true, "apollo_industry": null, "apollo_headcount": null, "apollo_revenue": null, "apollo_founded_year": null, "apollo_keywords": []}
```
Output:
```json
{"vertical": "disqualified", "offer_type": "course", "estimated_ticket": "under_3k", "audience_tier": "mid", "demand_signals": ["running_ads"], "disqualify_reason": "crypto/cryptocurrency keyword in bio"}
```

**Example 3: Segment B coach**
Input:
```json
{"name": "Marcus Reeves", "bio": "Helping investors build STR empires 🏡 500+ students | Book a call 👇", "website": "https://strempire.com", "category": "", "follower_count": 78000, "review_count": null, "rating": null, "has_booking_link": true, "is_running_ads": true, "apollo_industry": null, "apollo_headcount": 3, "apollo_revenue": null, "apollo_founded_year": 2022, "apollo_keywords": []}
```
Output:
```json
{"vertical": "str_coaching", "offer_type": "coaching", "estimated_ticket": "3k_5k", "audience_tier": "mid", "demand_signals": ["running_ads", "booking_calls", "active_content", "recent_launch"], "disqualify_reason": null}
```

## Guardrails

- Check hard disqualifiers first — they short-circuit everything else.
- If information is insufficient, prefer `unknown` or `other` over guessing. A wrong classification wastes more setter time than an `unknown` one.
- Don't add demand signals without evidence. The temptation is to be generous, but inflated signals cascade into inflated scores downstream.
