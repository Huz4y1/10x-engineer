---
tags: [python, library, matplotlib, plotly, visualisation]
---

# Matplotlib and Plotly

Index: [[Python libraries]] · In dashboards: [[Streamlit]]

### Installing

> **Same commands on Windows, WSL and Linux.** The differences are called out below where they exist.

```bash
uv add matplotlib plotly                      # in a uv project - PREFERRED
uv pip install matplotlib plotly              # into the active venv, no project file
pip install matplotlib plotly                 # plain pip, if you're not using uv
```

**If you don't have a project yet:**

```bash
uv init myproject && cd myproject     # creates pyproject.toml and .venv
uv add matplotlib plotly
uv run python -c "import matplotlib, plotly; print(matplotlib.__version__, plotly.__version__)"           # uv run uses the project venv automatically
```

**Without uv — a plain virtual environment:**

| | Windows (PowerShell) | WSL / Linux |
|---|---|---|
| Create | `python -m venv .venv` | `python3 -m venv .venv` |
| Activate | `.venv\Scripts\Activate.ps1` | `source .venv/bin/activate` |
| Install | `pip install matplotlib plotly` | `pip install matplotlib plotly` |
| Leave | `deactivate` | `deactivate` |

**Check it worked:**

```bash
python -c "import matplotlib, plotly; print(matplotlib.__version__, plotly.__version__)"
```

> ⚠️ **In WSL, `plt.show()` opens nothing by default** — there's no display attached. Three options:
> - **Save to a file instead:** `plt.savefig("out.png")` — the right answer on a server
> - **Use WSLg** (Windows 11) — GUI windows work with no setup; if not, `sudo apt install python3-tk`
> - **Work in a notebook or [[Streamlit]]**, where figures render inline
>
> To make headless explicit and stop it hanging:
> ```python
> import matplotlib
> matplotlib.use("Agg")            # file output only, no GUI - before importing pyplot
> ```

> **Plotly needs no display at all** — it renders as HTML in a browser, notebook or Streamlit. That's why it's the better default when you're working in WSL.

```bash
uv add kaleido                    # only needed for fig.write_image() to PNG/PDF
```

> ⚠️ **Never `pip install` outside a virtual environment.** On WSL and Linux it fights the system Python; newer versions refuse outright with *externally-managed-environment*. On Windows it silently pollutes your global install. `uv` makes a venv for you — use it.

> **On Windows: if `python` opens the Microsoft Store**, Python isn't really installed. Get it from python.org (tick *Add to PATH*), or `winget install Python.Python.3.12`.

> **In WSL: work in `~/`, not `/mnt/c/`.** Installing into a project on the Windows drive is dramatically slower, and file-watching (`--reload`, `pytest-watch`) often doesn't fire at all. See [[Setting up a dev machine]].

---

## Which one

| | Matplotlib | Plotly |
|---|---|---|
| Output | Static image | **Interactive HTML** |
| Best for | Papers, loss curves, saved PNGs | Dashboards, exploration |
| Hover / zoom | No | **Yes, free** |
| Learning curve | Fiddly | Gentler |
| In [[Streamlit]] | `st.pyplot(fig)` | `st.plotly_chart(fig)` |

> **Matplotlib for anything you save as a file; Plotly for anything a human clicks on.**

---

## Matplotlib

```python
import matplotlib
matplotlib.use("Agg")            # no display (servers, CI) - BEFORE importing pyplot
import matplotlib.pyplot as plt
```

### The object-oriented API — use this

```python
import matplotlib.pyplot as plt

fig, ax = plt.subplots(figsize=(10, 6))
ax.plot(x, y, label="train", color="C0", linewidth=2)
ax.plot(x, y2, label="val", linestyle="--")
ax.set_xlabel("Epoch")
ax.set_ylabel("Loss")
ax.set_title("Training curve")
ax.legend()
ax.grid(True, alpha=0.3)
fig.tight_layout()
fig.savefig("loss.png", dpi=150, bbox_inches="tight")
plt.close(fig)                   # free the memory
```

> **Use `fig, ax = plt.subplots()`, not the `plt.plot()` global API.** The global one has hidden state and breaks the moment you make two figures — especially in loops or notebooks.

> **`plt.close(fig)` in loops.** Matplotlib keeps every open figure in memory; generating 500 plots without closing them exhausts RAM.

### Plot types

```python
ax.plot(x, y)                    # line
ax.scatter(x, y, s=50, alpha=0.6, c=colours)
ax.bar(labels, values) · ax.barh(labels, values)
ax.hist(data, bins=30, edgecolor="black")
ax.boxplot([a, b]) · ax.violinplot([a, b])
ax.imshow(matrix, cmap="viridis") · ax.pcolormesh(X, Y, Z)
ax.fill_between(x, lower, upper, alpha=0.2)      # confidence bands
ax.errorbar(x, y, yerr=err, fmt="o")
ax.axhline(0, color="grey") · ax.axvline(x0, linestyle=":")
```

