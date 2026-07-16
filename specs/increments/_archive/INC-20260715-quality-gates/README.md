# Increment: quality-gates

Status: closed
Date: 2026-07-15

## Goal

Raise Axiom's installed QA + security disciplines to the level of a
role-specialized system:

- **P0** — add `axiom-qa-validator` as a new skill+agent+surface: a
  spec-driven test-plan discipline that derives a documentary test plan
  directly from acceptance criteria, with 1:N criterion-to-test-case
  traceability, marking any uncovered criterion explicitly.
- **P1** — give the existing `axiom-security-reviewer` agent a real
  installed body (risk-family checklist + severity scale), since today it
  is a ~48-line stub with no concrete checklist or severity model.

## Context

`Axiom/apps/cli/src/commands/workspace-process-surfaces.ts` currently
materializes `axiom-sdd-orchestrator` + `axiom-phase-reviewer` for the
`sdd` role. `axiom.config/agents-catalog.yaml` already has an
`axiom-security-reviewer` entry (`role: product-review`) whose source
(`axiom.spec/target-axiom-agents/axiom-security-reviewer.md`) is a thin
stub (inputs/outputs/skills/commands/workflows/notes, no checklist, no
severity model, no explicit read-only/defensive-only guardrails). There is
no QA/test-plan discipline installed anywhere in the catalogs today —
`axiom-tester` (agent) runs/interprets available validation, but nothing
derives a documentary test plan from acceptance criteria with explicit
traceability.

## Scope

- New skill source
  `Axiom/axiom.spec/target-axiom-skills/axiom-qa-validator.md` + catalog
  entry in `skills-catalog.yaml` (correct sha256 `bundleHash`).
- New agent source
  `Axiom/axiom.spec/target-axiom-agents/axiom-qa-validator.md` + catalog
  entry in `agents-catalog.yaml` (`role: product-review`, correct sha256
  `bundleHash`).
- New `axiom-qa-validator` entry in `SURFACES`
  (`workspace-process-surfaces.ts`), added to `surfaceIdsForRole('sdd')`
  (alongside `axiom-sdd-orchestrator`, `axiom-phase-reviewer`).
- Enriched `axiom.spec/target-axiom-agents/axiom-security-reviewer.md`
  (existing agent, id/role/name unchanged): real read-only/defensive-only
  security-review discipline (risk-family checklist, severity scale,
  findings format, explicit optional/non-blocking statement), and its
  `bundleHash` recomputed/updated in `agents-catalog.yaml`.
- Updated pre-existing guard suites that hard-code catalog counts/
  membership or the `surfaceIdsForRole('sdd')` invariant.

## Non-goals

- No new npm dependencies.
- No stack/tool/adapter/ADO specifics in either the qa-validator or the
  security-reviewer content (generic, adapter-agnostic).
- No executable/automated test scaffolding — `axiom-qa-validator` produces
  a documentary test plan, not code or a test runner.
- No exploit execution, no code execution, no secrets/PII reproduced in
  clear in the security-reviewer discipline.
- No enterprise lifecycle constructs.
- No change to any other existing skill/agent besides
  `axiom-security-reviewer`'s source + hash.

## Acceptance criteria

- [x] `axiom-qa-validator` skill + agent + surface installed for the `sdd`
      role with correct sha256 `bundleHash` values in both catalogs.
- [x] `axiom-qa-validator` discipline: 1:N criterion-to-test-case
      traceability table, marks `⚠️ SIN COBERTURA` for any criterion
      without a case, never invents scenarios beyond the criteria, never
      marks the increment validated with uncovered criteria, distinct from
      executable tests, references `axiom-functional-checklist-coverage`
      (`CF-xx`) and `axiom-structured-doubts`.
- [x] `axiom-security-reviewer` agent source enriched with a real body:
      read-only/defensive-only guardrails (secret obfuscation, no PII, no
      exploits/code execution), a 10-family risk checklist, a
      CRITICAL/HIGH/MEDIUM/LOW severity scale with rationale, a findings
      output format, and an explicit optional/non-blocking statement;
      catalog id/role/name unchanged, `bundleHash` updated.
- [x] Both new sources and the enriched source are generic/adapter-agnostic
      (no stack/tool/ADO specifics).
- [x] All guard suites green: `npm run build`;
      `packages/skills/tests/catalog.test.ts`;
      `packages/agents/tests/catalog.test.ts`;
      `packages/doctor/tests/skills.test.ts`;
      `packages/doctor/tests/agents.test.ts`;
      `apps/cli/tests/workspace-process-surfaces.test.ts`.
- [x] No unrelated files modified; no git commit/add/push performed.

## Open questions

none — resolved by orchestrator brief.

## Assumptions

- `axiom-qa-validator` is parameterized like `axiom-phase-reviewer`
  (`{specRepo}`, `{projectId}` placeholders) rather than tied to a single
  code repo/role, since it reads acceptance criteria from the spec MCP
  broker and is not scoped to one functional role's code.
