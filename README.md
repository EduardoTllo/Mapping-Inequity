# Mapping Inequity: Travel Time Barriers to Health Care in Peru and Implications for Universal Health Coverage

The Biostatistics team at the **University of Engineering and Technology (UTEC)** developed this project to provide policy-makers, researchers, and the public with timely, accurate, and openly accessible geospatial data on travel times to health care facilities across Peru. Using least-cost path modelling, we produce downloadable datasets of travel times to Level I, II, and III facilities, accompanied by fully reproducible R/Quarto scripts.

---

## 📦 Get the Data

Processed datasets (travel-time GeoTIFFs, summary tables) are archived at Zenodo:

> **DOI:** https://doi.org/10.5281/zenodo.17342579

---

## 🗂️ Repository Structure

```
Mapping-Inequity/
├── 01_DFcleaning.qmd               # Data cleaning & coordinate preparation
├── 02_final_cost.qmd               # Friction surface & travel-time modelling
├── 03_graphics and data export.qmd # Visualization & summary table export
├── 04_multivariate analysis.qmd    # Multivariate regression (Poisson / NB)
└── Manuscript.pdf                  # Full manuscript
```

---

## ⚙️ Prerequisites

Install the following R packages before running any script:

```r
install.packages(c(
  # Spatial
  "sf", "terra", "raster", "gdistance", "fasterize", "tmap",
  # Data wrangling
  "dplyr", "tidyr", "purrr", "stringr", "stringi", "forcats",
  # Modelling
  "MASS", "AER", "lmtest", "AICcmodavg", "PerformanceAnalytics",
  "corrplot", "broom",
  # I/O & reporting
  "readxl", "writexl", "knitr", "kableExtra", "ggplot2", "scales"
))
```

All scripts are written in **Quarto (`.qmd`)** and rendered to HTML. Run them **in numerical order**.

---

## 🔬 Project Workflow

### 1 · `01_DFcleaning.qmd` — Data Cleaning & Preparation

**Input:** `TB_en.csv` (RENIPRESS facility registry, semicolon-delimited, comma decimals)

**Steps:**
- Filters facilities whose `category` field starts with `I`, `II`, or `III` (primary, secondary, tertiary care).
- Extracts category prefix into a new column `cat_pref`.
- Converts `longitude` / `latitude` to numeric; drops rows with missing coordinates.
- Splits the dataset into three data frames by care level.

**Outputs:** `coords_I.csv`, `coords_II.csv`, `coords_III.csv`

---

### 2 · `02_final_cost.qmd` — Friction Surface & Travel-Time Modelling

**Inputs:** OSM road network (`.shp`), DEM (`cut_s30w090.tif`), Copernicus land-cover (`copernicus_landcover_peru.tif`), INEI provincial boundaries (`.shp`), coordinate CSVs from step 1.

**Steps:**
1. **Road network** — reads OSM roads, keeps `motorway` → `residential` classes, assigns speed (km/h) from `maxspeed` field with class-based fallbacks (motorway 100 km/h → residential 35 km/h); mph values are converted automatically.
2. **Rasterization** — rasterizes road speeds to a ~1 km (0.01°) grid; off-road cells default to walking speed (5 km/h).
3. **DEM processing** — projects DEM to UTM-18S (EPSG:32718), resamples to match road raster.
4. **Slope** — computes slope in degrees/tangent via `terra::terrain()`.
5. **Tobler's hiking function** — `vel_tobler = 6 × exp(−3.5 × |tan(slope) + 0.05|)` gives off-road walking speed; on-road driving friction is penalized by slope grade.
6. **Land-cover penalty** — Copernicus classes are reclassified to friction multipliers (e.g., urban = ×1.0, water/snow = ×6.0) and applied to off-road cells.
7. **Friction surface** — final raster combines driving friction (on-road) and slope-adjusted walk × land-cover friction (off-road); masked to Peru.
8. **Transition model** — `gdistance::transition()` with 8-directional connectivity; conductance = `1 / mean(friction)`.
9. **Accumulated cost** — `gdistance::accCost()` computed for each facility category (I, II, III) using a helper function `process_category()`.

**Outputs per category:** `costN.tif` (travel-time raster, minutes), `df_extN.rds` (pixel values extracted to provinces).

---

### 3 · `03_graphics and data export.qmd` — Visualization & Data Export

**Inputs:** `df_extN.rds`, `costN.tif` (from step 2), INEI department boundaries (`.shp`).

