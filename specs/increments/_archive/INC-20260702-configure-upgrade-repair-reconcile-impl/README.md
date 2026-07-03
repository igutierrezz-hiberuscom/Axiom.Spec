# Increment: Configure/upgrade/repair operations on existing installs — repair command implementation

Status: closed
Date: 2026-07-02

Closed by `INC-20260702-configure-upgrade-repair-reconcile-validator`
(validator-reviewer, final role of the INC-22 chain), which
independently re-traced `repair.ts`'s dispatch logic and `--dry-run`
path against source and confirmed both match this increment's own
claims exactly, with no discrepancies found.

## Goal

Implement a new, general, top-level `axiom repair` CLI command in the
`Axiom` monorepo, scoped exactly to the minimal composition-only v1
recommended by the migration-engineer audit
(`INC-20260702-configure-upgrade-repair-reconcile/README.md`): run
`axiom doctor`'s existing check suite, dispatch the narrow subset of
`fail`/`warn` findings that have a known, safe, mechanical fix to the
existing functions that already implement that fix, and report
everything else as "not auto-fixable; needs manual review."

## Context

Sibling increment in the INC-22 chain. The audit
(`INC-20260702-configure-upgrade-repair-reconcile/README.md`)
confirmed:

- `axiom doctor` (`@axiom/doctor`'s `runDoctorChecks`/`checks.ts`) is
  purely diagnostic — zero fix/repair/autofix code path, `DoctorCheck`
  has no fix-related field.
- Two narrow, real `repair` subcommands already exist
  (`axiom toolchain repair`, `axiom mcp repair --id`), both scoped to
  their own domain (toolchain tool detection paths; MCP binding
  registration record). Neither implements a general "repair
  manifests/indexes/structures" operation.
- No general `axiom repair` command existed. The audit recommended
  building one by composition only, over 3 already-tested functions:
  `installProfile()` (via `runConfigure`), `axiom index rebuild` (via
  `runIndexRebuild`), and the two existing repair functions
  (`runToolchainRepair`, `runMcpRepair`) for their own domains.
- The audit explicitly flagged a separate, larger `configure`-surface
  gap (7 of 15 source-doc-§17 operations with no implementation:
  add/remove repo, add/modify role, remove tool/MCP, update guides,
  install autoskills) as OUT of this increment's scope.

Role in the roadmap sequence: **cli-implementer** (this increment).
Next roles per the audit's own brief: **tui-developer** (flow/menu
wiring) and **validator-reviewer** (final review) — neither run in
this increment.

## Scope

- New file `Axiom/apps/cli/src/commands/repair.ts`: `runRepair` +
  `formatRepairResult` + `registerRepair`, mirroring `configure.ts`/
  `upgrade.ts`'s `runX`/`registerX` split (testable without spawning
  the CLI binary or touching `commander`/`process.exit` in the body).
- Registration of `registerRepair(program)` in
  `Axiom/apps/cli/src/index.ts`, after `registerBootstrap` and before
  `registerTui` (matches the file's existing registration-order
  convention/comments).
- `--dry-run` flag (report-only, no mutation), matching `axiom
  upgrade`'s existing `--dry-run` convention.
- Tests in `Axiom/apps/cli/tests/repair.test.ts` covering: dispatch
  for each of the 4 known-fixable categories, the "not auto-fixable"
  fallback, mixed fail/warn batches, pass/skip exclusion, `--dry-run`
  (no mutation), unresolved-project error, and a smoke test using the
  real (non-injected) `runDoctorChecks`.

## Non-goals

- No new fix-logic for any doctor check category beyond the 4 already
  covered by existing functions (`install-profiles`, `artifact-index`,
  `toolchain`, `memory`/MCP-bindings). All other `fail`/`warn`
  categories (`boundaries`, `policies`, `manifests`, `isolation`,
  `governance`, `capability-model`, `gateway`, `tool-routing`,
  `topology`, `adapters`, `skills`, `agents`, `write-scope`) are
  reported verbatim as manual-review items — never attempted.
- No changes to `axiom toolchain repair`'s or `axiom mcp repair`'s own
  CLI surfaces — `repair.ts` calls their underlying `runToolchainRepair`/
  `runMcpRepair` functions, does not duplicate or replace their
  commands.
- No TUI wiring (`tui/flows/repair.ts`, `MENU_ITEMS` entry) — explicitly
  the `tui-developer`'s job per the audit's brief, not this increment.
- No attempt at the larger 7-operation `configure`-surface gap
  (add/remove repo, add/modify role, remove tool/MCP, update guides,
  install autoskills) — a separate, larger body of work per the
  audit's own framing.
