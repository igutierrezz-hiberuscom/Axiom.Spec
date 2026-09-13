# 02 Cambios de Modelo

## Objetivo del documento

Detallar dónde se engancha la verificación y qué contratos quedan afectados.

## Punto de enganche

- El ejecutor de transiciones gobernado de `@axiom/workflow`, que ya coordina metadata, efectos declarados, movimiento a `_archive/` y receipts. La verificación se ejecuta como parte de la transición de cierre e integración, no como comando suelto.
- `apps/cli/src/commands/integrate.ts` y los wrappers de `axiom-increment` y `axiom-bug` consumen el resultado sin duplicar lógica.
- El launcher y las herramientas MCP obtienen el mismo comportamiento por consumir el mismo ejecutor.

## Estructuras nuevas

- Un contrato de alcance documental: qué categorías existen y cómo se determina cuáles aplican a un cambio.
- Un registro de resultado por cierre, con documento, resultado y motivo cuando aplica, emitido junto a los receipts existentes.
- Un modo de severidad declarado (aviso o bloqueo) con su condición de activación.

## Contratos o estados afectados

- El contrato de cierre gana un paso verificable. Un cierre con documentación en alcance sin revisar deja de considerarse completo.
- No cambia la semántica de las transiciones existentes: ninguna transición ilegal pasa a ser legal.
- No cambia el contrato de QA ni la política de aprobación.
- Los receipts existentes se mantienen; el resultado de la verificación se añade como información auditable, no los sustituye.

## Dependencias de implementación

`ACC-041`, `ACC-043` y `ACC-045` definen el ejecutor común, el contrato único de QA y la resolución única de workflows. Este incremento se apoya en ellos; si alguno no está cerrado, se implementa contra el ejecutor vigente sin duplicarlo y se declara la deuda.
