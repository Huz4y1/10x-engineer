---
tags: [testing, pytest, ci-cd, github-actions]
status: not-started
---

# Testing & CI/CD

> **What this is:** automated checks that your code still works, run automatically every time you change anything.
> **Why you care:** this is what separates a script from software you can trust in production. It's also the most common gap on a data person's CV, and interviewers know it.

---

## The idea in plain English

A test is a small program that runs your code and checks it did the right thing.

**Why bother, when you can just look?**

Because you can look at the thing you just changed. You cannot look at the forty things that thing might have broken. Tests look at all forty, in three seconds, every time.

The real payoff isn't catching bugs today — it's **being able to change things without fear**. Without tests, every refactor is a gamble, so nobody refactors, so the code rots. Tests are what make a codebase changeable.

**CI (Continuous Integration)** is a robot that runs those tests on every push, so "it works on my machine" stops being a thing anyone can say.

---

## 1. pytest basics

```python
# tests/test_transforms.py
from api.services.transforms import calculate_revenue

def test_calculate_revenue():
    assert calculate_revenue(quantity=6, unit_price=2.55) == 15.30
```

```bash
pytest                     # run everything
pytest -v                  # show each test name
pytest tests/test_api.py   # one file
pytest -k "revenue"        # tests matching a name
pytest -x                  # stop at the first failure
pytest --lf                # rerun only last-failed
```

Conventions pytest relies on: files named `test_*.py`, functions named `test_*`, and plain `assert`. No class hierarchy, no `self.assertEqual`.

### Arrange, Act, Assert

Every test has the same three parts. Keeping them visually separate makes tests readable:

```python
def test_drops_cancelled_invoices():
    # Arrange — set up the input
    rows = [
        {"invoice_no": "536365",  "quantity": 6},
        {"invoice_no": "C536367", "quantity": -6},
    ]

    # Act — run the thing
    result = clean_invoices(rows)

    # Assert — check the outcome
    assert len(result) == 1
    assert result[0]["invoice_no"] == "536365"
```

### Parametrize — one test, many cases

```python
import pytest

@pytest.mark.parametrize(
    "invoice_no,expected",
    [
        ("536365",  True),
        ("C536367", False),      # cancellation
        ("c536368", False),      # lowercase — do we handle it?
        ("",        False),      # empty
    ],
)
def test_is_valid_invoice(invoice_no, expected):
    assert is_valid_invoice(invoice_no) == expected
```

Four separate tests, one function. Each failure names the exact case. This is the highest value-per-line feature in pytest — **use it for every function with edge cases.**

### Testing that something raises

```python
import pytest

def test_rejects_negative_weeks():
    with pytest.raises(ValueError, match="weeks must be positive"):
        forecast(category="Kitchen", weeks=-1)
```

### Floats

```python
import pytest

assert calculate_revenue(3, 0.1) == 0.30000000000000004    # ✗ this is real
assert calculate_revenue(3, 0.1) == pytest.approx(0.30)    # ✓
```

---

## 2. Fixtures — shared setup

A fixture is a function that builds something your tests need. Request it by naming it as an argument.

```python
# tests/conftest.py — pytest finds this automatically, no import needed
import pytest
import pandas as pd

@pytest.fixture
def sample_sales():
    return pd.DataFrame({
        "invoice_no":  ["536365", "536365", "C536367"],
        "product_id":  ["P100", "P200", "P100"],
        "customer_id": ["17850", "17850", "17850"],
        "quantity":    [6, 2, -6],
        "unit_price":  [2.55, 3.39, 2.55],
    })

def test_revenue_excludes_cancellations(sample_sales):
    assert calculate_total_revenue(sample_sales) == pytest.approx(22.08)
```

**Scope** controls how often a fixture is rebuilt:

```python
from pyspark.sql import SparkSession
import pytest

@pytest.fixture(scope="session")     # once for the whole run
def spark():
    spark = SparkSession.builder.master("local[2]").appName("tests").getOrCreate()
    yield spark
    spark.stop()
```

