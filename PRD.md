# PRD -- AI Workforce Exposure Dataset Construction

What this pipeline produces and why.

---

## 1. Overview

This pipeline constructs the task-level AI exposure datasets for the paper *Mapping AI Exposure Across the U.S. Workforce*. It takes raw data from multiple AI scoring sources, O*NET occupational data, and BLS employment and wage statistics, and merges them into a unified format where each row is one task within one occupation.

This repository is the reference for how the data was built, how to reproduce it, and how to extend it with new dataset versions.

---

## 2. Data Sources

### AI Scoring Sources (Inputs)

**Anthropic Economic Index (AEI)** -- Derived from real Claude conversation data. Each O*NET task gets a `pct_normalized` (share of conversations) and `auto_aug_mean` (automatability score, 0--5). v1--v5 use O*NET v20.1 (2015) task statements and 2010 SOC codes. **v6.1/v6.2 (June 2026 release) switched to O*NET v30.1 (2025) task statements and 2019 SOC codes**, and use a new raw schema (`category_name`/`metric_id`/`node_name` instead of `facet`/`variable`/`cluster_name`).

- AEI Conv. v1--v5: Five snapshot versions (Dec 2024 -- present), conversation data only.
- AEI API v3--v5: Three snapshot versions, API/tool-use interactions only.
- AEI Conv./API v6.1 and v6.2: Two monthly snapshots (Apr 2026, May 2026) from the June 2026 release, shipped together in one raw file per platform.
- Input files: `task_pct_v{1..5}.csv`, `task_pct_api_v{3..5}.csv`, `task_pct_v6_{1,2}.csv`, `task_pct_api_v6_{1,2}.csv` (v6 files are month-filtered from `aei_raw_{claude_ai,1p_api}_2026-04-01_to_2026-06-01.csv`)
- Auto/aug scores: `automation_vs_augmentation_by_task_v{2..5}.csv`, `automation_vs_augmentation_by_task_api_v{3..5}.csv`, plus `_v6_{1,2}` and `_api_v6_{1,2}` variants
- v6 caveat: the June 2026 release rounds all values to 2 decimals and ships no conversation counts, so absolute conversation volumes are not recoverable and roughly half of the listed tasks arrive at `pct = 0.00`. Those zeros mean "observed, but below the reporting precision", not "unused": AEI only lists a task once it clears a suppression floor of 15 observations per 1,000,000, so a `0.00` is censored into `[0.0015, 0.005)`. The pipeline fills them with 0.0027 — the mean of that interval measured from v5, which reports it uncensored — so v6's low-usage tail stays comparable with earlier versions instead of dropping to zero at the format change. See ARCHITECTURE.md §3 and §10 pitfall 14.

**MCP Server Pipeline** -- AI task classifications from Model Context Protocol server logs. Four cumulative versions (Apr 2025 -- Feb 2026). Uses O*NET v30.1 (2025) task statements and 2019 SOC codes.

- Input files: `task_results_{date}.csv` (one per version date)

**Microsoft Copilot Analysis** -- Assessment of AI automation/augmentation from Copilot usage data (Sep 2024). Uses IWA-level metrics mapped to tasks. Also includes physical task flags.

- Input file: `iwa_metrics.csv`

### Structural & Economic Sources (Inputs)

**O*NET** -- Task statements (v20.1 for AEI, v30.1 for MCP/Microsoft/ECO), task ratings (frequency, importance, relevance), work activity taxonomy (DWA/IWA/GWA).

**BLS OEWS** -- Employment counts and median wages by occupation, national and all 54 US states/territories, for both 2025 and 2015.

**SOC Crosswalk** -- Maps 2010 SOC codes to 2019 SOC codes for AEI data compatibility.

