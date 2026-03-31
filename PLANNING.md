# PLANNING.md — Hillstrom A/B Test Analysis

## Project Goal

Learn and demonstrate core tech data science experimentation skills through end-to-end analysis of a real randomized controlled trial. The project targets the skills evaluated in tech DS interviews: SQL fluency, hypothesis testing, power analysis, segmentation, variance reduction, and causal inference.

## Dataset

**Hillstrom MineThatData Email Marketing Dataset (2008)**
- Source: https://blog.minethatdata.com/2008/03/minethatdata-e-mail-analytics-and-data.html
- Direct CSV: http://www.minethatdata.com/Kevin_Hillstrom_MineThatData_E-MailAnalytics_DataMiningChallenge_2008.03.20.csv
- 64,000 customers in a three-arm randomized experiment
- Arms: Men's email campaign, Women's email campaign, No email (control)
- Covariates: recency, history_segment, history (dollars), mens (prior purchase flag), womens (prior purchase flag), zip_code, newbie, channel
- Outcomes: visit (binary), conversion (binary), spend (continuous, dollars)
- Tracking window: two weeks post-email
- License: public, originally released as an open data mining challenge

## Reference Reading

**Primary text:** Kohavi, Tang, Xu — *Trustworthy Online Controlled Experiments: A Practical Guide to A/B Testing* (2020)

Section-aligned reading guide:
- Sections 1–2 (SQL, hypothesis testing): Chapters 1–3, 17 (metrics taxonomy, hypothesis testing fundamentals, OEC definition)
- Section 3 (power analysis): Chapter 15 (sample size, power, minimum detectable effect)
- Section 4 (segmentation): Chapter 20 (heterogeneous treatment effects), Chapter 19 (pitfalls of multiple testing)
- Section 5 (CUPED): Chapter 22 (variance reduction techniques)
- Section 6 (causal inference): Not covered in Kohavi — use Coursera causal inference course materials

## Timeline

Estimated 2–4 weeks. Sections 1–5 are the core — prioritize completing these. Section 6 is a strong addition but can be cut or abbreviated if time is short.

| Week | Focus | Deliverable |
|------|-------|-------------|
| 1 | Sections 1–2: Data loading, SQL workflow, hypothesis testing | Notebooks 01, 02 |
| 2 | Sections 3–4: Power analysis, segmentation | Notebooks 03, 04 |
| 3 | Section 5: CUPED | Notebook 05 |
| 3–4 | Section 6: Causal inference | Notebook 06 |
| 4 | README, cleanup, design decisions doc | README.md |

Adjust as needed — the important thing is that sections 1–5 are complete and clean before moving to section 6.

---

## Section 1: Data Loading and SQL Workflow

**Goal:** Load data into DuckDB, perform initial exploration and validation in SQL, demonstrate SQL fluency.

**Key concepts to understand:**
- Randomization balance checks: are covariates distributed similarly across treatment arms?
- Sample Ratio Mismatch (SRM): are the arm sizes what you'd expect from random assignment? A significant deviation suggests a data or randomization problem.
- Baseline metric rates: what are the visit, conversion, and spend rates in each arm before any formal testing?

**Analysis steps:**
1. Download CSV, load into DuckDB
2. SQL: row counts per arm — verify roughly 1/3 each, run a chi-squared test for SRM
3. SQL: compute baseline rates (visit rate, conversion rate, mean spend) per arm
4. SQL: check covariate balance — mean recency, channel distribution, newbie proportion, history by arm
5. SQL: explore the spend distribution — note the heavy zero-inflation and right skew
6. Pull summary results into Python for any plots (histograms of spend, covariate balance tables)

**Deliverable:** Notebook 01 — data loaded, validated, initial summary statistics, clear narrative about data quality.

**Notes:**
- Do SQL work in DuckDB via the `duckdb` Python package — you can run SQL directly in notebook cells
- This section should feel like "day one of looking at experiment results" — verify the data before analyzing it
- Keep SQL queries clean and commented — these are part of what a reviewer will evaluate

---

## Section 2: Hypothesis Testing and Experiment Evaluation

**Goal:** Test whether the email campaigns affected customer behavior. Frame results as business decisions.

**Key concepts to understand:**
- Two-sample t-test and z-test for proportions — when to use which
- Multiple comparisons: three arms × three outcomes = many tests. Understand Bonferroni correction and why it matters, even if the adjustment doesn't change your conclusions here
- Skewed outcomes: spend has many zeros and a long right tail. Understand winsorization (capping extreme values) and why you might use it
- Practical significance vs. statistical significance: a result can be statistically significant but too small to matter for the business
- Framing: the deliverable isn't "p = 0.03" — it's "we recommend sending the Men's email because it increased visits by X% with Y confidence, translating to an estimated $Z incremental revenue per 10,000 customers"

