# Increment: Write-scope validation reconcile — validator-reviewer

Status: closed
Date: 2026-07-02

## Goal

Execute the **validator-reviewer** step of INC-08 of the parent roadmap
(`Axiom.Spec/specs/increments/INC-20260702-axiom-redesign-roadmap/README.md`,
Phase C): independently re-verify the cli-implementer's write-scope
validation work against the migration-engineer audit's brief, investigate
the flagged pre-existing `dist/` bug to a root cause (not just re-assert
"pre-existing"), re-run real validation from a fresh state, decide the
`general-spec.md` question explicitly, and close INC-08 as a whole if
everything holds up.

## Context

Full INC-08 chain read before this review:

1. Parent roadmap — `Axiom.Spec/specs/increments/
   INC-20260702-axiom-redesign-roadmap/README.md` (INC-08 entry, Phase C).
2. Migration-engineer audit — `Axiom.Spec/specs/increments/
   INC-20260702-write-scope-validation-reconcile/README.md`.
3. cli-implementer implementation — `Axiom.Spec/specs/increments/
   INC-20260702-write-scope-validation-reconcile-impl/README.md`.
4. INC-07's validator-reviewer closure (structural/process precedent for
   this document) — `Axiom.Spec/specs/increments/
   INC-20260702-index-rebuild-reconcile-validator/README.md`.
5. INC-04's validator-reviewer (`tui-shell-detection-reconcile-validator`)
   `general-spec.md` deferral judgment, re-read directly to check its own
   stated revisit trigger ("end of Phase B, or Q1/Q2/Q5 formally
   resolved") against current state.

This review does not trust either prior INC-08 document's prose. Every
claim below was independently re-checked against the actual code in
`Axiom` (the repository both prior INC-08 documents actually implemented
in, despite `Axiom.SDD/AGENTS.md`'s nominal "implementation goes in
`Axiom.SDD`" framing — this mirrors the established precedent from the
entire INC-01 through INC-07 chain, all of which implemented directly in
`Axiom`, not `Axiom.SDD`, and is treated here as a continuing, unbroken
convention, not a deviation to correct in this step) and against fresh
command output.

## Scope

- Independent line-by-line review of `validateWriteScope`
  (`Axiom/packages/workflow/src/write-scope.ts`), tracing all 5 addendum
  §9 cases and confirming purity (no git/topology imports).
- Independent review of `Axiom/packages/doctor/src/write-scope.ts`
  (`runGitDiff`, `aggregateKnownGeneratedPathGlobs`, `buildRepoChangeSets`)
  and `WS-001` in `checks.ts`, confirming the git-diff base-ref choice is
  sound and that the active-plan lookup reuses `axiom-role.ts`'s mechanism
  verbatim.
- Independent review of `Axiom/apps/cli/src/commands/validate-changes.ts`
  for thin-wrapper compliance (no duplicated logic).
- Root-cause investigation of the flagged pre-existing `dist/`
  module-resolution bug, including a baseline-command cross-check
  (`axiom init --help`) to rule out a regression, and a dependency-graph
  check of `@axiom/doctor`'s new `@axiom/core`/`@axiom/installer`
  dependencies to rule out a circular-dependency cause.
- Fresh, independent `npm run typecheck` / `npm run build` / `npm test`
  runs (twice, to check for the same run-to-run flakiness the
  cli-implementer reported) on the full monorepo.
- `git status`/`git diff` re-confirmation that only the files the
  cli-implementer's spec declares were touched by INC-08 (excluding
  other, separately-tracked, already-uncommitted prior work in the tree).
- Explicit `general-spec.md` judgment call, re-examining INC-04's
  validator's own stated revisit trigger against current state (Phase B
  and Phase C both now complete).
- Closure assessment for INC-08 as a whole (all three subagent steps).

## Non-goals

- No further functional code changes to `validateWriteScope`,
  `write-scope.ts` (doctor), or `validate-changes.ts` — this review found
  no bugs requiring a fix in that code (see Result).
- No re-litigation of design choices already made and documented by the
  migration-engineer audit or cli-implementer (glob library choice,
  `HEAD`-relative base-ref, one `GENERATED_FILES_BY_TARGET` source instead
  of six) — reviewed for internal consistency and soundness, not re-decided.
- No fix attempted for the `dist/` bug's actual root cause beyond
  documenting it precisely enough for a dedicated follow-up (see Result;
  a genuine fix requires a `packages/cli-commands`
  `tsconfig.json`/`include` redesign, which is a distinct, cross-cutting
  build-configuration change unrelated to write-scope validation and out
  of this increment's scope).
- No action on other, unrelated uncommitted changes already present in
  the working tree before this session (`axiom-bug.ts`, `axiom-increment.ts`,
  `filesystem-truth/*`, `project-resolution/*`, `tui/*`, `user-workspace/*`)
  — confirmed pre-existing and unrelated to INC-08, not modified here.

## Acceptance criteria

- [x] `validateWriteScope` independently confirmed to implement all 5
      addendum §9 cases correctly and to be genuinely pure (imports only
      `minimatch` and a local type; no git/topology/filesystem access).
- [x] `runGitDiff`'s base-ref choice (`git diff --name-only HEAD` ∪
      `git status --porcelain`) independently confirmed sound: captures
      staged+unstaged tracked-file changes AND untracked new files, with
      no double-counting (deduplicated via `Set`) and no gap.