**Steps:**
1. **Load & stack** — reads all three category RDS files and combines into `df_all` (columns: `departamento`, `provincia`, `category`, `tiempo`).
2. **Departmental boxplots** — `plot_category()` generates per-category ggplot2 boxplots ordered by descending median travel time; provincial medians overlaid as red circles; 4-hour reference line highlighted.  
   Saved as: `boxplot_catI/II/III_departments.png`
3. **Provincial median distribution** — boxplot + jitter of provincial medians for category III.  
   Saved as: `boxplot_province_medians_peru.png`
4. **Summary Excel tables** — travel-time statistics (min, median, mean, max) pivoted wide by category, exported at two aggregation levels.  
   Saved as: `summary_tables_by_department.xlsx`, `summary_by_department_province_category.xlsx`
5. **Table 2** — median ± IQR (hours) by department for categories II and III, with province counts binned into `<1 h`, `1–<4 h`, `≥4 h`.  
   Saved as: `table2.xlsx`, `table_province_times.xlsx`
6. **Cost maps** — `tmap` choropleth maps for each category using a fixed colour scale (YlOrRd, 0–>12 h), with department borders, compass, scale bar.  
   Saved under: `fig_costs/map_cost_type_I/II/III.png`

---

### 4 · `04_multivariate analysis.qmd` — Multivariate Regression

**Inputs:** `IDE_IDH_population_medIII_data.xlsx` (sheets 3–5 containing HDI, IDE index, population, and category-III provincial medians).

**Steps:**
1. **Data preparation** — reads and joins HDI (sheet 3), IDE (sheet 4) and category-III median travel times (sheet 5) by province; normalizes text (whitespace, accents, case).  
   Exported as: `data_IDH_IDE_population_medIII.csv`
2. **Correlation analysis** — Pearson correlation matrix (`corrplot`) of HDI, IDE, population, and median travel time.
3. **Poisson models** — null, univariate (IDE, HDI, population), and full models fitted via `glm(family = poisson)`; compared by AIC (`AICcmodavg`); overdispersion tested with `AER::dispersiontest`; heteroskedasticity with Breusch-Pagan test.
4. **Negative Binomial models** — same model set refitted with `MASS::glm.nb()` to account for overdispersion; AIC comparison.
5. **Prevalence Ratios** — exponentiated coefficients (PR) and 95% CIs extracted for all NB models via `broom::tidy()`.  
   Exported as: `prevalence_ratios_table.xlsx`

> **Note:** Lima province is excluded from regression models to avoid leverage from its outlier population size; population is scaled to tens of thousands.

---

## 📊 Key Outputs Summary

| File | Description |
|------|-------------|
| `coords_I/II/III.csv` | Facility coordinates by care level |
| `cost1/2/3.tif` | Travel-time rasters (minutes) per level |
| `df_ext1/2/3.rds` | Pixel-level times extracted to provinces |
| `boxplot_cat*.png` | Departmental travel-time boxplots |
| `fig_costs/map_cost_type_*.png` | Choropleth accessibility maps |
| `summary_tables_by_department.xlsx` | Travel-time stats by dept / province / category |
| `table2.xlsx` | Median ± IQR table (categories II & III) |
| `prevalence_ratios_table.xlsx` | NB regression prevalence ratios |

---

## 🗃️ Data Sources

| Source | Description |
|--------|-------------|
| [RENIPRESS – MINSA](https://www.minsa.gob.pe/) | Health facility registry (`TB_en.csv`) |
| [OpenStreetMap](https://www.geofabrik.de/) | Road network (`gis_osm_roads_free_1.shp`) |
| [Copernicus Land Cover](https://lcviewer.vito.be/) | Land-cover classification |
| [SRTM / NASA](https://srtm.csi.cgiar.org/) | Digital Elevation Model |
| [INEI 2023](https://www.inei.gob.pe/) | Provincial & departmental boundaries |

---

## 📝 Methodology Notes

- **Projection:** All spatial analysis in UTM Zone 18S (EPSG:32718); final maps in WGS84 (EPSG:4326).
- **Friction surface:** On-road cells use road-speed friction penalized by slope; off-road cells use Tobler's hiking function modulated by land-cover type.
- **Least-cost path:** Computed with `gdistance::accCost()` using an 8-directional transition matrix.
- **Regression:** Overdispersion in Poisson models (verified by dispersion test) motivates the preferred Negative Binomial specification.

---

## 👥 Credits

- **Eduardo Tello, Cristobal Byrne, Fernanda Malaga & Maurizio de la Rosa** (UTEC Biostatistics Team)
- Methodology: Least-cost path modelling with the `gdistance` package in R
- Data sources: RENIPRESS (MINSA), OpenStreetMap, Copernicus, SRTM, INEI
