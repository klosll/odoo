# AGENTS.md — Odoo 18 Development (KLO)

## Big Picture

This is an **Odoo 18** development environment managed by **KLO Ingeniería Informática S.L.L.**  
The workspace root `/opt/odoo18_desarrollo/` contains:

```
/opt/odoo18_desarrollo/
├── odoo/              ← Odoo core (this repo)
├── extra-addons/
│   ├── klo/           ← ALL custom KLO development lives here
│   │   ├── _templates/    ← Scaffolding: icon.png + Tecnical_document.md to copy
│   │   ├── extra/         ← Generic, reusable KLO modules (prefix: klo_)
│   │   ├── klo_garridomontero/  ← Customer-specific: Garridomontero (single module)
│   │   ├── klo_myv/       ← Customer-specific: MyV (single module)
│   │   ├── oca/           ← KLO-maintained forks of OCA repos (e.g. oca-account-reconcile)
│   │   ├── avz-odoo-addons/    ← Fork: Avanzosc addons
│   │   ├── odoo-B10/      ← Fork: B10 modules
│   │   ├── infobit/       ← Fork: Infobit modules
│   │   └── odooapps/      ← Third-party apps (om_*)
│   ├── oca/           ← OCA repos (read-only, updated via script)
│   ├── odoo-attrs-replace/ ← Migration utility: converts Odoo ≤16 attrs=/states= XML syntax to Odoo 17+ individual attributes
│   └── temporales/    ← Short-lived experimental modules
├── config/odoo.conf   ← Server configuration (edit here to change DB, paths)
├── uv/                ← uv project: pyproject.toml + uv.lock + .venv (Python environment)
├── hdr/               ← EDI/XML header files (e.g. FacturaE)
├── log/               ← odoo.log written here
└── data/              ← Odoo filestore and sessions
```

## Running the Server

The project uses **uv** for dependency management. The virtual environment lives at `/opt/odoo18_desarrollo/uv/.venv/`.

```bash
# Standard start (uses JetBrains run config "Odoo 18 desarrollo")
python odoo-bin -c /opt/odoo18_desarrollo/config/odoo.conf

# Install a new module
python odoo-bin -c /opt/odoo18_desarrollo/config/odoo.conf -d ryp_dev -i klo_my_module --stop-after-init

# Update an existing module (most common dev task)
python odoo-bin -c /opt/odoo18_desarrollo/config/odoo.conf -d ryp_dev -u klo_my_module --stop-after-init

# Run tests for a module
python odoo-bin -c /opt/odoo18_desarrollo/config/odoo.conf -d ryp_dev --test-enable -u klo_my_module --stop-after-init

# Alternative: invoke via uv run (matches the _templates/ example commands)
/home/manolo/.local/bin/uv run /opt/odoo18_desarrollo/uv/.venv/bin/python3 \
    /opt/odoo18_desarrollo/odoo/odoo-bin \
    -c /opt/odoo18_desarrollo/config/odoo.conf \
    -d ryp_dev -u klo_my_module --stop-after-init
```

Web UI: `http://localhost:8018` · Active DB: set in `config/odoo.conf` → `db_name`  
Current active databases: `ryp_dev`, `myv_dev`, `proyecta79_dev`, `victorperez_dev`, `viliman_dev` (comment/uncomment to switch).

## Creating a New KLO Module

New generic modules go in `extra-addons/klo/extra/` with prefix `klo_`.  
Customer-specific modules get their own directory (e.g., `extra-addons/klo/klo_<customer>/`).

Mandatory structure:
```
klo_my_module/
├── __init__.py
├── __manifest__.py
├── models/
│   ├── __init__.py
│   └── my_model.py
└── views/
    └── my_model_views.xml
```

**`__manifest__.py` template:**
```python
# Copyright 2026 KLO Ingeniería Informática S.L.L.
# License AGPL-3.0 or later (http://www.gnu.org/licenses/agpl).
{
    "name": "KLO - Module Description",
    "version": "18.0.1.0.0",
    "summary": "Short description in Spanish",
    "license": "AGPL-3",
    "author": "KLO Ingenieria Informatica S.L.L.",
    "website": "https://www.klo.es",
    "category": "...",
    "depends": ["base_module"],
    "data": ["views/my_model_views.xml"],
    "installable": True,
    "auto_install": False,
}
```

