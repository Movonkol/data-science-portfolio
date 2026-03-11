# Global Climate Predictor — Can We Model a Century of Warming?

> *158 countries. 72 years. One question: what actually predicts temperature anomalies?*

Five models. Country-level annual data from 1950–2022. The answer is both simpler and more frustrating than expected: `year` explains almost everything, CO₂ explains almost nothing at this granularity, and the best model still misses by over 1°C on unseen data.

---

## What This Project Does

We build a global dataset combining weather station records, CO₂ emissions, population density, humidity, and precipitation for 158 countries from 1950 to 2022. We then attempt to predict **temperature anomalies** — each country's deviation from its own 1950–1980 baseline — using machine learning models trained on pre-2010 data and evaluated on 2011–2022.

The goal is not just prediction accuracy. It's understanding *what* drives warming at country level and *how much* the available features can explain.

---

## Dataset

Built from five sources merged on country code and year:

| Source | Features |
|---|---|
| Open-Meteo / Kaggle weather stations | `avg_temp_c`, `min_temp_c`, `max_temp_c` |
| Our World in Data | `co2_per_capita`, `annual_co2_emissions` |
| World Bank | `precipitation_mm_wb`, `Cap_density` (capital city) |
| Climate Excel dataset | `humidity` |
| Capitals geocoding | `capital_lat`, `capital_lng`, `hemisphere` |

**Final dataset:** 9,166 rows × 14 columns, no missing values. One row per country per year.

**Target variable:** `temp_anomaly = avg_temp_c − baseline_temp`, where `baseline_temp` is the per-country mean over 1950–1980. This measures warming relative to a pre-acceleration reference period rather than raw temperature, eliminating the geographic bias (tropical countries are always warmer than arctic ones regardless of trend).

---

## Exploratory Analysis

### CO₂ and temperature — correlated but not causally separable

![CO₂ and Temperature Timeseries](Data/plots/01_EDA/01_co2_temperature_timeseries.png)

Both global mean temperature and CO₂ emissions rise monotonically after ~1970. The correlation is strong (r ≈ 0.97 on global yearly aggregates), but this alone says nothing about causation — `year` correlates equally well with both, and the three variables are nearly collinear across the 1950–2022 window.

![CO₂ vs Temperature Regression](Data/plots/01_EDA/02_co2_temperature_regplot.png)

The scatter shows a clear positive relationship, but the confidence band widens substantially toward high CO₂ values. A handful of high-emission, high-latitude countries (Canada, Russia, Gulf states) drive the right tail.

### Sea level — a correlated but non-causal variable

![Sea Level Rise](Data/plots/01_EDA/03_sea_level_rise.png)

![Temperature vs Sea Level Correlation](Data/plots/01_EDA/04_temp_sea_level_correlation.png)

Temperature and sea level rise are strongly correlated (r ≈ 0.8), but cointegration tests show no stable long-run equilibrium relationship. Both are driven by a shared global forcing — time itself, acting as a proxy for cumulative anthropogenic impact — rather than one causing the other.

![CO₂ vs Sea Level Correlation](Data/plots/01_EDA/05_co2_sea_level_correlation.png)

CO₂ and sea level tell the same story: high bivariate correlation, near-zero first-difference correlation. The level relationship is spurious; the rate relationship is absent.

### Correlation structure

![Correlation Heatmap Raw](Data/plots/01_EDA/06_correlation_heatmap_raw.png)

![Correlation Heatmap Yearly](Data/plots/01_EDA/07_correlation_heatmap_yearly.png)

Raw (country-year level) correlations show moderate relationships between temperature, latitude, and humidity. The global yearly aggregate heatmap is dramatically stronger — aggregating across countries removes cross-sectional noise and exposes the shared temporal trend. This is the first hint that country-level models will underperform global ones.

### Distributions and transformations

![Distributions](Data/plots/01_EDA/08_distributions_histograms.png)

