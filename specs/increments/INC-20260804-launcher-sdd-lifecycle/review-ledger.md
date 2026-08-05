# Review ledger - INC-20260804-launcher-sdd-lifecycle

Status: closed
Review type: independent review after apply; N=1 re-review
Review date: 2026-08-04
Scope: ACC-007 / R-10, launcher registry, action routing, canonical run functions, integrate/archive, launcher front and focused tests.
Governance at N=1: no general specs or context were edited; no artifact was archived; no commit, branch, reset or other Git mutation was performed.
The prior hash `d118591cf24dc6303f57edfa8722f16001c07f469e02696467c42effa72f566f` is historical review evidence, not the final closure hash.

## Review basis

This N=1 review used the existing ledger, the current diff in the affected runtime files, and focused executable validation. No broad repository exploration was performed.

The prior review traced registry provenance, action routing, confirmation and preview gates, canonical lifecycle runners, integrate/archive behavior, HTTP error transport, non-executable actions, and the available diagnostics. Its three open findings are retained below with their current N=1 dispositions.

## Finding status

### D-001 - High - preview and execute disagree for role execution mode

Prior finding: `back-new` and `front-new` advertised `executionMode` in preview, but confirmed execution discarded it and used the synchronous role path. The prior ledger required routing through the async role wrapper and an end-to-end assertion of the actual mode.

N=1 disposition: `verified`. The current diff passes the mode from `buildCliArgs` through `executeSubcommand` to `runRoleSubcommandAsync`, awaits the handler, and normalizes `executionMode`, `worktreeStart` and `worktreeClose`. The confirmed HTTP test at [app-launcher.test.ts](../../../../Axiom/apps/cli/tests/app-launcher.test.ts#L321) is parameterized for both `back-new` and `front-new`, asserting the real `--worktree` command, `executionMode` and `worktreeStart`; the role test slice covers `worktreeClose`.

### D-002 - High - integrate rollback is compensating, not guaranteed

Prior finding: `runIntegrate` could leave active metadata marked archived/integrated if the physical move failed and the typed rollback write also failed. The prior ledger required a rollback-write failure test and a recovery or explicit repair signal.

N=1 disposition: `verified`. The current diff checks the archive destination before writing metadata and, after a move failure, tries typed rollback followed by raw snapshot restoration. The collision test and typed-rollback failure test confirm that active metadata remains non-archived/non-integrated; a final filesystem failure now returns `inconsistent: true` and the launcher renders a manual-repair warning.

### D-003 - Medium - front omits valid destination-only state and metadata path

Prior finding: the front rendered the state arrow only when `fromState` existed and did not render `paths.metadata` in the registry table. The prior ledger also requested separate provenance-backed folder, metadata and README lines plus DOM-level coverage.

N=1 disposition: `verified` for the requested fields. The registry test proves `paths.metadata` is present. The front condition at [launcher.js](../../../../Axiom/apps/cli/static/launcher/launcher.js#L1057) renders a destination-only state, and the path cell now displays `folder`, `readme` and `metadata` as separate lines.

## Gates and non-executable actions

PASS for the reviewed slice. The confirmation gate remains before mutating handlers; craft and unconfirmed execute remain read-only; the validation tests confirm read-only transition and change validation. `increment-change` and `plan-archive` remain explicitly non-executable, while HTTP 500 results still carry the canonical result body. The focused launcher, validator and e2e tests passed without gate regressions.

## Other risks and accepted boundaries

- `runValidateTransition` validates the declared graph but does not execute command-specific gates; callers must not present it as approval, freeze or write-scope execution.
- `runValidateChanges` enforces plan status and write-scope checks when explicitly invoked; archive does not silently run it.
- `requiresApproval` is enforced at the launcher confirmation boundary, not by the pure workflow state machine.
- In a multi-repo project, role affinity can reject a role action honestly when the selected primary project repo differs from the assigned role repo; the launcher route has no role-repo selector.
- Relation IDs without matching target entries remain sourced from real metadata links and are not fabricated into resolved paths.
- At N=1, the formal freeze, receipts and final orchestrator integration were
	still the orchestrator's responsibility; those gates are now completed by
	the final freeze/receipt pass.

## Validation observed

Prior evidence recorded by the ledger included the successful build, launcher/integrate tests, the focused lifecycle suite, launcher e2e test, frontend syntax check and clean TypeScript diagnostics. N=1 added and independently observed:

- `npx vitest run apps/cli/tests/app-launcher.test.ts apps/cli/tests/integrate.test.ts`: 2 files, 30 tests passed.
- `npx vitest run apps/cli/tests/e2e/launcher.e2e.test.ts apps/cli/tests/validate-transition.test.ts apps/cli/tests/validate-changes.test.ts`: 3 files, 17 tests passed.
- `npx vitest run apps/cli/tests/axiom-role-worktree.test.ts apps/cli/tests/e2e/axiom-role-worktree.e2e.test.ts`: 2 files, 8 tests passed.
- `npm run build`: passed (`tsc -b`).
- `node --check apps/cli/static/launcher/launcher.js`: passed.
- `get_errors` on `app-launcher.ts`, `app-api.ts` and `integrate.ts`: no errors found.

Remaining evidence gap: there is no browser/DOM harness in the repository for
	the static launcher; `node --check` and the HTTP/registry assertions cover
	the data contract, while the render branches are kept deliberately
	dependency-free.

## Recommendation

`closed` for implementation/review readiness.

Reason: the N=1 diff and focused validation resolve the three requested
behavior findings. The remaining browser/DOM harness limitation is explicit
and does not indicate an open runtime finding. Freeze, receipts and final
spec integration are complete; the folder is ready for physical archive.

## Documentary actions for the orchestrator

- Preserve the final post-apply freeze and its receipts; do not reuse the
	historical hash `d118591cf24dc6303f57edfa8722f16001c07f469e02696467c42effa72f566f`.
- Keep the explicit `inconsistent: true` manual-repair signal if a future
	filesystem failure defeats both rollback mechanisms.
- Reconcile stable launcher facts in the owning `specs/00..08` and `context/**`: actual async role execution, executable mode semantics, archive rollback guarantee, and the advisory nature of QA archive. The orchestrator should make those documentation changes after review.
- The final increment integration, freeze and receipts are now complete. No
	additional general spec/context edit is required for this review scope.

## Suggested commit message

`review(launcher): record N=1 findings and close lifecycle review`

## Final orchestrator governance

The three behavioral findings are verified, canonical integration is complete,
and the final freeze plus `verify` receipt supersede the historical N=1
governance boundary. The increment is closed and ready for physical archive.

No commit was created.
