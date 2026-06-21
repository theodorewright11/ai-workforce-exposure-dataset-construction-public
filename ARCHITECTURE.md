# ARCHITECTURE.md -- AI Workforce Exposure Dataset Construction

Technical reference for how the pipeline works. Read this before making changes.
Paired with [PRD.md](PRD.md) (what it produces).

---

## 1. Repository Structure

```
ai-workforce-exposure-dataset-construction-public/
├── scripts/
│   ├── data_merge.ipynb           # Main pipeline (3 parts)
│   └── code_storage.ipynb         # Provenance scripts for committed inputs (not part of the run)
├── data/
│   ├── (root)                     # Raw inputs (committed) + generated intermediate files
│   ├── final/                     # Final output CSVs (generated: 16 per-version + 44 cumulative = 60)
│   └── merged_data_files/         # Generated intermediate saves
├── README.md
├── PRD.md
├── ARCHITECTURE.md
├── requirements.txt               # pandas, numpy, matplotlib, seaborn, scipy, langdetect
├── .gitattributes                 # Keeps committed data files byte-exact across platforms
└── .gitignore                     # Commits raw inputs only; ignores generated files
```

---

## 2. Pipeline Overview

The main pipeline lives in `scripts/data_merge.ipynb`. It has three parts:

| Part | Scope | Input | Output | Run Pattern |
|------|-------|-------|--------|-------------|
| Part 1 | Pcts, wages, emp | Raw CSVs | `first_pass_*.csv` | Once per dataset (set `run_name`) |
| Part 2 | Dates, taxonomy, auto/aug | `first_pass_*.csv` | `second_pass_*.csv` | Once for all datasets |
| Part 3 | Ratings, cumulative, reorder | `second_pass_*.csv` | `final_*.csv` | Once for all datasets |

---

## 3. Configuration

### Run Names

The `runs` dictionary maps output filenames to their raw input config. Valid `run_name` values:

```
first_pass_aei_v1.csv          first_pass_aei_api_v3.csv
first_pass_aei_v2.csv          first_pass_aei_api_v4.csv
first_pass_aei_v3.csv          first_pass_aei_api_v5.csv
first_pass_aei_v4.csv          first_pass_mcp_v1.csv
first_pass_aei_v5.csv          first_pass_mcp_v2.csv
first_pass_microsoft.csv       first_pass_mcp_v3.csv
first_pass_eco_tasks_2015.csv  first_pass_mcp_v4.csv
first_pass_eco_tasks_2025.csv
```

Dataset type is auto-detected from the run name:
- `aei_run`: `"aei" in run_name`
- `mcp_run`: `"mcp" in run_name`
- `microsoft_run`: `"microsoft" in run_name`
- `eco_run_2015` / `eco_run_2025`: exact match
- `aei_v3_and_up`: AEI runs with v3, v4, or v5 (have auto/aug data from Anthropic)

### Constants

```python
frequency_weights = {1: 1/260, 2: 4/260, 3: 24/260, 4: 104/260, 5: 1, 6: 3, 7: 8}
jan_2020_inflation_factor = 1.24   # scraped wages to present value
may_2015_inflation_factor = 1.36   # 2015 BLS wages to present value
```

`frequency_weights` is defined twice in `data_merge.ipynb`: an earlier constants
cell (`{2: 3/260, 3: 48/260, 4: 130/260, 7: 12}`) and a later cell that redefines it
right before the task-ratings imputation (`{2: 4/260, 3: 24/260, 4: 104/260, 7: 8}`,
shown above). The later definition is the operative one for all `freq_mean` values.

### States

All 54 US states/territories are processed (not just Utah):
`al, ak, az, ar, ca, co, ct, de, dc, fl, ga, hi, id, il, in, ia, ks, ky, la, me, md, ma, mi, mn, ms, mo, mt, ne, nv, nh, nj, nm, ny, nc, nd, oh, ok, or, pa, ri, sc, sd, tn, tx, ut, vt, va, wa, wv, wi, wy, gu, pr, vi`

---

## 4. Part 1: Core Processing (Pcts, wages, emp)

### Helpers

