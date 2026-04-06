---
name: nr-setter-scorecard
description: Scores classified B2B leads 0-100 against the Net Revenue setter qualification scorecard, assigns score tiers, writes setter-ready notes, and routes leads to the correct outbound account and channel. Use this skill whenever leads need to be scored, ranked, tiered, or assigned for outreach — including after classification, during pipeline reviews, when re-scoring existing leads, or when the user asks to "score", "rank", "tier", or "prioritize" leads. This skill requires classification data from nr-icp-classifier as input.
metadata: {"openclaw":{"emoji":"📊"}}
---

# NR Setter Qualification Scorecard

Score a pre-classified lead against Net Revenue's setter qualification scorecard. This is step 2 of a two-step process — classification comes from `nr-icp-classifier`, scoring happens here.

## What it does

Takes a classified lead (with vertical, offer type, demand signals) plus raw lead data and Apollo enrichment, then:
1. Scores it 0-100 across 5 criteria (20 points each)
2. Assigns a tier (priority_a, standard, nurture, dq)
3. Writes 1-2 sentence notes a setter can reference during outreach
4. Routes it to the right outbound account and contact channel

## Inputs

Two JSON objects are required:

### Classification (from nr-icp-classifier)
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

### Lead data (raw + Apollo enrichment)
```json
{
  "name": "...", "bio": "...", "website": "...", "category": "...",
  "follower_count": null, "review_count": 85, "rating": 4.9,
  "has_booking_link": true, "is_running_ads": false,
  "sources": ["google_maps"], "email": "...", "phone": "...",
  "ig_handle": "...", "linkedin_url": "...",
  "apollo_headcount": 12, "apollo_revenue": 2500000,
  "apollo_industry": "Accounting", "apollo_founded_year": 2018,
  "apollo_funding_total": null, "apollo_technologies": [],
  "apollo_org_name": "...", "apollo_enriched": true
}
```

Any field can be null or missing.

## Scoring Rubric

Each criterion is scored independently, 0-20 points, for a total of 0-100. The five criteria are designed to answer one question: "Is this lead worth a setter's time?"

### 1. Social Proof / Reviews (0-20)

This measures whether the lead has public credibility signals. Leads with strong social proof are easier to close because prospects can verify their track record.

| Threshold | Points |
|-----------|--------|
| 100+ reviews OR 4-5 star Trustpilot | 20 |
| 50-99 reviews | 16 |
| 20-49 reviews | 12 |
| Under 20 reviews (but some exist) | 8 |
| No reviews / no data | 0 |

Use `review_count` and `rating` as primary signals.

### 2. Review / Proof Volume (0-20)

This overlaps with social proof but focuses on the raw volume of evidence — testimonials, case studies, client mentions. High volume means consistent delivery over time, which correlates with ability to pay for NR's services.

| Threshold | Points |
|-----------|--------|
| 100+ proof points | 20 |
| 50-99 | 16 |
| 20-49 | 12 |
| Under 20 | 8 |
| No data | 0 |

When review data is missing, estimate conservatively from social signals (follower engagement, testimonials in bio, case studies on website).

### 3. Real Audience Size — Cross-Platform (0-20)

Audience size indicates reach and demand capacity. But this criterion works differently for the two ICP segments because professional service firms don't build audiences the same way coaches do.

**For coaches/consultants (Segment B):**

| Threshold | Points |
|-----------|--------|
| 250K+ followers | 20 |
| 100K-249K | 16 |
| 50K-99K | 12 |
| 20K-49K | 6 |
| Under 20K | 0 |

**For professional service firms (Segment A) without social presence:**

Use alternative credibility signals instead — these firms don't need followers to be great prospects:
- Apollo headcount 50+ → 16-20
- Apollo headcount 20-49 → 12
- Apollo headcount 10-19 → 6-8
- Apollo revenue $5M+ → 16-20
- Apollo revenue $1-5M → 12
- Review count 50+ → 12-16
- Founded 10+ years ago → bonus 2-4 points

Use the highest applicable score, not a sum.

### 4. Offer Fit — High-Ticket, $3K+ (0-20)

NR's services are priced for businesses selling $3K+ offers. A lead selling $50 ebooks can't afford NR and won't see ROI. This criterion gates for economic fit.

| Fit Level | Points |
|-----------|--------|
| Clear high-ticket ($3K+ confirmed) | 20 |
| Partial (likely but unconfirmed) | 10 |
| No (low-ticket, free, or unknown) | 0 |

Signals for clear fit:
- `estimated_ticket` is `3k_5k` or higher
- Professional service firm with Apollo revenue > $500K and headcount > 3
- Booking link + running ads (paid consultation model)
- Bio/website mentions premium pricing, retainers, or enterprise clients

Signals for partial:
- `estimated_ticket` is `unknown` but industry norms suggest $3K+
- Coaching model with large audience but unclear pricing

### 5. Monthly Demand — Estimated Booked Calls (0-20)

This estimates whether the lead has enough inbound demand to justify NR's acquisition system. A lead with 5 calls a month doesn't need what NR offers. A lead drowning in 150+ calls/month is the ideal — they need systems to scale.

| Threshold | Points |
|-----------|--------|
| 150+ estimated monthly calls | 20 |
| 75-150 | 16 |
| 50-74 | 12 |
| 20-49 | 6 |
| Under 20 | 0 |

This is an estimate — triangulate from:
- `is_running_ads` = true → strong signal, likely 50+ calls/month
- `has_booking_link` = true → confirmed call-booking model
- Apollo headcount > 10 → team to handle volume
- Apollo revenue > $1M → revenue implies consistent inbound
- High review count → high customer throughput
- Large follower count + booking link → likely high call volume

