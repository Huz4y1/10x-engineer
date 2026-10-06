---
tags: [python, library, pydantic, validation]
---

# Pydantic

Validation and settings from type hints. **The trust boundary of every Python app in this vault.**

Index: [[Python libraries]] · Used by: [[FastAPI reference]]

### Installing

> **Same commands on Windows, WSL and Linux.** The differences are called out below where they exist.

```bash
uv add pydantic pydantic-settings                      # in a uv project - PREFERRED
uv pip install pydantic pydantic-settings              # into the active venv, no project file
pip install pydantic pydantic-settings                 # plain pip, if you're not using uv
```

**If you don't have a project yet:**

```bash
uv init myproject && cd myproject     # creates pyproject.toml and .venv
uv add pydantic pydantic-settings
uv run python -c "import pydantic; print(pydantic.VERSION)"           # uv run uses the project venv automatically
```

**Without uv — a plain virtual environment:**

| | Windows (PowerShell) | WSL / Linux |
|---|---|---|
| Create | `python -m venv .venv` | `python3 -m venv .venv` |
| Activate | `.venv\Scripts\Activate.ps1` | `source .venv/bin/activate` |
| Install | `pip install pydantic pydantic-settings` | `pip install pydantic pydantic-settings` |
| Leave | `deactivate` | `deactivate` |

**Check it worked:**

```bash
python -c "import pydantic; print(pydantic.VERSION)"
```

> **`pydantic-core` is written in Rust but ships prebuilt wheels** — you don't need a Rust toolchain on Windows or WSL.

> ⚠️ **Check you're on v2.** Most tutorials online are v1, and the API changed: `.dict()` became `.model_dump()`, `@validator` became `@field_validator`.

> ⚠️ **Never `pip install` outside a virtual environment.** On WSL and Linux it fights the system Python; newer versions refuse outright with *externally-managed-environment*. On Windows it silently pollutes your global install. `uv` makes a venv for you — use it.

> **On Windows: if `python` opens the Microsoft Store**, Python isn't really installed. Get it from python.org (tick *Add to PATH*), or `winget install Python.Python.3.12`.

> **In WSL: work in `~/`, not `/mnt/c/`.** Installing into a project on the Windows drive is dramatically slower, and file-watching (`--reload`, `pytest-watch`) often doesn't fire at all. See [[Setting up a dev machine]].

---

## The idea

You'd otherwise write twenty lines of `if not isinstance(...)` checks per function. Pydantic turns the type hints you'd write anyway into validation, parsing and documentation.

```python
from pydantic import BaseModel, Field

class Reading(BaseModel):
    device_id: str = Field(..., min_length=1)
    temp_c: float = Field(..., ge=-40, le=150)
    tags: list[str] = Field(default_factory=list)

Reading(device_id="1", temp_c=42.1)       # ✓
Reading(device_id="", temp_c=999)          # ✗ ValidationError, both fields named
```

> **Use it at every trust boundary** — API input, config, files, message payloads. Anywhere data enters your program from outside.

## Fields

```python
from pydantic import BaseModel, Field, EmailStr, HttpUrl
from datetime import datetime
from decimal import Decimal
from enum import Enum

class Status(str, Enum):
    active = "active"
    archived = "archived"

class Item(BaseModel):
    id: int
    name: str = Field(..., min_length=1, max_length=100)
    price: Decimal = Field(..., gt=0, decimal_places=2)     # money -> Decimal
    qty: int = Field(default=0, ge=0)
    tags: list[str] = Field(default_factory=list)           # NOT default=[]
    status: Status = Status.active
    email: EmailStr | None = None
    url: HttpUrl | None = None
    created: datetime
    secret: str = Field(..., repr=False)                    # hidden from repr
    alias_field: str = Field(..., alias="camelCaseName")
```

| Constraint | For |
|---|---|
| `gt` `ge` `lt` `le` | Numbers |
| `min_length` `max_length` | Strings, lists |
| `pattern` | Regex |
| `decimal_places` `max_digits` | Decimal |
| `default_factory` | **Mutable defaults** |

