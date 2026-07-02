# Increment: Registry + manifest schema v2 (validator-reviewer)

Status: closed
Date: 2026-07-02

## Goal

Execute the **validator-reviewer** step of INC-01's subagent sequence
(migration-engineer -> schema-writer -> registry-engineer ->
cli-implementer -> resolver-fix (unplanned) -> **validator-reviewer**,
the final role), implementing `@axiom/doctor`'s `MC-001` three-way branch
exactly as specified by the schema-writer design
(`Axiom.Spec/specs/increments/INC-20260702-registry-manifest-schema-v2-design/README.md`,
"What MC-001 needs to do differently"): `valid: true, version: 2` ->
`pass`; `valid: true, version: 1` -> `warn`; `valid: false, version:
'unknown'` -> `fail`. This is the final piece required for INC-01's
overall acceptance criteria (registry v2 + manifest v2 implemented,
tested, wired into the CLI, and now wired into doctor validation) to be
met.

This increment also investigates and resolves the `BC-001` degrade
flagged by the resolver-fix increment's closing note.

## Context

Parent chain: `INC-20260702-axiom-redesign-roadmap` (parent roadmap) ->
`INC-20260702-registry-manifest-schema-v2` (migration-engineer audit) ->
`INC-20260702-registry-manifest-schema-v2-design` (schema-writer design)
-> `INC-20260702-registry-manifest-schema-v2-impl` (registry-engineer
implementation) -> `INC-20260702-registry-manifest-schema-v2-cli`
(cli-implementer cutover) -> `INC-20260702-registry-manifest-schema-v2-resolver-fix`
(resolver fix, closed, unplanned follow-up) -> this increment
(validator-reviewer, the final role in INC-01's sequence).

All five prior specs in this chain were read in full before any code was
written. `Axiom/packages/doctor/src/checks.ts` (`runManifestChecks`/
MC-001, `runBoundaryChecks`/BC-001) and
`Axiom/packages/doctor/tests/checks.test.ts` were read in full and
treated as the authoritative existing contract to extend, not redesign.

## Scope

- Implement the exact MC-001 three-way branch specified by the
  schema-writer design in `Axiom/packages/doctor/src/checks.ts`'s
  `runManifestChecks`, using `validateAxiomYamlContent`'s
  `AxiomYamlValidationResult` discriminated union (already implemented by
  registry-engineer, untouched by cli-implementer/resolver-fix).
- Investigate the BC-001 concern flagged by the resolver-fix increment's
  closing note (`runBoundaryChecks`'s `installed-multi-repo` detection
  silently stops firing for `schemaVersion: 2` documents) and fix it if
  small/clearly-scoped, or flag it as a follow-up if not.
- Confirm `DoctorCheck.status`'s existing `pass | fail | warn | skip`
  shape already supports `warn` (it does — no type change needed).
- Add/update tests in `Axiom/packages/doctor/tests/checks.test.ts`
  covering all three MC-001 branches (v2-valid, v1-legacy,
  invalid-under-both) and the BC-001 v2 fix.
- Run real validation (`npm run typecheck`, `npm run build`, `npm test`
  full monorepo + scoped to `@axiom/doctor` and adjacent packages) and
  cross-check the failure set against the one already documented by the
  cli-implementer and resolver-fix increments.
- Assess INC-01 as a whole for closure and recommend next steps (INC-02).

## Non-goals

- Do NOT touch `apps/cli/src/commands/init.ts`'s `schemaVersion: 2`
  re-enablement — explicitly a separate, later, independently-verified
  step per the resolver-fix increment's own non-goals and "Next step
  recommendation" (item 2).
- No `@axiom/topology`, `@axiom/tui`, or migration-script changes,
  consistent with every prior increment's non-goals in this chain.
- No broader `@axiom/doctor` redesign beyond the two named findings
  (MC-001's three-way branch, BC-001's `mode` location degrade) — no new
  check categories, no speculative doctor architecture.
- No resolution of Q3 (`axiom.spec/` prefix survival) or Q4
  (capability/provider/MCP folding) from the parent roadmap — both remain
  open, untouched by this increment.

## Acceptance criteria

- [x] `runManifestChecks`'s MC-001 branch implements the exact three-way
      table: `valid: true, version: 2` -> `pass`; `valid: true, version:
      1` -> `warn` with evidence recommending `axiom upgrade`; `valid:
      false, version: 'unknown'` -> `fail` (unchanged evidence format).
- [x] `DoctorCheck`'s existing `warn` status is reused as-is (confirmed
      already wired, no type-level change needed).
