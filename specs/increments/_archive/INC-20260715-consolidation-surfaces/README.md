# Increment: consolidation-surfaces

Status: closed
Date: 2026-07-15

## Goal

Install two missing post-implementation "closure/consolidation" flows as
first-class Axiom surfaces (today they exist only in this workspace's
dogfooding, or not at all):

- **`axiom-spec-integrator`** — consolidate durable knowledge from an
  implemented increment/bug into the project's canonical spec + archive it
  (confirm-gated).
- **`axiom-tech-context`** — author/maintain the project's technical context
  (master doc + auxiliaries) + detect spec-drift against real code.

Both adapter-agnostic, generic (no stack/tool specifics).

## Context

`Axiom/apps/cli/src/commands/workspace-process-surfaces.ts` already
materializes 5 process surfaces (`axiom-spec-author` + `axiom-role-planner`
for the `spec` role, `axiom-role-implementer` for `code`,
`axiom-sdd-orchestrator` + `axiom-phase-reviewer` for `sdd`), each backed by a
catalog skill (`axiom.config/skills-catalog.yaml`, 15 entries) and agent
contract (`axiom.config/agents-catalog.yaml`, 11 entries). None of them
consolidates an implemented increment/bug's durable knowledge back into the
project's canonical spec, and none authors/maintains the project's technical
context or detects drift between an archived spec and the real code. This
increment adds both as reusable, generic content mirroring the existing
skill/agent shapes (`axiom-spec-author`, `axiom-phase-reviewer`).

## Scope

- New skill source
  `Axiom/axiom.spec/target-axiom-skills/axiom-spec-integrator.md` + catalog
  entry in `skills-catalog.yaml` (correct sha256 `bundleHash`).
- New skill source
  `Axiom/axiom.spec/target-axiom-skills/axiom-tech-context.md` + catalog
  entry in `skills-catalog.yaml` (correct sha256 `bundleHash`).
- New agent source
  `Axiom/axiom.spec/target-axiom-agents/axiom-spec-integrator.md` + catalog
  entry in `agents-catalog.yaml` (`role: planning`, correct sha256
  `bundleHash`).
- New agent source
  `Axiom/axiom.spec/target-axiom-agents/axiom-tech-context.md` + catalog
  entry in `agents-catalog.yaml` (`role: planning`, correct sha256
  `bundleHash`).
- Two new entries in `SURFACES` (`workspace-process-surfaces.ts`), both added
  to `surfaceIdsForRole('spec')`.
- Updated pre-existing guard suites that hard-code catalog counts/membership
  or the exact surface-id set for the `spec` role (see Implementation
  notes).

## Non-goals

- No new npm dependencies.
- No stack/adapter-specific content (no KVP/.NET/Angular/Cypress/ADO
  specifics) in the new skill/agent sources; "canonical spec" stays generic
  ("los ficheros de spec canónica del proyecto"), never hardcoding this
  workspace's own `00_*.md … 08_*.md` filenames.
- No new MCP tools invented — both skills only reference MCP tools that
  already exist in `packages/mcp-tools/src/registry.ts`
  (`spec.incrementRead`, `spec.bugRead`, `spec.technicalContextIndexRead`,
  `spec.recommendedContextList`, `sdd.skillIndexRead`, `sdd.transitionApply`,
  `memory.contextRecall`) plus the existing code-intelligence-with-fallback
  idiom already used by `axiom-role-planner`/`axiom-role-implementer`.
- Neither surface ever commits/pushes; both are confirm-gated for any state
  change.
- No enterprise lifecycle constructs.

## Acceptance criteria

- [x] `axiom-spec-integrator` skill + agent + surface installed for the
      `spec` role with correct sha256 `bundleHash` values in both catalogs.
- [x] `axiom-tech-context` skill + agent + surface installed for the `spec`
      role with correct sha256 `bundleHash` values in both catalogs.
- [x] `surfaceIdsForRole('spec')` returns `['axiom-spec-author',
      'axiom-role-planner', 'axiom-spec-integrator', 'axiom-tech-context']`.
- [x] Content is generic and adapter/stack-agnostic (no KVP/.NET/Angular/
      Cypress/ADO specifics; no hardcoded `00_*.md…08_*.md` filenames).
- [x] Both surfaces are confirm-gated for any state change and never
      commit/push (explicit absolute rules in both skill sources).
- [x] All guard suites green: `npm run build`;
      `packages/skills/tests/catalog.test.ts`;
      `packages/agents/tests/catalog.test.ts`;
      `packages/doctor/tests/skills.test.ts`;
      `packages/doctor/tests/agents.test.ts`;
      `apps/cli/tests/workspace-process-surfaces.test.ts`.
- [x] No unrelated files modified; only additions to existing
      skills/surfaces/catalogs plus the guard-test expectations they force.