`function` (default) rebuilds per test — safest, no shared state. `session` is for things too expensive to rebuild, like a SparkSession (which takes ~10s to start).

---

## 3. Mocking — not calling the real thing

Tests must not hit Azure SQL or ADLS. Reasons: they'd be slow, they'd need credentials, they'd fail when the network hiccups, and they'd change real data.

**Mocking** replaces a dependency with a fake that returns what you tell it.

```python
from unittest.mock import patch, MagicMock

@patch("api.services.sales.fetch_from_db")
def test_forecast_endpoint(mock_fetch):
    mock_fetch.return_value = [
        {"date": "2011-01-01", "revenue": 100.0},
        {"date": "2011-01-02", "revenue": 150.0},
    ]

    result = get_forecast(category="Kitchen", weeks=2)

    assert len(result) == 2
    mock_fetch.assert_called_once_with(category="Kitchen")
```

> **Patch where it's *used*, not where it's defined.** If `api/services/sales.py` does `from api.db import fetch_from_db`, you patch `"api.services.sales.fetch_from_db"`, not `"api.db.fetch_from_db"`. This is the single most common mocking mistake and it fails in a confusing way (the test passes but the mock was never used).

### The better alternative: dependency override

For FastAPI, you don't need `patch` at all — that's what `Depends` was for ([[FastAPI fundamentals]]):

```python
from api.main import app
from api.dependencies import get_db

async def fake_db():
    return FakeSession(rows=[{"date": "2011-01-01", "revenue": 100.0}])

app.dependency_overrides[get_db] = fake_db      # swap the real one out
# ... run tests ...
app.dependency_overrides.clear()                # always clean up
```

Cleaner, type-safe, no string paths to get wrong. **This is why dependency injection is worth using.**

> **What not to mock:** don't mock the thing you're testing, and don't mock pure functions. If you're mocking six things to test one function, the function is doing too much — that's the test telling you about a design problem.

---

## 4. Testing FastAPI

```python
# tests/test_api.py
from fastapi.testclient import TestClient
from api.main import app

client = TestClient(app)

def test_health():
    r = client.get("/health")
    assert r.status_code == 200
    assert r.json() == {"status": "ok"}

def test_forecast_validates_weeks():
    r = client.get("/sales/forecast", params={"category": "Kitchen", "weeks": 999})
    assert r.status_code == 422                          # Pydantic rejected it
    assert "weeks" in str(r.json()["detail"])

def test_forecast_returns_expected_shape():
    r = client.get("/sales/forecast", params={"category": "Kitchen", "weeks": 4})
    assert r.status_code == 200
    body = r.json()
    assert len(body["forecast"]) == 4
    assert {"week_starting", "predicted_revenue"} <= body["forecast"][0].keys()

def test_unknown_customer_returns_404():
    r = client.get("/customers/does-not-exist/segment")
    assert r.status_code == 404
```

`TestClient` calls your app **in-process** — no server, no network, no port. Tests run in milliseconds.

For an async app with a `lifespan` that loads models:

```python
import pytest
from httpx import AsyncClient, ASGITransport

@pytest.mark.asyncio
async def test_predict():
    async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as client:
        r = await client.post("/predict/segment", json={
            "customer_id": "17850", "recency_days": 30,
            "frequency": 5, "monetary_value": 250.0,
        })
    assert r.status_code == 200
    assert r.json()["segment"] in ["at_risk", "regular", "loyal", "champion"]
```

(Needs `pytest-asyncio` and `asyncio_mode = "auto"` in your config.)

**What's worth testing in an API:**
- ✅ Every endpoint returns the right status code
- ✅ Validation rejects bad input (this is testing *your Pydantic models*, which is real logic)
- ✅ The response shape matches the contract
- ✅ Error paths — 404, 422, and a dependency failure
- ❌ Not the framework itself. FastAPI's routing is already tested.

---

## 5. Testing PySpark

The trap: spinning up Spark takes ~10 seconds. Do it per test and your suite takes ten minutes and nobody runs it.

