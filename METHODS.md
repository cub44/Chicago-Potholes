# Chicago Potholes — Methods

**Version:** 0.3 (draft) · **Drafted:** 2026-09-16 · **Applies to:** snapshot pulls on or after 2026-09-16

This document defines every published number. If code and this document disagree, the code is wrong. Changing a definition, filter, threshold, or class break requires a version bump and a changelog entry (§13).

---

## 0. What the site measures

The project's original questions are restated to match what the data can observe.

| Original | Published as | Primary source |
|---|---|---|
| A. Is the number of potholes growing? | **Pothole reports:** how many distinct pothole problems residents and ward offices report to 311 each year | 311 PHF |
| B. How fast are potholes filled? | **Response time:** (1) time until 311 closes a report; (2) time until a patch is recorded, for reports that got one | 311 PHF; Potholes Patched linked to 311 |
| C. How many potholes are filled? | **Potholes filled:** the per-block count crews enter on completed work | Potholes Patched |

No dataset measures the stock or condition of potholes. Nothing on the site should be worded as if one does.

---

## 1. Sources

| Role | Dataset | ID | Fields used |
|---|---|---|---|
| Requests | 311 Service Requests | `v6vf-nfxy` | `sr_number`, `sr_short_code`, `status`, `origin`, `created_department`, `created_date`, `closed_date`, `duplicate`, `legacy_record`, `parent_sr_number`, `street_address`, `latitude`, `longitude`; `ward` (QA only) |
| Completed work | Potholes Patched | `wqdh-9gek` | `address`, `request_date`, `completion_date`, `number_of_potholes_filled_on_block`, `latitude`, `longitude` |
| Wards (current) | Boundaries – Wards (2023–) | `p293-wvbd` | `ward`, `the_geom` |
| Wards (prior) | WARDS_2015 (source table of "Boundaries – Wards (2015–2023)") | `k9yb-bpqx` | `ward`, `the_geom` — overlay and QA only |
| Community areas | Boundaries – Community Areas | `igwz-8jzy` | `area_numbe`, `community`, `the_geom` |
| Street centerlines | "transportation" (source table of the `6imu-meau` map view) | `pr57-gg9e` | `class`, `status`, `street_nam`, `the_geom`, `length` (QA) |
| Population, race/ethnicity | 2020 Census P.L. 94-171, blocks, Cook County (state 17, county 031) | Census API `2020/dec/pl` | `P1_001N`, `P2_001N`, `P2_002N`, `P2_005N`, `P2_006N`, `P2_008N` |
| Block/tract geometry | TIGER/Line 2020 | — | block internal points (`INTPTLAT20`, `INTPTLON20`), tract polygons |
| Socioeconomic covariates | ACS 5-year, tract | Census API `acs/acs5` (latest vintage available at build; record in `config/params.json`) | `B17001_001E`, `B17001_002E`; `C16002_001E`, `_004E`, `_007E`, `_010E`, `_013E`; `B28002_001E`, `B28002_004E` |

**Do not use:**
- `43wa-7qmu` ("Wards"). It is an unofficial user-uploaded copy of the 2015–2023 map (Ward 1 `shape_area` is identical to `k9yb-bpqx`), created 2015-04-22 and never updated. Use `k9yb-bpqx`.
- Any `:@computed_region_*` column. The ward-like region on both source datasets is keyed to `43wa-7qmu` (2015–2023 boundaries).
- The 311 `ward` and `community_area` fields for assignment. They reflect assignment at creation time; used only in QA check D9.
- `6imu-meau` directly. It is a map view with no exposed columns; `pr57-gg9e` is its tabular parent. Rows were last updated 2021-06.

**Snapshot rules**
- Every pull writes to `data/raw/{snapshot_id}/` with `snapshot_id` = UTC ISO timestamp of the pull. The build reads only from one snapshot directory and makes no network calls.
- At pull time, run the reconciliation aggregate queries in V1 against the portal and save their results beside the raw files.
- Socrata `calendar_date` values are floating timestamps in Chicago local time. Never convert time zones. All year, month, and day boundaries use local wall-clock time.
- `data_currency` = max(`created_date`) across PHF rows in the snapshot.
- **Parent lookup.** After the main pull, collect every non-null `parent_sr_number` that is not an `sr_number` in the pull. Query `v6vf-nfxy` by `sr_number` in batches of about 300, with no code, date, or legacy filter, and repeat for newly found parents until none remain or 10 rounds have run. Store `sr_number`, `sr_short_code`, `created_date`, `status`, `legacy_record`, and `parent_sr_number` as `parents.parquet` in the snapshot. Parents that the portal does not return are recorded as not found.

---

## 2. Record preparation

### 2.1 311 requests (PHF streets, PHB alleys)

**Row filter:** `sr_short_code IN ('PHF','PHB')` AND `created_date >= '2019-01-01T00:00:00'` AND `legacy_record = false`.

**Derived fields**

| Field | Rule |
|---|---|
| `status_class` | `closed` if `status='Completed'` and `closed_date` not null; `open` if `status='Open'`; `canceled` if `status='Canceled'`; `bad_close` if `status='Completed'` and `closed_date` null (dropped, counted in QA) |
| `channel` | §2.2 |
| `in_demand` | `channel IN ('resident_digital','resident_phone','resident_other','alderman')` AND `status_class IN ('closed','open')` |
| `root_sr` | §2.1.1 |
| `dup_eff` | §2.1.1 |
| `dur_days` | closed: `(closed_date − created_date).total_seconds() / 86400`; open: `(data_currency − created_date).total_seconds() / 86400` (censored). Negative values are dropped and counted in QA. |
| `event` | 1 if closed, 0 if open |
| `road_class` | §4.4 |
| geography | §4 |

`channel` and `in_demand` are always evaluated from the row's own `origin`, `created_department`, and `status`, including for duplicates. A repeat report is its own resident act.

#### 2.1.1 Effective duplicates and root requests

The raw `duplicate` flag is not used for counting. Many duplicates point to a parent that is not a pothole request: in 2024, 1,925 of 17,319 duplicates (11%) had a parent outside the PHF/PHB rows of Oct 2023–Dec 2024. A 40-row sample of those found 25 parents of other request types (mostly Inspect Public Way and Sewer Cave-In Inspection), 1 older PHF, and 14 not in the public dataset. No 2019–2026 duplicate has a null `parent_sr_number`, and no 2024 duplicate's parent was itself a duplicate.

