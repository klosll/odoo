# Propuesta de Entorno Multi-Agente en OpenCode para Odoo y su Sistema (PostgreSQL, Ubuntu, librerías)

**Fecha:** 17 de agosto de 2026
**Autor:** Manolo (entorno PAASA – Odoo 14)
**Herramientas base:** OpenCode (TUI/CLI), OpenCode Go (suscripción de bajo coste) y OpenCode Zen (pago por uso + modelos gratuitos)

---

## 1. Objetivo

Definir un entorno **multi-agente** en OpenCode donde un agente **Supervisor/Entrenador** controle, revise y entrene a un agente **Ejecutor/Aprendiz** para el trabajo profesional con **distintas versiones de Odoo**, y para el **mantenimiento del sistema en el que se ejecuta**. El entorno cubre:

- Adaptación y programación de módulos (convención `klo_*`, estructura mínima, icono `static/description/icon.png`).
- Mantenimiento y evolución de módulos entre versiones (13, 14, 15, 16, 17 y posteriores), incluida la parte JavaScript/OWL de Odoo 15+.
- Pruebas de operativa (instalación `-i`, actualización `-u`, escenarios de datos, tests).
- Configuración del sistema y de Odoo (`odoo.conf`, workers, cron, servicios systemd).
- Administración de **PostgreSQL** (bases, backups `pg_dump`, rendimiento, `EXPLAIN ANALYZE`, índices).
- Administración de **Ubuntu** y **librerías** (paquetes `apt`, entorno Python/venvs, dependencias, espacio en disco, logs).
- Solución de errores (log de Odoo, trazas Python, SQL, vistas/QWeb, assets, errores de sistema).

El documento incluye: recomendación de agentes para **OpenCode Go**, un apartado con los **agentes que usas actualmente** y las razones por las que la propuesta es mejor, un apartado específico con **modelos gratuitos de OpenCode Zen**, un ejemplo completo de configuración y flujos de trabajo de ejemplo.

---

## 2. Arquitectura multi-agente propuesta

Se propone un patrón **Actor-Crítico / Supervisor-Ejecutor**, maduro y recomendado para entornos de entrenamiento autónomo de IA sobre IA:

```
                    +---------------------------------------------+
                    |  SUPERVISOR / ENTRENADOR (agente primario)   |
  Usuario --------> |   - Planifica, desglosa tareas              |
                    |   - Delega (herramienta Task)               |
                    |   - Revisa y evalúa entregas                |
                    |   - Corrige y "entrena" al aprendiz         |
                    +---------------------+-----------------------+
                                          | contrato permission.task
              +---------------------------+---------------------------+
              |                           |                           |
   +----------v-----------+   +-----------v-----------+   +----------v----------+
   | EJECUTOR / APRENDIZ  |   |  SISTEMA / INFRA      |   | AGENTES DE APOYO     |
   | programacion Odoo    |   | PostgreSQL, Ubuntu,   |   | explorador, revisor, |
   | (modulos klo_*)      |   | librerias, servicios  |   | probador (read/exec) |
   +----------------------+   +-----------------------+   +----------------------+
```

### 2.1 Agente Supervisor / Entrenador (crítico)

- **Papel:** Recibe el requerimiento del usuario, lo analiza, lo desglosa en subtareas, delega en los ejecutores, revisa el resultado (calidad, seguridad, compatibilidad entre versiones de Odoo), rechaza entregas y formula instrucciones de corrección. Actúa como QA senior, formador y guardián del estándar del proyecto (`AGENTS.md`, convenciones `klo_*`).
- **Modelo:** alta capacidad de razonamiento y contexto largo (para leer logs extensos de Odoo y diffs). **Baja temperatura** (0.1-0.2) para determinismo.
- **Herramientas:** lectura, búsqueda y ejecución de comandos para validar, pero **sin necesidad de editar** directamente: su herramienta principal es `task` para delegar. Su `permission.task` define el contrato con los subagentes permitidos.

### 2.2 Agente Ejecutor / Aprendiz (actor)

- **Papel:** ejecuta el bloque de trabajo asignado: localizar código, escribir/adaptar el módulo (`__manifest__.py`, `models/`, `views/`, `static/`, JS/OWL), aplicar retroalimentación del Supervisor, ejecutar pruebas y corregir errores de forma iterativa.
- **Modelo:** orientado a codificación (Python + XML + JS/OWL), equilibrado entre calidad y coste. **Temperatura media** (0.2-0.3).
- **Herramientas:** edición de archivos, terminal (git, `odoo-bin -i/-u`, pruebas). Limitado al ámbito del proyecto.

### 2.3 Agente Sistema / Infraestructura (nuevo, clave para tu objetivo)

- **Papel:** mantiene el sistema sobre el que corre Odoo: **PostgreSQL** (bases, backups, rendimiento, índices, análisis de consultas), **Ubuntu** (paquetes `apt`, servicios `systemd`, permisos, espacio en disco, logs) y **librerías Python** (venvs, `requirements`, dependencias). Ejecuta cambios con confirmación del Supervisor y documenta cada acción.
- **Modelo:** equilibrado en código y administración de sistemas (bash/SQL), buena cuota de uso.
- **Herramientas:** edición de ficheros de configuración y terminal; comandos sensibles (`apt*`, `systemctl*`, `psql*`, `rm*`, `pg_dump*`) con `ask`.

### 2.4 Agentes de apoyo (mejoran la calidad)

- **`explorador-odoo`**: búsqueda y lectura de código (modo *read-only*). Ideal para rastrear cómo está implementado un campo/vista/API en varias versiones de Odoo.
- **`revisor-odoo`**: revisión de calidad, seguridad y buenas prácticas del código entregado (sin editar).
- **`probador-odoo`**: ejecuta la operativa real (instalación/actualización de módulos, tests, comprobación de logs y errores) y reporta trazas exactas.

> **Nota de seguridad:** los subagentes no reciben órdenes directas del usuario; solo el Supervisor (vía `permission.task`) los invoca. Esto evita conflictos y garantiza el contrato de supervisión. Con `hidden: true` se ocultan del menú `@` si se desea.

### 2.5 Mapa de tareas a agente

| Tarea (ejemplo) | Agente responsable |
|---|---|
| Adaptar módulo `klo_*` entre versiones de Odoo | `aprendiz-odoo` (o `aprendiz-odoo-hard` en migraciones críticas) |
| Localizar dónde se define un campo/vista/API en el núcleo | `explorador-odoo` |
| Revisar seguridad y adherencia a convenciones antes de aprobar | `revisor-odoo` |
| Instalar/actualizar un módulo y comprobar logs | `probador-odoo` |
| Backup, rendimiento o consultas lentas en PostgreSQL | `sistema-odoo` |
| Actualizar paquetes `apt`, servicio `odoo`, `odoo.conf`, venvs/librerías | `sistema-odoo` |
| Planificar, delegar, revisar y entrenar | `supervisor-odoo` |

---

## 3. Modelos recomendados en OpenCode Go

OpenCode Go es una suscripción de bajo coste (5 $ el primer mes, 10 $/mes después) con acceso a modelos abiertos seleccionados. Los identificadores usan el prefijo `opencode-go/<modelo>`.

| Modelo (ID Go) | Rol recomendado | Justificación |
|---|---|---|
| `opencode-go/glm-5.2` | **Ejecutor / Aprendiz principal (recomendado)** | De los mejores modelos "open" en programación, con gran equilibrio Python + XML + JS/OWL (imprescindible para Odoo 15+). Cuota alta (4.300 req/mes). El mejor coste/calidad para el desarrollo diario de módulos. |
| `opencode-go/glm-5.3` | Ejecutor de alta calidad (migraciones complejas) | Mayor capacidad de código que 5.2, pero cuota baja (1.080 req/mes). Reservarlo para tareas puntuales difíciles. |
| `opencode-go/qwen3.7-max` | Ejecutor para problemas muy difíciles | Excelente codificación y razonamiento; cuota muy baja (340 req/5h). Úsalo solo en migraciones críticas o depuración profunda. |
| `opencode-go/kimi-k2.7-code` | Ejecutor de volumen / economía | Especializado en código, muy barato y con cuota alta (6.750 req/mes). Ideal para iteraciones rápidas y tareas repetitivas del aprendiz. |
| `opencode-go/deepseek-v4-pro` | **Supervisor / Entrenador + Revisor (recomendado)** | Razonamiento sólido y contexto amplio (para logs y diffs grandes) a coste muy bajo, especialmente en horas valle. |
| `opencode-go/kimi-k3` | Supervisor de máxima calidad (puntual) | Razonamiento superior para planificar arquitectura y migraciones; cuota muy baja (110 req/5h), usar con moderación. |
| `opencode-go/grok-4.5` | Supervisor alternativo de máxima calidad | Gran razonamiento; cuota baja (120 req/5h). Alternativa a `kimi-k3`. |
| `opencode-go/deepseek-v4-flash` | Explorador + tareas de volumen | Muy rápido y económico (3.800 req/5h); ideal para búsquedas e iteraciones. |
| `opencode-go/gpt-5.6-luna` | Probador / operativa | Rápido y barato (2.050 req/5h); buen uso de herramientas para instalar/actualizar y validar. |

