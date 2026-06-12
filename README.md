# Global Education Policy Clusters

This project clusters 165 countries by combining national education policy laws with macroeconomic and demographic indicators. The goal is to identify whether distinct country archetypes emerge when you look at legal guarantees for education access alongside economic development, urbanization, and labor market data.

## Getting the Data

The raw datasets are not distributed with this repo. Both are freely available and take only a few minutes to download.

**WORLD Education Laws 2023**

Go to: https://microdata.worldbank.org/catalog/8210

Click "Download microdata" and you will receive a zip folder. Extract it, take the Excel file inside, and place it in the project root. Rename it to `World_Edu.xls`.

**World Development Indicators (World Bank)**

Go to: https://databank.worldbank.org/source/world-development-indicators

Select all countries, set the year to 2022, and add the following series by their codes:

- `NY.GDP.PCAP.PP.CD` (GDP per capita, PPP)
- `SI.POV.GINI` (Gini index)
- `SE.XPD.TOTL.GD.ZS` (Government expenditure on education, % of GDP)
- `SP.URB.TOTL.IN.ZS` (Urban population, % of total)
- `SP.POP.0014.TO.ZS` (Population ages 0 to 14, % of total)
- `SP.POP.1564.TO.ZS` (Population ages 15 to 64, % of total)
- `SP.POP.65UP.TO.ZS` (Population ages 65 and above, % of total)
- `SL.TLF.CACT.FE.ZS` (Labor force participation rate, female)
- `SL.TLF.CACT.MA.ZS` (Labor force participation rate, male)

Download as xlsx. Rename the file to `WDI.xlsx` and place it in the project root.

## Setup

```
pip install pandas numpy scikit-learn matplotlib seaborn plotly openpyxl xlrd
```

## Running the Notebooks

Run them in this order:

1. `01_data_cleaning.ipynb` -- merges the two datasets, engineers features, and saves `processed/cleaned_global_edu_policy.csv`
2. `02a_gmm.ipynb` -- runs PCA and Gaussian Mixture Model clustering, saves `processed/gmm_clusters.csv` and several plots
3. `02b_kmeans.ipynb` -- runs K-Means on the same PCA space for comparison, saves `processed/kmeans_clusters.csv` and comparison plots

## What This Project Does

### Data and Feature Engineering

The two datasets were merged on ISO country codes using an inner join, producing 193 countries with overlap across both sources. After dropping rows missing any of the eight core modeling features, the final sample came to 165 countries.

From the WDI, I kept GDP per capita (PPP) and urban population share as direct measures of economic development and urbanization. Rather than feeding in the three raw population age-share variables, I engineered a youth dependency ratio (the 0-14 population divided by the 15-64 population), which more directly captures the education demand pressure a country faces. The elderly population share was left out because it does not meaningfully affect primary education policy demand.

For gender inequality, I computed an LFP gender gap (male labor force participation rate minus female), which serves as a proxy for the expected economic return on female education. A country with a wide gap has a labor market where women remain substantially excluded regardless of whatever legal protections exist.

From the WORLD dataset, I selected one variable per policy domain to avoid redundancy:

- `finbar_prim`: tuition-free primary education guarantee
- `compend_prim`: compulsory primary education mandate
- `disc_sex_prim`: gender anti-discrimination protections in education
- `incl_edu`: disability inclusion guarantee

A fifth variable (`sh_edu`) was excluded because it overlaps heavily with `disc_sex_prim`. Each ordinal variable was remapped from its original sparse coding scheme (values like 1, 3, 4, 5) to consecutive integers starting from 0. This removes artificial numerical gaps in the scale while keeping the original ordering intact.

Two variables present in the raw WDI pull were excluded from the model: `gini_index` (62.7% missing, which would have shrunk the sample to 72 countries) and `gov_edu_spend` (19.7% missing, which would have cost another 29 countries). Both variables are still retained in the cleaned CSV for reference.

### Dimensionality Reduction

All eight features were standardized to zero mean and unit variance before PCA. Four principal components were retained, together accounting for 74.82% of total variance.

| Component | Variance Explained |
|-----------|-------------------|
| PC1 | 28.82% |
| PC2 | 19.43% |
| PC3 | 15.86% |
| PC4 | 10.71% |

### GMM Clustering

BIC was used to guide the choice of cluster count across k = 2 to 8. The first local minimum in BIC appeared at k = 3, where a group of exactly 7 countries separated from the rest. This same 7-country group appeared consistently at k = 3, 4, and 5, confirming it was not a modeling artifact. k = 4 was selected to preserve this group while splitting the remaining 158 countries into three well-sized, interpretable clusters.

