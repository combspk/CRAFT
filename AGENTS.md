# AGENTS.md

## Repo purpose

This repo configures and drives a mostly-autonomous code-writing agent using OpenCode and a NIEHS-internal LiteLLM proxy. The agent's job is to produce and iteratively improve working code through a repeating cycle — not to maintain an application of its own.

## Skills

Load the appropriate skill(s) at each workflow step. Skills live in `.opencode/skills/`.

### Code production skills
| Skill | When to load |
|-------|-------------|
| `python-code` | Step 1 / 10 when writing or improving Python |
| `r-code` | Step 1 / 10 when writing or improving R |
| `python-test` | Step 4 after writing Python — write and run tests immediately |
| `code-debug` | Step 6 and any time a traceback, import error, or test failure occurs |
| `accessibility-audit` | Step 7 (and again at step 8 after each accessibility fix pass) |
| `feature-analysis` | Step 9 before each implementation pass |

### Cheminformatics and toxicology skills
| Skill | When to load |
|-------|-------------|
| `cheminformatics-analysis` | When analyzing chemical structures or datasets: SMILES handling, molecular descriptors, fingerprints, QSAR, drug-likeness filters, scaffold and SAR analysis |
| `toxicology-assessment` | When performing or interpreting a hazard assessment: endpoint coverage, PoD derivation (NOAEL/BMDL/RfD), ToxCast interpretation, read-across, GHS classification, risk characterization |
| `data-visualization` | Step 1 / 8 when producing any chart or figure from chemical or biological data: dose-response curves, SAR heatmaps, chemical space plots, property distributions |
| `scientific-writing` | When drafting report text, hazard narrative, methods sections, or regulatory dossier language |

## Agent workflow

Execute these steps **in order** for every code generation task. Steps 3–5
form a mandatory bug-detection gate: code does not advance to accessibility
or feature review until every test — old and new — is green.

> **Before writing a single line of code on any change task:**
> Run `pytest tests/ -v --tb=short` and confirm the baseline is green.
> If pre-existing tests are already failing, fix them first.
> Never add new code on top of a broken baseline.

1. **Write** — Load `python-code` or `r-code` skill. Produce code to meet the
   given specifications. Load `cheminformatics-analysis` or
   `toxicology-assessment` when the task involves chemical analysis or hazard
   evaluation. Load `data-visualization` when producing any chart or figure.

2. **NIEHS chrome** — Every public-facing Dash app must include the NIEHS/NTP
   site-wide header and footer before any other layout work proceeds.
   - Import `niehs_header()` and `niehs_footer()` from `niehs_chrome.py` into
     `layout.py` (or the equivalent layout module).
   - Place `niehs_header()` as the **first** child of `build_layout()` (after
     the skip-navigation link).
   - Place `niehs_footer()` as the **last** child of `build_layout()`.
   - Call `write_chrome_asset()` from `app.py` before `app.run()` — this writes
     `assets/niehs_chrome_inject.js`, which uses a `MutationObserver` to inject
     the NIEHS HTML into placeholder divs the moment React renders them.
     **Never** use `dangerously_allow_html`, `dcc.Markdown`, or Dash clientside
     callbacks for NIEHS chrome injection — all three have known failure modes
     in Dash 4 (see "NIEHS chrome lessons learned" below).
   - Copy NIEHS CSS and JS assets from `niehs/` into the app's `assets/niehs/`
     directory so Dash serves them automatically — do **not** add external
     stylesheet or script tags for these files.
   - Remove any app-specific top navbar that duplicates NIEHS branding; replace
     with an app title bar styled below the NIEHS header.
   - Update `app.py` `title=` to include `"| NIEHS"` and add an `author` meta
     tag attributing NIEHS/NTP.
   - Write tests in `tests/test_niehs_chrome.py` covering: import smoke,
     `niehs_header()` returns `html.Div`, `niehs_footer()` returns `html.Div`,
     graceful fallback when HTML files are missing, and `build_layout()` contains
     both chrome ids.

