# baseline group B — init role identity (no repoRole leak into authority identity)

> **Código**: INC-20260913-baseline-group-b-init-role-identity
> **Estado**: Implementado (pendiente de revisión humana)
> **Fecha de creación**: 2026-09-13
> **Tipo de cambio**: corrección de identidad persistida en `axiom init` (Grupo B del plan de baseline)
> **Plan**: `PLAN-BASELINE-TESTS-20260913` (paso 2 de 7)
> **Bug relacionado**: `BUG-20260910-baseline-workspace-setup-adoption-contract`

## Goal

Restaurar el contrato canónico de identidad persistida en el `axiom.yaml` que
`axiom init` emite para el repo de autoridad (`kind: axiom` → `role: axiom`),
de modo que `runRepoAdd` sobre un proyecto bare-init no falle con
`AXIOM_STRUCTURAL_INVALID_AXIOM_YAML`, sin romper el contrato vigente del
`role: sdd` para el repo legacy de control (`kind: legacy`) que consumen el
fan-out de `axiom upgrade` y el guard de repo-affinity.

## Context

- Baseline del 2026-09-13 (`npx vitest run` en `Axiom/`): 16 tests fallidos en
  9 archivos; el Grupo B concentra 1 fallo determinista en
  `apps/cli/tests/workspace-incremental.test.ts` (scenario "runRepoAdd sobre
  proyecto bare-init (item 4)").
- El commit `b3be943` cambió `buildAxiomYaml` en
  `apps/cli/src/commands/init.ts` para emitir `role: sdd` cuando el rol de init
  es `sdd` (línea `role: ${role === 'sdd' ? 'sdd' : canonicalManagedRole(kind)}`).
- `expectedIdentity` en `workspace-structural-plan.ts` espera
  `canonicalManagedRole(repo.kind)` (`axiom` para `kind: axiom`) vía
  `identityMatches`/`optionalRoleMatches` (match estricto, sin alias).
- El comentario canónico en `workspace-axiom-yaml.ts` dice: "`repoRole` is an
  internal adapter/runtime role (for example `sdd`) and must never leak into
  the persisted identity role."
- Resultado: `runRepoAdd` sobre bare-init self-hosted falla porque la identidad
  existente (`role: sdd`) no coincide con el desired state (`role: axiom`).

## Verificación previa (evidencia)

1. **Consumidores runtime del `role` persistido** (grep `role` + `sdd` en
   `apps/cli/src/`, `packages/*/src/`):
   - `apps/cli/src/commands/_repo-affinity.ts:100`: branch defensivo que
     tolera `sdd` Y `axiom` (no exige `sdd`).
   - `packages/cli-commands/src/commands/upgrade.ts:308`:
     `if (resolution.role !== 'sdd') return null;` — el fan-out de
     `axiom upgrade` sólo se dispara desde un repo con `role: sdd` persistido
     (modelo legacy del control-repo, `kind: legacy`).
   - `apps/cli/src/commands/member-install.ts:405` y
     `apps/cli/src/commands/workspace-setup.ts:1353`: patrón canónico
     (`repoRole: 'sdd'` interno, identidad persistida `role: 'axiom'`).
   - `packages/project-resolution/src/resolver.ts:427`: expone
     `role: config.role ?? config.kind` first-class (consumido por los dos
     anteriores).
2. **Historial git**: `git log -S "role === 'sdd' ? 'sdd'"` → sólo `b3be943`
   introdujo la condición. El diff de `b3be943` en `init.ts` es exactamente
   esa 1 línea; ningún test nuevo en ese commit depende del valor `sdd`
   persistido para `kind: axiom`.
3. **Hallazgo no previsto en el plan**: las aserciones
   `expect(parsed.role).toBe('sdd')` en `init.test.ts:94` y
   `schemaversion2-e2e.test.ts:158` son del contrato pre-r13 (introducidas en
   `57d66bc`, 2026-07-03, cuando `init` emitía `DEFAULT_REPO_ROLE`
   directamente). Quedaron rotas en `8274c69` (que introdujo
   `canonicalManagedRole` en `init.ts`) y `b3be943` las "arregló de facto".
   Ambas asercionan el layout **default** `installed-multi-repo`
   (`kind: legacy`), no el repo de autoridad.
4. **Consecuencia**: el revert literal propuesto en el plan
   (`role: ${canonicalManagedRole(kind)}`) emitiría `role: legacy` para el
   init default y rompería `init.test.ts:94`,
   `schemaversion2-e2e.test.ts:158`, el gate de fan-out (`upgrade.ts:308`) y
   el guard de repo-affinity para repos init-default. El propio brief exige
   `init.test.ts` en verde, por lo que el revert literal es inviable.

## Scope

### Incluido

- `apps/cli/src/commands/init.ts` (`buildAxiomYaml`): sustituir la condición
  `role === 'sdd' ? 'sdd' : canonicalManagedRole(kind)` por
  `kind === 'legacy' ? 'sdd' : canonicalManagedRole(kind)`, con comentario que
  documenta el contrato dual.

### Excluido

- `workspace-structural-plan.ts`, `workspace-axiom-yaml.ts` y tests (restricción
  del brief).
- `packages/core/src/self-update/**` y gates AC-078/079.
- Resto de grupos (A, C–G) del plan de baseline.
- Cualquier cambio en `optionalRoleMatches` (no se encontró evidencia que
  justifique aceptar `sdd` como alias de `axiom`).

## Non-goals

- No unificar el vocabulario `sdd`/`axiom` del rol persistido (deuda
  preexistente del modelo legacy control-repo; requiere su propio incremento).
- No actualizar las aserciones de `init.test.ts:94` /
  `schemaversion2-e2e.test.ts:158` (siguen vigentes para `kind: legacy`).

## Acceptance criteria

- AC-B-01: `buildAxiomYaml` emite `role: axiom` para el repo de autoridad
  (`kind: axiom`, layout self-hosted o `--role spec`).
- AC-B-02: `buildAxiomYaml` emite `role: sdd` para `kind: legacy` (init
  default `installed-multi-repo`), preservando el contrato del fan-out y del
  guard de repo-affinity.
- AC-B-03: `npx vitest run apps/cli/tests/workspace-incremental.test.ts` →
  42/42 (incluye el scenario bare-init item 4).
- AC-B-04: `init.test.ts`, `join.test.ts`, `member-install.test.ts` en verde;
  `configure.test.ts` mantiene únicamente su fallo preexistente del Grupo E
  ("mergea sobre un workspace.json existente").
- AC-B-05: `npx tsc -b` → exit 0.

## Risks

- Bajo: el vocabulario dual (`sdd` para legacy, `axiom` para autoridad) sigue
  siendo una asimetría deliberada; un futuro incremento de unificación del
  vocabulario de roles debería abordar `upgrade.ts:308` y `_repo-affinity.ts`
  juntos.

## Open questions

Ninguna bloqueante. La decisión de dirección del plan ("revertir a
`canonicalManagedRole(kind)`") se ajustó con evidencia: el revert literal
rompía el contrato vigente del init default. El fix aplicado es idéntico a
`b3be943` en todos los casos salvo el que rompía el Grupo B
(`kind: axiom` + `role: sdd`), porque en `init` `kind: legacy ⟺ role: sdd`
(`kind` deriva de `role` en `buildAxiomYaml`).

## Assumptions

- El modelo canónico es el de `buildRoleAwareAxiomYaml` (`workspace-setup.ts`):
  `canonicalManagedRole(kind)` para la identidad persistida, `repoRole`
  interno para el vocabulario de topología/registry.
- El gate de fan-out de `upgrade` (`resolution.role === 'sdd'`) sigue ligado
  al control-repo legacy y no al repo de autoridad.

## Implementation notes

Cambio único en `buildAxiomYaml` (`apps/cli/src/commands/init.ts`):

```ts
// Antes (b3be943):
`role: ${role === 'sdd' ? 'sdd' : canonicalManagedRole(kind)}`,
// Después:
`role: ${kind === 'legacy' ? 'sdd' : canonicalManagedRole(kind)}`,
```

Con comentario de bloque que documenta: (a) el repoRole interno `sdd` no se
filtra a la identidad del repo de autoridad (`kind: axiom` → `role: axiom`,
contrato que `expectedIdentity`/`canonicalManagedRole` exigen y que
`runRepoAdd` valida); (b) para `kind: legacy` (init default
installed-multi-repo, que emite `legacyFunction: sdd`) se conserva
`role: sdd`, contrato que consumen el fan-out de `@axiom/cli-commands`
upgrade (`resolution.role === 'sdd'`), el guard de repo-affinity y los tests
e2e del schema v2.

## Validation

| Validación | Resultado |
| --- | --- |
| `npx vitest run apps/cli/tests/workspace-incremental.test.ts` | **42/42 passed** (159.29s) — antes: 41/42 |
| `npx vitest run apps/cli/tests/init.test.ts apps/cli/tests/join.test.ts apps/cli/tests/configure.test.ts apps/cli/tests/member-install.test.ts` | **46/47** — único fallo: `configure.test.ts > Scenario 5 > mergea sobre un workspace.json existente` (preexistente del Grupo E, fuera de alcance) |
| `npx tsc -b` | **exit 0** |
| Regresión extra: `schemaversion2-e2e`, `upgrade-fanout`, `bindings`, `workspace-adopt`, `repair`, `memory`, `mcp-serve` | **50/50 passed** |
| Regresión extra: `e2e/workspace-mcp.e2e.test.ts` | **1/1 passed** |

## Result

Implementado y validado. Estado de criterios: AC-B-01 a AC-B-05 verificados.
El incremento queda `pending` de cierre formal hasta revisión humana y
consolidación en la spec canónica.

## General spec integration

Al cierre, consolidar en la spec canónica del schema v2 de `axiom.yaml` el
contrato dual del campo `role` persistido emitido por `init`:

- `kind: axiom` (repo de autoridad) → `role: axiom` (identidad canónica; el
  repoRole interno `sdd` nunca se filtra a la identidad persistida).
- `kind: legacy` (control-repo del layout default installed-multi-repo, con
  `legacyFunction: sdd`) → `role: sdd` (contrato consumido por el fan-out de
  `axiom upgrade` y el guard de repo-affinity).

Y registrar que `expectedIdentity` (`workspace-structural-plan.ts`) valida la
identidad con match estricto (`optionalRoleMatches` sin alias), por lo que
cualquier emisor de `axiom.yaml` debe respetar ese contrato dual.

## Trazabilidad y fuentes

- Plan: `Axiom.Spec/plans/PLAN-BASELINE-TESTS-20260913.md` (Grupo B).
- Bug relacionado: `BUG-20260910-baseline-workspace-setup-adoption-contract`.
- Commits: `b3be943` (introduce la condición defectuosa), `8274c69`
  (introduce `canonicalManagedRole` en `init.ts`), `57d66bc` (introduce las
  aserciones `role: sdd` del init default, pre-r13).

## Estado de validación humana

Pendiente de revisión humana.
