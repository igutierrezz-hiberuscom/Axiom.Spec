# Criterios de aceptación — INC-20260811-acc-031-r08-docs

## CA-1: README de agents reconciliado

- [x] `Axiom/packages/agents/README.md` menciona 14 agents del catálogo real.
- [x] El README describe que el materializado real de agents en workspace lo
      hace `workspace-process-surfaces.ts`.

## CA-2: Comentario de DEFAULT_DESIRED_SKILLS corregido

- [x] El comentario en `apply.ts` ya no afirma sincronización con
      `toolchain.yaml` (subset `mvp: true`).

## CA-3: Sin regresiones

- [x] No hay cambios de comportamiento ni de lógica.
- [x] La validación disponible (build/tests) pasa sin regresiones nuevas.