# Changelog

All notable changes to [ren.ph](https://ren.ph) are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

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