**Analysis steps:**
1. Test visit rate: Men's vs. Control, Women's vs. Control, Men's vs. Women's (z-test for proportions)
2. Test conversion rate: same three comparisons
3. Test spend: same three comparisons (t-test, then repeat with winsorized spend at 95th or 99th percentile)
4. Apply Bonferroni correction across the family of tests, discuss impact
5. Compute effect sizes (Cohen's d or raw percentage point differences) alongside p-values
6. Summarize in a decision framework: which campaign should we roll out, and what's the expected business impact?

**Deliverable:** Notebook 02 — complete hypothesis testing with business-framed conclusions.

**Notes:**
- Show both the raw p-values and corrected p-values
- Discuss what changes (if anything) when you winsorize spend — this is a common interview topic
- Don't just report numbers — interpret them as a recommendation to a product team

---

## Section 3: Power Analysis

**Goal:** Using the observed data, characterize what effects this experiment could and couldn't detect. Discuss experiment design tradeoffs.

**Key concepts to understand:**
- Statistical power: probability of detecting a real effect (typically target 80%)
- Minimum detectable effect (MDE): the smallest effect size you can reliably detect given your sample size and variance
- The relationship between sample size, MDE, significance level, and power — changing one changes the others
- Prospective vs. retrospective framing: don't compute "the power this experiment had" (that's a known pitfall). Instead: given the observed variance, what MDEs could we detect at various sample sizes?
- Business framing: "we can detect a 0.5 percentage point lift in conversion with 80% power" — is that granularity useful for the business, or do they only care about 2+ point lifts?

**Analysis steps:**
1. Extract observed baseline rates and variances for each outcome from the control arm
2. Compute MDE for each outcome at the actual sample size (n ≈ 21,333 per arm) at 80% power, alpha = 0.05
3. Plot power curves: power as a function of effect size at the actual sample size
4. Plot sample size curves: required sample size as a function of MDE
5. Discuss: for the spend outcome (high variance due to skew), show how winsorization or log-transformation changes the power calculation
6. Frame as experiment design advice: "if this business wanted to detect a 0.3% lift in conversion, they'd need N users, which at X users/day means Y weeks of runtime"

**Deliverable:** Notebook 03 — power analysis with clear visualizations and business-framed interpretation.

**Notes:**
- Use `statsmodels.stats.power` or implement calculations directly — both are fine
- The framing matters more than the computation here. Anyone can call a power function. The value is in connecting it to a decision: is this experiment well-sized for the effects we care about?

---

## Section 4: Segmentation and Heterogeneous Treatment Effects

**Goal:** Investigate whether the email campaigns worked differently for different customer segments.

**Key concepts to understand:**
- Subgroup analysis: cutting results by a covariate to see if effects vary
- Exploratory vs. confirmatory analysis: if you pre-specified "we expect the effect to be larger for returning customers," that's confirmatory. If you're slicing by every covariate to see what pops out, that's exploratory — and you need to be honest about it
- Multiple comparisons problem in segmentation: testing 5 covariates × 3 arms × 3 outcomes = many chances for false positives
- Simpson's paradox potential: an effect that's positive overall could be negative within certain subgroups
- Interaction effects: is the treatment effect *different* across segments, or is it just that baseline rates differ?

**Analysis steps:**
1. Define segments from available covariates:
   - Recency: recent vs. lapsed (pick a sensible cutpoint or use tertiles)
   - Channel: phone, web, multichannel
   - Customer value: high vs. low prior spend (history)
   - Newbie: yes vs. no
