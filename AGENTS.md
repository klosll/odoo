# AGENTS.md - Reglas de desarrollo para Odoo 14 PAASA

## Convenciones para módulos klo_

Todos los módulos cuyo nombre comience con `klo_` deben incluir obligatoriamente:

```
static/description/icon.png
```

Este archivo se copia desde cualquier módulo `klo_` existente como plantilla.

Estructura mínima de un módulo `klo_`:

```
klo_<nombre_modulo>/
├── __init__.py
├── __manifest__.py
├── models/
│   ├── __init__.py
│   └── *.py
├── views/
│   └── *.xml
└── static/
    └── description/
        └── icon.png
```