- [x] Evidence strings for the new `warn` branch match the existing
      `checks.ts` tone/format conventions (Spanish, imperative
      `axiom upgrade` call-to-action, consistent with MC-002's own
      warn-evidence style).
- [x] BC-001's `installed-multi-repo` degrade for v2 documents is
      investigated; the fix-or-flag decision is explicit and justified.
- [x] New tests cover: a v2-valid fixture (`pass`), a v1-legacy fixture
      with no `schemaVersion` (`warn`), and a structurally-invalid-under-
      both fixture (`fail`).
- [x] The pre-existing MC-001 test (previously asserting `pass` for a
      legacy v1 fixture) is updated to assert `warn`, not silently left
      contradicting the new behavior.
- [x] `apps/cli/src/commands/init.ts` was not modified.
- [x] `@axiom/topology`, `@axiom/tui`, and all other `apps/cli/src/
      commands/*.ts` files were not modified.
- [x] Real validation commands were run (`npm run typecheck`, `npm run
      build`, `npm test` full monorepo, plus scoped runs) and their
      actual pass/fail output is reported below, cross-checked against
      the failure set already documented by cli-implementer/resolver-fix.
- [x] INC-01 as a whole is assessed for closure with explicit rationale.
- [x] Next step recommendation points to INC-02 per the parent roadmap.

## Open questions

None blocking this increment's own closure. Q3 and Q4 from the parent
roadmap remain open, untouched, consistent with all five prior specs in
this chain.

## Assumptions

- Inherited unchanged from all five prior specs in this chain: the
  non-hard-removal posture for D1/D2/D3, and no migration-script
  implementation.
