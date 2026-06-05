<link rel="stylesheet" href="assets/custom.css">


# What Makes Power Outages Last Longer?

**By Beckham Lee and Will Shin**

## Introduction

This project analyzes major power outage events in the continental United States from January 2000 to July 2016. Our central question is:

> **How does the reported cause of a major power outage relate to outage duration, and can we predict outage duration using information available early in an outage?**

Understanding outage duration matters for energy companies, grid operators, and emergency responders — a reliable early estimate of how long an outage will last shapes crew deployment, customer communication, and resource allocation. The dataset contains **1,534 outage records** with 56 columns. The columns most relevant to our analysis are:

| Column | Description |
|---|---|
| `OUTAGE.DURATION` | Duration of the outage in minutes (our target variable) |
| `CAUSE.CATEGORY` | High-level reported cause (e.g. severe weather, intentional attack) |
| `CLIMATE.REGION` | U.S. climate region where the outage occurred |
| `CLIMATE.CATEGORY` | Climate classification (warm, cold, normal) at time of outage |
| `NERC.REGION` | North American Electric Reliability Corporation region |
| `ANOMALY.LEVEL` | Oceanic El Niño/La Niña index at the time of the outage |
| `YEAR` / `MONTH` | Time context of the outage |
| `U.S._STATE` | State where the outage occurred |
| `OUTAGE.START` | Combined timestamp for when the outage began |
| `START.HOUR` / `START.DAYOFWEEK` / `IS.WEEKEND` | Time-based helper features engineered from `OUTAGE.START` |
| `CUSTOMERS.AFFECTED` | Number of customers affected, used only for descriptive context and excluded from prediction because it may not be known at the start of an outage |
| `TOTAL.CUSTOMERS` | Total electricity customers in the state |
| `POPULATION` | State population |

---

## Data Cleaning and Exploratory Data Analysis

### Data Cleaning

We performed the following cleaning steps on the raw Excel file:

1. **Dropped the formatting column.** The spreadsheet included an empty `variables` column used for layout. It was dropped immediately.

2. **Coerced numeric columns.** Several quantitative columns were stored as mixed-type objects due to Excel formatting. We explicitly converted them to numeric with `pd.to_numeric(..., errors='coerce')`, which also correctly introduces `NaN` for any remaining non-numeric entries.

3. **Combined date and time into datetime objects.** The dataset stored outage start and restoration information across four separate string columns. We combined each date+time pair into a single `pd.Timestamp` column: `OUTAGE.START` and `OUTAGE.RESTORATION`. This reflects the data generating process — a single outage event has one start moment and one end moment — and enables time-based feature engineering.

4. **Kept the provided `OUTAGE.DURATION`.** We did *not* recompute duration from the combined timestamps. We kept the provided `OUTAGE.DURATION` column rather than recomputing it from timestamps because it is the dataset’s official duration variable and is the response variable for our analysis. The combined timestamp columns were mainly used for time-based feature engineering.

5. **Created time-based helper columns** for modeling and EDA: `START.HOUR`, `START.DAYOFWEEK`, and `IS.WEEKEND`.

6. **Prepared modeling transformations inside the pipeline.** Later in the final model pipeline, we engineer `SEASON`, `LOG_POPULATION`, `LOG_TOTAL.CUSTOMERS`, and `CUSTOMERS.PER.PERSON`. Keeping these transformations inside the `sklearn` Pipeline makes the modeling process cleaner and avoids manually modifying the modeling data outside the pipeline.

The head of the cleaned DataFrame (key columns):