### Asignación final recomendada (configuración "tipo")

| Agente | Modelo Go | Temperatura | Modo |
|---|---|---|---|
| Supervisor / Entrenador | `opencode-go/deepseek-v4-pro` | 0.1 | `primary` |
| Ejecutor / Aprendiz (programación) | `opencode-go/glm-5.2` | 0.2 | `subagent` |
| Ejecutor alta calidad (migraciones) | `opencode-go/glm-5.3` | 0.2 | `subagent` |
| Sistema / Infraestructura | `opencode-go/glm-5.2` | 0.2 | `subagent` |
| Explorador Odoo | `opencode-go/deepseek-v4-flash` | 0.1 | `subagent` |
| Explorador Odoo (free) | `opencode/deepseek-v4-flash-free` | 0.1 | `subagent` |
| Revisor Odoo | `opencode-go/deepseek-v4-pro` | 0.1 | `subagent` |
| Probador Odoo | `opencode-go/gpt-5.6-luna` | 0.2 | `subagent` |
| Probador Odoo (free) | `opencode/deepseek-v4-flash-free` | 0.2 | `subagent` |

> **Consejo de coste:** usa `kimi-k3` o `grok-4.5` solo cuando el problema lo exija. `deepseek-v4-pro` en horas valle (fuera de 01:00-04:00 y 06:00-10:00 UTC) cuesta la mitad. `kimi-k2.7-code` y `deepseek-v4-flash` absorben el volumen diario sin gastar la cuota de los modelos premium.

---

## 4. Tu entorno actual vs. esta propuesta

### 4.1 Qué usas actualmente

Diagnóstico realizado sobre tu configuración (17-08-2026):

- **Agentes:** los integrados de OpenCode (`build`, `plan`, `general`, `explore`, `scout`), sin agentes personalizados.
- **Modelo único:** `opencode-go/deepseek-v4-flash` (OpenCode Go) para todo.
- **Configuración mínima:** el `opencode.json` del proyecto solo define `"permission": { "*": "allow" }`; no hay `AGENTS.md` específico para el agente ni skills de Odoo.

Con ese planteamiento, un único agente con un único modelo hace **todas** las tareas: planificar, programar, probar y administrar el sistema.

### 4.2 Tabla comparativa

| Aspecto | Entorno actual | Propuesta |
|---|---|---|
| Agentes | 1 rol genérico (build) | Supervisor + Ejecutor + Sistema + apoyo (6 roles) |
| Modelo | `deepseek-v4-flash` para todo | Modelo por rol: `pro` (razonar/revisar), `glm-5.2` (programar), `flash` (buscar), `luna` (probar) |
| Control de calidad | El propio agente se auto-aprueba | Contrato supervisor-ejecutor con revisión y rechazo |
| Contexto Odoo | Ninguno (adivina APIs) | Skill `odoo-dev` + prompts por versión + explorador del núcleo |
| Mantenimiento del sistema | No diferenciado | Agente `sistema-odoo` (PostgreSQL, Ubuntu, librerías) |
| Permisos | `"*": allow` (todo abierto) | Permisos por agente; comandos sensibles con `ask` |
| Coste/uso | Un modelo quema la cuota de Go | Distribución de carga + modelos free de Zen de respaldo |

### 4.3 Por qué la propuesta es mejor

1. **Modelo adecuado a cada tarea.** `deepseek-v4-flash` es excelente ejecutando, pero es un modelo de menor capacidad de razonamiento: como único agente planifica y revisa mal en problemas complejos (migraciones entre versiones, depuración profunda). En la propuesta, el **razonamiento** (planificar, revisar, entrenar) lo asume `deepseek-v4-pro` (o `kimi-k3` puntualmente), y la **ejecución rápida** la hace `flash`. La **programación de módulos** la hace `glm-5.2`, superior en código Python/XML/JS/OWL y con mucha más cuota.
2. **Separación de roles y QA independiente.** El patrón Actor-Crítico evita que el mismo agente que escribe se auto-apruebe: el Supervisor revisa, rechaza y devuelve reportes de corrección. Esto es clave en Odoo, donde un campo mal definido o una vista mal migrada rompe la instalación en producción.
3. **Contexto específico de Odoo.** Los agentes actuales "adivinan" las APIs de cada versión. La propuesta incluye la skill `odoo-dev` (diferencias 13-18+, estructura `klo_*`, operativa `-i/-u`) y un `explorador-odoo` que lee el núcleo para no inventar APIs.
4. **Mantenimiento real del sistema.** Tu objetivo incluye PostgreSQL, Ubuntu y librerías: el agente `sistema-odoo` se encarga de backups, rendimiento de consultas, paquetes, servicios y venvs, bajo supervisión y con permisos controlados. Hoy eso cae en el mismo agente genérico, sin reglas ni documentación.
5. **Seguridad y control.** `"*": allow` permite que el agente haga acciones destructivas sin avisar (`rm`, `git push`, `pg_dropdb`, `apt remove`). La propuesta usa permisos por agente y `ask` para comandos sensibles, además de `steps` para evitar bucles y `/undo` para revertir.
6. **Optimización de costes y continuidad.** Repartir el trabajo entre modelos de distinto precio y usar los modelos free de Zen (apartado 9) evita agotar la cuota mensual de Go y da respaldo cuando se alcanzan los límites.

---

## 5. Configuración de ejemplo del entorno

Los agentes se definen como **archivos markdown** (recomendado para configuraciones no triviales) en:

- Global: `~/.config/opencode/agent/<nombre>.md` (o `agents/`)
- Proyecto: `.opencode/agent/<nombre>.md` (o `agents/`)

### 5.1 Configuración global `~/.config/opencode/opencode.json`

```json
{
  "$schema": "https://opencode.ai/config.json",
  "model": "opencode-go/deepseek-v4-pro",
  "small_model": "opencode-go/deepseek-v4-flash",
  "default_agent": "supervisor-odoo",

  "permission": {
    "edit": "ask",
    "bash": {
      "git *": "allow",
      "*": "ask"
    },
    "external_directory": {
      "*": "allow"
    }
  },

  "agent": {
    "supervisor-odoo": {
      "description": "Supervisor y entrenador para Odoo multiversión y su sistema.",
      "mode": "primary",
      "model": "opencode-go/deepseek-v4-pro",
      "temperature": 0.1,
      "color": "primary"
    },
    "aprendiz-odoo": {
      "description": "Ejecutor/aprendiz que programa y mantiene módulos Odoo.",
      "mode": "subagent",
      "model": "opencode-go/glm-5.2",
      "temperature": 0.2,
      "permission": {
        "bash": {
          "*": "ask",
          "git *": "allow",
          "python odoo-bin*": "allow"
        }
      }
    },
    "sistema-odoo": {
      "description": "Administra el sistema de Odoo: PostgreSQL, Ubuntu, librerías y servicios.",
      "mode": "subagent",
      "model": "opencode-go/glm-5.2",
      "temperature": 0.2,
      "permission": {
        "edit": "ask",
        "bash": {
          "*": "ask",
          "git *": "allow",
          "python odoo-bin*": "allow"
        }
      }
    }
  }
}
```

> El `default_agent` apunta al Supervisor: al iniciar `opencode`, se trabaja siempre bajo la jerarquía de supervisión.

### 5.2 Archivo del Supervisor `~/.config/opencode/agent/supervisor-odoo.md`