Temperature is roughly uniform across countries (excess kurtosis ≈ −1 — bounded by geography). CO₂ per capita, precipitation, and capital density are heavily right-skewed: a few high-emission or high-rainfall countries dominate the raw scale.

![QQ Plots Raw](Data/plots/01_EDA/09_qq_plots.png)

![QQ Plots Log Transform](Data/plots/01_EDA/10_qq_plots_log_transform.png)

Log transformations substantially improve normality for `co2_per_capita`, `annual_co2_emissions`, `precipitation_mm_wb`, and `Cap_density`. Both Q-Q plots show the improvement — the tails collapse from heavy to near-normal after log scaling.

### Geographic distribution

![Temperature Map](Data/plots/01_EDA/11_temperature_map.png)

Mean temperature across the 158 countries shows the expected latitude gradient: tropical Africa and Southeast Asia cluster near 25–30°C, northern Europe and Canada near 0–10°C. The geographic spread is large (−11°C to 34°C) which is why anomalies (not raw temperatures) are used as the modeling target.

![Hemisphere CO₂](Data/plots/01_EDA/12_hemisphere_co2.png)

![Hemisphere Temperature](Data/plots/01_EDA/13_hemisphere_temperature.png)

Northern hemisphere countries emit substantially more CO₂ per capita and run warmer on average. The NE quadrant (Northern/Eastern hemisphere) contains most of the high-emission industrialized countries. This geographic signal will reappear as `capital_lat` dominating Random Forest feature importance.

---

## Clustering

### PCA of country climate profiles

![PCA Climate Drivers](Data/plots/02_Clustering/01_pca_climate_drivers.png)

PCA on 6 climate features (temperature, precipitation, humidity, CO₂, latitude, density) with 2 components explaining 83% of variance:

- **PC1** (61%): temperature/latitude gradient — hot, low-latitude, high-density capitals score high (Kuwait, Qatar, Singapore). Cold, high-latitude countries score low (Iceland, Norway, Canada).
- **PC2** (22%): precipitation axis — wet tropical countries (Malaysia, Indonesia, Colombia) score high. Arid countries (Egypt, Saudi Arabia, Mongolia) score low.

The geographic projection validates both components without using coordinates as inputs.

![PC1 World Map](Data/plots/02_Clustering/02_pca_pc1_map.png)

![PC2 World Map](Data/plots/02_Clustering/03_pca_pc2_map.png)

PC1 produces a clean latitude gradient on the world map — spatially contiguous countries cluster together despite coordinates not being PCA inputs. PC2 separates the Amazon basin and Southeast Asia (high precipitation) from the Sahara belt and Central Asia (low precipitation).

### K-Means clustering

![Elbow and Silhouette](Data/plots/02_Clustering/04_kmeans_elbow_silhouette.png)

Optimal k=3 by both elbow method and silhouette score (0.53 — moderate to good separation). Beyond k=3 the silhouette score declines and the elbow flattens.

![K-Means Clusters PCA](Data/plots/02_Clustering/05_kmeans_clusters_pca.png)

Three climate regimes emerge clearly in PC1–PC2 space:

| Cluster | Label | Characteristics | Example countries |
|---|---|---|---|
| 0 | Tropical | High temp, high humidity, high precip | India, Thailand, Nigeria, Brazil |
| 1 | Temperate | Moderate temp, moderate precip | UK, Germany, Japan, USA |
| 2 | Arid/Cold | Low temp or low precip | Canada, Egypt, Mongolia, Russia |

These clusters are used as a categorical feature in all downstream models.

---

## Temperature Anomaly Trends

### Global warming signal

![Global Mean Anomaly](Data/plots/03_Anomaly/01_global_mean_anomaly.png)

Global mean temperature anomaly reaches +1.3°C by 2022 relative to the 1950–1980 baseline. The trend is gradual from 1950–1980 (by construction, since the baseline period mean is subtracted), then accelerates sharply post-1980. The acceleration post-2000 is visible even in the country-level mean — this is not a smoothing artifact.

### Anomaly by climate cluster

