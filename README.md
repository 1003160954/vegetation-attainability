# Vegetation Cooling Potential and Attainability

Code accompanying the manuscript **"Climate Change Intensifies Urban Vegetation Cooling Potential but Limits Reliable Attainment"**.

This repository contains a Google Colab / Google Drive workflow for building a harmonized global urban cooling dataset, training a physics-guided land-surface-temperature model, estimating low-vegetation counterfactual cooling, and reproducing the main analyses of cooling potential, active cooling and strong-cooling attainability.

## Data Access

The training dataset used in the manuscript is available from Figshare:

https://figshare.com/s/b3eba93a87f51c558a39

Public source datasets used to build the data cube include MODIS EVI/LAI/LST and land cover products, ERA5-Land, MERRA-2, VIIRS nighttime lights, SRTM elevation, and JRC Global Surface Water. The Google Earth Engine scripts in this repository can be used to rebuild the raster stacks from public data.

## Repository Contents

| File | Purpose |
| --- | --- |
| `1. GEE data obtian` | Google Earth Engine export scripts for ERA5-Land, MODIS land cover, DEM, EVI, LAI, LST, nighttime lights, and MERRA-2 raster products. |
| `1.1. (GEE) MERRA2 data was obtained based on ERA5 point data and summarized into tabular data. data was obtained based on ERA5 point data` | Google Earth Engine script for extracting MERRA-2 values at matched ERA5 sample points. |
| `2. Dataset construction for training (ERA5)` | Builds `GlobalUrbanCooling_dataset_v2.parquet` from exported city GeoTIFF stacks. |
| `3. Based on ERA5 location data, the merra2 data is summarized into a tabular data.` | Deduplicates MERRA-2 extraction points, merges yearly MERRA-2 CSV files, validates ERA5/MERRA-2 consistency, and creates hybrid ERA5 + MERRA-2 data products. |
| `4. Nonlinear Analysis of Different Variables` | Main counterfactual modelling workflow. It trains the XGBoost LST model, applies the EVI=0.05 low-vegetation counterfactual, computes cooling strength and cooling efficiency, saves `GlobalUrbanCooling_dataset_v3_EVI005.parquet` and `_with_lat.parquet`, and then runs nonlinear response curves and active-cooling decomposition analyses. |
| `5. Draw the main image` | Main-figure workflows, including two-source cooling-efficiency comparison, potential-attainability curves, active cooling analysis, and city-level summaries. |
| `6. Uncertainty and robustness experiments` | Robustness analyses for counterfactual baseline, threshold dependence, model alternatives, collinearity, data-source uncertainty, and reduced Fig. 4 analysis. |
| `7. Unequal approach to the summer high-radiation peak` | Relative-to-peak and high-radiation attainability analyses. |
| `8. observation-only common-support benchmark` | Observation-only common-support benchmark for validating the model-based counterfactual framework. |

## Recommended Environment

The scripts were written as Colab-style research notebooks/scripts. For the easiest reproduction, use Google Colab with Google Drive mounted.

Install the main Python dependencies:

```bash
pip install numpy pandas scipy matplotlib seaborn scikit-learn xgboost shap catboost rasterio rioxarray xarray pyarrow fastparquet joblib statsmodels pillow tqdm
```

For local execution, remove Colab-specific lines such as `drive.mount(...)` and notebook shell commands such as `!pip install ...`, then replace all `/content/drive/MyDrive/Second_research/...` paths with local paths.

## Quick Reproduction from Figshare Data

1. Download the Figshare dataset and place it under a working directory, for example:

```text
/content/drive/MyDrive/Second_research/
  derived_datasets/
  MERRA/
  data/
```

2. Update path variables near the top of the scripts, especially:

```python
ROOT_DATA = "/content/drive/MyDrive/Second_research/data"
SAVE_DIR = "/content/drive/MyDrive/Second_research/derived_datasets"
DATA_PATH = "/content/drive/MyDrive/Second_research/derived_datasets/GlobalUrbanCooling_dataset_v2.parquet"
```

3. Run the workflows in this order:

```text
2. Dataset construction for training (ERA5)
3. Based on ERA5 location data, the merra2 data is summarized into a tabular data.
4. Nonlinear Analysis of Different Variables
5. Draw the main image
6. Uncertainty and robustness experiments
7. Unequal approach to the summer high-radiation peak
8. observation-only common-support benchmark
```

The updated `4. Nonlinear Analysis of Different Variables` script is the key bridge between the raw tabular dataset and the downstream analyses. It reads:

```text
GlobalUrbanCooling_dataset_v2.parquet
```

and writes:

```text
GlobalUrbanCooling_dataset_v3_EVI005.parquet
GlobalUrbanCooling_dataset_v3_EVI005_with_lat.parquet
```

Downstream figure and robustness scripts should use the v3 EVI005 dataset when they require `cooling_strength_pred`, `dEVI_ref`, `cooling_efficiency`, or `lat`.

## Counterfactual Model

