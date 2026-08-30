# 04 Interacciones UI

## Objetivo del documento

Declarar que no se introduce una interfaz gráfica ni interacción nueva.

## Superficie UI afectada

La única superficie observable es la ayuda y ejecución existente de la CLI `axiom mcp serve`; este incremento no cambia sus argumentos ni resultados.

## Flujo de interacción

El operador continúa configurando el broker project-scoped mediante las rutas existentes. La documentación corregida evita recomendar procesos históricos como alternativa.

## Estados visibles

Sin estados nuevos: la disponibilidad, errores y configuración son los que ya expone el runtime MCP.

## Cascadas y comportamiento reactivo

No aplica. No hay cambios de launcher, TUI ni frontend.
