# AI Workforce Exposure — Dataset Construction

Data-construction pipeline for the paper *Mapping AI Exposure Across the U.S. Workforce: Evidence from Millions of AI Conversations*.

It takes raw data from multiple AI scoring sources (Anthropic Economic Index, MCP server pipeline, Microsoft Copilot), O\*NET occupational data, and BLS employment and wage statistics, and merges them into a unified format where each row is one task within one occupation.

## Links

<!-- TODO: fill in Paper, this-repo, and Dashboard URLs when available -->
- Paper: TBD
- Dashboard: TBD
- Main paper and dashboard repository: TBD
- Dataset-construction repository (this repo): https://github.com/theodorewright11/ai-workforce-exposure-dataset-construction-public
- MCP → O\*NET classification repository: https://github.com/theodorewright11/mcp-onet-task-classification-public
- Final datasets (HuggingFace): https://huggingface.co/datasets/theodorewright11/ai-workforce-exposure-datasets-public

---

All raw input data is included in this repository except for one file that exceeds GitHub's 100MB file size limit: **`mcp_results_2026-02-18.csv`** (186MB). Download it from Hugging Face and place it in the `data/` directory before running the pipeline:

https://huggingface.co/datasets/theodorewright11/mcp-onet-task-classification-public

---

## Documentation

This project has detailed documentation split across two files:

- **[PRD.md](PRD.md)** -- What the pipeline produces and why. Covers data sources, output datasets, shared columns, pipeline workflow, and dataset dates.
- **[ARCHITECTURE.md](ARCHITECTURE.md)** -- How the code works. Reference for the three pipeline parts, input files, processing logic, and construction details (Microsoft IWA distribution, AEI O\*NET-version crosswalk, AEI v1 auto/aug imputation), with a Common Pitfalls section.

---

## Project Structure

```
ai-workforce-exposure-dataset-construction-public/
├── README.md                          # This file
├── PRD.md                             # What the pipeline produces
├── ARCHITECTURE.md                    # How the code works
├── requirements.txt                   # Python dependencies
├── .gitignore                         # Commits raw inputs only; ignores generated files
├── .gitattributes                     # Keeps committed data files byte-exact across platforms
├── scripts/
│   ├── data_merge.ipynb               # Main pipeline (3 parts)
│   └── code_storage.ipynb             # Provenance scripts for committed inputs (not part of the run)
└── data/                              # Raw inputs committed; intermediates + final/ are generated
    ├── final/                         # Final output datasets (generated, gitignored)
    └── merged_data_files/             # Intermediate saves (generated, gitignored)
```

---

## Data Sources

