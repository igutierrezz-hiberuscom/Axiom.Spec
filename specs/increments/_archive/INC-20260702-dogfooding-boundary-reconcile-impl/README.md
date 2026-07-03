# Increment: Dogfooding boundary enforcement — implementation (INC-23, cross-cutting)

Status: closed
Date: 2026-07-02

Closed by `INC-20260702-dogfooding-boundary-reconcile-final-validator`
(final validator-reviewer pass), which independently re-traced this
implementation against the design contract, re-ran all validation
(confirming 1641 passed / same 11 pre-existing unrelated failures), and
made the explicit `topology.yaml`-scope call requested in this
increment's own "Next step recommendation" (deferred, not part of
INC-23). See that increment's "INC-23 closure summary" for the
consolidated closure rationale.

## Goal

registry-engineer pass for INC-23 (roadmap's cross-cutting section), third
role in the chain: migration-engineer (done, audit) -> validator-reviewer
(done, design) -> **registry-engineer (this increment, implementation)** ->
validator-reviewer (final review, still pending). Implement
`runDogfoodingBoundaryChecks` (`DF-001`, category `dogfooding`) in
`Axiom/packages/doctor/src/checks.ts` exactly per the design contract in
`INC-20260702-dogfooding-boundary-reconcile-design/README.md`, wire it into
`runDoctorChecks`, add tests, and validate.

## Context

Predecessors (read in full before this implementation):
- `Axiom.Spec/specs/increments/INC-20260702-dogfooding-boundary-reconcile/README.md`
  (migration-engineer audit).
- `Axiom.Spec/specs/increments/INC-20260702-dogfooding-boundary-reconcile-design/README.md`
  (validator-reviewer design — the exact implementation contract this
  increment follows verbatim, not re-decided).

Roadmap: `Axiom.Spec/specs/increments/INC-20260702-axiom-redesign-roadmap/README.md`,
cross-cutting section, INC-23.

Key facts carried over, not re-litigated:
- `DF-001`, category `dogfooding`, one-directional (`code` must not depend
  on `sdd`/`spec`), data source `@axiom/topology`
  (`loadTopology`/`loadLocalBindings`/`resolveRepoPath`), no
  `runDoctorChecks` signature change.
- Skip conditions: unresolved project, `loadTopology` error,
  `mode === 'single-repo'`, empty `roleCodeRepositories`.
- Scan: `package.json` local/`file:`/relative dependencies + bounded
  `require`/`import` path-literal grep, excluding generated-path globs via
  `aggregateKnownGeneratedPathGlobs` (`./write-scope`).
- No `warn` state for this check — only `pass`/`fail`/`skip`.

## Scope

- Implement `runDogfoodingBoundaryChecks(resolution: ProjectResolution): DoctorCheck[]`
  in `Axiom/packages/doctor/src/checks.ts`, following the design's
  step-by-step algorithm and evidence string formats verbatim.
- Reuse `loadTopology`/`loadLocalBindings`/`resolveRepoPath` from
  `@axiom/topology` and `aggregateKnownGeneratedPathGlobs` from
  `./write-scope` (already imported in `checks.ts` for `WS-001`) — no
  reimplementation.
- Wire `runDogfoodingBoundaryChecks` into `runDoctorChecks` in
  `Axiom/packages/doctor/src/index.ts`, matching the exact pattern used for
  `WS-001` (import list, re-export list, category constant re-export,
  `checks` array).
- Add unit tests: skip-on-single-repo-mode, skip-on-empty-
  roleCodeRepositories, pass-when-clean, fail-on-package.json-dependency,
  fail-on-source-import, generated-path-glob exclusion.
- Real-world sanity check (manual, not part of the shipped suite): confirm
  what DF-001 would report if run against this workspace's own
  `Axiom`/`Axiom.SDD`/`Axiom.Spec` 3-repo layout.
- Run real validation: `npm run typecheck`, `npm run build`, `npm test`
  (full monorepo + scoped to `packages/doctor`).

## Non-goals

- No redesign of the check contract — the validator-reviewer's design
  (`DF-001`, category `dogfooding`, algorithm, evidence formats) is
  implemented as specified, not re-decided.
- No fix for pre-existing, unrelated test failures in the monorepo
  (confirmed baseline: 11 failed files / 11 failed tests / 1633 passed,
  same before and after this increment's changes — see Validation).
- No change to `runDoctorChecks`'s signature (confirmed unnecessary by the
  design; `resolution.rootPath` is sufficient).
- No implementation of the registry-v2 (`homeDir`-threaded) alternative
  data source — explicitly rejected by the design.
- Final validator-reviewer review pass is out of scope for this increment
  (next step in the chain, not this one).

## Acceptance criteria

- [x] `runDogfoodingBoundaryChecks` implemented in
      `Axiom/packages/doctor/src/checks.ts`, matching the design's check
      id (`DF-001`), category (`dogfooding`), description string, skip
      conditions (unresolved project, `loadTopology` error,
      `mode === 'single-repo'`, empty `roleCodeRepositories`), two-pass
      scan (`package.json` dependencies + bounded source path-literal
      grep excluding generated-path globs), and evidence string formats
      for `pass`/`fail`/`skip`.
- [x] Wired into `runDoctorChecks` in
      `Axiom/packages/doctor/src/index.ts`, following the `WS-001` wiring
      pattern (import, re-export, category re-export, checks array).
- [x] Tests added covering: skip-on-single-repo-mode,
      skip-on-empty-roleCodeRepositories, pass-when-clean,
      fail-on-package.json-dependency-violation,
      fail-on-source-import-violation, generated-path-glob exclusion
      (plus skip-on-unresolved-project and a `runDoctorChecks` integration
      test, following the existing `topology.test.ts`/`write-scope.test.ts`
      structure).
- [x] Real-world sanity check performed and documented: confirms whether
      DF-001, if run against this workspace's own `Axiom` repo, would
      report `pass` or `skip` today, with the concrete reason.
- [x] `npm run typecheck`, `npm run build`, and `npm test` executed with
      real output; results compared against a confirmed baseline (via
      `git stash` bisection), not assumed.
- [x] No unrelated files modified.

## Open questions

None — the design contract left no open decisions for this role to make.

## Assumptions

- The design's assumption that `resolveRepoPath`/`loadLocalBindings` are
  `homeDir`-free and already exported from `@axiom/topology` is correct —
  re-confirmed directly against `Axiom/packages/topology/src/index.ts` and
  `loader.ts` in this pass (both are re-exported from the package barrel).
- `aggregateKnownGeneratedPathGlobs` (from `./write-scope`) is reusable
  as-is; confirmed it is already imported in `checks.ts` for `WS-001` and
  exported from `Axiom/packages/doctor/src/write-scope.ts`.
- The design's "minimal sufficient scan" (no AST parse, no bundler alias
  resolution, no transitive `node_modules` walk) is implemented literally:
  the source-level pass is a per-line regex match against
  `require(...)`/`from '...'` literals, resolved as plain relative/
  absolute paths.

