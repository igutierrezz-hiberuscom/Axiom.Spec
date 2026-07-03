# Increment: Configure/upgrade/repair operations on existing installs — repair TUI wiring

Status: closed
Date: 2026-07-02

Closed by `INC-20260702-configure-upgrade-repair-reconcile-validator`
(validator-reviewer, final role of the INC-22 chain), which
independently confirmed the menu position and
`REQUIRES_PREVIEW`/`REQUIRES_CONFIRMATION` wiring, fixed the
ANSI-color-code bug this increment root-caused (see the validator
increment's own spec), and updated
`BUG-20260702-cli-commands-tsconfig-missing-emit`'s blast-radius
accounting for the 3 new `tsconfig.json` `include` entries this
increment added.

## Goal

Wire the `axiom repair` CLI command (landed by the sibling `-impl`
increment) into the interactive TUI (`Axiom/packages/tui`): a new
`repair` menu entry, a stateless flow file mirroring `upgrade.ts`'s
exact dry-run-preview-then-confirm shape, and the seam extension in
`@axiom/cli-commands` needed for the TUI to call `runRepair`/
`formatRepairResult` without importing `apps/cli/src` directly
(INC-04's layering rule).

## Context

Sibling increment in the INC-22 chain (`configure`/`upgrade`/`repair`
operations on existing installs). Prior increments in this chain:

- `INC-20260702-configure-upgrade-repair-reconcile` (migration-engineer
  audit): confirmed `axiom repair` did not exist as a general,
  top-level command; recommended a minimal, composition-only v1.
- `INC-20260702-configure-upgrade-repair-reconcile-impl`
  (cli-implementer): implemented `axiom repair`
  (`Axiom/apps/cli/src/commands/repair.ts` — `runRepair`,
  `formatRepairResult`, `registerRepair`), registered as a top-level
  `axiom repair [--dry-run]` command. Dispatches `axiom doctor`'s
  fail/warn findings across 4 categories (`install-profiles`,
  `artifact-index`, `toolchain`, `memory`) to
  `runConfigure`/`runIndexRebuild`/`runToolchainRepair`/`runMcpRepair`;
  everything else is reported as "not auto-fixable; needs manual
  review." Left TUI wiring and final review explicitly open, per its
  own "Next step recommendation."

Role in the roadmap sequence: **tui-developer** (this increment). Next
role: **validator-reviewer** (final review across the full chain).

## Scope

- Read `Axiom/packages/tui/src/flows/upgrade.ts` in full to confirm
  the exact dry-run-preview-then-confirm shape to mirror.
- Read `Axiom/packages/tui/src/router.ts`'s `MENU_ITEMS` fresh (it had
  grown to 6 items by C3b's "Model routing" entry, not 5 as an older
  audit assumed) and `@axiom/cli-commands`'s barrel.
- Add `runRepair`/`formatRepairResult` re-exports to
  `@axiom/cli-commands` (`packages/cli-commands/src/index.ts`),
  mirroring the existing `upgrade`/`index-cmd`/`validate-changes`
  re-export pattern (AD-01: trivial re-export, no new logic).
- Add `packages/cli-commands/tsconfig.json`'s `include` entries for
  `repair.ts` + its own further imports (`toolchain.ts`, `mcp.ts`) —
  required by the TS project-references graph, same pattern already
  used for `configure.ts`/`sync.ts`/`upgrade.ts`/etc.
- Implement `packages/tui/src/flows/repair.ts` (`runRepairFlow`)
  mirroring `upgrade.ts`'s flow shape 1:1: stateless, `dryRun` param
  (default `true`), `FlowResult` with mode-dependent `message`.
- Add `repair` to `router.ts`'s `ScreenId` union and `MENU_ITEMS`
  (positioned after `upgrade`, before `model-list`).
- Add `repair` to `flows/types.ts`'s `FlowRunners` (thin projected
  signature, same pattern as `upgrade`).
- Wire `repair` into `driver.ts`: `runFlowFor`'s switch, the default
  lazy-loaded runner set (`cliRunRepair` from `@axiom/cli-commands`),
  `REQUIRES_CONFIRMATION`, `REQUIRES_PREVIEW`, `FLOW_LABELS`, and the
  `previewFlowName` cast used when building the post-run summary.