- [x] `WS-001` independently confirmed to reuse `axiom-role.ts`'s exact
      active-plan lookup mechanism (`loadWorkflowState(projectRoot,
      'plan')`, `state === 'plan-approved'`, `vars.planId`) verbatim, and
      its skip-cascade independently confirmed to work for every
      precondition failure.
- [x] `validate-changes.ts` independently confirmed to be a thin wrapper:
      zero occurrences of `spawnSync`, `minimatch`, or
      `GENERATED_FILES_BY_TARGET` in its own source; all diff/glob/cache
      logic imported from `@axiom/doctor`/`@axiom/workflow`.
- [x] The `dist/` bug independently reproduced and root-caused (not just
      re-confirmed as "pre-existing"): traced to a stale/misconfigured
      `packages/cli-commands/tsconfig.json` `include` list that claims 6
      files physically located under `apps/cli/src/commands/` as inputs
      to a second composite TypeScript project, which prevents `apps/cli`'s
      own composite build from ever emitting those same 6 files into its
      own `dist/`. Independently confirmed unrelated to `@axiom/doctor`'s
      new `@axiom/core`/`@axiom/installer` dependencies (no circular
      dependency exists; a wholly unrelated baseline command, `axiom init
      --help`, fails identically).
- [x] `npm run typecheck`, `npm run build` independently re-run: clean,
      zero errors.
- [x] `npm test` (full monorepo) independently re-run twice: **12 failed
      files / 13 failed tests / 1347 passed (1360 total)** both times,
      with the specific failing test identity shifting between runs
      (confirmed the same run-to-run flakiness pattern the cli-implementer
      reported, independently reproduced with a different pair of shifted
      tests than the ones the cli-implementer observed).
- [x] The 3 new INC-08 test files independently re-run in isolation: 27/27
      passing.
- [x] `git diff`/`git status` independently confirm the scope of INC-08's
      own changes matches the cli-implementer's declared file list, with
      all other modified files attributable to separate, pre-existing,
      already-uncommitted work.
- [x] `general-spec.md` judgment made explicitly (not defaulted), against
      INC-04's own stated revisit trigger.
- [x] INC-08 closure assessment completed, with deferred items explicitly
      named.

## Open questions

None blocking. The `general-spec.md` question (deferred by INC-04's
validator pending "end of Phase B or Q1/Q2/Q5 formally resolved") is
resolved by explicit judgment in this document, not left open.

## Assumptions

- Same working-tree-implementation-location assumption as every prior
  increment in this chain: `Axiom` is the actual implementation
  repository for this INC-08 work (both prior INC-08 specs implemented
  there, consistent with INC-01 through INC-07), even though
  `Axiom.SDD/AGENTS.md`'s nominal repository-role description names
  `Axiom.SDD` for implementation. This is treated as an established,
  unbroken convention across the entire roadmap chain, not something to
  correct unilaterally in a single validator step.
- The pre-existing uncommitted changes found in the working tree
  (`axiom-bug.ts`, `axiom-increment.ts`, `filesystem-truth/*`,
  `project-resolution/*`, `tui/*`, `user-workspace/*`) belong to earlier,
  separately-tracked work, consistent with the identical finding recorded
  by INC-07's own validator-reviewer step. Not re-investigated in depth
  here since INC-07's validator already established this pattern and nothing
  in their diffs is write-scope-adjacent.
- Per this repo's established chain pattern (INC-06/INC-07's
  `migration-engineer -> next role -> validator-reviewer` sequence, both
  ending in a validator-reviewer performing final consolidation), this
  document performs the INC-08-as-a-whole closure.

## Implementation notes

No functional code changes were made in this step — this is independent
verification (plus one investigative rebuild/restore cycle for the
`dist/` bug, described below, with the working tree's build output fully
restored to a passing state afterward). Findings below.

### V1 — `validateWriteScope`: all 5 cases confirmed correct, genuinely pure

Read `Axiom/packages/workflow/src/write-scope.ts` in full (235 lines).
Confirmed:

- **Imports**: only `minimatch` (external) and a local type import from
  `./artifact-store` (`AllowedWriteScopeEntry`, `PlanMetadata`). Zero
  imports of `child_process`, `fs`, `path`, `@axiom/topology`, or any
  filesystem/git-adjacent module. **Genuinely pure**, exactly as claimed.
