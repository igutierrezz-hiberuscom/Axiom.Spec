# Increment: Reconcile contextual TUI shell and detection — implementation (tui-developer)

Status: pending
Date: 2026-07-02

## Goal

Implement the addendum-14 project-detection heuristic (4 checks: cwd
`axiom.yaml`, parent-walk `axiom.yaml`, registered-ancestor-path via
`~/.axiom/projects.yml`, git-root `axiom.yaml`) and the "no Axiom
detected via direct checks, but a registered/git-root match exists"
fallback UX in `axiom tui`, per the brief in the sibling audit
increment `INC-20260702-tui-shell-detection-reconcile`. This is the
**tui-developer** implementation pass; the audit increment remains
audit-only and unchanged.

## Context

This is the implementation follow-up to
`Axiom.Spec/specs/increments/INC-20260702-tui-shell-detection-reconcile/README.md`
(migration-engineer audit), itself part of INC-04 of
`Axiom.Spec/specs/increments/INC-20260702-axiom-redesign-roadmap/README.md`.
The audit confirmed:

- Checks 1/2 (cwd / parent `axiom.yaml`) already worked via
  `discoverAxiomRoot`.
- Checks 3/4 (registered-ancestor-path, git-root) were unimplemented.
- No shell rewrite was needed; the gap was additive.
- Today `axiom tui` with no resolved project throws `projectNotFound`
  and exits 1 with no "no Axiom detected but found via fallback" UX at
  all.

Three open questions were left for this pass to resolve explicitly
before implementation. **The user resolved all three up front** (not
re-litigated in this increment):

- **OQ1 (deferred)**: do not promote `mcp-inventory`/`memory-inventory`
  to first-class `MENU_ITEMS` here. Tracked separately at
  `Axiom.Spec/specs/increments/INC-20260702-tui-menu-promote-inventory-screens/README.md`.
- **OQ2 (resolved: new function in `@axiom/user-workspace`)**: the
  ancestor-match logic for check 3 lives in `@axiom/user-workspace` as
  a new function (`findByAncestorRepoPathV2`), keeping registry-shape
  knowledge encapsulated there instead of leaking into
  `@axiom/filesystem-truth`/`@axiom/project-resolution`.
- **OQ3 (resolved: build it now)**: the "no Axiom detected via
  cwd/parent-walk, but a registered/git-root match exists" fallback UX
  is implemented in this pass, not deferred.

## Scope

- Add git-root detection to `@axiom/filesystem-truth`
  (`findGitRoot`/`findGitRootAxiomConfig`).
- Add registered-ancestor-path matching to `@axiom/user-workspace`
  (`findByAncestorRepoPathV2`).
- Compose all 4 addendum-14 checks, in order, into a new
  `resolveProjectWithFallback` function in `@axiom/project-resolution`.
- Wire the fallback into `apps/cli/src/commands/tui.ts`'s default-mode
  control flow: when direct resolution (`resolveProject`) returns
  `not-found`, try `resolveProjectWithFallback` before throwing
  `projectNotFound`.
- Add a minimal, additive `initialMessage` option to `@axiom/tui`'s
  `runTui` (driver) to surface an informational note on the first menu
  render, reusing the existing `lastMessage` router field (no new
  `ScreenId`, no new screen).
- Add/extend tests for: `findGitRoot`/`findGitRootAxiomConfig`
  (filesystem-truth), `findByAncestorRepoPathV2` (user-workspace),
  `resolveProjectWithFallback` (project-resolution, all 4 checks +
  true-failure path), and the new `tui.ts` fallback control-flow branch
  (apps/cli).

## Non-goals

- No promotion of `mcp-inventory`/`memory-inventory` to `MENU_ITEMS`
  (OQ1, deferred — tracked in
  `INC-20260702-tui-menu-promote-inventory-screens`).
- No menu additions for increment/bug/plan creation or "guided
  implementation" (brief item 3 in the audit's tui-developer section;
  out of scope for this pass, which is detection/fallback-UX only).
- No `projects` screen v1→v2 registry migration (`listProjects` →
  `listProjectsV2`) — a separate, already-flagged finding from the
  audit, not part of this pass's scope.
- No changes to `@axiom/topology`, `@axiom/config-validation`,
  `@axiom/doctor`, `@axiom/orchestrator`, or the INC-01/02/03 CLI
  command files (`init.ts`/`join.ts`/`repo.ts`/`projects.ts`/
  `app-api.ts`).
