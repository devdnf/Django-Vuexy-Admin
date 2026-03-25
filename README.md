# django-vuexy-admin

A drop-in replacement for the default Django admin interface styled with the **Vuexy Bootstrap 5** theme. Fully self-contained — all Vuexy static assets ship with the package.

Features:
- Full Vuexy vertical layout (sidebar, navbar, footer, breadcrumbs)
- Dark / Light / System mode with zero flash on reload
- RTL support (Arabic and other RTL languages) via Django's i18n stack
- Config-driven sidebar: control app/model order, icons, and display names from `settings.py`
- Pluggable dashboard: inject any KPI stats or charts from your project via a single callback
- Reusable dashboard widgets: stat cards, sparklines, full ApexCharts cards, progress bars
- All Django admin views restyled (change list, change form, filters, inlines, login, etc.)

---

## Installation

```bash
pip install django-vuexy-admin
```

Or from a local clone during development:

```bash
pip install -e /path/to/django-vuexy-admin
```

---

## Setup

### 1. Add to `INSTALLED_APPS`

`vuexy_admin` must come **before** `django.contrib.admin` so its templates and static files take priority.

```python
# settings.py
INSTALLED_APPS = [
    "vuexy_admin",              # ← must be before django.contrib.admin
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",
    # ... your apps
]
```

### 2. Enable locale middleware and language URLs

Required for the language switcher and RTL support.

```python
# settings.py
MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "django.contrib.sessions.middleware.SessionMiddleware",
    "django.middleware.locale.LocaleMiddleware",   # ← add this
    "django.middleware.common.CommonMiddleware",
    # ...
]

LANGUAGES = [
    ("en", "English"),
    ("ar", "Arabic"),
    # add any other languages you need
]
```

```python
# urls.py
from django.urls import include, path

urlpatterns = [
    path("i18n/", include("django.conf.urls.i18n")),  # ← add this
    path("admin/", admin.site.urls),
    # ...
]
```

### 3. Template loading

Ensure `APP_DIRS` is `True` (default) so Django picks up templates from installed apps:

```python
# settings.py
TEMPLATES = [
    {
        "BACKEND": "django.template.backends.django.DjangoTemplates",
        "DIRS": [BASE_DIR / "templates"],
        "APP_DIRS": True,
        # ...
    }
]
```

### 4. Collect static files

```bash
python manage.py collectstatic
```

---

## Configuration

All options live under a single `VUEXY_ADMIN` dict in `settings.py`. Every key is optional.

```python
# settings.py
from django.utils.translation import gettext_lazy as _

VUEXY_ADMIN = {
    "SIDEBAR_CONFIG": { ... },              # see Sidebar section below
    "DASHBOARD_CONTEXT_CALLBACK": None,     # see Dashboard section below
}
```

---

## Sidebar

The sidebar is fully driven by `SIDEBAR_CONFIG`. You control:

- **`app_order`** — the order in which apps appear. Apps not listed are appended alphabetically.
- **`apps`** — per-app configuration:
  - `name` — optional display name override (falls back to `AppConfig.verbose_name`)
  - `models` — per-model configuration keyed by the model class name (`object_name`):
    - `icon` — Tabler Icon class name (e.g. `"ti-package"`)
    - `order` — integer sort order within the app (default: `999`)
    - `name` — optional display name override (falls back to `verbose_name`)

```python
VUEXY_ADMIN = {
    "SIDEBAR_CONFIG": {
        "app_order": ["myapp", "otherapp", "auth"],
        "apps": {
            "myapp": {
                "name": _("My Application"),
                "models": {
                    "MyModel":      {"icon": "ti-box",          "order": 1},
                    "AnotherModel": {"icon": "ti-file-text",    "order": 2},
                },
            },
            "auth": {
                "name": _("Users & Permissions"),
                "models": {
                    "User":  {"icon": "ti-users", "order": 1},
                    "Group": {"icon": "ti-lock",  "order": 2},
                },
            },
        },
    },
}
```

### Icon reference