| YEAR | MONTH | U.S._STATE | NERC.REGION | CLIMATE.REGION | CLIMATE.CATEGORY | ANOMALY.LEVEL | OUTAGE.START | OUTAGE.RESTORATION | OUTAGE.DURATION | CAUSE.CATEGORY | CUSTOMERS.AFFECTED | TOTAL.CUSTOMERS | POPULATION |
|---:|---:|:---|:---|:---|:---|---:|:---|:---|---:|:---|---:|---:|---:|
| 2011 | 7 | Minnesota | MRO | East North Central | normal | -0.3 | 2011-07-01 17:00:00 | 2011-07-03 20:00:00 | 3060 | severe weather | 70000 | 2595696 | 5348119 |
| 2014 | 5 | Minnesota | MRO | East North Central | normal | -0.1 | 2014-05-11 18:38:00 | 2014-05-11 18:39:00 | 1 | intentional attack | nan | 2640737 | 5457125 |
| 2010 | 10 | Minnesota | MRO | East North Central | cold | -1.5 | 2010-10-26 20:00:00 | 2010-10-28 22:00:00 | 3000 | severe weather | 70000 | 2586905 | 5310903 |
| 2012 | 6 | Minnesota | MRO | East North Central | normal | -0.1 | 2012-06-19 04:30:00 | 2012-06-20 23:00:00 | 2550 | severe weather | 68200 | 2606813 | 5380443 |
| 2015 | 7 | Minnesota | MRO | East North Central | warm | 1.2 | 2015-07-18 02:00:00 | 2015-07-19 07:00:00 | 1740 | severe weather | 250000 | 2673531 | 5489594 |

### Univariate Analysis

Outage duration is strongly right-skewed. Across all non-missing outages, the median duration is 701 minutes, while the mean is about 2,625 minutes. This gap shows that a small number of extremely long outages pull the mean upward. We plot duration on a log₁₀ scale to make both short and very long outages visible.

<iframe
  src="assets/duration_distribution.html"
  width="800"
  height="450"
  frameborder="0"
></iframe>

Severe weather and intentional attacks together account for over 75% of all major outages in the dataset. Fuel supply emergencies are rare but produce by far the longest outages when they occur.

<iframe
  src="assets/cause_counts.html"
  width="800"
  height="450"
  frameborder="0"
></iframe>

### Bivariate Analysis

Outage duration varies dramatically by cause. Fuel supply emergencies have the highest median duration (3,960 min), followed by severe weather (2,460 min). Intentional attacks have the lowest median duration (56 min), suggesting they are typically localized and quickly contained. This strong association between cause and duration motivates including `CAUSE.CATEGORY` as a feature in our model.

<iframe
  src="assets/duration_by_cause.html"
  width="800"
  height="500"
  frameborder="0"
></iframe>

Mean outage duration has declined overall since the mid-2000s, though with substantial year-to-year variability. This may reflect improved grid resilience, faster utility response protocols, or shifts in which types of outages are reported over time.

<iframe
  src="assets/duration_over_time.html"
  width="800"
  height="450"
  frameborder="0"
></iframe>

### Interesting Aggregates

The table below shows the **median outage duration (in minutes)** by cause category, along with counts and range. Fuel supply emergencies stand out with a median of 3,960 minutes — roughly 2.75 days — despite representing fewer than 3% of outages. Intentional attacks are the most common cause after severe weather, but have a median duration of only 56 minutes.

| CAUSE.CATEGORY | Count | Mean | Median | Min | Max |
|:---|---:|---:|---:|---:|---:|
| fuel supply emergency | 38 | 13,484.0 | 3,960.0 | 1 | 108,653 |
| severe weather | 744 | 3,884.0 | 2,460.0 | 0 | 49,320 |
| public appeal | 69 | 1,468.4 | 455.0 | 30 | 11,867 |
| equipment failure | 55 | 1,816.9 | 221.0 | 0 | 78,377 |
| system operability disruption | 123 | 728.9 | 215.0 | 0 | 23,187 |
| islanding | 44 | 200.5 | 77.5 | 1 | 1,254 |
| intentional attack | 403 | 430.0 | 56.0 | 0 | 21,360 |

---

## Assessment of Missingness

### NMAR Analysis

We selected `OUTAGE.DURATION` for the missingness analysis because it has non-trivial missingness and is central to the project question. The missing values in `OUTAGE.DURATION` are tied to missing restoration date and time values, so the missingness appears to be inherited from whether the restoration event was recorded.

