# 04 Interacciones UI

## Superficies

CLI `workspace`, `repo`, `role`, `projects`; launcher onboarding/workspace; MCP.

## Flujo

La operación resuelve localmente, valida identidad/topology, carga workspace state tipado y solo entonces muta. El catálogo se muestra como descubrimiento/recencia, no como autoridad.

## Estados visibles

Diferenciar proyecto local resuelto, catálogo ausente, proyección divergente, workspace state ausente/corrupto y mutación created/updated/unchanged. No se muestra attach ni kinds control/spec.

## Reactividad

Actualizar adapters/providers refresca `updatedAt` solo si hay cambio; consumidores ven el mismo estado. ACC-068 completará la paridad de pasos después del coordinador E.
