# AGENTS.md - Reglas de desarrollo para Odoo 14 PAASA

## Entorno
- Versión de Odoo: 14
- Python: 3.10 (venv en /opt/odoo14_paasa/venv)
- Base de datos de desarrollo: topapro_dev
- Usuario PostgreSQL: por defecto (db_user = False en odoo.conf)
- Comando de arranque: /opt/odoo14_paasa/venv/bin/python odoo-bin -c /opt/odoo14_paasa/config/odoo14_paasa.conf
- addons_path: /opt/odoo14_paasa/extra-addons y /opt/odoo14_paasa/odoo/addons

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

## Operativa de prueba
- Instalación:  /opt/odoo14_paasa/venv/bin/python odoo-bin -d topapro_dev -i <modulo> --stop-after-init
- Actualización: /opt/odoo14_paasa/venv/bin/python odoo-bin -d topapro_dev -u <modulo> --stop-after-init
- Ante un error, localizar el fichero:línea de la traza antes de corregir.