A **pothole row** is any PHF or PHB row in the main pull or in `parents.parquet`, regardless of date, status, or `legacy_record`.

**`root_sr`.** Start at the row. While the current row has `duplicate = true` and its `parent_sr_number` is a pothole row, move to the parent. Stop after 10 moves. `root_sr` is the `sr_number` of the last pothole row reached.
- If the walk returns to a row already visited (a cycle), `root_sr` is the starting row's own `sr_number`. Cycles are counted in QA.
- If the walk stops because of the 10-move cap, the count is recorded in QA.

**`dup_eff`** (effective duplicate) = `root_sr ≠ sr_number`. Equivalently: `duplicate = true` and the parent resolves to a pothole row, except for rows in a cycle.

**Promoted rows** are rows with `duplicate = true` and `dup_eff = false`. The parent is another request type, was not found, or is part of a cycle. A promoted row is the first pothole report of its problem and enters R (§6). Promoted rows are counted by year and by reason (`other_type`, `not_found`, `cycle`) in QA (W8).

**`link_channel`** (§3) remains the channel of `root_sr`.

### 2.2 Channel crosswalk

Rules are applied in order; the first match wins. Store them in `config/channel_rules.json`. The build fails on any non-null `origin` that matches no rule.

| # | Condition | `channel` |
|---|---|---|
| 1 | `origin = "Alderman's Office"` | `alderman` |
| 2 | `origin = "Generated In House"` OR `created_department = "CDOT - Department of Transportation"` | `crew_logged` |
| 3 | `created_department = "Alderman"` | `alderman` |
| 4 | `origin IN ("Salesforce Mobile App","City Department","CSCC","RMS","Mass Entry","Chicago Police Department","Chicago Fire Department","Radio","Mayor's Office","PPCA")` | `staff_observed` |
| 5 | `origin IN ("Mobile Device","Internet","Web","Open311","Open311-Interface","Social Media")` OR `origin` starts with `"spot-open311-"` | `resident_digital` |
| 6 | `origin = "Phone Call"` | `resident_phone` |
| 7 | `origin IN ("E-Mail","Fax","Mail","Walk-in","HealthProfessionals","Budget Town Hall Meeting","FY25 Budget Engagement")` or matches `/budget\|town hall\|engagement/i` | `resident_other` |
| 8 | `origin` is null | `unmapped` (excluded from demand; must be ≤ 0.1% per year) |

**Evidence for rule 2** (snapshot 2026-09-16; non-duplicate, completed PHF, 2019–2026). The share closed on the same calendar day was:
- 74–87% by year for `Phone Call` entered by CDOT.
- 81% for `Generated In House`.
- 2.5% for `Mobile Device`, 0.4% for `Internet`, 0.5% for `Alderman's Office`, and 0.6% for `Salesforce Mobile App`.

Records matched by rule 2 made up 7.6% of non-duplicate, non-canceled PHF in 2019 and 21–31% in each year from 2020 to 2025. They are treated as crew-initiated work logs, not resident demand. They are counted separately (`crew_n`) and used in fill attribution (`fill_crew_share`). Check D8 publishes a sensitivity analysis.

### 2.3 Potholes Patched

- Keep all rows. Every row is completed work (no null `completion_date` exists).
- `wo_key` = SHA-1 of `address|request_date|completion_date|number_of_potholes_filled_on_block|latitude|longitude`. Exact duplicate rows are kept and counted in QA.
- `fill_count` = `number_of_potholes_filled_on_block` as an integer.
- `extreme` = `fill_count > 150`. This fixed threshold is about 2× the pooled p99 of 70.
- `patch_days` = `(completion_date − request_date).total_seconds() / 86400`.
- Location: use the row's own `latitude`/`longitude`. If those are null, use the linked SR's coordinates (§3). If neither exists, the row is `UNASSIGNED`.

---

## 3. Linkage: Patched → 311

`request_date` in Patched equals the 311 `created_date` of the originating request (profile Q10: 98% match within ±60 s, with no gain at ±1 day).

**`norm_addr(s)`**
1. Uppercase, trim, collapse whitespace, and strip punctuation other than `-`.
2. House-number token: if it is numeric apart from the letters `O`, `I`, or `L`, map `O→0` and `I/L→1` (so `31OO` becomes `3100`). Strip leading zeros.
3. Directions: `NORTH/SOUTH/EAST/WEST → N/S/E/W`.
4. Suffixes: `AVENUE→AVE`, `STREET→ST`, `BOULEVARD→BLVD`, `ROAD→RD`, `DRIVE→DR`, `PLACE→PL`, `COURT→CT`, `PARKWAY→PKWY`, `TERRACE→TER`, `HIGHWAY→HWY`, `EXPRESSWAY→EXPY`.
5. Collapse a repeated trailing suffix, so `645 E NORTH AVE AVE` becomes `645 E NORTH AVE`.

**Match procedure**
1. Candidates are 311 rows of either code, any status, any channel, including duplicates, where `norm_addr(street_address) = norm_addr(address)` AND `|created_date − request_date| ≤ 60 s`.
2. If there are several candidates, prefer `PHF` over `PHB`, then `duplicate = false`, then the smallest `|Δt|`, then the lowest `sr_number`. Set `link_method = 'addr_time'`.
3. Fallback: `|Δt| ≤ 60 s` AND point distance ≤ 150 ft in EPSG:3435. Set `link_method = 'geo_time'`.
4. Otherwise set `link_method = 'none'`.

**Outputs per Patched row:** `sr_number`, `root_sr`, `link_code` (PHF/PHB), `link_channel` (the channel of `root_sr`), `link_method`.

Fill and patch attribution uses `root_sr`: its channel (`link_channel`) and its `created_date`. Report metrics use each row's own channel (§2.1).

---

## 4. Geography

### 4.1 Coordinate reference system
All geometric operations use EPSG:3435 (NAD83 / Illinois East, US survey feet). Miles = feet / 5280.

### 4.2 Point-in-polygon assignment
- Use a `covered_by` test. If a point falls in more than one polygon, assign it to the lowest numeric ID.
- A point with null coordinates or no containing polygon gets `geo_id = 'UNASSIGNED'`.