![Anomaly by Cluster](Data/plots/03_Anomaly/02_anomaly_by_cluster.png)

Warming is not uniform across climate regimes. Temperate and arid/cold clusters show the strongest anomaly increase (consistent with Arctic amplification and mid-latitude warming). The tropical cluster shows a weaker absolute anomaly — partly because tropical climates have lower interannual variability to begin with, not because they are warming less.

---

## Models

**Setup:** Train on all country-years ≤ 2010 (6,976 rows), test on 2011–2022 (1,673 rows). No random shuffle — the split is temporal to avoid data leakage from future warming patterns into training.

**Features:** `log_co2_per_capita`, `year`, `humidity`, `log_precipitation`, `log_cap_density`, `capital_lat`, `capital_lng`, `cluster_name`, `hemisphere`

### Linear Regression — the baseline

![Linear Regression Residuals](Data/plots/04_Models/01_linear_regression_residuals.png)

![Linear Bootstrap Coefficients](Data/plots/04_Models/02_linear_global_bootstrap_coefs.png)

**Country-level:** RMSE 1.37°C, R² 0.012. The model essentially predicts the mean — it captures the global trend via the `year` coefficient but fails on country-level variance. Bootstrap shows year coefficient is stable (std ≈ 0.001°C/year), but CO₂ and humidity coefficients have high variance and occasionally flip sign.

**Global aggregation** (yearly means across all countries): RMSE 0.19°C on 73 points. Aggregation removes country-level noise and the model fits the trend well, but R² = −0.21 on the 12-point test set — the denominator (total variance in test years) is tiny.

### Polynomial Regression — where it goes wrong

![Polynomial Bootstrap Coefficients](Data/plots/04_Models/03_polynomial_regression_bootstrap_coefs.png)

![Polynomial Predictions](Data/plots/04_Models/04_polynomial_regression_predictions.png)

Polynomial features (degree 2) on year, year², and three climate features. Training RMSE improves, but the quadratic term extrapolates unrealistically: the fitted parabola accelerates beyond the actual warming trajectory post-2010, overshooting by ~0.5°C by 2022.

The bootstrap makes this concrete: the year² coefficient is unstable across resamples (std comparable to mean), meaning the curvature is not a reliable signal in the data — it is fitting historical noise. **Negative R² (−4.99) on the test set.** Polynomial extrapolation is not suitable for this data.

### Random Forest — best overall

![Random Forest Feature Importances](Data/plots/04_Models/05_random_forest_importances.png)

![Random Forest Residuals](Data/plots/04_Models/06_random_forest_residuals.png)

**Country-level:** RMSE 1.11°C, R² 0.348. The ceiling is clear: even 300 trees with depth-15 splits explain only 35% of country-level anomaly variance. The remaining 65% is natural interannual variability not captured in the feature set.

Feature importance is unambiguous:

| Feature | Importance | Bootstrap std |
|---|---|---|
| `year` | 0.45 | 0.020 |
| `capital_lat` | 0.18 | 0.015 |
| `humidity` | 0.14 | 0.012 |
| `log_co2_per_capita` | 0.09 | 0.008 |
| `capital_lng` | 0.06 | 0.006 |

`year` alone accounts for nearly half the model's splits. Latitude follows because different latitude bands warm at different rates. CO₂ per capita contributes, but weakly — local emissions are a poor predictor of local anomaly because global forcing dominates.

![Random Forest Tree](Data/plots/04_Models/07_random_forest_tree.png)

A single illustrative tree shows the decision structure: the root split is on `year` (~2000), the next level separates by latitude, and CO₂ appears only deeper in the tree.

![Random Forest Global Predictions](Data/plots/04_Models/08_random_forest_global_predictions.png)

On globally aggregated yearly data the Random Forest tracks the trend reasonably well but shows the characteristic limitation of tree-based models: it cannot extrapolate beyond the training range. Post-2010 predictions plateau rather than continuing to rise, which is why the global MLP (which can extrapolate) outperforms it on global metrics.

