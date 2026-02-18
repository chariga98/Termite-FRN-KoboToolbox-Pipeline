# Termite FRN Data Pipeline

> **R data pipeline for ingesting, cleaning, and restructuring KoboToolbox termite field research submissions** — pulling multi-version forms from the KoboToolbox API, flattening nested repeat groups into analysis-ready wide datasets, deduplicating records, and exporting stage-specific Excel workbooks for the Long Rain and Short Rain 2025 seasons.

---

## Table of Contents

- [Overview](#overview)
- [Project Structure](#project-structure)
- [Pipeline Architecture](#pipeline-architecture)
- [Scripts](#scripts)
- [Data Sources (KoboToolbox Forms)](#data-sources-kobotoolbox-forms)
- [Output Datasets](#output-datasets)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Usage](#usage)
- [Repeat Group Reshaping Logic](#repeat-group-reshaping-logic)
- [Column Naming Convention](#column-naming-convention)

---

## Overview

Field data for the Termite FRN (Farmer Research Network) is collected via KoboToolbox mobile forms across three organizations — RFRN, Tembea, and SOFDI — using separate form versions. This pipeline automates the full journey from raw API submissions to clean, analysis-ready Excel files.

The core challenge is that KoboToolbox repeat groups (e.g., one row per plot, per termite location, per growth stage) come back from the API as nested list-columns. This pipeline flattens all repeat groups into a flat-wide format — one row per submission — where each plot's data occupies its own set of columns (`Plot_1_*`, `Plot_2_*`, etc.).

The final outputs feed directly into the **Termite FRN Dashboard** (`app.R`).

---

## Project Structure

```
Termite_FRN_Pipeline/
│
├── 01_kobo_fetch.R                  # KoboToolbox API fetch + flatten (all sources)
├── 02_clean_rfrn_tembea.R           # Clean & reshape RFRN/Tembea long rain data
├── 03_clean_sofdi.R                 # Fetch SOFDI long rain data
├── 04_merge_longrain.R              # Merge sources → Longrain_2025_master + stage splits
├── 05_shortrain_pipeline.R          # Short rain fetch, clean, split, and export
├── 06_dedup_check.R                 # Duplicate detection and flagging
├── 07_farmer_list.R                 # Consolidated farmer list generation + PDF export
│
└── outputs/
    ├── Longrains_Termite_FRN_Data_2025.xlsx    # Long rain workbook (6 sheets)
    ├── Shortrains_Termite_FRN_Data_2025.xlsx   # Short rain workbook (6 sheets + edit log)
    ├── Termite_Farmer_List_Shared.xlsx          # Farmer directory
    ├── Termite_Farmer_List_FullWidth.pdf        # Print-ready farmer list
    ├── duplicates_found.csv                     # Flagged duplicate records
    └── cleaned_no_duplicates.csv               # Deduplicated master dataset
```

---

## Pipeline Architecture

```
KoboToolbox API
    │
    ├── RFRN/Tembea Form ──┐
    ├── SOFDI Form         ├──► get_kobo_data_all_versions()
    └── Short Rain Form ───┘         │
                                     ▼
                              flatten_kobo()
                          [expand repeat groups → wide]
                                     │
                                     ▼
                         Normalize text identifiers
                         Merge helper columns
                         Assign round numbers (r1–r4)
                         Drop metadata columns
                                     │
                                     ▼
                    Reshape repeat groups → wide Plot_N_* columns
                    (emergence, vigour, weeding dates, pop reduction,
                     plot sizes, planting dates, termite locations)
                                     │
                                     ▼
                         Split by stage (filter on flag columns)
                    ┌────────────────┴────────────────┐
             Long Rain 2025                    Short Rain 2025
          Longrains_*.xlsx                  Shortrains_*.xlsx
         (6 stage sheets)                  (6 stage sheets
                                            + edit log + sheet protection)
```

---

## Scripts

| Script | Purpose |
|---|---|
| `01_kobo_fetch.R` | Authenticates with KoboToolbox API, fetches all form versions, expands nested repeat groups into wide indexed columns using `expand_repeat_column()` and `flatten_kobo()` |
| `02_clean_rfrn_tembea.R` | Normalizes text fields, merges helper columns (`other_*`), assigns round numbers (`r1`–`r4`), drops metadata, reshapes all repeat groups, reorders columns |
| `03_clean_sofdi.R` | Same fetch and flatten logic for the SOFDI form version |
| `04_merge_longrain.R` | Combines RFRN/Tembea and SOFDI into `Longrain_2025_master`; renames repeat columns to `Plot_N_*` convention; splits SOFDI November data into `SOFDI_shortrain`; filters and exports 6-sheet Excel workbook |
| `05_shortrain_pipeline.R` | Merges `ShortRain_2025` and `SOFDI_shortrain`; handles vigour repeat variants across form versions using `coalesce()`; exports password-protected workbook with edit log |
| `06_dedup_check.R` | Standardizes text; identifies potential duplicates on `farmer_name + round_number + datacollector`; flags exact full-row duplicates; exports clean and duplicate CSVs |
| `07_farmer_list.R` | Builds consolidated farmer directory from RFRN/Tembea and SOFDI data; exports Excel and paginated PDF via `gt` and `pagedown` |

---

## Data Sources (KoboToolbox Forms)

| Source Label | Organization | Form Name | KoboToolbox User |
|---|---|---|---|
| `RFRN_Tembea` | RFRN + Tembea | Termite Research (Tembea & RFRN Version) | `ae_hub_ea` |
| `SOFDI` | SOFDI | Termite Research | `sofdi` |
| `ShortRain_2025` | All | Termite Short Rain 2025 | `ae_hub_ea` |

>  **Credentials** are set at the top of each fetch script. Replace `kob_user` and `kob_pass` before running. Do not commit credentials to version control — use environment variables or a `.Renviron` file in production.

---

## Output Datasets

### Long Rain Workbook (`Longrains_Termite_FRN_Data_2025.xlsx`)

| Sheet | Filter condition | Key content |
|---|---|---|
| `Farmer_Profile` | `farmer_profile == "yes"` | Demographics, termite problem, contact details |
| `Trial_setup` | `farmer_profile == "yes"` | Varieties planted, plot sizes, planting dates, emergence |
| `First_Weeding` | `first_weed_data == "yes"` | Weeding dates, pop reduction, termite signs, vigour |
| `Second_Weeding` | `second_weeding_data == "yes"` | Weeding dates, pop reduction, termite signs, vigour |
| `Tasseling_Stage` | `tasseling_data == "yes"` | Pop reduction, termite locations, vigour |
| `Harvesting_Stage` | `data_harvest == "yes"` | Plant loss, inputs, stage most plants lost, vigour |

### Short Rain Workbook (`Shortrains_Termite_FRN_Data_2025.xlsx`)

Same 6-sheet structure as Long Rain, with an additional `EDIT_LOG` sheet. All data sheets are **password-protected** (`Shortrains_2025_Edit`) and the workbook is **locked** (`shortrains_2025`).

### Farmer List

- `Termite_Farmer_List_Shared.xlsx` — Farmer name, organization, subcounty, cluster; sorted alphabetically
- `Termite_Farmer_List_FullWidth.pdf` — Paginated A4 PDF (25 rows/page) generated via `gt` + `pagedown::chrome_print()`

### Deduplication Outputs

- `duplicates_found.csv` — Exact duplicate rows flagged for review
- `cleaned_no_duplicates.csv` — Master dataset with one copy of each unique record

---

## Prerequisites

- **R** ≥ 4.1.0
- **Google Chrome** (required by `pagedown::chrome_print()` for PDF export)
- Active KoboToolbox account with access to the relevant forms

---

## Installation

```r
install.packages(c(
  # API & data handling
  "httr", "jsonlite", "dplyr", "tidyr", "janitor", "purrr", "tibble",
  
  # Date handling
  "lubridate",
  
  # Excel output
  "openxlsx", "readxl",
  
  # String operations
  "stringr",
  
  # Farmer list PDF
  "gt", "htmltools", "pagedown"
))
```

---

## Usage

### Step 1 — Set credentials

At the top of each fetch script, replace:

```r
kob_user <- "your_username"
kob_pass <- "your_password"
```

Or load from environment:

```r
kob_user <- Sys.getenv("KOBO_USER")
kob_pass <- Sys.getenv("KOBO_PASS")
```

### Step 2 — Fetch raw data

```r
source("01_kobo_fetch.R")
# Produces: termite_data (RFRN/Tembea), Sofdi_termite_data, ShortRain_2025
```

### Step 3 — Clean and reshape

```r
source("02_clean_rfrn_tembea.R")   # → RFRN_Tembea_Cleaned_Termite_Data
source("03_clean_sofdi.R")          # → Sofdi_termite_data (cleaned)
```

### Step 4 — Build Long Rain master and export

```r
source("04_merge_longrain.R")
# Outputs: Longrains_Termite_FRN_Data_2025.xlsx
```

### Step 5 — Build Short Rain master and export

```r
source("05_shortrain_pipeline.R")
# Outputs: Shortrains_Termite_FRN_Data_2025.xlsx (password-protected)
```

### Step 6 — Deduplication check

```r
source("06_dedup_check.R")
# Outputs: duplicates_found.csv, cleaned_no_duplicates.csv
```

### Step 7 — Generate farmer list

```r
source("07_farmer_list.R")
# Outputs: Termite_Farmer_List_Shared.xlsx, Termite_Farmer_List_FullWidth.pdf
```

---

## Repeat Group Reshaping Logic

KoboToolbox repeat groups arrive from the API as list-columns where each cell contains a nested data frame or list. The pipeline handles this in two steps:

**`expand_repeat_column(col_values, colname)`** — Takes one list-column and expands it into multiple flat columns indexed by repeat number and inner field name. For example, a column `vigour_repeat` with up to 4 repeats and fields `name` and `for_each_plot` becomes:

```
vigour_repeat_1_name | vigour_repeat_1_for_each_plot
vigour_repeat_2_name | vigour_repeat_2_for_each_plot
...
```

**`flatten_kobo(dat)`** — Loops over all list-columns in a tibble, calling `expand_repeat_column()` for complex repeats and scalar simplification for single-value list columns, until no list-columns remain. Names are cleaned with `janitor::clean_names()`.

---

## Column Naming Convention

After flattening and renaming, all plot-level columns follow a consistent pattern:

| Pattern | Example | Meaning |
|---|---|---|
| `Plot_N_name_'Metric'` | `Plot_1_name_'Vigour'` | Plot name label for that metric's repeat |
| `Plot_N_Metric` | `Plot_1_Vigour` | Observed value for plot N |
| `Plot_N_Population.Reduction.Stage` | `Plot_2_Population.Reduction.First.Weeding` | Population reduction at a given stage |
| `Plot_N_First.weeding.Date` | `Plot_3_First.weeding.Date` | Per-plot weeding date |
| `Location_N_Termite_attacked` | `Location_1_Termite_attacked` | Location where termites attacked |
| `Location_N_Termite_Name` | `Location_1_Termite_Name` | Termite species at that location |

This naming convention is what `tidy_plots()` in `app.R` uses to parse and pivot plot data back into long format for visualization.

---

*Source: KoboToolbox Termite FRN Field Data — Long Rain & Short Rain 2025*
