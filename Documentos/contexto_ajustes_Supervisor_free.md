# Contexto de ajustes — Supervisor-Odoo-Free (modelo del agente)

> Documento de contexto para que una IA (o un técnico) replique este ajuste en otro servidor
> con el entorno multi-agente gratuito de OpenCode. Redactado a partir de la intervención
> real realizada en el servidor de producción de KLO (ia.klo.es) el 2026-11-17.

---

## 1. Objetivo

Corregir el campo `model:` del agente `supervisor-odoo-free`, que apunta a un ID de modelo
**que ya no existe en el catálogo**, y dejarlo apuntando a un modelo gratuito activo.

| | |
|---|---|
| **Modelo muerto (a sustituir)** | `opencode/muse-spark-1.3-contributor-free` |
| **Modelo nuevo (destino)** | `opencode/glm-5.3-flash-free` (coste 0, `status: active`) |
| **Fichero a editar** | `$HOME/.config/opencode/agent/supervisor-odoo-free.md` (línea 4, frontmatter) |
| **Copia espejo a sincronizar** | `/usr/local/share/opencode/agent/supervisor-odoo-free.md` (si existe) |

**Síntoma que origina el ajuste:** el agente arranca con el modelo global de pago
(`opencode-go/deepseek-v4-pro`) en vez de un modelo gratuito, porque el ID configurado
desapareció del catálogo y OpenCode hace *fallback* al `model` global de `opencode.jsonc`.

---

## 2. Prerrequisitos y comprobaciones previas

Antes de tocar nada, la IA debe verificar:

1. **Qué copias del agente existen** (puede haber 1 o 2, según cómo se instaló):

   ```bash
   find / -name "supervisor-odoo-free.md" -not -path "*/proc/*" 2>/dev/null
   ```

   Rutas habituales:
   - `~/.config/opencode/agent/supervisor-odoo-free.md` → **la activa** (la que lee el servicio; `~` es el HOME del usuario que corre el servicio OpenCode, p. ej. `/var/lib/opencode` o el usuario real).
   - `/usr/local/share/opencode/agent/supervisor-odoo-free.md` → plantilla de instalación (puede ser `root:root`, solo lectura).

2. **Línea actual del modelo** en cada copia:

   ```bash
   grep -n "^model:" <ruta>/supervisor-odoo-free.md
   ```

   Si ya muestra `glm-5.3-flash-free` (o cualquier ID `active` y coste 0), el ajuste ya está hecho.

3. **Que el modelo destino existe en el catálogo del servidor**:

   ```bash
   opencode api get /api/model | grep -o '"id":"opencode/glm-5.3-flash-free"'
   ```

   Si no existe en ese servidor, elegir un sustituto **gratuito** (`cost 0/0`, `status: active`)
   de la lista de la API (`opencode api get /api/model`). Candidatos verificados en el
   catálogo de KLO: `big-pickle`, `beam-free`, `flint-40b-free`, `mimo-v2.6-flash-free`,
   `glm-5.3-flash-free`, `hy4-flash-free`, `iceman-free`, `longcat-2.5-free`.
   Para un supervisor (planificación, revisión, entrenamiento) preferir los no-`flash`/no-`small`.

4. **Si el fichero está en un contenedor/sandbox con `no_new_privs=1`** (`grep NoNewPrivs /proc/self/status`),
   `sudo` no funcionará ahí aunque la clave sea correcta: es un bloqueo de kernel. En ese caso,
   la edición de `~/.config` la hace el propio usuario del servicio (sin sudo), y la copia de
   `/usr/local/share` solo puede hacerla un root **dentro de ese mismo entorno** (consola del
   contenedor/mantenimiento), nunca desde una shell del anfitrión que no ve esas rutas.

---

## 3. Procedimiento

### Paso 1 — Copia de seguridad de cada fichero que se vaya a editar

```bash
cp ~/.config/opencode/agent/supervisor-odoo-free.md ~/.config/opencode/agent/supervisor-odoo-free.md.bak-$(date +%Y%m%d)
# Si la plantilla existe y es editable:
sudo cp /usr/local/share/opencode/agent/supervisor-odoo-free.md /usr/local/share/opencode/agent/supervisor-odoo-free.md.bak-$(date +%Y%m%d)
```

### Paso 2 — Editar SOLO la línea `model:` del frontmatter

Editar `~/.config/opencode/agent/supervisor-odoo-free.md` (línea 4). Con sed:

```bash
sed -i 's|^model: opencode/muse-spark-1.3-contributor-free$|model: opencode/glm-5.3-flash-free|' \
    ~/.config/opencode/agent/supervisor-odoo-free.md
```

