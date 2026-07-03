# Increment: Configure/upgrade/repair operations on existing installs — validator review

Status: closed
Date: 2026-07-02

## Goal

Independently review the full INC-22 `configure`/`upgrade`/`repair`
chain (migration-engineer audit → cli-implementer implementation →
tui-developer implementation) by tracing code directly rather than
trusting prior increments' prose, decide and act on the newly
root-caused ANSI-color-code bug in `printPostRunSummary`, update the
already-tracked `cli-commands` tsconfig dist-emission bug's blast
radius, run fresh full-monorepo validation, and — if all of INC-22's
acceptance criteria are met — close INC-22 as a whole.

## Context

Final role of INC-22, per the roadmap's own sequencing. Three sibling
increments precede this one, in order:

1. `INC-20260702-configure-upgrade-repair-reconcile` (migration-engineer
   audit, audit-only): confirmed no general `axiom repair` existed;
   recommended a minimal, composition-only v1 dispatching to 4
   already-tested functions.
2. `INC-20260702-configure-upgrade-repair-reconcile-impl`
   (cli-implementer): implemented `axiom repair`
   (`Axiom/apps/cli/src/commands/repair.ts`).
3. `INC-20260702-configure-upgrade-repair-reconcile-tui`
   (tui-developer): wired `repair` into the TUI (`packages/tui/src/
   flows/repair.ts`, `router.ts`'s `MENU_ITEMS`, `driver.ts`), extended
   `@axiom/cli-commands`'s barrel and `tsconfig.json`, and root-caused
   (but did not fix) a pre-existing ANSI-color-code substring bug in
   `render.ts`'s `printPostRunSummary`.

## Scope

- Independently trace `repair.ts`'s dispatch logic: confirm each of
  the 4 categories calls the right function with correct arguments,
  and confirm the manual-review fallback never attempts an unsupported
  fix.
- Independently trace `--dry-run`'s code path to confirm zero mutating
  calls.
- Independently review the TUI flow wiring (menu position,
  `REQUIRES_PREVIEW`/`REQUIRES_CONFIRMATION` consistency with
  `upgrade`'s pattern).
- Decide on and act on the ANSI-color-code bug (fix now if narrow/safe,
  or file a new bug ticket otherwise).
- Check the `@axiom/cli-commands` tsconfig seam extension against the
  already-tracked `BUG-20260702-cli-commands-tsconfig-missing-emit`
  ticket; update it if the blast radius changed.
- Run `npm run typecheck`, `npm run build`, `npm test` (full monorepo)
  and independently confirm results against the documented baseline.
- Close INC-22 as a whole if acceptance criteria are met, naming
  deferred items explicitly.
- Update `general-spec.md`'s "Configure/upgrade/repair operations"
  section.

## Non-goals

- No changes to `axiom repair`'s dispatch logic, output format, or CLI
  contract.
- No attempt at the larger 7-of-15 `configure`-surface gap (add/remove
  repo, add/modify role, remove tool/MCP, update guides, install
  autoskills) — explicitly deferred to a future increment by every
  prior increment in this chain.
- No redesign of `packages/cli-commands/tsconfig.json`'s file
  ownership (the actual fix for the tracked dist-emission bug) — out
  of scope per that bug's own "fix scope (not done here)" section;
  this increment only updates the ticket's blast-radius accounting.

## Acceptance criteria

- [x] `repair.ts`'s dispatch logic independently traced (not
      re-cited from prior specs): all 4 categories
      (`install-profiles`, `artifact-index`, `toolchain`, `memory`)
      confirmed to call the correct function with correct arguments;
      manual fallback confirmed to never attempt an unsupported fix
      (switch has no default branch; anything outside
      `REPAIRABLE_CATEGORIES` goes straight to `manual`).
- [x] `--dry-run` independently traced: each of the 4 dispatch
      branches returns early with `applied: false` before the
      mutating call, never invoking it (not "call and discard").
- [x] TUI flow wiring independently reviewed: `repair`'s menu position
      (index 4, between `upgrade` and `model-list`) and
      `REQUIRES_PREVIEW`/`REQUIRES_CONFIRMATION` membership (both,
      alongside `upgrade`) confirmed consistent with `upgrade`'s risk
      profile and justified by the tui-developer's own rationale
      (contiguous "install lifecycle" grouping, source doc §17's own
      operation ordering).
- [x] ANSI-color-code bug read, root cause independently confirmed
      (not just re-cited), and a fix-now-vs-ticket decision made and
      acted upon.
- [x] `cli-commands` tsconfig seam extension checked against
      `BUG-20260702-cli-commands-tsconfig-missing-emit`; blast-radius
      change (8 → 11 files) independently reproduced and the ticket
      updated.
- [x] `npm run typecheck`, `npm run build`, `npm test` (full monorepo)
      independently re-run; results compared against the documented
      baseline.
- [x] INC-22 closure assessed against its own goal and the three
      sibling increments' acceptance criteria, with explicit rationale
      and explicit naming of deferred items.
- [x] `general-spec.md`'s "Configure/upgrade/repair operations"
      section updated to reflect final state.

## Open questions

None blocking.

## Assumptions

- The documented baseline (12 failed files / 13 failed tests / 1603
  passed pre-chain, 1631 passed after the `-tui` increment) is accurate
  as reported by the two prior implementation increments — verified
  independently in this review by re-running the full suite before and
  after this increment's own change (the ANSI-bug fix).
- Fixing the two pre-existing `driver.test.ts` assertions (`configure`,
  `upgrade`) via the already-established `stripAnsi` helper is within
  this role's scope: it is the direct, safe, narrow fix for a bug this
  same INC-22 chain root-caused (not a pre-existing, unrelated
  increment's regression), and leaving a known, deterministic,
  root-caused test bug unfixed when the fix is a two-line, zero-risk
  change would not meet AGENTS.md's "prefer simple, useful, validated
  changes" guidance.

## Implementation notes

### 1. `repair.ts` dispatch logic — independently traced, confirmed correct

Read `Axiom/apps/cli/src/commands/repair.ts` in full (fresh read, not
reused from prior increment context). Confirmed directly from source:

- `REPAIRABLE_CATEGORIES` is a `ReadonlySet` of exactly 4 strings:
  `install-profiles`, `artifact-index`, `toolchain`, `memory`.
- `runRepair` groups `fail`/`warn` checks by `category`
  (`groupByCategory`), then for each category: if it is in
  `REPAIRABLE_CATEGORIES`, calls `dispatchCategory`; otherwise pushes
  straight to `manual` — there is no code path where an
  unrecognized/unsupported category reaches `dispatchCategory`.
- `dispatchCategory`'s `switch` has exactly 4 `case` branches (one per
  `RepairableCategory` union member) and **no `default` branch** — this
  is exhaustive over the union by construction (TypeScript would flag
  a missing case at compile time if the union grew), so there is no
  silent-fallthrough risk.
- Each branch calls the exact function the spec claims:
  `install-profiles` → `runConfigure({ cwd, profilesYamlPath? })`
  (imported from `./configure`); `artifact-index` →
  `runIndexRebuild({ projectRoot: cwd })` (from `./index-cmd`);
  `toolchain` → `runToolchainRepair({ cwd })` (from `./toolchain`);
  `memory` → `runMcpRepair({ manifestPath: defaultMcpManifestPath(cwd),
  projectRoot: cwd })` (from `./mcp`). Arguments passed match each
  target function's own signature (confirmed by the fact that this
  compiles cleanly under `tsc -b`, and by the dedicated dispatch tests
  below asserting real filesystem side effects from each call).
