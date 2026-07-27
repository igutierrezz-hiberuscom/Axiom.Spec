# Increment: Worktree-mode close correctness (fast-follow fix)

Status: closed
Date: 2026-07-24

## Goal

This is **INC-W8**, a fast-follow FIX increment for the two HIGH-severity
defects the adversarial review `INC-20260724-cross-cutting-e2e-review`
(INC-X1) found AND reproduced (not just deduced) in the worktree-mode
`axiom-role complete` close flow, on top of an otherwise-shipped and
independently-verified Axiom evolution batch (Clusters A/T/S/W, all closed).
Fix both defects with the least invasive correct changes, without
re-opening W1–W7 internals beyond what the fixes require, and without
changing in-place-mode behavior at all.

## Context

Read first: `Axiom.SDD/AGENTS.md` (canonical lifecycle), the worktree
cluster plan (`giggly-imagining-moore.md`, Cluster W + "Pegas"), and INC-X1's
own spec (`Axiom.Spec/specs/increments/INC-20260724-cross-cutting-e2e-review/
README.md`) plus its e2e file
(`apps/cli/tests/e2e/cross-cutting-batch.e2e.test.ts`, TEST 2 and TEST 3),
which empirically reproduced both defects below.

Defect sites (all in `Axiom.SDD`, now `Axiom`):
- `apps/cli/src/commands/axiom-role.ts` — `runRoleSubcommand`'s `complete`
  review/verify gate wiring (~line 606-651) and `runRoleSubcommandAsync`'s
  `complete` branch (~line 951-990, worktree close wiring).
- `apps/cli/src/commands/_worktree-execution.ts` — `closeWorktreeExecution`.
- `packages/persistence/src/execution-harvest.ts` —
  `harvestAndCleanupExecution`'s `worktreeRemove`/`force` step.
- `packages/workflow/src/git/git-services.ts` — `worktreeRemove`'s dirty
  refusal.
- `apps/cli/src/commands/workspace-worktree-provision.ts` —
  `provisionWorktreeExecution` (rewrites the worktree's native MCP config to
  the worktree's own absolute path; reports `mcpConfig.path` + adapter
  surface info).
- `apps/cli/src/commands/_role-review.ts` — `runRoleReview`'s `gitDiff(...)`
  call.

## Scope

ONLY the two HIGH fixes below, plus their regression tests.

### FIX 1 — worktree-mode `complete` cannot close a normally-provisioned worktree, and leaves inconsistent state on failure

Root cause: provisioning (INC-W3) legitimately rewrites the worktree's
native MCP config (and adapter-surface outputs) to the worktree's own
absolute path — a genuinely modified *tracked* file — so `worktreeRemove`'s
dirty guard hard-stops on close, even with NO real uncommitted agent work.
Worse: `runRoleSubcommand`'s synchronous core already transitions+persists
the role to `archived` BEFORE the async close runs, so a close failure
leaves `archived` + an orphaned worktree, exit 1.

- **FIX 1a — neutralize provisioning's own generated files before the dirty
  check.** `provisionWorktreeExecution` now reports (and persists onto
  `Execution.provisionedPaths`) EXACTLY the paths it wrote/rewrote (native
  MCP config path + adapter-surface `writtenFiles`), relative to
  `worktreePath`. `harvestAndCleanupExecution` neutralizes exactly that set
  (`resetWorktreeGeneratedFiles`, `@axiom/workflow`: revert tracked paths via
  `git checkout --`, delete untracked ones) immediately before
  `worktreeRemove`'s dirty check — mirroring INC-W4's
  `teardownWorktreeCodeIntel` precedent (clear derived/regenerable state
  before the check, never touch anything else). Genuine uncommitted work
  (any path NOT in the provisioning-written set) still hard-stops removal —
  the safety guard is unchanged.
- **FIX 1b — state/close consistency.** `runRoleSubcommandAsync`'s `complete`
  branch now peeks the pre-transition `vars` before running the synchronous
  core, so it can resolve the `Execution` up front. If the subsequent
  worktree close still hard-stops (genuine dirty work), the branch reverts
  the JUST-persisted `archived` transition back to `fromState` (normally
  `in-progress`) via a compensating `saveWorkflowState` write — so the
  process never ends with a persisted `archived` role state next to an
  orphaned worktree. The `Execution` itself is left at `harvested` (not
  rolled back — harvest already ran for real); a later `complete` retry (or
  the existing, non-CLI-exposed `closeWorktreeExecution(..., {force:true})`
  escape hatch) resumes cleanly from there.

### FIX 2 — verify/review gates validate `projectRoot`, never the worktree

Root cause: on `complete`, `runRoleReview(args.projectRoot, ...)` and
functional-verify (`repoRoot: ... ?? args.projectRoot`) always target the
source repo, so in worktree mode a worktree whose tests FAIL still passes
the gate.

