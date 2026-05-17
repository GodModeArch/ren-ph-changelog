# Changelog

All notable changes to [ren.ph](https://ren.ph) are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

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
