# Data dictionary and limitations

Ten files, verified by `../checksums.sha256`. `METHODS.md` in the repository
root is the authority for every definition; this file is the working reference.

Read the limitations at the bottom before using any of it. They are not
boilerplate — the most natural reading of a pothole map ("this is where the
potholes are") is one the data cannot support.

## `potholes_long.csv`

74,158 rows, one per `(geo_type, geo_id, period, metric)`. That tuple is unique.

| Column | Values |
|---|---|
| `geo_type` | `city`, `community_area`, `ward2023`, `pov_quintile`, `race_group` |
| `geo_id` | Community area number 1–77, ward number 1–50, `CHI` for the city, a stratum label, or `UNASSIGNED` / `EXCLUDED` |
| `geo_name` | Display name — `Rogers Park`, `Ward 1`, `Chicago`, `Unassigned location`, `Excluded tracts` |
| `period` | `CY2019`…`CY2026`, `YTD{YYYY}-08`, pooled `CY2020-2022` / `CY2023-2025` / `CY2019-2025`, or a `CHG_{base}_{recent}` comparison |
| `metric` | One of the 45 below |
| `value` | Empty when suppressed or not reached. Counts are integers; `*_per_*` 2 decimals; `*_d` 1 decimal; `*_share` on a 0–1 scale with 4 decimals; `*_pct` and `*_pts` 1 decimal; `*_chgcls` an integer −2…2 |
| `n` | The count behind the figure. Always populated, **including on suppressed rows**, so you can see how thin a cell was |
| `flags` | Pipe-delimited, alphabetical, or empty. See the registry below |

### Metrics

**Reports** — R is PHF requests that are in-demand and not effective duplicates.

| Metric | Meaning | `n` is |
|---|---|---|
| `rpt_n` | Distinct reported pothole problems | the value |
| `rpt_per_mi` | Reports per street mile, excluding limited-access roads | numerator |
| `rpt_per_1k` | Reports per 1,000 residents (2020 census) | numerator |
| `rpt_all_n` | Reports including effective duplicates | the value |
| `rpt_dup_share` | Share of in-demand PHF that were effective duplicates | `rpt_all_n` |
| `rpt_alder_share` | Share filed through a ward office | `rpt_n` |
| `rpt_digital_share` | Share of non-ward-office reports filed digitally | denominator |
| `rpt_yoy_pct` | Change in `rpt_n` against the prior comparable period, in percent | prior `rpt_n` |
| `crew_n` | Records CDOT entered itself and closed the same day | the value |
| `staff_n` | Staff-observed records | the value |

**Response time** — Kaplan–Meier, so still-open reports are included rather than
dropped, and recent years do not look artificially fast.

| Metric | Meaning |
|---|---|
| `close_p50_d`, `close_p90_d` | Days until 311 closed the report, median and 90th percentile |
| `close_le7_share` | Share closed within 7 days |
| `close_open_share` | Share still open at the observation date |
| `patch_p50_d`, `patch_p90_d` | Days until a patch was recorded, for reports that got one |
| `rpt_patched_share` | Share of completed reports with a linked patch record |
| `close_p50_d__arterial`, `__local`, `__resident`, `__alderman` | The same median on one road class or reporting channel. `close_le7_share` carries the same four suffixes. City and strata only |

**Potholes filled** — `fill_count` is the per-block number crews enter.

| Metric | Meaning |
|---|---|
| `fill_n` | Potholes filled |
| `fill_wo_n` | Work orders |
| `fill_per_mi` | Potholes filled per street mile |
| `fill_crew_share` | Share on work CDOT logged itself |
| `fill_demand_share` | Share traceable to a demand channel |
| `fill_unlinked_share` | Share that could not be linked to a request |
| `fill_extreme_share` | Share on entries of 150+ potholes on one block |
| `fill_per_rpt` | Fills divided by reports. Not a percentage; routinely exceeds 1 |

**Alleys** — `alley_rpt_n`, `alley_close_p50_d`, `alley_close_le7_share`, from
PHB requests. There is no per-mile or per-capita variant: the city publishes no
alley centerline layer.

**Change** — on `CHG_*` periods only, for `rpt_per_mi`, `fill_per_mi` and
`close_le7_share`. `*_chg_pct` / `*_chg_pts` is the raw movement,
`*_vscity_pct` / `*_vscity_pts` is that movement relative to the city's, and
`*_chgcls` is the −2…2 class, which is 0 whenever the difference from the city
trend is within what chance would produce for an area that size.

### Flags

| Flag | Meaning |
|---|---|
| `PARTIAL_YEAR` | The calendar year has not ended. Not comparable to a full year |
| `YTD` | A January 1 – August 31 window, matched across years |
| `TRANSITION_2019` | Includes 2019, the first full year of the current 311 system. Duplicate flagging and entry practice differ from later years, and its records are misallocated at several times the later rate — see the limitation below |
| `PRE2023_WARD` | A ward figure for a period ending before the 2023 remap, shown on today's map |
| `SPLIT_2023` | A ward figure for a period straddling the May 15, 2023 remap |
| `SUPPRESSED_LOW_N` | Too few records for a stable figure. `value` is empty, `n` is not |
| `KM_NOT_REACHED` | The survival curve never reached that quantile |
| `COHORT_OPEN` | 1% or more of the cohort was still open, so patch metrics are withheld until the year settles |
| `EXTREME_FILL` | At least 10% of this area-year's fills came from entries of 150+ potholes on one block |
| `GEO_UNASSIGNED` | The `UNASSIGNED` bucket: records that could not be placed in a geography |
| `CHANGE_UNCLEAR` | The change is within normal variation for an area this size. Read as "close to the city trend", not "no change" |
| `WX_GAP_FILLED`, `WX_MISSING_OBS` | `winter_harshness.csv` only: a temperature reading was filled from Midway, or a precipitation or snow reading was missing and counted as zero. Neither occurs in the winters published so far |

## Companion tables

**`geo_denominators.csv`** (130 rows) — `geo_type`, `geo_id`, `street_mi`,
`pop2020`, `poverty_rate`, `lep_hh_share`, `broadband_hh_share`,
`pct_hispanic`, `pct_nh_black`, `pct_nh_white`, `pct_nh_asian`, `acs_vintage`,
`centerline_rows_updated`. The denominators behind every rate. Street mileage
excludes limited-access roads and splits boundary segments between the areas
that share them, so summing street miles over all areas gives the city total.

**`ward_legacy_mix.csv`** (2,439 rows) — `ward2023`, `period`, `ward2015`,
`share_of_rpt_n`. For each pre-remap period, which 2015-map wards a current ward
is built from, weighted by where reports actually came from. Ward numbers only;
no names anywhere.

**`strata_tracts.csv`** (799 rows) — `geoid`, `tract_pop_city`, `poverty_rate`,
`pov_quintile`, `race_group`, `excluded_reason`. The tract assignments behind
the `pov_quintile` and `race_group` rows. Poverty and race are strongly
collinear in Chicago; they are reported separately and never cross-tabulated.

**`breaks.json`** — the frozen 5-class quantile breaks, by mode, geography and
metric, plus the fixed change thresholds. Frozen until a methods version bump,
so adding a year does not recolor earlier ones.

**`change_summary.csv`** (27 rows) — `geo_type`, `period`, `metric`, `up`,
`down`, `flat`, `suppressed`.

**`qa_report.json`** — every Gate 0–2 check with status and detail. At this
snapshot Gate 1 passes 14 of 14. Six Gate 2 warnings are open, including
geography loss above 1%, drift in the current partial year since the pull,
mapped area-years where extreme fill entries dominate, and W9 (below).

**`winter_harshness.csv`** (8 rows) — `winter`, `cy_period`, `ytd_period`,
`wet_ft_days`, `ft_days`, `fdd`, `prcp_in`, `snow_in`, `z_wet_ft`, `z_fdd`,
`z_prcp`, `z_snow`, `harshness`, `harshness_pctile`, `category`,
`nov_dec_share`, `filled_days`, `missing_days`, `flags`. One row per winter,
`W2019` (Nov 1, 2018 – Apr 30, 2019) onward, built only from NOAA daily weather
records at O'Hare and never from 311 or patch data. Four components — wet
freeze–thaw days, freezing degree-days, precipitation and snowfall — are each
turned into a z-score against the 30 winters 1991–92 to 2020–21; `harshness` is
their plain average, and `category` labels it Mild (below −0.5), Typical,
Harsh (above 0.5) or Severe (above 1.0). `cy_period` and `ytd_period` are the
`potholes_long.csv` periods a winter is read beside. It is context only: no
pothole figure is adjusted for it, and a harsh winter beside a high report
count does not show that one caused the other. METHODS §11.

**`winter_reference_stats.json`** — the reference mean and standard deviation
behind each z-score, the 30 reference winters' own scores, and the reference
winters in which O'Hare's snow readings have gaps (1995–96 to 1998–99 and
2001–02; missing readings count as zero, which makes published snow z-scores
marginally high).

**`winter_sensitivity.csv`** (56 rows) — `winter`, `variant`, `value`, `rank`.
How each winter ranks (1 = harshest) under the published score and six
variants: each component alone, all freeze–thaw days in place of wet ones, and
wet freeze–thaw days counted twice. The 2018–19 winter ranks first under every
variant; the order of the milder winters depends on the variant.

The Midway check (W9 in `qa_report.json`) is open: Midway's record begins in
1997 and has almost no snowfall or snow-depth readings, so its version of the
score rests on three components and tracks O'Hare's at a rank correlation of
0.75 against a target of 0.85. Its cold and precipitation components agree
closely (0.99 and 0.90); the wet freeze–thaw count, which needs snow depth,
does not (0.48).

## Limitations

- **2019 is published, but it is not shown on the website map.** It is the first
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
  appear. Treat 2019 as a weak baseline rather than a comparable year, and say
  so if you publish a comparison against it.
- **These are not counts of potholes.** No dataset here measures how many
  potholes exist or the condition of any street. Reports measure attention;
  fills measure work.
- **More reports per mile does not mean more potholes per mile.** Weather,
  awareness, app use and ward-office practice all move the reporting number.
- **A closed report is not a patched pothole.** Closure is an administrative
  status. Only a linked Potholes Patched record indicates a recorded patch, and
  `rpt_patched_share` is how often that happened.
- **Fill counts are crew-entered and unverified.** One report can lead to
  patching a whole block, so fills routinely exceed reports, and the data says
  nothing about whether a patch held.
- **Some entries list 150+ potholes on one block.** They are kept as recorded.
  `EXTREME_FILL` marks area-years where they make up 10% or more of the total.
- **Why areas differ is not in this data.** There is no pavement condition,
  traffic volume, severity, local weather or crew deployment information here.
- **Nothing here grades an alderperson.** Ward offices file requests; CDOT fills
  potholes. Pre-2023 figures on today's ward map describe areas that different
  alderpersons represented.
- **Group comparisons are ecological.** Strata describe census tracts, not
  people, and say nothing about how any individual was treated.
- **Coverage stops at 311.** Expressways and other limited-access roads are
  excluded from per-mile rates, private drives are not covered, and problems
  reported outside 311 are not in the data at all.
- **Nothing before 2019, and no forecast.**