## Open questions

none — resolved by the brief.

## Assumptions

- `axiom-spec-integrator`'s "move state / archive the increment folder"
  wording maps onto the existing `sdd.transitionApply` (confirm-gated) tool —
  no new MCP tool for "archive" exists in the substrate, and the brief
  explicitly names `sdd.transitionApply` as the mechanism, consistent with
  how transition side-effects are already modeled as declarative per-
  transition data (`packages/workflow/src/transition-effects.ts`).
- `axiom-tech-context`'s relatedCommand `axiom context status` is a loose
  reference (same convention as `axiom-explorer`'s agent source, which also
  references `axiom context status` without that command literally
  performing drift detection today) — the actual authoring target
  (`<specRepo>/context/**/*.md` + its mechanical index) is `axiom context
  index`, mentioned inline in the skill body; the brief's catalog contract
  fixes `relatedCommands: ["axiom context status"]` explicitly, so it is
  honored verbatim.
- Doctor's TC-010/TC-011 smoke tests, the real-catalog integration tests in
  `packages/skills/tests/catalog.test.ts`/`packages/agents/tests/catalog.test.ts`,
  and `apps/cli/tests/workspace-process-surfaces.test.ts`'s
  `surfaceIdsForRole('spec')` expectation all hard-code the exact
  real-catalog count/membership/surface-id-set and therefore needed updating
  in lockstep with the new entries (grepped the repo for other hard-coded
  real-catalog assertions; none found beyond these five files/expectations).

## Implementation notes

- `axiom.spec/target-axiom-skills/axiom-spec-integrator.md`: mirrors the
  `axiom-spec-author.md`/`axiom-phase-reviewer.md` shape (`Qué hace` /
  `Cuándo se usa` / `Cómo lo hace` phases — anchor, durable-vs-narrative
  discovery, atomic integration plan for confirmation, application deferring
  major architecture to `axiom-tech-context`, confirm-gated archive / `Reglas
  absolutas` / `Herramientas MCP` / `Parámetros de materialización` /
  `Relación con otras piezas` / `Estado`). References
  `axiom-role-close-doc` and `axiom-structured-doubts` by id.
- `axiom.spec/target-axiom-skills/axiom-tech-context.md`: same shape,
  covering both authoring (master doc + auxiliaries, `[PENDIENTE DE
  VERIFICAR]` for unverifiable claims, no orphan docs, explicit environment
  block when the code repo is absent) and spec-drift detection
  (criterion-by-criterion, fixed vocabulary `✅ Alineado / ⚠️ Modificado /
  ❌ Desviado / ❓ No verificable`, detection-only — never decides
  resolution). References `axiom-plan-drift-alignment` by id. Uses the same
  code-intelligence-with-physical-fallback idiom as
  `axiom-role-planner`/`axiom-role-implementer`.
- Both agent sources are thin contracts mirroring `axiom-spec-author.md`/
  `axiom-phase-reviewer.md` (agent) shape; `role: planning` for both.
- `bundleHash` computed with:
  `node -e "const c=require('crypto'),fs=require('fs');console.log('sha256:'+c.createHash('sha256').update(fs.readFileSync(process.argv[1])).digest('hex'))" <path>`
  — see Validation below for the 4 verified values.
- `workspace-process-surfaces.ts`: added `SPEC_INTEGRATOR_BODY` and
  `TECH_CONTEXT_BODY` (condensed versions of the skill content, same
  `{role}`/`{repoPath}`/`{specRepo}`/`{projectId}` placeholder convention —
  neither body uses `{role}`/`{repoPath}` literally, same as
  `ROLE_PLANNER_BODY`) and two new entries in `SURFACES`
  (`agentRole: 'planning'`, `placement: 'mandatory'`;
  `relatedCommand: 'axiom sdd advance'` for the integrator,
  `relatedCommand: 'axiom context status'` for tech-context).
  `surfaceIdsForRole('spec')` now returns `['axiom-spec-author',
  'axiom-role-planner', 'axiom-spec-integrator', 'axiom-tech-context']`.
  Top-of-file doc comment updated to reflect the new `spec` role mapping.
- Updated `apps/cli/tests/workspace-process-surfaces.test.ts`: the
  `surfaceIdsForRole('spec')` unit expectation and the integration test's
  spec-repo materialization loop, both extended to the 4-id array; top
  comment updated for accuracy.
- Updated the 4 catalog-count guard suites (real-catalog integration +
  doctor smoke): `packages/skills/tests/catalog.test.ts` (15→17
  entries/ids), `packages/agents/tests/catalog.test.ts` (11→13
  entries/ids/roles), `packages/doctor/tests/skills.test.ts`
  (`15/15`→`17/17` regex + description), `packages/doctor/tests/agents.test.ts`
  (`11/11`→`13/13` regex + description).