```markdown
---
description: Supervisor y entrenador para Odoo multiversión y su sistema.
mode: primary
model: opencode-go/deepseek-v4-pro
temperature: 0.1
color: primary
permission:
  task:
    "*": deny
    aprendiz-odoo: allow
    aprendiz-odoo-hard: allow
    sistema-odoo: allow
    explorador-odoo: allow
    explorador-odoo-free: allow
    revisor-odoo: allow
    probador-odoo: allow
    probador-odoo-free: allow
---

Actúas como Ingeniero de QA Senior, Formador y Supervisor de un equipo de agentes que trabajan
con múltiples versiones de Odoo (13 a 18+) y con el sistema que lo sostiene.

Tus responsabilidades:

1. ANALIZAR el requerimiento del usuario, detectar la versión de Odoo afectada y los puntos de
   incompatibilidad (API antigua vs. nueva API, campos JSON, vistas, QWeb, assets, JS/OWL).
2. PLANIFICAR y desglosar la tarea en pasos concretos y asignables.
3. DELEGAR el trabajo en el subagente apropiado mediante la herramienta Task:
   - `aprendiz-odoo`: programación, adaptación y mantenimiento de módulos klo_*.
   - `aprendiz-odoo-hard`: migraciones complejas o depuración profunda (modelo superior).
   - `sistema-odoo`: PostgreSQL, Ubuntu, librerías, odoo.conf, servicios y cron.
   - `explorador-odoo`: búsquedas y lectura de código en el monorepo.
   - `revisor-odoo`: revisión de calidad y seguridad del código entregado.
   - `probador-odoo`: pruebas de operativa (instalación/actualización, logs, tests).
   - `explorador-odoo-free` / `probador-odoo-free`: variantes gratuitas (sin datos confidenciales).
4. REVISAR cada entrega contra los criterios de calidad: cumplimiento de AGENTS.md,
   convenciones klo_* (icono, estructura), compatibilidad entre versiones, seguridad, rendimiento.
5. ENTRENAR al aprendiz: cuando detectes errores, devuélvele un reporte claro con la causa,
   el contexto y las correcciones obligatorias. Repite el ciclo hasta validar.
6. CERRAR la tarea con un resumen de lo realizado, decisiones tomadas y pruebas ejecutadas.

Reglas de oro:
- No edites tú el código: delega y revisa.
- Exige que cada módulo klo_* incluya static/description/icon.png.
- Nunca apruebes entregas sin verificar que la operativa (install/update) funciona.
- Para cambios de sistema (backups, apt, servicios), exige confirmación y documentación.
```

### 5.3 Archivo del Ejecutor `~/.config/opencode/agent/aprendiz-odoo.md`

```markdown
---
description: Ejecutor y aprendiz que programa, adapta y mantiene módulos Odoo.
mode: subagent
model: opencode-go/glm-5.2
temperature: 0.2
permission:
  edit: allow
  bash:
    "*": ask
    "git *": allow
    "python odoo-bin*": allow
    "psql*": ask
  webfetch: allow
---

Actúas como Programador Junior de Odoo (aprendiz) bajo la supervisión de `supervisor-odoo`.

Tu misión es ejecutar los bloques de trabajo que te asigna el Supervisor:

1. Localizar el código afectado (módulos klo_* o núcleo) usando búsquedas dirigidas.
2. Escribir o adaptar el módulo respetando la estructura mínima:
   - __init__.py, __manifest__.py, models/__init__.py, models/*.py, views/*.xml
   - static/description/icon.png (obligatorio para klo_*)
   - Odoo 15+: ajusta el frontend JavaScript/OWL si el módulo lo requiere.
3. Mantener compatibilidad con la versión de Odoo indicada:
   - Odoo 14 o anterior: recordsets + nueva API, @api.model/@api.depends/@api.onchange.
   - Versiones nuevas: JSONB en place_type, cambios de vistas (tree/list), QWeb y assets.
4. Ejecutar pruebas de operativa cuando se te indique:
   - python odoo-bin -d <db> -i <modulo> --stop-after-init
   - python odoo-bin -d <db> -u <modulo> --stop-after-init
   - Revisar logs y trazas para corregir errores.
5. Aplicar SIEMPRE la retroalimentación del Supervisor: lee su reporte, corrige y vuelve a entregar.

Reglas de oro:
- No añadas comentarios innecesarios al código.
- No modifiques ficheros fuera del ámbito de la tarea asignada.
- Informa siempre de qué cambiaste, por qué y cómo lo comprobaste.
```

**Variante de alta calidad** `aprendiz-odoo-hard.md` (migraciones críticas):

```markdown
---
description: Ejecutor de alta capacidad para migraciones complejas y depuración profunda.
mode: subagent
model: opencode-go/glm-5.3
temperature: 0.2
permission:
  edit: allow
  bash:
    "*": ask
    "git *": allow
    "python odoo-bin*": allow
    "psql*": ask
---

Actúas como Programador Senior de Odoo. Igual que `aprendiz-odoo`, pero para tareas de
alta complejidad: migraciones entre versiones, reescrituras de API, problemas de rendimiento
o de concurrencia, y depuración profunda. Documenta las decisiones técnicas en detalle.
```

### 5.4 Agente Sistema / Infraestructura `~/.config/opencode/agent/sistema-odoo.md`

```markdown
---
description: "Administra el sistema de Odoo: PostgreSQL, Ubuntu, librerías y servicios."
mode: subagent
model: opencode-go/glm-5.2
temperature: 0.2
permission:
  edit: ask
  bash:
    "*": ask
    "git *": allow
    "python odoo-bin*": allow
    "grep *": allow
    "df -h": allow
  webfetch: allow
---

Actúas como Administrador de Sistemas Senior de la infraestructura Odoo, bajo la supervisión
de `supervisor-odoo`. Tu ámbito de trabajo es el SISTEMA, no el código de los módulos:

1. PostgreSQL: gestionar bases (crear, backup pg_dump/pg_restore, restaurar), revisar
   rendimiento (pg_stat_activity, EXPLAIN ANALYZE, índices), tamaños y espacio.
2. Ubuntu: instalar/actualizar paquetes apt, comprobar espacio en disco (df -h), permisos,
   usuarios y ficheros de log.
3. Servicios: estado y gestión del servicio Odoo (systemctl), configuración de odoo.conf
   (workers, db_user, addons_path, logfile) y cron del sistema.
4. Librerías Python: entornos virtuales (venv), requirements.txt, versiones de dependencias
   compatibles con la versión de Odoo.

Reglas de oro:
- No ejecutes acciones destructivas (borrar bases, apt remove, reinicios) sin confirmación
  explícita del Supervisor.
- Haz backup antes de modificar datos en producción.
- Documenta siempre qué cambiaste, el comando usado y cómo comprobarlo.
```

### 5.5 Agentes de apoyo (plantillas resumidas)

`explorador-odoo.md`:

```markdown
---
description: Explora y responde preguntas sobre el código Odoo (solo lectura).
mode: subagent
model: opencode-go/deepseek-v4-flash
temperature: 0.1
permission:
  edit: deny
  bash:
    "*": deny
    "git status *": allow
    "git log*": allow
---

Eres un explorador read-only. Busca en el repositorio (incluido el núcleo de Odoo)
implementaciones, campos, vistas o APIs. Devuelve rutas de archivo:línea y fragmentos relevantes.
No modifiques nada.
```

`revisor-odoo.md`:

```markdown
---
description: Revisa calidad, seguridad y compatibilidad del código Odoo sin editarlo.
mode: subagent
model: opencode-go/deepseek-v4-pro
temperature: 0.1
permission:
  edit: deny
  bash: deny
---

Eres un revisor senior. Analiza el código entregado y reporta:
seguridad (inyección SQL, seguridad de registros), rendimiento, adherencia a AGENTS.md,
convenciones klo_* y compatibilidad con la versión de Odoo objetivo. No edites código.
```

`probador-odoo.md`:

```markdown
---
description: "Ejecuta la operativa de Odoo: instalación, actualización, tests y análisis de logs."
mode: subagent
model: opencode-go/gpt-5.6-luna
temperature: 0.2
permission:
  edit: deny
  bash:
    "*": ask
    "python odoo-bin*": allow
    "git *": allow
    "grep *": allow
    "psql*": ask
---

Eres el encargado de pruebas de operativa. Ejecuta instalación (-i), actualización (-u),
tests y comprobación de logs. Reporta errores con la traza exacta y el fichero:línea.
No modifiques el código: solo ejecuta y reporta.
```

### 5.6 Skill de proyecto para Odoo (`.opencode/skills/odoo-dev/SKILL.md`)

```markdown
---
name: odoo-dev
description: "Usa cuando trabajes en módulos klo_* o en cualquier tarea de programación,
adaptación o mantenimiento de Odoo. Cubre estructura de módulos, compatibilidad entre
versiones y operativa de prueba."
---

# Desarrollo de módulos Odoo (convención klo_*)

## Estructura mínima de un módulo klo_*
klo_<nombre>/
|-- __init__.py
|-- __manifest__.py
|-- models/__init__.py, models/*.py
|-- views/*.xml
`-- static/description/icon.png   # OBLIGATORIO, copiar de cualquier klo_* existente

