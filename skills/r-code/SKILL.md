---
name: r-code
description: Write idiomatic, production-quality R following tidyverse conventions, with Shiny, Quarto, and package development patterns
license: MIT
compatibility: opencode
metadata:
  language: R 4.1+
---

## What I do

Guide the writing of idiomatic, readable, and maintainable R code — from data analysis scripts to Shiny applications, Quarto documents, and R packages — including conventions, patterns, and common pitfalls to avoid.

## When to use me

Load this skill at step 1 (Write) of the agent workflow whenever the target language is R. Also useful during step 5 (Implement improvements) when adding or refactoring R code.

---

## Environment & tooling assumptions

Unless the project specifies otherwise, assume:
- R 4.1+ (native pipe `|>` available in 4.1; use it over `%>%` for new code unless project already uses `magrittr`)
- Package management: `renv` for reproducible environments
- Style: [tidyverse style guide](https://style.tidyverse.org/) enforced by `styler`
- Linting: `lintr`
- Testing: `testthat` (≥ 3rd edition)
- Documentation: `roxygen2`

If the project has `.lintr`, `renv.lock`, or `DESCRIPTION`, read them first and follow whatever they define.

---

## Style & conventions

### Naming
- `snake_case` for variables, functions, and file names.
- `PascalCase` for R6 class names and S3/S4 class names.
- `UPPER_SNAKE_CASE` for constants.
- Prefix internal/unexported functions with `.` (e.g., `.compute_ratio()`).

### Syntax
- Use the native pipe `|>` (R 4.1+) over `magrittr::%>%` for new code.
- Prefer `\(x)` lambda syntax (R 4.1+) over `function(x)` for short anonymous functions.
- Use `<-` for assignment, not `=` (except inside function arguments).
- Use `TRUE`/`FALSE`, not `T`/`F` (can be overwritten).
- One statement per line; max ~80–100 characters per line.

### Functions
- One function = one responsibility.
- Default arguments should be simple scalars or `NULL`; compute complex defaults inside the function body.
- Validate inputs early with `stopifnot()` or `rlang::arg_match()` / `checkmate` assertions.
- Return values explicitly; avoid relying on implicit last-expression return in complex functions.
- Use `invisible()` for functions called for side effects (like print methods).

### Packages & namespace
- Always use `::` for non-base functions in package code: `dplyr::filter()`, not `filter()`.
- In scripts and Shiny apps, `library()` calls at the top of the file are acceptable; in package code use `::` exclusively.
- Never use `library()` or `require()` inside a function.

---

## Common patterns

### Data manipulation (tidyverse / data.table)
- Use `dplyr` + `tidyr` for readable transformations; `data.table` for large datasets (> millions of rows).
- Chain with `|>` into a single pipeline; assign only at meaningful checkpoints.
- Prefer `across()` for column-wise operations over repeated `mutate()` calls.
- Use `readr` for CSV; `arrow` for Parquet/large files; `haven` for SAS/SPSS/Stata.

### Shiny applications
- Separate UI and server into `ui.R` / `server.R` (or use modules for larger apps).
- Use Shiny modules (`moduleUI()` / `moduleServer()`) for any reusable component.
- Reactive graph discipline:
  - `reactive()` for derived values; `observe()` / `observeEvent()` for side effects.
  - Never put expensive computation inside `render*()` — delegate to `reactive()`.
  - Use `req()` to guard against `NULL` inputs before computing.
- For accessibility in Shiny:
  - All `*Input()` functions produce labeled inputs by default — always supply the `label` argument.
  - Avoid `tags$div(style="color:red")` for errors; use `shinyFeedback` or `validate()`/`need()` which produce accessible inline errors.
  - Use `bslib` for theming to inherit accessible USWDS-compatible defaults.
  - Test tab-key navigation through the app manually.

### Quarto / R Markdown
- Set `fig-alt` on all code-generated figures: `` #| fig-alt: "Description" ``.
- Use `knitr::kable()` or `gt` for tables, not raw HTML.
- Parameterized reports: define params in YAML front matter; reference with `params$name`.

### Package development
- Follow `devtools` workflow: `load_all()`, `check()`, `test()`, `document()`.
- Document every exported function with `roxygen2`: `@param`, `@return`, `@examples`.
- Use `@importFrom pkg fun` instead of `@import pkg`.
- Write tests in `tests/testthat/test-<topic>.R`.

---

## Testing conventions (testthat 3e)

- Enable 3rd edition in `DESCRIPTION`: `Config/testthat/edition: 3`.
- Use `expect_snapshot()` for complex output; `expect_equal()` / `expect_error()` for unit assertions.
- Test file per source file: `tests/testthat/test-<source-file>.R`.
- Use `withr` helpers (`withr::local_tempfile()`, `with_options()`) to manage side effects.
- Mock HTTP or file I/O with `httptest2` or `withr::with_file()`.

---

## Pitfalls to avoid

- `T` / `F` instead of `TRUE` / `FALSE` — dangerous if `T` is reassigned.
- `1:length(x)` → use `seq_along(x)` (safe when `x` is empty).
- `1:nrow(df)` → use `seq_len(nrow(df))`.
- `sapply()` in package code — use `vapply()` for type-stable output, or `purrr::map_*()`.
- `attach()` — never use it; creates hard-to-debug name collisions.
- `setwd()` inside scripts or functions — use `here::here()` for portable paths.
- `options(stringsAsFactors = TRUE)` behavior assumed — always be explicit with `read.csv(..., stringsAsFactors = FALSE)` or use `readr`.
- Growing objects in a loop (`x <- c(x, val)`) — pre-allocate or use `vector("list", n)` then fill.
- Leaving `browser()` or `print()` debug calls in committed code.
- Using `<<-` (global assignment) inside Shiny reactive expressions — use `reactiveValues()` instead.