### Neural Network (MLP)

![MLP Residuals](Data/plots/04_Models/09_mlp_residuals.png)

![MLP Training Loss](Data/plots/04_Models/10_mlp_training_loss.png)

**Country-level:** RMSE 1.37°C, R² 0.018 — no better than linear regression. The training loss curve shows the model learning well on pre-2010 data, but the gap between training and validation loss is substantial — classic overfitting.

![MLP Permutation Importances](Data/plots/04_Models/11_mlp_permutation_importances.png)

Permutation importance from the MLP largely agrees with the Random Forest: `year` and `capital_lat` dominate. The ordering is similar but the MLP assigns more weight to humidity, suggesting it learned a different climate interaction than the RF.

![MLP Global Predictions](Data/plots/04_Models/12_mlp_global_predictions.png)

On global aggregated data the MLP performs best: RMSE 0.15°C, R² 0.25 on 12 test years. The MLP correctly continues the warming trend post-2010 rather than plateauing, which is the key advantage over the Random Forest at the global scale. That said, 12 test points is a small basis for confidence.

### Ridge Regression with Interaction

![Ridge Global Predictions](Data/plots/04_Models/13_ridge_global_predictions.png)

![Ridge Bootstrap Coefficients](Data/plots/04_Models/14_ridge_bootstrap_coefs.png)

Adding a `year × log_co2_per_capita` interaction term to Ridge regression tests whether CO₂'s effect on temperature has changed over time. The result: the interaction coefficient's bootstrap standard deviation is 9× its mean — the sign flips across resamples. There is no stable interaction signal in this data. The interaction term makes the model *worse* (RMSE 0.33°C, R² −2.56 on the interaction version).

The Ridge model without interaction is the most interpretable physically: the `year` coefficient (0.02–0.03°C/year) represents the global trend, and the CO₂ and latitude coefficients are stable across bootstrap resamples.

### Model Comparison

![All Models Comparison](Data/plots/04_Models/15_all_models_comparison.png)

| Model | RMSE (test) | R² (test) | Notes |
|---|---|---|---|
| Linear Regression | 1.37°C | 0.012 | Baseline |
| Polynomial LR | 0.43°C | −4.99 | Extrapolation failure |
| **Random Forest** | **1.11°C** | **0.348** | Best country-level |
| Neural Network | 1.37°C | 0.018 | Overfits pre-2010 |
| Ridge + interaction | 1.37°C | 0.012 | No interaction signal |
| Global Linear | 0.19°C | −0.21 | 12 test points |
| Global RF | 0.25°C | −1.05 | Plateaus post-2010 |
| **Global MLP** | **0.15°C** | **0.25** | Best global RMSE |
| Global Ridge | 0.19°C | −0.17 | Most interpretable |

### 5-Year Means Analysis

![5-Year Linear Predictions](Data/plots/04_Models/16_linear_5yr_predictions.png)

![5-Year Bootstrap Coefficients](Data/plots/04_Models/17_linear_5yr_bootstrap_coefs.png)

![Yearly vs 5-Year Comparison](Data/plots/04_Models/18_yearly_vs_5yr_comparison.png)

Aggregating to 5-year means reduces noise substantially (RMSE 0.163°C vs 0.19°C for yearly linear). The 5-year bootstrap shows smoother coefficient distributions — but the CO₂ coefficient standard deviation remains 9× the mean, confirming it is not a stable predictor even after smoothing.

**R² on 5-year data is meaningless:** the temporal split at 2010 leaves only 2 test observations (2011–2015 and 2016–2020). R² on n=2 is mathematically unreliable — treat the RMSE and visual fit as the relevant metrics here.

---

## Key Findings

### 1. Year is everything, CO₂ is almost nothing

At country-year granularity, `year` explains ~45% of Random Forest splits. CO₂ per capita contributes ~9%. The reason: warming is a global phenomenon driven by cumulative atmospheric forcing, not local emissions. A country that doubles its CO₂ output doesn't become measurably warmer than its neighbors the same year. The relevant signal operates at decadal, global scale — and at that scale, `year` is a near-perfect proxy.

