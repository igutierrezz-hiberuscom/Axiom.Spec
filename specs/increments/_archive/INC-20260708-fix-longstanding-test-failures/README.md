# Increment: Fix longstanding logic-only test failures (AB2)

Status: closed
Date: 2026-07-08

## Goal

Fix the 4 pure-logic test failures left over after AB1
(`INC-20260708-product-repo-self-bootstrap`) so the product repo's full
`npx vitest run` suite is completely green (0 failing).

## Context

AB1 scaffolded `Axiom/axiom.config/` and `Axiom/axiom.spec/`, taking the
full suite from 10 failing to 4. Those 4 were explicitly deferred as
"logic-only, not config-missing" bugs:

1. `packages/model-routing/tests/opencode-projection.test.ts` —
   `slotCount` expected 10, got 7.
2. `packages/project-resolution/tests/resolver.test.ts` — v1-schema
   `exists` semantics for the `product` scope.
3. `packages/telemetry/tests/audit-trail-sink.test.ts` — retention-sweep
   window boundary appeared off-by-one.
4. `packages/toolchain/tests/repair-add-gitignore.test.ts` — multi-tool
   `repairToolchain` path substitution.

## Scope

- Investigate each of the 4 failures by reading both the implementation
  and the test, and decide per-failure whether the implementation or
  the test encodes the wrong behavior.
- Apply the minimal fix on whichever side is actually wrong.
- Re-run the full suite to confirm 0 regressions, 0 remaining failures.

## Non-goals

- No refactors beyond the specific bug in each case.
- No changes to unrelated packages/tests.
- No git commits (local-only, per orchestrator guardrails).
- No integration into `Axiom.Spec/specs/00-08` (reserved for the
  autopilot orchestrator's final consolidation pass across the batch).

## Acceptance criteria

- [x] `npm run build` (`tsc -b`, from `Axiom/`) is clean.
- [x] Full `npx vitest run` (from `Axiom/`) has 0 failing test files and
      0 failing tests.
- [x] No previously-passing test regressed.
- [x] Each of the 4 original failures has a documented root cause and a
      documented decision (impl fixed vs. test fixed), with rationale.

## Open questions

None blocking.

## Assumptions

- For the model-routing slot count, the canonical taxonomy is the
  7-slot set (`increment, bug, plan, implementation, qa-e2e, review,
  archive`) already declared in `packages/model-routing/src/types.ts`
  (`SlotId` union) and `packages/model-routing/src/slots.ts`
  (`SLOT_TAXONOMY`), and already asserted as canonical by the
  package's own `tests/slots.test.ts` (`toHaveLength(7)`). The "10
  slots" figure was stale/aspirational text (a leftover assertion in
  `opencode-projection.test.ts` plus stale comments in `effective.ts`)
  that was never backed by an actual 10-entry taxonomy anywhere in the
  codebase or in `Axiom.Spec`. No spec document declares a 10-slot
  taxonomy either. Reconciled everything to 7.
- For project-resolution, the `product` scope's `path: .` always
  resolves to `rootPath` itself (`path.join(rootPath, '.')`), which by
  construction always exists once `axiom.yaml` was written into that
  same directory. The v1 test's `exists === false` expectation was
  already flagged as a known-incorrect pre-existing expectation by a
  prior fix (see the sibling v2 test's inline comment in the same
  file, added before this increment), which deferred fixing the v1
  copy. This increment closes that gap by correcting the v1 test to
  match the v2 test's (correct) expectation.
- For telemetry, the retention-sweep boundary condition in
  `AuditTrailSink.runRetentionSweep` (`evTime >= cutoff`) is correct
  (inclusive of the cutoff edge). The actual defect was a
  non-hermetic test: the sink computes its cutoff from real
  `Date.now()`, but the test built its fixtures relative to a
  fictional reference date (`2026-06-25`) without pinning the system
  clock. As real wall-clock time advanced past that reference, the
  fixture that was "24 days old" (assumed inside the 30-day window)
  became genuinely 30+ days old, so the (correct) implementation
  purged it and the test's hardcoded expectation broke. Fixed by
  pinning the clock with `vi.useFakeTimers()` / `vi.setSystemTime()`.
- For toolchain, `repairTool`/`repairToolchain` and the underlying
  `resolveDetectionPath` are correct and already covered by a passing
  single-tool test in the same file. The multi-tool test's own
  path-resolution helper (hand-rolled `.replace(...)` calls) diverged
  from the real `resolveDetectionPath` contract: for a
  `detectionPath` template with no `${projectRoot}` placeholder, the
  real function treats it as relative to `projectRoot`
  (`path.join(projectRoot, template)`), by explicit design (0027/B1,
  to avoid false positives against process cwd). The test's manual
  replace skipped that `path.join`, so it checked existence of a path
  relative to the process cwd instead of `tmpDir`. Fixed by importing
  and using the real `resolveDetectionPath` in the test instead of
  duplicating its logic incorrectly.

