# Independent Review Ledger

Increment: `INC-20260804-cli-commands-package-output`
Flow: `increment`
Route: `sdd`
Scope: ACC-004 / R-01, post-apply worker diff
Review date: 2026-08-04
Sweeps: 4 (dry after the fourth sweep)

| id | lens | location | severity | status | evidence |
|---|---|---|---|---|---|
| REVIEW-001 | review | `candidate-freeze.json:2`; `receipts/2026-08-04T18-31-10.239Z-verify-success.json` | BLOCKER | resolved | The candidate was re-frozen after the final README update. `specsHash` equals the recomputed README SHA-256, and the `verify` receipt hash recomputes exactly from its payload with status `success`. |
| REVIEW-002 | review | `../../Axiom/docs/cli/tui.md:204` | WARNING | resolved | The guide now describes the package-owned modules and public `@axiom/cli-commands` entrypoint. The related moved-module comments were updated to remove stale owner claims. |
| REVIEW-003 | review | `../../Axiom/package-lock.json:2315` | WARNING | resolved | `npm install --package-lock-only --ignore-scripts` updated the workspace lock metadata to match the 26 declared runtime dependencies of `@axiom/cli-commands`. |
| REVIEW-004 | review | `README.md:117` | WARNING | resolved | The validation section now records the current 133 CLI files / 1268 tests and the combined 31 files / 402 tests, all passing. |
| REVIEW-005 | review | `../../Axiom/apps/cli/tests/e2e/cross-cutting-batch.e2e.test.ts:478` | SUGGESTION | info | The editor diagnostic says the `utf-8` argument gives 3 arguments where 2 are expected. The line is unchanged by this diff; the only hunk in this file changes the `runConfigure` import. The E2E passes (4/4), the affected build passes, and the full CLI suite passes, so this is pre-existing/not attributable to this increment. |
| REVIEW-006 | review | `../../Axiom/apps/cli/tests/e2e/cross-cutting-batch.e2e.test.ts:776` | SUGGESTION | info | Same classification as REVIEW-005 for the second editor diagnostic: the `writeFileSync` call is outside the changed hunk, the E2E passes, and no apply regression is evidenced. |
| REVIEW-007 | review | `../../Axiom/packages/cli-commands/src/commands/sync.ts:1` | SUGGESTION | info | Content-hash comparison against `HEAD` shows that 12 moved modules are byte-equivalent, while `sync.ts` and `workspace-adapter-templates.ts` retain local deltas. The README explicitly says those local changes were preserved during the move; focused and full tests pass, but the orchestrator should retain that provenance when closing the increment. |

## Validation observed

- `npm run build` from `Axiom/`: pass (`tsc -b`).
- Focused project-reference build for `@axiom/cli-commands`, `@axiom/tui`, `@axiom/mcp-tools`, and `@axiom/cli`: pass.
- Package entrypoints exist and resolve through Node: `packages/cli-commands/dist/index.js`, `packages/tui/dist/index.js`, and `packages/mcp-tools/dist/index.js`; their declarations exist at each `dist/index.d.ts`.
- Compiled CLI runtime checks pass for `--help`, `configure --help`, `sync --help`, `mcp --help`, and `model --help`; no `MODULE_NOT_FOUND` observed.
- Directed CLI tests: 11 files / 105 tests passed. TUI and MCP tests: 23 files / 268 tests passed.
- Cross-cutting E2E: 1 file / 4 tests passed.
- Full CLI suite: 133 files / 1268 tests passed, exit code 0.
- Static review found no runtime imports from `apps/cli/src` or `packages/*/src` in the moved package consumers, and no generated runtime paths under `dist/apps` or `dist/packages`. Remaining matches are comments or generated metadata outside the public runtime contract.
- The two editor diagnostics in `cross-cutting-batch.e2e.test.ts` remain visible, but are outside the changed hunk and are classified above as pre-existing/not attributable.

## Recommendation

`closed`. The runtime/build/ownership criteria are compliant, the actionable
findings are resolved, canonical integration is complete, and the remaining
editor diagnostics are pre-existing and outside the diff. The folder is ready
for physical archive.

