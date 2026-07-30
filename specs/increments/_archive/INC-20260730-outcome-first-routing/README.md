# INC-20260730-outcome-first-routing

Status: closed
Date: 2026-07-30

## Goal

Introducir en las superficies de tooling de Axiom un contrato outcome-first que separe la clasificación de la petición (`flow`) de la decisión de ejecución (`route`), manteniendo el workflow SDD ligero y explícito.

## Context

Las superficies actuales invocan directamente el lifecycle SDD de incrementos, bugs y revisiones, pero no distinguen cambios pequeños que pueden resolverse inline, delegación sin lifecycle, conocimiento reutilizable ni emergencias con control reforzado. El contrato debe ser coherente entre Copilot, Claude, `.agents` y OpenCode sin crear un control plane ni artefactos sintéticos.

## Scope

- Actualizar los prompts de `axiom-increment`, `axiom-bug` y `axiom-review`.
- Actualizar la skill `axiom-autopilot` y los agentes/workflows equivalentes que expresan el mismo contrato operativo.
- Definir `route`: `direct_inline`, `delegated_direct`, `sdd`.
- Definir `flow`: `increment`, `bug`, `knowledge_only`, `emergency`.
- Declarar que las rutas directas no crean incrementos, fases SDD ni artefactos sintéticos.
- Mantener `knowledge_only` sobre el flujo existente de Knowledge Harvest.

## Non-goals

- Modificar `Axiom/` o cualquier runtime, CLI o catálogo de producto.
- Crear un work-routing control plane, receipts o un índice de rutas.
- Convertir rutas directas en artefactos persistentes.
- Consolidar todavía cambios en `Axiom.Spec/specs/00_Resumen_Ejecutivo.md` a `08_Glosario.md`.
- Archivar este incremento.
- Habilitar auto-push para ningún flujo.

## Acceptance criteria

- [x] Existe esta spec concisa con contexto, alcance, límites, riesgos, validación y estado real.
- [x] El contrato `route`/`flow` aparece coherentemente en las superficies soportadas relevantes.
- [x] Se declara que `direct_inline` y `delegated_direct` no crean incrementos, fases SDD ni artefactos sintéticos.
- [x] `sdd` requiere aceptación explícita cuando se ofrece como alternativa y `axiom-autopilot` sigue siendo un orquestador SDD.
- [x] `emergency` exige confirmación y alcance explícitos y no habilita auto-push.
- [x] Cada workflow ejecutor condiciona la creación de lifecycle SDD a `route=sdd` y ofrece una salida explícita para rutas directas.
- [x] `knowledge_only` tiene una rama ejecutable que invoca Knowledge Harvest y `emergency` tiene un preflight bloqueante.
- [x] La redacción conserva Bootstrap Orchestrator Mode y no añade infraestructura pesada.
- [x] Todos los archivos modificados se validan mediante búsqueda y relectura; si no hay comando aplicable, se documenta la validación exigida por `AGENTS.md`.

## Risks

- La duplicación del contrato entre adaptadores puede derivar si una superficie se actualiza de forma incompleta.
- El término `flow` podría confundirse con la ruta de ejecución si no se mantiene la separación explícita.
- La referencia a Knowledge Harvest debe seguir siendo read-only y no crear un lifecycle SDD nuevo.

## Open questions

- Ninguna bloqueante para este incremento. La selección concreta de `route` se mantiene deliberadamente como decisión contextual y no se materializa en un registro persistente.

## Assumptions

- La ruta canónica de incrementos en este workspace es `Axiom.Spec/specs/increments/`.
- `axiom knowledge harvest --increment <id>` es el flujo existente que debe reutilizar `flow=knowledge_only`.
- `axiom-autopilot` continúa reservado para lotes que sí requieren orquestación SDD.

## Implementation notes

- Mantener los cambios confinados a documentación operativa de `Axiom.SDD` y este README.
- Usar una definición común y breve en cada superficie soportada, con el mismo vocabulario y guardrails.
- No añadir comandos, persistencia, receipts ni mecanismos de routing externos.

## Validation

No validation command was found. Performed best-effort validation by inspecting changed files and checking consistency against the requested behavior.

Additional checks completed:

- Búsqueda focalizada de `route`, `flow`, `direct_inline`, `delegated_direct`, `knowledge_only`, `emergency`, `auto-push` y aceptación explícita en cada superficie modificada.
- Relectura de los 17 archivos de tooling modificados y de este README.
- Chequeo global ejecutable: 17 superficies de tooling, seis gates ejecutores, el preflight del autopilot y este README, sin tokens ni guardrails ausentes.
- `git diff --check` sin errores en `Axiom.SDD`; el warning CRLF observado en `Axiom.Spec/specs/manuales/13_Skills_Agentes_y_Roles.md` pertenece a un cambio local no relacionado.
- Revisión independiente posterior a la reparación: confirmó los seis gates ejecutores, el preflight del autopilot y la coherencia de delegación en Claude; recomendación `closed`.
- No se ejecutó build o suite runtime porque el incremento es tooling/documentación-only y no modifica `Axiom/`.

## Result

- Contrato outcome-first implementado de forma coherente en Copilot, Claude, `.agents` y OpenCode, con gates operativos para rutas directas, `knowledge_only` y `emergency`.
- No se modificó `Axiom/`, `Axiom.Spec/specs/00..08` ni `Axiom.Spec/context/**`.
- No se introdujeron control plane, receipts, persistencia de rutas, comandos nuevos ni auto-push.
- Fallos introducidos: ninguno. Los cambios locales no relacionados de `Axiom.Spec` se conservaron intactos.

## General spec integration

No se consolida `Axiom.Spec/specs/00..08` en este worker, por instrucción explícita. El cambio es tooling-only y el conocimiento operativo queda en las superficies de `Axiom.SDD` y en esta spec.
