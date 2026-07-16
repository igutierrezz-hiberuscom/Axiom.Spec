# Increment: phase-reviewer

Status: closed
Date: 2026-07-15

## Goal

Install a first-class per-phase review discipline into target projects. Today
the rich review discipline (loop-until-dry sweep + findings ledger + scoped
re-review) exists only in this workspace's dogfooding
`.claude/agents/axiom-review.md` and is never installed into a project. Deliver:

- **P0** — an `axiom-phase-reviewer` process surface + catalog skill + catalog
  agent that encodes spec/plan/code review with an explicit OK/KO verdict and a
  loop-until-dry findings ledger.
- **P1** — expose review as launcher actions, reusing the existing
  `buildReviewPrompt` prompt engine.

## Context

`Axiom/apps/cli/src/commands/workspace-process-surfaces.ts` already materializes
3 process surfaces (`axiom-spec-author` + `axiom-role-planner` for the `spec`
role, `axiom-role-implementer` for `code`, `axiom-sdd-orchestrator` for `sdd`),
each backed by a catalog skill (`axiom.config/skills-catalog.yaml`) and, for
agent contracts, `axiom.config/agents-catalog.yaml`. None of them are a review
gate. `Axiom/packages/launcher` already ships `buildReviewPrompt` (spec/plan/code
`reviewKind` variants) but no launcher action ever calls it — `craftPrompt`'s
`buildPrompt` only branches on `spec`/`bug`/`plan`/`impl` prompt intents.

This increment condenses the bootstrap `.claude/agents/axiom-review.md`
discipline (exhaustive loop-until-dry sweep, findings ledger with
`REVIEW-NNN`/lens/location/severity/status/evidence, scoped re-review) into a
new installed skill+agent+surface, generalized to 3 explicit lenses (spec, plan,
code) with an explicit `VEREDICTO: OK | KO`, and wires it into the launcher as a
`review` action family.

## Scope

- New skill source `Axiom/axiom.spec/target-axiom-skills/axiom-phase-reviewer.md`
  + catalog entry in `skills-catalog.yaml` (correct sha256 `bundleHash`).
- New agent source `Axiom/axiom.spec/target-axiom-agents/axiom-phase-reviewer.md`
  + catalog entry in `agents-catalog.yaml` (`role: product-review`, correct
  sha256 `bundleHash`).
- New `axiom-phase-reviewer` entry in `SURFACES` (`workspace-process-surfaces.ts`),
  added to `surfaceIdsForRole('sdd')` (alongside `axiom-sdd-orchestrator`).
- `packages/launcher`:
  - `'review'` added to `LauncherPromptIntent`.
  - A `review` action family (`review-spec`/`review-plan`/`review-code`) in
    `createLauncherActions`, plus `review: 'increment'` in `FAMILY_TO_WORKFLOW`.
  - `buildPrompt` delegates to `buildReviewPrompt` for `promptIntent: 'review'`.
  - `ACTION_RECONCILIATION` entries for the 3 review keys, routed to a new
    `AXIOM_PHASE_REVIEWER_SKILL_ID` (`'axiom-phase-reviewer'`) skill target
    (distinct from `AXIOM_SDD_SKILL_ID`) via a new optional
    `ActionReconciliation.skillId` override.
- `apps/cli/src/commands/app-launcher.ts`: the 3 review keys in
  `LAUNCHER_ACTION_COMMAND` as non-executable (preview-only, read-only).
- Updated pre-existing guard suites that hard-code catalog counts/membership or
  the exact skill-id-per-target invariant (see Implementation notes).

## Non-goals

- No new npm dependencies.
- No stack/adapter-specific content (no KVP/.NET/Angular/Cypress/ADO specifics)
  in the new skill/agent sources.
- No change to `@axiom/tui`.
- The review surface never mutates: it crafts prompts / states verdicts, never
  writes code or spec content, never commits.
- No enterprise lifecycle constructs.

## Acceptance criteria

- [x] `axiom-phase-reviewer` skill + agent + surface installed for the `sdd`
      role with correct sha256 `bundleHash` values in both catalogs.
- [x] Review is exposed as 3 launcher actions (`review-spec`/`review-plan`/
      `review-code`) whose `craftPrompt` output contains the corresponding
      `buildReviewPrompt` review directive.
