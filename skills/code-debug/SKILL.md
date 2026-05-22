---
name: code-debug
description: Systematic triage and fix procedure for runtime errors, import failures, traceback analysis, and API compatibility breaks in generated Python code
license: MIT
compatibility: opencode
---

# Skill: code-debug

## What I do

Provide a structured, repeatable procedure for diagnosing and fixing runtime
errors in generated Python code — from `ImportError` and `AttributeError`
through framework version breaks, data-shape mismatches, and Dash callback
failures. Always follows the same triage loop: **Reproduce → Classify →
Isolate → Fix → Verify**.

## When to use me

Load this skill whenever:
- A Python process raises a traceback (in the terminal, in a Dash callback, or
  during test runs)
- An import fails at startup
- A third-party library rejects a keyword argument it previously accepted
- A test written with the `python-test` skill fails
- The app runs but produces wrong output (silent logic errors)

---

## Triage loop (always follow in order)

```
1. REPRODUCE  — confirm you can trigger the exact error reliably
2. READ       — read the full traceback from bottom (root cause) to top (call site)
3. CLASSIFY   — assign one error class from the taxonomy below
4. ISOLATE    — find the smallest code change that eliminates the error
5. FIX        — apply the minimal correct fix; do not refactor unrelated code
6. VERIFY     — re-run the failing test or command; confirm green
7. REGRESSION — run the full test suite to confirm nothing else broke
```

Never skip step 6 and 7. A fix that silences one error while introducing
another is not a fix.

---

## Error taxonomy

### Class A — ImportError / ModuleNotFoundError

**Symptoms:** `ModuleNotFoundError: No module named 'X'`

**Triage:**
1. Is the package in `requirements.txt`? If not, add it and `pip install`.
2. Is the package installed in the correct environment?
   ```bash
   python -c "import X; print(X.__file__)"
   ```
3. Is the working directory wrong? `sys.path` must include the project root.
   ```bash
   python -c "import sys; print(sys.path)"
   ```
4. Is it a relative import used in a script context? Change to absolute import.

**Fix pattern:**
```python
# Wrong — relative import outside a package
from .data import fetch_chemical_profile

# Right — absolute import when running as a script
from data import fetch_chemical_profile
```

---

### Class B — AttributeError on a module or object

**Symptoms:** `AttributeError: module 'X' has no attribute 'Y'`

**Triage:**
1. Print the installed version: `python -c "import X; print(X.__version__)"`.
2. Check the library's changelog for the version where `Y` was removed or renamed.
3. Search the installed package for the correct name:
   ```bash
   python -c "import X; print([a for a in dir(X) if 'y' in a.lower()])"
   ```

**Common cases and fixes:**

| Library | Old (broken) | New (correct) |
|---------|-------------|---------------|
| `httpx` ≥0.20 | `httpx.utils.quote(s)` | `urllib.parse.quote(s)` |
| `httpx` ≥0.20 | `httpx.URL(s).copy_with(...)` | `httpx.URL(s).copy_with(...)` (unchanged) |
| `plotly` ≥5 | `go.FigureWidget` in non-notebook | Use `go.Figure` |
| `pandas` ≥2 | `df.append(...)` | `pd.concat([df, new_row])` |
| `dash` ≥2.9 | `app.run_server(...)` | `app.run(...)` |

---

### Class C — TypeError: unexpected keyword argument

**Symptoms:**
```
TypeError: The `dash_bootstrap_components.Alert` component received an
unexpected keyword argument: `role`
```

This is the most common error class when framework major versions are updated.

**Triage:**
1. Identify the component and the rejected kwarg.
2. Check the component's allowed arguments from the error message itself —
   Dash prints the full list.
3. Determine if the kwarg is:
   - **A standard HTML attribute** (e.g. `role`, `aria-*`, `data-*`): in DBC v2+
     and Dash 4+, many components no longer forward arbitrary HTML attributes.
     Move the attribute to a wrapping `html.Div`.
   - **A renamed prop** (e.g. `className` → `class_name`): use the new name.
   - **A removed prop**: find the equivalent new API.

**Fix patterns:**

```python
# WRONG — DBC v2 rejects `role` on Alert
dbc.Alert("msg", color="danger", **{"role": "alert"})

# RIGHT — wrap in html.Div that carries the ARIA attribute
html.Div(
    dbc.Alert("msg", color="danger", dismissable=True),
    role="alert",
    **{"aria-live": "assertive"},
)
```

```python
# WRONG — DBC v2 rejects aria-* on Input
dbc.Input(id="x", **{"aria-describedby": "hint"})

# RIGHT — use dcc.Input (plain Dash) which accepts fewer restrictions,
# or use html.Input for full HTML attribute support
dcc.Input(id="x", className="form-control")
# (attach label via htmlFor on the <label> instead of aria-describedby)
```

