# AGENTS.md — Odoo 15 KLO

## Descripción general

Instancia de **Odoo 15** altamente personalizada para **KLO Ingeniería Informática S.L.L.** con tres capas de módulos por encima del núcleo:

| Capa | Ruta | Descripción |
|---|---|---|
| Núcleo Odoo | `addons/` | Módulos oficiales de Odoo 15 (no modificar directamente) |
| OCA | `extra-addons/oca/` | Módulos de Odoo Community Association; solo se actualizan vía `git pull` |
| KLO propios | `extra-addons/klo/klo_klo/`, `extra-addons/klo/extra/`, `extra-addons/klo/extra_privativo/` | Módulos de negocio desarrollados por KLO |
| Avanzosc portados | `extra-addons/klo_avanzosc/` | Módulos v14 de AvanzOSC portados a v15 por KLO |
| B10 | `extra-addons/klo/odoo-B10/` | Módulos de Batista10 (cliente externo) |

## Cómo arrancar el servidor

```bash
# Configuración de desarrollo (puerto 8015, BD klo_dev)
/opt/odoo15_klo/odoo/odoo-bin -c /opt/odoo15_klo/odoo/config/odoo15.conf

# Actualizar un módulo concreto sin reiniciar todo
/opt/odoo15_klo/odoo/odoo-bin -c /opt/odoo15_klo/odoo/config/odoo15.conf -u <nombre_modulo> --stop-after-init

# Instalar un nuevo módulo
/opt/odoo15_klo/odoo/odoo-bin -c /opt/odoo15_klo/odoo/config/odoo15.conf -i <nombre_modulo> --stop-after-init
```

- **Puerto HTTP**: 8015 | **BD**: `klo_dev` | **Datos**: `/opt/odoo15_klo/data/`
- `workers = 0` → modo mono-proceso (desarrollo); cambiar para producción.

## Depuración en VS Code (F5 / Ctrl+F5)

- Configuraciones en `.vscode/launch.json`:
  - **Odoo: Ejecutar servidor** (`noDebug: true`) → arranque normal con Ctrl+F5.
  - **Odoo: Debug** (`--dev all`) → depuración con F5.
- `.vscode/shortcuts.json` alimenta los botones de la extensión **Odoo Shortcuts** (mvintg.odoo-file): las dos configuraciones con `odooBinPath`, `odooConfPath` y `config`; la activa marca `"active": true`.
- `debugpy` debe estar instalado en el venv usado por VS Code: `/opt/odoo15_klo/odoo/venv` (symlink → `/opt/odoo15_klo/venv`).

### Pitfall conocido: "Depuración detenida" al pulsar Ctrl+F5

**Causa**: la extensión Python de VS Code guarda en su estado (`state.vscdb`, clave `venv:WORKSPACE_SELECTED`) una selección explícita de intérprete para la carpeta `/opt/odoo15_klo/odoo`. Esa selección tiene **prioridad sobre `python.defaultInterpreterPath`**. Si el venv seleccionado está roto (p. ej. `odoo/venv` con el symlink `python -> /usr/bin/python3.10` que ya no existe en el sistema), el debuggee no arranca y VS Code muestra "Depuración detenida".

**Diagnóstico**: en la pestaña **Output → Python Debugger**, `resolvedInterpreterPath` apuntaba al venv roto.

**Solución aplicada**: `odoo/venv` es ahora un symlink a `/opt/odoo15_klo/venv` (venv funcional), y `.vscode/settings.json` fija `python.defaultInterpreterPath`. Al recargar la ventana (Reload Window) la selección guardada ya resuelve a un intérprete válido.

**Cómo verificarlo**: simular el lanzamiento exacto de VS Code con una sesión DAP real: `python -m debugpy.adapter` + requests `initialize`/`launch`/`configurationDone` (el debuggee queda pausado hasta `configurationDone`, es normal), y comprobar que odoo responde HTTP 200 en 8015.

## Actualizar repositorios OCA

```bash
bash /opt/odoo15_klo/odoo/extra-addons/oca/actualizar_repositorios_oca_klo.sh
```

## Estructura de un módulo KLO