- Cross-checked against `apps/cli/tests/repair.test.ts`'s Scenario 1
  (4 dispatch tests, each asserting `result.dispatched[0]?.outcome
  .applied === true` and a real, distinct filesystem side effect per
  category — `install-profile.json` for `install-profiles`,
  `.sdd/local/mcp-bindings.json` for `memory`) — this test suite
  independently corroborates the trace without this review needing to
  trust the prior increment's own narrative.

**Conclusion**: the dispatch logic is correct exactly as documented by
the cli-implementer increment. No discrepancy found.

### 2. `--dry-run` — independently traced, confirmed zero mutating calls

Each of the 4 branches in `dispatchCategory` starts with
`if (args.dryRun) { return { ...applied: false }; }` **before** the
line that calls the real mutating function (`runConfigure`,
`runIndexRebuild`, `runToolchainRepair`, `runMcpRepair`). This is an
early return, not a "call and ignore result" pattern — the mutating
call is textually unreachable when `dryRun` is true. Confirmed by
reading all 4 branches directly (lines 156-227 of `repair.ts`).

Cross-checked against `apps/cli/tests/repair.test.ts`'s Scenario 4
(3 dry-run tests): asserts `install-profile.json` does NOT exist after
a dry-run with an `install-profiles` finding, and
`.sdd/local/mcp-bindings.json` does NOT exist after a dry-run with a
`memory` finding — both run against the real (non-mocked)
`runConfigure`/`runMcpRepair` functions, so a real mutation would have
been caught. Ran this test file independently
(`npx vitest run apps/cli/tests/repair.test.ts`): 12/12 passed.