**Task Time Estimates** -- Per-task time use from *Estimating Time Spent on Work* (17,525 tasks across 876 occupations, O*NET v30.1 task text and 2019 SOC). Estimates come from a linear program over LM pairwise task-duration comparisons, calibrated against CPS per-occupation hours. Supplies `time_per_day` (hours per day, budgeted so an occupation's tasks sum to a 7-hour workday) and `time_per_instance` (hours for one execution).

- Input file: `task_time_share_estimates.csv`

---

## 3. Output Datasets

All final outputs are saved to `data/final/`. Each row is one task within one occupation.

### Per-Version Datasets (19 files)

| Dataset | File | SOC Version |
|---------|------|-------------|
| AEI Conv. v1--v5 | `final_aei_v{1..5}.csv` | 2010 |
| AEI API v3--v5 | `final_aei_api_v{3..5}.csv` | 2010 |
| AEI Conv. v6.1--v6.2 | `final_aei_v6_{1,2}.csv` | 2019 |
| AEI API v6.1--v6.2 | `final_aei_api_v6_{1,2}.csv` | 2019 |
| MCP Cumul. v1--v4 | `final_mcp_v{1..4}.csv` | 2019 |
| Microsoft | `final_microsoft.csv` | 2019 |
| ECO 2025 | `final_eco_2025.csv` | 2019 |
| ECO 2015 | `final_eco_2015.csv` | 2010 |

### Cumulative Datasets (69 files)

Built by combining per-version datasets across sources. For each bucket, a new cumulative version is produced each time a new dataset arrives chronologically. For overlapping (occupation, task) pairs, `auto_aug_mean` is averaged across all contributing source versions (nulls skipped) and `pct_normalized` is summed across sources, after which the dataset is renormalized so unique (occupation, task) pairs sum to 100.

Microsoft is pre-cleaned upstream (Part 2 cleanup cell): physical rows under AI-dominant 2x IWAs and Eloundou-E0-on-both-sides rows are dropped. Cumulative buckets that include Microsoft inherit this cleanup automatically.

AEI v1 has no auto/aug data. In cumulative builds, v1's `auto_aug_mean` is imputed for v1-exclusive tasks from a DWA→IWA→GWA mean over the other AEI versions in the same build (starting from the 2025-03-06 version, once v2 is present). v1-exclusive tasks with no O*NET work-activity mapping are dropped. See ARCHITECTURE.md §6 for detail.

Output naming: `final_{bucket_name}_{end_date}.csv`

**Usage buckets:**

| Bucket | Description | Sources | Task Set | Versions |
|--------|-------------|---------|----------|----------|
| `all_confirmed_usage` | All confirmed usage | AEI Both + Microsoft | 2025 | 8 (2024-09-30 to 2026-05-31) |
| `confirmed_human_usage` | Confirmed human usage | AEI Conv + Microsoft | 2025 | 8 (2024-09-30 to 2026-05-31) |
| `aei_all_usage_eco2015` | AEI all confirmed usage | AEI Conv + AEI API | 2015 | 5 (2024-12-23 to 2026-02-12) |
| `aei_human_usage_eco2015` | AEI confirmed human usage | AEI Conv only | 2015 | 5 (2024-12-23 to 2026-02-12) |
| `aei_agentic_usage_eco2015` | AEI confirmed agentic usage | AEI API only | 2015 | 3 (2025-08-11 to 2026-02-12) |
| `aei_all_usage_eco2025` | AEI all confirmed usage | AEI Conv + AEI API | 2025 | 7 (2024-12-23 to 2026-05-31) |
| `aei_human_usage_eco2025` | AEI confirmed human usage | AEI Conv only | 2025 | 7 (2024-12-23 to 2026-05-31) |
| `aei_agentic_usage_eco2025` | AEI confirmed agentic usage | AEI API only | 2025 | 5 (2025-08-11 to 2026-05-31) |

The three AEI-only buckets ship in both task spaces. `_eco2015` preserves the native O*NET v20.1 structure used for work-activity analysis and ends at 2026-02-12, since AEI v6 cannot be expressed in v20.1 task space. `_eco2025` carries the same sources onto the ECO 2025 backbone, which admits v6.1/v6.2 and extends the series to 2026-05-31.

**Agentic bucket:**

| Bucket | Description | Sources | Task Set | Versions |
|--------|-------------|---------|----------|----------|
| `all_agentic_usage` | All possible agentic usage | MCP + AEI API | 2025 | 9 (2025-04-24 to 2026-05-31) |

**All bucket:**

| Bucket | Description | Sources | Task Set | Versions |
|--------|-------------|---------|----------|----------|
| `all_usage` | All usage potential | AEI Both + MCP + Microsoft | 2025 | 12 (2024-09-30 to 2026-05-31) |

2025 task set buckets use ECO 2025 as structural backbone. Pre-v6 AEI/API sources are filtered to task-occupation pairs that exist in ECO 2025 before combining; AEI v6.1/v6.2 are native O*NET v30.1 / 2019-SOC sources and match the backbone on `title_current` directly. 2015 task set buckets use native AEI row structure -- **AEI v6.1/v6.2 do not enter the 2015-task-set buckets** (different O*NET vintage), so those buckets end at 2026-02-12.

### ECO Baseline Datasets (2 files)

Economy-wide task baseline (denominator for exposure calculations). `pct_normalized` and `auto_aug_mean` are all zero/null -- values come from AI datasets only.

- `final_eco_2025.csv` -- Primary baseline for MCP/Microsoft (2019 SOC).
- `final_eco_2015.csv` -- Baseline for AEI work-activity analysis (2010 SOC).

---

## 4. Shared Output Columns

Every final dataset includes these columns (some may be null depending on dataset type):

| Column | Description |
|--------|-------------|
| `task`, `task_normalized` | Raw and normalized O*NET task text |
| `title` | Occupation title (2010 SOC for AEI/ECO 2015) |
| `title_current` | Occupation title (2019 SOC, MCP/Microsoft/ECO 2025 only) |
| `soc_code_2010` | O*NET SOC code (2010 system) |
| `dwa_title`, `iwa_title`, `gwa_title` | Work activity hierarchy |
| `broad_occ`, `minor_occ_category`, `major_occ_category` | SOC occupation hierarchy |
| `pct_normalized` | Share of AI usage for this task (sums to ~100 per dataset on unique title x task pairs) |
| `auto_aug_mean` | Automatability score (0--5) |
| `physical` | Boolean: is this a physical task |
| `freq_mean` | Task frequency (daily occurrence rate from O*NET survey) |
| `importance` | Task importance (1--5 from O*NET survey) |
| `relevance` | Task relevance (0--100 from O*NET survey) |
| `time_per_day` | Hours per day spent on this task. Renormalized so an occupation's full task list sums to a 7-hour workday on the ECO backbones; on an AI dataset (a subset of tasks) the per-occupation sum is therefore the hours of the workday that dataset covers |
| `time_per_instance` | Hours to complete one execution of the task (not renormalized) |
| `emp_tot_nat_2025` | National employment (BLS OEWS 2025) |
| `a_med_nat_2025` | National median annual wage (BLS OEWS 2025) |
| `job_zone` | O*NET Job Zone (1--5), ECO 2025 only |
| `emp_change_pct_2024_2034` | BLS projected % change in employment, 2024--2034 (ECO 2025 only) |
| `eloundu_human` | Human-labeled exposure (E0/E1/E2) from Eloundu et al. `full_labelset.tsv` |
| `eloundu_gpt4` | GPT-4-labeled exposure (E0/E1/E2) from Eloundu et al. `full_labelset.tsv` |
| `date` | Dataset snapshot date |

Plus state-level wage and employment columns for all 54 US states/territories (`emp_{abbrev}`, `a_median_{abbrev}`, `h_median_{abbrev}` for both 2025 and 2015), and 2015 wage/employment columns (nominal and inflation-adjusted).

---

## 5. Pipeline Workflow

The pipeline runs in three parts from a single notebook (`scripts/data_merge.ipynb`).

**Part 1** runs once per dataset (set `run_name` and re-execute). It maps raw AI data to O*NET tasks, adds SOC structure, merges BLS wage/employment for 2025 and 2015, adjusts employment for SOC decimal codes, and does initial cleanup. Output: `first_pass_*.csv`.

**Part 2** runs once across all datasets. It adds snapshot dates, O*NET taxonomy (DWA/IWA/GWA), physical task flags, and standardized auto_aug scores. Output: `second_pass_*.csv`.

**Part 3** runs once across all datasets. It merges task ratings (frequency, importance, relevance) and task time estimates (`time_per_day`, `time_per_instance`), adds task_prop for ECO 2025, builds cumulative AEI datasets, adds DWS ratings, and does final column reordering. Output: `third_pass_*.csv` then `final_*.csv`.

---

## 6. Dataset Dates

| Dataset | Snapshot Date |
|---------|--------------|
| AEI Conv. v1 | 2024-12-23 |
| AEI Conv. v2 | 2025-03-06 |
| AEI Conv. v3 | 2025-08-11 |
| AEI Conv. v4 | 2025-11-13 |
| AEI Conv. v5 | 2026-02-12 |
| AEI API v3 | 2025-08-11 |
| AEI API v4 | 2025-11-13 |
| AEI API v5 | 2026-02-12 |
| AEI Conv. v6.1 | 2026-04-30 |
| AEI API v6.1 | 2026-04-30 |
| AEI Conv. v6.2 | 2026-05-31 |
| AEI API v6.2 | 2026-05-31 |
| MCP Cumul. v1 | 2025-04-24 |
| MCP Cumul. v2 | 2025-05-24 |
| MCP Cumul. v3 | 2025-07-23 |
| MCP Cumul. v4 | 2026-02-18 |
| Microsoft | 2024-09-30 |

---

## 7. Other Scripts

### `code_storage.ipynb`

Provenance scripts, not part of the main pipeline run. Contains the one-off transformations that produced several committed inputs under `data/`: physical-flag merge (`tasks_dwa_iwa_gwa_v30.1_physical.csv`), AEI collaboration-pattern pivots (`automation_vs_augmentation_by_task_*.csv`), ECO baseline task lists (`task_pct_eco_{2015,2025}.csv`), and master title normalization (`master_pct_normalized.csv`), plus the GWA distributions used in the usage-debiasing analysis. Kept for transparency; see the header cell in the notebook for re-run caveats.
