# AI Skills Market Intelligence

Pulls live US job postings and developer-activity data from two public APIs and profiles the current AI/data skills landscape: what employers are advertising, what they pay, where the roles are, and what developers are actually asking about.

## Overview

AI hiring commentary is usually built on surveys or vendor reports. This project asks a narrower, more testable question: **what can you actually learn about the AI/data skills market from live, public APIs alone?**

The original brief was to track change since 2022. Building it surfaced a hard constraint: Adzuna's `/search` endpoint returns only *currently-listed* postings, so historical postings from 2022 are not retrievable through it. Rather than fake a time series, the scope became what the data genuinely supports — a **recent US market snapshot**, with the limitation documented rather than papered over. That constraint is the most useful thing the project surfaced, and it shaped the rest of the analysis.

## What I Analyzed

Five questions, all implemented end to end:

- Recent AI/data job posting activity, aggregated by month
- AI/data terminology appearing in job descriptions
- Salary distribution across the sampled postings
- Geographic distribution of postings
- Most-asked technology tags on Stack Overflow

## Data Sources

### Adzuna Jobs API

- **Endpoint:** `/v1/api/jobs/us/search/{page}` — the US endpoint, so this is US postings only
- **Query:** `"AI OR Data OR Machine Learning"`
- **Pagination:** 3 pages, 150 postings in the saved run
- **Fields retained:** title, company, location, salary_min, salary_max, category, created, description
- **Nature:** live and current — the endpoint returns postings open at request time, not an archive

### Stack Exchange / Stack Overflow API

- **Endpoint:** `/2.3/tags` (`site=stackoverflow`, `sort=popular`)
- **Sample:** top 100 tags by question volume
- **Fields retained:** tag_name, question_count, has_synonyms
- **Nature:** all-time cumulative question counts with **no time dimension**. These describe accumulated developer activity, not a trend, and cannot be aligned to the posting timeline.

## Data Pipeline

```
API retrieval → pagination → JSON flattening → pandas DataFrames → cleaning
→ datetime handling → salary handling → AI/ML keyword detection
→ aggregation → visualization
```

Adzuna returns deeply nested JSON (`company`, `location`, `category` are objects), so each posting is flattened to a flat record before it reaches pandas. Cleaning standardizes column names, parses `created` to timezone-naive datetimes, and fills missing salaries with `0` — which are then excluded from the salary chart so they are never treated as real compensation.

Keyword detection matches multi-word terms as plain substrings, but matches the short tokens `ai` and `ml` on **word boundaries**. Without that, `ml` matches `html` and `xml`, and `ai` matches `available`, `training` and `maintain` — enough to materially inflate the counts.

Credentials are never hardcoded. They load from Colab Secrets, falling back to environment variables outside Colab.

## Final Snapshot

From the saved Phase IV run — 150 US postings and 100 Stack Overflow tags:

| | |
|---|---|
| Adzuna postings analyzed | 150 (3 paginated pages) |
| Cleaned dataframe | 150 rows × 8 columns |
| Posting dates observed | August 2024 – September 2026 |
| Concentration | 120 of 150 (80.0%) created July–September 2026 |
| Largest single month | August 2026, 60 postings |
| Matched the AI keyword indicator | 79.3% (119 of 150) |
| Stack Overflow tags | 100 |

**Keyword counts across the 150 descriptions**

| Term | Count |
|---|---|
| ai | 105 |
| data | 89 |
| machine learning | 56 |
| python | 16 |
| sql | 8 |
| deep learning | 2 |

**Top Stack Overflow tags:** javascript (2,521,825), python (2,204,680), java (1,914,472), c# (1,621,768), php (1,461,062)

## Key Findings

**79.3% of the 150 sampled postings (119 of 150) matched at least one AI-related term** under the word-boundary keyword indicator. Given the query itself targets AI/data roles, this measures how consistently that vocabulary appears in the posting text, not the AI share of the wider job market.

**Postings are heavily weighted toward the most recent months** — 120 of 150 (80.0%) created in July–September 2026, peaking at 60 in August 2026, with the remainder trailing back sparsely to August 2024. This is a property of the endpoint, which lists only open postings, and is not evidence of growth or seasonality.

**Generic AI vocabulary far outweighs named tools.** `ai` (105), `data` (89) and `machine learning` (56) dominate, then counts fall sharply to `python` (16), `sql` (8) and `deep learning` (2). Adzuna truncates descriptions to roughly 500 characters, and concrete tools usually sit in the requirements section below that cutoff — so the tool-level numbers are a floor, not a demand estimate.

