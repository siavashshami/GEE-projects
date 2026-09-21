# RUSLE-Based Soil Erosion and Land Cover Change Analysis

A comprehensive workflow for modeling soil erosion using the **Revised Universal Soil Loss Equation (RUSLE)** in **Google Earth Engine (GEE)**, followed by multi-region land-cover and soil-erosion analysis, statistical stability assessment, and publication-quality visualization of results across climate zones and years.

---

## Overview

This project provides an integrated, end-to-end pipeline for soil erosion assessment:

1. **Google Earth Engine RUSLE Model** — Computes all five RUSLE factors (R, K, LS, C, P) at the pixel level using scientifically validated methods and multi-source remote-sensing datasets, then produces annual soil-loss maps and per-land-cover-class erosion statistics across multiple climate zones.

2. **Numerical Stability and Erosion Analysis** — Reads the exported per-region land-cover and soil-erosion CSVs and produces comprehensive statistical summaries: area/erosion means, total erosion (ton/year), annual change rates, region risk indices, coefficient-of-variation (CV) stability analysis, and cross-region comparisons.

3. **Publication-Quality Visualization** — Generates a composite figure combining stacked land-cover area bar charts with overlaid soil-erosion time series for each region, with a unified legend and high-resolution output suitable for journals.

Together, these scripts implement a fully reproducible scientific workflow for studying soil erosion dynamics under land-cover and climate variability.

---

## 1. Google Earth Engine RUSLE Model

### Overview

This is a **complete, scientific, article-ready RUSLE implementation** in Google Earth Engine. It computes annual soil loss (t/ha/yr) for a user-defined Area of Interest (AOI) over a range of years, based on the standard RUSLE equation:

```
A = R × K × LS × C × P
```

Each factor is computed using established, peer-reviewed methods, and the script performs **sanity checks** (slope vs. erosion, NDVI vs. erosion) to validate model behavior.

### Scientific Methods

| Factor | Method | Reference |
|--------|--------|-----------|
| **R** (Rainfall Erosivity) | CHIRPS daily precipitation → monthly sums → Modified Fournier Index (MFI) → `R = 1.735 × 10^(1.5·log10(MFI) − 0.8188)` | Renard et al. (1997), USDA-ARS Handbook 703 |
| **K** (Soil Erodibility) | OpenLandMap USDA texture classes remapped to NRCS K-values | Renard et al. (1997), USDA-NRCS |
| **LS** (Slope Length & Steepness) | Moore & Burch (1986) using HydroSHEDS flow accumulation (corrected v2.1); S-factor from McCool et al. (1987) | Moore & Burch (1986); McCool et al. (1987) |
| **C** (Cover Management) | NDVI-based exponential model with **hemisphere-aware growing-season weighting** | Van der Knijff et al. (2000) |
| **P** (Support Practice) | Slope-based uniform values (Wischmeier & Smith 1978) applied across all land-cover classes | Wischmeier & Smith (1978), USDA Handbook 537 |

### Input Data

* **AOI**: A user-provided `ee.FeatureCollection` (default: `projects/ee-siavashshami/assets/Cs`).
* **Land Cover**: MODIS `MCD12Q1` (LC_Type1), reclassified from 17 IGBP classes into **12 merged classes**:
  `EvergreenForest, DeciduousForest, MixedForest, Shrublands, Savanna, Grasslands, Wetlands, Agriculture, Urban, SnowIce, Barren, Water`.
* **Precipitation**: `UCSB-CHG/CHIRPS/DAILY`.
* **NDVI**: `MODIS/061/MOD13Q1`.
* **Elevation**: `USGS/SRTMGL1_003`.
* **Soil Texture**: `OpenLandMap/SOL/SOL_TEXTURE-CLASS_USDA-TT_M/v02`.
* **Flow Accumulation**: `WWF/HydroSHEDS/15ACC`.

### Configuration

Key parameters at the top of the script:

```javascript
var startYear = 2016;
var endYear   = 2016;
var climateZoneName = 'Cs';
var analysisScale = 2000;  // meters
```

### Outputs

For the first year, the script:

* Prints **original (17-class)** and **merged (12-class)** land-cover histograms.
* Adds all RUSLE factors (R, K, LS, C, P) and the final **RUSLE soil-loss map** to the map viewer.
* Performs **two sanity checks**:
  * Mean soil loss by **slope class** (should increase with slope).
  * Mean soil loss by **NDVI class** (should decrease with NDVI).