## Validation

Ran from `Axiom/` (repo root):

1. `npm run build` (`tsc -b`) — clean, no output.
2. Recomputed the 4 new `bundleHash` values with the node one-liner above and
   diffed them against both catalogs — exact match for all 4:

   ```
   skill axiom.spec/target-axiom-skills/axiom-spec-integrator.md MATCH sha256:65821b32802d405f1f869444c7af84d28f71aa7d5335678186a302165e6a389f
   skill axiom.spec/target-axiom-skills/axiom-tech-context.md MATCH sha256:f54badcc30e7d145e490ccc5fa4e405761f124424e07a27982d9adedf1ce74bc
   agent axiom.spec/target-axiom-agents/axiom-spec-integrator.md MATCH sha256:e5792dde07ee419d3aea98ef3583b8ae2837e379c1ac953de23f401535714203
   agent axiom.spec/target-axiom-agents/axiom-tech-context.md MATCH sha256:e9256fc49c69dfa5327863942840a627c3671b21ebbc3ebf55c856cf1e687ce9
   ```

3. `npx vitest run packages/skills/tests/catalog.test.ts packages/agents/tests/catalog.test.ts packages/doctor/tests/skills.test.ts packages/doctor/tests/agents.test.ts apps/cli/tests/workspace-process-surfaces.test.ts`:

   ```
    ✓ packages/skills/tests/catalog.test.ts (11 tests) 77ms
    ✓ packages/agents/tests/catalog.test.ts (13 tests) 94ms
    ✓ packages/doctor/tests/agents.test.ts (9 tests) 410ms
    ✓ packages/doctor/tests/skills.test.ts (9 tests) 417ms
    ✓ apps/cli/tests/workspace-process-surfaces.test.ts (5 tests) 1438ms

    Test Files  5 passed (5)
         Tests  47 passed (47)
   ```

   PRE-EXISTING baseline test counts (11+13+9+9+5 = 47, matching the baseline
   this brief stated: 15 skills, 11 agents, both doctor suites and the
   process-surface suite green) preserved exactly — only the real-catalog
   count/membership/surface-id-set assertions inside them were updated; no
   test removed, added, or weakened. No PRE-EXISTING or NEW failures.
4. Extra safety net beyond the brief's required commands: `npm test` (full
   workspace) — **285 test files / 2871 tests, all passed**, confirming no
   regression anywhere else in the monorepo from the shared-file edits
   (`workspace-process-surfaces.ts`, both catalogs, the 5 guard test files).

## Result

Closed. Both surfaces landed.

- `axiom-spec-integrator`: consolidation/archival flow — reads a
  technically-closed increment/bug, distinguishes durable knowledge from
  implementation narrative, presents an atomic integration plan for
  confirmation, applies it against the project's canonical spec with source
  references, updates technical context only for minor/moderate changes
  (deferring major architecture to `axiom-tech-context`), and archives the
  artifact via confirm-gated `sdd.transitionApply`. Authored as skill +
  agent (`role: planning`) + mandatory surface for the `spec` role.
- `axiom-tech-context`: technical-context authoring/maintenance (master doc +
  auxiliaries, verifiable against real code, `[PENDIENTE DE VERIFICAR]` for
  gaps, explicit environment block when the code repo is absent, no orphan
  docs) plus spec-drift detection (criterion-by-criterion against an
  archived increment/bug, fixed vocabulary `✅ Alineado / ⚠️ Modificado /
  ❌ Desviado / ❓ No verificable`, detection-only). Authored as skill + agent
  (`role: planning`) + mandatory surface for the `spec` role.
- `surfaceIdsForRole('spec')` now returns exactly `['axiom-spec-author',
  'axiom-role-planner', 'axiom-spec-integrator', 'axiom-tech-context']`.
- Both catalogs (`skills-catalog.yaml`: 15→17; `agents-catalog.yaml`: 11→13)
  updated with verified sha256 `bundleHash` values.
- All target guard suites are green (47/47 tests, same baseline count as
  before this increment).

## General spec integration

Integrado por el pase final de la orquestación (autopilot, 2026-07-15):
- `01_Requisitos_Funcionales.md` → **RF-AXM-036**.
- `04_Flujos_SDD_y_Ciclo_de_Vida.md` → sección "Ciclo con gates de calidad instaladas" (cierre: tech-context + spec-integrator).
- `06_Integraciones_y_Capacidades.md` → catálogo ampliado (consolidación y contexto).
- `07_Gobierno_y_Seguridad.md` → nota "Consolidación atómica y confirm-gated".
- `08_Glosario.md` → términos "`axiom-spec-integrator`" y "`axiom-tech-context`".

Archivado en `Axiom.Spec/specs/increments/_archive/`.
