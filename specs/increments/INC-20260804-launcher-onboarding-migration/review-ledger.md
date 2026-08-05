# Review ledger

Increment: `INC-20260804-launcher-onboarding-migration`
Action: `ACC-006 / R-13 / N=1 re-review`
Review mode: independent post-apply review (`flow=increment`, `route=sdd`)
Review date: 2026-08-04
Recommendation: `closed`

## Review boundary

Reviewed `Axiom.SDD/AGENTS.md`, this increment README, requirements and
acceptance criteria, and only the requested current surfaces in `Axiom`:

- `app-onboarding.ts` and `app-api.ts`;
- launcher static HTML, JavaScript and transport;
- `launcher-onboarding-migration.test.ts`;
- delegated workspace setup/adopt/bootstrap primitives and the setup-spec
  builder needed to verify their contracts.

For this N=1 re-review, the only review inputs were this existing ledger and the current fix diff. No broader spec/context search or governance mutation was performed.

The pre-apply candidate freeze remains the supplied historical hash
`45150e80579291001560fd8f56c343fc9a0388f9262093a50f613015a2729504`.
The README and runtime changed after that freeze; this review section records
the pre-final governance boundary and is not the final freeze evidence.

## Sweep log

| Sweep | Focus | Result |
| --- | --- | --- |
| 1 | Acceptance criteria, endpoint contract, confirmed and preview gates | Confirmed happy paths; identified the need to verify partial-result handling. |
| 2 | Delegation, workspace setup/adopt/bootstrap, path and registry isolation | Found `REVIEW-001` and `REVIEW-002`; both were reproduced against the current build. |
| 3 | Static launcher, transport, project refresh and coverage claims | Found `REVIEW-003`, `REVIEW-004`, `REVIEW-005` and `REVIEW-006`. |
| 4 | Build, type diagnostics, JavaScript syntax, focused suites and probes | No new finding; existing findings remained open. Ceiling of four sweeps reached. |
| 5 | N=1 post-fix re-review of REVIEW-001 through REVIEW-006 | All six findings are addressed by the current diff; focused onboarding tests, build and JavaScript syntax checks pass. |

## Compliance

| Area | Assessment |
| --- | --- |
| Preview and unconfirmed requests | Compliant on the exercised paths. Setup returns before `runWorkspaceSetup`; adoption delegates the read-only branch of `runWorkspaceAdopt`. |
| Absolute paths and home registry | Compliant on the reviewed paths. Required setup/adoption paths and legacy sources are checked as absolute, destination ownership is checked before adoption, and destination overlap is rejected. |
| Canonical delegation | Compliant. The endpoint calls `runWorkspaceSetup` and `runWorkspaceAdopt`; adoption continues through the existing bootstrap runners. No second writer was added in the endpoint. |
| Single-repo, multi-repo, adoption and idempotency | Compliant on the reviewed paths. Overlapping destinations and foreign or unidentified existing destinations are rejected before the canonical adoption primitive runs; the focused suite still passes the canonical adoption/idempotency path. |
| Install, join, roles, registry and doctor compatibility | Compliant on the reviewed onboarding path. Setup/adoption selection now requires a real registry project; the existing join behavior remains unchanged. |
| Honest results and partial success | Compliant on the reviewed path. Adoption returns a successful HTTP envelope with `partial` and the canonical result when writes occurred with `exitCode: 1`; transport and the static launcher preserve and render the result. |
| Build and focused validation | Compliant for the executed commands listed below. |
| Structured skip reporting | Compliant. Setup no longer fabricates `skipped.files: []`; it reports actual `filesCreated` and canonical warnings, while `skipped` contains only the repo facts available from the setup result. |

## Findings ledger