```python
# tests/conftest.py
import pytest
from pyspark.sql import SparkSession

@pytest.fixture(scope="session")                   # ← once for the entire run
def spark():
    spark = (SparkSession.builder
             .master("local[2]")
             .appName("tests")
             .config("spark.sql.shuffle.partitions", "2")     # ← default is 200. Huge win.
             .config("spark.ui.enabled", "false")
             .getOrCreate())
    yield spark
    spark.stop()
```

> **`spark.sql.shuffle.partitions = 2`** is the difference between a 3-second test and a 90-second one. The default of 200 means every shuffle creates 200 tiny tasks on your 3-row test DataFrame.

```python
# tests/test_silver.py
from pipelines.silver import clean_sales

def test_removes_cancelled_invoices(spark):
    df = spark.createDataFrame(
        [("536365", "P100", 6, 2.55), ("C536367", "P100", -6, 2.55)],
        ["invoice_no", "product_id", "quantity", "unit_price"],
    )

    result = clean_sales(df)

    assert result.count() == 1
    assert result.first()["invoice_no"] == "536365"

def test_deduplicates(spark):
    df = spark.createDataFrame(
        [("536365", "P100", "17850", 6), ("536365", "P100", "17850", 6)],
        ["invoice_no", "product_id", "customer_id", "quantity"],
    )
    assert clean_sales(df).count() == 1
```

**The structural requirement:** your transformation logic must be a **function taking a DataFrame and returning a DataFrame**, with no `spark.read` or `.write` inside it.

```python
from pyspark.sql import functions as F

# ✗ untestable — reads and writes are baked in
def run_silver():
    df = spark.read.table("retail.bronze.online_retail")
    cleaned = df.filter(...)
    cleaned.write.saveAsTable("retail.silver.sales")

# ✓ testable — pure transformation, I/O at the edges
def clean_sales(df: DataFrame) -> DataFrame:
    return (df
        .filter(~F.col("invoice_no").startswith("C"))
        .filter(F.col("quantity") > 0)
        .dropDuplicates(["invoice_no", "product_id", "customer_id"]))

def run_silver(spark):                              # thin wrapper, not unit-tested
    df = spark.read.table("retail.bronze.online_retail")
    clean_sales(df).write.saveAsTable("retail.silver.sales")
```

> **This is the main lesson of the section:** testability is a design property, not something you add afterwards. Separating I/O from logic makes code testable *and* reusable *and* easier to read. If something is hard to test, that's information about the code, not about testing.

---

## 6. Testing ML code

ML tests are different — you can't assert an exact accuracy. Test the **plumbing**, not the intelligence.

```python
import json
from torch import nn
import numpy as np
import torch

def test_model_output_shape():
    model = SalesForecastNet(n_features=10)
    out = model(torch.randn(32, 10))
    assert out.shape == (32, 1)

def test_training_step_changes_weights():
    """The classic 'are gradients actually flowing' test."""
    model = SalesForecastNet(n_features=10)
    before = model.net[0].weight.clone()

    optimizer = torch.optim.Adam(model.parameters(), lr=1e-2)
    loss = nn.MSELoss()(model(torch.randn(8, 10)), torch.randn(8, 1))
    optimizer.zero_grad(); loss.backward(); optimizer.step()

    assert not torch.equal(before, model.net[0].weight)

def test_onnx_matches_pytorch():
    """Guards the export contract from [[Model export and serving]]."""
    model.eval()
    x = torch.randn(8, N_FEATURES, dtype=torch.float32)     # batch ≠ export batch

    with torch.no_grad():
        torch_out = model(x).numpy()
    onnx_out = session.run(None, {"features": x.numpy()})[0]

    np.testing.assert_allclose(torch_out, onnx_out, rtol=1e-4, atol=1e-5)

def test_feature_order_is_stable():
    """Catches the silent wrong-answer bug."""
    with open("models/forecast_metadata.json") as f:
        assert json.load(f)["feature_order"] == FEATURE_ORDER

def test_no_time_leakage_in_split():
    """Guards against the worst ML mistake."""
    train, val = time_split(df, cutoff="2011-10-01")
    assert train["date"].max() < val["date"].min()

def test_model_beats_naive_baseline():
    """A behavioural test — allow a range, not an exact number."""
    assert evaluate(model, test_data)["mae"] < naive_forecast_mae(test_data)
```