| `geo_type` | Polygons | `geo_id` | `geo_name` |
|---|---|---|---|
| `city` | union of community areas | `CHI` | `Chicago` |
| `community_area` | `igwz-8jzy` | `area_numbe` as string, `"1"`–`"77"` | `community` in title case, with an override map (e.g., `OHARE → O'Hare`) |
| `ward2023` | `p293-wvbd` | `ward` as string, `"1"`–`"50"` | `Ward {n}` |
| `pov_quintile` | 2020 tracts (§4.7) | `Q1`–`Q5` (Q1 = lowest poverty) | `Lowest-poverty fifth` … `Highest-poverty fifth` |
| `race_group` | 2020 tracts (§4.7) | `hispanic`, `nh_black`, `nh_white`, `nh_asian`, `no_majority` | plain-English labels |

Tract strata (`pov_quintile`, `race_group`) are assigned in this order:
1. A point outside the `city` union (or with null coordinates) gets `UNASSIGNED`. This applies to every non-city `geo_type`, so all of them share the same `UNASSIGNED` rows and V2 compares like with like.
2. A point in a tract that fails the §4.7 filters gets `EXCLUDED`.
3. Otherwise, the point gets its tract's stratum.

Step 1 also keeps addresses outside the city from being credited to tracts whose population (§4.7) counts only the in-city part.

### 4.3 Wards before the 2023 remap
- Every period is assigned to `ward2023`.
- For periods that start before 2023-05-15, also assign `in_demand` PHF points to `k9yb-bpqx` and write `ward_legacy_mix.csv` with columns `ward2023, period, ward2015, share_of_rpt_n`.
- Period flags on `ward2023` rows: `PRE2023_WARD` for CY2019–CY2022 and YTD2019–YTD2022; `SPLIT_2023` for CY2023 and YTD2023.
- The site shows no alderperson names for any flagged period.
- A `CHG_*` period (§5) on `ward2023` carries the union of the flags of its two windows. The default ward comparison (§6.6) uses only unflagged windows.
- `ward_legacy_mix.csv` also covers pooled windows (`CY{A}-{B}`) that start before 2023-05-15.

### 4.4 Road class
Snap each point to the nearest `pr57-gg9e` segment within 100 ft, considering all classes except `RIV`.

| Segment `class` | `road_class` |
|---|---|
| 1, 9 | `limited_access` |
| 2, 3 | `arterial` |
| 4, 5, 7 | `local` |
| `E`, `99`, `S`, or no segment within 100 ft | `other_unsnapped` |

Class meanings are inferred from observed street names: class 1 includes the expressways, the Skyway, and Lake Shore Dr; class 9 contains ramps; class 7 contains lower-level streets. Check D5 confirms these.

### 4.5 Street-mile denominator
- Segments: `class IN ('2','3','4')` AND `status = 'N'`.
- Clip the segments to each polygon and sum the geometric length.
- A clipped piece that lies on a shared boundary (returned for more than one polygon of the same `geo_type`) counts only for the lowest numeric `geo_id`. Pieces inside no polygon count toward `UNASSIGNED`. Without this rule, the 2026-09-16 snapshot double-counted ≈ 92 mi on community-area boundaries (Σ CA ≈ 4,010 mi vs. ≈ 3,918 mi for their union) and failed V9.
- Expected citywide total ≈ 3,945 mi. At the 2026-09-16 snapshot, `length` sums to 422.7 mi (class 2), 473.7 mi (class 3), and 3,048.8 mi (class 4).
- Per-mile numerators exclude points with `road_class = 'limited_access'`, so the numerator and denominator cover the same network.

### 4.6 Population
- Source: 2020 `P1_001N` by block.
- Assign each block to a geography by the point-in-polygon location of its internal point.
- One population vintage is used for every period.

### 4.7 Equity strata (tract level)
1. Take 2020 tracts in Cook County. `tract_pop_city` = Σ `P1_001N` of blocks whose internal point lies inside the city.
2. Keep tracts with `tract_pop_city ≥ 500` AND `B17001_001E ≥ 200`. Other tracts become `EXCLUDED`.
3. `poverty_rate` = `B17001_002E / B17001_001E`. This is the whole-tract ACS rate; ACS values are not split across the city limit.
4. **Poverty quintile:** sort tracts by `poverty_rate` ascending (ties broken by GEOID). Compute cumulative `tract_pop_city`. A tract's quintile is `ceil(5 × (cum_pop − tract_pop_city/2) / total_pop)`, clamped to 1–5.
5. **Race group:** sum the 2020 P2 counts of in-city blocks per tract. Compute shares of Hispanic (`P2_002N`), non-Hispanic White (`P2_005N`), non-Hispanic Black (`P2_006N`), and non-Hispanic Asian (`P2_008N`) over `P2_001N`. The group is whichever share is ≥ 0.50; otherwise `no_majority`. If a group has fewer than 300 pooled `rpt_n` per year, merge it into `no_majority` and note this in QA.
6. **Covariates for community areas and wards** (tooltips only): allocate each tract's numerator and denominator counts to each geography by block-population share, sum them, then take the ratio. Medians are never aggregated.
   - `poverty_rate` = B17001_002 / B17001_001
   - `lep_hh_share` = (C16002_004 + _007 + _010 + _013) / C16002_001
   - `broadband_hh_share` = B28002_004 / B28002_001
   - race shares from P2
7. The build asserts Census variable labels against the API `variables.json` for the configured vintage (check V12).

---

## 5. Periods

| `period` | Window | Notes |
|---|---|---|
| `CY{YYYY}` | `YYYY-01-01T00:00:00 ≤ t < (YYYY+1)-01-01T00:00:00` | 2019 through the current year. The current year is flagged `PARTIAL_YEAR` and is not mapped. |
| `YTD{YYYY}-{MM}` | `YYYY-01-01T00:00:00 ≤ t < first day of month MM+1 in YYYY` | `MM` is the last full calendar month that ended ≥ 7 days before `data_currency`; the same `MM` is used for every year 2019–current. At this snapshot, `MM = 08`, which covered 83% of 2019–2025 resident reports. |
| `CY{A}-{B}` | Pooled full calendar years A–B | Equity charts (`CY2019-2025`) and change windows (§6.6). Per-mile rates over a pooled window are divided by the number of years (§6.6). |
| `CHG_{base}_{recent}` | Comparison of two windows, each a `CY{YYYY}`, `CY{A}-{B}`, or `YTD{YYYY}-{MM}` period | Only the three comparisons in §6.6. Used by change metrics only. |

**Date attribution by metric family**
- Reports: `created_date`.
- Close time: `created_date` (the cohort).
- Patch time: `created_date` of `root_sr`, which equals `request_date`.
- Fills: `completion_date`.