3. **`.gitignore`** — Every new project must have a `.gitignore` at the repo
   root before any code is committed. Create it immediately after the initial
   write step. It must always include at minimum:
   - `secrets/` and any file matching `*.key`, `*.pem`, `.env`, `.env.*`
   - Auto-generated files that are rebuilt at runtime (e.g.
     `assets/niehs_chrome_inject.js`)
   - Python bytecode: `__pycache__/`, `*.py[cod]`, `*.pyo`
   - Virtual environments: `.venv/`, `venv/`, `env/`
   - Test/coverage caches: `.pytest_cache/`, `.coverage`, `htmlcov/`
   - OS metadata: `.DS_Store` (macOS), `Thumbs.db` (Windows)
   - Editor files: `.vscode/`, `.idea/`
   - Build artifacts: `dist/`, `build/`, `*.egg-info/`

   **Rule:** if a file contains a credential, token, or API key — or is
   auto-generated and not meaningful to version-control — it belongs in
   `.gitignore`. When in doubt, exclude it.

4. **Test — write** — Load `python-test` skill. For every new or changed
   module, write the corresponding pytest suite covering:
   - Import / smoke tests (always)
   - Unit tests for pure functions (always)
   - Mock-based tests for any function that calls an external API or the
     filesystem (always)
   - Dash callback / figure tests where applicable
   Place all tests under `tests/` before running anything.

5. **Test — run & detect** — Execute `pytest tests/ -v --tb=short` and capture
   the full output. This run must cover **all** tests in the suite, not just the
   ones written in step 4. Record every failure. If all tests pass, proceed to
   step 7. Do **not** skip just because the app "seems to work" in the browser —
   every test must be green first.

   **Regression rule:** if any test that was passing before the current task is
   now failing, that is a regression and must be fixed before proceeding. Do not
   write new code on top of failing tests.

6. **Bug fix** — Load `code-debug` skill. For each failing test, follow the
   triage loop: Reproduce → Read traceback bottom-up → Classify error →
   Isolate minimal fix → Apply fix → Re-run only the failing test to confirm
   green → then re-run the **full** suite. Repeat until `pytest tests/ -v` is
   100% green. Do **not** move to step 7 with any red tests.

7. **Accessibility audit** — Load `accessibility-audit` skill. Review all UI
   code for Section 508 / WCAG 2.0 AA compliance. Produce a numbered violation
   list with file, line, criterion, and exact fix for each item.

8. **Accessibility fix** — Apply every fix from step 7. Re-run
   `pytest tests/ -v` after each batch of changes to confirm no regressions.

9. **Improvement analysis** — Load `feature-analysis` skill. Analyze the
   current code for bugs, performance issues, missing features, and UX
   improvements. Produce a prioritized backlog.

10. **Implement improvements** — Apply the highest-priority items from step 9.
    Reload the appropriate language skill. After each implementation pass:
    - Re-run `pytest tests/ -v` — fix any regressions before continuing
    - Add or update tests to cover the new code (back to step 4 scope)

11. **Repeat** — Return to step 7 (accessibility audit) after each
    implementation pass. The cycle ends when:
    - All tests are green
    - No new accessibility violations are found
    - The feature backlog has no Critical or High items remaining

> **Mandatory re-entry rules**
> - Any traceback, `AttributeError`, `TypeError`, or `ImportError` anywhere
>   (browser console, server log, test output) → immediately load `code-debug`
>   and return to step 6. Do not attempt ad-hoc fixes without the skill.
> - Any test regression introduced by a later step → return to step 6 before
>   proceeding further.
> - Accessibility fixes and improvements that change UI code → re-run tests
>   (step 5) immediately after.

> **Additional skill triggers** (load alongside workflow steps as needed):
> - Load `scientific-writing` when drafting narrative text for a report or
>   regulatory output.

---

## NIEHS chrome lessons learned

These failure modes were encountered during the header/footer implementation
and must **not** be repeated:

| Approach | Why it fails in Dash 4 |
|---|---|
| `html.Div(raw_html, dangerously_allow_html=True)` | `dangerously_allow_html` was removed from `html.Div` in Dash 4. Raises `TypeError`. |
| `dcc.Markdown(raw_html, dangerously_allow_html=True)` | Markdown's parser processes the HTML first and mangles complex nested tags, attributes, and whitespace. |
| `app.clientside_callback` with inline JS string | Two callbacks sharing the same `Input` — Dash silently drops the second. Also, large f-string-embedded HTML in the JS string causes `dc[namespace][function_name] is not a function` because the inline script is not reliably served before the renderer resolves it. |
| Static `assets/` JS with `DOMContentLoaded` | Fires on the initial HTML shell before React has rendered any components. `getElementById` returns `null` because the placeholder divs do not exist in the DOM yet. |
| **`MutationObserver` in static `assets/` JS** ✓ | Watches the DOM for React to insert each placeholder div, then injects immediately. Disconnects itself once both are done. Immune to React rendering timing. **This is the correct approach.** |