`normalize_text(text)` -- Lowercases, removes punctuation, collapses whitespace. Used for all task and title matching throughout the pipeline.

### Step 1: Map Raw Data to O*NET Tasks

Four separate code paths depending on dataset type:

**AEI path** (`pct_to_onet_tasks()`):
- Loads `task_pct_v*.csv` (Anthropic's task percentage mappings)
- For v1--v2: merges with O*NET v20.1 task statements on normalized task text
- For v3+: filters raw AEI data for `GLOBAL/onet_task` entries, extracts pct values
- Counts `n_occurrences` per task as a total of duplicate task names that map to different occupations, divides pct by occurrence count for `pct_normalized`
- Normalizes so all rows sum to 100

**MCP path**:
- Loads `task_results_{date}.csv`
- Winsorizes `n_ratings` at 75th percentile + 1.5*IQR
- Computes `pct = 100 * n_ratings_capped / sum(n_ratings_capped)` so unique (occupation, task) pairs sum to 100
- Maps to O*NET v30.1 task statements
- Crosswalks SOC codes to 2019 where needed

**Microsoft path**:
- Loads `iwa_metrics.csv` (IWA-level `share_user`, `share_ai` metrics)
- Computes `total_share = share_user + share_ai` per IWA, then normalizes so `total_share` sums to 1 across retained IWAs
- For each IWA, divides `total_share` by the count of **unique (Title, task) pairs** under that IWA to get `pair_share`, so each pair under that IWA gets an equal slice of the IWA's share
- For each unique (Title, task) pair, takes the **mean** `pair_share` across the IWAs it belongs to (each IWA-appearance represents the same underlying conversation, so the shares are split evenly rather than summed), then rescales so unique pairs sum to 100
- Attaches taxonomy (DWA/IWA/GWA) by inner-merging back onto the taxonomy table on `norm_iwa_title`, which drops taxonomy rows whose IWA was filtered out of `iwa_metrics` (share < 0.0005). A pair with IWAs A (surviving) and B (filtered) only appears under A's DWA/GWA rows, not B's. `pct` duplicates across the surviving taxonomy rows for each pair; `impact_scope_*` is attached per taxonomy row via the IWA key and is non-null by construction. The `impact_scope_*_adj` formulas use `(share_user + share_ai)` explicitly as the denominator so the within-IWA AI/user weighting still holds after `total_share` is normalized.

**ECO path**:
- Loads pre-built `task_pct_eco_{2015,2025}.csv`
- Maps to appropriate O*NET task statements (v20.1 for 2015, v30.1 for 2025)
- Crosswalks SOC codes for 2025 version

### Step 2: Add SOC Structure

`add_soc_structure()` merges occupation hierarchy from `soc_structure_2019.csv`:
- `major_occ_category` (23 categories, matched on first 2 digits of SOC)
- `minor_occ_category`
- `broad_occ`
- `broad_counts` (number of detailed occupations per broad category)

Hard-coded SOC anomaly fixes:
- 15-1200 -> 15-1000
- 31-1100 -> 31-1000
- 51-5100 -> 51-5000
- 29-122x/123x -> 29-1210

### Step 3: Add 2025 Wage & Employment

Six sub-functions, run sequentially:

1. `add_updated_soc_code()` -- Maps titles to 2019 SOC codes via crosswalk (needed for BLS merge)
2. `add_nat_wage_2025()` -- National wages from OEWS 2025. Fallback chain: detailed SOC -> broad SOC -> scraped O*NET wages (with 1.24x inflation) -> hourly/annual conversion
3. `add_state_wage_2025()` -- State wages from OEWS 2025. Fallback: hourly/annual conversion -> national wages
4. `add_nat_emp_2025()` -- National employment from OEWS 2025. Fallback: broad category divided by `broad_counts`
5. `add_state_emp_2025()` -- State employment from OEWS 2025. Fallback: national employment * (state total / national total)
6. `wage_emp_to_tasks_2025()` -- Merges all wage/emp columns into the task-level DataFrame

For titles that map to multiple 2019 SOC codes: wages are averaged, employment is divided by duplicate count then summed.

### Step 4: Add 2015 Wage & Employment

Same structure as Step 3 but with 2015 BLS data and 1.36x inflation factor. No crosswalk needed (merges on 2010 SOC directly). Produces both nominal and real (inflation-adjusted) wage columns.

### Step 5: Adjust Employment for Decimal SOC Codes

O*NET uses decimal SOC codes (e.g., 11-1011.00 vs 11-1011.03) but BLS only has 6-digit codes. Without adjustment, employment would be counted multiple times for occupations with multiple decimal variants.

Two functions that share the same logic but key on different columns:
- `adjust_emp_old()` -- For AEI/ECO: keys on `soc_code_2010` and `title`
- `adjust_emp_new()` -- For MCP/Microsoft: keys on `soc_code_2019_full` and `title_current`

Both work the same way:
1. Group rows by 6-digit SOC code (trim the decimal)
2. Use `master_pct_normalized` as the proportioning key, with **square root dampening** to reduce the influence of extreme values
3. For each SOC6 group where multiple titles share the same employment number: split the total proportionally by each title's dampened pct share
4. If pct data is all zeros for a group, fall back to equal split across titles

### Step 6: Final Cleanup

`final_cleanup()`:
- Data validation (column presence, missing values)
- Missing value imputation for task_type (Core if relevance >= 67 and importance >= 3, else Supplemental)
- Column ordering
- Saves `first_pass_*.csv`

---

## 5. Part 2: Taxonomy, Dates, Auto/Aug

Runs once across all datasets (loads all `first_pass_*.csv` files).

### Dates & Taxonomy

- Maps each dataset to its snapshot date
- Merges O*NET DWA/IWA/GWA taxonomy from `tasks_dwa_iwa_gwa_v{20.1,30.1}.csv`
- Merges physical task flags from `physical_tasks.csv` (sourced from Microsoft's analysis, with DWA majority-rule imputation for unmapped tasks)

### Microsoft cleanup cell

Inserted immediately after the dates/taxonomy/physical step and operates only on `first_pass_microsoft_dates_taxonomy_physical.csv`. AEI/MCP/ECO files pass through untouched. Two filters applied in sequence, file overwritten in place:

1. **AI-dominant physical filter.** Computes the set of "AI-dominant 2x" IWAs from `iwa_metrics.csv` (`share_ai >= 2 * share_user` AND `share_ai >= 0.0005`). Drops Microsoft rows where `physical == True` AND `iwa_title` is in that set. This mirrors the paper's `w_nonphys` adjustment: it filters only physical instances where Copilot's AI side dominates the user side, so the informational signal is not penalized along with the genuinely physical tasks.

2. **Eloundou no-exposure filter.** Temporarily merges Eloundou's `eloundu_human` (`human_exposure_agg`) and `eloundu_gpt4` (`gpt4_exposure`) labels from `full_labelset.tsv` using a pair lookup with task-only fallback (same logic as the later final-stage Eloundou cell). Drops rows where both labels are `"E0"` (no exposure). Eloundou columns are then dropped; the final-stage Eloundou cell reattaches them after the cumulative builds.

3. **Renormalize `pct_normalized`.** The two filters drop rows whose pct values still hold their pre-drop magnitudes, so the unique-pair sum falls below 100 after filtering. Rescale uniformly: `pct *= 100 / current_unique_pair_sum`. Proportional relationships are preserved; the ability to back-derive raw conversation counts is lost (it is already an approximation after the IWA-to-pair distribution step in Part 1).

The two filters together drop roughly a quarter of Microsoft's rows and unique pairs (the Eloundou filter accounts for most of it); renormalization then returns the unique-pair sum to 100. Filtering at this stage means every downstream consumer (cumulative buckets and any derived statistics) inherits the cleanup. No filter flags or filtered-variant output files exist downstream.

### Auto/Aug Standardization

Different processing per source:

**Microsoft**: The source data (`iwa_metrics.csv`) has `impact_scope_ai` and `impact_scope_user` at the IWA level, plus `share_ai` and `share_user`. These are combined into a usage-weighted `impact_scope_avg` (= `impact_scope_ai_adj + impact_scope_user_adj`). `auto_aug_mean` is then computed at the IWA level as **`5 * impact_scope_avg`** (CONT method): a direct linear remap of the 0–1 scope to the 0–5 rating scale. The IWA-level value is propagated to every task row under that IWA.

The previous distribution-based method (mapping each IWA scope through MCP rating distributions binned by scope) is preserved in the cell as a commented-out reference block. It was retired because it heavily compressed Microsoft's task-level distribution, which made Microsoft an uninformative source for the cumulative max where AEI typically dominates.

**AEI v2+**: Loads `automation_vs_augmentation_by_task_*.csv` from Anthropic's data release (v2--v5 conversation, api_v3--api_v5). Standardizes across versions into a single `auto_aug_mean` column.

**AEI v1**: No auto/aug data available (column stays null).

**MCP**: Already has `auto_aug_mean` and `auto_aug_mean_adj` (adjusted, preferred) from the classification pipeline.

Output: `second_pass_*.csv`

---

## 6. Part 3: Ratings, Cumulative, Final

Runs once across all datasets.

### Task Ratings

- Loads O*NET task ratings (`task_ratings_v30.1.csv`, `task_ratings_may_2025.csv`, `task_ratings_oct_2015.csv`)
- Applies frequency weights to convert survey responses to daily frequency values
- Merges `freq_mean`, `importance`, `relevance` into each dataset
- Imputes missing ratings with an 8-level fallback chain (min 5 values at each level to use it):
  1. (title, task) direct merge
  2. Task-only average (across all occupations sharing that task)
  3. Occupation-level average
  4. DWA (Detailed Work Activity) average
  5. IWA (Intermediate Work Activity) average
  6. GWA (General Work Activity) average
  7. Major occupation category average
  8. Global mean
- **Penalty**: if an occupation has >50% of its tasks imputed below occupation level, those imputed `freq_mean` values are halved
- For ECO 2025 only: computes `task_prop = count_2015_tasks / count_2025_tasks` per occupation (2015 tasks mapped to 2025 occupations via SOC crosswalk). Used downstream as a deflation factor when crosswalking AEI data.

### Cumulative Datasets

Nine buckets of cumulative datasets are built, each with one or more time-point versions (one per new dataset arrival). Output naming: `final_{bucket_name}_{end_date}.csv`.

**Buckets:**

| Bucket | Sources | Task Set | Versions |
|--------|---------|----------|----------|
| `all_confirmed_usage` | AEI Both + Microsoft | 2025 | 6 |
| `confirmed_human_usage` | AEI Conv + Microsoft | 2025 | 6 |
| `aei_all_usage` | AEI Conv + AEI API | 2015 | 5 |
| `aei_human_usage` | AEI Conv only | 2015 | 5 |
| `aei_agentic_usage` | AEI API only | 2015 | 3 |
| `all_agentic_usage` | MCP + AEI API | 2025 | 7 |
| `aei_agentic_usage_2025` | AEI API only | 2025 | 1 (latest date only) |
| `aei_all_usage_2025` | AEI Conv + AEI API | 2025 | 1 (latest date only) |
| `all_usage` | AEI Both + MCP + Microsoft | 2025 | 10 |

Microsoft is pre-filtered upstream in Part 2 (see §5 "Microsoft cleanup cell"), so any bucket that includes `final_microsoft.csv` inherits that cleanup automatically. There are no bucket-level filter flags or filtered-variant output files.

**AEI v1 auto_aug imputation** (`_impute_aei_v1_auto_aug`):

AEI v1 ships with no auto/aug data (`auto_aug_mean` all null). Before combining, every build that includes `final_aei_v1.csv` imputes v1's `auto_aug_mean`:

- The imputation pool is the **other AEI sources in that same build** (`_aei_pool_frames`): conv v2–v5 always, plus API v3–v5 in API-containing buckets (`aei_all_usage`, `all_confirmed_usage`, `all_usage`). Microsoft and MCP never enter the pool. The pool grows as later AEI versions arrive. A build's first v1-containing version (2024-12-23) has an empty pool, so v1 stays null; imputation effectively begins at the 2025-03-06 version.
- Only **v1-exclusive** `(title, task_normalized)` pairs (absent from every pool frame) are imputed. Shared pairs stay null so the real pool score wins the downstream `max` combine; an imputed mean must never override an observed score.
- Fallback chain: pool mean `auto_aug_mean` over **DWA → IWA → GWA**. Runs in native v20.1 taxonomy space (before any 2010→2019 crosswalk), so DWA/IWA/GWA titles match the pool exactly; the imputed value then rides through the title-fix crosswalk into the 2025 backbone.
- v1-exclusive rows with **no DWA/IWA/GWA mapping at all** (O*NET task statements never linked to a work activity) are **dropped**, since there is nothing to borrow from.
- Imputed values are written straight into `auto_aug_mean` with no provenance flag. The per-version `final_aei_v1.csv` is left untouched; imputation happens only on the in-build copy.

**2015 task set builds** (`build_cumulative_2015`):
- Merge key: `["title", "task_normalized", "dwa_title", "iwa_title", "gwa_title"]`
- For matched rows: `auto_aug_mean` takes **max**, `pct_normalized` is **summed** across contributing source versions, `date` takes **latest**, other columns from latest version
- After aggregation, `pct_normalized` is **renormalized** so unique `(title, task_normalized)` pairs sum to 100

**2025 task set builds** (`build_cumulative_2025`):
- Extracts scores per unique `(title_current, task_normalized)` from each source
- AEI/API sources match their `title` (2010 SOC) against ECO 2025's `title_current` (2019 SOC); MCP/Microsoft match `title_current` directly. Most AEI pairs match on the first pass.
- **AEI title-fix recovery**: AEI/API rows that fail the first-pass match go through `_recover_aei_via_title_fix()`. The row's `soc_code_2010` is crosswalked (via `2010_to_2019_soc_crosswalk.csv`, looked up by 6-digit SOC prefix) to the set of ECO 2025 `title_current` values for the corresponding 2019 SOC(s). Targets are filtered to those where `(target_title, task_normalized)` is already a pair in ECO 2025, so no synthetic (title, task) pairs are injected into the backbone. If K targets survive, the row's `pct_normalized` is split evenly (`pct / K`); `auto_aug_mean` and `date` are copied. This recovers a meaningful share of AEI pairs and pct mass that the text-only first pass misses (most impactful for AEI API). Pairs whose task is not in ECO 2025, or whose 2010 SOC has no usable crosswalk, are dropped.
- Combines scores: **max** auto_aug, **sum** pct across sources, **latest** date
- After aggregation, `pct_normalized` is **renormalized** so unique `(title_current, task_normalized)` pairs sum to 100
- Joins combined scores to ECO 2025 backbone on `(title_current, task_normalized)` for structural columns (DWA/IWA/GWA, SOC 2019 codes, etc.). The inner join is preserved: every recovered row lands on a pair ECO 2025 already contains, so no backbone extension is needed

Total output: 44 cumulative CSV files across all buckets (42 from the main builder + 2 additive files from the supplementary cell that runs immediately after).

**Supplementary builds (latest date only, ECO 2025 backbone).** Two files built in a single cell directly after the main cumulative builder, reusing `build_cumulative_2025`, `eco_2025`, `cx_lookup`, `AEI_API`, `AEI_CONV` from that cell:

1. `final_aei_agentic_usage_2025_2026-02-12.csv` — AEI API v3+v4+v5
2. `final_aei_all_usage_2025_2026-02-12.csv` — AEI Conv v1–v5 + AEI API v3–v5 (v1's `auto_aug_mean` is imputed via `_impute_aei_v1_auto_aug` from the in-build AEI pool)

Both go through standard text-match + 2010-SOC title-fix recovery into the ECO 2025 task space. Picked up automatically by downstream Eloundu and column-reorder cells via the `final_*.csv` glob (filenames contain `_usage_`, so they get cumulative column-reorder treatment).

### Job Zones

Merges O*NET Job Zone data (`job_zones_v30.1.csv`, sourced from `Job Zones.xlsx`) into ECO 2025 only. Job Zones classify occupations into one of five zones (1--5) based on education, experience, and training requirements.

- Normalizes `title_current` for matching against Job Zone titles
- Deduplicates occupations before merging to get one `job_zone` value per occupation
- Merges back into the full ECO 2025 task-level DataFrame on `title_current`
- `job_zone` is stored as `Int64` (nullable integer)

### Employment Projections

Merges BLS Employment Projections 2024–2034 into ECO 2025 only. Adds one column: `emp_change_pct_2024_2034` (projected % change in employment over the decade).

- Input: `data/emp_projections.csv` (BLS Table 1.2, headers cleaned, employment counts in raw units, em-dashes → NaN, types cast)
- Merge is on **title** (not SOC), via `normalize_text()` on both sides
- Imputation fallback chain for occupations whose detailed title doesn't match BLS: `title_current` → `broad_occ` → `minor_occ_category` → `major_occ_category`. BLS publishes minor-group rollups (codes ending in `000`) but skips the SOC broad level, so `minor_occ_category` carries most of the fallback weight
- Most occupations match directly; the remainder are imputed via broad/minor, with none left unfilled

### DWS Ratings

Merges `dws_ratings.csv` into ECO 2025 (star ratings for tasks).

### Eloundu Task Ratings

Merges human and GPT-4 exposure labels (`E0`/`E1`/`E2`) from `full_labelset.tsv` (Eloundu et al.) into **all** final datasets. Two columns are added:

- `eloundu_human` — from `human_exposure_agg`
- `eloundu_gpt4` — from `gpt4_exposure` (the original rubric, not the alt-rubric variant)

The Eloundu file is on **2019 SOC** (despite the paper's vintage, its SOC6 codes include 2019-only codes and none that are 2010-only). Match strategy:

1. **Pair match** on `(normalize_text(title_match_col), task_normalized)`. The match column is `title_current` for 2019-SOC datasets (MCP, Microsoft, ECO 2025, 2025-task-set cumulative buckets) and `title` for 2015-SOC datasets (AEI v1–v5, AEI API v3–v5, ECO 2015, 2015-task-set cumulative buckets).
2. **Task-only fallback** on `task_normalized` for unmatched rows. The fallback value is the **mode** of the label across all occupations sharing that task; ties resolve to NaN.
3. Unmatched rows stay NaN (no occupation/DWA/IWA/GWA fallback for now).

Coverage:
- 2019-SOC datasets: nearly all rows labeled.
- 2015-SOC datasets: most rows labeled. Coverage is lower because the 2010 SOC title key matches fewer Eloundou rows; the task fallback recovers most of the rest.

Helper `mode_or_na` is defined inline in the cell. Pair lookup is built via `groupby(["title_norm", "task_normalized"]).agg(mode_or_na)`; task lookup is `groupby("task_normalized").agg(mode_or_na)`.

### Column Reorder

Final column ordering applied. The reorder cell pulls a per-dataset list of columns out of their current positions and inserts them as a contiguous block immediately after `physical`. Within that block the order is: cumulative score cols (where applicable) → ratings (`freq_mean`, `importance`, `relevance`) → ECO/MCP extras (`task_prop`, `dws_star_rating`, `job_zone`, `emp_change_pct_2024_2034`, `top_mcps`/`top_mcp_urls`) → **Eloundu cols** (`eloundu_human`, `eloundu_gpt4`). Eloundu sits at the tail of the block so it lands right before the national + state wage/employment columns.

Output: `third_pass_*.csv` -> copied to `final/final_*.csv`

---

## 7. Input Data Files

### Raw AI Data

| File Pattern | Source | Notes |
|-------------|--------|-------|
| `task_pct_v{1..5}.csv` | AEI conversation snapshots | v1--v2 use original format, v3+ use GLOBAL/onet_task format |
| `task_pct_api_v{3..5}.csv` | AEI API snapshots | Same format as v3+ conversation |
| `task_pct_eco_{2015,2025}.csv` | ECO baselines | Pre-built task percentages (all zeros for pct/auto_aug) |
| `task_results_{date}.csv` | MCP server pipeline | Dates: 2025-04-24, 2025-05-24, 2025-07-23, 2026-02-18 |
| `iwa_metrics.csv` | Microsoft Copilot analysis | IWA-level metrics |
| `automation_vs_augmentation_by_task_*.csv` | AEI auto/aug scores | Separate files for v2, v3, v4, v5, api_v3, api_v4, api_v5 |

### O*NET Reference

| File | Version | Used By |
|------|---------|---------|
| `task_statements_v20.1.csv` | Oct 2015 | AEI v1--v2, ECO 2015 |
| `task_statements_v30.1.csv` | May 2025 | MCP, Microsoft, ECO 2025 |
| `tasks_dwa_iwa_gwa_v20.1.csv` | Oct 2015 | AEI taxonomy |
| `tasks_dwa_iwa_gwa_v30.1.csv` | May 2025 | MCP/Microsoft/ECO taxonomy |
| `tasks_dwa_iwa_gwa_v30.1_physical.csv` | May 2025 | With physical flags merged |
| `task_ratings_v30.1.csv` | May 2025 | Frequency/importance/relevance |
| `physical_tasks.csv` | Microsoft | Physical task boolean flags |
| `dws_ratings.csv` | O*NET | Star ratings for ECO |
| `job_zones_v30.1.csv` | O*NET | Job Zone classifications (1--5) for ECO 2025 |
| `full_labelset.tsv` | Eloundu et al. | Human + GPT-4 exposure labels (E0/E1/E2) per task, 2019 SOC |

### Economic Data

| File | Description |
|------|-------------|
| `oews_national_2025.csv` | BLS OEWS national wages & employment, May 2025 |
| `oews_states_2025.csv` | BLS OEWS state-level wages & employment, May 2025 |
| `oews_national_2015.csv` | BLS OEWS national wages & employment, May 2015 |
| `oews_states_2015.csv` | BLS OEWS state-level wages & employment, May 2015 |
| `scraped_wage_data.csv` | O*NET scraped wages (Jan 2020), used as fallback |
| `soc_structure_2019.csv` | SOC hierarchy (major/minor/broad categories) |
| `2010_to_2019_soc_crosswalk.csv` | SOC code mapping between versions |
| `detailed_occ_2019.csv` | Detailed occupation listings (2019 SOC) |
| `emp_projections.csv` | BLS Employment Projections 2024–2034 (national, by occupation) |

---

## 8. Intermediate File Stages

| Stage | Pattern | When Created |
|-------|---------|-------------|
| First pass | `first_pass_*.csv` | After Part 1 (Pct, emp, wage) |
| Second pass | `second_pass_*.csv` | After Part 2 (taxonomy + auto/aug) |
| Third pass | `third_pass_*.csv` | After Part 3 (ratings) |
| Final | `final/final_*.csv` | After cumulative build, dws ratings, column reorder (16 per-version + 44 cumulative = 60) |

All intermediate files are saved to `data/` root. Final files go to `data/final/`.

---

## 9. Secondary Scripts

### `code_storage.ipynb`

Provenance scripts. Not part of the pipeline. Contains the one-off scripts that produced several committed inputs:

- Physical metric extraction: merges Microsoft physical flags into O*NET taxonomy file
- Collaboration pattern extraction: pivots AEI raw data into collaboration type columns (directive, feedback loop, learning, task iteration, validation)
- ECO percentage creation: builds `task_pct_eco_{2015,2025}.csv` from O*NET task statements
- Master title normalization: aggregates pct_normalized across AEI versions into `master_pct_normalized.csv`

---

## 10. Common Pitfalls

1. **O*NET version mismatch.** AEI uses v20.1 task statements (2015). MCP/Microsoft/ECO 2025 use v30.1 (2025). Tasks that exist in one version may not exist in the other. The pipeline handles this by using the correct version per dataset type.

2. **SOC code anomalies.** Several SOC codes don't follow standard hierarchy rules. Hard-coded fixes in `add_soc_structure()` handle these. If adding new data, check that SOC codes resolve correctly.

3. **Employment reallocation.** BLS reports employment at the 6-digit SOC level, but O*NET splits some occupations into decimal variants (e.g., 11-1011.00, 11-1011.03). Without the `adjust_emp` step, employment would be double/triple counted.

4. **AEI v3+ format change.** AEI v1--v2 provide task percentages directly. v3+ provide raw conversation data that needs filtering for `GLOBAL/onet_task` entries before percentage extraction. The `pct_to_onet_tasks()` function handles both formats.

5. **Cumulative pct re-normalization.** When building any cumulative dataset (2015 or 2025 task set), `pct_normalized` values are summed across all contributing source versions per merge key, then the whole dataset is rescaled so that unique `(occ, task_normalized)` pairs sum to 100. Taking max instead of summing will not sum to 100. Skipping the rescale after summing will also not sum to 100; both steps are required. The per-version final files already sum to 100 on their own.

6. **State wage fallback chain.** State OEWS data has no broad O-GROUP category, so the fallback chain is shorter than national: hourly/annual conversion -> national wages. Don't add a broad-category fallback for states.

7. **Inflation factors are hard-coded.** The 1.24x (Jan 2020) and 1.36x (May 2015) factors need manual updates if the base year changes.

8. **MCP rating winsorization.** MCP ratings are winsorized at 75th percentile + 1.5*IQR before mapping. This reduces the impact of extreme outlier ratings from the LLM classification pipeline.

9. **Microsoft IWA → (Title, task) distribution.** Microsoft data arrives at the IWA level. Distribute each IWA's share by dividing by the count of **unique (Title, task) pairs** under that IWA, not by the count of taxonomy rows (which double-counts any pair that has multiple DWAs). Then sum across IWAs per pair, rescale so unique pairs sum to 100, and let the subsequent taxonomy merge duplicate the per-pair pct across DWA/IWA/GWA rows. Any new source that distributes weight through an IWA-level key needs the same treatment.

10. **Microsoft taxonomy inner-merge.** The final taxonomy merge in the Microsoft cell is `how='inner'` on `norm_iwa_title`, and must stay inner. With `how='left'`, taxonomy rows for filtered-out IWAs (share < 0.0005) return with NaN `impact_scope_*`, and Part 2's `iwa_agg.apply(compute_stats, ...)` raises `ValueError: cannot convert float NaN to integer` when `find_matching_bin` hits `int(NaN / 0.05)`.

11. **Renormalize after any row-level filter on Microsoft.** Part 2's Microsoft cleanup cell drops rows for two reasons (physical-under-AI-dominant-IWA, Eloundou E0+E0). Survivors retain pre-drop `pct_normalized` magnitudes, so the unique-pair sum falls below 100. The cell rescales `pct *= 100 / current_unique_pair_sum` so the invariant holds going into cumulative builds. Any future row-level filter on Microsoft must do the same; `scripts/check_pct_normalized.ipynb` will catch a missed rescale.

12. **AEI title-fix recovery must not inject synthetic pairs.** `_recover_aei_via_title_fix()` only emits rows where `(crosswalked_title_current, task_normalized)` is already a pair in the ECO 2025 backbone, which keeps the final `eco_base.merge(..., how="inner")` an inner join. Loosening this filter to keep crosswalk targets whose pair does not yet exist in ECO 2025 would require either (a) extending the backbone in-memory with skeleton rows filled from task-level and occupation-level reference tables, or (b) switching the final merge to `left`/`outer` and supplying the structural columns. The additional pct mass recovered that way is negligible.

13. **AEI v1 has no auto/aug; cumulative builds impute it.** v1's `auto_aug_mean` is null in the per-version file. Cumulative builds fill v1-exclusive tasks via a DWA→IWA→GWA mean over the other AEI sources in the build (`_impute_aei_v1_auto_aug`). Two invariants must hold: (a) impute **only v1-exclusive** pairs, since imputing a shared task lets an imputed mean beat the real v2–v5 score in the `max` combine; (b) impute in **native v20.1 space**, before the title-fix crosswalk, so DWA/IWA/GWA strings match the pool (v20.1 and v30.1 DWA titles diverge). v1-exclusive tasks with no work-activity mapping are dropped.
