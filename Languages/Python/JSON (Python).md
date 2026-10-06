---
tags: [python, json, serialisation]
---

# JSON (Python)

Language: [[Python]] · The format: [[JSON]]

---

```python
import json

json.dumps(obj)                              # -> str
json.dumps(obj, indent=2, sort_keys=True)
json.dumps(obj, default=str)                 # fallback for dates etc.
json.loads(text)                             # str -> object

with open(p, "w", encoding="utf-8") as f:
    json.dump(obj, f, indent=2)

with open(p, encoding="utf-8") as f:
    obj = json.load(f)
```

## Type mapping

| Python | JSON |
|---|---|
| `dict` | object |
| `list`, `tuple` | array |
| `str` | string |
| `int`, `float` | number |
| `True` / `False` | `true` / `false` |
| `None` | `null` |

## What doesn't serialise

```python
from datetime import datetime
import json

json.dumps({"d": datetime.now()})            # TypeError

json.dumps(obj, default=str)                 # quick fix
```

```python
from datetime import datetime
from decimal import Decimal
import json

class Encoder(json.JSONEncoder):
    def default(self, o):
        if isinstance(o, (datetime, date)): return o.isoformat()
        if isinstance(o, Decimal):          return str(o)
        if isinstance(o, Path):             return str(o)
        if isinstance(o, set):              return list(o)
        return super().default(o)

json.dumps(obj, cls=Encoder)
```

> **`Decimal` -> `str`, not `float`.** Converting money to float reintroduces the precision problem you used `Decimal` to avoid.

## Faster alternatives

```python
import orjson                                # 5-10x faster
orjson.dumps(obj)                            # returns BYTES
orjson.loads(data)
```

## JSON Lines — for data pipelines

```python
import json

with open(p, "w", encoding="utf-8") as f:
    for row in rows:
        f.write(json.dumps(row) + "\n")
```

> **JSONL is what you want for pipelines** — append-friendly, streamable line by line, and readable by [[PySpark reference|Spark]] and [[Polars]] directly.

## Validating JSON

```python
from pydantic import BaseModel

class Reading(BaseModel):
    device_id: str
    temp: float

reading = Reading.model_validate_json(raw)    # parse AND validate
```

> **Parsing is not validating.** `json.loads` gives you a dict of unknown shape. [[Pydantic]] gives you a checked object — use it at every trust boundary.

## Related

[[Python]] · [[Pydantic]] · [[JSON]] · [[File handling (Python)]]
