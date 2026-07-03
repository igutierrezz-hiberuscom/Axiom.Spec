# Increment: ADR/decision handling — cli-implementer (INC-11)

Status: closed
Date: 2026-07-02

Closed as part of INC-11 as a whole (schema-writer -> cli-implementer ->
validator-reviewer). See
`Axiom.Spec/specs/increments/INC-20260702-adr-decision-reconcile-validator/README.md`
for the chain's closure summary, independent verification of this
implementation's claims (including two real gaps found and fixed:
`@axiom/doctor`'s `IX-001` `ARTIFACT_INDEX_KINDS` omission, and an
untested `axiom adr supersede` reassignment edge case), and the
`general-spec.md` integration performed on closure.

## Goal

Implement the `adr` and `decision` artifact kinds designed by the
schema-writer increment (`Axiom.Spec/specs/increments/
INC-20260702-adr-decision-reconcile-design/README.md`) as real, working
code in the `Axiom` monorepo: extend the existing folder-per-artifact
`metadata.yml` mechanism (INC-06/07), add two new CLI commands
(`axiom-adr`, `axiom-decision`), and wire the two new kinds into the
existing derived index (`axiom index rebuild`/`validate`, INC-07). This
increment is `cli-implementer` in the roadmap's INC-11 sequence
(schema-writer -> cli-implementer -> validator-reviewer).

## Context

