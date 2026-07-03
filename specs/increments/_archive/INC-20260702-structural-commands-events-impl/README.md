# Increment: Structural command foundation and event model (registry-engineer implementation)

Status: pending
Date: 2026-07-02

## Goal

Execute the **registry-engineer** step of INC-02's subagent sequence
(migration-engineer -> registry-engineer -> validator-reviewer), per the
parent roadmap
(`Axiom.Spec/specs/increments/INC-20260702-axiom-redesign-roadmap/README.md`,
"INC-02 — Structural command foundation and event model") and the
migration-engineer audit
(`Axiom.Spec/specs/increments/INC-20260702-structural-commands-events/README.md`):
implement the audit's "Registry-engineer brief" items 1-6 in
`Axiom/`, exactly as scoped — an optional `onTransition` hook on the
already-gated `runCommand`/`RunResult` flow, and a lightweight
`RepoRegistered` notification at the already-ungated `addProjectV2`
call sites — without gating INC-01's registry writes and without
touching `CommandId`/`STATE_MACHINE`/`gateFor`.

## Context

Parent chain: `INC-20260702-axiom-redesign-roadmap` (parent roadmap) ->
`INC-20260702-registry-manifest-schema-v2` chain (INC-01, five roles,
closed) -> `INC-20260702-structural-commands-events` (INC-02,
migration-engineer audit, `pending`) -> this increment (INC-02,
registry-engineer, second of three roles).

This increment's contract is the migration-engineer audit's own
"Registry-engineer brief" (items 1-6, end of its Implementation notes)
and "Next step recommendation" section, both read in full before any
code change. Per the task instructions, the audit's corrections to the
parent roadmap's assumptions are treated as settled, not re-litigated:

- `@axiom/orchestrator`'s `CommandId` union has 27 members (8
  lifecycle-shaped + 19 intent-shaped), not the roadmap's assumed 22
  (7+15).
- `apps/cli/src/commands/intent.ts` is architecturally unrelated to the
  orchestrator's 19 intent-command stubs — not touched by this
  increment.
- INC-01's registry writes (`addProjectV2`/`saveRegistryV2`/etc.) are
  **not** routed through `runCommand`/`gateFor` — doing so would
  contradict the shipped 0020-C1 decision (project-scoped gate vs.
  user-level registry, deliberately separate) and would require a new
  precondition axis the state machine does not have today (audit's
  Finding 4).

## Scope

- Implement the audit's "Registry-engineer brief" items 1-6 verbatim:
  1. Do not implement any of the 19 `CommandId` intent stubs.
  2. Do not route `addProjectV2`/`saveRegistryV2`/etc. through
     `runCommand`/`gateFor`.
  3. Add an optional, synchronous `onTransition?: (trace: TraceEntry)
     => void` 4th parameter to `runCommand` (and forward it through
     `runOrchestrated`), invoked once after a successful `fn()` and
     before `runCommand` returns.
  4. Add a lightweight, synchronous `RepoRegistered` notification
     directly at the existing, already-working `addProjectV2` call
     sites (`init.ts`, `join.ts`, `repo.ts`, `projects.ts` — 5 call
     sites total), matching the existing best-effort
     `registryAttempt`/`[axiom <cmd>] WARN: ...` logging pattern — no
     gate, no new abstraction.
  5. Do not build speculative hooks for the six source-doc events with
     no live write path confirmed in the audit's scope
     (`PlanCreated`, `PlanLinkedToIncrement`, `AdrLinked`,
     `TechnicalContextUpdated`, `SkillInstalled`,
     `IncrementStatusChanged`).
  6. No new event-bus/pub-sub package, no `TraceEntry` persistence, no
     change to `CommandId`, `STATE_MACHINE`, `gateFor`, or any existing
     precondition.
- Add/extend tests in `packages/orchestrator/tests/` and
  `apps/cli/tests/` following existing conventions, preserving the
  integrity of `state-machine.test.ts`'s 7+19 documentation comment.
- Run real validation (`npm run typecheck`, `npm run build`, `npm
  test`, full monorepo + scoped) and cross-check the failure set
  against the known INC-01-chain baseline (12 failed files / 13 failed
  tests).
- Write this increment spec documenting implementation, validation,
  and any flagged mismatches.

## Non-goals