Land surface temperature is modelled with an XGBoost gradient-boosted regression tree model using histogram-based tree construction. The target variable is MODIS land surface temperature. Predictors include vegetation state variables (`EVI`, `LAI`), meteorological background variables (`T2M_C`, `TD_C`, `VPD_kPa`, `WIND10M_MS`, `SSRD_MJm2`, `TP_mm`, `SWVL1_m3m3`, `STL1_C`), and static geographic or urban-context variables (`DEM`, `LC`, `DIST_WATER_km`).

The main counterfactual model uses a low-vegetation reference:

```python
EVI_BASELINE = 0.05
MIN_DEVI_FOR_EFFICIENCY = 0.02
```

Cooling strength is computed as:

```text
CS = LST_pred(EVI_ref = 0.05) - LST_pred(EVI_observed)
```

Cooling efficiency is computed only when the vegetation contrast is large enough:

```text
CE = CS / (EVI_observed - EVI_ref), for EVI_observed - EVI_ref >= 0.02
```

The main model applies monotonic constraints as a physical prior for counterfactual interpretation:

```python
("EVI", -1), ("T2M_C", +1), ("STL1_C", +1), ("VPD_kPa", +1)
```

This means that, conditional on the other predictors in the model, increasing EVI cannot increase predicted LST, while increases in selected thermal background variables cannot decrease predicted LST. This should be interpreted as a physics-guided counterfactual assumption for estimating vegetation cooling capacity, not as a claim that vegetation always cools the surface under every real-world condition.

## Main Derived Variables

| Variable | Meaning |
| --- | --- |
| `LST_pred_obs` | Model-predicted LST under observed vegetation state. |
| `LST_pred_ref` | Model-predicted LST after setting EVI to the low-vegetation reference. |
| `EVI_baseline_used` | Low-vegetation reference value, usually `0.05`. |
| `cooling_strength_pred` | Counterfactual cooling strength, `LST_pred_ref - LST_pred_obs`. |
| `dEVI_ref` | Vegetation contrast, `EVI_observed - EVI_ref`. |
| `cooling_efficiency` | Unit-normalized cooling response, `cooling_strength_pred / dEVI_ref`. |
| `lat` | City latitude merged from the city metadata table. |

## Main Analyses in Script 4

After computing the v3 counterfactual dataset, `4. Nonlinear Analysis of Different Variables` also runs several diagnostic and analytical blocks:

- background-conditioned nonlinear response curves of `cooling_efficiency` against 13 drivers under selected `T2M_C x VPD_kPa` backgrounds;
- city-specific summer definition using the top 3 hottest baseline months;
- active cooling classification using the city baseline-summer CE quantile;
- yearly global `P(active)` trends with bootstrap confidence intervals over cities;
- decomposition of `P(active)` into background-frequency and within-background activation components.

Several later blocks keep results in memory as objects such as `dataset`, `response`, `diagnostics`, `summary`, `summer`, `cybg`, `cy_active`, and `trend`. If running in a fresh notebook session, load the saved v3 parquet first before running later blocks.

## Full From-Source Data Construction

Use the Google Earth Engine scripts when rebuilding the full dataset from public remote-sensing products.

1. In Earth Engine, edit the city parameters:

```javascript
var CITY_NAME = 'Guangzhou';
var CITY_LON = 113.2644;
var CITY_LAT = 23.1291;
var CITY_BUFFER_KM = 80;
```

2. Export harmonized stacks at `DEG_PER_PIXEL = 0.005` for each city. The scripts export 8-day stacks for 2002-2025 and align products to a common EPSG:4326 grid.

3. For MERRA-2 validation, first generate `merra2_match_points.csv`, upload the deduplicated point table to Earth Engine, and run the MERRA-2 point extraction script.

4. After all GeoTIFF and CSV exports are available in Google Drive, run script `2`, then script `3`, then script `4`.

## Robustness and Sensitivity

The robustness scripts evaluate data-source uncertainty, threshold dependence, model alternatives, collinearity, alternative low-vegetation references, and observation-only common-support benchmarks. For constraint sensitivity, compare the main EVI-constrained model with alternative specifications such as:

- no EVI monotonic constraint;
- no monotonic constraints;
- removing LAI to test EVI/LAI feature competition;
- changing both EVI and LAI to low-vegetation reference values;
- observation-only common-support matching.

These analyses are important because high predictive accuracy for LST does not by itself guarantee reliable EVI counterfactual responses.

## Notes for Reuse

The repository is organized as research notebook-style scripts rather than a command-line package. Many files contain multiple analysis blocks and assume that earlier notebook variables already exist in memory. When rerunning the analysis, execute blocks in order and verify that required intermediate objects and files exist before moving to the next block.

For new applications, the minimum practical workflow is:

1. Build or obtain a harmonized city data table with `city`, `date`, `EVI`, `LAI`, `LST`, climate variables, and static context variables.
2. Train the physics-guided LST response model or an explicitly documented alternative.
3. Predict observed LST and low-vegetation counterfactual LST with non-vegetation predictors held fixed.
4. Compute CS, CE, active cooling, HighCE, potential, and attainability summaries.
5. Bootstrap city-level summaries to quantify between-city uncertainty.