## Implementation notes

### Files changed

- `Axiom/packages/doctor/src/checks.ts`:
  - Added `loadLocalBindings`, `resolveRepoPath`, and the `RepoRef` type
    to the existing `@axiom/topology` import.
  - Added `CATEGORY_DOGFOODING = 'dogfooding'` category constant.
  - Added `runDogfoodingBoundaryChecks(resolution: ProjectResolution): DoctorCheck[]`
    (the `DF-001` check) plus its private helpers: `readPackageJson`,
    `isLocalDependencySpecifier`, `extractLocalDependencyPath`,
    `isInsideBoundary`, `scanPackageJsonForBoundaryViolations`,
    `looksLikePathLiteral`, `buildGeneratedPathPrefixes`,
    `isUnderGeneratedPath`, `walkSourceFiles`,
    `scanSourceFilesForBoundaryViolations`, and
    `collectWorkspacePackageJsonDirs` (bounded, non-recursive
    `workspaces` glob resolution, supporting the `apps/*`/`packages/*`/
    `packages/*/*` shape this monorepo itself uses, per the design's own
    "bounded to that repo's own workspace glob" scope note).
- `Axiom/packages/doctor/src/index.ts`:
  - Added `runDogfoodingBoundaryChecks` to the import list and the
    re-export list from `./checks`.
  - Added `export const CATEGORY_DOGFOODING = 'dogfooding';` alongside the
    file's other `CATEGORY_*` re-exports.
  - Added `...runDogfoodingBoundaryChecks(resolution)` to the `checks`
    array inside `runDoctorChecks`.