Parent roadmap: `Axiom.Spec/specs/increments/
INC-20260702-axiom-redesign-roadmap/README.md`, INC-11 ("Reconcile
ADR/decision handling"), additive/no-migration per the roadmap's own
marking (confirmed empty by the schema-writer's pre-design grep).

Direct input: the schema-writer design spec (read in full before this
increment started), which fully specified: the `ArtifactKind` extension,
the `BaseArtifactMetadataFields` extraction (resolving a real
`status: WorkflowState` vs `AdrStatus`/`DecisionStatus` typing conflict),
`AdrMetadata`/`DecisionMetadata` shapes with their own status
vocabularies and link shapes, ID prefixes/folder names, factory function
signatures, and the derived-index (not curated) recommendation for
`axiom index rebuild`/`validate`.

Files read fresh before implementing (per the design's own brief):
`Axiom/packages/workflow/src/artifact-id.ts`,
`Axiom/packages/workflow/src/artifact-store.ts`,
`Axiom/packages/workflow/src/index.ts`,
`Axiom/apps/cli/src/commands/index-cmd.ts`,
`Axiom/apps/cli/src/commands/axiom-bug.ts`,
`Axiom/apps/cli/src/commands/axiom-plan.ts`,
`Axiom/apps/cli/src/commands/artifact-metadata-cli.ts`,
`Axiom/apps/cli/src/index.ts`, plus the existing test files
(`artifact-store.test.ts`, `artifact-metadata-cli.test.ts`,
`index-cmd.test.ts`) as the test-pattern precedent.

### Baseline confirmed before implementing

Ran `npm run typecheck` (clean) and `npm test` (full monorepo) before
any change: **12 failed test files, 13 failed tests, 1408 passed / 1421
total** — matches the task brief's expected baseline exactly. These 12
files are pre-existing, environment-dependent failures unrelated to this
increment (TUI terminal-snapshot tests, tests reading real repo paths
like `axiom.spec/config/*.yaml` from a dev machine, a retention-sweep
timing test) — none touch `artifact-store`/`artifact-id`/CLI artifact
commands.

## Scope

- `Axiom/packages/workflow/src/artifact-id.ts`: extend `ArtifactKind`
  with `'adr' | 'decision'`; add `ADR`/`DEC` prefixes.
- `Axiom/packages/workflow/src/artifact-store.ts`: add `adr`/`decision`
  to `ARTIFACT_FOLDER`; extract `BaseArtifactMetadataFields` (no
  `status`) per the design, re-derive `BaseArtifactMetadata` from it
  without changing its effective shape; add `AdrMetadata`/
  `DecisionMetadata`, `AdrStatus`/`DecisionStatus`, `AdrLinks`/
  `DecisionLinks`; extend the `ArtifactMetadata` union; add
  status-vocabulary-aware parse/serialize branches; add
  `makeInitialAdrMetadata`/`makeInitialDecisionMetadata` factories.
- `Axiom/packages/workflow/src/index.ts`: re-export the new types and
  factories.
- New `Axiom/apps/cli/src/commands/axiom-adr.ts` and `axiom-decision.ts`:
  `create`, reused `link-plan`/`link-increment` (bidirectional, via
  `artifact-metadata-cli.ts`'s existing helper), reused `list`, plus
  ADR-only `supersede <old-id> <new-id>`.
- `Axiom/apps/cli/src/index.ts`: register both new commands.
- `Axiom/apps/cli/src/commands/index-cmd.ts`: add `'adr'`/`'decision'`
  to `SCANNED_KINDS`.
- Tests: `artifact-store.test.ts` (round-trip, folder layout, links,
  externalRefs, status-vocabulary-rejection), two new CLI test files
  (`axiom-adr.test.ts`, `axiom-decision.test.ts`), `index-cmd.test.ts`
  extended to cover the two new kinds.
- Three pre-existing narrowing bugs surfaced and fixed (see
  "Implementation notes").

## Non-goals

- No curated/versioned ADR index — per the design's explicit
  recommendation, `axiom index rebuild`/`validate` reuse the existing
  cache-less `listArtifacts` scan with zero new index-building logic.
- No `axiom.yml` `paths` configurability for folder names — out of
  scope per the design.
- No changes to `WorkflowState`, `workflows.yaml`, or the state-machine
  engine — ADR/Decision remain non-state-machine-driven artifacts.
- No semantic graph traversal for `supersedes`/`supersededBy` beyond the
  plain ID fields (e.g. no "resolve full supersession chain" helper).
- No new `@axiom/doctor` check — left to `validator-reviewer` to decide
  whether one is warranted, per the design's own hand-off note (this
  increment's own vitest coverage already validates shape, including
  the status-vocabulary-rejection case).

## Open questions (resolved during this increment)

The schema-writer design left two CLI-ergonomics questions to
`cli-implementer`:

1. **Should `create` require `context`/`decision`/`consequences` (ADR)
   or `rationale` (Decision) at creation time, or allow empty strings
   filled in later?** Resolved: **optional at creation**, defaulting to
   `''` via the factory functions (`makeInitialAdrMetadata`/
   `makeInitialDecisionMetadata` already default this way per the
   design's own factory spec) — consistent with `PlanMetadata`'s
   `taskType`/`targetRepos` empty-by-default precedent cited in the
   design.
2. **Should `link`/`supersede` update both sides atomically in one
   command, or require two calls?** Resolved: **one command, both
   sides**, for both `link-*` (reusing the existing bidirectional
   `runLinkCommand` helper, same as increment/bug/plan already do) and
   `supersede` (a single `axiom adr supersede <old-id> <new-id>` call
   updates `supersededBy` on the old ADR and appends to `supersedes` on
   the new one in the same invocation) — consistent with the existing
   link commands' established UX, and avoids leaving the two ADRs in an
   inconsistent state between two separate CLI invocations.

## Assumptions

- `supersede` sets the OLD ADR's `status` to `'superseded'`
  automatically (not left to the caller to set manually via a separate
  status-update command, since no such command exists for ADR/Decision
  today — they have no `update`/`status` sub-command, matching the
  design's brief which only asked for `create`/`link`/`list` +
  ADR-specific `supersede`). The NEW ADR's own `status` is left
  untouched by `supersede` (it does not become `'accepted'`
  automatically — that would be a separate, unrequested inference).
- `supersede` is idempotent on the `supersedes` side: calling it twice
  with the same `old-id`/`new-id` pair does not duplicate `old-id` in
  the new ADR's `links.supersedes` array (covered by a dedicated test).

## Implementation notes

### Files changed

- `Axiom/packages/workflow/src/artifact-id.ts` — `ArtifactKind` union
  and `ARTIFACT_ID_PREFIX` map extended with `'adr'` (`ADR`) and
  `'decision'` (`DEC`). No other change; `generateArtifactId`/
  `generateUniqueArtifactId` needed zero modification (pure
  `Record<ArtifactKind, string>` lookups), exactly as the design
  predicted.
- `Axiom/packages/workflow/src/artifact-store.ts` — the largest change:
  - `ARTIFACT_FOLDER` extended (`adr -> 'adr'`, `decision ->
    'decisions'`).
  - New `BaseArtifactMetadataFields` interface (kind-agnostic: `id`,
    `kind`, `title`, `createdAt`, `updatedAt`, `externalRefs`).
    `BaseArtifactMetadata` now extends it, adding only `status:
    WorkflowState` — its effective shape is unchanged, so
    `IncrementMetadata`/`BugMetadata`/`PlanMetadata` compile unchanged
    (verified by full `tsc -b`, zero errors).
  - New `AdrStatus`, `DecisionStatus`, `AdrLinks`, `DecisionLinks`,
    `AdrMetadata`, `DecisionMetadata` types, exactly matching the
    design's schema section verbatim.
  - `ArtifactMetadata` union extended to five members.
  - `parseBaseFields` narrowed to stop parsing/returning `status`
    (moved to a new `parseStatusForKind(obj, kind, filePath)` helper
    that validates `status` against the correct closed vocabulary for
    `kind === 'adr'` (`AdrStatus`, 4 values) or `kind === 'decision'`
    (`DecisionStatus`, 3 values) — for increment/bug/plan it keeps the
    exact pre-existing behavior (any non-empty string is accepted,
    `WorkflowState` has never had a closed-enum shape check here).
    This is the concrete implementation of the design's flagged
    "status-vocabulary-aware" parsing requirement.
  - `parseMetadata` gained two new branches (`kind === 'adr'`,
    `kind === 'decision'`) mirroring the existing `plan` branch's
    structure; the `kind` discriminant check widened to accept all
    five values.
  - `toYamlObject` gained matching `adr`/`decision` serialization
    branches.
  - New factories `makeInitialAdrMetadata`/`makeInitialDecisionMetadata`,
    copied verbatim from the design spec's own code block (both default
    `status: 'proposed'`, empty-string defaults for narrative fields).
  - `listArtifacts` required **zero changes** — confirmed by direct
    read; it was already fully kind-parameterized via `ARTIFACT_FOLDER`.
- `Axiom/packages/workflow/src/index.ts` — barrel re-exports the five
  new types (`AdrMetadata`, `DecisionMetadata`, `AdrStatus`,
  `DecisionStatus`, `AdrLinks`, `DecisionLinks`, plus
  `BaseArtifactMetadataFields`) and the two new factories.
- `Axiom/apps/cli/src/commands/axiom-adr.ts` (new) — `runCreateAdr`
  (generates ID, writes `metadata.yml` + `README.md` scaffold, no
  workflow/state-machine involvement since ADR is not
  state-machine-driven), `runSupersedeAdr` (the one ADR-specific
  operation — see below), plus commander wiring for `create`,
  `supersede <old-id> <new-id>`, reused `link-plan`/`link-increment`/
  `external-ref`/`list` via `artifact-metadata-cli.ts`'s existing
  helpers.
- `Axiom/apps/cli/src/commands/axiom-decision.ts` (new) — same
  structure minus `supersede` (no ADR-style supersession chain for
  Decision, per the design's explicit exclusion).
- `Axiom/apps/cli/src/commands/artifact-metadata-cli.ts` — the shared
  `LinkCommandOptions.ownLinkField`/`reverseLinkField` union type
  (previously `'planId' | 'incrementId' | 'bugId'`, inline) extracted
  to a named `LinkFieldName` type and extended with `'adrId'` so
  `decision link-adr`-shaped usages type-check; `supersedes`/
  `supersededBy` deliberately excluded from this generic link-field
  union since they are handled by the ADR-specific `supersede` command,
  not the generic bidirectional link helper.
- `Axiom/apps/cli/src/commands/index-cmd.ts` — `SCANNED_KINDS` extended
  from `['increment', 'bug', 'plan']` to `['increment', 'bug', 'plan',
  'adr', 'decision']`. No other change to this file, exactly as the
  design specified.
- `Axiom/apps/cli/src/index.ts` — `registerAxiomAdr`/
  `registerAxiomDecision` imported and registered, positioned after
  `registerAxiomPlan` and before `registerAxiomQaE2e` (same relative
  position as the increment/bug/plan block, kept together).

### Three pre-existing narrowing bugs surfaced (not designed, found while implementing)

Widening `ArtifactMetadata` from a 3-member to a 5-member union
surfaced a real type-safety gap in three existing call sites that the
design spec did not anticipate (it is design-only and made no code
changes, so this was invisible until actual implementation):

- `axiom-bug.ts`'s `syncBugMetadata`, `axiom-increment.ts`'s
  `syncIncrementMetadata`, and `axiom-plan.ts`'s plan-approve status
  sync each did `{ ...existing.value, status: toState, updatedAt: now }`
  where `existing.value: ArtifactMetadata` (the whole union) and
  `toState: WorkflowState`. Before this increment this compiled only
  because the union had exactly the three `WorkflowState`-based members
  (increment/bug/plan) — `tsc` could not previously prove this was
  unsafe because there was no member with an incompatible `status`
  type to trigger the error. Adding `AdrMetadata`/`DecisionMetadata`
  (whose `status` is `AdrStatus`/`DecisionStatus`, not `WorkflowState`)
  made the latent problem visible: TypeScript correctly rejected
  spreading `existing.value` (possibly an `AdrMetadata`) and then
  overwriting `status` with a `WorkflowState`-typed value, which no
  longer conforms to `AdrMetadata`'s own `status: AdrStatus` shape.
  **Fix**: added an explicit `existing.value.kind !== 'increment'`
  (or `'bug'`/`'plan'`) narrowing guard before each spread, in each of
  the three call sites — this is strictly a type-safety hardening (the
  runtime behavior was already correct in practice, since each function
  only ever calls `loadArtifactMetadata` with its own fixed `kind`
  argument; the guard makes that invariant explicit and
  compiler-checked instead of implicit). No behavior change; confirmed
  by `npm run typecheck` returning clean and no test regressions.

### Test coverage added

- `packages/workflow/tests/artifact-store.test.ts`: round-trip tests
  for `adr` (folder `adr/<id>`, singular) and `decision` (folder
  `decisions/<id>`, plural); factory default-value tests (status
  `'proposed'`, empty-string narrative fields, empty
  links/`supersedes`); links round-trip (`AdrLinks`/`DecisionLinks`,
  including `supersedes`/`supersededBy`); externalRefs round-trip;
  `listArtifacts` kind-scoping (`adr`/`decision` don't leak into each
  other or into increment/bug/plan listings); and the design's flagged
  **status-vocabulary-rejection** tests: an ADR with
  `status: 'in-progress'` (a `WorkflowState` value) is rejected as
  `invalid-shape`/`cause: 'status'`; a Decision with `status: 'draft'`
  (`WorkflowState`) is rejected; a Decision with `status: 'superseded'`
  (valid `AdrStatus`, but `DecisionStatus` excludes it) is rejected —
  this last case is the sharpest test of the two vocabularies actually
  being non-overlapping, not just "any string works." A companion test
  documents the pre-existing (unchanged) behavior that increment/bug/
  plan's own `status` field is NOT validated against a closed
  `WorkflowState` enum today, so the asymmetry is explicit rather than
  silently assumed. 29/29 tests pass in this file (was 21 before).
- `apps/cli/tests/axiom-adr.test.ts` (new, 9 tests): `runCreateAdr`
  happy path (status, folder, README scaffold, full metadata
  round-trip), optional fields, empty-title rejection;
  `runSupersedeAdr` happy path (both sides updated, old status becomes
  `'superseded'`, new ADR's own status untouched), idempotency (no
  duplicate in `supersedes`), old-id-missing / new-id-missing /
  old-id-equals-new-id rejections; one test confirming the shared
  `runLinkCommand` helper works for `adr link-plan`.
- `apps/cli/tests/axiom-decision.test.ts` (new, 6 tests): `runCreateDecision`
  happy path (status, folder `decisions/` plural, README scaffold, full
  round-trip), optional `rationale` (present + default-empty),
  empty-title rejection, an explicit assertion that no `supersede`
  function is exported from the module (documents the ADR/Decision
  asymmetry as intentional, not an oversight), and one test confirming
  `runLinkCommand` works for `decision link-plan`.
- `apps/cli/tests/index-cmd.test.ts`: existing `rebuild` tests updated
  to expect 5 summaries (was 3) with `adr`/`decision` added; a new
  `runIndexValidate` test confirms an ADR with an invalid
  (`WorkflowState`-vocabulary) status is surfaced as a `kind: 'adr'`
  failure with `errorKind: 'invalid-shape'`, closing the loop between
  the artifact-store-level rejection and the CLI-level `index validate`
  command that a real user/CI would run.

## Validation

Validation commands discovered via `package.json` (root, `Axiom`
monorepo): `npm run typecheck` (`tsc -b`), `npm run build` (`tsc -b`),
`npm test` (`vitest run`, full monorepo).

Baseline (captured before any change in this increment, via direct
`npm run typecheck` + `npm test` run, not assumed from the task brief):
`npm run typecheck` clean; `npm test`: **12 failed test files, 13
failed tests, 1408 passed / 1421 total**.

After implementation:

- `npm run typecheck` (full monorepo): **clean, zero errors.**
- `npm run build` (full monorepo, `tsc -b`): **clean, zero errors.**
- `npm test` (full monorepo, `vitest run`): **12 failed test files, 13
  failed tests, 1436 passed / 1449 total** — the same 12 files and 13
  tests as baseline, confirmed by diffing the `FAIL` lines from both
  runs (pre-existing, environment-dependent failures: TUI
  terminal-output snapshot tests in `packages/tui/tests/driver.test.ts`
  and `apps/cli/tests/start.test.ts`; tests reading real repo config
  files like `axiom.spec/config/agents-catalog.yaml`/
  `skills-catalog.yaml`/model-routing policy from the dev machine in
  `packages/agents`, `packages/skills`, `packages/model-routing`,
  `packages/doctor`, `packages/project-resolution`; a retention-sweep
  timing test in `packages/telemetry`; a toolchain-repair test in
  `packages/toolchain`). None of the 12 pre-existing failures touch
  `artifact-store`, `artifact-id`, or any of the CLI artifact commands
  (`axiom-increment`/`axiom-bug`/`axiom-plan`/`axiom-adr`/
  `axiom-decision`/`index-cmd`/`artifact-metadata-cli`). Total passing
  tests grew from 1408 to 1436 (+28 net, directly observed from two
  full-suite `npm test` runs, before and after implementation).
  Verified per-file `it()` counts for the touched/new test files:
  `artifact-store.test.ts` grew from 21 to 29 tests (+8, all new
  `it()` blocks for adr/decision); two brand-new files,
  `axiom-adr.test.ts` (9 tests) and `axiom-decision.test.ts` (6 tests);
  `index-cmd.test.ts` grew from 6 to 7 tests (+1: the new
  `runIndexValidate` test for the adr status-rejection case; the two
  pre-existing `rebuild` tests were edited in place to expect 5 kinds
  instead of 3, without adding a new `it()`); `artifact-metadata-cli
  .ts`'s `LinkFieldName` change is type-only and added no new tests
  (stayed at 17 tests, confirming no existing behavior broke). These
  four files account for +24 of the +28 total; the remaining +4 is
  not independently reconciled against a specific file in this report
  and should not be treated as fully explained — the authoritative,
  directly observed number is the full-suite delta (1408 → 1436
  passed). Zero pre-existing tests were broken or removed (confirmed
  by the FAIL-line diff below).

  Full new/modified test files run in isolation, all green:
  `packages/workflow/tests/artifact-store.test.ts` (29/29),
  `apps/cli/tests/axiom-adr.test.ts` (9/9),
  `apps/cli/tests/axiom-decision.test.ts` (6/6),
  `apps/cli/tests/index-cmd.test.ts` (7/7),
  `apps/cli/tests/artifact-metadata-cli.test.ts` (17/17, unchanged,
  confirms the `LinkFieldName` type extraction did not break any
  existing link-command behavior).

## Result

Implemented `adr` and `decision` as two new, fully working artifact
kinds inside the existing INC-06 folder-per-artifact `metadata.yml`
mechanism, exactly matching the schema-writer design with no schema
deviation: `ArtifactKind`/`ARTIFACT_ID_PREFIX`/`ARTIFACT_FOLDER`
extended; `BaseArtifactMetadataFields` extracted with
`IncrementMetadata`/`BugMetadata`/`PlanMetadata` compiling unchanged;
`AdrMetadata`/`DecisionMetadata` with their own non-overlapping status
vocabularies, validated by a concrete status-vocabulary-rejection test
suite; two new CLI commands (`axiom-adr`, `axiom-decision`) following
the existing `create`/`link-*`/`list` conventions, plus the one
genuinely ADR-specific `supersede` operation; `axiom index
rebuild`/`validate` extended to cover both new kinds via a one-line
`SCANNED_KINDS` change, with zero new index-building logic, confirming
the design's derived-index recommendation. Along the way, surfaced and
fixed three latent type-safety gaps in `axiom-bug.ts`/
`axiom-increment.ts`/`axiom-plan.ts` that widening `ArtifactMetadata`
exposed (not previously flagged by any earlier increment, including
the schema-writer's own design, since it made no code changes to
trigger the compiler check). Full monorepo `typecheck`/`build` are
clean; `test` shows zero regressions against a freshly-confirmed
baseline, with 28 new tests added and all passing.

## General spec integration

Nothing integrated into `Axiom.Spec/general-spec.md` yet. Per this
increment's own closure status (`pending`) and the schema-writer
increment's explicit instruction ("stable knowledge should be
integrated once cli-implementer and validator-reviewer close and the
schema is confirmed to have landed without change during
implementation"), integration is deferred until `validator-reviewer`
closes this INC-11 chain. Recording here explicitly, matching the
schema-writer's own forward note: once the chain closes, add a short
"ADR/Decision artifact kinds" subsection to `general-spec.md` alongside
the existing "Folder-per-artifact convention" section, stating the two
new kinds, their status vocabularies, the `supersede` operation, and
the derived-ADR-index decision (one paragraph, matching that section's
existing terse style).

## Next step recommendation

Hand off to **validator-reviewer** with this file plus the
schema-writer design spec as inputs. Concrete validator-reviewer task,
per the schema-writer's own hand-off brief: confirm the implementation
matches the design schema exactly (it does — no deviation was made;
this file documents each design-to-code mapping above), re-run
`npm run typecheck`/`npm run build`/`npm test` independently to confirm
the validation claims in this file, and decide whether a new
`@axiom/doctor` check (e.g. `AD-001`-style, mirroring `TC-012`/`TC-013`'s
precedent for skills/technical-context) is warranted for `adr`/
`decision` metadata validity — this increment's own vitest coverage
(including the status-vocabulary-rejection tests) already exercises
shape validation directly against `artifact-store.ts`, so
validator-reviewer's job is to judge whether a *doctor*-level
(project-health-check) surface is additionally needed, not to
re-implement shape validation. After validator-reviewer closes, this
INC-11 chain as a whole (schema-writer -> cli-implementer ->
validator-reviewer) can be considered closed, and the deferred
`general-spec.md` integration (see above) should happen at that point,
not before.