**Conclusion**: `--dry-run` makes zero mutating calls, confirmed both
by direct code trace and by independently re-running the existing
filesystem-effect assertions.

### 3. TUI flow wiring — independently reviewed, confirmed consistent

Read `Axiom/packages/tui/src/router.ts`'s `MENU_ITEMS` directly: 7
entries — `configure`, `sync`, `doctor`, `upgrade`, `repair`,
`model-list`, `exit`. `repair` is at index 4, between "Aplicar upgrade
del runtime" and "Model routing" — matches the tui-developer's own
documented, non-escalated design choice exactly.

Read `Axiom/packages/tui/src/driver.ts`'s `REQUIRES_CONFIRMATION`
(`configure`, `upgrade`, `repair`) and `REQUIRES_PREVIEW` (`configure`,
`sync`, `upgrade`, `repair`) sets directly. `repair` is in both sets,
alongside `upgrade` — `sync` is in `REQUIRES_PREVIEW` only (no
confirmation gate), which is a meaningful, correct distinction: `sync`
regenerates adapter outputs without composing over multiple
potentially-destructive operations the way `repair`/`upgrade` do.
`repair`'s inclusion in both sets is justified: it mutates the
filesystem by composition over `installProfile`/`index rebuild`/
`toolchain repair`/`mcp repair`, the same risk class as `upgrade`
(which also mutates via a multi-step process with rollback). This
review finds `repair`'s classification correct, not merely asserted.

The menu position choice (grouping `configure`/`upgrade`/`repair`
contiguously as the "install lifecycle" block, keeping `model-list`'s
unrelated LLM-routing concern after it) is a reasonable, low-stakes UX
call consistent with source doc §17's own operation ordering. This
review does not override it — no functional or architectural issue
found with the choice, and the tui-developer's own note confirms it is
a one-line, low-risk change if a future reviewer disagrees.

**Conclusion**: TUI wiring is consistent with `upgrade`'s pattern by
design, correctly justified, and requires no changes.

### 4. ANSI-color-code bug — root cause independently confirmed, fixed now

Read `Axiom/packages/tui/src/render.ts`'s `stylePostRunLine` (lines
130-157) directly. Confirmed the tui-developer's root-cause claim and
found it is **broader** than the single `OK:` case originally
described: `stylePostRunLine` splits *every* prefix-matched branch —
`OK:`, `ERROR:`, `Restore point:`, `Follow-ups:`, `→`, and `[Enter]` —
into two separately-`paint()`-ed segments (prefix in one color, the
remainder of the line in another), so any test asserting a contiguous
plain-text substring that crosses one of these prefix boundaries fails
deterministically. This is intentional visual design (two-color
prefix/body styling), not an application logic defect — real users see
correctly colored output; the bug only manifests in test assertions
that inspect the raw ANSI-laden stream with `toContain` across a color
boundary.

Ran `packages/tui/tests/driver.test.ts` independently before any
change: confirmed exactly 2 failures, matching the tui-developer's
count —