- `Axiom/packages/doctor/tests/dogfooding.test.ts` (new file): 8 tests —
  skip-on-not-found, skip-on-single-repo, skip-on-empty-
  roleCodeRepositories, pass-on-clean-multi-repo-fixture,
  fail-on-package-json-dependency (evidence contains `depende de sdd` and
  the dependency name), fail-on-source-import (evidence contains
  `depende de spec` and the offending `file:line`), generated-path-glob
  exclusion (a `.axiom/cache/**` file with a `require(...)` that would
  otherwise resolve into `sdd-repo` is correctly excluded — `pass`, not
  `fail`), and a `runDoctorChecks` integration test confirming the
  `dogfooding` category and `DF-001` id appear in the full report.

### Design deviations / notes for the final validator-reviewer pass

- **`collectWorkspacePackageJsonDirs`**: the design's step 1 says "every
  workspace member's `package.json` the repo declares via its own
  `workspaces` field ... bounded to that repo's own workspace glob, not a
  recursive `node_modules` walk." This was implemented with a simplified
  glob resolver that only supports a single `*` wildcard as a path
  segment (e.g. `packages/*`, `apps/*`, `packages/*/*`) — sufficient for
  this monorepo's own `workspaces` declaration and for any project using
  the same common convention, but it is NOT a full glob-matching
  implementation (no brace expansion, no `**`, no negation). This mirrors
  the design's own explicit "minimal sufficient scan, not the most
  thorough possible one" directive. Flagged for the final
  validator-reviewer pass in case a project's `workspaces` field uses a
  glob shape this simplified resolver does not support — such a project
  would silently scan fewer `package.json` files than expected (fail-open,
  not fail-crash), consistent with every other check's fail-open posture
  on partial/unusual input.
- **`buildGeneratedPathPrefixes`**: the design references
  `aggregateKnownGeneratedPathGlobs` directly but does not specify a glob
  matcher implementation for the source-scan pass. Implemented here as a
  prefix comparison (glob's segment before the first `*` treated as a
  literal directory prefix) rather than a full glob engine — sufficient
  for the actual globs `aggregateKnownGeneratedPathGlobs` currently
  produces (adapter-specific single-file paths like `.opencode/AGENTS.md`,
  plus `.axiom/cache/**`), all of which have a clean literal prefix before
  their first/only wildcard. Flagged as a documented simplification, not a
  silent gap.

## Real-world sanity check: this workspace's own `Axiom` repo

Per the task's explicit instruction, this is a manual verification step,
not a shipped test. Ran `loadTopology(rootPath)` and the full
`runDogfoodingBoundaryChecks(resolution)` pipeline directly (via a small
Node script against the built `dist/`) with `rootPath` /
`process.cwd()` set to `C:\repos\Axiom Workspace\Axiom`:

```json
// loadTopology(Axiom repo root)
{
  "ok": true,
  "value": {
    "schemaVersion": 1,
    "mode": "single-repo",
    "sddRepo": { "id": "sdd-repo", "ref": "C:\\repos\\Axiom Workspace\\Axiom", ... },
    "specRepo": { "id": "spec-repo", "ref": "C:\\repos\\Axiom Workspace\\Axiom", ... },
    "roleCodeRepositories": [],
    "assignments": [],
    "qaLane": "inline"
  }
}
```

`Axiom/axiom.spec/config/topology.yaml` does not exist in this repo, so
`loadTopology` falls back to `defaultSingleRepoManifest` (per
`tryLoadTopologyHint`, which also finds no `installed-multi-repo` mode hint
in `Axiom/axiom.yaml` — that file has no `project.mode` nor a top-level
`mode` field at all; it is a purely descriptive dogfooding manifest, not
the schema `loadTopology`/`resolveProject` actually parses for mode
detection). This alone would make DF-001 **skip** at step 3 of the design's
algorithm (`manifest.mode === 'single-repo'`).

