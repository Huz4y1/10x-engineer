---
tags: [django, python, web, reference]
---

# Django reference

The complete API. Your walkthrough notes: [[Django]] · Language: [[Python]] · Compared with dashboards: [[Streamlit vs Django]]

```bash
uv add django
django-admin startproject config .
python manage.py startapp core
```

---

## What Django is, and when to pick it

Django is **batteries-included**: ORM, admin panel, auth, forms, migrations, templates, security defaults — all in the box.

| Pick Django when | Pick something else when |
|---|---|
| You need auth, an admin panel and a database from day one | You need an async ML API → [[FastAPI reference]] |
| Real users, permissions, forms, sessions | You need a quick data dashboard → [[Streamlit]] |
| A content-heavy site with server-rendered pages | The frontend is React → [[Next.js reference]] + [[Actix Web]] |

> **The admin panel is the reason people choose Django.** `python manage.py createsuperuser` and you have a working CRUD interface over your whole database. Building that by hand takes weeks.

---

## Project layout

```
config/                 # the PROJECT
├── settings.py         # all configuration
├── urls.py             # root URL routing
├── asgi.py / wsgi.py   # the server entry point
core/                   # an APP (a project has many)
├── models.py           # database tables
├── views.py            # request -> response
├── urls.py             # this app's routes
├── admin.py            # admin registration
├── forms.py
├── migrations/
├── templates/core/
└── tests.py
manage.py
```

> **Project vs app:** the *project* is the site, an *app* is a feature within it. Keep apps small and single-purpose.

---

## The request cycle

```
URL -> urls.py -> middleware -> view -> model (ORM) -> database
                                  |
                              template -> HTML -> browser
```

---

## Models — the database in Python

```python
from django.db import models
from django.contrib.auth.models import User

class Engine(models.Model):
    name = models.CharField(max_length=100, unique=True)
    serial = models.CharField(max_length=50, db_index=True)
    installed_on = models.DateField(null=True, blank=True)
    active = models.BooleanField(default=True)
    owner = models.ForeignKey(User, on_delete=models.CASCADE, related_name="engines")
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        ordering = ["-created_at"]
        indexes = [models.Index(fields=["serial", "active"])]

    def __str__(self):
        return self.name        # what the admin shows - ALWAYS define this
```

| Field | For |
|---|---|
| `CharField(max_length=)` | Short text — `max_length` required |
| `TextField()` | Long text |
| `IntegerField` / `FloatField` | Numbers |
| **`DecimalField(max_digits, decimal_places)`** | **Money — never `FloatField`** |
| `BooleanField` | True/False |
| `DateField` / `DateTimeField` | Dates |
| `EmailField` / `URLField` / `SlugField` | Validated text |
| `JSONField` | Structured blobs |
| `FileField` / `ImageField` | Uploads |
| `ForeignKey` | Many-to-one |
| `ManyToManyField` | Many-to-many |
| `OneToOneField` | One-to-one |

> ⚠️ **Money in a `FloatField` loses pennies.** `0.1 + 0.2 != 0.3` — use `DecimalField` ([[Numerical methods in practice]], [[Data modeling]]).

> **`null=True` is for the database, `blank=True` is for forms.** They're different. On a `CharField`, use `blank=True` alone — an empty string, not `NULL`, so you don't have two kinds of "empty".

### `on_delete` — you must choose

| Value | When the parent is deleted |
|---|---|
| `CASCADE` | Delete the children too |
| `PROTECT` | **Refuse the delete** — safest for financial records |
| `SET_NULL` | Set to NULL (needs `null=True`) |
| `SET_DEFAULT` / `DO_NOTHING` | As named |

> ⚠️ **`CASCADE` is a real delete.** Deleting one user can silently remove thousands of rows. Use `PROTECT` for anything you'd hate to lose.

## Migrations

```bash
python manage.py makemigrations          # write the migration file
python manage.py makemigrations --dry-run --verbosity 3   # preview it
python manage.py sqlmigrate core 0002    # see the actual SQL
python manage.py migrate                 # apply
python manage.py showmigrations
python manage.py migrate core 0001       # roll back to 0001
```

> **Commit migration files to git.** They are part of your source code. A teammate pulling your models without your migrations gets an inconsistent database.

