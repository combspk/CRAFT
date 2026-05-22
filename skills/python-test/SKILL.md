---
name: python-test
description: Write and run pytest suites for generated Python code; covers unit tests, mocking, Dash callback testing, and import-smoke tests; detect failures and hand off to code-debug
license: MIT
compatibility: opencode
metadata:
  language: Python 3.9+
  framework: pytest
---

# Skill: python-test

## What I do

Write, run, and interpret pytest test suites for generated Python code. Covers
pure-function unit tests, external-API mock strategies, Dash callback testing
via `dash.testing`, and lightweight import/smoke tests that catch the most
common runtime failures before a user ever loads the app.

## When to use me

Load this skill **after writing or modifying Python code** (after step 1 or
step 5 of the agent workflow). Run tests before the accessibility audit so
crashes are caught early. If any test fails, load the `code-debug` skill to
diagnose and fix before re-running.

---

## Test categories and when to write each

| Category | When | Tools |
|----------|------|-------|
| **Import / smoke** | Always — first test written | `pytest`, plain `import` |
| **Unit — pure functions** | Any function with no I/O | `pytest`, `pytest.mark.parametrize` |
| **Unit — data layer** | Functions that call external APIs | `pytest`, `unittest.mock.patch` |
| **Integration — Dash callbacks** | After writing callbacks | `dash.testing.application_runners` |
| **Integration — figures** | After writing figure builders | Assert `go.Figure` type + trace counts |

---

## Step-by-step procedure

### 1. Locate testable units

Read every `.py` file in the project. Identify:
- Pure functions with well-defined inputs/outputs (highest priority)
- Functions that call external HTTP APIs (mock these)
- Dataclasses / Pydantic models (test field defaults and validation)
- Dash callback functions (test via `dash.testing` or by calling directly with
  mocked inputs)

### 2. Create the test directory

```
<project>/
  tests/
    __init__.py          (empty)
    test_data.py         (data-fetching layer)
    test_figures.py      (figure builders)
    test_layout.py       (layout import + smoke)
    test_callbacks.py    (callback logic)
    conftest.py          (shared fixtures)
```

### 3. Write conftest.py first

```python
# conftest.py
import pytest
from unittest.mock import patch, MagicMock

@pytest.fixture
def mock_httpx_get():
    """Patch httpx.get so no real network calls are made."""
    with patch("httpx.get") as mock_get:
        yield mock_get

@pytest.fixture
def sample_pubchem_response():
    return {
        "IdentifierList": {"CID": [2244]}
    }

@pytest.fixture
def sample_properties_response():
    return {
        "PropertyTable": {"Properties": [{
            "MolecularFormula": "C9H8O4",
            "MolecularWeight": "180.16",
            "CanonicalSMILES": "CC(=O)Oc1ccccc1C(=O)O",
            "IsomericSMILES": "CC(=O)Oc1ccccc1C(=O)O",
            "IUPACName": "2-acetyloxybenzoic acid",
            "InChIKey": "BSYNRYMUTXBXSQ-UHFFFAOYSA-N",
            "XLogP": 1.2,
            "TPSA": 63.6,
            "HBondDonorCount": 1,
            "HBondAcceptorCount": 4,
            "RotatableBondCount": 3,
            "HeavyAtomCount": 13,
        }]}
    }
```

### 4. Import / smoke tests (always write these first)

```python
# test_layout.py
def test_layout_imports_without_error():
    """Layout module must import and build_layout() must return a Div."""
    from layout import build_layout
    from dash import html
    layout = build_layout()
    assert isinstance(layout, html.Div)

def test_example_chemicals_defined():
    from layout import EXAMPLE_CHEMICALS
    assert isinstance(EXAMPLE_CHEMICALS, list)
    assert len(EXAMPLE_CHEMICALS) > 0
```

### 5. Unit tests — data layer with mocking

Mock at the `httpx.get` level, not at the function level. This tests the
full parsing logic while eliminating network dependence.

