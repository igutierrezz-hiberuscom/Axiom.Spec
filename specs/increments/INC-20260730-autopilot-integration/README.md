# INC-20260730-autopilot-integration: Autopilot Integration

## Goal
Integrar las nuevas piezas de gobierno verificable en el orquestador principal (`axiom-autopilot`), asegurando que las directivas de seguridad se apliquen en flujos desatendidos.

## Scope
- Actualizar `SKILL.md` de `axiom-autopilot` para requerir el uso de scope tipado.
- Requerir que el orquestador capture y verifique los recibos (`axiom phase receipt`) antes de integrar conocimientos.
- Requerir que verifique el hash congelado (`axiom freeze`) antes de lanzar subagentes.

## Non-goals
- Reescribir la lógica base de descomposición del orquestador.
- Modificar el comportamiento de los subagentes internos (ya cubiertos en los incrementos anteriores).

## Acceptance Criteria
- El skill de `axiom-autopilot` incluye las instrucciones explícitas.
- El ciclo de validación de candidate freeze y recibos es un requerimiento formal en el workflow de autopilot.
