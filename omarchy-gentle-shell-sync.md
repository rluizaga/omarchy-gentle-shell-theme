# Sincronización de tema Omarchy → Gentle Shell

> Documentación de diseño e implementación — generada y verificada en la sesión.

## Objetivo

Que **Gentle Shell** siga automáticamente el tema activo de **Omarchy** (el default o
cualquiera elegido), de la misma forma en que ya lo hace **pi-agent**, sin intervención
manual y sin `/reload`.

## Contexto previo (diagnóstico)

### Cómo pi-agent ya sigue el tema de Omarchy

El flujo completo de Omarchy al aplicar un tema:

1. El tema define `colors.toml`.
2. `omarchy-theme-set-templates` renderiza la plantilla
   `/usr/share/omarchy/default/themed/pi.json.tpl` y genera
   `~/.local/state/omarchy/current/theme/pi.json` (tema `omarchy-system`, con colores ya
   resueltos: `mix`, gradientes, etc.).
3. `omarchy-theme-set-pi` copia ese `pi.json` a
   `~/.pi/agent/themes/omarchy-system.json` y escribe `"theme": "omarchy-system"` en
   `~/.pi/agent/settings.json`.

### El problema

`/usr/bin/omarchy-theme-set-pi` tiene el directorio de pi **hardcodeado**:

```bash
PI_AGENT_DIR="$HOME/.pi/agent"
```

Por eso pi-agent (`~/.pi/agent`) sí sigue a Omarchy, pero Gentle Shell
(`~/.gentle-shell/agent`) nunca se entera: usa su propio `settings.json` (con el tema fijo
`Gentleman-Cute`) y no recibe el tema generado.

### Restricción

`/usr/share/omarchy/` es de solo lectura en la práctica (se sobrescribe en cada
`omarchy update`). La solución debe vivir en `~/.config/`, que es el punto de extensión
soportado.

## Solución implementada

Un hook en el punto de extensión oficial de Omarchy:

```
~/.config/omarchy/hooks/theme-set.d/sync-gentle-shell-theme.hook
```

Omarchy ejecuta todos los scripts de `theme-set.d` **después** de los comandos
`post_theme_commands` (incluido `omarchy-theme-set-pi`), pasando el nombre del tema en
snake-case como `$1`. Para cuando corre el hook, el `pi.json` ya existe en
`~/.local/state/omarchy/current/theme/pi.json`.

### Qué hace el hook

1. Lee `~/.local/state/omarchy/current/theme/pi.json` (fuente generada por Omarchy).
2. Resuelve el home de Gentle Shell dinámicamente.
3. Copia el tema a `<gentle-home>/themes/omarchy-system.json` (reemplazo **atómico**:
   `temp` + `mv`).
4. Escribe `"theme": "omarchy-system"` en `<gentle-home>/settings.json` (con `jq`, para
   preservar todos los demás campos).

## Decisiones tomadas

| Decisión | Elección | Motivo |
|---|---|---|
| Nombre del tema | Fijo `omarchy-system` | Es el mismo patrón de `omarchy-theme-set-pi`; no requiere mapear nombres de tema. Todos los temas de Omarchy renderizan al mismo `omarchy-system`. |
| Home de Gentle Shell | Resuelto dinámicamente, no hardcodeado | `PI_CODING_AGENT_DIR` → `GENTLE_PI_AGENT_HOME` → `~/.gentle-shell/agent`, replicando la resolución de `gentle-shell.mjs`. Soporta `--home` custom. |
| Escritura de `settings.json` | `jq '.theme = "omarchy-system"'` | Preserva `tuiMode`, `packages`, `extensions`, `defaultProvider`, `defaultModel`, etc. |
| Reemplazo del tema | Atómico (`temp` + `mv`) | El watcher de pi nunca lee un archivo a medio escribir. |
| Reintento de espera | Hasta ~3 s (30 × 0.1 s) | Defensivo; en la práctica el archivo ya existe porque el hook corre después de los generators. |
| `$1` (nombre de tema) | No se usa | Innecesario con tema fijo `omarchy-system`. |

## Por qué recarga en vivo

pi **watcha** el archivo del tema custom activo
(`<agentDir>/themes/<name>.json`) y lo recarga automáticamente (debounce de 100 ms).
Como el nombre del tema queda fijo en `omarchy-system`, cada cambio del archivo por parte
del hook se aplica en vivo, **sin `/reload`**.

## Estructura relevante

```
/usr/share/omarchy/default/themed/pi.json.tpl        # plantilla (solo lectura)
~/.local/state/omarchy/current/theme/pi.json         # render de Omarchy
/usr/bin/omarchy-theme-set-pi                        # sync propia de Omarchy → ~/.pi/agent
~/.config/omarchy/hooks/theme-set.d/sync-gentle-shell-theme.hook   # nuestro hook
~/.gentle-shell/agent/themes/omarchy-system.json     # tema destinado (Gentle Shell)
~/.gentle-shell/agent/settings.json                  # "theme": "omarchy-system"
```

## Cómo se usa (cambiar tema desde terminal)

```bash
omarchy theme current           # tema activo
omarchy theme list              # temas disponibles
omarchy theme set "Tokyo Night" # aplicar tema (dispara el hook automáticamente)
```

## Archivos cambiados

| Archivo | Acción | Descripción |
|---|---|---|
| `~/.config/omarchy/hooks/theme-set.d/sync-gentle-shell-theme.hook` | **Creado** (nuevo) | Hook que sincroniza el tema Omarchy hacia Gentle Shell. Ejecutable, 2593 bytes. |
| `~/.gentle-shell/agent/themes/omarchy-system.json` | **Creado** (nuevo) | Tema `omarchy-system` copiado desde el render de Omarchy. 2718 bytes. |
| `~/.gentle-shell/agent/settings.json` | **Modificado** | `"theme"` pasó de `Gentleman-Cute` a `omarchy-system`. Resto de campos intactos. |
| `~/Documents/omarchy-gentle-shell-theme-sync.md` | **Creado** (nuevo) | Este documento. |

Ningún archivo de `/usr/share/omarchy/` fue modificado (solo lectura).

## Verificación realizada

- `bash -n` (sintaxis) y `chmod +x` OK.
- Ejecución manual del hook: `exit 0`, sincronizó el tema y cambió `settings.json`
  (`Gentleman-Cute` → `omarchy-system`), preservando el resto de campos.
- JSON de tema válido (`jq`), sin placeholders `{{ }}` sin resolver.
- Colores renderizados de Tokyo Night (`background: #1a1b26`).

## Nota final

En el primer cambio real conviene reiniciar Gentle Shell (o `/reload`) una vez para que
tome el nuevo tema; a partir de ahí, cada cambio de tema en Omarchy se propaga solo.
