---
tags: [python, language, modules, packaging]
---

# Modules and packages (Python)

Language: [[Python]] · Tooling: [[Packaging and environments]]

---

## Importing

```python
import math
import numpy as np
from pathlib import Path
from mypackage.module import thing
from . import sibling                # relative - inside a package only
from .module import thing
```

```python
import pyspark.sql.functions as F     # ✅ the convention
from pyspark.sql.functions import *   # ✗ shadows sum, min, max, round
```

> **Never `import *`.** It silently shadows builtins and you get baffling errors ([[PySpark reference]]).

## Package layout

```
myproject/
├── pyproject.toml
├── src/
│   └── mypackage/
│       ├── __init__.py         # marks it a package; exports the public API
│       ├── config.py
│       └── pipelines/
│           ├── __init__.py
│           └── etl.py
└── tests/
    └── test_etl.py
```

```python
# src/mypackage/__init__.py
from .config import Settings
from .pipelines.etl import run

__all__ = ["Settings", "run"]        # what `from mypackage import *` gives
__version__ = "1.0.0"
```

> **Use a `src/` layout.** It stops Python accidentally importing your package from the working directory instead of the installed one — which hides packaging bugs until deployment.

## Script vs import

```python
def main():
    ...

if __name__ == "__main__":     # only runs when executed directly
    main()
```

```bash
python -m mypackage.cli        # run a module as a script
```

> **Required on Windows** if you use `multiprocessing` or a DataLoader with `num_workers > 0` ([[Tensors, autograd and the training loop]]).

## Circular imports

```python
# a.py imports b, b.py imports a -> ImportError
```

Fixes, in order of preference: move the shared thing to a third module; import inside the function rather than at module level; restructure so the dependency only goes one way.

## Related

[[Python]] · [[Packaging and environments]] · [[Testing (Python)]]