- [x] All guard suites green: `npm run build`; `packages/skills/tests/catalog.test.ts`;
      `packages/agents/tests/catalog.test.ts`; `packages/doctor/tests/skills.test.ts`;
      `packages/doctor/tests/agents.test.ts`; `apps/cli/tests/workspace-process-surfaces.test.ts`;
      `packages/launcher` (full, snapshots regenerated intentionally, no diff);
      `apps/cli/tests/app-launcher.test.ts`.
- [x] No unrelated files modified.

## Open questions

none — resolved by orchestrator brief.

## Assumptions

- `workflowId: 'increment'` (nearest reconciliation) and no `workflowCommand` is
  the correct `ACTION_RECONCILIATION` shape for `review-*`, since review is a
  read-only gate, never a state transition — confirmed by the brief.
- The pre-existing `adapter-routing.test.ts` assertion that every Claude
  Code/Copilot skill target resolves to `AXIOM_SDD_SKILL_ID` needed a scoped
  update (not a removal) once `review-*` targets started resolving to a
  different, deliberately distinct skill id (`axiom-phase-reviewer`); this is
  an in-scope test correction, not an unrelated-file edit, since it directly
  guards the routing table this increment changes.
- Doctor's TC-010/TC-011 smoke tests (`packages/doctor/tests/skills.test.ts`,
  `packages/doctor/tests/agents.test.ts`) and the real-catalog integration tests
  in `packages/skills/tests/catalog.test.ts` / `packages/agents/tests/catalog.test.ts`
  all hard-code the exact real-catalog count/membership and therefore needed
  updating in lockstep with the new catalog entries (grepped the repo for other
  hard-coded real-catalog assertions; none found beyond these four files).

## Implementation notes

### P0 — installed discipline

- `axiom.spec/target-axiom-skills/axiom-phase-reviewer.md`: mirrors the
  `axiom-spec-author.md` shape (`Qué hace` / `Cuándo se usa` / 3 lentes / `Cómo lo
  hace` (ancla, barrido hasta seco, ledger, veredicto, rerevisión acotada) /
  `Reglas absolutas` / `Herramientas MCP` / `Parámetros de materialización` /
  `Relación con otras piezas` / `Estado`). References `axiom-structured-doubts`
  and `axiom-functional-checklist-coverage` by id. Never invokes
  `sdd.transitionApply`.
- `axiom.spec/target-axiom-agents/axiom-phase-reviewer.md`: thin contract
  mirroring `axiom-reviewer.md`/`axiom-spec-author.md` (agent) shape;
  `role: product-review`.
- `bundleHash` computed with:
  `node -e "const c=require('crypto'),fs=require('fs');console.log('sha256:'+c.createHash('sha256').update(fs.readFileSync(process.argv[1])).digest('hex'))" <path>`
  — `sha256:fd584bc7e39c2c019bfee76632e19e3158a6a70427d7f1782337d3b2a2f4896b`
  (skill) and `sha256:f908b37f6bc2ef0fc56c316276d4480e0c4a229e38a3931a33a65f8459503c32`
  (agent); both verified to match the catalog entries.
- `workspace-process-surfaces.ts`: added `PHASE_REVIEWER_BODY` (condensed
  version of the skill content, with the same `{role}`/`{repoPath}`/
  `{specRepo}`/`{projectId}` placeholder convention) and a
  `'axiom-phase-reviewer'` entry in `SURFACES` (`agentRole: 'product-review'`,
  `relatedCommand: 'axiom sdd advance'`, `placement: 'mandatory'`).
  `surfaceIdsForRole('sdd')` now returns
  `['axiom-sdd-orchestrator', 'axiom-phase-reviewer']`.
- Updated `apps/cli/tests/workspace-process-surfaces.test.ts`'s
  `surfaceIdsForRole('sdd')` expectation to the 2-id array.
- Updated the 4 catalog-count guard suites (real-catalog integration + doctor
  smoke): `packages/skills/tests/catalog.test.ts` (14→15 entries/ids),
  `packages/agents/tests/catalog.test.ts` (10→11 entries/ids/roles),
  `packages/doctor/tests/skills.test.ts` (`14/14`→`15/15` regex + description),
  `packages/doctor/tests/agents.test.ts` (`10/10`→`11/11` regex + description).

### P1 — launcher exposure

- `types.ts`: `LauncherPromptIntent` gained `'review'`. `index.ts` re-exports the
  type as-is (no separate literal list to update).