* Prints a **complete factor summary** with Min, Max, Mean, and StdDev for R, K, LS, C, P, and total RUSLE.
* Produces **multi-year land-cover erosion statistics** (area in km²/ha, mean erosion rate, total erosion in ton/year per land-cover class).
* Generates interactive charts:
  * **Land Cover Area by Class** (grouped column chart).
  * **Mean Soil Erosion Rate by Land Cover** (line chart).

### How to Run

1. Open the script in the **Google Earth Engine Code Editor**.
2. Adjust the AOI asset path, `startYear`, `endYear`, `climateZoneName`, and `analysisScale`.
3. Click **Run**. All layers, charts, and statistics are displayed in the Console and Map panels.
4. Export per-region CSV files (land cover + soil erosion) for downstream analysis (see Section 2).

### References

* Renard, K.G., et al. (1997). *Predicting Soil Erosion by Water: A Guide to Conservation Planning with the Revised Universal Soil Loss Equation (RUSLE)*. USDA-ARS Handbook 703.
* Moore, I.D., & Burch, G.J. (1986). "Physical basis of the length-slope factor in the Universal Soil Loss Equation." *Soil Science Society of America Journal*, 50(5), 1294–1298.
* McCool, D.K., et al. (1987). "Revised slope steepness factor for the Universal Soil Loss Equation." *Transactions of the ASAE*, 30(5), 1387–1396.
* Van der Knijff, J.M., et al. (2000). *LISEM: A Single-Event Physically Based Hydrological and Soil Erosion Model for Drainage Basins*. CATENA.
* Wischmeier, W.H., & Smith, D.D. (1978). *Predicting Rainfall Erosion Losses: A Guide to Conservation Planning*. USDA Handbook 537.

---

## 2. Land Cover and Soil Erosion Changes (Numerical Analysis)

### Overview

This Python script performs **numerical analysis and stability assessment** on the RUSLE-exported CSVs across multiple climate regions. For each region, it computes:

* Mean area and mean erosion rate per land-cover class.
* Total erosion (ton/year) per class and for the whole region.
* Annual change rate (%/year) per class over the study period.
* **Region Risk Index** (ton/year/km²).
* **Coefficient of Variation (CV)** for area, erosion rate, and a combined stability score.
* A **stability ranking** (lowest CV → highest CV) of land-cover classes.
* A **cross-region comparison** (highest-erosion region, average risk index, highest-CV class per region).

### Input Data

The script expects **paired CSV files** in a folder, one pair per region:

* `<region>-land-cover.csv`
* `<region>-soil-erosion(tonnes-ha-year).csv`

Example: `Cs-land-cover.csv` and `Cs-soil-erosion(tonnes-ha-year).csv`.

Each CSV must contain a `year` column and one column per land-cover class.

### Configuration

```python
folder_path = r"C:\Users\siavashshami\Documents\test"
```

The script automatically scans the folder and pairs files by region code.

### Land Cover Classes

```
EvergreenForest, DeciduousForest, MixedForest, Shrublands,
Savanna, Grasslands, Wetlands, Agriculture, Urban,
SnowIce, Barren, Water
```

**Always filtered out** (not analyzed): `Wetlands, Urban, SnowIce, Water`.
Additionally, classes are only kept if they exceed **1000 km²** in at least one year.

### Method

1. **Data loading and cleaning** — Commas removed, values converted to float, years rounded to integers.
2. **Mean statistics** — Area and erosion rate averaged across years per class.
3. **Total erosion** — `Area_km² × Rate_t/ha/yr × 100 = ton/year`.
4. **Annual change rate** — Linear percent change per year from first to last observation.
5. **Risk index** — Total erosion normalized by total region area.
6. **Stability analysis** — CV = (std / mean) × 100 for area, erosion, and combined score.
7. **Ranking** — Classes sorted by combined CV (lower = more stable).
8. **Cross-region comparison** — Highest-erosion region, average risk index, and highest-CV class per region.

### Output

The script writes a plain-text report:

```
complete_numerical_stability_analysis.txt
```

containing:

* Region-by-region statistics.
* Stability analysis tables (CV per class).
* Stability rankings (lowest → highest CV).
* Summary statistics (dominant cover, top erosion contributor, risk index).
* Detailed erosion table per class.
* Cross-region comparison summary.

