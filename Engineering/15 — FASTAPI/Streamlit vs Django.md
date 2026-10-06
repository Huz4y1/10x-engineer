---
tags: [streamlit, django, dashboards]
status: not-started
---

# Streamlit vs. Django

> **What this is:** the two ways to put a screen in front of your data, and how to choose.
> **Why you care:** both consume your FastAPI layer. The choice is about audience and lifespan, not capability.

---

## The decision, up front

| | Streamlit | Django |
|---|---|---|
| Best for | Internal tools, analyst-facing dashboards, quick prototypes | A real multi-user product with accounts, permissions, a long life |
| Time to first screen | Hours | Days |
| Auth / admin | Bolt-on, minimal | Built in (`django.contrib.auth`, admin panel) |
| Data flow | Script re-runs top-to-bottom per interaction; cache with `@st.cache_data` | Request/response, ORM-backed, full control over routing |
| Who writes it | A data person | A web developer (or a data person who's willing to become one) |
| Concurrent users | Tens | Thousands |
| UI control | What Streamlit gives you | Anything you can build in HTML/CSS/JS |
| Skip it if | You need fine-grained UI control or many concurrent users | You just need something on screen today |

**For the capstone: Streamlit.** It's an analyst-facing dashboard with one user (you). Django is in this note so you understand the trade-off and can defend the choice in an interview — build the comparison screen, feel the difference, then move on.

---

## Part 1 — Streamlit

### The execution model (understand this or nothing makes sense)

Streamlit's entire model is one rule:

> **Every time anyone interacts with anything, your whole script runs again from line 1.**

Move a slider? Script reruns. Click a button? Script reruns. Type in a box? Script reruns.

It's not a web app with event handlers. It's a script that gets re-executed constantly, and Streamlit diffs the output to update the page.

```python
import streamlit as st

st.title("Retail Analytics")

count = 0                                  # ← reset to 0 on EVERY rerun
if st.button("Add one"):
    count += 1
st.write(count)                            # always shows 0 or 1. Never 2.
```

That's not a bug. The script genuinely started over, so `count = 0` ran again.

Two features exist entirely to work around this rule: **caching** and **session state**. Everything else in Streamlit is just widgets.

### Caching — because reruns re-fetch everything

Without caching, every slider nudge re-hits your API. Ten interactions, ten identical network calls.

```python
import streamlit as st
import requests

@st.cache_data(ttl=300)                    # remember results for 5 minutes
def fetch_forecast(category: str, weeks: int) -> dict:
    r = requests.get(
        f"{API_URL}/sales/forecast",
        params={"category": category, "weeks": weeks},
        timeout=10,
    )
    r.raise_for_status()
    return r.json()
```

Streamlit hashes the arguments. Same arguments → cached result, no network call. Different arguments → a real call, then cached too.

**The two decorators:**

| Decorator | For | Behaviour |
|---|---|---|
| `@st.cache_data` | **Data** — DataFrames, dicts, API responses | Returns a **copy** each time, so mutating it can't corrupt the cache |
| `@st.cache_resource` | **Connections and models** — DB pools, ML models | Returns the **same object**, shared across all sessions |

```python
import streamlit as st

@st.cache_resource                         # one shared session across all users
def load_model():
    return onnxruntime.InferenceSession("model.onnx")
```

> **Pick the wrong one and you get subtle bugs.** `cache_data` on a model would serialise and copy it on every call — slow and possibly broken. `cache_resource` on a DataFrame means one user's mutation is visible to everyone.
>
> **Always set a `ttl`** on anything fetching live data, or the dashboard shows this morning's numbers forever and you'll waste an hour hunting a bug that isn't there.

### Session state — for things that must survive a rerun

```python
import streamlit as st

if "selected_category" not in st.session_state:
    st.session_state.selected_category = "Kitchen"      # initialise once

st.selectbox(
    "Category",
    ["Kitchen", "Lighting", "Garden"],
    key="selected_category",                            # widget ↔ state, two-way
)

st.write(f"Showing: {st.session_state.selected_category}")
```

Giving a widget a `key` binds it to `st.session_state` automatically — that's the cleanest way to use it.

The counter, fixed:

```python
import streamlit as st

if "count" not in st.session_state:
    st.session_state.count = 0

if st.button("Add one"):
    st.session_state.count += 1

st.write(st.session_state.count)                        # now it counts properly
```

> `st.session_state` is **per browser session**. Two people on the dashboard have separate state. It's memory, not storage — a refresh clears it.

### Layout

```python
from datetime import date
import streamlit as st

st.set_page_config(page_title="Retail Analytics", layout="wide", page_icon="📊")

# sidebar for controls
with st.sidebar:
    st.header("Filters")
    category = st.selectbox("Category", categories)
    date_range = st.date_input("Date range", value=(date(2011, 1, 1), date(2011, 12, 9)))
    weeks = st.slider("Forecast weeks", 1, 12, 4)

# KPI row
c1, c2, c3 = st.columns(3)
c1.metric("Total revenue", f"£{total:,.0f}", delta=f"{change:+.1%}")
c2.metric("Orders", f"{orders:,}")
c3.metric("Avg basket", f"£{avg:,.2f}")

# tabs for sections
tab1, tab2 = st.tabs(["Sales & Forecast", "Customer Segments"])
with tab1:
    st.line_chart(df, x="date", y=["revenue", "forecast"])
with tab2:
    st.bar_chart(segments, x="segment", y="customers")
```

> **Controls in the sidebar, results in the main area.** It's the convention, users expect it, and it keeps the main panel for what people actually came to see.

### The capstone dashboard, whole

```python
import streamlit as st
import requests
import pandas as pd
from datetime import date

API_URL = st.secrets["API_URL"]          # from .streamlit/secrets.toml — not hardcoded

st.set_page_config(page_title="Retail Analytics", layout="wide")

@st.cache_data(ttl=300)
def api_get(path: str, **params) -> dict:
    r = requests.get(f"{API_URL}{path}", params=params, timeout=15)
    r.raise_for_status()
    return r.json()

st.title("📊 Retail Analytics")

with st.sidebar:
    st.header("Filters")
    category = st.selectbox("Category", ["Kitchen", "Lighting", "Garden"], key="category")
    weeks = st.slider("Forecast horizon (weeks)", 1, 12, 4, key="weeks")

try:
    with st.spinner("Loading…"):
        forecast = api_get("/sales/forecast", category=category, weeks=weeks)
        segments = api_get("/customers/segments/summary")
except requests.HTTPError as e:
    st.error(f"API returned {e.response.status_code}. Is the service running?")
    st.stop()
except requests.RequestException:
    st.error("Could not reach the API.")
    st.stop()

df = pd.DataFrame(forecast["forecast"])

c1, c2, c3 = st.columns(3)
c1.metric("Predicted revenue", f"£{df.predicted_revenue.sum():,.0f}")
c2.metric("Weeks", len(df))
c3.metric("Model", forecast["model_version"])

tab1, tab2 = st.tabs(["Forecast", "Segments"])

with tab1:
    st.subheader(f"{category} — {weeks} week forecast")
    st.line_chart(df, x="week_starting", y="predicted_revenue")
    with st.expander("Raw data"):
        st.dataframe(df, use_container_width=True)

with tab2:
    seg = pd.DataFrame(segments["segments"])
    st.bar_chart(seg, x="segment", y="customer_count")

st.caption(f"Generated {forecast['generated_at']}")
```

Notice: **every number comes from the API.** No database connection, no model file, no credentials. That's the architecture working.

> **Always wrap API calls in try/except.** Streamlit's default failure mode is a red Python traceback in the middle of the page — ugly and useless to a non-developer. `st.error()` + `st.stop()` gives a clean message instead.

### Deployment

Same pattern as the API ([[FastAPI data and deployment]]):

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY pyproject.toml uv.lock ./
RUN pip install --no-cache-dir uv && uv sync --frozen --no-dev
COPY dashboard/ ./dashboard/
EXPOSE 8501
CMD ["uv", "run", "streamlit", "run", "dashboard/app.py", \
     "--server.port=8501", "--server.address=0.0.0.0", "--server.headless=true"]
```

```bash
az containerapp create \
  --name retail-dashboard --resource-group retail-rg --environment retail-env \
  --image $ACR.azurecr.io/retail-dashboard:v1 \
  --target-port 8501 --ingress external \
  --env-vars "API_URL=https://retail-api.internal.<env>.azurecontainerapps.io"
```

> **`--server.address=0.0.0.0` and `--server.headless=true`.** Without headless, Streamlit tries to open a browser and prompts for an email on first run — in a container that just hangs.
>
> Both apps in the same Container Apps environment can talk over the **internal** URL, so you can set the API's ingress to `internal` and expose only the dashboard publicly. Nice architecture, one flag.

### Streamlit's real limits

Worth knowing before you over-invest:

- **Concurrency.** Every user gets a full script rerun on every interaction. Fine for tens of users; it won't serve hundreds.
- **Auth.** There's no built-in login. You bolt on `streamlit-authenticator`, or put it behind Azure AD via Container Apps auth.
- **UI control.** You get Streamlit's components. Custom layouts mean writing a React component.
- **Reruns are wasteful** by nature. Caching hides it; it doesn't remove it.

When you hit two of those, it's Django (or a React frontend on your existing API).

---

## Part 2 — Django

### The idea in plain English

Django is a **complete web framework**. Where Streamlit gives you one screen fast, Django gives you the machinery for an actual product: users, logins, permissions, an admin panel, URL routing, HTML templates, and a database ORM.

The trade: much more capable, much more to learn.

### The request cycle

```
Browser  →  URL router  →  View  →  Model (ORM)  →  Database
                            ↓
                         Template  →  HTML  →  Browser
```

Nothing "reruns". A request comes in, a **view** function handles it, it returns HTML, done. Standard web architecture — the same shape as FastAPI, plus templates and an ORM.

### Models → migrations → admin

```python
# dashboard/models.py
from django.db import models

class DailySales(models.Model):
    date = models.DateField(db_index=True)
    product_category = models.CharField(max_length=100, db_index=True)
    revenue = models.DecimalField(max_digits=12, decimal_places=2)
    units_sold = models.IntegerField()

    class Meta:
        unique_together = ("date", "product_category")
        ordering = ["-date"]

    def __str__(self):
        return f"{self.date} {self.product_category}: £{self.revenue}"
```

```bash
python manage.py makemigrations     # generate the SQL change script
python manage.py migrate            # apply it
python manage.py createsuperuser
```

```python
# dashboard/admin.py
from django.contrib import admin
from .models import DailySales

@admin.register(DailySales)
class DailySalesAdmin(admin.ModelAdmin):
    list_display = ("date", "product_category", "revenue", "units_sold")
    list_filter = ("product_category", "date")
    search_fields = ("product_category",)
```

Those six lines give you a complete, searchable, filterable, permission-aware CRUD interface at `/admin`. **This is Django's killer feature.** Building the equivalent in Streamlit is days of work.

> **Migrations are the real point.** They're version-controlled, reviewable, reversible database change scripts, generated from your model definitions. When someone asks "how do you manage schema changes across environments?", this is the answer.

### Views and URLs

```python
# dashboard/views.py
from django.shortcuts import render
from django.db.models import Sum
import requests

def sales_dashboard(request):
    category = request.GET.get("category", "Kitchen")

    history = (DailySales.objects
               .filter(product_category=category)
               .values("date", "revenue")
               .order_by("date"))

    forecast = requests.get(
        f"{settings.API_URL}/sales/forecast",
        params={"category": category, "weeks": 4},
        timeout=10,
    ).json()

    return render(request, "dashboard/sales.html", {
        "category": category,
        "history": list(history),
        "forecast": forecast["forecast"],
        "total": history.aggregate(t=Sum("revenue"))["t"],
    })
```

```python
# dashboard/urls.py
from django.urls import path
from . import views

urlpatterns = [
    path("sales/", views.sales_dashboard, name="sales"),
    path("customers/<str:customer_id>/", views.customer_detail, name="customer-detail"),
]
```

**Function-based vs. class-based views:** function-based is explicit and easy to read. Class-based (`ListView`, `DetailView`) removes boilerplate for standard CRUD but hides the flow. **Start function-based.** Reach for class-based when you've written the same list-view code four times.

### Templates

```html
<!-- templates/dashboard/sales.html -->
{% extends "base.html" %}

{% block content %}
<h1>{{ category }} Sales</h1>
<p>Total revenue: £{{ total|floatformat:2 }}</p>

<table>
  <tr><th>Week</th><th>Predicted</th></tr>
  {% for point in forecast %}
    <tr>
      <td>{{ point.week_starting }}</td>
      <td>£{{ point.predicted_revenue|floatformat:2 }}</td>
    </tr>
  {% empty %}
    <tr><td colspan="2">No forecast available.</td></tr>
  {% endfor %}
</table>
{% endblock %}
```

Django templates deliberately can't run arbitrary Python — logic belongs in the view. `{% empty %}` handles the empty case, which is easy to forget in a hand-rolled loop.

### Auth, for free

```python
from django.contrib.auth.decorators import login_required

@login_required
def sales_dashboard(request):
    ...
```

One decorator: unauthenticated users get redirected to a login page that already exists. Users, groups, permissions, password hashing, password reset emails, session management — all included and all correct. Getting auth right by hand is genuinely hard and genuinely dangerous to get wrong.

### The N+1 query trap

The Django-specific performance bug you *will* hit:

```python
# ✗ 1 query for the list, then 1 MORE per row = 1001 queries
for sale in DailySales.objects.all():
    print(sale.category.name)          # each access hits the database

# ✓ 2 queries total
for sale in DailySales.objects.select_related("category"):
    print(sale.category.name)
```

`select_related` for foreign keys (SQL join), `prefetch_related` for many-to-many. Install `django-debug-toolbar` — it shows the query count per page, and the number will shock you the first time.

### Django REST Framework — only if you need it

DRF turns Django into an API server. **You don't need it here** — that's what FastAPI is for. Know it exists so you understand what people mean when they mention it.

---

## Making the choice

Ask these in order:

1. **Does it need user accounts and permissions?** → Django.
2. **Will more than ~50 people use it at once?** → Django (or React + FastAPI).
3. **Is it a product with a multi-year life?** → Django.
4. **Otherwise** → Streamlit.

Answer for the capstone: no, no, no → **Streamlit**.

> **A third option worth knowing:** you already have a well-designed REST API. A React/Vue frontend against it gives complete UI control and scales to any number of users. It's just much more work. FastAPI + a JS frontend is the standard modern shape for a real product.

---

## When something goes wrong

| Symptom | Likely cause | Fix |
|---|---|---|
| **Streamlit:** a variable resets constantly | The script reran; ordinary variables don't survive | `st.session_state` |
| **Streamlit:** button appears to do nothing | State reset on rerun | Store the result in `st.session_state` |
| **Streamlit:** data never updates | Cached with no `ttl` | Add `ttl=300`, or a manual "Refresh" that calls `.clear()` |
| **Streamlit:** slow, hammering the API | No caching | `@st.cache_data` on the fetch function |
| **Streamlit:** "cannot hash argument" | An unhashable arg (a DB connection, a model) | Prefix the param with `_` to skip hashing, or use `cache_resource` |
| **Streamlit:** red traceback on the page | Unhandled API error | try/except + `st.error()` + `st.stop()` |
| **Streamlit:** container hangs at startup | Missing `--server.headless=true` | Add it |
| **Streamlit:** blank page in Docker | Bound to localhost | `--server.address=0.0.0.0` |
| **Django:** page is slow, many queries | N+1 | `select_related` / `prefetch_related` |
| **Django:** "no such table" | Migration not applied | `python manage.py migrate` |
| **Django:** "You have unapplied migrations" | Model changed, migration not generated | `makemigrations` then `migrate` |
| **Django:** CSS/JS missing in production | Static files not collected | `python manage.py collectstatic`; `DEBUG=False` doesn't serve them |
| **Django:** 500 with no detail in production | `DEBUG=False` hides it (correctly) | Read the logs, never turn `DEBUG` on in production |
| **Either:** can't reach the API from the container | Used `localhost` | Use the service/internal name ([[Dev environment - Git, Docker, CLI]]) |

---

## Practice checklist

### Streamlit
- [ ] **The rerun model** — why every interaction re-executes the whole script
- [ ] `st.cache_data` vs. `st.cache_resource`, and why `ttl` is essential
- [ ] Session state for anything that must persist across reruns
- [ ] Layout: columns, tabs, sidebar filters, `st.metric`
- [ ] Error handling with `st.error()` + `st.stop()`
- [ ] Its real limits: concurrency, auth, UI control

### Django
- [ ] Models → migrations → admin panel, in that order
- [ ] Views and URL routing (function-based vs. class-based)
- [ ] Templates, and why logic lives in the view
- [ ] `@login_required` and what you get for free
- [ ] The N+1 query problem and `select_related`
- [ ] Templates vs. Django REST Framework if you need Django to also serve an API

## Hands-on

- [ ] Build one dashboard in Streamlit reading from your FastAPI endpoint, with caching and at least one filter using session state
- [ ] Write the broken counter, watch it fail, fix it with session state
- [ ] Remove `ttl` from a cache and confirm the data goes stale — so you recognise it later
- [ ] Rebuild that dashboard's core screen in Django (model, view, template) to feel the contrast first-hand
- [ ] Add `@login_required` to the Django view and see the free login page
- [ ] Install `django-debug-toolbar` and find an N+1 query
- [ ] Decide, in one sentence, which you'd reach for by default — and why

## Resources

- [Streamlit docs](https://docs.streamlit.io/)
- [Streamlit: caching explained](https://docs.streamlit.io/develop/concepts/architecture/caching)
- [Django official tutorial](https://docs.djangoproject.com/en/stable/intro/tutorial01/) — all seven parts
- [Django: database optimization](https://docs.djangoproject.com/en/stable/topics/db/optimization/)

## Next

[[Tensors, autograd and the training loop]]
