# Changelog

All notable changes to [ren.ph](https://ren.ph) are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [2.17.2] - 2026-05-28

### Changed
- **Cloudflare cost reduction (Top-3 fixes shipped together).** Investigation traced the $46.81 May cycle to three structural drivers: every deploy invalidated the entire OpenNext R2 cache because `NEXT_BUILD_ID` was a fresh random nanoid each build (cache key includes the buildId), then crawler/user traffic re-hydrated all 60K cached pages via full SSR renders (Workers CPU + R2 PUTs); the Supabase auth middleware called `getUser()` on every public-page request even when OpenNext served HTML straight from R2; and 2-6 deploys per heavy day compounded the cache thrash. Three structural changes target an estimated $25-30/cycle savings ($15-22 final bill).

- **Stable build ID from a hash of tracked source.** `next.config.ts` now sets `generateBuildId` to a SHA-256 of `src/`, `content/` (MDX guides + reports), `next.config.ts`, `package.json`, `package-lock.json`, `tsconfig.json`, `postcss.config.mjs`, `open-next.config.ts`. The hash is computed at build time via `git ls-files -z` + Node `fs`/`crypto` (no shell pipeline), so a deploy with no source change reuses the same build ID, R2 cache keys stay stable, and existing cached pages serve instead of being cold-rendered. A source change still flips the ID and cold-caches everything as intended.

- **Middleware matcher narrowed to authenticated routes only.** `src/middleware.ts` previously ran `updateSession()` on every request matching `/((?!_next/static|...).*)`, which fired `supabase.auth.getUser()` on 60K+ cached public pages with no session-dependent rendering. The matcher now lists only the route trees whose server components or actions actually read the Supabase session: `/admin`, `/auth`, `/claim`, `/listings`, `/profile`, `/realty` (singular). Public realty profile pages live at `/realties` (plural) and are intentionally excluded. All `/api` routes were verified to make zero `getUser()` / `createServerClient` calls and are excluded too. A drift-warning comment in the matcher tells future authors to add new server-auth'd routes to the allowlist or users on the edge of token expiry will bounce to `/login`.

- **Bundle-deploys discipline reinforced (process change, no code).** Heavy days in the May billing cycle (May 9: 3 deploys; May 26: 6 deploys) compounded the per-deploy cold-cache cost. Now that the build ID only flips on actual source changes, non-source deploys (config tweak, dep bump that doesn't alter rendered output) become free for the cache, and the existing Deploy Batching workflow in `CLAUDE.md` matters more, not less.

### Added
- **Dirty-tree assertion in the deploy script.** `scripts/setup/build-with-prod-env.sh` now runs `git status --porcelain` against the same `HASH_INPUTS` paths as `generateBuildId` before building, failing loud with "working tree is dirty in build-ID hash inputs" if either (a) an untracked file under `src/` would be invisible to the hash but bundled into the deploy (producing a stale-cache deploy that looks fresh), or (b) an uncommitted edit would produce a build ID nobody can reproduce from a `git checkout`. The two `HASH_INPUTS` lists in `next.config.ts` and the shell script must move in lockstep (cross-reference comments in both files make this explicit); a future SSOT via shared `.build-hash-inputs.txt` is noted but deferred until the list grows past ~10 entries.

### Documentation
- `CLAUDE.md` "Deploy Batching" section updated to call out the new cache-flush reality: a deploy with zero source changes no longer flushes the OpenNext R2 cache because the build ID stays stable. Workaround for pure data-only deploys (Supabase reimport with no code change) is to bundle the deploy with at least one source change (cheapest: bump a comment in any hashed file). In practice, data fixes almost always ship alongside at least one code fix in the batch.

### Internal notes
- This release pairs with v2.17.1 (SPLE cohort URL gate). Both branches went through two adversarial premerge rounds; the round-1 findings on this branch (rewrote the shell pipeline as Node `execFileSync` + `crypto`, fixed "committed source" wording, added drift warning) and the round-2 finding (added `content/` to both hash and dirty-check lists, closing the silent-stale-cache path for MDX edits) were addressed before merge. Premerge verdict on the merged state was SAFE TO MERGE with the round-2 fix in place.
- The exact savings depend on traffic volume, deploy cadence, and whether a hidden `/api/revalidate { all: true }` caller exists. If May 9's $12 DO spike repeats on the next heavy deploy day, the next branch should hunt that caller down. Otherwise the structural fixes here should land the bill in the $15-22 range and a follow-up branch can pick up the next 4 items in the cost-reduction plan (revalidate caller audit, `/api/track-view` beacon batching, `generateStaticParams` coverage expansion, observability flag).

## [2.17.1] - 2026-05-28

### Fixed
- SPLE cohort URLs (`/cohorts/april-2023-sple`, `/cohorts/june-2024-sple`, `/cohorts/june-2025-sple`) now render. The cohort URL gate `isValidCohortFormat` in `src/domains/directory/queries/hubs.ts` rejected slugs with the `-sple` suffix because its regex only matched two parts (`month-year`), so every SPLE cohort page returned 404 even though the catalog metadata (`src/shared/lib/exam-catalog.ts`), narrative copy (`src/domains/directory/data/cohort-highlights.ts`), and era classification (`src/shared/lib/broker-content.ts`) were already in place for all three SPLE batches. The regex now accepts an optional `-sple` third group.
- Cohort sort and "latest cohort" stats are now deterministic when the same month/year appears as both a regular and SPLE cohort (e.g., june-2024 and june-2024-sple). Without the tie-breaker, sort order flipped between cache refreshes and a city's "Latest cohort" widget could show either label inconsistently. Regular cohort outranks its SPLE sibling, mirroring the administrative ordering of the exam program.

### Changed
- Cohort labels now append " SPLE" for SPLE cohorts across page headers, breadcrumbs, structured data, and broker profile cohort references. `formatCohortDisplay` returns "June 2024 SPLE" instead of "June 2024" when the slug carries the `-sple` suffix; `formatCohortName` returns "June 2024 SPLE Board Exam Passers" for JSON-LD CollectionPage labels.
- Unified the three previously-divergent cohort parsers (`formatCohortDisplay` in `hubs.ts`, `parseCohort` + `formatCohortDisplay` in `format.ts`, `formatCohortName` + `formatCohortMonthYear` in `schema-helpers.ts`) behind a single canonical parser `parseCohort` in `format.ts`. The parser now enumerates full English month names explicitly (`january|february|...|december`) so URL slugs like `foo-2024` no longer parse as valid cohorts. JSON-LD labels and the URL gate share one definition of "valid slug". Broken data renders raw rather than as a plausible-looking fake label.

### Internal notes
- New vitest specs lock in the URL-gate regex (`src/domains/directory/queries/hubs.test.ts`, 14 assertions) and the JSON-LD label formatters (`src/shared/lib/schema-helpers.test.ts`, 6 assertions). Tie-break coverage for `getLatestCohort` includes the regular-vs-SPLE sibling case and ignore-garbage behavior. 564 tests pass across the suite.
- No DB schema or RLS changes; no migrations. If `/cohorts/june-2024-sple` still 404s after the next deploy, the prod `profiles` table has no rows with `exam_cohort` ending in `-sple`. That is a data-import concern separate from this release.

## [2.17.0] - 2026-05-28

### Fixed
- 58 cities filed under the wrong region by a cross-region guard in the agentic parser that was over-firing on legitimate PSGC-grounded matches. The guard at `scripts/agentic-parser/generate_output.py:merge_into_regions()` exists to drop the Tagaytay/Ormoc class of leaks (RDO 89 in Eastern Visayas globally-matching "TAGAYTAY" to Cavite via the grounder's `global` tier and routing Leyte data into Cavite). Its shape is "if PSGC region differs from RDO region, drop the grounding," but that same shape also dropped legitimate matches where BIR's RDO structure deliberately covers cross-region LGUs. Two clusters were caught: (1) the 4 Rizal LGUs City of Antipolo (15 brgys / 948 streets), Rodriguez (11/250), San Mateo (14/419), and Teresa (9/119) carried by RDO 045 Marikina, an NCR RDO whose grounder pool already expands to Rizal/Bulacan/Cavite/Laguna via `METRO_MANILA_ADJACENT_PROVINCES` in `agents/grounding.py`; the guard wiped the calabarzon grounding and the 4 cities ended up under metro-manila with a slug-as-psgc fallback. (2) The 54 BARMM cities served by zamboanga-peninsula RDO 094 Basilan and RDO 096 Tawi-Tawi (15 cities) plus northern-mindanao RDO 102 Marawi/Lanao del Sur (39 cities); PSGC files all BARMM under region 19 which this codebase maps to soccsksargen (where Maguindanao N/S already live post-migration 00035). The 10 other Rizal cities (Cainta, Taytay, Angono, Baras, Binangonan, Cardona, Jala-Jala, Morong, Pililla, Tanay) served via RDO 046 Cainta-Taytay were always correct because RDO 046's manifest already says `region: calabarzon`, so the guard never fired on them. This release adds an explicit `_ALLOWED_CROSS_REGION_GROUNDINGS` whitelist in `generate_output.py` mirroring the grounder's adjacency intent: `metro-manila` is allowed to ground into calabarzon and central-luzon, `zamboanga-peninsula` and `northern-mindanao` are allowed to ground into soccsksargen. The Tagaytay/Ormoc defense stays intact because eastern-visayas has no whitelist entry, so global-tier leaks are still dropped. Negros Oriental (30 cities under RDO 079) intentionally not whitelisted: PSGC files it under region 18 (NIR, abolished by E.O. 38 in 2017 when Negros Oriental went back to Central Visayas administratively), and the cleaner fix there is a per-province `PSGC_REGION_MAP` refactor rather than a whitelist entry that would route 30 cities to Western Visayas listings against admin reality. Tracked as a known divergence in the validator with explicit follow-up notes; the cross-region guard correctly keeps Negros Oriental in central-visayas.

### Added
- Companion data migrations to mirror what the next reground will produce, so cached URLs and DB queries stop misfiling Rizal and BARMM before the reground lands. `supabase/migrations/00039_zonal_rizal_region_backup.sql` snapshots the 4 Rizal cities' current `province_id` into `_zonal_cities_region_backup_20260527`. `supabase/migrations/00040_zonal_rizal_region_fix.sql` repoints the 4 cities to `calabarzon/rizal` via slug-based UUID lookups (no hardcoded prod UUIDs, per the v2.10.2 / migration 00035 lesson). `supabase/migrations/00041_zonal_barmm_region_backup.sql` snapshots the 3 BARMM provinces basilan, tawi-tawi, lanao-del-sur into `_zonal_provinces_region_backup_20260528`. `supabase/migrations/00042_zonal_barmm_region_fix.sql` moves those 3 province rows from zamboanga-peninsula and northern-mindanao to soccsksargen; the 54 cities under them follow by parent-key cascade and need no per-city UPDATE. All four migrations dual-apply against `zonal_staging.zonal_*` per the schema-swap contract introduced in v2.15.0; `scripts/check-zonal-dual-apply.sh` passes clean. Migrations are idempotent on re-run (WHERE `<>` target guards plus ON CONFLICT DO NOTHING on backups).
- `src/shared/config/rizal-region-migration-redirects.ts` plus wiring in `next.config.ts` for 8 308-permanent redirects covering the 4 Rizal cities' old URLs under `/tools/zonal-value/metro-manila/...` plus their `:barangay` wildcards. BARMM URLs need no redirects because the URL pattern is `/tools/zonal-value/{province_slug}/{city_slug}` and the BARMM province slugs (basilan, tawi-tawi, lanao-del-sur) don't change; only the region the city appears under in listing pages and breadcrumbs shifts.
- `scripts/agentic-parser/test_cross_region_guard.py` (new, 7 tests) exercises the guard at the merge layer: Rizal cities on RDO 045 pass through, Tagaytay/Ormoc-style cross-region leak still dropped, Bulacan on NCR RDO passes through (central-luzon side), Cainta on RDO 046 baseline unaffected, Basilan on zamboanga-peninsula RDO 094 passes (BARMM), Lanao del Sur on northern-mindanao RDO 102 passes (BARMM), Negros Oriental on RDO 079 still drops (pinning the current deferral policy). The original resolver-level test `test_resolve_city_region.py` could not catch this class because the guard runs upstream of `_resolve_city_region` and erases the input the resolver would have decided on correctly; the new test sits at the same layer as the guard. `test_resolve_city_region.py` itself gains 8 cases (4 Rizal-on-RDO-045, 3 BARMM, 1 Negros Oriental policy pin) so both layers stay covered.
- A new C36 validator check in `scripts/agentic-parser/validate_province_attribution.py` that catches the broader class: ungrounded cities (psgc_code is the slug, not a 10-digit PSGC code) whose grounded barangays all sit in the same off-region province. C35 alone could not see this class because it skips non-numeric psgc_codes (line 74). C36 walks the barangays under each ungrounded city, counts which PSGC region they collectively point to, and flags when 90%+ of them agree on an off-file region. A `_KNOWN_C36_DIVERGENCES` set tracks expected noise (currently only Negros Oriental's central-visayas to western-visayas pair, pending the per-province `PSGC_REGION_MAP` refactor) so the validator can gate regrounds against new regressions while staying quiet about the deferred case. A `_C36_MIN_BARANGAY_COUNT = 2` threshold suppresses single-barangay false positives like the city-of-sagay case (which was actually the guard correctly dropping a Tagaytay/Ormoc-style global-tier false match; the original `_KNOWN_C36_DIVERGENCES` list had incorrectly lumped Sagay with the real bugs).

### Internal notes
- Sequencing: parser fix is the source-of-truth change that makes future regrounds correct; migrations 00040 and 00042 mirror what the next reground will produce so live URLs work in the meantime. Without the migrations, prod stays on its current misfile until the next reground completes (~20-40 min reground + import + cache-flush deploy). With the migrations applied first and the parser fix shipped, the next reground will re-import the same rows into the same shape and the migration is silently idempotent. Order doesn't matter operationally.
- Premerge findings (all minor, non-blocking): (1) `zonal_cities(province_id, slug)` and `zonal_provinces(region_id, slug)` are UNIQUE; the migrations rely on WHERE-clause idempotency rather than a defensive `EXCEPTION WHEN unique_violation` block. None of the moving slugs (city-of-antipolo, rodriguez, san-mateo, teresa under rizal; basilan, tawi-tawi, lanao-del-sur under soccsksargen) collide with existing entries on prod. (2) `_C36_MIN_BARANGAY_COUNT = 2` is intentionally generous; a genuinely-misrouted city with only 1 grounded barangay would slip past. All 58 cities in this fix have 9+ brgys, so the threshold catches the in-scope class. (3) Negros Oriental policy is deferred but documented; the redirects file has no entries for the eventual Negros move, so the per-province refactor PR will need to add them itself. (4) Rollback instructions are header comments in 00039 and 00041, not standalone `rollback-XXXXX.sql` scripts; acceptable for a 4-city / 3-province blast radius.
- Operational follow-ups for the next reground + import: the C36 violations report should drop from 56 to 0 in metro-manila + zamboanga-peninsula + northern-mindanao (the 30 Negros Oriental warnings stay until the per-province refactor); the 4 Rizal slugs disappear from `metro-manila.json` and surface in `calabarzon.json` with real PSGC codes (0405802000, 0405808000, 0405811000, 0405814000) instead of slug-as-psgc; the 54 BARMM cities consolidate under soccsksargen alongside Maguindanao N/S; users reaching the old `/tools/zonal-value/metro-manila/city-of-antipolo` family get a 308 to `/rizal/...`; BARMM URLs continue to resolve at their existing `/basilan/...`, `/tawi-tawi/...`, `/lanao-del-sur/...` paths.
- This release ships parser code, 4 migrations, 7 new merge-guard tests, 8 added resolver test cases, new C36 validator coverage, 8 redirects, and documentation. The visible improvement for Rizal takes effect at the next deploy once migration 00040 has applied (cache-flush deploy required since the URL plumbing is in `next.config.ts`); the BARMM improvement takes effect at the next reground and reimport since the migration 00042 province move shifts where queries land but the underlying cities still hold their slug-as-psgc until reground.
- Dual-apply contract from v2.15.0 is exercised here for the first time across two migrations; both 00040 and 00042 ship PHASE C blocks that mirror PHASE B against `zonal_staging` and gracefully no-op when the staging schema is empty (fresh local) or the staging province rows don't exist yet.

## [2.16.5] - 2026-05-27

### Fixed
- The 582 barangays still rendering empty effectivity dates after the v2.16.4 RDO 102 MM-DD-YYYY fix. Probed one source RDO per geographic concentration (western-visayas 332, ilocos-region 58, soccsksargen 41, northern-mindanao 29) and found four distinct BIR layout variants, all of which now recover via small targeted patches in `scripts/agentic-parser/agents/extractor.py`. (1) RDO 071 Kalibo Sheet 6 (`DO 010-2021`) misspells "Effectivity" as "Effrectivity" (extra R between EFF and E) on 130 per-section header rows; the v2.16.4 substring guard accepted only `EFFECTIV` and `EFFECIV`, so those 130 dates dropped silently — and the 208 dateless peer barangays in the same sheet that would inherit via `ctx.effectivity_date` propagation stayed blank too. Adding `EFFRECTIV` to the substring guard plus matching updates in `_clean_location_name`'s strip regex and `_extract_header_value`'s skip list recovers all 338 RDO 71 brgys (130 direct + 208 via ctx). (2) RDO 001 Laoag Sheet 8 (`DO 64-17`) writes per-section dates as `22-Dec.-17` with a period between the month abbreviation and the second dash; the existing pattern's `\w*` after the month didn't consume the period, so the row matched nothing. The pattern now allows an optional `\.?` between the month and the dash. (3) RDO 111 Koronadal Sheet 1 (`DO 78-94`) writes per-section dates as `09-28--94` with two consecutive dashes between day and year; the existing pattern required a single dash. A one-shot `_MULTI_DASH_RE.sub("-", row_text)` collapse runs after the substring guard passes, so date strings like `09-28--94` normalize to `09-28-94` before regex matching and downstream `_parse_date` handles them via the existing `%m-%d-%y` format. The collapse is scoped to rows that already contain `EFFECTIV`/`EFFECIV`/`EFFRECTIV`, so it cannot accidentally normalize unrelated cells. (4) RDO 101 Iligan Sheet 2 (`DO 6-95`) uses a multi-barangay BARANGAY header (a single header naming five-to-twenty barangays separated by commas across continuation rows, producing a slug like `alegria-babalaya-babalayan-townsite-binuni`) immediately followed by a MUNICIPALITY row that carries the effectivity date: `MUNICIPALITY: BALO-I | Effectivity Date | 1995-04-19`. The city handler's four return branches (MUNICIPALITY single-cell with colon, `MUNICIPALITY OF`, `CITY/MUNICIPALITY`/`CITY :` labels, and bare city header) used to set `ctx.city` and return without calling `_extract_effectivity_date(row)`, so the date never reached the active brgy ctx. All four branches now harvest same-row dates before returning. Same shape applies to RDO 109 Tacurong and RDO 111 Koronadal's multi-brgy headers. Real-source probe on the four sample sheets: RDO 71 Sheet 6 went from 338 dateless barangays to 1 (a classification-as-brgy noise row unrelated to this fix), RDO 1 Sheet 8 from many to 1, RDO 101 Sheet 2 22/22 recovered, RDO 111 Sheet 1 29/29 recovered. The visible improvement takes effect on the next zonal reground and reimport. The RDO 110 `Sarangani Province` summary tab (13 brgys with no DO references anywhere in the sheet body) is deferred — would need NOTICE-tab cross-referencing for a 13-brgy recovery, not worth the complexity for v2.16.5.
- The page-render layer's `ZonalCitySearch.tsx` component on barangay pages still showed only 7 of the 10 BIR zonal value classifications. The other three — industrial (already wired but undocumented), institutional (BIR code `X`), and government_land (BIR code `GL`) — were correctly parsed by the agentic parser, stored in the production database, and exposed in `ZonalValues` types and the Supabase loader as of earlier releases. Only the UI plumbing was missing. Forbes Park (Makati) has an institutional row in its `DO 38-2021` source at PHP 369,000 per sqm sitting in the production database since the v2.16.4 reground, but the table on `/tools/zonal-value/metro-manila/city-of-makati/forbes-park` silently dropped that row. This release extends `ZoneRow` in `src/domains/zonal-values/queries/city.ts` with `institutional` and `governmentLand` fields, populates both in `flattenZones` (city query) and `flattenBarangayStreets` (barangay query), adds `showInstitutional` and `showGovernmentLand` conditional columns and props to `ZonalCitySearch.tsx`, extends the `hasAnyValue` check to include both fields, and derives the show-flags on the barangay page from `streets.some(z => z.institutional !== null)` and `streets.some(z => z.governmentLand !== null)`. The city page is unaffected — it renders summary `barangayList` rows from a different query path. Headers read `Institutional` and `Govt Land`; the columns only render when at least one street in the barangay has a non-null value, so brgys with no `X` or `GL` data show no change.

### Added
- `scripts/agentic-parser/test_v2165_effectivity_recovery.py` — 10 unit tests covering each of the four layout variants (Effrectivity typo, dot-dash separator, double-dash separator, city-row date harvest after multi-brgy header), the no-date-still-sets-city regression case, and end-to-end round-trips through `generate_output._parse_date` for the three new date formats. Full 22-file agentic parser test suite continues to pass.

### Internal notes
- The `scripts/validate-zonal-staging.ts` barangay-count range hotfix flagged in the v2.15.0 cutover memory had already shipped in commit `768ed64` — the validator's `barangays: { min: 36000, max: 42000 }` matches actual prod numbers since v2.14.0 (~38,879 to 39,093). No code change needed; documenting here to close the loop. The `--skip-validate` workaround mentioned in earlier release notes is no longer required.
- The Effrectivity typo handler is intentionally loose — `EFFR?E?C?TIV\w*` in `_clean_location_name` also matches `EFFTIV`, `EFFETIV`, `EFFCTIV`. Premerge round flagged this as a minor permissiveness concern; the risk of false-positive against a real Philippine place name is effectively nil (no PSGC entry starts with EFF…TIV), but worth noting for future maintainers.
- Sister-RDO sweep deferred: the RDO 110 `Sarangani Province` summary tab and similar legacy no-DO-in-sheet tabs in soccsksargen would need a NOTICE-tab cross-reference mechanism to date their brgys. ~13 brgys nationwide fall in this class — out of scope for v2.16.5. Will revisit if a future probe surfaces a broader pattern.
- The dual-apply contract introduced in v2.15.0 is not exercised here — this release adds no migrations.
- This release ships parser code, UI plumbing, tests, and documentation; the visible barangay-table column improvement takes effect at the next deploy (no DB change required since the data is already there), and the effectivity-date improvement takes effect on the next zonal reground and reimport.
- CLAUDE.md gains a "Premerge Convention" section: every `/run-premerge` recommendation must now include the exact worktree path and branch the new session should open in, so the assistant no longer needs to be asked "which worktree?" each time.

## [2.16.4] - 2026-05-27

### Fixed
- Barangay zonal value pages displaying stale Department Order labels in their header. Forbes Park (Makati) read `DO 135-91 · Effective: 1992-07-31` at the top of its page despite every surviving street being from `DO 38-2021` at PHP 338,000 per sqm residential — modern 2021 prices under a 1991 DO label. Imus barangays Anabu I-A, Bayan Luma I, Medicion I-A and ~325 others nationwide read `DO 39-07` or similar headers while serving 2021 data. The street values themselves were correct (v2.16.2's R37b coverage supersession dropped the older-DO rows); the brgy-level `rdo_code`, `rdo_name`, `source_sheet`, and `additional_rdos` metadata that drives the header label was written pre-dedup by `generate_output._aggregate_barangay_rdo` and never refreshed after R37b dropped its older counterpart rows. If a barangay had 50 streets from RDO 049's 1991 DO and 24 from RDO 050's 2021 DO at cross-RDO merge time, the dominant-count algorithm picked 049 as primary; R37b then dropped the 50 older rows leaving only the 24 from RDO 050, but the brgy metadata still pointed at RDO 049. This release adds a post-dedup recompute pass in `scripts/agentic-parser/dedup_streets.py` (`recompute_barangay_metadata`) that mirrors the pre-dedup aggregator's tiebreaker (-count, -do_year, code) but operates on POST-R37b surviving streets. Wired into `process_region` AFTER all dedup/prune passes complete and BEFORE `strip_internal_fields` removes the source_rdo_code / do_year / source_sheet scaffolding the recompute reads. The same change adds a `metadata_refreshes` counter to the dedup_streets summary (2,649 brgys nationwide had their metadata corrected on the v2.16.4 reground), and adds `or metadata_refreshes` to the file-write trigger so a region whose only change is a metadata refresh still gets persisted. Forbes Park now reads `DO 38-2021 · Effective: Dec. 22, 2021` (after this release also unlocks the date plumbing, below).
- The 1,177 barangays in RDO 102 (Marawi City / Lanao del Sur / BARMM, covering 53 municipalities including Amai Manabilang, Balindong, Buadiposo, Ditsa-an Ramain, Ganassi, Madalum, Madamba, Marogong, Masiu, and others) showed empty effectivity dates on their zonal value pages. The dates were actually in the BIR sheet: per-section `Effectivity Date | 12-15-2020` rows immediately below each `BARANGAY: AMBOLONG` header. The agentic parser's `agents/extractor.py:_DATE_PATTERNS` had four regexes covering month-name formats, ISO `YYYY-MM-DD`, and `M/D/Y` with slashes — none matching `MM-DD-YYYY` with dashes. So `_extract_effectivity_date` returned empty for all 1,177 RDO 102 barangays, and the brgy-level `effectivity_date` field stayed `null` all the way through to the database and the rendered page. This release adds a fifth regex `\b(\d{1,2})-(\d{1,2})-(\d{2,4})\b` to `_DATE_PATTERNS` with word-boundary anchors so it doesn't match parts of longer numbers or the tail of the ISO `YYYY-MM-DD` pattern (which sits earlier in the list and wins for ISO strings). The mirroring change adds `%m-%d-%Y` and `%m-%d-%y` to both `generate_output._DATE_FORMATS` and `dedup_streets._DATE_FORMATS` so date comparisons agree across the pipeline. Per the 2026-05-20 `prompts/2026-05-20-zonal-do-date-investigation-handoff.md` and `memory/project_zonal_do_date_fixes.md`, this is the "future parser session" that the prior date-format work explicitly punted on. Same shape as v2.16.3's Bohol column-header trap — small extractor patch plus matching format-list additions. Nationwide brgys with empty effectivity_date drop from approximately 2,142 (per the pre-fix Investigation 1 baseline) to 582, a 1,560-brgy improvement; remaining concentrations sit in western-visayas (332), ilocos-region (58), soccsksargen (41), and northern-mindanao (29) and likely use yet another date-layout variant — tracked for v2.16.5.
- The BIR `Effecivity` typo (missing the second T) in RDO 050 Sheet 9 (`DO 38-2021`) and similar Makati sheets caused the extractor to silently skip the per-section effectivity date row, leaving Forbes Park, Pinagkaisahan, Dasmariñas, and likely dozens of other Makati barangays on a stale cross-RDO-merge date inherited from the older DO 135-91. The `_extract_effectivity_date` substring check `if "EFFECTIV" not in row_text.upper()` failed against `EFFECIVITY` (missing the T between EC and IV), so the date `Dec. 22, 2021` sitting in column 3 of row 1489 never reached the barangay. Bel-Air looked correct on the page (`1/8/2022`) only because it gets the date from RDO 049 where the spelling is right; barangays whose ONLY source was the typo'd sheet stayed broken. This release tolerates the typo by also accepting the `EFFECIV` substring; `EFFECIV` is a strict subset of correctly-spelled `EFFECTIVITY` so this doesn't false-fire on properly-spelled rows. Post-fix Forbes Park reads `effectivity_date = Dec. 22, 2021`; same fix unlocks Pinagkaisahan, Dasmariñas, and other Makati DO 38-2021 brgys whose date sources are typo'd. Bel-Air's metadata is unchanged (already correct via its other source RDO).

### Added
- `scripts/agentic-parser/test_metadata_recompute.py` — 9 unit tests covering the Forbes Park anchor shape (winners all from one DO, primary flips to that DO), mixed-winners (additional_rdos populated correctly), single-source no-op, no-streets edge, count/year/code tiebreaker (both rungs), partial-missing source_rdo_code handling, and idempotency under double-run.
- `scripts/agentic-parser/test_mm_dd_yyyy_dates.py` — 10 unit tests covering the MM-DD-YYYY round-trip from extractor through both `_parse_date` and `_to_iso`, no-regression on the existing 4 date patterns, ISO not mis-matched as MM-DD-YYYY, and the BIR `Effecivity` typo tolerance.

### Internal notes
- This release ships parser code, two test suites, and an audit script; the visible header-label improvement takes effect on the next zonal reground and reimport. All 19 existing parser test suites continue to pass — `recompute_barangay_metadata` runs strictly AFTER all dedup/prune passes (Pass 0 R37b coverage supersession + Pass 1 row dedup + Pass 2 subdivision prune + Pass 3 annotation prune + Pass 4 legacy short-form + Pass 5 stale-DO merge) and BEFORE `strip_internal_fields`, so downstream consumers see the same row shape they always did.
- Two premerge rounds. Round 1 covered Part A + Part B; round 2 (after the Effecivity typo commit) confirmed SAFE TO MERGE for all 4 commits with three minor non-blocking findings (no-counts contract divergence between recompute and pre-dedup aggregator — theoretical only; city-level RDO metadata still pre-dedup — out of scope for v2.16.4; one-off audit script has a hardcoded path).
- Out of scope and tracked for v2.16.5: the page-render layer's `ZonalCitySearch.tsx` component only shows 7 of 10 classification columns (residential, commercial, agricultural, general_purpose, parking_slot, residential_condo, commercial_condo). The other three — industrial, institutional, government_land — are correctly parsed and stored in the database but never rendered on the public page. Forbes Park has an `Institution / School / Embassy` row at `institutional: PHP 369,000 per sqm` in its DO 38-2021 source and in the production database, but the table currently hides it. v2.16.5 will add the missing three columns plus their show/hide flags.
- The sister-RDO scan for similar effectivity-date extraction misses (western-visayas, ilocos-region, soccsksargen, northern-mindanao concentrations totaling roughly 460 brgys) is tracked under the v2.16.5 follow-up; the layout variants probably differ from RDO 102's MM-DD-YYYY pattern and need their own probes.
- The dual-apply contract introduced in v2.15.0 is not exercised here — this release adds no migrations.

## [2.16.3] - 2026-05-27

### Fixed
- Bohol barangay zonal value pages trapped on 2002 prices despite 2022 and 2024 Department Orders being present in our spreadsheet stash. The full 48 municipalities of Bohol (under RDO 84 Tagbilaran City) plus Calasiao under RDO 4 (Central Pangasinan) had been rendering their pages off `DO 045-02` (Dec 28, 2002) values even though `DO 20-22` (May 14, 2022) and `DO 043-2024` cover the same barangays in BIR's newer sheets. The newer-DO data lived in 1,167 source rows but landed in phantom municipality-level slugs of the shape `all-barangays-of-tagbilaran-city`, `all-barangays-of-inabanga`, etc., holding 2,525 stranded street rows across the 49 phantom slugs that no real barangay page could ever read. Root cause: BIR's newer Bohol sheets emit a column-header row immediately after each barangay header reading `STREET NAME/BARANGAY  SUBDIVISION/CONDOMINIUM | VICINITY | CLASSIFICATION | 1ST REVISION ZV/SQM`. The agentic parser's `extract_location` did a substring match for `BARANGAY` against the row text; the literal token `BARANGAY` appears inside the column-header phrase `STREET NAME/BARANGAY`, so the match fired one row after the real barangay header had set `ctx.barangay = "EAST POBLACION"` (or "ILAUD (POBLACION)", or whichever section). The substring match then passed the row to `_extract_header_value`, which greedily took the next non-empty cell (`VICINITY`) as the barangay name, clobbering the real context. `VICINITY` is in the grounder's wildcard allow-list (`_is_wildcard_entry` at `agents/grounding.py:204-211`), so every subsequent data row got routed to the municipality-wildcard slug `all-barangays-of-{city}` with the municipality PSGC code instead of the real barangay. This release adds a `_is_column_header_row` discriminator in `scripts/agentic-parser/agents/extractor.py` that rejects rows whose cell 0 contains both `BARANGAY` and `STREET`, or whose cell 1 is the literal `VICINITY` with cell 2 or 3 carrying classification or ZV column headers. The guard runs immediately after `_is_data_row` in `extract_location`, so any row that already qualifies as a value row passes through untouched and the column-header row is now silently skipped. Nationwide probe (`audit/bohol-column-header-leak-probe.py`) confirms only two RDOs hit this trap: RDO 84 Bohol with 1,165 trap rows across Sheets 5 and 6, RDO 4 Calasiao with 2 trap rows in Sheet 12. Eighty-eight other RDOs have a similar `VICINITY | CLASSIFICATION` column-header layout but their col 0 reads `STREET NAME` or `STREET / SUBDIVISION` without `BARANGAY`, so they never tripped the original substring match and remain unaffected by the new guard. Expected impact on the next reground: ~49 phantom municipality-wildcard slugs disappear from the grounded artifact, the 2,525 stranded street rows re-attach to ~1,150 real barangay slugs across Bohol and Calasiao, and the existing R37b/R37c coverage supersession shipped in v2.16.2 then cascades — the 2002 `DO 045-02` rows that were also published in the same barangay get dropped in favor of the 2022/2024 data, dropping Bohol's "frozen on 2002" cluster from the old-DO cluster (an audit-side count of 3,283 brgys nationwide whose winning DO is itself pre-2010 shrinks to roughly 2,680).

### Added
- `scripts/agentic-parser/test_column_header_row_fix.py` — 13 unit tests covering all three observed trap variants (Bohol Sheets 5 and 6, RDO 4 Calasiao Sheet 12), confirming real barangay headers from both sheets are not flagged, data rows including ones with `BARANGAY` inside vicinity text are not flagged, municipality headers are not flagged, the end-to-end `extract_location` regression case where `ctx.barangay` must stay at `EAST POBLACION` across the column-header row, and the next-barangay-header-still-works guard. All 9 existing parser test suites continue to pass.
- `audit/phantom-multi-brgy-concat.py` and `audit/bohol-column-header-leak-probe.py` — two re-runnable diagnostic scripts plus four findings documents (`audit/bohol-rdo-84-root-cause-2026-05-27.md`, `audit/bohol-rdo-84-quantification-2026-05-27.md`, `audit/phantom-multi-brgy-concat-findings-2026-05-27.md`, original Bohol hypothesis preserved at `audit/bohol-rdo-84-format-bug-findings-2026-05-27.md`). The multi-brgy concat findings doc quantifies a second phantom class still to fix (124 HIGH and 456 MEDIUM concat phantoms nationwide where BIR DOs use a single `BARANGAYS:` header naming 6 or 25 barangays across 5 continuation rows, and our parser captures only the first row, silently dropping the continuation barangays).

### Internal notes
- This release ships parser code, a unit test, and audit scripts; the visible data improvement takes effect on the next zonal reground and reimport. Until then the on-disk grounded JSON is unchanged and the affected Bohol and Calasiao pages render the same as v2.16.2.
- Earlier hypothesis-vs-reality contrast preserved in the audit folder: the initial finding doc proposed a 50-200 line column-layout dispatcher to teach the extractor a new 4-column header format. Live-tracing `extract_location` against the failing rows broke that hypothesis in minutes — the real barangay header row was already parsing correctly; the bug was the column-header row one line later that re-fired the substring match. The corrected root-cause walkthrough at `audit/bohol-rdo-84-root-cause-2026-05-27.md` documents the trace.
- Premerge verdict: SAFE TO MERGE with three MINOR items, none blocking. All 19 parser test suites pass; three-way merge dry-run is clean against v2.16.2's `effectivity_refreshes` write trigger; the discriminator's belt-and-suspenders second clause (cell 1 == `VICINITY` plus classification/ZV in cells 2-3) is broader than strictly needed but verified safe — those rows were already no-ops in `extract_location` (they fell through every label loop without matching).
- Out of scope and tracked: the multi-barangay comma-concatenation phantom class quantified in `audit/phantom-multi-brgy-concat-findings-2026-05-27.md` is a separate parser fix (extractor-side header splitter for multi-row `BARANGAYS:` continuation rows). The follow-up worktree at `rendir-phantom-brgy/` still carries those findings and scripts pending a future version bump.

## [2.16.2] - 2026-05-27

### Fixed
- Stale older-Department-Order rows leaking into barangays whose newer DO has authoritative coverage. The previous row-level merge keyed each value on `(city, barangay, street_normalized, vicinity_normalized, class_code)` and let any older-DO row survive whose normalized slug did not appear in the newer DO. The legal authority on this is clear: the Supreme Court in *CIR v. Aquafresh Seafoods* (G.R. No. 170389, 2010) held that the zonal valuation existing at the time of the sale should be taken into account, and BIR Ruling No. 041-2001 lists the within-schedule fallbacks (`ALL OTHER STREETS`, adjacent barangay of similar conditions) for streets not individually listed in the current schedule — older Department Orders are not among those fallbacks. The old row-grain merge let older rows linger anyway: Anos in Los Baños carried `CARBERN SUBD` at PHP 550 per sqm and `BARANGAY ROAD` at PHP 750 per sqm sourced from `DO 8-91` (1991, RDO 057 Biñan, West Laguna) alongside the DO 059-23 (Oct 13, 2023, RDO 056 Calamba) data that covers the barangay; Quezon City's Tandang Sora carried 89 leaking rows from `DO 008-2020` against `DO 037-2024`; Bel-Air in Makati carried 55 rows from `DO 35-2021` against `DO 38-2021` in the same year; Masambong (QC) carried a 29-street `SAN JOSE` vicinity cluster flat at PHP 2,500 from `DO 29-93` (RDO 028) against a 2024 DO 033-2024 that already covered the barangay. This release adds a new first pass to the zonal parser (`coverage_supersession_pass` in `scripts/agentic-parser/dedup_streets.py`) that operates at `(city, barangay)` grain before the existing row-level dedup. For every barangay, the pass identifies the winning Department Order (newest year contributing rows after R45 footnote exclusions) and drops every row whose source DO is older than the winning DO; the existing R37-R40 row-level merge (vicinity supersession, equal-effectivity MAX selection, etc.) then runs on the survivors with its rules unchanged. The default same-year tiebreaker is `strict_year` (per spec R37c.1) — two same-year DOs covering one barangay both survive Pass 0 and fall through to the existing row-level dedup, so 91 Bel-Air-class rows from `DO 35-2021` stay alive next to `DO 38-2021` pending a BIR-domain confirmation that DO 38-2021 is a full re-issue rather than a partial amendment; an alternative `by_source_sheet` strategy is implemented and unit-tested for a future flip if the BIR-domain answer comes back the other way. Nationwide impact on the next reground: 12,070 row drops across 2,648 barangays. Top regions Central Luzon 4,960, Metro Manila 4,249, Calabarzon 2,017, Central Visayas 713. Dominant leak ages 1993 (3,908 rows, the 1993 NCR-wide DO that was still surviving in most QC and Manila barangays), 2009 (2,040), 2005 (2,006), 2007 (1,848). Visible effect on the affected pages: stale rows disappear, the page's apparent floor value rises to the current schedule's actual minimum, and any user who searches for a street that the current DO does not individually list resolves through the within-schedule fallback (the page's `ALL OTHER STREETS` catchall plus vicinity) rather than to a 32-year-old value.
- Stale `effectivity_date` rollup on barangays whose cross-RDO merge picked an older Department Order's per-section date. Pinagkaisahan in Makati read "Effectivity Date: 1992-07-31" on its zonal value page despite carrying 2021 streets, because the cross-RDO merge in v2.16.0 chose the older DO's parseable date over the newer DO's empty per-section row. The new pass now refreshes `brgy.effectivity_date` to the winning DO's date when surviving winners carry a parseable date; when they do not, a separate `effectivity_stale_no_refresh` counter surfaces the count for follow-up (429 brgys in metro-manila on the current artifact — the page would otherwise show an older-DO label on what are now newer-DO street values). The refresh fires even when no rows are dropped (single-DO barangay whose rollup was stamped with a stale date upstream), and the file-write trigger correctly persists the refresh-only case.
- `do_year=0` rows from BIR DOs predating 1990 (the `DO 10-89`, `DO 26-89` family) were treated as having no parseable year and skipped by the supersession comparison, leaving 35 metro-manila rows in Manila's barangay-649 and barangay-897 visible alongside 2017 DO data. The new `_effective_do_year` helper re-scans the source sheet name for a pre-1990 token; matches synthesize year 1989 (so the row drops against any newer DO, which is the correct outcome — a 1989 row next to a 2024 DO is stale by definition); non-matches return `None` (so the row is preserved when the year is genuinely unknown, since dropping it would be silent data loss). The same helper is used by the new `validate-c33-coverage-supersession.py` validator so audit results agree with the parser's behavior.

### Added
- `scripts/agentic-parser/validate-c33-coverage-supersession.py` — exits 1 if any post-fix row violates the supersession rule for its barangay's winning DO. Currently exits 0 (zero violators) against the grounded artifact, becoming a permanent regression gate on future regroundings.
- `scripts/agentic-parser/validate-c34-orphans.py` — non-fatal reporter for phantom barangays (`psgc=None`). Surfaces 3,834 phantoms nationwide today; resolution is a separate follow-up class (see Internal notes).

### Internal notes
- This release ships parser code, validators, and tests; the visible data improvement takes effect on the next zonal reground and reimport. Until then the on-disk grounded JSON is unchanged and the page renders the same as v2.16.1.
- 18 unit tests added under `scripts/agentic-parser/test_coverage_supersession.py` covering the canonical Anos and Masambong cases, the phantom-barangay-only-old-DOs case, both same-year tiebreaker strategies, do_year=0 disambiguation (pre-1990 floor vs parser failure), the Pinagkaisahan effectivity refresh, the refresh-fires-without-drops regression case, and the stale-no-refresh flag invariants. All existing parser tests (`test_all_lots_fix.py`, `test_subdivision_as_brgy_fix.py`, `test_legacy_short_form_fix.py`) continue to pass — the new Pass 0 runs strictly before the existing Pass 1 (`dedup_barangay_streets`) and only reduces `brgy.streets` cardinality, so downstream passes see the same row shape they always did. Four premerge rounds (the round-3 finding caught a missed file-write trigger for single-DO refresh-only barangays; the round-4 verdict was SAFE TO MERGE).
- Out of scope and tracked separately: a phantom-barangay class where newer-DO data lives under a different slug than the older-DO data on the same physical area (Makati's `legaspi-village` vs `san-lorenzo` shape; Bohol's `all-barangays-of-X` municipality-level slugs trapping newer-DO data behind a Structure-Agent header-format failure; sub-area / sitio / pre-redistricting BIR names in the BARMM region that don't match modern PSGC; multi-barangay comma-concatenation header rows that glue 6 or 20 barangays into one phantom slug). Together this class accounts for 3,283 brgys (8.3% of all barangays nationwide) whose winning Department Order is itself pre-2010, including the entire RDO 84 Bohol coverage that BIR last refreshed in DO 045-2002 plus the newer DO 043-2024 / DO 20-22 / DO 33-20 sitting in our spreadsheet stash but trapped behind the parser failure. The follow-up worktree at `rendir-phantom-brgy/` carries the audit scripts and findings.
- The dual-apply contract introduced in v2.15.0 (any change to a `public.zonal_*` table must also apply to `zonal_staging.zonal_*`) is not exercised here — this release adds no migrations.

## [2.16.1] - 2026-05-26

### Fixed
- Stale 1993-era zonal value rows surfacing on 2024 pages. Several Quezon City barangay pages were rendering both their current Department Order's prices and orphaned rows from the 1993 BIR DO 29-93 schedule under district-rollup vicinity labels (NOVALICHES, BAGO BANTAY, MURPHY, CUBAO, SAN JOSE, LA LOMA, CAPITOL, BALINTAWAK, QUADRANGLE, CENTRAL). The 1993 rows showed residential floor values as low as PHP 1,200 to PHP 2,500 per sqm next to the modern rows at PHP 16,000 to PHP 200,000 per sqm on the same page, dragging visible floors to misleading levels. Kaligayahan had 17 stale rows including bare `ZABARTE SUBD` at PHP 1,200 alongside its modern counterpart `NORTH ZABARTE SUBD (fmly: zabarte subd)` at PHP 16,000 from DO 037-2024. Masambong had 29 stale rows including an 11-street SAN JOSE vicinity cluster all at PHP 2,500 that also existed at modern values (PHP 32,000 to PHP 98,000) under granular real vicinities like MALASIMBO, MALAC, INAMIN, and DEL MONTE. Quezon City alone had 998 suffix-rename collisions across 110 of 140 barangays before the fix; nationwide the v2.16.0 per-barangay RDO metadata exposed 2,001 barangays carrying pre-2010 DO rows. The bug class was the same one progressively fixed by v2.13.4 (whole-vicinity older-DO batches), v2.13.5 (subdivision-as-barangay), and v2.13.6 (annotation classifiers), but with three remaining shapes those rules could not reach: explicit BIR rename annotations like `(fmly: zabarte subd)` or `(FORMERLY HIGHWAY 54)` that document a rename in the new DO but leave the legacy row behind; suffix renames across DO versions where the new DO publishes a street as `<NAME> VILLAGE TOWNHOMES` while the old DO keeps `<NAME> SUBD`; and known district-rollup vicinity labels that the old DOs used as catch-alls (BAGO BANTAY, MURPHY, CUBAO, etc.) while the new DOs publish under granular real vicinities (NORTH AVE, KATIPUNAN, DEL MONTE). This release adds three post-vicinity-prune rules in `dedup_streets.py` that close those three shapes: Rule 3 reads the parenthetical legacy name and drops the same-barangay older-DO row that matches; Rule 1 groups rows by aggressively normalized name (strip SUBD/VILLAGE/HOMES/HEIGHTS/TOWNHOMES/PARK/ESTATES suffixes, drop parentheticals, alphanumeric only) and drops the older row when any classification value differs by 3x or more from the newer row; Rule 2 hard-codes the known stale rollup vicinity allowlist and drops pre-2010 rows under those labels when the barangay also has 2020+ data, gated to Metro Manila region only to prevent generic labels like SAN JOSE or CENTRAL from false-firing in other regions. Live verification post-deploy: Kaligayahan 17 stale rows -> 0, Masambong 29 stale rows -> 0, QC citywide suffix-rename collisions 998 -> 1 (the 1 has missing source-DO metadata so all rules conservatively abort, which is the documented fallback pattern). Nationwide impact: 9,667 stale value rows removed from the live data (248,752 -> 239,085). The new rules emit a per-drop diagnostic CSV at `output/diagnostics/pruned-stale-do-merge.csv` carrying the kept row, dropped row, classification, value ratio, and the legacy name extracted from any annotation; 3,464 rows after this reground, attributed by rule.

### Internal notes
- All three rules abort cleanly when `do_year` is missing on any row in the barangay, mirroring the v2.13.4 conservative fallback. No regression risk on grounded JSON written before the per-street source-DO fields were propagated.
- The v2.16.0 `additional_rdos.street_count` metadata still reflects the pre-dedup row count per source RDO. Recomputing it post-dedup so the per-barangay RDO display matches the actual rendered street list is a follow-up; until then the metadata is slightly stale relative to the page contents.

## [2.16.0] - 2026-05-26

### Added
- Every barangay zonal value page now shows its own source RDO, not the city's. Previously, a Quezon City barangay page read `BIR RDOs 028, 038, 039, 040` regardless of which Department Order actually published the values; a Manila barangay read `BIR RDOs 029, 030, 031, 032, 033, 034`. The v2.13.1 prose relabel stopped the page from implying a single jurisdiction, but the underlying data was still the entire city's RDO set inherited by every barangay. This release puts honest per-barangay data behind that prose. Each barangay now carries its dominant source RDO (the Revenue District Office whose Department Order supplied the most winning streets) plus an optional list of secondary source RDOs that also contributed streets. A new italic subtitle reads "Also drawn from RDO 038 (North Quezon City)" when secondaries exist; single-source barangays show only the primary line. Quezon City's Kaligayahan reads `RDO 028 (Novaliches, Quezon City) — Also drawn from RDO 038 (North Quezon City)` with the DO number resolving to `DO 037-2024` from the dominant source instead of the union of every DO that ever touched the city. Multi-source barangays are roughly 70% of Quezon City, 37% of Makati, 11% of Manila barangays (driven by Department Order vintage, not jurisdiction — recent comprehensive DOs revalue whole cities under one RDO file while older DOs cover individual streets in other RDO files). The 4,612-or-so barangays that previously inherited a comma-joined city RDO now show their actual attribution. About These Values prose extends to mention secondary sources when present: "Some streets in this barangay were last revalued under Department Orders published by RDO 038 (North Quezon City)." A new italic note in the same section spells out the source-vs-jurisdictional distinction so readers don't infer where to file taxes from the displayed RDO: jurisdictional RDO assignments are governed by BIR's Revenue Memorandum Orders (territorial issuances) and may differ from the source RDO whose DO is reflected in these prices.
- Barangay and city pages now display PSGC codes in the header metadata. The barangay header reads `PSGC 1381300046 · City PSGC 1381300000`; the city header gains a single `PSGC 1381300000` line below the existing DO and effectivity row. Codes come from the Philippine Standard Geographic Code reference data already in the zonal schema; no new external dependency.
- Per-barangay DO number now reads from the barangay's source sheet rather than the city's. Previously, `extractDoNumber()` in the barangay page query took `cityData.source_sheet`, which for multi-RDO cities was the union of every Department Order across every contributing RDO (often four or five DOs joined by commas). The barangay's DO now reflects the Department Order that published the barangay's primary values; the source sheet for the dominant RDO becomes the single source.

### Changed
- Database schema: `zonal_barangays` adds four nullable columns. `rdo_code` and `rdo_name` carry the dominant source RDO per barangay. `source_sheet` carries the BIR Excel sheet name for the dominant RDO's Department Order, used to derive the DO number. `additional_rdos` is a JSONB array of `{code, name, source_sheet, street_count}` entries for secondary source RDOs that contributed at least one winning street, sorted by `street_count` descending with `do_year` and code as tiebreakers; `NULL` for single-source barangays. The schema additions also dual-apply to `zonal_staging.zonal_barangays` per the schema-swap contract introduced in v2.15.0, so the next Phase 3 atomic promote does not silently reverse the public-side change.
- Agentic parser: `process_rdo` now stamps `source_rdo_code`, `source_rdo_name`, and `source_sheet` on each street value at parse time. `merge_into_regions` carries these through the cross-RDO city merge unchanged; a new aggregation pass (`_aggregate_barangay_rdo`) runs after merge, counts winning streets per source RDO inside each barangay, writes the dominant attribution to the barangay row, and drops the per-street scaffolding via `strip_internal_fields`. The grounded JSON shape stays clean for downstream consumers (the importer, the page renderer) — the per-street source fields are internal only.
- The import script gains four passthrough fields on the barangay row (`rdo_code`, `rdo_name`, `source_sheet`, `additional_rdos`) and feeds them into both the `--staging` and direct-public paths.

### Internal notes
- This release ships the code path and schema; the visible per-barangay RDO change takes effect after the next zonal data reground and reimport (where `_aggregate_barangay_rdo` populates the new columns). Until then, the four new columns are `NULL` on every existing row and the page falls back to the city's RDO via the existing inheritance logic, so no user-visible regression during the gap.
- The reground and reimport themselves take roughly 20-40 minutes for the parser plus another 10 minutes for the staging import; the schema-swap path introduced in v2.15.0 means the cutover itself is sub-second with no user-visible window.

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