> **Never assert an exact metric** (`assert accuracy == 0.847`). Training isn't perfectly deterministic across hardware, and the test will fail for no real reason. Assert a *threshold* or a *comparison to a baseline* instead.

---

## 7. GitHub Actions

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install uv
        uses: astral-sh/setup-uv@v3
        with:
          enable-cache: true

      - name: Set up Python
        run: uv python install 3.12

      - name: Install dependencies
        run: uv sync --frozen

      - name: Lint
        run: |
          uv run ruff check .
          uv run ruff format --check .

      - name: Type check
        run: uv run mypy api/

      - name: Test
        run: uv run pytest --cov=api --cov-report=term-missing -v

      - name: Build Docker image
        run: docker build -t retail-api:${{ github.sha }} .
```

**Reading it:**

- **`on:`** — when it runs. `pull_request` is the important one: checks run *before* merging.
- **`runs-on`** — a fresh Ubuntu VM, every time. That's the point: it can't depend on anything on your laptop.
- **`steps`** — run in order; the job fails at the first failure.

### Branch protection — where CI gets teeth

CI that reports failure and lets you merge anyway is decoration.

**Settings → Branches → Add rule** on `main`:
- ✅ Require status checks to pass before merging
- ✅ Require branches to be up to date before merging

Now a failing test **physically blocks the merge button.** That's the mechanism that keeps `main` working.

> **What to gate on:** tests passing, lint passing. Not "coverage above 90%" — that just gets gamed with meaningless tests. Coverage is a diagnostic, not a target.

### Secrets in CI

```yaml
      - name: Integration tests
        env:
          AZURE_SQL_URL: ${{ secrets.AZURE_SQL_URL }}
        run: uv run pytest tests/integration/
```

Set them in **Settings → Secrets and variables → Actions**. They're masked in logs. Never in the YAML.

> Secrets aren't available to workflows triggered by pull requests from forks — deliberately, so a stranger's PR can't exfiltrate your keys. Keep unit tests secret-free so PR checks always work.

---

## 8. Linting and formatting

```bash
uv add --dev ruff mypy pytest pytest-cov pytest-asyncio
```

**Ruff** replaces black, isort, flake8 and more, and is roughly 100× faster:

```toml
# pyproject.toml
[tool.ruff]
line-length = 100
target-version = "py312"

[tool.ruff.lint]
select = ["E", "F", "I", "UP", "B"]   # errors, pyflakes, imports, pyupgrade, bugbear

[tool.pytest.ini_options]
testpaths = ["tests"]
asyncio_mode = "auto"

[tool.mypy]
python_version = "3.12"
ignore_missing_imports = true
```

```bash
uv run ruff format .        # format
uv run ruff check --fix .   # fix what it can
uv run mypy api/            # type check
```

> **Formatting arguments are a waste of everyone's time.** Run a formatter, commit the result, never discuss it again. `ruff format` on save in your editor and it stops being a thing you think about.

---

## 9. What to test — and what not to

A test suite that takes twenty minutes doesn't get run. Aim for **under a minute** for unit tests.

| Priority | Test |
|---|---|
| **High** | Business logic — cleaning rules, revenue calculation, RFM computation |
| **High** | Edge cases — empty input, nulls, single row, negative numbers |
| **High** | API contracts — status codes, validation, response shape |
| **High** | The ONNX ↔ PyTorch equivalence check |
| **High** | No-time-leakage in the train/val split |
| Medium | Integration: one real database round trip, marked slow |
| Low | Trivial getters, config loading |
| **Never** | Third-party libraries. Pandas is already tested. |

### The test pyramid

```
        ╱ E2E ╲          few  — slow, brittle, but they catch real integration bugs
      ╱ Integr. ╲        some — real DB, real files
    ╱   Unit      ╲      many — fast, isolated, run constantly