- Add `computeRepairPreview` to `flows/preview.ts` (mirrors
  `computeUpgradePreview`: delegates to the same injected
  `runners.repair` with `dryRun: true`), plus a `repair` entry in
  `GATES_BY_FLOW` and `PREVIEW_COMPUTERS`.
- Add a `repair` branch to `flows/postrun.ts`'s `buildPostRunSummary`
  follow-up switch (exhaustive over `PreviewFlowName`, so TypeScript
  forces this once `repair` joins the union).
- Tests: new `runRepairFlow` suite in `flows.test.ts` (mirrors the
  `runUpgradeFlow` suite 1:1), new `computeRepairPreview` suite +
  `repair` follow-up tests in `preview.test.ts`, a `repair`
  preview-confirm-postrun integration test (plus a cancel-path test)
  in `driver.test.ts`, and updates to the two `router.test.ts`/
  `driver.test.ts` tests whose hardcoded menu-index assertions shifted
  because of the new menu entry.

## Non-goals

- No changes to `axiom repair`'s CLI contract, dispatch logic, or
  output format (`Axiom/apps/cli/src/commands/repair.ts` untouched;
  read-only reference for this increment).
- No changes to `axiom toolchain repair`'s or `axiom mcp repair`'s own
  surfaces.
- No attempt at the larger 7-of-15 `configure`-surface gap the audit
  flagged separately (add/remove repo, add/modify role, remove
  tool/MCP, update guides, install autoskills) — out of scope for
  every increment in this chain so far.
- No new confirmation-screen or preview-screen renderer logic:
  `screens/confirm.ts` and `screens/summary.ts` are already generic
  over `ScreenId`/`FlowResult` and needed zero changes; `render.ts`'s
  `printPreview`/`printPostRunSummary` are already generic over
  `PreviewPayload`/`PostRunSummary` and needed zero changes.
- No `--json` mode, no new CLI flags on the TUI side — `repair`'s TUI
  entry has exactly the same shape as `upgrade`'s (dry-run preview,
  Y/n confirm, real run, post-run summary).
- Did not fix the two pre-existing, already-baseline-documented
  ANSI-color-code assertion issues in `configure`'s and `upgrade`'s
  own driver tests (see "Validation" below) — out of scope; flagged
  for whoever picks up test-infrastructure cleanup next.

## Acceptance criteria

- [x] `packages/tui/src/flows/upgrade.ts` read in full before writing
      `repair.ts`.
- [x] `router.ts`'s `MENU_ITEMS` re-confirmed fresh (6 items pre-change:
      configure/sync/doctor/upgrade/model-list/exit — INC-04's original
      5-item claim was already stale before this increment, due to
      C3b's "Model routing" addition; this increment's own note
      corrects that drift for whoever reads the roadmap next).
- [x] `@axiom/cli-commands` barrel extended with `runRepair`/
      `formatRepairResult` re-exports, mirroring the existing
      `upgrade`/`index-cmd`/`validate-changes` pattern — TUI never
      imports `apps/cli/src` directly.
- [x] `packages/tui/src/flows/repair.ts` implemented, mirroring
      `upgrade.ts`'s flow shape (stateless, `dryRun` param, mode-
      dependent `message`).
- [x] `repair` added to `MENU_ITEMS` and wired into `driver.ts`'s
      dispatch, following `upgrade`'s exact pattern (preview → confirm
      → real run → post-run summary).
- [x] Tests written/updated for: the new flow file, the new preview
      computer, the post-run follow-ups, the menu/router wiring, and
      the seam re-export (exercised transitively via `packages/tui`'s
      tests, since `@axiom/cli-commands` has no dedicated test
      directory — consistent with the existing pattern for
      `upgrade`/`index-cmd`/`validate-changes`, none of which have
      dedicated `cli-commands` tests either).
- [x] `npm run typecheck`, `npm run build`, and `npm test` (full
      monorepo, plus scoped `packages/tui` and `packages/cli-commands`)
      executed; results compared against a freshly confirmed baseline.

