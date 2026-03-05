# Data Science Portfolio

Welcome to my data science portfolio. This repository stores my personal data science projects. My goal is to improve my skills in exploratory data analysis, statistics, and machine learning by working with real-world datasets.

## Projects

---

### Climate Predictor
Located in the `Climate_predictor` folder.

An end-to-end analysis of global climate data covering **158 countries from 1950 to 2022**, investigating whether CO2 emissions and geographic features can predict country-level temperature anomalies.

**Dataset**
- One row per country per year (~9,200 observations)
- Features: average temperature, CO2 per capita, annual CO2 emissions, humidity, precipitation, capital coordinates, population density
- Target: temperature anomaly relative to each country's 1950-1980 baseline

**Exploratory Data Analysis**
- Time-series analysis of CO2 and global temperature trends (1950-2022)
- Sea level rise correlation: confirmed cointegration between annual CO2 emissions and sea level rise (Engle-Granger p = 0.017), while temperature correlation was found to be spurious (p = 0.51)
- Correlation heatmaps on both raw and yearly-aggregated data to separate physical relationships from time-trend artifacts
- Distribution analysis with log-transformations for heavily right-skewed CO2 features

**Clustering and Dimensionality Reduction**
- PCA on country climate profiles (73.2% variance in 2 components): PC1 captures an aridity + development axis, PC2 separates humid tropics from arid zones
- K-Means clustering (k=3, silhouette score 0.53) recovered three geographically coherent climate zones without using coordinates during fitting: *Tropical humid / low-CO2*, *Arid / mixed-CO2*, *High-latitude / industrial*

**Temperature Anomaly Analysis**
- Global mean anomaly has been permanently above the 1950-1980 baseline since ~1990, reaching +1.35 C by 2022
- Arctic amplification confirmed: the high-latitude / industrial cluster warms ~0.8 C faster than tropical regions by 2022

**Models Compared**

| Model | RMSE | R2 | Data |
|:------|-----:|----:|:-----|
| Linear Regression | 1.37 C | 0.01 | country-level |
| Ridge + Interaction (year x CO2) | 1.37 C | 0.01 | country-level |
| MLP Neural Network | 1.38 C | 0.001 | country-level |
| **Random Forest (tuned)** | **1.11 C** | **0.35** | **country-level** |
| Linear Regression (global yearly) | 0.19 C | -0.21 | global yearly |
| Polynomial Regression | 0.43 C | -4.99 | global yearly |
| Random Forest (global yearly) | 0.25 C | -1.05 | global yearly |
| Ridge + Interaction (global yearly) | 0.19 C | -0.17 | global yearly |
| **MLP Neural Network (global yearly)** | **0.15 C** | **0.25** | **global yearly** |
| Linear Regression (5-year means) | 0.16 C | n/a* | global 5-year |

*R2 is not meaningful with only 2 test points.*

**Key Findings**
- The **Random Forest** (R2 0.35) is the best model for country-level prediction, with `year` and `log_co2_per_capita` as the top two features -- confirming CO2 adds genuine signal beyond the time trend
- Tree-based models cannot extrapolate: the RF flatlines at ~1.0 C post-2010 while actual warming reaches 1.35 C. MLP and Ridge + interaction were the only models to track the continued upward trend
- Bootstrap analysis revealed that adding the `year x CO2` interaction term without regularisation destroys model stability. Ridge regularisation resolves the multicollinearity and lets the interaction term carry real signal (coefficient 0.06, stable across bootstrap samples)
- 5-year smoothing reduces noise (RMSE 0.16 C) but CO2 features become completely unstable at 14 data points -- bootstrap std is ~9x the mean and the coefficient flips sign

---

### Abalone Analysis (PCA and Deep Learning)
Located in the `Abalone` folder.

This project combines dimensionality reduction with neural networks.
- **Principal Component Analysis (PCA):** Implemented PCA from scratch to understand the mathematics behind feature extraction and dimensionality reduction.
- **Deep Learning:** Trained a Multi-Layer Perceptron (MLPClassifier) on the PCA-transformed data to predict the gender of abalones.
- **Evaluation:** Detailed steps on checking for overfitting, evaluating model accuracy, and analyzing training loss history.

---

### Genome Analytics (IGSR Samples)
Located in the `Genome_Project` folder.

This project focuses on cleaning and analyzing biological metadata from IGSR samples.
- **Data Cleaning:** Processed raw TSV data and handled missing values across complex biological attributes.
- **Exploratory Analysis:** Analyzed population distributions and demographics across different continents.
- **Visualization:** Created clear, informative visualizations to communicate findings effectively.

---

*More projects will be added as I learn new techniques and tools.*