### 2. The ceiling at R² ≈ 0.35 is structural, not a modeling failure

Country-level annual temperature anomalies contain irreducible natural variability from ENSO cycles, volcanic forcing, the Atlantic Multidecadal Oscillation, and local weather noise — none of which are in the feature set. No amount of model tuning will push R² substantially above 0.35 with these features. The ceiling is informative: it tells us what fraction of warming can be explained by static geographic and slowly-varying economic factors vs. dynamic climate system variability.

### 3. CO₂ and year are too collinear to separate at this scale

Both `year` and `co2_per_capita` rise monotonically over 1950–2022. Granger causality, interaction terms, and bootstrap coefficient analysis all yield unstable estimates for CO₂'s isolated contribution. This isn't evidence that CO₂ doesn't cause warming — it's evidence that this observational dataset lacks the variation needed to disentangle the two at country-year granularity.

### 4. Global aggregation inflates performance metrics

Collapsing 158 countries to 73 yearly means removes enormous cross-sectional noise. MLP achieves RMSE 0.15°C on global data vs. 1.37°C on country data — a 9× improvement. But the global test set has only 12 observations. Any R² or RMSE comparison between global and country-level models should be treated with skepticism.

### 5. Polynomial extrapolation fails systematically

The quadratic year term fits the 1950–2010 acceleration well in-sample but produces a parabola that overshoots post-2010 reality by ~0.5°C. The warming trend post-2010 is actually closer to linear than quadratic at this time scale. R² = −4.99 is the clearest possible signal to stay away from polynomial features when extrapolating.

### 6. Random Forest can't extrapolate; MLP can

Tree-based models are bounded by the training range — they can't predict anomalies beyond values seen during training. Post-2010, the global anomaly exceeds the training maximum in many years, causing RF predictions to plateau. The MLP learns a smooth function and continues the trend. This is the main reason MLP wins at the global level despite worse country-level performance.

---

## Setup

### Environment

```bash
conda activate Genome_analysis  # or any env with the packages below
pip install pandas numpy scikit-learn matplotlib seaborn plotly scipy jupyter
```

### Run order

```
dataset_create.ipynb   # builds final_yearly_weather_df.csv from raw sources
Data_analysis.ipynb    # EDA → clustering → anomaly analysis → all models
```

---

## Project Structure

```
Climate_predictor/
├── README.md
├── dataset_create.ipynb          # Data pipeline: merge, clean, interpolate, export
├── Data_analysis.ipynb           # Full analysis: EDA, PCA, clustering, 5 models
└── Data/
    ├── final_yearly_weather_df.csv   # 9,166 rows × 14 cols (output of dataset_create)
    └── plots/
        ├── 01_EDA/               # 13 plots: timeseries, correlations, maps, distributions
        ├── 02_Clustering/        # 5 plots: PCA, world maps, elbow/silhouette, cluster viz
        ├── 03_Anomaly/           # 2 plots: global trend, anomaly by cluster
        └── 04_Models/            # 18 plots: residuals, coefficients, importances, predictions
```

## Dataset Columns

| Column | Description |
|---|---|
| `country`, `year`, `Code` | Identifiers |
| `avg_temp_c`, `min_temp_c`, `max_temp_c` | Annual temperature (°C) |
| `capital_lat`, `capital_lng` | Capital city coordinates |
| `hemisphere` | NE / NW / SE / SW |
| `co2_per_capita` | Tonnes CO₂ per person |
| `annual_co2_emissions` | Total national CO₂ (tonnes) |
| `Cap_density` | Capital city population density |
| `humidity` | Mean annual relative humidity (%) |
| `precipitation_mm_wb` | Mean annual precipitation (mm, World Bank) |

## Tech Stack

`pandas` · `numpy` · `scikit-learn` · `matplotlib` · `seaborn` · `plotly` · `scipy`
