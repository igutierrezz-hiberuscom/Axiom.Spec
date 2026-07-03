# Increment: Registry + manifest schema v2 (resolver fix)

Status: closed
Date: 2026-07-02

## Goal

Fix the blocking mismatch flagged by the **cli-implementer** step of
INC-01's subagent sequence
(`Axiom.Spec/specs/increments/INC-20260702-registry-manifest-schema-v2-cli/README.md`,
"Flagged design/live-code mismatch #1"): make
`@axiom/project-resolution/src/resolver.ts`'s `resolveProject`
version-aware, so it correctly resolves both `axiom.yaml` schemaVersion
absent (v1, `project.name`/`project.mode`/`scopes`) and `schemaVersion: 2`
(`projectId`/`name`/`repoId`/`role`/`paths`) documents, using the same
version-dispatch already implemented in
`@axiom/config-validation/src/validator.ts`'s `validateAxiomYamlContent`.

This is a small, focused follow-up increment, not one of INC-01's
original five subagent roles, but required to unblock `init.ts`'s
originally-requested `schemaVersion: 2` emission and to make
`join`/`repo attach`/`tui`'s topology/model-validate modes correctly
resolve v2 projects once they exist.

## Context

Parent chain: `INC-20260702-axiom-redesign-roadmap` (parent roadmap) ->
`INC-20260702-registry-manifest-schema-v2` (migration-engineer audit) ->
`INC-20260702-registry-manifest-schema-v2-design` (schema-writer design)
-> `INC-20260702-registry-manifest-schema-v2-impl` (registry-engineer
implementation) -> `INC-20260702-registry-manifest-schema-v2-cli`
(cli-implementer cutover) -> this increment (resolver fix).

