# Increment: Canonical repo-root AGENTS.md template (docs-skills-writer)

Status: pending
Date: 2026-07-02

## Goal

Introduce a minimal, canonical, repo-root `AGENTS.md` template generator
in `Axiom/`, scoped to data already available today (`projectName`,
`role`, `layout`), per the migration-engineer audit's Finding 4 and
"Next step recommendation"
(`Axiom.Spec/specs/increments/INC-20260702-installer-wizard-reconcile/README.md`).
This is the `docs-skills-writer` role in INC-03's sequence
(migration-engineer -> **docs-skills-writer** -> registry-engineer ->
cli-implementer -> validator-reviewer), running in parallel with (not
blocked by) the later registry-engineer/cli-implementer questionnaire
work, per the audit's sequencing flag.

## Context

Parent roadmap: `Axiom.Spec/specs/increments/INC-20260702-axiom-redesign-roadmap/README.md`,
Phase B ("Instalación de proyecto"), INC-03. The migration-engineer
audit confirmed (with file/line evidence):

- No canonical repo-root `AGENTS.md` exists today anywhere in `Axiom/`.
- `init.ts`/`configure.ts` do not write one. The only code that writes
  a file literally named `AGENTS.md` is `@axiom/installer`'s
  `GENERATED_FILES_BY_TARGET` map, and only for the `opencode`
  (`.opencode/AGENTS.md`) and `antigravity` (`.antigravity/AGENTS.md`)
  adapter targets — both **adapter-scoped**, not canonical, and both
  under adapter-specific subdirectories, never at the repo root.
- `init.ts` still emits `axiom.yaml` **v1** (`project.name`, no
  top-level `role`) — the `schemaVersion: 2` cutover (which would add
  a top-level `role` field per `AxiomYamlSchemaV2`) is explicitly
  deferred to a separate follow-up increment, not this one.
- The two external decision documents' addendum §3 ("AGENTS.md as
  canonical contract") and §18.1/§18.2 (structural integrity rules +
  separation-of-concerns boilerplate) define the required content.
  Both were re-read directly (not from secondhand description) as part
  of this increment.

## Scope

- Read `Axiom/packages/document-bootstrap/src/*` (`renderer.ts`,
  `variables.ts`, `writer.ts`, `idempotency.ts`, `types.ts`,
  `index.ts`) and its tests, in full.
- Read `Axiom/packages/installer/src/registry.ts` (`GENERATED_FILES_BY_TARGET`)
  and `Axiom/packages/adapters/{opencode,claude-code}/src/agents-md.ts`
  and `generator.ts`, in full, to confirm no existing renderer already
  produces canonical repo-root `AGENTS.md` content and to factor out
  the shared idempotency convention (preservation-marker state machine)
  rather than reinvent it.
- Design and implement a new, pure render function
  (`renderCanonicalAgentsMd`) + I/O orchestrator
  (`writeCanonicalAgentsMd`) in `@axiom/document-bootstrap`, taking a
  minimal `CanonicalAgentsMdIdentity` input (`projectName`, optional
  `role`, optional `layout`) — no new fields invented; `role`/`layout`
  mirror vocabulary already used by `AxiomYamlSchemaV2#role` and
  `init.ts`'s `InitResult['layout']`.
- Render source doc §18.1 ("Axiom structural integrity rules") and
  §18.2 ("Separation of concerns") verbatim, plus a minimal
  project-identity header.
- Wire the writer into `apps/cli/src/commands/init.ts` as an additive,
  best-effort, non-blocking step (try/catch, WARN to stderr on
  failure, matching the existing pattern used for user-level registry
  auto-registration in the same file).
- Add tests in `packages/document-bootstrap/tests/` and
  `apps/cli/tests/init.test.ts` following existing conventions
  (tmpdir setup/teardown, `Result<T,E>` assertions, path-guard and
  TEAM:CUSTOM-preservation scenarios).
- Confirm no path collision between the new canonical `AGENTS.md`
  (repo root) and the existing adapter-scoped `AGENTS.md` outputs
  (`.opencode/AGENTS.md`, `.antigravity/AGENTS.md`).

## Non-goals