## Open questions

None blocking. One design choice made without escalation, documented
below under "Implementation notes — design choices made without
escalation."

## Assumptions

- `MENU_ITEMS` already had 6 entries (not 5) before this increment,
  because C3b's "Model routing" entry landed after INC-04's TUI
  detection work but the roadmap's audit trail hadn't been corrected.
  This increment's own read of `router.ts` (step 2 of the task brief)
  confirmed the actual pre-change list directly rather than trusting
  either the roadmap's or the audit's older "5 items" framing.
- `packages/cli-commands` has no dedicated test directory. This is
  consistent with every prior seam addition (`upgrade`, `index-cmd`,
  `validate-changes` — none have `cli-commands`-scoped tests either);
  the barrel's re-exports are exercised transitively through
  `packages/tui`'s test suite, which imports `@axiom/cli-commands` for
  its default runners. Adding a dedicated `cli-commands` test suite
  for a one-line re-export would be speculative scope beyond what any
  prior increment in this chain established, so none was added here.
- The 12-failed-file/13-failed-test pre-existing baseline (documented
  by the `-impl` increment) was re-confirmed directly by this
  increment via a full `vitest run` before any TUI changes, and found
  identical (same file list, same test names).

## Implementation notes

### Flow file (`flows/repair.ts`) — mirrors `upgrade.ts` 1:1

`runRepairFlow(ctx, runners, dryRun = true)`:

- Same two-mode shape as `runUpgradeFlow`: `dryRun: true` runs the
  read-only preview path; `dryRun: false` runs the real dispatch.