**Quick reference — DBC v2 breaking changes:**
- `dbc.Navbar`: removed `aria-*` kwargs → use wrapping `html.Header`
- `dbc.NavbarBrand`: removed `aria-current` → wrap or omit
- `dbc.Input`: removed `aria-*`, `autofocus` (use `autoFocus`) → use `dcc.Input`
- `dbc.Alert`: removed `role` → wrap in `html.Div(role=...)`
- `dbc.Button`: removed `aria-label` → use visible text (preferred) or `title`

---

### Class D — TypeError / ValueError in data parsing

**Symptoms:** `float("N/A")` → `ValueError`; `None["key"]` → `TypeError`

**Triage:** Find the exact line. Ask: "What value arrived instead of what was
expected?" Add a `logger.debug("value=%r", val)` line just before the crash.

**Fix pattern:** Validate before converting.
```python
# Wrong
profile.mw = float(props.get("MolecularWeight"))  # crashes if None

# Right
mw = props.get("MolecularWeight")
profile.mw = float(mw) if mw is not None else None
```

**Defensive dict access for nested API responses:**
```python
# Wrong — crashes if "PropertyTable" or "Properties" absent
row = data["PropertyTable"]["Properties"][0]

# Right
row = (data or {}).get("PropertyTable", {}).get("Properties", [{}])[0]
```

---

### Class E — HTTP / network errors

**Symptoms:** `httpx.HTTPStatusError`, `httpx.ConnectError`, `httpx.TimeoutException`

**Triage:**
1. Is the endpoint URL correct? Test with `httpx.get(url)` in a REPL.
2. Is the network accessible from this host?
3. Has the API changed its endpoint path or authentication requirements?

**Fix pattern:** All HTTP calls must be wrapped in try/except and return a
sentinel (None or empty dict) on failure — never let network errors propagate
to the UI layer.

```python
def _get(url: str, params=None):
    try:
        r = httpx.get(url, params=params, timeout=_TIMEOUT, follow_redirects=True)
        r.raise_for_status()
        return r.json()
    except Exception as exc:
        logger.warning("GET %s failed: %s", url, exc)
        return None
```

---

### Class F — Dash callback errors

**Symptoms:** Red error banner in the browser; `PreventUpdate`; callback
returning wrong number of outputs.

**Triage:**
1. **Output count mismatch**: count `Output(...)` decorators vs. return tuple length.
   They must match exactly — including `no_update` placeholders.
2. **ID not found**: `dash.exceptions.NonExistentIdException` — the component
   id is not in the layout. Check for typos; check `suppress_callback_exceptions`.
3. **Circular dependency**: Dash raises `CyclicCallbackError` — map the
   Input/Output chain to find the loop.
4. **`prevent_initial_call` not firing**: the callback uses `n_submit` which is
   `None` initially — this is correct behaviour, not a bug.

**Output count fix:**
```python
# 6 Outputs declared → return must always have exactly 6 elements
@app.callback(
    Output("a", "children"),  # 1
    Output("b", "style"),     # 2
    Output("c", "children"),  # 3
    Output("d", "data"),      # 4
    Output("e", "data"),      # 5
    Output("f", "style"),     # 6
    ...
)
def my_callback(...):
    # Every return path must have 6 elements
    return no_update, no_update, no_update, no_update, no_update, no_update
```

---

### Class G — Silent logic errors (wrong output, no crash)

**Symptoms:** App runs, no exception, but data is wrong or missing.

**Triage:**
1. Add `logger.debug` at each data transformation step.
2. Check API response shapes: print `repr(data)[:500]` to the log.
3. Write a unit test that pins the expected output for a known input — if the
   test was already passing, a silent regression will now fail.
4. Use `assert` statements temporarily during debug (remove before committing).

---

## Fix discipline

- **Minimal diff**: fix only the error. Do not refactor unrelated code.
- **One fix per commit** (if committing): makes rollback clean.
- **Always re-run the full test suite** after a fix (`pytest tests/ -v`).
- **Document the root cause** in a code comment if the fix is non-obvious:
  ```python
  # httpx ≥0.20 removed httpx.utils; use stdlib urllib.parse instead
  from urllib.parse import quote as _url_quote
  ```
- **Update requirements.txt** with a version pin if the fix depends on a
  specific version boundary:
  ```
  httpx>=0.27.0   # httpx.utils.quote removed in 0.20
  dash-bootstrap-components>=2.0.0
  ```

---

## Verify checklist

After applying every fix, run through this list before marking the issue closed:

- [ ] The specific failing test/command now passes
- [ ] No new exceptions appear in server logs
- [ ] `pytest tests/ -v` is fully green (or no worse than before the fix)
- [ ] The browser UI renders correctly for at least one example search
- [ ] No `no_update` being returned where a real value is expected
- [ ] All `Output` counts match their callback return tuple lengths
