# NLST CT Acquisition Parameter Analysis Notebook

## Goal
Create a Python Jupyter notebook to analyze DICOM CT acquisition parameter distributions from the NLST collection in IDC, enabling identification of a representative subset that covers the variety of acquisitions.

## Data Source
- **BigQuery table:** `bigquery-public-data.idc_current.dicom_all`
- **Collection:** `nlst` (National Lung Screening Trial)
- **Modality:** CT
- **Authentication:** Default Google credentials (`gcloud auth application-default login`)

---

## Key DICOM Attributes Affecting CT Image Appearance

*Redundant attributes removed: XRayTubeCurrent & ExposureTime (derived from Exposure), Rows/Columns (derived from PixelSpacing & FOV), SoftwareVersions (too granular)*

### 1. Spatial Resolution
| Attribute | DICOM Tag | Impact |
|-----------|-----------|--------|
| SliceThickness | (0018,0050) | Z-axis resolution; thinner = more detail + noise |
| PixelSpacing | (0028,0030) | In-plane resolution (row, column spacing in mm) |
| SpacingBetweenSlices | (0018,0088) | Gap/overlap between slices (independent of SliceThickness) |

### 2. Contrast/Exposure
| Attribute | DICOM Tag | Impact |
|-----------|-----------|--------|
| KVP | (0018,0060) | Tube voltage; affects contrast and noise |
| Exposure | (0018,1152) | Total exposure (mAs); determines noise level |
| CTDIvol | (0018,9345) | Standardized dose metric; better for cross-scanner comparison |

### 3. Reconstruction (Most Visually Significant)
| Attribute | DICOM Tag | Impact |
|-----------|-----------|--------|
| ConvolutionKernel | (0018,1210) | Reconstruction filter; sharp vs smooth appearance |

### 4. Hardware
| Attribute | DICOM Tag | Impact |
|-----------|-----------|--------|
| Manufacturer | (0008,0070) | Different detector characteristics |
| ManufacturerModelName | (0008,1090) | Scanner generation/capabilities |

### 5. Geometry
| Attribute | DICOM Tag | Impact |
|-----------|-----------|--------|
| PatientPosition | (0018,5100) | HFS, FFS, etc. |
| GantryDetectorTilt | (0018,1120) | Affects slice alignment |
| SpiralPitchFactor | (0018,9311) | Table feed ratio; affects coverage/quality |

---

## Notebook Structure

### Section 1: Setup & Connection
- Install/import dependencies (google-cloud-bigquery, pandas, matplotlib, seaborn, scipy, scikit-learn)
- Configure BigQuery client with default credentials
- Helper functions for query execution and cost estimation
- **Visualization style:** Static plots using Matplotlib/Seaborn

### Section 2: Initial Exploration
- Count records at each DICOM hierarchy level (patient/study/series/instance)
- Check attribute completeness (NULL counts)
- Create series-level summary table

**Visualizations:**
- Bar chart of record counts
- Heatmap of attribute completeness

### Section 3: Distribution Analysis

#### 3.1 Spatial Resolution
- Query SliceThickness, PixelSpacing, SpacingBetweenSlices distributions
- Histograms for each parameter
- Box plots by manufacturer

#### 3.2 Contrast/Exposure
- Query KVP, Exposure, CTDIvol distributions
- Bar chart for KVP (discrete values: 80, 100, 120, 140)
- Box plots by manufacturer
- Scatter: Exposure vs CTDIvol (check correlation)

#### 3.3 Reconstruction Kernels
- Query ConvolutionKernel frequency by manufacturer
- Horizontal bar chart of top 20 kernels
- Stacked bar chart showing kernel-manufacturer combinations

#### 3.4 Hardware
- Query Manufacturer, ManufacturerModelName distributions
- Pie chart of manufacturer market share
- Bar chart of top scanner models

#### 3.5 Geometry
- Query PatientPosition, GantryDetectorTilt, SpiralPitchFactor
- Bar charts and histograms

### Section 4: Statistical Assumption Checking & Correlation Analysis

#### 4.1 Distribution Analysis (Before Statistical Tests)
- **Normality tests:** Shapiro-Wilk or D'Agostino-Pearson for continuous variables
- **Visual inspection:** Q-Q plots for continuous variables
- **Homoscedasticity:** Levene's test for variance equality across groups
- Based on results, choose appropriate tests:
  - If normal + homoscedastic → parametric (Pearson, ANOVA)
  - Otherwise → non-parametric (Spearman, Kruskal-Wallis)

#### 4.2 Correlation Analysis
- Correlation heatmap (Pearson or Spearman based on normality)
- Pair plots colored by manufacturer (subsample if needed)

#### 4.3 Group Comparisons
- For categorical associations: Chi-square test (check expected cell counts ≥ 5)
  - If expected counts < 5 in >20% cells → use Fisher's exact or combine categories
- For continuous across groups: ANOVA or Kruskal-Wallis based on assumptions
- Report effect sizes (eta-squared, Cramer's V)

### Section 5: Clustering for Representative Subsets
- Feature engineering (normalize continuous, encode categorical)
- Optimal cluster selection (elbow method, silhouette score)
- K-means clustering
- PCA visualization of clusters
- Cluster characterization (mean/mode of parameters per cluster)
- Select **5-10 representative series** closest to each centroid (~50-100 total)

### Section 6: Export
- Export cluster assignments (CSV)
- Export representative series UIDs (for download)
- Generate idc-index download script

---

## Key SQL Query Pattern

```sql
SELECT
  SeriesInstanceUID,
  Manufacturer,
  ManufacturerModelName,
  SAFE_CAST(SliceThickness AS FLOAT64) as SliceThickness,
  SAFE_CAST(KVP AS FLOAT64) as KVP,
  ConvolutionKernel,
  -- ... other attributes
  COUNT(*) as instance_count
FROM `bigquery-public-data.idc_current.dicom_all`
WHERE collection_id = 'nlst'
  AND Modality = 'CT'
GROUP BY SeriesInstanceUID, Manufacturer, ManufacturerModelName,
         SliceThickness, KVP, ConvolutionKernel, ...
```

---

## Statistical Tests

**Assumption checks performed first:**
- Normality: Shapiro-Wilk / D'Agostino-Pearson + Q-Q plots
- Homoscedasticity: Levene's test
- Chi-square validity: Expected cell counts ≥ 5

| Analysis | Test (if assumptions met) | Alternative (if violated) | Effect Size |
|----------|---------------------------|--------------------------|-------------|
| Continuous correlations | Pearson | Spearman | r or ρ |
| Categorical associations | Chi-square | Fisher's exact | Cramer's V |
| Group comparisons | One-way ANOVA | Kruskal-Wallis | eta-squared |

---

## Output Files
- `nlst_analysis_output/all_series_clusters.csv` - All series with cluster assignments
- `nlst_analysis_output/representative_series.csv` - Selected representative series with parameters
- `nlst_analysis_output/representative_series_uids.txt` - UIDs for download
- `download_representatives.py` - Script to download using idc-index

---

## Verification
1. Run all notebook cells without error
2. Verify BigQuery queries return expected data volumes
3. Check visualizations render correctly
4. Confirm cluster analysis produces interpretable groupings
5. Test export files are generated with correct format
6. Optionally: Download a few representative series and verify DICOM files

---

## File to Create
- `/Users/af61/github/nlst-exploration/nlst_ct_acquisition_analysis.ipynb`
