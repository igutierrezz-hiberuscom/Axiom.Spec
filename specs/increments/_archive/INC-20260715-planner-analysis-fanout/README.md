# Increment: planner-analysis-fanout

Status: closed
Date: 2026-07-15

## Goal

Give Axiom's planner (`axiom-role-planner`) an OPTIONAL, complexity-gated
**scope-analysis fan-out** so complex multi-layer increments get a deeper
per-dimension analysis before role plans are written — the generic,
adapter-agnostic equivalent of a role-specialized planner's per-domain
analysis. Express it as an analysis DISCIPLINE (depth chosen by complexity
signals), never as a requirement to spawn named subagents, so it works on any
adapter, including ones that cannot spawn subagents.

## Context

`axiom-role-planner` already had a "Fase 1 — Análisis de alcance" that
classifies which roles have real work before deciding how many role plans to
emit, but it had no notion of *depth* of analysis, and no structured
per-dimension (backend/frontend/qa/transversal) breakdown for genuinely
complex specs. The skill exists in two representations that must stay
consistent: the full catalog source
(`axiom.spec/target-axiom-skills/axiom-role-planner.md`, hashed in
`axiom.config/skills-catalog.yaml`) and the condensed process-surface mirror
(`ROLE_PLANNER_BODY` in
`Axiom/apps/cli/src/commands/workspace-process-surfaces.ts`, unhashed). This
increment ports the "per-domain analysis" shape a role-specialized planner
would give a complex change into a generic, portable discipline that any
adapter can execute either inline (single agent working through the
dimensions sequentially) or, only where supported, by delegation — without
ever hard-requiring subagent-spawning machinery or naming a concrete
subagent.

## Scope

- Extend the existing "Fase 1 — Análisis de alcance" in
  `axiom.spec/target-axiom-skills/axiom-role-planner.md` with:
  - A "Señales de complejidad → profundidad del análisis" subsection that
    picks analysis DEPTH (not a subagent count) from complexity signals:
    *alcance simple* (single layer/role, no cross-layer contracts, no
    permissions/multi-tenancy, no breaking changes, no wide regression) plans
    directly with minimal inline analysis; *alcance con complejidad real*
    (multi-layer, or ≥2 strong signals among permissions/multi-tenancy,
    breaking change, complex reactive/UI behavior, wide regression,
    integrations) triggers a per-dimension analysis before writing role
    plans.
  - An "Análisis por dimensión" subsection (backend, frontend, qa,
    transversal — cross-layer contracts, permissions/claims, multi-tenant
    isolation, integrations/jobs, docs to update), each dimension feeding its
    matching role's plan, framed as a portable sequence of analysis steps
    (never a forced subagent spawn), with a block-if-insufficient contract
    routed through `axiom-structured-doubts` back to `axiom-spec-author`.
  - A new "Disciplinas que referencia" section citing
    `axiom-structured-doubts`, `axiom-plan-drift-alignment`, and
    `axiom-functional-checklist-coverage` by id, and light amendments to
    `Reglas absolutas` / `Relación con otras piezas` reinforcing the
    "never require subagent spawning" and "choose analysis depth" rules.
  - Recomputed sha256 `bundleHash` in `axiom.config/skills-catalog.yaml` for
    the `axiom-role-planner` entry.
- Mirror the same shape (condensed) into `ROLE_PLANNER_BODY` in
  `Axiom/apps/cli/src/commands/workspace-process-surfaces.ts` (step 2 of "Cómo
  lo hace" expanded with the complexity-gated depth choice and per-dimension
  analysis; a new "Disciplinas que referencia" section; `Reglas absolutas`
  amended in lockstep).

## Non-goals

- No new catalog skill or agent entry — this edits the existing
  `axiom-role-planner` skill only, no id/count changes.
- No adapter-specific subagent machinery, no named subagent, no hard
  dependency on subagent-spawning support.
- No change to any other process surface, skill, or catalog entry.
- No new npm dependencies.
- No stack/adapter-specific content (no .NET/Angular/ADO specifics).

## Acceptance criteria