Following the lecture definition, a column is **NMAR** if the probability that a value is missing depends on the actual missing value itself. For `OUTAGE.DURATION`, NMAR is plausible: extremely long, unresolved, or unusually short outages may be less likely to have a clean restoration entry. In that case, the probability that duration is missing would depend on the unobserved true duration.

We believe `OUTAGE.DURATION` is the strongest plausible NMAR candidate in this dataset, but the observed evidence is more directly consistent with MAR through `CAUSE.CATEGORY`. Because NMAR depends on the unobserved true duration values, we cannot confirm or rule out NMAR using the observed data alone. The observed columns suggest another possible explanation: missingness may be related to reporting completeness and outage type, especially whether restoration information was recorded for certain cause categories. Additional data about whether the outage was still active when the dataset was compiled, the reporting source, utility reporting practices, and submission deadlines would help explain the missingness and could make the missingness more clearly MAR rather than NMAR.


### Missingness Dependency

`OUTAGE.DURATION` has **58 missing values** (~3.8% of rows). We tested whether this missingness depends on two other columns: `CAUSE.CATEGORY` (categorical) and `YEAR` (numeric).

- **Test statistic for `CAUSE.CATEGORY`:** Total Variation Distance (TVD) between the cause category distribution when duration is missing vs. not missing.
- **Test statistic for `YEAR`:** Absolute difference in group means of year when duration is missing vs. not missing.
- **Significance level:** 0.05 with 5,000 permutation repetitions.

| Column | Statistic | Observed Value | p-value | Conclusion |
|---|---|---|---|---|
| `CAUSE.CATEGORY` | TVD | 0.252 | 0.001 | Missingness **depends** on cause category |
| `YEAR` | Abs. diff in means | 0.249 | 0.627 | Missingness does **not** depend on year |

The plot below shows the empirical distribution of the TVD under the null hypothesis for the `CAUSE.CATEGORY` test. The observed TVD of 0.252 falls far into the right tail — fewer than 0.1% of shuffled statistics were as large — confirming that the result is not consistent with random chance.

<iframe
  src="assets/missingness_permutation.html"
  width="800"
  height="450"
  frameborder="0"
></iframe>

The overlay below shows the distribution of `YEAR` when duration is missing vs. not missing. The missing records lean toward the earliest and latest years of the collection window, but because those clusters sit on both sides of the average year, the mean year is almost the same for both groups (a gap of about 0.25 years). Our test statistic is the absolute difference in mean year, so that small gap is exactly why the result is non-significant (p = 0.63).

<iframe
  src="assets/missingness_year.html"
  width="800"
  height="450"
  frameborder="0"
></iframe>

**Interpretation:** At the 0.05 significance level, the permutation test for `CAUSE.CATEGORY` gives p = 0.001, so we reject the null hypothesis. This means the missingness of `OUTAGE.DURATION` depends on the observed `CAUSE.CATEGORY` column. Following the lecture flowchart, this is evidence that `OUTAGE.DURATION` is MAR dependent on `CAUSE.CATEGORY`, assuming the main missingness mechanism does not depend directly on the unobserved duration itself.

For `YEAR`, the p-value of 0.627 is above 0.05, so we fail to reject the null hypothesis. This means we do not find evidence that duration missingness depends on year. However, this does not make `OUTAGE.DURATION` MCAR, because MCAR would require missingness to be independent of every other column, and we already found dependence on `CAUSE.CATEGORY`.

Overall, the observed permutation tests show that `OUTAGE.DURATION` is not fully MCAR and is consistent with MAR through `CAUSE.CATEGORY`. NMAR remains plausible because the missingness could still depend on the unobserved true duration, but that cannot be confirmed or ruled out using observed data alone.

---

## Hypothesis Testing

**Question:** Do severe weather outages last significantly longer on average than intentional attack outages?

From the bivariate analysis, severe weather outages appeared to have much higher median and mean durations. We use a permutation test to assess whether this difference is statistically significant.

