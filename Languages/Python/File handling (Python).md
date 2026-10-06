---
tags: [python, language, files, io]
---

# File handling (Python)

Language: [[Python]]

---

## Paths — use `pathlib`

```python
from pathlib import Path

p = Path("data") / "raw" / "sales.csv"       # / works on every OS
p.exists() · p.is_file() · p.is_dir()
p.name          # "sales.csv"
p.stem          # "sales"
p.suffix        # ".csv"
p.parent        # Path("data/raw")
p.absolute() · p.resolve()

p.parent.mkdir(parents=True, exist_ok=True)
p.unlink(missing_ok=True)                    # delete
p.rename(new_path)

list(Path("data").glob("*.csv"))
list(Path("data").rglob("*.csv"))            # recursive

p.read_text(encoding="utf-8")
p.write_text("content", encoding="utf-8")
p.read_bytes() · p.write_bytes(b"...")
```

> **Use `pathlib`, not `os.path` string concatenation.** It handles separators, and `Path("a") / "b"` is unambiguous on Windows and Linux alike.

## Reading and writing

```python
with open(path, "r", encoding="utf-8") as f:      # ALWAYS specify encoding
    content = f.read()
    # or
    for line in f:                                 # streams - doesn't load it all
        process(line.rstrip("\n"))

with open(path, "w", encoding="utf-8") as f:
    f.write("text")
    f.writelines(lines)
```

| Mode | Does |
|---|---|
| `"r"` | Read (default) |
| `"w"` | Write — **truncates the file** |
| `"a"` | Append |
| `"x"` | Create, fail if it exists |
| `"rb"` / `"wb"` | Binary |
| `"r+"` | Read and write |

> **Always `with open(...)`.** It closes the file even if an exception is raised. And **always pass `encoding="utf-8"`** — the default is platform-dependent, which is how you get mojibake on Windows.

> **Iterate the file object for large files.** `f.read()` loads the whole thing into memory.

## CSV

```python
import csv

with open(path, newline="", encoding="utf-8") as f:
    for row in csv.DictReader(f):
        print(row["invoice_no"])

with open(path, "w", newline="", encoding="utf-8") as f:
    w = csv.DictWriter(f, fieldnames=["a", "b"])
    w.writeheader()
    w.writerows(rows)
```

> `newline=""` is required, or you get blank lines between rows on Windows.

For data work, use [[pandas]] or [[Polars]] rather than the `csv` module.

## Temporary files

```python
import tempfile

with tempfile.NamedTemporaryFile(suffix=".csv", delete=False) as f:
    f.write(b"data")
    temp_path = f.name

with tempfile.TemporaryDirectory() as d:
    ...                                    # deleted on exit
```

## Compressed and binary

```python
import gzip, pickle, json

with gzip.open("f.gz", "rt", encoding="utf-8") as f: ...

import shutil
shutil.copy(src, dst) · shutil.move(src, dst) · shutil.rmtree(dir)
shutil.make_archive("out", "zip", "folder")
```

> ⚠️ **`pickle.load` executes arbitrary code.** Only unpickle files you created. For data interchange use JSON, Parquet or [[Model export and serving|ONNX]]. See [[26 — SECURITY]].

## Related

[[Python]] · [[JSON (Python)]] · [[pandas]] · [[Polars]]
