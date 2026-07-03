# Increment: Write-scope validation implementation (cli-implementer)

Status: pending
Date: 2026-07-02

## Goal

Execute the **cli-implementer** step of INC-08 of the parent roadmap
(`Axiom.Spec/specs/increments/INC-20260702-axiom-redesign-roadmap/README.md`,
Phase C), per the concrete brief handed off by the migration-engineer
audit (`Axiom.Spec/specs/increments/INC-20260702-write-scope-validation-reconcile/README.md`,
"Next step recommendation"): implement real write-scope validation
(addendum §9) against `Axiom`'s already-landed `allowedWriteScope`
schema (INC-06), exposed both as a new `axiom validate changes` CLI
command and a new `@axiom/doctor` `WS-001` check, sharing one
comparison function.

## Context

Parent chain read in full before this implementation:

1. Parent roadmap — `Axiom.Spec/specs/increments/INC-20260702-axiom-redesign-roadmap/README.md`
   (INC-08 entry, Phase C).
2. Migration-engineer audit (this increment's direct predecessor) —
   `Axiom.Spec/specs/increments/INC-20260702-write-scope-validation-reconcile/README.md`.
   Confirmed: `allowedWriteScope`/`targetRepos`/`taskType` schema already
   landed by INC-06; no git-diff utility, no glob library, no
   `axiom validate changes` command, no write-scope doctor check existed;
   all 5 addendum §9 detection cases are derivable from `git diff` +
   `allowedWriteScope` + topology's repo-role resolution + an
   aggregation of generated/cache paths, with no new persistent tracking
   state.
3. INC-06 impl closure (schema precedent) —
   `Axiom.Spec/specs/increments/INC-20260702-increment-bug-plan-commands-reconcile-impl/README.md`.
4. INC-07 impl closure (direct structural precedent: `index-cmd.ts`'s
   `run*`/`register*` split over a shared `@axiom/workflow` primitive,
   also consumed by a new doctor check) —
   `Axiom.Spec/specs/increments/INC-20260702-index-rebuild-reconcile-impl/README.md`
   (if present) / the audit document for the same increment.

Code read directly in `Axiom` (not from memory/summary) before
implementing: `packages/workflow/src/artifact-store.ts` (full),
`apps/cli/src/commands/index-cmd.ts` (full, structural precedent),
`apps/cli/src/commands/axiom-plan.ts` (full, flag conventions —
`--project`/`--id`/`--target-repo`/`--task-type`, `requiredOption`,
`--json`), `apps/cli/src/commands/axiom-role.ts` (full — active-plan
lookup mechanism: `loadWorkflowState(projectRoot, 'plan')`, state
`'plan-approved'`, `vars.planId`), `packages/doctor/src/checks.ts`
(full — `DoctorCheck` shape, `pass/fail/warn/skip` helpers,
`runArtifactIndexChecks`/`IX-001` as the closest precedent for a
`@axiom/workflow`-backed check), `packages/doctor/src/index.ts` (full —
`runDoctorChecks` aggregation), `packages/topology/src/types.ts` +
`loader.ts` (full — `TopologyManifest`/`RepoRef`/`RoleAssignment`
shape, `sdd-repo`/`spec-repo` role vocabulary), `packages/installer/src/registry.ts`
(full — `GENERATED_FILES_BY_TARGET`, the only real source of
generated-file declarations in the monorepo; `document-bootstrap`
references it but does not declare a second list), `apps/cli/src/commands/self-update.ts`
(the `spawnSync` + dependency-injection precedent for shelling out to
an external process in a testable way), `apps/cli/src/index.ts` (full,
registration order convention).

## Scope

- Add `minimatch` as a new dependency of `@axiom/workflow` (no
  transitive glob/minimatch dependency existed anywhere in the
  monorepo's `package-lock.json`, confirmed by direct grep before
  adding).
- Implement `validateWriteScope` in `@axiom/workflow`
  (`packages/workflow/src/write-scope.ts`): the single shared
  comparison primitive, covering all 5 addendum §9 cases. Pure
  function — no filesystem/git access, no `@axiom/topology` dependency
  (callers resolve repo roles and supply them).
- Implement topology-aware / git-shelling helpers
  (`packages/doctor/src/write-scope.ts`, in `@axiom/doctor` — not
  `@axiom/workflow` — because they depend on `@axiom/topology` and
  `@axiom/installer`, neither of which `@axiom/workflow` depends on
  today): `runGitDiff` (shell out via `spawnSync`, injectable),
  `aggregateKnownGeneratedPathGlobs` (aggregates
  `GENERATED_FILES_BY_TARGET` + `.axiom/cache/**`), `buildRepoChangeSets`
  (topology repo enumeration + role resolution + git-diff per repo).
- Add `@axiom/doctor`'s `WS-001` check (`runWriteScopeChecks` in
  `checks.ts`), reusing the same active-plan lookup mechanism
  `axiom-role.ts` already uses (`loadWorkflowState(projectRoot, 'plan')`,
  state `'plan-approved'`), skipping gracefully when no plan is active.