- No redesign of the state machine, `CommandId` union, or gate
  mechanism (per the audit's own non-goals, carried forward).
- No implementation of any of the 19 stubbed intent commands.
- No routing of registry/topology writes through `runCommand`/`gateFor`.
- No event-bus framework, message queue, pub/sub abstraction, or
  `TraceEntry` persistence.
- No hooks for `PlanCreated`/`PlanLinkedToIncrement` (INC-06),
  `AdrLinked` (INC-11), `TechnicalContextUpdated` (INC-10),
  `SkillInstalled` (INC-09), or the generic `IncrementStatusChanged`.
- No change to `apps/cli/src/commands/intent.ts` (out of INC-02's
  scope per the audit's Finding 2 and the roadmap's own INC-17).
- No notification wired to `useProjectV2` (`projects use`) — the
  brief's item 4 names `addProjectV2` call sites specifically, and
  "use" is not a registration event; see "Flagged mismatches" below
  for the explicit judgment call this required.

## Acceptance criteria

- [x] `runCommand` accepts an optional 4th parameter
      `onTransition?: (trace: TraceEntry) => void`, invoked exactly
      once, synchronously, after a successful `fn()` and before
      `runCommand` returns `{ result, trace }`.
- [x] `onTransition` is NOT invoked on gate failure (Scenario 5 path)
      or on `fn` throwing (Scenario 7 path) — only on the success path.
- [x] `runOrchestrated` (`apps/cli/src/commands/_shared.ts`) forwards
      an optional 4th `onTransition` parameter to `runCommand`
      unchanged; no existing caller's behavior changes (all existing
      3-arg call sites keep working — confirmed by full test suite).
- [x] A lightweight `notifyRepoRegistered` function emits a one-line,
      best-effort, non-throwing stderr notification
      (`event=RepoRegistered source="..." projectId="..." role="..."`)
      at all 5 `addProjectV2` call sites (`init.ts`, `join.ts`,
      `repo.ts`, `projects.ts` x2), called only after
      `addResult.ok === true`.
- [x] No `CommandId`, `STATE_MACHINE`, `gateFor`, or `Precondition` was
      modified.
- [x] `addProjectV2`/`saveRegistryV2`/`useProjectV2` were NOT routed
      through `runCommand`/`gateFor` anywhere.
- [x] `state-machine.test.ts`'s 7+19 documentation comment and
      `LIFECYCLE_COMMANDS` (8 entries) are untouched and still pass.
- [x] New/updated tests exist in `packages/orchestrator/tests/runner.test.ts`
      and `apps/cli/tests/{_shared,init,repo,projects}.test.ts`.
- [x] `npm run typecheck` and `npm run build` pass with zero errors.
- [x] `npm test` (full monorepo) failure set matches the known baseline
      (12 failed files / 13 failed tests) exactly — no new failures, no
      unexpected passes masking a regression.
- [x] Any mismatch between the audit's brief and live code encountered
      during implementation is flagged explicitly, not silently
      resolved (see "Flagged mismatches").

## Open questions

None blocking this increment's own closure. One judgment call was made
explicitly (see "Flagged mismatches" #1: `useProjectV2`/`projects use`
scope) rather than left open, since the brief's own wording (item 4
names `addProjectV2` call sites specifically) gives enough signal to
decide without escalating.

Carried forward, unresolved, not this increment's job: the six
source-doc events with no live write path (item 5), and whether to
extend the notification pattern to `axiom-increment.ts`/`axiom-bug.ts`'s
`create` subcommands (the brief's own item 5 explicitly leaves this as
"a judgment call for registry-engineer to make explicitly" — this
increment does NOT extend there, keeping scope to the one event
(`RepoRegistered`) with a confirmed, already-implemented live write
path, consistent with the brief's minimal framing).

## Assumptions

- "Lightweight, synchronous notification... a plain callback or a
  one-line console/trace log... not a new abstraction" (brief item 4)
  is implemented as a single exported function
  (`notifyRepoRegistered`) called directly at each call site — not a
  callback parameter threaded through `runRepoAttach`/`runProjectsAdd`/
  etc.'s public signatures, since none of those functions currently
  accept behavioral callbacks and adding one would be a bigger surface
  change than the brief's "one-line log" framing implies.
- `onTransition`'s error-propagation behavior (a throwing
  `onTransition` is NOT caught by the runner) is a deliberate design
  choice consistent with the runner's existing philosophy (`fn`
  failures are re-thrown, not swallowed) — no consumer of the hook
  exists yet, so this is a forward-looking contract decision, not an
  audit requirement; flagged for validator-reviewer's attention.

