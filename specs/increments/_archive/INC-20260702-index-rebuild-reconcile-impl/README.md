# Increment: Reconcile derived caches / index rebuild — implementation (index-engineer)

Status: pending
Date: 2026-07-02

## Goal

Execute the **index-engineer** step of INC-07 of the parent roadmap
(`Axiom.Spec/specs/increments/INC-20260702-axiom-redesign-roadmap/README.md`,
Phase C), implementing the concrete brief left by the migration-engineer
audit at `Axiom.Spec/specs/increments/INC-20260702-index-rebuild-reconcile/
README.md`:

1. A direct-scan bulk-enumeration function (`listArtifacts`) in
   `@axiom/workflow`'s `artifact-store.ts`, closing the audit's Finding 2
   (no way — cached or direct-scan — existed to list increments/bugs/plans).
2. `list` subcommands on `axiom-increment`, `axiom-bug`, `axiom-plan`.
3. `axiom index rebuild` / `axiom index validate` as thin, cache-less
   wrappers over the same scan, closing the audit's Finding 1 (the
   generated `AGENTS.md` boilerplate already promises these commands;
   they didn't exist).
4. A new `@axiom/doctor` check category (`IX-001`, `artifact-index`)
   validating `metadata.yml` parse-validity, closing Finding 3.

Explicitly NOT built: a `.axiom/cache/*.index.json` persistent cache file
(the audit's two-tier recommendation defers this until a concrete
consumer or measured performance need justifies it).

## Context

Parent chain read in full before this implementation:

1. Parent roadmap — `Axiom.Spec/specs/increments/
   INC-20260702-axiom-redesign-roadmap/README.md` (INC-07 entry, Phase C).
2. migration-engineer audit — `Axiom.Spec/specs/increments/
   INC-20260702-index-rebuild-reconcile/README.md` (this increment's
   direct predecessor; its "Next step recommendation" section is the
   literal brief followed here).
3. INC-06 design/impl/validator chain (referenced by the audit) for the
   `artifact-store.ts` shape (`metadata.yml` folder-per-instance) that
   `listArtifacts` builds on top of, unmodified.

