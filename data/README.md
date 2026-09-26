# Data dictionary and limitations

<!-- gen:release -->Data release **2026-09-26** by Connor Ulrich Blandford, licensed under [CC BY 4.0](../LICENSE). Cite as: Blandford, Connor Ulrich. “Chicago potholes.” Data set, release 2026-09-26. connorblandford.com. https://connorblandford.com/data/chicago-potholes/. Source snapshot: **2026-09-26**, holding records through **2026-09-25**; methods version **0.5.1**; coverage **2019 to 2026-08** in full-year and year-to-date periods, with a partial **CY2026** that runs to the last record.<!-- /gen:release -->

Thirteen files, verified by `../checksums.sha256`. The map’s files are under `../site/`, verified by `../site/checksums.sha256`. `METHODS.md` in the repository
root is the authority for every definition; this file is the working reference.

Read the limitations at the bottom before using any of it. They are not
boilerplate — the most natural reading of a pothole map ("this is where the
potholes are") is one the data cannot support.

## `potholes_long.csv`

<!--n:potholes_long.csv-->77,274<!--/n--> rows, one per `(geo_type, geo_id, period, metric)`. That tuple is unique.

| Column | Values |
|---|---|
| `geo_type` | `city`, `community_area`, `ward2023`, `pov_quintile`, `race_group` |
| `geo_id` | Community area number 1–77, ward number 1–50, `CHI` for the city, a stratum label, or `UNASSIGNED` / `EXCLUDED` |
| `geo_name` | Display name — `Rogers Park`, `Ward 1`, `Chicago`, `Unassigned location`, `Excluded tracts`. The merged race stratum says what it holds: `No majority group (incl. majority-Asian tracts)` |
| `period` | `CY2019`…`CY2026`, <!-- gen:ytd-period -->`YTD{YYYY}-08`<!-- /gen:ytd-period -->, pooled `CY2020-2022` / `CY2023-2025` / `CY2020-2025`, or a `CHG_{base}_{recent}` comparison. No pooled period or comparison includes 2019 |
| `metric` | One of the 46 below |
| `value` | Empty when suppressed, withheld, blank by rule, or not reached. Counts are integers; `*_per_*` 2 decimals; `*_d` 1 decimal; `*_share` on a 0–1 scale with 4 decimals; `*_pct` and `*_pts` 1 decimal; `*_chgcls` an integer −2…2. The ending of the name decides: `rpt_per_mi_chg_pct` is a percentage (1 decimal), `close_le7_share_vscity_pts` percentage points (1 decimal) |
| `n` | The count behind the figure. Always populated, **including on suppressed rows**, so you can see how thin a cell was |
| `flags` | Pipe-delimited, alphabetical, or empty. A row carries the flags of every period it draws on, so a comparison inherits its earlier window’s flags. See the registry below |

### Metrics

**Reports** — R is PHF requests that are in-demand and not effective duplicates.

| Metric | Meaning | `n` is |
|---|---|---|
| `rpt_n` | Distinct reported pothole problems | the value |
| `rpt_per_mi` | Reports per street mile, excluding expressways, the Skyway and ramps (Lake Shore Drive is included) | numerator |
| `rpt_per_1k` | Reports per 1,000 residents (2020 census) | numerator |
| `rpt_all_n` | Reports including effective duplicates | the value |
| `rpt_dup_share` | Share of in-demand PHF that were effective duplicates | `rpt_all_n` |
| `rpt_alder_share` | Share filed through a ward office | `rpt_n` |
| `rpt_digital_share` | Share of non-ward-office reports filed digitally | denominator |
| `rpt_yoy_pct` | Change in `rpt_n` against the prior comparable period, in percent. Blank for the partial year (`CY2026`) and for 2020, whose prior year is 2019 | prior `rpt_n` |
| `crew_n` | Records CDOT entered itself and closed the same day | the value |
| `crew_per_mi` | Those records per street mile, on the same roads as `rpt_per_mi` | numerator |
| `staff_n` | Staff-observed records | the value |

**Response time** — Kaplan–Meier, so still-open reports are included rather than
dropped, and recent years do not look artificially fast.

| Metric | Meaning |
|---|---|
| `close_p50_d`, `close_p90_d` | Days until 311 closed the report, median and 90th percentile |
| `close_le7_share` | Share closed within 7 days |
| `close_open_share` | Share still open at the observation date |
| `patch_p50_d`, `patch_p90_d` | Days until a patch was recorded, for reports that got one. The patch date equals the 311 closing date (within a minute for 99.95% of linked pairs), so this is closing time on the reports that have a patch record, not a separate clock |
| `rpt_patched_share` | Share of completed reports with a linked patch record |
| `close_p50_d__arterial`, `__local`, `__resident`, `__alderman` | The same median on one road class or reporting channel. `close_le7_share` carries the same four suffixes. City and strata only |

**Potholes filled** — `fill_count` is the per-block number crews enter.

| Metric | Meaning |
|---|---|
| `fill_n` | Potholes filled |
| `fill_wo_n` | Work orders |
| `fill_per_mi` | Potholes filled per street mile, on the same roads as `rpt_per_mi` |
| `fill_crew_share` | Share on work CDOT logged itself |
| `fill_demand_share` | Share traceable to a demand channel |
| `fill_unlinked_share` | Share that could not be linked to a request |
| `fill_extreme_share` | Share on entries of 150+ potholes on one block |
| `fill_per_rpt` | Fills divided by reports. Not a percentage; routinely exceeds 1 |

**Alleys** — `alley_rpt_n`, `alley_close_p50_d`, `alley_close_le7_share`, from
PHB requests. There is no per-mile or per-capita variant: the City publishes no
alley centerline layer.

**Change** — on `CHG_*` periods only, for `rpt_per_mi`, `fill_per_mi` and
`close_le7_share`. `*_chg_pct` / `*_chg_pts` is the raw movement,
`*_vscity_pct` / `*_vscity_pts` is that movement relative to the city’s, and
`*_chgcls` is the −2…2 class. A class is non-zero only when the difference
from the city trend is larger than areas of that size actually vary — each
layer’s dispersion factor, published in `change_summary.csv`, scales the
chance-only error up to what is observed — and survives a 5% false discovery
rate across every area tested on that layer. METHODS §6.6.

### Flags

| Flag | Meaning |
|---|---|
| `PARTIAL_YEAR` | The calendar year has not ended. Not comparable to a full year |
| `YTD` | A <!-- gen:ytd-window -->Jan 1–Aug 31<!-- /gen:ytd-window --> window, matched across years |
| `TRANSITION_2019` | Includes 2019, or compares against it (a blank 2020 `rpt_yoy_pct`). 2019 is the first full year of the current 311 system; duplicate flagging and entry practice differ from later years, and its records are misallocated at several times the later rate — see the limitation below |
| `PRE2023_WARD` | A ward figure for a period ending before the 2023 remap, shown on today’s map, or a comparison that uses one |
| `SPLIT_2023` | A ward figure for a period straddling the 2023-05-15 remap, or a comparison that uses one |
| `SUPPRESSED_LOW_N` | Too few records for a stable figure (fewer than 30; 100 for a 90th percentile). `value` is empty, `n` is not |
| `KM_NOT_REACHED` | The survival curve never reached that quantile |
| `COHORT_OPEN` | 1% or more of the cohort was still open, so patch metrics are withheld until the year settles |
| `EXTREME_FILL` | At least 10% of this area-year’s fills came from entries of 150+ potholes on one block |
| `GEO_UNASSIGNED` | The `UNASSIGNED` bucket: records that could not be placed in a geography |
| `CHANGE_UNCLEAR` | The change is within how much areas of this size usually vary, or does not survive the false-discovery control. Read as "close to the city trend", not "no change" |
| `WX_GAP_FILLED`, `WX_MISSING_OBS` | `winter_harshness.csv` only: a temperature reading was filled from Midway, or a precipitation or snow reading was missing and counted as zero. Neither occurs in the winters published so far |

## Companion tables

**`geo_denominators.csv`** (<!--n:geo_denominators.csv-->143<!--/n--> rows) — the denominators behind every
rate, for the city, community areas, wards, and each poverty quintile and race
group.

| Column | Values |
|---|---|
| `street_mi` | Street miles, at most 4 decimals. Excludes expressways, the Skyway and ramps; includes Lake Shore Drive. A segment on a shared boundary counts once, for the lower `geo_id` |
| `pop2020` | Residents, 2020 Census |
| `poverty_rate`, `lep_hh_share`, `broadband_hh_share` | **Shares on a 0–1 scale**, at most 6 decimals — not percentages. Poverty rate, limited-English households, households with broadband (ACS 5-year) |
| `pct_hispanic`, `pct_nh_black`, `pct_nh_white`, `pct_nh_asian` | Also **0–1 shares** despite the `pct_` prefix, at most 6 decimals. 2020 Census, non-Hispanic except `pct_hispanic` |
| `acs_vintage` | The ACS 5-year vintage the covariates come from, as a year |
| `centerline_rows_updated` | When the City last updated the street centerline dataset, ISO-8601 UTC |

The covariate columns are empty on the `UNASSIGNED` and `EXCLUDED` rows.

**Do not add the `UNASSIGNED` row into a total.** It is not a place: its
`street_mi` is city mileage that falls inside no polygon of that `geo_type`,
and its `pop2020` is the 2020 population of Cook County *outside* the city —
2.53 million people, which is why the row must never be read as a denominator.
A geography’s street-mile total is the sum of its real areas, `EXCLUDED`
included for the strata, with `UNASSIGNED` dropped: that comes to 3,950.4 mi
for community areas and for both stratum types, against the city’s 3,950.4391.
Wards come to 3,938.9 mi, 0.29% low, because the ward boundaries do not tile
the community-area union exactly; check V9 allows 0.5%.

**`ward_legacy_mix.csv`** (<!--n:ward_legacy_mix.csv-->2,437<!--/n--> rows) — `ward2023`, `period`, `ward2015`,
`share_of_rpt_n`. For each pre-remap period, which 2015-map wards a current ward
is built from, weighted by where reports actually came from. Ward numbers only;
no names anywhere.

**`strata_tracts.csv`** (<!--n:strata_tracts.csv-->799<!--/n--> rows) — `geoid`, `tract_pop_city`, `poverty_rate`,
`pov_quintile`, `race_group`, `race_majority`, `excluded_reason`. The tract
assignments behind the `pov_quintile` and `race_group` rows. `race_majority` is
a tract’s majority group before any merge; `race_group` is the group it is
counted in. `poverty_rate` here is the whole-tract ACS rate as a 0–1 share, at
full floating-point precision — the fixed per-measure rounding described above
applies to `potholes_long.csv` alone, so round it yourself before displaying
it. A group with fewer than 300 reports a year is merged into `no_majority`:
at this release that is the six majority-Asian (non-Hispanic) tracts, about
20,900 residents. Poverty and race are strongly collinear in Chicago; they
are reported separately and never cross-tabulated.

**`breaks.json`** — the frozen 5-class quantile breaks, by mode, geography and
metric, plus the fixed change thresholds. Full years are cut from 2020–2025;
each of the twelve year-to-date months is cut from its own January-to-month
windows in 2020–2025. A set too thin to cut (January alley closing times) is
listed under `unavailable`. Frozen until a methods version bump, so adding a
year does not recolor earlier ones.

**`change_summary.csv`** (<!--n:change_summary.csv-->27<!--/n--> rows) — `geo_type`, `period`, `metric`, `up`,
`down`, `flat`, `suppressed`, `dispersion`. `dispersion` is the layer’s
dispersion factor: how many times more areas vary than counting chance alone
would predict (empty for the city, which is not tested).

**`qa_report.json`** — every Gate 0 diagnostic with its result, threshold and
decision, and every Gate 1 and Gate 2 check with status and detail. `summary`
counts the checks the methods define, not their sub-checks. A failure that is
published rather than fixed carries `accepted` with its reason.

<!-- gen:validation -->Of the 15 build-blocking checks the methods define (V1–V15), 13 pass and 2 fail and are published rather than loosened. **V3** caps geography loss at 1.5% of each count in each year; 6 count-years exceed it: crew-logged records in 2021, 2022 and 2026, reports in 2019, staff-observed records in 2019 and reports including duplicates in 2019 (worst: crew-logged records in 2021, 2.16% on wards). **V7b** asks the survival-curve median to sit within 0.5 day of the median of closed reports alone where under 0.5% of reports are open; it misses in 12 of 2,398 cohorts, where counting the one or two open reports moves the median one report along, across a gap of more than half a day. An exact check (**V7c**) confirms the estimator in all 2,398 cohorts, those 12 included. 5 warning checks are open: V3w, geography loss between 1% and the 1.5% cap; W3, 56 area-years where crew entries of 150+ potholes make up 10% or more of fills; W4, 125 area-years above three times their own median, mostly alley closing times (57), street closing times (33) and patch times (25); W6, a race group too thin to stand alone, merged into “no majority” (majority-Asian tracts); W9, the Midway cross-check of the O’Hare winter score (ρ = 0.75 against a 0.85 target). 2 diagnostics miss their targets: D6, where 93.69% of reports lie within 100 ft of a street centerline against a 97% target (the rest count as unsnapped); D9, where the 311 ward field disagrees with the coordinates for 4.18% of pre-remap reports against a 2% trigger (placement never uses that field; the cause is not yet investigated). The winter score’s rank correlation with citywide reports is ρ = 0.2 over 2020–2025 (n = 6) and 0.5 with 2019 (n = 7); neither is distinguishable from zero (exact p = 0.71 and 0.27). Every result is in `qa_report.json`.<!-- /gen:validation -->

**`winter_harshness.csv`** (<!--n:winter_harshness.csv-->8<!--/n--> rows) — `winter`, `cy_period`, `ytd_period`,
`wet_ft_days`, `ft_days`, `fdd`, `prcp_in`, `snow_in`, `z_wet_ft`, `z_fdd`,
`z_prcp`, `z_snow`, `harshness`, `harshness_pctile`, `category`,
`nov_dec_share`, `filled_days`, `missing_days`, `flags`. One row per winter,
`W2019` (2018-11-01 – 2019-04-30) onward, built only from NOAA daily weather
records at O’Hare and never from 311 or patch data. Four components — wet
freeze–thaw days, freezing degree-days, precipitation and snowfall — are each
turned into a z-score against the 30 winters 1991–92 to 2020–21; `harshness` is
their plain average, and `category` labels it Mild (below −0.5), Typical,
Harsh (above 0.5) or Severe (above 1.0). `cy_period` and `ytd_period` are the
`potholes_long.csv` periods a winter is read beside. It is context only: no
pothole figure is adjusted for it, its components stand for conditions the
pavement literature associates with pothole formation rather than measured
damage, and a harsh winter beside a high report count does not show that one
produced the other. Its rank correlation with citywide reports is D10 in
`qa_report.json`, summarized under `qa_report.json` above; with six or seven
years it cannot establish an association either way. METHODS §11.

**`winter_reference_stats.json`** — the reference mean and standard deviation
behind each z-score, the 30 reference winters’ own scores, and the reference
winters in which O’Hare’s snow readings have gaps (1995–96 to 1998–99 and
2001–02; missing readings count as zero, which makes published snow z-scores
marginally high).

**`winter_sensitivity.csv`** (<!--n:winter_sensitivity.csv-->56<!--/n--> rows) — `winter`, `variant`, `value`, `rank`.
How each winter ranks (1 = harshest) under the published score and six
variants: each component alone, all freeze–thaw days in place of wet ones, and
wet freeze–thaw days counted twice. The 2018–19 winter ranks first under every
variant; the order of the milder winters depends on the variant.

**`community_area_summary.csv`** (<!--n:community_area_summary.csv-->77<!--/n--> rows) and
**`ward_summary.csv`** (<!--n:ward_summary.csv-->50<!--/n--> rows) reshape `potholes_long.csv`
for the latest complete calendar year, `UNASSIGNED` excluded: `geo_id`, `geo_name`,
`period`, then `rpt_n`, `rpt_per_mi`, `close_p50_d`, `rpt_patched_share` and
`fill_per_mi`, each followed by its `n` and `flags` columns. The value text is exactly
what the long table prints for that row; a withheld cell is empty with its `n` and flag
intact. Nothing is recomputed, so read the flags here as you would there.

**`facts.json`** — every figure the project page states, one entry per figure with
`value` (the exact figure), `display` (the string the page prints, thousands separators
and unit included), `label`, a one-sentence `definition`, `source_file`, `sources`,
`rounding` and `unit`, read from the files above and the deterministic parts of
`qa_report.json`. The top level carries the release date, `snapshot_date`,
`data_currency`, `methods_version` and the coverage window. Percents are on a 0–100
scale; nothing carries a metric unit. METHODS §8.2.

The Midway check (W9 in `qa_report.json`) is open: Midway’s record begins in
1997 and has almost no snowfall or snow-depth readings, so its version of the
score rests on three components and tracks O’Hare’s at a rank correlation of
0.75 against a target of 0.85. Its cold and precipitation components agree
closely (0.99 and 0.90); the wet freeze–thaw count, which needs snow depth,
does not (0.48).

## Limitations

- **2019 is published, but it enters no comparison.** It is the first
  full year of the current 311 system and its records are misallocated at
  several times the rate of any later year:

  | Full year | Reports not placed in a community area | Duplicate-flagged share |
  |---|---:|---:|
  | 2019 | **2.09%** (891 of 42,535) | **46.8%** |
  | 2020 | 0.73% | 21.6% |
  | 2021 | 0.70% | 22.0% |
  | 2022 | 0.60% | 32.9% |
  | 2023 | 0.37% | 21.7% |
  | 2024 | 0.78% | 30.7% |
  | 2025 | 0.65% | 40.2% |

  The rows stay in this release — excluding data from a chart is not a reason to
  withhold it from the record — and carry `TRANSITION_2019` wherever they
  appear. Nothing else uses them: no pooled period, year-over-year change,
  change comparison, color class or diagnostic headline includes 2019, and the
  website’s map and charts start at 2020. Treat 2019 as a weak baseline rather
  than a comparable year, and say so if you publish a comparison against it.
- **These are not counts of potholes.** No dataset here measures how many
  potholes exist or the condition of any street. Reports measure attention;
  fills measure work.
- **More reports per mile does not mean more potholes per mile.** Weather,
  awareness, app use and ward-office practice all move the reporting number.
- **A closed report is not a patched pothole.** Closure is an administrative
  status. Only a linked Potholes Patched record indicates a recorded patch, and
  `rpt_patched_share` is how often that happened.
- **Time to patch is time to close on the patched subset.** The patch
  completion and the 311 closure are recorded within a minute of each other
  for 99.95% of linked pairs, so `patch_*` does not time a separate step.
- **Fill counts are crew-entered and unverified.** One report can lead to
  patching a whole block, so fills routinely exceed reports, and the data says
  nothing about whether a patch held.
- **Some entries list 150+ potholes on one block.** They are kept as recorded.
  `EXTREME_FILL` marks area-years where they make up 10% or more of the total.
- **Why areas differ is not in this data.** There is no pavement condition,
  traffic volume, severity, local weather or crew deployment information here.
- **Nothing here grades an alderperson.** Ward offices file requests; CDOT fills
  potholes. Pre-2023 figures on today’s ward map describe areas that different
  alderpersons represented.
- **Group comparisons are ecological.** Strata describe census tracts, not
  people, and say nothing about how any individual was treated. Majority-Asian
  tracts are counted under "no majority".
- **Coverage stops at 311.** Expressways, the Skyway and ramps are excluded
  from per-mile rates (Lake Shore Drive is included), private drives are not
  covered, and problems reported outside 311 are not in the data at all.
- **Few changes stand out once areas’ real variability is allowed for.** On
  most change layers no area differs from the city trend by more than areas
  usually vary; `CHANGE_UNCLEAR` rows are "close to the city trend", not
  evidence of no change.
- **Nothing before 2019, and no forecast.**
