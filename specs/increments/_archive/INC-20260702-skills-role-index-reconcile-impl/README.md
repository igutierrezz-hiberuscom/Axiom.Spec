# Increment: Reconcile SDD/role skills (`@axiom/skills` catalog) — docs-skills-writer implementation

Status: closed
Date: 2026-07-02

Closed as part of INC-09's overall closure. See
`INC-20260702-skills-role-index-reconcile-validator/README.md` ("INC-09
closure assessment") for the chain-wide closure rationale and deferred
items.

## Goal

Execute the **docs-skills-writer** step of INC-09's subagent sequence
(migration-engineer -> schema-writer -> **docs-skills-writer** ->
validator-reviewer), implementing the schema-writer design
(`Axiom.Spec/specs/increments/INC-20260702-skills-role-index-reconcile-design/README.md`)
as real, additive code and example content inside the `Axiom` monorepo:
the `SkillsRoleIndex` TypeScript type, a loader
(`loadSkillsRoleIndex`), a reusable validator
(`validateSkillsRoleIndex`), tests, and one fully worked
`backend-developer` example (both the machine-readable YAML index and
the human-readable role-catalog markdown), placed as fixtures rather
than inside a real project's `axiom.spec/` (which does not exist yet,
per the migration-engineer audit's finding, still true at the time of
this increment).

## Context

Parent chain: `INC-20260702-axiom-redesign-roadmap` (parent roadmap) ->
`INC-20260702-skills-role-index-reconcile` (migration-engineer audit,
status `pending`) -> `INC-20260702-skills-role-index-reconcile-design`
(schema-writer design, status `pending`) -> this increment
(docs-skills-writer implementation).

The schema-writer design is treated as the exact implementation
contract and is not re-litigated here. Its resolved decisions were
implemented verbatim:

- File placement: `<rootPath>/axiom.spec/config/skills-index/<role>.yaml`
  (Q-skills-1), one file per role.
- Independent `schemaVersion: 1` on the new artifact; the existing
  `skills-catalog.yaml`'s `schemaVersion: 1` untouched (Q-skills-2).
- Role-catalog markdown (source doc 9.3) placement:
  `<rootPath>/axiom.spec/target-axiom-skills/catalogs/<role>.md`
  (sibling decision under Q-skills-3).
- The full `SkillsRoleIndex` / `SkillsRoleIndexMandatoryEntry` /
  `SkillsRoleIndexAvailableEntry` shape, including the two additive
  optional fields (`reason`, `summary`) and the optional `repoKinds`
  scoping field.
- The worked `backend-developer` example (five skill ids, one
  mandatory + four available), reused verbatim from the design.

Live code read fresh this session before implementing (not reused from
prior prose): `Axiom/packages/skills/src/{types.ts,catalog.ts,apply.ts,
materialize.ts,refresh.ts,index.ts}`, `Axiom/packages/skills/tests/
catalog.test.ts` and `loader.test.ts` (for the exact test-file
conventions: `beforeEach`/`afterEach` tmpdir helpers, scenario-numbered
`describe` blocks, discriminated-union assertions), `Axiom/packages/
skills/package.json` (confirmed no `zod` dependency exists in this
package — validation is 100% manual narrowing functions returning
`{kind: 'ok'|'absent'|'malformed'}`, unlike `@axiom/config-validation`
which does use `zod`), and `Axiom/packages/config-validation/src/
validator.ts` (read for contrast, per the task's own suggestion to
check whether its pattern should be mirrored — it should not be,
because `@axiom/skills` never adopted `zod`; see "Implementation notes"
below for why the manual-narrowing pattern was matched instead).

## Scope

- Add `Axiom/packages/skills/src/role-index.ts`: `SkillsRoleIndex`,
  `SkillsRoleIndexMandatoryEntry`, `SkillsRoleIndexAvailableEntry`
  types; `loadSkillsRoleIndex(rootPath, role)` loader;
  `validateSkillsRoleIndex(parsed)` reusable validator;
  `getRoleIndexEntryById(index, id)` lookup helper;
  `skillsRoleIndexPath(rootPath, role)` and
  `SKILLS_ROLE_INDEX_RELATIVE_DIR` path helpers.