- The doctor smoke tests and the real-catalog integration tests
  (`packages/skills/tests/catalog.test.ts`,
  `packages/agents/tests/catalog.test.ts`,
  `packages/doctor/tests/skills.test.ts`,
  `packages/doctor/tests/agents.test.ts`) hard-code the exact real-catalog
  count/membership and therefore needed updating in lockstep with the new
  `axiom-qa-validator` catalog entries (17→18 skills, 13→14 agents).
- `axiom-security-reviewer`'s catalog entry (id, name, role,
  responsibility, inputs/outputs/relatedSkills/relatedCommands/
  relatedWorkflows, source path) stays byte-identical except for
  `bundleHash`, per the brief ("only the hash entry changes").

## Implementation notes

### P0 — `axiom-qa-validator`

- `axiom.spec/target-axiom-skills/axiom-qa-validator.md`: mirrors the
  `axiom-functional-checklist-coverage.md` / `axiom-phase-reviewer.md` shape
  (`Qué hace` / `Cuándo se usa` / `Cómo lo hace` (ancla, derivación 1:N,
  tabla de trazabilidad, regla de cierre, distinción con tests ejecutables,
  ambigüedad) / `Reglas absolutas` / `Herramientas MCP que invoca` /
  `Parámetros de materialización` / `Relación con otras piezas` /
  `Estado`). References `axiom-functional-checklist-coverage` (`CF-xx`
  anchoring) and `axiom-structured-doubts` (stop-and-ask on ambiguity) by
  id. Never invokes `sdd.transitionApply`; never writes executable test
  code.
- `axiom.spec/target-axiom-agents/axiom-qa-validator.md`: thin contract
  mirroring `axiom-phase-reviewer.md` (agent) shape; `role: product-review`.
- `workspace-process-surfaces.ts`: added `QA_VALIDATOR_BODY` (condensed
  version of the skill content, same `{specRepo}`/`{projectId}` placeholder
  convention as `axiom-phase-reviewer`) and an `'axiom-qa-validator'` entry
  in `SURFACES` (`agentRole: 'product-review'`, `relatedCommand: 'axiom
  doctor'`, `placement: 'mandatory'`). `surfaceIdsForRole('sdd')` now
  returns `['axiom-sdd-orchestrator', 'axiom-phase-reviewer',
  'axiom-qa-validator']`. Updated the top-of-file role-mapping comment to
  match.
- Catalog entries added to `skills-catalog.yaml` (`axiom-qa-validator`,
  `bundleHash: sha256:d10bfaa8fea0368a6037f92039d0cfd6991cf578bdd1dcdadb103dc803c7bfec`)
  and `agents-catalog.yaml` (`axiom-qa-validator`, `role: product-review`,
  `bundleHash: sha256:5972d23ced6e7aca4e82ec1f85b032f02b082593bb1d8433c4eeda4538dfde49`).

### P1 — `axiom-security-reviewer` enrichment

- Replaced the ~48-line stub body of
  `axiom.spec/target-axiom-agents/axiom-security-reviewer.md` with a
  self-sufficient discipline (mirroring the rich embedded-contract style of
  `axiom-reviewer.md`): `Rol` / `Responsabilidad` / `Guardrails (read-only,
  defensive-only)` (no exploits, no code execution, secret obfuscation e.g.
  `AKIA****XXXX`, no PII) / `Checklist de familias de riesgo` (10 families:
  authn/authz, injection, XSS/CSRF, crypto, secrets management,
  deserialization, SSRF, input validation, security headers, CI/CD supply
  chain) / `Escala de severidad` (CRITICAL/HIGH/MEDIUM/LOW with rationale
  per tier) / `Formato de salida de hallazgos` (`SEC-NNN` table) /
  `Carácter opcional y no bloqueante` (explicit statement it never
  determines `closed`/`pending` by itself) / `Inputs` / `Outputs` / `Skills
  relacionadas` / `Comandos relacionados` / `Workflows relacionados` /
  `Notas`. Fully generic/adapter-agnostic — no .NET/Angular/ADO/stack
  specifics.
- Catalog entry's id/role/name/responsibility/inputs/outputs/
  relatedSkills/relatedCommands/relatedWorkflows/source in
  `agents-catalog.yaml` left byte-identical; only `bundleHash` updated to
  `sha256:b63bb29c415dcd4d596e223bd983407481d6aa118009a0e0e2354ff493b84c55`.

### Guard suites updated (catalog count/membership + surface invariant)

- `packages/skills/tests/catalog.test.ts`: 17→18 entries/ids (comment +
  assertions).
- `packages/agents/tests/catalog.test.ts`: 13→14 entries/ids/roles (comment
  + assertions; `product-review` role count 4→5).
- `packages/doctor/tests/skills.test.ts`: `17/17`→`18/18` regex +
  description.
- `packages/doctor/tests/agents.test.ts`: `13/13`→`14/14` regex +
  description.
- `apps/cli/tests/workspace-process-surfaces.test.ts`:
  `surfaceIdsForRole('sdd')` expectation extended to the 3-id array.