The model converged with an overall mean assignment probability of 0.980. Only 6 of 165 countries were assigned with confidence below 0.80.

**GMM Cluster Profiles**

| Cluster | Archetype | n | Mean GDP per Capita (PPP) | Mean Urban | Mean LFP Gap |
|---------|-----------|---|--------------------------|------------|--------------|
| 0 | High-Income / Complete Legal Protections | 34 | $59,192 | 76.2% | 11.7 pp |
| 1 | Middle-to-High Income / High Gender Equity Gaps | 50 | $30,494 | 60.1% | 24.3 pp |
| 2 | Lower-Middle Income / Progressive Education Laws | 74 | $14,018 | 55.6% | 20.5 pp |
| 3 | Low-Income / Severe Legislative Gaps | 7 | $8,752 | 37.1% | 8.0 pp |

**Cluster 0 (n=34)** holds high-income countries with full or near-full legislative guarantees across all four policy domains. All G7 members except Canada land here, along with most of the European Union and a handful of developed economies from other regions. Representative countries: Australia, France, Germany, Italy, Japan, New Zealand, Spain, United Kingdom, United States of America.

**Cluster 1 (n=50)** holds middle-to-high income countries that have formal education protections on paper but show a wide gap between male and female labor force participation. The defining characteristic is not weak law but weak labor market translation of that law into outcomes for women. Representative countries: Austria, Canada, Norway, Portugal, Singapore, Switzerland, and most Gulf states. Canada lands here rather than in Cluster 0 because its position in the PCA space reflects a higher measured gender gap than the rest of the G7.

**Cluster 2 (n=74)** is the largest group, covering lower-middle income developing economies that nonetheless have adopted strong progressive education law frameworks. Countries here include Brazil, India, Mexico, Argentina, Indonesia, Vietnam, Egypt, and the majority of sub-Saharan Africa and Latin America.

**Cluster 3 (n=7)** is a small outlier group defined by a legislative gap rather than by income level alone. Despite Botswana sitting at roughly $20,000 GDP per capita and Burundi below $1,200, all seven countries share weak or absent guarantees on compulsory education and tuition-free primary access. Countries: Bhutan, Botswana, Burundi, Papua New Guinea, Solomon Islands, South Africa, Vanuatu.

### K-Means Comparison

K-Means was run on the same four-component PCA space. Silhouette scores peaked at k = 7, but k = 4 was chosen to allow a direct comparison against GMM.

**K-Means Cluster Profiles**

| Cluster | Archetype | n | Mean GDP per Capita (PPP) | Mean Urban | Mean LFP Gap |
|---------|-----------|---|--------------------------|------------|--------------|
| 0 | Critical legislative gaps | 7 | $8,752 | 37.1% | 8.0 pp |
| 1 | High income and developed economies | 72 | $51,248 | 76.3% | 15.8 pp |
| 2 | Developing economies with high gender labor exclusion | 38 | $13,117 | 51.1% | 35.0 pp |
| 3 | Low and lower-middle income, baseline gender inclusion | 48 | $8,046 | 47.4% | 13.9 pp |

K-Means Cluster 0 contains the exact same 7 countries as GMM Cluster 3. Two completely different algorithms, applied independently to the same data, isolated the same group with perfect agreement.

**Cross-Algorithm Agreement**

The Adjusted Rand Index between GMM and K-Means assignments was 0.1891, indicating limited overall agreement. The cross-tabulation shows where the disagreement sits:

| | KM 0 | KM 1 | KM 2 | KM 3 |
|---|------|------|------|------|
| **GMM 0** | 0 | 34 | 0 | 0 |
| **GMM 1** | 0 | 17 | 23 | 10 |
| **GMM 2** | 0 | 21 | 15 | 38 |
| **GMM 3** | 7 | 0 | 0 | 0 |

GMM Cluster 0 maps entirely to KM Cluster 1, and GMM Cluster 3 maps entirely to KM Cluster 0. The two algorithms agree completely on both the high-income group and the legislative gap outliers. The low ARI comes entirely from disagreement on how to divide the 124 middle-income countries that GMM places in Clusters 1 and 2. GMM separates them primarily by gender gap profile; K-Means draws a different boundary in the same PCA space. The outlier cluster is not part of that ambiguity at all.

## References

WORLD Policy Analysis Center (WORLD). WORLD Education Laws 2023 [dataset]. Version 1. Los Angeles: WORLD Policy Analysis Center [producer], 2025. Cape Town: DataFirst [distributor], 2025. DOI: https://doi.org/10.25828/e659-w175

World Bank. World Development Indicators [dataset]. Washington, D.C.: The World Bank Group. Available at: https://databank.worldbank.org/source/world-development-indicators (2022 data).