```

Most of your tests should be unit tests. Mark the slow ones so they can be skipped locally:

```python
import pytest

@pytest.mark.slow
def test_full_pipeline_against_real_database():
    ...
```

```bash
pytest -m "not slow"       # fast loop while developing
pytest                     # everything, in CI
```

### Coverage

```bash
pytest --cov=api --cov-report=html      # then open htmlcov/index.html
```

Use it to **find untested code**, not as a score to hit. 100% coverage with meaningless assertions is worse than 60% with good ones — it gives false confidence.

---

## When something goes wrong

| Symptom | Likely cause | Fix |
|---|---|---|
| `ModuleNotFoundError: api` | Package not installed or wrong rootdir | `uv sync`; add `testpaths` to `pyproject.toml` |
| Mock never takes effect | Patched where defined, not where used | `@patch("module_under_test.thing")` |
| Tests pass alone, fail together | Shared state between tests | Use function-scoped fixtures; clear `dependency_overrides` |
| Spark tests take minutes | Session per test, or 200 shuffle partitions | `scope="session"`; `spark.sql.shuffle.partitions=2` |
| Async test never runs / warns | `pytest-asyncio` missing or not configured | Install it; `asyncio_mode = "auto"` |
| Float assertion fails by 1e-17 | Floating point | `pytest.approx()` |
| ML test fails randomly | Asserting an exact metric | Assert a threshold; set a seed |
| CI passes locally, fails on GitHub | Local env has something the CI VM doesn't | Pin deps with a lockfile; commit `uv.lock` |
| CI can't reach the database | Secrets not set, or IP not allowed | Add secrets; mock instead for unit tests |
| Failing check but merge still allowed | No branch protection | Enable "require status checks" |
| Everyone ignores the test suite | It's slow, or it's flaky | Speed it up; delete or fix flaky tests immediately |

---

## Practice checklist

- [ ] `pytest` basics: `assert`, naming conventions, useful flags
- [ ] Arrange / Act / Assert structure
- [ ] `parametrize` for edge cases
- [ ] Fixtures and `conftest.py`; fixture scope
- [ ] Mocking external calls (Azure SQL, ADLS) — and **patching where it's used**
- [ ] FastAPI `dependency_overrides` as a cleaner alternative to mocking
- [ ] Testing a FastAPI app with `TestClient` — no live server needed
- [ ] Testing a PySpark job on a small local sample instead of the full dataset
- [ ] **Separating pure transformations from I/O so code is testable at all**
- [ ] Testing ML code: shapes, gradient flow, ONNX equivalence, leakage, baseline comparison
- [ ] Never asserting exact metrics
- [ ] GitHub Actions: a workflow file that runs tests on every push
- [ ] **Branch protection** — what actually gives CI teeth
- [ ] Ruff for lint/format, mypy for types
- [ ] The test pyramid, marking slow tests, and using coverage as a diagnostic

## Hands-on

- [ ] Write tests for the FastAPI endpoints you built in stage 4, including a 422 and a 404
- [ ] Write a parametrized test with at least 4 cases for one cleaning function
- [ ] Refactor a Spark job so the transformation is a pure `DataFrame → DataFrame` function, then test it
- [ ] Write the ONNX-vs-PyTorch equivalence test
- [ ] Write a test that fails if the train/val split leaks future data
- [ ] Add a GitHub Actions workflow that runs the test suite on every pull request
- [ ] Turn on branch protection and try to merge a PR with a failing test

## Resources

- [pytest docs](https://docs.pytest.org/)
- [FastAPI: Testing](https://fastapi.tiangolo.com/tutorial/testing/)
- [GitHub Actions docs](https://docs.github.com/actions)
- [Ruff docs](https://docs.astral.sh/ruff/)

## Next

[[Data modeling]]
