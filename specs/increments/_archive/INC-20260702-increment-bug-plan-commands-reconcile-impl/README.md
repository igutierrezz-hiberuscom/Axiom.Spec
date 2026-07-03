# Increment: Reconcile increment/bug/plan commands (cli-implementer)

Status: pending
Date: 2026-07-02

## Goal

Execute the **cli-implementer** step of INC-06 of the parent roadmap
(`Axiom.Spec/specs/increments/INC-20260702-axiom-redesign-roadmap/README.md`),
implementing the schema-writer design
(`Axiom.Spec/specs/increments/INC-20260702-increment-bug-plan-commands-reconcile-design/README.md`)
verbatim in the `Axiom` monorepo: `generateArtifactId`, the
`metadata.yml` folder-per-artifact store, wiring
`axiom-increment.ts`/`axiom-bug.ts`/`axiom-plan.ts` to read/write it
alongside (not instead of) `@axiom/workflow`'s existing state machine,
adding the missing `plan create` subcommand, adding
`link-plan`/`link-increment`/`link-bug`, and adding `externalRefs`
add/list support.

## Context

Parent chain (read in full before implementing, per the task brief):

1. migration-engineer audit —
   `Axiom.Spec/specs/increments/INC-20260702-increment-bug-plan-commands-reconcile/README.md`.
2. schema-writer design —
   `Axiom.Spec/specs/increments/INC-20260702-increment-bug-plan-commands-reconcile-design/README.md`.

The design document is treated as an exact contract, not a proposal to
re-litigate. Where the design left an explicit open point (link
directionality), this document states the resolution taken and why.

## Scope

