# 04 Interacciones UI

## Objetivo del documento

Presentar la misma decisión de transición y la misma confirmación explícita en
la CLI headless, el launcher y MCP, sin que cada superficie replique reglas de
workflow.

## Superficie UI afectada

- CLI: subcomandos increment, bug, plan, role, QA E2E e integrate.
- Launcher: endpoints de acciones lifecycle y su preview HTTP.
- MCP: `sdd.transitionApply`.

## Flujo de interacción

1. La superficie solicita una transición sin confirmación y recibe preview:
   legalidad, estado origen/destino, necesidad de aprobación, efectos y QA.
2. El operador confirma con `--confirm` o `confirmed: true`.
3. El runner aplica la transición o devuelve un error accionable sin ocultar
   bloqueos de QA, persistencia o archive.

## Estados visibles

Se distinguen preview no persistido, confirmación requerida, aplicación
persistida, bloqueado por QA y fallo recuperado o inconsistente. MCP conserva
su interacción de dos llamadas: preview y posterior apply confirmado.

## Cascadas y comportamiento reactivo

El launcher reenvía `confirmed` a cada subcomando y sólo anuncia una acción
como ejecutada tras el resultado del runner. Las acciones git, review y
verificación funcional de role mantienen sus gates propios; no cambian la
política de aprobación declarada del workflow.
