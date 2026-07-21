# North American & European CO₂ Facility, Pipeline & Storage Map

An interactive [Folium](https://python-visualization.github.io/folium/)/Leaflet map of industrial CO₂-emitting facilities across **Canada, the United States, and Europe**, built for carbon-capture opportunity screening. Facilities are plotted as points sized by emissions and colored by tier, sector, or region, with multi-dimensional filtering, company/facility search, a radius proximity tool, and a multi-factor **strandedness** screen. The map overlays **CO₂ pipelines**, **geological storage layers** (saline formations, coal seams, sedimentary basins, basalt & mafic volcanic rocks), and **proposed/existing CO₂ transport & storage infrastructure** (marine terminals and the CATF national CCUS storage-project database).

The map is generated from a Google Colab notebook (`canada_emissions_map_clean.ipynb`) and saved as a single self-contained `index.html` that can be opened locally or hosted (e.g. GitHub Pages).

---

## Contents

- [Quick start](#quick-start)
- [Facility data — sources & acquisition](#facility-data--sources--acquisition)
- [Pipeline data — sources & acquisition](#pipeline-data--sources--acquisition)
- [Storage-geology data — sources & acquisition](#storage-geology-data--sources--acquisition)
- [CO₂ infrastructure points (storage projects & marine terminals)](#co2-infrastructure-points-storage-projects--marine-terminals)
- [Distance computations](#distance-computations)
- [Strandedness screen](#strandedness-screen)
- [Map features & interaction](#map-features--interaction)
- [Sector categories & tiers](#sector-categories--tiers)
- [Known limitations & caveats](#known-limitations--caveats)
- [Data provenance summary](#data-provenance-summary)
- [Deployment notes](#deployment-notes)

---

## Quick start

1. Open the notebook in Google Colab.
2. Ensure all source files are in your Drive folder (default `DATA = /content/drive/MyDrive/Data`).
3. Run the cells **top to bottom**. Layers that fetch from live government servers (marked *"DON'T RE-RUN"*) only need to run once — after that they read the saved GeoJSONs.
4. The final cell writes `index.html`.

> **Cell order matters.** The map cell computes all facility distances and strandedness factors at the top, then builds the marker array, filters, and overlays. The consolidated control (`FilterMap`) is added **last**, after every pipeline/geology/point layer group exists — this ordering is required or the combined sidebar will error.

> **Kernel-restart safety.** The map cell reads in-memory GeoDataFrames produced by the "run once" fetch cells. After a kernel restart those variables are gone, and a missing layer is **silently skipped** (this previously caused the Canadian basalt layer to vanish). The map cell defensively reloads each layer from its saved GeoJSON if the in-memory variable is absent. If you add a new layer, add the same guarded reload.

---

## Facility data — sources & acquisition

The facility layer merges three government emissions programs into `fac_low` with a common schema: `Facility, Company, Latitude, Longitude, NAICS, CO2_tonnes, Country`.

### Canada — ECCC GHGRP
- **File:** `canada1_2024_classified.csv` — ECCC Greenhouse Gas Reporting Program, 2024. Pre-classified to one record per facility; **only region carrying an operating-company name**.

### United States — EPA GHGRP (FLIGHT)
- **File:** `ghgp_data_2023.xlsx`, sheet *Direct Emitters* (`skiprows=3`) — EPA GHGRP, 2023. No operating-company field.

### Europe — E-PRTR
- **File:** `F1_4_Air_Releases_Facilities.csv` — EEA European Pollutant Release and Transfer Register, 2023. Facility name only.

---

## Pipeline data — sources & acquisition

CO₂ pipelines are combined into a single **"CO₂ pipelines"** section of the control with four status sub-toggles. Legend swatches (short colored lines; dashed for Proposed) appear next to each. Colors: Active = solid green, Proposed = dashed blue, Cancelled = light grey, Discontinued = medium grey.

- **Alberta** (`Pipelines_SHP/`, GCS_NAD83): Government of Alberta pipeline shapefile, filtered to CO₂. Per-segment status: operating → Active, else → Discontinued. Survey-grade.
- **US CCS shapefiles** (`CCS_Pipelines-selected/`, six regional files): State/regulatory & project GIS. Texas (Active), Wyoming–Denbury (Active), Wyoming corridor (Proposed), Summit Midwest Carbon Express & Project A (Proposed), Navigator Heartland Greenway (Cancelled), OH/WV/PA routes (Proposed). Survey-grade.
- **US CO₂ pipelines — NETL/PHMSA** (`netl_co2_pipelines.geojson`): DOE/NETL FeatureServer, digitized from the PHMSA National Pipeline Mapping System. Clipped to Gulf Coast (`lon ≥ -96 & lat ≤ 33.5`) and Dakotas (`lat ≥ 45 & -104 ≤ lon ≤ -96`) to fill the gap left by the survey-grade shapefiles. Status = Active. **Replaced** the earlier hand-traced Denbury/Souris approximations.

---

## Storage-geology data — sources & acquisition

All geology layers render as **single dissolved, flat-fill polygons** — overlapping source polygons are merged with `unary_union` (with `buffer(0)` + `make_valid` repair for invalid geometries) so there is no darkening where polygons stack, and outlines are removed (one flat color per layer, `fillOpacity 0.45`). Each is toggleable (off by default) and carries a matching legend swatch in the control.

### Saline formations (North America + Europe, ONE combined layer)
- **North America — NatCarb** (`natcarb_saline.geojson`, DOE/NETL NATCARB v1502, set to EPSG:5070 → 4326).
- **Europe — CO2StoP** (`eu_co2stop_storage.geojson`, EU JRC "European CO₂ storage database" open-format KML, `StorageUnits_March13.kml`, 418 storage units). **Note:** CO2StoP units are saline aquifers *plus* some hydrocarbon-field storage; "Saline formations" is a screening simplification for the European portion, and this is the 2012–2014 public subset (older than the 2025 GSEU/EGDI atlas, which was not obtainable as a direct download).

### Coal seams — NatCarb (`natcarb_coal.geojson`, NATCARB v1502).

### Sedimentary basins — NETL Atlas V (`na_sed_basins.geojson`, `North_America_Sedimentary_Basins` REST service, 2015). US + Canada.

### Basalt & mafic volcanic rocks (US + Canada, ONE combined layer)
- **US — NETL basalt** (`basalt_formations.geojson`, `Basalt_Formations_AtlasV_2025` REST service): basalt-specific (primarily Columbia River Basalt Group).
- **Canada — Wheeler mafic volcanic** (`ca_mafic_volcanic.geojson`, NRCan `geological_map_canada_wheeler_en` REST service, GSC Map 1860A, Wheeler et al. 1996, 1:5M). Filtered server-side to `SUBRXTP LIKE '%mafic volcanic%'`. The 1:5M map has no basalt-specific class, so **mafic volcanic rocks are used as a basalt proxy** (accepted simplification). Basalt-family rock matters because it enables CO₂ **mineralization** (permanent conversion to carbonate).
- **Note:** this combined layer reloads `ca_mafic` from `ca_mafic_volcanic.geojson` if not in memory — a missing fetch previously made the Canadian half silently disappear.

---

## CO₂ infrastructure points (storage projects & marine terminals)

Two distinct point layers, each with its own legend swatch. Both are **snapshots**; verify against primary sources before decisions.

### CO₂ storage projects (CATF, US) — blue down-triangles
- **File:** `co2_storage_sites.geojson`, derived from the **Clean Air Task Force CCUS Database (US)** — 263 total CCUS projects, filtered to the **dedicated-storage-sector projects**, of which **36 have publishable coordinates** and are mapped (national: LA, TX, CA, AL, WY, CO, IN, IL, OR, and an OH/WV/PA project).
- **Fields (in popup):** project name, entities/operator, state, location, storage type, status, capacity, year announced/operational, and the per-project **CATF reference link**.
- **Status:** nearly all "In Development" (announced/permitting); styled/labeled by status so "In Development" is not read as "operating."
- **This is the single storage-project layer.** The earlier hand-built Gulf storage points were **retired** — every hand-built point that could be verified (Bayou Bend, Libra, Virgo, Harvest Bend, Donaldsonville) already exists in CATF, so keeping them would have duplicated CATF with less-authoritative coordinates. Per the "drop unless verified AND not already in CATF" rule, they were removed rather than kept as unverifiable duplicates.
- **De-duplication:** the Tampa hub (**T-RICH**) appears in CATF tagged as "Storage," but it is mapped here as a **marine terminal** (see below) because its distinctive role is CO₂ shipping. It is **excluded** from this storage layer to avoid a double marker at Port Tampa Bay.

### Proposed CO₂ marine terminals — pink hollow-diamonds
- **File:** `co2_marine_terminals_approx.geojson` (3 points, hand-built, `source_url` each). T-RICH (Port Tampa Bay loading hub), LBC Baton Rouge/Geismar discharge terminal (Sunshine LA, verified), Alaska import-terminal study (placeholder).
- **Not in CATF** as terminals (CATF has no marine-terminal layer), so this stays a separate hand-built layer. Feeds the marine-shipping strandedness factor via `term_km`.
- **Terminal vs. storage distinction:** integrated "hubs" like T-RICH blur the line (they bundle transport + storage). This map places a hub as a *terminal* when its role is where CO₂ enters shipping, and as *storage* when its role is injection/sequestration. T-RICH is treated as a terminal; its CATF "Storage" sector tag is not used, to avoid duplication.

---

## Distance computations

All distances are **straight-line great-circle approximations** (spatial-index nearest + haversine). Polygon targets (saline, ocean) return 0 km when the facility is inside. Screening-grade, not routed.

- **`pipe_km`** — nearest **active** CO₂ pipeline (shown in facility popups).
- **`pipe_prop_km`** — nearest **proposed** pipeline.
- **`pipe_ap_km`** — nearest active-or-proposed pipeline.
- **`saline_km`** — nearest **saline formation** (NatCarb + CO2StoP combined).
- **`term_km`** — nearest **proposed CO₂ marine terminal**.
- **`coast_km`** — nearest **ocean** (Natural Earth 10m **Ocean polygon**, not a coastline, so inland shores like the Great Lakes are excluded — only genuine sea coasts count).

> **Storage-site distance (`store_km`) is no longer computed** — storage-site proximity was removed from the marine-shipping factor (see below), so the metric is not needed.

---

## Strandedness screen

Four **independent, separately-filterable factors** nested under a "Strandedness" group. Each factor filters the whole map; within a factor, no boxes = all, some boxes = narrow to those; across factors the logic is AND.

1. **Distance to pipeline** — Active/Proposed × near/far (4 bands).
2. **CO₂ produced** — > 200k / < 200k t/yr.
3. **Distance to saline formation** — < 200 km / > 200 km.
4. **Marine shipping** — not-stranded/stranded, using **only** marine options (onshore pipeline proximity and storage-site proximity both excluded). Not stranded if **within 200 km of an existing CO₂ shipping terminal**, **OR** within 200 km of the **coast (ocean)** AND producing **≥ 500k t/yr**. An inline caption states this in the filter.

> **Change history:** the marine factor previously also counted proximity to proposed storage sites; this was removed so the factor reflects only genuine marine-shipping viability (a terminal to ship from, or the scale + coastal access to build one).

---

## Map features & interaction

- **Consolidated control** (top-right, single scrollable panel, "Map Filters & Layers"): search (facility OR company), Color-by selector, primary filters (CO₂ scale, Sectors, Region), the nested Strandedness factors, the **CO₂ pipelines** master+status section, the **Map layers** section (geology + infrastructure), and the Pin distance tool — all previously separate controls, now merged into one panel. Legend swatches appear next to every pipeline status and map layer.
- **Facility markers** — sized by CO₂ (log scale), recolorable by the Color-by dimension; popups show facility, operator (Canada), NAICS, CO₂, sector, region, nearest active pipeline.
- **Search, Pin distance tool** (concentric rings 5/50/100/200/300 km, per-band CO₂ + sector breakdown, View list / Download CSV).
- **Point-layer popups** carry `source_url` links (CATF reference, or the hand-built terminal source).
- **Last-updated watermark** (build-time UTC, bottom-left).

---

## Sector categories & tiers

17 sector categories assigned from NAICS/industry descriptions with correction rules (Cement/lime; Power; Oil & gas; Refining & petrochemicals; Industrial gases/H₂/ammonia; Other chemicals; Steel & metal; Mining; Pulp/paper/wood; Manufacturing food & drink; Manufacturing other; Agriculture; Waste & wastewater; Buildings/transport/services; Fermentation; Bioethanol; Biogas). Emission tiers: grey "0/not reported", five viridis middle tiers, a "250,000+ t" top tier.

---

## Known limitations & caveats

- **Reporting thresholds.** All three programs only require facilities above a size threshold to report; smaller emitters are absent.
- **Company/operator — Canada only.**
- **Straight-line distances.** All metrics and strandedness factors are great-circle approximations, not routed.
- **Marine "coast" = ocean only.** Great Lakes and inland waters excluded by design.
- **Canadian basalt is a mafic-volcanic proxy** (1:5M); US basalt is basalt-specific. Combined layer mixes resolutions.
- **European saline = CO2StoP** (saline + hydrocarbon, 2012–2014 public subset; older than the 2025 EGDI atlas).
- **CATF storage layer caveats:** it is (1) a **dated snapshot** (CATF updates periodically; likely weeks-to-months behind the newest EPA/state filings); (2) tracked at the **project level**, not the well level (one project = one point even if it spans many Class VI wells — so it will not match EPA's 175+ well-application count, a different unit); (3) potentially **lagging the primacy states** (ND, WY, LA, WV run their own programs; recent/small state-permitted projects may be missing); (4) **EOR under-counted** (legacy CO₂-EOR storage not framed as CCUS projects is absent); and (5) **coordinate-incomplete** — only 36 of the storage-sector projects have publishable coordinates (after excluding the duplicate T-RICH), so a number of real projects (e.g. Roughrider ND, Bluebonnet & Milestone TX, several LA hubs) are in the source but not on the map. For a definitive regulatory inventory, cross-check the EPA UIC Class VI dashboard plus the four primacy-state registries.
- **Marine terminals are hand-built/approximate** (port-level), with `source_url`. Alaska is a placeholder (study, no site).
- **Geology is overview-scale** (1:5M or coarser) — continental screening, not site selection. "0 km to saline" means "over a mapped saline basin," not confirmed injectable.

---

## Data provenance summary

| Layer | Source | Format / access | Precision |
|-------|--------|-----------------|-----------|
| Canada facilities | ECCC GHGRP 2024 | CSV (pre-classified) | Reported |
| US facilities | EPA GHGRP 2023 | XLSX (FLIGHT) | Reported |
| Europe facilities | EEA E-PRTR 2023 | CSV | Reported |
| Alberta pipelines | Government of Alberta | Shapefile | Survey-grade |
| US CCS pipelines | State/regulatory + project GIS | Shapefiles | Survey-grade |
| US CO₂ pipelines (Gulf/Dakotas) | DOE/NETL, digitized from PHMSA NPMS | ArcGIS REST (GeoJSON) | Digitized |
| Saline formations — N. America | DOE/NETL NatCarb v1502 | File geodatabase | Overview |
| Saline formations — Europe | EU JRC CO2StoP (2012–2014) | Open-format KML | Overview, public subset |
| Coal seams | DOE/NETL NatCarb v1502 | File geodatabase | Overview |
| Sedimentary basins | DOE/NETL Atlas V (2015) | ArcGIS REST (GeoJSON) | Overview |
| Basalt — US | DOE/NETL Atlas V (2015) | ArcGIS REST (GeoJSON) | Overview, basalt-specific |
| Mafic volcanic — Canada | NRCan GSC Map 1860A (Wheeler 1996) | ArcGIS REST (GeoJSON) | 1:5M, basalt proxy |
| Ocean (coast distance) | Natural Earth 10m Ocean | Shapefile | Overview |
| CO₂ storage projects (US) | Clean Air Task Force CCUS Database | CSV (36 mapped) | Project-level snapshot |
| Proposed marine terminals | Project announcements | Hand-built GeoJSON + source_url | Approximate (concept) |

---

## Deployment notes

- **Local vs GitHub Pages.** The map renders locally by double-clicking `index.html`. If it renders locally but is **blank on GitHub Pages**, add an empty **`.nojekyll`** file at the repo root (or `/docs`) — GitHub's default Jekyll processing can break the self-contained Folium HTML. This was the confirmed fix for the blank-page issue.
- **File size** (~8 MB) is well within GitHub Pages limits.
- If still blank after `.nojekyll`, hard-refresh (Ctrl+Shift+R), confirm the Pages branch/folder, and check the browser F12 Console for the specific error.

---

*Built for carbon-capture opportunity screening. Emissions figures are as-reported to government programs (ECCC GHGRP, EPA GHGRP, EU E-PRTR) and inherit their coverage, thresholds, and gaps. Pipeline, geology, and infrastructure layers combine survey-grade, digitized, overview-scale, aggregated-database, and hand-built sources as noted. All distance and strandedness metrics are straight-line approximations for screening, not routed or survey-grade.*