- **Null Hypothesis:** The average outage duration for severe weather outages is the same as for intentional attack outages. Any observed difference is due to random chance.
- **Alternative Hypothesis:** Severe weather outages last longer on average than intentional attack outages.
- **Test Statistic:** Difference in group means (mean duration of severe weather minus mean duration of intentional attack). We use a difference in means rather than TVD because we are comparing a numeric outcome across two groups and have a directional alternative hypothesis.
- **Significance Level:** 0.05
- **Method:** One-sided permutation test with 5,000 repetitions.

**Result:** The observed difference in means was **3,454 minutes** (severe weather mean: 3,884 min; intentional attack mean: 430 min). Under 5,000 permutations, none of the shuffled statistics reached the observed value, giving **p < 0.001**.

<iframe
  src="assets/hypothesis_permutation.html"
  width="800"
  height="450"
  frameborder="0"
></iframe>

**Conclusion:** We reject the null hypothesis. The data provide strong evidence that severe weather outages last longer on average than intentional attack outages in this dataset. This is consistent with the physical explanation: severe weather causes widespread infrastructure damage requiring crew deployment and physical repair, while intentional attacks tend to be localized and more quickly reversible. We note this is observational data — we cannot establish causation, and confounders such as geographic region and grid age may also play a role.

---

## Framing a Prediction Problem

**Prediction Problem:** Predict the duration of a major power outage (`OUTAGE.DURATION`, in minutes).

**Type:** Regression — the target is a continuous numeric variable.

**Why this response variable:** Duration is the most direct and actionable measure of outage severity. An energy company responding to a new outage would want to estimate how long it will last in order to allocate repair crews, issue customer communications, and coordinate with emergency services.

**Evaluation metric:** We use **MAE (Mean Absolute Error)** as the primary metric. MAE is measured in the same units as the target (minutes), making it directly interpretable. It is also more robust to the extreme outlier outages in this dataset than RMSE, which squares errors and disproportionately amplifies large residuals. We additionally report RMSE and R² for context.

**Features available at time of prediction:** When an outage is first reported, a utility would know the cause category, state, NERC region, climate region, climate category, anomaly level, time of occurrence, and population/customer-base context. They would *not* yet know the restoration time, customers affected, or demand loss — those are outcomes measured after resolution. Our model uses only features available at the start of an outage.

---

## Baseline Model

The baseline is a **linear regression** model trained inside a single `sklearn` Pipeline. It uses five original features:

- **Quantitative (2):** `YEAR`, `MONTH`
- **Nominal (3):** `CAUSE.CATEGORY`, `CLIMATE.CATEGORY`, `NERC.REGION`
- **Ordinal:** none

Categorical columns are imputed with the most frequent value and one-hot encoded (with `drop='first'` to avoid multicollinearity). Numeric columns are imputed with their median. All steps are contained in a `ColumnTransformer` inside the `Pipeline`.

**Performance on held-out test set (25% split, random_state=42):**

| Model | MAE (min) | RMSE (min) | R² |
|---|---|---|---|
| Baseline linear regression | 2,604.7 | 6,911.0 | 0.173 |

The baseline explains about 17% of variance in outage duration. This is a difficult regression problem — duration varies enormously even within the same cause category — so a simple linear model with five original features is not expected to perform strongly. However, it establishes a clear starting point and confirms that `CAUSE.CATEGORY` does carry predictive signal.

---

## Final Model

The final model uses a **Random Forest Regressor**, chosen because outage duration depends on nonlinear interactions between cause, climate, location, and time that a linear model cannot capture. Random forests handle these interactions naturally, are robust to outliers, and do not require the target to be normally distributed.

**Engineered features (created inside the pipeline):**

| Feature | Type | Justification |
|---|---|---|
| `START.HOUR` | Quantitative | Outages starting at night may take longer to mobilize repair crews |
| `START.DAYOFWEEK` | Quantitative | Weekend vs. weekday affects crew availability |
| `IS.WEEKEND` | Binary | Explicit flag for weekend, when utility staffing is reduced |
| `SEASON` | Nominal (OHE) | Seasonal demand and weather patterns affect grid stress |
| `LOG_POPULATION` | Quantitative | Compresses the scale of a highly skewed population column; for a tree model, this is not expected to be the main source of improvement |
| `LOG_TOTAL.CUSTOMERS` | Quantitative | Compresses the scale of a highly skewed customer-base column; the more meaningful scale feature is the ratio `CUSTOMERS.PER.PERSON` |
| `CUSTOMERS.PER.PERSON` | Quantitative | Captures customer density relative to population, a relationship the forest cannot directly construct from the two original columns |