- Grepped the repo for any other hard-coded catalog count/membership or
  `surfaceIdsForRole('sdd')` assertions beyond these 5 files; none found.

## Validation

Ran from `Axiom/` (repo root):

1. `npm run build` (`tsc -b`) — clean, no output.
2. Recomputed the 3 bundleHash values with the node one-liner from the
   brief and diffed them against both catalogs — exact match for all
   three (qa-validator skill, qa-validator agent, security-reviewer agent):

   ```
   axiom.spec/target-axiom-skills/axiom-qa-validator.md -> sha256:d10bfaa8fea0368a6037f92039d0cfd6991cf578bdd1dcdadb103dc803c7bfec
   axiom.spec/target-axiom-agents/axiom-qa-validator.md -> sha256:5972d23ced6e7aca4e82ec1f85b032f02b082593bb1d8433c4eeda4538dfde49
   axiom.spec/target-axiom-agents/axiom-security-reviewer.md -> sha256:b63bb29c415dcd4d596e223bd983407481d6aa118009a0e0e2354ff493b84c55
   ```

3. `npx vitest run packages/skills/tests/catalog.test.ts packages/agents/tests/catalog.test.ts packages/doctor/tests/skills.test.ts packages/doctor/tests/agents.test.ts apps/cli/tests/workspace-process-surfaces.test.ts`:

   ```
   ✓ packages/agents/tests/catalog.test.ts (13 tests) 93ms
   ✓ packages/skills/tests/catalog.test.ts (11 tests) 80ms
   ✓ packages/doctor/tests/agents.test.ts (9 tests) 516ms
   ✓ packages/doctor/tests/skills.test.ts (9 tests) 539ms
   ✓ apps/cli/tests/workspace-process-surfaces.test.ts (5 tests) 2126ms

   Test Files  5 passed (5)
        Tests  47 passed (47)
   ```

   PRE-EXISTING baseline test counts (13+11+9+9+5 = 47) preserved exactly —
   only the real-catalog count/membership/surface-id assertions inside them
   were updated, no test removed, added, or weakened.
4. Extra safety net beyond the brief's required commands: `npm test` (full
   workspace):

   ```
   Test Files  285 passed (285)
        Tests  2871 passed (2871)
   ```

   Matches the pre-increment baseline exactly (285 files / 2871 tests),
   confirming no regression anywhere else in the monorepo.

No PRE-EXISTING or NEW failures anywhere.

## Result

Closed. Both P0 and P1 landed.

- P0: `axiom-qa-validator` skill
  (`axiom.spec/target-axiom-skills/axiom-qa-validator.md`) + agent
  (`axiom.spec/target-axiom-agents/axiom-qa-validator.md`) authored,
  catalogued in `skills-catalog.yaml`/`agents-catalog.yaml` with verified
  sha256 `bundleHash` values, and materialized as a new mandatory process
  surface for the `sdd` role (alongside `axiom-sdd-orchestrator` and
  `axiom-phase-reviewer`) in `workspace-process-surfaces.ts`. It encodes a
  spec-driven QA discipline — a documentary test plan derived directly
  from acceptance criteria, 1:N criterion-to-test-case traceability,
  explicit `⚠️ SIN COBERTURA` marks, never inventing scenarios, never
  marking validation complete with uncovered criteria, and distinct from
  executable tests — anchored to `axiom-functional-checklist-coverage`
  (`CF-xx`) and `axiom-structured-doubts`.
- P1: `axiom-security-reviewer`'s agent source was enriched from a
  ~48-line stub into a real, self-sufficient security-review discipline:
  read-only/defensive-only guardrails (secret obfuscation, no PII, no
  exploits/code execution), a 10-family risk checklist, a
  CRITICAL/HIGH/MEDIUM/LOW severity scale with rationale, a `SEC-NNN`
  findings output format, and an explicit optional/non-blocking gate
  statement. Its catalog id/role/name/responsibility and every other field
  besides `bundleHash` stayed unchanged.
- Both new/enriched sources are generic and adapter/stack-agnostic (no
  .NET/Angular/ADO/CI-tool specifics).
- All target guard suites are green (47/47), plus a full-workspace `npm
  test` run (285 files / 2871 tests) confirms no regression.

## General spec integration

Integrado por el pase final de la orquestación (autopilot, 2026-07-15):
- `01_Requisitos_Funcionales.md` → **RF-AXM-037**.
- `04_Flujos_SDD_y_Ciclo_de_Vida.md` → sección "Ciclo con gates de calidad instaladas" (QA-validation + revisión de seguridad).
- `06_Integraciones_y_Capacidades.md` → catálogo ampliado (QA + seguridad).
- `07_Gobierno_y_Seguridad.md` → notas "Revisión de seguridad defensiva y no bloqueante" + "QA-validation trazable".
- `08_Glosario.md` → término "`axiom-qa-validator`".

Archivado en `Axiom.Spec/specs/increments/_archive/`.
