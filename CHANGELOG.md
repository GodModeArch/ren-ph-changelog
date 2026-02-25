# Changelog

All notable changes to [ren.ph](https://ren.ph) are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
- Blog system: full MDX-powered blog with post listing, TOC sidebar, share buttons, and SEO metadata
- First blog post: "How to Compute Property Value in the Philippines"
- 4-node JSON-LD @graph schema for blog posts: WebPage (reviewedBy, lastReviewed), TechArticle (mentions, dependencies), HowTo (5 steps for rich snippets), Person (root identity with knowsAbout/affiliation)
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
- External nofollow links now correctly get both `nofollow` and `noopener noreferrer` rel attributes
- Markdown tables in report pages rendering as raw pipe-delimited text (missing remark-gfm plugin)
- Narrow tables (2-3 columns) no longer stretch to full width, auto-sized to content instead
- All tables (reports + zonal values) now horizontally scrollable on mobile with sticky first column and fade hint, replacing the card-based mobile layout
- State of RE 2025 report: merged split city name variants in Top 20 ranking (e.g. "City of Calamba" + "Calamba" = 22), clarified total vs. residential unit counts (619,937 total / 329,852 residential), added Data Dictionary, Transformation Notes, and Limitations to Methodology

### Changed
- Updated MASTER-TODO with live database stats: 25,264 broker profiles, 5,046 projects, 2,282 developers, 7,820 LTS records, 1,441 cities, 33,633 barangays, 199,674 zonal value records

## [2026-02-20]

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