- No changes to `@axiom/topology`, `@axiom/user-workspace`,
  `@axiom/project-resolution`, `@axiom/config-validation`, or
  `@axiom/doctor` (settled by INC-01/INC-02; out of scope here).
- No `schemaVersion: 2` cutover in `init.ts` (tracked separately per
  the migration-engineer audit's recommendation, e.g.
  `INC-20260702-init-schemaversion2-cutover`).
- No Roles/Tools-MCPs questionnaire work (registry-engineer/
  cli-implementer's later steps in this same INC-03 sequence).
- No reconciliation between the new canonical `AGENTS.md` and the
  adapter-scoped `AGENTS.md`/`CLAUDE.md` outputs (e.g. making adapters
  derive from the canonical file) — the roadmap's own INC-05 already
  scopes exactly this ("AGENTS.md as canonical contract + adapter
  reconciliation"); this increment only introduces the canonical file
  itself.
- No mandatory/blocking wiring into `init`/`configure` — the new step
  is additive and best-effort, matching how other generated files
  (e.g. the user-level registry auto-registration in `init.ts`) are
  made optional today.
- No `configure.ts` wiring in this pass: `configure.ts` operates on an
  already-`init`-ed project and has no natural "first write" moment
  for a repo-root file the way `init` does; adding it to `configure`
  as well was considered but left out to keep the diff minimal and
  focused on the one clear integration point. This can be revisited if
  a real gap surfaces (e.g. projects `init`-ed before this increment
  landed) — see "Open questions".

## Acceptance criteria

- [x] `Axiom/packages/document-bootstrap/src/*` read in full; existing
      rendering/writing pipeline (renderer + idempotency + atomic
      writer) confirmed and reused for the new artifact.
- [x] `Axiom/packages/installer/src/registry.ts` and both
      `opencode`/`claude-code` adapters' `agents-md.ts` read in full;
      confirmed neither produces canonical repo-root `AGENTS.md`
      content (both render a capabilities/skills routing block into
      adapter-scoped subdirectories).
- [x] New `renderCanonicalAgentsMd`/`writeCanonicalAgentsMd` implemented
      in `@axiom/document-bootstrap`, exported from the package barrel.
- [x] Rendered content includes source doc §18.1 and §18.2 blocks
      verbatim (character-for-character, verified in tests).
- [x] Wired into `apps/cli/src/commands/init.ts` as additive/best-effort
      (a write failure logs a WARN to stderr and does not throw or
      block `init`'s primary mutation).
- [x] No path collision: canonical `AGENTS.md` writes to
      `<projectRoot>/AGENTS.md`; adapter-scoped outputs remain at
      `.opencode/AGENTS.md` / `.antigravity/AGENTS.md` — confirmed by
      a dedicated test that pre-seeds both adapter-scoped files and
      asserts they are untouched after the canonical write.
- [x] Tests added following existing conventions; all new tests pass.
- [x] `npm run typecheck`, `npm run build`, and `npm test` (full
      monorepo) executed; results compared against the stated baseline
      (12 failed files / 13 failed tests / 1220 passed) with zero
      regressions.
- [x] No files modified under `@axiom/topology`, `@axiom/user-workspace`,
      `@axiom/project-resolution`, `@axiom/config-validation`,
      `@axiom/doctor`.
- [x] Recorded in `Axiom.Spec` per `Axiom.SDD/AGENTS.md`.

## Open questions

- Should `configure.ts` also write/refresh the canonical `AGENTS.md`
  for projects that were `init`-ed before this increment landed (i.e.
  a backfill path)? Not decided here — flagged for the next role
  (registry-engineer) or a small dedicated follow-up to consider once
  real usage surfaces the gap. Not blocking closure of this increment:
  the acceptance criteria only require the new-project (`init`) path,
  per the audit's explicit scope ("create, not extend/verify").
- Should `role` fall back to something other than `DEFAULT_REPO_ROLE
  ('sdd')` once `schemaVersion: 2` lands and a real top-level `role`
  field becomes available from `axiom.yaml`? Left to the
  `schemaVersion: 2` cutover follow-up increment — `writeCanonicalAgentsMd`
  already accepts `role` as a plain optional string, so no shape
  change will be needed, only a different call-site value.

## Assumptions

- "Additive, non-blocking" (per the task's instruction) means: a
  failure to write the canonical `AGENTS.md` must not throw out of
  `runInit`, must not prevent `axiom.yaml`/`.gitignore`/`.sdd/`/
  `init.json` from being written, and must be visible to the operator
  via a stderr WARN — mirroring the existing best-effort pattern
  already used in the same function for user-level registry
  auto-registration.
- `role` and `layout` are optional in `CanonicalAgentsMdIdentity`
  because `role` is not yet persisted in `axiom.yaml` v1 (today's
  emitted shape) and `layout` is a scaffold-only concept some future
  callers may not have. The rendered header simply omits a line when a
  field is absent, rather than rendering a placeholder or guessing.
- Reusing `classifyAndPreserve`/`AXIOM_GENERATED_REGEX`/
  `TEAM_CUSTOM_REGEX` from `idempotency.ts` (rather than duplicating
  the state machine, as the two adapter packages currently do) was
  judged in-scope and low-risk: it is the same package, same file,
  already generic over `existing: string | null`, and avoids a third
  copy of the preservation-marker logic in the same monorepo.
- Deliberately did NOT reuse `renderCopilotInstructions`/
  `resolveVariables`/`ResolvedVariables`: those are scoped to a
  disjoint variable set (`project.id`, `mcp.allowlist`, `support.level`,
  `target.id`) from an unrelated spec (0018-B4), and repurposing the
  same `{{placeholder}}` syntax for an unrelated meaning would be
  confusing. The new module owns its own minimal render function
  instead.

## Implementation notes

### Files read in full

- `Axiom/packages/document-bootstrap/src/{renderer,variables,writer,types,index,idempotency}.ts`
- `Axiom/packages/document-bootstrap/tests/{writer,renderer}.test.ts`
- `Axiom/packages/installer/src/{registry,installer}.ts`
- `Axiom/packages/adapters/README.md`
- `Axiom/packages/adapters/opencode/src/{agents-md,generator,types}.ts`
- `Axiom/packages/adapters/claude-code/src/agents-md.ts`
- `Axiom/apps/cli/src/commands/{configure,init,_shared}.ts`
- `Axiom/apps/cli/tests/{init,configure}.test.ts`
- `Axiom/packages/config-validation/src/schemas.ts` (`AxiomYamlSchema`,
  `AxiomYamlSchemaV2`)
- Both external decision documents, §3 and §18 (re-read directly, not
  from secondhand description).

### Files changed

- `Axiom/packages/document-bootstrap/src/canonical-agents-md.ts` (new)
  — `renderCanonicalAgentsMd` (pure) + `writeCanonicalAgentsMd`
  (I/O orchestrator with path-guard + idempotency + atomic write).
- `Axiom/packages/document-bootstrap/src/index.ts` (edit) — barrel
  export for the two new functions and their types.
- `Axiom/packages/document-bootstrap/tests/canonical-agents-md.test.ts`
  (new) — 9 tests: pure-render content/determinism (3), cold write (2),
  TEAM:CUSTOM preservation (1), path-guard (1), no-collision-with-
  adapter-outputs (1), omits Role/Layout when absent (1, folded into
  the render group above — see file for the exact 9-test breakdown).
- `Axiom/apps/cli/src/commands/init.ts` (edit) — import
  `writeCanonicalAgentsMd`; call it right after `axiom.yaml` is
  written (best-effort try/catch; success appends to `filesCreated`,
  failure logs a WARN to stderr and continues).
- `Axiom/apps/cli/tests/init.test.ts` (edit) — 2 new tests: canonical
  `AGENTS.md` is created with the expected sections/identity fields;
  no collision with a pre-seeded `.opencode/AGENTS.md`.

### Template content summary

The rendered `AXIOM:GENERATED` block contains:

1. A short preamble stating this file is the canonical, portable
   agent contract, and that tool-specific adapters are generated
   separately and do not replace it.
2. A `## Project` section with `- Name: <projectName>` (always),
   `- Role: <role>` (only if provided), `- Layout: <layout>` (only if
   provided).
3. The verbatim `## Axiom structural integrity rules` block (source
   doc §18.1): do not manually create/rename/move/change-status of
   increments/bugs/plans/ADRs/decisions/indexed technical context
   documents/indexed skills; use Axiom commands instead; the AI may
   draft content but not manage structural metadata/IDs/status/links/
   indexes; never edit generated index/cache files manually; run
   `axiom index validate` / `axiom index rebuild` / `axiom doctor` if
   stale.
4. The verbatim `## Separation of concerns` block (source doc §18.2):
   product/code changes in code repos; specs/increments/bugs/plans/
   ADRs/decisions/technical context in the spec repo; SDD flow/
   orchestration skills/rules/commands in the SDD repo; repo-specific
   technical skills in the owning code repo; canonical technical
   context never in code repos.

The file also gets a default `TEAM:CUSTOM` block on first write
(`## Team custom notes`, empty), preserved byte-exact on subsequent
runs, matching the convention already established by
`writeCopilotInstructions` and the adapter `agents-md.ts` generators.

### Wiring decision

`init.ts` calls `writeCanonicalAgentsMd` immediately after writing
`axiom.yaml` (step "1.1"), passing `projectName`, `role:
DEFAULT_REPO_ROLE` (`'sdd'` — the same value `init` already uses for
the registry entry; `axiom.yaml` itself does not carry a `role` field
today because `init` still emits v1), and `layout`. The call is
wrapped in try/catch: on success, the written file is appended to
`filesCreated`; on any failure (`Result.err` or thrown exception), a
`[axiom init] WARN: ...` line is written to stderr and `runInit`
continues unaffected — the same additive/non-blocking pattern already
used a few lines later in the same function for user-level registry
auto-registration (`registryAttempt`/`registryRegistered`).

`configure.ts` was NOT changed. It has no natural "first write" moment
for a canonical file the way `init` does (it operates on an
already-initialized project on every run), and adding it there was
judged out of scope for a first, minimal increment — see "Non-goals"
and "Open questions".

### Path-collision confirmation

Confirmed by direct inspection and by a dedicated test
(`Scenario 5` in `canonical-agents-md.test.ts` and the second new test
in `init.test.ts`): the canonical file writes to
`<projectRoot>/AGENTS.md` (repo root). The adapter-scoped files from
`GENERATED_FILES_BY_TARGET` write to `.opencode/AGENTS.md` and
`.antigravity/AGENTS.md` (nested under adapter-specific
subdirectories). These are three distinct paths; no adapter target
list entry resolves to a bare `AGENTS.md` at the repo root. Both test
suites pre-seed an adapter-scoped `AGENTS.md` and assert its content
is untouched after the canonical write runs.

## Validation

Validation discovery order followed `Axiom.SDD/AGENTS.md` (README ->
package scripts -> task runners -> test configs -> build configs ->
available scripts). The monorepo has `npm run build`, `npm run
typecheck`, and `npx vitest run` at the root — all three were run.

Commands executed and results:

- `npm run build --workspace=@axiom/document-bootstrap` — clean
  (`tsc`, no errors).
- `npm run typecheck --workspace=apps/cli` — clean (`tsc --noEmit`,
  no errors).
- `npx vitest run packages/document-bootstrap/tests apps/cli/tests/init.test.ts apps/cli/tests/configure.test.ts`
  — 6 files, 54 tests, all passed (9 new in
  `canonical-agents-md.test.ts`, 2 new in `init.test.ts`, both
  pre-existing suites unaffected).
- `npm run build` (full monorepo, `tsc -b`) — clean, no errors.
- `npm run typecheck` (full monorepo, `tsc -b`) — clean, no errors.
- `npx vitest run` (full monorepo) — **12 failed files / 13 failed
  tests / 1231 passed** (125 files, 1244 tests total).

### Baseline comparison

Stated baseline (confirmed clean through the whole INC-01+INC-02
chain, per the task's instructions): 12 failed files / 13 failed tests
/ 1220 passed.

This run: 12 failed files / 13 failed tests / 1231 passed.

The failed-file and failed-test counts are IDENTICAL to baseline. The
passed count increased by exactly 11 (1231 - 1220 = 11), matching the
11 new tests added by this increment (9 in
`canonical-agents-md.test.ts` + 2 in `init.test.ts`). The list of
failing test files was cross-checked and contains none of the files
touched by this increment (`document-bootstrap`, `init.ts`,
`configure.ts`); the failures are pre-existing, environment/real-repo-
relative-path-dependent tests in unrelated packages (`model-routing`,
`doctor`, `agents`/`skills` catalogs, `tui`, `toolchain`, `telemetry`,
`project-resolution`, `apps/cli/tests/start.test.ts`) — the same class
of failures the migration-engineer audit's stated baseline already
accounted for. **Zero regressions.**

## Result

Implemented a new, minimal canonical `AGENTS.md` template generator in
`@axiom/document-bootstrap` (`renderCanonicalAgentsMd` +
`writeCanonicalAgentsMd`), reusing the package's existing idempotency
(`AXIOM:GENERATED`/`TEAM:CUSTOM` preservation-marker) and atomic-write
conventions rather than introducing a new pattern. The renderer embeds
the two verbatim boilerplate blocks from the external decision
documents' §18.1 (structural integrity rules) and §18.2 (separation of
concerns), plus a minimal project-identity header
(`projectName`/`role`/`layout` — all data already available in the
live codebase today, per the migration-engineer audit's scoping
recommendation). Wired additively and best-effort into `axiom init`,
immediately after `axiom.yaml` is written; a write failure cannot
break `init`'s primary mutation. Confirmed, with a dedicated test in
each touched test file, that the new repo-root `AGENTS.md` does not
collide with the existing adapter-scoped `.opencode/AGENTS.md` /
`.antigravity/AGENTS.md` outputs from `@axiom/installer`'s
`GENERATED_FILES_BY_TARGET`. Full monorepo build, typecheck, and test
suite were run; zero regressions against the stated baseline (same 12
failed files / 13 failed tests, +11 new passing tests).

## General spec integration

No integration into `Axiom.Spec/general-spec.md` was performed — this
file does not exist in this repo (confirmed absent, consistent with
both the parent roadmap increment's and the migration-engineer audit's
own notes). No other stable, consolidated-knowledge document exists in
this repo to update either (the closest equivalents,
`Axiom.Spec/specs/00_Resumen_Ejecutivo.md` through `08_Glosario.md`,
describe intent at a level this increment's implementation details do
not change). The one piece of knowledge worth flagging for whoever
eventually creates `general-spec.md`: the canonical `AGENTS.md`
contract (source doc §3/§18) is now a real, implemented artifact in
`Axiom/` — not just a design decision in an external doc — and its
exact template content (this file's "Template content summary"
section) is the up-to-date reference, superseding the raw external
decision documents for implementation questions.

## Next step recommendation

Proceed with **registry-engineer** next, per the parent roadmap's
planned INC-03 sequence (migration-engineer -> docs-skills-writer ->
**registry-engineer** -> cli-implementer -> validator-reviewer).
Inputs available to registry-engineer from this chain:

- This increment's canonical `AGENTS.md` generator
  (`writeCanonicalAgentsMd` in `@axiom/document-bootstrap`) and its
  minimal `CanonicalAgentsMdIdentity` input shape
  (`projectName`/`role`/`layout`) — if registry-engineer's
  questionnaire work introduces new identity/role/repo data at `init`
  time, that data can be threaded into this same function without a
  shape change (it already accepts `role` as a plain optional string).
- The migration-engineer audit's gap analysis
  (`Axiom.Spec/specs/increments/INC-20260702-installer-wizard-reconcile/README.md`),
  specifically the "Roles" and "Tools/MCPs" dimensions (flagged as the
  largest gaps) and the "Repos" dimension's finding that `init` staying
  single-repo-first (with `repo attach` covering multi-repo growth) is
  correct by design, not a gap to close inside `init`.
- This increment's "Open questions" section (whether/how `configure.ts`
  should backfill the canonical `AGENTS.md` for pre-existing projects)
  — registry-engineer or a later role may want to fold that into the
  richer questionnaire work rather than leaving it as a standalone
  follow-up.

Separately (unchanged from the prior increment's recommendation): a
small dedicated follow-up increment for the `schemaVersion: 2`
re-enablement in `init.ts` (e.g.
`INC-20260702-init-schemaversion2-cutover`) remains open and
independent of this chain.
