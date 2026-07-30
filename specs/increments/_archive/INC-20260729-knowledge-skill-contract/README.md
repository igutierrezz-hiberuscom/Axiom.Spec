# INC-20260729-knowledge-skill-contract

## Goal

Actualizar las skills de fase del SDD para incluir un contrato explícito de memoria Engram: cuándo guardar, qué guardar, qué NO guardar, y qué metadata incluir.

## Scope

- Añadir sección "Engram Memory Contract" a las skills de fase canónicas en `Axiom.Spec/specs/manuales/13_Skills_Agentes_y_Roles.md`
- Actualizar las plantillas de skill en `Axiom/axiom.spec/target-axiom-skills/` si existen
- El contrato cubre: analysis, architecture, frontend, backend, qa, validator
- Incluye reglas de qué guardar y qué NO guardar por fase
- Incluye la metadata obligatoria (project, increment, phase, actorRole, knowledgeKind, stability)

## Non-goals

- No modificar el runtime de skills
- No crear nuevas skills
- No modificar el catálogo de skills

## Acceptance Criteria

1. El manual `13_Skills_Agentes_y_Roles.md` incluye una sección "Engram Memory Contract" con reglas por fase
2. Las reglas por fase cubren: analysis, architecture, frontend, backend, qa, validator
3. Cada fase especifica qué guardar y qué NO guardar
4. Se documenta la metadata obligatoria
5. No se rompe la estructura existente del manual

## Implementation Notes

- Añadida sección "Contrato de memoria Engram por fase" al final de `Axiom.Spec/specs/manuales/13_Skills_Agentes_y_Roles.md`
- La sección cubre: metadata obligatoria (7 campos), reglas generales, reglas por fase (analysis, architecture, frontend, backend, QA/validator), flujo entre fases, y relación con el Knowledge Harvest
- Sin cambios en runtime — puramente documental

## Validation

- No validation command was found. Performed best-effort validation by inspecting changed files and checking consistency against the requested behavior.
- El manual mantiene su estructura original; la nueva sección es aditiva al final

## Result

- 1 archivo modificado: `Axiom.Spec/specs/manuales/13_Skills_Agentes_y_Roles.md`
- 0 fallos

## Status

closed