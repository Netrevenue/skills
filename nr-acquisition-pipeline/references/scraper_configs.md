# Scraper Configurations

Read this file when executing Steps 1 and 2 of the pipeline. It contains the exact Apify actor IDs, search queries, and parameters for each scraping source.

## Table of Contents
- [Google Maps Scraper](#google-maps-scraper)
- [Meta Ad Library Scraper](#meta-ad-library-scraper)
- [Instagram Profile Scraper](#instagram-profile-scraper)
- [Website Email Scraper](#website-email-scraper)
- [Output Normalization](#output-normalization)

---

## Google Maps Scraper

**Actor:** `compass~crawler-google-places`

**Purpose:** Find professional service firms (Segment A ICPs) with real offices, reviews, and websites.

**Input parameters:**
- `countryCode`: "us"
- `languageCode`: "en"
- `maxCrawledPlacesPerSearch`: 200 (override with max_results if testing)

**Search queries (8 total, 200 results each = 1,600 max/run):**
1. "tax advisory firm"
2. "tax strategy consultant"
3. "CPA firm business advisory"
4. "estate planning attorney"
5. "wealth management firm"
6. "fractional CFO services"
7. "business law firm small business"
8. "business funding consultant"

**Normalize output to:**
```json
{
  "name": "title or name field",
  "phone": "phone or phoneUnformatted",
  "website": "website field",
  "address": "address or street",
  "rating": "totalScore or rating (float)",
  "review_count": "reviewsCount (int)",
  "category": "categoryName",
  "place_id": "placeId",
  "search_query": "the query that found this result",
  "source": "google_maps"
}
```

**Dedup within source:** By `place_id`. If the same place appears in multiple queries, keep the first occurrence but merge search_queries into an array.

---

## Meta Ad Library Scraper

**Actor:** `curious_coder~facebook-ads-library-scraper`

**Purpose:** Find active advertisers in NR ICP verticals. Active ad spend is one of the strongest qualifying signals — it means the business is investing in growth.

**Input parameters:**
- Construct Meta Ad Library URLs: `https://www.facebook.com/ads/library/?active_status=active&ad_type=all&country=US&q={search_query}`
- `maxAds`: 100
- `timeout_secs`: 1800

**Search queries (7 total, 100 results each = 700 max/run):**
1. "tax strategy"
2. "real estate coaching"
3. "business funding"
4. "wealth management"
5. "short term rental coaching"
6. "business coaching high ticket"
7. "CPA firm"

**Normalize output to:**
```json
{
  "page_name": "page_name or pageName",
  "page_url": "snapshot.page_profile_uri or pageUrl or page_url or ad_library_url",
  "ad_count": "aggregated during dedup (count of ads per page)",
  "website": "snapshot.link_url or website or externalUrl",
  "search_query": "the query that found this result",
  "source": "meta_ad_library"
}
```

**Dedup within source:** By `page_url`. Aggregate `ad_count` (count how many ads each page is running).

---

## Instagram Profile Scraper

**Actor:** `apify~instagram-profile-scraper`

**Purpose:** Scrape public profiles of followers of existing NR clients. These followers are likely in adjacent niches and are warm prospects.

**Input parameters:**
- `usernames`: seed account list (see below)
- `resultsLimit`: 500 (followers per seed account)
- `proxy.useApifyProxy`: true
- `proxy.apifyProxyGroups`: ["RESIDENTIAL"]

**Seed accounts (15 current, expanding to 61):**
```
beardybrandon, creditwithcolin, amanda_han_cpa, samfasterfreedom,
austinrutherfordofficial, realkingkhang, thejustincolby, ajosborne,
mikebuontempo, wadehouston, adamerhart, nicholaskusmich,
grantmitt, charlietpham, avamistruzzi
```

**Normalize output to:**
```json
{
  "username": "username field",
  "full_name": "fullName or full_name",
  "bio": "biography or bio",
  "follower_count": "followersCount or follower_count (int)",
  "following_count": "followsCount or following_count (int)",
  "post_count": "postsCount or post_count (int)",
  "external_url": "externalUrl or external_url or website",
  "is_business_account": "isBusinessAccount (bool)",
  "source_client": "the NR client handle this follower came from",
  "source": "ig_followers"
}
```

**Dedup within source:** By `username` (case-insensitive). If same person follows multiple NR clients, merge `source_client` into an array.

---

## Website Email Scraper

**Actor:** `vdrmota~contact-info-scraper`

**Purpose:** Crawl business websites from the other 3 scrapers to extract email, phone, LinkedIn, and booking links. Runs AFTER the other scrapers.

**Input parameters:**
- `startUrls`: array of URLs from Google Maps, Meta Ads, and IG bio external_urls
- `maxDepth`: 2 (homepage + 1 click deep)
- `maxPagesPerDomain`: 5
- `sameDomainOnly`: true
- `extractEmails`: true
- `extractPhones`: true
- `extractLinkedIn`: true
- `extractInstagram`: true
- `proxy.useApifyProxy`: true

**Batch size:** 50 URLs per actor run. For large URL lists, run multiple batches.

**Priority pages (checked first for contact info):**
```
/contact, /about, /team, /about-us, /contact-us,
/our-team, /schedule, /book, /consultation
```

**Booking keywords (detect if lead books calls):**
```
calendly.com, acuityscheduling.com, hubspot.com/meetings,
bookme, schedule, book-a-call, consultation, book-now,
appointment, setmore, oncehub, tidycal
```

**Normalize output to:**
```json
{
  "input_url": "the URL that was scraped",
  "emails_found": ["array of emails"],
  "phones_found": ["array of phones"],
  "linkedin_urls_found": ["array of LinkedIn URLs"],
  "instagram_handles_found": ["array of IG handles"],
  "booking_link_found": "URL or keyword if detected, else null"
}
```

---

## Output Normalization

Each scraper returns data in different formats depending on the Apify actor version. The normalization rules above handle common field name variations. When in doubt:

- Try the most common field name first, then fallbacks
- Strip whitespace and normalize URLs (add https:// if missing)
- Convert string numbers to integers where noted (review_count, follower_count)
- Dates should be ISO format (YYYY-MM-DD)