- [x] `axiom-role-planner.md` (source) gains the optional, complexity-gated
      per-dimension scope-analysis phase, framed as a portable analysis
      discipline (never a forced subagent spawn), with a
      block-if-insufficient contract referencing `axiom-structured-doubts`.
- [x] `ROLE_PLANNER_BODY` (process-surface condensed mirror) reflects the same
      new phase in condensed form, consistent with the source.
- [x] Existing planner rules stay intact: one plan per role with real work,
      minimal `targetRepos` + `allowedWriteScope`, `CF-xx` coverage without id
      dilution, drift handling (`none`/`review`/`replan`/`reopen`).
- [x] Content is adapter/stack-agnostic (no forced subagent spawning, no named
      subagent, no .NET/Angular/ADO specifics); prose in Spanish.
- [x] `axiom-role-planner`'s sha256 `bundleHash` in `skills-catalog.yaml` is
      recomputed and matches the updated source bytes exactly.
- [x] `npm run build` stays clean.
- [x] `packages/skills/tests/catalog.test.ts`,
      `packages/doctor/tests/skills.test.ts`, and
      `apps/cli/tests/workspace-process-surfaces.test.ts` stay green with no
      id/count changes (18 skills), only the `axiom-role-planner` content/hash
      changed.
- [x] No unrelated files modified.

## Open questions

none — resolved by the orchestrator brief (self-contained; substrate and
exact target files/tests were verified and specified up front).

## Assumptions

- The brief's explicit guidance that no skill/count changes are expected was
  confirmed: only `axiom-role-planner.md` and its `bundleHash` entry, plus the
  unhashed `ROLE_PLANNER_BODY` mirror, needed edits — no test hard-asserts the
  planner's body prose (grepped both
  `apps/cli/tests/workspace-process-surfaces.test.ts` and
  `packages/skills/tests/catalog.test.ts` /
  `packages/doctor/tests/skills.test.ts` for planner-body text assertions;
  found none beyond id/count/hash checks), so the target test suites required
  no edits at all.
- Inserting the new subsections inside the existing "Fase 1" (rather than
  renumbering Fases 2-4) keeps the diff minimal and avoids touching anchors
  other content might reference.
- Placing the new "Disciplinas que referencia" section between "Reglas
  absolutas" and "Parámetros de materialización" in the source mirrors the
  precedent set by `axiom-phase-reviewer.md`'s "Disciplinas que referencia"
  section without reordering the planner's pre-existing sections.

## Implementation notes

- `Axiom/axiom.spec/target-axiom-skills/axiom-role-planner.md`:
  - "Fase 1 — Análisis de alcance" gained two subsections: "Señales de
    complejidad → profundidad del análisis (opcional, gated)" (simple vs.
    complex-real scope, picking depth not a subagent count) and "Análisis por
    dimensión (solo cuando hay complejidad real)" (backend/frontend/qa/
    transversal, each feeding its role's plan; explicit "adapter-agnóstica y
    portable... nunca exijas spawnear subagentes" framing; block-if-
    insufficient via `axiom-structured-doubts`).
  - `Reglas absolutas`: added "exigir el spawneo de subagentes o nombrar un
    subagente concreto..." to `Prohibido`, and "elegir la profundidad del
    análisis de alcance según las señales de complejidad..." to `Obligatorio`.
  - New `## Disciplinas que referencia` section (between `Reglas absolutas`
    and `Parámetros de materialización`) citing all three reusable discipline
    skills by id with their exact role in the planner's flow.
  - `Relación con otras piezas` lightly extended to also name the three
    discipline skills alongside the pre-existing collaborators.
  - `bundleHash` recomputed:
    `node -e "const c=require('crypto'),fs=require('fs');console.log('sha256:'+c.createHash('sha256').update(fs.readFileSync(process.argv[1])).digest('hex'))" axiom.spec/target-axiom-skills/axiom-role-planner.md`
    → `sha256:1fdaab97479119649c73fce14c725a58027a3a098c77b4b165f49d694dcbe20f`,
    written into `axiom.config/skills-catalog.yaml`'s `axiom-role-planner`
    entry (only that one line changed).