- Wire the new exports into `Axiom/packages/skills/src/index.ts`
  (barrel), additive only.
- Author the worked `backend-developer` role-index YAML example as a
  real file under `Axiom/packages/skills/tests/fixtures/axiom.spec/
  config/skills-index/backend-developer.yaml`.
- Author the corresponding `backend-developer.md` role-catalog
  markdown (source doc 9.3 style) as a real file under
  `Axiom/packages/skills/tests/fixtures/axiom.spec/
  target-axiom-skills/catalogs/backend-developer.md`.
- Add `Axiom/packages/skills/tests/role-index.test.ts` (20 tests, 5
  scenarios), following `catalog.test.ts`/`loader.test.ts`'s existing
  conventions exactly.
- Run typecheck/build/test scoped to `packages/skills` and across the
  full monorepo; record the exact before/after numbers.
- Update this increment spec's own file with implementation notes,
  validation results, and closure status.

## Non-goals

- No change to `CatalogEntry`, `SkillsCatalog`, `loadSkillsCatalog`,
  `getSkillById`, `apply.ts`, `materialize.ts`, `refresh.ts`, or the
  `axiom skills` CLI command. Confirmed untouched (see "Files changed"
  below — none of these files appear in the diff).
- No `doctor` check implementation (working name TC-012, per the
  audit's brief item 6 and the design's own forward note). That remains
  validator-reviewer's job, the next step in INC-09's sequence.
- No creation of a real project's `axiom.spec/config/skills-catalog.yaml`
  or `axiom.spec/config/skills-index/` directory inside `Axiom`'s own
  dogfooded root. The worked example lives under `packages/skills/
  tests/fixtures/` instead, because (a) no fixture/example convention
  exists elsewhere in `@axiom/skills` to extend, and (b) creating a
  real `axiom.spec/config/skills-catalog.yaml` was explicitly out of
  scope for the audit and design increments (a separate, pre-existing
  gap, not something this increment should silently fix as a side
  effect).
- No introduction of `zod` as a new dependency of `@axiom/skills`. The
  package's existing validation mechanism (manual narrowing functions
  returning a discriminated union) was matched instead — see
  "Implementation notes" for the rationale.
- No renaming, restructuring, or version bump of the existing
  `skills-catalog.yaml` schema.

## Acceptance criteria

- [x] `SkillsRoleIndex` type (and its two entry types) implemented in
      `Axiom/packages/skills/src/role-index.ts`, matching the
      schema-writer design's shape field-for-field (including
      `schemaVersion`, optional `repoKinds`, optional `path`/`reason`
      on mandatory entries, required `tags` + optional `path`/`summary`
      on available entries).
- [x] `loadSkillsRoleIndex(rootPath, role)` loader implemented,
      following the exact same read/parse/validate conventions as
      `loadSkillsCatalog` (`fs.existsSync` → `absent`, YAML parse
      failure → `malformed`, shape validation → `malformed`, valid →
      `ok`). Never throws.
- [x] `validateSkillsRoleIndex(parsed)` exported and reusable on an
      already-parsed value, independent of file I/O, so a future
      `doctor` check (TC-012) can call it directly.
- [x] The worked `backend-developer` YAML example authored as a real
      file, reachable by a loader call in a test
      (`loadSkillsRoleIndex(fixtureRoot, 'backend-developer')`), not
      just inlined as a string constant.
- [x] The corresponding `backend-developer.md` role-catalog markdown
      authored as a real file, source doc 9.3 style (prose + `##
      Available skills` section, one row per skill with a "use when"
      description), with every skill `id` cross-referenced against the
      YAML example.