**Administrative censoring**
- CY cohorts are observed as of `data_currency`.
- YTD cohorts for year Y are observed as of `asof_Y` = the month, day, and time of `data_currency` placed in year Y (Feb 29 maps to Feb 28).
- For a YTD cohort: `t' = min(dur_days, (asof_Y − created_date)/86400)` and `event' = event AND closed_date ≤ asof_Y`.

---

## 6. Metrics

Notation:
- **R** = PHF rows with `in_demand = true` and `dup_eff = false` (§2.1.1). R includes promoted rows.
- **C_P** = rows of R created in period P.
- **F** = Potholes Patched rows.
- **A** = PHB equivalent of R.

### 6.1 Reports

| `metric` | Formula | `n` | Mapped |
|---|---|---|---|
| `rpt_n` | count(R) by `created_date` | = value | tooltip |
| `rpt_per_mi` | count(R, `road_class ≠ limited_access`) / `street_mi` | numerator | **default** |
| `rpt_per_1k` | count(R) / `pop2020` × 1000 | numerator | toggle |
| `rpt_all_n` | count(PHF, `in_demand`), including effective duplicates | = value | tooltip |
| `rpt_dup_share` | count(PHF, `in_demand`, `dup_eff = true`) / `rpt_all_n` | `rpt_all_n` | tooltip |
| `rpt_alder_share` | count(R, `channel = alderman`) / `rpt_n` | `rpt_n` | toggle |
| `rpt_digital_share` | count(R, `resident_digital`) / count(R, `channel ≠ alderman`) | denominator | equity charts |
| `rpt_yoy_pct` | (`rpt_n`[P] / `rpt_n`[P−1] − 1) × 100, CY-to-CY or YTD-to-YTD only | `rpt_n`[P−1] | city chart and tooltips |
| `crew_n` | count(PHF, `duplicate = false`, `channel = crew_logged`, `status_class ≠ canceled`) | = value | city chart |
| `staff_n` | same filter as `crew_n` with `channel = staff_observed` | = value | QA |

**Pooled periods.** For a rate metric (`*_per_mi`, `*_per_1k`) over a pooled period `CY{A}-{B}`:
- value = Σ numerator over the window ÷ denominator ÷ (B − A + 1). The value is annualized, so it sits on the same scale as single-year values.
- `n` = the metric's `n` from the tables above, summed over the window.

This matches the §6.6 window values.

### 6.2 Response time

**Kaplan–Meier estimator.** Input: pairs `(t_i, e_i)` from C_P, with `t_i = dur_days ≥ 0` (or `t'` for YTD).
1. Let τ₁ < … < τ_k be the distinct times with `e = 1`.
2. `d_j` = the number of events at τ_j.
3. `r_j` = #{i : t_i ≥ τ_j}. Rows censored at τ_j count as at risk.
4. `S(τ_j) = Π_{l ≤ j} (1 − d_l / r_l)`. S is a right-continuous step function with S(t) = 1 for t < τ₁. Events at t = 0 are kept.
5. Quantile q: the smallest τ_j with `S(τ_j) ≤ 1 − q`. If there is none, the value is null with flag `KM_NOT_REACHED`.
6. Implement in the standard library. Validate against `lifelines` in tests only.

| `metric` | Formula | `n` | Mapped |
|---|---|---|---|
| `close_p50_d` | KM q = 0.5 on C_P, rounded to 0.1 day | \|C_P\| | **default speed** |
| `close_p90_d` | KM q = 0.9 | \|C_P\| | tooltip |
| `close_le7_share` | `1 − S(7.0)` | \|C_P\| | toggle |
| `close_open_share` | count(`event = 0`) / \|C_P\| at the observation date | \|C_P\| | QA / flag driver |
| `patch_p50_d` | Among C_P rows whose `root_sr` has ≥ 1 linked F row: median of `(min completion_date − created_date)` in days. Completed-only by construction. | count of such rows | toggle |
| `patch_p90_d` | Same population, p90 | same | tooltip |
| `rpt_patched_share` | Among C_P rows with `event = 1`: share whose `root_sr` has ≥ 1 linked F row | count(`event = 1`) | toggle |

**Uncensored quantiles.** `patch_p50_d`, `patch_p90_d`, and the completed-only p50 in V7 use nearest-rank on unrounded days: x₍⌈q·n⌉₎ of the n sorted values (Hyndman–Fan type 1; numpy `method="inverted_cdf"`). This is what the KM quantile rule above reduces to when no row is censored, so `patch_*` and `close_*` use the same estimator.

**Rounding.** Every `*_d` value is rounded to 0.1 day with `decimal` `ROUND_HALF_UP`, applied to the unrounded value. Python's built-in `round()` is not used, because it rounds half to even on binary floats and would make V11 comparisons fragile.

- `patch_*` and `rpt_patched_share` are **withheld** (value null, flag `COHORT_OPEN`) when `close_open_share ≥ 0.01` for that geography and period. They are not computed for YTD periods.
- **Stratified variants** (only for `geo_type` in `city`, `pov_quintile`, `race_group`):
  - `close_p50_d__arterial`, `close_p50_d__local`
  - `close_p50_d__resident`, `close_p50_d__alderman`
  - the same suffixes on `close_le7_share`
  - Here `__resident` means `channel IN (resident_digital, resident_phone, resident_other)`.

### 6.3 Potholes filled

| `metric` | Formula | `n` | Mapped |
|---|---|---|---|
| `fill_n` | Σ `fill_count` over F by `completion_date` (uncapped) | `fill_wo_n` | tooltip |
| `fill_wo_n` | count(F) | = value | tooltip |
| `fill_per_mi` | Σ `fill_count` (`road_class ≠ limited_access`) / `street_mi` | work orders in numerator | **default fills** |
| `fill_crew_share` | Σ `fill_count` where `link_channel = crew_logged` / `fill_n` | `fill_wo_n` | toggle |
| `fill_demand_share` | Σ `fill_count` where `link_channel` is a demand channel / `fill_n` | `fill_wo_n` | equity charts |
| `fill_unlinked_share` | Σ `fill_count` where `link_method = none` / `fill_n` | `fill_wo_n` | QA |
| `fill_extreme_share` | Σ `fill_count` where `extreme` / `fill_n` | `fill_wo_n` | flag driver |
| `fill_per_rpt` | `fill_n` / `rpt_n`. Not a percentage; can exceed 1. | `rpt_n` | **equity charts only; never mapped** |