## Implementation notes

Files edited (all in `Axiom/`, no git commits performed):

- `packages/model-routing/tests/opencode-projection.test.ts` — changed
  `expect(r.value.slotCount).toBe(10)` to `toBe(7)` (the same test
  already separately asserts `Object.keys(onDisk.slots).length` is 7
  a few lines below; the `slotCount` assertion was the odd one out,
  contradicting its own test body as well as the package's own
  canonical taxonomy).
- `packages/model-routing/src/effective.ts` — corrected two stale
  "10 slots en el MVP" doc comments to "7 slots en el MVP" (comment-only,
  no behavior change; `resolveAllSlots` already iterated
  `SLOT_TAXONOMY`'s real keys, which is 7).
- `packages/project-resolution/tests/resolver.test.ts` — changed
  `expect(result.scopes['product'].exists).toBe(false)` to `toBe(true)`
  in the v1 scope-resolution test, with an explanatory comment mirroring
  the sibling v2 test in the same file, which had already documented
  this exact issue.
- `packages/telemetry/tests/audit-trail-sink.test.ts` — wrapped the
  Scenario 3 retention-sweep test body in `vi.useFakeTimers()` /
  `vi.setSystemTime(now)` / `finally { vi.useRealTimers() }` so the
  fixture timestamps' day-deltas are evaluated against the intended
  fictional "now" instead of the real wall clock.
- `packages/toolchain/tests/repair-add-gitignore.test.ts` — imported
  `resolveDetectionPath` from `../src/index` and replaced the test's
  incorrect hand-rolled placeholder substitution with a call to the
  real function.

## Validation

- `npm run build` (from `Axiom/`, `tsc -b`): clean, no errors.
- `npx vitest run` (full suite, from `Axiom/`):
  ```
   Test Files  178 passed (178)
        Tests  1859 passed (1859)
  ```
  Before this increment: 4 failed / 174 passed test files (1855/1859
  tests passing). After: 0 failed, all 178 test files / 1859 tests
  passing. No regressions (delta is strictly 4 fixed, nothing new
  broken).
- Targeted re-run of the 4 originally-failing files together:
  ```
   ✓ packages/telemetry/tests/audit-trail-sink.test.ts (8 tests)
   ✓ packages/toolchain/tests/repair-add-gitignore.test.ts (12 tests)
   ✓ packages/model-routing/tests/opencode-projection.test.ts (7 tests)
   ✓ packages/project-resolution/tests/resolver.test.ts (25 tests)

   Test Files  4 passed (4)
        Tests  52 passed (52)
  ```

## Result

All 4 pre-identified logic-only failures are fixed. Root cause per
failure:

1. **model-routing** — stale test assertion (`slotCount` expected 10)
   contradicting the package's own canonical 7-slot taxonomy and even
   its own test body's other assertion (7). Fixed the test (+ 2 stale
   doc comments in `effective.ts`).
2. **project-resolution** — stale/incorrect test expectation
   (`product.exists === false`) that a sibling v2 test in the same
   file had already flagged as a known pre-existing bug. Fixed the
   test to match actual (correct) filesystem semantics.
3. **telemetry** — non-hermetic test (relied on real wall-clock time
   drifting past a fictional fixture reference date) rather than an
   actual off-by-one in the sink's boundary check (which was already
   correct: `evTime >= cutoff`, inclusive). Fixed the test by pinning
   the clock.
4. **toolchain** — test-only bug: a hand-rolled path-substitution
   helper in the test diverged from the real, already-correct
   `resolveDetectionPath` contract for no-placeholder templates. Fixed
   by using the real exported function instead of duplicating it.

In all 4 cases the implementation was correct and the test was wrong
or non-deterministic; no production behavior changed. The product repo
now self-validates cleanly end to end (build + full test suite green).

## General spec integration

Integrated into `Axiom.Spec/specs/00-08` by the batch's final
consolidation pass (autopilot orchestrator). Landed in:

- **02_Requisitos_No_Funcionales.md** — NFR-AXM-010 note: full suite
  green (this increment closed the 4 remaining logic-only failures,
  all test-side defects); plus a reconciled taxonomy note that model
  routing is canonically **7 slots** (`increment, bug, plan,
  implementation, qa-e2e, review, archive`), not 10.
- **00_Resumen_Ejecutivo.md** / **07_Gobierno_y_Seguridad.md** — cited
  alongside `product-repo-self-bootstrap` as the pair that restored the
  product's end-to-end self-validation (build + full suite green).

The "10 slots" figure was checked during the pass and does not appear
in any `00-08` doc, so no correction was needed there.