Going one layer deeper: `resolveProject(process.cwd())` run from
`Axiom/` in this exact workspace actually returns
`status: 'ambiguous'`, not `'resolved'` — confirmed live, and also
independently confirmed as a pre-existing, unrelated condition (see
Validation: the same `resolveProject(process.cwd())` ambiguity already
causes one pre-existing failing assertion in
`packages/doctor/tests/checks.test.ts`, present on the untouched baseline
via `git stash`). Because of this, DF-001 actually skips at step 1 of the
algorithm (`resolution.status !== 'resolved'`), before even reaching the
single-repo-mode check:

```json
[
  {
    "id": "DF-001",
    "category": "dogfooding",
    "description": "El repo de código no depende físicamente del repo sdd ni del repo spec (dogfooding boundary)",
    "status": "skip",
    "evidence": "No se puede verificar: proyecto en estado \"ambiguous\"."
  }
]
```

**Conclusion, confirming the task's anticipated nuance (and refining it)**:
DF-001 does **not** produce a `pass` result if run today, inside this exact
workspace, against `axiom doctor`'s real CWD-based resolution. It **skips**
— but for two independent, stacked reasons, not just one:

1. `Axiom/axiom.spec/config/topology.yaml` does not exist, so
   `loadTopology` defaults to `mode: 'single-repo'` (the hypothesized
   reason in the task).
2. `resolveProject(process.cwd())`, run from inside `Axiom/` in this
   workspace, itself resolves to `status: 'ambiguous'` rather than
   `'resolved'` — a pre-existing, unrelated condition of this specific
   Windows workspace's directory layout (already surfaced by one
   pre-existing failing test in `checks.test.ts`), which triggers DF-001's
   very first skip condition regardless of (1).

This is **not a test failure or a defect in DF-001** — it is exactly the
fail-open-to-skip behavior the design specifies for both "project not
resolved" and "single-repo mode," applied correctly to real,
environment-specific input. It confirms the design's own framing: DF-001
is built for the target multi-repo model (a project with a real
`topology.yaml` declaring `mode: multi-repo` and `roleCodeRepositories`),
not for immediate exercise against this orchestration's own current
`Axiom` product repo, which — as far as `@axiom/topology`'s own resolution
logic is concerned — does not (yet) declare itself as a multi-repo
project via the mechanism DF-001 reads. The migration-engineer audit's "no
violation found" conclusion (section 2 of the audit, a manual grep-based
finding) remains true independently of this; it simply is not the same
thing as "DF-001 would report `pass` if run here today," which it would
not — it would `skip`.

## Validation

