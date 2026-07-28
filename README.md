# North American & European CO₂ Facility, Pipeline & Storage Map

An interactive [Folium](https://python-visualization.github.io/folium/)/Leaflet map of industrial CO₂-emitting facilities across **Canada, the United States, and Europe**, built for carbon-capture opportunity screening. Facilities are plotted as points sized by emissions and colored by tier, sector, or region, with multi-dimensional filtering, company/facility search, a radius proximity tool, a fully tunable **strandedness** screen, and a live **emissions-breakdown** panel. The map overlays **CO₂ pipelines**, **geological storage layers**, **proposed CO₂ marine terminals**, and a merged **CO₂ storage & capture projects** layer spanning North America and Europe.

The map is generated from a Google Colab notebook (`canada_emissions_map_clean.ipynb`) and saved as a single self-contained `index.html`, published to GitHub Pages.

---

## Contents

- [Quick start](#quick-start)
- [Facility data](#facility-data)
- [Pipeline data](#pipeline-data)
- [Storage-geology data](#storage-geology-data)
- [CO₂ storage & capture projects (US + Canada + Europe)](#co2-storage--capture-projects-us--canada--europe)
- [Marine terminals](#marine-terminals)
- [Territory tagging](#territory-tagging)
- [Distance computations](#distance-computations)
- [Strandedness screen (tunable)](#strandedness-screen-tunable)
- [Emissions-breakdown panel](#emissions-breakdown-panel)
- [Map controls & interaction](#map-controls--interaction)
- [Automated publishing to GitHub Pages](#automated-publishing-to-github-pages)
- [Known limitations & caveats](#known-limitations--caveats)
- [Data provenance summary](#data-provenance-summary)
- [Attribution](#attribution)

---

## Quick start

1. Open the notebook in Google Colab.
2. Ensure all source files are in your Drive folder (default `DATA = /content/drive/MyDrive/Data`).
3. Run the cells **top to bottom**. Layers that fetch from live government servers (marked *"DON'T RE-RUN"*) only need to run once — afterwards they read the saved GeoJSONs.
4. The map cell writes `index.html`.
5. Run the **publish cell** to push `index.html` to GitHub Pages (see [Automated publishing](#automated-publishing-to-github-pages)).

> **Cell order matters.** The map cell computes all facility distances and strandedness values at the top, then builds the marker array, filters, and overlays. The consolidated control (`FilterMap`) is added **last**, after every pipeline / geology / point-layer group exists — this ordering is required.

> **Kernel-restart safety.** The map cell reads in-memory GeoDataFrames produced by the "run once" fetch cells. After a kernel restart those variables are gone; the map cell defensively reloads each layer from its saved GeoJSON if the in-memory variable is absent. A diagnostic line prints the storage/capture point count and region breakdown on each run, so a stale or partial file load is visible immediately.

---

## Facility data

Three government emissions programs are merged into `fac_low` with a common schema (`Facility, Company, Latitude, Longitude, NAICS, CO2_tonnes, Country`):

- **Canada** — ECCC GHGRP 2024 (`canada1_2024_classified.csv`); the only region with operating-company names.
- **United States** — EPA GHGRP 2023 (`ghgp_data_2023.xlsx`, *Direct Emitters*, `skiprows=3`).
- **Europe** — EEA E-PRTR 2023 (`F1_4_Air_Releases_Facilities.csv`).

Facilities are sized by CO₂ (log scale) and recolorable by CO₂ emission tier, sector (17 categories), or region.

---

## Pipeline data

CO₂ pipelines are a single **"CO₂ pipelines"** control section with four status sub-toggles and matching legend swatches (Active = solid green, Proposed = dashed blue, Cancelled = light grey, Discontinued = medium grey):

- **Alberta** (`Pipelines_SHP/`) — Government of Alberta, filtered to CO₂; survey-grade.
- **US CCS shapefiles** (`CCS_Pipelines-selected/`) — TX / WY (Active), WY corridor / Summit / OH-WV-PA (Proposed), Navigator Heartland Greenway (Cancelled); survey-grade.
- **US CO₂ pipelines — NETL/PHMSA** (`netl_co2_pipelines.geojson`) — DOE/NETL, digitized from PHMSA NPMS, clipped to Gulf Coast and the Dakotas.

---

## Storage-geology data

All geology layers render as **single dissolved, flat-fill polygons** (`unary_union` with `buffer(0)` + `make_valid` repair; one flat color per layer, `fillOpacity 0.45`), each toggleable (off by default) with a legend swatch:

- **Saline formations** — NatCarb (`natcarb_saline.geojson`, NATCARB v1502) + EU CO2StoP (`eu_co2stop_storage.geojson`), combined.
- **Coal seams** — NatCarb (`natcarb_coal.geojson`).
- **Sedimentary basins** — NETL Atlas V (`na_sed_basins.geojson`).
- **Basalt & mafic volcanic rocks** — US NETL basalt (`basalt_formations.geojson`) + Canada Wheeler mafic-volcanic proxy (`ca_mafic_volcanic.geojson`, GSC Map 1860A, 1:5M), combined. Basalt matters because it enables CO₂ **mineralization**.

---

## CO₂ storage & capture projects (US + Canada + Europe)

A single merged layer of **568 storage & capture projects** across all three regions, built from four sources into one file (`co2_storage_capture.geojson`):

| Source | Region | Count |
|--------|--------|-------|
| CATF CCUS Database — US | US | 254 |
| CATF CCUS Database — Europe | Europe | 231 |
| CCS Knowledge Centre + AER tenure | Canada | 83 |
| EPA UIC Class VI tracker (deduped into CATF where matched) | US | (18 new + 2 merged) |

**This layer is its own top-level control section** (not inside "Map layers"), structured like the pipeline section: a master toggle with **Active** and **Proposed** sub-toggles, each with its triangle swatch. Both default **off** — nothing shows until checked.

- **Active** = operational or under construction → **dark-blue** triangles (`#0b3d91`).
- **Proposed** = in development / planned / in review → **lighter-blue** triangles (`#74b3f0`).
- Split totals: **61 Active, 507 Proposed**.

**Tooltips** carry as much as each source provides: project name, use type (dedicated saline / EOR / utilization / capture etc.), status, storage type, subsector, operator, location + country, capacity, operational or announced year, a descriptive note, the data source, and a reference link. Points with only approximate coordinates show a **red "⚠ Approximate location" warning** (see caveats).

**Coordinate accuracy by source:**
- **CATF (US + Europe):** approximate site coordinates supplied by CATF.
- **Canada:** 21 Alberta storage hubs upgraded to **coordinate-accurate AER carbon-sequestration tenure centroids** (with the target geological formation named in the tooltip); the remaining ~62 Canadian projects are **town-level** (geocoded from the named town), flagged approximate.
- **EPA (US in-review):** **county-level** centroids only — EPA publishes no site coordinates — flagged approximate.

---

## Marine terminals

`co2_marine_terminals_approx.geojson` — hand-built proposed CO₂ shipping/discharge terminals (T-RICH Port Tampa Bay, LBC Baton Rouge/Geismar, an Alaska study placeholder). Rendered as **pink hollow diamonds**, its own toggle in "Map layers," off by default, with `source_url` per point. All are proposed/study-stage; no operational CO₂ marine terminals exist in the dataset yet. These feed the marine-terminal strandedness distance.

---

## Territory tagging

Each facility is tagged to a **province/state (North America)** or **country (Europe)** via a per-region nearest-match spatial join against Natural Earth 10m Admin-1 boundaries (`ne_10m_admin_1_states_provinces.shp`). Matching is constrained by the facility's own country (US facilities match only US polygons, etc.), with a 50 km snap tolerance, so coordinates never borrow a foreign polygon (this fixes US↔Canada cross-assignment and Mexican-border spillover). Facilities beyond tolerance of any same-country polygon are tagged "Unknown."

---

## Distance computations

All distances are **straight-line great-circle approximations** (spatial-index nearest + haversine); polygon targets return 0 km when the facility is inside. Screening-grade, not routed. Raw values ship into the browser so the strandedness screen can threshold them live:

- **Pipeline** — nearest Active (`pipe_active_km`) and nearest Proposed (`pipe_proposed_km`).
- **Saline formation** — `saline_km`.
- **Storage/capture** — nearest Active (`store_active_km`) and nearest Proposed (`store_prop_km`), computed against the full merged NA+EU file.
- **Marine terminal** — nearest Existing (`term_exist_km`) and nearest Proposed (`term_prop_km`).
- **Coast** — `coast_km`, distance to the Natural Earth 10m **Ocean polygon** (Great Lakes and inland waters excluded by design — only genuine sea coasts count).

---

## Strandedness screen (tunable)

Six independent, live-tunable sections under the "Strandedness" group. Each is disabled until you enable it; enabled sections combine with **AND**. Every section has a **slider + synced editable number box** (type a value beyond the slider max to extend it) and a **"less than" / "more than"** toggle:

1. **Distance to pipeline** — Active / Proposed checkboxes (either or both; uses the nearest of the selected).
2. **CO₂ produced** — amount threshold (defaults to "more than").
3. **Distance to saline formation** — distance threshold.
4. **Distance to CO₂ storage/capture project** — Active / Proposed checkboxes (Active = operational/under construction, Proposed = in development). Covers all three regions.
5. **Distance to coast (marine shipping)** — distance threshold to the ocean.
6. **Distance to marine shipping terminal** — Existing / Proposed checkboxes (Proposed on by default; no existing terminals in the dataset yet, so the Existing option currently matches nothing).

All thresholds are user-set at run time — there are no frozen buckets. A **"↺ Reset filters"** button restores every filter, slider, toggle, the search box, color-by, and the breakdown-panel state to defaults.

---

## Emissions-breakdown panel

A live panel (bottom-right) showing total CO₂ currently on the map, respecting every active filter and search. It nests **Region → Territory → Sector**: click a region (Canada / US / Europe) to expand its territories sorted by CO₂, click a territory to expand its sector breakdown. Recomputes instantly on every filter change; collapsible.

---

## Map controls & interaction

- **Consolidated control** (top-right, single scrollable panel): Reset button, search (facility OR company), Color-by selector, primary filters (CO₂ scale, Sectors, Region), the six Strandedness sections, the CO₂ pipelines section, the CO₂ storage & capture section, the Map layers section (geology + terminals), and the Pin distance tool. Legend swatches throughout.
- **Facility markers** — popups show facility, operator (Canada), NAICS, CO₂, sector, region, territory, and nearest active pipeline.
- **Pin distance tool** — concentric rings (5/50/100/200/300 km) with per-band CO₂ + sector breakdown, View-list, and CSV export.
- **Last-updated watermark** (build-time UTC, bottom-left).

---

## Automated publishing to GitHub Pages

The map self-publishes from Colab: a publish cell pushes the freshly-built `index.html` to `Backwater-Carbon-Solutions/emissions_mapping` (main branch, repo root) via the GitHub Contents API. Setup:

1. **Fine-grained personal access token** — resource owner `Backwater-Carbon-Solutions`, scoped to the `emissions_mapping` repo only, **Contents: Read and write**. The token must be **approved by an org owner** (org Settings → Third-party Access → Personal access tokens) before writes succeed — a pending token gives 200 on reads but 403 on writes.
2. **Colab secret** — store the token as `GH_TOKEN` (key icon in the Colab sidebar).
3. Run the publish cell after the map cell. Live within ~1–2 min at `https://backwater-carbon-solutions.github.io/emissions_mapping/`.

Only `index.html` is pushed — source data never leaves Drive. An empty **`.nojekyll`** file at the repo root prevents GitHub's Jekyll processing from breaking the self-contained Folium HTML (this was the fix for an earlier blank-page issue). Everything in this pipeline (token, API, Pages, Colab) is free at this scale.

---

## Known limitations & caveats

- **Reporting thresholds.** All three emissions programs only require facilities above a size threshold to report; smaller emitters are absent.
- **Company/operator — Canada only.**
- **Straight-line distances.** All metrics and strandedness values are great-circle approximations, not routed.
- **Marine "coast" = ocean only.** Great Lakes and inland waters excluded by design.
- **Canadian basalt is a mafic-volcanic proxy** (1:5M); US basalt is basalt-specific. The combined layer mixes resolutions.
- **European saline = CO2StoP** (saline + hydrocarbon, 2012–2014 public subset; older than the 2025 EGDI atlas).
- **Storage & capture layer mixes storage and capture.** It includes dedicated storage, EOR, utilization, and pure capture facilities (power, cement, DAC). The tooltip's use-type field is the distinguisher; all points share the same triangle (colored by Active/Proposed status only).
- **Approximate coordinates (80 points, flagged in-tooltip):** the ~62 town-level Canadian projects and the county-level EPA in-review projects sit at town/county centroids, not exact sites. CATF and AER-sourced points are more precise. The 21 Alberta AER-tenure points are coordinate-accurate.
- **Snapshots.** CATF, CCS Knowledge, and EPA trackers update periodically; this is a dated snapshot, likely behind the newest filings. It is project-level, not well-level, and may lag primacy states. For a definitive US regulatory inventory, cross-check the EPA UIC Class VI dashboard and the four primacy-state registries (ND, WY, LA, WV).
- **Geology is overview-scale** — continental screening, not site selection. "0 km to saline" means "over a mapped saline basin," not confirmed injectable.

---

## Data provenance summary

| Layer | Source | Precision |
|-------|--------|-----------|
| Canada facilities | ECCC GHGRP 2024 | Reported |
| US facilities | EPA GHGRP 2023 | Reported |
| Europe facilities | EEA E-PRTR 2023 | Reported |
| Alberta / US CCS pipelines | Gov. Alberta / state & project GIS | Survey-grade |
| US CO₂ pipelines (Gulf/Dakotas) | DOE/NETL via PHMSA NPMS | Digitized |
| Saline / coal | DOE/NETL NatCarb v1502 | Overview |
| Saline (Europe) | EU JRC CO2StoP (2012–2014) | Overview, public subset |
| Sedimentary basins / US basalt | DOE/NETL Atlas V | Overview |
| Canada mafic-volcanic | NRCan GSC Map 1860A (Wheeler) | 1:5M, basalt proxy |
| Ocean (coast distance) | Natural Earth 10m Ocean | Overview |
| Territory boundaries | Natural Earth 10m Admin-1 | Overview |
| Storage & capture — US | Clean Air Task Force CCUS Database | Project-level snapshot |
| Storage & capture — Europe | Clean Air Task Force CCUS Database | Project-level snapshot |
| Storage & capture — Canada | International CCS Knowledge Centre + Alberta Energy Regulator tenure | Project snapshot; AB hubs coordinate-accurate, rest town-level |
| US Class VI (in review) | EPA UIC Class VI permit tracker | County-level |
| Marine terminals | Project announcements | Hand-built, approximate |

---

## Attribution

Canadian CO₂ storage & capture project data is **adapted from The International CCS Knowledge Centre** (ccsknowledge.com), used under its Open License Agreement. *This does not constitute an endorsement by the Knowledge Centre of this application of its Licensed Materials.* Alberta storage-hub coordinates are derived from the **Alberta Energy Regulator** carbon-sequestration tenure dataset. US and European project data are from the **Clean Air Task Force CCUS Database**, and US in-review storage from the **US EPA UIC Class VI** program.

*Built for carbon-capture opportunity screening. Emissions figures are as-reported to government programs and inherit their coverage, thresholds, and gaps. Pipeline, geology, and project layers combine survey-grade, digitized, overview-scale, aggregated-database, and hand-built sources as noted. All distance and strandedness metrics are straight-line approximations for screening, not routed or survey-grade.*