- Same injection pattern: `runners.repair` is optional; if absent,
  returns `{ ok: false, error: 'No hay runner de \`repair\` inyectado.' }`
  (identical wording style to `upgrade`'s own error).
- One deliberate divergence from `upgrade.ts`, called out explicitly
  in the flow's own doc comment: `ctx.projectName` is **not**
  required. `runUpgradeFlow` requires it because `runUpgrade` needs it
  explicit; `runRepair`'s actual signature
  (`Axiom/apps/cli/src/commands/repair.ts`) only takes `{ cwd,
  dryRun, doctorFn?, profilesYamlPath? }` — it resolves the project
  internally via `resolveProject(args.cwd)`, the same pattern
  `configure`/`doctor` already use. Adding a `projectName` requirement
  that the underlying function doesn't need would have been an
  unrequested behavior change, not a faithful mirror.
- `message` shape: `Repair {DRY-RUN|OK}. totalFindings=N.
  dispatched=D. manual=M` — mirrors `upgrade`'s
  `Upgrade {DRY-RUN|OK}. from V1 to V2. ...` convention (mode prefix,
  then the counts the operator needs).

### Seam extension (`@axiom/cli-commands`)

Trivial re-export, no new logic (AD-01 of the barrel's own header
comment):

```ts
export {
  runRepair,
  formatRepairResult,
} from '../../../apps/cli/src/commands/repair';
export type {
  RepairArgs,
  RepairResult,
  RepairableCategory,
} from '../../../apps/cli/src/commands/repair';
```

`packages/cli-commands/tsconfig.json`'s `include` list needed 3 new
entries, not just `repair.ts`: TypeScript's project-references graph
requires every file `repair.ts` imports to also be explicitly listed
(`toolchain.ts`, `mcp.ts`, since `repair.ts` calls
`runToolchainRepair`/`runMcpRepair`). This is the same constraint the
barrel's existing entries already satisfy transitively (e.g.
`components.ts`/`model.ts` don't reach into other command files the
way `repair.ts` does, so they didn't need this).

### Preview computer (`flows/preview.ts`)

`computeRepairPreview` mirrors `computeUpgradePreview` exactly: calls
`ctx.runners.repair({ cwd, dryRun: true })`, maps the result to a
`PreviewPayload`. Differences from upgrade's mapping, driven by
`RepairResult`'s actual shape (not a stylistic choice):

- `affectedFiles` lists the **dispatched categories** (e.g.
  `['artifact-index', 'toolchain']`), not filenames — `RepairResult`
  doesn't enumerate individual files the way `UpgradePlan.touchedFiles`
  does; categories are the closest equivalent "what will be touched"
  signal available.
- `checkpointToCreate` is always `null` — `repair` does not create a
  checkpoint (unlike `upgrade`, which always does when non-no-op).
  This is a factual property of `runRepair`, not an oversight.
- `warnings` includes a "nothing to do" warning when
  `totalFindings === 0`, mirroring upgrade's "no-op" warning for
  `plan.isNoOp`.

### Driver wiring (`driver.ts`)

`repair` joins `upgrade` in exactly two sets (`REQUIRES_CONFIRMATION`,
`REQUIRES_PREVIEW`) because it shares upgrade's risk profile: it
mutates the filesystem by composition over
`installProfile`/`index rebuild`/`toolchain repair`/`mcp repair`. The
driver's confirm-branch cast (`flowName as 'configure' | 'sync' |
'upgrade' | 'doctor'`, used when calling `buildPostRunSummary`) was
widened to include `'repair'` — this is the only place the driver
needed a type-level touch beyond adding `repair` to the various
`Record`/`Set` literals.

### Design choices made without escalation

**`repair`'s menu position**: placed immediately after `upgrade`
("Aplicar upgrade del runtime") and before `model-list` ("Model
routing"), rather than at the end of the list (before `exit`). Made
without escalation because: (1) it groups the three "install
lifecycle" operations (configure/upgrade/repair) contiguously, keeping
`model-list`'s unrelated concern (LLM routing config) after the
install-lifecycle block rather than between two members of it; (2) it
mirrors source doc §17's own listed operation order
(`configure`/`upgrade`/`repair` as siblings) referenced by the audit
increment. This shifted `model-list`'s menu index from 4 to 5 and
`exit`'s from 5 to 6 — reflected in the two hardcoded-index test
updates below. If a reviewer prefers `repair` at the end of the list
instead, this is a one-line `MENU_ITEMS` reorder with no other code
impact (the router and driver are index-agnostic; only the two
hardcoded-index tests would need re-updating).

### Test updates required by the new menu entry

Two pre-existing tests hardcoded menu indices that shifted:

- `router.test.ts`'s `'clampea exactamente al tamaño del menú menos
  uno'` asserted `MENU_ITEMS.length === 6`; updated to `7`.
- `driver.test.ts`'s exit-via-empty-line test asserted
  `selectedIndex: 5` reaches `exit`; updated to `6`.
- `driver.test.ts`'s two C3b model-list tests (`press 3`, `press 0`)
  selected menu item `'4'` expecting `model-list`; updated to `'5'`
  (repair is now index 4).

### Unrelated build pollution found and removed

Before any TUI wiring could be tested, `packages/tui/tests/flows.test.ts`,
`driver.test.ts`, `projects.test.ts`, and `topology.test.ts` all failed
with `Cannot find module './configure'` from a require stack rooted at
`apps/cli/src/commands/repair.js`. Root cause: three stray compiled
artifacts (`repair.js`/`.js.map`/`.d.ts`, `toolchain.js`/`.js.map`/
`.d.ts`, `mcp.js`/`.js.map`/`.d.ts`) were sitting directly inside
`apps/cli/src/commands/` (not `dist/`) — untracked build output from
an earlier ad-hoc `tsc` invocation during the `-impl` increment's own
work, left behind because `.gitignore` only excludes `dist/` folders,
not stray compiled siblings inside `src/`. These shadowed the `.ts`
sources for Node's CJS module resolution once Vitest's transform
picked up `repair.js` as a real CJS module instead of transforming
`repair.ts`. Removed all 9 stray files (they were untracked — `git
status` showed `??`, confirming they were never part of the tracked
source tree). This was necessary cleanup to get an accurate signal on
the TUI wiring; it is not a change to any tracked file.

## Validation

Validation commands discovered via `Axiom/package.json`'s `scripts`
(`typecheck`, `build`, `test`) — same as the sibling `-impl` increment,
no README override found.

**Baseline** (re-confirmed via a full `vitest run` before any TUI
changes, matching the `-impl` increment's own documented baseline
exactly):

```
Test Files  12 failed | 148 passed (160)
Tests       13 failed | 1615 passed (1628)
```

Failed-file list confirmed byte-identical to the `-impl` increment's
own documented list (`start.test.ts`, `catalog.test.ts` ×2,
`checks.test.ts`, `assignments.test.ts`, `loader.test.ts`,
`opencode-projection.test.ts`, `resolver.test.ts` ×2,
`audit-trail-sink.test.ts`, `repair-add-gitignore.test.ts`,
`driver.test.ts`) — all pre-existing, environment-dependent flake
unrelated to this increment (real-repo-relative fixture loads, a
toolchain repair fixture path issue, a retention-sweep timing test,
a `start.test.ts` scenario).

**`driver.test.ts`'s own 2 baseline failures — root cause identified
(not just "timing")**: the `-impl` increment's baseline notes
attributed `driver.test.ts`'s 2 failures (`configure` and `upgrade`
preview-confirm-postrun tests) to "TUI driver timing." This increment
traced the actual cause while building an analogous `repair` test:
`render.ts`'s `printPostRunSummary` paints the `OK:`/`ERROR:` prefix
and the rest of the result message with different ANSI color codes
(`render.ts` line ~137-138), so the raw output stream never contains
`OK: Upgrade OK` as a contiguous substring — an ANSI reset+color
sequence sits between `OK:` and `Upgrade`. `expect(text).toContain('OK:
Upgrade OK')` therefore fails deterministically, not intermittently,
confirmed by 3 consecutive local re-runs producing the identical 2
failures every time. This increment's own new `repair` driver test
uses a local `stripAnsi` helper (same pattern already used in
`preview.test.ts`) to avoid the same false failure; the two
pre-existing `configure`/`upgrade` tests were left untouched (fixing
them is a test-infrastructure cleanup outside this increment's scope,
flagged in "Non-goals").