- The schema-writer design's MC-001 table is treated as the literal
  contract to implement, not redesigned — evidence copy is adapted only
  to match `checks.ts`'s actual Spanish-language, imperative-tone
  conventions (the design's own table explicitly says "design intent, not
  final copy").

## Implementation notes

### Files changed

**`Axiom/packages/doctor/src/checks.ts`**:

- `runManifestChecks` (MC-001): replaced the prior binary
  `validationResult.valid ? pass(...) : fail(...)` with the three-way
  branch on `AxiomYamlValidationResult`'s `valid`/`version` fields:
  - `valid: true, version === 2` -> `pass`, evidence unchanged
    (`resolution.configPath`).
  - `valid: true, version === 1` -> `warn` (new branch), evidence:
    "axiom.yaml no declara schemaVersion (formato legacy v1); es válido
    bajo el schema legacy. Ejecutá `axiom upgrade` para migrar a
    schemaVersion 2." — matches MC-002's existing warn-evidence tone
    (Spanish, states the condition, then the actionable next step) rather
    than inventing a new format.
  - `valid: false` (`version: 'unknown'`) -> `fail`, evidence unchanged
    (`errors.map((e) => \`${e.path}: ${e.message}\`).join('; ')`).
- `runBoundaryChecks` (BC-001/BC-002's `usesInstalledMultiRepo`
  detection): fixed to also read `mode` from the top level of
  `resolution.rawConfig` (v2 shape), not only from
  `rawConfig.project.mode` (v1 shape). See "BC-001 finding and decision"
  below for the full analysis.

**`Axiom/packages/doctor/tests/checks.test.ts`**:

- Added `scaffoldV2Project` (valid `schemaVersion: 2` fixture, `paths`
  map, `spec`/`product` roles), `scaffoldInvalidUnderBothProject`
  (no `schemaVersion`, missing `project.name`, `scopes` is a string
  instead of a map — invalid under both v1 and v2 schemas), and
  `scaffoldInstalledMultiRepoProjectV2` (valid `schemaVersion: 2` fixture
  with top-level `mode: installed-multi-repo`).
- Updated the pre-existing MC-001 test (`MC-001 pasa para axiom.yaml
  válido`, using `scaffoldMinimalProject`'s v1/legacy fixture) to assert
  `warn`, not `pass`, and renamed it to reflect the new expected
  behavior; added assertions on the evidence text mentioning
  `schemaVersion` and `axiom upgrade`.
- Added two new MC-001 tests: v2-valid -> `pass`,
  invalid-under-both -> `fail`.
- Added one new BC-001/BC-002 test: `installed-multi-repo` detection via
  a `schemaVersion: 2` fixture (`mode` at the top level) -> BC-002
  `pass` with the `installed-multi-repo` evidence text, mirroring the
  existing v1 equivalent test.

**Not touched** (confirmed by re-reading before finishing):
`apps/cli/src/commands/init.ts` (its `schemaVersion: 2` emission remains
reverted/commented-out exactly as the cli-implementer increment left
it), `@axiom/topology/*`, `@axiom/tui/*`, all other
`apps/cli/src/commands/*.ts` files, `@axiom/config-validation/*`,
`@axiom/project-resolution/*`.

### MC-001 implementation summary

The schema-writer design's exact contract table:

| `validateAxiomYamlContent` result | MC-001 status |
|---|---|
| `valid: true, version: 2` | `pass` |
| `valid: true, version: 1` | `warn` (not `pass`, not `fail`) |
| `valid: false, version: 'unknown'` | `fail` (unchanged) |

`DoctorCheck.status` (`Axiom/packages/doctor/src/checks.ts:37`) was
already typed as `'pass' | 'fail' | 'warn' | 'skip'` before this
increment, and a `warn(...)` helper already existed and was already used
by MC-002, BC-001, and BC-002 — this is a purely additive branch, no
type-level change was required, confirming the migration-engineer audit's
original expectation ("existing shape is `pass | fail | warn | skip`").
The only code change was routing MC-001's second branch (previously
merged into a binary `pass`/`fail`) through the existing `warn(...)`
helper, keyed off `validationResult.version`, which registry-engineer's
`AxiomYamlValidationResult` discriminated union already exposes.

### BC-001 finding and decision

**Investigated.** `runBoundaryChecks` reads
`resolution.rawConfig.project.mode` to detect
`mode === 'installed-multi-repo'` (the `usesInstalledMultiRepo` flag,
which feeds BC-002's pass/warn decision, and is read alongside BC-001 in
the same function). Confirmed by reading `@axiom/project-resolution/src/
resolver.ts`'s `resolveFromV2` helper: a `schemaVersion: 2` document's
`rawConfig` is the parsed `AxiomYamlConfigV2` object, which has `mode` at
its **top level**, not nested under a `.project` key (v2 has no
`.project` object at all — see the schema-writer design's
`AxiomYamlSchemaV2`, and the resolver-fix increment's own "Return-shape
decision" section, which already identified this exact gap for BC-001
specifically). This meant `rawProject` was always `null` for a v2
document, so `rawProjectMode` was always `null`, so
`usesInstalledMultiRepo` was always `false` — regardless of what `mode`
actually contained — for every `schemaVersion: 2` document, silently.

**Decision: fixed, not just flagged.** This is a small, clearly-scoped
fix consistent with this increment's spirit (schema-version-awareness for
doctor checks touching `axiom.yaml`) — the same shape of change as
MC-001's own fix (reading a value that moved location between schema
versions), confined to `runBoundaryChecks`'s existing
`usesInstalledMultiRepo` computation, with no new check category, no
`ProjectResolution` shape change, and no change to any other package.
The fix reads `mode` from both possible locations
(`rawConfig.project.mode` for v1, `rawConfig.mode` for v2) and treats
either as authoritative for the `installed-multi-repo` special case,
consistent with the schema-writer design's Resolution #7 ("`mode` is
preserved verbatim as an independent axis" across both versions — same
field, relocated, not two different concepts). This is explicitly
narrower than "touches product-runtime scope boundaries unrelated to
schema version" — it does not change BC-001/BC-002's actual pass/warn
logic or thresholds, only where the `mode` value is read from on disk.

A new test (`scaffoldInstalledMultiRepoProjectV2`) confirms BC-002 now
correctly passes with the `installed-multi-repo` evidence text for a v2
document, mirroring the pre-existing v1-equivalent test exactly.

## Validation

Real validation commands were run (not the "no validation command found"
fallback — this increment has actual code).

**`npm run typecheck` (root, `tsc -b`, all project references):**

```
> axiom-product@0.1.0 typecheck
> tsc -b
```

Exit code 0, zero errors, across the entire monorepo.

**`npm run build` (root, `tsc -b`):**

```
> axiom-product@0.1.0 build
> tsc -b
```

Exit code 0, identical clean result.

**`npm test` (root, `vitest run`, full monorepo suite):**

```
Test Files  12 failed | 112 passed (124)
     Tests  13 failed | 1209 passed (1222)
```