The `write_chrome_asset()` function in `niehs_chrome.py` generates
`assets/niehs_chrome_inject.js` using the `MutationObserver` pattern.
It must be called from `app.py` before `app.run()` so the file is
written/refreshed on every server start.

### Accessibility checklist (step 2 minimum bar)

Derived from the [Revised 508 Standards](https://www.access-board.gov/guidelines-and-standards/communications-and-it/about-the-ict-refresh/final-rule/text-of-the-standards-and-guidelines) (36 C.F.R. Part 1194) and the [Section 508 Guide to Accessible Web Design & Development](https://www.section508.gov/develop/guide-accessible-web-design-development/). All WCAG 2.0 Level A and AA criteria apply.

**Images (WCAG 1.1.1)**
- Meaningful images have descriptive `alt` text. Decorative images use `alt=""`.
- Avoid images of text; use real text styled with CSS instead.
- Background images that convey meaning must have an on-page text equivalent.

**Color & Contrast (WCAG 1.4.1, 1.4.3)**
- Contrast ratio ≥ 4.5:1 for normal text; ≥ 3:1 for large text (≥ 18pt or ≥ 14pt bold).
- Color is never the sole means of conveying information — pair color with text, icons, or patterns.

**Keyboard & Focus (WCAG 2.1.1, 2.1.2, 2.4.3, 2.4.7, 3.2.1)**
- All functionality is operable by keyboard alone; no mouse-only interactions.
- No keyboard trap — users can always tab away from any component.
- Focus order follows logical reading/visual order (left-to-right, top-to-bottom).
- Visible focus indicator is never intentionally removed (`outline: none` is a red flag).
- Receiving focus does not trigger a context change (no auto-navigation on focus).

**Forms (WCAG 1.3.1, 3.2.2, 3.3.1, 3.3.2, 3.3.3, 3.3.4)**
- Every input has an associated `<label>`, or `aria-label`/`aria-labelledby` if label is not visible.
- Related inputs are grouped with `<fieldset>`/`<legend>` (e.g., radio groups, address blocks).
- Input errors are identified in text — not by color alone — and include a description of the error and, where possible, a suggestion for correction.
- Context changes (new windows, navigation, page rearrangement) are not triggered on input change without prior warning; use an explicit button to confirm.
- Legal, financial, or data-deletion forms must support at least one of: reversible submission, error checking, or review-before-submit.

**Headings & Structure (WCAG 1.3.1, 2.4.6)**
- Heading hierarchy is logical and sequential (`h1` → `h2` → `h3` …); do not skip levels.
- Heading tags are used only for actual headings, not for visual styling.
- Descriptive headings label each distinct section of content.

**Page & Navigation (WCAG 2.4.1, 2.4.2, 2.4.4, 2.4.5, 3.2.3, 3.2.4)**
- Every page has a descriptive `<title>` that reflects its topic or result.
- A skip-navigation link (or equivalent bypass mechanism) is provided for repeated content blocks.
- Link and button text is descriptive on its own — avoid "click here" or "read more".
- Navigation repeated across pages appears in consistent order and is identified consistently.
- Multiple pathways to reach a page are provided (e.g., nav menu + search + sitemap).

**Dynamic Content & ARIA (WCAG 4.1.2)**
- All custom UI components expose name, role, state, and value to assistive technologies.
- `aria-live` regions announce dynamic updates (status messages, errors, loading states).
- `role="alert"` for non-modal notifications; `role="alertdialog"` when user action is required.
- After focus-moving operations (modal open/close, deletion), focus is explicitly set to a logical target — not left to fall back to the top of the DOM.

**Language (WCAG 3.1.1, 3.1.2)**
- `<html lang="en">` (or correct BCP 47 tag) is always set.
- Passages in a language other than the page default carry a `lang` attribute on a parent element.

**Tables (WCAG 1.3.1)**
- Data tables use `<th scope="col|row">` for headers; layout tables use `role="presentation"` and no structural table elements.
- Complex tables with multiple header levels use `id`/`headers` associations.

**Media (WCAG 1.2.1–1.2.5, 1.4.2, 2.2.2, 503.4)**
- Prerecorded audio-only: text transcript provided.
- Prerecorded video-only: text transcript or audio track provided.
- Synchronized video: captions and audio descriptions provided; live audio: real-time captions.
- Audio that auto-plays for > 3 seconds must have a pause/stop/volume control.
- Moving, blinking, or scrolling content that lasts > 5 seconds must have a pause/stop/hide control.
- Media player exposes caption (CC) and audio description (AD) controls at the same level as volume/program controls (Section 508 §503.4).

**Flashing (WCAG 2.3.1)**
- No content flashes more than 3 times per second.

**Timed Events (WCAG 2.2.1)**
- Any time limit allows users to turn off, adjust (≥ 10× default), or extend (≥ 20-second warning, ≥ 10 extensions) — unless real-time or > 20 hours.

**Frames & iFrames (WCAG 2.4.1, 4.1.2)**
- Every `<iframe>` has an accessible name via `title` attribute.
- Frames are obsolete in HTML5; prefer native elements.

**Parsing & Code Quality (WCAG 4.1.1)**
- Markup has complete start/end tags, no duplicate attributes, and unique `id` values.
- Validate HTML to catch structural errors that break assistive technology parsing.

**Text Resize & Responsive Layout (WCAG 1.4.4)**
- Text scales to 200% zoom without loss of content or functionality; use relative units (`em`, `rem`) for sizing.

**Sensory Independence (WCAG 1.3.3)**
- Instructions do not rely solely on shape, size, position, color, or sound (e.g., avoid "click the red button on the left").

**Reference tools**: [ANDI (SSA)](https://www.ssa.gov/accessibility/andi/help/install.html) for automated name/role/value inspection; [WebAIM Contrast Checker](https://webaim.org/resources/contrastchecker/) for color contrast.

## Structure

```
opencode.json          # OpenCode config — provider, models, API key reference
secrets/
  niehs-litellm-key   # Plain-text API key file; read by opencode.json at runtime
codingagent/          # MCP server and tools
  server.py           # FastMCP entry point (34 tools across 6 modules)
  pyproject.toml
  tools/
    episuite.py       # EPA EpiSuite tools
    pubchem.py        # PubChem tools
    chembl.py         # ChEMBL tools
    ncbi.py           # NCBI PubMed + Gene tools
    uniprot.py        # UniProt tools
    comptox.py        # EPA CompTox tools
  README.md           # MCP server usage and tool reference
.opencode/skills/
  accessibility-audit/SKILL.md     # Section 508 / WCAG 2.0 audit procedure
  cheminformatics-analysis/SKILL.md  # SMILES, descriptors, fingerprints, QSAR, SAR
  code-debug/SKILL.md              # Error triage: tracebacks, import errors, API breaks
  data-visualization/SKILL.md      # Plotting conventions for chemical/biological data
  feature-analysis/SKILL.md        # Bug, UX, and feature improvement backlog
  python-code/SKILL.md             # Python conventions and patterns
  python-test/SKILL.md             # pytest suites, mocking, Dash callback testing
  r-code/SKILL.md                  # R / tidyverse / Shiny conventions and patterns
  scientific-writing/SKILL.md      # Hazard narrative, methods, regulatory text
  toxicology-assessment/SKILL.md   # PoD derivation, risk characterization, GHS
```

## OpenCode config notes

- Provider: `niehs-litellm` — an OpenAI-compatible endpoint at `https://litellm.toxpipe.niehs.nih.gov/v1`
- Available models: `azure-gpt-5`, `azure-gpt-5.4`, `claude-4.6-sonnet`
- API key is loaded from a file via `{file:C:/Users/combspk/Desktop/opencode/secrets/niehs-litellm-key}` — do not hardcode or commit the key
- Timeout is set to 300 000 ms (5 min) — long-running tool calls are expected

## Key constraints

- Do not modify or commit `secrets/niehs-litellm-key`
- Do not change the `baseURL` without verifying the new endpoint is reachable on the NIEHS network
- There are no build, test, lint, or CI steps in this repo — nothing to run