- Add the CLI command `axiom validate changes --project <id> --plan
  <planId>` (`apps/cli/src/commands/validate-changes.ts`), reusing the
  `@axiom/doctor` helpers (no duplicated diff-vs-scope logic).
- Register the new CLI command in `apps/cli/src/index.ts`.
- Add tests for the shared comparison function (all 5 cases, both
  violation and clean paths), the new CLI command, and the new doctor
  check (including its skip-when-no-active-plan behavior, using a real
  ephemeral git repo for the pass/fail cases).
- Run `npm run typecheck`, `npm run build`, `npm test` (full monorepo)
  and record results against a freshly-measured baseline.

## Non-goals

- No hand-rolled glob engine — a small, standard library
  (`minimatch`) is used instead, per the audit's own recommendation and
  `Axiom.SDD/AGENTS.md`'s minimalism (a hand-rolled matcher for the
  `**`/`*` vocabulary actually used would be exactly the kind of
  unnecessary complexity the guardrails warn against).
- No new persistent tracking state (no run-ledger of "which files did
  this workflow run touch") — confirmed unnecessary by the audit's
  Finding 4 and re-confirmed during implementation.
- No resolution of real multi-repo `LocalBindings`/absolute paths for
  `roleCodeRepositories` beyond what `topology.yaml`'s `ref` already
  encodes — proper multi-repo local-binding resolution for write-scope
  validation is a P1 concern once a real multi-repo project exists with
  non-trivial `topology-bindings.yaml` content; MVP validated against
  the current single-repo-default topology shape.
- No new "which plan is active" mechanism — reused
  `axiom-role.ts`'s existing `loadWorkflowState(projectRoot, 'plan')` +
  `'plan-approved'` lookup verbatim.
- No change to `PlanMetadata`/`AllowedWriteScopeEntry`'s schema shape
  (already landed by INC-06, confirmed sufficient by the audit).
- No fix for the pre-existing, unrelated `apps/cli/dist/index.js`
  runtime module-resolution issue discovered during manual smoke-testing
  (see Validation section) — it affects commands unrelated to this
  increment (e.g. `axiom index rebuild`) identically and predates this
  work; out of scope to fix here.

## Acceptance criteria

- [x] `minimatch` added as a dependency of `@axiom/workflow` only
      (confirmed via `package.json`/`package-lock.json` diff — no other
      package needed it directly).
- [x] `validateWriteScope` implemented in `@axiom/workflow`, covering:
      outside-scope, undeclared-repo, unexpected-sdd-change,
      generated-cache-tampering, inconsistent-metadata — with tests for
      each case's violation AND clean path.
- [x] `axiom validate changes --project <id> --plan <planId>`
      implemented and registered in the CLI, printing a pass/fail
      report and exiting 1 on any violation.
- [x] `@axiom/doctor`'s `WS-001` check implemented, skipping gracefully
      (not failing) when no plan is currently approved/active, reusing
      `axiom-role.ts`'s existing active-plan lookup.
- [x] Tests added/passing for: the shared comparison function (15
      tests, all 5 cases × violation/clean), the CLI command (6 tests),
      the doctor check (6 tests, including 2 real-git-repo pass/fail
      cases and the skip-when-no-active-plan case).
- [x] `npm run typecheck` and `npm run build` pass clean (0 errors)
      across the full monorepo.
- [x] `npm test` (full monorepo) shows no regression: same 12
      failed files / 13 failed tests as the freshly-measured baseline
      (all pre-existing, unrelated — see Validation), plus 27 new
      passing tests from this increment's 3 new test files.

## Open questions

None blocking. One design decision made explicitly during
implementation (flagged as open by the audit) is documented below.

## Assumptions

- **Base-ref for "real changes"**: `git diff --name-only HEAD` (tracked,
  modified-but-uncommitted files) UNION `git status --porcelain`
  (untracked files) — i.e. the working tree's current diff relative to
  `HEAD`, not the plan's approval commit/branch-point (no such ref is
  tracked anywhere today; introducing one would be new persistent state,
  which the audit's Finding 4 concluded is unnecessary). This mirrors
  what a developer/agent sees via a plain `git status` right before
  committing — the natural point to run this validation as a
  pre-commit-style gate.
- A repo path that is not a git repository (or where `git` is
  unavailable) is treated as "no changes detected" (`[]`), not an
  error — silence is the correct signal there, not a validation
  failure.
- `WS-001`'s "active plan" is exactly the plan `axiom-role.ts`'s
  `blockRoleStartIfPlanIsNotApproved` safety gate already recognizes:
  the `plan` workflow's state record when `state === 'plan-approved'`,
  via `vars.planId`. No second "active plan" concept was invented.
- `EXPECTED_PLAN_STATUSES` for write-scope validation is `['plan-approved']`
  only — the same threshold `axiom-role.ts` already treats as "plan in
  flight." A plan in any other status triggers an
  `inconsistent-metadata` violation from `validateWriteScope` itself
  (case 5), not a separate mechanism.
- `GENERATED_FILES_BY_TARGET` (from `@axiom/installer/registry.ts`) is
  the only real source of generated-file declarations in the monorepo
  today — the audit's brief mentioned "all 6 adapters'
  `GENERATED_FILES_BY_TARGET` lists" as if each adapter package declared
  its own; direct grep during implementation confirmed there is exactly
  **one** such list (in `@axiom/installer`, covering all 8 adapter
  targets including `litellm`'s empty array), not six separate lists.
  `document-bootstrap` references this same list in comments; it does
  not declare a second one. This is a minor factual correction to the
  brief, not a design change — the aggregation still happens exactly as
  recommended, just from one source instead of several.
- `.axiom/cache/**` is included in the known-generated-paths glob list
  even though no real files exist under that path anywhere in the
  monorepo yet (confirmed by grep) — per the brief's explicit
  instruction to future-proof this cheaply rather than skip it.

## Implementation notes

### Package/dependency changes

- `packages/workflow/package.json`: added `"minimatch": "^9.0.5"` (npm
  resolved to `^9.0.9` in `package-lock.json`; both satisfy the same
  intent — the latest v9.x, chosen over v10.x because v10 requires
  ESM-only resolution while the monorepo's `tsconfig.base.json` uses
  `module: Node16`/CJS-style output; v9 is the last major with a plain
  CJS named export (`{ minimatch }`) that resolves cleanly under the
  existing `esModuleInterop` setup).
- `packages/doctor/package.json`: added `"@axiom/core": "*"` and
  `"@axiom/installer": "*"` as new dependencies (needed for `AXIOM_DIR`
  and `GENERATED_FILES_BY_TARGET` respectively). Confirmed no circular
  dependency: `@axiom/installer` depends on `@axiom/capability-model`,
  `@axiom/core`, `@axiom/install-profiles`, `@axiom/persistence`,
  `@axiom/project-resolution` — none of which depend on `@axiom/doctor`.
- `packages/doctor/tsconfig.json`: added the corresponding TS project
  references + `paths` entries for `@axiom/core` and `@axiom/installer`
  (required by `tsc -b`'s project-reference build mode, separate from
  `package.json`'s `dependencies`).

### `validateWriteScope` (`packages/workflow/src/write-scope.ts`)

Pure function, no filesystem/topology/git access. Signature:

```ts
validateWriteScope(args: {
  plan: Pick<PlanMetadata, 'status' | 'targetRepos' | 'allowedWriteScope'>;
  repoChanges: ReadonlyArray<RepoChangeSet>; // { repoId, roles, changedPaths }
  sddRoles: ReadonlyArray<string>;
  knownGeneratedPathGlobs: ReadonlyArray<string>;
  expectedStatuses: ReadonlyArray<string>;
}): { violations: ReadonlyArray<WriteScopeViolation>; ok: boolean }
```

Design decision: **caller-supplied `sddRoles` and `roles` per repo**,
instead of this module depending on `@axiom/topology` directly. This
keeps `@axiom/workflow` dependency-light (unchanged dependency graph
besides `minimatch`) and keeps the comparison logic testable with plain
fixtures, no topology/filesystem fixtures needed. The
topology-resolution work happens one layer up, in `@axiom/doctor`'s
`buildRepoChangeSets`.

Glob matching uses `minimatch(path, pattern, { dot: true })` after
normalizing Windows path separators (`\` → `/`) — globs are always
POSIX-style even when `git status --porcelain` on Windows might emit
backslash-separated paths in some configurations (defensive
normalization, verified via a dedicated test).

The 5 cases map to violation kinds: `outside-scope`, `undeclared-repo`,
`unexpected-sdd-change` (same underlying mechanism as
`undeclared-repo`, differentiated only by whether the repo's roles
intersect `sddRoles` — confirmed by the audit's Finding 4.3 that this
is a labeling nicety, not new infrastructure), `generated-cache-tampering`
(checked independently of scope-membership — a generated file changed
manually is suspicious even if the repo is otherwise in scope),
`inconsistent-metadata` (two sub-cases: `allowedWriteScope[].repo` not
present in `targetRepos`, and `plan.status` outside
`expectedStatuses`).

### Topology-aware helpers (`packages/doctor/src/write-scope.ts`)

- `runGitDiff(repoPath)`: shells out to `git diff --name-only HEAD` +
  `git status --porcelain` via `spawnSync` (same precedent as
  `apps/cli/src/commands/self-update.ts`'s `runInstallGlobal`).
  Injectable (`GitDiffRunner` type) so both the CLI command and the
  doctor check tests can supply a fake without a real git repo, and so
  the doctor check's real-git-repo tests can exercise actual `git init`
  + untracked-file scenarios.
- `aggregateKnownGeneratedPathGlobs()`: `Object.values(GENERATED_FILES_BY_TARGET).flat()`
  plus `${AXIOM_DIR}/cache/**` appended.
- `buildRepoChangeSets(projectRoot, manifest, gitDiff)`: enumerates
  `manifest.sddRepo`, `manifest.specRepo`, and
  `manifest.roleCodeRepositories`, resolves each repo's role labels
  from `manifest.assignments` (plus the implicit `sdd-repo`/`spec-repo`
  role for the two fixed slots), resolves each repo's path (absolute
  `ref` used as-is; relative `ref` joined with `projectRoot` — matching
  `@axiom/topology`'s own `resolveRepoPath` heuristic, without pulling
  in `LocalBindings` resolution, which is a P1 concern per Non-goals),
  and calls `gitDiff` once per repo.

Exported from `@axiom/doctor`'s barrel (`SDD_ROLES`,
`EXPECTED_PLAN_STATUSES`, `runGitDiff`, `aggregateKnownGeneratedPathGlobs`,
`buildRepoChangeSets`, `GitDiffRunner`) so the CLI command imports them
rather than duplicating.

### `WS-001` doctor check (`packages/doctor/src/checks.ts`)

`runWriteScopeChecks(resolution)`. Skip conditions, in order: project
not resolved; no `plan` workflow state record, or state !==
`'plan-approved'`; `vars.planId` missing/empty; plan's `metadata.yml`
fails to load; `topology.yaml` fails to load. Each skip reason is
distinct and surfaced in `evidence` (mirrors the granular skip
messaging style of `runGatewayStateChecks`/`checkPlanIsApproved`).
Once all preconditions are met, calls `validateWriteScope` exactly as
the CLI command does and maps `ok`/violations to `pass`/`fail`.

Registered in `runDoctorChecks` (`packages/doctor/src/index.ts`), after
`runArtifactIndexChecks` (IX-001), consistent with the existing
end-of-list convention for the newest check categories.

### `axiom validate changes` CLI command (`apps/cli/src/commands/validate-changes.ts`)

Flag conventions confirmed against `axiom-plan.ts`/`axiom-role.ts`
before writing: `--project <project>` (informative label; the actual
resolution root is `process.cwd()`, same convention every other CLI
command in this codebase uses — no command in the existing surface
resolves an arbitrary `--project`-named path), `--plan <planId>`
(`requiredOption`, matching `--id <planId>` semantics from
`axiom-plan.ts`'s `approve` subcommand but named `--plan` per the
addendum's literal invocation `axiom validate changes --project X
--plan PLAN-ID`), `--json` (matching every other command's convention).
Registered as `validate` (top-level) → `changes` (subcommand), since
addendum §9 explicitly names the two-word invocation `validate
changes` and no `validate` top-level command existed to collide with.

`run*`/`register*` split (INC-07's `index-cmd.ts` precedent):
`runValidateChanges` takes an injectable `gitDiff` (defaults to the
real `runGitDiff` from `@axiom/doctor`), never touches `commander`/
`process.exit`, and is what tests call directly.

Registered in `apps/cli/src/index.ts` immediately after `registerIndex`
(INC-07) and before `registerTui`, consistent with the existing
ADD-AT-THE-END convention for new top-level commands in this file.

## Validation

Freshly-measured baseline (re-run before any code change in this
session, via `npm test` at the current uncommitted working-tree state —
the repository has substantial pre-existing uncommitted work from prior
increments; this baseline reflects that exact state, not a clean
checkout):

```
Test Files  12 failed | 121 passed (133)
Tests       13 failed | 1320 passed (1333)
```

This matches the increment brief's stated baseline exactly (12/13/1320/1333).

Commands run after implementation:

- `npm run typecheck` (`tsc -b`, full monorepo): **0 errors.**
- `npm run build` (`tsc -b`, full monorepo): **0 errors.**
- `npm test` (`vitest run`, full monorepo), run twice for consistency:

  ```
  Test Files  12 failed | 124 passed (136)
  Tests       13 failed | 1347 passed (1360)
  ```

  Same **12 failed files / 13 failed tests** as the baseline — i.e. no
  regression. The **specific** failing test names differ slightly
  between the baseline run and the post-implementation runs (e.g.
  `packages/tui/tests/driver.test.ts`'s exact failing scenario names
  shifted, and a couple of "real repo/real axiom.spec"-dependent tests
  like `packages/doctor/tests/checks.test.ts`'s "repositorio Axiom
  real" and `packages/model-routing/tests/*`'s "real policy from
  axiom.spec" appeared/disappeared across runs) — this is pre-existing
  flakiness in tests that depend on the actual, currently-uncommitted,
  in-flux state of `axiom.spec`/repo paths on disk, confirmed
  unrelated to this increment's changes by running the suite twice in
  a row with zero code changes in between and observing the same
  count-but-different-names pattern. The **count** (12/13) is stable
  and matches the pre-implementation baseline in both files-failed and
  tests-failed totals.
  - Delta: **+27 passing tests** (1347 − 1320), **+3 test files**
    (136 − 133) — exactly the 3 new test files added by this increment
    (`packages/workflow/tests/write-scope.test.ts`: 15 tests,
    `apps/cli/tests/validate-changes.test.ts`: 6 tests,
    `packages/doctor/tests/write-scope.test.ts`: 6 tests). Verified by
    running these 3 files in isolation: `27 passed (27)`, `3 passed (3)`
    files, 0 failed.
- Manual smoke test: `node apps/cli/dist/index.js validate changes
  --help` was attempted and failed with a pre-existing,
  unrelated runtime error (`Cannot find module
  '../../src/commands/_shared.js'` inside the compiled `dist/`
  output). Confirmed this is NOT caused by this increment's changes by
  running the identical smoke test against an untouched command
  (`node apps/cli/dist/index.js index rebuild --help`), which fails
  with the exact same error. This is a pre-existing `dist/` build
  artifact/module-resolution issue in the working tree (likely a stale
  or partially-rebuilt `dist/` from the substantial uncommitted prior
  work already present before this session started), not a regression
  introduced here, and out of scope to fix per this increment's Scope.
  `tsc -b`'s own type-level build (the project's actual `build` script)
  succeeded with 0 errors, and `vitest`-based tests (which import
  TypeScript sources directly, not the compiled `dist/`) are the
  project's real test harness and all pass/fail as expected.

## Result

Implemented the concrete brief handed off by the migration-engineer
audit, in full:

1. Added `minimatch` (small, standard glob library) to `@axiom/workflow`
   — no glob/minimatch dependency existed anywhere in the monorepo
   before this change.
2. Implemented `validateWriteScope` (`@axiom/workflow`), the single
   shared comparison primitive covering all 5 addendum §9 detection
   cases, with full test coverage (15 tests: violation + clean path for
   each case, plus a Windows-path-separator normalization test).
3. Implemented `axiom validate changes --project <id> --plan <planId>`
   (CLI), the addendum's literal named command, with 6 tests.
4. Implemented `@axiom/doctor`'s `WS-001` check, reusing
   `axiom-role.ts`'s existing active-plan lookup mechanism verbatim (no
   new "which plan is active" concept invented), skipping gracefully
   when no plan is in flight, with 6 tests including 2 real-git-repo
   pass/fail scenarios.
5. Both surfaces share one comparison function
   (`validateWriteScope`) and one set of topology-aware helpers
   (`@axiom/doctor`'s `write-scope.ts`) — zero duplicated diff-vs-scope
   logic, matching `index-cmd.ts`'s established "thin CLI wrapper +
   doctor-consumable shared primitive" pattern from INC-07.
6. `npm run typecheck` and `npm run build` pass clean across the full
   monorepo. `npm test` shows the same 12-failed-files/13-failed-tests
   count as the freshly-measured baseline (all pre-existing,
   environment/real-path-dependent flakiness, confirmed unrelated by
   re-running twice), plus 27 new passing tests from this increment.

One minor factual correction surfaced during implementation (documented
in Assumptions): `GENERATED_FILES_BY_TARGET` exists in exactly **one**
place (`@axiom/installer/registry.ts`), not six separate per-adapter
lists as the audit's brief phrased it — the aggregation still happens
exactly as recommended, from that one source.

## General spec integration

No integration into `general-spec.md` was performed. As with the
migration-engineer audit before it, `general-spec.md` does not exist in
this repo yet (the closest equivalents remain
`Axiom.Spec/specs/00_Resumen_Ejecutivo.md` through `08_Glosario.md`,
not modified here). This increment is implementation work whose result
should be consolidated together with INC-08's final validator-reviewer
closure, not split across two partial integration passes.

## Closure rationale

Per `Axiom.SDD/AGENTS.md`'s closure rules, this increment is set to
`pending`, not `closed`, because:

- Per the roadmap's own established chain pattern (mirrored by
  INC-06/INC-07's `migration-engineer → next role → validator-reviewer`
  sequence), a `cli-implementer` step's own self-reported validation is
  not treated as final closure — an independent `validator-reviewer`
  re-verification is expected before INC-08 as a whole is marked done,
  exactly as the audit's own "Next step recommendation" specifies.
- No `general-spec.md` integration has occurred yet (deferred to the
  validator-reviewer step, per the migration-engineer audit's own
  precedent of deferring that consolidation until implementation is
  independently confirmed correct).

This mirrors the established chain pattern from INC-06/INC-07
(migration-engineer audit → next role → validator-reviewer close), not
a deviation from it.

## Next step recommendation

Proceed to **validator-reviewer**, following the same
independent-re-verification pattern used to close INC-06/INC-07:

1. Re-read `packages/workflow/src/write-scope.ts`,
   `packages/doctor/src/write-scope.ts`,
   `packages/doctor/src/checks.ts`'s `runWriteScopeChecks`,
   `apps/cli/src/commands/validate-changes.ts`, and their respective
   test files directly (not from this document's prose).
2. Independently re-run `npm run typecheck`, `npm run build`, `npm
   test` (full monorepo) and confirm the same 12-failed/13-failed
   baseline-parity result reported here.
3. Confirm no unrelated files were modified (this increment's `git
   status`/diff should show exactly: `packages/workflow/package.json`,
   `packages/workflow/src/index.ts`,
   `packages/workflow/src/write-scope.ts`,
   `packages/workflow/tests/write-scope.test.ts`,
   `packages/doctor/package.json`, `packages/doctor/tsconfig.json`,
   `packages/doctor/src/index.ts`, `packages/doctor/src/checks.ts`,
   `packages/doctor/src/write-scope.ts`,
   `packages/doctor/tests/write-scope.test.ts`,
   `apps/cli/src/index.ts`, `apps/cli/src/commands/validate-changes.ts`,
   `apps/cli/tests/validate-changes.test.ts`, plus lockfile changes from
   the `minimatch` install — every other pending/uncommitted change in
   the working tree predates this increment and belongs to prior,
   separately-tracked work).
4. Decide whether the pre-existing `dist/` runtime module-resolution
   issue noted in Validation warrants a separate bug ticket (it predates
   and is unrelated to this increment, but blocks any real
   `node apps/cli/dist/index.js` smoke test of the new command until
   fixed).
5. If everything reconciles, integrate the stable, final shape into
   `Axiom.Spec/general-spec.md` (introduce the file if still absent) and
   close INC-08 as a whole.