**Additional original features** used by the final model but not the baseline: `CLIMATE.REGION`, `U.S._STATE`, `ANOMALY.LEVEL`, the residential/commercial/industrial customer-percentage columns (`RES.CUST.PCT`, `COM.CUST.PCT`, `IND.CUST.PCT`), and the urban/area/water context columns (`POPPCT_URBAN`, `AREAPCT_URBAN`, `PCT_WATER_TOT`). These describe the location, climate, customer base, and geography of each outage. They are background characteristics known early in an outage, not outcomes that require the final restoration time, so they do not leak the target.


We used `GridSearchCV` with 3-fold cross-validation, scored using negative MAE, to select the random forest hyperparameters. We tuned these values because they control the bias-variance tradeoff of the forest: `n_estimators` controls how many trees are averaged, while `max_depth` and `min_samples_leaf` control how complex each tree can become. Deeper trees can capture more complex patterns, but they can also overfit; larger leaf sizes reduce overfitting by preventing very small leaves.

| Hyperparameter | Values Searched | Best Value |
|---|---:|---:|
| `n_estimators` | 50, 100 | 100 |
| `max_depth` | 4, 8 | 8 |
| `min_samples_leaf` | 1, 5 | 5 |

**Performance on the same held-out test set as the baseline:**

| Model | MAE (min) | RMSE (min) | R² |
|---|---|---|---|
| Baseline linear regression | 2,604.7 | 6,911.0 | 0.173 |
| Final random forest | 2,312.5 | 6,539.6 | 0.260 |

The final model reduces MAE by **292 minutes** and improves R² from 0.173 to 0.260. The improvement comes primarily from the random forest's ability to model nonlinear cause-region-season interactions, and from the additional state-level and time-of-day features that give the model more granular context about when and where each outage occurred.

<iframe
  src="assets/actual_vs_predicted.html"
  width="1000"
  height="500"
  frameborder="0"
></iframe>

---

## Fairness Analysis

**Question:** Does the final model perform worse for severe weather outages than for outages caused by other categories?

- **Group X:** Severe weather outages (n = 181 in test set)
- **Group Y:** All other cause categories (n = 188 in test set)
- **Evaluation metric:** RMSE, an appropriate metric for a regression model that weights the large errors that matter most for long outages
- **Null Hypothesis:** The model is fair. Its RMSE for severe weather and other outages is roughly equal; any observed difference is due to random chance.
- **Alternative Hypothesis:** The model performs worse (higher RMSE) for severe weather outages than for other causes.
- **Test Statistic:** RMSE(severe weather) minus RMSE(other causes)
- **Significance Level:** 0.05, one-sided permutation test with 5,000 repetitions.

**Observed group RMSEs:**

| Group | Count | RMSE (min) |
|---|---|---|
| Severe weather | 181 | 5,780.2 |
| Other causes | 188 | 7,195.4 |

**Observed difference:** -1,415.2 minutes. **p-value approximately 0.49.**

<iframe
  src="assets/fairness_permutation.html"
  width="1000"
  height="450"
  frameborder="0"
></iframe>

**Conclusion:** We fail to reject the null hypothesis. Under the RMSE parity test, we do not find evidence that the model performs worse for severe weather outages than for other outage causes. The observed RMSE is actually lower for severe weather outages in this test split: about 5,780 minutes compared to about 7,195 minutes for other causes. This does not prove the model is equally good for every type of outage; it only means that, using RMSE as the fairness metric, the observed difference does not support the claim that severe weather outages have worse prediction performance. Future work could fit a separate model for extremely long-duration outages or add weather-severity and infrastructure-age features.
