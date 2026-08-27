# Understanding Rejected AI-Generated Pull Requests

**Tools:** Python &middot; Pandas &middot; NumPy &middot; Seaborn &middot; Matplotlib &middot; Statsmodels &middot; Jupyter Notebook &middot; Hugging Face Datasets &middot; GitHub API &middot; Statistical Analysis

## Project Overview

An investigation into what distinguishes rejected AI-generated pull requests
from merged ones, using the **AIDev dataset** (`hao-li/AIDev` on Hugging
Face) — **33,596 GitHub pull requests** authored by 5 AI coding agents
(`claude_code`, `copilot`, `cursor`, `devin`, `openai_codex`). The analysis
combines quantitative hypothesis testing, effect-size estimation, and
multivariate logistic regression with a structured qualitative coding
framework built around a published taxonomy of PR rejection reasons.

## ⚠️ Important note on the data files

Two data files are **excluded from this repo** because they exceed GitHub's
file size limits:

| File | Size | Why excluded |
|---|---|---|
| `pr_commit_details.parquet` (top level) | ~485 MB | Exceeds GitHub's 100MB hard limit |
| `data/pr_commit_details.parquet` | ~1.3 GB | Same reason, larger snapshot |

**To reproduce the full analysis**, run the notebook's data-loading cells —
`main.ipynb` downloads every AIDev table directly from Hugging Face
(`hf://datasets/hao-li/AIDev/`) on first run and caches it locally, so no
manual download is required for these two files specifically. All other
data files used in this analysis (well under GitHub's limits) **are**
included in this repo.

## Files in this project

| File | Description |
|---|---|
| `main.ipynb` | The full analysis notebook |
| `data/pr_reviews.parquet` | Review-level data (included, ~21MB) |
| `pull_request.parquet`, `pull_requests.parquet`, `pr_task_type.parquet`, `pr_review_comments_v2.parquet`, `pr_ci_status_progress.parquet` | Supporting AIDev tables used throughout the notebook (included) |
| `outlier_sample_for_manual_review.csv` / `.xlsx` | A stratified sample of 100 PRs (large-but-merged and small-but-rejected counterexamples) exported for manual review |
| `deductive_coding_sheet.xlsx` | The coding spreadsheet + taxonomy reference, built from Ehsani et al.'s (2026) published rejection-reason taxonomy |
| `*.png` | Saved visualizations (KDE plots, merge-rate charts) |
| `requirements.txt` | Python dependencies |

## Methodology

### Quantitative analysis
- **Mann-Whitney U tests** comparing PR size (lines changed, files touched) between merged and not-merged PRs
- **Cliff's delta** effect-size estimation alongside every significance test (since with n=33,596, even trivial differences reach statistical significance — effect size tells you whether the difference actually matters)
- **A median-split outlier analysis**: PRs above/below the sample median (94 lines changed, 3 files touched) were labeled "Large"/"Small", then counterexamples to the dominant size-vs-rejection trend (large PRs that still merged; small PRs that were still rejected) were sampled for manual review
- **Multivariate logistic regression** predicting merge outcome from PR size, CI check outcomes, and review activity together, to see which factors matter once the others are controlled for

### Qualitative coding framework (infrastructure built, not yet run on real data)
- A full rejection-reason taxonomy adapted from Ehsani et al. (2026) — 11 categories across 4 levels (Reviewer, Pull Request, Code, Agentic)
- An empty coding sheet generator, ready to populate once an independently sampled set of rejected PRs is manually coded
- A **Cohen's Kappa** inter-rater reliability function, verified against placeholder data (κ = 0.737 on the test run — the function itself is confirmed correct; this is not a real reliability result)
- A **chi-square goodness-of-fit** function ready to compare this study's recomputed category frequencies against Ehsani et al.'s published frequencies (n=562), pending completion of the manual coding round
- A rule for resolving PRs that plausibly fit more than one rejection category, based on the reviewer's explicitly stated reason

**Honest status:** the quantitative analysis below is complete and run on
the real dataset. The qualitative coding framework is fully built and
functionally verified, but requires a manual annotation pass (per the
project's methodology) before it produces real category-frequency results
— that step was not yet completed at the time of this analysis.

## Key Findings

### Overall merge rates
| Agent | Total PRs | % Merged |
|---|---|---|
| openai_codex | 21,799 | 82.6% |
| cursor | 1,541 | 65.2% |
| claude_code | 459 | 59.0% |
| devin | 4,827 | 53.8% |
| copilot | 4,970 | 43.0% |
| **Overall** | **33,596** | **71.5%** |

Merge rates vary substantially by agent — nearly a 2x gap between the
highest (openai_codex) and lowest (copilot) — suggesting agent choice
itself is associated with acceptance likelihood, independent of PR content.

### PR size vs. rejection
- Merged and not-merged PRs differ significantly in size (Mann-Whitney U,
  p ≈ 6.8×10⁻¹⁵³ for lines changed, p ≈ 2.4×10⁻⁵⁴ for files touched) —
  unsurprising at this sample size, but the **effect sizes are small**
  (Cliff's delta: -0.184 for lines changed, -0.107 for files changed),
  meaning PR size alone is a weak practical predictor of rejection.
- This is confirmed by real counterexamples: **13,582 "large" PRs were
  still merged**, and **3,003 "small" PRs were still rejected** — size
  correlates with outcome, but plenty of exceptions exist, which is why a
  stratified sample of 100 of these counterexamples was pulled for manual
  review.

### CI status is the strongest single predictor
- Cliff's delta for failed/neutral CI checks (-0.245) is the **largest
  effect size found across any single factor** in the analysis — notably
  larger than PR size or review activity.
- This holds up in the multivariate logistic regression: controlling for
  PR size and review activity simultaneously, `num_failed_or_neutral` CI
  checks remains highly significant (p < 0.001) and has by far the
  largest standardized effect — each additional failed/neutral check is
  associated with roughly a 14.5% reduction in the odds of a PR being
  merged (odds ratio ≈ 0.855).

### Review activity is a weak predictor once other factors are controlled
- Cliff's delta for number of review comments (-0.052), number of reviews
  (-0.052), and review-driven code changes (-0.038) are all small.
- In the logistic regression, `num_review_comments` (p = 0.855) and
  `review_total_changes` (p = 0.625) were **not statistically significant**
  once PR size and CI status were accounted for — suggesting review
  discussion volume, on its own, doesn't meaningfully predict rejection.

### Model fit
- The 5-feature logistic regression has a pseudo R² of 0.045 — these
  measurable PR characteristics explain only a small share of the variance
  in merge outcome, meaning **most of what determines rejection likely
  isn't captured by size/CI/review-volume metrics alone** and points to the
  qualitative coding phase as the necessary next step to explain the
  remaining variance (e.g., task relevance, code correctness, alignment
  with project conventions).

### Task type
- `Feat` (14,450 PRs, 71.1% merged) and `Fix` (8,106 PRs, 65.0% merged)
  are the two largest task categories.
- `Docs` has the highest merge rate among substantial categories (84.1%),
  while `Perf` has the lowest among categories with meaningful sample size
  (55.3%).

## How to run this analysis

1. `pip install -r requirements.txt`
2. Open `main.ipynb` in Jupyter
3. Run all cells top to bottom — the notebook downloads any missing AIDev
   tables directly from Hugging Face on first run and caches them locally
   (see the "One-Click Data Setup" section), so it does not depend on the
   two large excluded parquet files being present locally.

## CV Bullet Points (for reference)

- Analysed the AIDev dataset (hao-li/AIDev), containing 33,596 GitHub pull requests, to investigate patterns associated with rejected AI-generated pull requests.
- Applied Python-based data cleaning, exploratory analysis and statistical techniques to identify relationships between pull-request characteristics and rejection outcomes.
- Used visualisation and statistical modelling to interpret trends and generate evidence-based insights into the performance of AI-generated contributions.