Fix: `runRoleSubcommandAsync`'s `complete` branch resolves the `Execution`
(same pre-transition peek as FIX 1b) BEFORE calling the synchronous core,
and — only when the role is worktree-mode — injects
`execution.worktreePath` as `reviewDeps.repoRoot`/`verifyDeps.repoRoot`.
`_role-review.ts`'s `RoleReviewDeps` gained a new, optional `repoRoot`
field used ONLY for the `gitDiff` call (topology identity/plan resolution
still uses `projectRoot` — a worktree isn't itself a registered repo).
`_functional-verify.ts` already had an injectable `repoRoot` seam; this
increment is the first caller to actually set it in worktree mode. In-place
mode is fully unchanged (the peek finds no worktree execution, so the args
passed to the synchronous core are byte-identical to before).

## Non-goals

- The MODERATE finding (manifest schema duplication) and the LOW findings
  (stale dist `.d.ts`, execution-store JSON.parse validation, Windows
  MAX_PATH) from INC-X1's review — separately-tracked follow-ups, untouched.
- Any change to in-place-mode behavior.
- Re-opening W1–W7 internals beyond what these two fixes need.
- A new CLI-exposed `--force-close` flag. The brief allows one as an
  ADDITIONAL operator escape hatch but explicitly forbids it as the primary
  fix; FIX 1a already makes a normal, non-dirty worktree close with no
  force, so adding CLI surface for the force path was judged unnecessary
  scope for this increment. The primitive-level escape hatch
  (`closeWorktreeExecution(execution, {store, force:true})`) already existed
  pre-fix and remains available, unchanged.
- Migrating Axiom's own repos (`Axiom.SDD`/`Axiom.Spec`).

## Acceptance criteria

- [x] A normally-provisioned worktree (real `axiom configure` + a committed
      native MCP config that provisioning rewrites) with NO real agent work
      closes cleanly on `complete`, with no `force`: exit 0, `archived`,
      worktree removed, `Execution` state `removed`.
- [x] A worktree with genuine uncommitted work (a file provisioning never
      wrote) alongside provisioning's own rewritten files STILL refuses
      removal — no data loss, safety guard intact.
- [x] That refusal does not leave the role state persisted `archived` next
      to an orphaned worktree — the on-disk role state reverts to
      `in-progress` (verified via a fresh, independent state-file read).
- [x] A worktree-mode role whose worktree has a FAILING test script fails
      the `complete` functional-verify gate (`repoRoot` targets the
      worktree), even though the source repo's own committed script passes.
- [x] In-place mode behavior is provably unchanged.
- [x] `npm run build` (tsc -b) passes.
- [x] Full `npx vitest run` stays green (pre-existing vs new failures
      classified).

## Open questions

None blocking.

## Assumptions

1. **Rollback-on-failure (FIX 1b), not gate reordering.** Reordering the
   worktree close to run BEFORE the review/verify gates was considered and
   rejected: it would let a worktree be harvested+removed even when a
   SUBSEQUENT review/verify failure means the role should not have
   completed at all — a worse regression (destroying an in-progress
   worktree pre-emptively) than the narrow, single-process, pre-return
   window during which `archived` is persisted then immediately reverted
   if the close fails. The compensating-rollback approach also avoids
   touching `runRoleSubcommand`'s documented synchronous, unconditional-
   persist contract, which other callers (MCP `sdd.transitionApply`,
   `app-api.ts`) rely on unchanged.
2. **`Execution.provisionedPaths` persistence, not re-derivation.** Because
   `complete` can run in a separate process from `start`, the exact set of
   provisioning-written paths is persisted onto the `Execution` record
   itself (`ExecutionStore.update`'s patch gained `provisionedPaths`) rather
   than recomputed at close time from `agentTarget`/install-profile
   heuristics — the persisted set is exactly what was written, not a
   best-guess reconstruction.
3. **No CLI-exposed force-close flag added** — see Non-goals. The
   pre-existing primitive-level escape hatch is unchanged and sufficient.

## Implementation notes

Files changed:
- `packages/isolation/src/execution.ts` — `Execution.provisionedPaths?`.
- `packages/persistence/src/execution-store.ts` —
  `UpdateExecutionPatch.provisionedPaths?`, wired into `update()`.
- `apps/cli/src/commands/sync.ts` — `MaterializeAdapterOutputsResult` gains
  `writtenFiles` (verbatim from each adapter generator), all 8 branches.
- `apps/cli/src/commands/workspace-worktree-provision.ts` —
  `ProvisionWorktreeExecutionResult.adapterSurface.writtenFiles` +
  top-level `provisionedPaths` (adapter surface + relative mcpConfig path);
  persisted via `store.update(..., {state:'provisioned', provisionedPaths})`.
- `packages/workflow/src/git/git-services.ts` — new
  `resetWorktreeGeneratedFiles` primitive (git-tracked paths reverted via
  `git checkout --`, untracked ones deleted; paths outside the worktree
  refused, never silently dropped). Exported from the git module barrel and
  the package's top-level barrel.
- `packages/persistence/src/execution-harvest.ts` — new step (c.5)
  `resetProvisionedArtifacts`, between teardown (c) and `worktreeRemove`
  (d); `HarvestAndCleanupResult.provisioningArtifactsReset`.
- `apps/cli/src/commands/_role-review.ts` — `RoleReviewDeps.repoRoot?`,
  used only for the `gitDiff` call.
- `apps/cli/src/commands/axiom-role.ts` — `runRoleSubcommandAsync`'s
  `complete` branch: pre-transition peek + Execution resolution BEFORE the
  synchronous core runs (FIX 2's `reviewDeps.repoRoot`/`verifyDeps.repoRoot`
  injection); compensating rollback of the persisted transition on a close
  hard-stop (FIX 1b).

Tests added/extended:
- `packages/workflow/tests/git-worktree-services.test.ts` —
  `resetWorktreeGeneratedFiles` unit coverage (reverts tracked, deletes
  untracked, refuses paths outside the worktree, absent path is a no-op).
- `packages/persistence/tests/execution-harvest.test.ts` — neutralizes
  `execution.provisionedPaths` before the dirty check (closes with no
  force); a genuine extra dirty file (not in `provisionedPaths`) still
  hard-stops.
- `apps/cli/tests/_role-review.test.ts` — `reviewDeps.repoRoot` overrides
  `gitDiff`'s target while identity/plan resolution keeps using
  `projectRoot`.
- `apps/cli/tests/e2e/cross-cutting-batch.e2e.test.ts` — TEST 2 updated to
  assert the FIX (clean close, no force) instead of the original finding,
  plus a new sibling test proving genuine dirty work still blocks and the
  role state is correctly rolled back (not archived+orphaned); TEST 3
  updated to assert verify now targets the worktree and correctly fails.

## Validation

Ran from `C:\repos\Axiom Workspace\Axiom`:

- `npm run build` (`tsc -b`) — PASSED, clean, exit 0 (re-run multiple times
  across the iteration, always clean).
- Targeted `npx vitest run` for every touched scope — ALL GREEN:
  - `packages/workflow/tests/git-worktree-services.test.ts` — 27/27 (22
    pre-existing + 5 new, `resetWorktreeGeneratedFiles`).
  - `packages/persistence/tests/execution-harvest.test.ts` — 12/12 (8
    pre-existing + 4 new, FIX 1a orchestration-level neutralization).
  - `apps/cli/tests/_role-review.test.ts` — 8/8 (5 pre-existing + 3 new,
    FIX 2's `repoRoot` deps override).
  - `apps/cli/tests/workspace-worktree-provision.test.ts` — 10/10 (widened
    assertions on the new `writtenFiles`/`provisionedPaths` fields).
  - `apps/cli/tests/e2e/cross-cutting-batch.e2e.test.ts` — 4/4 (TEST 1
    unchanged; TEST 2 split into "closes cleanly" + "genuine dirty work
    still blocks + FIX 1b rollback"; TEST 3 updated to assert the fix). Ran
    twice back-to-back for stability — both green.
  - `apps/cli/tests/axiom-role.test.ts`, `axiom-role-worktree.test.ts`,
    `worktree-execution.test.ts` — all pre-existing tests green, unchanged.
- Full `npx vitest run` (whole monorepo, no path filter) — **317 files, 3218
  tests: 3214 passed, 4 failed** (all `Test timed out in 5000ms`, in
  `member-install.test.ts`, `workspace-incremental.test.ts` (x2),
  `workspace-setup.test.ts`). Test-file count matches INC-X1's own
  last-recorded baseline (317); the +13 test count is exactly this
  increment's additions (5+4+3+1, accounted for above). All 4 failures
  re-ran GREEN in isolation (`member-install.test.ts` +
  `workspace-incremental.test.ts` + `workspace-setup.test.ts` together:
  64/64 passed) — confirmed pre-existing parallel-load flakiness (same
  class already documented by the brief for `context.test.ts`/
  `workspace-setup.test.ts`; this run additionally surfaced
  `member-install.test.ts`/`workspace-incremental.test.ts` under the same
  contention), not a regression. None of the 5 historically-flaky files
  touch worktree-close/provisioning/review/verify code.

See the executor's final report (returned to the requester) for verbatim
tails.

## Result

Implemented. Both HIGH defects are fixed with the least invasive changes
identified in Assumptions above; regression tests reproduce the ORIGINAL
failing scenarios and now assert the fixed behavior, plus the required
safety-net scenario (genuine dirty work still blocks). See the executor's
final report for verbatim validation output.

## General spec integration

Integrado en la spec canónica al cierre del batch INC-20260724-* (integración
única de toda la tanda). Este incremento corrige 2 defectos en el
comportamiento de cierre en worktree ya integrado; el conocimiento estable
del fix se refleja en:

- `04_Flujos_SDD_y_Ciclo_de_Vida.md` — subsección "Correctitud del cierre en
  worktree": neutralización de los ficheros generados por provisioning antes
  del dirty check (el trabajo genuino sigue bloqueando), rollback
  compensatorio de la transición `archived`, y gates verify/review apuntando a
  `execution.worktreePath` en modo worktree.
- `07_Gobierno_y_Seguridad.md` — cleanup seguro (reset de ficheros generados +
  rollback compensatorio para no dejar `archived` + worktree huérfano).
- `03_Modelo_Operativo_y_Datos.md` — `Execution.provisionedPaths` (set exacto
  de ficheros escritos por el provisioning, persistido para el cierre).