> ⚠️ **Adding a non-nullable field to an existing table prompts for a default.** On a big table this rewrites every row and locks it — a real outage in production. Add it nullable, backfill, then make it required.

---

## The ORM

```python
Engine.objects.all()
Engine.objects.get(pk=1)                   # raises DoesNotExist / MultipleObjectsReturned
Engine.objects.filter(active=True).exclude(serial="")
Engine.objects.filter(name__icontains="turbo")
Engine.objects.order_by("-created_at")[:10]
Engine.objects.count()
Engine.objects.first()
```

| Lookup | SQL |
|---|---|
| `__exact` / `__iexact` | `=` / case-insensitive |
| `__contains` / `__icontains` | `LIKE %x%` |
| `__gt __gte __lt __lte` | Comparisons |
| `__in=[...]` | `IN` |
| `__isnull=True` | `IS NULL` |
| `__startswith` / `__endswith` | Prefix / suffix |
| `__range=(a, b)` | `BETWEEN` |
| `__date` `__year` `__month` | Date parts |
| `field__related__field` | Follows a relation — **joins** |

```python
from django.db.models import Count, Sum, Avg, Q, F

Engine.objects.aggregate(Avg("hours"))                       # one number
Engine.objects.values("owner").annotate(n=Count("id"))       # group by

Engine.objects.filter(Q(active=True) | Q(hours__gt=500))     # OR
Engine.objects.filter(hours__gt=F("threshold"))              # compare two columns
Engine.objects.update(hours=F("hours") + 1)                  # atomic, no race
```

> **`F()` does the arithmetic in the database.** `obj.hours += 1; obj.save()` reads then writes, and two simultaneous requests lose an increment. `F("hours") + 1` cannot.

### The N+1 problem — the Django performance bug

```python
# 1 query for engines, then ONE MORE PER ENGINE. 1000 engines = 1001 queries.
for e in Engine.objects.all():
    print(e.owner.username)

# 1 query, joined
for e in Engine.objects.select_related("owner"):
    print(e.owner.username)

# 2 queries, for many-to-many / reverse FK
for e in Engine.objects.prefetch_related("readings"):
    print(len(e.readings.all()))
```

> ⚠️ **This is the single most common Django performance problem.** It's invisible in development with ten rows and fatal with ten thousand.
> - **`select_related`** — forward `ForeignKey`/`OneToOne` (does a SQL JOIN)
> - **`prefetch_related`** — `ManyToMany` and reverse relations (does a second query)

```python
print(Engine.objects.filter(active=True).query)     # the SQL
from django.db import connection; connection.queries  # what actually ran
```

> **Install `django-debug-toolbar` in development.** It shows the query count per page. If a page runs 300 queries, you have an N+1.

### QuerySets are lazy

```python
qs = Engine.objects.filter(active=True)   # NO query yet
qs = qs.order_by("name")                  # still none
list(qs)                                  # NOW it runs
```

```python
Engine.objects.only("name", "serial")     # fetch just these columns
Engine.objects.defer("big_text_field")
Engine.objects.values("name")             # dicts, not model objects - faster
Engine.objects.iterator()                 # don't load 1M rows into memory
Engine.objects.bulk_create(objs, batch_size=1000)   # one INSERT, not 1000
```

> ⚠️ **A QuerySet caches after evaluation, but re-filtering re-queries.** Calling `.count()` and then iterating hits the database twice. Assign to a list if you need it more than once.

---

## Views

```python
from django.shortcuts import render, get_object_or_404, redirect

def engine_list(request):
    engines = Engine.objects.select_related("owner").filter(active=True)
    return render(request, "core/engine_list.html", {"engines": engines})

def engine_detail(request, pk):
    engine = get_object_or_404(Engine, pk=pk)      # 404 instead of a 500
    return render(request, "core/engine_detail.html", {"engine": engine})
```

```python
from django.views.generic import ListView, DetailView, CreateView
from django.contrib.auth.mixins import LoginRequiredMixin

class EngineList(LoginRequiredMixin, ListView):
    model = Engine
    paginate_by = 25

    def get_queryset(self):
        return Engine.objects.select_related("owner").filter(owner=self.request.user)
```

> **Start with function views.** They're explicit and easy to read. Move to class-based views when you're writing the same CRUD boilerplate for the fifth time.

