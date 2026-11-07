# Just Pick Something — Link Import Plan

Status: **draft**, not scheduled. Companion to the [project plan](project-plan.md);
expands the "Paste enrichment" idea under
[Later](project-plan.md#later--down-the-line-ideas).

Paste an Airbnb or VRBO link into a room and get an option with its columns
already filled in: title, bedrooms, bathrooms, sleeps, rating, location and
amenities like hot tub or pool.

> Example: in a Rental homes room, someone pastes
> `https://www.airbnb.com/rooms/748651517360941371`. A new option appears titled
> "Lakeview sauna retreat in Zephyr Cove" with Link, Bedrooms `3`,
> Bathrooms `2.5`, Sleeps `6`, Location `Glenbrook`, Guest rating `4.95`,
> Pool `yes` and Parking `yes`. Price per night stays empty, for a person to fill
> in.

---

## What we found (tested 2026-11-07)

Tested with `curl` from a home connection, no cookies, no JavaScript. **Not yet
tested from Vercel's servers**; see [Open questions](#open-questions).

### Airbnb: works with a plain fetch

`GET https://www.airbnb.com/rooms/{id}` returns `200` and about 575 KB of
server-rendered HTML, even with Go's default `User-Agent`. Three places in that
HTML hold listing data, from most to least stable:

| Source | What's in it | Notes |
| --- | --- | --- |
| Open Graph `<meta>` tags | `og:title` = `"Cabin in Glenbrook · ★4.95 · 3 bedrooms · 4 beds · 2.5 baths"`; `og:description` = the listing name; `og:image`; `twitter:title` adds city, state, country | Made for link previews (iMessage, Slack), so least likely to change or be blocked. Same format on every listing tried (4 of 4); baths can read "1 private bath". |
| JSON-LD `<script type="application/ld+json">` | schema.org `VacationRental`: `name`, `description`, `image[]`, `latitude`, `longitude`, `address.addressLocality`, `aggregateRating.ratingValue` / `ratingCount`, `containsPlace.occupancy` | Made for search engines, so fairly stable. **`occupancy` didn't match the page**: 4 here vs. 6 guests on the listing. Don't use it for Sleeps. |
| `<script id="data-deferred-state-0">` | ~200 KB of Airbnb's internal page JSON: `personCapacity` (guests), full amenity list as objects with `title` and `available`, photos, sections | Internal and undocumented: field names and nesting can change any time. Use only for Sleeps and amenities, and fail soft. |

**Not available: price.** Even with `?check_in=…&check_out=…&adults=4` the
server-rendered page has no price; it's loaded later by JavaScript through
Airbnb's private API. Calling that API means reproducing its keys and request
format, which is exactly the fragile, terms-breaking scraping to avoid. Price
stays a manual column.

### VRBO: blocked

Every request to `www.vrbo.com`, home page or listing, got `429` with a
"Bot or Not?" captcha page (`x-page-id: wildcard-challenge-handler`), including
with a desktop browser `User-Agent` and with Facebook's link-preview
`User-Agent`. VRBO (Expedia Group) runs bot protection in front of the whole
site. A server-side fetch won't work, and getting around a captcha is off the
table. VRBO needs one of the client-side options below.

### Rules and terms

- **Airbnb's `robots.txt`** allows general crawlers on `/rooms/{id}` but
  disallows its sub-pages (`/rooms/*/amenities`, `/reviews`, `/photos`, …) and
  blocks AI crawlers (`anthropic-ai`, `PerplexityBot`, `cohere-ai`) from
  `/rooms/` entirely. We fetch only the main listing page, once, when a person
  pastes its link, the same thing a chat app's link preview does. We aren't a
  crawler and we don't call the model on fetched pages (see
  [AI assistant plan](ai-assistant.md#out-of-scope)).
- **Both sites' terms of service forbid automated scraping** of their content.
  A one-off fetch on a user's paste, for a free personal app, is low-risk, but
  it is still against the terms. Keep the footprint small: one request per
  paste, no crawling, no following links, no storing their photos or text
  beyond the few column values, and a kill switch (`LINK_IMPORT_ENABLED`).
- **Neither site has a public listing API.** Airbnb's API is for approved
  hosting partners managing their own listings; Expedia's partner APIs are for
  hosts and affiliates. Neither fits.
- **Paid scraping services** (e.g. Apify actors, unofficial APIs on RapidAPI)
  would handle VRBO and prices, but they cost money per call, add a third-party
  key, and are still scraping. Out, under the
  [free-tier rule](project-plan.md#free-tier-only).

## Options compared

| Option | Airbnb | VRBO | Price | Cost / free tier | Effort |
| --- | --- | --- | --- | --- | --- |
| **A. Server fetch** (API fetches the URL, parses OG + JSON-LD + deferred state) | ✅ (if Vercel's IPs aren't blocked) | ❌ captcha | ❌ | One function call per paste | Small |
| **B. Bookmarklet** (user clicks it on the listing page; it reads the page they already have open and sends it to the app) | ✅ | ✅ probably (page is already loaded past the captcha) | ✅ if dates are picked and the price is on screen | Free | Medium |
| **C. Paste page text** (user copies the listing page, pastes it; we parse it) | ⚠️ messy | ⚠️ messy | ⚠️ | Free with heuristics; an LLM call parses it well but [costs money](ai-assistant.md#blocker-cost) | Medium |
| **D. Browser extension** | ✅ | ✅ | ✅ | Free | Large (store review, two browsers) |
| **E. Paid scraping API** | ✅ | ✅ | ✅ | ❌ paid | Small |

**Recommendation: A first, for Airbnb only, then B for VRBO.** A covers the
most common case with the least code and no user setup. B is the only free way
to cover VRBO without fighting its bot protection, because the user's own
browser has already loaded the page. C and D aren't worth it yet; E breaks the
free-tier rule.

## Phase 1 — server-side import for Airbnb

### API

| Method | Path | Auth | Body → Response |
| --- | --- | --- | --- |
| `POST` | `/rooms/{slug}/import` | ✓ | `{url}` → `200 {source, title, url, fields: {key: value}, imageUrl?}`; `400` not a supported link; `422` fetched but nothing usable found; `429` over the rate limit; `502` the site refused or timed out; `503` feature off |

This only **previews**: it returns values and writes nothing. The frontend maps
them onto the room's columns and adds the option with the existing
`POST /rooms/{slug}/items` (`{items: [{title, fields}]}`). So:

- No new write path, and the admin-only edit rule and `maxItems` still apply.
- The user sees what was found and can fix it before adding.
- Any member can import (adding options is open to everyone), but not in a
  locked room (`409`, same as adding).

`fields` uses fixed keys, not column IDs, because the backend doesn't know
which column means what:

| Key | Type | Airbnb source |
| --- | --- | --- |
| `bedrooms` | text | `og:title` "N bedroom(s)", or "Studio" |
| `beds` | text | `og:title` "N bed(s)" |
| `bathrooms` | text | `og:title` "N bath(s)" / "N private bath" / "N shared bath" |
| `sleeps` | text | deferred state `personCapacity` |
| `rating` | text | JSON-LD `aggregateRating.ratingValue` (fallback: `og:title` "★N") |
| `reviewCount` | text | JSON-LD `aggregateRating.ratingCount` |
| `location` | text | JSON-LD `address.addressLocality`; `twitter:title` "… in City, State, Country" for more |
| `propertyType` | text | `og:title` before " in " (e.g. "Cabin") |
| `latitude`, `longitude` | text | JSON-LD |
| `hotTub`, `pool`, `poolTable`, `petFriendly`, `parking` | boolean | deferred state amenities with `available: true`, matched by title (see below) |

Title: JSON-LD `name`, falling back to `og:description`. Truncate to 200
characters (the item title limit).

Amenity matching, case-insensitive on the amenity `title`:

| Key | Matches |
| --- | --- |
| `hotTub` | "Hot tub" (includes "Private hot tub", "Shared hot tub") |
| `pool` | "Pool", "Private pool", "Shared pool", "… outdoor pool" (not "Pool table") |
| `poolTable` | "Pool table" |
| `petFriendly` | "Pets allowed" |
| `parking` | "Free parking on premises", "Paid parking on premises", "Free street parking" |

Only report a boolean as `true` when the amenity is found and available. Don't
report `false` when it's missing: a missing amenity might be a parse failure,
so leave the column empty.

### Backend

- New package `pkg/linkimport`:
  - `Parse(url) (source, canonicalURL, error)`: recognizes the link and
    returns a canonical URL, or `ErrUnsupported`.
  - One parser per source, `parseAirbnb(html []byte) (Result, error)`, a pure
    function so it's unit-tested on saved HTML.
  - A fetcher behind an interface, so handler tests use a fake.
- **Supported Airbnb links:** `airbnb.{com,ca,co.uk,com.au,…}/rooms/{id}`,
  `/rooms/plus/{id}`, `/luxury/listing/{id}`, `/h/{slug}`, and the short
  `abnb.me/…` links (these redirect; follow at most 3 redirects and only to
  Airbnb hosts). Rebuild the fetch URL as `https://www.airbnb.com/rooms/{id}`
  and drop the query string, so tracking parameters aren't passed along and the
  stored Link is clean. Keep the user's original link in the Link column.
- **Parsing:** use `golang.org/x/net/html` to walk `<meta>` and `<script>`
  tags (`x/net` is the only new dependency; avoid a full scraping library).
  Then `encoding/json` on the JSON-LD and deferred-state scripts. For the
  deferred state, walk the JSON generically looking for `personCapacity` and
  objects with `title` + `available`, rather than hardcoding the nesting path:
  the nesting is the part most likely to change.
- **Each source is optional.** OG alone gives title, bedrooms, beds, baths,
  rating and type. If JSON-LD or the deferred state is missing or changes
  shape, return what OG gave. Only `422` if there's no title at all.
- **SSRF safety:** the server fetches a URL a user typed, so:
  - Only `https`, only hosts on the allowlist (after every redirect).
  - No user-chosen ports; refuse IP-literal hosts.
  - 8-second timeout; stop reading after 2 MB (`io.LimitReader`).
  - Send `Accept-Language: en-US` so the text we match on is English; a plain
    honest `User-Agent` like `JustPickSomething/1.0 (+https://pick.rajwalgautam.com)`.
- **Log** source, status and which parts parsed (OG / JSON-LD / deferred), not
  the URL's query string. A jump in OG-only results is the signal that Airbnb
  changed its page.
- Env vars: `LINK_IMPORT_ENABLED` (`false` → `503`, so the frontend hides the
  feature).

### Limits

Reuse the M5 Postgres rate-limit table:

- Per member: 20 imports per hour.
- App-wide: a daily cap (e.g. 200), so one room can't get our IP blocked by
  Airbnb or use up Vercel execution time.

Each import is one function call of up to 8 seconds, so check it against the
Vercel Hobby execution-time budget alongside [polling](project-plan.md#polling-budget).

### Frontend

- In the table's input row, pasting into the title cell anything that looks
  like a supported link calls `/import` and fills the row: title, Link, and
  every column whose name matches a field key. Show a small spinner in the row
  while it loads. The user checks the row, fills in price, and presses
  **+** as usual.
- Column matching by name, case-insensitive, including the
  [template alternatives](project-plan.md#templates):

  | Field key | Column names |
  | --- | --- |
  | (url) | Link, Listing |
  | `bedrooms` | Bedrooms |
  | `bathrooms` | Bathrooms |
  | `sleeps` | Sleeps, Max guests |
  | `rating` | Guest rating, Review score, Rating |
  | `location` | Location, Neighborhood |
  | `hotTub` / `pool` / `poolTable` / `petFriendly` / `parking` | Hot tub / Pool / Pool table / Pet friendly / Parking |

  Templates are frontend constants, so keep this table next to them in
  `lib/` and give each template column an optional `importKey` instead of
  matching only by name.
- Unmatched fields are dropped. Don't create columns automatically: columns
  are admin-only and shared by everyone.
- Pasting several links at once (one per line) imports them one by one and
  fills several input rows, the same as pasting spreadsheet rows.
- On `400`, add the text as a plain title like today. On `422`/`502`, add the
  link as the title and Link value and show "Couldn't read that listing; fill
  it in by hand." On `503`, never call `/import` again this session.
- VRBO links in phase 1: recognized on the frontend, not sent to `/import`.
  Show a hint: "VRBO doesn't let us read listings automatically; fill it in by
  hand." (Phase 2 replaces this.)

### Tests

- Save a few real Airbnb pages (strip personal info like host names) into
  `pkg/linkimport/testdata/`: a cabin with amenities, a hotel room ("1 private
  bath"), a studio, a page with no JSON-LD. Unit-test each parser layer.
- URL tests: every supported link form, query strings stripped, other hosts
  and IP literals rejected, redirect to a non-Airbnb host rejected.
- Handler tests with the fake fetcher: auth, locked room `409`, `429`, `503`.
- A manual check, not in CI (CI shouldn't hit Airbnb): `make import-check
  URL=…` fetches a live listing and prints the result. Run it after Airbnb
  redesigns or when logs show OG-only results.

## Phase 2 — bookmarklet for VRBO (and Airbnb prices)

Only if phase 1 ships and VRBO links come up often.

A bookmarklet is a bookmark whose URL is a small piece of JavaScript. The user
opens a listing in their own browser, then clicks the "Add to Just Pick
Something" bookmark. Their browser has already loaded the page, past any
captcha, so the script can read what's on it.

- The script reads `og:*` tags, JSON-LD, and for VRBO its embedded page state,
  plus the visible price if dates are selected. It builds the same
  `{source, title, url, fields}` shape as `/import`.
- It opens `https://pick.rajwalgautam.com/import#<base64 JSON>`. Data goes in
  the **URL fragment**, never the query string: fragments aren't sent to any
  server or logged. Keep it small (no
  images, no description) to stay under URL length limits.
- The `/import` page asks which of the user's rooms to add it to (from
  `localStorage`), then shows the prefilled row for review.
- The bookmarklet can't call our API directly: the API only accepts requests
  from `pick.rajwalgautam.com` (see
  [API access](project-plan.md#api-access-https-only-from-pickrajwalgautamcom)).
  Opening our page and handing over the data is what makes it work.
- VRBO's page format needs its own investigation, from a real browser, before
  building this (we couldn't load it from `curl`).
- Downsides: bookmarklets are awkward on phones, and the user has to set one
  up. Show a "drag this to your bookmarks bar" link in the room's help.

## Out of scope

- Prices from Airbnb's private API.
- Getting around VRBO's (or anyone's) captcha or bot protection.
- Paid scraping services.
- Crawling: following links, fetching sub-pages, search results or reviews.
- Storing listing photos or descriptions (a thumbnail URL at most, later).
- Re-fetching listings to keep values up to date. Import is a one-time fill.
- Other sites (Booking.com, hotels, Google Maps for restaurants). The parser
  design takes more sources, but each one needs its own look first.
- Using the AI assistant to read fetched pages.

## Open questions

- **Does Airbnb serve the same page to Vercel's IPs?** Our tests ran from a
  home connection. Datacenter IPs are often treated differently. Test with a
  throwaway Vercel preview deployment before building the rest. If it's
  blocked, phase 1 is dead and the bookmarklet becomes the only path.
- Is the terms-of-service risk acceptable for a personal app? Record the
  decision in the [decisions log](project-plan.md#decisions-log).
- Should import also save `imageUrl` for a thumbnail in the table? (It points
  at Airbnb's CDN; hotlinking it is another terms question.)
- Should the Rental homes template gain Beds and Rating columns, now that
  they'd fill in for free?
