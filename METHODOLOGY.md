# LBNL-LOAD Phase 4 Dataset: Structure and Methodology

This document describes the dataset in `LBNL_Phase_4/` and summarizes the
methodology behind it, drawn from the underlying study's final report and
technical appendices.

**Source study**: Gerke, B.F., Smith, S.J., Murthy, S., Baik, S.H., Agarwal,
S., Alstone, P., Khandekar, A., Zhang, C., Brown, R.E., Liu, J., and Piette,
M.A. *The California Demand Response Potential Study, Phase 4: Report on
Shed and Shift Resources Through 2050.* Lawrence Berkeley National
Laboratory, LBNL-2001596, May 21, 2024 (analysis substantially completed
October 2022; published after CPUC review). Prepared for the California
Public Utilities Commission.

- Full report: https://eta-publications.lbl.gov/sites/default/files/phase_4_dr_potential_study_final_2024-05-21.pdf
- Appendices: https://eta-publications.lbl.gov/sites/default/files/appendices_2024-05-21.pdf

---

## 1. Dataset structure

```
LBNL_Phase_4/
    anonymized-<weather>-<demand-trajectory>-<year>/
        cluster_summary.csv
        <cluster_name>.csv          (one per cluster, 8760 hourly rows)
        ...
```

Each top-level folder is one **scenario** — a combination of weather year,
demand/EE/FS trajectory, and forecast year (see §3). There is no shared
scenario-summary file; every scenario folder is self-contained.

### 1.1 `cluster_summary.csv`

One row per cluster. Key columns:

| Column | Meaning |
|---|---|
| `name` | Cluster identifier; matches the per-cluster CSV filename (`name + ".csv"`) |
| `sector`, `util`, `building_type`, `size`, `climate`, `care`, `lca`, `lshp`, `kwh_bin` | The segmentation hierarchy that defines the cluster (§2.2) |
| `customer_count` | Weighted number of real customers this cluster represents (not a raw sample count — see §2.1) |
| `kwh_ann_gross` / `kwh_ann_net` | Annual cluster-total kWh, gross (consumption) vs. net (after behind-the-meter PV) |
| `<end_use>_penetration` | Fraction of this cluster's customers with that end-use, for most (not all) end-uses |
| `*_frac` columns | Fraction of the cluster falling in each climate/size/CARE/LCA category (relevant mainly for merged categories, e.g. `Kern+Fresno`) |
| `*_count` (EV/truck/bus) | Assigned vehicle counts for electrified-fleet clusters (forecast scenarios only) |

### 1.2 Per-cluster CSVs

8760 hourly rows (`hour_ending` 1–8760), one column per end-use plus:

- **`total`** = sum of the true consumption end-use columns only. It does **not** subtract `pv_generation`. Treat it as **gross** load.
- **`pv_generation`** = modeled behind-the-meter PV output for the cluster, reported separately, always positive.
- **Net load** = `total − pv_generation` (not present as a column — must be computed).

End-use columns vary by sector/building type — there are 10 distinct
column schemas in this dataset, from `other, heating, cooling, pv_generation`
(coarse) up to ~20 end-uses for detailed residential clusters (cooking,
dishwasher, dryer, freezer, ventilation, indoor/outdoor lighting, pool_pump,
refrigeration, solar_water_heating, spa_heater, spa_pump, television,
washer, water_heating, office_equipment, pc, ev_level1, ev_level2, heating,
cooling), to a distinct schema for electrified-fleet clusters (truck/bus
categories, no per-vehicle-type penetration published).

### 1.3 Customer weighting

`customer_count` and `kwh_ann_*` are **energy-weighted extrapolations**
from the sampled AMI data back to the full utility population — not counts
of sampled customers. All cluster-level 8760 series are similarly weighted
sums, not simple averages, so dividing by `customer_count` gives a true
population-average per-customer profile for that cluster.

---

## 2. Underlying methodology

### 2.1 Data collection

Three California IOUs (PG&E, SCE, SDG&E) provided:

- Descriptive/demographic data for all active 2019 accounts (tariff,
  building type, location, DERs, EV indicators, DR program enrollment).
- Hourly AMI interval data for a ~3% sample by count, selected to be
  representative by sector/size/utility: **411,000 of 13.6 million
  accounts**, covering **66.1 of 188 TWh (35%) of total load** (sampling
  favored large and non-residential accounts, which is why the energy
  share sampled is much higher than the account share).
- 2018–2019 interval data, PSPS event timing/accounts, and CA DGStats PV
  identifiers for BTM PV correction.

Accounts with anomalously large net exports inconsistent with normal
distributed PV ("Large Generators") were excluded from clustering
entirely and do not appear anywhere in the anonymized public dataset.

### 2.2 Clustering ("LBNL-Load")

Customers are grouped by a multi-step hierarchical process:

1. **Level 1a** — each customer's 365 daily load shapes (normalized by
   daily total) are K-means clustered into prototypical 24-hour shapes.
2. **Level 1b** — prototypical shapes are qualitatively grouped into
   "superclusters" by peak count/timing/width (e.g. `NitePeak`, `DayEve`).
   This yielded **9 residential and 7 commercial** superclusters
   (load-shape clustering was not performed for ag/industrial/other,
   given smaller, more heterogeneous samples).
3. **Level 2** — customers are K-means clustered again, this time on the
   frequency with which each supercluster appears across their year —
   producing the `lshp` values you see in cluster names.
4. **Level 3 (final clustering)** — customers are split hierarchically, in
   this fixed order: **sector → utility → building type → size category →
   climate region → CARE status → LCA → load-shape cluster → kWh
   quintile** (each customer's climate region is first assigned from CEC
   Title 24 zones: Marine = zones 1–6, Hot-dry = 7–15, Cold = 16).

