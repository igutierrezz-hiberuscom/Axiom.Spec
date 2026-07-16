# Increment: reusable-discipline-skills

Status: closed
Date: 2026-07-15

## Goal

Expose Axiom's cross-cutting SDD disciplines as four reusable, adapter-agnostic
catalog skills, so they can be referenced/evolved independently instead of only
living embedded inline in the process-surface bodies. This is the Axiom-native,
generic equivalent of KVP25's shared `.github/skills/shared/*` (structured-doubts,
functional-checklist-coverage, plan-drift-metadata-alignment,
role-technical-close-doc) — without any KVP/stack-specific content.

## Context

The 3 NS-1 process skills (`axiom-spec-author`, `axiom-role-planner`,
`axiom-role-implementer`, from `INC-20260713-ns1-process-skills`) each embed
cross-cutting disciplines inline in their own prose (e.g. "stop and ask before
inventing", functional coverage checklist handling, spec/plan drift
reconciliation, post-close technical documentation). KVP25 solved this by
factoring those disciplines into shared skill files
(`.github/skills/shared/structured-doubts.md`,
`functional-checklist-coverage.md`, `plan-drift-metadata-alignment.md`,
`role-technical-close-doc.md`) referenced by all its role prompts. This
increment ports that shape into the Axiom skills catalog, generalized and
stripped of any .NET/Angular/Cypress/ADO specifics, as four standalone catalog
entries. Wiring them as references from the existing process-surface bodies is
explicitly deferred to a later increment.

## Scope

- Author 4 new skill source files under
  `Axiom/axiom.spec/target-axiom-skills/`:
  - `axiom-structured-doubts.md`
  - `axiom-functional-checklist-coverage.md`
  - `axiom-plan-drift-alignment.md`
  - `axiom-role-close-doc.md`
- Register each as a new entry in `Axiom/axiom.config/skills-catalog.yaml`
  (`version: 0.1.0`, `status: approved`, `securityCheckStatus: ok`, and the
  correct sha256 `bundleHash` of the raw source bytes), appended after the
  existing 10 entries without changing `schemaVersion`.
- Update the two test suites that hard-code the real catalog's skill count
  (`packages/skills/tests/catalog.test.ts`, `packages/doctor/tests/skills.test.ts`)
  from 10 to 14 entries/ids, since they assert an exact count/membership against
  the real repo catalog.

## Non-goals

- Do not modify any existing skill source, existing catalog entries, the
  process-surface bodies (`workspace-process-surfaces.ts`), or
  `agents-catalog.yaml`.
- Do not wire the 4 new skills into `axiom-spec-author` / `axiom-role-planner` /
  `axiom-role-implementer` (or any other surface) yet — later increments will
  reference them.
- Do not change `schemaVersion` in `skills-catalog.yaml`.
- No new npm dependencies.
- No KVP/.NET/Angular/Cypress/ADO-specific content anywhere in the new sources.

## Acceptance criteria

- [x] 4 new skills present in `skills-catalog.yaml`, each with a correct sha256
      `bundleHash` (verified against the raw source bytes, matching how
      `packages/skills/src/materialize.ts` computes it and how doctor TC-010
      re-verifies it).
- [x] Each new source is adapter-agnostic and stack-agnostic (no KVP/.NET/
      Angular/Cypress/ADO specifics); prose refers to Axiom concepts and
      `sdd.*`/`spec.*` MCP brokers generically.
- [x] `npm run build` (tsc -b) stays clean.
- [x] `packages/skills/tests/catalog.test.ts` and
      `packages/doctor/tests/skills.test.ts` are green (updated from 10 to 14
      entries, since both hard-code the real catalog's count/id membership).
- [x] No unrelated files modified.

## Open questions

none — resolved by orchestrator

## Assumptions

- The four skill ids, purposes, and required content sections were fully
  specified by the orchestrator brief; no further discovery was needed.
- Only the two test files that assert an exact count/membership against the
  real `axiom.config/skills-catalog.yaml` needed updating (confirmed by
  grepping the repo for other hard-coded real-catalog assertions — none
  found besides `packages/skills/tests/catalog.test.ts` and
  `packages/doctor/tests/skills.test.ts`).
- `axiom.spec/templates/product-skill-template.md`,
  `packages/skills/src/catalog.ts` and other files that mention "7 skills" in
  comments/docs refer to older platform-meta-skill-only counts and are
  documentation prose, not executable assertions — left untouched per the
  "do not modify unrelated files" guardrail.

## Implementation notes

- Mirrored the shape of `axiom-spec-author.md` (top `# Title`, then
  `## Qué hace`, `## Cuándo se usa`, `## Cómo lo hace`, `## Reglas absolutas`,
  `## Relación con otras piezas`, `## Estado`) for all 4 new sources, in
  Spanish prose with identifiers/ids in code, consistent with the rest of the
  catalog.
- Each source is self-contained content only (no frontmatter — metadata lives
  exclusively in the catalog YAML, per the existing convention).
- `bundleHash` was computed per source with:
  `node -e "const c=require('crypto'),fs=require('fs');console.log('sha256:'+c.createHash('sha256').update(fs.readFileSync(process.argv[1])).digest('hex'))" axiom.spec/target-axiom-skills/<id>.md`
- Catalog entries were appended after the existing 10, under a new comment
  block explaining they are cross-cutting discipline skills from this
  increment; `schemaVersion` untouched.
- Updated `packages/skills/tests/catalog.test.ts` (real-catalog integration
  test: length 10 → 14, id list extended with the 4 new ids, comment updated)
  and `packages/doctor/tests/skills.test.ts` (smoke test description and
  `/10\/10 entries consistentes/` → `/14\/14 entries consistentes/`).

## Validation

Ran from `Axiom/` (repo root):

1. `npm run build` — clean, no output (tsc -b succeeded).
2. `npx vitest run packages/skills/tests/catalog.test.ts packages/doctor/tests/skills.test.ts`
   — both suites green: `packages/skills/tests/catalog.test.ts` (11 tests) and
   `packages/doctor/tests/skills.test.ts` (9 tests), 20/20 passed. This matches
   the pre-increment baseline test counts (11 + 9); no test was removed or
   weakened, only the real-catalog count/membership assertions were updated to
   reflect the intentionally expanded catalog (10 → 14 entries).

No pre-existing or newly-introduced failures.

## Result

Closed. Added 4 new adapter/stack-agnostic discipline skills
(`axiom-structured-doubts`, `axiom-functional-checklist-coverage`,
`axiom-plan-drift-alignment`, `axiom-role-close-doc`) as standalone sources
under `Axiom/axiom.spec/target-axiom-skills/` and registered them in
`Axiom/axiom.config/skills-catalog.yaml` with correct sha256 `bundleHash`
values. Updated the two test suites that hard-code the real catalog's exact
skill count/membership so they reflect the new 14-entry catalog. Build and
both target test suites are green. Wiring these skills into the existing
process-surface bodies is explicitly deferred to a later increment, per scope.

## General spec integration

Integrado por el pase final de la orquestación (autopilot, 2026-07-15):
- `01_Requisitos_Funcionales.md` → **RF-AXM-034**.
- `06_Integraciones_y_Capacidades.md` → sección "Ampliación del catálogo de skills/agents SDD" (las 4 disciplinas transversales).
- `08_Glosario.md` → término "Disciplinas transversales (skills reutilizables)".
- `00_Resumen_Ejecutivo.md` → mención en la tanda de alineación.

Archivado en `Axiom.Spec/specs/increments/_archive/`.