- No shell/router rewrite: the router, driver dispatch model, and
  `@axiom/cli-commands` layering seam are unchanged in structure.

## Acceptance criteria

- [x] Addendum-14 checks 1-4 are all implemented and composed in a
      single ordered function (`resolveProjectWithFallback`).
- [x] Check 3 (registered-ancestor-path) lives in
      `@axiom/user-workspace` (`findByAncestorRepoPathV2`), not leaked
      into `filesystem-truth`/`project-resolution`.
- [x] Check 4 (git-root) is implemented with a minimal directory walk
      (no shell-out), consistent with the bootstrap's no-speculative-
      infra limit; no existing git-root utility was found to reuse
      (confirmed via full-file grep of `@axiom/filesystem-truth`
      before writing new code).
- [x] `axiom tui`'s default-mode control flow in
      `apps/cli/src/commands/tui.ts` falls through to the fallback
      when direct resolution is `not-found`, and preserves the
      existing `projectNotFound` behavior for the true-failure case
      (all 4 checks exhausted) and for `ambiguous`/`invalid-config`
      statuses (an `axiom.yaml` was found, just invalid — fallback is
      not attempted, matching the audit's own reasoning).
- [x] When resolution succeeds via check 3 or check 4, the TUI opens
      normally against the discovered root and shows an informational
      note (not an error) explaining how the project was found.
- [x] `mcp-inventory`/`memory-inventory` are untouched.
- [x] `@axiom/topology`, `@axiom/config-validation`, `@axiom/doctor`,
      `@axiom/orchestrator`, and the INC-01/02/03 CLI command files are
      untouched (confirmed via `git status`/diff review below).
- [x] New tests cover: git-root detection, ancestor-match, the
      composed fallback function (all 4 checks + true-failure path,
      including a case where a registered repo has no `axiom.yaml` at
      its root), and the new `tui.ts` control-flow branch's success
      and true-failure paths.
- [x] Full monorepo validation executed (`typecheck`, `test`); results
      compared against a freshly re-confirmed baseline (not assumed).
- [x] Result documented; closure status set honestly (see below).

## Open questions

None outstanding for this pass — OQ1/OQ2/OQ3 from the audit increment
were resolved by the user before implementation started (see Context).

## Assumptions

- The audit increment's "Assumptions" correction (INC-01/02/03 do have
  individual spec files as sibling folders in
  `Axiom.Spec/specs/increments/`) is taken as accurate and is not
  re-verified here; this increment follows the same sibling-folder
  naming precedent (`-impl` suffix for the implementation role,
  parallel to `-registry-wiring`/`-validator`/`-agents-md` suffixes
  used by the INC-03 chain).
- `MAX_TRAVERSAL_DEPTH` (10) in `@axiom/filesystem-truth` is treated as
  an intentional, Axiom-specific convention-depth bound for
  `discoverAxiomRoot` (looking for `axiom.yaml`, a convention with a
  bounded expected nesting depth). `findGitRoot` does NOT reuse that
  same bound: a real Git working tree can be nested arbitrarily deep
  independent of where `axiom.yaml` lives, and `git rev-parse
  --show-toplevel` itself has no artificial depth cap (it walks to the
  filesystem root or `GIT_CEILING_DIRECTORIES`). This was discovered
  while writing tests: an initial depth-capped `findGitRoot` made check
  4 practically unreachable in any scenario distinct from checks 1/2
  (if `axiom.yaml` sits at a git-root within the walkable range, checks
  1/2 already find it; if it's beyond that range, a depth-capped
  `findGitRoot` fails identically). The fix (removing the depth cap
  from `findGitRoot` only) is a deliberate, minimal correction validated
  by the new tests, not scope creep — it does not change
  `discoverAxiomRoot`'s existing bound or behavior at all.

## Implementation notes

### Files changed

**`@axiom/filesystem-truth`** (`Axiom/packages/filesystem-truth/`):
- `src/discovery.ts`: added `findGitRoot(startPath)` (unbounded parent
  walk looking for `.git` as file-or-directory, to support
  worktrees/submodules where `.git` is a `gitdir:` pointer file) and
  `findGitRootAxiomConfig(startPath)` (composes `findGitRoot` +
  `axiom.yaml` existence check at that root — addendum-14 check 4).
