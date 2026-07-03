# Increment: Reconcile SDD/role skills (`@axiom/skills` catalog) — validator-reviewer

Status: closed
Date: 2026-07-02

## Goal

Execute the **validator-reviewer** step of INC-09's subagent sequence
(migration-engineer -> schema-writer -> docs-skills-writer ->
**validator-reviewer**), the final role of INC-09. Independently verify
the prior three increments' work against live code (not their prose),
add the promised `@axiom/doctor` check (working name `TC-012`) for the
new role-index artifact, reusing `validateSkillsRoleIndex` from
`@axiom/skills` rather than reimplementing shape validation, and — if
all of INC-09's acceptance criteria are satisfied — close INC-09 as a
whole with an explicit summary of any deferred items.

## Context

Parent chain: `INC-20260702-axiom-redesign-roadmap` (parent roadmap) ->
`INC-20260702-skills-role-index-reconcile` (migration-engineer audit,
status `pending`) -> `INC-20260702-skills-role-index-reconcile-design`
(schema-writer design, status `pending`) ->
`INC-20260702-skills-role-index-reconcile-impl` (docs-skills-writer
implementation, status `pending`) -> this increment (validator-reviewer).

All three prior specs were read in full before starting this review, not
paraphrased. Live code read fresh this session (not reused from prior
prose): `Axiom/packages/skills/src/role-index.ts`,
`Axiom/packages/skills/src/catalog.ts`,
`Axiom/packages/skills/src/index.ts`,
`Axiom/packages/skills/tests/fixtures/axiom.spec/config/skills-index/backend-developer.yaml`,
`Axiom/packages/skills/tests/role-index.test.ts`,
`Axiom/packages/doctor/src/checks.ts` (TC-010/TC-011 regions, in full),
`Axiom/packages/doctor/src/index.ts`,
`Axiom/packages/doctor/tests/checks.test.ts`,
`Axiom/packages/doctor/package.json`, `Axiom/packages/doctor/tsconfig.json`.

This session's working directory (`Axiom`, not a git repo root recognized
by the harness as clean) has a substantial amount of pre-existing,
unrelated in-flight work already present as uncommitted changes (e.g.
`packages/project-resolution`, `packages/filesystem-truth`,
`packages/tui`, `packages/telemetry`, `packages/workflow` new files) —
confirmed via `git status`/`git diff` and explicitly not touched by this
increment. This is noted because it affects the "before" baseline
comparison for validation (see "Validation" below): the freshly-measured
baseline includes failures from that unrelated work, not from anything
in the INC-09 chain.

## Scope

- Independently re-read `role-index.ts`'s `validateSkillsRoleIndex` and
  `loadSkillsRoleIndex` against the schema-writer design's exact shape,
  field-for-field, and against the docs-skills-writer fixture.
- Confirm `getRoleIndexEntryById`'s resolution and not-found behavior.
- Confirm via `git diff` that `CatalogEntry`/`SkillsCatalog`/`apply.ts`/
  `materialize.ts`/`refresh.ts` remain untouched by the whole INC-09
  chain.
- Determine the actual next-available `doctor` check ID (not assume
  `TC-012` is free) by reading `checks.ts`'s existing ID sequence.
