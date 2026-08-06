# Bug: `packages/cli-commands/tsconfig.json` prevents CLI command files from being emitted to `dist/`

Status: closed
Date: 2026-07-02
Closed: 2026-08-06
Found during: INC-08 validator-reviewer (`Axiom.Spec/specs/increments/INC-20260702-write-scope-validation-reconcile-validator/README.md`)
Rediscovered during: INC-20260702-schemaversion2-multirepo-primary-reconcile-impl
(cli-implementer) and independently re-confirmed, same root cause, during
INC-20260702-schemaversion2-multirepo-primary-reconcile-validator
(validator-reviewer) — see "Rediscovery" section below.
Rediscovered again (blast radius widened 8 → 11) during
INC-20260702-configure-upgrade-repair-reconcile-validator (validator-reviewer,
final role of INC-22) — see "Second rediscovery" section below.

## Estado actual verificado

La correccion del problema de ownership esta incorporada en `Axiom` desde
`c4df64c`. Los comandos compartidos tienen ownership unico en
`packages/cli-commands/src/commands/` y su proyecto compuesto usa
`rootDir: "src"` e `include: ["src/**/*"]`. La CLI importa sus wrappers
`registerX` desde `@axiom/cli-commands`; por tanto, esos modulos ya no deben
duplicarse bajo `apps/cli/dist/commands/`. Los comandos que siguen siendo
propios de `apps/cli` continuan emitiendose bajo `apps/cli/dist/commands/`.

La retirada de gateway de `ACC-018` tambien dejo un residuo independiente: el
archivo fuente `apps/cli/src/commands/gateway.ts` ya no estaba registrado, pero
seguia dentro del `include` global de `apps/cli` y compilaba contra los tipos
retirados. Ese archivo y sus salidas stale de `dist` se eliminaron el
2026-08-06.

## Expected behavior

The TypeScript build must emit every shared command exactly once under
`packages/cli-commands/dist/commands/*.js`, publish
`packages/cli-commands/dist/index.js` and `dist/index.d.ts`, and let
`node apps/cli/dist/index.js <command> --help` resolve every registered
command. App-owned commands must continue to emit under `apps/cli/dist`.

## Actual behavior

The original missing-emission behavior is no longer reproducible. A focused
build of `packages/cli-commands/tsconfig.json` exits with code 0 and emits all
11 modules tracked by this bug, including `configure`, `sync`, `upgrade`,
`model`, `components`, `index-cmd`, `validate-changes`, `repair`,
`toolchain` and `mcp`. The built CLI returns exit code 0 for each affected
command's `--help` invocation.

The two TypeScript errors that appeared during the gateway retirement
(`functionalProfile` and `gatewayExpectation`) came from that residual source
file, not from `@axiom/cli-commands`. After deleting it, the clean root build
completes successfully again.

## Rediscovery (historical: same root cause, 2 more affected files)

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
the ownership/skip mechanism described below was unchanged. At the time,
neither increment fixed the issue: both treated it as out of scope
and used the same `packages/cli-commands/dist/` files as a throwaway,
uncommitted, local workaround to run the real CLI binary for their walkthroughs,
deleting the workaround afterward.

## Second rediscovery (historical: blast radius widened again, 8 → 11 files)

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
**The affected set reached 11 files**: the original 6
(`_shared`, `configure`, `sync`, `upgrade`, `model`, `components`), the 2 from
the first rediscovery (`index-cmd`, `validate-changes`), and these 3 new ones
(`repair`, `toolchain`, `mcp`). This is a **widening of an already-tracked
bug's blast radius**, not a new or distinct defect — every new `tsconfig.json`
`include` entry that cross-includes an `apps/cli/src/commands/*.ts` file into
`cli-commands`'s composite project reproduces the same mechanism
automatically. At that time it remained out of scope for INC-22; the ownership
change described above resolves the same mechanism for all 11 files.

## Root cause (historical, confirmed)

`Axiom/packages/cli-commands/tsconfig.json` cross-included command files that
were physically located under `apps/cli/src/commands/` as inputs to a second,
separate composite TypeScript project. The two project references disagreed
about the emitting owner, so the files were absent from the expected CLI
output even though the build reported no errors.

## Confirmed NOT caused by

INC-08's write-scope validation work — `axiom init --help` (unrelated to
INC-08, predates INC-01-08 entirely) fails identically. No circular
dependency between `@axiom/doctor`, `@axiom/core`, `@axiom/installer`
was found.

## Impact (historical)

The defect affected end users running the standalone compiled CLI. Source-based
Vitest tests did not expose it because they did not require the missing
artifact layout.

## Fix notes

The fix establishes one physical owner for shared commands:

- Move shared command sources under `packages/cli-commands/src/commands/`.
- Compile them with the package-local `rootDir` and `include`.
- Remove the duplicate CLI project reference and path alias for the package.
- Re-export `registerX` from the package barrel so `apps/cli` registers
  Commander without compiling the shared sources a second time.
- Keep app-only commands in the `apps/cli` project and its output tree.
- Delete app-owned retired command sources when the command is removed from the
  registry; `tsconfig` includes the whole app source tree.

This is the minimal correction for the root cause: it removes the conflicting
project ownership instead of copying generated files between `dist` folders.

## Acceptance criteria

- [x] `packages/cli-commands/tsconfig.json` has a package-local `rootDir` and
      `include`.
- [x] Every shared module affected by the historical report is emitted under
      `packages/cli-commands/dist/commands/`.
- [x] `apps/cli/src/index.ts` registers shared commands through the published
      package barrel.
- [x] The affected compiled CLI commands return exit code 0 for `--help`.
- [x] App-owned commands remain owned by and emitted from `apps/cli`.

## Validation

- `npx tsc -p packages/cli-commands/tsconfig.json --pretty false` — passed.
- Verified `packages/cli-commands/dist/index.js` and `dist/index.d.ts`, plus
  all 11 affected command modules — present.
- Smoke-tested `configure`, `sync`, `upgrade`, `model`, `components`,
  `index rebuild`, `validate changes`, `repair`, `toolchain` and `mcp` with
  `--help` — all returned exit code 0.
- `npx tsc -b --clean` followed by `npx tsc -b --pretty false` — passed after
  deleting the residual gateway source.
- `npx vitest run apps/cli/tests/start.test.ts apps/cli/tests/sync.test.ts
  packages/install-profiles/tests/composer.test.ts
  packages/install-profiles/tests/normalize.test.ts
  packages/installer/tests/installer.test.ts` — 5 archivos y 41 tests pasan.
- Se eliminaron `apps/cli/dist/commands/gateway.js`, su `.d.ts` y su source map
  stale; no vuelven a aparecer tras la reconstruccion.

## Result

Closed. The ownership correction was already present in the current Axiom
baseline (`c4df64c`). As a consequence of the gateway retirement in ACC-018,
the residual source that blocked the root build was also removed. This artifact
now records both pieces of evidence, and unrelated worktree changes were left
untouched.

## General spec integration

The stable implementation fact is already represented in
`Axiom.Spec/context/architecture/03-ciclo-de-vida-cli-y-orquestacion.md`:
`@axiom/cli-commands` owns and publishes the shared command modules. No change
to the product behavior specification in `specs/00..08` is required because
this bug changes build ownership, not the user-facing command contract.