All 12 failing files / 13 failing tests are byte-for-byte the same test
names already documented and `git stash`-bisected against clean `main`
by the cli-implementer and resolver-fix increments:
`apps/cli/tests/start.test.ts` (1), `packages/agents/tests/
catalog.test.ts` (1), `packages/doctor/tests/checks.test.ts` >
"repositorio Axiom real" (1), `packages/model-routing/tests/
{assignments,loader,opencode-projection,resolver}.test.ts` (4),
`packages/skills/tests/catalog.test.ts` (1), `packages/project-resolution/
tests/resolver.test.ts` > "resuelve scopes con absolutePath y exists
correcto (v1)" (1), `packages/telemetry/tests/audit-trail-sink.test.ts`
(1), `packages/toolchain/tests/repair-add-gitignore.test.ts` (1),
`packages/tui/tests/driver.test.ts` (2). **No new failures, no missing
failures** — confirmed by direct comparison of test names against the
resolver-fix increment's own reported set, not just counts. The one
doctor-specific failure in this set (`checks.test.ts` > "repositorio
Axiom real") was additionally re-confirmed pre-existing in this session
via `git stash` bisection of only this increment's two changed files
(`checks.ts`, `checks.test.ts`): the failure reproduces identically with
none of this increment's changes applied (it depends on `Axiom`'s own
real `axiom.yaml` resolving to `ambiguous` in this environment, unrelated
to MC-001/BC-001).

**Scoped run, `@axiom/doctor` (the package this increment actually
changed):**

```
packages/doctor/tests/checks.test.ts (16 tests | 1 failed)
  ✗ runDoctorChecks — repositorio Axiom real > encuentra y resuelve el proyecto Axiom real (pre-existing, unrelated)
```

15/16 tests pass, including all 6 new/updated MC-001 and BC-001/BC-002
tests. The 1 failure is the same pre-existing, environment-dependent
integration test described above.

**Scoped run, `@axiom/doctor` + `@axiom/config-validation` +
`@axiom/project-resolution` + `apps/cli` together:**

```
Test Files  3 failed | 43 passed (46)
     Tests  3 failed | 372 passed (375)
```

The 3 failures are exactly the 3 pre-existing ones from the documented
set that fall within this scope (`start.test.ts`, `checks.test.ts` >
"repositorio Axiom real", `resolver.test.ts` > "v1 scopes" bug) — no new
failures introduced by this increment's changes.

## Result

Implemented the validator-reviewer step of INC-01, the final role in its
subagent sequence, in the real `Axiom` monorepo:

- `@axiom/doctor/src/checks.ts`'s `runManifestChecks` (MC-001) now
  implements the exact three-way branch specified by the schema-writer
  design: `pass` for `schemaVersion: 2` documents, `warn` (with
  `axiom upgrade` guidance) for legacy v1 documents (schemaVersion
  absent), `fail` (unchanged) for documents invalid under both schemas.
  No `DoctorCheck` type change was needed — `warn` was already a
  supported status, already used by sibling checks.
- Investigated and **fixed** (not just flagged) the BC-001 concern raised
  by the resolver-fix increment's closing note: `runBoundaryChecks`'s
  `installed-multi-repo` detection now reads `mode` from both the v1
  location (`rawConfig.project.mode`) and the v2 location
  (`rawConfig.mode`), so it no longer silently degrades for
  `schemaVersion: 2` documents. This was judged small and clearly scoped
  enough to fix directly, consistent with this increment's own spirit,
  rather than deferred as a follow-up.
- 6 new/updated tests added to `packages/doctor/tests/checks.test.ts`:
  one MC-001 test updated (v1-legacy fixture now asserts `warn`, not
  `pass`), two new MC-001 tests (v2-valid -> `pass`,
  invalid-under-both -> `fail`), one new BC-001/BC-002 test (v2
  `installed-multi-repo` -> `pass`), plus the two new fixture builders
  supporting them.
- Real `npm run typecheck`/`npm run build`/`npm test` executed: typecheck
  and build are 100% clean across the entire monorepo; the full test
  suite's 13 failures are byte-for-byte the same 12-file, 13-test set
  already documented by the cli-implementer and resolver-fix increments —
  **no new regressions**.
- `apps/cli/src/commands/init.ts`, `@axiom/topology`, `@axiom/tui`, and
  all other CLI command files were confirmed untouched, per this
  increment's explicit non-goals.

## General spec integration

