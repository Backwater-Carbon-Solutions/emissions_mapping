# North American & European CO₂ Facility, Pipeline & Storage Map

An interactive [Folium](https://python-visualization.github.io/folium/)/Leaflet map of industrial CO₂-emitting facilities across **Canada, the United States, and Europe**, built for carbon-capture opportunity screening. Facilities are plotted as points sized by emissions and colored by tier, sector, or region, with multi-dimensional filtering, company/facility search, a radius proximity tool, and a multi-factor **strandedness** screen. The map also overlays **CO₂ pipelines**, **geological storage layers** (saline formations, coal seams, sedimentary basins, basalt & mafic volcanic rocks), and **proposed CO₂ transport & storage infrastructure** (marine terminals, Gulf Coast storage sites).

The map is generated from a Google Colab notebook (`canada_emissions_map_clean.ipynb`) and saved as a single self-contained `index.html` that can be opened locally or hosted (e.g. GitHub Pages).

---

## Contents

- [Quick start](#quick-start)
- [Notebook structure](#notebook-structure)
- [Facility data — sources & acquisition](#facility-data--sources--acquisition)
- [Pipeline data — sources & acquisition](#pipeline-data--sources--acquisition)
- [Storage-geology data — sources & acquisition](#storage-geology-data--sources--acquisition)
- [Proposed CO₂ infrastructure (hand-built)](#proposed-co2-infrastructure-hand-built)
- [Distance computations](#distance-computations)
- [Strandedness screen](#strandedness-screen)
- [Map features & interaction](#map-features--interaction)
- [Sector categories & tiers](#sector-categories--tiers)
- [Known limitations & caveats](#known-limitations--caveats)
- [Data provenance summary](#data-provenance-summary)

---

## Quick start

1. Open `canada_emissions_map_clean.ipynb` in Google Colab.
2. Ensure all source files are in your Drive folder (default `DATA = /content/drive/MyDrive/Data`).
3. Run the cells **top to bottom**. The layers that fetch from live government servers (marked *"DON'T RE-RUN"*) only need to run once — after that they read the saved GeoJSONs.
4. The final cell writes `index.html`.

> **Cell order matters.** The map cell computes all facility distances and strandedness factors at the top, then builds the marker array, filters, and overlays. It depends on `fac_low` (facilities), the pipeline objects, the storage-geology layers, the coastline/ocean file, and the proposed-infrastructure GeoJSONs all existing first. Run sequentially after any change.

---

## Notebook structure

| Cell | Purpose |
|------|---------|
| **Setup** | Imports, mounts Google Drive, sets the `DATA` path. |
| **Geology fetches** *(run once)* | Pull US basalt and North America sedimentary basins from the NETL REST services; Canadian mafic-volcanic rocks from the NRCan Wheeler REST service. Saved as GeoJSONs. |
| **NatCarb prep** | Reads the NatCarb geodatabase, extracts saline & coal storage polygons. |
| **Facility data build** | Builds the merged Canada + US + Europe facility table `fac_low`. |
| **Pipeline loading** | Loads Alberta CO₂ pipelines, the US CCS shapefiles, and the NETL PHMSA-derived US CO₂ pipelines. |
| **Map assembly** | Computes all distances & strandedness factors; builds facility markers, filters, search, pin tool, pipeline overlays, geology layers, proposed-infrastructure layers; writes `index.html`. |

---

## Facility data — sources & acquisition

The facility layer merges three government emissions programs into one table (`fac_low`) with a common schema: `Facility, Company, Latitude, Longitude, NAICS, CO2_tonnes, Country`.

### Canada — ECCC GHGRP
- **File:** `canada1_2024_classified.csv`
- **Source:** Environment and Climate Change Canada, Greenhouse Gas Reporting Program (GHGRP), 2024 reporting year.
- **Processing:** Pre-processed by a one-time classification script that rolls raw per-emission-source rows up to one record per facility, derives a sector category, and — uniquely among the three regions — carries a real **operating-company** name (GHGRP "Reporting Company Legal Name"). Only region with company/operator data.

### United States — EPA GHGRP (FLIGHT)
- **File:** `ghgp_data_2023.xlsx`, sheet *Direct Emitters* (`skiprows=3`).
- **Source:** US EPA Greenhouse Gas Reporting Program, 2023.
- **Processing:** Total CO₂ = non-biogenic + biogenic; sector inferred from the EPA reporting sector. No operating-company field, so `Company` is blank.

### Europe — E-PRTR
- **File:** `F1_4_Air_Releases_Facilities.csv`
- **Source:** European Environment Agency, European Pollutant Release and Transfer Register (E-PRTR), 2023 reporting year.
- **Processing:** CO₂ releases summed per facility (kg → tonnes); sector from the E-PRTR Annex I activity code. Facility name only, no company.

---

## Pipeline data — sources & acquisition

CO₂ pipelines are combined into a single **"CO₂ pipelines"** map control with four status sub-toggles (Active / Proposed / Cancelled / Discontinued). Colors: Active = solid green, Proposed = dashed blue, Cancelled = light grey, Discontinued = medium grey.

### Alberta pipelines (survey-grade)
- **File:** `Pipelines_SHP/` (GCS_NAD83 version).
- **Source:** Government of Alberta pipeline shapefile (~322k segments, all commodities).
- **Processing:** Filtered to CO₂ only. Per-segment status: operating → Active, everything else → Discontinued.

### US CCS pipeline shapefiles (survey-grade)
- **Folder:** `CCS_Pipelines-selected/` — six regional shapefiles.
- **Source:** State/regulatory & project GIS (Texas RRC, Wyoming, Summit Carbon Solutions, Navigator, OH/WV/PA).
- **Layers & status:** Texas network (Active); Wyoming–Denbury (Active); Wyoming corridor initiative (Proposed — reserved rights-of-way, not operating pipeline); Summit Midwest Carbon Express & Project A (Proposed); Navigator Heartland Greenway (Cancelled, Oct 2023); OH/WV/PA potential routes (Proposed).

### US CO₂ pipelines — NETL (PHMSA-derived)
- **File:** `netl_co2_pipelines.geojson` (built from a NETL REST service).
- **Source:** DOE/NETL `Co2_Transportation_Pipeline_wma84` FeatureServer, **digitized from the PHMSA National Pipeline Mapping System (NPMS)** public map of active CO₂ pipelines.
- **Processing:** Pulled as GeoJSON, then **geographically clipped to the Gulf Coast and Dakotas only** (keep-box: Gulf `lon ≥ -96 & lat ≤ 33.5`; Dakotas `lat ≥ 45 & -104 ≤ lon ≤ -96`) so it fills the gap left by the survey-grade Texas/Wyoming shapefiles without duplicating them. Segments relabeled by location: *Souris Valley (Beulah ND → Weyburn SK)*, *Denbury — NEJD/Free State/Delta (MS–LA)*, *Denbury — Green Pipeline (LA → TX)*. Status = Active.
- **Note:** This PHMSA-derived layer **replaced** the earlier hand-traced Denbury Gulf and Souris Valley approximations, which have been retired.

---

## Storage-geology data — sources & acquisition

All geology layers are rendered as **single dissolved, flat-fill polygons** — overlapping source polygons are merged with `unary_union` so there is no darkening where polygons stack, and outlines are removed (one flat color per layer). Each is toggleable and off by default.

### Saline formations (North America + Europe, ONE combined layer)
- **North America — NatCarb.** Files `natcarb_saline.geojson` from the DOE/NETL National Carbon Sequestration Database (NATCARB v1502 geodatabase). Assigned EPSG:5070 (ships with no CRS declared), reprojected to 4326, simplified.
- **Europe — CO2StoP.** File `eu_co2stop_storage.geojson`, from the EU Joint Research Centre "European CO₂ storage database" (CO2StoP project, 2012–2014), open-format KML re-release (`StorageUnits_March13.kml`), 418 storage units. **Note:** CO2StoP units are saline aquifers *plus* some hydrocarbon-field storage units — the "Saline formations" label is a screening simplification for the European portion.
- Both sources are merged into one **"Saline formations"** toggle.

### Coal seams — NatCarb
- **File:** `natcarb_coal.geojson` (NATCARB v1502). Unmineable coal storage-resource polygons.

### Sedimentary basins — NETL Atlas V (North America)
- **File:** `na_sed_basins.geojson`, pulled from the NETL `North_America_Sedimentary_Basins` REST service (Atlas 5th Edition, 2015). Covers US + Canada; carries `basin_name`, `ccs_rating`, capacity, and `assessed` attributes.

### Basalt & mafic volcanic rocks (US + Canada, ONE combined layer)
- **US — NETL basalt (basalt-specific).** File `basalt_formations.geojson` from the NETL `Basalt_Formations_AtlasV_2025` REST service (Atlas V, 2015). Primarily the Columbia River Basalt Group.
- **Canada — Wheeler mafic volcanic (basalt proxy).** File `ca_mafic_volcanic.geojson`, pulled from the NRCan `geological_map_canada_wheeler_en` REST service (GSC Map 1860A, Wheeler et al. 1996, 1:5,000,000). Filtered server-side to `SUBRXTP LIKE '%mafic volcanic%'`. The 1:5M map has no basalt-specific class, so **mafic volcanic rocks are used as a basalt proxy** (a deliberate, accepted simplification — basalt is the dominant mafic volcanic rock).
- Both merged into one **"Basalt & mafic volcanic rocks"** toggle.
- Basalt-family rocks matter because they enable **CO₂ mineralization** (permanent conversion to carbonate minerals), which needs mafic/ultramafic rock (basalt, gabbro, peridotite) — not felsic volcanics.

---

## Proposed CO₂ infrastructure (hand-built)

Because North America has essentially no operating CO₂ shipping terminals and no consolidated dataset of proposed storage sites, two small point layers were hand-built from project announcements and SEC filings. **All points are proposed / study-stage / permitting — none operating — and coordinates are approximate** (county/parish or port-level).

### Proposed CO₂ marine terminals
- **File:** `co2_marine_terminals_approx.geojson` (3 points, pink hollow-diamond markers).
- T-RICH — Tampa Regional Intermodal Carbon Hub (OSG, Port Tampa Bay FL; DOE feasibility study); LBC Mississippi River LCO₂ terminal (LA); Alaska CO₂ import terminal (US–Japan study).

### Proposed CO₂ storage sites (Gulf Coast)
- **File:** `co2_storage_sites_approx.geojson` (11 points, blue down-triangle markers).
- Bayou Bend CCS (offshore lease + onshore, Chevron/Talos/Equinor); Houston CCS Hub (ExxonMobil consortium, offshore concept); Denbury/ExxonMobil portfolio (Leo, Weyerhaeuser, Libra, St. Helena, SE Louisiana, Aries, Gemini); Talos Harvest Bend.

---

## Distance computations

All distances are **straight-line great-circle ("as the crow flies") approximations** — computed once at build time via a spatial index (`STRtree.nearest`) + haversine to the nearest point on the nearest geometry. For polygon targets (saline, ocean), a facility *inside* the polygon returns 0 km. These are approximations suitable for screening, not routed/survey distances.

Per-facility distances computed:
- **`pipe_km`** — nearest **active** CO₂ pipeline (shown in facility popups).
- **`pipe_prop_km`** — nearest **proposed** CO₂ pipeline.
- **`pipe_ap_km`** — nearest **active-or-proposed** pipeline.
- **`saline_km`** — nearest **saline formation** (NatCarb + CO2StoP combined).
- **`term_km`** — nearest **proposed CO₂ marine terminal**.
- **`coast_km`** — nearest **ocean** (Natural Earth 10m Ocean polygon). The **ocean polygon** is used rather than a coastline so inland shores (Great Lakes, etc.) are correctly excluded — only genuine sea coasts count.

---

## Strandedness screen

"Strandedness" is not a single verdict but a set of **four independent, separately-filterable factors** nested under a "Strandedness" group in the filter panel. Each factor filters the whole map; within a factor, checking no boxes shows all, checking some narrows to those; across factors the logic is AND. This lets the user define "stranded" flexibly, since its meaning changes by context.

1. **Distance to pipeline** — nested Active/Proposed × near/far: `Active < 200 km`, `Active > 200 km`, `Proposed < 200 km`, `Proposed > 200 km`. (A facility holds both an active-band and a proposed-band membership.)
2. **CO₂ produced** — `> 200k t/yr` / `< 200k t/yr`.
3. **Distance to saline formation** — `< 200 km` / `> 200 km` (against the combined NatCarb + CO2StoP saline layer).
4. **Marine shipping** — a not-stranded/stranded verdict using **only** marine options (onshore pipelines deliberately excluded):
   - Not stranded if **within 200 km of an existing CO₂ shipping terminal** (small sources can truck to it for aggregation), **OR**
   - within 200 km of the **coast** (ocean) **AND** producing **≥ 500k t/yr** (big enough to justify building its own terminal).
   - Otherwise stranded. *(The filter shows this logic as an inline caption.)*

---

## Map features & interaction

- **Facility markers** — `CircleMarker` sized by CO₂ (log scale; 0/unreported small, 250k+ one step larger), recolorable by the "Color by" dimension. Popup shows facility, operator (Canada only, when it differs), NAICS, CO₂ t/yr, sector, region, and nearest active pipeline (with a "none within 500 km" cutoff for remote/European sites).
- **Filter panel** (top-right) — collapsible tree with: search box (facility OR company), "Color by" selector, primary filters (CO₂ scale, Sectors, Region), the nested Strandedness factors, and the pin distance tool. "None checked = all; check to narrow" within each group; AND across groups.
- **Search** — matches facility and company name, acts as a filter, lists matches (biggest first), click to zoom.
- **Pin distance tool** — click the map or enter coordinates to drop a pin; draws concentric rings for each band up to the chosen max radius (5/50/100/200/300 km); per-band cumulative popup with facility count, total CO₂, and a sector breakdown; "View list" and "Download CSV" export every facility in range.
- **CO₂ pipelines control** (top-right) — master toggle + four status sub-toggles (Active / Proposed / Cancelled / Discontinued); none checked = all show.
- **Layer control** (top-right) — geology layers (Saline formations, Coal seams, Sedimentary basins, Basalt & mafic volcanic rocks) and the proposed-infrastructure layers (marine terminals, storage sites).
- **Last-updated watermark** (bottom-left, build-time UTC).

---

## Sector categories & tiers

**17 sector categories** assigned from NAICS/industry descriptions with correction rules: Cement/lime & mineral products; Power generation; Oil & gas; Refining & petrochemicals; Industrial gases, hydrogen & ammonia; Other chemicals & plastics; Steel & metal; Mining & quarrying; Pulp, paper & wood; Manufacturing (food & drink); Manufacturing (other); Agriculture & greenhouses; Waste & wastewater; Buildings, transport & services; Fermentation; Bioethanol; Biogas.

**Emission tiers** color/size the points: a grey "0 / not reported" tier, five graduated middle tiers (viridis), and a "250,000+ t" top tier.

---

## Known limitations & caveats

- **Reporting thresholds.** All three programs only require facilities above a size threshold (Canada GHGRP ~10,000 t CO₂e/yr) to report. Smaller emitters are absent; the map is biased toward larger, more capture-relevant sources.
- **Zero / unreported facilities.** Some real emitters report 0 t (e.g. certain lime plants reporting under other programs) and show as grey "0 / not reported."
- **Company/operator — Canada only.** US and Europe sources have no operator field.
- **Straight-line distances.** All distance metrics and strandedness factors use great-circle approximations, not routed distances. "Within 200 km" is as-the-crow-flies.
- **Marine "coast" = ocean only.** The Great Lakes and other inland waters are excluded (ocean polygon used). This is intentional — CO₂ can't be shipped to sea from the Great Lakes.
- **Canadian basalt is a mafic-volcanic proxy.** GSC Map 1860A (1:5M) has no basalt-specific class; "mafic volcanic rocks" is used as an accepted proxy. The US NETL basalt layer, by contrast, is basalt-specific. The combined layer mixes these two resolutions.
- **European saline = CO2StoP (saline + hydrocarbon).** The "Saline formations" layer's European portion also includes some hydrocarbon-field storage units, and is from the 2012–2014 CO2StoP public subset (older than the 2025 GSEU/EGDI atlas, which was not obtainable as a direct download).
- **Proposed infrastructure is speculative and approximate.** The marine-terminal and storage-site points are proposed/study-stage with county/port-level coordinates, hand-built from announcements; they are a Gulf-Coast-focused snapshot, not an exhaustive inventory.
- **Traced pipelines retired.** Earlier hand-traced Denbury/Souris lines were replaced by the NETL PHMSA-derived layer.
- **Geology is overview-scale.** NatCarb, Atlas V, Wheeler 1860A, and CO2StoP are all generalized (1:5M or coarser), suitable for continental screening, not site selection.

---

## Data provenance summary

| Layer | Source | Format / access | Precision |
|-------|--------|-----------------|-----------|
| Canada facilities | ECCC GHGRP 2024 | CSV (pre-classified) | Reported |
| US facilities | EPA GHGRP 2023 | XLSX (FLIGHT) | Reported |
| Europe facilities | EEA E-PRTR 2023 | CSV | Reported |
| Alberta pipelines | Government of Alberta | Shapefile | Survey-grade |
| US CCS pipelines | State/regulatory + project GIS | Shapefiles | Survey-grade |
| US CO₂ pipelines (Gulf/Dakotas) | DOE/NETL, digitized from PHMSA NPMS | ArcGIS REST (GeoJSON) | Digitized (better than traced, not survey) |
| Saline formations — N. America | DOE/NETL NatCarb v1502 | File geodatabase | Overview |
| Saline formations — Europe | EU JRC CO2StoP (2012–2014) | Open-format KML | Overview, public subset |
| Coal seams | DOE/NETL NatCarb v1502 | File geodatabase | Overview |
| Sedimentary basins | DOE/NETL Atlas V (2015) | ArcGIS REST (GeoJSON) | Overview |
| Basalt — US | DOE/NETL Atlas V (2015) | ArcGIS REST (GeoJSON) | Overview, basalt-specific |
| Mafic volcanic — Canada | NRCan GSC Map 1860A (Wheeler 1996) | ArcGIS REST (GeoJSON) | 1:5M, basalt proxy |
| Ocean (coast distance) | Natural Earth 10m Ocean | Shapefile | Overview |
| Proposed marine terminals | Project announcements | Hand-built GeoJSON | Approximate (concept) |
| Proposed storage sites | Operator announcements / SEC filings | Hand-built GeoJSON | Approximate (concept) |

---

*Built for carbon-capture opportunity screening. Emissions figures are as-reported to the respective government programs (ECCC GHGRP, EPA GHGRP, EU E-PRTR) and inherit those programs' coverage, thresholds, and gaps. Pipeline, geology, and proposed-infrastructure layers combine survey-grade, digitized, overview-scale, and hand-built sources as noted above. All distance and strandedness metrics are straight-line approximations for screening, not routed or survey-grade measurements.*