```
FAIL  runTui — B3: preview + confirm + yes (configure) > ... post-run summary
  AssertionError: expected '...' to contain '[Enter] volver al menú'
FAIL  runTui — B3: preview + upgrade con confirmación (real upgrade) > ...
  AssertionError: expected '...' to contain 'OK: Upgrade OK'
```

Note the `configure` test's actual failing assertion is
`'[Enter] volver al menú'` (line 413), not the `'OK:'`-only assertion
at line 408 (which does not cross a color boundary and already
passed) — the tui-developer's own note attributed both failures to the
`OK:`/`upgrade` case specifically; this review found the `configure`
failure is a sibling instance of the same mechanism via the `[Enter]`
branch, not the `OK:` branch. Same root cause, different trigger line.

**Decision**: fix now. This is narrow (2 test files' worth of
assertions), safe (test-only change, zero production code touched,
`render.ts`'s visual design is explicitly preserved), and the fix
pattern is already established and proven in this same codebase (the
`-tui` increment's own `repair` driver test already uses a local
`stripAnsi` helper for the identical problem). Filing a new bug ticket
for a two-line, already-patterned, zero-risk test fix would be
disproportionate process overhead — this does not rise to the bar that
warrants a tracked ticket (contrast with the `cli-commands` tsconfig
bug below, which requires an actual production-code redesign and has
already been deliberately deferred three times for that reason).

**Action taken**: applied the existing `stripAnsi` helper (already
defined in `driver.test.ts` for the `repair` test) to both pre-existing
assertions:

- `configure` test (was: `const text = output.text();` at post-run
  summary assertion) → `const text = stripAnsi(output.text());`.
- `upgrade` test (was: `const text = output.text();` before
  `toContain('OK: Upgrade OK')`) → `const text =
  stripAnsi(output.text());`.

Both edits include an inline comment explaining the root cause and
cross-referencing the sibling case, so a future reader does not need to
re-derive it. Re-ran `packages/tui/tests/driver.test.ts` after the fix:
21/21 passed (0 failures, up from 19/21).

### 5. `cli-commands` tsconfig seam extension — blast radius independently checked, widened

Read `Axiom/packages/cli-commands/tsconfig.json`'s `include` list
directly: 11 cross-included `apps/cli/src/commands/*.ts` files today
(`_shared`, `configure`, `sync`, `upgrade`, `model`, `components`,
`index-cmd`, `validate-changes`, `repair`, `toolchain`, `mcp`) — the
last 3 were added by the `-tui` increment specifically to satisfy the
TS project-references graph for `repair.ts`'s own imports of
`runToolchainRepair`/`runMcpRepair`.

Independently reproduced the already-tracked
`BUG-20260702-cli-commands-tsconfig-missing-emit` defect against this
new state: ran `rm -rf apps/cli/dist packages/cli-commands/dist
*.tsbuildinfo` followed by a clean `npm run build` (exit 0, zero
errors). Confirmed `apps/cli/dist/commands/{repair,toolchain,mcp}.js`
are absent (alongside the previously documented 8 files), and
`packages/cli-commands/dist/apps/cli/src/commands/
{repair,toolchain,mcp}.js` are emitted instead — byte-for-byte the same
ownership/skip mechanism already described in the bug ticket, now
affecting 3 more files.

**Decision**: same root cause, not a distinct issue — no new ticket
filed. Updated the existing
`Axiom.Spec/bugs/BUG-20260702-cli-commands-tsconfig-missing-emit/
README.md` with a new "Second rediscovery" section documenting the
8 → 11 file count change, the specific 3 new files, and the
confirmation method, plus updated the "Actual behavior" and "Fix
scope" sections' file counts and rediscovery-count language ("three
times" → "four times").

## Validation

Validation commands discovered via `Axiom/package.json`'s `scripts`
(`typecheck`, `build`, `test`) — consistent with both prior
increments in this chain, no README override found.

**Independently re-confirmed baseline** (documented by prior
increments as 12 failed files / 13 failed tests / 1631 passed, after
the `-tui` increment's 16 new tests):

```
npx vitest run (full monorepo, before this increment's ANSI-bug fix) →
  Test Files  12 failed | 148 passed (160)
  Tests       13 failed | 1631 passed (1644)
```

Matches the `-tui` increment's own documented post-implementation
numbers exactly — confirmed independently, not re-cited.

**After this increment's fix** (2 pre-existing `driver.test.ts`
assertions patched with `stripAnsi`):

```
npm run typecheck   → clean (tsc -b, no errors)
npm run build       → clean (tsc -b, no errors; independently verified
                       via a full clean rebuild — all dist/ and
                       tsbuildinfo files deleted first — used for the
                       tsconfig blast-radius check in section 5 above)

npx vitest run packages/tui/tests/driver.test.ts →
  Test Files  1 passed (1)
  Tests       21 passed (21)   [was 19 passed | 2 failed]

npx vitest run apps/cli/tests/repair.test.ts →
  12/12 passed (unchanged; this increment did not touch repair.ts)

npx vitest run (full monorepo) →
  Test Files  11 failed | 149 passed (160)
  Tests       11 failed | 1633 passed (1644)
```

The 11 remaining failed files are the same pre-existing,
environment-dependent flake documented by every prior increment in
this chain (real-repo-relative fixture loads in `packages/agents`,
`packages/skills`, `packages/model-routing` ×4, `packages/
project-resolution`, `packages/doctor`; a `start.test.ts` scenario; a
toolchain repair fixture path issue in `packages/toolchain/tests/
repair-add-gitignore.test.ts`; a retention-sweep timing test in
`packages/telemetry`) — independently diffed against the documented
12-file baseline list and confirmed to be exactly that list minus
`driver.test.ts` (the file this increment fixed). Passed count
increased by exactly **+2** (the two assertions fixed), with **zero
regressions and zero new failures**.

## Result

Independent review confirms the full INC-22 chain's implementation
matches its specs with no discrepancies:

1. `repair.ts`'s dispatch logic is correct — all 4 categories call the
   right function with the right arguments, and the manual-review
   fallback structurally cannot attempt an unsupported fix (no
   `default` branch in an exhaustive `switch`).
2. `--dry-run` makes zero mutating calls — verified both by direct code
   trace (early-return-before-mutation pattern in all 4 branches) and
   by re-running the existing filesystem-effect assertions.
3. TUI flow wiring (menu position, `REQUIRES_PREVIEW`/
   `REQUIRES_CONFIRMATION`) is consistent with `upgrade`'s risk
   profile and correctly justified — no changes needed.
4. The ANSI-color-code bug in `printPostRunSummary` was root-caused
   more precisely (the bug affects every prefix-styled line, not just
   `OK:`) and **fixed** — a narrow, test-only, zero-risk change using
   the already-established `stripAnsi` pattern. `render.ts`'s visual
   design was left untouched (correct — it is intentional, not a
   defect).
5. `BUG-20260702-cli-commands-tsconfig-missing-emit`'s blast radius was
   independently reproduced and confirmed to have widened from 8 to 11
   files (adding `repair.ts`, `toolchain.ts`, `mcp.ts`); the existing
   ticket was updated in place — no new ticket filed, same root cause.
6. Full validation (typecheck, build, full test suite) is green modulo
   the same pre-existing, unrelated flake documented since the
   `-impl` increment — now with 2 fewer failures than the chain's own
   documented baseline, thanks to this increment's own fix.

## Closure summary — INC-22 as a whole

**INC-22 (`configure`/`upgrade`/`repair` operations on existing
installs) is closed.** All four AGENTS.md closure-rule conditions are
satisfied across the full chain:

- **Goal is clear**: confirmed/corrected whether a `repair` operation
  exists (source doc §17), and — since it did not — built a minimal,
  composition-only v1 and wired it end-to-end (CLI + TUI).
- **Acceptance criteria exist and are met**: all 4 sibling increments'
  acceptance criteria (audit, CLI implementation, TUI wiring, this
  validation) are checked off, independently re-verified by this role
  rather than taken on faith.
- **Changes were implemented**: `Axiom/apps/cli/src/commands/repair.ts`
  (new), `Axiom/apps/cli/src/index.ts` (registration),
  `Axiom/packages/tui/src/flows/repair.ts` (new), `router.ts`/
  `driver.ts`/`preview.ts`/`postrun.ts`/`types.ts` (TUI wiring),
  `Axiom/packages/cli-commands/src/index.ts` +`tsconfig.json` (seam
  extension), plus this increment's own fix to `packages/tui/tests/
  driver.test.ts`.
- **Available validation was executed**: `typecheck`/`build`/`test`
  run fresh by this role, not re-cited from prior increments.
  Zero regressions found; 2 fewer pre-existing failures than the
  chain's own baseline, thanks to this increment's fix.
- **Review against intent and acceptance criteria was done**: sections
  1-5 above, all independently traced against source rather than
  trusting prior increments' prose.
- **Stable knowledge integrated into `general-spec.md`**: done (see
  below).

**Deferred items, named explicitly (not silently dropped)**:

1. **The larger 7-of-15 `configure`-surface gap** (source doc §17
   operations with no confirmed implementation: add repo, remove repo,
   add role, modify role, remove tool/MCP, update guides, install
   autoskills in repo) — flagged by the original migration-engineer
   audit, re-flagged by the cli-implementer and tui-developer
   increments, and re-confirmed here as **out of scope for INC-22**.
   This is a distinct, larger body of work (`configure`'s current
   single-shot "re-apply the whole profile" design has no add/remove
   surface at all) and should be scoped as its own future increment,
   not folded into any `repair`-labeled work — `repair` and this gap
   are orthogonal (one fixes drift in what's already configured, the
   other adds new configuration surface).
2. **`BUG-20260702-cli-commands-tsconfig-missing-emit`'s actual fix**
   (redesigning `packages/cli-commands/tsconfig.json`'s file ownership)
   — deliberately not attempted here; this increment only updated the
   ticket's blast-radius accounting per its own "fix scope (not done
   here)" section. Recommended for prioritization independent of any
   specific feature increment, given it has now grown 4 times across 4
   increments.
3. **D1's deprecation-warning mechanism** (`defaultSingleRepoManifest`'s
   silent v1-manifest fallback) — pre-existing open item from INC-01's
   D1/D3 split, unrelated to INC-22's own scope, unaffected by this
   chain.

## General spec integration

Updated `Axiom.Spec/general-spec.md`'s existing "Configure/upgrade/
repair operations" section (in place, not appended as new content):

1. Changed the section's opening framing from "audited by INC-22,
   pending" to "INC-22 is closed," summarizing the full 4-role chain.
2. Updated the `axiom repair implemented`/`reachable from the TUI`
   bullets to drop "pending — validator-reviewer still open" language,
   replacing it with a note that dispatch logic and TUI wiring were
   independently re-traced and confirmed correct by this role.
3. Added a new bullet documenting the ANSI-color-code bug's precise
   root cause and its fix (test-only, `stripAnsi`, `render.ts`
   untouched).
4. Added a new bullet documenting the `cli-commands` tsconfig bug's
   blast-radius widening (8 → 11 files) with a pointer to the updated
   ticket.

This is stable, load-bearing, consolidated knowledge (final state of a
now-closed increment's commands, TUI wiring, and two bug tickets'
current status) — not implementation history — so it qualifies for
`general-spec.md` per `AGENTS.md`'s integration rules.

## Next step recommendation

With INC-22 closed, **Phase G (Versionado y upgrades) of the parent
roadmap is complete**: INC-20 (`@axiom/versioning` reconcile, closed),
INC-21 (migration engine/upgrade commands reconcile, closed), the D3
side-quest (`axiom.yaml schemaVersion: 2`/multi-repo-primary, closed
and independently validated per
`INC-20260702-schemaversion2-multirepo-primary-reconcile-validator`),
and now INC-22 (configure/upgrade/repair reconcile, closed) are all
resolved. No open Phase G items remain.

Per the parent roadmap's own sequencing, the recommended next step is
**INC-23 ("Dogfooding boundary enforcement")** — the last cross-cutting
increment before INC-24 (Workbench, deferred/optional). This
recommendation follows the roadmap's stated ordering directly; this
increment did not independently re-verify INC-23's own scope or
readiness (that is INC-23's own migration-engineer role's job, not
this validator-reviewer's).