It also prints a numerical summary to the console.

### Requirements

* Python 3
* Pandas
* NumPy

---

## 3. Land Cover and Soil Erosion Changes (Bar Chart and Time Series)

### Overview

This Python script produces a **composite, publication-quality figure** that combines, for each region:

* **Stacked bar charts** of annual land-cover area (km²) per class.
* **Overlaid line plots** of soil-erosion rate (t/ha/yr) on a secondary y-axis.
* A **unified legend** at the bottom, spanning both area and erosion series.

The layout arranges up to six regions in a 3×2 grid, making it ideal for multi-climate-zone comparison figures in journals.

### Input Data

Same paired CSV structure as Section 2:

* `<region>-land-cover.csv`
* `<region>-soil-erosion(tonnes-ha-year).csv`

### Configuration

```python
folder_path = r"C:\Users\siavashshami\Documents\test"
```

Output resolution: 600 DPI (both `figure.dpi` and `savefig.dpi`).

### Method

For each region:

* **Bar chart** — One bar per class per year, positioned by integer year + offset.
* **Erosion time series** — Line + markers for each class, plotted on a twin y-axis.
* **X-axis** — Integer years, filtered to show only odd years for readability.
* **Y-axis labels** — Only displayed on the middle row (left for area, right for erosion).
* **Region label** — Displayed in a rounded box in the top-right corner of each subplot.

A **shared legend** below the figure shows combined area and erosion series for all classes used across all regions.

### Custom Colors

The script uses a fixed palette of 12 colors, one per land-cover class:

```python
custom_colors = ['#05450a', '#78d203', '#009900', '#c6b044', '#fbff13',
                 '#b6ff05', '#27ff87', '#ff6d4c', '#a5a5a5', '#69fff8',
                 '#f9ffa4', '#1c0dff']
```

### Output

Two high-resolution PNG files:

* `landcover_barchart_by_year.png`
* `landcover_erosion_legend_fixed.png`

Both at 600 DPI with tight bounding boxes and white background.

### Requirements

* Python 3
* Pandas
* NumPy
* Matplotlib

---

## Project Workflow

```
┌───────────────────────────────┐
│   GEE RUSLE Script (Sec. 1)   │
│  Computes R, K, LS, C, P and  │
│  annual soil loss per class   │
└───────────────┬───────────────┘
                │  exports CSV
                ▼
┌───────────────────────────────┐
│  <region>-land-cover.csv      │
│  <region>-soil-erosion.csv    │
└───────────────┬───────────────┘
                │
       ┌────────┴────────┐
       ▼                 ▼
┌──────────────┐  ┌────────────────────┐
│ Section 2    │  │ Section 3          │
│ Numerical    │  │ Visualization      │
│ Stability    │  │ (bar + time series)│
│ Analysis     │  │                    │
└──────────────┘  └────────────────────┘
       │                 │
       ▼                 ▼
  .txt report       .png figures
```

---

## Repository Structure (Suggested)

```
rusle-soil-erosion-analysis/
├── README.md
├── gee/
│   └── rusle_complete_scientific.js
├── analysis/
│   ├── land_cover_erosion_analysis.py
│   └── land_cover_erosion_plots.py
├── data/
│   └── README.md          # instructions for exporting GEE CSVs
├── outputs/
│   ├── complete_numerical_stability_analysis.txt
│   ├── landcover_barchart_by_year.png
│   └── landcover_erosion_legend_fixed.png
└── LICENSE
```

---

## Requirements Summary

| Component | Requirements |
|-----------|--------------|
| GEE script | Google Earth Engine account; AOI as `ee.FeatureCollection` |
| Numerical analysis | Python 3, Pandas, NumPy |
| Visualization | Python 3, Pandas, NumPy, Matplotlib |

---

## Notes and Best Practices

* **Consistent AOI and years** must be used across all three scripts to ensure comparability.
* **CSV column names** must match the 12 merged land-cover classes exactly.
* **Filtered classes** (`Wetlands, Urban, SnowIce, Water`) and classes with area <1000 km² are excluded from analysis to avoid spurious statistics.
* **Sanity checks** in the GEE script are essential — if erosion does not increase with slope or decrease with NDVI, the model inputs should be revisited.
* **Scientific references** for each RUSLE factor are listed in the Console output and in Section 1 above.

---

## Citation

Test