## Implementation notes

### Item 3 — `onTransition` hook on `runCommand`/`runOrchestrated`

`packages/orchestrator/src/runner.ts`: `runCommand<T>` gained a 4th
optional parameter `onTransition?: (trace: TraceEntry) => void`,
documented as AD-07. It is invoked synchronously, once, immediately
after `fn()` resolves successfully and the `pass`-outcome `TraceEntry`
is constructed — before `return { result, trace }`. It is deliberately
**not** invoked on the gate-failure path (AD-05's `fn` never runs, so
there is no real transition to notify) nor on the `fn`-throws path
(Scenario 7's existing stderr `FAIL` log already covers that case, and
"transition" implies success). A throwing `onTransition` is not
try/caught — it propagates, since a broken callback is the caller's
bug, not something the runner should mask.

`apps/cli/src/commands/_shared.ts`: `runOrchestrated<T>` gained the
matching 4th optional parameter (documented as AD-08) and forwards it
verbatim to `runCommand`. `result.trace` is still discarded for the
return value (no contract change for existing callers); `onTransition`
is the only way to observe the trace from `apps/cli` today. No existing
call site (`audit.ts`, `configure.ts`, `init.ts`, `join.ts`,
`start.ts`, `sync.ts`, `upgrade.ts`) was changed to actually pass an
`onTransition` — the brief only asked for the hook to exist and be
available for a future consumer, not for any command to use it yet.

### Item 4 — `RepoRegistered` notification at ungated `addProjectV2` call sites

`apps/cli/src/commands/_shared.ts` gained `notifyRepoRegistered(notification:
RepoRegisteredNotification): void` and the `RepoRegisteredNotification`
interface (`source`, `projectId`, `role?`). It writes exactly one line
to stderr (`[axiom] event=RepoRegistered source="<cmd>"
projectId="<id>" role="<role>"`, role suffix omitted when absent) and
never throws — mirroring the existing `registryAttempt`/`[axiom init]
WARN: ...` best-effort pattern already used at the same call sites for
the error path. No gate, no queue, no subscriber list.

Wired at all 5 confirmed `addProjectV2` call sites, each immediately
after `addResult.ok` is confirmed `true` (never on the `duplicate` or
`error` early-return branches):

| # | File | Function | `source` value |
|---|---|---|---|
| 1 | `init.ts` | `runInit`'s registry-attempt path | `'init'` |
| 2 | `join.ts` | `runJoin`'s registry-attempt path | `'join'` |
| 3 | `repo.ts` | `runRepoAttach` | `'repo attach'` |
| 4 | `projects.ts` | `runProjectsAdd` | `'projects add'` |
| 5 | `projects.ts` | `runProjectsJoin` | `'projects join'` |

This matches the audit's own count ("INC-01 landed... and cut over 5
CLI commands").

### Items 1, 2, 6 — explicit non-actions, confirmed by inspection

- No `CommandId` intent stub was implemented (item 1) — confirmed by
  diff: `types.ts`'s `CommandId` union is byte-identical to the
  pre-increment version (only `runner.ts`'s signature changed, not the
  type).
- No registry write was routed through `runCommand`/`gateFor` (item
  2) — confirmed by grep: zero new `runCommand`/`gateFor`/
  `runOrchestrated` call sites were added around
  `addProjectV2`/`saveRegistryV2`/`useProjectV2`; the notification
  calls are plain function calls, not gate-wrapped.
- No event-bus/pub-sub package, no `TraceEntry` persistence, no
  `STATE_MACHINE`/`gateFor`/`Precondition` change (item 6) — confirmed
  by diff scope: only `runner.ts` (signature + doc comment),
  `_shared.ts` (forwarding + new helper), and the 4 CLI command files'
  call sites were touched in `apps/cli`/`packages/orchestrator`.

### Item 5 — six events left unimplemented, as instructed

`PlanCreated`, `PlanLinkedToIncrement`, `AdrLinked`,
`TechnicalContextUpdated`, `SkillInstalled`, and the generic
`IncrementStatusChanged` were not given hooks — no live write path was
confirmed for them in the audit's scope, and the brief explicitly
forbids speculative hooks for unconfirmed paths.

### Incidental fix required for validation to be meaningful (flagged, not silent)

