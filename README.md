# North American & European CO₂ Facility & Storage Map

An interactive [Folium](https://python-visualization.github.io/folium/)/Leaflet map of industrial CO₂-emitting facilities across **Canada, the United States, and Europe**, built for carbon-capture opportunity screening. Facilities are plotted as points sized by emissions and colored by tier, sector, or region, with filtering, company/facility search, and a radius proximity tool. The map also overlays **CO₂ pipelines** and **geological storage layers** (saline basins, coal seams, sedimentary basins, and basalt formations).

The map is generated from a Google Colab notebook (`canada_emissions_map_clean.ipynb`) and saved as a single self-contained `index.html` that can be opened locally or hosted (e.g. GitHub Pages).

---

## Contents

- [Quick start](#quick-start)
- [Notebook structure](#notebook-structure)
- [Facility data — sources & acquisition](#facility-data--sources--acquisition)
- [Pipeline data — sources & acquisition](#pipeline-data--sources--acquisition)
- [Storage-geology data — sources & acquisition](#storage-geology-data--sources--acquisition)
- [Mapping process](#mapping-process)
- [Sector categories & tiers](#sector-categories--tiers)
- [Known limitations & caveats](#known-limitations--caveats)
- [Data provenance summary](#data-provenance-summary)

---

## Quick start

1. Open `canada_emissions_map_clean.ipynb` in Google Colab.
2. Ensure all source files are in your Drive folder (default `DATA = /content/drive/MyDrive/Data`).
3. Run the cells **top to bottom** (see structure below). The geology-fetch cells (marked *"DON'T RE-RUN"*) hit live government servers and only need to run once — after that they read the saved GeoJSONs.
4. The final cell writes `index.html`.

> **Cell order matters.** The map cell depends on `fac_low` (facilities), the pipeline objects, and the geology GeoJSONs all existing. Running out of order can leave a variable stale. Run sequentially after any change.

---

## Notebook structure

| Cell | Purpose |
|------|---------|
| **1 — Setup** | Imports, mounts Google Drive, sets the `DATA` path. |
| **2 — Basalt fetch** *(run once)* | Pulls US basalt formations from the NETL REST service to `basalt_formations.geojson`. |
| **3 — Sedimentary basins fetch** *(run once)* | Pulls North America sedimentary basins from the NETL REST service to `na_sed_basins.geojson`. |
| **4 — NatCarb prep** | Reads the NatCarb geodatabase, extracts saline & coal storage polygons to `natcarb_saline.geojson` / `natcarb_coal.geojson`; then loads all four geology layers. |
| **5 — Facility data** | Builds the merged Canada + US + Europe facility table `fac_low`. |
| **6 — Pipeline loading** | Loads Alberta CO₂ pipelines, the traced Denbury/ND approximate lines, and the US CCS pipeline shapefiles. |
| **7 — Map assembly** | Builds the Leaflet map: facility markers, filters, search, pin tool, pipeline overlays, geology layers; writes `index.html`. |

---

## Facility data — sources & acquisition

The facility layer merges three government emissions programs into one table (`fac_low`) with a common schema: `Facility, Company, Latitude, Longitude, NAICS, CO2_tonnes, Category, CO2_Concentration, Country`.

### Canada — ECCC GHGRP
- **File:** `canada1_2024_classified.csv`
- **Source:** Environment and Climate Change Canada, Greenhouse Gas Reporting Program (GHGRP), 2024 reporting year.
- **Acquisition & processing:** Downloaded from the ECCC GHGRP facility emissions dataset, then pre-processed by a separate one-time classification script (kept outside this notebook) that rolls the raw per-emission-source rows up to one record per facility, derives the fossil/biogenic split and a concentration estimate, assigns a sector category, and — uniquely among the three regions — carries a real **operating-company** name (from the GHGRP "Reporting Company Legal Name" field).
- **Notes:** This is the only region with company/operator data. Not pre-filtered by emissions (includes zero and very large emitters).

### United States — EPA GHGRP (FLIGHT)
- **File:** `ghgp_data_2023.xlsx`, sheet *Direct Emitters*, header on row 4 (`skiprows=3`).
- **Source:** US EPA Greenhouse Gas Reporting Program, 2023.
- **Acquisition & processing:** Downloaded from EPA FLIGHT. In-notebook, total CO₂ = non-biogenic + biogenic; `Category` (Fossil/Biogenic) is set by whichever is larger; `CO2_Concentration` (Low/Medium/High) is inferred from the facility's dominant reporting sector via a lookup (e.g. ammonia/hydrogen → High; cement/lime/steel/etc. → Medium; otherwise Low). No operating-company field exists in the EPA export, so `Company` is blank.

### Europe — E-PRTR
- **File:** `F1_4_Air_Releases_Facilities.csv`
- **Source:** European Environment Agency, European Pollutant Release and Transfer Register (E-PRTR), 2023 reporting year.
- **Acquisition & processing:** Downloaded from the EEA E-PRTR data portal. In-notebook, CO₂ releases are summed per facility (total and "excluding biomass"), biogenic is derived as the difference, tonnes = kg / 1000; `Category` from the fossil/biogenic comparison; `CO2_Concentration` inferred from the E-PRTR Annex I activity code. No company field (facility name only).

---

## Pipeline data — sources & acquisition

CO₂ pipelines are drawn from four sources of differing precision, combined into a single **"CO₂ pipelines"** map toggle and colored by status (green = existing, dashed blue = proposed, dotted grey = cancelled/discontinued).

### Alberta pipelines (survey-grade)
- **File:** `Pipelines_SHP/` (the `GCS_NAD83` geographic version).
- **Source:** Government of Alberta pipeline shapefile (~322k segments, all commodities).
- **Processing:** Filtered in-notebook to CO₂ only (matching "carbon dioxide"/"CO2" across three substance columns), reprojected to EPSG:4326. Per-segment status coloring (green if operating, grey dotted otherwise).

### US CCS pipeline shapefiles (survey-grade)
- **Folder:** `CCS_Pipelines-selected/` — six regional shapefiles.
- **Source:** State/regulatory and project GIS, compiled (Texas Railroad Commission, Wyoming, Summit Carbon Solutions filings, Navigator, OH/WV/PA).
- **Layers & status:**
  - *Texas CO₂ network* — Existing
  - *Wyoming — Denbury* — Existing
  - *Wyoming corridor initiative (planned)* — Proposed (Wyoming Pipeline Corridor Initiative reserved corridors, **not** operating pipelines; reclassified as Proposed after review)
  - *Summit Midwest Carbon Express* — Proposed
  - *Summit Project A* — Proposed
  - *Navigator Heartland Greenway* — Cancelled (project cancelled Oct 2023)
  - *OH/WV/PA potential routes* — Proposed
- **Processing:** Most are EPSG:4269; OH/WV/PA is EPSG:3174. All reprojected to EPSG:4326 on load.

### Denbury Gulf Coast network (approximate — traced)
- **File:** `denbury_gulf_co2_approx.geojson`
- **Source:** Hand-traced from published Denbury / Healthy Gulf maps, using cities and facilities as coordinate anchors.
- **Contents:** Green Pipeline, NEJD, Delta, Free State and associated laterals (~890 mi across LA / MS / east TX).
- **Accuracy:** City-level (~km); **not survey-grade**. Real operating pipelines, but the geometry is an approximation and every feature is tagged as such. Built to fill the Gulf Coast gap not covered by downloadable shapefiles (US CO₂ pipeline centerlines are largely access-restricted by PHMSA).

### Souris Valley ND→SK line (approximate — traced)
- **File:** `nd_souris_co2_approx.geojson`
- **Source:** Hand-traced from basemap screenshots and the Canada Energy Regulator Saskatchewan CO₂ map, anchored on Beulah (Dakota Gas), the Lake Sakakawea east-arm crossing, Minot, Estevan, Boundary Dam, and Weyburn.
- **Contents:** Souris Valley / Dakota Gasification CO₂ pipeline, Beulah ND → Weyburn SK (~230 mi drawn; ~205 mi documented).
- **Accuracy:** City-level (~km); **not survey-grade**.

---

## Storage-geology data — sources & acquisition

Four geological layers show where captured CO₂ could be stored. All are toggleable and off by default.

### Saline basins & coal seams — NatCarb
- **Files:** `natcarb_saline.geojson`, `natcarb_coal.geojson`
- **Source:** DOE/NETL National Carbon Sequestration Database (NATCARB), version v1502, distributed as a file geodatabase (`NATCARB_v1502.gdb`).
- **Acquisition & processing:** Geodatabase downloaded from NETL EDX. In-notebook, the saline and coal storage-resource polygon layers are read, assigned EPSG:5070 (the geodatabase ships with **no CRS declared** — NatCarb v1502 is USA Contiguous Albers), reprojected to EPSG:4326, and simplified (1000–1500 m tolerance) to shrink file size for browser rendering. The oil & gas layer (~68k polygons) and the 10 km grid layers were deliberately skipped as too large / duplicative.

### Sedimentary basins — NETL Atlas V (North America)
- **File:** `na_sed_basins.geojson`
- **Source:** DOE/NETL, *North America Sedimentary Basins*, National Carbon Sequestration Atlas 5th Edition (2015).
- **Acquisition:** Pulled directly from the NETL ArcGIS REST service as GeoJSON (no download step) — service `EDXSpatialCaches_North_America_Sedimentary_Basi_223_remake`, layer 3. Reprojected from Web Mercator (EPSG:3857) to 4326 and simplified (1000 m). Covers both the US and Canada; carries useful attributes including `basin_name`, `ccs_rating`, storage-capacity estimates, and an `assessed` flag.
- **Note:** This single continental layer replaced earlier separate US (USGS `usprov12`) and Canadian (NRCan REST) basin layers, giving one consistent source.

### Basalt formations — NETL Atlas V (US)
- **File:** `basalt_formations.geojson`
- **Source:** DOE/NETL, *Basalt Formations Atlas V*, National Carbon Sequestration Atlas 5th Edition (2015).
- **Acquisition:** Pulled directly from the NETL ArcGIS REST service as GeoJSON — service `Basalt_Formations_AtlasV_2025`, layer 7. Reprojected from Web Mercator to 4326 and simplified (500 m).
- **Scope:** US only (primarily the Columbia River Basalt Group, the main US basalt-CCS target). A Canadian basalt layer was investigated (GSC Map 1860A / Wheeler geology via NRCan REST) but that national map has no basalt-specific class — only broader "mafic volcanic rocks" at 1:5,000,000 scale — so Canadian basalt was left out pending a decision.

---

## Mapping process

The map-assembly cell (Cell 7) builds everything in Leaflet via Folium:

**Facility markers.** Each facility in `fac_low` becomes a `CircleMarker`. Radius scales with CO₂ (log-based; zero/unreported = small fixed size, 250k+ = one step larger). Color is assigned by the active "color by" dimension (tier, sector, or region). A tooltip/popup shows facility name, operator (Canada only, when it differs from the facility name), NAICS, CO₂ tonnes, sector, and region.

**Filtering.** A collapsible control offers three filter dimensions — CO₂ emissions scale (tier), sectors, and region — as checkboxes (OR within a dimension, AND across). A "color by" dropdown recolors points; a facility count and "clear all" are included.

**Search.** A search box matches typed text against **both facility name and company name** (case-insensitive substring) and acts as a filter — non-matching facilities are hidden. Matches are listed below the box (biggest emitters first) and clicking one zooms to it.

**Pin distance tool.** Enable, then click the map or type coordinates to drop a pin. Concentric rings are drawn for every band up to a chosen max radius (5/50/100/200/300 km), and a nested popup reports facility count, total CO₂, and a sector breakdown per band (each band cumulative). "View list" and "Download CSV" export every facility in the radius, tagged with the smallest band it falls in.

**Pipeline overlay.** All pipeline sources are added to one `FeatureGroup` ("CO₂ pipelines") so a single toggle shows/hides them. Alberta uses per-segment status coloring; US CCS layers and the traced Denbury/ND lines are colored by their assigned status (existing/proposed/cancelled).

**Geology overlays.** Saline basins, coal seams, sedimentary basins, and basalt are each added as translucent, toggleable `FeatureGroup`s (off by default) drawn beneath the facility points, so you can see which emitters sit over viable storage.

**Output.** The map is written to a self-contained `index.html`.

---

## Sector categories & tiers

**17 sector categories** are assigned from NAICS/industry descriptions with correction rules: Cement/lime & mineral products; Power generation; Oil & gas; Refining & petrochemicals; Industrial gases, hydrogen & ammonia; Other chemicals & plastics; Steel & metal; Mining & quarrying; Pulp, paper & wood; Manufacturing (food & drink); Manufacturing (other); Agriculture & greenhouses; Waste & wastewater; Buildings, transport & services; Fermentation; **Bioethanol**; **Biogas**.

- **Bioethanol** — a set of Canadian fuel-ethanol plants pulled into their own sector by name override. Remain "Fossil" on the fossil/biogenic axis (reported emissions are natural-gas combustion, not fermentation CO₂).
- **Biogas** — Canadian waste/sewage/landfill facilities split out by NAICS. A facility-*type* label, not a verified biomethane producer.

**Emission tiers** color/size the points: a distinct grey "0 / not reported" tier, five graduated middle tiers (viridis), and a bright red "250,000+ t" top tier.

---

## Known limitations & caveats

- **Reporting thresholds.** All three programs only require facilities above a size threshold (Canada GHGRP ~10,000 t CO₂e/yr) to report. Smaller emitters are absent entirely. This biases the map toward the larger, more capture-relevant sources.
- **Zero / unreported facilities.** Some real, sizable emitters report 0 t in GHGRP (e.g. certain lime plants whose process emissions are reported under provincial/OBPS programs not included here). They show as grey "0 / not reported" points.
- **Fossil/Biogenic and Concentration are carried but not currently shown** on the map — they were unreliable across sources (e.g. bioethanol plants misclassifying) and are inferred, not measured.
- **Company/operator — Canada only.** US and Europe sources have no operator field.
- **Coarser sector mapping outside Canada.** US uses the EPA sector label and Europe the E-PRTR sector name in place of true NAICS.
- **Traced pipelines are approximate.** The Denbury Gulf Coast and Souris Valley ND→SK layers are hand-traced from published maps (~km accuracy), not survey-grade shapefiles like the Alberta and US CCS data. They are labeled "(approx.)".
- **Wyoming corridor initiative** is a planned-corridor dataset (reserved rights-of-way), not operating pipeline — classified as Proposed.
- **Basalt is US-only** (NETL Atlas V); no Canadian basalt layer is included.
- **Atlas V geology is 2015-vintage and generalized**; sedimentary-basin and basalt layers are continental-overview scale, not site-specific.

---

## Data provenance summary

| Layer | Source | Format / access | Precision |
|-------|--------|-----------------|-----------|
| Canada facilities | ECCC GHGRP 2024 | CSV (pre-classified) | Reported |
| US facilities | EPA GHGRP 2023 | XLSX (FLIGHT) | Reported |
| Europe facilities | EEA E-PRTR 2023 | CSV | Reported |
| Alberta pipelines | Government of Alberta | Shapefile | Survey-grade |
| US CCS pipelines | State/regulatory + project GIS | Shapefiles | Survey-grade |
| Denbury Gulf Coast pipelines | Traced from published maps | GeoJSON (built here) | Approximate (~km) |
| Souris Valley ND→SK pipeline | Traced from CER map + basemaps | GeoJSON (built here) | Approximate (~km) |
| Saline basins, coal seams | DOE/NETL NatCarb v1502 | File geodatabase | Overview |
| Sedimentary basins | DOE/NETL Atlas V (2015) | ArcGIS REST (GeoJSON) | Overview |
| Basalt formations (US) | DOE/NETL Atlas V (2015) | ArcGIS REST (GeoJSON) | Overview |

---

*Built for carbon-capture opportunity screening. Emissions figures are as-reported to the respective government programs (ECCC GHGRP, EPA GHGRP, EU E-PRTR) and inherit those programs' coverage, thresholds, and reporting gaps. Pipeline and geology layers combine survey-grade, approximate-traced, and overview-scale sources as noted above.*
