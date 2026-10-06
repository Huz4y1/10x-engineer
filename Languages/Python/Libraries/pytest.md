---
tags: [python, library, pytest, testing]
---

# pytest

Index: [[Python libraries]] · Practices: [[Testing and CI-CD]] · Basics: [[Testing (Python)]]

### Installing

> **Same commands on Windows, WSL and Linux.** The differences are called out below where they exist.

```bash
uv add --dev pytest pytest-cov pytest-asyncio pytest-mock                      # in a uv project - PREFERRED
uv pip install --dev pytest pytest-cov pytest-asyncio pytest-mock              # into the active venv, no project file
pip install --dev pytest pytest-cov pytest-asyncio pytest-mock                 # plain pip, if you're not using uv
```

**If you don't have a project yet:**

```bash
uv init myproject && cd myproject     # creates pyproject.toml and .venv
uv add --dev pytest pytest-cov pytest-asyncio pytest-mock
uv run python -c "import pytest; print(pytest.__version__)"           # uv run uses the project venv automatically
```

**Without uv — a plain virtual environment:**

| | Windows (PowerShell) | WSL / Linux |
|---|---|---|
| Create | `python -m venv .venv` | `python3 -m venv .venv` |
| Activate | `.venv\Scripts\Activate.ps1` | `source .venv/bin/activate` |
| Install | `pip install --dev pytest pytest-cov pytest-asyncio pytest-mock` | `pip install --dev pytest pytest-cov pytest-asyncio pytest-mock` |
| Leave | `deactivate` | `deactivate` |

**Check it worked:**

```bash
python -c "import pytest; print(pytest.__version__)"
```

> **`--dev` keeps test tools out of your production image.** They belong in the dev dependency group, not `dependencies`.

```bash
uv run pytest                     # run tests using the project venv
```

> ⚠️ **If `pytest` runs but can't import your package**, you haven't installed your own project. `uv run pytest` handles it; with plain pip use `pip install -e .`.

> ⚠️ **Never `pip install` outside a virtual environment.** On WSL and Linux it fights the system Python; newer versions refuse outright with *externally-managed-environment*. On Windows it silently pollutes your global install. `uv` makes a venv for you — use it.

> **On Windows: if `python` opens the Microsoft Store**, Python isn't really installed. Get it from python.org (tick *Add to PATH*), or `winget install Python.Python.3.12`.

> **In WSL: work in `~/`, not `/mnt/c/`.** Installing into a project on the Windows drive is dramatically slower, and file-watching (`--reload`, `pytest-watch`) often doesn't fire at all. See [[Setting up a dev machine]].

---

## Conventions

Files `test_*.py`, functions `test_*`, plain `assert`. No classes, no `self.assertEqual`.

```python
import pytest

def test_revenue():
    assert calculate(6, 2.55) == pytest.approx(15.30)
```

## Running

```bash
pytest                     # everything
pytest -v                  # show each test name
pytest -q                  # quiet
pytest -x                  # stop at first failure
pytest --maxfail=3
pytest -k "revenue and not slow"     # match by name
pytest -m "not slow"                 # by marker
pytest --lf                          # last failed only
pytest --ff                          # failed first
pytest tests/test_api.py::test_health   # one test
pytest -s                            # don't capture stdout (see prints)
pytest --durations=10                # 10 slowest tests
pytest -n auto                       # parallel (needs pytest-xdist)
pytest --cov=src --cov-report=term-missing
pytest --cov=src --cov-report=html   # then open htmlcov/index.html
```

## Assertions

```python
import pytest

assert x == 5
assert x in [1,2,3]
assert isinstance(x, dict)
assert 0.1 + 0.2 == pytest.approx(0.3)
assert result == pytest.approx([1.0, 2.0], rel=1e-3)

with pytest.raises(ValueError, match="must be positive"):
    forecast(weeks=-1)

with pytest.raises(ValueError) as exc:
    f()
assert "positive" in str(exc.value)

with pytest.warns(DeprecationWarning):
    old_function()
```

> **`pytest.approx` for every float comparison.** `0.1 + 0.2 != 0.3` ([[24 — SCIENTIFIC COMPUTING]]).

## Parametrize — the highest value-per-line feature

```python
import pytest

@pytest.mark.parametrize("value,expected", [
    ("536365",  True),
    ("C536367", False),      # cancellation
    ("c536368", False),      # lowercase - do we handle it?
    ("",        False),      # empty
])
def test_is_valid(value, expected):
    assert is_valid(value) == expected
```

```python
import pytest

@pytest.mark.parametrize("a", [1, 2])
@pytest.mark.parametrize("b", [10, 20])          # stacking = 4 combinations
def test_pairs(a, b): ...

@pytest.mark.parametrize("x,expected", [
    pytest.param(1, 2, id="simple"),
    pytest.param(0, 0, marks=pytest.mark.xfail(reason="known bug")),
])
def test_ids(x, expected): ...
```

> **Four separate tests, one function, and each failure names the exact case.** Use it for every function with edge cases.

## Fixtures