If check D3 finds that PHB-linked rows are ≥ 1% of F, split all `fill_*` metrics into `fill_*` (link_code ≠ PHB) and `alley_fill_*` before publishing.

### 6.4 Alleys (PHB)

- `alley_rpt_n`, `alley_close_p50_d`, `alley_close_le7_share` use the same definitions as §6.1–6.2, with A in place of R.
- There are no per-mile or per-capita variants. The source data has no alley centerline layer, and per-capita rates would mostly reflect how common alleys are.
- Mapped: `alley_close_p50_d` only.

### 6.5 Map offer matrix

| Layer | community_area | ward2023 |
|---|---|---|
| Reports | `rpt_per_mi` (default), `rpt_per_1k`, `rpt_alder_share` | same |
| Response time | `close_p50_d` (default), `close_le7_share`, `patch_p50_d`, `rpt_patched_share` | same |
| Filled | `fill_per_mi` (default), `fill_crew_share` | same |
| Alleys | `alley_close_p50_d` | same |
| Change (§6.6) | `rpt_per_mi_chgcls` (default), `fill_per_mi_chgcls`, `close_le7_share_chgcls` | same |

- **Default view:** Change layer × `community_area` × `rpt_per_mi_chgcls` × the `long` comparison (§6.6). The Level layers above keep their defaults: `rpt_per_mi` × the latest `CY` period without `PARTIAL_YEAR`.
- **Default ward comparison:** `last` (§6.6), because it uses only post-remap years.
- When the user selects the current year, the map switches to YTD mode and all periods in the selector become `YTD{YYYY}-{MM}`.
- **Never offered:** fills per capita; any denominator on time or share metrics; `fill_per_rpt` on a map; raw-count choropleths; alley per-mile; `CY` partial years; equity strata on the map; alderperson names on flagged periods; segment- or hex-level metrics; bivariate classes; change classes for KM quantiles (`*_p50_d`, `*_p90_d`).

### 6.6 Change over time

**Comparisons.** Let L = the last full CY, and Y = the current year.

| ID | Base window | Recent window | Example at this snapshot |
|---|---|---|---|
| `long` | `CY{L−5}-{L−3}`, starting no earlier than 2020 | `CY{L−2}-{L}` | `CHG_CY2020-2022_CY2023-2025` |
| `last` | `CY{L−1}` | `CY{L}` | `CHG_CY2024_CY2025` |
| `ytd` | `YTD{Y−1}-{MM}` | `YTD{Y}-{MM}` | `CHG_YTD2025-08_YTD2026-08` |

Change metrics are computed for `geo_type` ∈ {`city`, `community_area`, `ward2023`}. Base metrics are `rpt_per_mi`, `fill_per_mi`, and `close_le7_share`.

**Window values.** For a rate metric M, M[W] = numerator over window W ÷ (`street_mi` × years in W). For a single CY or a YTD window, years = 1. For `close_le7_share`, W's cohort is the union of its years' cohorts, with the §5 censoring rules. Pooled level values are published as ordinary rows with period `CY{A}-{B}`, and §7.1 applies to them.

**Rate metrics** (`rpt_per_mi`, `fill_per_mi`), for geography g and comparison (B, R):
- `M_chg_pct` = (M_g[R] / M_g[B] − 1) × 100
- `M_vscity_pct` = ((M_g[R] / M_g[B]) / (M_city[R] / M_city[B]) − 1) × 100
- se = √( Σ_R w² / (Σ_R w)² + Σ_B w² / (Σ_B w)² ), summed over the numerator rows of g (`road_class ≠ limited_access`). w = 1 per report for `rpt_per_mi`, and w = `fill_count` per work order for `fill_per_mi`, which accounts for block-level entries.
- The change is **clear** if |ln(1 + `M_vscity_pct`/100)| ≥ 1.96 × se.

**Share metric** (`close_le7_share`). Write C[W] = `close_le7_share` = 1 − Ŝ_W(7), where Ŝ_W is the §6.2 KM survival function for window W.
- `close_le7_share_chg_pts` = (C_g[R] − C_g[B]) × 100
- `close_le7_share_vscity_pts` = ((C_g[R] − C_g[B]) − (C_city[R] − C_city[B])) × 100
- Var(C[W]) uses Greenwood's formula: Ŝ_W(7)² × Σ_{τ_j ≤ 7} d_j / (r_j (r_j − d_j)). Terms with r_j = d_j are skipped. se = √(Var_g[R] + Var_g[B]).
- The change is **clear** if |`close_le7_share_vscity_pts`| ≥ 1.96 × se × 100.

City sampling variance is ignored (the city term is treated as fixed). `n` = min(base, recent) numerator rows (reports, work orders, or cohort size). The implementation uses the standard library, and the V14 fixtures validate it.

**Classes** (`M_chgcls`, integer −2…2). The sign follows `vscity`: + means higher than the city trend.
- 0 if the change is not clear, or if |vscity| < t1.
- ±1 if t1 ≤ |vscity| < t2.
- ±2 if |vscity| ≥ t2.
- Rates use t1 = 10 and t2 = 25 (%). `close_le7_share` uses t1 = 5 and t2 = 10 (points).
- Thresholds live in `config/params.json` and are fixed, not quantile-based.
- A row whose change is not clear also gets the flag `CHANGE_UNCLEAR`.

**Display orientation** (§7.2) marks the direction residents are likely to read as concerning: more reports, fewer fills, or a lower share closed within 7 days. The legend words stay literal.

---

## 7. Suppression and classification

### 7.1 Minimum-n rules
Suppressed cells have `value = null` and the flag `SUPPRESSED_LOW_N`. Their `n` is still published.

| Metric type | Show only if |
|---|---|
| Rate (`*_per_mi`, `*_per_1k`) | `expected_n ≥ 30`, where `expected_n` = (citywide numerator ÷ city denominator) × the geography's denominator, for the same metric and period. The numerator is counted in the units of the metric's `n` (see below). This rule depends on the denominator, not on the observed count, so genuinely low areas are not hidden. |
| Share | denominator count ≥ 30 |
| KM p50, `*_le7_share` | \|C_P\| ≥ 30 |
| KM p90, `patch_p90_d` | population ≥ 100 and quantile reached |
| `patch_p50_d` | ≥ 30 linked rows |
| `rpt_yoy_pct` | `rpt_n`[P−1] ≥ 30 |
| Equity stratum, single year | stratum `rpt_n` ≥ 300; otherwise show the pooled period only |
| Change metrics (§6.6) | the base metric passes this table in both windows, and `n` ≥ 30; otherwise every change metric for that row is null |