Icons use [Tabler Icons](https://tabler.io/icons) — prefix every class name with `ti-`.
Browse all icons at https://tabler.io/icons.

---

## Dashboard

The default dashboard shows app model cards and recent actions. To add KPI stats, charts, or any other content, you have two options:

### Option A — Context callback (recommended)

Define a function that receives `request` and returns a dict. The dict is merged into the admin `index` template context:

```python
# settings.py
VUEXY_ADMIN = {
    "DASHBOARD_CONTEXT_CALLBACK": "myapp.admin_dashboard.get_context",
}
```

```python
# myapp/admin_dashboard.py

def get_context(request):
    # Query whatever your project needs and return a plain dict.
    # The keys become template variables available in your index.html override.
    return {
        "my_stat": 42,
        "my_chart_data": "[10, 20, 15, 30, 25]",
    }
```

### Option B — Override `index.html`

Create `templates/admin/index.html` in your project's templates directory. Django's loader will prefer it over the package version. Extend the package template and fill the `{% block dashboard %}` block:

```html
{% extends "admin/base_site.html" %}
{% load i18n static %}

{% block extrastyle %}{{ block.super }}
  {% include "admin/widgets/_apex_scripts.html" %}
{% endblock %}

{% block content_title %}<h1 class="mb-0">{% trans "Dashboard" %}</h1>{% endblock %}

{% block content %}
<div class="row g-4 mt-0">

  {# Your KPI cards go here — use the widget templates below #}
  <div class="col-xl-3 col-sm-6">
    {% include "admin/widgets/_stat_card.html" with title="..." value=my_stat icon="ti-chart-bar" color="primary" %}
  </div>

  {# Include the default app cards + recent actions from the package #}
  {{ block.super }}

</div>
{% endblock %}
```

---

## Dashboard widgets

Five reusable widget templates are available. Include them anywhere in your admin templates.

---

### `_stat_card.html` — KPI tile

A card showing a single metric with an icon badge, optional trend indicator, and subtitle.

```html
{% include "admin/widgets/_stat_card.html" with
    title="..."
    value=my_value
    icon="ti-chart-bar"
    color="primary"
    trend="+12.5%"
    trend_up=True
    subtitle="vs. last month"
%}
```

| Variable | Required | Description |
|---|---|---|
| `title` | yes | Label above the value |
| `value` | yes | The formatted value to display |
| `icon` | no | Tabler icon class (default: `ti-chart-bar`) |
| `color` | no | Bootstrap color name: `primary` `success` `danger` `warning` `info` (default: `primary`) |
| `trend` | no | Trend string, e.g. `"+5.2%"` |
| `trend_up` | no | `True` → green up arrow, `False` → red down arrow |
| `subtitle` | no | Small text below the trend |

---

### `_sparkline_card.html` — Stat + sparkline chart

A stat card with a compact bar, line, or area sparkline underneath. **Requires ApexCharts** — include `_apex_scripts.html` once per page.

```html
{% include "admin/widgets/_sparkline_card.html" with
    chart_id="unique_id"
    title="..."
    subtitle="Last 7 days"
    value=my_value
    chart_data=my_json_list
    trend="+8.0%"
    trend_up=True
    chart_type="bar"
    color="#7367f0"
    chart_height=80
%}
```

| Variable | Required | Description |
|---|---|---|
| `chart_id` | yes | Unique HTML element ID |
| `title` | yes | Card title |
| `chart_data` | yes | JSON list of numbers, e.g. `"[10, 20, 15, 30]"` |
| `value` | no | Summary value shown above the chart |
| `subtitle` | no | Small text below the title |
| `trend` / `trend_up` | no | Same as `_stat_card` |
| `chart_type` | no | `"bar"` `"line"` `"area"` (default: `"bar"`) |
| `color` | no | Hex color for the series (default: theme primary) |
| `chart_height` | no | Chart height in px (default: `80`) |

`chart_data` must be a JSON-serialised list. Generate it in your callback:

```python
import json
chart_data = json.dumps([10, 25, 18, 40, 32, 27, 45])
```

---

### `_chart_card.html` — Full ApexCharts card

A card with a full-size ApexCharts chart. Dark-mode styles are applied automatically. **Requires ApexCharts.**

```html
{% include "admin/widgets/_chart_card.html" with
    chart_id="unique_id"
    chart_config=my_chart_config_json
    title="..."
    subtitle="..."
    height=280
%}
```

| Variable | Required | Description |
|---|---|---|
| `chart_id` | yes | Unique HTML element ID |
| `chart_config` | yes | ApexCharts config as a JSON string |
| `title` | no | Card header title |
| `subtitle` | no | Card header subtitle |
| `height` | no | Chart height in px (default: `250`) |
| `extra_value` | no | Large value shown in the header |
| `extra_badge` | no | Badge text next to `extra_value` |
| `extra_badge_color` | no | Bootstrap color for the badge (default: `success`) |
| `menu_items` | no | List of `{"label": "...", "url": "..."}` dicts for a header dropdown |

`chart_config` is a standard ApexCharts options object serialised to JSON. You only need to provide the series and type — dark-mode axis colors, grid, and tooltip theme are merged in automatically:

```python
import json

chart_config = json.dumps({
    "chart": {"type": "bar"},
    "series": [{"name": "Value", "data": [30, 40, 35, 50, 49, 60]}],
    "xaxis": {"categories": ["Jan", "Feb", "Mar", "Apr", "May", "Jun"]},
    "colors": ["#7367f0"],
    "plotOptions": {"bar": {"columnWidth": "45%", "borderRadius": 4}},
    "dataLabels": {"enabled": False},
})
```

---

### `_progress_card.html` — Progress bar breakdown

A card with a header stat and multiple labelled progress bars — useful for showing how a total breaks down into components.

```html
{% include "admin/widgets/_progress_card.html" with
    title="..."
    subtitle="..."
    total_value="$45.3k"
    total_badge="+12%"
    metrics=my_metrics_list
%}
```

| Variable | Required | Description |
|---|---|---|
| `title` | yes | Card title |
| `metrics` | yes | List of metric dicts (see below) |
| `subtitle` | no | Card subtitle |
| `total_value` | no | Large value in the header |
| `total_badge` | no | Badge text next to `total_value` |
| `total_badge_color` | no | Bootstrap color (default: `success`) |

Each item in `metrics` is a dict:

```python
metrics = [
    {"label": "Category A", "value": "$30k", "pct": 66, "color": "primary",  "icon": "ti-circle"},
    {"label": "Category B", "value": "$10k", "pct": 22, "color": "success",  "icon": "ti-circle"},
    {"label": "Category C", "value": "$5k",  "pct": 11, "color": "danger",   "icon": "ti-circle"},
]
```

| Key | Required | Description |
|---|---|---|
| `label` | yes | Metric name |
| `value` | yes | Formatted value string |
| `pct` | yes | Integer 0–100 for the progress bar width |
| `color` | no | Bootstrap color name (default: `primary`) |
| `icon` | no | Tabler icon class shown in the badge |

---

### `_apex_scripts.html` — ApexCharts loader

Include this **once per page** on any page that uses `_sparkline_card.html` or `_chart_card.html`.

```html
{% block extrastyle %}{{ block.super }}
  {% include "admin/widgets/_apex_scripts.html" %}
{% endblock %}
```

---

## Building and publishing

```bash
# Install build tools
pip install build twine

# Build source + wheel distributions
python -m build

# Upload to PyPI
twine upload dist/*

# Or install locally for development
pip install -e .
```
