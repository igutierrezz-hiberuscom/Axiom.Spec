# Review ledger: INC-20260804-launcher-safe-plugins

Date: 2026-08-04
Flow: increment
Route: sdd
Scope: independent review after apply, ACC-008 / R-12
Prior freeze: `8073fda966c740b2f5392f5b13c9213cdb075b5e24b3ad32790529d87e215e04`
(historical review evidence; not the final closure hash).

## Sweeps

1. Contract and ownership: resolver, dispatcher, discovery, built-in bridge,
   HTTP routes, legacy alias, tracker ports and response projection.
2. Adversarial data flow: filesystem shadowing, command authority, field values,
   arrays/multiselect, mapping writes, secrets, externalRefs and provider errors.
3. Acceptance and coverage: absence/duplicates, malformed discovery, UI and
   transport behavior, preview/confirmed gates, NullTracker and fake ADO tests.
4. Validation: build, TypeScript diagnostics, JavaScript syntax, diff checks;
   no new findings after the preceding sweep.

## Historical baseline findings (pre-fix)

| id | lens | location | severity | status | evidence |
| --- | --- | --- | --- | --- | --- |
| REVIEW-001 | review | apps/cli/src/commands/app-api.ts:371 | CRITICAL | open | Introduced by the apply. `appPluginCatalogView` spreads the parsed plugin and each action into the HTTP catalog. The runtime validator is structural and does not reject or strip unknown properties, so a filesystem plugin containing fields such as `token`, `pat` or `credentials` is returned by `GET /api/projects/:id/plugins` and forwarded by the launcher transport/UI. This violates RQ-007 and the explicit ban on moving secrets through plugin JSON, registry, UI or responses. |
| REVIEW-002 | review | apps/cli/src/commands/app-launcher.ts:207 | CRITICAL | open | The plugin response projection copies `result.message` verbatim and accepts any URL matching only `https?://`; it does not reject URL userinfo or redact credential-bearing/error content. The raw ADO response body is included by the pre-existing tracker at `packages/tracker-ado/src/ado-tracker.ts:706`, and the pre-existing `external-sync` catch preserves that message. The new plugin bridge makes that sensitive content an HTTP/UI result, so the exposure is introduced by this apply even though the lower-level raw-error source predates it. |
| REVIEW-003 | review | apps/cli/src/commands/app-launcher.ts:156 | WARNING | open | Introduced by the apply. `normalizePluginValues` checks only string/number/boolean or an array of strings/numbers, stringifies all values and joins arrays with `;`. It does not enforce each declared field's `type`, `select` options, `multiselect` cardinality/options, numeric `workItemId`, or the relation between declared fields and handler allowlisted fields. A direct request can therefore send arbitrary `workItemType`, `tags`, boolean/array `workItemId` or malformed mapping IDs into `runExternalSync`; the JSON transport prevents shell injection, but data integrity and the declared input contract are not enforced. The request envelope is also only TypeScript-cast at `apps/cli/src/commands/app-api.ts:1164`, so non-object `pluginId`/`actionId` values can become a 500 instead of a controlled 400. |
| REVIEW-004 | review | apps/cli/src/commands/app-plugins.ts:372 | WARNING | open | Introduced by the apply. Non-JSON files are silently skipped with `continue`, while RQ-002 requires warnings for non-JSON files as part of tolerant deterministic discovery. The existing test at `apps/cli/tests/app-plugins.test.ts:109` checks only that no plugin is loaded, not that the required warning is emitted. |
| REVIEW-005 | review | apps/cli/tests/app-launcher.test.ts:380 | WARNING | open | Validation coverage is incomplete for the highest-risk contract edges: no focused assertion covers unknown catalog properties/secrets, raw provider error redaction, credential-bearing externalRef URLs, select/number/multiselect validation, extra request values, malformed request envelopes, non-JSON discovery warnings, or a filesystem shadow that attempts to replace a built-in action while preserving handler authority. The positive gate/allowlist tests pass, but they do not falsify these remaining hypotheses. |

## Compliance observed

- `command` is not executed, tokenized, spawned or passed to a runner. Resolution
  requires a registered handler, exact command label and matching action kind;
  execution uses the registry's `externalSyncAction`.
- Filesystem discovery is sorted and tolerant for missing directories, malformed
  JSON, unsupported schema and duplicate plugin IDs. Built-in collision handling
  is deterministic and emits a warning; the filesystem declaration wins as the
  documented precedence.
- `POST /api/projects/:id/plugins/execute` and the plugin compatibility branch
  of `POST /api/projects/:id/launcher/execute` converge on the same dispatcher.
  The alias does not forward a client command.
- Read actions execute without the mutation confirmation gate; mutation actions
  return a preview and require `confirmed: true`. Preview does not call
  `runExternalSync`.
- `kind: none` uses `NullTracker` with no transport and empty `externalRefs`.
  `kind: ado` tests use injected fake transports and do not call the live ADO
  service.
- The UI disables non-executable declarations and renders results with text
  nodes; the transport uses the plugin-scoped route and preserves the gate.

## Introduced versus pre-existing

- Introduced in the apply: catalog pass-through of untrusted plugin properties,
  raw result projection in the plugin bridge, incomplete plugin value
  validation, and silent non-JSON discovery.
- Pre-existing lower-level behavior: `AdoWorkItemTracker` includes a provider
  response body in an exception and `runExternalSync` preserves that exception
  message. Those files were not part of the reviewed apply diff; the new plugin
  HTTP/UI surface is the introduced exposure path.
- No command execution path, NullTracker network path, or fake-transport bypass
  was found in the reviewed increment.

## Validation observed

