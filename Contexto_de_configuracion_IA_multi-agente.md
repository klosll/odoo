# Contexto de Configuración del Entorno Multi-Agente de OpenCode para Odoo

**Propósito:** este fichero contiene el contexto y las instrucciones completas para que una IA realice la **instalación y configuración completa del entorno multi-agente** en **servidores Linux Ubuntu**, replicando el entorno descrito en el documento `Propuesta_entorno_multi_agente_opencode.md`.

**Uso previsto:** copiar ambos ficheros (este y `Propuesta_entorno_multi_agente_opencode.md`) al servidor Ubuntu de destino y entregarlos a una IA con la petición: *"Instala y configura el entorno multi-agente descrito en estos ficheros"*. Este fichero es autosuficiente y accionable: contiene el contenido exacto de todos los ficheros de configuración y un **runbook** (apartado 10) que la IA debe seguir de principio a fin.

---

## 1. Resumen de la arquitectura que se va a crear

Patrón **Actor-Crítico / Supervisor-Ejecutor** sobre OpenCode:

| Agente | Rol | Modelo (OpenCode Go) | Temperatura | Modo |
|---|---|---|---|---|
| `supervisor-odoo` | Supervisor y entrenador: planifica, delega, revisa y entrena | `opencode-go/deepseek-v4-pro` | 0.1 | `primary` |
| `aprendiz-odoo` | Ejecutor/aprendiz: programa, adapta y mantiene módulos Odoo | `opencode-go/glm-5.2` | 0.2 | `subagent` |
| `aprendiz-odoo-hard` | Ejecutor de alta capacidad para migraciones complejas | `opencode-go/glm-5.3` | 0.2 | `subagent` |
| `sistema-odoo` | Administra PostgreSQL, Ubuntu, librerías y servicios | `opencode-go/glm-5.2` | 0.2 | `subagent` |
| `explorador-odoo` | Búsqueda y lectura de código (solo lectura) | `opencode-go/deepseek-v4-flash` | 0.1 | `subagent` |
| `revisor-odoo` | Revisión de calidad, seguridad y compatibilidad | `opencode-go/deepseek-v4-pro` | 0.1 | `subagent` |
| `probador-odoo` | Operativa: instalación/actualización, tests, logs | `opencode-go/gpt-5.6-luna` | 0.2 | `subagent` |
| `configurador-odoo` | Instala/replica el entorno multi-agente en servidores Ubuntu | `opencode-go/deepseek-v4-flash` | 0.1 | `subagent` |
| `configurador-odoo-free` | Ídem con modelo gratuito (sin datos confidenciales) | `opencode/deepseek-v4-flash-free` | 0.1 | `subagent` |

Reglas clave:

- Los **agentes se definen globales** en `~/.config/opencode/agent/`: funcionan en cualquier proyecto Odoo, de cualquier versión (13 a 18+).
- El supervisor solo puede delegar en los subagentes permitidos mediante `permission.task` (contrato de delegación con `"*": deny`).
- El supervisor es el agente por defecto (`default_agent`), no edita código directamente: delega y revisa.
- Los modelos de pago tienen retentiva 0 días; los modelos gratuitos de Zen pueden usar datos para entrenamiento y **no** deben usarse con datos confidenciales.

---

## 2. Requisitos previos en el servidor de destino (Ubuntu)

1. **Sistema:** Ubuntu (20.04/22.04/24.04). La instalación se realiza con el **usuario de trabajo habitual (no root)**: la configuración se crea en `~/.config/opencode/`.
2. **Dependencias base:** instalar antes de nada si no existen:

   ```bash
   sudo apt update && sudo apt install -y curl git python3 python3-yaml
   ```

   (`curl` es necesario para el instalador de OpenCode; `git` para el control de versiones; `python3` y `python3-yaml` para el script de validación del apartado 8.)
3. **OpenCode instalado:** si no lo está, instalarlo con `curl -fsSL https://opencode.ai/install | bash` y comprobar con `opencode --version`.
4. **Suscripción OpenCode Go conectada** (los modelos `opencode-go/*` deben aparecer en `opencode models`). Es el **único paso interactivo**: `/connect` dentro de OpenCode, elegir `OpenCode Go` y pegar la API key (se obtiene en `https://opencode.ai/auth`). La IA debe solicitar al usuario que realice este paso.
5. **Opcional:** saldo en OpenCode Zen y/o modelos gratuitos (`opencode/*-free`, `opencode/big-pickle`) como respaldo y para descargar consumo.
6. **Directorios a crear** por la IA (paso 1): `~/.config/opencode/` y `~/.config/opencode/agent/`.
7. Acceso de escritura al directorio del proyecto Odoo de destino.