**After implementation**:

```
npm run typecheck  → clean (tsc -b, no errors)
npm run build      → clean (tsc -b, no errors)

npx vitest run packages/tui →
  Test Files  1 failed | 7 passed (8)
  Tests       2 failed | 143 passed (145)
  (the 2 failures are the pre-existing configure/upgrade driver tests
  above — confirmed present before this increment's changes too)

npx vitest run packages/cli-commands →
  No test files found (consistent with every prior seam addition —
  see "Assumptions")

npx vitest run apps/cli/tests/repair.test.ts →
  12/12 passed (unchanged; this increment did not touch repair.ts)

npx vitest run (full monorepo) →
  Test Files  12 failed | 148 passed (160)
  Tests       13 failed | 1631 passed (1644)
```

Failed-file list after implementation is byte-identical to the
baseline list (diffed directly, zero differences). Passed count
increased by exactly **+16** (6 `runRepairFlow` tests + 4
`computeRepairPreview` tests + 4 `repair` post-run follow-up tests + 2
`repair` driver integration tests), with **zero regressions and zero
new failures** beyond the pre-existing, now-root-caused baseline.

## Result

`axiom repair` is now reachable from the interactive TUI. New file
`Axiom/packages/tui/src/flows/repair.ts` (`runRepairFlow`) mirrors
`upgrade.ts`'s exact flow shape (stateless, `dryRun`-gated,
`FlowResult` with a mode-dependent message). `repair` was added to
`router.ts`'s `MENU_ITEMS` (position 4, between "Aplicar upgrade del
runtime" and "Model routing") and wired through `driver.ts`'s full
preview → confirm → run → post-run-summary pipeline, joining `upgrade`
in `REQUIRES_CONFIRMATION`/`REQUIRES_PREVIEW` since both mutate the
filesystem. `@axiom/cli-commands` gained a trivial `runRepair`/
`formatRepairResult` re-export (plus 3 new `tsconfig.json` `include`
entries needed to satisfy the TS project-references graph), so the
TUI never imports `apps/cli/src` directly, preserving INC-04's
layering rule. 16 new tests cover the flow, the preview computer, the
post-run follow-ups, and full driver integration (including a cancel
path). Full validation (typecheck, build, full test suite) shows the
failed-file list is byte-identical to the freshly re-confirmed
baseline — zero regressions.