## Compatibilidad entre versiones
- Odoo 14 o anterior: recordsets + nueva API; usa @api.model, @api.depends, @api.onchange.
- Odoo 15+: JSONB en place_type, cambios en vistas (tree/list), QWeb, assets y JS/OWL.
- Antes de migrar un módulo: verificar __manifest__.py (version, depends, installable),
  cambios de API de modelos, renames de campos/vistas y módulos dependientes.

## Operativa de prueba
- La base de datos activa se define en el fichero `.conf` de `config/` con el parámetro `db_name`
  (cambia según el proyecto; no la hardcodees).
- Instalación:  python odoo-bin -d <db_name> -i klo_<modulo> --stop-after-init
- Actualización: python odoo-bin -d <db_name> -u klo_<modulo> --stop-after-init
- Revisa siempre el log; ante un error, localiza el fichero:línea de la traza antes de corregir.
```

### 5.7 Skill de sistema (`.opencode/skills/odoo-sysadmin/SKILL.md`)

```markdown
---
name: odoo-sysadmin
description: "Usa en tareas de administración del sistema Odoo: PostgreSQL, Ubuntu,
librerías, servicios y configuración. Complementa a odoo-dev."
---

# Administración del sistema Odoo

## PostgreSQL
- Backup:  pg_dump -h localhost -U <db_user> <db_name> -F c -f <fichero>.dump
  (`<db_name>` se lee de `db_name` y `<db_user>` de `db_user` en el `.conf` de `config/`.)
- Estado:  psql -c "SELECT datname, pg_size_pretty(pg_database_size(datname)) FROM pg_database;"
- Rendimiento: EXPLAIN ANALYZE sobre consultas lentas; revisar índices y pg_stat_activity.

## Ubuntu / servicios
- Servicio Odoo: systemctl status odoo / restart
- Paquetes: apt update && apt list --upgradable
- odoo.conf: workers, db_user, addons_path, logfile.

## Reglas de seguridad
- Nunca borrar bases ni reiniciar servicios sin confirmación del Supervisor.
- Hacer backup antes de modificar producción.
```

### 5.8 Comando útil (`.opencode/command/entrenar.md`)

```markdown
---
description: Lanza al supervisor una sesión de entrenamiento sobre un módulo klo_* o del sistema.
agent: supervisor-odoo
---

Tarea de entrenamiento: $ARGUMENTS