**Reglas estrictas:**
- Cambiar únicamente esa línea. **No tocar**: `description`, `mode: primary`, `temperature: 0.1`,
  `steps: 40`, el bloque `permission.task` (`"*": deny` + los 7 agentes `-free` en `allow`) ni el cuerpo del prompt.
- El frontmatter debe seguir delimitado por `---` (líneas 1 y 18) y ser YAML válido.

### Paso 3 — Sincronizar la plantilla de `/usr/local/share` (si existe y no coincide)

⚠️ **Sentido obligatorio: de la config activa → hacia la plantilla.** Nunca al revés
(machacaría el ajuste con el modelo muerto).

```bash
sudo cp ~/.config/opencode/agent/supervisor-odoo-free.md /usr/local/share/opencode/agent/supervisor-odoo-free.md
```

Si `/usr/local/share/opencode` no existe en el servidor, omitir este paso (no es imprescindible:
esa copia es solo plantilla de instalación y el servicio la tiene ensombrecida).

### Paso 4 — Verificación

```bash
# a) Las copias (si hay 2) deben ser idénticas:
diff ~/.config/opencode/agent/supervisor-odoo-free.md /usr/local/share/opencode/agent/supervisor-odoo-free.md

# b) La API del servicio ya debe servir el modelo nuevo:
opencode api get /api/agent | grep -A3 '"supervisor-odoo-free"'
#   Esperado:  supervisor-odoo-free -> opencode/glm-5.3-flash-free

# c) El resto del equipo free no debe haber cambiado:
opencode api get /api/agent | grep -B1 -A3 '"[a-z-]*-free"'
#   Esperado: aprendiz-odoo-free, aprendiz-odoo-hard-free, probador-odoo-free,
#             revisor-odoo-free, sistema-odoo-free  -> opencode/big-pickle
#             configurador-odoo-free, explorador-odoo-free -> opencode/mimo-v2.6-flash-free
```

Si el servicio no refleja el cambio, revisar que se editó el `~/.config` del **usuario que corre
el servicio** (no el de otro usuario) y reiniciar el servicio OpenCode.

---

## 4. Aplicarlo en la sesión del usuario (manual, no scriptable)

Estos puntos no los puede ejecutar la IA; hay que indicárselos al usuario:

1. **Las sesiones ya abiertas conservan su modelo anclado.** El `model:` del frontmatter solo
   aplica a sesiones nuevas. Solución: crear una sesión nueva del agente, o cambiar el modelo
   a mano en la sesión actual.
2. **Selector de modelos:** los modelos se agrupan por proveedor. Si la sesión está en un modelo
   de pago (`opencode-go/*`), el picker abre la sección *"OpenCode Go"* y los gratuitos
   (`opencode/*`, sección **"KLO / OpenCode"**) no aparecen hasta cambiar de sección.
   También conviene recargar el cliente (F5) si estaba abierto antes del cambio.
3. Recordar que la config global (`opencode.jsonc`) define `model` global
   (`opencode-go/deepseek-v4-pro` en KLO) y `default_agent` (`supervisor-odoo`, el de pago):
   eso es intencional y **no debe tocarse** para este ajuste.

---

## 5. Checklist final

- [ ] Localizadas todas las copias de `supervisor-odoo-free.md`
- [ ] Modelo destino verificado en el catálogo (`active`, coste 0)
- [ ] Backups creados antes de editar
- [ ] Línea 4 cambiada solo en la config activa del usuario del servicio
- [ ] Plantilla `/usr/local/share` sincronizada (sentido activo → plantilla) u omitida con justificación
- [ ] `diff` idéntico entre copias
- [ ] `/api/agent` devuelve `opencode/glm-5.3-flash-free` para `supervisor-odoo-free`
- [ ] Los otros 7 agentes `-free` intactos
- [ ] Informado al usuario: sesión nueva o selección manual en sección "KLO / OpenCode"

---

## 6. Lecciones aprendidas (para la IA que ejecute esto)

- Un `model:` que apunte a un ID inexistente **no da error visible**: el agente arranca con el
  modelo global (de pago) y solo se detecta comparando la API con el fichero. Verificar siempre
  el ID contra `opencode api get /api/model` antes de escribirlo en un agente.
- Los catálogos cambian: los IDs `-free` de hoy (`big-pickle`, `mimo-v2.6-flash-free`,
  `glm-5.3-flash-free`) pueden desaparecer. Este documento define el **procedimiento**, no el
  modelo definitivo; re-elegir entre los `active` de coste 0 disponibles en el momento.
- Distinguir siempre: config **activa** (`~/.config` del usuario del servicio) vs **plantilla**
  (`/usr/local/share`), y comprobar en qué entorno (host vs contenedor) se ejecuta cada comando
  antes de usar `sudo`.