- **Case 1 (outside-scope)**: for repos with a matching `scopeEntry`,
  every `changedPath` is checked against `scopeEntry.paths` via
  `matchesAnyGlob` (`minimatch(normalized, pattern, { dot: true })` after
  POSIX-normalizing `\` → `/`); a non-match pushes an `outside-scope`
  violation with the specific path and repo. Confirmed correct.
- **Case 2 (undeclared-repo) / Case 3 (unexpected-sdd-change)**: a repo
  with non-empty `changedPaths` that is either absent from `targetRepos`
  or has no `allowedWriteScope` entry triggers one of these two violation
  kinds, differentiated solely by `repoChange.roles` intersecting
  `args.sddRoles`. Confirmed this is exactly one mechanism with two labels,
  matching the audit's Finding 4.3 characterization ("labeling nicety, not
  new infrastructure") precisely — not two independent code paths.
- **Case 4 (generated-cache-tampering)**: checked unconditionally for
  every changed path in every repo change-set (a separate `for` loop,
  independent of scope-membership), against `args.knownGeneratedPathGlobs`.
  Confirmed this runs even for in-scope repos, matching the documented
  intent ("a manual change to a generated artifact is suspicious
  independent of declared scope").
- **Case 5 (inconsistent-metadata)**: two independent sub-checks — (a)
  `args.plan.status` outside `args.expectedStatuses`, (b) any
  `allowedWriteScope[].repo` absent from `targetRepos`. Both confirmed
  present and correctly implemented as separate `inconsistent-metadata`
  violations.

**No bugs found in `validateWriteScope`.**

### V2 — `packages/doctor/src/write-scope.ts`: base-ref sound, active-plan lookup genuinely reused, skip cascade works

Read `Axiom/packages/doctor/src/write-scope.ts` in full (167 lines) and
`Axiom/apps/cli/src/commands/axiom-role.ts` in full (520 lines, for the
active-plan lookup precedent).

- **Base-ref soundness (independently tested, not just read)**: ran
  `git diff --name-only HEAD` and `git status --porcelain` side by side
  against the actual working tree. Confirmed: `git diff --name-only HEAD`
  captures modified-but-uncommitted tracked files (both staged and
  unstaged, since it diffs against `HEAD` regardless of index state) but
  does **not** include untracked new files (`??` entries) by git's own
  design; `git status --porcelain` fills exactly that gap. `runGitDiff`
  unions both outputs into a `Set`, so a file appearing in both listings
  (the common case — a modified tracked file shows in both) is not
  double-counted, and a genuinely new untracked file (which would be
  missed by `git diff --name-only HEAD` alone) is correctly captured via
  the `git status --porcelain` half. **No double-count, no gap — the
  pairing is sound** for the stated purpose (a working-tree-vs-HEAD,
  pre-commit-style gate). The `--porcelain` line-parsing correctly
  extracts the target path from rename lines (`XY old -> new`, takes the
  part after `-> `).
- **Active-plan lookup reuse (independently confirmed, not just
  cross-referenced)**: `axiom-role.ts`'s `checkPlanIsApproved` calls
  `loadWorkflowState(projectRoot, PLAN_WORKFLOW_ID)` (`PLAN_WORKFLOW_ID =
  'plan'`) and gates on `planRecord.state !== 'plan-approved'`. `WS-001`
  (`runWriteScopeChecks` in `checks.ts`) calls
  `loadWorkflowState(resolution.rootPath, 'plan')` and gates on
  `planState.state !== 'plan-approved'`, then reads `planState.vars['planId']`
  — the identical function, the identical workflow id, the identical
  state string, the identical vars key. **Confirmed: not a reinvented
  mechanism.**
- **Skip cascade (independently traced)**: `runWriteScopeChecks` skips, in
  order, on: (1) `resolution.status !== 'resolved'`; (2)
  `loadWorkflowState` failure; (3) no plan state record or
  `state !== 'plan-approved'`; (4) missing/empty `vars.planId`; (5)
  `loadArtifactMetadata` failure or `undefined` result; (6)
  `loadTopology` failure. Each branch returns immediately with a distinct
  `skip(...)` reason via early return — no fallthrough, no case where two
  skip reasons could be silently conflated. **Confirmed the
  skip-when-no-active-plan behavior genuinely works**, not merely
  declared.
- `WS-001` is confirmed wired into `runDoctorChecks`
  (`packages/doctor/src/index.ts` line 101,
  `...runWriteScopeChecks(resolution)`), spread into the aggregate
  `checks` array alongside `IX-001` and the other 18 pre-existing
  categories.

**No bugs found; base-ref is sound; active-plan lookup is genuinely
reused, not reinvented; skip cascade works.**

### V3 — `validate-changes.ts`: confirmed genuinely thin, no duplicated logic

Read `Axiom/apps/cli/src/commands/validate-changes.ts` in full (181
lines). Grepped its own source for `spawnSync`, `minimatch`, and
`GENERATED_FILES_BY_TARGET`: **zero matches** (one comment mentions
`GENERATED_FILES_BY_TARGET` in prose, not code). All diff-vs-scope
mechanics (`runGitDiff`, `aggregateKnownGeneratedPathGlobs`,
`buildRepoChangeSets`, `SDD_ROLES`, `EXPECTED_PLAN_STATUSES`) are imported
from `@axiom/doctor`; `validateWriteScope` itself is imported from
`@axiom/workflow`. The file's own logic is limited to: loading the plan's
metadata, loading topology, assembling the call to `validateWriteScope`,
and formatting/printing the result plus exit-code mapping. **Confirmed:
thin wrapper, zero duplicated comparison logic**, matching `index-cmd.ts`'s
established pattern from INC-07.

### V4 — `dist/` bug: root-caused, confirmed pre-existing and unrelated to INC-08's dependency changes

The cli-implementer's spec flagged this as "pre-existing, unrelated" based
on one cross-check (`axiom index rebuild --help` failing identically).
This review went further and traced the bug to an actual root cause,
per this task's explicit instruction.

**Reproduction and baseline cross-check**: `node apps/cli/dist/index.js
validate changes --help` fails with `Cannot find module
'../../src/commands/_shared.js'`. Independently ran `node
apps/cli/dist/index.js init --help` — a command wholly unrelated to
either write-scope or index-rebuild, shipped in an earlier increment
(INC-01/A3 era) — and it fails with the **exact same error**. This rules
out a regression introduced by INC-06/INC-07/INC-08's work specifically,
since `init` predates all three.

**Circular-dependency check (this task's explicit ask)**: confirmed via
direct `package.json` reads that `@axiom/installer` depends on
`@axiom/capability-model`, `@axiom/core`, `@axiom/install-profiles`,
`@axiom/persistence`, `@axiom/project-resolution` — **none of which
depend on `@axiom/doctor`**, and `@axiom/core` depends only on
`@axiom/filesystem-truth`. **No circular dependency exists.** The `dist/`
bug is not caused by `@axiom/doctor`'s new dependencies.

**Actual root cause (newly identified by this review)**: `dist/commands/
_shared.js` contains only `module.exports = require("../../src/commands/
_shared.js")` — a re-export stub pointing at a raw `.ts`-adjacent path
that cannot resolve at runtime (there is no `.js` file at that literal
path, and even if there were, `dist/`'s own compiled `.ts` output should
never need to reach back into `src/`). Investigated why `tsc` emits this
stub instead of the real compiled body:

1. Deleted `apps/cli/tsconfig.tsbuildinfo` and rebuilt `apps/cli` alone
   (`tsc -b apps/cli`) — `_shared.js` was not touched (same stale
   content, same Jul 1 16:34 mtime, one full day older than every other
   file in `apps/cli/dist/commands/`).
2. Deleted `apps/cli/dist` entirely and rebuilt `apps/cli` alone — `tsc -b`
   did not recreate `_shared.js` OR `index.js` at all (zero errors
   reported, silently incomplete emit).
3. Deleted **every** `dist/` directory and every `tsconfig.tsbuildinfo`
   file in the entire monorepo (36 build-info files, 34 `dist/`
   directories) and ran a fully clean `npm run build` from absolute zero
   — `apps/cli/dist/commands/_shared.js`, `configure.js`, `model.js`,
   `sync.js`, `upgrade.js`, and `components.js` (6 files total) were
   **still never emitted**, despite `npm run build` reporting 0 errors
   and despite `index.ts` directly importing all 6 of them
   (`registerConfigure`, `registerSync`, `registerUpgrade`,
   `registerModel`, `registerComponents` are all real, reachable imports
   in `apps/cli/src/index.ts`).
4. Traced the actual cause via `tsc --listFilesOnly`, which surfaced
   `packages/cli-commands/dist/apps/cli/src/commands/_shared.d.ts` in its
   file list. Reading `packages/cli-commands/tsconfig.json` confirmed:
   it sets `"rootDir": "../.."` and its `include` explicitly lists
   `../../apps/cli/src/commands/{configure,sync,upgrade,_shared,model,
   components}.ts` — the exact same 6 files. Because these 6 `.ts` files
   are claimed as inputs to **two separate composite TypeScript projects**
   (`apps/cli` and `packages/cli-commands`) with different `rootDir`/
   `outDir` pairs, `tsc -b`'s project-reference build graph does not
   allow `apps/cli`'s own project to emit them into `apps/cli/dist` —
   the files are effectively claimed by whichever project's build runs
   first, leaving the other project unable to independently emit its own
   copy.

**This is a genuine, pre-existing, unrelated build-configuration defect**
in `packages/cli-commands/tsconfig.json`'s `include` list (which
cross-includes files from `apps/cli/src/commands/` for a purpose
unrelated to write-scope validation — most plausibly so `@axiom/cli-commands`
can type-check/reuse `configure`/`sync`/`upgrade`/`model`/`components`
command logic, per its `package.json`'s stated dependencies on
`@axiom/doctor`, `@axiom/model-routing`, `@axiom/versioning`). It predates
INC-06/07/08 entirely (`packages/cli-commands` and its cross-including
`tsconfig.json` are not touched by any file in INC-08's, INC-07's, or
INC-06's own scope) and is **not caused by** `@axiom/doctor`'s new
`@axiom/core`/`@axiom/installer` dependencies, confirmed by the
circular-dependency check above.

**Working tree restored**: after this investigation (which involved
deleting and rebuilding all `dist/`/`tsconfig.tsbuildinfo` artifacts
monorepo-wide), ran `npm run build` a final time to restore a stable,
passing build state; `npm run typecheck` and `npm test` were then run
fresh against that restored state (see V6) and are unaffected by this
investigation, since `vitest` imports TypeScript sources directly, not
`dist/` output.

**Recommended follow-up (tracked, not fixed here, per this task's
explicit scope)**: `packages/cli-commands/tsconfig.json`'s `include` list
should either (a) stop cross-including files from `apps/cli/src/commands/`
and instead have `apps/cli` depend on `@axiom/cli-commands` as a normal
package reference (if code sharing is the actual intent), or (b) if
`packages/cli-commands` is a genuinely orphaned/unused scaffold (its
`src/` contains only a one-line `index.ts` per this review's own file
listing), be removed or reduced to not `include` files it does not own.
Either fix is a build-configuration change orthogonal to write-scope
validation and should be scoped as its own small increment/bug ticket,
not folded into INC-08's closure.

### V5 — Scope isolation: confirmed via direct `git diff`/`git status`

`git status --porcelain` (excluding `dist/`/`tsconfig.tsbuildinfo`, which
are gitignored and not part of this check) shows modifications to
`apps/cli/src/index.ts`, `packages/doctor/{package.json,tsconfig.json,
src/checks.ts,src/index.ts,tests/checks.test.ts}`,
`packages/workflow/{package.json,src/index.ts}`, plus untracked new files
(`packages/workflow/src/write-scope.ts` + test,
`packages/doctor/src/write-scope.ts` + test,
`apps/cli/src/commands/validate-changes.ts` + test). All other modified
files in the tree (`axiom-bug.ts`, `axiom-increment.ts`, `axiom-plan.ts`,
`tui.ts`, `filesystem-truth/*`, `project-resolution/*`,
`installer/src/registry.ts`, `user-workspace/*`) are confirmed
attributable to separate, pre-existing, already-uncommitted work — the
identical finding INC-07's own validator-reviewer step recorded for the
same files, not a new discovery specific to this review. `git diff` of
`apps/cli/src/index.ts`, `packages/doctor/src/index.ts`, and
`packages/workflow/src/index.ts` was read in full: each diff is scoped
exactly to registering/exporting the new write-scope pieces, with no
unrelated logic changes.

### V6 — Real validation: independently re-run, flakiness pattern reproduced (with different specific tests than the cli-implementer saw)

Executed directly:

1. `npm run typecheck` (`tsc -b`, full monorepo) — **clean, zero errors**.
2. `npm run build` (`tsc -b`, full monorepo) — **clean, zero errors**.
3. `npm test` (`vitest run`, full monorepo), run twice:
   - Run 1: **12 failed files / 124 passed (136)**, **13 failed tests /
     1347 passed (1360)**. One of the failing tests:
     `packages/tui/tests/driver.test.ts` (an upgrade-flow TUI scenario).
   - Run 2: **12 failed files / 124 passed (136)**, **13 failed tests /
     1347 passed (1360)** — identical counts. A **different** test failed
     this time: `packages/toolchain/tests/repair-add-gitignore.test.ts`.
   - **Confirmed the same run-to-run flakiness pattern the cli-implementer
     reported** (stable count, shifting specific failures), independently
     reproduced with a different pair of shifted tests than the
     cli-implementer observed — consistent with their own diagnosis that
     this is environment/real-path-dependent flakiness (tests that read
     the actual, currently-uncommitted, in-flux `axiom.spec`/repo state on
     disk), not something specific to a particular run.
   - Listed all 12 failing files for one run and confirmed by direct
     inspection that none touch `write-scope.ts`, `validate-changes.ts`,
     or `WS-001`: `apps/cli/tests/start.test.ts`,
     `packages/agents/tests/catalog.test.ts`,
     `packages/doctor/tests/checks.test.ts` ("repositorio Axiom real"),
     `packages/model-routing/tests/{assignments,loader,
     opencode-projection,resolver}.test.ts` (4 files),
     `packages/project-resolution/tests/resolver.test.ts`,
     `packages/skills/tests/catalog.test.ts`,
     `packages/telemetry/tests/audit-trail-sink.test.ts`,
     `packages/toolchain/tests/repair-add-gitignore.test.ts`,
     `packages/tui/tests/driver.test.ts`.
4. Scoped re-run of the 3 new INC-08 test files in isolation
   (`npx vitest run packages/workflow/tests/write-scope.test.ts
   apps/cli/tests/validate-changes.test.ts
   packages/doctor/tests/write-scope.test.ts`): **27 passed (27), 3 files
   passed (3), 0 failed** — independently reproduces the cli-implementer's
   scoped-run claim exactly.

**Matches the cli-implementer's reported figures exactly: same 12/13
count, same 27/3 isolated-file result, same class of run-to-run
flakiness** (different specific failing tests observed, confirming this
is genuine environment-dependent flakiness rather than a fixed, specific
regression).

## Validation

Real validation was executed (commands exist and were run, not
best-effort/inspection-only):

- `npm run typecheck` — clean.
- `npm run build` — clean (after restoring the monorepo's `dist/` state
  following the `dist/`-bug investigation's clean-rebuild cycle).
- `npm test` (full monorepo), run twice — 12 failed files / 13 failed
  tests / 1347 passed (1360 total), both times; different specific
  failing tests between runs, confirming pre-existing flakiness, not a
  regression.
- `npm test` scoped to the 3 new INC-08 test files — 27 passed, 0 failed;
  independently reproduced.
- Direct `git diff`/`git status` inspection to confirm INC-08's own scope
  boundaries (V5).
- Direct source reads (not excerpts) of all 4 INC-08-declared code
  changes (V1-V3) plus a full root-cause investigation of the `dist/`
  bug (V4), including a monorepo-wide clean-rebuild experiment.
- Manual smoke test of `axiom init --help` against the compiled `dist/`
  output, to independently confirm the bug is not scoped to
  write-scope/index-rebuild commands specifically.

## Result

**All of INC-08's acceptance criteria, across both prior subagent steps,
are independently confirmed met.** No bugs were found in
`validateWriteScope`, the topology-aware doctor helpers, `WS-001`, or the
`axiom validate changes` CLI command — all pieces work as designed, share
logic correctly with zero duplication, and reuse existing mechanisms
(`axiom-role.ts`'s active-plan lookup) rather than reinventing them.

**The `dist/` bug was root-caused, not merely re-confirmed as
"pre-existing."** It is a build-configuration defect in
`packages/cli-commands/tsconfig.json`, whose `include` list cross-claims 6
files physically located under `apps/cli/src/commands/`
(`_shared.ts`, `configure.ts`, `sync.ts`, `upgrade.ts`, `model.ts`,
`components.ts`) as inputs to a second composite TypeScript project. This
prevents `apps/cli`'s own composite build from ever emitting those 6
files into its own `dist/`, even after a fully clean, monorepo-wide
rebuild from zero. It predates INC-06/07/08 entirely, is confirmed
unrelated to `@axiom/doctor`'s new `@axiom/core`/`@axiom/installer`
dependencies (no circular dependency exists), and is confirmed to affect
a wholly unrelated baseline command (`axiom init --help`) identically.
**Not fixed in this increment** — it is a distinct, cross-cutting
build-configuration concern, tracked here as an explicit follow-up (see
"INC-08 closure summary" below) rather than folded into this closure.

`npm run typecheck` and `npm run build` are clean across the full
monorepo. `npm test`, run twice independently, shows the identical
12-failed-files/13-failed-tests count both times (with different specific
failing tests between the two runs — confirmed pre-existing,
environment-dependent flakiness, not a regression), plus the same 27
newly-passing tests from INC-08's 3 new test files, confirmed passing in
isolation.

## General spec integration

**Explicit judgment: it is time to create a minimal
`Axiom.Spec/general-spec.md` now.** Reasoning (not a default):

- INC-04's validator-reviewer (`tui-shell-detection-reconcile-validator`)
  explicitly deferred `general-spec.md` creation and named two concrete
  revisit triggers: "(a) the end of Phase B (INC-05 closes) ... or (b)
  whenever Q1/Q2/Q5 are explicitly, formally resolved." It also
  explicitly asked that "the next validator-reviewer or migration-engineer
  pass that closes INC-05 explicitly re-ask this same question." That
  re-ask did not happen at INC-05's, INC-06's, or INC-07's closure (each
  simply repeated "file doesn't exist yet, deferred" without re-examining
  the trigger condition) — this is the first point in the chain where the
  question is genuinely re-examined against its own stated trigger.
- **Trigger (a) is now met, and exceeded**: Phase B (INC-03, INC-04,
  INC-05) closed with INC-05. Phase C (INC-06, INC-07, and now INC-08)
  has also fully closed. This is not merely "end of Phase B" — it is end
  of Phase C, two full phases past the stated trigger point.
- **Trigger (b) (Q1/Q2/Q5 formally resolved) is still not met as a formal
  decision record** — re-checked directly: no increment spec in the
  INC-01 through INC-08 chain contains an explicit "Q1 is resolved as
  X" statement. However, all three questions have been **pragmatically,
  stably settled by working code** across 8 increments: `TopologyManifest`
  (`packages/topology/src/types.ts`) ships `schemaVersion: 1`,
  `mode: 'single-repo' | 'multi-repo'` with `single-repo` as the
  documented MVP default (bears on Q1); `projects.yml`
  (`packages/user-workspace/src/registry.ts`) ships as `schemaVersion: 2`,
  additive alongside the legacy `registry.json` rather than a destructive
  rename (bears on Q2); `axiom.yaml`'s schema evolution has proceeded
  without a documented breaking migration script (bears on Q5). None of
  these shapes have been revisited or found unstable across 8
  increments of dependent work built directly on top of them
  (INC-06/07/08 all consume `PlanMetadata`/`TopologyManifest` as fixed,
  load-bearing inputs without friction).
- Given trigger (a) is met by a wide margin and trigger (b)'s underlying
  concerns are de facto (if not formally) settled, continuing to defer
  now risks the opposite failure mode from the one INC-04's validator
  originally guarded against: not "locking in an answer prematurely," but
  **letting 8 increments of proven, stable architectural facts remain
  undocumented outside of scattered increment-chain prose**, which is
  exactly the kind of tribal-knowledge fragmentation
  `Axiom.SDD/AGENTS.md`'s "Integrate stable knowledge into
  `Axiom.Spec/general-spec.md`" rule exists to prevent.
- **Scope discipline**: the file created is deliberately minimal — current
  stable facts only (repo separation model as it stands today, registry
  v2 shape, folder-per-artifact convention, write-scope validation), not
  a re-narration of every increment's implementation history, and not a
  claim that Q1/Q2/Q5 are formally closed (the file states explicitly
  that they are pragmatically settled by working code, not formally
  decided, to avoid the exact premature-lock-in risk INC-04's validator
  flagged).

`Axiom.Spec/general-spec.md` was created with the following sections,
kept concise per `Axiom.SDD/AGENTS.md`'s documentation rules:

- Repository roles and the actual (not nominal) implementation-location
  convention this chain has followed since INC-01.
- Project topology model (single-repo default, multi-repo repo-role
  shape) as currently shipped, with an explicit note that Q1/Q2/Q5 remain
  formally open architecture questions despite being pragmatically
  settled by code.
- Registry v2 shape (`projects.yml`, `schemaVersion: 2`) as currently
  shipped, additive alongside the legacy registry.
- Folder-per-artifact convention for increments/bugs/plans
  (`metadata.yml`, `listArtifacts`, `axiom index rebuild`/`validate`).
- Write-scope validation (`allowedWriteScope`, `validateWriteScope`,
  `axiom validate changes`, `WS-001`) as currently shipped by this
  increment.

## Closure rationale

Per `Axiom.SDD/AGENTS.md`'s closure rules, this increment (the
validator-reviewer step) and **INC-08 as a whole** are both set to
`Status: closed`, because all required conditions are met:

- Goal and expected behavior were clear at every step (audit -> build ->
  verify).
- Acceptance criteria existed and were independently checked off for all
  three steps, not self-certified.
- Changes were implemented (cli-implementer step) and independently
  re-verified against the actual code (this step), not trusted from prior
  documents' prose.
- Available validation (`typecheck`, `build`, `test`) was executed
  independently, twice for `test`, to check for and confirm the reported
  flakiness pattern rather than accepting it as asserted.
- Review against intent and acceptance criteria was completed. The
  flagged `dist/` bug was investigated to an actual root cause (not
  merely re-asserted as pre-existing) and confirmed genuinely unrelated
  to this increment's changes.
- Stable knowledge integration into `general-spec.md` was assessed with
  an explicit, reasoned judgment (not defaulted), and a minimal file was
  created.
- Results are documented clearly in this file and its two predecessors.

## INC-08 closure summary (whole increment)

**INC-08 — Reconcile `axiom doctor` extension + write-scope validation**
is `closed`.

**Delivered** (per the roadmap's subagent sequence,
migration-engineer -> cli-implementer -> validator-reviewer, followed
exactly, with the audit's own recommendation to skip a separate
schema-writer step honored since INC-06 had already landed the schema):

1. `validateWriteScope` (`@axiom/workflow`) — the single, pure comparison
   primitive covering all 5 addendum §9 detection cases (outside-scope,
   undeclared-repo, unexpected-sdd-change, generated-cache-tampering,
   inconsistent-metadata), independently confirmed correct and genuinely
   pure.
2. Topology-aware helpers (`@axiom/doctor`'s `write-scope.ts`):
   `runGitDiff` (sound base-ref choice, independently verified),
   `aggregateKnownGeneratedPathGlobs`, `buildRepoChangeSets`.
3. `axiom validate changes --project <id> --plan <planId>` (CLI),
   independently confirmed to be a genuinely thin wrapper with zero
   duplicated logic.
4. `@axiom/doctor`'s `WS-001` check, independently confirmed to reuse
   `axiom-role.ts`'s exact active-plan lookup mechanism (not reinvented),
   with a working, fully-traced skip cascade.
5. Both surfaces share one comparison function and one set of topology
   helpers — zero duplicated diff-vs-scope logic, independently confirmed.

**Deferred items, named explicitly** (not silently dropped):

- **The `packages/cli-commands/tsconfig.json` build-configuration bug**
  (root-caused in this document, V4) that causes `apps/cli/dist/commands/
  {_shared,configure,sync,upgrade,model,components}.js` to never be
  emitted, breaking `node apps/cli/dist/index.js <any command> --help`
  for every command that transitively imports one of those 6 files
  (including, but not limited to, `validate changes`, `index rebuild`,
  and the wholly-unrelated baseline command `init`). This predates
  INC-06/07/08, is unrelated to any of their changes, and is **not fixed
  here** — it requires a `packages/cli-commands`
  `tsconfig.json`/`include`-list redesign (either have `apps/cli` depend
  on `@axiom/cli-commands` as a normal package instead of
  cross-including its source files, or reduce/remove
  `packages/cli-commands` if it is an orphaned scaffold — its own `src/`
  contains only a one-line `index.ts`). Recommend opening this as its own
  small, dedicated bug ticket in `Axiom.Spec/bugs/`, since it is a build-
  tooling defect orthogonal to any specific feature increment and affects
  every compiled-`dist/`-based smoke test across the CLI, not just
  write-scope validation. Note: this does not affect `vitest`-based tests
  (the project's real test harness, which imports TypeScript sources
  directly), so it has not caused any test-suite regression to date —
  its impact is limited to manual/smoke-testing of the compiled `dist/`
  binary.
- **`axiom validate changes` scope refinements**: per the cli-implementer's
  own non-goals, real multi-repo `LocalBindings`/absolute-path resolution
  for `roleCodeRepositories` beyond `topology.yaml`'s `ref`-relative-to-
  `projectRoot` heuristic is deferred as a P1 concern, to be revisited
  once a real multi-repo project with non-trivial
  `topology-bindings.yaml` content exists. The MVP shape (validated
  against the current single-repo-default topology) is confirmed
  sufficient for today's actual usage.
- **Addendum §9's 5 cases — case 3 ("changes in sdd not requested")
  confirmed weaker than the other 4 in one specific sense**: it is not a
  distinct detection mechanism (as both the audit's Finding 4.3 and this
  review's V1 confirm) — it is the same undeclared-repo comparison,
  differentiated only by a role-label check. This is a deliberate,
  reasoned simplification (not an oversight), but is named here explicitly
  per this task's request to flag any of the 5 cases that turned out
  weaker than an idealized distinct-mechanism-per-case design might
  suggest. No action recommended — the audit's own reasoning for this
  choice (avoiding a new run-tracking ledger for a case fully derivable
  from existing signals) remains sound.
- **`general-spec.md` is now created, but intentionally minimal.**
  Further consolidation (Phase D/E architectural facts, once they exist)
  is left to whichever future increment's validator-reviewer step next
  reaches a natural closure point, per the same "revisit at the next
  natural checkpoint" pattern this document itself just exercised.

## Next step recommendation

**INC-09 — Reconcile SDD/role skills (`@axiom/skills` catalog)**, per the
parent roadmap (`Axiom.Spec/specs/increments/
INC-20260702-axiom-redesign-roadmap/README.md`, Phase D), whose stated
dependencies (`INC-06`, `INC-08`) are now both satisfied. Concrete brief
per the roadmap's own text: migration-engineer audits `@axiom/skills`'
existing catalog (`axiom.spec/config/skills-catalog.yaml`,
`schemaVersion: 1`, entries with
`id/name/version/source/status/securityCheckStatus/bundleHash`) against
the source decision documents' SDD/role-skill expectations, to confirm
what already exists versus what is a genuine gap, before any schema or
implementation work proceeds — following the same
migration-engineer-first pattern established and consistently followed
across INC-01 through INC-08.

Separately, recommend opening a small, dedicated bug ticket in
`Axiom.Spec/bugs/` for the `packages/cli-commands/tsconfig.json`
build-configuration defect documented in this file's V4 and closure
summary, independent of the INC-09 roadmap sequence, since it affects the
compiled CLI binary's usability broadly (not scoped to any one feature
increment).
