# Changelog

All notable changes to [ren.ph](https://ren.ph) are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [2.15.2] - 2026-05-26

### Fixed
- The Makati city zonal value page and several Makati barangay pages were surfacing 1987 to 1998 prices alongside the current 2022 to 2024 prices, dragging visible floor values down to misleading levels. The Makati listing showed 32 barangay cards instead of the 23 real Makati barangays, and the city page strip reported a minimum residential value around PHP 6,000 to PHP 25,000 per sqm pulled from a 1987 LEGASPI VILL row. Direct URLs like `/tools/zonal-value/metro-manila/city-of-makati/legaspi-village`, `/salcedo-village`, `/ayala-center`, `/makati-commercial-center`, `/makati-comml-center`, `/arnaiz-p-del-pilar-north`, `/arnaiz-p-del-pilar-south`, `/legaspi-vill`, and `/rizal` all served full Zonal Values pages on barangays that do not exist. These nine entries were BIR zone labels (vicinity-style section headers from older Department Orders like DO 23-12) that the parser persisted as if they were real barangays, carrying values from BIR sheets dating back to 1987. A blanket fix to hide every ungrounded barangay nationwide would have broken 70 other city pages where most barangays are legitimately ungrounded due to PSGC versus BIR naming mismatches (Cebu City, Angeles City, Cavite City, etc.), so this release ships a targeted per-row exclude list for the nine audited Makati fakes only. The exclude list is wired into the Makati city page card grid, direct-URL barangay access (now returns the Not Found template), the static build params, the sitemap, and the autocomplete search results. The homepage summary count of total barangays subtracts the fakes too, and the Makati city page stats strip (averages, minimum, maximum, median, total barangay count) is recomputed in JavaScript over the filtered list so the numbers no longer pull from filtered-out rows. As a side effect of the recompute, Makati's reported residential and commercial averages now fold the BIR condo classifications (RC and CC) into their parent classes, matching the province card behavior already in place since v2.10.x but ahead of the rest of the country; expect Makati's numbers to read higher post-deploy versus the previous build. Cached pages drain on cache TTL or next deploy. The permanent fix (parser-side rejection of ungrounded NCR barangay candidates plus a clean reground) is drafted and will collapse the exclude list to empty when it lands.

## [2.15.1] - 2026-05-26

### Fixed
- Internal-only: `scripts/validate-zonal-staging.ts` pre-promote barangay-count range lowered from 40,000-44,000 to 36,000-42,000 to match actual prod state. The previous range was based on aspirational CLAUDE.md guardrails that never reflected reality (prod has run 36,251 to 39,434 barangays across the 2026 series, currently 38,879 since v2.14.0). False-FAILed the first Phase 4 zonal-staging cutover. No runtime or user-facing change; CLI tooling only.

## [2.15.0] - 2026-05-26

### Added
- Zero-downtime zonal data reimports via a parallel `zonal_staging` schema and atomic schema swap. Previously, every reimport ran `TRUNCATE ... CASCADE` against the live zonal tables and re-inserted at ~250 rows/batch for 10 to 15 minutes; during that window any SSR hit on a half-filled city or barangay returned 0 rows, the OpenNext page cache locked in that empty response, and the affected pages stayed stale until cache TTL or the next deploy. Several user-visible "this barangay shows 0 streets" incidents in the v2.11 to v2.13 series traced back to this exact window. The new path mirrors the 6 active zonal tables (regions, provinces, cities, barangays, values, slug_aliases) in a separate `zonal_staging` schema, runs the reimport against the mirror while the live `public.zonal_*` keeps serving traffic, validates the staged result against per-table count ranges (cities 1,500 to 1,700, barangays 40,000 to 44,000, etc.), then atomically promotes via `ALTER TABLE ... SET SCHEMA public` inside a single transaction. The cut is sub-second instead of 10 to 15 minutes; no user-visible window with empty tables. Operator-curated `zonal_slug_aliases` entries are preserved across the swap via in-transaction snapshot-and-restore so manual city-rename redirects don't get wiped. The 4 FK constraints, 24 RLS policies, and all indexes follow the table through `SET SCHEMA` by OID, empirically verified on local Supabase via a 3-cycle dress rehearsal. The previous `TRUNCATE` path remains available for now; the staging path will become the default after two clean prod cutovers.

### Fixed
- Production builds were sometimes shipping stale zonal data even after a fresh import landed in the database. Next.js `unstable_cache` persisted Supabase fetch results in `.next/cache/fetch-cache` across builds, so the build that immediately followed a reimport would render listing pages from the PRIOR import's data and the resulting HTML went out to the edge. v2.14.0's deploy hit this: three fake Metro Manila cities were removed in the DB and the build went out with the pre-removal HTML because the build re-used cached fetches. The prod build script now clears `.next/cache/fetch-cache` before every build (webpack/turbopack incremental compilation cache survives, so only fetch results are dropped). Every deploy from this version forward picks up the live database state at build time.
- Migration `00035_zonal_province_misattribution_fix.sql` hardcoded 11 production region UUIDs that don't exist on a fresh local Supabase, breaking `supabase db reset` on a clean clone with a foreign-key violation. The fix rewrites PHASE A as a slug-based JOIN against `zonal_regions`, so it silently no-ops on fresh local (empty regions table) and continues to insert the 11 missing provinces correctly on production-like data. Pure dev-experience improvement; production was already past this migration before the rewrite.

## [2.14.0] - 2026-05-26

### Fixed
- Three URLs under Metro Manila served zonal value pages titled with raw BIR footnote text, presenting non-existent jurisdictions as if they were real cities: `/tools/zonal-value/metro-manila/belongs-to-makati` showed up as "BIR Zonal Values in ** -  BELONGS TO MAKATI, Metro Manila" with 27 fake barangays under it; `/tools/zonal-value/metro-manila/unang-hakbang-under-jurisdiction-of-quezon` showed "***UNANG HAKBANG UNDER JURISDICTION OF QUEZON, Metro Manila" with 3 fake barangays; and `/tools/zonal-value/metro-manila/all-lots-within-global` showed "ALL LOTS WITHIN GLOBAL, Metro Manila" with one barangay (Fort Bonifacio) orphaned from its real parent city Taguig. All three came from the same parser defect: `_extract_bare_city_header` accepted any single-cell row that ended in the literal word "CITY" as a city header, so BIR footnote prose like "** - Belongs to Makati City" (RDO 51 Pasay, DO 38-09), "***UNANG HAKBANG under jurisdiction of Quezon City" (RDO 32 Quiapo, DO 21-19), and "ALL LOTS WITHIN GLOBAL CITY" (RDO 44 Taguig, DO 005-24) all got promoted to standalone cities, and the next BARANGAY: rows in the sheet then attached under those fake cities. The same defect at barangay level was fixed in v2.13.6 with an annotation-pattern classifier on the BARANGAY: label path; this release wires the same classifier to the city-header paths (bare-city and labeled CITY/MUNICIPALITY), reusing the existing pattern blocklist (BELONGS TO, UNDER JURISDICTION OF, ALL LOTS, etc.) since all three cases match patterns that were already present. A post-parse safety-net pass also drops any ungrounded city whose display_name matches the blocklist, in case future BIR formatting variants slip the parse-time gate. Fort Bonifacio reattaches to Taguig automatically (its 19 streets under the fake city were mostly duplicates of streets already filed correctly under Taguig's real Fort Bonifacio barangay, and the dedup pass merges the overlap). The 3 fake city URLs serve the Not Found template after the next zonal data refresh. All 1,639 real Metro Manila and nationwide cities are unchanged, and the mis-regioned-but-real Rizal cohort (Antipolo, Rodriguez, San Mateo, Teresa, separately tracked) is explicitly preserved by the classifier. Corrections apply on the next zonal data refresh.

## [2.13.7] - 2026-05-25

### Fixed
- Eight numbered barangays in Pasay City stopped appearing in the zonal value data after the v2.13.6 parser change took effect. Pasay has 201 numbered barangays per PSGC; the rebuild surfaced only 177 of them, dropping `Barangay 183`, `Barangay 187` through `Barangay 191`, `Barangay 197`, and `Barangay 201` outright (43 streets total). Cause: the v2.13.6 annotation classifier added a regex that was meant to catch zone-range descriptors like `195 TO 196 & 198 ZONE 20`, but it also matched the BARANGAY: header cell in BIR's newer Pasay Department Order 43-2023, which names every real barangay as `<N>     ZONE <Z>` (e.g. `183 ZONE 19`). 498 real Pasay rows were silently rejected; older Department Orders (9-89, 19-93, 38-09, 36-2018) papered over most of the loss with fallback data, but eight barangays only exist in the 2023 sheet and disappeared. Patch tightens the regex to require an explicit multi-number separator (comma, ampersand, or hyphen between digits) before `ZONE`, so single-number rows pass through. The two patterns that catch real range descriptors (`TO`-form and `,`-form) are unchanged. After the fix: all 8 barangays recovered, and two of them (Barangay 191, Barangay 197) now show more streets than they did before v2.13.6 because the newer 2023 Department Order data finally flows through. Corrections apply on the next zonal data refresh.

## [2.13.6] - 2026-05-25

### Fixed
- Several barangay zonal value pages in Metro Manila listed real-looking street data under a fake barangay name. Paranaque had `MARCELO` (1993 values from PHP 500 to 3,000 per sqm) sitting next to the real `Marcelo Green Village` (2023 values from PHP 24,000 to 179,000 per sqm); the 1987 BIR Department Order 85-87 used the legacy short form `MARCELO (7)` for Zone 7, which the parser was treating as a separate barangay. The same DO's row for San Martin De Porres read `SAN M DE PORES (13)` (BIR Zone 13), which surfaced as a fake `/san-m-de-pores` page next to the real `/san-martin-de-porres`. Paranaque, Makati, Pasay, Muntinlupa, Manila, and Taguig also had fake barangay pages built from BIR category labels like `LIST OF CONDOMINIUMS (CCT)`, `VARIOUS BARANGAYS (MAIN THOROUGHFARES)`, `POST PROPER NORTH SIDE`, and zone-range descriptors like `195 TO 196 & 198 ZONE 20`. DO 23-12 (East Makati, 2012) even wrote its own disclaimer in the source spreadsheet: "AYALA CENTER (MAKATI COMMERCIAL CENTER) IS NOT A BARANGAY BUT ONLY A PORTION OF BARANGAY SAN LORENZO". The parser was reading whatever followed `BARANGAY :` as a barangay name and trusting it. The parser now gates the `BARANGAY :` label match against a blocklist for these category and zone-descriptor patterns, drops the resulting empty-name barangays in a post-parse pass, and folds away legacy short-form names that are word-prefixes of a modern barangay in the same city (Iloilo's `LEGASPI`, `PALE`, `YULO`, `BURGOS`, `LOBOC` and Calaca's `CORAL` clear up in the same pass, alongside Paranaque's MARCELO). About 207 fake barangay pages drop nationwide. Corrections apply on the next zonal data refresh.

## [2.13.5] - 2026-05-24

### Fixed
- Quezon City had two distinct orphan-barangay artifacts in the zonal value data, both originating in the 1988 Department Order 138-87 sheet. First, the Alta Vista subdivision in White Plains and Loyola Heights was parsed as its own BARANGAY: header in the 1988 sheet, so it surfaced as `/tools/zonal-value/metro-manila/quezon-city/alta-vista` with three streets all priced at PHP 1,000 per sqm in 1988 currency, while the real Alta Vista Subdivision rows under Loyola Heights and White Plains carry current 2024 values. Second, a fake city `boundary-of-caloocan-and-quezon` held 41 barangays at 1988 PHP 500-1,000 per sqm values, e.g. `/tools/zonal-value/metro-manila/boundary-of-caloocan-and-quezon/holy-spirit` showed PHP 500-1,000 per sqm while the real `/tools/zonal-value/metro-manila/quezon-city/holy-spirit` showed PHP 15,000-126,000 per sqm. The fake city came from the parser misreading the second line of a multi-line VICINITY: description ("ALONG QUIRINO HIGHWAY AND KATIPUNAN ROAD, / BOUNDARY OF CALOOCAN AND QUEZON CITY") as a city header, then attributing the next 40+ barangays under it. The parser now disqualifies a bare-city-header candidate when its row is indented and sits inside an open VICINITY: continuation block (catches the boundary-of-caloocan signature at source). A separate post-parse safety net drops an ungrounded pre-1995 barangay when its name appears as a subdivision-suffix street or vicinity (SUBD, SUBDIVISION, VILLAGE, HOMES, HEIGHTS, PARK) in another barangay of the same city (catches Alta Vista). Both fix layers ride the next zonal data refresh. Both URL families stop being generated and drop from the sitemap; cached pages drain on cache TTL or next deploy.

## [2.13.4] - 2026-05-24

### Fixed
- Some barangay zonal value pages still showed a stale floor price (around PHP 3,500-6,000 per sqm from 1988-1993 Department Orders) next to the correct modern values (PHP 55,000-350,000 per sqm from 2021-2024 DOs), e.g. Makati's Guadalupe Nuevo showed "PHP 3,500 to PHP 350,000 per sqm" because 47 stale 1991 EDSA streets sat under the same barangay as the 2021 streets in other vicinities. The previous fix in 2.13.3 dedupes same-named streets across vicinity renames, but it cannot reach this pattern: an entire batch of old-DO rows preserved under one vicinity label with no same-name twin in the modern DO. The parser now drops a vicinity outright when every street under it carries an older Department Order than the barangay's dominant DO and the values match the urban stale-DO clash signature (a cluster of at least 3 low rows under PHP 10,000 alongside another row in the same barangay above PHP 30,000 at a 5x ratio). A stricter variant of the same rule fires when a single vicinity contains both modern and stale rows: only the stale older-DO rows are dropped, and the in-vicinity modern rows survive (e.g. Quezon City's San Bartolome had 26 stale 1993 rows at PHP 1,200-6,000 alongside 2 modern 2024 rows at PHP 39,000 under the same vicinity label). About 1,300 stale street rows across roughly 200 barangays nationwide will be repaired in this pass, concentrated in Metro Manila (Makati's Guadalupe Nuevo, plus Quezon City's San Bartolome, Matandang Balara, Paraiso, and several Manila numbered barangays). A smaller residual class where stale rows spread thinly across multiple vicinities below the 3-row cluster threshold is deferred to a follow-up. Corrections apply on the next zonal data refresh.

## [2.13.3] - 2026-05-23

### Fixed
- Quezon City barangay zonal value pages still surfaced a stale 1988-era floor (around PHP 2,500-6,000 per sqm) next to the correct 2024 values (PHP 55,000-350,000 per sqm) on the same street, e.g. Bagong Lipunan ng Crame showed "PHP 2,500 to PHP 170,000 per sqm" because the page reported the lowest street as the floor. The duplicates come from BIR Department Orders that rename the vicinity label across revisions, so the same physical street ends up as two database rows under different vicinity labels (1ST AVENUE under "CUBAO" at PHP 2,500 from an older DO, and under "BONNY SERRANO - NORTH RD." at PHP 55,000 from the 2024 DO). The street dedup step required vicinity labels to look alike before treating two same-name rows as a stale pair, so vicinity renames slipped through. The dedup now also fires on dissimilar-vicinity pairs when the values match the urban DO-vintage clash signature (residential ratio at least 5x with the low side under PHP 6,000 and the high side over PHP 30,000). The signature does not occur naturally in legitimate multi-vicinity streets, so rural agricultural sub-categories (Fishpond, Riceland, etc.) and other within-DO variation are not affected. About 843 stale street rows across 150 barangays nationwide will be repaired in this pass, overwhelmingly Quezon City (122 barangays) with a few in Manila, Makati, Cebu City, and Cagayan de Oro. Corrections apply on the next zonal data refresh.

## [2.13.2] - 2026-05-23

### Fixed
- Some barangay zonal value pages were still serving 1988-1993 prices (around PHP 2,500-6,000 per sqm) when the 2024 prices (PHP 55,000-350,000 per sqm) for the same barangay were available, e.g. Quezon City's Bagong Lipunan ng Crame. The parser keeps the newest Department Order per barangay, but it identifies barangays by their PSGC code, and older DOs sometimes spell the barangay differently from the current one ("B. Lipunan ng Crame" vs "Bagong Lipunan ng Crame", "St Ignatius" vs "Saint Ignatius", "Old Balara" vs "Matandang Balara"). The old spelling failed PSGC matching, the entry got a synthetic placeholder identity, and the "newest DO wins" step kept it alive alongside the correctly identified 2024 entry, so the page showed both. The barangay matcher now expands the periodless abbreviations "ST"/"SN" to "Saint"/"San" and applies a small curated translation map (Pagasa to Bagong Pag-asa, Bagong Buhay to Bagumbuhay). A safety net then folds an unmatched barangay onto a matched one in the same city when they share an exact rare token, with strict uniqueness guards so distinct barangays (e.g. San Jose vs San Jose Sico) are never merged. About 335 affected barangays across 123 cities were repaired in this pass. Corrections apply on the next zonal data refresh.

## [2.13.1] - 2026-05-23

### Fixed
- Barangay zonal value pages in the 126 cities covered by more than one BIR Revenue District Office (including Manila, Quezon City, and Makati) listed every RDO for the whole city, e.g. "RDO 028, 038, 039, 040", as if a single office governed the barangay. A given barangay's values come from only one or two of those offices, so the page now reads "sourced from the BIR RDOs that cover {city}" instead of implying one jurisdiction. Single-RDO barangays are unchanged. (Display fix; per-street RDO attribution is planned separately.)

## [2.13.0] - 2026-05-23

### Added
- The Pag-IBIG Housing Loan Calculator now includes a step-by-step sample computation (a worked PHP 2.25M example with a year-by-year breakdown) and a quick-reference monthly amortization table covering common loan amounts at both the Standard 6.5% rate and the 4PH 3% rate. Both are computed from the same rate data the calculator uses, so they stay accurate, and they render as static HTML for search engines and AI Overviews.
- BreadcrumbList structured data and an author reference on the calculator page, alongside the existing WebApplication, FAQPage, and HowTo schemas.
- Internal links to the Pag-IBIG calculator from the homepage tools grid, the Transfer Tax Calculator, the BIR Zonal Value lookup, and the broker and License to Sell verification pages.

### Changed
- Updated the Pag-IBIG calculator's page title to lead with the 4PH 3% rate, and added a page-specific Twitter/X card so shared links no longer fall back to the site-wide default.
- Removed the generic site-wide meta keywords tag, which search engines ignore.

## [2.12.1] - 2026-05-22

### Fixed
- The barangays section of the sitemap shipped in 2.12.0 listed only 1,000 barangay pages instead of all eligible ones. The query that counts streets per barangay to decide which pages have real data returns one row per barangay (tens of thousands), but Supabase silently caps a single response at 1,000 rows, so only the first 1,000 barangays were considered and the rest were dropped. That count query now pages through its full result set, so every barangay with at least one street of zonal data is included.

## [2.12.0] - 2026-05-22

### Fixed
- The XML sitemap listed only about 2,800 URLs and was missing the vast majority of the site. Two causes: the broker query asked for up to 10,000 profiles but Supabase silently caps a single request at 1,000 rows, so roughly 34,000 of the ~35,000 broker profiles never made it into the sitemap; and barangay-level zonal value pages (tens of thousands of pages with unique street-level data) were left out entirely. Search engines were therefore left to discover most pages on their own through internal links, which is slow and incomplete.

### Changed
- Rebuilt the sitemap as a sitemap index. `/sitemap.xml` is now an index that points to per-type sitemap files (core pages, brokers, realties, barangays), each well under the 50,000-URL limit. Broker and realty lists now page through the full result set instead of stopping at 1,000. Broker profiles are included when they are backed by real PRC or exam-cohort data, and barangay pages are included when they have at least one street of zonal data, so empty placeholder pages stay out of the index. The sitemap rebuilds once daily.

## [2.11.1] - 2026-05-21

### Fixed
- Some zonal value pages were missing their catch-all "ALL LOTS" street, and the street listed just above it showed prices that did not belong to it. BIR sheets name this street on a row whose first classification is footnoted out (e.g. `ALL LOTS | RR | ***`) and then list the real values on the rows beneath it, which rely on inheriting the street name from the row above. The parser recorded the street name only after deciding a row's own value survived, so the naming row (with the footnoted, value-less first classification) was skipped before its name was saved; the value rows beneath then inherited the previous street's name, welding "ALL LOTS" prices onto it. Where "ALL LOTS" was a barangay's first street, its rows became nameless and were dropped entirely. Example: Hermosa, Bataan. The parser now records the street/vicinity context for any row with a valid classification, even when that row's own value is footnoted out, so "ALL LOTS" (and any street named on a footnoted row) keeps its values. Bataan alone recovered 156 streets. Corrections apply on the next zonal data refresh

## [2.11.0] - 2026-05-21

### Fixed
- Some zonal value pages listed the same subdivision twice with contradictory prices, e.g. Santa Rosa, Laguna's "Dila" barangay showed GOLDEN CITY SUBDIVISION at ₱11,000/sqm next to GOLDEN CITY SUBD. at ₱5,500/sqm. The rows came from two different Department Orders (a recent DO and an older superseded one) that the parser failed to reconcile because the orders spell the city differently (e.g. `STA. ROSA CITY` vs `SANTA ROSA`, or `LEGAZPI` vs the older `LEGASPI`), so the "keep only the newest order" step treated them as different cities and kept both. The deduplication now runs on each location's official PSGC identity (which both spellings resolve to) and keeps the newest order per barangay, so superseded values are dropped wherever a newer order covers the barangay. The same root cause was also producing duplicate barangay entries (a barangay listed twice under variant city spellings); those are now merged. Separately, this corrected a long-standing under-count: the previous per-city logic silently discarded barangays that an older order covered but a newer one for the same city did not re-issue, so several thousand barangays with valid values were missing entirely and are now restored. Corrections apply on the next zonal data refresh
- Some zonal value street rows showed a stray "((NEW))" label. BIR marks newly added streets with a "(NEW)" annotation, which the parser was storing as the street's vicinity (location) and then wrapping in its own parentheses on display. The annotation is now recognized and dropped from the vicinity, leaving the street name clean. Corrections apply on the next zonal data refresh

## [2.10.8] - 2026-05-21

### Fixed
- Claiming a broker profile via "Continue with Google" dropped you on the search page instead of the profile you were trying to claim. The intended destination rode along as a `?redirect=` query parameter on the sign-in callback URL, but that parameter is not reliably preserved through the Google OAuth round-trip. When it was lost, the callback fell back to `/profile`, and the dashboard then bounced brand-new users with no claimed profile to `/search`, so the claim flow never completed. The destination is now stashed in a short-lived first-party cookie before the round-trip and read back on return (the query parameter remains a fallback). Both values are sanitized so a tampered cookie cannot redirect you off-site. Magic-link and email-signup confirmations use the same mechanism

## [2.10.7] - 2026-05-20

### Fixed
- Broker profile and project view counters were stuck at 0 (or 1) and never increased. Both pages set `revalidate=false` and called `trackProfileView` / `trackProjectView` from inside their Server Components, so the tracking RPC only ran when Next.js first rendered the page (top 500 brokers at build time, the rest on first on-demand visit). Every subsequent visit served cached HTML without re-running server code. Compounded on Cloudflare Workers, where the fire-and-forget async call was not wrapped in `waitUntil` and could be cancelled before the RPC landed, leaving many pages at 0 even after the first render. Replaced server-side tracking with a client beacon: a new `/api/track-view` route handler (same-origin guard via Origin header with Referer fallback, `cf-connecting-ip` for honest per-IP dedupe, `force-dynamic` to defeat caching) is called once per session per entity from a tiny `<TrackView>` client component using `navigator.sendBeacon` with a `fetch keepalive` fallback. Pages stay fully static, the cached HTML is unchanged, and view counts now reflect actual visitors
- The view-tracking database functions (`track_profile_view` / `track_project_view`) had been throwing on every call since they were written: their `ON CONFLICT (…, DATE(viewed_at))` targeted an expression with no matching unique index (the unique index is on the stored `viewed_date` column), so Postgres rejected every insert and the error was swallowed by the caller, recording zero views. Repointed the conflict target at `viewed_date`, which carries the matching index and defaults to the current date on insert, preserving the one-view-per-viewer-per-day dedupe. This was the deepest cause of the stuck counters; without it the client-beacon rewrite above would still have recorded nothing
- Deleted the orphaned server-side helpers (`src/shared/lib/view-tracking.ts`, `src/domains/directory/actions/views.ts`, `src/domains/projects/actions/views.ts`) so the obsolete pattern cannot be re-imported

## [2.10.6] - 2026-05-20

### Fixed
- Some zonal value pages showed phantom "streets" that were really subdivision/vicinity labels, carrying values fused from unrelated rows and frozen on an old, superseded Department Order. Example: Tagaytay/Calabuso listed a `BELLE CORP/TAGAYTAY HIGHLANDS` "street" that merged the barangay's land catchall with a separate condominium block at 2017 values, even though the barangay's current DO is from 2021. Root cause was two parser defects: (1) bare section headers like `TAGAYTAY CITY` / `CITY OF TAGAYTAY` in older multi-city Department Order sheets were not recognized, so a barangay's old-DO data was filed under the wrong city and the "keep only the newest DO" merge could not replace it; (2) the layout detector sometimes collapsed the street and vicinity columns into one, discarding the real street names and mislabeling rows with the vicinity. Both are fixed (313 affected sheets across 90 revenue districts corrected); the example barangay now shows only its 12 real streets. Corrections apply on the next zonal data refresh

## [2.10.5] - 2026-05-20

### Fixed
- Zonal value Department Order (DO) effectivity dates were silently dropped for 6,487 barangays. The agentic parser's date reader only understood 5 formats, but BIR DOs heavily use an abbreviated-month-with-period style (`Sept. 2, 2022`, `Oct. 13, 2023`, `Feb. 9, 2024`) plus minor `29-July-2015` and `06-Jan-2022` variants that none of the 5 matched. Unmatched dates fell to a sentinel that sorts as oldest-possible, so mostly-recent (2019-2024) dates were treated as ancient. No zonal values were lost (dates only affect ordering, never inclusion), but the effectivity metadata was wrong. The parser now normalizes the string (collapse spaces, drop periods, map `Sept`->`Sep`) and covers the missing formats; all 6,487 dates parse with zero regressions. Corrected dates apply on the next zonal data refresh
- City "last updated" date (`effective_date`) could show an older date than the city's newest DO. The importer picked the most common barangay date string, so a recent DO that only touched some barangays of a city was masked by the older majority. 25 cities were affected (incl. Anao, Camiling, Mayantoc, Moncada, Ramos, San Clemente in Tarlac, plus parts of Bohol and Bulacan). The city date now reflects the newest DO across its barangays. Applies on the next zonal data refresh

## [2.10.4] - 2026-05-20

> Note: this is the property-value guide IA migration that was originally stamped as 2.10.0 on a feature branch but never merged to main (main went v2.9.0 -> v2.10.1, skipping 2.10.0). Re-applied onto current main and shipped here as 2.10.4. The old [2.10.0] section below is retained as a historical marker.

### Changed
- Information architecture for long-form how-to content. `/insights/blog/how-to-compute-property-value-philippines` (the only post on the blog) was structurally a step-by-step guide with HowTo schema, not commentary. Moved to `/guides/how-to-compute-property-value-philippines` so all procedural how-to content lives at the same URL pattern as the title-transfer guide shipped in v2.9.0. `/insights/blog/` and `/insights/` stay alive for future commentary-style posts (no HowTo schema, opinion/market-take content). The four pillars are now distinct: `/guides/*` for procedural how-to, `/insights/reports/*` for data-heavy research with Dataset schema, `/insights/blog/*` for commentary, `/resources/*` for reference material (checklists, templates)
- Old `/insights/blog/how-to-compute-property-value-philippines` URL now returns HTTP 308 Permanent Redirect to the new `/guides/` URL. Implemented via `next.config.ts` `redirects()`, so it runs at the Worker level before route matching. Existing inbound links continue to work without breaking. Google should re-index the new URL within 1-4 weeks; the redirect preserves link equity per Google's documented behavior

### Added
- New dynamic route `/guides/[slug]` that renders MDX from `content/guides/` using the same MDXRemote pipeline as `/insights/blog/[slug]` and `/insights/reports/[slug]`. Reuses the existing `mdx-components`, `TableOfContents`, and `ShareButtons` from the reports renderer. Extends `getContentBySlug` / `getAllSlugs` / `getAllContent` in `src/shared/lib/mdx.ts` to accept `'guides'` as a content type. The static title-transfer guide at `/guides/how-to-transfer-property-title-philippines/page.tsx` keeps its own JSX page (Next.js explicit segments win over `[slug]`, so no routing conflict). New MDX-based guides only need an `.mdx` file in `content/guides/` plus an entry in the static `guideSchemas` overlay for per-guide HowTo steps, tools, and entity mentions
- Same schema rigor as the title-transfer guide on the new dynamic route: `WebPage` + multi-type `Article` + `LearningResource` (with `learningResourceType: Guide`, `educationalLevel: Beginner`, `audience`, `image` `ImageObject` pointing to the OG card) + `HowTo` with 5 anchored steps and 5 tools (the renderer conditionally emits the `HowTo` node only when the slug overlay provides steps, so a future MDX guide added without a registry entry will never ship an invalid empty `HowTo` to Search Console) + `BreadcrumbList`. Author `hasCredential` linking the PRC license to the Professional Regulation Commission as `GovernmentOrganization` `recognizedBy`. 10 entity `mentions` for the property-value guide, each with `sameAs` arrays linking Wikidata first then Wikipedia (BIR Q4998387, LRA Q17075354, PRC Q7248019, HDB Singapore Q1934365, Quezon City Q1475, Ang Mo Kio Q4761845, Philippines Q928, Singapore Q334) for stronger Knowledge Graph entity resolution and AI Overview citation on valuation queries
- Per-guide OG card at `/og/how-to-compute-property-value-philippines.png`, generated by `npm run og:generate` after adding the variant to `scripts/generate-og-images.tsx`. Wired into the renderer at three places: `Article.image` `ImageObject`, `openGraph.images`, and `twitter.images`. The renderer accepts an optional `image` field on each `guideSchemas` overlay entry and falls back to `/og/default.png` when not set, so social shares of MDX guides no longer fall back to the generic site default
- Visible author bio card on every dynamic guide: PRC license number, year licensed, founder credentials, link to `/about`. Same E-E-A-T pattern as the title-transfer guide. Renderer adds it automatically for any MDX guide
- The migrated guide gets a richer related-links section linking to the title-transfer guide, BIR Zonal Value tool, Transfer Tax Calculator, and Due Diligence Checklist. Cross-link from the title-transfer guide back to this one is implicit via the `/guides` index

### Removed
- `content/blog/how-to-compute-property-value-philippines.mdx` source file. Migrated to `content/guides/`. The 308 redirect handles any inbound traffic to the old URL. `/insights/blog/` index now shows its "Coming soon" fallback until commentary-style posts are added

## [2.10.3] - 2026-05-20

### Fixed
- Zonal value pages served decades-old values for cities whose newest BIR Department Order uses a padding-spaces city header. The agentic parser matched the literal label `"CITY :"`, but newer DOs write `"CITY                      : TAGAYTAY"` with many spaces before the colon. When a city header was missed, its barangays inherited the previously-detected city, so `_merge_by_newest_do` kept stale older-DO values as the city's "newest." Concrete cases: Tagaytay served 1989 DO 48-89 values (e.g. Mendez Junction at ₱400/sqm with a 1990-03-08 effectivity) while its real 2021 barangays were mis-attributed to Imus; Tabaco was stuck on 1997 values, Ligao on 2009, plus El Salvador, Oroquieta, Ozamiz, and Trece Martires. A sweep of all 118 RDO files found 13 RDOs using this header format in their newest DO (051, 054A, 065, 066, 067, 068, 070, 086, 089, 098, 100, 101, 102). The parser now whitespace-normalizes both the joined row text and each cell before matching location labels (`scripts/agentic-parser/agents/extractor.py`). After regenerating and reimporting: Tagaytay now has 31 barangays all on the Oct 23 2021 DO with zero pre-2021 fossils, Imus is back to its PSGC-correct 97 barangays, and Tabaco/Ligao are fully on their April 2024 DO. Production reimported 2026-05-20 (232,547 value rows)
- Closed a parser regression risk that surfaced during this fix: the import script's `--truncate` hardcoded local psql credentials and its DELETE-all fallback timed out on prod-sized tables. It now honors `DATABASE_URL`/`SUPABASE_DB_URL` for the fast psql TRUNCATE and pages the fallback delete by id so each statement stays under the server statement timeout

## [2.10.2] - 2026-05-19

### Fixed
- Applied the prod data migration promised in v2.10.1. Migration `00035_zonal_province_misattribution_fix.sql` inserts 11 missing `zonal_provinces` rows (Batanes, Apayao, Dinagat Islands, Davao de Oro, Davao Occidental, Biliran, Camiguin, Sarangani, Maguindanao del Norte, Maguindanao del Sur, Guimaras) and repoints 96 mis-attributed cities to their correct PSGC provinces. Each UPDATE is guarded by `province_id <> target` so the migration is idempotent. Companion migration `00034_zonal_province_misattribution_backup.sql` snapshots the pre-fix `(city_id, slug, province_id, current_province_slug, rdo_code)` tuples into `_zonal_cities_province_backup_20260519` for rollback. Both migrations rely on `ON CONFLICT DO NOTHING` and can be re-run safely
- Added 192 permanent 308 redirects in `next.config.ts` covering the 96 moved cities and their barangay subroutes (`/tools/zonal-value/{old-province}/{city}` and `/tools/zonal-value/{old-province}/{city}/:barangay` -> the new province paths). Source list lives in `src/shared/config/province-migration-redirects.ts`. Preserves SEO equity for indexed URLs like `/tools/zonal-value/cagayan/basco` which now route to `/tools/zonal-value/batanes/basco`

### Added
- `scripts/generate-province-migration.ts` is the canonical regenerator: reads prod, applies the PSGC region/province mapping with the NCR slug alias, and emits the JSON audit payload, both SQL migrations, and the redirect TS module in one shot. Idempotent; safe to re-run if the source-of-truth in `migration/zonal-province-misattribution.json` ever needs to be refreshed
- `scripts/preflight-task3.ts` and `scripts/preflight-task3-deeper.ts`, read-only audit scripts that verified before applying: which target province slugs already exist, no slug collisions at `(target_province_slug, city_slug)`, no other tables denormalize province text by id, and a spot-check that real Cagayan cities (e.g. Aparri) are not in the move set

## [2.10.1] - 2026-05-19

### Fixed
- Zonal value pages were attributing 97 cities to the wrong province because the agentic parser hardcoded one province per RDO file in `rdo_files.json` and the resolver only re-routed cities when their PSGC region differed from the RDO's region. Same-region province splits silently inherited the RDO's manifest province. Concrete cases: the six Batanes municipalities (`/zonal-values/cagayan/{basco,itbayat,...}`) were filed under Cagayan because BIR's RDO 013 covers both Cagayan and Batanes through a shared Excel file; 37 cities of the 2022 Maguindanao split were filed under "Maguindanao" rather than del Norte / del Sur; Compostela Valley municipalities still showed as Davao del Norte even after the 2019 rename to Davao de Oro; and the splits for Biliran, Apayao, Dinagat Islands, Sarangani, Davao Occidental, Guimaras, and Camiguin all carried the parent province's slug. `scripts/agentic-parser/generate_output.py:_resolve_city_region` now derives both region (PSGC positions 1-2) and province (positions 1-5) unconditionally from the city's grounded PSGC code, with an alias mapping the PSGC slug `national-capital-region-ncr` to the app's canonical `metro-manila`. The RDO manifest is only used as a last-resort fallback for ungrounded cities. The fix only affects the data pipeline going forward; an explicit prod data migration will repoint the existing ~96 mis-attributed cities and add 301 slug aliases for old URLs

### Added
- `scripts/agentic-parser/validate_province_attribution.py`, a regression gate (validator C35) that walks every grounded region JSON, looks up each city's PSGC province at positions 1-5, and exits non-zero if any city's assigned `province_slug` disagrees. Honors the same NCR alias as the parser. HUC/ICC PSGCs whose province segment is the city's own 3-digit code (Lucena 04312, Baguio 14303, Butuan 16304, etc.) get warned but not failed because PSGC has no province row at those prefixes and the parser correctly falls back to the RDO manifest there. Verified against the pre-fix grounded output: catches all 96 known intra-region mismatches and exits 1. Intended to run between the parser pipeline and any DB import
- `scripts/agentic-parser/test_resolve_city_region.py`, focused tests for the parser resolver covering all 11 multi-province RDO pairs from the sweep, the NCR alias, the ungrounded fallback, and a non-split control case (15 cases, all passing)
- `scripts/agentic-parser/rdo_config.py` header note: clarifies that the `province` / `province_slug` fields in `rdo_files.json` are RDO-tier metadata only, no longer authoritative for grounded cities. Project CLAUDE.md pre-push checklist gains item 8: run the C35 validator before any push or import of zonal data, must exit 0
- `docs/plans/2026-05-19-batanes-cagayan-investigation.md` and four read-only investigation scripts (`find-10yr-old-zonal.ts`, `investigate-batanes-cagayan.ts`, `check-batanes-values.ts`, `sweep-multi-province-rdos.ts`) documenting the original 97-city blast radius and the multi-province RDO sweep that surfaced the 11 problem pairs

## [2.10.0] - never released

This version number was stamped on the `feature/migrate-property-value-guide` branch but never merged to main. Main advanced v2.9.0 -> v2.10.1 (province work) without it, so 2.10.0 was skipped on the mainline. The work it described (property-value guide IA migration) was later re-applied onto current main and shipped as **[2.10.4]** above. This stub is kept so the version sequence is not silently missing an entry.

## [2.9.0] - 2026-05-18

### Added
- New top-level `/guides` route as the home for long-form how-to walkthroughs, distinct from `/resources` (reference material like checklists and templates). First entry is the title-transfer guide; the index is a stub that will grow as more guides ship. Both routes added to `sitemap.ts` at priority 0.85 (index) and 0.9 (guide), with monthly change frequency on the guide and weekly on the index
- `/guides/how-to-transfer-property-title-philippines` (~3,200 words). Four-agency walkthrough covering BIR (CGT/DST/eCAR), Local Treasurer (Transfer Tax), Registry of Deeds (TCT/CCT issuance), and Assessor's Office (Tax Declaration update). Includes a TL;DR card for AI Overview extraction, the embedded MiniTransferTaxCalculator inline in the Quick Cost Estimate section, the 5-step procedural walkthrough with anchored section IDs (`#step-1` through `#step-5`), a Required Documents Checklist, realistic Timeline + Cost tables, six named pitfalls, a Special Cases section pointing to future guides (estate, donation, corporate, foreign, bank/Pag-IBIG, foreclosure), and 10 article-specific FAQs. Author bylined Aaron Zara with PRC license #0025157 and a visible E-E-A-T credentials Card. Last-updated freshness line. Disclaimer at the end
- JSON-LD on the guide page: `Article` + `LearningResource` (multi-type, with `learningResourceType: Guide`, `audience`, `educationalLevel: Beginner`), `HowTo` with `totalTime: P90D`, 8 `HowToTool` entries, and 5 anchored `HowToStep` entries; `FAQPage` mirroring the 10 visible FAQs; `BreadcrumbList`. `estimatedCost` was intentionally omitted from HowTo — the real outlay (8-10% of property value in CGT/DST/transfer tax/registration) scales with the deal, so a flat MonetaryAmount would misrepresent it. The Timeline and Costs section in the article body covers the percentage breakdown thoroughly. Author has `hasCredential` (EducationalOccupationalCredential) linking the PRC license to the Professional Regulation Commission as `GovernmentOrganization` `recognizedBy`. Article publisher and author use `@id` linking into the existing identity graph (`ren.ph/#organization`, `godmode.ph/#principal`)
- 12 entity `mentions` on the guide's Article schema, five with Wikipedia `sameAs` links for AI Overview citation strength: Bureau of Internal Revenue, Land Registration Authority, DHSUD, TRAIN Law (RA 10963), Local Government Code (RA 7160). Plus Property Registration Decree (PD 1529), Real Estate Service Act (RA 9646), and topical Things for CGT, DST, TCT, CCT, and eCAR
- `<MiniTransferTaxCalculator />` client component at `src/domains/tools/components/mini-transfer-tax-calculator.tsx`. Two-input inline calculator (selling price + optional BIR zonal value) that computes CGT (6%) and DST (1.5%) on the higher of the two. Imports the same `tax-calculations.ts` util as the full `/tools/transfer-tax-calculator` form, so math is single-sourced. "Open Full Calculator" CTA deep-links into the full calculator with `?price=N` so the user's number carries over. Not iframed; chose a shared-utility React component so the guide page passes SEO value and avoids cross-origin fragmentation
- `/services/title-transfer` landing page for the managed title-transfer service. Flat fee ₱30-50k for typical residential transactions (complex cases priced separately), 2-3 month timeline, run by PRC-licensed brokers. Hero, pricing strip, what's included / what's not, 5-stage sample case timeline with icons, trust signals (PRC license display, realistic timelines, flat fee), 6 service-specific FAQs, and two prominent CTAs (mailto + /contact). JSON-LD: `Service` + `Offer` + `PriceSpecification` MonetaryAmount, plus `FAQPage` and `BreadcrumbList`. Two CTAs inside the title-transfer guide link here
- Responsive hero image on the guide. `<picture>` element with three WebP variants (800/1200/1600w, 28/50/70 KB) and JPEG fallbacks (800/1200/1600w, 36/67/104 KB) at `public/images/guides/transfer-title-hero-*`. Mobile gets the 28 KB WebP, desktop the 50 KB, high-DPI the 70 KB. Sizes attribute `(min-width: 1024px) 768px, 100vw` matches the article's max-w-3xl reading column. `loading="eager"` + `fetchpriority="high"` because the hero is in the LCP candidate set. Explicit `width`/`height` prevent CLS. Semantic alt text describes the photo (Philippine land title document on a desk with house keys and a pen)
- Replaced the satori-generated OG card for the guide (`public/og/title-transfer-guide.png`) with a 1200x630 crop of the real hero photo, rendered via sharp at quality 90. Removed the `title-transfer-guide` variant from `scripts/generate-og-images.tsx` (with a comment explaining the manual replacement) so `npm run og:generate` no longer overwrites it. The OG card for the service page (`public/og/title-transfer-service.png`) stays as the auto-generated satori card
- Inbound links to the new guide added from `/tools/transfer-tax-calculator` (intro paragraph + Related Resources sidebar), `/tools/zonal-value` (CTA card copy), `/resources/due-diligence-checklist` (Related Resources sidebar), `/resources/condo-turnover-guide` (Related Resources footer), `/resources/hoa-guide` (Related Resources footer). Cross-link from the guide's "Need Help?" section and FAQ to `/services/title-transfer`

### Changed
- Root `<html lang>` from `en` to `en-PH`. The article schema already used `inLanguage: en-PH`; aligning the document-level attribute closes the inconsistency and gives crawlers a stronger geo signal for the Philippines-specific catalog
- Page metadata `title` and `description` on the new guide are tuned to fit Google SERP truncation: 63 chars for `<title>` (including the auto-appended `| REN.PH` suffix) and 143 chars for the meta description. Page-level `title` no longer carries the `| REN.PH` suffix in either openGraph or twitter blocks, deferring to the root layout's title template, which fixes the double-suffix issue (`Foo | REN.PH | REN.PH`) that also affected `/services/title-transfer` and `/guides` index on first render

### Fixed
- v2.8.2 shipped `src/app/opengraph-image.tsx` and `src/app/twitter-image.tsx` as Next.js dynamic OG routes backed by `next/og` `ImageResponse`. Those routes returned HTTP 500 in production with `TypeError: Cannot read properties of undefined (reading 'default')`. Root cause: `next/og` pulls in `@vercel/og` which uses WASM-based satori + resvg; that bundle does not load through OpenNext's Cloudflare Workers pipeline. The pre-existing `/brokers/[slug]/opengraph-image` route was also silently broken under the same bug, returning 404 for valid broker slugs since the Cloudflare migration. Replaced both with static PNGs generated at build time using the same renderer (`satori` + `@resvg/resvg-js`) in a Node script, shipped from `/public/og/`. Social previews on Facebook, X, LinkedIn, Slack, Discord, iMessage, and AI link unfurlers now render the branded card instead of a broken image
- Removed `src/app/opengraph-image.tsx`, `src/app/twitter-image.tsx`, `src/shared/lib/og-default-image.tsx`, and the latent broken `src/app/(public)/brokers/[slug]/opengraph-image.tsx` so future maintainers do not re-add the dynamic route pattern that does not work on this stack

### Added
- 17 branded OG cards generated by `npm run og:generate` from `scripts/generate-og-images.tsx`. Variants for the default fallback plus the homepage, `/verify/broker`, `/verify/lts`, `/tools`, `/tools/transfer-tax-calculator`, `/tools/pagibig-housing-loan-calculator`, `/tools/zonal-value`, `/resources`, `/resources/due-diligence-checklist`, `/resources/foreclosure-guide`, `/resources/condo-turnover-guide`, `/resources/hoa-guide`, `/resources/legal-templates`, `/insights`, `/academy`, and `/about`. Each is 1200x630 PNG, ~140 KB. Green gradient matches the brand, with page-specific title, subtitle, and three contextual chips (e.g. `LRA · BIR · DHSUD` on the due-diligence card). Re-run the script whenever copy changes
- `openGraph.images` block added to the 14 main pages that previously inherited the root default, so each page now advertises its variant. Layout fallback (`siteConfig.ogImagePath`) repointed at `/og/default.png` so any page without an explicit override still ships a working image

## [2.8.2] - 2026-05-18

### Fixed
- Social previews (Open Graph and Twitter Card images) were 404ing on every page because `siteConfig.ogImage` pointed at `/og-image.png`, a static asset that does not exist in `/public`. Facebook, X, LinkedIn, Slack, iMessage, and AI link unfurlers all rendered a broken image (or no preview) when ren.ph URLs were shared. Replaced with Next.js dynamic OG routes (`/opengraph-image` and `/twitter-image`) backed by a shared `next/og` renderer that produces a branded 1200x630 PNG with the site name, tagline, and pillar chips at request time. Root layout, homepage, and `/resources/due-diligence-checklist` now point at the dynamic routes; per-page `images` blocks include explicit `width`, `height`, and contextual `alt` so the metadata is unambiguous under Next.js metadata shallow-merge. `siteConfig.ogImage` is renamed to `ogImagePath` (with a new `twitterImagePath` sibling) to make the role of the field clear at call sites
- Organization JSON-LD on the homepage was still hardcoding `${siteConfig.url}/og-image.png` in its `image` field, shipping the same 404'ing URL to Google's knowledge graph and to any AI crawler that consumes Schema.org Organization data. Repointed at the dynamic OG route so the homepage's structured data, OpenGraph, and Twitter Card all advertise the same valid asset
- Twitter Card `images` blocks in the root layout and on `/resources/due-diligence-checklist` were emitting bare URL strings, which causes the X card validator to omit `twitter:image:alt` and to drop image dimensions. Expanded both blocks to structured objects with `width: 1200`, `height: 630`, and contextual `alt` text matching the OpenGraph metadata, so `summary_large_image` renders consistently across X, LinkedIn, and Discord

### Changed
- Dropped the redundant intro paragraph on `/resources/due-diligence-checklist`. The AIO opening added in v2.8.1 (226bbe8) already covers the same 24-step / title / tax / seller framing, so the second paragraph was duplicative for both readers and crawlers

## [2.8.1] - 2026-05-18

### Added
- FAQ section on `/verify/broker` with 12 questions covering RA 9646, the PRC LERIS relationship, unclaimed vs verified profiles, DHSUD/LTS overlap, the broker vs salesperson distinction, and OFW remote-buying precautions. Server-rendered with native `<details>` (no client JS), first three open by default, content available to crawlers regardless of collapsed state
- `FAQPage` JSON-LD on `/verify/broker` mirroring all 12 questions, so AI Overviews / ChatGPT / Perplexity can cite passage-level answers without scraping the rendered DOM. Cross-links the on-page answers to `/academy/courses/ra-9646-essentials` and `/verify/lts`

## [2.8.0] - 2026-05-18

### Added
- Pag-IBIG housing loan calculator at `/tools/pagibig-housing-loan-calculator`. Three calculation modes: "How much can I borrow?" computes the max loan from income against the 35% affordability rule with the right program ceiling applied; "Monthly on this property" derives the loan as price minus down payment and auto-detects whether 4PH applies based on income, property ceiling, and OFW status; "Monthly on this loan" runs the amortization formula directly on a chosen loan amount. Standard tier rate is iterated to a true fixed point so the displayed (rate, loan) pair always matches what Pag-IBIG would actually approve together
- Repricing scenarios on the same calculator. For 4PH and the non-socialized promo, the post-subsidy scenarios anchor on the standard Pag-IBIG tier rate the loan actually reprices into (currently 5.875% to 9.75%), not on the subsidized rate plus or minus a small move. A 4PH borrower at 3% now sees an honest +30% to +75% jump in monthly amortization after the 5-year subsidy ends, instead of a misleadingly soft range around 3%
- Prepayment what-if (compute time and interest saved from extra monthly payments) and a paginated full amortization schedule on the calculator
- Standalone Pag-IBIG eligibility checker on the same page: contribution months, age at application, planned term, stable income, prior defaults, property location. Returns eligible / has issues / not eligible with hard vs soft severity. Age below 18 is treated as a hard issue, with distinct copy for the empty-input case
- JSON-LD structured data on the calculator page: `WebApplication` (typed as `FinanceApplication`), `FAQPage` mirroring the on-page FAQs, and `HowTo` for the amortization formula
- Pag-IBIG calculator wired into the Tools dropdown in the site header, promoted from "Coming Soon" into the `/tools` index grid, and surfaced as a related-tools cross-link on the new page

### Changed
- `/tools/transfer-tax-calculator` now accepts an optional `?price=` query parameter (e.g. `?price=3000000`). When present and positive, the selling-price field is prefilled and the tax breakdown auto-computes on first render. Used by the Pag-IBIG calculator's "Compute closing costs for this property" cross-link so users carry the same property value across both tools without retyping. Server-side `searchParams` shape (Next 16 `Promise<{ price?: string }>`) feeds an `initialPrice` prop into the form; no client-side `useSearchParams` in the form, so the React Compiler "no setState in useEffect" rule is satisfied

### Notes
- Standard tier rates (5.875% to 9.75%) in the rate config are indicative and should be cross-checked against pagibigfund.gov.ph before heavy promotion. The 4PH 3% rate and the 4.5% Non-Socialized Promo rate are confirmed from public news sources. The calculator surfaces this caveat under the hero and in the results disclaimer
- Mortgage Redemption Insurance (MRI) premium is not included in the monthly amortization. Pag-IBIG's official calculator adds roughly PHP 45 + 0.225% of loan annually for MRI. The on-page disclaimer flags this
- 4PH eligibility check sums primary + co-borrower income against the program's individual income ceiling. This is the conservative direction (more likely to flag a borrower as ineligible for the 3% subsidy). Pag-IBIG's actual rule applies the income ceiling to the principal applicant only; users wanting to check this can uncheck the co-borrower toggle
- 4PH program defaults to a 5-year fixed pricing period in the calculator. Early-bird qualifiers (first 30,000 slots) get 10 years fixed but the calculator doesn't expose that flag yet; default 5 is the conservative assumption

## [2.7.3] - 2026-05-17

### Fixed
- Canonical URLs (`<link rel="canonical">`, `<meta property="og:url">`, sitemap, robots.txt, JSON-LD across every page, and the X/Facebook/LinkedIn share buttons on insights posts) were emitting `http://localhost:3000` on the live site because `NEXT_PUBLIC_APP_URL` was inlined as a localhost value during the last production build. Google reads a canonical pointing at localhost as a directive to treat the public URL as a non-canonical alias of an unreachable origin, with real risk of de-ranking and de-indexing
- `siteConfig.url` now resolves through a guarded helper that throws at module load if `NODE_ENV=production` and the env value contains `localhost` or `127.0.0.1`, instead of silently falling through to a default. `scripts/setup/build-with-prod-env.sh` was extended to refuse the build if the prod env file still carries a localhost `NEXT_PUBLIC_APP_URL`, mirroring the existing Supabase URL guard. Same bug class as the February incident; the previous fix relied on operator memory to pass the right env var on the build command line, the fix now lives in code
- Reverted v2.7.2's `middleware.ts` -> `proxy.ts` rename. Next 16 mandates `proxy.ts` runs on the Node runtime with no route-segment config allowed; OpenNext for Cloudflare Workers only supports Edge middleware. The two specs are currently incompatible. Restored `src/middleware.ts` with `export const runtime = 'edge'`, which is what OpenNext requires. The v2.7.2 rename was acting on a theory that `middleware.ts` execution under OpenNext was the root of intermittent session loss, but that rename had never reached production. The four other auth fixes in v2.7.2 (OAuth callback `x-forwarded-host`, `safeRedirectPath` util, login-form race, claim integrity migration) are independently sufficient candidates and all still ship

### Notes
- `NEXT_PUBLIC_*` env vars are inlined into the JS bundle at build time, so `wrangler.jsonc [vars]` (a runtime mechanism) cannot fix them after the fact. The `NEXT_PUBLIC_APP_URL` line in `wrangler.jsonc` was correct the whole time the bug was live; it just had no way to influence the already-compiled bundle. Any `NEXT_PUBLIC_*` var that affects URLs, metadata, or SEO output must be correct in the shell environment at the moment `next build` (or `opennextjs-cloudflare build`) runs

## [2.7.2] - 2026-05-16

### Security
- Open-redirect check on auth flows replaced with `safeRedirectPath` util. The previous inline guard (`raw.startsWith('/') && !raw.startsWith('//')`) allowed `/\evil.com` because the WHATWG URL parser normalizes backslashes in special schemes into forward slashes, turning the path into `//evil.com` after the check passed. Tabs and newlines were also stripped during URL resolution, so `/\t/evil.com` slipped through. The new util resolves against a sentinel origin and rejects anything that doesn't land back on the sentinel. One implementation, 10 tests, used by `callback/route.ts` and `login-form.tsx`

### Added
- Migration `00033_claim_integrity_constraints.sql`: enforces `UNIQUE(profiles.user_id)` (one auth account = one claimed profile) and a partial `UNIQUE(user_id, profile_id)` on incomplete claim challenges (dedupes in-progress quizzes). Includes a non-destructive pre-flight that raises with the duplicate count before any constraint is added

### Fixed
- `verifyAndClaimProfile` returned `{ success: true }` for zero rows after the `.eq('tier','unclaimed')` race-condition guard matched nothing. Action now selects the updated row and checks `updated.length`. The user no longer sees "claimed" UI while the DB still says unclaimed
- Login form left users stranded after signup-with-session by clearing loading state before the navigation completed. Form now navigates via `window.location.assign` so the post-signup transition is reliable
- OAuth callback redirect on Cloudflare Workers + OpenNext: `request.url`'s origin can be an internal host or wrong protocol behind the proxy. Callback now uses `x-forwarded-host` + `x-forwarded-proto` for the user-facing redirect base, falling back to `origin` for local dev. Fixes "signed in with Google, still logged out"
- `verifyAndClaimProfile` now maps PG `23505` (unique_violation) from the new `UNIQUE(profiles.user_id)` constraint to `"You already have a claimed profile."` instead of the generic `"Failed to claim profile"`. Users who already own a profile now see why the second claim fails

### Changed
- `src/middleware.ts` renamed to `src/proxy.ts` to match the Next.js 16 spec. The deprecated `middleware.ts` name is not guaranteed to execute under OpenNext on Workers, which was the most likely root of intermittent "logged in but session lost" behavior

### Notes
- Migration `00033` deletes orphan in-progress claim challenges (per `(user_id, profile_id)` keep-most-recent dedup). Users mid-quiz at deploy time will need to restart the quiz; claim cooldowns are not affected. Coordinate deploy timing accordingly
- The OAuth callback's reliance on `x-forwarded-host` is safe under the current Cloudflare Workers + OpenNext deployment because the Worker is the origin and CF normalizes the header. Trust boundary documented inline in `callback/route.ts` so the assumption is reviewable if the deployment topology ever changes

## [2.7.1] - 2026-05-15

### Changed
- Zonal value search (`/tools/zonal-value` autocomplete) rewritten as a single ranked Postgres RPC (`search_zonal_locations`). Trigram indexes (`gin_trgm_ops`) replace unindexed ILIKE scans on 36K barangays + 1,620 cities + 72 provinces; results are ranked by similarity + prefix bonus across all three entity types in one query. Autocomplete execution drops from seconds to single-digit milliseconds, and barangays are no longer starved out of the result set by client-side push order
- Cities formatted "City of X" or "Municipality of X" (135 of 1,620 entries) now rank correctly against barangay namesakes. Previously, typing "tagaytay" surfaced 8 obscure barangays before "City of Tagaytay" because the formal prefix diluted the similarity score. Score now compares against the stripped name; full display name still drives recall
- When scores tie, broader entities win: a query of "cebu" surfaces Cebu province, then City of Cebu, then barangay Cebu, not the other way around

### Fixed
- Zonal search no longer strips ñ and punctuation from the query, so accented place names like "Las Piñas" and "Sto. Niño" match when typed with the correct characters. Cross-accent folding ("pinas" matching "piñas") remains a separate follow-up that needs the `unaccent` extension
- Autocomplete RPC errors now surface in server logs (`console.error` with code + message) instead of silently degrading to an empty dropdown, so permission-denied or RPC-missing failures are debuggable
- `search_zonal_locations` carries an explicit `GRANT EXECUTE` to `authenticated, anon` matching the pattern used by every other RPC in the repo, so it works regardless of role-default state on production

### Notes
- Migration `00030_zonal_search_trgm_indexes.sql` is fully idempotent (`CREATE EXTENSION IF NOT EXISTS`, `CREATE INDEX IF NOT EXISTS`, `CREATE OR REPLACE FUNCTION`). Drops the now-unused tsvector FTS indexes from migration 00022 (`idx_zonal_cities_search`, `idx_zonal_barangays_search`) that the old `.fts(simple)` branch relied on
- Applied to production via direct psql, not `supabase db push` (consistent with the 00031/00032 pattern)

## [2.7.0] - 2026-05-15

### Added
- `LicensedBadge` component on broker pages: visible PRC licensure trust signal that shows the PRC license number when claimed and the cohort-verified credential when unclaimed. Returns null when no evidence on file rather than making an unbacked claim
- Social media icons (Facebook, LinkedIn, Twitter/X, Instagram, YouTube, plus a generic fallback) rendered from the previously-unused `social_links` column. Same URLs also feed `Person.sameAs` in structured data
- Badges section on broker pages rendering the previously-unused `badges` array as muted pills, visually distinct from verified badges so it's clear these are broker-supplied labels
- `generateMetaDescription` utility composing varied per-broker meta descriptions from cohort era, seniority, exam location, service areas, and specializations, capped at 160 chars at a word boundary. 11 vitest cases

### Changed
- Broker JSON-LD `@type` upgraded from `Person` to `['Person', 'RealEstateAgent']` for Google rich-results compatibility
- `Person.sameAs` populated from broker `website` plus valid http(s) URLs in `social_links` for entity disambiguation and AI search citation
- `Person.workLocation` now carries `containedInPlace: Philippines` hierarchy on every Place; falls back to exam location for unclaimed brokers so every broker has at least one geographic signal
- Broker pages now declare `isPartOf: REFS.dataset`, wiring the 24K profiles into the site-level identity graph in `schema-entities.ts`
- Broker page meta description no longer a near-duplicate sentence across 24K pages: built from cohort, location, experience, service areas, specializations (replaces the previous one-sentence templated fallback)
- Page title now includes the licensure city when known (`{Name} - Licensed Real Estate Broker, {City}`) so search snippets and tab labels differ between brokers with the same name root
- Cohort narrative now fills the About section for any broker without a user-written bio (previously gated to unclaimed only), so claimed brokers who haven't written a bio see useful 3rd-person prose instead of an empty section. The broker's own bio still wins when set
- Unclaimed claim CTA copy: "Are you {firstName}?" preamble above the Claim button so the empty state reads as a direct prompt

### Notes
- No content is AI-generated. The 24K unclaimed broker pages stay honestly thin in voice (claim-flow-supplied bios only); enhancement is via structured data, metadata variation, and surfacing fields that already existed
- `profile.licenseStatus` is deliberately not consumed by `LicensedBadge` because that field is `'pending'` for every broker today (the claim flow never advances it). Fix belongs in the deferred PRC-verification workstream

## [2.6.0] - 2026-05-15

### Added
- `lts_records` table on production: immutable DHSUD source data with normalized fields (project name, developer, city slug, region), confidence scoring (high/medium/low), parsed dates, and an optional FK to `projects` for linked records. RLS: public read, service-role write
- `get_lts_stats()` RPC: returns total records, confidence breakdown, linked/unlinked counts, active/expired LTS counts, unique developer/city counts. SECURITY DEFINER with hardened search_path
- `link_lts_to_project` and `exclude_lts_record` admin RPCs (gated by `is_admin_or_service()`) for the admin LTS queue
- `scripts/verify-lts-records.ts`: end-to-end regression check (table read, RPC call, row count band, search query) usable against local or production via env vars
- 9,922 LTS records imported to production from the 2026-05-15 DHSUD callback dump (97.5% high confidence, 2.5% medium, 0% low; covers 2,994 unique developers across 550 cities)

### Fixed
- `/verify/lts` and `/admin/lts` showed "0 LTS records" on production: schema drift caused by the original migration `00025_lts_pipeline_rebuild.sql` welding `CREATE TABLE lts_records` to a destructive `TRUNCATE projects/developers CASCADE`, so the migration was deferred indefinitely. Split the migration into a non-destructive form (table + RPCs only, fully idempotent), applied to production, and populated the table. `projects` and `developers` were not touched

### Notes
- **Local-dev recovery:** any developer who previously applied the OLD destructive `00025_lts_pipeline_rebuild.sql` to their local Supabase will have empty `projects` and `developers` tables. The renamed `00025_lts_records_table.sql` is purely additive and will not restore that data. Reseed locally or re-pull from production

## [2.5.0] - 2026-05-14

### Added
- Lead capture modal on barangay zonal value pages: 3-step Yelp-style flow (intent, timeline, contact), automatic lead scoring (hot/warm/cool/cold tiers), Resend transactional email notification for hot/warm tiers
- `/admin/leads` inbox with tier and status filters, location and date filters, per-lead detail view with status updates (new/notified/called/qualified/closed/dead/nurture)
- `/admin/diagnostics` page for verifying production Cloudflare Worker secrets by calling the real APIs (Resend, Turnstile, Supabase), without ever displaying values
- Cloudflare Turnstile bot protection on the lead form with per-IP rate limit (sha256-hashed IP) and honeypot field
- PH mobile number normalizer (`+63` country code) shared across the lead form and future forms
- Migration `00031_lead_captures`: `leads` table with deny-all RLS (anon+authenticated cannot read or write directly; service-role server actions only)
- Setup script runbook under `scripts/setup/` covering: Resend secrets, Turnstile keys, Supabase service-role key, production migration application via direct psql (bypassing the deferred 00025/00026 push trap), DB password rotation across local files, and connection-string testing

### Changed
- Content Security Policy allows `https://challenges.cloudflare.com` in `script-src` and `frame-src` so the Turnstile widget can load

## [2.4.2] - 2026-05-09

### Fixed
- Academy "Last reviewed" pill: force UTC timezone in date formatter so 1st-of-month dates (e.g. 2026-06-01) no longer display the previous month when the runtime is west of UTC
- Academy JSON-LD `datePublished` no longer drops silently when `datePublished` is malformed but defined; falls back to `lastReviewed` independently of `datePublished` validity

## [2.4.1] - 2026-05-09

### Fixed
- Academy Capital Gains Tax lesson: principal-residence exemption corrected from "once in a lifetime" to "once every 10 years" per NIRC Sec. 24(D)(2), in both the Quick Answer block and the pre-existing FAQ
- Academy VAT on Real Estate lesson: Quick Answer now reflects CREATE Act (residential lot exemption repealed except socialized housing under RA 7279) and references the current BIR threshold instead of the pre-TRAIN P1.5M/P2.5M figures
- Academy Documentary Stamp Tax lesson: filing deadline corrected to "5 days after the close of the month of notarization" per Sec. 200 NIRC; customary payor aligned with the FAQ on the same lesson
- Academy Transfer Tax & LRA Fees lesson: scenario and FAQ now agree on how the LRA registration fee scales across the P3M-P5M range
- Academy Commission Calculations lesson: description and Quick Answer reconciled on leasing commission convention
- Academy date validation rejects impossible ISO dates (e.g., 2026-02-30) via round-trip check

### Changed
- Academy lesson and course schemas: separate `datePublished` and `lastReviewed` fields so future review bumps update only `dateModified` in JSON-LD, preserving original publication date for SEO freshness signals
- Academy Quick Answer component renamed from `LessonQuickAnswer` to `QuickAnswerCallout` to reflect dual use on lesson and course pages
- 11 Academy SEO titles trimmed to 60 chars or less to avoid SERP truncation: property classifications, commission calculations, DST, transfer tax, VAT, CWT, security deposit, eviction grounds, lease renewal, max allowable increase, rent control violations

## [2.4.0] - 2026-05-06

### Added
- Academy Quick Answer block: above-the-fold answer callout on every lesson page and on the rent-control-essentials course hub
- Last reviewed date stamp in lesson and course page headers
- `datePublished` and `dateModified` (Schema.org) on Academy `LearningResource` and `Course` JSON-LD via the new `lastReviewed` field
- LRA registration fee schedule topic and FAQ entry in the Transfer Tax & LRA Fees lesson, pointing to the LRA ERCF tool

### Changed
- Rewrote titles, meta descriptions, and Quick Answer copy across 17 Academy lessons and the rent-control-essentials course hub for SERP clarity and consumer comprehension
- Lesson page header metadata row wraps on small screens to accommodate the new "Last reviewed" pill

## [2.3.1] - 2026-04-13

### Fixed
- Empty Kalinga city in extractor (clean name before length guard)
- Garbage "236/22" Manila city from BIR zone-reference formatting errors (reject numeric zone refs like `\d+/\d+`)
- DO number "037-2022" misidentified as Atimonan barangay (REVISION skip now anchored to start-of-string, allowing location names that mention "revision" to pass through)
- Footnote definition rows shadowing data rows with asterisked street names (skip footnote detection when row has a classification code)
- Leading asterisks stripped from street display names (footnote markers, not part of the name)

### Changed
- Barangay page H1, SEO title/description, and Dataset schema now prefix "Brgy." to disambiguate from city pages (368 slug collisions nationwide)
- Zonal import batch size reduced from 500 to 250 for remote Supabase stability
- Import script now retries each batch up to 3 times with fresh Supabase client and exponential backoff; adds 2s delay between batches to avoid socket exhaustion

### Added
- `--resume` flag on zonal import script to continue value inserts after partial failure

## [2.3.0] - 2026-04-05

### Added
- Cloudflare Workers deployment: migrated from Vercel to OpenNext on Cloudflare Workers with R2 cache, D1 tag cache, and ISR
- Content-Security-Policy header with restrictive default policy (includes GA4 allowlist)
- Security headers: HSTS (63072000s, includeSubDomains, preload), X-Frame-Options DENY, nosniff, Referrer-Policy, Permissions-Policy
- Street dedup post-processor with vicinity-aware merge and phantom street filter
- Reverse validator for zonal value pipeline (99.75% accuracy on 1000-sample test)
- Max-value sanity cap and zero-value rejection for zonal value pipeline
- Condo classification columns folded into residential/commercial stats

### Changed
- Hosting: Vercel to Cloudflare Workers ($5/month Workers Paid plan)
- Images set to unoptimized (CF Free plan has no Image Resizing)
- Static page generation limited to 200 cities + 500 barangays (ISR handles rest)
- Removed `x-powered-by: Next.js` header disclosure
- Barangay display names normalized to collapse multiple spaces

### Fixed
- `localhost:3000` baked into all canonical URLs, og:url, sitemap, robots.txt, and JSON-LD when building with local env vars. Build now requires production `NEXT_PUBLIC_APP_URL`
- PostgREST 1000-row limit truncating city zonal values for large cities (batch fetch with size 10)
- Barangay cards showing no values when only street-level data exists (fall back to street-level averages)
- Manila NNN/DD barangay format grounding and stale-DO dedup across all regions
- Phantom streets where display_name equals vicinity stripped from output
- `city-of-` slug prefix normalization across all zonal value query paths
- 6 grounding gaps: province expansion (Davao de Oro/Occidental), IGACOS districts, reverse base number matching, bare Poblacion fallback, compound barangay splitting, manual override ordering
- 4 grounding issues: float barangays, zone sub-headers parsed as cities, province-as-city, footnote filter ordering
- False city import from BIR zone/block numbers parsed as barangays
- Duplicate `validBarangays` declaration in import script preventing compilation
- 10 ghost barangays (empty name/slug parser artifacts) deleted from production
- Ungrounded BIR continuation sections merged into grounded parent barangays
- Rizal cities (Antipolo, Rodriguez, San Mateo, Teresa) misclassified under Metro Manila
- City highlights showing stale barangay defaults instead of street-level data
- Empty-name barangay parsing artifacts appearing in city highlights

### Data Quality
- 95.1% barangay grounding rate (up from ~93%)
- 1,621 cities, 36,080 barangays, 219,644 value rows in production
- 310 barangays recovered via 6 systematic grounding fixes
- Davao City grounding improved from 70% to 94%

## [2.2.0] - 2026-03-19

### Added
- LTS pipeline rebuild: new `lts_records` table replaces `lts_verification_queue` and `project_research_queue`. Single-table design with direct DHSUD re-parse import, shared normalizer module, and streamlined admin review UI
- LTS admin RPC functions (`link_lts_to_project`, `exclude_lts_record`) with SECURITY DEFINER and admin permission checks, replacing direct table writes that were silently failing due to RLS
- Shared normalizer module (`src/shared/lib/normalize.ts`): consolidated developer name normalization, region normalization, and LTS grouping logic extracted from scattered scripts
- React.cache() request deduplication on 8 query functions (profiles, realties, projects, accreditations) to reduce Supabase egress from layout/page/generateMetadata triple-calls
- Query bounds (`.limit()`) on unbounded listing and hub fallback queries to prevent 35K+ row fetches
- Zonal parser: street name normalization for DO merge (periods, initials, suffixes), DO vicinity superseding for stale entries, cross-city PSGC barangay deduplication

### Changed
- LTS admin pages rewritten for `lts_records` schema: queue review, manual entry, expiring records views
- Admin sidebar updated with LTS record counts from new `get_lts_stats` RPC

### Fixed
- LTS admin write actions silently failing: RLS only allowed `service_role` but admin client used `authenticated` session. Now uses SECURITY DEFINER RPCs with `is_admin_or_service()` checks
- 14 audit findings from adversarial review: RPC security (caller permission checks), publish_status validation, DHSUD parser newline handling, date fallbacks, regional format support
- DROP FUNCTION statements with wrong parameter signatures leaving orphaned functions in database
- `get_lts_stats` SECURITY DEFINER function missing `SET search_path = public`
- Manila numbered barangay grounding: plain numbers ("10", "100") now matched as "BARANGAY N". Tondo coverage 0.2% to 52%, Caloocan 0% to 85%, Binondo 0% to 97%
- 18 ungrounded cities resolved via BIR district overrides and DO-number-as-city overrides
- Street suffix stripping preserving distinct street types (RIZAL ST vs RIZAL AVE no longer falsely merged)
- Zonal import truncate timeout on large tables: uses psql TRUNCATE CASCADE instead of PostgREST DELETE

### Removed
- Dead LTS pipeline code: matcher, verification queue, research queue, bulk-create scripts, seed scripts
- Old `lts_verification_queue` and `project_research_queue` tables (replaced by `lts_records`)

## [2.1.0] - 2026-03-16

### Added
- Agentic zonal parser pipeline: 3-stage Python pipeline (structure, extraction, grounding) replaces rule-based parser. Processes 120 BIR RDO Excel files with PSGC entity resolution
- Supabase zonal schema: 6 tables (regions, provinces, cities, barangays, values, slug aliases) with RLS, full-text search indexes, and RPC functions for stats/highlights
- Import script with key-based ID mapping, street slug deduplication, and looped truncate for reliable re-imports
- Paginated query layer (supabase-loader.ts) replacing static JSON data loader. Handles 39K+ barangays and 225K+ value rows without Supabase 1000-row limit truncation
- RPC functions: get_city_stats, get_city_highlights, get_top_cities_by_residential, get_street_counts_per_barangay
- City slug alias resolution for both city and barangay page routes
- Popular cities homepage section powered by server-side aggregation

### Changed
- Zonal value data source: static JSON files (164 MB) replaced by Supabase queries
- City stats computed server-side via single RPC (was duplicated client-side)
- Barangay pages now show real city stats and RDO code/name from parent city
- Zone types expanded: residential_condo, commercial_condo, institutional, government_land added to ZonalValues interface
- Search sanitized against ILIKE wildcards and tsquery parse errors

### Fixed
- Import ID mapping: key-based composite matching replaces positional array index (prevented silent data corruption)
- Deleted values excluded from stats, highlights, and assembled barangay data
- Zero-value zone types preserved (|| null changed to ?? null)
- City redirect loop guard on aliased slugs
- Homepage metadata updated from 2025 to 2026

## [2.0.1] - 2026-03-04

### Fixed
- LTS timezone off-by-one: `getFutureDatePH` used +08:00 offset that caused `setDate`/`getDate` to operate on UTC day, making all "expiring within N days" windows one day short on UTC servers (Vercel)
- LTS placeholder dates: DHSUD sentinel values like "31-Dec-1899" now rejected (pre-2000 dates treated as missing)
- LTS data quality: LTS number format classifier added (standard, regional, amendment, numeric_only) with confidence adjustments for amendment records
- LTS expiry UI: verified projects with missing expiry dates now show dashed badge border and warning message instead of silently omitting the field
- All date comparisons in LTS queries now use Philippine timezone (Asia/Manila) instead of UTC

## [2.0.0] - 2026-03-03

### Added
- Zonal values footnote extraction: BIR Excel footnote legends parsed and linked to street entries. 5,825 streets now carry footnote annotations, 494 streets marked as deleted/defunct (not existing, transferred, merged, consolidated). 19,653 streets and 1,061 barangays had embedded asterisks stripped from display names
- Zonal values validation tool: standalone script comparing parsed JSON against source Excel files, producing diff reports (JSON + markdown) showing parsed, missed, and mismatched values
- Blog system: full MDX-powered blog with post listing, TOC sidebar, share buttons, and SEO metadata
- First blog post: "How to Compute Property Value in the Philippines"
- 3-node JSON-LD @graph schema for blog posts: WebPage (reviewedBy, lastReviewed), TechArticle (mentions, dependencies), HowTo (5 steps for rich snippets). Person references unified to site-wide `/#principal` @id.
- Image MDX component with figure/figcaption rendering
- /insights section with Reports and Blog subsections
- "State of Philippine Real Estate 2025" data report: MDX-powered with sticky TOC, intersection observer active section tracking
- AEO enhancements: JSON-LD Report + Dataset schemas, Key Figures box, APA/MLA citation block with copy buttons
- Downloadable CSV data supplement (864 LTS records from DHSUD 2025 data)
- Enhanced Open Graph tags with structured description for report pages
- Responsive ReportTable component: bordered table on desktop, stacked cards on mobile with auto numeric alignment
- Share buttons (X/Twitter, Facebook, LinkedIn, copy link) on report pages
- Insights link in site header (desktop + mobile) and footer

### Fixed
- LTS Report v1.1: corrected Calamba emerging cities count (12 → 22 to match Top 20 table), clarified city name deduplication methodology note, added changelog section to report
- Zonal values parser audit: 17 bugs fixed (BUG-1 through BUG-17) across critical, moderate, and minor severity. All 120 RDO files re-parsed. Value extraction jumped from ~99K to 1,017,019 values (10x improvement across 2,053 cities)
- Zonal values BUG-1 (critical): multi-sheet parsing now captures all municipalities per Excel file, not just the latest sheet
- Zonal values BUG-2 (critical): street/vicinity state reset correctly between barangays, fixing misattributed streets
- Zonal values BUG-4 (critical): bare "A" agricultural classification code no longer silently dropped
- Zonal values BUG-5 (critical): improved barangay header detection reduces false triggers from data rows
- Zonal values BUG-6: avg_commercial calculated properly in city merges instead of hardcoded 0
- Zonal values BUG-7: summary stats use merged cityIndex to avoid overcounting multi-RDO cities
- Zonal values BUG-9: footnotes split into city-level and barangay-level dictionaries so city footnotes survive barangay-level changes
- Zonal values BUG-10: DO year extraction rejects values < 1990 (were picking up DO number ranges)
- Zonal values: slug collisions resolved. Streets like "A. MABINI ST." and "A. MABINI" both slugified to `a-mabini`, causing 11,432 value mismatches across 61 RDOs. Duplicate slugs now get `-2`, `-3` suffixes. Empty street slugs become `area-1`, `area-2`. L3 accuracy: 98.3% to 100.0%
- Zonal values: deleted/defunct streets excluded from search index, province zone counts, and city min/max residential ranges (premerge finding)
- Zonal values: newlines collapsed in street and barangay display names (premerge finding)
- Zonal values: Sta. Rosa City (Laguna) now correctly separated from Biñan City in RDO 057. 16 barangays (Dita, Balibago, Macabling, Sinalhan, Tagapo, etc.) were incorrectly grouped under Biñan due to a parser regex that didn't handle abbreviated city names with periods (STA., STO., GEN.)
- Zonal values: mixed barangay header formats within a single BIR Excel sheet now handled correctly. RDO 013 (Cagayan) had 19 municipalities stuck at 1 barangay each because some used "Barangay : NAME" format while others used "ZONE/BARANGAY NAME". RDO 078 (Negros Occidental) and RDO 091 (Zamboanga del Norte) had similar issues. Parser now detects barangay headers format-agnostically.
- Zonal values: city names normalized to uppercase across all extraction formats, fixing duplicates from mixed-case Excel headers (e.g., "Himamaylan" vs "HIMAMAYLAN" merged)
- Zonal values: leading colons and trailing underscores stripped from city/barangay names extracted from BIR Excel files
- Zonal values: 3-cell colon format (`Label | : | VALUE`) now detected for barangay/municipality headers, fixing 33 cities in RDO 009 (Benguet), 011 (Kalinga/Apayao), 012 (Ifugao) that had all barangays collapsed into 1
- Zonal values: plural "BARANGAYS" label recognized (RDO 072 Capiz), fixing 4 municipalities (Mambusao, Panay, Panitan, Pilar)
- Zonal values: false positive city detection blocked for parenthetical context like "(WITHIN THE MUNICIPALITY)" (RDO 087 Samar)
- Zonal values: hyphenated city name variants now merge with spaced versions (PRIETO-DIAZ merged with PRIETO DIAZ in RDO 068)
- Zonal values: duplicate "CITY CITY" suffix collapsed (GENERAL SANTOS CITY CITY in RDO 110)
- Zonal values: garbage city names filtered (vicinity descriptions, classification codes as city names)
- Person @id aligned to `/#principal` across all JSON-LD graphs so Google connects them as a single entity
- External nofollow links now correctly get both `nofollow` and `noopener noreferrer` rel attributes
- Markdown tables in report pages rendering as raw pipe-delimited text (missing remark-gfm plugin)
- Narrow tables (2-3 columns) no longer stretch to full width, auto-sized to content instead
- All tables (reports + zonal values) now horizontally scrollable on mobile with sticky first column and fade hint, replacing the card-based mobile layout
- State of RE 2025 report: merged split city name variants in Top 20 ranking (e.g. "City of Calamba" + "Calamba" = 22), clarified total vs. residential unit counts (619,937 total / 329,852 residential), added Data Dictionary, Transformation Notes, and Limitations to Methodology

### Changed
- Updated MASTER-TODO with live database stats: 25,264 broker profiles, 5,046 projects, 2,282 developers, 7,820 LTS records, 1,441 cities, 33,633 barangays, 199,674 zonal value records

## [1.5.0] - 2026-02-20

### Added
- Full effectivity dates extracted from Excel barangay headers (100% coverage: 33,632/33,633 barangays, 1,441/1,441 cities)
- Barangay-level search: 33K+ barangays searchable by name or "barangay city" combo
- RDO-to-barangay mapping: each barangay now carries its specific RDO code and name
- RDO grouping on multi-RDO city pages (Makati, QC, Manila, Cebu City, Davao City)
- GitHub profile (GodModeArch) in person sameAs across all schema entities
- Interactive tax estimator on zonal value pages
- General Purpose (GP) column as distinct BIR classification
- AEO infrastructure: llms.txt, answer-first hero, homepage FAQ schema

### Fixed
- Barangay headline values use median instead of max, fixing 930 barangays inflated by condo outliers
- Province assignments for 85 cities across 13 regions (parser used global name map instead of RDO config)
- Barangay pages in multi-RDO cities now show their specific RDO, not the merged city RDO list
- Parking Slot (PS) separated from Commercial classification
- Street vicinity preservation in barangay headline values

### Changed
- Province derivation: cities inherit province from their RDO config, with cross-province overrides only for RDO 045 (Rizal) and RDO 079 (Siquijor)
- Removed 260-line CITY_PROVINCE_MAP and data-loader workarounds (PROVINCE_FIX, buildAcceptedRegions)
- All pages set to fully static with 24h cache and on-demand revalidation
- AEO bots allowed in robots.txt

## [1.4.0] - 2026-02-13

### Added
- Barangay-level zonal value pages with zone distribution charts
- ISR rendering for barangay and broker profile pages

### Fixed
- Latest DO spreadsheet per RDO used for data generation
- City key normalization strips punctuation correctly

## [1.3.0] - 2026-02-06

### Added
- Enhanced zonal value parser: extracts all municipalities from multi-sheet Excel files
- 1,764 city-level zonal value pages

### Fixed
- Province tagging for Metro Manila cities
- Effective date computed from actual DO year instead of hardcoded 2023

## [1.2.0] - 2026-02-01

### Added
- Trust chain schema for broker credentials
- Verification provenance layer for AEO trust signals
- .well-known identity verification and data provenance table
- FAQPage glossary for RAG system targeting
- Circular trust schema linking ren.ph and godmode.ph identity

### Fixed
- JSON-LD schemas rendered outside Suspense for initial HTML visibility
- Provenance stats updated to 97.7% official / 2.3% third-party
- Zonal value parser accuracy improved to 91% quality score

## [1.1.0] - 2026-01-28

### Added
- Market Snapshot section on zonal value city pages
- BIR source citation on zonal value pages
- Print-friendly due diligence checklist
- Legal pages (Terms of Service, Privacy Policy)

### Fixed
- Duplicate keys in popular cities list
- Multi-RDO city slug normalization

## [1.0.0] - 2026-01-03

### Added
- Broker verification engine with 25,264+ PRC-verified records
- License to Sell verification against 8,690 DHSUD records
- BIR Zonal Values lookup for 34,000+ barangays
- Developer directory with project-centric search
- Schema.org structured data (Phases 1-3: foundation, core entities, directory schemas)
- Broker profile pages with contextual content and SEO enrichment
- Project pages with amenities, units, landmarks, and FAQ schema
- REN Academy with free courses: RA 9646 Essentials, PD 957, Maceda Law, Condominium Act
- Due diligence checklist
- Google Analytics integration
- EEAT micro-copy and footer trust signals
- Trigram indexes for search performance
- Static generation for top 1,000 broker profiles