While adding tests for `notifyRepoRegistered`
(`apps/cli/tests/_shared.test.ts`), a pre-existing repository defect
was discovered and is flagged here rather than silently patched over:
`apps/cli/src/commands/{_shared,configure,sync,upgrade}.{js,js.map,d.ts,d.ts.map}`
(16 files total) were stray, compiled CommonJS build artifacts
committed directly into `src/` (not `dist/`) by an unrelated prior
commit (`67c35f8`, "feat: add project readiness verification script
and TypeScript configuration", dated before this increment). Because
Node/Vite's extensionless module resolution can resolve
`import ... from './_shared'` to the stale `_shared.js` sibling instead
of the real `_shared.ts` source depending on the importing module,
these stale files silently shadowed my new `notifyRepoRegistered`
export for any test importing `_shared` directly
(`tests/_shared.test.ts`) or importing a command module
(`init.ts`/`repo.ts`/`projects.ts`) that in turn imports the shadowed
`_shared`. This produced `TypeError: notifyRepoRegistered is not a
function` in 5 test files even though the real `.ts` source was
correct — confirmed by direct inspection (the `.js` files still had
`"use strict"` + `exports.*` CJS shape from the pre-cutover
`registry.json`/v1-era code, missing `saveRegistryV2` and the new
`notifyRepoRegistered` entirely).

This was not introduced by this increment (git blame confirms
`67c35f8` predates it) but directly blocked this increment's own
validation from being meaningful, since 5 of my own new/updated test
files would otherwise fail for a reason unrelated to my actual code.
The 16 stray files were removed (`git rm --cached` + `rm`) as the
minimal fix; `npm run build`'s real output goes to `dist/`, confirmed
unaffected by the removal (`npm run typecheck`/`npm run build` both
pass cleanly before and after). This is reported here per the task's
explicit instruction to flag rather than silently resolve mismatches,
even though this particular issue is a build-hygiene defect rather
than an audit/brief mismatch. **Recommendation for validator-reviewer**:
confirm the removal is acceptable, and consider a `.gitignore` rule or
CI check to prevent `tsc`/build output from being committed under
`src/` again.

### Auto-commit observed during this session (informational, not an action taken)

An automated commit (`4ba8abe`, "feat: implement registry v2 with YAML
support and migration handling", timestamped mid-session) appeared in
`git log` during this increment's work, bundling the previously
uncommitted INC-01 registry-v2 implementation together with this
increment's `runner.ts`/`_shared.ts`/`init.ts`/`join.ts`/test changes
into a single commit. This was not initiated by this increment's work
(no `git commit` was run) — it is flagged here for transparency since
it affects how `git diff`/`git status` read during validation (most of
this increment's source changes appear as already-committed in `HEAD`
rather than as working-tree diffs; only the stray-`.js` removal and two
test-file edits remained as uncommitted working-tree changes at the
time of this report).

## Validation

Validation commands discovered via `package.json` scripts at the
monorepo root (`Axiom/package.json`): `npm run typecheck`, `npm run
build`, `npm test`.

**`npm run typecheck`** (`tsc -b`): passes with zero errors, both
before and after the stray-`.js` removal.

**`npm run build`** (`tsc -b`): passes with zero errors, both before
and after the stray-`.js` removal. Confirms the stray `src/*.js` files
were never part of the real build graph (`dist/` output unaffected).

**`npm test`** (full monorepo, vitest across all packages/apps):

- Result: **12 failed test files / 13 failed tests / 1220 passed**
  (1233 total).
- Cross-checked against the task's stated known baseline for the
  INC-01 chain (12 failed files / 13 failed tests): **identical** —
  no regressions, no new failures.
- Failing files (all pre-existing, confirmed by name against the
  documented baseline pattern — real-repo `axiom.spec` integration
  tests + `tui/driver.test.ts`):
  - `apps/cli/tests/start.test.ts` (1 test)
  - `packages/agents/tests/catalog.test.ts` (1 test)
  - `packages/doctor/tests/checks.test.ts` (1 test)
  - `packages/model-routing/tests/assignments.test.ts` (1 test)
  - `packages/model-routing/tests/loader.test.ts` (1 test)
  - `packages/model-routing/tests/opencode-projection.test.ts` (1 test)
  - `packages/model-routing/tests/resolver.test.ts` (1 test)
  - `packages/project-resolution/tests/resolver.test.ts` (1 test)
  - `packages/skills/tests/catalog.test.ts` (1 test)
  - `packages/telemetry/tests/audit-trail-sink.test.ts` (1 test)
  - `packages/toolchain/tests/repair-add-gitignore.test.ts` (1 test)
  - `packages/tui/tests/driver.test.ts` (2 tests)
- None of these 12 files were touched by this increment; all failures
  are pre-existing environment/fixture issues (real-repo path
  dependencies, TUI snapshot brittleness) unrelated to the
  `onTransition` hook or the `RepoRegistered` notification.

**Scoped test runs** (packages/apps actually touched):

- `packages/orchestrator` (`npx vitest run`): **31/31 passed** (3 test
  files: `state-machine.test.ts` 15, `gates.test.ts` 7, `runner.test.ts`
  9 — includes 4 new tests for the `onTransition` hook).
- `apps/cli` scoped to `_shared.test.ts`, `init.test.ts`, `join.test.ts`,
  `repo.test.ts`, `projects.test.ts`: **55/55 passed** (includes new
  tests for `notifyRepoRegistered` and per-call-site `RepoRegistered`
  stderr assertions).

**Bisection note**: re-running the full suite against the pre-change
tree (via `git stash`) initially showed 17 failed files / 40 failed
tests — worse than the documented 12/13 baseline. Root-causing this
discrepancy led directly to the stray-`.js`-shadowing discovery above:
the stash captured both my code changes AND the stray-file removal
together, so the "pre-change" comparison run still had the shadowing
bug active, inflating failures across files that import `_shared.ts`
indirectly. Once isolated correctly (working tree with my changes
applied, stray files removed), the result matches the documented
baseline exactly (12/13).

## Result

Implemented the migration-engineer audit's "Registry-engineer brief"
items 1-6 exactly as scoped:

1. Added an optional `onTransition?: (trace: TraceEntry) => void` 4th
   parameter to `runCommand` (`packages/orchestrator/src/runner.ts`,
   AD-07) and to `runOrchestrated`
   (`apps/cli/src/commands/_shared.ts`, AD-08), invoked once on the
   success path only, forwarded transparently with zero behavior
   change for existing 3-arg callers.
2. Added `notifyRepoRegistered` (`apps/cli/src/commands/_shared.ts`), a
   non-throwing, one-line stderr notification, wired at all 5
   confirmed `addProjectV2` call sites (`init.ts`, `join.ts`,
   `repo.ts`, `projects.ts` x2) immediately after a successful add —
   no gate, no new abstraction, matching the existing best-effort
   logging pattern.
3. Did not implement any `CommandId` intent stub, did not route any
   registry write through `runCommand`/`gateFor`, did not touch
   `CommandId`/`STATE_MACHINE`/`gateFor`/`Precondition`, and did not
   build hooks for the six events with no confirmed live write path —
   all per the brief's explicit non-goals.
4. Discovered and flagged (not silently fixed) a pre-existing
   repository defect (16 stray compiled `.js`/`.d.ts` artifacts
   committed into `apps/cli/src/commands/`) that was actively shadowing
   the new `notifyRepoRegistered` export for direct/indirect importers
   of `_shared.ts`; removed the stray files as the minimal fix required
   for this increment's own validation to be meaningful, confirmed
   `typecheck`/`build` unaffected.
5. Ran real validation: `npm run typecheck` and `npm run build` both
   pass with zero errors; `npm test` (full monorepo) failure set is
   **12 failed files / 13 failed tests**, an exact match to the known
   INC-01-chain baseline — no regressions introduced.
6. Made one explicit judgment call (documented in "Flagged mismatches"
   below): did not add a `RepoRegistered` notification to
   `useProjectV2`/`projects use`, since the brief's item 4 names
   `addProjectV2` call sites specifically and "use" is not a
   registration event.

## Flagged mismatches (between the audit's brief and live code / judgment calls made)

1. **`useProjectV2`/`projects use` scope decision.** The brief's item
   4 says "making the existing, already-working `addProjectV2` call
   sites... emit the same kind of lightweight, synchronous
   notification." `projects.ts`'s `runProjectsUse` calls `useProjectV2`,
   not `addProjectV2` — it is a "mark as used" operation, not a
   registration. This increment interpreted the brief literally (5
   `addProjectV2` call sites, not 6 if `useProjectV2` were included)
   and did NOT add a notification to `runProjectsUse`. This is a
   judgment call, not a re-litigation of the brief — flagged for
   validator-reviewer to confirm or override.
2. **Pre-existing stray build artifacts shadowing new exports.**
   Documented in full under "Incidental fix required for validation to
   be meaningful" above. Not an audit/brief mismatch, but a real
   repository defect this increment's validation surfaced and had to
   resolve to produce a meaningful test result. Flagged, not silently
   folded into the feature diff without explanation.
3. **Auto-commit observed mid-session.** Documented under "Auto-commit
   observed during this session" above — informational only, no action
   taken by this increment beyond noting it for transparency.

No other mismatches were found: the brief's items 1, 2, 3, 5, and 6
matched live code exactly as described by the migration-engineer audit,
with no further corrections needed.

## General spec integration

No integration into `Axiom.Spec/general-spec.md` was performed — that
file does not exist in this repo, consistent with every increment in
the INC-01 chain and with the migration-engineer audit's own posture.

Stable finding worth flagging for eventual consolidation (once
`general-spec.md` exists): **`@axiom/orchestrator`'s `runCommand` now
exposes an optional `onTransition` hook (AD-07) as the sanctioned,
minimal extension point for structural-command traceability/cache
work — any future increment wanting to observe successful gated
command transitions should use this hook rather than inventing a new
mechanism. Separately, `RepoRegistered`-shaped notifications for the
deliberately-ungated user-level registry writes now go through
`notifyRepoRegistered` in `apps/cli/src/commands/_shared.ts` — any
future event added for a similarly ungated write path should follow
the same one-line, non-throwing, best-effort stderr pattern rather than
building new infrastructure.**

## Closure rationale

`Status: pending`, not `closed`, because:

- This increment's own scope (registry-engineer implementation: brief
  items 1-6, tests, real validation) is fully complete, evidenced, and
  documented above, with test results matching the known baseline
  exactly (no regressions).
- However, INC-02 as a whole (per the parent roadmap's own three-role
  sequence: migration-engineer -> registry-engineer -> validator-reviewer)
  has one more role pending: **validator-reviewer** has not yet run.
- Consistent with the INC-01 chain's own closure discipline and with
  the migration-engineer audit's own closure rationale, this increment
  follows the same pattern: `pending` for the sub-step, not `closed`,
  even though this sub-step's own work is finished and validated.
- Two explicit flagged items (the `useProjectV2` scope judgment call
  and the stray-build-artifact removal) are called out above
  specifically so validator-reviewer can confirm or override them
  rather than having them silently baked in.

## Next step recommendation

Launch **validator-reviewer** next, per INC-02's subagent sequence.
Exact inputs it needs from this document and its predecessor:

1. **This document's "Implementation notes"** as the precise record of
   what changed: `runCommand`/`runOrchestrated`'s `onTransition` hook
   (AD-07/AD-08) and `notifyRepoRegistered`'s 5 call sites.
2. **Confirm the `onTransition` hook does not alter any existing
   `GateResult`/`RunResult` contract** — per the audit's own "Next step
   recommendation" for validator-reviewer. This increment's tests
   confirm existing 3-arg `runCommand`/`runOrchestrated` callers are
   unaffected (full suite baseline match), but validator-reviewer
   should independently verify this claim.
3. **The two "Flagged mismatches"** (`useProjectV2` scope decision,
   stray build artifact removal) — confirm both are acceptable or
   provide corrective direction.
4. **The exact validation results above** (12/13 baseline match) as
   the current known-good state to re-verify, plus this increment's
   specific new tests
   (`packages/orchestrator/tests/runner.test.ts`'s new "AD-07" describe
   block; `apps/cli/tests/_shared.test.ts`'s new `notifyRepoRegistered`
   describe block; the per-call-site `RepoRegistered` stderr assertions
   added to `init.test.ts`/`repo.test.ts`/`projects.test.ts`).
5. Per the audit's own recommendation: extend relevant `@axiom/doctor`
   checks or vitest coverage as appropriate, following the "extend,
   don't replace" discipline the INC-01 chain's validator-reviewer used
   for MC-001/BC-001 — evaluate whether any doctor check should surface
   `RepoRegistered` notification failures (unlikely, since the function
   never throws) or whether none is warranted (likely, given the
   deliberately best-effort, non-blocking nature of the notification).

After validator-reviewer confirms or corrects this work, INC-02 as a
whole can be closed by the same role, per the parent roadmap's sequence.
