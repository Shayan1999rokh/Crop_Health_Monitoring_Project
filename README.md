# Multi-Temporal SAR-Based Crop Mapping, Health Assessment & Yield Estimation

**A farm-level remote sensing workflow for monitoring agricultural fields using synthetic aperture radar (SAR) imagery, geospatial analysis, texture features, and unsupervised machine learning.**

This project investigates how multi-date SAR observations can be transformed into interpretable agricultural indicators. Focusing on **966 field parcels in Sokhda village, Gujarat, India**, it combines four radar acquisitions, field-boundary geometries, handcrafted backscatter and texture descriptors, and clustering to estimate **crop type**, a **relative crop-health index**, and **yield to date** for each field.

The workflow is implemented as a documented Python notebook and includes diagnostic plots, spatial visualizations, quality checks, and a structured CSV export.

## Contents

- [Project overview](#project-overview)
- [Research objectives](#research-objectives)
- [Study area and input data](#study-area-and-input-data)
- [Methodology](#methodology)
- [Technical implementation](#technical-implementation)
- [Results](#results)
- [Output format](#output-format)
- [Installation and execution](#installation-and-execution)
- [Limitations](#limitations)
- [Potential extensions](#potential-extensions)
- [Project scope and attribution](#project-scope-and-attribution)

## Project overview

Timely information about crop distribution and crop condition is important for agricultural planning, monitoring, and resilience. Synthetic aperture radar provides a useful complement to optical remote sensing because radar observations are not dependent on sunlight and are less affected by cloud cover. Changes in radar backscatter and image texture can help characterize agricultural surfaces over time.

This project builds an end-to-end, **field-based analytical pipeline** that:

1. Loads and preprocesses multi-temporal geocoded SAR images.
2. Connects raster observations to individual agricultural field polygons.
3. Extracts radiometric and spatial texture descriptors for each field and acquisition date.
4. Discovers field groups through unsupervised dimensionality reduction and clustering.
5. Assigns crop categories using regional crop-distribution information and temporal crop signatures.
6. Derives a crop-relative health indicator and an estimated yield rate.
7. Produces maps, diagnostic figures, and a complete farm-level output table.

**Approach:** interpretable feature engineering and unsupervised learning; the notebook does **not** train a convolutional neural network or a supervised yield-prediction model.

## Research objectives

- **Crop mapping:** estimate the dominant crop category for each agricultural parcel.
- **Temporal characterization:** describe differences in SAR backscatter and texture across the growing season.
- **Relative condition assessment:** identify fields with stronger or weaker observed crop-development signatures compared with fields assigned to the same crop.
- **Yield estimation:** combine crop-specific reference yields, seasonal progress, and relative condition into estimated tonnes per hectare.
- **Spatial completeness:** maintain a prediction for every field, including those with incomplete image coverage.
- **Interpretability:** expose intermediate features, clustering diagnostics, coverage maps, and the assumptions behind derived indicators.

## Study area and input data

| Property | Description |
|---|---|
| Location | Sokhda village, Gujarat, India |
| Village identifier | `22` |
| Field polygons | **966** agricultural parcels |
| SAR platform / product | Capella Space, geocoded SAR imagery (GEO-HH files preferred by the input-discovery routine) |
| Observation dates | **6 June, 19 June, 14 August, and 13 October 2025** |
| Spatial inputs | SAR GeoTIFF rasters and ESRI Shapefiles describing field and village boundaries |
| Target crop categories | **Rice, Cotton, Maize, Bajra (pearl millet), Groundnut** |
| Auxiliary inputs | Regional crop-mixture proportions, reference crop yields, and crop-specific season-completion fractions |

### Input organization

The notebook searches its configured input root recursively for:

- `.tif` imagery matching the four acquisition-date stamps (`20250606`, `20250619`, `20250814`, `20251013`), preferentially selecting filenames containing `GEO`;
- a field-boundary `.shp` file whose path contains `farm`;
- a village-boundary `.shp` file whose path contains `village`.

Shapefiles must be accompanied by their supporting files (such as `.shx`, `.dbf`, and `.prj`). A representative local layout is:

```text
project/
├── notebook1b3affd622.ipynb
├── README.md
├── data/
│   ├── sar/
│   │   ├── ...20250606...GEO.tif
│   │   ├── ...20250619...GEO.tif
│   │   ├── ...20250814...GEO.tif
│   │   └── ...20251013...GEO.tif
│   └── boundaries/
│       ├── farms.shp        # plus supporting Shapefile files
│       └── village.shp      # plus supporting Shapefile files
└── out/                    # generated figures and tables
```

*Illustrative directory and filenames only; use the names of your actual files.*

### Regional crop composition used for crop labeling

| Crop | Reference share |
|---|---:|
| Rice | 13.90% |
| Cotton | 37.45% |
| Maize | 8.16% |
| Bajra | 15.32% |
| Groundnut | 25.17% |

These reference shares are used **after** clustering to help associate unlabeled groups with crop categories; they are not provided to K-Means as training labels.

## Methodology

```text
Multi-date SAR GeoTIFFs                   Farm and village polygons
            │                                        │
            ▼                                        │
Log-scale transformation (pseudo-dB)                │
            │                                        │
Area-of-interest cropping ◄─────────────────────────┤
            │                                        │
Speckle reduction and spatial trend correction      │
            │                                        │
            ▼                                        ▼
      Preprocessed rasters ─────────────► Field rasterization
                                                   │
                                                   ▼
                                       Valid-data coverage screening
                                                   │
                                                   ▼
                                      Farm-level SAR + GLCM descriptors
                                                   │
                                                   ▼
                                          StandardScaler → PCA
                                                   │
                                                   ▼
                                              K-Means (K=7)
                                                   │
                                                   ▼
                                 Crop-share matching + phenology checks
                                                   │
                                                   ▼
                                    Spatial nearest-neighbor completion
                                                   │
                                       ┌───────────┴───────────┐
                                       ▼                       ▼
                             Relative health index      Yield-to-date rate
                                       └───────────┬───────────┘
                                                   ▼
                                       Maps + farm-level CSV
```

### 1. SAR ingestion and log transformation

Each image is read with **Rasterio**. Nonpositive raster values are masked, and valid pixel values are transformed using:

\[
B_{\mathrm{pseudo\text{-}dB}}=10\log_{10}(DN)
\]

where `DN` represents the image pixel value. The notebook treats this as a **pseudo-dB transformation**, rather than claiming a complete sensor-specific radiometric calibration. Georeferencing metadata are preserved for downstream geospatial operations.

### 2. Area-of-interest cropping

The village boundary defines the spatial region of interest, with a **200 m buffer** added around its bounding box. The images are cropped to reduce memory and computation while retaining nearby edge parcels.

### 3. Speckle reduction

A NaN-aware **7 × 7 boxcar mean filter** smooths radar-image variability while excluding missing pixels from the local averaging calculation. The notebook includes before-and-after image comparisons.

### 4. Spatial trend correction

For each acquisition, a second-degree polynomial regression relates pixel position to the transformed backscatter signal. The fitted low-frequency spatial trend is removed and the image is recentered. The notebook reports an `R²` diagnostic for each date.

### 5. Field rasterization and coverage screening

Field polygons are transformed into a rasterized field-ID mask. This links image pixels to individual parcels and enables field-level extraction.

The notebook screens coverage across the four dates and retains fields meeting its **90% valid-coverage criterion** for direct feature-based classification.

**Observed coverage:** 896 of the 966 fields pass the initial screening.

### 6. Multi-temporal feature extraction

For each accepted field and each observation date, the workflow derives six descriptors:

| Feature | What it captures |
|---|---|
| Mean pseudo-dB | Average radar signal within a field |
| GLCM contrast | Local variations between neighboring gray levels |
| GLCM homogeneity | Similarity of neighboring image intensities |
| GLCM energy | Concentration or regularity of texture patterns |
| GLCM correlation | Statistical dependence between neighboring gray levels |
| GLCM entropy | Texture complexity and uncertainty |

GLCM refers to the **Gray-Level Co-occurrence Matrix**. The notebook quantizes imagery to **16 gray levels** and derives texture information from several angular relationships.

**Feature vector:** `6 descriptors × 4 dates = 24 features per field`.

The recorded feature matrix contains **896 rows × 24 columns**.

### 7. Scaling, dimensionality reduction, and clustering

The features pass through three operations:

1. **StandardScaler:** normalizes the statistical scale of the input variables.
2. **Principal Component Analysis (PCA):** retains components accounting for 90% of feature variance. The recorded run retained **8 components**.
3. **K-Means (`K=7`):** separates fields into seven unlabeled groups. This deliberately allows more groups than the five target crop categories.

The notebook plots a two-dimensional PCA projection, evaluates **silhouette scores for K=3–8**, and compares temporal feature trends between clusters.

### 8. Crop-category assignment

Because the groups are initially unlabeled, a separate interpretation step maps selected clusters to the five crop categories. It combines:

- **Crop composition matching:** searching for a mapping whose group frequencies resemble the regional reference crop shares.
- **Temporal signature review:** comparing group-level backscatter and texture patterns with expected growing-season behavior.

Two groups are set aside during this assignment; the remaining groups are associated with Rice, Cotton, Maize, Bajra, and Groundnut. The notebook includes diagnostic checks of the proposed assignments.

### 9. Spatial completion for remaining fields

To provide complete spatial coverage, fields without a direct crop assignment receive the crop category of the **nearest directly classified field**, using a spatial `cKDTree` search.

In the recorded output:

- **671 fields** receive direct crop assignments.
- **295 fields** are completed using the spatial fallback procedure.

The workflow tracks this distinction internally for interpretation.

### 10. Relative crop-health index

A field-level **health index between 0 and 100** combines three signals:

1. **Seasonal growth amplitude:** change in mean corrected radar signal between early June and mid-October.
2. **Phenological consistency:** similarity of a field's temporal pattern to the corresponding crop-group pattern.
3. **Late-season homogeneity:** a texture measure from the October image.

Each signal is expressed as a percentile within the relevant crop cohort, and the three percentiles are averaged:

\[
H_i=100\,\frac{P_{\mathrm{growth},i}+P_{\mathrm{phenology},i}+P_{\mathrm{homogeneity},i}}{3}
\]

Here, `Hᵢ` is a **relative indicator of observed crop condition**, not a direct field-measured physiological health score.

### 11. Yield-to-date estimation

The notebook converts the relative health index into an estimated yield rate by combining:

- a crop-specific reference yield;
- a crop-specific fraction of the growing season completed by **13 October 2025**;
- a relative adjustment based on the field's health index.

Its core relationship is:

\[
\widehat{Y}_i=Y_{\mathrm{ref},c}\,F_c\left(0.5+\frac{H_i}{100}\right)
\]

where:

- `Ŷᵢ` = estimated yield to date (tonnes/hectare);
- `Yref,c` = reference full-season yield for crop `c` (tonnes/hectare);
- `Fᶜ` = seasonal completion fraction;
- `Hᵢ` = relative health index.

The notebook also documents a coarse timing adjustment inferred from the sparse multi-date backscatter series. The output represents an **assumption-based yield estimate**, not a supervised prediction fitted to observed harvest totals.

### 12. Reporting and quality checks

The final steps generate crop and condition maps, yield distribution plots, and a consolidated dashboard. The exported CSV is checked for:

- one record per field and no duplicated field IDs;
- valid crop-category names;
- health indices within the `[0, 100]` range;
- yield estimates within the configured acceptable output range;
- absence of missing values.

## Technical implementation

| Component | Tools and techniques |
|---|---|
| Language | Python |
| Notebook environment | Jupyter-compatible notebook |
| Raster analysis | `rasterio`, `numpy` |
| Geospatial analysis | `geopandas`, Shapefile geometries, rasterization |
| Signal and texture processing | `scipy.ndimage`, `skimage.feature` |
| Machine learning | `scikit-learn`: StandardScaler, PCA, K-Means, LinearRegression |
| Spatial fallback | `scipy.spatial.cKDTree` |
| Data handling | `pandas` |
| Plotting | `matplotlib` |

## Results

The existing notebook contains a completed run with the following recorded outcomes.

### Processing summary

| Metric | Recorded value |
|---|---:|
| Agricultural parcels | **966** |
| Dates analyzed | **4** |
| Features per accepted parcel | **24** |
| Parcels passing initial SAR coverage screening | **896** |
| PCA components retained | **8** |
| K-Means groups | **7** |
| Directly labeled fields | **671** |
| Spatially completed fields | **295** |
| Final field records | **966** |

### Estimated crop distribution

| Crop | Number of fields |
|---|---:|
| Cotton | 335 |
| Groundnut | 294 |
| Bajra | 191 |
| Rice | 101 |
| Maize | 45 |
| **Total** | **966** |

These figures describe the workflow's **estimated assignments**, rather than an independently verified crop census.

### Estimated mean yield-to-date by crop

| Crop | Mean estimate (t/ha) |
|---|---:|
| Bajra | 1.796 |
| Cotton | 0.191 |
| Groundnut | 2.581 |
| Maize | 1.860 |
| Rice | 2.051 |

The estimates refer to the notebook's evaluation date and assumptions; they should not be interpreted as observed harvest yields.

### Visual analyses

The notebook includes visualizations of:

- agricultural parcel boundaries and the study area;
- SAR imagery before and after despeckling;
- valid SAR coverage by field;
- spatial maps of extracted texture descriptors;
- feature correlations;
- PCA embedding and cluster separation;
- silhouette scores and cluster phenology;
- estimated crop distribution;
- relative crop health and yield by field;
- an integrated crop-health-yield dashboard.

Example figure written by the notebook: `out/04_final_crop_map.png`. Additional plots are displayed inline during execution.

## Output format

The primary machine-readable output is:

```text
out/round2_submission.csv
```

It contains **966 records** with the following columns:

| Column | Meaning | Units / type |
|---|---|---|
| `village_id` | Study-area identifier | Integer (`22`) |
| `farm_id` | Unique field identifier | Integer / source ID |
| `crop_type` | Estimated crop category | Categorical string |
| `health_index` | Crop-relative condition index | 0–100 |
| `yield_estimate_to_date` | Estimated yield per unit area at the observation date | Tonnes/hectare |

The internal classification-source indicator is used for analysis but is not part of this final exported schema.

## Installation and execution

### Prerequisites

- Python 3.10+ recommended
- JupyterLab or Jupyter Notebook
- Access to the four geocoded SAR images and associated geospatial boundaries
- Sufficient RAM and disk capacity for raster processing

### Install the dependencies

Create and activate a virtual environment, then install:

```bash
python -m pip install numpy pandas geopandas rasterio scipy scikit-image scikit-learn matplotlib pillow jupyterlab
```

### Configure input and output directories

The supplied notebook contains environment-specific absolute paths. **Update the `BASE` and `OUT_DIR` variables** in the setup/data-discovery cells for your local folder structure, for example:

```python
from pathlib import Path

BASE = str(Path("data").resolve())
OUT_DIR = str(Path("out").resolve())
Path(OUT_DIR).mkdir(parents=True, exist_ok=True)
```

Keep the remaining input-discovery logic compatible with the SAR acquisition date strings and boundary filenames. Check that the detected files are correct before running the analysis.

### Run

```bash
jupyter lab notebook1b3affd622.ipynb
```

Execute the cells **in order from top to bottom**, since subsequent sections rely on arrays, tables, and variables generated in earlier sections. The notebook creates the output directory, displays figures, performs assertions, and exports the farm-level CSV.

**Reproducibility note:** the outcome depends on the availability and arrangement of the same underlying SAR imagery, vector boundaries, and reference inputs. Package versions are not pinned in the supplied notebook; pinning them is recommended for archived reproducible runs.

## Limitations

This is an exploratory remote-sensing workflow. Crop labels and derived health/yield indicators depend on the available SAR acquisitions, crop-composition assumptions, and an unsupervised classification strategy. Sparse temporal coverage, simplified spatial completion, and the lack of independent field-level validation mean that the outputs are best treated as **estimates suitable for analysis and further evaluation** rather than confirmed agronomic measurements.

## Potential extensions

Several extensions could build on the existing pipeline:

- Incorporate denser SAR time series and complementary multispectral observations.
- Compare handcrafted SAR descriptors with learned representations from CNNs or vision transformers.
- Evaluate field-level classification against independently collected crop labels.
- Explore temporal models for phenological monitoring.
- Fit and test yield-prediction models when observed field-level harvest data become available.
- Apply geographically separated validation to assess transferability to other regions.

These are **future research directions**, not capabilities claimed for the present implementation.

## Project scope and attribution

This README documents the methods and outputs **implemented in the accompanying notebook**. It does not establish authorship, ownership, or a redistribution license for the code or imagery. Before publishing the notebook or associated datasets, verify the applicable permissions and attribution requirements.

---

**Keywords:** Remote Sensing · Synthetic Aperture Radar · SAR Image Analysis · Precision Agriculture · Crop Classification · GLCM Texture Features · Unsupervised Learning · K-Means · PCA · Crop Health Monitoring · Yield Estimation · Geospatial Data Science
