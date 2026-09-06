# Power Outages and Economic Inequality

**By Carlos Lopez**

An investigation into whether economically disadvantaged states experience more severe power outages in the continental U.S. (January 2000 – July 2016).

## Introduction



This project investigates major power outages in the continental United States from January 2000 to July 2016. The central question is: **Are major power outages more severe in economically disadvantaged states than in wealthier states?**

This question matters because power outages disproportionately impact vulnerable communities. If lower-income states experience longer or more damaging outages, it suggests systemic issues with infrastructure investment and resource allocation that policymakers should address.

The dataset contains **1,534 rows** representing individual major outage events. The columns most relevant to this investigation are:

| Column | Description |
|--------|-------------|
| OUTAGE.DURATION | Duration of the outage in minutes |
| CUSTOMERS.AFFECTED | Number of customers affected |
| CAUSE.CATEGORY | Category of the cause of the outage |
| PC.REALGSP.STATE | Per capita real gross state product (GDP) |
| U.S._STATE | State where the outage occurred |
| CLIMATE.REGION | U.S. climate region |
| ANOMALY.LEVEL | Oceanic El Nino/La Nina index |

## Data Cleaning and Exploratory Data Analysis

### Data Cleaning

The raw data was provided as an Excel file with metadata rows at the top. I performed the following cleaning steps:

- Skipped the title and description rows and extracted the correct column headers
- Removed the Units row and the unused variables and OBS columns
- Combined OUTAGE.START.DATE and OUTAGE.START.TIME into a single start_datetime timestamp column, and did the same for restoration columns
- Created an income_group column by classifying each state as low or high income relative to the median PC.REALGSP.STATE within each year, so comparisons account for economic changes over time

Here is the head of the cleaned DataFrame:

| YEAR | MONTH | U.S._STATE | POSTAL.CODE | NERC.REGION | CLIMATE.REGION | ANOMALY.LEVEL | CLIMATE.CATEGORY | CAUSE.CATEGORY | OUTAGE.DURATION | income_group |
|------|-------|-----------|-------------|-------------|----------------|---------------|-----------------|----------------|-----------------|--------------|
| 2011 | 7 | Minnesota | MN | MRO | East North Central | -0.3 | normal | severe weather | 3060 | high |
| 2014 | 5 | Minnesota | MN | MRO | East North Central | -0.1 | normal | intentional attack | 1 | high |
| 2010 | 10 | Minnesota | MN | MRO | East North Central | -1.5 | cold | severe weather | 3000 | high |
| 2012 | 6 | Minnesota | MN | MRO | East North Central | -0.1 | normal | severe weather | 2550 | high |
| 2015 | 7 | Minnesota | MN | MRO | East North Central | 1.2 | warm | severe weather | 1740 | low |

### Univariate Analysis

The histogram below shows the distribution of outage causes. Severe weather is by far the most common cause, accounting for over 750 outages, followed by intentional attacks and system operability disruptions.

<iframe
  src="cause-plot.html"
  width="800"
  height="600"
  frameborder="0"
></iframe>

### Bivariate Analysis

The box plot below compares outage duration across the top three cause categories, split by income group. For severe weather outages, low-income states show a slightly higher median duration than high-income states.

<iframe
  src="assets/duration-by-income.html"
  width="800"
  height="600"
  frameborder="0"
></iframe>

### Interesting Aggregates

The table below shows mean outage duration (in minutes) by cause category and income group:

| CAUSE.CATEGORY | high | low |
|----------------|------|-----|
| equipment failure | 456 | 3,858 |
| fuel supply emergency | 15,217 | 5,811 |
| intentional attack | 419 | 443 |
| islanding | 196 | 213 |
| public appeal | 1,347 | 1,636 |
| severe weather | 3,677 | 4,074 |
| system operability disruption | 583 | 909 |

Equipment failure shows a striking gap, with low-income states averaging 3,858 minutes compared to 456 for high-income states. Severe weather outages also last longer in low-income states (4,074 vs 3,677 minutes).

## Assessment of Missingness

### NMAR Analysis

I believe CUSTOMERS.AFFECTED is likely **MNAR** (Missing Not at Random). Utilities may be less likely to report customer impact numbers for smaller or less notable outages, meaning the missingness is related to the value itself. Smaller customer counts are less likely to be reported. Additional data such as internal utility reporting policies could potentially explain this missingness and make it MAR.

### Missingness Dependency
<iframe
  src="assets/missingness-plot.html"
  width="800"
  height="600"
  frameborder="0"
></iframe>

I analyzed the missingness of CUSTOMERS.AFFECTED (443 missing values out of 1,534).