- [x] Existing `SkillsCatalog`/`CatalogEntry` types, `apply.ts`,
      `materialize.ts`, `refresh.ts` confirmed untouched (git diff
      shows no changes to these files).
- [x] Tests written/updated following existing conventions in
      `packages/skills/tests/`; new tests pass; no existing test
      regressed.
- [x] `npm run typecheck`, `npm run build`, `npm test` executed both
      scoped to `packages/skills` and across the full monorepo; results
      recorded with real pass/fail counts, compared against a
      freshly-confirmed baseline (not assumed).
- [x] This spec file documents files changed, fixture placement
      rationale, and test results.
- [ ] Stable knowledge integrated into `Axiom.Spec/general-spec.md`
      when applicable — deferred; see "General spec integration"
      below (not yet applicable, chain not fully closed).

## Open questions

None blocking this increment's own scope. One item is carried forward
for validator-reviewer, not newly opened: the design's own "What
validator-reviewer will need later" section (TC-012 working name) is
unchanged by this increment — `validateSkillsRoleIndex` was built
specifically to be reusable by that future check without re-deriving
shape validation, but the check itself (pass/fail/warn/skip + evidence,
per TC-010's convention) is not implemented here.

## Assumptions

- "Role" continues to mean the source doc's product/task role concept
  (e.g. `backend-developer`), unchanged from the audit's and design's
  own Assumptions sections.
- Fixture placement under `packages/skills/tests/fixtures/axiom.spec/...`
  (mirroring the real `<rootPath>/axiom.spec/...` tree, just rooted at
  a test-fixtures directory instead of a real project root) is close
  enough in shape to the design's file-placement decision that a loader
  call against the fixture root (`loadSkillsRoleIndex(fixtureRoot,
  'backend-developer')`) exercises the exact same relative-path
  resolution logic (`SKILLS_ROLE_INDEX_RELATIVE_DIR`) a real project
  would use — this was chosen over inlining the example only as a
  string constant (like `catalog.test.ts`'s `VALID_7_SKILLS_YAML`)
  specifically so the design's "worked example" requirement produces an
  actual reusable file, not just prose.
- The pre-existing, unrelated 12-failed-file/13-failed-test baseline
  (across `tui`, `telemetry`, and one `packages/skills` integration
  test expecting a real `skills-catalog.yaml` that does not exist on
  disk) is out of scope to fix — confirmed by direct measurement before
  and after this increment's changes (see "Validation" below), not
  assumed from the task's stated prior baseline.

## Implementation notes

### Files changed

- `Axiom/packages/skills/src/role-index.ts` (new): `SkillsRoleIndex`,
  `SkillsRoleIndexSchemaVersion`, `SkillsRoleIndexMandatoryEntry`,
  `SkillsRoleIndexAvailableEntry`, `SkillsRoleIndexResult`,
  `SkillsRoleIndexValidationResult` types; `loadSkillsRoleIndex`,
  `validateSkillsRoleIndex`, `getRoleIndexEntryById`,
  `skillsRoleIndexPath` functions; `SKILLS_ROLE_INDEX_RELATIVE_DIR`
  constant.
- `Axiom/packages/skills/src/index.ts` (modified, additive only): new
  barrel exports for everything in `role-index.ts`. No existing export
  line was changed or removed.
- `Axiom/packages/skills/tests/role-index.test.ts` (new): 20 tests
  across 5 scenarios (valid load + fixture integration test, lookup
  helper, 8 malformed-shape sub-cases, absent-file case + path-resolver
  unit test, reusable-validator unit tests).
- `Axiom/packages/skills/tests/fixtures/axiom.spec/config/skills-index/backend-developer.yaml`
  (new): the worked example YAML, copied verbatim from the
  schema-writer design's "Worked example" section.
- `Axiom/packages/skills/tests/fixtures/axiom.spec/target-axiom-skills/catalogs/backend-developer.md`
  (new): the role-catalog markdown, source doc 9.3 style.

No other file in `Axiom/packages/*`, `Axiom/apps/*`, or `Axiom.SDD/`
was modified.

### Validation mechanism: manual narrowing, not Zod

`@axiom/skills`'s `package.json` has no `zod` dependency, and
`catalog.ts`'s `loadSkillsCatalog` validates shape entirely with hand-
written type-narrowing helpers (`isObject`, `isNonEmptyString`) plus
per-field checks, returning a `{kind: 'ok'|'absent'|'malformed'}`
discriminated union. `@axiom/config-validation` (a different package,
used by `axiom.yaml`/`policy-as-code.yaml`/etc.) does use `zod`, but
that is a different package with a different existing dependency
footprint — introducing `zod` into `@axiom/skills` purely for this one
new file would add a new dependency to a package that has deliberately
avoided one so far, for a shape no more complex than what the existing
manual pattern already handles. `role-index.ts` therefore mirrors
`catalog.ts`'s exact pattern: the same two narrowing helpers plus two
new ones (`isStringArray`, for `tags`/`repoKinds`), a duplicate-id
check mirroring `catalog.ts`'s `seenIds` set (extended to check
uniqueness across `mandatory`+`available` combined, per the design's
own note that this "mirrors the existing catalog's own duplicate-id
rejection"), and the identical never-throws contract.

### Loader/I/O conventions matched

- `loadSkillsRoleIndex` reuses the identical existence-check → read →
  `yaml.load` → shape-validate pipeline as `loadSkillsCatalog`,
  including identical error-message phrasing style (Spanish,
  `"YAML inválido: ..."`, `"... ausente o vacío."`, etc.) so tooling or
  humans reading `malformed` errors from either loader see a consistent
  voice.
- Shape validation was factored into a separate, exported
  `validateSkillsRoleIndex(parsed: unknown)` function (not present as a
  separate export in `catalog.ts`, since the audit/design's own
  "What validator-reviewer will need later" section explicitly asked
  for this to be reusable by a future `doctor` check without
  re-deriving parsing). `loadSkillsRoleIndex` itself is now a thin
  wrapper: existence check + read + YAML parse + delegate to
  `validateSkillsRoleIndex`.
- No new atomic-write helper was needed: this increment is read-only
  (a loader + validator), matching `catalog.ts`'s own read-only scope.
  `apply.ts`/`materialize.ts`'s atomic-write (`*.tmp` + `renameSync`)
  pattern was reviewed but is not applicable here since nothing in this
  increment writes to `<rootPath>/axiom.spec/...` at runtime — only the
  test fixtures were authored by hand, once, as static example content.

### Fixture placement decision

No `fixtures/` (or equivalent example/seed-content) directory convention
existed anywhere in `@axiom/skills` or, on inspection, in any other
`Axiom/packages/*` package before this increment — every existing test
(`catalog.test.ts`, `loader.test.ts`, etc.) inlines its YAML/JSON test
bodies as string constants written to a `beforeEach`-created tmpdir.
Given the task's explicit requirement to author the worked example "as
a real file" (not only as an inline string), and the audit/design's own
finding that no real project has bootstrapped `axiom.spec/` yet, the
fixture was placed at
`Axiom/packages/skills/tests/fixtures/axiom.spec/config/skills-index/backend-developer.yaml`
and
`Axiom/packages/skills/tests/fixtures/axiom.spec/target-axiom-skills/catalogs/backend-developer.md`
— a `tests/fixtures/` root that mirrors the exact relative-path shape a
real project's `rootPath` would have (`axiom.spec/config/skills-index/
<role>.yaml`, `axiom.spec/target-axiom-skills/catalogs/<role>.md`), so
a test can call `loadSkillsRoleIndex(fixtureRoot, 'backend-developer')`
and exercise the real `SKILLS_ROLE_INDEX_RELATIVE_DIR` resolution logic
end-to-end, not just a hand-rolled tmpdir path. This is new-but-minimal:
one new subdirectory (`tests/fixtures/`) under an existing test root,
not a new top-level convention, and it does not touch or require the
still-absent real `axiom.spec/` at the `Axiom` repo root.

### What was intentionally not built

- No `doctor` check (TC-012) — validator-reviewer's job next, per the
  audit's brief item 6 and the design's forward note. `validateSkillsRoleIndex`
  was specifically factored out to make that check's future
  implementation straightforward (parse YAML, call the exported
  validator, map `ok`/`malformed` to `pass`/`fail` plus the
  catalog-cross-reference and duplicate-id checks TC-012 will need).
- No `id`-to-catalog cross-reference resolution (e.g. a helper that
  resolves `entry.path` via `getSkillById(catalog, entry.id)?.source`
  when `path` is absent, as the design's "id reuses catalog id" section
  describes as a future loader behavior). The design explicitly framed
  this as "a design note for the future loader implementer, not code
  shipped by this increment" — and since none of the five worked-example
  skill ids exist in the current (also-still-absent) `skills-catalog.yaml`,
  there is nothing to cross-reference yet; building this resolution now
  would be speculative ahead of an actual consumer needing it.

## Validation

Real validation was executed (not the "no validation command" fallback
— `package.json` scripts exist and were run).

### Baseline (measured fresh before implementing, not assumed)

- Full monorepo `npm test`: **12 failed test files / 13 failed tests /
  1347 passed** (out of 1360 total). Failures are pre-existing and
  unrelated to `@axiom/skills`: `packages/tui/tests/driver.test.ts`
  (upgrade-flow TUI rendering), `packages/telemetry/tests/
  audit-trail-sink.test.ts` (retention-sweep timing), and others not
  touched by this increment.
- `packages/skills` scoped (`npx vitest run packages/skills`) before
  this increment's changes: **1 failed test file / 1 failed test / 50
  passed** (out of 51). The single failure:
  `catalog.test.ts`'s integration test expecting a real
  `Axiom/axiom.spec/config/skills-catalog.yaml` with 7 entries — the
  file does not exist on disk, confirming the migration-engineer
  audit's own finding is still true at this increment's start. This
  failure is one of the 13 counted in the full-monorepo baseline above.

### After implementation

- `packages/skills` scoped `npm run typecheck`: clean, no errors.
- `packages/skills` scoped `npm run build`: clean, no errors.
- `packages/skills` scoped `npx vitest run packages/skills`: **1 failed
  test file / 1 failed test / 70 passed** (out of 71) — the same
  pre-existing `catalog.test.ts` integration-test failure, unchanged;
  all 20 new `role-index.test.ts` tests pass; no existing test
  regressed.
- Full monorepo `npm run typecheck`: clean, no errors.
- Full monorepo `npm run build`: clean, no errors.
- Full monorepo `npm test`: **12 failed test files / 13 failed tests /
  1367 passed** (out of 1380 total) — identical failed-file and
  failed-test counts to the freshly-measured baseline above (same 13
  pre-existing failures, none new), and passed count grew by exactly 20
  (the new `role-index.test.ts` scenarios). Confirms zero regressions
  introduced by this increment.

## Result

Implemented the schema-writer design's `SkillsRoleIndex` artifact as
real, additive TypeScript in `Axiom/packages/skills/src/role-index.ts`:
the type shape (schema-versioned, optional `repoKinds`, optional
`path`/`reason` on `mandatory`, required `tags` + optional `path`/
`summary` on `available`), a defensive never-throws loader
(`loadSkillsRoleIndex`) matching `catalog.ts`'s exact I/O/validation
conventions, a reusable standalone validator (`validateSkillsRoleIndex`)
for a future `doctor` check to consume directly, and a lookup helper
(`getRoleIndexEntryById`). Authored the worked `backend-developer`
example as two real files (YAML index + markdown catalog) under
`packages/skills/tests/fixtures/`, since no real project has
bootstrapped `axiom.spec/` yet and no fixture-directory convention
existed elsewhere in the package to extend — this fixture root mirrors
the real relative-path shape so the loader's actual path-resolution
logic is exercised end-to-end by a test, not bypassed. Added 20 new
tests across 5 scenarios, matching `catalog.test.ts`/`loader.test.ts`'s
existing structure exactly. Confirmed via direct measurement (not
assumption) that the pre-existing 12-failed-file/13-failed-test
baseline is completely unchanged after this increment, with 20 new
passing tests added on top. `CatalogEntry`, `SkillsCatalog`,
`loadSkillsCatalog`, `apply.ts`, `materialize.ts`, and `refresh.ts` are
confirmed untouched — this was a purely additive, new-file change.

Status is `pending`, not `closed`: validator-reviewer (the final step
of INC-09's subagent sequence, per both the audit's and design's own
next-step recommendations) has not yet run, and `general-spec.md`
integration is deferred until that step closes out the full chain (see
below), consistent with the design increment's own closure rationale.

## General spec integration

No integration into `Axiom.Spec/general-spec.md` performed yet.
Rationale, consistent with both prior increments in this chain: this
increment implements a design that has not yet survived the final
step of its own subagent sequence — validator-reviewer's `doctor` check
(TC-012) has not been built, so there is no automated, repeatable way
yet to confirm a role-index file in the wild actually satisfies this
shape end-to-end (parse + cross-reference + duplicate-id check) the way
TC-010 does for the existing catalog. Locking the role-index shape into
`general-spec.md` now would consolidate an artifact whose full
validation story is still one step short of complete. The correct
integration point remains after validator-reviewer closes the full
INC-09 chain (migration-engineer -> schema-writer -> docs-skills-writer
-> validator-reviewer), at which point `general-spec.md` should gain a
concise "Skills role-index" section documenting: the file placement
pattern, the schema shape, the `id`-reuses-catalog convention, and the
existence of TC-012 as its validation gate — mirroring how the existing
catalog facts are already implicitly documented via TC-010's presence
in this codebase's tooling.

## Next step recommendation

Hand off to **validator-reviewer** next, the final step of INC-09's
subagent sequence. Exact inputs it needs:

1. **This increment's `role-index.ts`** (`SkillsRoleIndex`,
   `loadSkillsRoleIndex`, `validateSkillsRoleIndex`,
   `getRoleIndexEntryById`) as the artifact to build a `doctor` check
   against. `validateSkillsRoleIndex` is already reusable and should be
   called directly by the new check rather than re-implemented.
2. **The design's "What validator-reviewer will need later" section**
   (working name TC-012, next available number after TC-011): verify
   per role-index file that (a) it parses and matches the schema — call
   `loadSkillsRoleIndex`/`validateSkillsRoleIndex` directly, (b) every
   `mandatory`/`available` entry's `id` either resolves in
   `skills-catalog.yaml` (via `getSkillById`) or has an explicit `path`
   that exists on disk, and (c) no duplicate `id` within the combined
   `mandatory`+`available` lists — already enforced by
   `validateSkillsRoleIndex` itself, so TC-012 mainly needs to surface
   that as a `fail` rather than re-deriving it.
3. **`Axiom/packages/doctor/src/checks.ts`'s TC-010 region** (already
   read and cited by both the audit and design increments) as the
   `pass`/`fail`/`warn`/`skip` + evidence convention to follow — do not
   invent a new check-result shape.
4. **The fixture files this increment authored**
   (`packages/skills/tests/fixtures/axiom.spec/config/skills-index/backend-developer.yaml`
   and the sibling `.md` catalog) as a ready-made valid + invalid test
   corpus base for TC-012's own tests (invalid variants can be derived
   by mutating the fixture, following this increment's
   `role-index.test.ts` malformed-shape sub-cases as a template).

After validator-reviewer closes TC-012, integrate the consolidated
"Skills role-index" section into `Axiom.Spec/general-spec.md`, closing
out the full INC-09 chain.