- `src/index.ts`: exported both new functions.
- `tests/discovery.test.ts`: added `findGitRoot`/
  `findGitRootAxiomConfig` describe blocks (directory `.git`, file
  `.git` pointer, parent-walk, no-match, git-root-without-axiom-yaml).

**`@axiom/user-workspace`** (`Axiom/packages/user-workspace/`):
- `src/registry.ts`: added `findByAncestorRepoPathV2(homeDir,
  candidatePath)` — addendum-14 check 3. Matches a registered repo path
  that is EQUAL TO or an ANCESTOR OF `candidatePath` (path-segment
  prefix match with a trailing separator, not naive substring, so
  `/repos/foo` does not falsely match `/repos/foobar`). Distinct from
  the existing `findByRepoPathV2` (exact match only).
- `src/index.ts`: exported `findByAncestorRepoPathV2`.
- `tests/registry-v2.test.ts`: added a `findByAncestorRepoPathV2`
  describe block (exact match, descendant match, sibling-prefix
  non-match, no-match).

**`@axiom/project-resolution`** (`Axiom/packages/project-resolution/`):
- `package.json` / `tsconfig.json`: added `@axiom/user-workspace` as a
  dependency + TS path/project reference (no circular dependency:
  `user-workspace` only depends on `@axiom/core`/`js-yaml`).
- `src/resolver.ts`: added `resolveProjectWithFallback(startPath,
  homeDirOverride?)`, composing all 4 addendum-14 checks in the
  document's stated order. Returns `{ resolution, detectionMethod }`
  where `detectionMethod` is `'direct' | 'registered-ancestor-path' |
  'git-root'`. If direct resolution
  (`resolveProject`) is anything other than `not-found` (i.e.
  `resolved`, `ambiguous`, or `invalid-config` — an `axiom.yaml` was
  already found, just possibly invalid), the fallback is not attempted
  and `detectionMethod` is `'direct'`. If check 3 or check 4 find a
  root, that root is RE-RESOLVED with the normal, version-aware
  `resolveProject` (not just returned as a bare path) so the result is
  a fully validated `ProjectResolution`.
- `src/index.ts`: exported `resolveProjectWithFallback`,
  `DetectionMethod`, `ProjectResolutionWithDetection`.
