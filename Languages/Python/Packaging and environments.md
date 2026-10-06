---
tags: [python, packaging, uv, tooling]
---

# Packaging and environments

Language: [[Python]] · Deployment: [[Docker deep dive]]

---

## The problem

Project A needs `pandas 1.5`. Project B needs `pandas 2.2`. Install globally and one of them breaks — every time.

A **virtual environment** is a private box of packages for one project.

## Use uv

There are six tools that do this. Pick one and stop thinking about it. **uv** is the current best — 10-100× faster than pip, and it replaces `pip`, `venv`, `virtualenv`, `pip-tools` and `pyenv`.

```bash
uv init myproject && cd myproject
uv add pandas fastapi              # installs AND records the dependency
uv add --dev pytest ruff mypy      # dev-only
uv remove pandas
uv sync                            # recreate the exact env from the lockfile
uv sync --frozen                   # fail if the lockfile is stale (use in CI/Docker)
uv run python main.py              # runs inside the env - no "activate" needed
uv run pytest
uv lock --upgrade                  # update the lockfile
uv python install 3.12             # manages Python versions too
uv tool install ruff               # global CLI tools
```

## The two files that matter

```toml
# pyproject.toml - what you ASKED for (human-written)
[project]
name = "myproject"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = ["pandas>=2.0", "fastapi>=0.110"]

[dependency-groups]
dev = ["pytest>=8", "ruff", "mypy"]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.ruff]
line-length = 100
target-version = "py312"

[tool.ruff.lint]
select = ["E", "F", "I", "UP", "B"]     # errors, pyflakes, imports, pyupgrade, bugbear

[tool.pytest.ini_options]
testpaths = ["tests"]
asyncio_mode = "auto"

[tool.mypy]
python_version = "3.12"
ignore_missing_imports = true
```

`uv.lock` — what you **actually got**, every package pinned including transitive dependencies. Machine-written.

> **Commit both.** `pyproject.toml` is intent; `uv.lock` is reproducibility. The lockfile is the difference between "works on my machine" and "works on the CI runner, in Docker, and in six months' time".

## Version specifiers

```
pandas>=2.0          # at least
pandas~=2.1.0        # 2.1.x only - compatible release
pandas==2.1.3        # exact
pandas>=2.0,<3.0     # range
```

## Linting and formatting — ruff

```bash
uv run ruff check .            # lint
uv run ruff check --fix .      # autofix
uv run ruff format .           # format (replaces black)
```

> **Run a formatter, commit the result, never discuss formatting again.** Set `ruff format` on save in your editor and it stops being a thing you think about.

## The older tools

If you inherit a project using them:

```bash
python -m venv .venv
source .venv/bin/activate          # Linux/macOS
.venv\Scripts\activate             # Windows
pip install -r requirements.txt
pip freeze > requirements.txt
```

> `pip freeze` is **not** a lockfile — it captures whatever happens to be installed, including things you removed from your code. Prefer uv or Poetry.

## In Docker

```dockerfile
COPY pyproject.toml uv.lock ./
RUN pip install --no-cache-dir uv && uv sync --frozen --no-dev
COPY src/ ./src/
```

> **Dependency files before code** — that's the layer-caching rule from [[Docker deep dive]]. Copy code first and every edit reinstalls PyTorch.

## Related

[[Python]] · [[Modules and packages (Python)]] · [[Testing (Python)]] · [[Docker deep dive]] · [[Dev environment - Git, Docker, CLI]]