At each level, categories with too few customers are merged with a
*defined* neighboring category (e.g. `Kern+Fresno`, `Flat+LongDay`); if
merging two isn't enough, the whole level collapses to `all...` (e.g.
`allLCA`). The kWh-quintile step itself cascades quintile → quartile →
half → single bin as needed. This is why cluster names contain combined
or `all`-prefixed segments, and why cluster counts/schemas vary by
segment population.

This process produced **5,422 base clusters**. Forecast-year scenarios add
12 more (medium/heavy-duty EV fleet clusters, generated only for forecast
years — see §3), for **5,434**.

### 2.3 End-use disaggregation

- **Heating/cooling**: a change-point regression separates each customer's
  load into temperature-independent, heating, and cooling components,
  fit separately by season/day-type/time-of-day, with heating/cooling
  changepoints swept 50–90°F. **Caveat**: this method attributes *all*
  temperature-correlated load to space conditioning; other loads that
  vary seasonally (e.g. pool pumps, water heating) can leak into these
  two columns to some degree.
- **Appliance end-uses**: allocated using saturation/consumption data from
  RASS 2019 (residential), CEUS/CCSS (commercial), and MECS (industrial/
  agricultural NAICS-code mapping), combined with ADM-developed end-use
  load shapes.
- **EV charging**: Level 1 (120V) and Level 2 (240V) charging disaggregated
  via EVI-Pro load shapes, calibrated to RASS EV data assuming 35 miles/day
  of travel; commercial EV load splits into private (partial workplace
  charging) and commercial-fleet charging using a CEC-derived 5.8:1 ratio.
- **PV correction / PSPS correction**: BTM PV output is estimated and
  backed out of raw meter data first; load during 2019 Public Safety
  Power Shutoff events is reconstructed from similar non-PSPS hours.

### 2.4 Forecasting

Starting from the disaggregated 2019 cluster shapes:

1. **2019 → 2025/2030**: scaled using the CEC 2021 IEPR "Mid Demand"
   baseline forecast (population, industry, LDEV growth, rooftop PV).
2. **EE/FS impacts, through 2030**: end-use-level load-saving (EE) and
   load-adding (FS/electrification) factors from the CPUC's 2021 EE
   Potential & Goals study, using the **AAEE/AAFS "Scenario 3" ("Mid-Mid")**
   trajectory — the same trajectory used in the CEC's 2021 IEPR.
3. **2030 → 2050**: extended using the E3 PATHWAYS model's **"High
   Biofuels"** scenario (as used for CA's 2021 SB100 Joint Agency Report /
   ongoing IRP modeling). **EE impacts are held fixed at their 2030 level**
   through 2050 — only electrification/other PATHWAYS-driven growth
   continues evolving after 2030.
4. **MHDEV (medium/heavy-duty EV fleet) clusters** are newly generated for
   every forecast year using LBNL's HEVI-LOAD model — they don't exist in
   the 2019 baseline at all.

No alternative demand-growth trajectories were modeled — "Mid Demand" /
Scenario 3 Mid-Mid is the only trajectory in this dataset.

### 2.5 Weather scenarios

Two synthetic weather years were constructed from 20 years of NOAA
station data preceding 2019, ranked by cooling degree-days per station:

- **1-in-2** — the median year (a "typical" year).
- **1-in-10** — the 90th-percentile year (an "extreme" year). The
  underlying study uses 1-in-10 as its primary/reference scenario
  throughout, reflecting the increasing frequency of extreme weather.

The reconstructed weather is applied to the *same* 2019 customer base via
the temperature-response model described in §2.3.

---

## 3. Scenario naming

`anonymized-<weather>-<trajectory>-<year>`, e.g.
`anonymized-1in2-midDemand-with_EE_FS-2025`.

| Component | Values | Meaning |
|---|---|---|
| weather | `1in2`, `1in10`, `actual` | See §2.5. `actual` (2019 only) is the literal observed 2019 weather, not a reconstructed year. |
| trajectory | `midDemand-with_EE_FS` vs. `noForecast-no_EE-no_FS` | Forecast (EE+FS applied, §2.4) vs. baseline (raw disaggregated 2019 data, no forecasting at all) |
| year | `2019`, `2025`, `2030`, `2040`, `2050` | Base year (baseline only) or forecast year |

**Baseline and forecast scenarios are not directly comparable as raw
files**: baseline has 5,422 clusters and lacks the `*_count` fleet-vehicle
columns and MHDEV clusters entirely; forecast scenarios have 5,434 and
include them. Cluster *names* are otherwise identical within each family
(verified: all 8 forecast scenarios share the exact same 5,434 names; all
3 baseline scenarios share the exact same 5,422), but the same cluster
name can carry different penetration/consumption values across scenarios
(e.g. across weather years or forecast years) — scenario is part of a
cluster's identity, not just its name.

---

## 4. Known data caveats

- **PG&E residential clusters cannot be split into single-family vs.
  multi-family** — PG&E did not supply that field to the study (SCE and
  SDG&E did). PG&E `building_type` is only `master_mtr` (master-metered
  buildings) vs. `res_unknown` (everything else).
- **`master_mtr`** = a multifamily building on one shared, landlord-paid
  utility meter. Because the whole building is one account, it can't be
  disaggregated to appliance-level end-uses the way an individually
  metered home can — hence the coarse `cooling`/`heating`/`other`-only
  schema for this building type.
- **Large exporting accounts are excluded from the dataset entirely**
  (§2.1) — not folded into any cluster, just absent.
- **Cooling/heating may include some non-HVAC seasonal load** (§2.3).
- Every scenario file has exactly 8760 rows regardless of calendar year
  (including leap years) — this is a normalized synthetic year, not a
  literal calendar mapping with DST transitions.