Baseline confirmed via `git stash` bisection before implementing (matches
the task's stated expectation): `npm test` on the untouched working tree
reports **11 failed test files / 11 failed tests / 1633 passed** (150 test
files total before this increment's new file). The 11 pre-existing
failures span `packages/doctor/tests/checks.test.ts` (1),
`packages/topology/tests/*`, `packages/project-resolution/tests/
resolver.test.ts`, `packages/skills/tests/catalog.test.ts`,
`packages/telemetry/tests/audit-trail-sink.test.ts`,
`packages/toolchain/tests/repair-add-gitignore.test.ts`, and others —
none touch `@axiom/doctor`'s check logic added here, and none changed
after this increment's changes.

Commands run, in order, after implementation:

1. `npm run typecheck` (root, `tsc -b`) — **clean, no errors.**
2. `npx vitest run packages/doctor/tests/dogfooding.test.ts` — **8/8
   passed.**
3. `npx vitest run packages/doctor` (scoped) — **156 passed / 1 failed**
   (`checks.test.ts`'s pre-existing `resolveProject(process.cwd())`
   ambiguity assertion, independently re-confirmed via `git stash` to be
   present on the untouched baseline with the identical failure message —
   not caused by this increment).
4. `npm run build` (root, `tsc -b`) — **clean, no errors.**
5. `npm test` (root, full monorepo) — **11 failed test files / 11 failed
   tests / 1641 passed** (161 test files total). Same 11 failing files/
   tests as the confirmed baseline; the increase from 1633 to 1641 passed
   is exactly this increment's 8 new `dogfooding.test.ts` tests. No
   regression introduced.
6. Manual sanity check (not part of the automated suite): confirmed via a
   direct Node script against the built `dist/` that `runDogfoodingBoundaryChecks`,
   run against this workspace's own `Axiom` repo root, correctly `skip`s
   (see "Real-world sanity check" above) — not a `pass`, and not a `fail`.
7. Confirmed the built `packages/doctor/dist/index.js` actually contains
   `runDogfoodingBoundaryChecks` after `npm run build` (not a stale dist
   artifact), before running the manual sanity check in step 6.

## Result

`DF-001` (`runDogfoodingBoundaryChecks`, category `dogfooding`) is
implemented in `Axiom/packages/doctor/src/checks.ts` exactly per the
validator-reviewer's design contract: same check id, category, description
string, skip-condition ordering, two-pass scan (package.json local
dependencies + bounded source path-literal grep excluding known generated
paths), single aggregated pass/fail result, and evidence string formats.
Wired into `runDoctorChecks` in `Axiom/packages/doctor/src/index.ts`
following the exact `WS-001` precedent (import, re-export, category
re-export, checks array). 8 new tests added and passing, covering every
scenario the design's own "next step recommendation" asked for, plus a
`runDoctorChecks` integration test. `npm run typecheck` and `npm run
build` are clean. `npm test` shows the exact same pre-existing 11 failed
files/11 failed tests as the confirmed baseline, plus 8 new passing tests
from this increment — no regression. The manual real-world sanity check
against this workspace's own `Axiom` repo confirms DF-001 would `skip`
(not `pass`) if run here today, for two independent reasons (missing
`topology.yaml` defaulting to `single-repo` mode, and a pre-existing,
unrelated `resolveProject(process.cwd())` ambiguity in this specific
Windows workspace) — this is documented as an expected nuance of DF-001
targeting the future multi-repo model, not a defect.

## General spec integration

Per the design increment's own stated convention (confirmed again here,
not re-decided): integration into `Axiom.Spec/general-spec.md` happens
once the chain closes with real implemented AND verified code — i.e.
after the final validator-reviewer pass, not at this implementation step.
This increment's status remains `pending`, consistent with the roadmap's
own explicit 4-role sequence (migration-engineer -> validator-reviewer ->
**registry-engineer** -> validator-reviewer). No integration was performed
here; the final validator-reviewer increment should add a "Dogfooding
boundary check" section to `general-spec.md` documenting: the check's
final id/category (`DF-001`/`dogfooding`), the role-parameterized design
(topology-based, not name-hardcoded), the audit's "no violation found
today" baseline finding (migration-engineer's grep-based conclusion), and
this implementation's real-world sanity-check nuance (DF-001 currently
`skip`s rather than `pass`es against this workspace's own `Axiom` repo,
because that repo does not declare a `topology.yaml`/multi-repo mode the
check's data source can read — not because a violation exists).

## Next step recommendation

Hand off to the final **validator-reviewer** pass with these inputs:

1. Confirm `runDogfoodingBoundaryChecks`'s implementation in
   `Axiom/packages/doctor/src/checks.ts` matches the design contract
   verbatim (check id, category, skip/fail/pass conditions, evidence
   formats) — this increment's own review found no deviation from the
   contract other than the two documented, design-consistent
   simplifications noted above (`collectWorkspacePackageJsonDirs`'s
   single-`*`-segment glob support, `buildGeneratedPathPrefixes`'s
   literal-prefix glob matching).
2. Confirm the real-world sanity-check finding (DF-001 `skip`s, not
   `pass`es, against this workspace's own `Axiom` repo today, for the two
   stacked reasons documented above) is an acceptable/expected outcome
   for this increment's closure, or decide whether a follow-up increment
   should give this workspace's own `Axiom`/`Axiom.SDD`/`Axiom.Spec` setup
   a real `topology.yaml` (`mode: multi-repo`) so DF-001 can actually
   exercise a `pass` here — explicitly out of scope for this increment
   (would require its own increment: writing `Axiom/axiom.spec/config/
   topology.yaml`, which does not exist today).
3. Run `npm run doctor` (or `axiom doctor`) once a real `topology.yaml`
   exists in some test/reference project, to observe `DF-001` reporting a
   live `pass`/`fail`/`skip` end-to-end through the CLI (not just via the
   unit tests and the manual Node-script sanity check performed here).
4. Once confirmed, integrate the "Dogfooding boundary check" section into
   `Axiom.Spec/general-spec.md` and close the INC-23 chain.