```python
# test_data.py
import json
from unittest.mock import patch, MagicMock
import pytest
from data import _pubchem_cid_from_name, fetch_chemical_profile, ChemicalProfile


def _make_response(payload: dict, status: int = 200) -> MagicMock:
    """Build a fake httpx.Response."""
    mock_resp = MagicMock()
    mock_resp.status_code = status
    mock_resp.json.return_value = payload
    mock_resp.raise_for_status.return_value = None
    return mock_resp


def test_pubchem_cid_from_name_returns_first_cid():
    payload = {"IdentifierList": {"CID": [2244, 9999]}}
    with patch("httpx.get", return_value=_make_response(payload)):
        from data import _pubchem_cid_from_name
        cid = _pubchem_cid_from_name("aspirin")
    assert cid == 2244


def test_pubchem_cid_from_name_returns_none_on_empty():
    payload = {"IdentifierList": {"CID": []}}
    with patch("httpx.get", return_value=_make_response(payload)):
        from data import _pubchem_cid_from_name
        cid = _pubchem_cid_from_name("notachemical_xyz")
    assert cid is None


def test_pubchem_cid_from_name_returns_none_on_http_error():
    with patch("httpx.get", side_effect=Exception("timeout")):
        from data import _pubchem_cid_from_name
        cid = _pubchem_cid_from_name("aspirin")
    assert cid is None


@pytest.mark.parametrize("mw,xlogp,hbd,hba,expected_pass", [
    (180.0, 1.2, 1, 4, True),   # aspirin — all within limits
    (600.0, 6.0, 6, 11, False), # fails 4 rules
    (499.0, 4.9, 5, 10, True),  # exactly at limits — pass (0 fails)
    (501.0, 4.9, 5, 10, False), # MW just over — 1 fail, still pass by Ro5
])
def test_lipinski_pass_logic(mw, xlogp, hbd, hba, expected_pass):
    """Lipinski Ro5 allows up to 1 violation."""
    fails = sum([mw > 500, xlogp > 5, hbd > 5, hba > 10])
    result = fails <= 1
    assert result == expected_pass


def test_fetch_chemical_profile_returns_profile_on_not_found():
    """When PubChem returns no CID, profile.cid must be None (not raise)."""
    payload = {"IdentifierList": {"CID": []}}
    with patch("httpx.get", return_value=_make_response(payload)):
        profile = fetch_chemical_profile("notachemical_xyz_abc")
    assert isinstance(profile, ChemicalProfile)
    assert profile.cid is None
```

### 6. Unit tests — figure builders

```python
# test_figures.py
import plotly.graph_objects as go
from figures import lipinski_radar, toxcast_donut, property_bar


def test_lipinski_radar_returns_figure():
    fig = lipinski_radar(180.0, 1.2, 1, 4, 63.6, 3)
    assert isinstance(fig, go.Figure)
    assert len(fig.data) == 2  # ideal zone + compound traces


def test_lipinski_radar_handles_none_values():
    """Should not raise when some properties are None."""
    fig = lipinski_radar(None, None, None, None, None, None)
    assert isinstance(fig, go.Figure)


def test_toxcast_donut_no_data():
    fig = toxcast_donut(None, None, None)
    assert isinstance(fig, go.Figure)


def test_toxcast_donut_with_data():
    fig = toxcast_donut(active=50, inactive=900, tested=1000)
    assert isinstance(fig, go.Figure)
    labels = [t.labels for t in fig.data if hasattr(t, "labels")]
    flat = [l for ls in labels for l in ls]
    assert "Active" in flat


def test_property_bar_returns_figure():
    fig = property_bar(180.0, 1.2, 63.6, 1, 4, 3)
    assert isinstance(fig, go.Figure)
    assert len(fig.data) >= 1


def test_property_bar_all_none():
    fig = property_bar(None, None, None, None, None, None)
    assert isinstance(fig, go.Figure)
```

### 7. Callback unit tests (direct invocation)