Code read directly (not from prior specs' prose) before implementing:

- `Axiom/packages/workflow/src/artifact-store.ts` (full file) —
  confirmed the exact `ArtifactMetadata`/`ArtifactStoreError` shapes and
  the `resolveSpecRoot`/`resolveArtifactDir` path helpers to build
  `listArtifacts` on top of, consistently.
- `Axiom/apps/cli/src/commands/context.ts` (full file) — the
  `run*`/`register*` split pattern reused verbatim for
  `apps/cli/src/commands/index-cmd.ts`.
- `Axiom/apps/cli/src/commands/axiom-increment.ts`, `axiom-bug.ts`,
  `axiom-plan.ts`, and `artifact-metadata-cli.ts` (full files) — confirmed
  the existing shared-helper pattern (`addLinkPlanSubcommand`,
  `addExternalRefSubcommands` in `artifact-metadata-cli.ts`, reused by
  all three CLI files) as the right place to add a shared
  `addListSubcommand`/`runListCommand`, rather than duplicating list
  logic three times.
- `Axiom/apps/cli/src/index.ts` — confirmed the actual CLI entrypoint
  filename (`index.ts`) to justify naming the new file `index-cmd.ts`
  (avoids collision) and confirmed the registration-order convention.
- `Axiom/packages/doctor/src/checks.ts` (full `DoctorCheck` shape, the
  `pass/fail/warn/skip` helpers, and `runAgentsCatalogCoverageCheck` as
  a representative check with the same "read → skip-if-unresolved →
  fail/pass with evidence" shape) + `Axiom/packages/doctor/src/index.ts`
  (barrel + `runDoctorChecks` aggregator).
- `Axiom/packages/doctor/package.json` and `packages/doctor/tsconfig.json`
  — confirmed `@axiom/doctor` had NO existing dependency on
  `@axiom/workflow` (needed to add both the `package.json` dependency
  and the TS project-reference/path-mapping entry for the import to
  resolve under `tsc -b`).
- Existing test files (`packages/workflow/tests/artifact-store.test.ts`,
  `apps/cli/tests/context.test.ts`, `apps/cli/tests/artifact-metadata-cli.
  test.ts`, `apps/cli/tests/axiom-increment-metadata.test.ts`,
  `packages/doctor/tests/checks.test.ts`) — read in full to match test
  conventions (tmpdir-per-test, `beforeEach`/`afterEach` cleanup,
  `resolveProject` + `scaffoldMinimalProject` helper for doctor checks).

## Scope

- `Axiom/packages/workflow/src/artifact-store.ts`: add `listArtifacts`
  (bulk-enumeration, direct-scan, skip-on-parse-failure) + re-export from
  `Axiom/packages/workflow/src/index.ts`.
- `Axiom/apps/cli/src/commands/artifact-metadata-cli.ts`: add a shared
  `runListCommand`/`addListSubcommand`, reused by all three CLI files
  (same pattern as the existing `runLinkCommand`/`addLinkPlanSubcommand`).
- `Axiom/apps/cli/src/commands/axiom-increment.ts`,
  `axiom-bug.ts`, `axiom-plan.ts`: wire `addListSubcommand` into each
  `register*` function.
- `Axiom/apps/cli/src/commands/index-cmd.ts` (new file): `axiom index
  rebuild` / `axiom index validate`, cache-less, wrapping `listArtifacts`.
- `Axiom/apps/cli/src/index.ts`: register `index-cmd`'s `registerIndex`.
- `Axiom/packages/doctor/src/checks.ts`: add `runArtifactIndexChecks`
  (`IX-001`, category `artifact-index`), reusing `listArtifacts`.
- `Axiom/packages/doctor/src/index.ts`: wire the new check into
  `runDoctorChecks` + barrel exports.
- `Axiom/packages/doctor/package.json` + `packages/doctor/tsconfig.json`:
  add the `@axiom/workflow` dependency + TS path/reference (previously
  absent — `@axiom/doctor` had zero dependency on `@axiom/workflow`).
- Tests: `packages/workflow/tests/artifact-store.test.ts` (extended),
  `apps/cli/tests/artifact-metadata-cli.test.ts` (extended),
  `apps/cli/tests/index-cmd.test.ts` (new),
  `packages/doctor/tests/checks.test.ts` (extended),
  `apps/cli/tests/axiom-increment-metadata.test.ts`,
  `axiom-bug-metadata.test.ts`, `axiom-plan-create.test.ts` (each
  extended with one end-to-end `list`-sees-the-created-artifact scenario).

## Non-goals

- No `.axiom/cache/*.index.json` file or any persistent cache — explicit
  out-of-scope per the audit's Recommendation item 2. `listArtifacts` and
  `axiom index rebuild` are both pure, cache-less, direct-scan.
- No `search.index`, `graph.index`, `state.index` — no concrete consumer
  exists for these; unchanged from the audit's non-goals.
- No changes to `@axiom/workflow`'s state machine, `state-store.ts`,
  `hooks.ts`, `workflows-loader.ts`, or the `WorkflowState`/transition
  model. `listArtifacts` reads `metadata.yml` only, exactly like
  `loadArtifactMetadata` already does; it does not touch
  `workflow-state.json`.
- No change to the `link-*`/`external-ref` command surface landed by
  INC-06 — `artifact-metadata-cli.ts` gained one new shared helper
  (`runListCommand`/`addListSubcommand`) additively, alongside the
  existing ones, unmodified.
- No renaming or restructuring of `axiom-increment.ts` /
  `axiom-bug.ts` / `axiom-plan.ts`'s existing subcommand wiring beyond
  appending the one new `addListSubcommand(...)` call to each
  `register*` function.
- No unified `axiom artifact list --kind <kind>` command (the audit's
  Recommendation item 1 offered this as an alternative to three
  `list` subcommands) — the three-subcommand form was chosen because it
  matches the increment's explicit instruction ("Wire this into new
  `list` subcommands on `axiom-increment.ts`, `axiom-bug.ts`,
  `axiom-plan.ts`") and keeps each artifact kind's CLI surface
  self-contained, consistent with how `create`/`link-*`/`external-ref`
  are already kind-scoped rather than unified.

## Acceptance criteria

- [x] `listArtifacts(projectRoot, kind, specRelPath?)` implemented in
      `@axiom/workflow`'s `artifact-store.ts`, exported from the package
      barrel. Empty-folder case returns `{ entries: [], failures: [] }`
      without throwing. A single malformed `metadata.yml` is collected in
      `failures` without aborting the rest of the scan.
- [x] `list` subcommands added to `axiom-increment`, `axiom-bug`,
      `axiom-plan`, showing id/title/status per instance (human-readable
      by default, `--json` for the full result).
- [x] `axiom index rebuild` implemented: re-scans increments/bugs/plans
      via `listArtifacts` and reports counts per kind + total. No
      `.axiom/cache/*.json` file is written (verified by an explicit
      test asserting `.axiom/` does not exist after `rebuild`).
- [x] `axiom index validate` implemented: re-scans and reports any
      `metadata.yml` parse failures (kind/id/errorKind/errorMessage per
      failure). Exit code `1` if any failure is found, `0` otherwise.
- [x] New `@axiom/doctor` check category `IX-001` (`artifact-index`)
      added, validating `metadata.yml` parse-validity across
      increments/bugs/plans via `listArtifacts` (not a re-implementation
      of the scan). Follows the existing `pass/fail/warn/skip` + evidence
      convention; `skip`s cleanly when the project is not resolved, same
      condition as every other check in `checks.ts`.
- [x] No `.axiom/cache/*.index.json` file or persistent cache introduced
      anywhere in this changeset.
- [x] `@axiom/workflow`'s state machine, state-store, hooks, and
      workflows-loader are unmodified (only `artifact-store.ts` and
      `index.ts`'s barrel changed in that package).
- [x] Tests added for: `listArtifacts` (happy path, skip-on-parse-failure,
      empty-folder case, custom `specRelPath`), the shared
      `runListCommand` (all three kinds, empty case, parse-failure case),
      `axiom index rebuild`/`validate` (empty project, mixed counts,
      cache-less assertion, corrupt-metadata case), the new doctor check
      (`skip`/`pass`/`fail` branches), and one end-to-end
      create-then-list scenario per CLI file (increment/bug/plan).
- [x] `npm run typecheck`, `npm run build`, and `npm test` executed for
      the full monorepo; results compared against the confirmed baseline
      (12 failed files / 13 failed tests / 1298 passed) rather than
      assumed.
- [x] Increment spec written in `Axiom.Spec`, documenting files changed,
      design decisions, and real test results.

## Open questions

None blocking implementation. One design choice made without asking
(documented below as an assumption, not deferred): whether `list` should
be three kind-scoped subcommands or one unified `axiom artifact list
--kind <kind>` command. The parent increment's instruction (item 2 of
the index-engineer brief) explicitly named three subcommands on the
three existing files, so this was treated as already decided rather than
re-litigated — see "Non-goals" above.

## Assumptions

- `listArtifacts` silently skips (does not report as a `failure`) an
  artifact-instance folder that exists but has no `metadata.yml` at all
  (e.g. a folder created by some other process, or a pre-INC-06 relic).
  This mirrors `loadArtifactMetadata`'s own existing contract (`ok(
  undefined)` for "file absent", which is not a `metadata.yml`
  parse-error) — a missing file is a different condition from an
  unparseable one, and conflating them would make `axiom index validate`
  noisy for a case that is not actually a "corrupted index" symptom.
- `axiom index rebuild`'s exit code is always `0` regardless of
  individual parse failures (it reports them in the message/JSON output,
  but the "job" of a rebuild — re-confirming a clean scan is possible —
  succeeded even if some entries are individually broken). `axiom index
  validate`'s exit code is `1` on any failure, matching the
  addendum's framing of `validate` as the coherence-check surface (a
  clear CI-friendly signal) versus `rebuild` as the "prove the scan
  works" surface. This mirrors the audit's own recommendation language
  ("rebuild re-scans and reports what it found... validate re-scans and
  reports any metadata.yml that fails to parse").
- The doctor check's `evidence` string embeds up to N failure details
  inline (no truncation was added) since realistic artifact volumes
  today are near-zero (per the audit's own volume assessment) — if this
  becomes a real evidence-string-length problem at scale, that is a
  natural, deferred follow-up once volume actually exists, not something
  to speculatively engineer now.
- `@axiom/doctor` previously had zero dependency on `@axiom/workflow`.
  Adding it (via `package.json` `dependencies` + `tsconfig.json`
  `paths`/`references`) was necessary and in-scope — the audit's own
  Recommendation item 3 explicitly asked for a doctor check reusing
  `@axiom/workflow`'s scan, which is impossible without this dependency
  edge. This is treated as a natural, minimal consequence of implementing
  the recommendation, not scope creep.

## Implementation notes

### 1. `listArtifacts` (`Axiom/packages/workflow/src/artifact-store.ts`)

```ts
export interface ArtifactListEntry {
  readonly id: string;
  readonly metadata: ArtifactMetadata;
}
export interface ArtifactListFailure {
  readonly id: string;
  readonly error: ArtifactStoreError;
}
export interface ListArtifactsResult {
  readonly entries: ReadonlyArray<ArtifactListEntry>;
  readonly failures: ReadonlyArray<ArtifactListFailure>;
}

export function listArtifacts(
  projectRoot: string,
  kind: ArtifactKind,
  specRelPath?: string,
): ListArtifactsResult
```

Implementation: `fs.readdirSync(kindDir, { withFileTypes: true })` over
`<specPath>/{increments,bugs,plans}/`, wrapped in `try/catch` so an
absent folder (no artifacts of this kind created yet) returns an empty
result rather than throwing. For each subdirectory, delegates to the
existing `loadArtifactMetadata` (no parsing logic duplicated) — `err(...)`
results go to `failures`, `ok(undefined)` (folder without `metadata.yml`)
is silently skipped, `ok(metadata)` goes to `entries`. Exported from the
package barrel (`packages/workflow/src/index.ts`) alongside the existing
artifact-store exports.

### 2. `list` subcommands (shared helper + 3 call sites)

Added to `Axiom/apps/cli/src/commands/artifact-metadata-cli.ts`
(the file INC-06 already established as the shared home for
cross-kind, metadata-only CLI helpers):

```ts
export function runListCommand(args: RunListCommandArgs): RunListCommandResult
export function addListSubcommand(cmd: Command, options: {...}): void
```

`runListCommand` calls `listArtifacts`, maps entries to
`{ id, title, status }`, and builds a human-readable message
(`<id>  [<status>]  <title>` per line, or `"No hay <kind>s
registrados."` for the empty case). Parse failures are appended as a
warning line (`N entrada(s) con metadata.yml inválido, omitida(s) del
listado: ...`) without failing the command (`exitCode: 0` always —
listing what parsed successfully is still a successful list operation;
`axiom index validate` is the surface responsible for treating parse
failures as an error condition).

Wired via one `addListSubcommand(cmd, { ownKind: '<kind>', tag: TAG })`
call appended to each of `registerAxiomIncrement`, `registerAxiomBug`,
`registerAxiomPlan` in their respective files.

### 3. `axiom index rebuild` / `axiom index validate`
   (`Axiom/apps/cli/src/commands/index-cmd.ts`, new file)

Confirmed `apps/cli/src/index.ts` is the actual CLI entrypoint filename
before naming this file — `index-cmd.ts` avoids the collision the
increment's instructions flagged as a risk.

`run*`/`register*` split, pattern-matched on `context.ts`:

```ts
export function runIndexRebuild(args: IndexRebuildArgs): IndexRebuildResult
export function runIndexValidate(args: IndexValidateArgs): IndexValidateResult
export function registerIndex(program: Command): void
```

`runIndexRebuild` calls `listArtifacts` once per kind (`increment`,
`bug`, `plan`), reports `{ kind, count, failureCount }` per kind plus
totals. Always `exitCode: 0` (see Assumptions). Does not write any file
— verified explicitly by a test asserting `.axiom/` does not exist under
the project root after `rebuild` runs.

`runIndexValidate` calls `listArtifacts` once per kind and surfaces every
`ArtifactListFailure` as `{ kind, id, errorKind, errorMessage }`.
`exitCode: 1` if `failures.length > 0`, else `0`.

Registered in `apps/cli/src/index.ts` via `registerIndex(program)`,
placed after `registerGateway` (0034's last command) and before
`registerTui`, matching the file's existing registration-order
convention (new commands appended just before the TUI registration).

### 4. `@axiom/doctor` check `IX-001` (`artifact-index` category)

Added to the end of `Axiom/packages/doctor/src/checks.ts`:

```ts
export function runArtifactIndexChecks(resolution: ProjectResolution): DoctorCheck[]
```

Follows the exact `pass/fail/warn/skip` + evidence convention already
established (mirrors `runAgentsCatalogCoverageCheck`'s shape most
closely): `skip`s with the same `resolution.status !== 'resolved'`
condition used by every other check in the file; otherwise calls
`listArtifacts(resolution.rootPath, kind)` for all three kinds, and
`fail`s with a consolidated `kind '<id>' [<errorKind>]: <message>` list
if any `failures` were found, `pass`es with a count otherwise. Wired into
`runDoctorChecks` in `packages/doctor/src/index.ts` and exported from the
package barrel + a new `CATEGORY_ARTIFACT_INDEX` constant (matching the
existing `CATEGORY_TOOL_ROUTING`/`CATEGORY_MEMORY`/etc. barrel exports).

**Dependency edge added**: `@axiom/doctor` had no existing dependency on
`@axiom/workflow`. Added `"@axiom/workflow": "*"` to
`packages/doctor/package.json#dependencies` and a matching
`paths`/`references` entry in `packages/doctor/tsconfig.json` (same
pattern as the package's other 9 internal dependencies). Confirmed via
`npm run typecheck`/`npm run build` (both `tsc -b`, which resolves the
whole project-reference graph) that this resolves cleanly.

## Validation

Real validation was executed (not best-effort/inspection-only — commands
exist and were run):

1. **Baseline confirmed first**, before any change, via `npm test` on
   the unmodified tree: **12 failed files / 13 failed tests / 1298
   passed (1311 total)** — matches exactly what the migration-engineer
   audit and this increment's brief stated as the expected baseline.
2. `npm run typecheck` (`tsc -b`, full monorepo, all project references)
   — **clean, zero errors**, after adding the changes above (including
   the new `@axiom/doctor` → `@axiom/workflow` project reference).
3. `npm run build` (`tsc -b`, full monorepo) — **clean, zero errors**.
4. `npm test` (`vitest run`, full monorepo) — **12 failed files / 13
   failed tests / 1320 passed (1333 total)**. The failed-file list and
   failed-test count are byte-for-byte identical to the pre-change
   baseline (verified by listing every `FAIL` line and cross-checking
   against the baseline run): `apps/cli/tests/start.test.ts`,
   `packages/agents/tests/catalog.test.ts`,
   `packages/doctor/tests/checks.test.ts` (pre-existing
   `runDoctorChecks — repositorio Axiom real` failure —
   `resolveProject(process.cwd())` returns `'ambiguous'` in this
   environment, not `'resolved'`/`'not-found'` as the test's own
   comment anticipated; unrelated to `IX-001`, confirmed by scoped run
   below), `packages/model-routing/tests/*` (4 files, all
   "loads/parses the real policy from axiom.spec" — environment-path
   dependent), `packages/project-resolution/tests/resolver.test.ts`,
   `packages/skills/tests/catalog.test.ts`,
   `packages/telemetry/tests/audit-trail-sink.test.ts`,
   `packages/toolchain/tests/repair-add-gitignore.test.ts`,
   `packages/tui/tests/driver.test.ts` (2 tests, ANSI-color/terminal-
   width dependent assertions). None of these touch
   `artifact-store.ts`, `axiom-increment/bug/plan.ts`,
   `artifact-metadata-cli.ts`, `index-cmd.ts`, or `doctor/checks.ts`.
   **The +22 net new passing tests (1298 → 1320) are exactly the new
   tests added by this increment; zero regressions.**
5. **Scoped re-run** of only the touched/new test files
   (`packages/workflow/tests/artifact-store.test.ts`,
   `apps/cli/tests/artifact-metadata-cli.test.ts`,
   `apps/cli/tests/index-cmd.test.ts`,
   `packages/doctor/tests/checks.test.ts`,
   `apps/cli/tests/axiom-increment-metadata.test.ts`,
   `axiom-bug-metadata.test.ts`, `axiom-plan-create.test.ts`) via
   `npx vitest run <files>`: **72 passed, 1 failed** — the 1 failure is
   the same pre-existing `runDoctorChecks — repositorio Axiom real`
   baseline failure (environment-dependent `resolveProject` status in
   this workspace, unrelated to any check added here); every one of the
   **22 new tests this increment added passed**.

## Result

Implemented the full index-engineer brief left by the migration-engineer
audit, closing both concretely-evidenced gaps it found:

- **Finding 2 closed**: `listArtifacts` (in `@axiom/workflow`) plus
  `list` subcommands on `axiom-increment`/`axiom-bug`/`axiom-plan` now
  provide a working, direct-scan way to enumerate increments/bugs/plans
  — something that did not exist under any name before this increment.
- **Finding 1 closed**: `axiom index rebuild` and `axiom index validate`
  now exist as real CLI commands, matching what the generated
  `AGENTS.md` boilerplate has been promising since INC-03. Both are
  thin, cache-less wrappers over `listArtifacts`, deliberately not
  writing any `.axiom/cache/*.json` file (per the audit's two-tier
  recommendation).
- **Finding 3 closed**: `@axiom/doctor` gained its first check
  (`IX-001`, category `artifact-index`) over the artifact-store/
  metadata.yml surface introduced by INC-06, following the file's
  existing `pass/fail/warn/skip` convention exactly, and reusing
  (not duplicating) `listArtifacts`'s scan logic.

All three implementation pieces share one underlying scan primitive
(`listArtifacts`), avoiding the duplication risk the audit specifically
flagged when noting that a doctor check "reusing" the scan should not
reimplement it. No `.axiom/cache/*.index.json` persistent cache was
introduced anywhere, keeping the change consistent with
`Axiom.SDD/AGENTS.md`'s explicit bootstrap limit against speculative
mandatory indexes.

Full-monorepo `typecheck`/`build`/`test` were run and produced a clean
typecheck/build and a test result with the exact same pre-existing
baseline failures as before this change, plus 22 new passing tests and
zero regressions.

## General spec integration

No integration into `Axiom.Spec`'s consolidated knowledge base was
performed as part of this step. As with the migration-engineer audit
before it, `general-spec.md` does not exist in this repo (confirmed by
every prior spec in the INC-06/INC-07 chain; the closest equivalents
remain `Axiom.Spec/specs/00_Resumen_Ejecutivo.md` through
`08_Glosario.md`, not modified here). Per this repo's own established
pattern (INC-06's validator-reviewer step performed the final
consolidation once the full chain closed), stable-knowledge integration
is deferred to the validator-reviewer step that follows this one, once
the implementation itself has been independently re-verified —
consolidating now, before that independent check, would risk documenting
a shape that a validator-reviewer finding might still change.

## Closure rationale

Per `Axiom.SDD/AGENTS.md`'s closure rules, this increment is set to
`pending`, not `closed`, because:

- Changes ARE implemented and validated (goal, scope, and acceptance
  criteria are all satisfied on this implementer's own review).
- However, per the roadmap's own established chain pattern for this
  increment (migration-engineer → index-engineer → validator-reviewer),
  the validator-reviewer step — an INDEPENDENT re-verification of the
  diffs and a fresh re-run of `typecheck`/`build`/`test`, not trusting
  this document's own prose — has not yet occurred. INC-06's own closure
  was gated on exactly this same independent step, and this increment
  follows the same discipline rather than self-certifying closure.

## Next step recommendation

**validator-reviewer**, per the roadmap's chain pattern (same role that
closed INC-06). Concrete brief, derived from this implementation:

1. Independently re-read the diffs in `Axiom` directly (not from this
   document's prose): `packages/workflow/src/artifact-store.ts` +
   `index.ts`, `apps/cli/src/commands/artifact-metadata-cli.ts`,
   `axiom-increment.ts`, `axiom-bug.ts`, `axiom-plan.ts`,
   `index-cmd.ts` (new file), `apps/cli/src/index.ts`,
   `packages/doctor/src/checks.ts` + `index.ts`,
   `packages/doctor/package.json`, `packages/doctor/tsconfig.json`.
2. Independently re-run `npm run typecheck`, `npm run build`, and
   `npm test` from a clean state and re-confirm the exact same baseline
   failure set (12 files / 13 tests) is unchanged, with the new tests
   passing.
3. Confirm no `.axiom/cache/*.index.json` file or persistent cache was
   introduced anywhere (explicit non-goal check).
4. Confirm `@axiom/workflow`'s state machine/state-store/hooks were not
   touched beyond `artifact-store.ts`/`index.ts` (explicit non-goal
   check per this increment's own scope).
5. Close INC-07 (the parent roadmap entry) once satisfied, or report
   back with specific gaps if not.
