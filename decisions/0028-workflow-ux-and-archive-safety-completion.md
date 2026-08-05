# Incremento 0028 — Workflow UX and Archive Safety Completion

> **Estado**: archived 2026-06-30
> **Plan**: `axiom.spec/plans/PLAN-INC-0028-workflow-ux-and-archive-safety-completion.md`
> **Archive**: `openspec/changes/2026-07-01-0028-workflow-ux-and-archive-safety/`

## Resumen ejecutivo

Incremento 0028 completa el Lote B de `0022` sobre ergonomía de
comandos de workflow, gate visible de archivado con QA paralelo y
consolidación del parser compartido de workflows.

## Mini-lotes

### E1 — Intent commands como chain wrappers

- `apps/cli/src/commands/intent.ts` (nuevo): `INTENT_CHAINS` con
  los 3 intents declarados en
  `command-protocol.yaml#intentCommandBehavior`.
- Cada intent encadena sub-comandos existentes en orden.

### E2 — Archive-gate visible

- `apps/cli/src/commands/qa-archive-gate.ts` (nuevo):
  `checkQaArchiveGate` retorna warning si `qaLane=parallel` y
  QA pendiente. NO bloquea (GATE 0028).
- Wire-up en `axiom-increment.ts`: el sub-comando `archive`
  invoca el gate antes de la transición.

### E3 — Parser `workflows.yaml` centralizado

- `packages/workflow/src/workflows-loader.ts` (nuevo):
  `loadWorkflowsConfig` con shape guard tolerante.
- 4 kinds de error tipados: io-error, invalid-yaml, invalid-shape,
  unsupported-schema.

## Métricas

- **1015/1015 tests verde** (105/105 files). +15 tests de 0028.
- **`tsc -b` verde**.
- **GATE 0022 honrada**: intent commands son chain wrappers.
- **GATE 0028 honrada**: archive-gate es warning visible, no
  bloqueo.

## Siguiente paso

Iniciar incremento 0015 (cavekit discipline post-MVP): capability
contracts nativos, no copia de tooling externo.