Files changed:

- `Axiom/packages/cli-commands/src/index.ts` (seam extension: `runRepair`/`formatRepairResult` re-export)
- `Axiom/packages/cli-commands/tsconfig.json` (3 new `include` entries: `repair.ts`, `toolchain.ts`, `mcp.ts`)
- `Axiom/packages/tui/src/flows/repair.ts` (new: `runRepairFlow`)
- `Axiom/packages/tui/src/flows/types.ts` (`FlowRunners.repair` added)
- `Axiom/packages/tui/src/flows/preview.ts` (`computeRepairPreview`, `GATES_BY_FLOW.repair`, `PREVIEW_COMPUTERS.repair`, `PreviewableFlowName` widened)
- `Axiom/packages/tui/src/flows/postrun.ts` (`computeRepairFollowUps` + switch case)
- `Axiom/packages/tui/src/router.ts` (`ScreenId` + `MENU_ITEMS` gained `repair`)
- `Axiom/packages/tui/src/driver.ts` (import, default runner, `REQUIRES_CONFIRMATION`/`REQUIRES_PREVIEW`/`FLOW_LABELS`, `runFlowFor` switch, confirm-branch cast)
- `Axiom/packages/tui/tests/flows.test.ts` (new `runRepairFlow` suite)
- `Axiom/packages/tui/tests/preview.test.ts` (new `computeRepairPreview` suite + `repair` follow-up tests)
- `Axiom/packages/tui/tests/driver.test.ts` (new `repair` integration tests + cancel-path test; 2 pre-existing hardcoded-index assertions updated; new `stripAnsi` helper; `repair` stub added to `makeStubRunners`)
- `Axiom/packages/tui/tests/router.test.ts` (`MENU_ITEMS.length` assertion updated 6 → 7)

Also removed (untracked build pollution, not a tracked-file change):
`Axiom/apps/cli/src/commands/{repair,toolchain,mcp}.{js,js.map,d.ts}`
(9 stray compiled files blocking correct test resolution — see
"Implementation notes — Unrelated build pollution found and removed").

No other files were modified. Pre-existing uncommitted changes found
dirty in the working tree at the start of this increment (`apps/cli/
src/commands/configure.ts`, `init.ts`, `axiom-bug.ts`,
`axiom-increment.ts`, `axiom-plan.ts`, `tui.ts`,
`apps/cli/tests/tui.test.ts`, `packages/cli-commands/package.json`,
and ~100 others) belong to prior, unrelated increments and were left
untouched, consistent with the `-impl` increment's own note about the
same pre-existing dirty state.

## General spec integration

One addition to `Axiom.Spec/general-spec.md`'s "Configure/upgrade/
repair operations" section (added by the audit increment, extended by
the `-impl` increment): a note that `axiom repair` is now also
reachable from the interactive TUI (`packages/tui/src/flows/repair.ts`,
menu entry "Reparar instalación"), following the same dry-run-preview-
then-confirm shape as `axiom upgrade`. This is stable, load-bearing,
consolidated knowledge (a new command's TUI reachability) — not
implementation history — so it qualifies for `general-spec.md` per
`AGENTS.md`'s integration rules.

## Next step recommendation

One role remains open per the roadmap's own sequencing: **validator-
reviewer**, to do a final, fresh-context review of the full INC-22
`configure`/`upgrade`/`repair` chain (audit → CLI implementation → TUI
wiring) against the original audit's "Repair scope recommendation" and
"Brief for the next role" sections, and to make an explicit call on
the one design choice this increment made without escalation (menu
position of `repair` — see "Implementation notes — Design choices made
without escalation"). The validator-reviewer should also decide
whether the newly root-caused ANSI-color-code assertion bug in
`configure`'s/`upgrade`'s pre-existing `driver.test.ts` tests warrants
its own small follow-up (it is a real, deterministic bug, not flake —
see "Validation" — but fixing pre-existing tests unrelated to `repair`
is outside this increment's own scope).

Status is `pending` (not `closed`) because the validator-reviewer role
has not yet run, consistent with `AGENTS.md`'s closure rules.
