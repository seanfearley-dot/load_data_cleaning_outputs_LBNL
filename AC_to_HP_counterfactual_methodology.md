# AC-to-Heat-Pump Counterfactual — Methodology

## Dataset overview

- **FLEXvalue DEER database** (`2021.db`, ~1 GB SQLite, from `flexvalue-public-resources`)
  - `deer_load_shapes` table: wide layout, 8,760 hourly rows × 78 columns — one column per utility/sector/measure normalized load shape (e.g. `PGE_RES_HVAC_EFF_AC`, `PGE_RES_HVAC_EFF_HP`)
  - Each shape sums to ~1.0 across the year (normalized savings/usage profile); no climate-zone dimension — shapes are utility-wide only, not CZ-specific
  - Also contains `acc_electricity`, `acc_electricity_utilities_climate_zones`, `acc_gas` (avoided-cost tables; not used in the final counterfactual)
- **eTRM measure packages** (California's statewide deemed-measure repository, caetrm.com)
  - `SWHC045-06` — "Heat Pump HVAC, Residential, Fuel Substitution" — `permutations.csv` (7,644 rows) + workpaper PDF. Source of the baseline/measure UEC scaling used in the counterfactual.
  - `SWHC049-08` — "Ducted AC and HP HVAC Equipment, Residential" — `permutations.csv` (11,232 rows) + workpaper PDF. Non-fuel-substitution AC-to-AC comparison, used to validate cooling-only efficiency and to isolate scope differences from SWHC045.
- **DNV/CEDARS sizing study** — internal DEER working file (`DEER_EnergyPlus_..._DNV_Sizing_results-summary...xlsx`) documenting the derivation of DEER's EnergyPlus equipment auto-sizing correction factors.
- **RASS** (Residential Appliance Saturation Survey) — user-supplied annual UECs (central AC: 1,217 kWh/yr; heat pump: 2,057 kWh/yr), used in the earliest version of this analysis before the eTRM-based approach replaced it.
- **ART / SmartAC ex-ante program data** — external benchmark (1-in-2 system peak, all-loads average residential customer), used to sanity-check and recalibrate the final system-size (tonnage) assumption.

## DEER measure methodology (SWHC045-06)

**Scenario characterized:**
- Climate Zone CZ04, Building Type SFm (single-family), Measure Application Type NR (Natural Replacement)
- Baseline: "AC and Gas Furnace," Standard Practice case (code-minimum, SEER2 14.3)
- Measure: "SEER2-Rated HP" at SEER2 ≥ 15.2 (code-minimum heat pump tier)

**Simulation basis:**
- EnergyPlus v22.2.0, DEER2024 residential prototypes, CEC CZ2022 weather files
- Baseline building prototypes: `SFm-1 Story-1975-Combi`, `SFm-1 Story-1985-Combi`, `SFm-2 Story-1975-Combi`, `SFm-2 Story-1985-Combi` (blended)
- Baseline UEC captures the *entire* AC-and-furnace system's electric draw, including the indoor blower/air handler (shared between heating and cooling calls) — a wider scope than SWHC049's AC-only baseline, which explicitly excludes blower/air-handler energy as "beyond the scope" of a pure equipment swap. This scope difference is the primary driver of the ~3.4x UEC gap between the two measures for nominally comparable equipment.
- Equipment auto-sized in EnergyPlus, then scaled by an explicit **1.8x sizing factor**
  - Derived from a DNV/CEDARS analysis: EnergyPlus's native, uncorrected autosizer produced capacities far below both a Manual J benchmark (~647 sq ft/ton) and real-world field-installed systems (~510 sq ft/ton), per a 2010–12 CALMAC HVAC Impact Evaluation (WO32) of 50 sites across CZ8–CZ16
  - The empirically-implied correction factor varies by climate zone/building type (1.76x–7x+); **1.8 is a single flat value applied uniformly**, not a CZ-specific fit
  - For CZ04/SFm specifically, the DNV-implied factor is closer to **~2.2x** (1-story 2.55x, 2-story 1.88x) — the baseline may therefore be modeled somewhat undersized relative to DNV's own field-calibrated benchmark
- **Normalized Unit = Cap-Tons**: every UEC figure (814 kWh/yr baseline, 1,460 kWh/yr measure, both per ton) is a *rate per ton of installed cooling capacity*, not a whole-system annual total — confirmed via SWHC049's near-flat UEC across different capacity tiers (234 vs. 241 kWh for `<45` vs. `45-65` kBtu/hr)
- DEER's own hourly-impact convention applies the *measure's* own load shape (`DEER:HVAC_Eff_HP`) to the net annual Electric Savings figure directly, rather than differencing two independently-scaled baseline/measure shapes. This analysis uses the latter (reconstruction) method instead, for transparency; the reconstructed annual total (−1,937.7 kWh at 3 tons) validated closely against eTRM's own reported Electric Savings scaled the same way (−1,926.0 kWh).

**System-size (tonnage) assumption:**
- Neither SWHC045 nor SWHC049's workpaper exposes the exact SFm/CZ04 floor-area-per-ton ratio (both cite an external `DEER_Res_Tables.xlsm` / "Res Key Prototype Values" source not included in either export)
- Initial estimate: 3 tons, based on adjacent reference ratios (DMo 354.9 ft²/ton flat across CZs; MFm/CZ04 676.1 ft²/ton; SWHC049 flat reference 545.5 ft²/ton) applied to a typical ~1,700–1,900 ft² CA home
- **Recalibrated to 2.0–2.25 tons** after back-checking modeled baseline peak demand against ART/SmartAC ex-ante 1-in-2 system peak (all-loads average residential customer): the DEER-floor-area-based 3-ton estimate overstated peak demand by 0.5–1 kW relative to that program benchmark

## Connection to RASS

- RASS is a population survey of actual as-installed equipment and usage, not an equipment or building energy simulation — its UECs are already whole-customer annual totals (not per-ton, unlike the eTRM figures).
- **Initial approach:** DEER's normalized 8760 shapes were scaled directly by RASS UECs (central AC 1,217 kWh/yr; heat pump 2,057 kWh/yr) to build the counterfactual, before eTRM data was available.
- **Key finding — this produced a misleadingly large implied cooling-efficiency improvement:**
  - Full-year comparison: heat pump consumption is ~69% *higher* than the AC baseline — driven by added heating load, not a cooling-efficiency signal (AC is cooling-only; HP covers heating + cooling)
  - Isolated to peak cooling months (Jun–Sep) to exclude heating-season contamination, the RASS pairing still implied a **~50% relative reduction** in cooling-season energy for the heat pump vs. the AC baseline
  - That figure is roughly 2.5x larger than the SEER-rating-implied efficiency gain for a comparable equipment upgrade
- **Root causes identified:**
  - End-use scope mismatch: RASS's AC figure is cooling-only; its heat pump figure bundles heating and cooling, so the comparison was never isolating cooling efficiency in the first place
  - Composition bias: RASS is a survey-based population average, not a controlled comparison — it bakes in differences in home size, insulation, and climate-zone mix between AC-only and heat-pump households that have nothing to do with equipment efficiency
- **Resolution:** a properly-scoped, same-fuel, cooling-only comparison (SWHC049: code-minimum AC → SEER2 ≥ 15.2 replacement, non-fuel-substitution) produced a **22.2%** relative efficiency improvement — consistent with the commonly-cited ~20% SEER-based expectation, and confirming the RASS-implied ~50% figure was an artifact of scope mismatch and population composition, not a real efficiency finding.