When unsure between two tiers, pick the lower one. False positives waste setter time — a setter who shows up to a call with inflated expectations loses trust in the pipeline.

## Score Tiers

| Tier | Range | What happens |
|------|-------|--------------|
| **priority_a** | 80-100 | Push to GHL leadgen pipeline, top of queue |
| **standard** | 65-79 | Push to GHL leadgen pipeline, normal queue |
| **nurture** | 50-64 | Push to GHL nurture pipeline, long-term follow-up |
| **dq** | Below 50 | Logged but not pushed |

If the classification vertical is `disqualified`, assign score 0 and tier `dq` automatically.

The tier assignment follows the score mechanically — the scorer doesn't override tiers based on gut feel. The rubric is the system of record.

## Setter Notes

Write 1-2 sentences a setter can glance at before reaching out. These should be actionable — reference specific data points, not generic praise.

Good: "Tax advisory firm with 85+ Google reviews and 12-person team in Dallas. Lead with how NR helped Amanda Han CPA scale client acquisition."

Bad: "Looks like a promising lead with good potential."

Include:
- What the lead does (plain language, not the vertical code)
- The key signal that drove the score (why they're good or why they fell short)
- A suggested outreach angle if score >= 50

## Account & Channel Assignment

After scoring, route the lead to the right outbound account and contact channel. Read `references/account_assignment.md` for the full mapping tables and the reasoning behind each rule.

The short version:
- Each vertical maps to a primary and secondary setter
- Channel priority: LinkedIn for professional service verticals (if they have a profile), email for 100K+ audience leads, IG for everyone else
- Flag partnership-track leads if they're in agency verticals

## Output format

Return only valid JSON — no markdown wrapping, no explanation:

```json
{
  "setter_score": 76,
  "score_breakdown": {
    "social_proof": 16,
    "review_volume": 16,
    "audience_size": 12,
    "offer_fit": 20,
    "demand_indicator": 12
  },
  "score_tier": "standard",
  "notes": "CPA firm with 65+ Google reviews and 12-person team in Dallas. Has booking link but not running ads. Lead with tax strategy pain points.",
  "assigned_account": "Britt",
  "secondary_account": "Wade",
  "primary_channel": "linkedin",
  "is_partnership_track": false
}
```

## Examples

**Example 1: Strong Segment A lead**
Classification: `{"vertical": "tax_strategy", "offer_type": "service", "estimated_ticket": "10k_25k", "audience_tier": "unknown", "demand_signals": ["high_reviews", "has_team", "booking_calls"]}`
Lead data: review_count=92, rating=4.9, has_booking_link=true, apollo_headcount=8, apollo_revenue=1800000, apollo_founded_year=2015, linkedin_url="https://linkedin.com/company/apex-tax"

Output:
```json
{"setter_score": 82, "score_breakdown": {"social_proof": 16, "review_volume": 16, "audience_size": 12, "offer_fit": 20, "demand_indicator": 18}, "score_tier": "priority_a", "notes": "Tax advisory firm, 92 Google reviews at 4.9 stars, 8-person team, $1.8M revenue. Has booking link and strong fundamentals. Lead with scale story — similar firms 2x'd inbound with NR.", "assigned_account": "Britt", "secondary_account": "Wade", "primary_channel": "linkedin", "is_partnership_track": false}
```

**Example 2: Mid-tier coach**
Classification: `{"vertical": "str_coaching", "offer_type": "coaching", "estimated_ticket": "3k_5k", "audience_tier": "mid", "demand_signals": ["running_ads", "booking_calls"]}`
Lead data: follower_count=78000, is_running_ads=true, has_booking_link=true, review_count=null, apollo_headcount=3, email="marcus@strempire.com"

Output:
```json
{"setter_score": 62, "score_breakdown": {"social_proof": 0, "review_volume": 0, "audience_size": 12, "offer_fit": 20, "demand_indicator": 16}, "score_tier": "nurture", "notes": "STR coaching program with 78K followers, running ads and booking calls. No review data which drags down score. Worth nurturing — if they can show testimonials, score jumps to standard.", "assigned_account": "Wade", "secondary_account": "Ben", "primary_channel": "ig", "is_partnership_track": false}
```

**Example 3: Disqualified lead**
Classification: `{"vertical": "disqualified", "offer_type": "course", "estimated_ticket": "under_3k", "audience_tier": "mid", "demand_signals": ["running_ads"], "disqualify_reason": "crypto keyword in bio"}`

Output:
```json
{"setter_score": 0, "score_breakdown": {"social_proof": 0, "review_volume": 0, "audience_size": 0, "offer_fit": 0, "demand_indicator": 0}, "score_tier": "dq", "notes": "Disqualified — crypto/trading vertical. Not an NR ICP fit regardless of audience size.", "assigned_account": "Scott", "secondary_account": "Ben", "primary_channel": "ig", "is_partnership_track": false}
```

## Guardrails

- The 5 breakdown scores must sum to the total `setter_score`. This is a mechanical check — if they don't add up, the output is wrong.
- Zero supporting data for a criterion means 0 points for that criterion. Inflating scores on thin data is the most common failure mode — it sends weak leads to setters who then lose confidence in the pipeline.
- Disqualified leads from the classifier get score 0 across the board, tier `dq`. Still write notes (explaining the DQ) and assign account/channel (for logging, even though they won't be pushed).
- When classification data is missing entirely, return score 0, tier `dq`, and note that classification is required first.

## Failure handling

- If the lead is missing critical fields (no name, no website, no bio, no Apollo data), score based on whatever is available and note the data gaps in setter notes. A partial score is more useful than refusing to score.
- If you're unsure between two point values on a criterion, go lower. The cost of a false positive (wasted setter call) is higher than the cost of a false negative (lead stays in nurture a bit longer).
