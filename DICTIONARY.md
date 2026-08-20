# ASX Hiring Panel — Data Dictionary

**Publisher:** au-data-factory · **Cadence:** daily · **Coverage:** ASX-listed
companies plus notable Australian private employers · **Timezone:** all dates
are UTC calendar days.

## Provenance and method

Postings are collected once per day directly from each employer's own public,
unauthenticated applicant-tracking-system job board (Workday, Greenhouse,
Lever, Ashby, SmartRecruiters, Oracle Cloud Recruiting, Cornerstone, PageUp).
No job-board aggregator is used and no authenticated endpoint is accessed.

Each row is derived from exactly one immutable raw capture, named in
`source_part`, which carries its own manifest (sha256, capture timestamp,
collector git SHA, licence). Daily manifest hashes are chained
(`meta/manifest_chain.ndjsonl`) so the archive is tamper-evident: any
retrospective change to a past day breaks the chain.

**Point-in-time guarantee.** Rows are stamped with the day they were observed
and are never restated. A value never changes after publication.

## Field reference — `hiring_daily.csv` / `.jsonl`

| Field | Type | Meaning |
|---|---|---|
| `as_of` | date (UTC) | Day the board was observed. |
| `ticker` | string | ASX code. Empty for private employers (retained for context; exclude if listed-only coverage is required). |
| `company` | string | Employer name as carried in the entity registry. |
| `ats` | string | Platform the board is hosted on. Affects field availability — see limitations. |
| `postings_open` | int | Open postings on the board that day. |
| `postings_new` | int | Postings present today, absent the previous observed day. Empty on an entity's first day. |
| `postings_removed` | int | Postings present the previous observed day, absent today. Empty on first day. |
| `distinct_locations` | int | Distinct location strings among open postings. |
| `top_location` | string | Most frequent location label. Vendor-supplied, not normalised. |
| `top_category` | string | Most frequent department/function label. Empty where the platform does not publish one. |
| `au_share` | float 0–1 | Share of postings whose location matches an Australian marker. A low value is normal for ASX-listed multinationals and is **not** a data error. |
| `source_part` | string | Path of the raw capture this row derives from. |

## `panel_summary.csv` — read this before any trend analysis

One row per day with the panel top line, and the columns that stop you drawing
a false conclusion from it.

| Field | Meaning |
|---|---|
| `asx_tickers`, `entities` | Panel size that day. |
| `postings_open` | Raw total across the panel. **Not comparable across days.** |
| `postings_new`, `postings_removed` | Sum of per-ticker flows. Each ticker is compared to itself, so these are safe. |
| `entities_entered`, `entities_left` | Panel composition change that day. |
| `lfl_entities` | Entities present on BOTH this day and the previous one. |
| `lfl_postings_open`, `lfl_postings_prev` | Totals over `lfl_entities` only. |
| `lfl_change_pct` | The **only** field to use for a day-over-day trend. |

**Why this matters.** The panel grows as employers are onboarded, so raw totals
mix hiring activity with panel composition. On 2026-08-17 `postings_open` rose
2,513 → 4,840 (+93%) — but 32 employers were added that day, and the
like-for-like change across the 18 employers present on both days was **+1.31%**.
The same happened on 2026-08-20: +11% raw, **+0.5%** like-for-like. Use
`lfl_change_pct` for trends and the raw totals only for "how big is the panel
today".

## `coverage.csv`

Published with the data so gaps are disclosed rather than inferred from
absence: `entities_attempted`, `entities_collected` (200 with ≥1 posting),
`entities_failed` per day. Every collection attempt — success or failure — is
logged in the archive.

## Known limitations (stated, not hidden)

1. **`top_category` is unavailable on Workday, Oracle, PageUp and Cornerstone.**
   Their list endpoints do not expose a department field. Empty means
   unavailable, never zero.
2. **Location strings are vendor-native and unnormalised** ("Sydney CBD Area",
   "Docklands (VIC)", "Sydney, AU-NSW, AU"). Normalisation is deliberately left
   to the consumer so no inference is baked into the archive.
3. **`au_share` measures the workforce, not domicile.** ASX multinationals
   legitimately post mostly offshore.
4. **Flows compare consecutive *observed* days.** If a day is missing from
   `coverage.csv`, the following day's flow spans a longer interval.
5. **SuccessFactors employers are out of scope permanently** — those career
   hosts publish `robots.txt: Disallow: /`, so they are not collected.
6. **Panel composition changes over time.** Backfill does not exist: an
   employer's history starts the day it entered the panel, and
   `entities_entered` records when. Aggregate trends must use the `lfl_*`
   columns.
7. **No ad text, and no personal data.** Only counts and short categorical
   labels are published; contact details are scrubbed before emission and an
   assertion blocks any raw text field from reaching this product.

## Licence

Derived metrics computed by au-data-factory from public employer job boards.
Underlying advertisement text remains the employers' copyright and is **not**
redistributed. This product contains facts (counts, flows, categorical labels).

## Enrichment fields (added 2026-08-20)

| Field | Type | Meaning |
|---|---|---|
| `postings_with_salary` | int | Postings that day carrying an extractable advertised salary (structured fields first, then parsed from ad text into numbers — text itself never leaves the raw layer). |
| `salary_min_median`, `salary_max_median` | float | Medians of the advertised range ends, **annualised** (hourly ×1976, daily ×247) so mixed pay periods are comparable. Empty when no salaries were advertised. |
| `top_seniority` | enum | Most common of executive/head_of/manager/senior/mid/junior among titles that declare one; empty when none do. Deterministic tie-break. |

**Coverage honesty:** salary yield is structural, not uniform — Ashby-hosted boards advertise pay ~34% of the time, Greenhouse ~50%, while Workday/Oracle/SmartRecruiters/Cornerstone/PageUp list feeds carry no pay or ad-text fields at all (verified against every field of real records). A blank is "not advertised via this platform", never zero.