**Rate details**
- **Pooled periods.** For `CY{A}-{B}`, `expected_n` uses the citywide numerator summed over the whole window and is not annualized. Suppression depends on how many events stand behind the number, so the whole window counts. The single-year stratum gate (`rpt_n` ≥ 300) still applies separately.
- **`fill_per_mi`.** The gate counts work orders, not potholes: `expected_n` = (city `fill_wo_n` with `road_class ≠ limited_access` ÷ city `street_mi`) × geography `street_mi`. Potholes cluster within work orders (one order may carry 34, another 467), so stability depends on the number of independent crew visits. This matches the `n` given in §6.3.
- **`crew_n` per street mile** (§9) uses the same rule, counted in records.

### 7.2 Class breaks
- For each `(geo_type, metric)` that is mapped, pool all non-suppressed values across CY2019 through the last full CY.
- Compute 5-class quantile breaks, round them to 2 significant figures, and deduplicate. The top class is open-ended.
- **YTD breaks.** YTD mode stores one break set per `MM` (`"01"`–`"12"`) in `breaks.json` under `ytd.{MM}.{geo_type}.{metric}`.
  - Each set pools `YTD{Y}-{MM}` for Y = 2019 through the last full CY at the time of the version bump. The current year is excluded, because its values change with every build.
  - When computing breaks, time metrics use as-of date = window end + 7 days (the earliest cutoff the §5 `MM` rule allows) in place of `asof_Y`.
  - All 12 sets are computed together, only at a version bump. The build reads them and never writes them. The build fails if the key for the current `MM` is missing.
  - Current-year values beyond the historical range fall in the open-ended top class.
- Breaks are written to `data/processed/breaks.json` with `methods_version`. They are frozen until a version bump, so adding a new year does not recolor past years.
- Colors: one sequential, colorblind-safe ramp for all Level metrics. Legend direction words: "more reports", "slower", "higher share". Suppressed cells use a hatched pattern that is distinct from the lowest class.
- Change layers use the fixed §6.6 classes, not quantiles, with one diverging, colorblind-safe ramp. Class 0 is the neutral midpoint. Legend words are literal (e.g., "rose more than the city" / "fell more than the city").
- Legend footnote on Change layers: "Colored only where the difference from the city trend is larger than normal variation for an area this size."
- Legend footnote on report layers: "More reports per mile ≠ more potholes per mile."

---

## 8. Output tables

### 8.1 `data/processed/potholes_long.csv` (primary)

| Column | Type | Allowed values / rules |
|---|---|---|
| `geo_type` | string | `city`, `community_area`, `ward2023`, `pov_quintile`, `race_group` |
| `geo_id` | string | See §4.2. Also `UNASSIGNED` (any `geo_type` except `city`) and `EXCLUDED` (strata only). |
| `geo_name` | string | Display name. `Unassigned location` / `Excluded tracts` for the special IDs. |
| `period` | string | Matches `^(CY\d{4}\|YTD\d{4}-\d{2}\|CY\d{4}-\d{4}\|CHG_(CY\d{4}(-\d{4})?\|YTD\d{4}-\d{2})_(CY\d{4}(-\d{4})?\|YTD\d{4}-\d{2}))$` |
| `metric` | string | From §6. Matches `^[a-z0-9_]+(__[a-z]+)?$` |
| `value` | float or empty | Empty when suppressed or not reached. Counts are integers. `*_per_*` values have 2 decimals. `*_d` values have 1 decimal. `*_share` values are on a 0–1 scale with 4 decimals. `*_pct` values have 1 decimal. `*_pts` values (percentage points) have 1 decimal. `*_chgcls` values are integers from −2 to 2. |
| `n` | int | Meaning is defined per metric in §6. Always populated, including for suppressed rows. |
| `flags` | string | Pipe-delimited codes from §8.3, sorted alphabetically, or empty. |

- **Key:** `(geo_type, geo_id, period, metric)` must be unique.
- **Sort order:** `geo_type`, `geo_id` (numeric-aware), `period`, `metric`.
- **Scope:** `UNASSIGNED` and `EXCLUDED` rows are written only for count metrics (`rpt_n`, `rpt_all_n`, `crew_n`, `staff_n`, `fill_n`, `fill_wo_n`, `alley_rpt_n`).

### 8.2 Companion tables
- `geo_denominators.csv`: `geo_type, geo_id, street_mi, pop2020, poverty_rate, lep_hh_share, broadband_hh_share, pct_hispanic, pct_nh_black, pct_nh_white, pct_nh_asian, acs_vintage, centerline_rows_updated`
- `ward_legacy_mix.csv`: `ward2023, period, ward2015, share_of_rpt_n` (see §4.3)
- `strata_tracts.csv`: `geoid, tract_pop_city, poverty_rate, pov_quintile, race_group, excluded_reason`
- `breaks.json`: `{methods_version, level: {cy: {geo_type: {metric: [b1..b4]}}, ytd: {MM: {geo_type: {metric: [b1..b4]}}}}, change: {rate_pct: [t1, t2], share_pts: [t1, t2]}}`. It is written only at a version bump (§7.2).
- `change_summary.csv`: `geo_type, period, metric, up, down, flat, suppressed`. Counts of geographies with `*_chgcls` > 0, < 0, = 0, and null. Feeds §8.4.
- `qa_report.json`: outputs of every Gate 0–2 check, with snapshot metadata

All companion tables are listed in `checksums.sha256` under the existing release contract.

### 8.3 Flags and caveat registry

| Flag | Set when | Caveat shown |
|---|---|---|
| `PARTIAL_YEAR` | period is a CY that has not ended | C-YTD |
| `YTD` | period is a YTD | C-YTD |
| `TRANSITION_2019` | period includes 2019 | C-2019 |
| `PRE2023_WARD` | `ward2023` and period ends before 2023-01-01, or is a `CHG_*` period with such a window | C-WARD-PRE2023 |
| `SPLIT_2023` | `ward2023` and period is CY2023/YTD2023, a pooled window containing 2023, or a `CHG_*` period with such a window | C-2023 |
| `SUPPRESSED_LOW_N` | §7.1 rule failed | C-SUPPRESS |
| `KM_NOT_REACHED` | KM quantile not reached | C-SUPPRESS |
| `COHORT_OPEN` | `close_open_share ≥ 0.01` (patch metrics withheld) | C-PATCH |
| `EXTREME_FILL` | `fill_extreme_share ≥ 0.10` (on `fill_*` rows) | C-EXTREME |
| `GEO_UNASSIGNED` | `geo_id = UNASSIGNED` | — |
| `CHANGE_UNCLEAR` | change metric row whose §6.6 test is not met | C-CHANGE |

