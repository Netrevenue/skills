# Scraper Configurations

Read this file when executing Steps 1 and 2 of the pipeline. It contains the exact Apify actor IDs, search queries, input payloads, and output field mappings — all verified with real API calls.

## Table of Contents
- [Google Maps Scraper](#google-maps-scraper)
- [Meta Ad Library Scraper](#meta-ad-library-scraper)
- [Instagram Profile Scraper](#instagram-profile-scraper)
- [Website Email Scraper](#website-email-scraper)
- [Output Normalization](#output-normalization)
- [Dry Run Config](#dry-run-config)

---

## Google Maps Scraper

**Actor:** `compass~crawler-google-places`

**Purpose:** Find professional service firms (Segment A ICPs) with real offices, reviews, and websites.

**Verified input payload:**
```json
{
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
  "proxyConfig": {
    "useApifyProxy": true
  }
}
```

Key input notes:
- `searchStringsArray` is an array of strings (all queries run in one actor call)
- `maxCrawledPlaces` controls total results across ALL queries (not per query). Set to 200 for production, lower for testing.
- `proxyConfig` is required — omitting it causes failures on some queries

**8 search queries, ~200 results total per run.**

**Actual output fields** (verified 2026-04-06):
```json
{
  "title": "Block Advisors",
  "categoryName": "Tax consultant",
  "categories": ["Tax consultant", "Accountant", "Accounting firm"],
  "address": "1320 Huffman Park Dr Ste S190, Anchorage, AK 99515",
  "street": "1320 Huffman Park Dr Ste S190",
  "city": "Anchorage",
  "state": "Alaska",
  "postalCode": "99515",
  "countryCode": "US",
  "website": "https://www.blockadvisors.com/...",
  "phone": "(907) 348-7338",
  "phoneUnformatted": "+19073487338",
  "totalScore": 4.6,
  "reviewsCount": 115,
  "placeId": "ChIJBeEuHZuZyFYR8eWPkIlAMKg",
  "searchString": "tax advisory firm",
  "rank": 1,
  "isAdvertisement": false,
  "location": {"lat": 61.10, "lng": -149.85},
  "permanentlyClosed": false,
  "temporarilyClosed": false,
  "url": "https://www.google.com/maps/search/...",
  "scrapedAt": "2026-04-06T18:00:48.641Z"
}
```

**Normalize to unified schema:**

| Actor output field | Unified lead field | Notes |
|---|---|---|
| `title` | `name` | |
| `phone` | `phone` | Fallback: `phoneUnformatted` |
| `website` | `website` | |
| `address` | `address` | |
| `totalScore` | `rating` | Float (e.g., 4.6) |
| `reviewsCount` | `review_count` | Integer |
| `categoryName` | `category` | |
| `placeId` | `place_id` | |
| `searchString` | `search_query` | |
| — | `source` | Set to `"google_maps"` |

**Dedup within source:** By `placeId`. If the same place appears from multiple queries, keep the first occurrence but merge `searchString` values into an array.

Skip any result where `permanentlyClosed` is `true`.

---

## Meta Ad Library Scraper

**Actor:** `curious_coder~facebook-ads-library-scraper`

**Purpose:** Find active advertisers in NR ICP verticals. Active ad spend is one of the strongest qualifying signals.

**Verified input payload:**
```json
{
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
```

Key input notes:
- `urls` MUST be an array of **objects** with a `url` key — NOT plain strings. Plain strings cause an `invalid-input` error.
- Each URL is a pre-built Facebook Ad Library search URL with `active_status=active` and `country=US`
- `maxAds` is the total limit across all URLs
- No proxy config needed — the actor handles it internally

**7 search queries, ~100 results total per run.**

**Actual output fields** (verified 2026-04-06):
```json
{
  "ad_archive_id": "852645850996857",
  "page_id": "241760048297",
  "page_name": "Fidelity Investments",
  "is_active": true,
  "collation_count": 2,
  "start_date": "2026-03-15",
  "end_date": null,
  "spend": {"lower_bound": "100", "upper_bound": "499"},
  "currency": "USD",
  "publisher_platform": ["facebook", "instagram"],
  "categories": [],
  "ad_library_url": "https://www.facebook.com/ads/library/?id=852645850996857",
  "url": "...",
  "snapshot": {
    "page_name": "Fidelity Investments",
    "page_profile_uri": "https://www.facebook.com/fidelityinvestments/",
    "caption": "FIDELITY.COM/WEALTH",
    "cta_text": "Learn more",
    "cards": [
      {
        "body": "Build a flexible plan...",
        "link_url": "https://ad.doubleclick.net/...",
        "title": "Make a plan you can stick to",
        "cta_text": "Learn More"
      }
    ]
  }
}
```

**Normalize to unified schema:**

| Actor output field | Unified lead field | Notes |
|---|---|---|
| `page_name` | `page_name` | Top-level field |
| `snapshot.page_profile_uri` | `page_url` | Facebook page URL |
| `snapshot.cards[0].link_url` | `website` | External website from ad |
| `page_id` | — | Used for dedup |
| — | `ad_count` | Aggregated: count ads per `page_id` |
| — | `source` | Set to `"meta_ad_library"` |

**Dedup within source:** By `page_id` (not `ad_archive_id` — one page runs multiple ads). Count unique `ad_archive_id` values per page as `ad_count`.

Note: `snapshot.cards[0].link_url` often points to tracking URLs (doubleclick, etc.) rather than the actual business website. Extract the final destination domain where possible, or use the `caption` field which sometimes contains the real domain.

---

## Instagram Profile Scraper

**Actor:** `apify~instagram-profile-scraper`

**Purpose:** Scrape public profiles of followers of existing NR clients. These followers are likely in adjacent niches and are warm prospects.

**Verified input payload:**
```json
{
  "usernames": [
    "beardybrandon", "creditwithcolin", "amanda_han_cpa",
    "samfasterfreedom", "austinrutherfordofficial", "realkingkhang",
    "thejustincolby", "ajosborne", "mikebuontempo", "wadehouston",
    "adamerhart", "nicholaskusmich", "grantmitt", "charlietpham",
    "avamistruzzi"
  ]
}
```

Key input notes:
- `usernames` is a simple array of strings — no objects, no `@` prefix
- No proxy config needed — the actor handles it internally
- This scrapes the **profile data** for each username, not their followers. To scrape followers, you would need a different actor or approach. For pipeline purposes, pass the seed account usernames to get their profile info, and use the profile data (bio, follower count, external URL) for lead qualification.

**15 seed accounts (expanding to 61).**

**Actual output fields** (verified 2026-04-06):
```json
{
  "inputUrl": "https://www.instagram.com/thejustincolby",
  "id": "21650014",
  "username": "thejustincolby",
  "url": "https://www.instagram.com/thejustincolby",
  "fullName": "Justin Colby | Entrepreneur | Real Estate Investor | Coach",
  "biography": "The Entrepreneur DNA Community\n🎙️Host of 'The Entrepreneur DNA'...",
  "externalUrl": "http://www.theentrepreneurdna.com/",
  "externalUrls": [
    {
      "title": "The Entrepreneur DNA Community",
      "url": "http://www.theentrepreneurdna.com",
      "link_type": "external"
    }
  ],
  "followersCount": 127862,
  "followsCount": 998,
  "postsCount": 4521,
  "isBusinessAccount": true,
  "businessCategoryName": "None,Entrepreneur",
  "private": false,
  "verified": true,
  "profilePicUrl": "https://scontent-...",
  "relatedProfiles": [...],
  "latestPosts": [...]
}
```

**Normalize to unified schema:**

| Actor output field | Unified lead field | Notes |
|---|---|---|
| `username` | `ig_handle` | Lowercase for matching |
| `fullName` | `full_name` | |
| `biography` | `bio` | |
| `followersCount` | `follower_count` | Integer |
| `followsCount` | `following_count` | Integer |
| `postsCount` | `post_count` | Integer |
| `externalUrl` | `external_url` | Primary external link |
| `isBusinessAccount` | `is_business_account` | Boolean |
| — | `source_client` | The NR client handle this profile came from |
| — | `source` | Set to `"ig_followers"` |

**Dedup within source:** By `username` (case-insensitive). If the same person appears from multiple seed account runs, merge `source_client` into an array.

Skip any profile where `private` is `true` — we can't use their data for outreach.

---

## Website Email Scraper

**Actor:** `vdrmota~contact-info-scraper`

**Purpose:** Crawl business websites from the other 3 scrapers to extract email, phone, LinkedIn, and booking links. Runs AFTER the other scrapers.

**Verified input payload:**
```json
{
  "startUrls": [
    {"url": "https://www.example1.com"},
    {"url": "https://www.example2.com"}
  ],
  "maxDepth": 2,
  "maxPagesPerDomain": 5,
  "sameDomain": true,
  "proxy": {
    "useApifyProxy": true
  }
}
```

Key input notes:
- `startUrls` MUST be an array of **objects** with a `url` key — NOT plain strings
- The field is `sameDomain` (NOT `sameDomainOnly`)
- `proxy` uses nested object format (NOT flat `proxy.useApifyProxy`)
- No `extractEmails`/`extractPhones` flags — the actor extracts all contact types by default
- Batch at 50 URLs per actor run to stay within memory limits

**Actual output fields** (verified 2026-04-06):
```json
{
  "originalStartUrl": "https://www.taxplanningpros.com",
  "domain": "taxplanningpros.com",
  "scrapedUrls": [
    "https://www.taxplanningpros.com/",
    "https://www.taxplanningpros.com/about-5",
    "https://www.taxplanningpros.com/services-2",
    "https://www.taxplanningpros.com/blog"
  ],
  "depth": 0,
  "emails": ["rebecca@taxplanningpros.com", "info@mysite.com"],
  "phones": [],
  "phonesUncertain": ["208-231-3555", "816-565-9904"],
  "linkedIns": [],
  "instagrams": [],
  "facebooks": [],
  "twitters": [],
  "youtubes": [],
  "tiktoks": [],
  "threads": [],
  "telegrams": [],
  "reddits": [],
  "whatsapps": [],
  "pinterests": [],
  "discords": [],
  "snapchats": []
}
```

**Normalize to unified schema:**

| Actor output field | Unified lead field | Notes |
|---|---|---|
| `originalStartUrl` | `input_url` | The URL we submitted |
| `domain` | — | Use for matching back to leads |
| `emails` | `emails_found` | Array; take first non-generic email |
| `phones` + `phonesUncertain` | `phones_found` | Merge both arrays; `phones` are higher confidence |
| `linkedIns` | `linkedin_urls_found` | Array of LinkedIn profile URLs |
| `instagrams` | `instagram_handles_found` | Array of IG handles |
| — | `booking_link_found` | Not a direct field — scan `scrapedUrls` for booking keywords |

**Booking link detection:** The actor doesn't flag booking links directly. Scan the `scrapedUrls` array for these patterns:
```
calendly.com, acuityscheduling.com, hubspot.com/meetings,
bookme, schedule, book-a-call, consultation, book-now,
appointment, setmore, oncehub, tidycal
```

**Email filtering:** Ignore generic emails like `info@mysite.com`, `admin@example.com`, etc. Prefer emails that contain a person's name or the business domain.

---

## Output Normalization

Each scraper returns data in the formats documented above (all verified with real API calls on 2026-04-06). When normalizing:

- Strip whitespace and normalize URLs (add `https://` if missing)
- Convert string numbers to integers where noted (`reviewsCount`, `followersCount`)
- Lowercase all IG handles for consistent matching
- Dates should be ISO format (YYYY-MM-DD)
- Null/missing fields should be set to `null` in the unified schema, not empty strings

---

## Dry Run Config

For testing with small datasets, use these overrides:

```json
{
  "google_maps": {
    "searchStringsArray": ["tax advisory firm"],
    "maxCrawledPlaces": 5
  },
  "meta_ads": {
    "urls": [{"url": "https://www.facebook.com/ads/library/?active_status=active&ad_type=all&country=US&q=tax+strategy"}],
    "maxAds": 5
  },
  "instagram": {
    "usernames": ["thejustincolby"]
  },
  "contact_scraper": {
    "maxDepth": 1,
    "maxPagesPerDomain": 2
  }
}
```

This produces ~10-15 leads and costs < $1 in Apify compute. Use this for end-to-end pipeline validation.
