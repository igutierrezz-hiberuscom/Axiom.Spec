# Review ledger: INC-20260804-tui-retirement

- Flow: `increment`
- Route: `sdd`
- Action: `ACC-005 / R-14`
- Review date: 2026-08-04
- Review mode: independent post-apply review
- Sweeps: 4, with the final sweep adding no new active runtime consumer
- Governance at review time: the candidate freeze `2db3c619...` was historical
	evidence and was not used to close the increment
- Scope constraint: no general specs, context files, archives, or Git mutations were performed

## Findings

| id | lens | location | severity | status | evidence |
|---|---|---|---|---|---|
| REVIEW-001 | review | `Axiom.Spec/specs/00..08`; `Axiom.Spec/context/**` | CRITICAL | resolved | Active claims now describe launcher/CLI/MCP as current surfaces; TUI references are explicitly historical or define the retired surface. No active source/context claim presents `axiom tui` as available. |
| REVIEW-002 | review | `metadata.yml`; `03_Criterios_Aceptacion.md`; `candidate-freeze.json`; `receipts/` | CRITICAL | resolved | Metadata/criteria are closed and integrated; the final freeze and verify receipt are emitted after the final README update and their hashes are independently recomputed. |
| REVIEW-003 | review | `Axiom/apps/cli/tests/tui-retirement.test.ts` | WARNING | resolved | The regression test now invokes compiled `axiom tui` and bare `axiom`, asserting non-zero behavior and absence of `@axiom/tui`; the compiled help regex checks the exact subcommand shape. |
| REVIEW-004 | review | `Axiom/apps/cli/dist/commands/tui.*` | WARNING | resolved | Generated stale TUI JavaScript, declarations and maps were removed from ignored `dist`; source/package wiring was already absent. |
| REVIEW-005 | review | `README.md` validation section | WARNING | resolved | Validation evidence now records 23 files and 221 tests, plus direct command regressions, doctor and readiness. |
| REVIEW-006 | review | predecessor `INC-20260804-cli-commands-package-output/03_Criterios_Aceptacion.md` | WARNING | resolved | The predecessor now states that TUI output was normalized historically and that ACC-005 supersedes the public interface contract. |

## Verified compliance

- The repo-wide matrix covers the implicit no-subcommand action, all five public flags (`--projects`, `--use`, `--topology`, `--model-validate`, `--components-show`), driver, router, screens, flows, prompts, previews, summaries, both init/workspace wizards, autoskills, and selection/cancellation behavior.
- The matrix distinguishes presentation-only TUI elements from operations. The operational replacements are real CLI, launcher, or MCP paths and were exercised by the directed tests rather than accepted from descriptions alone.
- Current source/config searches found no active `@axiom/tui`, `registerTui`, `commands/tui`, `packages/tui`, or `axiom tui` consumer in `Axiom`, `Axiom.SDD`, or `Axiom.Pruebas`; `Axiom` retains only negative retirement assertions. Stale generated TUI artifacts were removed from `dist`.
- Root and CLI TypeScript project references, the CLI path alias, Vitest alias, workspace dependency, and lockfile entry for TUI are removed. `npm ls @axiom/tui --depth=0` is empty.
- Shared runners remain owned by `@axiom/cli-commands` and are still consumed by CLI/launcher/MCP paths. Build and tests confirm the moved command modules compile and load.
- Archived increment history still contains TUI references; those matches were not treated as active consumers.

## Observed validation

- `npm run build`: PASS.
- `npx vitest run` directed CLI/launcher/MCP and retirement suite: 23 files, 221 tests, PASS.
- `npm run doctor`: PASS, 46/61 OK, 0 failures, 3 warnings, 12 skipped.
- `npm run readiness:first-project`: PASS.
- `node apps/cli/dist/index.js --help`: exit 0; no `tui` command.
- `node apps/cli/dist/index.js tui`: exit 1; `error: unknown command 'tui'`.
- Bare compiled CLI: no implicit TUI action; exit 1 with help.
- `get_errors` on the touched CLI entrypoint, retirement tests, onboarding regression, and `@axiom/cli-commands` barrel: no errors.
- `git diff --check`: exit 0; only existing LF/CRLF conversion warnings were emitted.

## Recommendation

`closed` for implementation, review and canonical integration. The runtime
retirement, direct regressions, stale dist cleanup, metadata/criteria closure,
canonical claim reconciliation, final freeze and verify receipt are complete;
the folder is ready for physical archive.

## Final orchestrator governance

The historical review boundary is superseded by the final freeze and `verify`
receipt emitted after the last README/criteria reconciliation. All findings
are resolved and the increment is ready for physical archive.