---

## 3. Paso 1: configuración global (`~/.config/opencode/`)

Primero crear los directorios (no fallan si ya existen):

```bash
mkdir -p ~/.config/opencode/agent
```

A continuación crear los siguientes ficheros con el contenido exacto indicado.

### 3.1 `~/.config/opencode/opencode.jsonc`

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

### 3.2 `~/.config/opencode/agent/supervisor-odoo.md`

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
    revisor-odoo: allow
    probador-odoo: allow
    configurador-odoo: allow
    configurador-odoo-free: allow
---

Actúas como Ingeniero de QA Senior, Formador y Supervisor de un equipo de agentes que trabajan
con múltiples versiones de Odoo (13 a 18+) y con el sistema que lo sostiene.

Tus responsabilidades:

1. ANALIZAR el requerimiento del usuario, detectar la versión de Odoo afectada y los puntos de
   incompatibilidad (API antigua vs. nueva API, campos JSON, vistas, QWeb, assets, JS/OWL).
2. PLANIFICAR y desglosar la tarea en pasos concretos y asignables.
3. DELEGAR el trabajo en el subagente apropiado mediante la herramienta Task:
   - `aprendiz-odoo`: programación, adaptación y mantenimiento de módulos propios.
   - `aprendiz-odoo-hard`: migraciones complejas o depuración profunda (modelo superior).
   - `sistema-odoo`: PostgreSQL, Ubuntu, librerías, odoo.conf, servicios y cron.
   - `explorador-odoo`: búsquedas y lectura de código en el monorepo.
   - `revisor-odoo`: revisión de calidad y seguridad del código entregado.
   - `probador-odoo`: pruebas de operativa (instalación/actualización, logs, tests).
4. REVISAR cada entrega contra los criterios de calidad: cumplimiento de AGENTS.md,
   convenciones del proyecto, compatibilidad entre versiones, seguridad, rendimiento.
5. ENTRENAR al aprendiz: cuando detectes errores, devuélvele un reporte claro con la causa,
   el contexto y las correcciones obligatorias. Repite el ciclo hasta validar.
6. CERRAR la tarea con un resumen de lo realizado, decisiones tomadas y pruebas ejecutadas.

Reglas de oro:
- No edites tú el código: delega y revisa.
- Exige que los módulos propios cumplan la estructura y convenciones definidas en AGENTS.md.
- Nunca apruebes entregas sin verificar que la operativa (install/update) funciona.
- Para cambios de sistema (backups, apt, servicios), exige confirmación y documentación.
```

### 3.3 `~/.config/opencode/agent/aprendiz-odoo.md`

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

1. Localizar el código afectado (módulos propios o núcleo) usando búsquedas dirigidas.
2. Escribir o adaptar el módulo respetando la estructura mínima definida en AGENTS.md:
   - __init__.py, __manifest__.py, models/__init__.py, models/*.py, views/*.xml
   - static/description/icon.png cuando sea obligatorio según las convenciones del proyecto
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

### 3.4 `~/.config/opencode/agent/aprendiz-odoo-hard.md`

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

### 3.5 `~/.config/opencode/agent/sistema-odoo.md`

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

### 3.6 `~/.config/opencode/agent/explorador-odoo.md`

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

### 3.7 `~/.config/opencode/agent/revisor-odoo.md`

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
convenciones del proyecto y compatibilidad con la versión de Odoo objetivo. No edites código.
```

### 3.8 `~/.config/opencode/agent/probador-odoo.md`

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