- `action-catalog.ts`: `FAMILY_TO_WORKFLOW.review = 'increment'`; a new
  `buildReviewActions()` builds the 3 `review-*` actions as an explicitly
  `LauncherActionDefinition[]`-typed array (not via `.map()`, to keep the
  string-literal fields — `promptIntent`, `mode`, etc. — narrowed instead of
  widened to `string` under `tsc -b`'s strict checking), each with
  `promptIntent: 'review'`, `promptMode: 'start'`, `family: 'review'`,
  `mode: 'execute'`, and fields `[reviewIdField (optional id), sharedPromptTailField]`.
- `prompt-builder.ts`: `buildPrompt` now short-circuits when
  `action.promptIntent === 'review'`: derives `reviewKind` from `action.key`
  (`review-plan`→`'plan'`, `review-code`→`'code'`, else `'spec'`), calls
  `buildReviewPrompt(options.agentMention, options.resolvedContext, reviewKind)`,
  and — per the brief — prepends ONLY the agent-tuning preamble (no
  structured/context/notes blocks) when `options.agentTuning` is present.
- `adapter-routing.ts`: new `AXIOM_PHASE_REVIEWER_SKILL_ID = 'axiom-phase-reviewer'`
  constant; `ActionReconciliation` gained an optional `skillId` override (default
  stays `AXIOM_SDD_SKILL_ID` for every pre-existing key); 3 new
  `ACTION_RECONCILIATION` entries (`review-spec`/`review-plan`/`review-code`)
  with `workflowId: 'increment'`, no `workflowCommand`, `cli: 'axiom-review
  <kind>'`, `skillId: AXIOM_PHASE_REVIEWER_SKILL_ID`; `skillRoutingMap()` updated
  to `id: rec.skillId ?? AXIOM_SDD_SKILL_ID` (a 1-line generalization —
  `cliRoutingMap()` needed no change since it already used `rec.cli` verbatim).
  `AXIOM_PHASE_REVIEWER_SKILL_ID` re-exported from `index.ts`.
- `apps/cli/src/commands/app-launcher.ts`: `LAUNCHER_ACTION_COMMAND` gained the 3
  review keys as `{ executable: false, reason: 'Revisión: genera un prompt de
  revisión (sin mutación).' }`, so `execute` stays preview-only (mirrors how
  `plan-new`/etc. are guarded).
- Test updates (all in-scope — either exercising the new behavior or guarding
  invariants this increment changes):
  - `packages/launcher/tests/adapter-routing.test.ts`: the "every skill target
    resolves to `AXIOM_SDD_SKILL_ID`" assertion is now keyed per-target (skips
    `review-*`, which resolve to `AXIOM_PHASE_REVIEWER_SKILL_ID` instead); added
    a new test asserting the 3 review keys route to
    `AXIOM_PHASE_REVIEWER_SKILL_ID` with no `workflowCommand`.
  - `packages/launcher/tests/adapter-routing-snapshot.test.ts`: added a new
    `describe` block asserting `craftPrompt('review-spec', ...)` produces
    `/axiom-phase-reviewer` as the header and a prompt containing the spec
    review directive, plus a check that `review-plan`/`review-code` select the
    plan/code review directive text.
  - `packages/launcher/tests/action-catalog.test.ts` and
    `apps/cli/tests/app-launcher.test.ts` needed NO changes: neither asserts an
    exact action/family count (both use `expect.arrayContaining`/substring
    checks), and the "every family reconciles to a workflow" test passes once
    `FAMILY_TO_WORKFLOW.review` exists.
  - `packages/launcher` snapshots were regenerated (`vitest run -u`); the diff
    was empty (the only pre-existing snapshot covers `increment-new`, untouched
    by this change) — re-ran without `-u` to confirm stability.

## Validation

Ran from `Axiom/` (repo root):

1. `npm run build` (`tsc -b`) — clean, no output.
2. Recomputed both new `bundleHash` values with the node one-liner above and
   diffed them against the catalog entries — exact match for both the skill and
   the agent source.
3. `npx vitest run packages/skills/tests/catalog.test.ts packages/agents/tests/catalog.test.ts packages/doctor/tests/skills.test.ts packages/doctor/tests/agents.test.ts apps/cli/tests/workspace-process-surfaces.test.ts`:

   ```
   ✓ packages/skills/tests/catalog.test.ts (11 tests) 129ms
   ✓ packages/agents/tests/catalog.test.ts (13 tests) 163ms
   ✓ packages/doctor/tests/agents.test.ts (9 tests) 463ms
   ✓ packages/doctor/tests/skills.test.ts (9 tests) 473ms
   ✓ apps/cli/tests/workspace-process-surfaces.test.ts (5 tests) 1148ms

   Test Files  5 passed (5)
        Tests  47 passed (47)
   ```

   PRE-EXISTING baseline test counts (11+13+9+9+5 = 47) preserved exactly — only
   the real-catalog count/membership assertions inside them were updated, no
   test removed or weakened.