### Subplots

```python
import matplotlib.pyplot as plt

fig, axes = plt.subplots(2, 2, figsize=(12, 8), sharex=True)
axes[0, 0].plot(x, y)
axes.flat[3].hist(data)

fig, (a1, a2) = plt.subplots(1, 2, gridspec_kw={"width_ratios": [2, 1]})
```

### Styling

```python
import matplotlib.pyplot as plt

plt.style.use("seaborn-v0_8-darkgrid")
plt.style.available                              # list them

ax.set_xlim(0, 100) · ax.set_ylim(bottom=0)
ax.set_xticks([0, 50, 100]) · ax.tick_params(axis="x", rotation=45)
ax.spines["top"].set_visible(False)              # remove chart junk
ax.spines["right"].set_visible(False)
```

### The loss curve you'll draw most

```python
import matplotlib.pyplot as plt
import mlflow

fig, ax = plt.subplots(figsize=(8, 5))
ax.plot(history["train"], label="train")
ax.plot(history["val"], label="val")
ax.set_xlabel("Epoch"); ax.set_ylabel("Loss"); ax.legend(); ax.grid(alpha=0.3)
mlflow.log_figure(fig, "loss_curve.png")         # [[MLflow experiment tracking]]
plt.close(fig)
```

> **Always plot train and val together.** A single final number tells you almost nothing; the *shape* tells you overfitting, underfitting or instability ([[Tensors, autograd and the training loop]]).

---

## Plotly

### Express — one-liners

```python
import plotly.express as px

fig = px.line(df, x="date", y="revenue", color="category", title="Revenue")
fig = px.bar(df, x="segment", y="count", color="segment")
fig = px.scatter(df, x="frequency", y="monetary", color="segment",
                 size="recency", hover_data=["customer_id"])
fig = px.histogram(df, x="revenue", nbins=50)
fig = px.box(df, x="segment", y="revenue")
fig = px.imshow(corr_matrix, text_auto=True)     # heatmap
fig = px.scatter_matrix(df, dimensions=["a","b","c"])
fig = px.choropleth(df, locations="iso", color="revenue")

fig.show()                                       # opens a browser
fig.write_html("chart.html") · fig.write_image("chart.png")   # png needs kaleido
```

> **`hover_data` is Plotly's superpower.** Put the customer ID in the tooltip and the chart becomes explorable, not just decorative.

### Graph Objects — full control

```python
import plotly.graph_objects as go

fig = go.Figure()
fig.add_trace(go.Scatter(x=df.date, y=df.actual, name="Actual", mode="lines"))
fig.add_trace(go.Scatter(x=df.date, y=df.forecast, name="Forecast",
                         mode="lines", line=dict(dash="dash")))
fig.add_trace(go.Scatter(                                    # confidence band
    x=list(df.date) + list(df.date[::-1]),
    y=list(df.upper) + list(df.lower[::-1]),
    fill="toself", fillcolor="rgba(0,100,200,0.15)",
    line=dict(color="rgba(255,255,255,0)"), name="95% CI"))

fig.update_layout(
    title="Forecast", xaxis_title="Date", yaxis_title="Revenue",
    hovermode="x unified",                       # one tooltip for all series
    template="plotly_dark", height=500,
    legend=dict(orientation="h", y=1.1),
)
```

> **`hovermode="x unified"` on any time series.** One tooltip showing every series at that date, instead of chasing individual points.

### Subplots

```python
import plotly.graph_objects as go
from plotly.subplots import make_subplots

fig = make_subplots(rows=2, cols=1, shared_xaxes=True,
                    subplot_titles=("Revenue", "Volume"))
fig.add_trace(go.Scatter(x=d, y=r), row=1, col=1)
fig.add_trace(go.Bar(x=d, y=v), row=2, col=1)
```

---

## In Streamlit

```python
import streamlit as st

st.plotly_chart(fig, use_container_width=True)   # interactive
st.pyplot(fig)                                    # static
st.line_chart(df, x="date", y="revenue")          # built-in, quickest
```

> **`use_container_width=True` on every chart**, or it won't resize and the layout breaks on other screens ([[Streamlit]]).

## Performance

| Problem | Fix |
|---|---|
| Browser freezes on a Plotly chart | Sample or aggregate — charts choke well before 100k points |
| Matplotlib memory grows in a loop | `plt.close(fig)` |
| Slow in a container / CI | `matplotlib.use("Agg")` before importing pyplot |
| `write_image` fails | `uv add kaleido` |

## Related

[[Python libraries]] · [[Streamlit]] · [[pandas]] · [[MLflow experiment tracking]] · [[Tensors, autograd and the training loop]]
