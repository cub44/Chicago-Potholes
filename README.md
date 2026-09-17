# Chicago Potholes — data release

Published figures on Chicago pothole **reports**, **response time**, and
**potholes filled**, by community area and ward, 2019 through the current year.
This repository is the release of record: the tables here are what the map on
[connorblandford.com](https://connorblandford.com/projects/chicago-potholes/)
reads, and `checksums.sha256` is the contract that says so.

Source snapshot 2026-09-16 · methods version 0.3 · MIT licensed.

## What these numbers are

Three questions, restated to match what the data can actually observe:

| The question | What is published | Source |
|---|---|---|
| Is the number of potholes growing? | **Pothole reports** — distinct pothole problems residents and ward offices report to 311 each year, with repeat reports removed | 311 service requests, code PHF |
| How fast are potholes filled? | **Response time** — time until 311 closes a report, and separately, time until a patch is recorded for reports that got one | 311 PHF, linked to Potholes Patched |
| How many potholes are filled? | **Potholes filled** — the per-block count crews enter on completed work | Potholes Patched |

**No source in this release measures how many potholes exist, or the condition
of any street.** Reports measure attention. Fills measure work. Neither measures
pavement. [`METHODS.md`](METHODS.md) §12 lists what the data cannot support, and
that list is not decorative — most obvious readings of a pothole map are on it.

## Citywide, at this snapshot

Reports are counted by the date they were filed; fills by the date crews closed
the work. Year-to-date columns compare January 1 – August 31 in every year.

| | 2019 | 2024 | 2025 | 2026 (Jan–Aug) |
|---|---|---|---|---|
| Reports | 42,535 | 27,828 | 20,911 | 25,780 |
| Reports per street mile | 10.72 | 6.97 | 5.23 | 6.46 |
| Median days to close | 25.0 | 8.8 | 14.9 | 13.0 |
| Closed within 7 days | 25.4% | 45.2% | 31.6% | 35.7% |
| Potholes filled | 434,689 | 319,240 | 248,541 | 255,637 |

Reports through August 2026 are running about 50% above the same months of 2025
(25,780 against 17,169). 2019 carries the `TRANSITION_2019` flag — it was the
first full year of the current 311 system, and its duplicate flagging and entry
practices differ from later years.

## Files

Everything is in [`data/processed/`](data/processed), with a column dictionary
and the limitations that travel with each table in
[`data/README.md`](data/README.md).

| File | Rows | What it holds |
|---|---|---|
| `potholes_long.csv` | 74,158 | The primary table. One row per `(geo_type, geo_id, period, metric)`, with `value`, `n` and `flags`. 45 metrics, 22 periods. |
| `geo_denominators.csv` | | Street miles, 2020 population and ACS covariates per geography. |
| `ward_legacy_mix.csv` | | For periods before the 2023 remap, which 2015-map wards sit behind each current ward, and in what share. Numbers only. |
| `strata_tracts.csv` | | Census tracts with their poverty quintile and majority race/ethnicity grouping. |
| `breaks.json` | | The frozen 5-class quantile breaks the map colors by. |
| `change_summary.csv` | | Per comparison, how many geographies moved up, down, flat, or were suppressed. |
| `qa_report.json` | | Every validation check with its result. Gate 1 passes 14 of 14 at this snapshot; 5 Gate 2 warnings are open and described there. |

Verify a download against the release:

```bash
shasum -a 256 -c checksums.sha256
```

## Reading the table

`potholes_long.csv` is long, not wide. To pull one metric for one geography:

```bash
awk -F, '$1=="community_area" && $5=="rpt_per_mi"' data/processed/potholes_long.csv
```

- `value` is **empty** when a figure is suppressed or a survival quantile was
  never reached. `n` is still published, so you can see how thin the cell was.
- `flags` is pipe-delimited. Never read a value without reading its flags —
  `PRE2023_WARD`, `SPLIT_2023`, `PARTIAL_YEAR`, `COHORT_OPEN` and
  `SUPPRESSED_LOW_N` each change what the number means. METHODS §8.3 has the
  full registry with the caveat text for each.
- Figures are suppressed below roughly 30 underlying records (METHODS §7.1).
  Suppression is based on the expected count for an area that size, not the
  observed one, so genuinely quiet areas are not hidden.

## Wards and alderpersons

Ward boundaries changed on May 15, 2023. Earlier years are shown on today's ward
map so areas can be compared over time; `ward_legacy_mix.csv` records which old
wards a current ward is made of. **Nothing here is a measure of an
alderperson.** Ward offices file requests; CDOT fills potholes. A pre-2023
figure on a current ward describes an area that different alderpersons
represented, and no alderperson name appears anywhere in this release.

## How it was built

[`METHODS.md`](METHODS.md) defines every number in this release — sources,
record preparation, the linkage between patches and requests, geography
assignment, the Kaplan–Meier estimator used for response times, suppression,
class breaks, and the validation gates. If the code and that document disagree,
the code is wrong.

The pipeline itself, the raw snapshots, the source profiling and the working
notes are in a separate private repository. Everything published here is
reproducible from the dataset IDs and parameters recorded in METHODS §1.