### 3.9 `~/.config/opencode/agent/configurador-odoo.md`

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
1. Comprobar dependencias base: sudo apt update && sudo apt install -y curl git python3 python3-yaml.
2. Si OpenCode no está instalado, instalarlo (curl -fsSL https://opencode.ai/install | bash)
   y avisar de que el usuario debe conectar el proveedor con /connect antes de continuar.
3. Leer el fichero Contexto_de_configuracion_IA_multi-agente.md completo.
4. Crear directorios: mkdir -p ~/.config/opencode/agent y los .opencode/skills/... del proyecto.
5. Aplicar el paso 1 (global): ~/.config/opencode/opencode.jsonc y los 9 agentes
   (7 de trabajo + configurador-odoo + configurador-odoo-free).
6. Aplicar el paso 2 (por proyecto): opencode.json, AGENTS.md, skills y comando /entrenar.
7. Reproducir el contenido de cada fichero EXACTAMENTE como indica el contexto.
8. Ejecutar la validación del apartado 7 del contexto y corregir errores YAML/JSON.
9. Entregar el checklist final cumplimentado y las acciones manuales pendientes
   (conectar proveedor, reiniciar OpenCode).

Reglas de oro:
- No modifiques el contenido de las plantillas salvo los placeholders <...> del AGENTS.md.
- No uses modelos gratuitos para datos confidenciales de clientes.
- Reporta cualquier desviación y cómo la resolviste.
```

### 3.10 `~/.config/opencode/agent/configurador-odoo-free.md`

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

---

## 4. Paso 2: configuración por proyecto Odoo

Ejecutar desde el directorio del proyecto Odoo de destino (donde se va a lanzar `opencode`). Los ficheros del apartado 3 no se duplican: son globales.

### 4.1 `opencode.json` (raíz del proyecto)

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

> Nota: no debe existir `"permission": { "*": "allow" }`, porque neutralizaría los permisos con `ask` de los agentes. `external_directory` permite leer/escribir fuera del directorio del proyecto (monorepos o `addons_path` externos).

### 4.2 `AGENTS.md` (raíz del proyecto)

Plantilla con placeholders `<...>` que hay que rellenar con los datos del proyecto concreto:

```markdown
# Convenciones del proyecto <NOMBRE_PROYECTO>

## Entorno
- Versión de Odoo: <VERSION> (ej. 14, 17)
- Python: <PYTHON_VERSION>
- Base de datos de desarrollo: <DB_NAME>
- Usuario PostgreSQL: <DB_USER>
- Comando de arranque: <CMD> (ej. python odoo-bin)
- addons_path: <RUTA> (dónde viven los módulos propios y de terceros)

## Convenciones de módulos propios
- Prefijo: <PREFIJO> (ej. klo_, custom_, my_)
- Estructura mínima: __init__.py, __manifest__.py, models/, views/,
  static/description/icon.png (obligatorio según convención del proyecto)
- Idioma de las vistas y traducciones: <IDIOMA> (ej. es_ES)

## Operativa de prueba
- Instalación:  <CMD> -d <DB_NAME> -i <modulo> --stop-after-init
- Actualización: <CMD> -d <DB_NAME> -u <modulo> --stop-after-init
- Ante un error, localizar el fichero:línea de la traza antes de corregir.
```

### 4.3 `.opencode/skills/odoo-dev/SKILL.md`

```markdown
---
name: odoo-dev
description: "Usa cuando trabajes en módulos propios o en cualquier tarea de programación,
adaptación o mantenimiento de Odoo. Cubre estructura de módulos, compatibilidad entre
versiones y operativa de prueba."
---

# Desarrollo de módulos Odoo

## Estructura mínima de un módulo
<nombre_modulo>/
|-- __init__.py
|-- __manifest__.py
|-- models/__init__.py, models/*.py
|-- views/*.xml
`-- static/description/icon.png   # OBLIGATORIO si la convención del proyecto lo exige

## Compatibilidad entre versiones
- Odoo 14 o anterior: recordsets + nueva API; usa @api.model, @api.depends, @api.onchange.
- Odoo 15+: JSONB en place_type, cambios en vistas (tree/list), QWeb, assets y JS/OWL.
- Antes de migrar un módulo: verificar __manifest__.py (version, depends, installable),
  cambios de API de modelos, renames de campos/vistas y módulos dependientes.

## Operativa de prueba
- Instalación:  <CMD> -d <DB_NAME> -i <modulo> --stop-after-init
- Actualización: <CMD> -d <DB_NAME> -u <modulo> --stop-after-init
- Revisa siempre el log; ante un error, localiza el fichero:línea de la traza antes de corregir.
```

### 4.4 `.opencode/skills/odoo-sysadmin/SKILL.md`

```markdown
---
name: odoo-sysadmin
description: "Usa en tareas de administración del sistema Odoo: PostgreSQL, Ubuntu,
librerías, servicios y configuración. Complementa a odoo-dev."
---

# Administración del sistema Odoo

## PostgreSQL
- Backup:  pg_dump -h localhost -U <user> <db> -F c -f <fichero>.dump
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

### 4.5 `.opencode/command/entrenar.md`

```markdown
---
description: Lanza al supervisor una sesión de entrenamiento sobre un módulo o del sistema.
agent: supervisor-odoo
---

Tarea de entrenamiento: $ARGUMENTS

Planifica, delega en los subagentes (aprendiz-odoo, sistema-odoo, explorador-odoo, revisor-odoo,
probador-odoo), revisa y corrige. Al final, entrega un resumen de lo aprendido por el aprendiz
y las recomendaciones de mejora.
```

---

## 5. Paso 3: conexión del proveedor y arranque

> Este es el **único paso que la IA no puede automatizar del todo**: requiere interacción del usuario (pegar la API key).

1. Ejecutar `opencode` en el directorio del proyecto.
2. Si aún no hay proveedor conectado: `/connect` → `OpenCode Go` → pegar API key → `/models` para confirmar que aparecen los modelos `opencode-go/*`.
3. **Reiniciar OpenCode** tras cualquier cambio de configuración (no se recarga en caliente).

---

## 6. Verificación del entorno

1. Con **Tab**: debe aparecer `supervisor-odoo` como agente principal.
2. Escribir `@aprendiz-odoo` (y el resto de agentes): deben aparecer en el autocompletado.
3. Prueba funcional: `/entrenar revisa la estructura de los módulos propios y entrena al aprendiz si algún módulo no cumple las convenciones del proyecto`.
4. Prueba de delegación: pedir al supervisor *"Busca con @explorador-odoo dónde se define el campo X"* y confirmar que delega.

---

## 7. Validación técnica (ejecutar antes de dar por completada la instalación)

Ejecutar este script para validar los ficheros YAML/JSON del entorno. Se ejecuta así:

```bash
python3 validar_config.py /ruta/al/proyecto/odoo
```

(si no se indica ruta, valida solo la configuración global del usuario).

```python
#!/usr/bin/env python3
import os, sys, json

try:
    import yaml
except ImportError:
    sys.exit("Falta pyyaml -> instalar: sudo apt install -y python3-yaml")

proj = sys.argv[1] if len(sys.argv) > 1 else None
base = os.path.expanduser("~/.config/opencode")

# 1) Frontmatter YAML de los agentes globales
for f in sorted(os.listdir(os.path.join(base, "agent"))):
    if f.endswith(".md"):
        txt = open(os.path.join(base, "agent", f)).read()
        yaml.safe_load(txt.split("---")[1])
        print("OK yaml:", f)

# 2) opencode.jsonc global (ignorando comentarios //)
jc = os.path.join(base, "opencode.jsonc")
b = open(jc).read()
lines = [l for l in b.splitlines() if not l.strip().startswith("//")]
json.loads("\n".join(lines))
print("OK jsonc: opencode.jsonc")

# 3) Configuracion y skills del proyecto
if proj:
    pj = os.path.join(proj, "opencode.json")
    if os.path.exists(pj):
        json.load(open(pj)); print("OK json:", pj)
    for root, _, files in os.walk(os.path.join(proj, ".opencode")):
        for f in files:
            if f.endswith(".md"):
                txt = open(os.path.join(root, f)).read()
                if txt.startswith("---"):
                    yaml.safe_load(txt.split("---")[1])
                    print("OK yaml:", os.path.relpath(os.path.join(root, f), proj))

print("Validacion completada")
```

Comprobar además:

- Los 9 ficheros de agente existen en `~/.config/opencode/agent/` (`supervisor-odoo`, `aprendiz-odoo`, `aprendiz-odoo-hard`, `sistema-odoo`, `explorador-odoo`, `revisor-odoo`, `probador-odoo`, `configurador-odoo`, `configurador-odoo-free`).
- `default_agent: supervisor-odoo` apunta a un agente `mode: primary` y no `hidden`.
- En `supervisor-odoo.md` el `permission.task` contiene exactamente: `"*": deny` y los 8 subagentes `allow`.
- En el proyecto existen `opencode.json`, `AGENTS.md`, las 2 skills y el comando.

---

## 8. Solución de problemas frecuentes

| Síntoma | Causa probable | Solución |
|---|---|---|
| `supervisor-odoo` no aparece con Tab | Config no recargada; o `default_agent` apunta a agente no primary/hidden | Reiniciar OpenCode; revisar `mode: primary` en el fichero del supervisor |
| Un agente no aparece en `@` | Frontmatter mal escrito | Validar YAML; descripciones con `:` deben ir entre comillas |
| El supervisor no delega en un subagente | Ese agente no existe o no está en `permission.task` | Crear el fichero `.md` del agente y añadir `allow` en `permission.task` |
| Los permisos `ask` no piden confirmación | `opencode.json` del proyecto tiene `"*": allow` | Eliminarlo (ver apartado 4.1) |
| OpenCode no arranca por config rota | Fichero de configuración inválido | Arrancar con `OPENCODE_DISABLE_PROJECT_CONFIG=1` para editar; luego reiniciar sin la variable |
| Errores de YAML difíciles de localizar | Descripción con `:` o multilínea sin comillas | Poner la descripción entre comillas dobles en una sola línea |
| Activación de logs | Para diagnosticar | Añadir temporalmente `"logLevel": "DEBUG"` en `opencode.jsonc`; retirar después |

---

## 9. Checklist final de la IA

- [ ] `~/.config/opencode/opencode.jsonc` creado con `default_agent: supervisor-odoo`
- [ ] 9 agentes creados en `~/.config/opencode/agent/` (`supervisor-odoo`, `aprendiz-odoo`, `aprendiz-odoo-hard`, `sistema-odoo`, `explorador-odoo`, `revisor-odoo`, `probador-odoo`, `configurador-odoo`, `configurador-odoo-free`)
- [ ] `opencode.json` del proyecto creado sin `"*": allow`
- [ ] `AGENTS.md` del proyecto rellenado (sin `<...>` sin sustituir)
- [ ] Skills `odoo-dev` y `odoo-sysadmin` creadas
- [ ] Comando `/entrenar` creado
- [ ] Proveedor OpenCode Go conectado y modelos `opencode-go/*` visibles en `/models`
- [ ] OpenCode reiniciado y `supervisor-odoo` accesible con Tab
- [ ] Prueba de delegación superada (el supervisor invoca subagentes)
- [ ] Script de validación (apartado 7) ejecutado sin errores

---

## 10. Flujo de ejecución autónomo para la IA (runbook)

Orden exacto que la IA debe seguir para realizar la instalación y configuración **completa** en un servidor Ubuntu:

1. **Preparación:** comprobar que se ejecuta como el usuario de trabajo (no root). Instalar dependencias base: `sudo apt update && sudo apt install -y curl git python3 python3-yaml`.
2. **Instalar OpenCode** si falta: `curl -fsSL https://opencode.ai/install | bash` y verificar con `opencode --version`.
3. **Conectar el proveedor (paso manual del usuario):** solicitar al usuario que ejecute `/connect`, elija `OpenCode Go` y pegue la API key; verificar después con `opencode models` que aparecen los modelos `opencode-go/*`.
4. **Configuración global (sección 3):** `mkdir -p ~/.config/opencode/agent`; crear `opencode.jsonc` y los 9 agentes con el contenido exacto de los apartados 3.1 a 3.10.
5. **Configuración por proyecto (sección 4):** en el directorio del proyecto, crear `opencode.json`, rellenar `AGENTS.md` sustituyendo **todos** los placeholders `<...>`, y crear las skills y el comando de los apartados 4.3 a 4.5.
6. **Validación (sección 7):** ejecutar `python3 validar_config.py <ruta_proyecto>`; corregir cualquier error YAML/JSON y repetir hasta que pase sin errores.
7. **Comprobaciones manuales finales:** el usuario reinicia OpenCode y ejecuta las verificaciones de la sección 6 (Tab → `supervisor-odoo`, `@agentes`, `/entrenar`).
8. **Entrega:** devolver el checklist final (sección 9) cumplimentado e indicar qué pasos quedaron en manos del usuario (conexión del proveedor y reinicio).

Reglas generales:

- Reproducir el contenido de cada fichero **EXACTAMENTE** como indica el contexto; no "mejorar" ni reformatear las plantillas.
- Si un comando falla, leer el error, corregir la causa y reintentar; si se requiere `sudo`, pedir confirmación.
- No usar modelos gratuitos con datos confidenciales de clientes.
- No borrar ni sobrescribir configuración existente del servidor sin confirmación del usuario.

---

## 11. Referencias

- `Propuesta_entorno_multi_agente_opencode.md` (documento de diseño completo, mismo directorio)
- Esquema oficial de configuración: <https://opencode.ai/config.json>
- Documentación de agentes: <https://opencode.ai/docs/agents>
- OpenCode Go: <https://opencode.ai/docs/go>
- OpenCode Zen (incluye modelos gratuitos): <https://opencode.ai/docs/zen>
