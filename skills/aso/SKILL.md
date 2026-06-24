---
name: aso
description: "When the user wants to audit or optimize an App Store or Google Play listing. Also use when the user mentions 'ASO audit,' 'app store optimization,' 'optimize my app listing,' 'improve app visibility,' 'app store ranking,' 'audit my listing,' 'why aren't people downloading my app,' 'improve my app conversion,' 'keyword optimization for app,' or 'compare my app to competitors.' Use when the user shares an App Store or Google Play URL and wants to improve it."
metadata:
  version: 2.1.0-local.3
  localPatches:
    - "Mandate Astro ASO MCP (mcp__astro__*) as the primary keyword/ranking/competitor source when available"
    - "Apple: parse the `serialized-server-data` SPA block (`data[0].data.shelfMapping.product_media_phone_.items[]`) as the PRIMARY source for screenshots + preview video. iTunes Lookup is stale on modern apps and silently returns empty."
    - "Google Play: parse `AF_initDataCallback` block `ds:5` for screenshots, preview video, icon, feature graphic, description, install range, category, last-updated. Concrete paths in `Fetch the listing`. No JSON-LD on Play."
    - "JSON-LD (Apple only) second for icon + description + aggregateRating; iTunes Lookup third for fields the other two don't expose."
    - "Forbid scoring visual assets as ?/10 unless the SPA block itself is unreachable; only score 0/10 if the block confirms empty items[]"
---

# ASO Audit

Analyze App Store and Google Play listings against ASO best practices. Fetches
live listing data, scores metadata, visuals, and ratings, then produces a
prioritized action plan.

## When to Use

- User shares an App Store or Google Play URL
- User asks to audit or optimize an app listing
- User wants to compare their app against competitors
- User asks about app store ranking, visibility, or download conversion

## Before Auditing

**Check for product marketing context first:**
If `.agents/product-marketing.md` exists (or `.claude/product-marketing.md`, or the legacy `product-marketing-context.md` filename, in older setups), read it before asking questions. Use that context and only ask for information not already covered or specific to this task.

## Phase 1 — Identify Store & Fetch

### Data sources — use in this order

1. **Astro ASO MCP first (when available).** If `mcp__astro__*` tools are exposed, they are the authoritative source for tracked keyword rankings, popularity, difficulty, ratings history, AI keyword suggestions, and competitor app sets. Prefer these over WebFetch for anything keyword-related:
   - `mcp__astro__list_apps` — confirm the app is tracked and resolve the App Store ID
   - `mcp__astro__get_app_keywords` — full ranked keyword set with `currentRanking`, `previousRanking`, `rankingChange`, `popularity`, `difficulty`
   - `mcp__astro__get_app_ratings` — current + historical rating/review counts
   - `mcp__astro__search_rankings` — per-keyword detail with optional `includeHistory`/`includeStatistics`
   - `mcp__astro__search_app_store` — top-N apps for a keyword (competitor extraction); pass `appId` to also get your app's position
   - `mcp__astro__get_keyword_suggestions` — AI-suggested keywords with pop/difficulty
   - `mcp__astro__extract_competitors_keywords` — keyword ideas mined from competitors ranking for a tracked term
2. **Apple iTunes Lookup API + Google Play JSON-LD second.** These are programmatic and expose creative URLs (icon, screenshots, preview video) that the rendered page does not — see "Fetch the listing" below.
3. **WebFetch on the public listing third.** Useful for human-visible truth (placeholder `1x1.gif` blocks vs real screenshots, "What's New" copy, brand voice) but the page renders client-side so it will under-report creatives that ARE live.

If Astro MCP is not available, skip step 1 and proceed with steps 2 and 3.

### Detect store type from URL

```
Apple:  apps.apple.com/{country}/app/{name}/id{digits}
Google: play.google.com/store/apps/details?id={package}
```

If the user gives an app name instead of a URL, search the web for:
`site:apps.apple.com "{app name}"` or `site:play.google.com "{app name}"`

### Fetch the listing

**Apple — primary source (parse the SPA hydration block).** iTunes Lookup is stale for modern apps and will silently return `screenshotUrls: []` and omit `previewMovieUrl` even when both are live. The reliable source is the App Store SPA's own hydration data, embedded as a `<script>` block on the public listing page.

Fetch the listing HTML with a standard browser User-Agent:

```
curl -fsSL "https://apps.apple.com/{cc}/app/_/id{APP_ID}" \
  -A "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0 Safari/537.36"
```

Extract the block:

```python
import re, json
m = re.search(r'<script[^>]*id="serialized-server-data"[^>]*>(.*?)</script>', html, re.DOTALL)
obj = json.loads(m.group(1).strip())
```

Navigate to `obj["data"][0]["data"]["shelfMapping"]`:

- `product_media_phone_.items[]` — iPhone media in display order. Each item has either a `screenshot` key (image, with `template`, `width`, `height`) or a `video` key (with `videoUrl` pointing to an `.m3u8` and `preview.template` for the poster image). **Count screenshots and check for the video here — do NOT trust iTunes Lookup.**
- `product_media_pad_.items[]` — iPad media in the same shape.
- The `template` URL has placeholders like `/{w}x{h}{c}.{f}` — substitute (e.g. `/1290x2796bb.png` or `/600x600wa.webp`) to fetch the actual image.

**Apple — secondary source (JSON-LD).** The listing page also contains a `<script type="application/ld+json">` block with a `SoftwareApplication` schema. Use it for: `name`, `description`, `image` (icon URL), `aggregateRating.ratingValue`, `aggregateRating.reviewCount`, `offers.price`, `applicationCategory`.

**Apple — tertiary source (iTunes Lookup API).** Use this only for fields the other two don't expose: `languageCodesISO2A[]` (every localization), `releaseNotes`, `version`, `currentVersionReleaseDate`, `minimumOsVersion`, `fileSizeBytes`, `contentAdvisoryRating`, `sellerName`, `genres[]`. **Do NOT use it for `screenshotUrls` or `previewMovieUrl`** — those will be empty/missing even when the assets are live.

```
https://itunes.apple.com/lookup?id={APP_ID}&country={CC}
```

**Google Play — primary source (parse the `AF_initDataCallback` `ds:5` block).** Play stopped embedding usable JSON-LD years ago. The listing page instead embeds a series of `AF_initDataCallback({key: 'ds:N', ..., data: <JSON>, sideChannel: {}})` calls. Block `ds:5` carries the app detail tree.

Fetch the listing HTML with a standard browser User-Agent (use `&hl=en&gl=us` to pin locale):

```
curl -fsSL "https://play.google.com/store/apps/details?id={pkg}&hl=en&gl=us" \
  -A "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0 Safari/537.36"
```

Extract every block, then drill into `ds:5`:

```python
import re, json
blocks = {}
for m in re.finditer(r"AF_initDataCallback\(\{key:\s*'(ds:\d+)'.*?data:\s*(\[.*?\])\s*,\s*sideChannel:", html, re.DOTALL):
    try: blocks[m.group(1)] = json.loads(m.group(2))
    except: pass
ds5 = blocks['ds:5']
def at(o, *path, default=None):
    try:
        for p in path: o = o[p]
        return o
    except: return default
```

Known paths inside `ds:5[1][2]` (battle-tested against `com.mobiletechmedia.realadai` 2026-06-04):

| Field | Path | Notes |
|---|---|---|
| Title (30 chars) | `[0][0]` | |
| Content rating | `[9][0]` | e.g. `"Everyone"` |
| Released date | `[10][0]` | formatted string, e.g. `"Jan 6, 2026"` |
| Install range | `[13][0]` | e.g. `"100+"`, `"10K+"`, `"1M+"` |
| Rating value | `[51][0][1]` | `null` if 0 ratings |
| Rating count | `[51][2][1]` | `null` if 0 ratings |
| Formatted price | `[57][0][0][0][0][1][0][0]` | `"0"` for free |
| Developer name | `[68][0]` | |
| Full description (HTML) | `[72][0][1]` | supports `<b>`, `<i>`, `<u>`, `<br>`, `<h1>`–`<h3>` |
| **Screenshots[]** | `[78][0]` | list of items, each has the URL at `item[3][2]` — len is the screenshot count (Play max 8 per device; 10 was seen for phone) |
| Primary category | `[79][0][0][0]` | e.g. `"Video Players & Editors"` |
| Icon URL | `[95][0][3][2]` | 512×512 |
| Feature graphic URL | `[96][0][3][2]` | 1024×500 — absent = missing feature graphic (Play penalty) |
| Preview video URL | `[100][0][0][3][2]` | YouTube watch URL — Play uses YouTube for previews |
| Preview video thumbnail | `[100][1][0][3][2]` | poster image |
| What's New | `[144][1][1]` | per-locale release notes; `null` if not set |

If JSON-LD is also present (some legacy listings), fall back to it. Otherwise grep for `play-lh.googleusercontent.com` URLs in the raw HTML to at least confirm presence.

Then use WebFetch on the public page only if you need fields the above don't expose (visible review text, etc.):

