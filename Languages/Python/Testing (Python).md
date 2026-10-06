---
tags: [python, testing, pytest]
---

# Testing (Python)

Language: [[Python]] · Full note: [[pytest]] · Practices: [[Testing and CI-CD]]

---

## The minimum

```python
# tests/test_transforms.py
import pytest
from mypackage.transforms import calculate_revenue

def test_calculate_revenue():
    assert calculate_revenue(quantity=6, unit_price=2.55) == pytest.approx(15.30)
```

```bash
uv run pytest -v
```

Conventions pytest relies on: files named `test_*.py`, functions named `test_*`, plain `assert`.

## Arrange, Act, Assert

```python
def test_drops_cancelled_invoices():
    rows = [{"invoice_no": "536365"}, {"invoice_no": "C536367"}]   # Arrange
    result = clean_invoices(rows)                                   # Act
    assert len(result) == 1                                         # Assert
```

## The features you'll use daily

```python
import pytest

@pytest.mark.parametrize("value,expected", [(1, True), (0, False), (-1, False)])
def test_is_positive(value, expected):
    assert is_positive(value) == expected

def test_raises():
    with pytest.raises(ValueError, match="must be positive"):
        forecast(weeks=-1)

def test_floats():
    assert 0.1 + 0.2 == pytest.approx(0.3)     # never == on floats
```

```python
import pandas as pd
import pytest

@pytest.fixture
def sample_df():
    return pd.DataFrame({"qty": [1, 2], "price": [1.0, 2.0]})

def test_revenue(sample_df):                    # request it by name
    assert calculate_total(sample_df) == 5.0
```

## Structuring for testability

```python
import pandas as pd

# ✗ untestable - I/O baked in
def run():
    df = pd.read_csv("data.csv")
    cleaned = df[df.qty > 0]
    cleaned.to_parquet("out.parquet")

# ✓ pure function, testable on 3 rows
def clean(df: pd.DataFrame) -> pd.DataFrame:
    return df[df.qty > 0]

def run():                                      # thin wrapper, not unit-tested
    clean(pd.read_csv("data.csv")).to_parquet("out.parquet")
```

> **This is the main lesson.** Separating pure logic from I/O makes code testable, reusable and easier to read. If something is hard to test, that's information about the code, not about testing.

## Running

```bash
pytest -v                  # verbose
pytest -x                  # stop at first failure
pytest -k "revenue"        # match by name
pytest --lf                # last failed only
pytest -m "not slow"       # by marker
pytest --cov=src --cov-report=term-missing
```

## Related

[[pytest]] — the full reference · [[Testing and CI-CD]] · [[Python]] · [[Packaging and environments]]