| id | lens | location | severity | status | evidence |
| --- | --- | --- | --- | --- | --- |
| REVIEW-001 | review | `apps/cli/src/commands/app-onboarding.ts` (`destinationOwnershipError`, `apiLauncherWorkspaceAdopt`) | CRITICAL | verified | The current diff checks every destination `axiom.yaml` before calling `runWorkspaceAdopt`. A different `projectId`, or an unreadable/unknown identity represented by `tryReadExistingProjectId(...) === null`, returns HTTP 409 and no migration is attempted. The focused HTTP test verifies the foreign-project case, preserves the original `axiom.yaml`, and confirms that no destination migration directory is created. |
| REVIEW-002 | review | `apps/cli/src/commands/app-onboarding.ts` (`pathsOverlap`, `parseWorkspaceRoles`, `prepareWorkspaceInput`) | CRITICAL | verified | The current diff rejects equality and ancestor/descendant paths among roles, between roles and control/spec, and between control and spec. The focused HTTP test exercises an overlapping nested role pair and confirms HTTP 400 with no setup mutation. |
| REVIEW-003 | review | `apps/cli/src/commands/app-onboarding.ts`, `apps/cli/src/commands/app-api.ts`, `apps/cli/static/launcher/transport.js`, `apps/cli/static/launcher/launcher.js` | CRITICAL | resolved | Adoption now returns HTTP 200 with `executed: true`, `partial: true` and the complete result when `wrote` is true and `exitCode !== 0`. The API forwards that body, transport treats an executed body as usable even for a non-2xx response, and `renderOnboardResult` explicitly labels partial application while rendering the full result, including warnings, paths and provenance. |
| REVIEW-004 | review | `apps/cli/static/launcher/launcher.js` (`handleOnboardResult`) | WARNING | resolved | Automatic selection now requires a registry project for setup/adoption and accepts it only when the registry reports `registered: true` or the project object itself is present. A missing registry project therefore cannot trigger project-scoped refreshes. |
| REVIEW-005 | review | `apps/cli/tests/launcher-onboarding-migration.test.ts` | WARNING | verified | The new HTTP suite has eight passing tests and explicitly covers `dryRun: true` with confirmation, a foreign destination, and an overlapping nested role path. The same suite also verifies preview read-only behavior, canonical setup/adoption, provenance and idempotency. |
| REVIEW-006 | review | `apps/cli/src/commands/app-onboarding.ts` (`workspaceSetupResult`) | WARNING | resolved | The current diff removes the unsupported `skipped.files: []` claim. Setup reports `filesCreated` from the canonical result and preserves canonical warnings; no fabricated file-skip list is emitted. |
## Deviations and risks

The six reviewed findings are resolved. The remaining validation limitation is
that the new suite does not inject a late canonical write failure to exercise
the browser rendering of `exitCode: 1`, nor does it add a separate unknown-owner
fixture; those branches are directly represented in the current diff and the
build/syntax checks pass. This is a test-depth gap, not an open behavioral
finding for this re-review.

No introduced regression was observed in install, join or the existing role
register/assign wrappers. The existing primitive no-clobber, provenance and
idempotency guarantees remain covered by the focused validation.
## Validation observed

- `npm run build` from `Axiom/` — passed (`tsc -b`).
- `get_errors` on all five TypeScript command surfaces — no errors.
- `node --check apps/cli/static/launcher/launcher.js` — passed.
- `node --check apps/cli/static/launcher/transport.js` — passed.
- `npx vitest run apps/cli/tests/launcher-onboarding-migration.test.ts` — passed (8 tests).
- Focused launcher set: 3 files, 45 tests passed in the earlier review pass.
- Final increment validation set: 7 files, 90 tests passed, including the
  available workspace-adopt E2E file.
- The current diff was checked for the ownership guard, overlap guard, partial
  response propagation, registry-gated selection and removal of the fabricated
  file-skip field.
- No mutating Git command, archive operation or general-spec/context edit was
  performed.
## Closure recommendation

`closed`. REVIEW-001 and REVIEW-002 now reject unsafe destinations before
mutation, REVIEW-003 preserves and renders partial adoption results, and
REVIEW-004 through REVIEW-006 are addressed by the current launcher and test
diff. The focused suite, build and JavaScript syntax checks pass. The residual
test-depth limitation is recorded above and does not leave an open finding.
## Required documentation actions for the orchestrator

No edit to `specs/00..08` or `context/**` was made or is requested by this
re-review. The orchestrator should retain only the verified stable facts when
performing its normal final integration:

- destination-owner and overlap validation for launcher workspace operations;
- the partial-success response contract and UI rendering rules;
- structured skip/provenance and registry-refresh semantics.

The increment README and candidate freeze were reconciled by the orchestrator
as a separate governance action; this ledger itself did not perform either
mutation.

## Final orchestrator governance

The historical pre-apply freeze and the review boundary above are superseded
by the final freeze and `verify` receipt emitted after canonical integration.
All six findings remain resolved/verified, and the increment is closed and
ready for physical archive.
## Suggested commit message

`fix(launcher): guard workspace destinations and surface partial adoption results`