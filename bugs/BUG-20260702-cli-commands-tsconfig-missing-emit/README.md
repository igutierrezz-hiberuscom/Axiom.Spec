# Bug: `packages/cli-commands/tsconfig.json` prevents CLI command files from being emitted to `dist/`

Status: pending
Date: 2026-07-02
Found during: INC-08 validator-reviewer (`Axiom.Spec/specs/increments/INC-20260702-write-scope-validation-reconcile-validator/README.md`)
Rediscovered during: INC-20260702-schemaversion2-multirepo-primary-reconcile-impl
(cli-implementer) and independently re-confirmed, same root cause, during
INC-20260702-schemaversion2-multirepo-primary-reconcile-validator
(validator-reviewer) — see "Rediscovery" section below.
Rediscovered again (blast radius widened 8 → 11) during
INC-20260702-configure-upgrade-repair-reconcile-validator (validator-reviewer,
final role of INC-22) — see "Second rediscovery" section below.

## Expected behavior

`apps/cli/dist/commands/*.js` should contain a compiled output for every
`apps/cli/src/commands/*.ts` file after `npm run build`, so `node dist/
index.js <command> --help` works for every registered command.

## Actual behavior

`apps/cli/dist/commands/{_shared,configure,sync,upgrade,model,components}.js`
are never emitted, even after a full clean rebuild (all `tsconfig
.tsbuildinfo` files and `dist/` directories deleted first). This breaks
`node dist/index.js <cmd> --help` for any command that transitively
imports one of these 6 files (confirmed: `axiom validate changes --help`,
`axiom index rebuild --help`, and the pre-existing baseline `axiom init
--help` all fail identically). As of the second rediscovery below, the
affected set has grown to **11** files total (see "Second rediscovery").

## Rediscovery (same root cause, 2 more affected files)

Independently rediscovered twice during the D3 (`axiom.yaml schemaVersion:
2`) cutover chain, confirmed to be the exact same `cli-commands/tsconfig
.json` cross-include mechanism, now affecting **8** files instead of 6:
the original 6 (`_shared`, `configure`, `sync`, `upgrade`, `model`,
`components`) plus 2 new ones added since this bug was filed
(`index-cmd`, `validate-changes` — both post-date INC-08). Confirmed by
the validator-reviewer via direct reproduction: a clean `npm run build`
(exit 0, zero errors) still leaves these 8 files unemitted in
`apps/cli/dist/commands/`, and `packages/cli-commands/dist/apps/cli/src/
commands/` does correctly emit them (from the same source), confirming
the ownership/skip mechanism described below is unchanged. Not fixed by
either rediscovery — both treated it as out of scope and used the same
`packages/cli-commands/dist/` files as a throwaway, uncommitted, local
workaround to run the real CLI binary for their respective increments'
walkthroughs, deleting the workaround afterward.

## Second rediscovery (blast radius widened again, 8 → 11 files)

Independently reproduced a third time during INC-22's `configure`/`upgrade`/
`repair` chain (validator-reviewer role). The `-tui` increment
(`INC-20260702-configure-upgrade-repair-reconcile-tui`) added 3 new entries to
`packages/cli-commands/tsconfig.json`'s `include` list — `repair.ts`,
`toolchain.ts`, `mcp.ts` — needed because the new `@axiom/cli-commands`
`runRepair`/`formatRepairResult` re-export transitively requires
`repair.ts`'s own imports (`runToolchainRepair` from `toolchain.ts`,
`runMcpRepair` from `mcp.ts`) to be part of the same TS project-references
graph. Confirmed by direct reproduction (clean `rm -rf apps/cli/dist
packages/cli-commands/dist *.tsbuildinfo` + `npm run build`, exit 0, zero
errors): `apps/cli/dist/commands/{repair,toolchain,mcp}.js` are absent,
exactly like the original 8 files, and
`packages/cli-commands/dist/apps/cli/src/commands/{repair,toolchain,mcp}.js`
are emitted instead — the identical ownership/skip mechanism, unchanged.
**Total affected files is now 11**: the original 6
(`_shared`, `configure`, `sync`, `upgrade`, `model`, `components`), the 2 from
the first rediscovery (`index-cmd`, `validate-changes`), and these 3 new ones
(`repair`, `toolchain`, `mcp`). This is a **widening of an already-tracked
bug's blast radius**, not a new or distinct defect — every new `tsconfig.json`
`include` entry that cross-includes an `apps/cli/src/commands/*.ts` file into
`cli-commands`'s composite project reproduces the same mechanism
automatically. Not fixed here — same "fix scope (not done here)" rationale as
prior rediscoveries: `npm run typecheck`/`npm test` (vitest, source-based) are
unaffected; only `node apps/cli/dist/index.js repair --help` (the real,
standalone-built CLI binary) would be affected, and INC-22's own increments
already used the same documented `cli-commands/dist/` throwaway workaround
pattern where a real-binary smoke check was attempted.

## Root cause (confirmed, not just observed)

`Axiom/packages/cli-commands/tsconfig.json` cross-includes those exact 6
files (physically located under `apps/cli/src/commands/`) as inputs to a
second, separate composite TypeScript project. This prevents `apps/cli`'s
own project from being the one that emits them, and `cli-commands`'
project apparently doesn't emit them to the expected `apps/cli/dist/`
location either.

## Confirmed NOT caused by

INC-08's write-scope validation work — `axiom init --help` (unrelated to
INC-08, predates INC-01-08 entirely) fails identically. No circular
dependency between `@axiom/doctor`, `@axiom/core`, `@axiom/installer`
was found.

## Impact

Does not block any current increment's test suite (all tests run against
source via `vitest`, not the compiled `dist/`). Only affects a real
end-user running the built CLI binary via `node dist/index.js` for the
affected 6 commands' `--help`/execution paths.

## Fix scope (not done here)

Requires redesigning `packages/cli-commands/tsconfig.json`'s file
ownership so the (now 11) cross-included files are emitted exactly once,
to the correct `apps/cli/dist/` location — likely either removing the
redundant `include` entries from `cli-commands/tsconfig.json` (since
`apps/cli`'s own build already compiles those files) or restructuring
`cli-commands`'s re-export barrel to not need direct compilation of
`apps/cli/src/...` sources. Out of scope for the increment that found it
(INC-08) and for all increments that rediscovered it since (including
INC-22's `-tui` and `-validator` roles); tracked here per
`Axiom.SDD/AGENTS.md`'s bug-tracking convention. Not independently
blocking: `npm run typecheck`/`npm test` (vitest, source-based) are both
unaffected; only `node apps/cli/dist/index.js <cmd>` (the real,
standalone-built CLI binary) is affected. Given this has now surfaced
four times across four different increments, and its blast radius grows
by 2-3 files with every new command added to the `cli-commands` barrel,
it is an increasingly strong candidate for prioritization independent of
any specific feature increment — noted here, not escalated into a new
ticket (the mechanism, impact, and fix direction are unchanged from the
original filing; only the affected-file count has grown, tracked as an
update to this same ticket, not a distinct bug).
