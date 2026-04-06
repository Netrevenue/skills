# Apollo.io Enrichment

Read this file when executing Step 4 of the pipeline. It covers the Apollo API calls, rate limiting, caching, and field mapping.

---

## API Details

- **Base URL:** `https://api.apollo.io/api/v1`
- **Auth:** `x-api-key` header with APOLLO_API_KEY
- **Rate limit:** 200 calls/hour on free plan. Stay under 180/hour to be safe.

## Organization Enrichment

**Endpoint:** `GET /organizations/enrich?domain={domain}`

Deduplicate domains across leads first — don't enrich the same domain twice.

**Fields to extract:**
| Apollo field | Lead field |
|---|---|
| estimated_num_employees | apollo_headcount |
| annual_revenue | apollo_revenue |
| industry | apollo_industry |
| keywords | apollo_keywords (array) |
| founded_year | apollo_founded_year |
| linkedin_url | apollo_linkedin_url |
| technologies | apollo_technologies (array) |
| total_funding | apollo_funding_total |
| name | apollo_org_name |

## Person Matching

**Endpoint:** `POST /people/match`

**Payload:** `{"domain": "example.com", "first_name": "John", "last_name": "Smith"}` (split lead name on whitespace). If only one name part, use `{"domain": "...", "name": "John"}`.

**Fields to extract:**
| Apollo field | Lead field |
|---|---|
| person.email | apollo_contact_email |
| person.title | apollo_contact_title |
| person.linkedin_url | apollo_contact_linkedin |

Skip person matching on small test runs (max_domains <= 20) to conserve rate budget.

## Caching

Maintain a persistent domain cache so enrichment can resume across runs:
- Cache successful enrichments AND failed lookups (empty object `{}` means "tried, no data")
- Save cache every 25 domains
- On next run, skip domains already in cache
- Pause 60 seconds between batches of 50
- Sleep 1.5 seconds between individual requests

## Merge Back

After enrichment, merge Apollo fields into each lead record by domain. Set `apollo_enriched: true` if any Apollo data was found, `false` otherwise.