- `Axiom/apps/cli/src/commands/workspace-process-surfaces.ts`
  (`ROLE_PLANNER_BODY`): step 2 ("Análisis de alcance") of "Cómo lo hace"
  expanded in condensed form with the same simple-vs-complex-real depth
  choice and per-dimension breakdown; steps 4-5 lightly touched to name
  `axiom-functional-checklist-coverage` / `axiom-plan-drift-alignment` /
  `axiom-structured-doubts` where they already implicitly applied; `Reglas
  absolutas` amended in lockstep with the source; new condensed `##
  Disciplinas que referencia` section added before `## MCP que invoca`. No
  other process surface or body touched.

## Validation

Ran from `Axiom/` (repo root):

1. `npm run build` (`tsc -b`) — clean, no output.
2. Recomputed `axiom-role-planner.md`'s sha256 with the node one-liner above
   and diffed it against `skills-catalog.yaml`'s entry — exact match:
   `sha256:1fdaab97479119649c73fce14c725a58027a3a098c77b4b165f49d694dcbe20f`.
3. `npx vitest run packages/skills/tests/catalog.test.ts packages/doctor/tests/skills.test.ts apps/cli/tests/workspace-process-surfaces.test.ts`:

   ```
   ✓ packages/doctor/tests/skills.test.ts (9 tests)
   ✓ apps/cli/tests/workspace-process-surfaces.test.ts (5 tests)
   ✓ packages/skills/tests/catalog.test.ts (11 tests)

   Test Files  3 passed (3)
        Tests  25 passed (25)
   ```

   PRE-EXISTING baseline (11 + 9 + 5 = 25) preserved exactly; no test added,
   removed, or weakened — the real-catalog integration test still asserts 18
   skills / the same id list (`axiom-role-planner` unchanged as an id), only
   its `bundleHash` value now differs from the pre-increment hash on disk (it
   reads the live catalog file, which was updated in lockstep with the
   source). No NEW or PRE-EXISTING failures.

## Result

Closed. `axiom-role-planner` gained an optional, complexity-gated
per-dimension scope-analysis phase in both representations:

- The catalog source (`axiom.spec/target-axiom-skills/axiom-role-planner.md`)
  now picks analysis DEPTH from complexity signals (simple single-layer scope
  vs. real multi-layer/≥2-strong-signal complexity) and, only in the complex
  case, walks a backend/frontend/qa/transversal per-dimension analysis before
  writing role plans, each dimension feeding its matching role's plan, with a
  block-if-insufficient contract via `axiom-structured-doubts`. The framing is
  explicitly adapter-agnostic and portable: the same agent can walk the
  dimensions itself, or delegate only where the adapter supports it — never a
  forced subagent spawn, never a named subagent.
- The condensed process-surface mirror (`ROLE_PLANNER_BODY` in
  `workspace-process-surfaces.ts`) reflects the same shape.
- All pre-existing planner rules (one plan per role with real work, minimal
  `targetRepos`/`allowedWriteScope`, `CF-xx` coverage without dilution, drift
  classification) are intact and now explicitly cross-reference
  `axiom-structured-doubts`, `axiom-plan-drift-alignment`, and
  `axiom-functional-checklist-coverage` by id.
- The source's `bundleHash` was recomputed and updated in
  `skills-catalog.yaml`; the 3 target guard suites (25 tests total) are green
  with no id/count changes; `npm run build` is clean; no unrelated file was
  modified.

## General spec integration

Integrado por el pase final de la orquestación (autopilot, 2026-07-15):
- `01_Requisitos_Funcionales.md` → **RF-AXM-038**.
- `04_Flujos_SDD_y_Ciclo_de_Vida.md` → sección "Ciclo con gates de calidad instaladas" (análisis de alcance opcional en el planner).
- `06_Integraciones_y_Capacidades.md` → catálogo ampliado (planner).
- `08_Glosario.md` → término "Análisis de alcance del planner (fan-out opcional)".

Archivado en `Axiom.Spec/specs/increments/_archive/`.