- No `--json` output mode for `axiom repair` (not requested; `axiom
  doctor`/`axiom index rebuild` already have `--json` for their own
  scopes; `repair`'s v1 stays zero/minimal-flag per the audit's brief
  item 3).

## Acceptance criteria

- [x] `axiom repair` exists as a new top-level command, distinct from
      `axiom toolchain repair`/`axiom mcp repair`.
- [x] `runRepair` calls `runDoctorChecks` directly (no reimplementation
      of check logic).
- [x] Each `fail`/`warn` finding is dispatched by category to exactly
      one of the 3 buildable cases (regenerate via `installProfile()`
      through `runConfigure`; rebuild indexes via `runIndexRebuild`;
      delegate to `runToolchainRepair`/`runMcpRepair`) — 4 categories
      total, since MCP-binding-coherence findings (TC-007) are filed
      under `@axiom/doctor`'s `memory` category, not a distinct `mcp`
      category.
- [x] Every other `fail`/`warn` category is reported as "not
      auto-fixable; needs manual review," never silently skipped and
      never guessed at.
- [x] `pass`/`skip` results are never touched.
- [x] `--dry-run` reports the planned action without invoking any
      mutation (verified by asserting no filesystem side effects occur
      in dry-run for all 4 dispatchable categories).
- [x] Output summary shows, at minimum, which categories/files were
      touched (or would be touched) and which findings need manual
      review.
- [x] Tests cover dispatch (each category), the manual fallback, and
      `--dry-run`.
- [x] `npm run typecheck`, `npm run build`, and `npm test` (full
      monorepo) executed; results compared against a confirmed
      baseline.

## Open questions

None blocking. The audit's Q-repair-1 and Q-repair-2 were already
resolved by the audit itself (general `axiom repair`, reusing the two
narrow `repair` subcommands as internal building blocks; standalone
top-level command, not a `doctor` flag) — this increment followed
those resolutions as given.

## Assumptions

- The audit's "3 buildable cases" map to exactly 4 `@axiom/doctor`
  category strings once grounded in the actual `checks.ts` source:
  `install-profiles` (IP-001..004), `artifact-index` (IX-001),
  `toolchain` (TC-004/005/006), and `memory` (TC-007's MCP-bindings-
  coherence check — filed under `memory`, not a dedicated `mcp`
  category, because `checks.ts` groups TC-007/TC-008 under
  `CATEGORY_MEMORY`). This is a grounding refinement of the audit's
  prose ("missing/stale generated file", "missing derived index",
  "toolchain/MCP-binding-related failures"), not a scope deviation —
  the audit's own brief (item 2) named exactly these functions
  (`installProfile()`, `axiom index rebuild`, `repairToolchain`,
  `runMcpRepair`) as the dispatch targets; this increment confirmed
  which doctor `category` strings actually route to each.
- `axiom repair`'s `install-profiles` dispatch calls `runConfigure`
  (not `installProfile()` directly) because `runConfigure` already
  encapsulates the full re-materialization flow (`init.json` read +
  `installProfile()` + conditional opencode/copilot generators) that
  the audit's brief item 1 describes as "the same regeneration
  `configure` already performs." Calling `installProfile()` directly
  would have skipped that same flow's copilot/opencode generator
  steps, which the audit explicitly wants reused, not
  partially-duplicated.
- The pre-existing baseline of 12 failed test files / 13 failed tests
  / 1603 passed (unrelated flake: real-repo-relative fixture loads,
  TUI driver timing, retention-sweep timing) was confirmed directly
  via a full `vitest run` before any changes, matching the task
  brief's stated approximate baseline exactly.

## Implementation notes

### Command naming decision

`axiom repair` is a **new top-level command**, not a subcommand of an
existing namespace and not a flag on `axiom doctor`. Rationale,
grounded in the actual CLI namespace conventions found in
`Axiom/apps/cli/src/index.ts` and `commands/`:

- The two existing `repair` commands (`axiom toolchain repair`, `axiom
  mcp repair`) are subcommands of their own domain namespace
  (`toolchain`, `mcp`) — each repairs exactly one domain's own
  manifest-derived state. A general repair that spans multiple domains
  doesn't fit under any single existing domain namespace without
  arbitrarily picking one and implying the others are secondary.
