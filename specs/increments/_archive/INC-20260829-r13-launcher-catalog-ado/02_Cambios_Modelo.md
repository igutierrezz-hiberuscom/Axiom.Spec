# 02 Cambios de Modelo

## Acción reconciliada

Cada action referencia `workflowId`, `workflowCommand` cuando aplica, `cliInvocation`, required fields e intent. Al construir catálogo se verifica que workflow/command existen; action lifecycle sin mapping es error.

## Identidad

`LifecycleMutationInput` incluye `id` obligatorio. El guard canónico compara ID solicitado con metadataId/workflow activo y devuelve mismatch tipado antes del governed transition.

## Resultado ADO

`LocalFirstResult { local; remote?: pending|created|linked|failed }` mantiene estados independientes. URLs se proyectan mediante `safeExternalHttpUrl`.

## Compatibilidad

No hay modo de derivación implícita ni fallback para entradas lifecycle.
