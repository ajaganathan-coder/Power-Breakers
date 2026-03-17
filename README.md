# Power Breakers - DSC259R Final Project

**Names**:

1. Mitchell Farrington mfarring@ucsd.edu - A15378625
2. Anuradha Jaganathan ajaganathan@ucsd.edu - A69038119
3. Kyle Packer kpacker@ucsd.edu - A69036932
4. Alex Twoy atwoy@ucsd.edu - A17580384


**Website Link**: [https://ajaganathan-coder.github.io/Power-Breakers/](https://ajaganathan-coder.github.io/Power-Breakers/)  
---

## Introduction

Prolonged power outages during summer and winter months pose significant risks, particularly when they exceed 24 hours(>1440 mins). These disruptions can trigger serious health crises for populations in affected regions. Leveraging historical outage data to predict the occurrence of major events can enable cities to prepare proactively and minimize potential impacts. We discussed on the below questions 

  1. Which areas suffered the most power outages and why?
  2. What was the average length of power outages?
  3. Does urban vs. rural area type affect outage frequency?                                                            
  4. Can we predict whether a major severe-weather outage will occur in a NERC region during a given week?

and finally decided on our research ### Research Question
> **For each NERC region r and week w, estimate the probability that at least one major outage triggered by severe weather will occur in region r during week w.?**

Answering this question has direct operational value and help cities and repair crews to prepare ahead of time.

### Dataset
The **Purdue Major Power Outage Risk dataset** [https://engineering.purdue.edu/LASCI/research-data/outages](https://engineering.purdue.edu/LASCI/research-data/outages), which records 1,534 documented outage events across the contiguous United States from January 2000 to July 2016 was chosen for the project. Each event describes one outage event with 55 attributes spanning:
- **Temporal context** — year, month, start/restore timestamps
- **Geographic identifiers** — state, NERC region, climate region
- **Meteorological context** — climate category (Normal / El Niño / La Niña),   anomaly level
- **Cause information** — broad cause category and detailed descriptor
- **Socioeconomic covariates** — population, urban fraction, GSP per capita
- **Outage outcome** — duration (minutes), customers affected, MW demand lost

The columns relevant to our analysis are:

- **OUTAGE_DURATION_MINS** 
- **US_STATE**  
- **NERC_REGION** 
- **CAUSE_CATEGORY** 
- **POPPCT_URBAN** 
- **POPPCT_RURAL** 

## Data Cleaning and Exploratory Data Analysis

To prepare the data for exploratory and further analysis, the below data cleansing steps were performed:

### Drop irrelevant columns
The raw workbook includes price/sales columns (retail electricity tariffs and consumption by sector) and GSP/utility columns that describe the state's economic profile *in aggregate* rather than the outage event. These are redundant given we already have `PC_REALGSP_*` aggregates and would introduce multicollinearity. We also drop the Purdue internal `variables` descriptor column.

### Rename columns
Dots in column names make attribute access fragile. We replace every `.` with `_`.

### Merge date + time → Timestamps
Outage start and restoration are stored as separate date and clock-time columns. We concatenate them into proper `pd.Timestamp` objects so we can compute arithmetic like `OUTAGE_DURATION_MINS = (restore - start).total_seconds() / 60`.

### Cast integer columns
Several columns arrive as float because of NaNs. After cleaning we downcast where the column only ever holds whole numbers.

### Drop rows with missing `OUTAGE_DURATION_MINS`
Rows where the duration is unknown cannot be labelled and are excluded. This affects 58 of 1 534 rows (3.8 %).

### Exploratory Analysis

We then addressed the below questions to understand the data and filter to the required content.

*Key questions:*
- Is outage duration right-skewed? (Almost certainly yes — most outages are short, but a tail of catastrophic events pulls the mean up.)
  
- **Univariate Analysis**

Distribution of outage durations: full range and filtered to the interquartile range (≤ 2,880 mins = 75th percentile). The IQR-filtered view reveals the distribution structure that extreme outliers obscure.
<iframe src="iframe_figures/figure_10.html" width=800 height=600 frameBorder=0></iframe>

- **Bivariate Analysis**

- Which states / NERC regions suffer the most events or the longest outages?
- 
<iframe src="iframe_figures/figure_11.html" width=800 height=600 frameBorder=0></iframe>
<iframe src="iframe_figures/figure_14.html" width=800 height=600 frameBorder=0></iframe>

- Is there a seasonal signal in duration?

<iframe src="iframe_figures/figure_15.html" width=800 height=600 frameBorder=0></iframe>
<iframe src="iframe_figures/figure_16.html" width=800 height=600 frameBorder=0></iframe>

- How correlated are the numeric features?
<iframe src="iframe_figures/figure_corr.png" width=800 height=600 frameBorder=0></iframe>

## Assessment of Missingness

Missing data in a dataset can be **Missing Completely At Random (MCAR)**, **Missing At Random (MAR)**, or **Missing Not At Random (MNAR)**. 

Columns containing nulls are: CLIMATE_REGION CAUSE_CATEGORY_DETAIL HURRICANE_NAMES DEMAND_LOSS_MW CUSTOMERS_AFFECTED RES_PERCENT, COM_PERCENT, IND_PERCENT POPDEN_UC, POPDEN_RURAL. Before analysis, we know HURRICANE_NAMES is Missing by Design, since there can only be a hurricane name if there is a hurricane. Now, we have to determine whether the remaining columns are missing at random or not, and then appropriately deal with them.

We applied two statistical tests for each column with missing values:
1. **Chi-squared test** (categorical columns) — tests whether the missingness indicator is independent of the state (`U_S__STATE`).
2. **Welch t-test** (numeric columns) — tests whether the mean of a reference numeric feature (`YEAR`) differs between rows where the column is missing vs. not missing.

Based on the results, all of the columns with missing values are correlated with some other columns in the dataset. Therefore, we completely rule out NMAR.

But getting rid of these values would introduce bias into our data. The next step therefore is imputation. We are doing a group-wise imputation of median/most common category, grouped by US state. We are doing it group-wise, since there are many variable values which would be median nationwide but not make any sense at all in certain states, such as a snow storm in Florida or a hurricane in Alaska. Based on the analysis, we determined that almost each column/state combination has other values for that state (except for Hurricane names, which we expected, so this supports our group-wise imputation.


## Hypothesis Testing

### Question
> Is the proportion of severe-weather outages the same across all NERC regions, or do some regions face disproportionately more severe-weather disruptions?

### Why this matters
NERC regions are the primary planning entities for North American grid reliability. If severe-weather incidence is non-uniform across regions, region-level infrastructure investment and resilience planning must reflect those differences. It also tells us whether `NERC_REGION` is a useful predictor in downstream models.

### Hypotheses
- **H₀**: Severe-weather outage incidence is independent of NERC region   (no association).
- **H₁**: At least one NERC region has a severe-weather rate that differs from   the overall rate.

### Test
We used a **Pearson chi-squared test of independence** on the contingency table of *is_severe* × *NERC_REGION*. The significance level is **α = 0.05**.

A chi-squared test is appropriate because:
- Both variables are categorical
- All expected cell counts exceed 5
- Observations are independent (each row is a distinct outage event)


### Test Interpretation

The chi-squared statistic is large and the p-value is far below 0.05, so we **reject H₀** and conclude that severe-weather outages are distributed non-uniformly across NERC regions.

Reading the bubble chart:
- **RFC** (ReliabilityFirst Corporation — Ohio/Pennsylvania corridor) and   **SERC** (Southeast) show large positive residuals, meaning they experience   *more* severe-weather outages than expected under independence.
- **WECC** (Western Interconnection) shows a negative residual, consistent with   the drier, less storm-prone climate of the interior West.

These regional patterns motivate including `NERC_REGION` as a predictor and performing a fairness analysis to check whether model performance varies systematically across regions.

## Framing a Prediction Problem

*Problem:*

For each NERC region r and week w, predict whether at least one major outage caused by severe weather will start in region r during that week w.

*Type:* 
Binary classification; the model outputs a probability p(y=1|r,w).

We will define a week as disjoint, Monday-anchored weeks [w_start, w_end) = [Mon 00:00, next Mon 00:00).

*Response Variable*: y(r,w)=1 if any outage with is_severe=1 denoting severe weather and meeting the “major” criterion starts in region r with OUTAGE_START_TIMESTAMP ∈ [w_start, w_end); else 0. Major = customers_affected ≥ 50,000 OR outage_duration_mins ≥ 60 OR demand_loss_MW ≥ 100.

*Primary metric*: PR-AUC (precision–recall AUC), due to class imbalance and our focus on the positive class.

Weekly windows raise the base rate versus daily, better matching medium-term planning (crew rotations, equipment prep, and general readiness) while staying within the 5–7 day weather forecast horizons; this improves learnability without sacrificing operational relevance.


## Baseline Model

### Logistic Regression

A **logistic regression** with L2 regularisation (C = 1) is a sensible baseline because:
1. It is interpretable — we can directly inspect coefficient magnitudes.
2. It scales well and converges reliably.
3. It provides calibrated probability estimates out of the box, making it easy    to draw a precision–recall curve.

The preprocessing pipeline applies:
- **StandardScaler** to all numeric features (logistic regression is sensitive   to feature scale).
- **OneHotEncoder** (drop = 'first' to avoid perfect multicollinearity) to all   categorical features.

We evaluated using **PR-AUC** (area under the precision–recall curve) rather than accuracy, because the data are moderately imbalanced (about 42 % major outages) and PR-AUC better captures performance on the positive class.

<iframe src="iframe_figures/baseline_model.html" width=800 height=600 frameBorder=0></iframe>


## Final Model


We evaluated four additional classifiers beyond the baseline:

| Model | Notes |
|---|---|
| Random Forest | Good out-of-the-box; can overfit on small datasets |
| XGBoost | Highly tunable; requires careful regularisation |
| Ridge Logistic Regression | Higher regularisation than baseline |
| **Gradient Boosting (sklearn)** | **Best PR-AUC on validation set** |


### GBC Hyperparameters
```
GradientBoostingClassifier(
    n_estimators  = 200,
    learning_rate = 0.05,
    max_depth     = 3,
    random_state  = 42
)
```
- `max_depth=3` keeps each tree shallow (high bias, low variance per tree),   relying on the ensemble to reduce bias.
- `learning_rate=0.05` with `n_estimators=200` follows the *shrinkage*   principle: many weak updates are more robust than few strong ones.

### What we improved over the baseline

1. Replaced linear decision boundary (logistic) with a non-linear ensemble.
2. Added `is_severe` as a derived feature (created from `CAUSE_CATEGORY` *before*    the target split — this is valid because the cause category is often coded at    dispatch time for natural hazard events).
3. Tuned regularisation.
4. Retained the same preprocessing pipeline for consistency.

<iframe src="iframe_figures/figure_final_model.html" width=800 height=600 frameBorder=0></iframe>

## Fairness Analysis

We evaluated whether the Gradient Boosting model treats urban and rural states equitably. States where ≤ 80% of the population lives in urban areas are classified as **rural** (the dataset mean is ~84%).

**Null Hypothesis (H₀):** The model's TPR (and FPR) is the same for urban and rural states.

**Alternative Hypothesis (H₁):** There is a statistically significant difference in TPR or FPR between groups.

**Test:** Permutation test (10,000 iterations each for TPR difference and FPR difference).

**Key Findings:**

- Both permutation tests return p-values ≈ 0, so we **reject H₀** for both metrics.
- The model correctly identifies major outages significantly less often for **urban** states (lower urban TPR).
  
- **Rural** areas are more frequently false-alarmed (higher rural FPR).
- 
- In a production deployment, this bias could lead to missed emergency responses in urban areas and unnecessary resource mobilization in rural areas.
<iframe src="iframe_figures/TPR_Difference_Urban_-_Rural.png" width=800 height=600 frameBorder=0></iframe>
<iframe src="iframe_figures/FPR_Difference_Urban_-_Rural.png" width=800 height=600 frameBorder=0></iframe>