**Depends on CAUSE.CATEGORY (p-value = 0.0):** The rate of missing customer data varies dramatically by cause. Only 6% of severe weather outages are missing customer data, but 86% of fuel supply emergencies are. This makes sense because severe weather events are high-profile and well-tracked, while fuel supply issues may not involve direct customer impact measurement.

**Does not depend on POPPCT_URBAN (p-value = 0.128):** The mean urban population percentage is similar whether customer data is missing (81.7%) or not (80.7%). This suggests that how urban an area is does not significantly affect whether customer impact gets reported.

## Hypothesis Testing

**Question:** Are severe weather outages more severe in low-income states?

- **Null Hypothesis:** The average outage duration for severe weather outages is the same for low-income and high-income states. Any observed difference is due to random chance.
- **Alternative Hypothesis:** The average outage duration for severe weather outages is longer in low-income states.
- **Test Statistic:** Difference in means (low income mean minus high income mean)
- **Significance Level:** 0.05

**Results:** Low-income states averaged 4,074 minutes versus 3,677 minutes for high-income states, a difference of about 397 minutes (~6.6 hours). However, the permutation test yielded a **p-value of 0.152**, which is above our significance level of 0.05.

**Conclusion:** We fail to reject the null hypothesis. While low-income states do show longer average severe weather outages, the difference is not statistically significant at the 0.05 level. The observed gap could plausibly be due to random variation rather than a systematic economic effect.

## Framing a Prediction Problem

**Prediction Problem:** Predict the duration of a major power outage (OUTAGE.DURATION, in minutes).

**Type:** Regression

**Response Variable:** OUTAGE.DURATION. Duration is the primary measure of outage severity in this analysis and directly connects to the question of whether economic disadvantage affects outage outcomes.

**Evaluation Metric:** RMSE (Root Mean Squared Error). Chosen because it is in the same units as the target variable (minutes), making it directly interpretable. RMSE also penalizes large errors more heavily, which is important since some outages last orders of magnitude longer than others.

**Justification of features:** At the time of prediction (when an outage begins), we would know the location (state, region), the cause, the time of year, climate conditions, and economic characteristics of the state. We would not yet know the restoration time, total customers affected, or demand loss.

## Baseline Model

The baseline model is a **Linear Regression** implemented in a single sklearn Pipeline with two features:

- **CAUSE.CATEGORY** (nominal) — one-hot encoded using OneHotEncoder
- **ANOMALY.LEVEL** (quantitative) — passed through as-is

**Performance:**
- Train RMSE: 4,916 minutes
- Test RMSE: 7,182 minutes

This model is not very good, as it is off by an average of about 5 days. This is expected since outage duration depends on many more factors than just the cause and temperature anomaly. The large gap between train and test RMSE also suggests the model struggles to generalize.

## Final Model

The final model is a **Random Forest Regressor** with the following features added on top of the baseline:

- **CLIMATE.REGION** (nominal, one-hot encoded) — different regions face different weather patterns and have different infrastructure, affecting recovery time
- **MONTH** (quantitative, passed through) — seasonal patterns affect outage duration; winter storms vs summer heat waves require different repair approaches
- **PC.REALGSP.STATE** (quantitative, standardized) — state economic resources affect infrastructure quality and repair speed; scaling prevents high-GDP states from dominating
- **POPULATION** (quantitative, standardized) — larger populations may have more repair resources but also more complex grids
- **POPPCT_URBAN** (quantitative, standardized) — urban areas may see faster restoration due to denser infrastructure and repair crews

I tuned max_depth and n_estimators using GridSearchCV with 5-fold cross-validation. The best parameters were max_depth=5 and n_estimators=200.

**Performance:**
- Train RMSE: 3,506 minutes
- Test RMSE: 7,169 minutes

The final model improved over the baseline on both train and test RMSE, though the test improvement is modest. The constrained max_depth=5 helps prevent overfitting compared to an unconstrained tree.

## Fairness Analysis

**Question:** Does the model perform worse for low-income states than high-income states?

- **Group X:** Low-income states
- **Group Y:** High-income states
- **Evaluation Metric:** RMSE
- **Null Hypothesis:** The model is fair. Its RMSE for low-income and high-income states are roughly the same, and any differences are due to random chance.
- **Alternative Hypothesis:** The model is unfair. Its RMSE for low-income states is higher than for high-income states.
- **Test Statistic:** Difference in RMSE (low-income RMSE minus high-income RMSE)
- **Significance Level:** 0.05

**Results:** The model's RMSE was 3,944 minutes for low-income states and 8,881 minutes for high-income states. The permutation test yielded a **p-value of 0.747**.

**Conclusion:** We fail to reject the null hypothesis. The model actually performs better (lower RMSE) for low-income states than high-income states, and the p-value of 0.747 indicates there is no statistically significant difference in model performance between groups. The model appears to be fair across income groups.