- `axiom doctor` itself is the precedent for a general, top-level,
  cross-cutting command (it runs ALL check categories, not one
  domain's). `axiom repair` mirrors that shape: top-level, cross-
  cutting, but for fixes instead of checks. This is consistent with
  the audit's own brief item 1, which explicitly asked for
  `apps/cli/src/commands/repair.ts` (a new file, not an addition to
  `toolchain.ts`/`mcp.ts`/`doctor`'s own registration block).
- A `--repair` flag on `axiom doctor` was considered (the audit's
  Q-repair-2) but rejected by the audit itself: source doc §17 lists
  `repair` as a sibling operation type to `configure`/`upgrade`, not a
  diagnostic-mode variant. This increment followed that resolution.

### Dispatch logic design

`runRepair(args: RepairArgs): Promise<RepairResult>`:

1. Resolves the project via `resolveProject(args.cwd)`; throws with a
   message pointing at `axiom init` if unresolved (same contract
   `axiom doctor`'s own `index.ts` registration already uses).
2. Calls `doctorFn(resolution)` — defaults to `runDoctorChecks` from
   `@axiom/doctor`, reused verbatim. A `doctorFn` injection seam
   (mirroring `upgrade.ts`'s existing `syncFn`/`doctorFn` injection
   pattern) lets tests supply a synthetic `DoctorReport` without
   coupling `repair.ts`'s dispatch tests to which specific doctor
   checks fail on a given fixture — this isolates ROUTING logic (this
   increment's scope) from CHECK logic (already tested in
   `packages/doctor/tests/`).
3. Filters to `fail`/`warn` only; groups by `category` via a small
   `groupByCategory` helper (preserves first-appearance order).
4. For each category in `REPAIRABLE_CATEGORIES` (`install-profiles`,
   `artifact-index`, `toolchain`, `memory`), calls `dispatchCategory`,
   which — per category — either invokes the real fix function (real
   run) or returns a descriptive no-op outcome (`--dry-run`):
   - `install-profiles` → `runConfigure({ cwd, profilesYamlPath? })`
     (from `./configure`; `profilesYamlPath` is a pass-through test
     seam identical to `ConfigureArgs.profilesYamlPath`, never used by
     the real CLI path).
   - `artifact-index` → `runIndexRebuild({ projectRoot: cwd })` (from
     `./index-cmd`).
   - `toolchain` → `runToolchainRepair({ cwd })` (from `./toolchain`;
     reuses the CLI-level function, which itself calls
     `repairToolchain`/`repairTool` from `@axiom/toolchain` — no
     duplication of the package-level repair logic).
   - `memory` → `runMcpRepair({ manifestPath: defaultMcpManifestPath(cwd), projectRoot: cwd })`
     (from `./mcp`; reuses the CLI-level function and its default
     manifest path resolver).
5. Every other category is pushed to `manual` — never dispatched, never
   attempted.
6. Returns `{ dryRun, totalFindings, dispatched, manual }`.

`formatRepairResult` renders a human-readable summary: mode
(`OK`/`DRY-RUN`), total findings, a per-category breakdown of what was
(or would be) repaired with the underlying action's own summary line,
and a per-category breakdown of what needs manual review — satisfying
addendum §8's "show per-repo changes" spirit at the file/action
granularity available in this single-repo-default context.

### `--dry-run`

Threaded through `RepairArgs.dryRun` → `dispatchCategory`'s `dryRun`
flag. When true, each of the 4 dispatch branches returns early with a
`Re-ejecutaría …`-prefixed summary and `applied: false`, without
calling the underlying mutating function at all (not "call it and
discard the result" — the mutating call is skipped entirely). Verified
by tests asserting the relevant on-disk side effects
(`install-profile.json`, `.sdd/local/mcp-bindings.json`) do NOT appear
after a dry-run.

### Registration

`registerRepair(program)` added to
`Axiom/apps/cli/src/index.ts`, positioned after `registerBootstrap`
(INC-18) and before `registerTui`, consistent with the file's existing
"register after the newest lifecycle-adjacent command, before the TUI"
convention used by every prior increment's registration comment block.

## Validation

Validation commands discovered via `Axiom/package.json`'s `scripts`
(`typecheck`, `build`, `test`) — no README override found for these.

**Baseline** (confirmed via a full `vitest run` before any changes,
matching the task brief's stated ~12/13/~1603 baseline):

```
Test Files  12 failed | 147 passed (159)
Tests       13 failed | 1603 passed (1616)
```

All 12 baseline-failing files are pre-existing, environment-dependent
flake unrelated to this increment (real-repo-relative fixture loads
in `packages/agents`, `packages/skills`, `packages/model-routing`,
`packages/project-resolution`, `packages/doctor`; TUI driver timing in
`packages/tui/tests/driver.test.ts`; a toolchain repair fixture path
issue in `packages/toolchain/tests/repair-add-gitignore.test.ts`; a
retention-sweep timing test in `packages/telemetry`; a `start.test.ts`
scenario). Confirmed via `git stash`/`git stash pop` round-trip that
the exact same 12 files fail identically with this increment's changes
stashed out.

**After implementation**:

```
npm run typecheck   → clean (tsc -b, no errors)
npm run build       → clean (tsc -b, no errors)
npx vitest run apps/cli/tests/repair.test.ts → 12/12 passed
npx vitest run (full monorepo) →
  Test Files  12 failed | 148 passed (160)
  Tests       13 failed | 1615 passed (1628)
```

Failed-file list after implementation is byte-identical to the
baseline list (`start.test.ts`, `catalog.test.ts` ×2, `checks.test.ts`,
`assignments.test.ts`, `loader.test.ts`, `opencode-projection.test.ts`,
`resolver.test.ts` ×2, `audit-trail-sink.test.ts`,
`repair-add-gitignore.test.ts`, `driver.test.ts`). Passed count
increased by exactly +12 (the new `repair.test.ts` suite) with zero
regressions and zero new failures.

Manual CLI smoke check (`node apps/cli/dist/index.js --help`) hit a
pre-existing `MODULE_NOT_FOUND` error when executing the compiled
`dist/` output directly from `apps/cli` — confirmed via
`git stash`/`git stash pop` to reproduce identically at the last
committed baseline with this increment's changes removed. This is an
unrelated, pre-existing dist-execution/workspace-resolution issue, not
a regression introduced by `repair.ts`; `tsc -b` (the authoritative
build check) and the full `vitest` suite (which imports TypeScript
sources directly) are both green.

## Result

`axiom repair` is implemented in
`Axiom/apps/cli/src/commands/repair.ts` (`runRepair`,
`formatRepairResult`, `registerRepair`) and registered in
`Axiom/apps/cli/src/index.ts`. It runs `runDoctorChecks` (reused
verbatim), dispatches `fail`/`warn` findings from 4 known-fixable
`@axiom/doctor` categories (`install-profiles`, `artifact-index`,
`toolchain`, `memory`) to their corresponding existing fix functions
(`runConfigure`, `runIndexRebuild`, `runToolchainRepair`,
`runMcpRepair`), and reports every other category as "not
auto-fixable; needs manual review." `--dry-run` reports the same
dispatch without mutating. No new fix-logic was written for any
individual doctor check category — every fix path is a call into an
already-tested function, per the audit's explicit guardrail. 12 new
tests cover dispatch, the manual fallback, pass/skip exclusion,
dry-run non-mutation, the unresolved-project error path, and a smoke
test against the real (non-injected) doctor. Full validation
(typecheck, build, full test suite) shows zero regressions against the
confirmed pre-change baseline.

Files changed:

- `Axiom/apps/cli/src/commands/repair.ts` (new)
- `Axiom/apps/cli/tests/repair.test.ts` (new)
- `Axiom/apps/cli/src/index.ts` (2 additions: import + registration
  call for `registerRepair`)

No other files were modified. Pre-existing uncommitted changes to
`apps/cli/src/commands/configure.ts`, `init.ts`, `axiom-bug.ts`,
`axiom-increment.ts`, `axiom-plan.ts`, `tui.ts`, and others found dirty
in the working tree at the start of this increment belong to prior,
unrelated increments and were left untouched.

## General spec integration

One addition to `Axiom.Spec/general-spec.md`'s "Configure/upgrade/
repair operations" section (added by the audit increment): a note that
a general, top-level `axiom repair` command now exists
(`Axiom/apps/cli/src/commands/repair.ts`), implemented as a
composition-only dispatcher over `runDoctorChecks` findings across 4
categories (`install-profiles`, `artifact-index`, `toolchain`,
`memory`), delegating to `runConfigure`/`runIndexRebuild`/
`runToolchainRepair`/`runMcpRepair` for the actual fixes, with every
other category reported as manual-review-only. This is stable,
load-bearing, consolidated knowledge (a new command's existence and
its dispatch contract) — not implementation history — so it qualifies
for `general-spec.md` per `AGENTS.md`'s integration rules.

## Next step recommendation

Two roles remain open per the roadmap's own sequencing (unchanged from
the audit's recommendation):

1. **tui-developer**: wire `axiom repair` into the TUI — a new
   `tui/flows/repair.ts` (stateless, no confirmation logic inside the
   flow itself, mirroring `configure.ts`'s/`upgrade.ts`'s exact flow-
   file shape) plus a new `MENU_ITEMS` entry in `packages/tui/src/
   router.ts`. Should follow `upgrade.ts`'s flow pattern specifically
   (preview via `--dry-run`, then a second real call after user
   confirmation) since `repair` now has the same dry-run/real-run
   shape.
2. **validator-reviewer**: final review of this increment's dispatch
   logic, output format, and test coverage against the audit's
   original "Repair scope recommendation" and "Brief for the next
   role" sections, before closing this increment chain.

Status is `pending` (not `closed`) because these two roles have not
yet run, consistent with `AGENTS.md`'s closure rules and this
increment's own acceptance criteria (which do not claim TUI wiring or
independent review as done).
