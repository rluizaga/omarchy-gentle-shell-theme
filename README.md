# Restaurar sincronización de tema Omarchy → Gentle Shell

Carpeta de respaldo para recrear la integración de tema entre Omarchy y Gentle
Shell en caso de restablecer la configuración. Copias espejo de las rutas de
origen.

## Estructura (rutas de origen → ubicación en esta carpeta)

| Origen | Respaldo |
|---|---|
| `~/.config/omarchy/hooks/theme-set.d/sync-gentle-shell-theme.hook` | `config/omarchy/hooks/theme-set.d/sync-gentle-shell-theme.hook` |
| `~/.gentle-shell/agent/themes/omarchy-system.json` | `gentle-shell/agent/themes/omarchy-system.json` |
| `~/.gentle-shell/agent/settings.json` | `gentle-shell/agent/settings.json` |
| `~/Documents/omarchy-gentle-shell-theme-sync.md` | `omarchy-gentle-shell-sync.md` |

## Restaurar

```bash
# 1. Hook (obligatorio; mantiene el vínculo Omarchy → Gentle Shell)
mkdir -p ~/.config/omarchy/hooks/theme-set.d
cp config/omarchy/hooks/theme-set.d/sync-gentle-shell-theme.hook \
   ~/.config/omarchy/hooks/theme-set.d/sync-gentle-shell-theme.hook
chmod +x ~/.config/omarchy/hooks/theme-set.d/sync-gentle-shell-theme.hook

# 2. Tema generado por Omarchy (opcional: el hook lo regenera en el próximo cambio)
mkdir -p ~/.gentle-shell/agent/themes
cp gentle-shell/agent/themes/omarchy-system.json \
   ~/.gentle-shell/agent/themes/omarchy-system.json

# 3. settings.json (opcional: el hook lo escribe por sí solo al correr)
cp gentle-shell/agent/settings.json ~/.gentle-shell/agent/settings.json
```

## Verificación tras restaurar

```bash
bash -n ~/.config/omarchy/hooks/theme-set.d/sync-gentle-shell-theme.hook   # sintaxis
ls -l ~/.config/omarchy/hooks/theme-set.d/sync-gentle-shell-theme.hook     # debe ser -rwxr-xr-x

# Prueba manual (idempotente y segura)
~/.config/omarchy/hooks/theme-set.d/sync-gentle-shell-theme.hook "tokyo_night"
```

Un reinicio de Gentle Shell (o `/reload`) hace efectivo el tema la primera vez.
A partir de ahí, cada `omarchy theme set "..."` propaga el cambio solo.

## Documentación completa

Ver `omarchy-gentle-shell-sync.md` en esta misma carpeta para el diagnóstico,
el diseño y las decisiones tomadas.
