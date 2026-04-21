# A/B Testing and Causal Inference

End-to-end analysis of an email marketing experiment, a three-arm RCT with 64,000 customers from Hillstrom. Covers experiment validation, hypothesis testing, power analysis, segmentation, and variance reduction via CUPED across three outcome metrics: site visits, conversions, and revenue.

**Business question:** Does sending a promotional email increase site visits, purchases, and revenue? If so, which campaign performs better?

---

## Key Findings

| Comparison | Visit Rate | Conversion Rate | Spend |
|---|---|---|---|
| Men's Email vs Control | +7.7pp*** | +0.68pp*** | +$0.77*** |
| Women's Email vs Control | +4.5pp*** | +0.31pp*** | +$0.42*** |
| Men's vs Women's Email | +3.1pp*** | +0.37pp*** | +$0.35* |

*p < 0.05, ***p < 0.001 after Benjamini-Hochberg correction across 9 tests.
pp = percentage points

**Per 10,000 customers, the Men's email is expected to generate:**
- 766 additional visits (95% CI: 700–832)
- 68 additional conversions (95% CI: 50–86)
- $7,698 additional spend (95% CI: $4,851–$10,545)

**Segmentation:** The campaign effect was consistent across all segments tested: No statistically significant heterogeneous treatment effects were found after multiple comparisons correction.

**CUPED:** Variance reduction was under 1% for all outcomes, consistent with the weak correlations between the available covariates (`history`, `recency`) and outcomes.

---

## Dataset

64,000 customers who last purchased within twelve months were randomly assigned to one of three arms:
- 1/3 received an email campaign featuring Men's merchandise
- 1/3 received an email campaign featuring Women's merchandise
- 1/3 received no email

Outcomes were tracked over two weeks following the email.

### Features

- **Recency:** Months since last purchase
- **History_Segment:** Categorization of dollars spent in the past year
- **History:** Actual dollar value spent in the past year
- **Mens:** 1/0 indicator, customer purchased Men's merchandise in the past year
- **Womens:** 1/0 indicator, customer purchased Women's merchandise in the past year
- **Zip_Code:** Urban, Suburban, or Rural
- **Newbie:** 1/0 indicator, new customer in the past twelve months
- **Channel:** Purchase channel in the past year (Web, Phone, Multichannel)
- **Segment:** Email campaign received (Mens E-Mail, Womens E-Mail, No E-Mail)
- **Visit:** 1/0 indicator, customer visited website in the following two weeks
- **Conversion:** 1/0 indicator, customer purchased in the following two weeks
- **Spend:** Dollars spent in the following two weeks

---

## Tech Stack

Python · DuckDB · pandas · NumPy · SciPy · statsmodels · matplotlib

---

## Setup

**Environment**

```bash
conda env create -f environment.yml
conda activate ab-testing-causal-inference
```

**Data**

The data downloads automatically when running Notebook 01. Alternatively, download manually from the [MineThatData challenge page](https://blog.minethatdata.com/2008/03/minethatdata-e-mail-analytics-and-data.html) and place the CSV at `data/raw/hillstrom.csv`.

---

## Notebooks

Run in numbered order. Each notebook loads the raw data independently.

| Notebook | Description |
|---|---|
| `01_dataset_validation.ipynb` | Loads data into DuckDB, verifies arm sizes via Sample Ratio Mismatch test, checks covariate balance, and characterizes the spend distribution. |
| `02_hypothesis_testing.ipynb` | Tests visit, conversion, and spend outcomes across all three arms using z-tests for proportions and Welch's t-test. Applies Benjamini-Hochberg correction and frames results as a business recommendation. |
| `03_power_analysis.ipynb` | Computes minimum detectable effects at the observed sample size, plots power and sample size curves, and discusses the effect of winsorization on power for the spend outcome. |
| `04_segmentation.ipynb` | Estimates treatment effects within segments defined by recency, channel, prior spend, and customer tenure. Tests for interaction effects via OLS regression with BH correction. |
| `05_cuped.ipynb` | Implements CUPED from scratch using single and multiple covariates, quantifies variance reduction, and explains why the available covariates produced minimal improvement. |
| `06_causal_inference.ipynb` | **(in progress)** Uses the experimental ground truth to validate propensity score methods on an artificially biased observational dataset.