Planifica, delega en los subagentes (aprendiz-odoo, sistema-odoo, explorador-odoo, revisor-odoo,
probador-odoo), revisa y corrige. Al final, entrega un resumen de lo aprendido por el aprendiz
y las recomendaciones de mejora.
```

---

## 6. Configuración lista para aplicar en tu entorno actual

### 6.1 Respuesta directa: ¿la configuración del apartado 5 funciona tal cual?

**No.** La configuración del apartado 5 es un **ejemplo ilustrativo y parcial**, no un kit listo para copiar y pegar. Razones concretas:

1. En el apartado 5 solo se definen `supervisor-odoo`, `aprendiz-odoo` y `sistema-odoo` (inline en JSON). Faltan los agentes `aprendiz-odoo-hard`, `explorador-odoo`, `revisor-odoo` y `probador-odoo` que el supervisor referencia en su `permission.task`: si esos agentes no existen, el supervisor **no podrá invocarlos**.
2. El contrato real de delegación (`permission.task`) y los prompts completos viven en ficheros `.md` que **aún no existen** en tu entorno (tu `~/.config/opencode/agent/` está vacía y el proyecto no tiene `.opencode/`).
3. Tu configuración global actual es `~/.config/opencode/opencode.jsonc` (JSON con comentarios), mientras que el ejemplo del apartado 5 usa JSON puro.
4. Tu `opencode.json` del proyecto define `"permission": { "*": "allow" }`, que permite todas las acciones y **neutraliza** los permisos con `ask` de la propuesta.
5. Faltan las skills `odoo-dev` y `odoo-sysadmin` y el comando `/entrenar`.

**Con la configuración completa de este apartado sí funciona:** los modelos `opencode-go/*` citados están disponibles en tu suscripción Go (verificados con `opencode models`) y tu instalación de OpenCode soporta agentes, skills y comandos personalizados. La única condición técnica es **reiniciar OpenCode** después de aplicar los cambios (la configuración no se recarga en caliente).

### 6.2 Pasos de aplicación (checklist)

1. Crear los ficheros de agentes en `~/.config/opencode/agent/` (contenido en 6.4).
2. Sustituir `~/.config/opencode/opencode.jsonc` por el contenido de 6.3.
3. Sustituir `/opt/odoo14_paasa/odoo/opencode.json` por el contenido de 6.3 (elimina el `"*": allow`).
4. Crear las skills en `.opencode/skills/` del proyecto y el comando `/entrenar` (apartados 6.5 y 6.6).
5. **Reiniciar OpenCode.**
6. Verificar: con **Tab** se cambia al agente `supervisor-odoo`; en cualquier conversación se puede invocar `@aprendiz-odoo`, `@explorador-odoo`, etc.; el supervisor delega solo en los agentes permitidos.

### 6.3 Ficheros de configuración

#### `~/.config/opencode/opencode.jsonc` (nuevo contenido completo)

```jsonc
{
  // Configuracion global multi-agente Odoo (OpenCode Go)
  "$schema": "https://opencode.ai/config.json",
  "model": "opencode-go/deepseek-v4-pro",
  "small_model": "opencode-go/deepseek-v4-flash",
  "default_agent": "supervisor-odoo",
  "permission": {
    "edit": "ask",
    "bash": {
      "git *": "allow",
      "*": "ask"
    }
  }
}
```

#### `/opt/odoo14_paasa/odoo/opencode.json` (nuevo contenido completo)

```json
{
  "$schema": "https://opencode.ai/config.json",
  "permission": {
    "edit": "ask",
    "bash": {
      "git *": "allow",
      "*": "ask"
    }
  }
}
```

> Nota: se elimina `"*": allow` para no neutralizar los permisos. Los permisos **por agente** definidos en los ficheros `.md` toman prioridad sobre estos valores globales.

### 6.4 Ficheros de agentes (crear en `~/.config/opencode/agent/`)

#### `supervisor-odoo.md`

```markdown
---
description: Supervisor y entrenador para Odoo multiversión y su sistema.
mode: primary
model: opencode-go/deepseek-v4-pro
temperature: 0.1
color: primary
steps: 40
permission:
  task:
    "*": deny
    aprendiz-odoo: allow
    aprendiz-odoo-hard: allow
    sistema-odoo: allow
    explorador-odoo: allow
    explorador-odoo-free: allow
    revisor-odoo: allow
    probador-odoo: allow
    probador-odoo-free: allow
---

Actúas como Ingeniero de QA Senior, Formador y Supervisor de un equipo de agentes que trabajan
con múltiples versiones de Odoo (13 a 18+) y con el sistema que lo sostiene.

Tus responsabilidades:

1. ANALIZAR el requerimiento del usuario, detectar la versión de Odoo afectada y los puntos de
   incompatibilidad (API antigua vs. nueva API, campos JSON, vistas, QWeb, assets, JS/OWL).
2. PLANIFICAR y desglosar la tarea en pasos concretos y asignables.
3. DELEGAR el trabajo en el subagente apropiado mediante la herramienta Task:
   - `aprendiz-odoo`: programación, adaptación y mantenimiento de módulos klo_*.
   - `aprendiz-odoo-hard`: migraciones complejas o depuración profunda (modelo superior).
   - `sistema-odoo`: PostgreSQL, Ubuntu, librerías, odoo.conf, servicios y cron.
   - `explorador-odoo`: búsquedas y lectura de código en el monorepo.
   - `revisor-odoo`: revisión de calidad y seguridad del código entregado.
   - `probador-odoo`: pruebas de operativa (instalación/actualización, logs, tests).
   - `explorador-odoo-free` / `probador-odoo-free`: variantes gratuitas (sin datos confidenciales).
4. REVISAR cada entrega contra los criterios de calidad: cumplimiento de AGENTS.md,
   convenciones klo_* (icono, estructura), compatibilidad entre versiones, seguridad, rendimiento.
5. ENTRENAR al aprendiz: cuando detectes errores, devuélvele un reporte claro con la causa,
   el contexto y las correcciones obligatorias. Repite el ciclo hasta validar.
6. CERRAR la tarea con un resumen de lo realizado, decisiones tomadas y pruebas ejecutadas.

Reglas de oro:
- No edites tú el código: delega y revisa.
- Exige que cada módulo klo_* incluya static/description/icon.png.
- Nunca apruebes entregas sin verificar que la operativa (install/update) funciona.
- Para cambios de sistema (backups, apt, servicios), exige confirmación y documentación.
```

#### `aprendiz-odoo.md`

```markdown
---
description: Ejecutor y aprendiz que programa, adapta y mantiene módulos Odoo.
mode: subagent
model: opencode-go/glm-5.2
temperature: 0.2
steps: 15
permission:
  edit: allow
  bash:
    "*": ask
    "git *": allow
    "python odoo-bin*": allow
    "psql*": ask
  webfetch: allow
---

Actúas como Programador Junior de Odoo (aprendiz) bajo la supervisión de `supervisor-odoo`.

Tu misión es ejecutar los bloques de trabajo que te asigna el Supervisor:

1. Localizar el código afectado (módulos klo_* o núcleo) usando búsquedas dirigidas.
2. Escribir o adaptar el módulo respetando la estructura mínima:
   - __init__.py, __manifest__.py, models/__init__.py, models/*.py, views/*.xml
   - static/description/icon.png (obligatorio para klo_*)
   - Odoo 15+: ajusta el frontend JavaScript/OWL si el módulo lo requiere.
3. Mantener compatibilidad con la versión de Odoo indicada:
   - Odoo 14 o anterior: recordsets + nueva API, @api.model/@api.depends/@api.onchange.
   - Versiones nuevas: JSONB en place_type, cambios de vistas (tree/list), QWeb y assets.
4. Ejecutar pruebas de operativa cuando se te indique:
   - python odoo-bin -d <db> -i <modulo> --stop-after-init
   - python odoo-bin -d <db> -u <modulo> --stop-after-init
   - Revisar logs y trazas para corregir errores.
5. Aplicar SIEMPRE la retroalimentación del Supervisor: lee su reporte, corrige y vuelve a entregar.

Reglas de oro:
- No añadas comentarios innecesarios al código.
- No modifiques ficheros fuera del ámbito de la tarea asignada.
- Informa siempre de qué cambiaste, por qué y cómo lo comprobaste.
```

#### `aprendiz-odoo-hard.md`

```markdown
---
description: Ejecutor de alta capacidad para migraciones complejas y depuración profunda.
mode: subagent
model: opencode-go/glm-5.3
temperature: 0.2
steps: 20
permission:
  edit: allow
  bash:
    "*": ask
    "git *": allow
    "python odoo-bin*": allow
    "psql*": ask
---

Actúas como Programador Senior de Odoo. Igual que `aprendiz-odoo`, pero para tareas de
alta complejidad: migraciones entre versiones, reescrituras de API, problemas de rendimiento
o de concurrencia, y depuración profunda. Documenta las decisiones técnicas en detalle.
```

#### `sistema-odoo.md`

```markdown
---
description: "Administra el sistema de Odoo: PostgreSQL, Ubuntu, librerías y servicios."
mode: subagent
model: opencode-go/glm-5.2
temperature: 0.2
steps: 15
permission:
  edit: ask
  bash:
    "*": ask
    "git *": allow
    "python odoo-bin*": allow
    "grep *": allow
    "df -h": allow
  webfetch: allow
---

Actúas como Administrador de Sistemas Senior de la infraestructura Odoo, bajo la supervisión
de `supervisor-odoo`. Tu ámbito de trabajo es el SISTEMA, no el código de los módulos:

1. PostgreSQL: gestionar bases (crear, backup pg_dump/pg_restore, restaurar), revisar
   rendimiento (pg_stat_activity, EXPLAIN ANALYZE, índices), tamaños y espacio.
2. Ubuntu: instalar/actualizar paquetes apt, comprobar espacio en disco (df -h), permisos,
   usuarios y ficheros de log.
3. Servicios: estado y gestión del servicio Odoo (systemctl), configuración de odoo.conf
   (workers, db_user, addons_path, logfile) y cron del sistema.
4. Librerías Python: entornos virtuales (venv), requirements.txt, versiones de dependencias
   compatibles con la versión de Odoo.

Reglas de oro:
- No ejecutes acciones destructivas (borrar bases, apt remove, reinicios) sin confirmación
  explícita del Supervisor.
- Haz backup antes de modificar datos en producción.
- Documenta siempre qué cambiaste, el comando usado y cómo comprobarlo.
```

#### `explorador-odoo.md`

```markdown
---
description: Explora y responde preguntas sobre el código Odoo (solo lectura).
mode: subagent
model: opencode-go/deepseek-v4-flash
temperature: 0.1
steps: 10
permission:
  edit: deny
  bash:
    "*": deny
    "git status *": allow
    "git log*": allow
---

Eres un explorador read-only. Busca en el repositorio (incluido el núcleo de Odoo)
implementaciones, campos, vistas o APIs. Devuelve rutas de archivo:línea y fragmentos relevantes.
No modifiques nada.
```

#### `explorador-odoo-free.md`

```markdown
---
description: Explora y responde preguntas sobre el código Odoo (solo lectura) con modelo gratuito.
mode: subagent
model: opencode/deepseek-v4-flash-free
temperature: 0.1
steps: 10
permission:
  edit: deny
  bash:
    "*": deny
    "git status *": allow
    "git log*": allow
---

Eres un explorador read-only con modelo gratuito. Busca en el repositorio (incluido el núcleo
de Odoo) implementaciones, campos, vistas o APIs. Devuelve rutas de archivo:línea y fragmentos
relevantes. No modifiques nada. No lo uses en servidores con datos confidenciales de clientes.
```

#### `revisor-odoo.md`

```markdown
---
description: Revisa calidad, seguridad y compatibilidad del código Odoo sin editarlo.
mode: subagent
model: opencode-go/deepseek-v4-pro
temperature: 0.1
steps: 15
permission:
  edit: deny
  bash: deny
---

Eres un revisor senior. Analiza el código entregado y reporta:
seguridad (inyección SQL, seguridad de registros), rendimiento, adherencia a AGENTS.md,
convenciones klo_* y compatibilidad con la versión de Odoo objetivo. No edites código.
```

#### `probador-odoo.md`

```markdown
---
description: "Ejecuta la operativa de Odoo: instalación, actualización, tests y análisis de logs."
mode: subagent
model: opencode-go/gpt-5.6-luna
temperature: 0.2
steps: 10
permission:
  edit: deny
  bash:
    "*": ask
    "python odoo-bin*": allow
    "git *": allow
    "grep *": allow
    "psql*": ask
---

Eres el encargado de pruebas de operativa. Ejecuta instalación (-i), actualización (-u),
tests y comprobación de logs. Reporta errores con la traza exacta y el fichero:línea.
No modifiques el código: solo ejecuta y reporta.
```

#### `probador-odoo-free.md`

```markdown
---
description: "Ejecuta la operativa de Odoo: instalación, actualización, tests y análisis de logs con modelo gratuito."
mode: subagent
model: opencode/deepseek-v4-flash-free
temperature: 0.2
steps: 10
permission:
  edit: deny
  bash:
    "*": ask
    "python odoo-bin*": allow
    "git *": allow
    "grep *": allow
    "psql*": ask
---

Eres el encargado de pruebas de operativa con modelo gratuito. Ejecuta instalación (-i),
actualización (-u), tests y comprobación de logs. Reporta errores con la traza exacta y el
fichero:línea. No modifiques el código: solo ejecuta y reporta. No lo uses en servidores
con datos confidenciales de clientes.
```

### 6.5 Skills (crear en el proyecto `.opencode/skills/`)

#### `.opencode/skills/odoo-dev/SKILL.md`

```markdown
---
name: odoo-dev
description: "Usa cuando trabajes en módulos klo_* o en cualquier tarea de programación,
adaptación o mantenimiento de Odoo. Cubre estructura de módulos, compatibilidad entre
versiones y operativa de prueba."
---

# Desarrollo de módulos Odoo (convención klo_*)

## Estructura mínima de un módulo klo_*
klo_<nombre>/
|-- __init__.py
|-- __manifest__.py
|-- models/__init__.py, models/*.py
|-- views/*.xml
`-- static/description/icon.png   # OBLIGATORIO, copiar de cualquier klo_* existente

## Compatibilidad entre versiones
- Odoo 14 o anterior: recordsets + nueva API; usa @api.model, @api.depends, @api.onchange.
- Odoo 15+: JSONB en place_type, cambios en vistas (tree/list), QWeb, assets y JS/OWL.
- Antes de migrar un módulo: verificar __manifest__.py (version, depends, installable),
  cambios de API de modelos, renames de campos/vistas y módulos dependientes.

## Operativa de prueba
- La base de datos activa se define en el fichero `.conf` de `config/` con el parámetro `db_name`
  (cambia según el proyecto; no la hardcodees).
- Instalación:  python odoo-bin -d <db_name> -i klo_<modulo> --stop-after-init
- Actualización: python odoo-bin -d <db_name> -u klo_<modulo> --stop-after-init
- Revisa siempre el log; ante un error, localiza el fichero:línea de la traza antes de corregir.
```

#### `.opencode/skills/odoo-sysadmin/SKILL.md`

```markdown
---
name: odoo-sysadmin
description: "Usa en tareas de administración del sistema Odoo: PostgreSQL, Ubuntu,
librerías, servicios y configuración. Complementa a odoo-dev."
---

# Administración del sistema Odoo

## PostgreSQL
- Backup:  pg_dump -h localhost -U <db_user> <db_name> -F c -f <fichero>.dump
  (`<db_name>` se lee de `db_name` y `<db_user>` de `db_user` en el `.conf` de `config/`.)
- Estado:  psql -c "SELECT datname, pg_size_pretty(pg_database_size(datname)) FROM pg_database;"
- Rendimiento: EXPLAIN ANALYZE sobre consultas lentas; revisar índices y pg_stat_activity.

## Ubuntu / servicios
- Servicio Odoo: systemctl status odoo / restart
- Paquetes: apt update && apt list --upgradable
- odoo.conf: workers, db_user, addons_path, logfile.

## Reglas de seguridad
- Nunca borrar bases ni reiniciar servicios sin confirmación del Supervisor.
- Hacer backup antes de modificar producción.
```

### 6.6 Comando `/entrenar` (crear en el proyecto `.opencode/command/entrenar.md`)

```markdown
---
description: Lanza al supervisor una sesión de entrenamiento sobre un módulo klo_* o del sistema.
agent: supervisor-odoo
---

Tarea de entrenamiento: $ARGUMENTS

Planifica, delega en los subagentes (aprendiz-odoo, sistema-odoo, explorador-odoo, revisor-odoo,
probador-odoo), revisa y corrige. Al final, entrega un resumen de lo aprendido por el aprendiz
y las recomendaciones de mejora.
```

### 6.7 Verificación tras aplicar

- Ejecuta `opencode` y pulsa **Tab**: debe aparecer `supervisor-odoo` como agente principal.
- Escribe `@aprendiz-odoo` en el chat: el agente debe aparecer en el autocompletado (si no, reinicia OpenCode).
- Comando de prueba: `/entrenar revisa la estructura del módulo klo_* recién creado y entrena al aprendiz si falta el icono`.
- Si algo falla, revisa los logs con `logLevel: DEBUG` temporal en `opencode.jsonc`, corrige el fichero y reinicia.

---

## 7. Guía para preparar el entorno en otros proyectos de Odoo

Esta guía explica cómo preparar el entorno multi-agente en **cualquier otro proyecto de Odoo**, de cualquier versión, ejecutando `opencode` desde el directorio del propio proyecto. Los agentes de la propuesta son **independientes de la versión**: lo que cambia entre proyectos son los ficheros de configuración, las convenciones (`AGENTS.md`) y los parámetros de operativa (base de datos, ruta de `odoo-bin`, `addons_path`, prefijo de módulos).

### 7.1 Qué se configura una sola vez y qué se crea por proyecto

| Componente | Ubicación | ¿Una vez o por proyecto? |
|---|---|---|
| Configuración global (`opencode.jsonc`) | `~/.config/opencode/` | **Una vez** (vale para todos los proyectos) |
| Agentes (`supervisor-odoo`, `aprendiz-odoo`, `aprendiz-odoo-hard`, `sistema-odoo`, `explorador-odoo`, `explorador-odoo-free`, `revisor-odoo`, `probador-odoo`, `probador-odoo-free`) | `~/.config/opencode/agent/` | **Una vez** (se cargan en cualquier directorio) |
| Configuración del proyecto (`opencode.json`) | raíz del proyecto | **Por proyecto** |
| `AGENTS.md` (convenciones y versión de Odoo) | raíz del proyecto | **Por proyecto** |
| Skills (`odoo-dev`, `odoo-sysadmin`) | `.opencode/skills/` del proyecto (o globales) | Opcional: globales si son genéricas; por proyecto si se adaptan a la versión |
| Comando `/entrenar` | `.opencode/command/` del proyecto (o global) | Opcional |

> Los agentes se definen globales a propósito: OpenCode carga los agentes de `~/.config/opencode/agent/` en **cualquier** proyecto, de modo que el mismo Supervisor y el mismo aprendiz trabajan en Odoo 13, 14, 15, 16, 17, 18... sin duplicar ficheros. Esto incluye a las **variantes gratuitas** (`explorador-odoo-free`, `probador-odoo-free`, `configurador-odoo-free`): se crean una vez con el resto y quedan disponibles y delegables en todos los proyectos.

### 7.2 Pasos para preparar un proyecto nuevo

1. **Agentes y configuración global (una sola vez):** si aún no existen, crear los ficheros del apartado 6.3 (global `opencode.jsonc`) y 6.4 (los agentes del equipo). En los siguientes proyectos este paso ya no hace falta.
2. **Entrar en el proyecto:** abrir una terminal **en el directorio del proyecto** y ejecutar `opencode`.
3. **Generar `AGENTS.md`:** usar `/init` para que OpenCode analice el proyecto y cree el `AGENTS.md`, o crearlo manualmente con la plantilla del apartado 7.4.
4. **Crear `opencode.json` del proyecto:** copiar la plantilla del apartado 7.4. Si el proyecto usa módulos en repos o rutas externas (`addons_path`), ampliar `permission.external_directory`.
5. **Crear las skills:** copiar `.opencode/skills/odoo-dev` y `.opencode/skills/odoo-sysadmin` del apartado 6.5 al proyecto (o instalarlas globales) y ajustar las notas de versión con la tabla del apartado 7.3.
6. **Copiar el comando:** crear `.opencode/command/entrenar.md` (apartado 6.6) si se quiere disponer de `/entrenar`.
7. **Reiniciar OpenCode** y verificar (apartado 7.5).

### 7.3 Personalización por versión de Odoo

La operativa y los cambios de API por versión deben reflejarse en `AGENTS.md` y en la skill `odoo-dev` de cada proyecto:

| Versión de Odoo | Python | Puntos críticos a documentar en AGENTS.md |
|---|---|---|
| 13 | 3.6-3.8 | Transición a la nueva API; controllers; QWeb. |
| 14 | 3.6-3.8/3.10 | Nueva API de recordsets; `@api.model/@api.depends/@api.onchange`; vistas clásicas. |
| 15 | 3.8-3.10 | JSONB en `place_type`; vistas `tree/list`; comienzo de JavaScript OWL. |
| 16 | 3.10 | OWL completo en el backend; `field_autocomplete`; cambios en listas. |
| 17 | 3.10-3.12 | OWL/TypeScript; cambios en reportes y en definición de campos con XML IDs. |
| 18 | 3.10-3.12+ | OWL moderno; mejoras de rendimiento; revisar cambios de API antes de migrar. |

Y para cualquier versión, en `AGENTS.md` conviene fijar:

- Ruta y comando real de arranque (`./odoo-bin`, `python odoo-bin`, `odoo`...).
- Base de datos activa: SIEMPRE la que define `db_name` en el fichero `.conf` de `config/` (cambia según el proyecto; no hardcodear). Usuario PostgreSQL en `db_user`.
- `addons_path` y dónde viven los módulos propios y de terceros.
- Convención de nombres de los módulos propios (p. ej. `klo_*`, `custom_*`, `my_*`; cada proyecto usa la suya).
- Operativa de prueba habitual (`-i` / `-u` con `--stop-after-init`).

> **Regla de la base de datos:** en todos los proyectos la base de datos activa se define con el
> parámetro `db_name` del fichero `.conf` de `config/` (el nombre del fichero puede variar:
> `odoo.conf`, `odoo14_paasa.conf`, `odoo15.conf`, etc.). Cambia según el proyecto/entorno sobre el
> que se trabaje en cada momento; **nunca se hardcodea**.

### 7.4 Plantillas por proyecto

#### `opencode.json` (raíz de cada proyecto)

```json
{
  "$schema": "https://opencode.ai/config.json",
  "permission": {
    "edit": "ask",
    "bash": {
      "git *": "allow",
      "*": "ask"
    },
    "external_directory": {
      "*": "allow"
    }
  }
}
```

> El resto (modelos, `default_agent`, agentes) viene de la configuración global; este fichero solo ajusta permisos del proyecto. `external_directory` permite a los agentes leer/escribir fuera del directorio de trabajo (útil en monorepos o `addons_path` externos).

#### `AGENTS.md` (raíz de cada proyecto, plantilla)

```markdown
# Convenciones del proyecto <NOMBRE_PROYECTO>

## Entorno
- Versión de Odoo: <VERSION> (ej. 14, 17)
- Python: <PYTHON_VERSION>
- Base de datos de desarrollo: <DB_NAME> (el valor de `db_name` del fichero `.conf` de `config/`; no hardcodear)
- Usuario PostgreSQL: <DB_USER> (el valor de `db_user` del fichero `.conf`)
- Comando de arranque: <CMD> (ej. python odoo-bin)
- addons_path: <RUTA> (dónde viven los módulos propios y de terceros)

## Convenciones de módulos propios
- Prefijo: <PREFIJO> (ej. klo_, custom_, my_)
- Estructura mínima: __init__.py, __manifest__.py, models/, views/,
  static/description/icon.png (obligatorio)
- Idioma de las vistas y traducciones: <IDIOMA> (ej. es_ES)

## Operativa de prueba
- `<DB_NAME>` se lee del parámetro `db_name` del fichero `.conf` de `config/` (no hardcodear).
- Instalación:  <CMD> -d <DB_NAME> -i <modulo> --stop-after-init
- Actualización: <CMD> -d <DB_NAME> -u <modulo> --stop-after-init
- Ante un error, localizar el fichero:línea de la traza antes de corregir.
```

### 7.5 Comprobación en el proyecto nuevo

1. Ejecutar `opencode` desde el directorio del proyecto.
2. Pulsar **Tab**: debe aparecer `supervisor-odoo` como agente principal.
3. Escribir `@aprendiz-odoo`: debe aparecer en el autocompletado.
4. Pedir al supervisor una tarea pequeña del proyecto (p. ej. `/entrenar localiza el __manifest__.py de los módulos propios y revisa que cumplen la estructura mínima`).
5. Si no aparece `supervisor-odoo`, reiniciar OpenCode; si persiste, revisar `AGENTS.md` y los ficheros globales con `logLevel: DEBUG` temporal.

---

## 8. Agente recomendado para configurar el entorno en otros servidores Ubuntu

Esta recomendación responde a la pregunta de qué modelo/agente de OpenCode usar para realizar la **configuración y replicación del entorno multi-agente** en otros servidores Ubuntu siguiendo el fichero `Contexto_de_configuracion_IA_multi-agente.md`.

### 8.1 Características de la tarea y criterios de selección

La configuración en otros servidores consiste en: instalar OpenCode, conectar el proveedor, crear ficheros con **contenido exacto** (configuración, agentes, skills, comandos), ejecutar comandos de verificación y corregir errores YAML/JSON. Por tanto, el modelo ideal debe cumplir:

1. **Fiabilidad en el uso de herramientas** (bash, edición de ficheros): la tarea es repetitiva y determinista, no creativa.
2. **Fidelidad al copiar contenido exacto** de las plantillas (un carácter mal puesto rompe YAML/JSON).
3. **Bajo coste y alto volumen**: el trabajo se repite en cada servidor.
4. **Razonamiento suficiente** para resolver problemas típicos (JSONC con comentarios, descripciones YAML con `:`, permisos, reinicios).
5. **Confidencialidad**: en servidores de clientes conviene modelos con retentiva de datos 0 días.

### 8.2 Recomendación en OpenCode Zen (modelos gratuitos)

| Modelo (Zen free) | Valoración | Uso recomendado |
|---|---|---|
| **`opencode/deepseek-v4-flash-free`** | **Recomendado (gratis)** | Misma familia que el modelo de trabajo diario: rápido, fiable en ejecución de comandos y copia de ficheros. Ideal para el volumen de instalaciones en varios servidores. |
| `opencode/big-pickle` | Alternativa gratuita general | Buen perfil general si se quiere variar de familia; mismo procedimiento. |
| `opencode/nemotron-3-ultra-free` | Alternativa gratuita con más razonamiento | Útil si se prevén muchos problemas de entorno (versiones raras de Ubuntu, Python, permisos). |
| `opencode/mimo-v2.5-free` | Alternativa ligera | Para servidores donde solo hay que copiar ficheros sin incidencias. |

> **Advertencia:** los modelos free de Zen pueden usar los datos para entrenamiento. **No** usarlos para configurar servidores con datos confidenciales de clientes (ver apartado 9.3); en ese caso usar OpenCode Go (retentiva 0 días).

### 8.3 Recomendación en OpenCode Go (pago)

| Modelo (Go) | Valoración | Uso recomendado |
|---|---|---|
| **`opencode-go/deepseek-v4-flash`** | **Recomendado (pago)** | Rápido, barato y con cuota muy alta (18.900 req/mes); retentiva 0 días. La mejor opción para desplegar en muchos servidores sin agotar la suscripción. |
| `opencode-go/gpt-5.6-luna` | Alternativa equilibrada | Más robusto en casos raros; cuota alta (10.250 req/mes). |
| `opencode-go/glm-5.2` | Alternativa de calidad superior | Si el servidor de destino tiene un entorno inusual (Python compilado, rutas no estándar, monorepos complejos) que exija más razonamiento. |

### 8.4 Agente propuesto: `configurador-odoo` (subagente)

Además del modelo, se propone crear un agente dedicado a la instalación/replicación. Fichero a crear en `~/.config/opencode/agent/configurador-odoo.md`:

```markdown
---
description: Replica y configura el entorno multi-agente de OpenCode en servidores Ubuntu.
mode: subagent
model: opencode-go/deepseek-v4-flash
temperature: 0.1
steps: 30
permission:
  edit: allow
  bash:
    "*": ask
    "git *": allow
    "python3 *": allow
    "curl *": ask
  webfetch: allow
---

Actúas como instalador del entorno multi-agente de OpenCode para Odoo.

Tu tarea es replicar en este servidor el entorno descrito en el fichero
Contexto_de_configuracion_IA_multi-agente.md (en el mismo directorio).

Procedimiento:
1. Si OpenCode no está instalado, instalarlo (curl -fsSL https://opencode.ai/install | bash)
   y avisar de que el usuario debe conectar el proveedor con /connect antes de continuar.
2. Leer el fichero Contexto_de_configuracion_IA_multi-agente.md completo.
3. Aplicar el paso 1 (global): ~/.config/opencode/opencode.jsonc y los 11 agentes
   (7 de trabajo + explorador-odoo-free + probador-odoo-free + configurador-odoo + configurador-odoo-free).
4. Aplicar el paso 2 (por proyecto): opencode.json, AGENTS.md, skills y comando /entrenar.
5. Reproducir el contenido de cada fichero EXACTAMENTE como indica el contexto.
6. Ejecutar la validación del apartado 7 del contexto y corregir errores YAML/JSON.
7. Entregar el checklist final cumplimentado y las acciones manuales pendientes
   (conectar proveedor, reiniciar OpenCode).

Reglas de oro:
- No modifiques el contenido de las plantillas salvo los placeholders <...> del AGENTS.md.
- No uses modelos gratuitos para datos confidenciales de clientes.
- Reporta cualquier desviación y cómo la resolviste.
```

**Variante gratuita** `configurador-odoo-free.md` (para servidores sin datos confidenciales):

```markdown
---
description: "Replica el entorno multi-agente de OpenCode usando un modelo gratuito de Zen."
mode: subagent
model: opencode/deepseek-v4-flash-free
temperature: 0.1
steps: 30
permission:
  edit: allow
  bash:
    "*": ask
    "git *": allow
    "python3 *": allow
    "curl *": ask
  webfetch: allow
---

Igual que `configurador-odoo`, pero con modelo gratuito. No lo uses en servidores
con datos confidenciales de clientes.
```

> Si se quiere que el Supervisor (`supervisor-odoo`) pueda delegar la instalación, añadir en su `permission.task`: `configurador-odoo: allow` (y `configurador-odoo-free: allow` si procede).

### 8.5 Flujo de uso en cada servidor de destino

1. Copiar al servidor los ficheros `Contexto_de_configuracion_IA_multi-agente.md` y `Propuesta_entorno_multi_agente_opencode.md`.
2. Instalar OpenCode y conectar el proveedor (`/connect` → OpenCode Go/Zen) si no está hecho.
3. Crear el agente `configurador-odoo` (apartado 8.4) o usar el Supervisor directamente.
4. Lanzar: `opencode` y pedir: *"Aplica la configuración descrita en Contexto_de_configuracion_IA_multi-agente.md"* (o `@configurador-odoo ...`).
5. El agente crea los ficheros, valida y entrega el checklist final; el usuario reinicia OpenCode y verifica con Tab / `@agente`.

---

## 9. Apartado: agentes y modelos gratuitos en OpenCode Zen

OpenCode Zen incluye varios modelos **gratuitos** (tiempo limitado, sujetos a disponibilidad). Sus identificadores usan el prefijo `opencode/<modelo>`.

### 9.1 Modelos gratuitos disponibles

| Modelo (ID Zen free) | Perfil | Rol recomendado |
|---|---|---|
| `opencode/deepseek-v4-flash-free` | Flash gratuito de DeepSeek | Ejecutor/Explorador de bajo coste; gran volumen de iteraciones. |
| `opencode/mimo-v2.5-free` | Modelo ligero y rápido | Búsquedas masivas, resúmenes, tareas repetitivas del aprendiz. |
| `opencode/hy3-free` | Modelo general económico | Asistente general, documentación y soporte de configuración. |
| `opencode/laguna-s-2.1-free` | Modelo general | Tareas auxiliares de bajo coste. |
| `opencode/nemotron-3-ultra-free` | Modelo grande de NVIDIA | Razonamiento de soporte sin coste (con límites de prueba). |
| `opencode/nemotron-3.5-lightning-free` | Modelo rápido de NVIDIA | Ejecución de tareas simples en paralelo. |
| `opencode/big-pickle` | Modelo "stealth" gratuito | Alternativa general sin coste para tareas del supervisor en pruebas. |

### 9.2 Cómo integrar los modelos free en el entorno

Los modelos gratuitos permiten **descargar el consumo de Go/Zen de pago** y garantizar continuidad cuando se alcanzan los límites de suscripción. Dos estrategias:

1. **Agente de apoyo con modelo free** (recomendado):

```markdown
---
description: Explorador read-only del código Odoo usando modelo gratuito.
mode: subagent
model: opencode/deepseek-v4-flash-free
temperature: 0.1
permission:
  edit: deny
  bash: deny
---

Explora el repositorio Odoo y responde con rutas fichero:línea y fragmentos. No modificas nada.
```

En esta configuración ya existen **dos agentes gratuitos definidos**: `explorador-odoo-free` y `probador-odoo-free` (mismo modelo `opencode/deepseek-v4-flash-free`), listos para que el Supervisor delegue en ellos en tareas de exploración y de pruebas sin gastar cuota de Go y sin datos confidenciales.

2. **Cambio dinámico de modelo en el Supervisor** (`opencode models` / `/models`) cuando el problema sea sencillo o se quiera ahorrar presupuesto:

```
/model opencode/big-pickle          # Supervisor en modo económico
/model opencode-go/deepseek-v4-pro  # Supervisor en modo completo
```

### 9.3 Advertencias importantes

- Los modelos free son **temporales** y pueden desaparecer o dejar de ser gratuitos; revisa `https://opencode.ai/zen/v1/models` para el listado vigente.
- Algunos modelos free pueden **usar datos para entrenamiento** (según política de Zen/NVIDIA). **No** los uses con datos confidenciales de clientes de Odoo (bases de datos reales, credenciales). Reserva los modelos de pago (retentiva 0 días) para datos sensibles.
- Cuando se alcanza el límite de uso de Go y se tiene saldo en Zen, conviene activar la opción **"Use balance"** en la consola para continuar sin bloquearse.

---

## 10. Flujos de trabajo de ejemplo

### 10.1 Desarrollo: migrar un módulo entre versiones

**Escenario:** *"Adaptar el módulo klo_facturas_electronicas de Odoo 14 a Odoo 17 y verificar la operativa"*.

1. **Supervisor analiza:** detecta que hay que revisar `__manifest__.py`, campos, vistas y controllers afectados por la migración 14 a 17. Crea el plan.
2. **Delega en `explorador-odoo`:** localiza los ficheros del módulo y los puntos con API de Odoo 14 (p. ej. `@api.one`, vistas `tree`, JS/OWL, módulos dependientes obsoletos).
3. **Delega en `aprendiz-odoo`:** adapta el código a la API de Odoo 17, actualiza dependencias y mantiene la estructura `klo_*` (incluido el icono).
4. **Delega en `revisor-odoo`:** revisa compatibilidad, seguridad y adherencia a `AGENTS.md`.
5. **Delega en `probador-odoo`:** ejecuta `python odoo-bin -d <db> -i klo_facturas_electronicas --stop-after-init` y `-u`; reporta errores de log.
6. **Supervisor entrena:** si hay errores, devuelve al aprendiz un reporte con causa y correcciones; el aprendiz corrige y se repite el ciclo.
7. **Cierre:** el supervisor valida los criterios, resume el trabajo y emite recomendaciones de mejora continua.

### 10.2 Mantenimiento del sistema

**Escenario:** *"PostgreSQL está lento y el disco casi lleno; además hay que actualizar las librerías del entorno"*.

1. **Supervisor analiza** el síntoma y delega en `sistema-odoo`.
2. **`sistema-odoo`** revisa `df -h`, `pg_stat_activity` y consultas lentas con `EXPLAIN ANALYZE`; propone índices o limpieza; comprueba actualizaciones de paquetes (`apt list --upgradable`) y el venv de Odoo.
3. **Supervisor revisa** el plan de acción; `sistema-odoo` ejecuta los cambios **con confirmación** (backup previo `pg_dump`, reinicio del servicio solo si es necesario).
4. **Cierre:** se documenta qué se cambió, el comando usado y cómo verificarlo.

### 10.3 Entrenamiento del aprendiz

**Escenario:** el Supervisor detecta que el aprendiz comete siempre el mismo error (p. ej. usa APIs de Odoo 14 en módulos para Odoo 17).

1. El Supervisor genera un **reporte de entrenamiento** con ejemplos correctos e incorrectos.
2. Se guarda la lección en `AGENTS.md` o en una skill (`odoo-dev`), para que se aplique en futuras sesiones.
3. El aprendiz re-entrega aplicando la lección; el Supervisor valida y cierra.

---

## 11. Buenas prácticas y optimización de costes

- **Contrato de delegación estricto:** limita `permission.task` del Supervisor a los subagentes permitidos y deniega el resto (`"*": deny`).
- **Límites de pasos (`steps`)**: define `steps` en los subagentes (p. ej. 10-15) para evitar bucles infinitos de corrección.
- **`/undo` con Git:** ante una entrega que rompa el entorno, revierte el último estado válido.
- **Uso de `small_model`:** OpenCode usa el modelo pequeño para títulos/resúmenes; asigna `opencode-go/deepseek-v4-flash` para ahorrar.
- **Horario valle de DeepSeek:** programa tareas pesadas (instalaciones de módulos, migraciones masivas, backups) fuera de horas punta.
- **Modelos free para volumen:** exploración y tareas repetitivas con `opencode/deepseek-v4-flash-free` o `opencode/mimo-v2.5-free`.
- **Confidencialidad:** datos reales de clientes solo con modelos de pago con retentiva 0 días.
- **Sistema:** siempre backup (`pg_dump`) antes de tocar producción; comandos destructivos con `ask`.

---

## 12. Limitaciones a tener en cuenta

- Los **subagentes** se invocan vía la herramienta Task desde el Supervisor; el *entrenamiento* real (ajuste de pesos) no existe en OpenCode: el "entrenamiento" es **contextual e instruccional** (prompts, skills, retroalimentación en sesión y `AGENTS.md`).
- La calidad del ciclo depende del modelo del Supervisor; si es débil, aprobará entregas erróneas. Por eso se recomienda `deepseek-v4-pro` (o `kimi-k3`/`grok-4.5` puntualmente) como crítico.
- OpenCode **no recarga la configuración en caliente**: tras editar `opencode.json` o ficheros de agentes, hay que reiniciar la sesión.
- Los modelos free de Zen son temporales y con políticas de datos variables (ver apartado 9.3).
- Los modelos con mayor cuota (GLM-5.2, Kimi K2.7 Code, Flash) son los que soportan el volumen diario; los premium (K3, Grok, Qwen Max) se reservan para tareas puntuales.

---

## 13. Conclusión

La arquitectura recomendada es un Supervisor/Entrenador (`opencode-go/deepseek-v4-pro`, temperatura 0.1, modo primario) que delega en un Ejecutor/Aprendiz de programación (`opencode-go/glm-5.2`, con `glm-5.3` para migraciones críticas), un Ejecutor de sistema e infraestructura (`sistema-odoo`, también con `glm-5.2`) y agentes de apoyo (`explorador-odoo`, `revisor-odoo`, `probador-odoo`), con los modelos gratuitos de Zen como capa de volumen y respaldo. Esta configuración permite trabajar con múltiples versiones de Odoo de forma controlada, mantener PostgreSQL/Ubuntu/librerías bajo supervisión, entrenar al aprendiz mediante retroalimentación en sesión y conservar el estándar `klo_*` y la operativa verificada en cada entrega.
