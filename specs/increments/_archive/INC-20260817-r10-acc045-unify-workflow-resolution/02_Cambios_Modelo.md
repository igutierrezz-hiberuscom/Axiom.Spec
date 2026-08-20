# 02 Cambios de Modelo

## Objetivo del documento

Hacer que el YAML de workflows sea la única definición editable y que todos los canales compartan una resolución fail-closed.

## Entidades o estructuras afectadas

- `axiom.config/workflows.yaml` y su asset empaquetado en `@axiom/workflow`.
- `DEFAULT_WORKFLOWS`, `resolveWorkflowsConfig`, `resolveWorkflowConfig` y la paridad de subcomandos.
- Consumers CLI, launcher, MCP e integrate.

## Contratos o estados afectados

Sólo un YAML ausente elige el default empaquetado. YAML presente válido resuelve el proyecto; YAML presente inválido o con schema no soportado devuelve error explícito, sin fallback silencioso. El launcher y las superficies de ayuda derivan el grafo efectivo.

## Notas de compatibilidad

No se añaden workflows ni se ejecutan transiciones durante la resolución. El detalle de empaquetar el asset es implementación; el contrato estable es un grafo único y fail-closed.