4. `npx vitest run -u packages/launcher` then `npx vitest run packages/launcher`
   (both to confirm the snapshot update is a no-op and the suite is stable):

   ```
   ✓ packages/launcher/tests/agent-tuning.test.ts (3 tests)
   ✓ packages/launcher/tests/action-catalog.test.ts (6 tests)
   ✓ packages/launcher/tests/prompt-builder.test.ts (4 tests)
   ✓ packages/launcher/tests/adapter-routing-snapshot.test.ts (7 tests)
   ✓ packages/launcher/tests/launcher.test.ts (11 tests)
   ✓ packages/launcher/tests/adapter-routing.test.ts (8 tests)

   Test Files  6 passed (6)
        Tests  39 passed (39)
   ```

   PRE-EXISTING baseline (agent-tuning 3, action-catalog 5→6 NEW test added,
   prompt-builder 4, adapter-routing-snapshot 5→7 NEW 2 tests added,
   launcher 11, adapter-routing 6→8 NEW 2 tests added). All NEW tests pass; all
   PRE-EXISTING tests still pass unmodified in behavior (only the 2 test-file
   edits described above touch assertions, and only where the routing table
   itself changed).
5. `npx vitest run apps/cli/tests/app-launcher.test.ts`:

   ```
   ✓ apps/cli/tests/app-launcher.test.ts (13 tests)

   Test Files  1 passed (1)
        Tests  13 passed (13)
   ```

   PRE-EXISTING baseline (13 tests), unchanged, all green — no edits were needed
   in this file.
6. Extra safety net beyond the brief's required commands: `npm test` (full
   workspace) — **285 test files / 2871 tests, all passed**, confirming no
   regression anywhere else in the monorepo from the shared-file edits
   (`action-catalog.ts`, `adapter-routing.ts`, `types.ts`, `prompt-builder.ts`,
   `app-launcher.ts`, `workspace-process-surfaces.ts`).

No PRE-EXISTING or NEW failures anywhere.

## Result

Closed. Both P0 and P1 landed.

- P0: `axiom-phase-reviewer` skill (`axiom.spec/target-axiom-skills/axiom-phase-reviewer.md`)
  + agent (`axiom.spec/target-axiom-agents/axiom-phase-reviewer.md`) authored,
  catalogued in `skills-catalog.yaml`/`agents-catalog.yaml` with verified sha256
  `bundleHash` values, and materialized as a new mandatory process surface for
  the `sdd` role (alongside `axiom-sdd-orchestrator`) in
  `workspace-process-surfaces.ts`. It encodes the 3-lens (spec/plan/code)
  read-only review discipline — loop-until-dry sweep (2 consecutive dry sweeps,
  4-sweep ceiling), a `REV-NNN` findings ledger, an explicit `VEREDICTO: OK|KO`,
  and a scoped re-review — condensed from the bootstrap
  `.claude/agents/axiom-review.md`.
- P1: review is now exposed as 3 launcher actions (`review-spec`/`review-plan`/
  `review-code`) that reuse the existing `buildReviewPrompt` engine via a new
  `promptIntent: 'review'` branch in `buildPrompt`, routed through
  `adapter-routing.ts` to the dedicated `axiom-phase-reviewer` skill/CLI target
  (never to a state-transition command), and marked non-executable
  (preview-only) in the `axiom app` launcher server — the review surface never
  mutates.
- All target guard suites are green, plus a full-workspace `npm test` run
  (285 files / 2871 tests) confirms no regression.

## General spec integration

Integrado por el pase final de la orquestación (autopilot, 2026-07-15):
- `01_Requisitos_Funcionales.md` → **RF-AXM-035**.
- `04_Flujos_SDD_y_Ciclo_de_Vida.md` → sección "Ciclo con gates de calidad instaladas" (revisión spec/plan/código).
- `05_Interfaces_Operativas.md` → sección "Superficies SDD instaladas + revisión en el launcher" (familia `review`).
- `06_Integraciones_y_Capacidades.md` → catálogo ampliado (gate de revisión).
- `07_Gobierno_y_Seguridad.md` → nota "Revisión por fase de solo lectura".
- `08_Glosario.md` → término "`axiom-phase-reviewer` (gate de revisión por fase)".

Archivado en `Axiom.Spec/specs/increments/_archive/`.