- `npx vitest run apps/cli/tests/app-plugins.test.ts apps/cli/tests/app-launcher.test.ts apps/cli/tests/external-sync.test.ts apps/cli/tests/launcher-ado-bridge.test.ts apps/cli/tests/launcher-ado-workflow.test.ts apps/cli/tests/launcher-ado-peripherals.test.ts packages/tracker/tests/null-tracker.test.ts packages/tracker-ado/tests/ado-tracker.test.ts apps/cli/tests/e2e/launcher.e2e.test.ts`: 9 files and 124 tests passed.
- `npm run build`: passed.
- `get_errors` on the six reviewed TypeScript files: no errors found.
- `node --check apps/cli/static/launcher/launcher.js` and
  `node --check apps/cli/static/launcher/transport.js`: passed.
- `git diff --check` on the reviewed apply files: no whitespace errors.
- No live ADO call was observed; the tracker tests use fakes and the local-only
  tests use `NullTracker`.

## Historical recommendation (before N=1 re-review)

Status: pending at the initial review boundary.

Closure is not justified while REVIEW-001 and REVIEW-002 remain open, the
input contract is not enforced, and the non-JSON warning criterion is unmet.
The unchecked acceptance criterion for freeze/receipts/review/integration is
also intentionally left pending at that historical review boundary. This
historical review did not close governance or create final receipts.

## Actions for the orchestrator

- The initial review routed the open findings back to the apply owner; the N=1
  re-review below records their resolution.
- After the code is corrected, consolidate only stable rules into the owning
  files under `specs/00..08` or `context/**` as appropriate: strict plugin
  projection, field/value validation, safe error/URL projection and tolerant
  discovery warnings. No canonical files were edited by this review.
- The final freeze/receipt pass is recorded below under orchestrator authority.

Suggested commit message: `fix(launcher): harden plugin validation and response redaction`

## N=1 final re-review (2026-08-04)

Inputs were restricted to this ledger and the current fix diff. The baseline
rows above are historical; this section is the current review result.

### Current finding states

- REVIEW-001: verified, CRITICAL. `appPluginCatalogView` constructs an
  explicit response DTO. It copies only known plugin, tab, action and field
  properties; `field.options` is rebuilt as `{ value, label }`. `sourcePath`,
  `token`, `pat`, `credentials` and other unknown properties are not
  serialized. `command` is included only after the static handler, exact
  command label and action kind resolve successfully; non-allowlisted commands
  are omitted from the catalog.
- REVIEW-002: verified, CRITICAL. Result messages are redacted for URL
  userinfo, Bearer/Basic and token/PAT forms. External refs are projected to
  HTTP(S) host and pathname, dropping userinfo, query and fragment.
- REVIEW-003: verified, WARNING. `apiExecuteLauncherAction` rejects null,
  arrays and non-object bodies before reading `pluginId` or `actionId`.
  `apiExecutePluginAction` validates the JSON envelope and field types before
  resolving the project, action or bridge; both HTTP routes use that guarded
  path.
- REVIEW-004: verified, WARNING. Sorted discovery warns for non-JSON entries,
  and the focused test asserts the warning without aborting discovery.
- REVIEW-005: verified, WARNING. The current tests cover required and numeric
  validation, exact kind mismatch, extra-field allowlisting, alias null,
  catalog secrets, result redaction, non-JSON warnings, filesystem shadowing,
  and unsafe command/handler/path rejection. Invalid values are asserted to
  stop before the bridge is called.

### Compliance

- `command` is not sufficient authority and is never parsed, spawned or passed
  to a runner. Resolution requires the explicit static handler id, the exact
  allowlisted command label and the matching action kind; execution passes
  `handler.externalSyncAction` to `runExternalSync`.
- Filesystem shadowing is deterministic: the discovered plugin wins its
  duplicate id with a warning, while execution still resolves through the
  static handler registry. Read actions execute without confirmation; mutation
  actions preview until `confirmed: true`.
- `/plugins/execute` and the plugin-shaped `/launcher/execute` alias converge
  on `apiExecutePluginAction`; null and malformed envelopes return controlled
  400 responses.
- Confirmed local-only coverage uses `NullTracker`, reports `localOnly` and
  returns empty `externalRefs`; no live ADO call was observed.

### Residual risks

- The redactor is pattern-based; unlabelled provider secrets or sensitive URL
  path segments may still require a stricter provider-safe projection.
- The shadowing test verifies deterministic merge precedence rather than a
  full end-to-end execution of a malicious shadow plugin. The code path still
  enforces static handler resolution and uses `externalSyncAction`, so this is
  residual coverage risk, not an open finding in this review.
- The prior freeze is historical evidence only; the final freeze supersedes it.

### Validation observed

- Current final-diff validation: `npx vitest run --reporter=dot` over the nine
  launcher/plugin/bridge/tracker/E2E files: 9 files and 133 tests passed.
- Existing ledger evidence remains valid for the reviewed apply: `npm run
  build`, TypeScript diagnostics, JavaScript syntax checks and `git diff --check`
  passed; no live ADO call was observed.

### Recommendation

Status: closed. REVIEW-001 through REVIEW-005 are verified against the final
diff and validation above. The final freeze, receipt and canonical integration
are complete; the folder is ready for physical archive.

### Actions for the orchestrator

- The N=1 review status is now combined with the completed final governance
  pass; the final freeze and receipt are the closure evidence.
- If canonical consolidation is required later, update only the owning files
  under `specs/00..08` or `context/**` with the stable rules for explicit plugin
  projection, field validation, safe result projection and tolerant discovery.
  This re-review did not edit those files.

Suggested commit message: `fix(launcher): harden plugin validation and response redaction`

## Final orchestrator governance

REVIEW-001 through REVIEW-005 are closed/verified, canonical integration is
complete, and the final freeze plus `verify` receipt supersede the historical
review hash. The increment is closed and ready for physical archive.
