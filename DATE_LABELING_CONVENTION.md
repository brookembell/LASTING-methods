# Date-Labeling Convention

This document defines how dates are attached to file names in the LASTING
methods-manual pipeline. It applies to every `.Rmd` chapter and the `.R`
files generated from them.

## The standard

**Format: ISO 8601 — `YYYY-MM-DD`.** For example, `2026-06-21`.

Use this format for every date that appears in a file name. It is
unambiguous, it sorts chronologically (a plain alphabetical sort of the
folder puts the files in date order), and it matches R's default
`Sys.Date()` output, so no reformatting is ever required.

Do **not** use any of the older formats that had accumulated in the
project:

| Old style | Example | Problem |
|-----------|---------|---------|
| `MMDDYY` | `population_subgroups_060923` | Ambiguous; sorts wrong |
| `MM-DD-YY` | `Food_to_FNDDS_mapping_04-06-25` | Ambiguous; sorts wrong |
| `MM-DD-YYYY` | `..._02-03-2025` | Sorts wrong (month first) |
| `M.D.YY` | `Unit_conversions_1.4.24` | Ambiguous; sorts wrong |
| `MMYY` | `FCID_0118` | Missing day; ambiguous |

## Two kinds of dated files

**1. Generated pipeline files** — anything the cleaning / finalize / model
chapters (02–08) *write* and later *read back*: `population_distribution`,
`unit_conversions`, `input_data`, `Impacts_cost`, `rr_agesexrace`, and so
on. These are the files this convention controls directly.

**2. Raw / source input files** — files received from outside the pipeline
(NHANES pulls, mapping tables, effect-size files, etc.) that already carry
a date in their name. Their existing names are left as-is so we don't break
references or lose provenance. **When you add or re-receive a raw file,
name it with the ISO 8601 date** (e.g. `RR_BMI_cvd_2023-03-02.csv`).

## How generated files are dated

There is a single source of truth: **`code/version_date.R`**.

```r
# code/version_date.R
version_date <- as.Date("2026-06-21")          # <- the one place to change

dated <- function(stem, ext = ".csv") {         # builds every dated name
  paste0(stem, "_", format(version_date, "%Y-%m-%d"), ext)
}
```

Every pipeline chapter loads it in its setup chunk (after `rm(list = ls())`):

```r
source("code/version_date.R")
```

and builds every dated file name with the `dated()` helper:

```r
write_csv(pop1, dated("data_inputs/FINAL/cleaned_raw_data/population_distribution"))
# -> "data_inputs/FINAL/cleaned_raw_data/population_distribution_2026-06-21.csv"

read_csv(dated(paste0(file_location, "input_data")))
# -> "data_inputs/FINAL/model_data/input_data_2026-06-21.csv"

saveRDS(x, dated("data_inputs/IMPACT_FACTORS/temp_data/enviro_input", ".rds"))
# -> "..._2026-06-21.rds"
```

The helper keeps the separator (`_`), the date format, and the extension
defined in exactly one place, so no file name can drift out of the
convention.

### Why centralize it

The chapters hand data off to each other: 02–07 **write** dated files that
06–08 later **read**. Previously each chapter stamped its own `Sys.Date()`
(and used three different variable names — `my_date`, `date`,
`version.date` — plus a hand-maintained hard-coded string in chapter 08).
If you ran the writer on one day and the reader on the next, the reader
looked for a date that was never written and the pipeline failed. Chapter
08 even carried a comment acknowledging this: *"set manually in case the
day you calculated the input files is not today."*

Sourcing one `version_date` everywhere makes writers and readers always
agree and makes a full run reproducible.

## How to cut a new data version

1. Open `code/version_date.R` and set `version_date` to the date you are
   (re)generating the inputs — a fixed date for a reproducible version, or
   `Sys.Date()` to stamp with today.
2. Re-run chapters 02–07 so all generated files are written with the new
   stamp.
3. Downstream chapters (08) need no edits — they read the same value.
4. Re-run chapter 09 (`purl`) to regenerate the `code/*.R` scripts used on
   the cluster.

## Notes / exceptions

- **Model output directory** (`outputs/model/output_<date>/` in chapter 08)
  intentionally keeps `Sys.Date()`. That folder marks *when the model was
  run*, which can legitimately differ from the input version — you may run
  the model many times against one set of inputs.
- **`index.Rmd`** uses `` `r Sys.Date()` `` for the manual's rendered date,
  which is correct — it reflects the render, not a data version.
- The `code/*.R` files are generated from the `.Rmd` chapters by chapter 09
  (`knitr::purl`). Always edit the `.Rmd`, then regenerate.