> ⚠️ **`LoginRequiredMixin` checks you're logged in, not that the object is yours.** Filter by `request.user` in `get_queryset` or anyone can read anyone's records — see [[Security in practice]].

## URLs

```python
# config/urls.py
from django.contrib import admin
from django.urls import path, include
urlpatterns = [
    path("admin/", admin.site.urls),
    path("engines/", include("core.urls")),
]

# core/urls.py
app_name = "core"
urlpatterns = [
    path("", views.engine_list, name="list"),
    path("<int:pk>/", views.engine_detail, name="detail"),
    path("<slug:slug>/", views.by_slug, name="by-slug"),
]
```

```python
from django.urls import reverse
reverse("core:detail", args=[42])          # "/engines/42/"
```

```html
<a href="{% url 'core:detail' engine.pk %}">{{ engine.name }}</a>
```

> **Never hardcode a URL.** Use `{% url %}` and `reverse()` — then changing a route doesn't break every link.

---

## Templates

```html
{% extends "base.html" %}
{% block content %}
  <h1>{{ engine.name }}</h1>

  {% for e in engines %}
    <li>{{ e.name }} — {{ e.hours|floatformat:1 }}</li>
  {% empty %}
    <li>No engines.</li>
  {% endfor %}

  {% if user.is_authenticated %}
    <a href="{% url 'logout' %}">Log out</a>
  {% endif %}

  {{ text|truncatewords:20|upper }}
  {{ value|default:"n/a" }}
  {{ when|date:"Y-m-d" }}
{% endblock %}
```

> **Templates auto-escape HTML.** That's what protects you from XSS. `{{ x|safe }}` and `{% autoescape off %}` turn it off — **never do that with user input** ([[Security in practice]]).

> **Templates deliberately can't call functions with arguments.** That's a design choice pushing logic into views where it's testable.

## Forms

```python
from django import forms

class EngineForm(forms.ModelForm):
    class Meta:
        model = Engine
        fields = ["name", "serial", "installed_on"]

    def clean_serial(self):
        s = self.cleaned_data["serial"]
        if not s.startswith("EN-"):
            raise forms.ValidationError("Serial must start with EN-")
        return s
```

```python
from django.shortcuts import redirect, render

def create(request):
    form = EngineForm(request.POST or None)
    if request.method == "POST" and form.is_valid():
        obj = form.save(commit=False)
        obj.owner = request.user          # set server-side, NEVER from the form
        obj.save()
        return redirect("core:detail", pk=obj.pk)
    return render(request, "core/form.html", {"form": form})
```

```html
<form method="post">{% csrf_token %}
  {{ form.as_p }}
  <button>Save</button>
</form>
```

> ⚠️ **`{% csrf_token %}` in every POST form.** Without it Django rejects the request — which is the protection working, not a bug to disable.

> ⚠️ **Never take `owner`, `is_staff`, `price` or any privileged field from the form.** A user can post any value. Set them in the view.

---

## Admin

```python
from django.contrib import admin

@admin.register(Engine)
class EngineAdmin(admin.ModelAdmin):
    list_display = ("name", "serial", "owner", "active", "created_at")
    list_filter = ("active", "created_at")
    search_fields = ("name", "serial")
    list_select_related = ("owner",)      # avoids N+1 in the admin list
    readonly_fields = ("created_at",)
    date_hierarchy = "created_at"
```

```bash
python manage.py createsuperuser
```

> ⚠️ **The admin is for trusted staff only.** It's a full read/write interface to your database with no business rules. Never give it to end users.

## Auth

```python
from django.contrib.auth.decorators import login_required, permission_required
from django.contrib.auth import authenticate, login, logout

@login_required
def dashboard(request): ...

@permission_required("core.change_engine")
def edit(request, pk): ...
```

```python
# settings.py - do this on day one, even if you don't need it yet
AUTH_USER_MODEL = "core.User"
```

> ⚠️ **Set a custom user model before your first migration.** Changing it later is one of the genuinely painful migrations in Django. Even an empty `class User(AbstractUser): pass` saves you.

---

## Settings and secrets