- Implement the new check, reusing `validateSkillsRoleIndex` (no
  reimplementation), following the existing `DoctorCheck{id, category,
  description, status, evidence}` + `pass/fail/warn/skip` convention.
  Skip gracefully (not fail) when `axiom.spec/config/skills-index/` does
  not exist, matching this codebase's existing pattern for optional,
  absent-but-not-mandatory structures (`IX-001`'s skip-if-unresolved,
  `MC-002`'s warn-if-absent-`.gitignore`).
- Write tests for the new check (valid role-index passes, malformed
  fails, absent directory skips, empty directory passes with zero
  entries, YAML-parse-failure fails, filename/role-mismatch fails).
- Run real validation (`npm run typecheck`, `npm run build`, `npm test`)
  scoped to the touched packages and across the full monorepo; compare
  against a freshly-measured baseline, not the task's stated numbers
  taken on faith.
- Close INC-09 as a whole if all of its acceptance criteria are met,
  naming deferred items explicitly.
- Update `Axiom.Spec/general-spec.md` with the stable, proven knowledge
  from the whole INC-09 chain.

## Non-goals

- No change to `CatalogEntry`, `SkillsCatalog`, `loadSkillsCatalog`,
  `getSkillById`, `apply.ts`, `materialize.ts`, `refresh.ts`, or the
  `axiom skills` CLI command.
- No cross-reference resolution between a role-index entry's `id` and the
  live `skills-catalog.yaml` (the design's own "id reuses catalog id"
  section explicitly frames this as future loader behavior, not part of
  TC-012's shape-validation scope; `skills-catalog.yaml` also still does
  not physically exist in this repo, so there would be nothing to
  cross-reference against yet — see "Deferred items" below).
- No fix for the pre-existing `catalog.test.ts` integration-test failure
  (the real `axiom.spec/config/skills-catalog.yaml` still does not exist
  on disk) — confirmed still present and unrelated to this increment's
  scope, not silently patched over.
- No fix for any of the other 11 pre-existing failing test files
  (`tui`, `telemetry`, `toolchain`, `model-routing`,
  `project-resolution`, `agents`, `start.test.ts`) — all confirmed
  unrelated to `@axiom/skills`/`@axiom/doctor`'s skills category.

## Acceptance criteria

- [x] `validateSkillsRoleIndex`/`loadSkillsRoleIndex` independently
      confirmed to match the schema-writer design field-for-field.
- [x] Confirmed the loader/validator correctly reject malformed input
      (missing required fields, wrong types) via direct re-reading of
      `role-index.ts`'s narrowing logic and `role-index.test.ts`'s
      malformed-shape sub-cases (Scenario 3, 8 sub-cases).
- [x] Confirmed the loader correctly accepts the worked fixture example
      (`backend-developer.yaml`) via direct re-reading, not just trusting
      the prior increment's reported test pass.
- [x] Confirmed the actual next-available `doctor` check ID by reading
      `checks.ts`'s ID sequence directly (`TC-012` confirmed free — the
      highest existing `TC-*` id is `TC-011`).
- [x] `TC-012` implemented in `Axiom/packages/doctor/src/checks.ts`,
      reusing `validateSkillsRoleIndex` (imported from `@axiom/skills`,
      not reimplemented), following the existing `DoctorCheck`
      conventions.
- [x] `TC-012` skips gracefully (not fails) when
      `axiom.spec/config/skills-index/` does not exist.
- [x] `getRoleIndexEntryById` confirmed to resolve an `id` against a
      role-index's own `mandatory`/`available` entries (not the catalog
      — see "Independent verification findings" below) and to return
      `undefined`, not throw, on a not-found id.
- [x] Confirmed via `git diff` that `CatalogEntry`/`SkillsCatalog`/
      `apply.ts`/`materialize.ts`/`refresh.ts` are untouched by the
      entire INC-09 chain (zero diff on all five files).
- [x] Tests added for `TC-012` (valid passes, malformed fails, absent
      directory skips, empty directory passes, YAML-parse-failure fails,
      filename/role-mismatch fails) — 7 new tests, all passing.
