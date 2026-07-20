# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

This repo recodes survey data from the **Encuesta CONCLAT** (FONDECYT Regular No. 1230056), a comparative Chile/Argentina survey on class, unions, and labor/political conflict. Output is a Quarto book (`_quarto.yml`, `project: type: book`) with one chapter per country, meant to become book chapters.

## Commands

- Render the whole book: `quarto render` (output goes to `docs/`, per `_quarto.yml`).
- Render a single chapter interactively: open the `.qmd` in RStudio and use "Render", or `quarto render 01-recodificaciones-conclat-cl.qmd`.
- There is no test suite, linter, or package manager config (no `renv`, no `DESCRIPTION`). Dependencies are declared inline per chapter via `pacman::p_load(...)` at the top of each `.qmd`.
- To iterate on a single recoded variable, run chunks interactively in RStudio (`conclat_reco.Rproj`) rather than rendering the whole document — chapters are long and rendering rebuilds every chunk.

## Architecture

- **`01-recodificaciones-conclat-cl.qmd`** — the active, country-specific recoding chapter for Chile. This is the one wired into `_quarto.yml` and the one currently maintained.
- **`01-recodificaciones-conclat.qmd`** — a generic CL/AR version of the same recoding logic, parameterized on a `pais` variable (`"cl"` / `"ar"`). Where coding differs by country (`a07`, `b14`, `d01`, `d03`, `c09`, and the multi-response condition in `b24`), it has `if (pais == "cl") {...} else {...}` branches. This is not yet wired into `_quarto.yml` (the Argentina chapter is commented out in the book's `chapters:` list) — treat it as the template to extend when adding the Argentina chapter.
- Keep both files in sync when fixing a recoding bug that isn't country-specific — the logic is currently duplicated between them.

### Recoding pattern used throughout

Each `.qmd` chapter follows the same structure top to bottom:

1. **Load libraries** (`pacman::p_load`), **load raw data** from `input/data/data-conclat-{cl,ar}-*-clases.rds`, define shared **helpers**:
   - `rec_escala(x, missing = c(-888, -999))` — cleans numeric scale items, mapping missing/NS-NR sentinel codes to `NA` while preserving the original scale.
   - `respondio_multi(df, prefijo, n)` — for dichotomized multi-response batteries (`{prefijo}_cat1`...`{prefijo}_catN`, 1 = yes), returns `TRUE` if the respondent selected at least one option. Used to distinguish "answered but chose none of the grouped options" (→ a residual/"Other" category) from "skip pattern, never saw the question" (→ `NA`). Prefer `%in% 1` over `== 1` when checking these `_cat` columns, since `NA | FALSE` under `==` propagates `NA` and breaks category assignment.
2. **Recode variables section by section**, organized under the survey's own chapter headings (Cap. 2–5: union membership/motives, class identification, workplace conflict/mobilization, political participation/identification). Each variable block follows: `frq(data$raw_var)` to inspect raw codes → `mutate`/`case_when` to build the recoded variable (often both a dummy and an ordered/numeric version) → `set_variable_labels()` / `set_value_labels()` (from `labelled`/`sjlabelled`) → `frq()` again to verify the result.
3. Multi-group categorical recodes (e.g. `b23_why_member`, `b24_why_not_member`, `b14_causes`, `b15_measures`) build temporary boolean flag columns prefixed with `.` (e.g. `.econ`, `.wage`), combine them in a `case_when` with an explicit priority order, then `select(-starts_with("."))` to drop the scratch columns.
- **Social class schemes** (`owners`, `owners2`, `skills`, `organization_assets` and final class variables) are NOT built in these chapters — they come from an earlier, separate processing step (`01-proce-data-conclat-cl.qmd`, not currently in this repo) and are just expected to be present on the incoming `.rds`.
- **Saving output is currently commented out** (`saveRDS(...)` at the end of each chapter) — recoded data isn't being persisted yet.

### Known open decisions (in-code notes, not yet resolved)

Several chunks contain a Spanish-language `> Nota:` callout flagging a recoding decision that's provisional pending manual review — mostly cases needing free-text (`_esp`) coding of "Otra/Otro" responses into existing categories (e.g. `d03_esp`, `c09_esp`), or ambiguous priority rules for multi-response batteries. When touching these variables, check for an existing note above/below the chunk before changing the logic — the ambiguity is usually already documented there.

### Data files

- `input/data/data-conclat-{ar-es,cl-en}-clases.rds` — raw input, already has class-scheme variables attached.
- `input/codebook_conclat_{ar_ES,cl_ES,cl_EN}.xlsx` — codebooks for raw variable codes; consult these instead of guessing what a numeric code means.
- `input/CONCLAT_Recodificaciones.docx` — the source recoding specification the `.qmd` chapters implement; when a recoding rule is unclear in code, check this doc.