```python
import os
from pathlib import Path

BASE_DIR = Path(__file__).resolve().parent.parent
SECRET_KEY = os.environ["DJANGO_SECRET_KEY"]         # crash if missing
DEBUG = os.environ.get("DJANGO_DEBUG", "") == "1"    # default OFF
ALLOWED_HOSTS = os.environ.get("ALLOWED_HOSTS", "").split(",")

DATABASES = {"default": {
    "ENGINE": "django.db.backends.postgresql",
    "NAME": os.environ["DB_NAME"],
    "CONN_MAX_AGE": 600,                             # reuse connections
}}
```

> ⚠️ **`DEBUG = True` in production shows your settings, your SQL and a full stack trace to anyone who triggers an error.** It is the classic Django breach. Default it to *off* so a missing variable fails safe.

```bash
python manage.py check --deploy     # audits your production settings. Run it.
```

| Production setting | Value |
|---|---|
| `DEBUG` | `False` |
| `ALLOWED_HOSTS` | Your actual domains |
| `SECURE_SSL_REDIRECT` | `True` |
| `SESSION_COOKIE_SECURE` / `CSRF_COOKIE_SECURE` | `True` |
| `SECURE_HSTS_SECONDS` | `31536000` |

---

## Django REST Framework

```bash
uv add djangorestframework
```

```python
from rest_framework import permissions, serializers, viewsets

class EngineSerializer(serializers.ModelSerializer):
    class Meta:
        model = Engine
        fields = ["id", "name", "serial", "active"]
        read_only_fields = ["id"]

class EngineViewSet(viewsets.ModelViewSet):
    serializer_class = EngineSerializer
    permission_classes = [permissions.IsAuthenticated]

    def get_queryset(self):
        return Engine.objects.filter(owner=self.request.user)   # ownership filter
```

> **For an ML inference API, use [[FastAPI reference]] instead** — async, faster, Pydantic validation, automatic OpenAPI docs. Django's strength is the database-and-admin side.

---

## Commands and testing

```bash
python manage.py runserver
python manage.py shell_plus            # with django-extensions
python manage.py dbshell
python manage.py collectstatic         # required before deploying
python manage.py test
python manage.py createsuperuser
```

```python
from django.test import TestCase

class EngineTests(TestCase):
    def setUp(self):
        self.engine = Engine.objects.create(name="E1", serial="EN-1")

    def test_detail_requires_login(self):
        r = self.client.get(f"/engines/{self.engine.pk}/")
        self.assertEqual(r.status_code, 302)

    def test_no_n_plus_one(self):
        with self.assertNumQueries(2):          # catches N+1 in CI
            list(Engine.objects.select_related("owner"))
```

> **`assertNumQueries` is how you stop an N+1 coming back.** It fails the build when someone removes a `select_related`.

## Deploying

```bash
uv add gunicorn psycopg[binary] whitenoise
gunicorn config.wsgi:application --bind 0.0.0.0:8000 --workers 4
```

> **`runserver` is a development server. Never use it in production** — single-threaded, no security hardening. Use gunicorn (WSGI) or uvicorn (ASGI) behind nginx ([[Docker deep dive]], [[Deployment patterns]]).

---

## Common mistakes

| Mistake | Consequence |
|---|---|
| `DEBUG = True` in production | **Settings and stack traces exposed publicly** |
| N+1 queries | Page fine at 10 rows, unusable at 10,000 |
| No custom user model from the start | A painful migration later |
| `FloatField` for money | Lost pennies |
| `LoginRequiredMixin` without an ownership filter | Users read each other's data |
| Privileged field taken from the form | Privilege escalation |
| `{{ x\|safe }}` on user input | XSS |
| Missing `{% csrf_token %}` | Form rejected (correctly) |
| Migrations not committed | Teammates' databases diverge |
| Non-nullable column added to a big table | Table rewrite and lock |
| `obj.n += 1; obj.save()` | Lost updates under concurrency — use `F()` |
| Hardcoded URLs | Every link breaks on a route change |
| `runserver` in production | Slow and insecure |

## Related

[[Django]] · [[Python]] · [[FastAPI reference]] · [[Streamlit vs Django]] · [[PostgreSQL reference]] · [[SQL fundamentals]] · [[Security in practice]] · [[Data modeling]] · [[HTML]] · [[CSS]] · [[Docker deep dive]] · [[Deployment patterns]]