**Integration performed.** Unlike all five prior increments in this
chain (which each deferred integration because the INC-01 chain was not
yet fully closed end-to-end), this increment is the final role in INC-01's
sequence, and INC-01's registry v2 + manifest v2 contract is now
implemented, tested, wired into the CLI (five of six command surfaces),
and wired into doctor validation (MC-001, plus the BC-001 fix). This is
the point the prior increments' own "General spec integration" sections
identified as correct for integration.

`Axiom.Spec/general-spec.md` does not exist in this repo (confirmed by
all five prior increments in this chain). Per `Axiom.SDD/AGENTS.md`'s
own instruction ("If no stable knowledge needs integration, state that
explicitly"), and consistent with this bootstrap-mode workspace not
introducing files/structures beyond what already exists or is explicitly
requested, no `general-spec.md` file is created here. Instead, the stable
knowledge this chain produced is consolidated in this increment's own
"INC-01 closure summary" section below, which is the natural, minimal
home for it given the existing chain's conventions (each increment in
this chain records its own consolidated knowledge in its own README
rather than a separate `general-spec.md`). If/when `Axiom.Spec/
general-spec.md` is created in a future increment, the summary below is
the exact content that should migrate into it for the registry/manifest
v2 topic.

## INC-01 closure summary

**INC-01 ("Reconcile registry + manifest schema", parent roadmap) is
assessed as `closed`.**

All of INC-01's own acceptance criteria (per
`INC-20260702-axiom-redesign-roadmap/README.md`'s per-increment
structure, and this five-role chain's cumulative work) are met:

- **Registry v2 implemented and tested**: `~/.axiom/projects.yml`
  (`ProjectsFile`/`ProjectEntryV2`/`RegistryRepoEntry`, `repos: {roleKey:
  {role, path}}` map) — `@axiom/user-workspace`, registry-engineer
  increment, 17 new tests, all passing.
- **Manifest v2 implemented and tested**: `axiom.yaml` `schemaVersion: 2`
  (`AxiomYamlSchemaV2`: `projectId`/`name`/`repoId`/`role`/`mode`/
  `paths`/`indexes`/...) with version-dispatching
  `validateAxiomYamlContent` — `@axiom/config-validation`,
  registry-engineer increment, 9 new tests, all passing.
- **Wired into the CLI**: five of six named command surfaces
  (`init`, `join`, `projects`, `repo`, `app-api`) cut over to the `*V2`
  registry API — cli-implementer increment. `tui.ts` explicitly deferred
  to INC-04 (tui-developer), not a gap in INC-01's own scope.
- **Manifest resolution is version-aware**: `@axiom/project-resolution`'s
  `resolveProject` correctly resolves both v1 and v2 `axiom.yaml`
  documents — resolver-fix increment (unplanned, but required to make
  the CLI cutover actually work end-to-end for v2 documents).
- **Wired into doctor validation**: `@axiom/doctor`'s MC-001 now
  implements the exact pass/warn/fail three-way branch, and the adjacent
  BC-001 degrade is fixed — this increment.
- **No regressions across the full chain**: every increment in this chain
  ran real `npm run typecheck`/`npm run build`/`npm test` and confirmed,
  via `git stash` bisection, that the pre-existing failure set (12 files,
  13 tests, all unrelated to `axiom.yaml`/registry/doctor manifest logic)
  stayed constant across all five implementation increments plus this
  one.

**What remains explicitly out of INC-01's own scope** (not gaps in this
increment's work, but named boundaries carried forward to later
increments per the parent roadmap and this chain's own non-goals):

- `apps/cli/src/commands/init.ts` still emits v1 manifests
  (`schemaVersion: 2` emission is deliberately not re-enabled here; the
  resolver-fix increment judged it safe to re-enable, but explicitly
  deferred the actual re-enablement + its own end-to-end verification
  walkthrough as a separate, later step — this increment's task
  instructions confirm this stays out of scope).