- `tests/resolver.test.ts`: added a `resolveProjectWithFallback`
  describe block covering all 4 checks (including a "check 3 takes
  priority over check 4" case and a "registered repo exists but has no
  axiom.yaml at its root" case) and the true-failure path. Tests use
  filesystem trees nested 12 levels deep to genuinely exercise the
  fallback checks rather than incidentally passing via the existing
  10-level parent-walk in `discoverAxiomRoot`.

**`@axiom/tui`** (`Axiom/packages/tui/`):
- `src/driver.ts`: added an optional `initialMessage?: string` field to
  `RunTuiArgs`. If present, it seeds `state.lastMessage` at driver
  startup (before the first render), reusing the router's existing
  `lastMessage` field (already rendered by `renderMenuScreen` — see
  `packages/tui/src/screens/menu.ts`). No new `ScreenId`, no new
  screen file, no router change.

**`apps/cli`** (`Axiom/apps/cli/`):
- `src/commands/tui.ts`: in the default-mode branch (no
  `--projects`/`--topology`/`--model-validate`/`--components-show`),
  when `resolveProject(args.cwd).status === 'not-found'`, calls
  `resolveProjectWithFallback(args.cwd, args.homeDirOverride)` before
  throwing `projectNotFound`. If the fallback's `detectionMethod` is
  `'registered-ancestor-path'` or `'git-root'`, opens the TUI against
  the discovered root with an `initialMessage` built by
  `buildDetectionNote(detectionMethod, rootPath)` explaining how the
  project was found. If the fallback also fails (or the status was
  `ambiguous`/`invalid-config`, in which case the fallback is not
  attempted at all), throws `projectNotFound` exactly as before —
  the true-failure path is unchanged. Added `homeDirOverride?: string`
  to `TuiArgs` (test-only; not exposed as a CLI flag) so tests can
  exercise the registered-ancestor-path check without touching the
  real `~/.axiom/`.
- `tests/tui.test.ts` (new file): 4 scenarios — check 3 success, check
  4 success, true-failure (all 4 checks exhausted, `projectNotFound`
  unchanged), and `ambiguous` status not triggering the fallback. Tests
  use a non-TTY input to force `runTuiDriver` to throw `ttyRequired`
  immediately as an inline probe: since the driver's TTY check is its
  first action (before touching `rootPath`/`projectName`/
  `initialMessage`), observing `ttyRequired` instead of
  `projectNotFound` is sufficient proof that project resolution
  (including the new fallback) succeeded and control reached the
  driver with a resolved root.

### Design decisions

1. **Fallback UX mechanism**: reused the router's existing
   `lastMessage` field instead of introducing a new `ScreenId` or
   screen file. This keeps the change purely additive to the existing
   shell (per the audit's own confirmed "no shell rewrite needed"
   judgment) and avoids the complexity of a dedicated "detection
   summary" screen that OQ3 did not actually require — the brief
   asked for the user to "see what was found," which a header-level
   informational note satisfies without new navigation states.
2. **Re-resolve, don't shortcut**: when checks 3/4 find a candidate
   root, the implementation calls the full `resolveProject(root)`
   again rather than fabricating a `ProjectResolution` from the raw
   match. This guarantees the fallback path produces the exact same
   validated, version-aware shape (v1/v2 `axiom.yaml`, scopes, mode)
   as the direct path, with no special-cased partial resolution.
3. **`findGitRoot` has no depth cap** (see Assumptions) — a deliberate
   correction discovered while writing tests, scoped narrowly to that
   one new function; `discoverAxiomRoot`'s existing bound is untouched.
4. **No CLI flag for `homeDirOverride`**: it exists only on the
   `TuiArgs` TypeScript interface for test injection, mirroring the
   existing test-only `input`/`output` overrides already on that same
   interface. Production behavior always resolves the real
   `~/.axiom/`.

### Confirmed non-interference with other work

- `git status`/diff confirms no changes to `@axiom/topology`,
  `@axiom/config-validation`, `@axiom/doctor`, `@axiom/orchestrator`,
  or `apps/cli/src/commands/{init,join,repo,projects,app-api}.ts`.
- `mcp-inventory`/`memory-inventory` screen files and router entries
  are untouched (confirmed via diff review — no matches for those
  identifiers in the changed-files list).
- Pre-existing uncommitted work in `Axiom`'s working tree from prior
  increments was left untouched except where this task's own edits
  landed in the same files (`apps/cli/src/commands/tui.ts`,
  `packages/project-resolution/src/{index,resolver}.ts`,
  `packages/user-workspace/src/{index,registry}.ts`,
  `packages/filesystem-truth/src/{index,discovery}.ts`,
  `packages/tui/src/driver.ts`) — edits were made on top of existing
  content, nothing was discarded.

## Validation

Validation command discovery (per `Axiom.SDD/AGENTS.md`): `Axiom`'s
`package.json` defines `npm run typecheck` (`tsc -b`) and `npm test`
(`vitest run`). Both were run for real.

**Baseline re-confirmation** (via `git stash` / `git stash pop`
bisection, not assumed): `12 failed test files / 13 failed tests /
1231 passed` — matches the number quoted in the task brief exactly.
The single failing test in `packages/project-resolution/tests/
resolver.test.ts` (`resuelve scopes con absolutePath y exists correcto
(v1)`) is part of this pre-existing baseline, confirmed present before
any of this increment's changes.

**Post-implementation results**:

- `npm run typecheck` (`tsc -b`, full monorepo): clean, zero errors.
- `npm test` (`vitest run`, full monorepo): **12 failed test files / 13
  failed tests / 1254 passed** (up from 1231 passed at baseline — +23
  new passing tests, zero new failures, same 12 failing files as
  baseline, confirmed by diffing the `FAIL` file list against the
  stashed baseline run).
- Scoped reruns during development (`packages/filesystem-truth`,
  `packages/user-workspace`, `packages/project-resolution`,
  `apps/cli/tests/tui.test.ts`): all new tests green; the one
  pre-existing failure in `project-resolution` reproduced identically
  with and without this increment's changes.

New tests added: 4 in `filesystem-truth` (`findGitRoot`/
`findGitRootAxiomConfig`), 4 in `user-workspace`
(`findByAncestorRepoPathV2`), 8 in `project-resolution`
(`resolveProjectWithFallback`), 4 in `apps/cli` (new `tui.test.ts`,
addendum-14 fallback control flow) — 20 counted individually as `it()`
blocks across these files (some pre-existing describe blocks share
files, so the file-level new-test count differs slightly from the
raw `+23` passed delta, which also reflects a few pre-existing tests
in touched files that were not modified).

## Result

Implemented addendum-14's full 4-check detection heuristic
(cwd/parent-walk `axiom.yaml` via the pre-existing `discoverAxiomRoot`;
registered-ancestor-path via the new `findByAncestorRepoPathV2`
in `@axiom/user-workspace`; git-root via the new
`findGitRoot`/`findGitRootAxiomConfig` in `@axiom/filesystem-truth`),
composed in `@axiom/project-resolution#resolveProjectWithFallback`.
Wired the fallback into `axiom tui`'s default-mode control flow in
`apps/cli/src/commands/tui.ts`: when direct resolution fails but a
registered-ancestor-path or git-root match succeeds, the TUI now opens
against the discovered root with an informational note, instead of
unconditionally throwing `projectNotFound`. The true not-found path (all
4 checks exhausted) and the `ambiguous`/`invalid-config` paths are
unchanged. No shell rewrite was needed, confirming the audit's own
judgment; the fallback reuses the router's existing `lastMessage`
field rather than introducing a new screen. `mcp-inventory`/
`memory-inventory` (OQ1) were left untouched, as directed. Full
validation (`typecheck` + `test`) was run for real, with the baseline
re-confirmed via `git stash` bisection rather than assumed; the
implementation added 1254-1231 = 23 net passing tests with zero new
failures.

One implementation-time correction was made beyond the original brief:
`findGitRoot`'s parent-walk had to be unbounded (no reuse of
`MAX_TRAVERSAL_DEPTH`), discovered while writing tests for check 4 —
a depth-capped git-root walk made check 4 practically indistinguishable
from checks 1/2 in every reachable test scenario. This is documented
in Assumptions/Implementation notes and does not affect
`discoverAxiomRoot`'s existing behavior.