**Caveat text** (≤ 40 words each; dates are templated):

- **C-REPORTS** — Reports count when residents and ward offices tell 311 about a pothole, with repeat reports removed. They are not a count of potholes; weather, awareness, app use, and ward-office practice all move this number.
- **C-CREW** — Many city pothole records are entered by CDOT itself and closed the same day. We treat them as crew-initiated work, not resident reports, and count them separately.
- **C-CLOSE** — Time to close is how long 311 took to mark a report complete. Closing doesn't always mean patching. Still-open reports are included with a survival estimate, so recent years don't look artificially fast.
- **C-PATCH** — Time to patch covers only reports that led to a recorded patch, so it describes those repairs, not every report. Years with many reports still open are withheld until they settle.
- **C-FILLS** — Potholes filled is the per-block count crews enter when they finish work. One report can lead to patching a whole block, so fills can exceed reports.
- **C-EXTREME** — Some crew entries list 150+ potholes on one block. They're kept as recorded, and they make up at least 10% of this area's total for this year.
- **C-YTD** — {YYYY} isn't over. Year-to-date views compare Jan. 1–{Mon DD} in every year, and response times use only what was known by the same date each year.
- **C-2019** — 2019 was the first full year of the current 311 system. Its duplicate flags and entry practices differ from later years, so compare it with caution.
- **C-WARD-PRE2023** — Years before May 15, 2023, are shown on today's ward map for comparison. This area was then split among different wards and alderpersons. These figures say nothing about any current alderperson.
- **C-2023** — 2023 straddles the ward remap. Reports before May 15 were made under the old boundaries.
- **C-ALLEY** — Alley potholes are a separate request type with different handling, and city data has no alley mileage. They're shown only as counts and response times.
- **C-EQUITY** — Groups are census tracts ranked by poverty rate (ACS) or grouped by majority race/ethnicity (2020 Census). Differences are descriptive and don't show why service differs or how any individual was treated.
- **C-SUPPRESS** — Not shown: too few requests, or too little street mileage or population, for a stable figure.
- **C-CHANGE** — Change compares the same months or years before and after. Areas are colored only where the gap from the citywide trend exceeds normal variation for an area that size. This shows where things moved, not why.
- **C-WARDOFFICE** — Ward office requests are pothole reports filed by an alderperson's office, often for residents who called it. A high share can reflect how an office handles calls, not how many potholes there are.

### 8.4 Generated sentences

Headline and panel sentences are templates stored in `config/copy.json` and versioned with this document.
- **Slots:** only published values from §8.1 and counts from `change_summary.csv`. The front end fills slots and does no arithmetic.
- **Verbs:** "rose", "fell", "was faster/slower", and "more/less than the city". No causal verbs ("because", "due to", "led to") and no grades ("better", "worse", "failing").
- **Subjects:** sentences describe reports, fills, or closing times, never potholes or street condition.
- **Names:** no alderperson names. A sentence about a flagged ward period appends the short form of its caveat.
- **Unclear changes:** a `CHANGE_UNCLEAR` row is described only as "close to the city trend."

---

## 9. Equity analysis (chart panel, not the map)

For each stratum in `pov_quintile` and `race_group`, show the pooled period and, where §7.1 allows, single years. Compare:

1. **Reporting vs. work on the asset.** `rpt_per_mi` against `fill_per_mi`, plus `fill_per_rpt`. If fills per mile track reports per mile while reports diverge across strata, the work is following the complaints.
2. **Access to the reporting channel.** `rpt_per_1k`, `rpt_digital_share`, `rpt_alder_share`.
3. **Work the city allocates on its own.** `fill_crew_share` and `crew_n` per street mile (suppressed per §7.1). This is the only fill stream not triggered by a complaint.
4. **Speed on like-for-like roads and channels.** `close_p50_d` and `close_le7_share`, overall, `__arterial` vs. `__local`, and `__resident` vs. `__alderman`.
5. **Outcome of a report.** `rpt_patched_share`.

Race and poverty are strongly collinear in Chicago. Do not cross-tabulate them.

---

## 10. Validation

### Gate 0 — diagnostics (run before freezing v1.0; results go to `docs/diagnostics.md`; any of these can change a decision above)