- `@axiom/workflow` (`Axiom/packages/workflow/src/`): new
  `artifact-id.ts` (`generateArtifactId`, `generateUniqueArtifactId`)
  and `artifact-store.ts` (`metadata.yml` schema types, load/save,
  README scaffolding, initial-metadata factories). Both exported from
  the package barrel (`index.ts`). `js-yaml` added as a declared
  dependency (previously used undeclared/hoisted in
  `workflows-loader.ts`; now declared for the new files' explicit use).
- `apps/cli/src/commands/axiom-increment.ts`,
  `axiom-bug.ts`: additive `metadata.yml` create/refresh alongside the
  existing `saveWorkflowState` call; new `link-plan` and `external-ref
  add|list` subcommands.
- `apps/cli/src/commands/axiom-plan.ts`: new `create` subcommand
  (previously entirely absent); `approve` now also refreshes an
  existing plan's `metadata.yml` status; new `link-increment`,
  `link-bug`, and `external-ref add|list` subcommands.
- `apps/cli/src/commands/artifact-metadata-cli.ts` (new, shared):
  `link-<kind>` and `external-ref` commander wiring reused by all
  three CLI files, plus their testable `run*` functions
  (`runLinkCommand`, `runExternalRefAdd`, `runExternalRefList`,
  `parseExternalRefFlag`).
- Tests: `packages/workflow/tests/artifact-id.test.ts`,
  `artifact-store.test.ts` (new); `apps/cli/tests/axiom-plan-create.test.ts`,
  `artifact-metadata-cli.test.ts`, `axiom-increment-metadata.test.ts`,
  `axiom-bug-metadata.test.ts` (new).

## Non-goals

- No change to `@axiom/workflow/src/state-machine.ts`, `state-store.ts`,
  or `hooks.ts` — confirmed unmodified (see Validation below: zero
  diff lines in these three files).
- No change to `axiom-role.ts` — its `checkPlanIsApproved` gate reads
  `workflow-state.json`'s `plan` record directly, unaffected by the
  new `metadata.yml` layer. Confirmed by re-reading the gate and by
  `axiom-role.test.ts` passing unmodified (6/6, same as baseline).
- No migration script for pre-existing `workflow-state.json` data
  (open question 1 from the design spec) — out of scope per the
  design's own stated default (treat as a future increment if needed).
- No `axiom validate changes` / `allowedWriteScope` enforcement logic —
  the field exists on plan `metadata.yml` per addendum 9's shape, but
  no validator consumes it yet (explicitly out of scope per the design).
- No new flag-parsing library for `externalRefs` — kept to a single
  colon-delimited `--ref provider:type:id[:url]` flag, consistent with
  this codebase's existing single-flag conventions.

## Acceptance criteria

- [x] `generateArtifactId(kind, options?)` implemented verbatim per the
      design's signature/format
      (`<PREFIX>-<YYYYMMDD>-<HHMMSS>-<6-char-suffix>`), plus a
      `generateUniqueArtifactId` collision-check-and-retry helper (the
      design left the retry as the caller's responsibility; this
      increment centralizes it as a small reusable helper rather than
      duplicating the check-and-retry loop three times).
- [x] `metadata.yml` read/write implemented for increment, bug, and
      plan, matching the design's shared-base + per-kind schema
      exactly (`id`, `kind`, `title`, `status`, `createdAt`,
      `updatedAt`, `externalRefs`, `links`, plus plan's `targetRepos`/
      `taskType`/`allowedWriteScope`).
- [x] Atomic tmp+rename YAML write pattern reused verbatim
      (`js-yaml` `dump` → `writeFileSync(tmpPath)` →
      `renameSync(tmpPath, filePath)`), same shape as
      `@axiom/topology`'s `saveLocalBindings` and
      `@axiom/user-workspace`'s `saveRegistryV2`.
- [x] `axiom-increment.ts`/`axiom-bug.ts`/`axiom-plan.ts` wired to
      create/read/update `metadata.yml` ALONGSIDE their existing
      `@axiom/workflow` state-machine calls — `saveWorkflowState` calls
      are untouched (same call sites, same arguments), metadata sync
      is additive code inserted after each existing save succeeds.
- [x] `axiom-plan create` added (previously absent entirely).
- [x] `link-plan` added on `axiom-increment`/`axiom-bug`;
      `link-increment`/`link-bug` added on `axiom-plan`.
- [x] `externalRefs` add/list support added
      (`external-ref add --id <id> --ref provider:type:id[:url]`,
      `external-ref list --id <id>`) on all three commands.
- [x] `@axiom/workflow`'s state machine, state-store, and hooks files
      not modified — confirmed by inspection (no edits made to
      `state-machine.ts`, `state-store.ts`, `hooks.ts`,
      `branch-naming.ts`, `types.ts`, or `workflows-loader.ts`).
- [x] `axiom-role.ts` not modified — its single-active-plan gate keeps
      working unmodified per the design's own expectation; confirmed,
      not just assumed (re-read + tests re-run).
- [x] Tests written/updated for: `generateArtifactId`/
      `generateUniqueArtifactId` (format, determinism, uniqueness,
      collision retry, exhaustion); `metadata.yml` round-trip for all
      three kinds (including `links` and `externalRefs`); atomic write
      (`no .tmp` residual); `README.md` non-clobber; `plan create`;
      link commands (bidirectional, own-missing, target-missing
      warning path); `externalRefs` add/list; increment/bug `create`
      metadata sync + subsequent-transition status refresh.
- [x] Real validation executed: `npm run typecheck`, `npm run build`,
      `npm test` (full monorepo + the new/touched test files run in
      isolation first).
- [x] Baseline re-confirmed before changes (see Validation) rather than
      trusted from the task brief's stated range.

## Open questions

None blocking. The link-directionality question the design spec left
open is resolved below (see "Design decisions made where the schema
left something open").

## Assumptions

- `<specPath>` resolves to `<projectRoot>/axiom.spec` (the
  `DEFAULT_SPEC_REL_PATH` constant in `artifact-store.ts`), matching
  every existing `defaultWorkflowsYamlPath()` in the four CLI files.
  No `axiom.yaml` `scopes.spec.path`/`paths.spec.path` override
  plumbing was added in this increment — none of the four CLI files
  read that override today either (each hardcodes `axiom.spec`), so
  wiring the artifact store to honor a *different* resolution
  mechanism than its sibling `workflows.yaml` lookup would be an
  inconsistency, not a fix. If a future increment adds real
  `scopes.spec.path` honoring to `defaultWorkflowsYamlPath()`, this
  artifact store's `specRelPath` parameter is already designed to
  accept an override the same way `workflowsYamlPath` does.
- Existing `workflow-state.json` data pre-dating this increment is not
  auto-migrated (design's own stated default, open question 1) —
  running `refine`/`specify`/etc. on a pre-existing in-flight
  increment/bug that has no `vars.metadataId` is a silent no-op for
  metadata sync (does not error, does not fabricate a new metadata.yml
  instance). This is by design (see `syncIncrementMetadata`/
  `syncBugMetadata`'s early-return comment) to avoid duplicating an
  instance under a new generated ID for state that was never tracked
  per-instance before.

## Implementation notes

### Package placement

Followed the design's Option A recommendation: added `artifact-id.ts`
and `artifact-store.ts` as sibling modules inside `@axiom/workflow`
(`Axiom/packages/workflow/src/`), both re-exported from the package
barrel. `js-yaml` (`^4.1.0`) and `@types/js-yaml` (`^4.0.9`) were added
as declared dependencies in `packages/workflow/package.json`, matching
the exact version pins already used by `@axiom/user-workspace` and the
CLI app. (Side note, not part of this increment's scope: the package
already had an *undeclared* runtime dependency on `js-yaml` via
`workflows-loader.ts`, resolved only through npm workspace hoisting —
this increment does not touch that file, but declaring `js-yaml`
properly for the new files also incidentally documents the pre-existing
dependency correctly going forward.)

### `generateArtifactId` / `generateUniqueArtifactId`

Implemented `generateArtifactId(kind, options?)` verbatim per the
design's code block (same format, same UTC field padding, same
injectable `now`/`randomSuffix` for deterministic tests). Added one
small addition not explicitly in the design's code sample but implied
by its prose ("Caller... MUST verify the target folder does not
already exist... and retry generation on the rare collision"):
`generateUniqueArtifactId(kind, exists, options?)`, which centralizes
that check-and-retry loop once instead of duplicating it in
`axiom-increment.ts`, `axiom-bug.ts`, and `axiom-plan.ts` separately.
It throws after `maxAttempts` (default 10) exhausted collisions — an
extremely unlikely path given the 6-character base36 suffix space,
kept simple (a thrown `Error`, not a new `Result` error kind) since
the design's own pure function contract already treats this as a
near-impossible caller-side condition, not a normal control-flow
outcome.

### `metadata.yml` schema and store

Implemented `artifact-store.ts` with the exact shared-base +
per-kind-extension shape from the design's three worked examples:
`BaseArtifactMetadata` (`id`, `kind`, `title`, `status`, `createdAt`,
`updatedAt`, `externalRefs`), `IncrementMetadata`/`BugMetadata` (base +
`links: { planId }`), `PlanMetadata` (base + `links: { incrementId,
bugId }` + `targetRepos` + `taskType` + `allowedWriteScope`). `status`
reuses `@axiom/workflow`'s existing `WorkflowState` type verbatim, per
the design's explicit instruction not to invent a second state
vocabulary.

`loadArtifactMetadata`/`saveArtifactMetadata` follow the same
`Result<T, Error>` discriminated-union style as `state-store.ts` and
`workflows-loader.ts` (`io-error`, `not-found` — unused currently but
kept for shape completeness matching sibling error unions,
`invalid-yaml`, `invalid-shape`). `saveArtifactMetadata` is the atomic
tmp+rename write, verified in tests to leave no `.tmp` residual.
`ensureArtifactReadme` never overwrites an existing `README.md` (tested
explicitly), so human-authored spec prose surviving a later `create`
re-run (idempotency scenarios) is protected.

### Wiring into `axiom-increment.ts` / `axiom-bug.ts`

Both files follow an identical additive pattern: at step "6.5" (right
after the existing `newVars` construction, before `newRecord` is
built), a `metadataId` is generated via `generateUniqueArtifactId` and
cached into `vars['metadataId']` — but ONLY on `create`/init. This
means the generated ID gets persisted into the SAME
`saveWorkflowState` call that already existed (no second
`workflow-state.json` write), avoiding a two-write race between
`workflow-state.json` and `metadata.yml` disagreeing about the
`metadataId` if the process were killed between writes.

After the existing `saveWorkflowState` call succeeds (step 8, call
site and arguments unchanged), a new step 8.5
(`syncIncrementMetadata`/`syncBugMetadata`) runs: on `create`/init it
builds the initial `metadata.yml` (`title` = the caller's `--slug`,
falling back to `--id` then a hardcoded default) and scaffolds
`README.md`; on any other subcommand, it loads the existing
`metadata.yml` by the cached `vars.metadataId` and refreshes only
`status`/`updatedAt`, leaving `id`/`title`/`createdAt`/`links`/
`externalRefs` untouched. If `vars.metadataId` is absent (pre-existing
instance from before this increment), the sync is a silent no-op —
per the Assumptions section above.

### `axiom-plan create` (the widest gap)

Unlike increment/bug, `axiom-plan.ts`'s existing `approve` command
uses the caller-supplied `--id <planId>` directly as the plan's
identity (`vars.planId`) — there is no separate free-text `--id`/
`--slug` pair to preserve. Given that, `runPlanCreate` generates the
plan's `metadata.yml` `id` via `generateUniqueArtifactId('plan', ...)`
and prints it to the caller; the caller then passes that generated ID
as `--id <planId>` to `axiom-plan approve`. `create` does **not** call
`saveWorkflowState` or `applyTransition` at all — it is a pure
metadata-layer operation (matches the design's explicit statement that
`plan`'s only state-machine transition is `plan-approve`; `create`
naturally has nothing to transition yet). `runPlanApprove` was extended
with a best-effort metadata refresh: if a `metadata.yml` already exists
for the approved `planId` (created via `create`), its `status` is
updated to the post-transition state; if no `metadata.yml` exists (a
plan approved without ever calling `create` — the pre-INC-06 workflow,
still fully supported), nothing is fabricated, consistent with not
inventing `title`/`targetRepos`/`taskType` out of nothing.

### `link-plan` / `link-increment` / `link-bug`

Implemented as shared, parameterized commander wiring in the new
`artifact-metadata-cli.ts` (`addLinkPlanSubcommand`), registered three
times (once per direction) across the three CLI files, rather than
one-off duplicated command bodies. `runLinkCommand` is exported
separately for direct unit testing without spawning the CLI/hitting
`process.exit`.

### `externalRefs` add/list

Chose a single, minimal colon-delimited flag —
`--ref provider:type:id[:url]` (`url` optional, 3- or 4-segment form) —
over a multi-flag or JSON-blob approach, consistent with this
codebase's existing single-flag conventions (`--id`, `--slug`, no
`--json`-body-as-arg pattern anywhere in the four command files).
`parseExternalRefFlag` is a small pure parser, unit-tested directly for
valid/invalid inputs. `external-ref add`/`external-ref list` are
sub-groups under each of the three top-level commands
(`axiom increment external-ref add --id ... --ref ...`), mirroring the
nested-subcommand style already used by `axiom-plan approve`.

### Design decisions made where the schema left something open

**Link directionality.** The design spec's worked YAML examples show
`links.planId` on increment/bug and `links.incrementId`/`links.bugId`
on plan, which implies bidirectional intent, but the design's own
scope statement explicitly deferred the *commands* (not just the
field shapes) to cli-implementer: *"the CLI commands themselves — that
is cli-implementer's job."* This increment resolves the directionality
as **bidirectional, best-effort**: calling `increment link-plan`
updates both the increment's `links.planId` AND the target plan's
`links.incrementId` in the same command invocation (and symmetrically
for `bug link-plan`, `plan link-increment`, `plan link-bug`). If the
target artifact's `metadata.yml` does not exist yet (e.g. linking to a
plan ID before that plan folder was created), the OWN side still
updates successfully and the command prints a non-fatal warning rather
than failing — an increment/bug can legitimately reference a plan ID
that is about to be created, or vice versa, without a strict creation
order being enforced. This was chosen over a strictly one-directional
implementation because: (1) the design's own examples show both sides
populated in a consistent worked scenario (increment's `links.planId`
= `PLAN-20260702-190112-9b7a2e`, and that same plan's
`links.incrementId` = `INC-20260702-183012-a7f3c9`), suggesting the
intended steady-state is symmetric; (2) a one-directional
implementation would leave the reverse field permanently `null` unless
a second, separate command was also run, which is a worse ergonomic
default than an auto-synced best-effort update; (3) the "best-effort,
non-fatal on missing target" behavior avoids introducing an ordering
constraint (create increment before plan, or vice versa) that neither
source document nor the design spec asked for.

**`metadata.yml` `title` source for increment/bug `create`.** The
design's schema requires a `title` field but `axiom-increment.ts`/
`axiom-bug.ts`'s `create` only ever accepted `--id`/`--slug` (no
`--title` flag existed, and the task brief did not ask to add one to
these two commands — only `plan create` was called out as needing a
new command entirely, which does have `--title`). Resolution: `title`
for increment/bug metadata defaults to `--slug` (already the closest
existing human-readable label; used for branch naming today), falling
back to `--id` if `--slug` is somehow absent (defensive; today's
`addIncrementSubcommand`/`addBugSubcommand` already require `--slug`
on `create` so this fallback should not trigger in practice), and a
final hardcoded `'increment'`/`'bug'` literal if both are absent
(construction-time record, not reachable via the CLI's required-flag
validation, kept only so the pure `syncIncrementMetadata`/
`syncBugMetadata` functions never throw on missing input). This was
not a source-doc requirement to add a new `--title` flag to
increment/bug `create` — the task brief scoped the new `--title` flag
specifically to `axiom-plan create` (the one CLI-visible gap named in
the brief), and adding an extra required/optional flag to two already-
shipped commands' `create` was judged out of this increment's minimal-
disruption scope; flagged here explicitly as a deliberate choice, not
an oversight, for the validator-reviewer to confirm or challenge.

## Validation

Baseline re-confirmed before any change (git working tree was already
at HEAD with the two prior specs' commits applied, no `git stash`
bisection was needed since no code existed yet for this increment):

```
npm test (before)  → Test Files  12 failed | 114 passed (126)
                      Tests       13 failed | 1254 passed (1267)
npm run typecheck (before) → clean (tsc -b, no errors)
npm run build (before)     → clean (tsc -b, no errors)
```

After implementation:

```
npm run typecheck → clean (tsc -b, no errors)
npm run build     → clean (tsc -b, no errors)
npm test          → Test Files  12 failed | 120 passed (132)
                     Tests       13 failed | 1298 passed (1311)
```

The 12 failed files / 13 failed tests are byte-for-byte the same
failing test names before and after (confirmed by listing failed test
titles from both runs):
`packages/agents/tests/catalog.test.ts`,
`apps/cli/tests/start.test.ts`,
`packages/doctor/tests/checks.test.ts`,
`packages/model-routing/tests/{assignments,loader,opencode-projection,resolver}.test.ts`,
`packages/skills/tests/catalog.test.ts`,
`packages/project-resolution/tests/resolver.test.ts`,
`packages/telemetry/tests/audit-trail-sink.test.ts`,
`packages/toolchain/tests/repair-add-gitignore.test.ts`,
`packages/tui/tests/driver.test.ts` (2 tests). None of these touch
`axiom-increment.ts`, `axiom-bug.ts`, `axiom-plan.ts`, `axiom-role.ts`,
or any file under `packages/workflow/`; all are pre-existing failures
unrelated to this increment (mostly tests that expect a real
`axiom.spec/` tree checked into the `Axiom` repo root, which does not
exist in this environment — a pre-existing environment gap, not
introduced here).

New tests added: 44 (21 in `packages/workflow/tests/` —
`artifact-id.test.ts` 9, `artifact-store.test.ts` 12 — matches;
21 in `apps/cli/tests/` — `axiom-plan-create.test.ts` 5,
`artifact-metadata-cli.test.ts` 13, `axiom-increment-metadata.test.ts` 3
— totals 21; `axiom-bug-metadata.test.ts` 2 additional). 1254 → 1298 is
exactly +44, confirming no existing test was silently dropped or
skipped.

Existing test files for the four touched commands
(`axiom-increment.test.ts` 9/9, `axiom-bug.test.ts` 5/5,
`axiom-plan.test.ts` 2/2, `axiom-role.test.ts` 6/6) were re-run and
pass unmodified — none of their assertions needed updating, confirming
the wiring is genuinely additive (no existing behavior changed).

## Result

Implemented the cli-implementer step of INC-06 per the schema-writer
design, with zero modification to `@axiom/workflow`'s state machine,
state-store, or hooks, and zero modification to `axiom-role.ts`.

Delivered:

1. `generateArtifactId`/`generateUniqueArtifactId`
   (`Axiom/packages/workflow/src/artifact-id.ts`).
2. `metadata.yml` folder-per-artifact store: schema types, atomic
   load/save, README scaffolding, initial-metadata factories
   (`Axiom/packages/workflow/src/artifact-store.ts`), exported from
   `@axiom/workflow`'s barrel.
3. `axiom-increment.ts`/`axiom-bug.ts` additively wired to generate a
   `metadataId` on `create` and sync `metadata.yml` on every
   transition, without touching their existing `saveWorkflowState`
   call sites.
4. `axiom-plan create` (new), `axiom-plan approve` extended to
   best-effort refresh an existing plan's `metadata.yml`.
5. `link-plan` (increment, bug), `link-increment`/`link-bug` (plan) —
   bidirectional, best-effort, shared implementation in the new
   `artifact-metadata-cli.ts`.
6. `external-ref add`/`external-ref list` on all three commands, via
   a minimal `provider:type:id[:url]` flag.
7. 44 new tests across `@axiom/workflow` and the CLI app, all passing;
   zero regressions in the existing suite (same 12 failed files / 13
   failed tests as the pre-change baseline, all pre-existing and
   unrelated).

Two design-level decisions were made where the schema-writer spec left
an explicit gap (link directionality; increment/bug metadata `title`
source), both documented above with rationale for the
validator-reviewer to confirm.

## General spec integration

No integration into `Axiom.Spec/general-spec.md` was performed — that
file does not exist in this repo yet (confirmed by both prior specs in
this chain; the closest equivalents are
`Axiom.Spec/specs/00_Resumen_Ejecutivo.md` through `08_Glosario.md`,
not modified here). Consistent with the migration-engineer and
schema-writer specs' own stated reasoning, this implementation has not
yet passed a validator-reviewer pass; consolidating it into a stable,
reusable general spec before that review would risk documenting
implementation details (e.g. the exact `--ref` flag shape, the
`metadataId` bridge field) that a review might still ask to adjust.
Deferring general-spec integration to (or after) the validator-reviewer
step, consistent with this chain's own precedent.

## Next step recommendation

**validator-reviewer**, per the migration-engineer audit's originally
proposed chain (migration-engineer → schema-writer → cli-implementer →
validator-reviewer), now that cli-implementer is complete.

Exact inputs the validator-reviewer needs from this document:

1. The full `npm run typecheck` / `npm run build` / `npm test` output
   summarized above (clean typecheck/build; test suite at 1298 passed,
   13 failed — same 13 pre-existing failures as baseline, not
   introduced by this increment).
2. The list of touched/created files (Implementation notes above) to
   scope its own re-read.
3. The two explicit design decisions made in this step (link
   directionality; increment/bug `title` source) — the
   validator-reviewer should confirm these against the source docs
   (section 6.1's command list, addendum 11's `externalRefs`/ID
   stability intent) and either accept them or flag a correction.
4. Confirmation still needed (not yet independently re-verified by a
   second pass) that `axiom-role.ts`'s safety gate genuinely still
   reads the correct `workflow-state.json` shape end-to-end in a
   scenario where a plan was created via the NEW `plan create` +
   `approve` path (this increment's own tests cover `approve` after
   `create` returning `plan-approved` state correctly, and
   `axiom-role.test.ts` passes unmodified, but a validator-reviewer
   cross-check combining both new and old code paths together was not
   separately executed here beyond the full suite run).
