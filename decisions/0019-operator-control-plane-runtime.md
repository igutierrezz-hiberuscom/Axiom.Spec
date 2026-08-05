# Incremento 0019 — Operator Control Plane Runtime

> **Estado**: closed (2026-06-30)
> **Plan**: `axiom.spec/plans/PLAN-INC-0019-operator-control-plane-runtime.md`
> **Archive**: `openspec/changes/archive/2026-06-29-0019-*/`
> **Summary**: `openspec/changes/archive/2026-06-29-0019-plan/0019-archive-summary.md`

## Resumen ejecutivo

Este incremento cierra el gap entre el runtime MVP de Axiom
y una superficie operativa comparable a GentleAI, manteniendo
el modelo Axiom-first: configuración declarativa versionada en
`axiom.config/*.yaml`, estado mutable project-scoped en
`.axiom-state/<project>/`, y proyección por adapter target. El primer
target con cobertura completa es `opencode`; el resto entra
con fallback explícito.

## Comandos nuevos

| Comando | Descripción |
|---|---|
| `axiom upgrade` | Aplica el plan de migraciones del runtime de versionado, con checkpoint pre-upgrade y rollback automático. |
| `axiom tui` | Abre la TUI operativa (configure, sync, doctor, upgrade, model routing). |
| `axiom model show` | Muestra el routing efectivo por slot. |
| `axiom model set/unset/reset` | Gestiona overrides por slot. |
| `axiom model validate` | Corre los doctor checks MRC-001..004 del routing. |
| `axiom components list/show` | Catálogo de componentes gestionados. |
| `axiom components install/uninstall/restore` | Mutaciones con checkpoint pre-mutación. |
| `axiom skills list/refresh/drift` | Registry y drift del catálogo de skills. |

## Packages nuevos

- `@axiom/versioning` (Lote A; ya estaba en el MVP; esta
  versión cierra el contrato de managed state + checkpoints
  + migrations + upgrade).
- `@axiom/tui` — shell TUI + router + 4 flows + previews +
  summaries.
- `@axiom/cli-commands` — barrel de re-export de los `runX`
  que la TUI consume.
- `@axiom/model-routing` — loader, validator, slots,
  resolver, fallback, assignments, mutations, checks,
  opencode projection.
- `@axiom/components` — catalog, install/uninstall/restore.
- `@axiom/skills` — registry, refresh, drift.

## Support por target

| Target | Support level | Multi-mode routing | Fallback |
|---|---|---|---|
| `opencode` | `multi-mode` | sí | no |
| `claude-code` | `single-mode` | no | `medium` |
| `copilot-vscode` | `fallback-only` | no | `medium` |
| (otros) | `fallback-only` | no | `medium` |

Cuando el target no soporta multi-mode, todos los slots caen
a `medium` con `fallbackReason` visible. El operador ve el
warning en `axiom model validate` y en
`axiom model show --target <otro>`.

Para detalles del contrato de cada `ModelClass` (cheap,
medium, strong, local), ver `axiom.config/model-routing-policy.yaml`.

## Métricas

- **Tests**: 681/681 verde (tolera flake pre-existente de
  tool-routing).
- **Doctor**: 40/0 estable.
- **Build**: `tsc -b` exit 0.

## Próximos pasos (Lote 0020+)

> **Estado al 2026-06-30**: los 5 gaps abiertos declarados en este doc **fueron migrados a `roadmap.md#deferred-work`** durante el cierre del rango 0020-0025. El rango completo (0020 a 0025) está archivado. El flake de `tool-routing` y los 4 gaps de integration/coverage siguen abiertos; quedaron documentados en `axiom.spec/product/roadmap.md` para fase posterior.

1. Cerrar el flake de `tool-routing`.  ← **Migrado a deferred work** (roadmap §Deferred work).
2. Integrar el model routing con el opencode adapter
   propiamente (actualmente la projection es estática).  ← **Migrado a deferred work**.
3. TUI integration para `model validate` y `components show`.  ← **Migrado a deferred work**.
4. Coverage real de `claude-code` y `copilot-vscode` (no
   solo fallback).  ← **Migrado a deferred work**.
5. `axiom skills apply` idempotente (actualmente
   `refresh` re-deriva sin materializar).  ← **Migrado a deferred work**.
