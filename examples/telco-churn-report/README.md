# Example: Telco Customer Churn Analysis

An end-to-end demonstration of the `data-analysis-report` skill, run against a public business dataset with an English-language output.

**[→ View the rendered report](./index.html)** (download/open locally, or view via GitHub Pages / `raw.githack.com` if enabled)

## What this shows

- Data quality audit (duplicate check, missing-value handling, cleaning decisions disclosed)
- Two-proportion Z-tests with Wilson 95% confidence intervals for group comparisons
- Benjamini–Hochberg correction for multiple comparisons (7 simultaneous tests)
- A dual-panel "efficiency vs. structure" chart used to avoid a Simpson's-paradox-style misread of the churn-by-tenure trend
- Explicit confounder callouts where a correlation reverses direction once a hidden variable is controlled for
- Evidence-strength badges (Reliable / Directional / Hypothesis) on every conclusion, with quantified numbers instead of vague claims

## Dataset

**IBM Telco Customer Churn** — a publicly released sample dataset (7,043 customers, 21 fields) widely used for churn-analytics tutorials and education. Included here as `telco-customer-churn.csv` for reproducibility.

- Original source: [Kaggle — Telco Customer Churn](https://www.kaggle.com/datasets/blastchar/telco-customer-churn)
- Mirror used to fetch this copy: [IBM/telco-customer-churn-on-icp4d](https://github.com/IBM/telco-customer-churn-on-icp4d)

## Note

This is a demonstration report showcasing the skill's analytical process on a public dataset — not an operational business report for any real company.
