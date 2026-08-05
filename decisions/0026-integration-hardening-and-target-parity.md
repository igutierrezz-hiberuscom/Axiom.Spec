# Incremento 0026 — Integration Hardening and Target Parity

> **Estado**: archived 2026-06-30
> **Plan**: `axiom.spec/plans/PLAN-INC-0026-integration-hardening-and-target-parity.md`
> **Archive**: `openspec/changes/2026-07-01-0026-integration-hardening/`

## Resumen ejecutivo

Este incremento cierra la paridad operativa de integraciones project-scoped,
resuelve el flake de `tool-routing`, integra el routing real con el adapter
de opencode, hace `axiom skills apply` idempotente, y agrega TUI integration
para `model validate` y `components show`. Resultado: `0014` deja de estar
`runtime partial`; el soporte por target refleja comportamiento real verificado.

## Mini-lotes

### A1 — Flake tool-routing eliminado

- `routeTool` y los emitters derivan `emittedAt` de `call.requestedAt`,
  garantizando pureza observable byte a byte.
- 31 tests en `route-tool.test.ts` quedan verdes sin tocar el test.

### A2 — `axiom skills apply` idempotente

- Sub-comando `axiom skills apply [--skill <id>...] [--dry-run]`.
- Materializa `skills-pending.json` project-scoped. NO toca el lockfile
  (D3 read-only respetado).
- 12 tests nuevos en `apply.test.ts`.

### A3 — Opencode adapter consume routing proyectado

- `loadRoutingSnapshot(projectRoot)` lee `<root>/.opencode/model-routing.json`
  si existe; tolera ausencia y malformación.
- 7 tests nuevos en `routing-snapshot.test.ts`.

### A4 — TUI integration

- Nuevos screens `model-validate` y `components-show` con flags
  `--model-validate` y `--components-show <id>` en `axiom tui`.
- 0 tests focales nuevos (la integración es de wire-up).

### A5 — Support matrix 5 targets MVP

- `SUPPORT_MATRIX` extendido con `antigravity` y `visual-studio-2026`.
- Helpers `MVP_TARGETS` e `isMvpTarget` exportados.
- 10 tests en `support-matrix.test.ts` (4 originales + 6 nuevos).

### A6 — Documentación y openspec

- Roadmap y plan actualizados; openspec `2026-07-01-0026-...` con
  proposal + tasks + spec + archive-report.

## Métricas

- **963/963 tests verde** (98/98 files). +19 tests nuevos vs baseline.
- **`tsc -b` verde** sin output.
- **40/40 doctor-contracts** verde.
- **GATE 0026 honrada**: la support matrix refleja comportamiento real
  verificado, no intención documental.

## Siguiente paso

Iniciar incremento 0027 (toolchain provider expansion and repair).