```
klo_<nombre>/
├── __manifest__.py   # versión patrón: "15.0.X.Y.Z", license: "AGPL-3"
├── __init__.py
├── models/           # herencias con _inherit; comentarios "# KLO." marcan cambios
├── views/            # XML de herencia con <xpath>; inherit_id con ref=
├── report/           # QWeb PDF/XLS
├── security/         # ir.model.access.csv + security.xml
├── i18n/             # .po para traducciones
└── static/           # JS/CSS
```

## Convenciones de código KLO

- **Todos los cambios sobre modelos estándar** usan `_inherit`; nunca se modifica el núcleo directamente.
- Los comentarios de negocio específicos de KLO llevan el prefijo `# KLO.` para facilitar el `grep`.
- Al sobrescribir métodos se llama siempre a `super()` excepto cuando la intención explícita es *cancelar* el comportamiento del padre (ej. `klo_project_no_auto_analytic`).
- Los campos relacionados se añaden con `related=` y `store=True` solo cuando se necesita filtrar/ordenar en BD.
- Las vistas heredadas usan `<xpath>` con `position="after|before|replace"` dentro de `<field name="inherit_id" ref="..."/>`.

## Patrones de extensión frecuentes

```python
# Añadir campo a res.users (patrón usado en klo_klo y stock_inventory_adjustment)
class KloResUsers(models.Model):
    _inherit = "res.users"
    analytic_account_id = fields.Many2one('account.analytic.account', ...)

# Cancelar comportamiento padre (klo_project_no_auto_analytic)
def _create_analytic_account(self):
    return  # Sin llamar a super()

# Importación Excel en wizard (stock_inventory_adjustment/models/xls.py)
# Usa pandas + base64; columnas obligatorias: CODE, QTY; opcional: LOT
```

## Módulos KLO clave

| Módulo | Ruta | Qué hace |
|---|---|---|
| `klo_klo` | `extra-addons/klo/klo_klo/` | Adaptación central: timesheets, proyectos, facturas, pedidos de venta |
| `klo_xf_sale_project` | `extra-addons/klo/extra/klo_xf_sale_project/` | Recomputa `invoice_ids` del proyecto para multi-pedido; pasa `payment_term_id` al crear pedido desde proyecto |
| `klo_project_no_auto_analytic` | `extra-addons/klo/extra/klo_project_no_auto_analytic/` | Desactiva la creación automática de cuenta analítica en proyectos |
| `stock_inventory_adjustment` | `extra-addons/klo/extra_privativo/stock_inventory_adjustment/` | Ajuste de inventario con contraseña de validación, tipos (all/categ/one/xls) e importación desde Excel |
| `klo_purchase_invoice_final_partner` | `extra-addons/klo/extra/klo_purchase_invoice_final_partner/` | Añade "partner final" a compras y facturas; lo propaga a apuntes analíticos |
| `klo_invoice_number_editable` | `extra-addons/klo/extra/klo_invoice_number_editable/` | Permite editar el número de factura ya publicada |
| `klo_analytic_lines_by_partner_category` | `extra-addons/klo/extra/klo_analytic_lines_by_partner_category/` | Filtra apuntes analíticos por categoría de contacto |

## Integración usuario-cuenta analítica

`res.users` lleva el campo `analytic_account_id`. Al crear un `project.project`, el método `create()` de `KloProjectProject` asigna automáticamente la cuenta analítica del usuario responsable (ver `extra-addons/klo/klo_klo/models/project_project.py`).

## Dependencias Python no estándar

- `pandas` — importación Excel en `stock_inventory_adjustment`
- `lxml` — requerido en `klo_account_payment_order_total_amount` y timesheets
- `pytz` — manejo de zonas horarias en timesheets (`klo_klo/models/hr_timesheet.py`)

## Licencias y autoría

- Módulos propios de KLO: **AGPL-3** (`author: KLO Ingeniería Informática S.L.L.`)
- Módulos privativos en `extra_privativo/`: licencias propietarias (OPL-1 u otras); **no publicar**.
- Los módulos OCA mantienen su licencia original (AGPL-3 / LGPL-3).