**Apple App Store fields:**

- App name (title) — 30 char limit
- Subtitle — 30 char limit
- Description (long) — not indexed for search, but matters for conversion
- Promotional text — 170 chars, updatable without new release
- Category (primary + secondary)
- Screenshots (count, order, caption text)
- Preview video (presence, duration)
- Rating (average + count)
- Recent reviews (visible ones)
- Price / in-app purchases
- Developer name
- Last updated date
- Version history notes
- Age rating
- Size
- Languages / localizations listed
- In-app events (if any visible)

**Google Play fields:**

- App name (title) — 30 char limit
- Short description — 80 char limit
- Full description — 4,000 char limit, IS indexed for search
- Category + tags
- Feature graphic (presence)
- Screenshots (count, order)
- Preview video (presence)
- Rating (average + count)
- Recent reviews (visible ones)
- Price / in-app purchases
- Developer name
- Last updated date
- What's new text
- Downloads range
- Content rating
- Data safety section
- Languages listed

If WebFetch returns incomplete data (stores render client-side), note gaps and
work with what's available. Ask the user to paste missing fields if critical.

### Visual asset assessment

**Always verify presence programmatically first — never punt to `?/10`.**

1. **Apple:** from the SPA hydration block at `data[0].data.shelfMapping`:
   - `product_media_phone_.items[]` → count items with `screenshot` key (= iPhone screenshots), check for any item with `video` key (= preview video, with `videoUrl` pointing to the `.m3u8`)
   - `product_media_pad_.items[]` → iPad screenshots / preview video
   - From the JSON-LD block: `image` → app icon URL
   - **DO NOT trust iTunes Lookup `screenshotUrls` / `previewMovieUrl`** — they will return empty even when the assets are live. The hydration block is the source of truth.
2. **Google Play:** from `AF_initDataCallback` `ds:5[1][2]`, count `[78][0]` (screenshots), check `[100][0][0][3][2]` (YouTube preview URL — null/absent = no video), `[95][0][3][2]` (icon), `[96][0][3][2]` (feature graphic — absent is a real ranking penalty on Play). See concrete code in `Fetch the listing` above. DO NOT use JSON-LD on Play — the listing no longer emits it.
3. **Caption text + visual quality** (composition, message hierarchy, device frames) requires rendering an actual image — fetch each screenshot URL using the `template` with substituted dimensions (e.g. `1290x2796bb.png`), or have the user share one. The image template URL is the per-screenshot canonical reference; the dimensions `{w}x{h}` are interchangeable.

**Scoring rule:** score `0/10` ONLY if `product_media_phone_.items[]` is empty or missing — and even then, double-check the JSON-LD `screenshot[]` field. `?/10` is reserved strictly for: hydration block could not be fetched, parse failed, or the app is region-locked from the auditor's vantage point. **Never score `?/10` because iTunes Lookup said zero** — iTunes Lookup is wrong for screenshots on modern apps.

**Promotional text (Apple):** This 170-char field appears above the description
but is often indistinguishable from it in scraped HTML. If you cannot confirm
its presence, note this and recommend the user check App Store Connect.

---

## Phase 1.5 — Assess Brand Maturity

Before scoring, classify the app into one of three tiers. This determines how
you interpret "textbook ASO" deviations — a deliberate brand choice by a
household name is not the same as a missed opportunity by an unknown app.

### Tier definitions

| Tier            | Signals                                                                                                                              | Examples                                    |
| --------------- | ------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------- |
| **Dominant**    | Household name, 1M+ ratings, top-10 in category, near-universal brand recognition. Users search by brand name, not generic keywords. | Instagram, Uber, Spotify, WhatsApp, Netflix |
| **Established** | Well-known in their category, 100K+ ratings, strong organic installs, recognized brand but not universally known.                    | Strava, Notion, Duolingo, Cash App, Calm    |
| **Challenger**  | Building awareness, <100K ratings, needs discovery through keywords and ASO tactics. Most apps fall here.                            | Your app, most indie/startup apps           |

### How tier affects scoring

**Dominant apps** get adjusted scoring in these areas:

- **Title:** Brand-only or brand-first titles are valid (score 8+ if brand is the keyword). These apps don't need generic keyword discovery.
- **Description:** Score purely on conversion quality, not keyword presence. If the app is a household name, a well-crafted brand description beats a keyword-stuffed one.
- **Visual Assets:** Lifestyle/brand photography instead of UI demos is a legitimate conversion strategy. No video is acceptable if the product is hard to demo in 30s or brand awareness is near-universal.
- **What's New:** Generic release notes at weekly+ cadence are acceptable (score 8+). At scale, detailed changelogs have minimal ROI and risk backlash.
- **In-app events:** Missing events for utility apps with massive install bases (Uber, WhatsApp) is not a penalty. These apps don't need discovery help.
- **Localization:** Score relative to actual market, not absolute count. A US-only fintech with 2 languages (English + Spanish) is appropriately localized.