```python
# conftest.py - pytest finds this automatically, no import needed
import pandas as pd
import pytest

@pytest.fixture
def sample_df():
    return pd.DataFrame({"qty": [1,2], "price": [1.0, 2.0]})

def test_revenue(sample_df):              # request by name
    assert total(sample_df) == 5.0
```

### Setup and teardown

```python
import pytest

@pytest.fixture
def db_connection():
    conn = connect()
    yield conn                            # test runs here
    conn.close()                          # teardown, always runs
```

### Scope

```python
from pyspark.sql import SparkSession
import pytest

@pytest.fixture(scope="session")          # once for the whole run
def spark():
    s = (SparkSession.builder.master("local[2]")
         .config("spark.sql.shuffle.partitions", "2")   # 200 default = slow tests
         .getOrCreate())
    yield s
    s.stop()
```

| Scope | Rebuilt |
|---|---|
| `function` (default) | Every test — safest |
| `class` / `module` | Per class / file |
| `session` | Once — for expensive things like Spark |

> **`spark.sql.shuffle.partitions=2` is the difference between a 3-second and a 90-second test suite** ([[PySpark reference]]).

### Parametrized and factory fixtures

```python
import pytest

@pytest.fixture(params=["postgres", "sqlite"])
def db(request):
    return connect(request.param)         # test runs once per param

@pytest.fixture
def make_user():
    def _make(name="test", age=30):
        return User(name=name, age=age)
    return _make                          # a FACTORY - call it in the test

def test_x(make_user):
    u = make_user(name="alice")
```

### Built-in fixtures

```python
def test_files(tmp_path):                 # a fresh temp directory
    (tmp_path / "f.csv").write_text("a,b")

def test_env(monkeypatch):
    monkeypatch.setenv("API_KEY", "test")
    monkeypatch.setattr("mymod.CONSTANT", 5)
    monkeypatch.chdir(tmp_path)

def test_output(capsys):
    print("hi")
    assert "hi" in capsys.readouterr().out

def test_logs(caplog):
    do_thing()
    assert "warning" in caplog.text
```

## Markers

```python
import sys
import pytest

@pytest.mark.slow
def test_full_pipeline(): ...

@pytest.mark.skip(reason="not implemented")
@pytest.mark.skipif(sys.platform == "win32", reason="posix only")
@pytest.mark.xfail(reason="known bug", strict=True)
```

```toml
# pyproject.toml
[tool.pytest.ini_options]
testpaths = ["tests"]
markers = ["slow: long-running tests"]
addopts = "-v --strict-markers"
asyncio_mode = "auto"
```

```bash
pytest -m "not slow"       # fast loop while developing
pytest                     # everything, in CI
```

## Mocking

```python
from unittest.mock import patch, MagicMock

@patch("api.services.sales.fetch_from_db")        # patch where it's USED
def test_forecast(mock_fetch):
    mock_fetch.return_value = [{"revenue": 100.0}]
    assert get_forecast("Kitchen") is not None
    mock_fetch.assert_called_once_with(category="Kitchen")
```

> ⚠️ **Patch where it's used, not where it's defined.** If `api/services/sales.py` does `from api.db import fetch`, patch `"api.services.sales.fetch"`. Patching `"api.db.fetch"` silently does nothing — the test passes and proves nothing.

```python
def test_with_mocker(mocker):             # pytest-mock, cleaner
    m = mocker.patch("mymod.api_call", return_value={"ok": True})
```

### FastAPI — override instead of patching

```python
app.dependency_overrides[get_db] = fake_db
# ... tests ...
app.dependency_overrides.clear()
```

> **Cleaner than `patch`** — type-safe, no string paths. This is why dependency injection is worth using ([[FastAPI reference]]).

## Async tests

```python
from httpx import ASGITransport, AsyncClient
import pytest

@pytest.mark.asyncio
async def test_predict():
    async with AsyncClient(transport=ASGITransport(app=app), base_url="http://t") as c:
        r = await c.post("/predict", json={"id": "1"})
    assert r.status_code == 200
```

Needs `pytest-asyncio` and `asyncio_mode = "auto"`.

## What to test

| Priority | Test |
|---|---|
| **High** | Business logic, edge cases, API contracts |
| **High** | The ONNX ↔ PyTorch equivalence check |
| **High** | No time leakage in a train/val split |
| Medium | One real integration test, marked slow |
| **Never** | Third-party libraries — pandas is already tested |

> **Keep the unit suite under a minute.** A slow suite doesn't get run.

## Troubleshooting

| Symptom | Fix |
|---|---|
| `ModuleNotFoundError` | `uv sync`; add `testpaths` to pyproject |
| Mock never applies | Patch where used, not defined |
| Passes alone, fails together | Shared state — use function-scoped fixtures |
| Spark tests take minutes | `scope="session"` + `shuffle.partitions=2` |
| Async test warns / skipped | Install `pytest-asyncio`, set `asyncio_mode` |
| Float assertion off by 1e-17 | `pytest.approx` |

## Related

[[Python libraries]] · [[Testing (Python)]] · [[Testing and CI-CD]] · [[FastAPI reference]] · [[CI-CD pipelines]]