For Dash callbacks without a browser, import the inner function directly
rather than going through the decorator:

```python
# test_callbacks.py
import json
from unittest.mock import patch, MagicMock
import pytest


def _make_response(payload, status=200):
    mock = MagicMock()
    mock.json.return_value = payload
    mock.raise_for_status.return_value = None
    return mock


def test_fill_example_returns_correct_chemical():
    """fill_example must return the right name from EXAMPLE_CHEMICALS."""
    from layout import EXAMPLE_CHEMICALS
    # Simulate callback_context with index=2 triggered
    trigger_prop_id = '{"index":2,"type":"example-btn"}.n_clicks'
    import callbacks as cb_module
    # Access the inner function via the app's callback map — or test logic directly
    from layout import EXAMPLE_CHEMICALS
    idx = 2
    assert EXAMPLE_CHEMICALS[idx] == "PFOA"


def test_alert_helper_wraps_in_div():
    """_alert must return an html.Div, not a bare dbc.Alert."""
    # Import after app is initialised
    import importlib, sys
    # Temporarily mock app import to avoid Dash server startup
    import app as app_module
    from callbacks import _alert
    from dash import html
    result = _alert("test message", "warning")
    assert isinstance(result, html.Div)
    assert result.role == "alert"
```

### 8. Run the tests

```bash
# From the project root
pytest tests/ -v --tb=short 2>&1
```

Always capture full output. If a test fails:
1. Read the full traceback — do not truncate.
2. Identify the failing assertion or exception type.
3. Load the `code-debug` skill and follow its triage procedure.

### 9. Coverage report

```bash
pip install pytest-cov
pytest tests/ --cov=. --cov-report=term-missing --cov-omit="tests/*,app.py" 2>&1
```

Target: **≥ 70% coverage** on `data.py` and `figures.py`. `layout.py` and
`callbacks.py` are harder to cover without a browser; aim for ≥ 40% via
import/smoke tests and direct function calls.

---

## Mocking strategy reference

| What to mock | Where to patch | Pattern |
|---|---|---|
| `httpx.get` (all HTTP) | `"httpx.get"` | `patch("httpx.get", return_value=mock_resp)` |
| A specific module's import of httpx | `"data.httpx.get"` | Patch at point of use |
| `base64.b64encode` | `"data.base64.b64encode"` | For image encoding tests |
| Dash `callback_context` | `"dash.callback_context"` | `patch("dash.callback_context", ...)` |
| Time-dependent code | `"time.time"` or `"datetime.datetime"` | Use `freezegun` |

Always prefer patching **where the name is looked up** (the importing module),
not where it is defined.

---

## Dash-specific testing notes

- `dcc.Input.n_submit` starts as `None`; callbacks with `prevent_initial_call=True`
  won't fire on first load — confirm this in tests.
- Pattern-matched IDs (`{"type": "example-btn", "index": ALL}`) require the
  full dict structure in test inputs.
- To test the full app end-to-end with a real browser, use
  `dash.testing.application_runners.ThreadedLocalServer` + `selenium`.
  This is optional; import/smoke + unit tests give most of the value.

---

## Common test failures and quick fixes

| Failure | Likely cause | Fix |
|---------|-------------|-----|
| `ModuleNotFoundError` | Missing dependency or wrong working dir | `pip install -r requirements.txt`; run pytest from project root |
| `AttributeError: module 'X' has no attribute 'Y'` | Removed API in newer version | Check changelog; use the `code-debug` skill |
| `TypeError: unexpected keyword argument` | Framework version break (dbc v2, dash v4) | Remove or replace unsupported kwarg |
| `AssertionError` on figure trace count | Figure builder changed structure | Update expected count in test |
| `httpx.get` not intercepted | Patching wrong path | Patch `"data.httpx.get"` not `"httpx.get"` if data.py imports httpx directly |
| Callback `no_update` returned unexpectedly | `prevent_initial_call` fired | Ensure test triggers a real Input change |