**Salaries concentrate between roughly \$85k and \$205k**, with the heaviest bin around \$165k–\$190k and a thin tail past \$400k. Adzuna flags part of its salary data as model-predicted, so this blends employer-stated and estimated pay.

**Stack Overflow's most-asked tags are general-purpose languages** — javascript, python, java, c# — not AI-specific tooling. Since these are undated cumulative totals, they sit alongside the job data as context, not as a comparison or a leading indicator.

## Visual Analysis

1. **Monthly posting snapshot** (Bokeh, interactive) — postings per month with hover detail. A 3-month moving average is drawn purely as a smoothing aid across the observed window; with this few points it is not evidence of a trend.
2. **Skill keyword frequency** (Matplotlib) — term counts across descriptions, top term highlighted.
3. **Stack Overflow tag popularity** (Bokeh, interactive) — top 10 tags by cumulative question count.
4. **Salary distribution** (seaborn) — histogram with KDE over positive salaries, banded by pay range.
5. **Geographic distribution** (Matplotlib) — top 10 locations by posting count.

> The two Bokeh charts render as interactive JavaScript and will not display in GitHub's static notebook preview. Open those notebooks in Colab or nbviewer to see them.

## Methodological Lessons / Limitations

**Endpoint choice determines historical coverage.** The gap between the original 2022–present ambition and what shipped comes down to one API decision. A search endpoint that serves current listings cannot answer a historical question, no matter how the data is processed downstream.

**Pagination bugs silently change your sample size.** An early version paginated across 3 pages into a 150-row frame, then rebuilt that frame from `response.json()` — the final page only. Everything downstream ran on 50 rows while appearing to run on 150. Nothing errored; the charts just quietly described a third of the data.

**Short tokens need anchored matching.** `ai` and `ml` as bare substrings match `available`, `training`, `maintain`, `html` and `xml`. Word boundaries fixed it while still catching `AI/ML`, `AI-powered` and `ml-ops`.

**Truncated text caps what text analysis can claim.** With ~500 characters per description, skill extraction sees the opening pitch rather than the requirements, which biases tool-level counts downward.

**Cross-source correlation needs an aligned time dimension.** Adzuna postings carry dates; Stack Overflow tag counts are cumulative totals that do not. The two datasets are analyzed side by side and never joined — no correlation is claimed, and no statistical tests were performed.

**Live APIs limit reproducibility.** Both sources change continuously, so re-running produces different numbers. Every figure here is tied to one specific execution rather than a fixed dataset.

## Repository Structure

```
notebooks/
  02_data_collection_and_cleaning.ipynb
  03_analysis_and_visualization.ipynb
  04_final_analysis.ipynb
```

- [02_data_collection_and_cleaning.ipynb](notebooks/02_data_collection_and_cleaning.ipynb) — Phase II: connects to both APIs, flattens nested JSON into DataFrames, and handles column names, dates, missing salaries, and the AI keyword indicator.
- [03_analysis_and_visualization.ipynb](notebooks/03_analysis_and_visualization.ipynb) — Phase III: introduces pagination to 150 postings and establishes the five analytical questions with a first pass of Matplotlib charts.
- [04_final_analysis.ipynb](notebooks/04_final_analysis.ipynb) — Phase IV: the final run, refining the same five questions with interactive Bokeh and seaborn visualizations. **This notebook holds the results reported above.**

Phases III and IV answer the same five questions; Phase IV refines the visualizations and carries the corrected, final numbers.

## Tech Stack

Python, pandas, NumPy, requests, Matplotlib, seaborn, Bokeh, Adzuna API, Stack Exchange API, Google Colab, Git/GitHub.

## Running the Project

You will need your own free Adzuna API credentials from [developer.adzuna.com](https://developer.adzuna.com/):

```
ADZUNA_APP_ID
ADZUNA_APP_KEY
```

**In Google Colab** — add both as Colab Secrets (sidebar → key icon) and enable notebook access. The notebooks read them with `userdata.get()`.

**Locally** — export the same two names as environment variables:

```bash
export ADZUNA_APP_ID="your_app_id"
export ADZUNA_APP_KEY="your_app_key"
```

Then run the notebooks in order. The Stack Overflow API needs no key.

Both APIs are live, so your outputs will differ from the ones saved here — posting counts, dates, salaries, and tag totals all move between runs.

## Project Context

Originally developed as a multi-phase project for NYU's Dealing with Data course, then cleaned, debugged, and reorganized as a public technical portfolio project.
