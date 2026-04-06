# Outbound Account & Channel Assignment Rules

After scoring, every lead gets assigned to an outbound account and a primary contact channel. This routing ensures each setter works verticals they know and each lead is reached via the channel most likely to get a response.

## Account by Vertical

Each vertical has a primary setter (who owns outreach) and a secondary (fallback if primary is at capacity or unavailable).

| Vertical | Primary | Secondary |
|----------|---------|-----------|
| real_estate_coaching | Tristan | Ben |
| str_coaching | Wade | Ben |
| tax_strategy | Britt | Wade |
| cpa_accounting | Britt | Wade |
| legal_services | Wade | Scott |
| wealth_management | Wade | Ben |
| financial_services | Ben | Scott |
| fractional_cfo | Scott | Wade |
| business_coaching | Scott | Wade |
| business_funding | Ben | Scott |
| career_advancement | Scott | Tristan |
| other | Scott | Ben |

## Channel Priority

The goal is to reach leads on the platform where they're most responsive. Professional service firms tend to treat LinkedIn as a business inbox, so it takes priority there. High-audience leads get email because it scales better than DMs. Everyone else gets IG since that's NR's home turf.

1. **LinkedIn** — Use when the lead has a `linkedin_url` AND the vertical is one of: `tax_strategy`, `cpa_accounting`, `legal_services`, `wealth_management`, `fractional_cfo`, `financial_services`
2. **Email** — Use when the lead has an email AND `follower_count` >= 100,000
3. **IG** — Default for everything else

## Partnership Track

Some leads aren't direct clients — they're potential agency partners who could refer business to NR. Flag `is_partnership_track: true` if the lead operates in one of these verticals:

`marketing_agency`, `demand_gen`, `branding_agency`, `fractional_cmo`, `pr_agency`, `compliance_ops_agency`

These verticals aren't in the standard classification list — they may show up as `other` from the classifier. Check Apollo industry data and bio keywords for signals.