> **`default_factory=list`, never `default=[]`.** A mutable default is created once and shared by every instance ([[Variables (Python)]]).

## Validators

```python
from pydantic import field_validator, model_validator, BaseModel

class Booking(BaseModel):
    code: str
    start: date
    end: date

    @field_validator("code")
    @classmethod
    def upper(cls, v: str) -> str:
        return v.upper()

    @field_validator("code", mode="before")       # BEFORE type coercion
    @classmethod
    def strip(cls, v):
        return v.strip() if isinstance(v, str) else v

    @model_validator(mode="after")                # cross-field, after all set
    def dates_ordered(self):
        if self.end <= self.start:
            raise ValueError("end must be after start")
        return self
```

| Mode | Sees | Use for |
|---|---|---|
| `before` | Raw input | Cleaning, coercion |
| `after` (default) | Typed value | Business rules |
| `model_validator(mode="after")` | The whole model | Cross-field rules |

## Methods

```python
m.model_dump()                    # -> dict
m.model_dump(exclude={"secret"}, exclude_none=True, exclude_unset=True)
m.model_dump_json()
m.model_copy(update={"qty": 5})

Item.model_validate(some_dict)
Item.model_validate_json(raw_json)          # parse AND validate in one step
Item.model_json_schema()
```

> **`model_validate_json` beats `json.loads` + construct** — one pass, and it validates ([[JSON (Python)]]).

## Nested models

```python
from pydantic import BaseModel

class Address(BaseModel):
    city: str
    postcode: str

class User(BaseModel):
    name: str
    address: Address                    # nested, validated recursively
    addresses: list[Address] = []
```

## Settings — config from the environment

```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    env: str = "local"
    database_url: str                   # REQUIRED - crashes at startup if missing
    api_key: str
    log_level: str = "INFO"
    cors_origins: list[str] = []

    model_config = {"env_file": ".env", "env_file_encoding": "utf-8"}

settings = Settings()
```

> **Crashing at startup on a missing variable is a feature.** The alternative is a 500 at 3am from a `None` config value. This is the pattern used in every stack in [[Pipeline setup - overview]].

## Reading the error

```json
{"detail":[{"type":"greater_than","loc":["body","price"],
            "msg":"Input should be greater than 0","input":-5}]}
```

> **`loc` names the exact field.** When you see a 422 from [[FastAPI reference|FastAPI]], read `loc` first.

## Custom reusable types

```python
from typing import Annotated
from pydantic import AfterValidator, Field

def must_be_upper(v: str) -> str:
    if not v.isupper(): raise ValueError("must be uppercase")
    return v

CountryCode = Annotated[str, Field(min_length=2, max_length=2), AfterValidator(must_be_upper)]
```

## Strict vs lax

```python
from pydantic import BaseModel

class M(BaseModel):
    n: int

M(n="5")                              # ✓ coerced to 5 by default

class M(BaseModel):
    model_config = {"strict": True}
    n: int

M(n="5")                              # ✗ ValidationError
```

## v1 → v2

| v1 | v2 |
|---|---|
| `.dict()` | `.model_dump()` |
| `.json()` | `.model_dump_json()` |
| `parse_obj()` | `model_validate()` |
| `@validator` | `@field_validator` + `@classmethod` |
| `@root_validator` | `@model_validator` |
| `class Config` | `model_config = {...}` |

> **Most search results are v1.** If a snippet warns "deprecated", that's why.

## Pydantic vs dataclass

| | `@dataclass` | `BaseModel` |
|---|---|---|
| Validates | ✗ | ✅ |
| Parses/coerces | ✗ | ✅ |
| JSON schema | ✗ | ✅ |
| Speed | Faster | Fast (Rust core) |
| Use for | Internal data | **Anything from outside** |

## Related

[[Python libraries]] · [[FastAPI reference]] · [[Type hints]] · [[Classes (Python)]] · [[JSON (Python)]]