2. For each segment, compute treatment effects (visit rate, conversion rate, spend) vs. control
3. Test for interaction: is the treatment effect significantly different between segments? (e.g., test whether the Men's email effect differs for newbies vs. non-newbies)
4. Visualize: forest plot or grouped bar chart showing effect sizes by segment with confidence intervals
5. Discuss which findings are robust and which are likely noise. Be explicit about the exploratory nature.

**Deliverable:** Notebook 04 — segmentation analysis with honest discussion of multiple comparisons and what's trustworthy.

**Notes:**
- The intellectual honesty in this section is what separates a strong project from a weak one. Resist the temptation to present every significant subgroup as a real finding.
- A good interview answer to "we found it works for segment X" is: "that's interesting but exploratory — I'd want to run a follow-up experiment specifically powered for that segment before making a decision"

---

## Section 5: Variance Reduction (CUPED)

**Goal:** Implement CUPED (Controlled-experiment Using Pre-Experiment Data) to improve precision of treatment effect estimates.

**Key concepts to understand:**
- CUPED reduces the variance of the treatment effect estimator by adjusting for pre-experiment covariates that predict the outcome
- Mechanically: for each user, compute an adjusted outcome Y_adj = Y - theta * X, where X is a pre-experiment covariate and theta = Cov(Y, X) / Var(X). Then run the standard test on Y_adj instead of Y
- The variance reduction is proportional to the squared correlation between X and Y — the more predictive the covariate, the bigger the gain
- This is equivalent to regression adjustment but framed in a way that's easy to implement in an experimentation platform
- The practical impact: variance reduction means you can detect smaller effects or run experiments for less time. This directly translates to faster product iteration.
- Hillstrom's `history` (prior purchase dollars) and `recency` (months since last purchase) are natural pre-experiment covariates

**Analysis steps:**
1. Compute the correlation between each pre-experiment covariate and each outcome — this tells you which covariates are useful for CUPED
2. Implement CUPED adjustment for the most predictive covariate(s)
3. Re-run the hypothesis tests from Section 2 using the CUPED-adjusted outcomes
4. Compare: standard errors before and after CUPED — quantify the variance reduction as a percentage
5. Translate to practical terms: "CUPED reduced the standard error by X%, equivalent to needing Y% fewer users to detect the same effect"
6. Discuss: when does CUPED help a lot vs. a little? What makes a good CUPED covariate?

**Deliverable:** Notebook 05 — CUPED implementation with before/after comparison and practical interpretation.

**Notes:**
- Implement CUPED from scratch rather than using a library — this forces understanding and is what you'd need to explain in an interview
- This section is the strongest differentiator in the project. Most portfolio projects stop at hypothesis testing. CUPED signals that you understand how experimentation works at scale.
- After building this with LLM assistance, make sure you can re-derive and explain the adjustment from scratch — this will come up in interviews

---

## Section 6: Causal Inference from Observational Data

**Goal:** Demonstrate what happens when you can't randomize, and how causal inference methods attempt to recover the true effect. Use the experimental ground truth as validation.

**Key concepts to understand:**
- Selection bias: if treatment assignment is correlated with potential outcomes, naive comparisons are biased
- Propensity score: the probability of receiving treatment given covariates. Matching or weighting on propensity scores attempts to create pseudo-randomization
- Inverse Probability Weighting (IPW): weight each observation by the inverse of its probability of receiving the treatment it actually received
- Propensity Score Matching: pair treated and control units with similar propensity scores
- Doubly robust estimation: combines outcome modeling and propensity weighting — consistent if either model is correct
- Key insight: you have the experimental effect as ground truth. You can show exactly how much bias observational analysis introduces, and how well (or poorly) causal methods recover the truth

**Analysis steps:**
1. Establish ground truth: the experimental ATE from Section 2
2. Introduce artificial selection bias: create a biased "observational" dataset where treatment assignment depends on covariates. For example: make high-recency, high-history customers more likely to be in the treatment group. Document the bias mechanism clearly
3. Show the naive (biased) estimate: compute the treatment effect without adjustment. It should differ from the experimental ground truth
4. Apply propensity score matching: estimate propensity scores via logistic regression, match treated and control, compute the adjusted ATE
5. Apply IPW: weight observations by inverse propensity scores, compute the weighted ATE
6. Compare all estimates to ground truth: experimental ATE, naive biased ATE, propensity matching ATE, IPW ATE
7. Discuss: how close did each method get? What assumptions did they require? When would these methods fail?

**Deliverable:** Notebook 06 — causal inference analysis with clear comparison to experimental ground truth.

**Notes:**
- The value of this section is the comparison to ground truth — that's something you can rarely do with real observational data, and it makes the lesson concrete
- Keep the bias mechanism simple and well-documented. Complex bias is harder to reason about and doesn't add pedagogical value
- This section connects to your Coursera causal inference course, which is on your resume — the project makes that credential tangible
- If time is short, a clean implementation of just propensity matching + comparison to ground truth is sufficient. IPW and doubly robust are bonuses.

---

## Project Structure

```
ab-testing-causal-inference/
├── data/
│   └── raw/                    # Original CSV (downloaded, never modified)
├── notebooks/
│   ├── 01_data_loading_and_sql.ipynb
│   ├── 02_hypothesis_testing.ipynb
│   ├── 03_power_analysis.ipynb
│   ├── 04_segmentation.ipynb
│   ├── 05_cuped.ipynb
│   └── 06_causal_inference.ipynb
├── reports/
│   └── figures/                # Saved plots for README
├── PLANNING.md
├── README.md
└── environment.yml
```

Keep the structure simple. This project is analysis-focused — it doesn't need the script/notebook duality of the TCGA project since there's no pipeline to reproduce. The notebooks *are* the deliverable.

---

## README Plan

Model after the TCGA project README. Include:

1. **One-paragraph summary** of what the project is and what it demonstrates
2. **Key findings** — a results table or summary of the main conclusions
3. **Setup instructions** — environment, data download
4. **Notebook descriptions** — one sentence per notebook describing what it covers
5. **Design decisions** — document non-obvious choices:
   - Why Hillstrom (real RCT, individual-level data, interpretable covariates, three arms)
   - Why DuckDB
   - Why winsorization approach chosen for spend
   - CUPED covariate selection rationale
   - How artificial bias was introduced in section 6 and why that specific mechanism
   - What the dataset can and can't support (no time-series structure, no sequential testing, limited covariates for CUPED)
6. **Limitations and extensions** — what you'd do with a richer dataset (more covariates for CUPED, timestamp data for sequential testing, larger sample for subgroup analysis)

---

## Daily Practice (Parallel to Project)

- **SQL:** 15–20 min/day on DataLemur, no LLM assistance
- **Reading:** Kohavi chapters aligned to whichever section you're currently building
- **Reimplement:** After finishing each notebook with LLM help, rewrite the core statistical logic (test statistics, CUPED adjustment, propensity scores) from scratch without assistance. This is interview prep.