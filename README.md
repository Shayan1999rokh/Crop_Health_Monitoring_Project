[README.md](https://github.com/user-attachments/files/34007419/README.md)
# Multi-Temporal SAR Crop Mapping, Health Monitoring and Yield Estimation

**An interpretable, field-level remote sensing workflow integrating SAR image analysis, geospatial processing, texture-based feature engineering, and unsupervised machine learning.**

This project examines how multi-temporal Synthetic Aperture Radar (SAR) imagery can support agricultural monitoring when field surveys and frequent optical observations are limited. Four radar acquisitions, farm boundary geometries, and seasonal crop information are brought together to characterize **966 agricultural parcels in Sokhda village, Gujarat, India**. The resulting analytical pipeline assigns a crop category, derives a **relative crop health index (0–100)**, and calculates an **estimated yield-to-date rate (tonnes/ha)** for every field.

Rather than relying on a black-box neural network, the implementation emphasizes **transparent feature engineering, interpretable clustering, spatial visualization, and explicit quality checks**. It is implemented in Python as a reproducible, stepwise research notebook.

> **Scope:** This is an exploratory, unsupervised SAR-based agricultural analysis. Crop identities and health/yield values are model-derived estimates, not independently surveyed crop labels, clinical-style vegetation diagnoses, or measurements of harvested yield.

---

## Contents

- [Research motivation](#research-motivation)
- [Objectives and deliverables](#objectives-and-deliverables)
- [Study area and observations](#study-area-and-observations)
- [End-to-end workflow](#end-to-end-workflow)
- [Methodology](#methodology)
- [Results from the recorded run](#results-from-the-recorded-run)
- [Visualizations](#visualizations)
- [Output dataset](#output-dataset)
- [Software and requirements](#software-and-requirements)
- [Running the notebook](#running-the-notebook)
- [Interpretation and methodological notes](#interpretation-and-methodological-notes)
- [Further research](#further-research)
- [Attribution and data permissions](#attribution-and-data-permissions)

## Research motivation

Understanding which crops are grown, where field conditions differ, and how productivity varies across a landscape is fundamental to precision agriculture, food-system planning, and crop monitoring. Optical imagery can be restricted by cloud cover during active growing seasons. SAR offers complementary observations by sensing microwave backscatter, with responses influenced by vegetation structure, surface moisture, and observation geometry.

A single radar image provides only a snapshot. **Multi-date SAR measurements**, by contrast, offer a view of how fields change through the growing season. When combined with mapped farm boundaries, these observations can be summarized at the scale of individual agricultural parcels rather than interpreted solely as isolated pixels.

The study uses these properties to construct an integrated workflow connecting radar imagery with crop-group identification, within-crop condition comparisons, and approximate productivity estimates.

## Objectives and deliverables

The analytical goals are to:

1. **Prepare multi-temporal radar images** through logarithmic transformation, geographic subsetting, noise suppression, and spatial trend adjustment.
2. **Align observations with real farm boundaries** and assess the availability of valid SAR measurements for every parcel.
3. **Represent fields quantitatively** through radar intensity and Gray-Level Co-occurrence Matrix (GLCM) texture descriptors.
4. **Discover groups of similar fields** with PCA and K-Means, without supplying crop names to the clustering algorithm.
5. **Interpret the groups as crop categories** using regional crop composition and seasonal SAR behavior as complementary evidence.
6. **Retain all parcels** by assigning a spatially based fallback label when direct classification is unavailable.
7. **Generate relative field-health and yield-to-date estimates**, accompanied by maps and consistency checks.

**Core deliverables:** a farm-level CSV, spatial crop and condition maps, clustering diagnostics, feature-analysis figures, and a consolidated visual summary.

## Study area and observations

| Item | Specification |
|---|---|
| Study location | Sokhda village, Gujarat, India |
| Study-area identifier | `22` |
| Agricultural parcels | **966** |
| SAR source | Capella Space, geocoded HH radar products |
| Observation dates | **6 Jun, 19 Jun, 14 Aug, 13 Oct 2025** |
| Spatial inputs | SAR GeoTIFF images; village and farm Shapefiles |
| Target crops | Rice, Cotton, Maize, Bajra (pearl millet), Groundnut |
| Analysis level | Individual agricultural field |
| Primary output units | Health index: 0–100; estimated yield: t/ha |

### Seasonal imagery

Four acquisitions represent distinct points within the 2025 growing season. The temporal dimension is important because seasonal changes in radar intensity and texture can help separate crops with different development patterns.

| Acquisition | Notebook label | Date token |
|---|---|---|
| First | `Jun6` | `20250606` |
| Second | `Jun19` | `20250619` |
| Third | `Aug14` | `20250814` |
| Fourth | `Oct13` | `20251013` |

### Reference crop composition

The notebook uses the following previously established proportions during **post-clustering label assignment**. They are not used as labels for training K-Means.

| Crop | Reference share |
|---|---:|
| Rice | 13.90% |
| Cotton | 37.45% |
| Maize | 8.16% |
| Bajra | 15.32% |
| Groundnut | 25.17% |

Crop-specific reference yield values and growing-season completion fractions are also supplied as constants in the notebook for the yield calculation.

## End-to-end workflow

```text
             FOUR GEOREFERENCED SAR ACQUISITIONS
                        (Jun–Oct 2025)
                                  |
                    Logarithmic transformation
                                  |
              Village-based subsetting (+200 m buffer)
                                  |
                  NaN-aware speckle filtering
                                  |
             Second-order spatial trend adjustment
                                  |
       Farm polygons --> Farm-ID raster --> Coverage screening
                                              |
                                  Per-field radar descriptors
                               (6 features × 4 dates = 24)
                                              |
                                 Standardization and PCA
                                              |
                                      K-Means (K=7)
                                              |
                       Reference shares + phenology review
                                              |
                         Five crop categories / two groups
                                  set aside during labeling
                                              |
                         Spatial 1-nearest-neighbor fallback
                                              |
                           +------------------+----------------+
                           |                                   |
                 Crop-relative health                  Estimated yield
                   index (0–100)                         (tonnes/ha)
                           |                                   |
                           +------------------+----------------+
                                              |
                          Maps, summary figures, validated CSV
```

## Methodology

### 1. Data discovery and initialization

The notebook imports numerical, geospatial, image-processing, machine-learning, and visualization libraries. It locates SAR images by acquisition-date tokens, favoring filenames containing `GEO`, and searches for farm and village Shapefiles. It also defines the reference crop proportions, reference yields, and crop-specific seasonal factors.

The input search directory and output directory are defined in the setup cells. These paths should be updated for a new computing environment.

### 2. SAR intensity transformation

The first band of each radar image is read as a floating-point raster. Nonpositive pixel values are masked, and the remaining values are transformed as

\[
B_{\mathrm{pseudo\text{-}dB}} = 10\log_{10}(DN),
\]

where `DN` is the stored pixel value. In this project the transformation is treated as a **pseudo-dB representation**, enabling comparisons of relative radar brightness across fields; it should not be confused with a complete, sensor-specific absolute backscatter calibration.

Raster coordinate systems and affine transformation information are retained for spatial processing.

### 3. Study-area extraction

The mapped village boundary establishes the analysis footprint. A **200 m buffer** is added around its bounding box to preserve nearby farm edges. A corresponding pixel window is calculated and applied to the images, and the georeferencing transform is updated to represent the cropped rasters.

Subsetting reduces the volume of image data processed by subsequent stages.

### 4. Speckle reduction and spatial bias adjustment

Radar imagery contains granular speckle associated with coherent imaging. A **NaN-aware 7 × 7 boxcar filter** averages local pixel values without incorporating missing-data cells into valid observations. A before-and-after crop of the imagery is visualized to illustrate the filtering effect.

The notebook then estimates a smooth positional intensity trend separately for each acquisition. A **second-order polynomial regression**, fitted using up to **200,000 randomly sampled valid pixels**, models intensity as a function of image row and column. The estimated trend is subtracted and the original mean level restored.

An **R² diagnostic** is reported for each date to indicate how much sampled intensity variation is explained by the fitted positional model. This is an empirical trend adjustment, not a substitute for full geometric and radiometric sensor calibration.

### 5. Rasterization and field coverage assessment

The 966 field polygons are converted into a raster mask that associates image pixels with corresponding `farm_id` values. For every field, the notebook calculates the average fraction of valid pixels across the four dates.

Fields with **at least 90% valid coverage** are eligible for direct feature extraction. Those with less coverage remain in the analysis and receive labels through the later spatial-completion procedure.

**Recorded result: 896 of 966 fields passed the initial coverage screen.**

### 6. Feature engineering: SAR intensity and GLCM texture

For every eligible field, the workflow computes six descriptors per acquisition. Across four dates this yields a **24-feature representation** for each parcel.

| Descriptor | Interpretation | Agricultural relevance |
|---|---|---|
| Mean pseudo-dB | Mean transformed radar intensity | Broad changes in vegetation and surface scattering |
| GLCM contrast | Local differences between gray levels | Heterogeneity and intensity transitions |
| GLCM homogeneity | Similarity of neighboring gray levels | Spatial uniformity of radar response |
| GLCM energy | Concentration of co-occurrence probabilities | Regularity and organization of texture |
| GLCM correlation | Association between neighboring intensities | Spatial dependencies within field patterns |
| GLCM entropy | Dispersion/uncertainty of gray-level relationships | Texture complexity |

The implementation quantizes valid values to **16 gray levels** and calculates co-occurrence information at a pixel distance of one, across **0°, 45°, 90°, and 135°**. Texture descriptors summarize the resulting GLCM statistics.

**Implementation detail:** In the supplied notebook, the valid pixels of a field are packed into a square array before GLCM computation. Consequently, the reported texture features are best described as features of this implementation, rather than guaranteed measurements of the original geographic pixel adjacency. More spatially faithful masked-GLCM extraction is a natural future enhancement.

**Recorded feature matrix:** `896 × 24`.

A feature-correlation heatmap and individual descriptor maps help examine the information represented by the variables.

### 7. Standardization, PCA and K-Means clustering

Before clustering, features are standardized using `StandardScaler`. Principal Component Analysis is then used to reduce redundancy while preserving **90% of the standardized feature variance**. The recorded run retained **eight principal components**.

K-Means operates on the PCA representation with **seven clusters** (`random_state=42`, `n_init=10`). Selecting more groups than the five target crop categories allows for additional variation and potentially atypical land-cover signatures.

The notebook examines a two-dimensional PCA projection, evaluates silhouette scores for **K=3–8**, and plots average feature trajectories for the unlabeled groups.

**Recorded cluster sizes:** `113, 42, 58, 4, 221, 186, 272`.

The seven-cluster configuration is a design choice informed by the workflow's intended crop-labeling strategy; it is not presented as the silhouette-optimal configuration.

### 8. Mapping unsupervised groups to crop types

Cluster indices do not carry agricultural meaning by themselves. The notebook therefore applies an additional interpretation stage:

**Crop-share matching.** Every five-cluster subset is considered, and candidate mappings to the five target crops are scored according to the sum of squared deviations from the reference crop proportions. The best-scoring assignment is selected.

**Phenological review.** Seasonal changes in group-average radar intensity and texture are compared with plausible growing-season characteristics of the assigned crops. A rule-based diagnostic records which assignments receive additional texture-based support and which remain uncertain.

In the recorded run, the proposed correspondence is:

| Cluster | Assigned crop |
|---|---|
| 2 | Rice |
| 6 | Cotton |
| 1 | Maize |
| 0 | Bajra |
| 5 | Groundnut |
| 3, 4 | Set aside from direct crop labeling |

The diagnostic supported three crop assignments, flagged one as not meeting its chosen texture criterion, and left one inconclusive. This review **does not constitute an independent ground-truth accuracy assessment** or automatically revise the proportion-based mapping.

### 9. Spatial classification completion

Not every field can be assigned a crop directly. The notebook handles two situations: insufficient SAR coverage and membership in a cluster set aside during crop interpretation.

For these fields, the centroid of the parcel is compared with the centroids of directly classified fields using `scipy.spatial.cKDTree`. The crop label and extracted features of the **single nearest directly classified field** are transferred to the parcel.

| Assignment route | Fields |
|---|---:|
| Direct classification | **671** |
| Nearest-neighbor completion | **295** |
| — Due to low image coverage | 70 |
| — Due to excluded clusters | 225 |
| **Total** | **966** |

The notebook preserves a `data_source` field internally to distinguish assignment routes.

### 10. Relative crop-health assessment

A field-level health index aggregates three indicators:

1. **Seasonal backscatter change:** the change in date-offset-adjusted mean radar intensity between 6 June and 13 October.
2. **Phenological similarity:** the correlation between an individual field's four-date response and the average response of its assigned crop group.
3. **Late-season homogeneity:** the October texture-homogeneity descriptor.

The three indicators are transformed to **within-crop percentile ranks**, and their average is rescaled to 0–100:

\[
H_i = \frac{100}{3}\left(P_{\mathrm{growth},i}+P_{\mathrm{phenology},i}+P_{\mathrm{homogeneity},i}\right).
\]

A higher index means a field has comparatively stronger values under these particular radar-derived indicators; the score is **relative to a crop cohort**, not an absolute measurement of crop vigor or stress. The supplied implementation ranks the directly classified and fallback-assigned subsets separately within crop categories.

### 11. Yield-to-date estimation

The final health index is converted into a per-hectare yield estimate using a reference yield and a predefined crop-specific season-completion fraction:

\[
\widehat{Y}_i = Y_{\mathrm{ref},c}\, F_c\left(0.5+\frac{H_i}{100}\right).
\]

Here, `Yref,c` is the reference full-season yield in tonnes per hectare, `F_c` is the fixed completion factor for crop `c` as of 13 October, and `H_i` is the computed relative health index. A health value of 50 produces the seasonally scaled reference yield.

| Crop | Reference yield (kg/ha) | Season factor at 13 Oct |
|---|---:|---:|
| Cotton | 635 | 0.30 |
| Groundnut | 3,026 | 0.85 |
| Bajra | 1,787 | 1.00 |
| Rice | 2,538 | 0.80 |
| Maize | 2,022 | 0.90 |

**Important:** The implemented function uses a **fixed season factor for each crop**, not a separately inferred planting-date factor for each farm. The estimated value is a modeled, season-adjusted productivity indicator rather than a supervised prediction validated against harvest observations.

### 12. Reporting, export and checks

The workflow generates crop-distribution maps, health and yield visualizations, summary figures, and a final field-level CSV. Before export, assertions check the number and uniqueness of farms, permitted crop names, acceptable numeric ranges, and missing values.

The output remains one record per farm and contains exactly the expected five columns.

## Results from the recorded run

The figures below are **outputs saved in the provided notebook**, not results from a new execution performed for this README.

### Key processing statistics

| Indicator | Recorded value |
|---|---:|
| Fields in study area | **966** |
| SAR acquisition dates | **4** |
| Features per accepted field | **24** |
| Fields passing coverage screening | **896** |
| Retained PCA dimensions | **8** |
| K-Means clusters | **7** |
| Direct classifications | **671** |
| Spatial fallback assignments | **295** |
| Validated output records | **966** |

### Estimated crop distribution

| Crop | Fields | Share of all fields |
|---|---:|---:|
| Cotton | 335 | 34.7% |
| Groundnut | 294 | 30.4% |
| Bajra | 191 | 19.8% |
| Rice | 101 | 10.5% |
| Maize | 45 | 4.7% |
| **Total** | **966** | **100% (rounded)** |

These are **estimated crop labels**, not a verified census. Percentages are rounded individually.

### Average estimated yield by crop

| Crop | Mean yield-to-date estimate (t/ha) |
|---|---:|
| Bajra | 1.796 |
| Cotton | 0.191 |
| Groundnut | 2.581 |
| Maize | 1.860 |
| Rice | 2.051 |

Yield estimates depend on the reference-yield and seasonal assumptions described above.

## Visualizations

The notebook provides diagnostic and reporting figures at multiple stages of the analysis:

- **Study-area overview:** village boundary and individual farm polygons.
- **SAR image comparison:** image appearance before and after speckle filtering.
- **Coverage map:** location of fields with incomplete observations.
- **Texture-feature maps:** spatial differences in the extracted descriptors.
- **Feature-correlation heatmap:** relationships and potential redundancy among 24 variables.
- **PCA and clustering views:** separation and internal structure of field groups.
- **Silhouette diagnostic:** sensitivity to different cluster counts.
- **Temporal signatures:** seasonal radar/texture patterns across groups and assigned crops.
- **Crop distribution:** map of estimated agricultural categories.
- **Field health and yield:** spatial representations of derived indicators.
- **Integrated dashboard and village summary:** an end-to-end view of the final results.

Examples of files generated by the notebook include:

```text
out/
├── 04_final_crop_map.png
├── 05a_glcm_feature_maps.png
├── 07_proportion_match_map.png
├── 10b_village_summary.png
└── round2_submission.csv
```

The notebook also shows additional figures inline. Some image names may be reused across later visualization cells; preserve or rename them if every intermediate figure must be retained.

<!-- Optional: after generating figures and committing them, uncomment a real relative path:
![Estimated crop map](out/04_final_crop_map.png)
-->

## Output dataset

The principal machine-readable deliverable is **`out/round2_submission.csv`**.

| Column | Description | Type / units |
|---|---|---|
| `village_id` | Identifier for the study area | Integer (`22`) |
| `farm_id` | Parcel identifier | Integer / source ID |
| `crop_type` | Assigned crop category | String |
| `health_index` | Relative crop-condition indicator | Numeric, 0–100 |
| `yield_estimate_to_date` | Estimated production rate as of the final acquisition | Tonnes per hectare |

The internal `data_source` field distinguishes direct classifications from nearest-neighbor assignments but is intentionally **not included** in the exported five-column dataset.

## Software and requirements

### Core libraries

| Task | Python tools |
|---|---|
| Array and tabular operations | `numpy`, `pandas` |
| Geospatial data | `geopandas`, `rasterio` |
| Image and texture processing | `scipy.ndimage`, `scikit-image`, `Pillow` |
| Dimensionality reduction and clustering | `scikit-learn` |
| Spatial nearest neighbor | `scipy.spatial.cKDTree` |
| Figures and maps | `matplotlib` |
| Interactive execution | Jupyter Notebook / JupyterLab |

Recommended environment: Python 3.10 or later, with sufficient memory to load and process the SAR raster arrays. Package versions are not fixed in the supplied notebook.

### Installation

Create a virtual environment and install the packages:

```bash
python -m venv .venv
# Windows PowerShell:
.venv\Scripts\Activate.ps1
# macOS / Linux:
# source .venv/bin/activate

python -m pip install --upgrade pip
python -m pip install numpy pandas geopandas rasterio scipy scikit-image scikit-learn matplotlib pillow jupyterlab
```

### Suggested project layout

```text
crop-health-monitoring/
├── Crop Health Monitoring (1).ipynb
├── README.md
├── data/
│   ├── sar/
│   │   ├── <image_with_20250606_and_GEO>.tif
│   │   ├── <image_with_20250619_and_GEO>.tif
│   │   ├── <image_with_20250814_and_GEO>.tif
│   │   └── <image_with_20251013_and_GEO>.tif
│   └── boundaries/
│       ├── farms.shp      # with .shx, .dbf, .prj, etc.
│       └── village.shp    # with .shx, .dbf, .prj, etc.
└── out/
```

This tree illustrates a local arrangement; the actual input filenames may differ.

## Running the notebook

1. Place the four geocoded SAR rasters and the associated farm/village boundary files in a directory accessible to the notebook.
2. In the first configuration cells, **replace the environment-specific `BASE` and `OUT_DIR` paths** with directories for your own setup. For example:

   ```python
   from pathlib import Path
   BASE = str(Path("data").resolve())
   OUT_DIR = str(Path("out").resolve())
   Path(OUT_DIR).mkdir(parents=True, exist_ok=True)
   ```

3. Check that the date-based raster lookup resolves the expected four images and that the Shapefile paths point to the correct boundary datasets.
4. Open the notebook:

   ```bash
   jupyter lab "Crop Health Monitoring (1).ipynb"
   ```

5. Run the cells **sequentially from top to bottom**. Intermediate arrays, raster transforms, field tables, and derived features are reused in later steps.
6. Inspect the generated plots and the exported `out/round2_submission.csv`.

**Data integrity checks before rerunning:** ensure that the images use compatible coordinate systems, resolution, raster alignment, and georeferencing; confirm that the Shapefile geometries are valid and that `farm_id` values remain unique. The source notebook does not automatically harmonize grids from different acquisitions.

## Interpretation and methodological notes

The workflow is intended to support exploratory agricultural mapping and demonstrate how image analysis, geospatial data, and unsupervised models can be combined at field scale. A few considerations matter when interpreting its outputs:

- **Crop categories** are inferred from unsupervised clusters with reference-distribution matching and phenological diagnostics; independently observed crop labels would be needed to establish classification accuracy.
- **SAR texture descriptors** reflect the current pixel-reshaping implementation and could be strengthened with methods preserving native spatial adjacency.
- **Fields completed spatially** inherit information from a single nearest classified parcel; those results carry a different evidential basis from direct classifications.
- **Health and yield indicators** are relative or formula-derived estimates and have not been calibrated against independent field-health or harvest records in the notebook.

These considerations identify practical directions for validation without changing the documented outputs of the current workflow.

## Further research

Promising extensions of the existing methodology include:

- More frequent observations to characterize crop development with finer temporal resolution.
- Fusion of SAR and optical or multispectral imagery for more robust agricultural mapping.
- Spatially faithful, field-masked texture extraction and multiscale descriptors.
- Comparison of handcrafted features with learned representations from CNNs, temporal networks, and vision transformers.
- Evaluation using field-survey crop labels and spatially separated train/test regions.
- Yield model calibration using measured harvest data and uncertainty estimates.
- Sensitivity analyses for clustering choices, crop reference proportions, and spatial completion methods.

These are **proposed research extensions**, not functions already implemented in the notebook.

## Attribution and data permissions

This documentation describes the workflow and outputs observable in the accompanying notebook. It does not itself determine code authorship or license terms, or grant redistribution rights for the source imagery or boundaries. Any public repository should retain applicable notices and comply with the original code and data permissions.

---

**Keywords:** SAR · Remote Sensing · Agricultural Image Analysis · Computer Vision · Precision Agriculture · Crop Mapping · GLCM · Geospatial Data Science · Unsupervised Learning · PCA · K-Means · Crop Health Index · Yield Estimation
