# 03 Criterios de Aceptación

## Criterios de aceptación

### AC-DEC-01: Artefactos de decisión formalizados
Cada una de las decisiones D-01 a D-06 cuenta con un artefacto propio en `Axiom.Spec/decisions/` con status `proposed`, justificación clara y alternativa descartada documentada.

### AC-DEC-02: Trazabilidad cruzada
Cada decisión está vinculada a `PLAN-INC-20260911-r14-operator-control-surfaces` y a `INC-20260911-r14-operator-decisions-baseline`.

### AC-DEC-03: Validación de índice
`axiom index validate` valida satisfactoriamente la totalidad de metadata del repositorio sin advertencias ni errores.

### AC-DEC-04: Cero mutación en runtime
No se altera ningún archivo de código en `Axiom/` dentro de este incremento.