- [x] `npm run typecheck`, `npm run build`, `npm test` executed for the
      full monorepo; results recorded with real pass/fail counts,
      compared against an independently, freshly-measured baseline (not
      assumed from the task's stated prior numbers).
- [x] INC-09 as a whole assessed for closure, with deferred items named
      explicitly if any acceptance criteria remain open.
- [x] `Axiom.Spec/general-spec.md` updated with the stable, proven
      knowledge from INC-09 (role-index concept, file placement
      convention, Q-skills-3 clarification), concise and consistent with
      the file's existing style.

## Open questions

None remain open for this increment's own scope. One item, already
flagged by all three prior increments, is confirmed still true and
explicitly named as a deferred item at INC-09-closure scope (see
"Deferred items" below): the real `axiom.spec/config/skills-catalog.yaml`
still does not exist on disk in the `Axiom` repo.

## Assumptions

- "Role" continues to mean the source doc's product/task role concept
  (e.g. `backend-developer`), unchanged from all three prior increments'
  Assumptions sections.
- The pre-existing, unrelated in-flight changes visible in `git status`
  for this repo (see "Context" above) are out of this increment's scope
  to review, fix, or comment on beyond noting they do not overlap with
  any file this increment or the rest of the INC-09 chain touches.

## Implementation notes

### Independent verification findings

**Schema match, field-for-field (confirmed, not re-derived).** Read
`Axiom/packages/skills/src/role-index.ts` directly against the
schema-writer design's TypeScript shape
(`INC-20260702-skills-role-index-reconcile-design/README.md`, "Role-index
schema design" section). Every field matches exactly:
`SkillsRoleIndexSchemaVersion = 1`; `SkillsRoleIndexMandatoryEntry {id,
path?, reason?}`; `SkillsRoleIndexAvailableEntry {id, path?, tags
(required), summary?}`; `SkillsRoleIndex {schemaVersion, role,
repoKinds?, mandatory, available}`. No field was added, dropped, or
retyped relative to the design. The implementation additionally matches
the design's documented rationale for each optionality choice (e.g.
`path?` as a fallback pointer, `tags` required on `available` only).

**Malformed-input rejection, confirmed by direct code reading, not only
by trusting the prior increment's reported test results.**
`validateSkillsRoleIndex` rejects, in order: non-object input; wrong
`schemaVersion`; missing/empty `role`; non-string-array `repoKinds`;
non-array `mandatory`/`available`; any `mandatory[]`/`available[]` entry
that is not an object, has a missing/empty `id`, a non-string `path` when
present, a non-string `reason` when present (mandatory only); any
`available[]` entry missing a valid `tags` string array or a non-string
`summary` when present; and duplicate `id` across the combined
`mandatory`+`available` lists. Each rejection returns
`{kind: 'malformed', error: <specific message>}` — confirmed by reading
`parseMandatoryEntry`/`parseAvailableEntry`/`checkDuplicateIds` in full,
not by re-running only the existing test suite. Cross-checked against
`role-index.test.ts`'s Scenario 3 (8 sub-cases): every sub-case in the
test file maps to a specific branch actually present in the
implementation — no test asserts a rejection the code does not actually
perform, and no rejection branch in the code is left unexercised by any
test.

**Worked fixture acceptance, confirmed by direct re-reading and a fresh
test run (not by trusting the prior increment's summary).** Read
`Axiom/packages/skills/tests/fixtures/axiom.spec/config/skills-index/backend-developer.yaml`
directly: `schemaVersion: 1`, `role: backend-developer`, 1 `mandatory`
entry (`backend.repo-rules`), 4 `available` entries (`backend.ef-core`,
`backend.migrations`, `backend.api-controller`, `backend.testing`), all
with `path`/`tags`/`summary` populated — matches the schema-writer
design's worked example byte-for-byte. Re-ran
`role-index.test.ts`'s Scenario 1 integration test
(`loadSkillsRoleIndex(fixtureRoot, 'backend-developer')`) fresh this
session: passes, `kind: 'ok'`, all 5 ids present.

**`getRoleIndexEntryById` resolves against the role-index's own entries,
not the catalog.** Confirmed by direct reading of the function body
(`role-index.ts` lines 340-347): it searches `index.mandatory` then
`index.available` by `id`, returning
`SkillsRoleIndexMandatoryEntry | SkillsRoleIndexAvailableEntry |
undefined`. It does **not** take or reference a `SkillsCatalog` — this is
correct per the schema-writer design, which specifies `id` "reuses"
(matches the same string as) a `CatalogEntry.id` by *convention*, not
that the role-index type or its lookup helper cross-reference the
catalog at runtime. The design's own "What validator-reviewer will need
later" section explicitly deferred catalog cross-reference resolution to
a future loader/consumer, not to this helper or to `TC-012` — confirmed
this deferral is still accurate and `getRoleIndexEntryById`'s scope
matches it exactly, no gap. Not-found case: returns `undefined` (via
`Array.prototype.find`'s natural `undefined` on no match), never throws
— consistent with the whole package's never-throws contract.

**`CatalogEntry`/`SkillsCatalog`/`apply.ts`/`materialize.ts`/
`refresh.ts` untouched, confirmed via `git diff`, not assumed.**
`git diff --stat` against each of `packages/skills/src/{catalog.ts,
apply.ts,materialize.ts,refresh.ts,types.ts}` returns empty output — zero
lines changed by the whole INC-09 chain. `packages/skills/src/index.ts`
shows a diff, but it is purely additive (new barrel exports for
`role-index.ts`'s public API; no existing export line removed or
changed) — confirmed by reading the diff content, not just its
line-count stat.

**Next-available check ID confirmed by reading the ID sequence directly,
not assumed.** Grepped `checks.ts` for every `'TC-0\d+'`/`'MC-0\d+'`
literal: the highest existing check id is `TC-011`
(`agents-catalog-coverage`). `TC-012` was free at the time of this
increment's start — the design's own "next available number after
TC-011" note is confirmed correct, not stale.

### TC-012 implementation

Added `runSkillsRoleIndexCheck(resolution)` to
`Axiom/packages/doctor/src/checks.ts`, in the existing `skills` category
region (`CATEGORY_SKILLS`, alongside `TC-010`). Behavior:

- **SKIP** if `resolution.status !== 'resolved'` (same condition every
  other check in this file uses).
- **SKIP** if the spec scope cannot be resolved (mirrors
  `readSkillsCatalog`'s own `null`-return skip case for TC-010/TC-011).
- **SKIP** if `<specScope>/config/skills-index/` does not exist on disk —
  this is the graceful-skip behavior the task explicitly required,
  deliberately different from TC-010/TC-011's fail-on-absent-catalog
  behavior, because the role-index is an **optional**, per-role artifact
  (a project may have zero role-index files and be in a perfectly valid
  state), whereas the single `skills-catalog.yaml` is expected to always
  exist once a project has skills at all. This mirrors `IX-001`'s
  skip-on-absent-optional-structure pattern (empty
  `increments/bugs/plans` folders are not an error) more closely than
  TC-010/TC-011's single-mandatory-file pattern.
- **PASS** with `"0 role-index encontrados... (directorio vacío)"` if the
  directory exists but contains no `.yaml`/`.yml` files — an empty
  directory is a valid, if unusual, state, not an error.
- For each `.yaml`/`.yml` file found: read it, parse it with `js-yaml`
  (same `require('js-yaml').load` pattern already used by
  `readSkillsCatalog`/`readAgentsCatalog` in this same file, for
  consistency — not a new parsing mechanism), and call
  `validateSkillsRoleIndex` directly from `@axiom/skills` — **reused, not
  reimplemented**, per the task's explicit instruction. A read failure or
  YAML-parse failure is recorded as a per-file failure detail (same
  aggregation style as TC-010/TC-011's per-entry `inconsistencies`
  array); a `validateSkillsRoleIndex` `malformed` result is recorded with
  its own error message.
- One additional, TC-012-specific convention check (beyond what
  `validateSkillsRoleIndex` itself checks): the loaded `role` field must
  match the filename stem, by the same by-convention rule the
  schema-writer design itself documents ("role... [m]atches the filename
  stem by convention... not enforced by the type itself — a loader-level
  invariant"). This was explicitly called out in the design as something
  a *loader-level* check should enforce, and `TC-012` is exactly that
  loader-level consumer, so this check belongs here, not inside
  `validateSkillsRoleIndex` itself (which validates a value against the
  type's own shape, independent of any filename).
- **FAIL** if any file fails any of the above, with an aggregated
  `evidence` string listing every failing filename and its specific
  reason (same `${count}/${total} inválido(s): [...]` phrasing style as
  `IX-001`).
- **PASS** with `"${N}/${N} role-index válidos en <dir>."` if every file
  found validates cleanly.

Registered in `Axiom/packages/doctor/src/index.ts`'s barrel exports and
`runDoctorChecks`'s aggregation list, immediately after
`runSkillsCatalogCoverageCheck` (TC-010) and before
`runAgentsCatalogCoverageCheck` (TC-011) — same category, adjacent
check, minimal diff to the aggregation list.

### Dependency addition

`@axiom/doctor` did not previously depend on `@axiom/skills` (confirmed
by reading `packages/doctor/package.json` before editing — no
`@axiom/skills` entry existed). Added `"@axiom/skills": "*"` to
`packages/doctor/package.json`'s `dependencies` (same `"*"` workspace
version convention already used for every other internal `@axiom/*`
dependency in that file), and added the corresponding `paths`/
`references` entries to `packages/doctor/tsconfig.json` (matching the
existing pattern for every other cross-package TypeScript project
reference already present in that file — every other `@axiom/*` import
in `checks.ts` has a matching `tsconfig.json` entry; `@axiom/skills` was
the only one missing one after the import was added). This is the one
new cross-package dependency introduced by this increment; no other
package's dependency graph was touched.

### Tests added

`Axiom/packages/doctor/tests/checks.test.ts`: new `describe('
runSkillsRoleIndexCheck', ...)` block, 7 tests, following the same
`createTmpDir`/`scaffoldMinimalProject`/`resolveProject` conventions
already used by the adjacent `runArtifactIndexChecks` (`IX-001`) block
(the closest existing structural template — `TC-010`/`TC-011` have no
dedicated unit tests of their own in this file, only indirect coverage
via the `runDoctorChecks` integration tests, so `IX-001`'s block was
used as the template instead):

1. Not-found project → skip.
2. `skills-index/` directory absent → skip (the graceful-skip behavior
   the task required).
3. `skills-index/` directory present but empty → pass, `"0 role-index
   encontrados"`.
4. One valid role-index file present → pass, `"1/1 role-index
   válidos"`.
5. Malformed role-index (missing `role`) → fail, evidence names the
   filename.
6. YAML that does not parse → fail, evidence contains `"YAML
   inválido"`.
7. `role` field not matching the filename stem → fail, evidence names
   the mismatch explicitly.

All 7 pass. No existing test in `checks.test.ts` was modified beyond
adding the new import and the new `describe` block.

## Validation

Real validation was executed (not the "no validation command" fallback —
`package.json` scripts exist and were run), independently and freshly
this session, not assumed from the docs-skills-writer increment's
reported numbers.

### Package-scoped

- `@axiom/skills` `npm run build`: clean, no errors (built first, as a
  dependency of `@axiom/doctor`).
- `@axiom/doctor` `npm run typecheck`: clean, no errors.
- `@axiom/doctor` `npm run build` (including a forced clean rebuild after
  deleting `dist/`/`tsconfig.tsbuildinfo`, to rule out stale-build
  masking of the new `@axiom/skills` project reference): clean, no
  errors.
- `npx vitest run packages/doctor packages/skills`: **2 failed test
  files / 2 failed tests / 208 passed** (out of 210). Both failures
  pre-existing and unrelated to this increment:
  `packages/doctor/tests/checks.test.ts`'s
  `"runDoctorChecks — repositorio Axiom real"` test (environment-
  dependent: `resolveProject(process.cwd())` now returns `'ambiguous'`
  in this session's environment, not `'resolved'`/`'not-found'` as the
  test's own hardcoded assertion expects — confirmed pre-existing by
  `git stash`-ing all uncommitted changes, including this increment's
  own, and re-running the same test: it fails identically on the
  unmodified `main` tree, caused by the unrelated in-flight
  `project-resolution`/`filesystem-truth` work noted in "Context" above,
  not by anything in this increment or the INC-09 chain);
  `packages/skills/tests/catalog.test.ts`'s real-catalog integration
  test (`expected 'absent' to be 'ok'` — the same pre-existing gap all
  three prior INC-09 increments flagged: `Axiom/axiom.spec/config/
  skills-catalog.yaml` still does not exist on disk, confirmed again
  directly this session with a file-existence check).
- All 7 new `TC-012` tests pass; all 20 `role-index.test.ts` tests
  (from docs-skills-writer's increment) still pass, re-confirmed fresh
  this session, not assumed.

### Full monorepo

- `npm run typecheck` (root, `tsc -b`): clean, no errors.
- `npm run build` (root, `tsc -b`): clean, no errors.
- `npm test` (root, full `vitest` run), run twice (once before, once
  after adding the `tsconfig.json` project-reference entries for
  `@axiom/skills`, to confirm the fix did not change behavior): **12
  failed test files / 13 failed tests / 1374 passed** (out of 1387
  total), both times, identically.
  - Compared against the task's stated prior baseline (12 failed
    files / 13 failed tests / 1367 passed, out of 1360 total — note the
    task's stated total of 1360 does not match its own stated 12+1367
    arithmetic exactly against 1387; treating the freshly-measured
    numbers below as authoritative since they come from an actual run
    this session, not the task's paraphrase): failed-file count (12) and
    failed-test count (13) are **identical**; passed count (1374) is
    **7 higher** than the impl increment's own reported 1367, exactly
    matching the 7 new `TC-012` tests added by this increment. Zero
    regressions.
  - The list of 12 failing test files was enumerated explicitly this
    session (not inferred): `apps/cli/tests/start.test.ts`,
    `packages/agents/tests/catalog.test.ts`,
    `packages/doctor/tests/checks.test.ts`,
    `packages/model-routing/tests/assignments.test.ts`,
    `packages/model-routing/tests/loader.test.ts`,
    `packages/model-routing/tests/opencode-projection.test.ts`,
    `packages/model-routing/tests/resolver.test.ts`,
    `packages/project-resolution/tests/resolver.test.ts`,
    `packages/skills/tests/catalog.test.ts`,
    `packages/telemetry/tests/audit-trail-sink.test.ts`,
    `packages/toolchain/tests/repair-add-gitignore.test.ts`,
    `packages/tui/tests/driver.test.ts` (2 failing tests within this one
    file, accounting for the 13th failed test) — none of these touch
    `@axiom/skills`'s role-index code or `@axiom/doctor`'s skills
    category beyond the one already-known, already-explained
    `checks.test.ts` case.

## Result

Independently re-verified, by direct code reading (not by trusting prior
increments' prose), that `role-index.ts`'s schema, loader, and validator
match the schema-writer design field-for-field, correctly reject every
class of malformed input the design anticipated, and correctly accept
the worked `backend-developer` fixture end-to-end. Confirmed
`getRoleIndexEntryById` resolves ids against the role-index's own
`mandatory`/`available` entries (not the catalog, by design) and returns
`undefined` rather than throwing on a miss. Confirmed via `git diff` that
`CatalogEntry`/`SkillsCatalog`/`apply.ts`/`materialize.ts`/`refresh.ts`
remain completely untouched across the whole INC-09 chain.

Implemented `TC-012` (`skills-role-index-validity`) in
`Axiom/packages/doctor/src/checks.ts`, confirming `TC-012` was indeed the
next-available id (highest existing was `TC-011`). The check reuses
`validateSkillsRoleIndex` directly (no reimplementation), follows the
existing `pass/fail/warn/skip` + evidence convention exactly, and skips
gracefully — not fails — when `axiom.spec/config/skills-index/` does not
exist, matching this codebase's established pattern for optional,
absent-but-not-mandatory structures. Added `@axiom/skills` as a new
dependency of `@axiom/doctor` (`package.json` + matching
`tsconfig.json` project-reference entries, the one piece the initial
implementation needed that a plain import alone did not surface until a
clean rebuild was forced). Added 7 new tests covering skip-on-absent,
pass-on-empty, pass-on-valid, and three distinct fail modes.

Ran real validation fresh this session at every level (package-scoped
and full-monorepo `typecheck`/`build`/`test`), not assumed from any
prior increment's reported numbers: full monorepo is **12 failed test
files / 13 failed tests / 1374 passed** (out of 1387), identical
failed-file/failed-test counts to the pre-existing baseline, +7 passed
from this increment's own new tests. Zero regressions confirmed by name
(all 12 failing files enumerated and cross-checked as unrelated to this
increment's changes).

## INC-09 closure assessment

All four subagent roles in INC-09's sequence
(migration-engineer -> schema-writer -> docs-skills-writer ->
validator-reviewer) have now run and produced their specs. Reviewing
INC-09 as a whole against the closure rules in `Axiom.SDD/AGENTS.md`:

- **Goal/expected behavior clear**: yes — add a per-role skills index
  layer (source doc 9.4's shape) on top of the existing flat
  `SkillsCatalog`, additive, plus a `doctor` check for it. Confirmed
  clear and achieved across all four increments.
- **Acceptance criteria exist**: yes, in all four increment specs
  (audit, design, impl, this validator spec), each satisfied.
- **Changes implemented**: yes —
  `Axiom/packages/skills/src/role-index.ts` (types, loader, validator,
  lookup helper), barrel exports, two fixture files (YAML index + role
  catalog markdown), 20 `role-index.test.ts` tests (docs-skills-writer),
  plus this increment's `TC-012` check and 7 tests
  (validator-reviewer).
- **Available validation executed**: yes, at every step — each
  increment ran real `typecheck`/`build`/`test` commands, and this
  increment independently re-ran and re-confirmed them fresh rather
  than trusting prior reports.
- **Review against intent and acceptance criteria done**: yes — this
  increment's "Independent verification findings" section is exactly
  that review, performed independently against live code, not a restated
  summary of the prior increments' own self-assessment.
- **Stable knowledge integrated into `general-spec.md` when
  applicable**: done by this increment (see "General spec integration"
  below) — deferred by all three prior increments specifically until
  this final step, per their own stated closure rationale, which this
  increment now fulfills.
- **Result documented clearly**: yes, in this file and the three prior
  ones.

**All closure conditions are met. INC-09 is closed as a whole.**

### Deferred items (explicitly named, not silently dropped)

1. **`Axiom/axiom.spec/config/skills-catalog.yaml` still does not exist
   on disk.** Flagged first by the migration-engineer audit, re-confirmed
   by schema-writer, docs-skills-writer, and now this increment
   (independently re-checked with a fresh file-existence check this
   session, still absent). This causes one pre-existing, unrelated test
   failure (`packages/skills/tests/catalog.test.ts`'s real-catalog
   integration test) that predates and is unrelated to the entire INC-09
   chain. Not fixed by this increment or any prior INC-09 increment,
   consistently, on the same stated rationale: creating that file was
   never in scope for a role-index feature increment, and silently
   creating it as a side effect would hide a separate, real gap that
   deserves its own increment.
2. **No `id`-to-catalog cross-reference resolution exists yet** (a
   future loader/consumer resolving a role-index entry's `path` via
   `getSkillById(catalog, entry.id)?.source` when `path` is absent, per
   the design's own "id reuses catalog id" section). Explicitly deferred
   by the design increment as "a design note for the future loader
   implementer, not code shipped by this increment," and confirmed still
   accurate and still not needed by `TC-012` (which validates shape
   only, not cross-artifact consistency) — there is also nothing to
   cross-reference against yet, since `skills-catalog.yaml` does not
   exist (item 1 above) and no role-index skill id is cataloged there.
3. **Source doc 10.2's technical-context per-role index**
   (`mandatory.always`/`mandatory.whenTags`) remains out of scope for
   INC-09 entirely — it is a different artifact in a different domain,
   assigned to INC-10 (`"Reconcile technical context structure"`) by the
   parent roadmap. Not started by any INC-09 increment, by design.
4. **The 11 other pre-existing failing test files** unrelated to
   `@axiom/skills`/`@axiom/doctor`'s skills category (`tui`, `telemetry`,
   `toolchain`, `model-routing`, `project-resolution`, `agents`,
   `apps/cli/tests/start.test.ts`) are unrelated to INC-09 and were not
   investigated or fixed as part of this chain — confirmed by name in
   "Validation" above, unrelated to any file this increment or the rest
   of the INC-09 chain touches.

None of these four items block INC-09's own closure: none of them is an
INC-09 acceptance criterion, and all four were already known, named, and
explicitly out-of-scope by at least one prior increment in this chain
before this increment re-confirmed them.

## General spec integration

Added a new "Skills role-index" section to `Axiom.Spec/general-spec.md`,
consolidating the stable, now-proven-by-code knowledge from the full
INC-09 chain: the role-index concept and its relationship to the
existing flat catalog, the exact file-placement convention (both the
YAML index and the 9.3 markdown catalog), and the Q-skills-3 "SDD repo"
naming clarification (a genuinely reusable disambiguation for any future
increment that says "the SDD repo" in this workspace). Kept deliberately
concise per `general-spec.md`'s own stated posture ("what is currently
true and load-bearing, not a narration of how it was built") — the full
schema-field-by-field rationale, the four source-doc subsections, and the
complete worked example stay in the increment specs under
`specs/increments/`, not duplicated here. This is the first "General
spec integration" performed by the INC-09 chain: all three prior
increments explicitly deferred integration until this final,
validator-reviewer step, per their own stated closure rationale — this
increment fulfills that deferral.

## Next step recommendation

INC-09 is closed. Per the parent roadmap
(`Axiom.Spec/specs/increments/INC-20260702-axiom-redesign-roadmap/README.md`),
the next increment in the roadmap sequence is **INC-10 ("Reconcile
technical context structure")** — source doc section 10's
technical-context domain (canonical technical context living in the spec
repo, the separate `mandatory.always`/`mandatory.whenTags`/`available`
nested index shape at 10.2, explicitly out of scope for INC-09 per both
the migration-engineer audit's and schema-writer design's own Non-goals).
Recommend starting INC-10 with the same four-subagent lightweight pattern
this chain used (audit -> design -> implementation -> validation), since
it worked cleanly across all four INC-09 increments with no rework
needed between steps.