## General spec integration

No integration into `Axiom.Spec/general-spec.md` was performed: this
file does not exist in this repo yet (same situation noted by both the
parent roadmap increment and the audit increment this implements).
Once **validator-reviewer** confirms this implementation against the
audit's acceptance criteria (this increment's own status is `pending`
below), the stable knowledge worth consolidating — "addendum-14
detection order: cwd → parent-walk → registered-ancestor-path (via
`@axiom/user-workspace#findByAncestorRepoPathV2`) → git-root (via
`@axiom/filesystem-truth#findGitRootAxiomConfig`), composed in
`@axiom/project-resolution#resolveProjectWithFallback`" — should be
folded into a stable spec document at that point, not before.

## Next step recommendation

Hand off to **validator-reviewer** with this file as the input brief.
Concrete inputs for that subagent:

- This spec file:
  `Axiom.Spec/specs/increments/INC-20260702-tui-shell-detection-reconcile-impl/README.md`
- The audit spec it implements:
  `Axiom.Spec/specs/increments/INC-20260702-tui-shell-detection-reconcile/README.md`
- Changed files to review:
  `Axiom/packages/filesystem-truth/src/discovery.ts`,
  `Axiom/packages/filesystem-truth/src/index.ts`,
  `Axiom/packages/filesystem-truth/tests/discovery.test.ts`,
  `Axiom/packages/user-workspace/src/registry.ts`,
  `Axiom/packages/user-workspace/src/index.ts`,
  `Axiom/packages/user-workspace/tests/registry-v2.test.ts`,
  `Axiom/packages/project-resolution/package.json`,
  `Axiom/packages/project-resolution/tsconfig.json`,
  `Axiom/packages/project-resolution/src/resolver.ts`,
  `Axiom/packages/project-resolution/src/index.ts`,
  `Axiom/packages/project-resolution/tests/resolver.test.ts`,
  `Axiom/packages/tui/src/driver.ts`,
  `Axiom/apps/cli/src/commands/tui.ts`,
  `Axiom/apps/cli/tests/tui.test.ts` (new file).
- Ask validator-reviewer to specifically confirm: (a) no regression in
  the pre-existing `topology`/`projects`/`model-list` screen tests
  (unaffected by this change, but worth a direct assertion per the
  audit's own recommendation), (b) the `findGitRoot` depth-cap removal
  does not have unintended consequences for very large/deep
  repositories in practice (a judgment call, not something unit tests
  alone can fully validate), and (c) whether `general-spec.md` should
  be created now that two related increments (this one and its audit)
  are ready to consolidate, or deferred further.