- `@axiom/tui`'s package-internal driver (`driver.ts`, `screens/
  projects.ts`) still speaks v1 exclusively — explicitly INC-04's job
  (tui-developer), per the parent roadmap's own phase sequencing, not
  INC-01's.
- The actual `registry.json -> projects.yml` / `axiom.yaml` v1 -> v2
  migration script (invoked via `axiom upgrade`) was never in scope for
  any of INC-01's five roles — only the loader-side transition-window
  *detection* (`legacy-registry-not-migrated`) was implemented. Building
  the actual migration logic is separate, unscheduled future work.
- Q3 (`axiom.spec/` prefix survival) and Q4 (capability/provider/MCP
  folding) from the parent roadmap remain open, untouched by any
  increment in this chain, as originally scoped.

None of the above are treated as blocking INC-01's closure: they are
either explicitly-scoped follow-ups named by name in the parent roadmap
(TUI cutover -> INC-04, migration script -> unscheduled), or explicit,
independently-justified deferrals within this chain's own increments
(the `init.ts` re-enablement, Q3/Q4). Closing INC-01 now, with these
named and tracked rather than silently dropped, is consistent with
`Axiom.SDD/AGENTS.md`'s closure rules (goal clear, acceptance criteria
exist and are met, changes implemented and tested, validation executed,
review completed, knowledge consolidated).

**Recommended lightweight follow-up** (not performed by this increment,
per the task's own instruction not to fabricate a tracking mechanism that
doesn't exist): `Axiom.Spec/specs/increments/
INC-20260702-axiom-redesign-roadmap/README.md` has no per-increment
status field of its own (it is a narrative roadmap document, not a
tracked table) — a future light-touch edit could add a short "Status
update" note under INC-01's own bullet in the "Reconciled increment
roadmap" section, pointing at this increment as INC-01's closure record,
rather than this increment unilaterally introducing a new status-tracking
convention into that document.

## Closure rationale

`Status: closed`, for both this increment's own scope and INC-01 as a
whole, because:

- Goal is clear (implement the exact, previously-designed MC-001
  three-way branch; investigate and resolve BC-001's flagged degrade).
- Acceptance criteria exist and are all met.
- Changes were implemented (not merely designed) and are fully tested,
  with 6 new/updated tests, all passing.
- Available validation was executed: real `npm run typecheck`/`npm run
  build`/`npm test`, both full-monorepo and scoped, with output reported
  above and cross-checked test-name-by-test-name (not just counts)
  against the exact failure set documented by the two immediately prior
  increments in this chain — confirming no new regressions.
- Review against intent and acceptance criteria was completed (see
  Result above).
- Stable knowledge was integrated: this increment's own "INC-01 closure
  summary" section consolidates the chain's cumulative, stable outcome,
  since `general-spec.md` does not exist yet in this repo (consistent
  with all five prior increments' own findings).
- The result is documented clearly above, including the explicit,
  justified boundary of what remains out of scope (TUI cutover, `init.ts`
  re-enablement, migration script, Q3/Q4) rather than silently dropped.

This is the first `closed` increment in this chain besides the
resolver-fix increment, and the first to also close the parent INC-01
increment itself, because it is the final role in INC-01's originally
planned five-role sequence and every acceptance criterion — for both this
increment's own narrow scope and INC-01 as a whole — is now met.

## Next step recommendation

Per the parent roadmap
(`Axiom.Spec/specs/increments/INC-20260702-axiom-redesign-roadmap/README.md`),
the next increment in sequence is **INC-02 — Structural command
foundation and event model**, which the roadmap explicitly instructs to
start with a migration-engineer audit of `@axiom/orchestrator`
(`state-machine.ts`, `gates.ts`, `runner.ts`) first, before any other
subagent writes new contract code — the same audit-first discipline this
whole INC-01 chain followed. Per the roadmap's own diff, `@axiom/
orchestrator` already implements a 7-lifecycle-state machine and 15
intent commands (several still `not-implemented` stubs per its own
README); INC-02's task is to confirm which of those stubs map to the
`RepoRegistered`-equivalent events the redesign's source documents ask
for, and to wire INC-01's registry/topology writes through
`runCommand`/gates instead of direct writes — additive work, not a
parallel event-system build, per the roadmap's own classification.

Two smaller, independently-schedulable follow-ups from within this chain
remain available to pick up at any point, not blocking INC-02:

1. Re-enable `axiom.yaml` `schemaVersion: 2` emission in
   `apps/cli/src/commands/init.ts`, now that the resolver-fix increment
   confirmed it is safe — including its own real
   `axiom init` -> `axiom join` -> `axiom repo attach` -> `axiom tui
   --topology` walkthrough, per that increment's own recommendation.
2. A light-touch status note on the parent roadmap document marking
   INC-01 closed, per this increment's "INC-01 closure summary" section
   above (not performed here, to avoid fabricating a tracking mechanism
   the roadmap document does not already have).