External inputs and where they come from. **[ARCHITECTURE.md §7](ARCHITECTURE.md#7-input-data-files)** has the full per-file detail (versions, which datasets use each file). Files the pipeline builds internally from these — derived AEI percentages, auto-aug scores, physical-merged taxonomy, ECO baselines, master pct — are not listed; they are produced by `scripts/code_storage.ipynb` and the main pipeline.

### Anthropic Economic Index (AEI)

| File | What it is | Link |
|------|------------|------|
| `task_pct_v1.csv` | AEI conversation v1 (released as processed percentages) | https://huggingface.co/datasets/Anthropic/EconomicIndex/blob/main/release_2025_02_10/onet_task_mappings.csv |
| `task_pct_v2.csv` | AEI conversation v2 (released as processed percentages) | https://huggingface.co/datasets/Anthropic/EconomicIndex/blob/main/release_2025_03_27/task_pct_v2.csv |
| `automation_vs_augmentation_by_task_v2.csv` | AEI conv. v2 collaboration patterns | https://huggingface.co/datasets/Anthropic/EconomicIndex/blob/main/release_2025_03_27/automation_vs_augmentation_by_task.csv |
| `aei_raw_claude_ai_2025-08-04_to_2025-08-11.csv` | AEI conversation v3 raw snapshot | https://huggingface.co/datasets/Anthropic/EconomicIndex/blob/main/release_2025_09_15/data/intermediate/aei_raw_claude_ai_2025-08-04_to_2025-08-11.csv |
| `aei_raw_claude_ai_2025-11-13_to_2025-11-20.csv` | AEI conversation v4 raw snapshot | https://huggingface.co/datasets/Anthropic/EconomicIndex/blob/main/release_2026_01_15/data/intermediate/aei_raw_claude_ai_2025-11-13_to_2025-11-20.csv |
| `aei_raw_claude_ai_2026-02-05_to_2026-02-12.csv` | AEI conversation v5 raw snapshot | https://huggingface.co/datasets/Anthropic/EconomicIndex/blob/main/release_2026_03_24/data/aei_raw_claude_ai_2026-02-05_to_2026-02-12.csv |
| `aei_raw_1p_api_2025-08-04_to_2025-08-11.csv` | AEI API v3 raw snapshot | https://huggingface.co/datasets/Anthropic/EconomicIndex/blob/main/release_2025_09_15/data/intermediate/aei_raw_1p_api_2025-08-04_to_2025-08-11.csv |
| `aei_raw_1p_api_2025-11-13_to_2025-11-20.csv` | AEI API v4 raw snapshot | https://huggingface.co/datasets/Anthropic/EconomicIndex/blob/main/release_2026_01_15/data/intermediate/aei_raw_1p_api_2025-11-13_to_2025-11-20.csv |
| `aei_raw_1p_api_2026-02-05_to_2026-02-12.csv` | AEI API v5 raw snapshot | https://huggingface.co/datasets/Anthropic/EconomicIndex/blob/main/release_2026_03_24/data/aei_raw_1p_api_2026-02-05_to_2026-02-12.csv |

### Microsoft Copilot

| File | What it is | Link |
|------|------------|------|
| `iwa_metrics.csv` | IWA-level user/AI usage + scope metrics | https://github.com/microsoft/working-with-ai/blob/main/iwa_metrics.csv |
| `physical_tasks.csv` | Physical/non-physical task flags | https://github.com/microsoft/working-with-ai/blob/main/physical_tasks.csv |

### MCP → O\*NET classification (our separate project)

| File | What it is | Link |
|------|------------|------|
| `task_results_2025-04-24.csv` | MCP automation scores, v1 | https://huggingface.co/datasets/theodorewright11/mcp-onet-task-classification-public/blob/main/task_results_2025-04-24.csv |
| `task_results_2025-05-24.csv` | MCP automation scores, v2 | https://huggingface.co/datasets/theodorewright11/mcp-onet-task-classification-public/blob/main/task_results_2025-05-24.csv |
| `task_results_2025-07-23.csv` | MCP automation scores, v3 | https://huggingface.co/datasets/theodorewright11/mcp-onet-task-classification-public/blob/main/task_results_2025-07-23.csv |
| `task_results_2026-02-18.csv` | MCP automation scores, v4 | https://huggingface.co/datasets/theodorewright11/mcp-onet-task-classification-public/blob/main/task_results_2026-02-18.csv |
| `mcp_results_2026-02-18.csv` (not committed; 186 MB) | MCP raw per-server results | https://huggingface.co/datasets/theodorewright11/mcp-onet-task-classification-public/blob/main/mcp_results_2026-02-18.csv |

### O\*NET

| File | What it is | Link |
|------|------------|------|
| `task_statements_v30.1.csv` | Task statements (May 2025) | https://www.onetcenter.org/dictionary/30.1/excel/task_statements.html |
| `task_statements_v29.0.csv` | Task statements (earlier vintage) | https://www.onetcenter.org/dictionary/29.0/excel/task_statements.html |
| `task_statements_v20.1.csv` | Task statements (Oct 2015) | https://www.onetcenter.org/dictionary/20.1/excel/task_statements.html |
| `tasks_dwa_iwa_gwa_v30.1.csv` | Work-activity taxonomy, built from O\*NET Tasks→DWAs + DWA reference (v30.1) | https://www.onetcenter.org/dictionary/30.1/excel/tasks_to_dwas.html<br>https://www.onetcenter.org/dictionary/30.1/excel/dwa_reference.html |
| `tasks_dwa_iwa_gwa_v20.1.csv` | Work-activity taxonomy, built from O\*NET Tasks→DWAs + DWA reference (v20.1) | https://www.onetcenter.org/dictionary/20.1/excel/tasks_to_dwas.html<br>https://www.onetcenter.org/dictionary/20.1/excel/dwa_reference.html |
| `task_ratings_v30.1.csv` | Task freq/importance/relevance (v30.1) | https://www.onetcenter.org/dictionary/30.1/excel/task_ratings.html |
| `task_ratings_may_2025.csv` | Task ratings (May 2025 vintage) | https://www.onetcenter.org/dictionary/29.3/excel/task_ratings.html |
| `task_ratings_oct_2015.csv` | Task ratings (Oct 2015 vintage) | https://www.onetcenter.org/dictionary/20.1/excel/task_ratings.html |
| `job_zones_v30.1.csv` | Job Zones (1–5) | https://www.onetcenter.org/dictionary/30.1/excel/job_zones.html |
| `scraped_wage_data.csv` | O\*NET scraped wages (Jan 2020 fallback) | https://github.com/adamkq/onet-dataviz |

### BLS / SOC

| File | What it is | Link |
|------|------------|------|
| `oews_national_2025.csv` | OEWS national wages/employment (May 2025) | https://www.bls.gov/oes/tables.htm |
| `oews_states_2025.csv` | OEWS state wages/employment (May 2025) | https://www.bls.gov/oes/tables.htm |
| `oews_national_2015.csv` | OEWS national wages/employment (May 2015) | https://www.bls.gov/oes/tables.htm |
| `oews_states_2015.csv` | OEWS state wages/employment (May 2015) | https://www.bls.gov/oes/tables.htm |
| `emp_projections.csv` | Employment projections 2024–2034 (Table 1.10) | https://www.bls.gov/emp/data/occupational-data.htm |
| `soc_structure_2019.csv` | SOC major/minor/broad hierarchy (2018 SOC Structure on the page) | https://www.bls.gov/soc/2018/home.htm |
| `2010_to_2019_soc_crosswalk.csv` | 2010 → 2019 SOC mapping | https://www.onetcenter.org/taxonomy/2019/walk.html |
| `detailed_occ_2019.csv` | Detailed occupation listings (2019 SOC) | https://www.onetcenter.org/taxonomy/2019/list.html |

### Other

| File | What it is | Link |
|------|------------|------|
| `dws_ratings.csv` | Utah DWS star ratings (use search and download button appears) | https://jobs.utah.gov/utwid/almiswage/all-occupations-search |
| `full_labelset.tsv` | Eloundou et al. human + GPT-4 exposure labels | https://github.com/openai/GPTs-are-GPTs/blob/main/data/full_labelset.tsv |

The O\*NET technology-skills, skills, knowledge, and abilities files are **not** inputs to this pipeline (they feed the downstream analysis), so they are not listed here.

---

## Output Datasets

Running the pipeline writes 16 per-version datasets and 44 cumulative datasets (60 total) to `data/final/`. Each row is one task within one occupation.

`data/final/` is generated output and is not committed to this repo. The finished datasets are hosted on HuggingFace:

https://huggingface.co/datasets/theodorewright11/ai-workforce-exposure-datasets-public

### Per-Version (16 files)

| Dataset | File |
|---------|------|
| AEI Conv. v1--v5 | `final_aei_v{1..5}.csv` |
| AEI API v3--v5 | `final_aei_api_v{3..5}.csv` |
| MCP Cumul. v1--v4 | `final_mcp_v{1..4}.csv` |
| Microsoft | `final_microsoft.csv` |
| ECO 2025 | `final_eco_2025.csv` |
| ECO 2015 | `final_eco_2015.csv` |

The 16th per-version file is `final_eco_2025_with_task_properties.csv` (ECO 2025 with extra task-property columns).

### Cumulative (44 files)

Cumulative datasets combine per-version datasets across sources. For each bucket, a new cumulative version is produced each time a new dataset arrives chronologically. Combining logic for overlapping (occupation, task) pairs: `auto_aug_mean` takes the **max** across sources; `pct_normalized` is **summed** across sources, then the whole dataset is **renormalized** so unique (occupation, task) pairs sum to 100. (The per-version files already sum to 100 on their own.) See [ARCHITECTURE.md](ARCHITECTURE.md) §6 and Pitfall #5 for why both the sum and the rescale are required.

Output naming: `final_{bucket_name}_{end_date}.csv`

| Bucket | Description | Sources | Task Set | Versions |
|--------|-------------|---------|----------|----------|
| `all_confirmed_usage` | All confirmed usage | AEI Both + Microsoft | 2025 | 6 |
| `confirmed_human_usage` | Confirmed human usage | AEI Conv + Microsoft | 2025 | 6 |
| `aei_all_usage` | AEI all confirmed usage | AEI Conv + AEI API | 2015 | 5 |
| `aei_human_usage` | AEI confirmed human usage | AEI Conv only | 2015 | 5 |
| `aei_agentic_usage` | AEI confirmed agentic usage | AEI API only | 2015 | 3 |
| `all_agentic_usage` | All possible agentic usage | MCP + AEI API | 2025 | 7 |
| `aei_agentic_usage_2025` | AEI confirmed agentic, ECO 2025 backbone | AEI API only | 2025 | 1 |
| `aei_all_usage_2025` | AEI confirmed all usage, ECO 2025 backbone | AEI Conv + AEI API | 2025 | 1 |
| `all_usage` | All usage potential | AEI Both + MCP + Microsoft | 2025 | 10 |

2025 task set buckets use ECO 2025 as structural backbone (DWA/IWA/GWA, `title_current`, SOC 2019 codes). AEI sources match their `title` (2010 SOC) against ECO 2025's `title_current` (2019 SOC). 2015 task set buckets use native AEI row structure.

---

## Key Output Columns

Every final dataset includes these columns (some may be null depending on dataset type). There are additional columns not listed here -- see [PRD.md](PRD.md) for the complete specification.

| Column | Description |
|--------|-------------|
| `task` | Raw O\*NET task text |
| `task_normalized` | Normalized task text (lowercased, punctuation removed) |
| `title` | Occupation title (2010 SOC for AEI/ECO 2015) |
| `title_current` | Occupation title (2019 SOC, for MCP/Microsoft/ECO 2025) |
| `soc_code_2010` | O\*NET SOC code (2010 system) |
| `dwa_title`, `iwa_title`, `gwa_title` | Work activity hierarchy (Detailed, Intermediate, General) |
| `broad_occ`, `minor_occ_category`, `major_occ_category` | SOC occupation hierarchy |
| `pct_normalized` | Share of AI usage for this task (sums to ~100 per dataset on unique title x task pairs) |
| `auto_aug_mean` | Automatability score (0--5 scale) |
| `physical` | Boolean flag for physical tasks |
| `freq_mean` | Task frequency (daily occurrence rate from O\*NET survey) |
| `importance` | Task importance (1--5 from O\*NET survey) |
| `relevance` | Task relevance (0--100 from O\*NET survey) |
| `emp_tot_nat_2025` | National employment (BLS OEWS 2025) |
| `a_med_nat_2025` | National median annual wage (BLS OEWS 2025) |
| `job_zone` | O\*NET Job Zone (1--5), ECO 2025 only |
| `date` | Dataset snapshot date |

---

## Pipeline Workflow

The pipeline runs in three parts from a single notebook (`scripts/data_merge.ipynb`):

1. **Part 1** -- Runs once per dataset (set `run_name`). Maps raw AI data to O\*NET tasks, adds SOC structure, merges BLS wage/employment, and adjusts employment for decimal SOC codes.
2. **Part 2** -- Runs once across all datasets. Adds snapshot dates, O\*NET taxonomy (DWA/IWA/GWA), physical task flags, and standardized auto\_aug scores.
3. **Part 3** -- Runs once across all datasets. Merges task ratings, adds DWS outlook numbers, adds Job Zones (ECO 2025), builds cumulative datasets, and does final column reordering.

---

## How to Reproduce

The pipeline is a single notebook, `scripts/data_merge.ipynb`, run in three parts. Part 1 is run once per input dataset; Parts 2 and 3 are each run once over all datasets. The notebook writes intermediate CSVs to `data/` and `data/merged_data_files/` between parts.

**0. Setup.**
- `pip install -r requirements.txt`
- Download the one large input not in this repo (186 MB, over GitHub's per-file limit) and place it in `data/`:
  `mcp_results_2026-02-18.csv` from <https://huggingface.co/datasets/theodorewright11/mcp-onet-task-classification-public>
- The empty `data/final/` and `data/merged_data_files/` directories are tracked (via `.gitkeep`) so the notebook can write into them on a fresh clone.

**1. Part 1 — run once per dataset.** Set the `run_name` variable at the top of Part 1 to each value below in turn and execute Part 1's cells, producing one `first_pass_*.csv` each time:

```
first_pass_aei_v1.csv          first_pass_aei_api_v3.csv      first_pass_mcp_v1.csv
first_pass_aei_v2.csv          first_pass_aei_api_v4.csv      first_pass_mcp_v2.csv
first_pass_aei_v3.csv          first_pass_aei_api_v5.csv      first_pass_mcp_v3.csv
first_pass_aei_v4.csv          first_pass_microsoft.csv       first_pass_mcp_v4.csv
first_pass_aei_v5.csv          first_pass_eco_tasks_2015.csv
                               first_pass_eco_tasks_2025.csv
```

`master_pct_normalized.csv` is consumed by Part 1 (employment reallocation) but was itself built from Part 1 outputs (see `scripts/code_storage.ipynb`). It is committed as a fixed artifact to break that circular dependency. Use the committed copy; do not regenerate it.

**2. Part 2 — run once.** Loads all `first_pass_*.csv`, adds dates / O\*NET taxonomy / physical flags / standardized auto-aug, applies the Microsoft cleanup filters, and writes `second_pass_*.csv`.

**3. Part 3 — run once.** Loads all `second_pass_*.csv`, merges task ratings, builds the cumulative datasets, attaches ECO-only extras (Job Zones, projections, DWS, Eloundou labels), reorders columns, and writes the final CSVs to `data/final/`.

Intermediate files (`first_pass_*`, `second_pass_*`, `third_pass_*`) and everything in `data/final/` are generated by this process and are gitignored. Only the raw inputs are committed.