**Established apps** get partial adjustment:

- Brand-first titles are fine but should still include 1-2 keywords
- Strategic description choices get benefit of the doubt
- Other dimensions scored normally

**Challenger apps** are scored strictly against textbook ASO best practices — every character, screenshot, and keyword matters.

**Key principle:** Before docking points, ask: "Is this a mistake or a deliberate
choice by a team that has data I don't?" If the app has 1M+ ratings and a
dedicated ASO team, assume their choices are data-informed unless clearly wrong.

---

## Phase 2 — Score Each Dimension

Score each dimension 0-10 using the criteria in `references/scoring-criteria.md`.
Apply the brand maturity tier adjustments from Phase 1.5.

Reference files for platform specs and benchmarks:

- `references/apple-specs.md` — Official Apple character limits, screenshot/video specs, CPP/PPO rules, rejection triggers
- `references/google-play-specs.md` — Official Google Play limits, screenshot specs, Android Vitals thresholds, policies
- `references/benchmarks.md` — Conversion data, rating impact, video lift, screenshot behavior, CPP/event benchmarks

### Dimensions and Weights

| #   | Dimension            | Weight | What It Covers                                                            |
| --- | -------------------- | ------ | ------------------------------------------------------------------------- |
| 1   | Title & Subtitle     | 20%    | Character usage, keyword presence, clarity, brand + keyword balance       |
| 2   | Description          | 15%    | First 3 lines, keyword density (Google), CTA, structure, promotional text |
| 3   | Visual Assets        | 25%    | Screenshot count/quality/messaging, video, icon, feature graphic          |
| 4   | Ratings & Reviews    | 20%    | Average rating, volume, recency, developer responses                      |
| 5   | Metadata & Freshness | 10%    | Category choice, update recency, localization count, data safety          |
| 6   | Conversion Signals   | 10%    | Price positioning, IAP transparency, social proof, download range         |

**Final score** = weighted sum, out of 100.

### Score interpretation

| Score  | Grade | Meaning                                                   |
| ------ | ----- | --------------------------------------------------------- |
| 85-100 | A     | Well-optimized; focus on A/B testing and iteration        |
| 70-84  | B     | Good foundation; clear opportunities to improve           |
| 50-69  | C     | Significant gaps; prioritized fixes will have high impact |
| 30-49  | D     | Major optimization needed across multiple dimensions      |
| 0-29   | F     | Listing needs a complete overhaul                         |

---

## Phase 3 — Competitor Comparison (Optional)

If the user provides competitor URLs or asks for comparison:

1. Fetch 2-3 top competitors in the same category
2. Run the same scoring on each
3. Build a comparison table highlighting where the user's app is weaker/stronger
4. Identify keyword gaps — terms competitors rank for that the user's app doesn't target

If no competitors are specified, suggest the user provide 2-3 or offer to search
for top apps in their category.

---

## Phase 4 — Generate Report

Use the template in `references/report-template.md` to structure the output.

The report must include:

1. **Score card** — table with all 6 dimensions, scores, and grade
2. **Top 3 quick wins** — changes that take <1 hour and have highest impact
3. **Detailed findings** — per-dimension breakdown with specific issues and fixes
4. **Keyword suggestions** — based on title/description analysis and competitor gaps
5. **Visual asset recommendations** — specific screenshot/video improvements
6. **Priority action plan** — ordered list of changes by impact vs effort

### Report rules

- Every recommendation must be **specific and actionable** ("Change subtitle from X to Y" not "Improve subtitle")
- Include character counts for all text recommendations
- Flag platform-specific differences (Apple vs Google) when relevant
- Note what CANNOT be assessed without paid tools (search volume, exact rankings)
- When suggesting keyword changes, explain WHY each keyword matters

---

## Platform-Specific Rules

### Apple App Store — Key Facts