The cli-implementer spec attempted to flip `init.ts`'s `buildAxiomYaml`
to emit `schemaVersion: 2`, then reverted it after discovering that
`resolveProject` only understood the v1 shape (`project.name`): a v2
document has no `project.name` (it has a top-level `name`/`projectId`),
so `resolveProject` returned `status: 'ambiguous'` for every newly-`init`-
ed v2 project, which broke `join` (`assertUnambiguous` throws),
`projects join`/`repo attach` (`status !== 'resolved'` -> exit 1), and
`tui`'s `--topology`/`--model-validate`/`--components-show` modes. That
spec explicitly named the fix shape it expected ("a version-dispatching
`resolveProject`, mirroring `validateAxiomYamlContent`'s `{valid,
version, data}` design") and flagged it as new, small, focused follow-up
work, sequenced either before or in parallel with validator-reviewer.

All four prior specs in this chain were read in full before any code was
written. `@axiom/project-resolution/src/resolver.ts`,
`tests/resolver.test.ts`, `src/index.ts`,
`@axiom/config-validation/src/validator.ts`, and
`@axiom/config-validation/src/schemas.ts` were read in full and treated as
the authoritative existing contracts to build on, not redesign.

## Scope

- Make `resolveProject` (`@axiom/project-resolution/src/resolver.ts`)
  version-aware by delegating version-dispatch to
  `validateAxiomYamlContent` (`@axiom/config-validation`) instead of its
  own narrower, ad hoc `AxiomYamlConfig` interface and manual `yaml.load`.
- Correctly extract identifying/mode/scope information from whichever
  version is found: v1 (`project.name`/`project.mode`/`scopes`) or v2
  (`name`/`projectId`/`repoId`/`role`/`mode`/`paths`).
- Decide and implement the `ProjectResolution` return-shape change (if
  any) needed to carry version-specific data, after checking actual
  consumer usage (`join.ts` via `_shared.ts`, `projects.ts`, `repo.ts`,
  `tui.ts`, `app-api.ts`, `@axiom/doctor`, `@axiom/tool-routing`,
  `@axiom/isolation`).
- Update `@axiom/project-resolution/package.json` and `tsconfig.json` to
  declare the new dependency on `@axiom/config-validation`.
- Update/add tests in `tests/resolver.test.ts` covering v1, v2, ambiguous,
  and invalid-config cases for both versions.
- Run real validation (`npm run typecheck`, `npm run build`, `npm test`,
  full monorepo and scoped to `project-resolution` + `apps/cli`) and
  report actual pass/fail output, cross-checked against the failure sets
  already documented by the registry-engineer and cli-implementer
  increments.
- Judge (but do not implement) whether it is now safe to re-enable
  `schemaVersion: 2` emission in `apps/cli/src/commands/init.ts`.

## Non-goals

- Do NOT re-enable `axiom.yaml` `schemaVersion: 2` emission in `init.ts`
  in this increment. That re-enablement is a separate, later, explicitly
  verifiable step, per the task's own instruction — this increment only
  unblocks it and states whether it now looks safe.
- No `@axiom/doctor/src/checks.ts` changes (MC-001's pass/warn/fail
  three-way branch remains validator-reviewer's job, the final role in
  INC-01's original sequence).
- No CLI command changes beyond what is strictly required by a
  `ProjectResolution` shape change. As it turned out, no shape change was
  needed (see Implementation notes), so **zero CLI command files were
  touched**.
- No `@axiom/topology`, `@axiom/tui`, or migration-script changes,
  consistent with every prior increment's non-goals in this chain.
- No fix to the pre-existing, unrelated `resolver.test.ts` failure
  ("resuelve scopes con absolutePath y exists correcto" asserting
  `product.exists === false` for a scope whose `path: '.'` always
  resolves to `rootPath`, which always exists in the test fixture) — this
  is a latent, pre-existing test bug confirmed identical on a clean
  `main` checkout via `git stash` bisection (see Validation), unrelated to
  this fix, and out of this increment's scope to correct.

## Acceptance criteria

- [x] `resolveProject` uses `validateAxiomYamlContent`
      (`@axiom/config-validation`) for version-dispatch instead of its own
      ad hoc `AxiomYamlConfig` parsing.
- [x] A `schemaVersion: 2` `axiom.yaml` document resolves to
      `status: 'resolved'` with `name` populated from the v2 document's
      top-level `name` field, mirroring v1's `project.name` resolution.
- [x] A `schemaVersion` absent (v1) `axiom.yaml` document continues to
      resolve identically to the pre-fix resolver (same `status`, `name`,
      `scopes`, `mode` for every existing test fixture, modulo the one
      pre-existing, unrelated failure documented above).
- [x] `scopes` (v1) and `paths` (v2) both populate `ProjectResolution
      .scopes` via the same `buildScopeInfo`, since `paths` is the direct
      renamed-successor of `scopes` per the schema-writer design's
      Resolution #8.
- [x] `mode` (v1 `project.mode`, v2 top-level `mode`) is normalized to the
      same closed `ProjectMode` union in both versions, since `mode` is
      preserved verbatim as an independent axis per the schema-writer
      design's Resolution #7.
- [x] The pre-existing `ambiguous` vs `invalid-config` distinction is
      preserved exactly: a structurally-parseable document missing its
      identifying field (`project.name` in v1, `name`/`projectId` in v2)
      is `ambiguous`, not `invalid-config`; malformed YAML or a
      non-object document is `invalid-config`.
- [x] The `ProjectResolution` return shape change was decided based on
      actual consumer usage, not assumed; the decision and its rationale
      are documented below.
- [x] `tests/resolver.test.ts` covers v1 resolved, v1 ambiguous, v1
      invalid, v2 resolved, v2 scopes/paths, v2 mode default and explicit,
      v2 invalid (missing required fields), and `assertUnambiguous` for
      both versions.
- [x] Real validation commands were run (`npm run typecheck`, `npm run
      build`, `npm test` full monorepo, plus scoped runs for
      `project-resolution`, `config-validation`, and `apps/cli`) and their
      actual pass/fail output is reported below.
- [x] Every pre-existing test failure encountered is cross-checked against
      the failure sets already documented by the registry-engineer and
      cli-implementer increments, not re-investigated from scratch.
- [x] A judgment on whether `schemaVersion: 2` emission in `init.ts` is
      now safe to re-enable is stated explicitly, without implementing it.

## Open questions

None blocking this increment's own closure. Two items remain open at the
parent-roadmap level (Q3: `axiom.spec/` prefix survival; Q4:
capability/provider/MCP folding) and are untouched by this increment,
consistent with all four prior specs in this chain.

## Assumptions

- Inherited unchanged from all four prior specs in this chain: the
  non-hard-removal posture for D1/D2/D3, `@axiom/core`'s `Result<T, E>`
  pattern is not applicable here (this package predates that pattern and
  was not asked to adopt it), and no migration-script implementation.
- `@axiom/config-validation` has no dependency (direct or transitive) on
  `@axiom/project-resolution`, confirmed by reading its `package.json` —
  adding `@axiom/config-validation` as a dependency of
  `@axiom/project-resolution` does not introduce a cycle.
- `role` (v2, per-repo) is a new, independent axis from `mode`
  (execution-topology) per the schema-writer design's Resolution #7 — it
  is not resolved into `ProjectResolution` as a new field because no
  current consumer needs it; it remains accessible via `rawConfig.role`
  if a future consumer needs it, consistent with `rawConfig` already
  being the resolver's existing escape hatch for raw-document access
  (used today by `@axiom/doctor`'s BC-001).

## Implementation notes

### Files changed

**`Axiom/packages/project-resolution/src/resolver.ts`**:

- Removed the internal, narrow `AxiomYamlConfig` interface and the direct
  `yaml.load`/manual-shape-check parsing it drove.
- `resolveProject` now calls `validateAxiomYamlContent(rawContent)`
  (`@axiom/config-validation`) and dispatches on its
  `AxiomYamlValidationResult` discriminated union:
  - `valid: true, version: 1` -> `resolveFromV1` (new helper): same logic
    as the pre-fix resolver, reading `config.project.name`,
    `config.project.mode`, `config.scopes`.
  - `valid: true, version: 2` -> `resolveFromV2` (new helper): reads
    `config.name` (not `config.projectId`) as the resolver's `name`,
    `config.mode` (top-level, independent axis), `config.paths` (renamed
    successor of `scopes`).
  - `valid: false` -> `resolveInvalid` (new helper, see "Preserving the
    `ambiguous` vs `invalid-config` distinction" below).
- Added a shared `buildScopesFromEntries` helper (used by both
  `resolveFromV1` and `resolveFromV2`) since `scopes` (v1) and `paths`
  (v2) have the identical entry shape (`path`, `product_runtime`).
- Added a shared `normalizeMode` helper (used by both), since `mode`
  normalization to the closed `ProjectMode` union is identical in both
  versions.
- `js-yaml` import retained (see "Preserving the `ambiguous` vs
  `invalid-config` distinction" below) — it is still needed for a loose,
  non-Zod re-parse in one specific fallback path, not removed as
  initially planned.

**`Axiom/packages/project-resolution/package.json`**: added
`@axiom/config-validation` to `dependencies` (verified no dependency
cycle — `@axiom/config-validation`'s own `package.json` depends only on
`js-yaml`/`zod`). `js-yaml`/`@types/js-yaml` remain declared (see above;
initially removed, then restored once the `ambiguous`-preservation fix
required them again — see "Mismatches found and fixed during this
increment's own validation" below).

**`Axiom/packages/project-resolution/tsconfig.json`**: added a
`@axiom/config-validation` path mapping and a
`{ "path": "../config-validation" }` project reference, mirroring the
existing pattern for `@axiom/filesystem-truth` in the same file and the
pattern used by `@axiom/topology`'s `tsconfig.json` for its own
dependency on `@axiom/core`. No change needed in `Axiom/vitest.config.ts`
— `@axiom/config-validation` was already aliased there for other
consumers.

**`Axiom/packages/project-resolution/tests/resolver.test.ts`**: added a
`VALID_AXIOM_YAML_V2` fixture and 8 new test cases (v2 resolved, v2
scopes/paths with `exists`/`isProductRuntime`, v2 explicit `mode`, v2
default `mode`, v2 invalid-config for missing required fields, v2
deterministic resolution, `assertUnambiguous` non-throw for v2). Existing
v1 test names were annotated `(v1)` for clarity now that both versions
are covered in the same file; no existing assertion was weakened, only
one new v2 test (see "Mismatches" below) deliberately avoided reproducing
a known pre-existing, unrelated assertion bug from the v1 fixture instead
of copying it into v2.

**Not touched** (confirmed by re-reading before finishing): `@axiom/
doctor/src/checks.ts`, `@axiom/topology/*`, `@axiom/tui/*`, all
`apps/cli/src/commands/*.ts` files (including `init.ts` — its
`schemaVersion: 2` emission remains reverted/commented-out exactly as the
cli-implementer increment left it).

### Return-shape decision: no `version` field added, both versions normalize into the same external `ProjectResolution`

Before deciding, every consumer of `resolveProject`/`ProjectResolution`
was grepped and inspected (`import ... from '@axiom/project-resolution'`,
33 matches across `apps/cli/src/commands/{_shared,repo,tui,projects,
toolchain,upgrade}.ts`, `apps/cli/src/index.ts`, `@axiom/doctor`
(`checks.ts`, `report.ts`, `index.ts`, `governance-checks.ts`),
`@axiom/tool-routing` (`fallback.ts`, `events.ts`, `dispatcher.ts`),
`@axiom/isolation/src/p0.ts`, `@axiom/tui/src/driver.ts`, and their
respective test files). None of them branch on a manifest "version" —
every real consumer reads `name`, `rootPath`, `configPath`, `scopes`,
`status`, or `mode`, which are already normalized, version-independent
concepts. The single exception is `@axiom/doctor`'s `runBoundaryChecks`
(BC-001), which reads `resolution.rawConfig.project.mode` directly to
special-case `'installed-multi-repo'` — this already only works for v1
documents (a v2 document has no `.project` object at all, so this reads
as `null`/`false` for v2, a safe degrade-not-crash, not a new problem
introduced by this fix: BC-001 was never in this increment's scope to
change, per the cli-implementer spec's own non-goals, and it is flagged
here for `@axiom/doctor`'s benefit, not fixed).

Decision: **do not add a `version` field to `ProjectResolution`.**
Both v1 and v2 normalize into the exact same external shape
(`name`/`rootPath`/`configPath`/`scopes`/`status`/`mode`/
`localOverlayPath`/`hasLocalOverlay`/`rawConfig`), consistent with
`validateAxiomYamlContent`'s own precedent of exposing a `version`
discriminant *at its own boundary* (where two structurally different Zod
types need distinguishing) without forcing every downstream consumer to
re-discriminate on it. `rawConfig` continues to expose the raw parsed
document (v1 or v2 shape) unchanged, as it already did pre-fix, for the
one consumer (`@axiom/doctor`'s BC-001) that inspects it structurally.
This is the minimal, non-speculative shape change: **zero** — no CLI
command, doctor check, or any other consumer file needed to change,
which was verified empirically (typecheck/build are clean across the
whole monorepo with zero other files touched).

### Preserving the `ambiguous` vs `invalid-config` distinction

A real bug was found and fixed during this increment's own validation,
not silently left in: `validateAxiomYamlContent` validates against the
**full** Zod schema (`AxiomYamlSchema`/`AxiomYamlSchemaV2`), which
requires `project.name` (v1) / `projectId`+`name`+`repoId`+`role` (v2).
The pre-fix resolver, by contrast, used a bare `yaml.load` with no Zod
validation at all, and only special-cased a missing `project.name` as
`status: 'ambiguous'` — a distinct, milder status than
`status: 'invalid-config'` (malformed YAML), preserved by
`assertUnambiguous`'s own distinct error message ("la resolución es
ambigua" vs "axiom.yaml inválido").

Naively routing everything through `validateAxiomYamlContent` and
treating any `valid: false` as `invalid-config` collapsed this
distinction: a v1 document missing only `project.name` (which the
pre-fix resolver correctly called `ambiguous`) now failed Zod entirely
and became `invalid-config` instead. This was caught by this increment's
own pre-existing test (`devuelve status=ambiguous cuando falta
project.name`) failing after the initial implementation — not
discovered by manual reasoning alone, but confirmed empirically by
running the test suite before finishing.

**Fix**: `resolveProject`'s `!validation.valid` branch now calls a new
`resolveInvalid` helper, which re-parses the raw content with a loose
(non-Zod) `yaml.load` and applies the exact same rule the pre-fix
resolver used: if the document parses to an object but lacks its
identifying field (`project.name` for a v1-shaped document, `name` or
`projectId` for a document that declares `schemaVersion: 2`), the status
is `ambiguous`; any other failure (malformed YAML, non-object document)
remains `invalid-config`. This restores the exact pre-fix `ambiguous`
contract for v1 (verified: the pre-existing `ambiguous` test now passes
identically to before this fix) and extends the identical rule to v2
documents that declare `schemaVersion: 2` but omit their own identifying
fields (new test added, passing). This is why `js-yaml` remains a direct
dependency of `@axiom/project-resolution` — it was not, in the end,
fully replaced by `validateAxiomYamlContent`, because the validator's
Zod-strict `valid: false` result does not carry enough information on
its own to distinguish "merely missing the identifying field" from "any
other structural problem" without re-parsing loosely once.

### Mismatches found and fixed during this increment's own validation

1. **`ambiguous` collapsing to `invalid-config`** — described in full
   above. Found by the increment's own pre-existing test failing, fixed
   before finishing (not left as a known issue).
2. **A new v2 test copying a pre-existing, unrelated v1 test bug** — the
   v1 fixture's `product` scope (`path: '.'`) always resolves to
   `rootPath`, which always exists in the test's own tmp-directory setup;
   the pre-fix v1 test asserting `product.exists === false` for this
   fixture was already failing on a clean `main` checkout before this
   increment touched anything (confirmed via `git stash` /
   `git stash pop` bisection, run once in this increment's own session,
   reproducing the identical single failure with zero changes applied).
   This increment's first draft of the analogous v2 test copied the same
   flawed assertion into a new test, which would have encoded a second
   instance of a known-incorrect expectation. Fixed by asserting
   `product.exists === true` for the v2 test instead (the factually
   correct value for `path: '.'`), with an inline comment explaining why
   the v2 test does not mirror the v1 fixture's own long-standing
   assertion bug, rather than silently reproducing it. The pre-existing
   v1 test itself was left untouched, per this increment's own
   non-goals — fixing it is unrelated to the resolver version-dispatch
   fix and belongs to whichever future work touches
   `resolver.test.ts`'s v1 fixtures directly.

## Validation

Real validation commands were run (not the "no validation command found"
fallback — this increment has actual code).

**`npm run typecheck` (root, `tsc -b`, all project references):**

```
> axiom-product@0.1.0 typecheck
> tsc -b
```

Exit code 0, zero errors, across the entire monorepo (including the new
`project-resolution -> config-validation` project reference).

**`npm run build` (root, `tsc -b`):**

```
> axiom-product@0.1.0 build
> tsc -b
```

Exit code 0, identical clean result.

**`npm test` (root, `vitest run`, full monorepo suite), run twice for
stability, same method used by the registry-engineer and cli-implementer
increments:**

```
Test Files  12 failed | 112 passed (124)
     Tests  13 failed | 1206 passed (1219)
```

All 13 failures cross-checked against the cli-implementer increment's own
documented, `git stash`-bisected-against-`main` failure list (12 files,
same test names): `apps/cli/tests/start.test.ts` (1, missing
`axiom.spec/config/profiles.yaml` fixture), `packages/agents/tests/
catalog.test.ts` (1), `packages/doctor/tests/checks.test.ts` (1,
"repositorio Axiom real" integration test), `packages/model-routing/
tests/{assignments,loader,opencode-projection,resolver}.test.ts` (4),
`packages/project-resolution/tests/resolver.test.ts` (1 — the same
pre-existing `product.exists` case described above, confirmed via this
increment's own `git stash`/`git stash pop` bisection against clean
`main`, reproducing identically with none of this increment's changes
applied), `packages/skills/tests/catalog.test.ts` (1), `packages/
telemetry/tests/audit-trail-sink.test.ts` (1), `packages/toolchain/tests/
repair-add-gitignore.test.ts` (1), `packages/tui/tests/driver.test.ts`
(2). **This is the exact same 12-file, 13-test failure set the
cli-implementer increment already reported and bisected against clean
`main`** — no new failures, no missing failures, full consistency
confirmed by direct comparison of test names, not just counts.

**Scoped run, the two packages most directly involved
(`npx vitest run packages/project-resolution packages/config-validation`):**

```
✓ packages/config-validation/tests/validator.test.ts (16 tests)
 ❯ packages/project-resolution/tests/resolver.test.ts (17 tests | 1 failed)
   ❯ resolveProject > resuelve scopes con absolutePath y exists correcto (v1)
     → expected true to be false

Test Files  1 failed | 1 passed (2)
     Tests  1 failed | 32 passed (33)
```

`config-validation` unaffected (unchanged by this increment) — 16/16.
`project-resolution` — 16/17, the 1 failure being the pre-existing,
unrelated case described above (confirmed via `git stash` bisection
against clean `main` in this same session: identical single failure with
none of this increment's changes applied).

**Scoped run, `apps/cli` (consumer of `resolveProject`, to confirm no
downstream regression despite zero CLI files being touched):**

```
Test Files  1 failed | 30 passed (31)
     Tests  1 failed | 219 passed (220)
```

Identical to the cli-implementer increment's own reported baseline
(219/220, the 1 failure being the same pre-existing `start.test.ts`
case) — confirms this increment's resolver change is fully
backward-compatible with the CLI layer's existing v1 usage, with zero
CLI files modified.

## Result

Made `@axiom/project-resolution/src/resolver.ts`'s `resolveProject`
version-aware by delegating version-dispatch to
`@axiom/config-validation`'s `validateAxiomYamlContent`, reusing its
already-implemented `{valid, version, data}` discriminated union instead
of re-inventing version detection. Both `axiom.yaml` schemaVersion absent
(v1) and `schemaVersion: 2` documents now resolve correctly:

- v1: unchanged behavior (`project.name`/`project.mode`/`scopes`),
  verified against every pre-existing test.
- v2: `name` (top-level) becomes `ProjectResolution.name`, `mode`
  (top-level, independent axis) is normalized the same way as v1's
  `project.mode`, `paths` populates `ProjectResolution.scopes` via the
  same `buildScopeInfo` used for v1's `scopes` (direct renamed-successor
  per the schema-writer design).

**No `ProjectResolution` shape change was needed** — verified by
inspecting every real consumer (33 import sites across `apps/cli`,
`@axiom/doctor`, `@axiom/tool-routing`, `@axiom/isolation`, `@axiom/tui`)
before deciding; none branch on manifest version, so both versions
normalize into the identical existing external shape. Consequently
**zero CLI command files, zero `@axiom/doctor` files, and zero other
consumer files were touched** — this is a fully self-contained,
single-package fix plus its `package.json`/`tsconfig.json` dependency
wiring.

One real regression was found and fixed during this increment's own
validation, not shipped silently: naively treating any Zod-invalid
document as `invalid-config` collapsed the pre-existing `ambiguous`
status (document parses but lacks its identifying field) into
`invalid-config` (malformed document) — a real, user-facing status/error-
message regression for v1 documents missing only `project.name`. Fixed
by adding a `resolveInvalid` helper that re-parses loosely (non-Zod) to
restore the exact pre-fix `ambiguous`/`invalid-config` distinction for
v1, and extends the identical rule to v2.

Real `npm run typecheck`/`npm run build`/`npm test` were executed:
typecheck and build are 100% clean across the entire monorepo; the full
test suite's 13 pre-existing failures are the exact same 12-file,
13-test set already documented and `git stash`-bisected by the
cli-implementer increment (byte-for-byte the same test names, not just
the same count); `project-resolution` itself is 16/17 (the 1 failure
being a pre-existing, unrelated fixture bug in a v1 test, confirmed via
this increment's own `git stash` bisection); `config-validation`
(unchanged) is 16/16; `apps/cli` (consumer, unmodified) is 219/220,
identical to the cli-implementer increment's own reported baseline.

### Is it now safe to re-enable `schemaVersion: 2` emission in `init.ts`?

**Judgment: yes, the specific blocking mismatch that caused the
cli-implementer increment to revert `init.ts`'s `schemaVersion: 2`
emission is now resolved** — `resolveProject` correctly returns
`status: 'resolved'` for a `schemaVersion: 2` document (verified by this
increment's new tests), which means `join`'s `withProjectContext`/
`assertUnambiguous`, `projects.ts`'s `runProjectsJoin`, `repo.ts`'s
`runRepoAttach`, and `tui.ts`'s `--topology`/`--model-validate`/
`--components-show` modes will no longer see a v2-manifested project as
ambiguous or not-found.

This increment deliberately does **not** flip `init.ts` itself, per the
task's explicit instruction, for two reasons beyond just following
instructions: (1) re-enabling manifest emission is a behavior change to
what every new `axiom init` writes to disk, which deserves its own
focused verification pass (a real `axiom init` -> `axiom join` -> `axiom
repo attach` -> `axiom tui --topology` walkthrough against the actual
CLI binary, not just this package's unit tests) rather than being bundled
into a resolver-internals fix; (2) `@axiom/doctor`'s BC-001 special-case
(`rawConfig.project.mode === 'installed-multi-repo'`) will silently stop
firing for any newly-v2-`init`-ed project (v2 has no `.project` object),
which is a safe degrade (confirmed above, not a crash) but is a doctor-
side behavior change that the validator-reviewer role — the one
responsible for `@axiom/doctor` in this chain — should be aware of when
it does its own MC-001 work, rather than this increment silently
triggering it as a side effect.

## General spec integration

No integration into `Axiom.Spec/general-spec.md` was performed — that
file does not exist in this repo, consistent with all four prior
increments in this chain. `@axiom/project-resolution` is now
version-aware and unit-tested for both `axiom.yaml` versions, but the
INC-01 chain as a whole is still not fully closed end-to-end:
`@axiom/doctor`'s MC-001 still does the old binary pass/fail
(validator-reviewer has not run), `@axiom/tui`'s package-internal driver
still speaks v1 exclusively (INC-04, not part of this chain), and
`init.ts` still emits v1 manifests (this increment's own judgment above
says the resolver-side blocker is gone, but the actual re-enablement and
its own verification pass have not happened yet). Per the same reasoning
every prior increment in this chain gave, the correct point to integrate
stable knowledge into `general-spec.md`-equivalent documents is once the
full v2 shape is designed, implemented, wired through the CLI, resolvable
end-to-end (this increment), AND validated by `@axiom/doctor` — not
before all of those are true together.

## Closure rationale

`Status: closed`, unlike all four prior increments in this chain, because
this increment's own scope is a small, fully self-contained, single-
package fix with an unambiguous goal (make one function version-aware,
reusing an already-built version-dispatch mechanism) and every closure
rule is satisfied:

- Goal is clear and narrowly scoped (unlike the five INC-01 subagent
  roles, which were explicitly sequential slices of one larger,
  not-yet-complete parent increment).
- Acceptance criteria exist and are all met.
- Changes were implemented (not merely designed) and are fully tested.
- Real validation was executed and its output is reported above,
  cross-checked against the exact failure sets already documented by two
  prior increments in this chain.
- Review against intent and acceptance criteria was completed (see
  Result above): the specific blocking mismatch named by the
  cli-implementer increment is fixed, verified by new passing tests, with
  zero shape changes forced onto any other package.
- No stable-knowledge integration into `general-spec.md` was needed
  (explained above; the file does not exist yet and the broader INC-01
  chain is still open).
- The result (a version-aware resolver, zero consumer-shape changes, one
  incidental regression found-and-fixed before shipping) is documented
  clearly above.

This increment does not close INC-01 itself — INC-01 still has
validator-reviewer pending, and `init.ts`'s actual re-enablement of
`schemaVersion: 2` emission is still a distinct, unimplemented future
step (per this increment's own non-goals) — but this increment's own,
narrower goal is fully and verifiably done.

## Next step recommendation

1. **Immediate**: launch **validator-reviewer**, the final role in
   INC-01's original five-role subagent sequence, per the schema-writer
   design's exact MC-001 contract table (`Axiom.Spec/specs/increments/
   INC-20260702-registry-manifest-schema-v2-design/README.md`, "What
   MC-001 needs to do differently"):

   | `validateAxiomYamlContent` result | MC-001 status |
   |---|---|
   | `valid: true, version: 2` | `pass` |
   | `valid: true, version: 1` | `warn` (not `pass`, not `fail`) |
   | `valid: false, version: 'unknown'` | `fail` (unchanged) |

   Exact file/lines: `Axiom/packages/doctor/src/checks.ts`,
   `runManifestChecks` (MC-001, currently binary pass/fail on
   `validationResult.valid`). This increment did not change
   `@axiom/config-validation`'s `validateAxiomYamlContent` return shape,
   so validator-reviewer's inputs from the registry-engineer/schema-writer
   increments remain valid and current. Validator-reviewer should also be
   made aware of this increment's finding that BC-001
   (`runBoundaryChecks`, a *different* doctor check from MC-001) silently
   stops detecting `installed-multi-repo` for v2-shaped `axiom.yaml`
   documents (safe degrade, not a crash, but worth a deliberate decision
   rather than an unnoticed side effect once v2 documents start existing
   in the wild).

2. **Optional, small, separately verifiable**: re-enable `schemaVersion:
   2` emission in `apps/cli/src/commands/init.ts`'s `buildAxiomYaml`, now
   that this increment's fix removes the specific blocker that caused the
   cli-implementer increment to revert it. This increment's own judgment
   (above) is that the resolver-side blocker is resolved, but recommends
   this be its own explicit, independently verified step — including a
   real `axiom init` -> `axiom join` -> `axiom repo attach` -> `axiom tui
   --topology` walkthrough against the actual CLI binary — rather than
   being bundled into this resolver-internals fix, both per the task's
   explicit instruction and because it is a distinct, independently
   testable behavior change (what `init` writes to disk) from what this
   increment changed (how existing documents are read).