## Model Extension Pattern

KLO modules **always extend** existing Odoo models via `_inherit` — never create standalone apps.

```python
# models/sale_order.py
from odoo import api, fields, models

class SaleOrder(models.Model):
    _inherit = "sale.order"

    my_field = fields.Boolean(string="Mi Campo", default=False, copy=False, tracking=True)
```

## Addons Path Priority

The `addons_path` in `config/odoo.conf` controls load order. KLO modules listed **before** OCA repos will override them. When a KLO module needs to override an OCA module, ensure the KLO path appears earlier in the list.

The actual load order (from `config/odoo.conf`) is, highest priority first:
1. `extra-addons/` (root — modules placed directly here)
2. `odoo/addons`
3. `extra-addons/temporales`
4. `extra-addons/klo` (and its sub-paths: `avz-odoo-addons`, `extra`, `infobit`, `infobit/stock-infobit`, `klo/oca/oca-account-reconcile`, `odoo-B10`, `odooapps`)
5. `extra-addons/oca/*` (read-only OCA repos)

## Updating External Repositories

```bash
# Update all OCA repos
/opt/odoo18_desarrollo/extra-addons/oca/actualizar_repositorios_oca.sh

# Update Infobit forks
/opt/odoo18_desarrollo/extra-addons/klo/infobit/actualizar_repositorios_fork_infobit.sh
```

> **`odoo-attrs-replace` utility**: If working on modules migrated from Odoo ≤16, use
> `/opt/odoo18_desarrollo/odoo/extra-addons/odoo-attrs-replace/replace_attrs.py` to convert
> legacy `attrs=`/`states=` XML attributes to the Odoo 17+ individual (`invisible`, `readonly`,
> `required`, `column_invisible`) syntax. Run `python3 replace_attrs.py` and supply the module path.

## Key Conventions

- **Language**: Code comments and summaries in **Spanish**; Python identifiers in English.
- **License header**: Always `AGPL-3.0 or later` + KLO copyright on new files.
- **Version format**: `18.0.<major>.<minor>.<patch>` (e.g., `18.0.1.0.0`).
- **Report overrides**: XML reports go in `report/` or `reports/` subdirectory; inherit via `inherit_id`.
- **Queue jobs**: `queue_job` is loaded server-wide (`server_wide_modules = web,queue_job`). Use OCA's `job` decorator for async tasks.
- **No demo data**: KLO modules do not include `demo/` directories.

## Mandatory assets for every `klo_*` module

Every module whose technical name starts with `klo_` **must** include:

### 1. Module icon
Copy the shared KLO icon from the scaffolding template:
```bash
cp /opt/odoo18_desarrollo/odoo/extra-addons/klo/_templates/static/description/icon.png \
   <new_module>/static/description/icon.png
```

### 2. `Technical_context.md` / `Tecnical_document.md`
Create `static/description/Technical_context.md` documenting the module for future AI agents.
A ready-to-fill scaffold exists at `extra-addons/klo/_templates/static/description/Tecnical_document.md`
(copy and rename it; existing modules in `extra/` use the name `Technical_context.md`).
Required sections:
- **Módulo** — metadata table (name, version, author, license, path)
- **Descripción** — what the module does and why
- **Campo(s) añadido(s)** — each field: model, type, string, behaviour
- **Dependencias** — Odoo modules and paths of OCA/external modules
- **Lógica** — key methods overridden, triggers, conditions
- **Vistas modificadas** — which views are inherited, what is added and where
- **Estructura de archivos** — directory tree
- **Instalación / Actualización** — exact CLI commands
- **Posibles adaptaciones futuras** — extension hints for future devs/AI

Reference example: `extra-addons/klo/extra/klo_purchase_order_supplierinfo_date_update/static/description/Technical_context.md`

