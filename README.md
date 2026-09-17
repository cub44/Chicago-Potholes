# Chicago potholes

[Explore the project](https://connorblandford.com/projects/chicago-potholes/). This is the dataset behind an exploratory map; the accompanying article is forthcoming. It covers what Chicago's own records show about pothole **reports**, **response time** and **potholes filled**, by community area and ward. Source snapshot: **16 September 2026**; methods version **0.3**; coverage **2019 through August 2026**.

No software is required. Open the CSVs in a spreadsheet or your preferred analysis tool.

## Download

| File | Rows | One row represents |
|---|---:|---|
| [potholes_long.csv](data/processed/potholes_long.csv) | 74,158 | One measure, for one place, for one period |
| [geo_denominators.csv](data/processed/geo_denominators.csv) | 130 | One place, with the street mileage and population its rates are divided by |
| [ward_legacy_mix.csv](data/processed/ward_legacy_mix.csv) | 2,439 | One current ward's overlap with one pre-2023 ward, in one period |
| [strata_tracts.csv](data/processed/strata_tracts.csv) | 799 | One census tract, with its poverty quintile and majority race/ethnicity group |
| [change_summary.csv](data/processed/change_summary.csv) | 27 | One comparison, counting places that rose, fell, held flat, or were withheld |
| [breaks.json](data/processed/breaks.json) | — | The frozen colour-class boundaries the map uses |
| [qa_report.json](data/processed/qa_report.json) | — | Every validation check, with its result |

Join `potholes_long.csv` to `geo_denominators.csv` on `geo_type` + `geo_id`, both read as text — community areas are `"1"`–`"77"`, wards `"1"`–`"50"`, the city `"CHI"`, so a numeric read will collide them. Join `ward_legacy_mix.csv` on `ward2023` + `period`. City rows are totals, not another area: never add them to the community-area or ward rows. [SHA-256 checksums](checksums.sha256) identify the download versions; verify with `shasum -a 256 -c checksums.sha256`.

## How this data is structured, and why

The shape is meant to make the file hard to misread rather than quick to skim. Four decisions do most of that work.

**One long table, not a spreadsheet per topic.** Every figure in `potholes_long.csv` sits on its own row keyed by `(geo_type, geo_id, period, metric)`. Filter to the metric and period you want and you have a tidy table of places; no column means a different thing in a different year, and adding a year or a measure adds rows rather than reshaping anything you have already written against. The cost is that you filter before you chart. That is deliberate — a wide file invites people to compare two columns that were never comparable.

**The count behind every number travels with it.** `n` is published on every row, *including rows where the value is blank*. A rate of 12 per street mile means something different behind 40 reports than behind 900, and a figure withheld for thinness should still tell you how thin it was. Nothing here asks you to take a number on faith about its own weight.

**Caveats are data, not a footnote.** `flags` is a pipe-delimited column on every row, and the conditions that make a figure hard to read — a partial year, a ward period before the 2023 remap, a cohort still open, a crew entry of 150+ potholes on one block — are attached to the exact rows they apply to. A caveat written in prose gets separated from the number the first time someone copies a cell. A caveat in a column does not. **Never read a value without reading its flags.**

**Thin cells are blank, and blank on a published rule.** A figure is withheld when roughly 30 or fewer records sit behind it, and the test uses the *expected* count for a place of that size, not the observed one. That distinction matters: if suppression keyed off the observed count, genuinely quiet areas would vanish from the map and the map would systematically overstate how much of the city has a pothole problem. Under an expected-count rule, a quiet area shows up as quiet.

Two smaller things follow from the same intent. Colour-class boundaries in `breaks.json` are **frozen** at the methods version, so adding a year does not silently recolour earlier years. And values arrive **pre-rounded** to a fixed precision per measure, so the file and the map cannot disagree about what a number is.

## Sources and method

Inputs are the City of Chicago datasets `v6vf-nfxy` (311 service requests, pothole codes PHF for streets and PHB for alleys), `wqdh-9gek` (Potholes Patched), `p293-wvbd` (wards, 2023–), `k9yb-bpqx` (wards 2015–2023, used for the overlay and for quality checks only), `igwz-8jzy` (community areas) and `pr57-gg9e` (street centrelines), plus the 2020 Census P.L. 94-171 tables, the ACS 5-year tract tables and TIGER/Line 2020 geometry from the Census API. Raw snapshots are preserved privately; every published figure is reproducible from the dataset IDs and parameters recorded in [METHODS.md](METHODS.md) §1.

Patched records are linked back to the 311 request that prompted them, so "how long until a patch" can be measured for the reports that got one. Records are placed by point-in-polygon on their own coordinates, **not** by the ward or community-area field 311 stored when the request was filed — those reflect assignment at creation time and disagree with the boundaries on about 4% of pre-remap rows. Per-mile rates use centreline mileage excluding expressways and other limited-access roads, with boundary segments split between the places that share them, so street miles sum to the city total.

Response times are estimated with a Kaplan–Meier survival curve rather than by averaging closed reports. Reports still open on the snapshot date are counted as open rather than dropped, which is what stops a recent year looking artificially fast because its slow cases have not finished yet. **This is also why there is no mean days-to-close anywhere in the release:** a mean cannot be computed from censored data without inventing closing dates for reports that have not closed. The published figures are medians and 90th percentiles from the survival curve.

[METHODS.md](METHODS.md) defines every number — filters, linkage, geography, the estimator, suppression, class breaks and the validation gates. If the code and that document disagree, the code is wrong.

## Before reporting

- **These are not counts of potholes.** Nothing in this release measures how many potholes exist or the condition of any street. Reports measure attention; fills measure work. Neither measures pavement.
- **More reports per street mile is not more potholes per street mile.** Weather, awareness, app use and ward-office practice all move the reporting number. A quiet area may be a well-paved one or one that does not call.
- **A closed report is not a patched pothole.** Closure is an administrative status. Only a linked Potholes Patched record indicates a recorded patch; `rpt_patched_share` is how often that happened — 71.2% of completed 2025 reports, 76.7% in 2024.
- **2019 is published here but is not shown on the website map, because its records are misallocated at several times the later rate.** 2.09% of 2019 reports could not be placed in a community area at all (891 of 42,535), against 0.37%–0.78% in every later full year, and 46.8% of its in-demand requests carry a duplicate flag, the highest share in the series. It was the first full year of the current 311 system and its entry practices differ from later years. The rows are kept because excluding data is not the same as hiding it, and they carry the `TRANSITION_2019` flag wherever they appear. Treat 2019 as a weak baseline, not as a comparable year.
- **2026 is not over.** Use the `YTD{YYYY}-08` periods to compare it with earlier years; they cover January 1 – August 31 in every year and observe response times as of the same date each year. A `CY2026` row is a partial year and is flagged `PARTIAL_YEAR`.
- **Fill counts are crew-entered and unverified.** One report can lead to patching a whole block, so fills routinely exceed reports — about 249,000 fills against 20,911 reports in 2025. Some entries record 150 or more potholes on one block; they are kept as recorded, and `EXTREME_FILL` marks the place-years where such entries are 10% or more of the total.
- **Patch timings are withheld for years that have not settled.** When 1% or more of a place's cohort is still open, `patch_p50_d`, `patch_p90_d` and `rpt_patched_share` are blank with the flag `COHORT_OPEN`, rather than published from the subset that happens to have finished.
- **Nothing here grades an alderperson.** Ward offices file requests; CDOT fills potholes. Ward figures before 15 May 2023 are shown on today's ward map so a place can be followed over time, and describe an area that different alderpersons represented. `ward_legacy_mix.csv` records the overlap, by number. No alderperson name appears anywhere in this release.
- **Alley figures are counts and response times only.** The city publishes no alley centreline mileage, so there is no alley rate per mile and no per-capita variant. Alley reports are also sparse enough per place that roughly a fifth of community areas are withheld in a full year.
- **Group comparisons are ecological.** `pov_quintile` and `race_group` rows describe census tracts, not people, and say nothing about how any individual was treated. Poverty and race are strongly collinear in Chicago; they are reported separately and are never cross-tabulated.
- **Coverage stops at 311.** Expressways and other limited-access roads are excluded from per-mile rates, private drives are not covered, and a pothole nobody reported is not in the data.
- **`UNASSIGNED` is a bucket, not a place.** Records whose coordinates fall outside every boundary are totalled under that id for count measures only. It is never mapped, and it should not be charted beside real places.
- **Portal datasets get revised retroactively.** The build reads a dated snapshot, so these figures stay reproducible after an upstream revision — and will differ from a query run against the live portal today.

Five validation warnings are open at this snapshot, including geography loss above 1% on some measures and drift in the current partial year since the pull. All of them are written out in [qa_report.json](data/processed/qa_report.json); the fourteen build-blocking checks all pass.

## Reuse and corrections

Data and prose are CC BY 4.0; code is MIT. See [LICENSE](LICENSE). The underlying City of Chicago and Census records remain subject to their publishers' terms.

Suggested attribution: "Chicago potholes, Connor Ulrich Blandford, source snapshot 16 September 2026," with a link to this repository. Cite the snapshot date and the file you used. Report corrections through [Issues](https://github.com/cub44/POTHOLES_REPO/issues), including the filename and the `geo_type`, `geo_id`, `period` and `metric` of the disputed row.
