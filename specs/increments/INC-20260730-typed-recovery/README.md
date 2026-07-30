# INC-20260730-typed-recovery: Tipado Estricto de Errores

## Goal
Asegurar que todos los errores del framework expuestos a agentes o usuarios tengan un `code` bien definido para facilitar la recuperación automática (recovery) sin depender del frágil "message matching".

## Scope
- Definir un tipo de error base `AxiomError` (o adaptar el existente) que requiera un campo `code` (e.g. `AXIOM_NO_PLAN`, `AXIOM_NO_CANDIDATE`).
- Reemplazar lanzamientos de error en el framework y orquestador (en particular los de validación y fallas de dependencias) por errores tipados con `code`.
- Esto aplica principalmente a `packages/core`, `packages/workflow` y `apps/cli`.

## Non-goals
- No se reescribirán todos los try-catch de bajo nivel de librerías externas.
- Los errores de `fs` o red puros pueden mantenerse envueltos o conservar el original, siempre que el error principal emitido al orquestador sea tipado.

## Acceptance Criteria
- Se exporta la clase `AxiomError` (o se extiende la estructura del framework).
- Se audita y cambian varios casos clave para usar el nuevo `code`.
- Los subagentes pueden consultar `error.code` para tomar decisiones deterministas.