- Title (30 chars) + Subtitle (30 chars) + Keyword field (100 **bytes**, hidden) = indexed text
- Keywords field is bytes not chars — Arabic/CJK use 2-3 bytes per char
- Long description is NOT indexed for search — optimize for conversion only
- Promotional text (170 chars) does NOT affect search (Apple confirmed)
- Never repeat words across title/subtitle/keyword field (Apple indexes each word once)
- Keyword field: commas, no spaces ("photo,editor,filter" not "photo, editor, filter")
- Screenshots: up to 10 per device. First 3 visible in search — 90% never scroll past 3rd
- Screenshot captions indexed since June 2025 (AI extraction)
- In-app events: max 10 published at once, max 31 days each. Indexed and appear in search
- Custom Product Pages (up to 70) in organic search since July 2025. +5.9% avg conversion lift
- App preview video: up to 3, 15-30s each. Autoplays muted — +20-40% conversion lift
- SKStoreReviewController: max 3 prompts per 365 days
- Apple has human editorial curation — quality and design matter more
- See `references/apple-specs.md` for full specs, dimensions, and rejection triggers

### Google Play — Key Facts

- Title (30 chars) + Short description (80 chars) + Full description (4,000 chars) = indexed text
- Full description IS indexed — target 2-3% keyword density naturally
- No hidden keyword field — all keywords must be in visible text
- Google NLP/semantic understanding — keyword stuffing detected and penalized
- Prohibited in title: emojis, ALL CAPS, "best"/"#1"/"free", CTAs (enforced since 2021)
- Screenshots: min 2, **max 8** per device (not 10 like Apple)
- Feature graphic (1024x500, exact) required for featured placements
- Video does NOT autoplay — only ~6% of users tap play (low ROI vs iOS)
- Android Vitals directly affect ranking: crash >1.09% or ANR >0.47% = reduced visibility
- Promotional Content: submit 14 days early for featuring. Apps see 2x explore acquisitions
- Custom Store Listings: up to 50 (can target churned users, specific countries, ad campaigns)
- Store Listing Experiments: test up to 3 variants, run 7+ days, 1 experiment at a time
- See `references/google-play-specs.md` for full specs and policy details

### What Apple Indexes vs What Google Indexes

| Field                 | Apple Indexed?   | Google Indexed?        |
| --------------------- | ---------------- | ---------------------- |
| Title                 | Yes              | Yes (strongest signal) |
| Subtitle / Short desc | Yes              | Yes                    |
| Keyword field         | Yes (hidden)     | Does not exist         |
| Long description      | No               | Yes (heavily)          |
| Screenshot captions   | Yes (since 2025) | No                     |
| In-app events         | Yes              | N/A (LiveOps instead)  |
| Developer name        | No               | Partial                |
| IAP names             | Yes              | Yes                    |

---

## Common Issues Checklist

Flag these if found. Items marked _(tier-dependent)_ should be evaluated against
the app's brand maturity tier — they may be deliberate choices for Dominant apps.

**Always flag (all tiers):**

- [ ] Rating below 4.0
- [ ] Last update > 3 months ago
- [ ] Google Play description has no keyword strategy (under 1% density)
- [ ] Google Play missing feature graphic
- [ ] Apple keyword field likely has repeated words (inferred from title+subtitle)
- [ ] Category mismatch — app would face less competition in a different category
- [ ] Fewer than 5 screenshots

**Flag for Challenger/Established only** _(not mistakes for Dominant apps):_

- [ ] Title wastes characters on brand name only (no keywords) _(Dominant: brand IS the keyword)_
- [ ] Subtitle/short description duplicates title keywords
- [ ] Description first 3 lines are generic _(Dominant: may be brand-voice choice)_
- [ ] No preview video _(Dominant: may be rational if product is hard to demo)_
- [ ] Screenshots are just UI dumps with no messaging/captions _(Dominant: lifestyle/brand shots may convert better)_
- [ ] Only 1-2 localizations _(score relative to actual market, not absolute count)_
- [ ] No in-app events or promotional content _(Dominant utility apps may not need discovery help)_

**Flag for all tiers but note context:**

- [ ] No developer responses to negative reviews _(note volume — responding at 10M+ reviews is a different challenge than at 1K)_
- [ ] Generic "What's New" text _(acceptable at weekly+ release cadence for Established/Dominant)_

---

## Task-Specific Questions

1. What is the App Store or Google Play URL?
2. Is this your app or a competitor's?
3. What category does the app compete in?
4. Do you have competitor URLs to compare against?
5. Are you focused on search visibility, conversion rate, or both?
6. Do you have access to App Store Connect or Google Play Console data?

---

## Related Skills

- **cro**: For optimizing the conversion of web-based landing pages that drive app installs
- **ad-creative**: For creating App Store and Google Play ad creatives
- **analytics**: For setting up install attribution and in-app event tracking
- **customer-research**: For understanding user needs and language to inform listing copy
