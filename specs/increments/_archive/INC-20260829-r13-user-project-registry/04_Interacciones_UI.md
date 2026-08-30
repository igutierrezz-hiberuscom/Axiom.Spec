# 04 Interacciones UI

## Superficies

CLI `axiom projects`, proyección de proyectos del launcher y handlers MCP que consumen el catálogo.

## Flujo

`list` carga/valida y proyecta disponibilidad; `add|join` valida identidad/ownership antes del lock; `use` actualiza solo recencia y opcionalmente imprime un path. En `--json` no se mezcla copy humana.

## Estados visibles

Las vistas muestran disponibilidad física precisa, resolubilidad solo donde aplica y errores de dominio accionables. Se retira cualquier copy de proyecto “activo” o seleccionado.

## Comportamiento reactivo

Launcher/MCP reutilizan tipos y loader; no reinterpretan `stale`, no silencian corrupción y no introducen selección global. La seguridad HTTP del launcher queda para F.