| ID | Check | Decision it informs |
|---|---|---|
| D1 | Reverse linkage: share of closed R rows with a linked F row, by year and channel | If any year is below ~50%, re-examine what "Completed" means before close time becomes the headline speed metric |
| D2 | Distribution of F rows per `root_sr` | If > 5% of SRs have more than one row, revisit the `min(completion_date)` rule |
| D3 | Share of F rows linked to PHB | ≥ 1% → split street and alley fills (§6.3) |
| D4 | For linked pairs: distribution of `completion_date − closed_date` | Whether closing and patching are the same event |
| D5 | Confirm meanings of centerline `class` (5, 7, 9, E, 99, S) and `status` (N, P, V, UC, C). Share of `fill_n` snapped to Lake Shore Dr segments | If LSD > 0.5% of `fill_n`, add LSD segments to both the numerator and the denominator |
| D6 | Distribution of snap distances | 100 ft should capture ≥ 97% of points |
| D7 | 2020 population deviation across `ward2023` | If max \|dev\| ≤ 5%, add a tooltip note that per-capita ≈ raw count on wards |
| D8 | Citywide `rpt_n` and `close_p50_d` by year under three variants: as defined; with `crew_logged` included; with `Salesforce Mobile App` moved into demand; with the raw `duplicate` flag in place of `dup_eff` | Publish as a methods note |
| D9 | 311 `ward` field vs. point-in-polygon: rows before 2023-05-15 vs. `k9yb-bpqx`; later rows vs. `p293-wvbd` | Mismatch > 2% → investigate geocoding (profile Q12's 15% compared 2021 rows against the wrong map) |

### Gate 1 — build fails if any check fails

| ID | Check | Threshold |
|---|---|---|
| V1 | **Source reconciliation.** Saved pull-time portal aggregates vs. raw rows: Patched `count(*)` and `sum(number_of_potholes_filled_on_block)` by completion year; 311 `count(*)` by year × `sr_short_code` × `duplicate` × `status` | exact |
| V2 | **Additivity.** For every count metric and period: Σ `ward2023` + `UNASSIGNED` = `city` = Σ `community_area` + `UNASSIGNED` = Σ `pov_quintile` + `EXCLUDED` + `UNASSIGNED` (same for `race_group`) | exact |
| V3 | **Geography loss.** `UNASSIGNED` share of each count metric, each CY | ≤ 1.5% (warn at 1.0%) |
| V4 | **Channel completeness.** No unmatched non-null `origin`; `unmapped` share per year | ≤ 0.1% |
| V5 | **Linkage.** `link_method ≠ none` per completion year; `geo_time` share | ≥ 95%; ≤ 3% |
| V6 | **Durations.** No negative `dur_days` or `patch_days` in published sets; `bad_close` count recorded | 0 negative |
| V7 | **KM correctness.** Fixture tests match hand-computed / `lifelines` values. For cohorts with `close_open_share < 0.005`, KM p50 is within 0.5 day of the completed-only p50. | exact / 0.5 day |
| V8 | **Suppression.** No non-null value where §7.1 fails; every null value has a flag | exact |
| V9 | **Denominators.** City `street_mi` in [3,850, 4,050]; Σ geo `street_mi` = city ± 0.5%. Σ `ward2023` `pop2020` = Σ `community_area` = 2,746,388 ± 0.1% | as stated |
| V10 | **Schema.** Unique key; enum and regex validity; shares in [0, 1]; days ≥ 0; every mapped `(geo_type, metric)` present in `breaks.json` | exact |
| V11 | **Reproducibility.** Two builds from the same snapshot are byte-identical; every output is listed and hash-matched in `checksums.sha256` | exact |
| V12 | **Census labels.** Configured variable IDs match the expected label strings for the configured vintage | exact |
| V13 | **Change consistency.** Every `*_chg_*` and `*_vscity_*` value recomputes from the published window rows within rounding. `chgcls` matches §6.6. `change_summary.csv` matches the rows. | exact / ±0.1 |
| V14 | **Change statistics.** Fixture tests for the weighted log-ratio se and for Greenwood variance at t = 7 match hand-computed and `lifelines` values | 1e-9 |
| V15 | **Sentences.** Every `copy.json` template renders for every row it applies to. No template contains a banned verb or a name field. | exact |

### Gate 2 — warnings (written to `qa_report.json` and reviewed before publishing)

| ID | Check |
|---|---|
| W1 | Citywide `rpt_dup_share` moves > 15 pts from the prior year |
| W2 | Citywide `crew_logged` share of non-duplicate PHF moves > 10 pts from the prior year |
| W3 | Any mapped geography-year has `EXTREME_FILL` |
| W4 | Any geography-year value > 3× that geography's own 2019–(last full year) median |
| W5 | `data_currency` is more than 3 days older than `snapshot_id` |
| W6 | Any race group merged under §4.7 step 5 |
| W7 | More than 40% of mapped geographies are non-zero on any Change layer (check whether a citywide shift is leaking into `vscity`) |
| W8 | Promoted rows (§2.1.1) exceed 15% of `duplicate = true` PHF/PHB rows in any year. Report by year × reason (`other_type`, `not_found`, `cycle`), plus 10-move-cap hits. |

---

## 11. Optional context (not a metric)

A citywide chart of freeze–thaw days may accompany the reports series.
- Definition: days with Tmin < 32°F and Tmax > 32°F.
- Source: NOAA GHCN-Daily, O'Hare station.
- Grouping: by calendar year and by winter.
- It is never used to normalize or adjust any metric.

---

## 12. What this site doesn't claim

- **How many potholes exist, or whether streets are getting worse.** Reports and fills measure attention and work, not pavement condition.
- **That a closed report was filled.** A 311 closure is an administrative status. Only a linked Patched record indicates a recorded patch.
- **That fill counts are verified or that patches last.** Counts are crew-entered, and the data says nothing about repeat failures.
- **Why areas or groups differ.** The data has no pavement condition, traffic volume, pothole severity, local weather, or crew deployment information. Differences may reflect any of these, or reporting behavior, as much as service decisions.
- **Anything about individuals.** No claims are made about reporters, residents, or crews. Group comparisons are ecological.
- **Anything about the performance of an alderperson.** Ward offices file requests; CDOT fills potholes. Ward comparisons describe city service in an area and how much its ward office files. Neither is a grade of an alderperson. Pre-2023 figures on today's map describe areas that different alderpersons represented.
- **That a colored change is caused by anything in particular,** or that an uncolored area had no change. Change classes mark differences from the city trend that exceed count-based chance variation. They do not account for weather, reporting habits, or crew deployment.
- **That crew-logged records are definitely proactive.** That classification is inferred from the entering department and same-day closure.
- **Coverage of expressways, other limited-access roads, or private drives,** or of problems reported outside 311.
- **Anything before 2019,** or any forecast.

---

## 13. Changelog

| Version | Date | Change |
|---|---|---|
| 0.1 | 2026-09-16 | Initial draft following methods review. Pending Gate 0 results and open questions. |
| 0.2 | 2026-09-16 | Map design review (MAP_SPEC 1.0). Added: §6.6 change metrics, `CHG_*` periods, and pooled change windows; the Change map layer, now the default view; the `CHANGE_UNCLEAR` flag; C-CHANGE and C-WARDOFFICE caveats; §8.4 sentence rules; `change_summary.csv`; checks V13–V15 and W7. Fixed: double-counting of boundary segments in §4.5. Considered and rejected: segment-level fill metrics (every segment fails §7.1, even pooled) and bivariate covariate classes (they conflict with §4.7 and §9). |
| 0.3 | 2026-09-16 | Decisions on open questions. **Duplicates:** effective-duplicate rule and multi-hop `root_sr` (§2.1.1); parent lookup at fetch (§1); report metrics use each row's own channel; W8; D8 raw-flag variant. **Strata:** out-of-city points go to `UNASSIGNED` (§4.2). **Pooled rates:** annualized values; `expected_n` on the window total; `fill_per_mi` gated on work orders (§6.1, §7.1). **YTD breaks:** 12 frozen sets keyed by `MM`, current year excluded (§7.2, §8.2). **Quantiles:** nearest-rank for uncensored quantiles; `ROUND_HALF_UP` for `*_d` (§6.2). |
