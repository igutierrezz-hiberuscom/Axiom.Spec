# Increment: ADR/decision handling — validator-reviewer (INC-11)

Status: closed
Date: 2026-07-02

## Goal

Independently verify the `adr`/`decision` schema and implementation
produced by the schema-writer (`INC-20260702-adr-decision-reconcile-design`)
and cli-implementer (`INC-20260702-adr-decision-reconcile-impl`) increments,
fix any real gaps found, re-run validation from scratch, and close INC-11
("Reconcile ADR/decision handling") as a whole if all acceptance criteria
are met. This increment is `validator-reviewer`, the final role in the
roadmap's INC-11 sequence (schema-writer -> cli-implementer ->
validator-reviewer).

## Context

Parent roadmap: `Axiom.Spec/specs/increments/
INC-20260702-axiom-redesign-roadmap/README.md`, INC-11.

Direct inputs (read in full before this increment started):
`INC-20260702-adr-decision-reconcile-design/README.md` (schema-writer),
`INC-20260702-adr-decision-reconcile-impl/README.md` (cli-implementer).

Per the task brief, every claim from both prior specs was independently
re-derived from the actual source files rather than trusted at face value.
Implementation code for this chain lives in the sibling `Axiom` monorepo
(`Axiom/packages/workflow/src/*`, `Axiom/apps/cli/src/commands/*`,
`Axiom/packages/doctor/src/checks.ts`) — both prior increments in this
chain implemented there directly (not in `Axiom.SDD`, which contains no
product code), and this increment follows that same established,
already-precedented location rather than relocating code as an unrelated
side effect of a validation pass.

## Scope

- Independent verification of the `BaseArtifactMetadataFields` extraction,
  `parseStatusForKind`'s vocabulary enforcement, and the three
  `existing.value.kind` narrowing-guard fixes in `axiom-bug.ts`/
  `axiom-increment.ts`/`axiom-plan.ts`.
- Independent trace of `axiom adr supersede`'s happy path and the
  already-superseded-by-a-different-ADR edge case, with a fix if the
  current behavior is unacceptable.
- Confirmation that `axiom-decision.ts` deliberately omits `supersede`.
- Confirmation that `index-cmd.ts`'s `SCANNED_KINDS` covers `adr`/
  `decision`.
- Decision on whether a new `@axiom/doctor` check is warranted for
  `adr`/`decision` metadata validity, with a fix if a gap is found in the
  existing generic mechanism instead.
- A fresh, independent `npm run typecheck` / `npm run build` / `npm test`
  run (full monorepo).
- Closure of INC-11 as a whole if criteria are met.

## Non-goals

- No new artifact kinds, commands, or schema changes beyond what the two
  prior increments in this chain already specified.
- No curated/versioned ADR index (already decided against by
  schema-writer, reaffirmed here, not re-litigated).
- No relocation of implementation code between `Axiom` and `Axiom.SDD` —
  out of scope for a validation pass; noted as a pre-existing convention
  in this increment chain, not fixed here.

## Acceptance criteria

- [x] `BaseArtifactMetadataFields` extraction independently confirmed
      correct by reading `artifact-store.ts` directly (not by trusting
      cli-implementer's report).
- [x] `parseStatusForKind`'s per-kind vocabulary enforcement independently
      traced and confirmed to reject cross-kind status values in both
      directions.
- [x] The three `existing.value.kind` narrowing-guard fixes independently
      confirmed correct, including tracing a concrete scenario under which
      the bug they fix could actually occur (not just "the types compile").
- [x] `axiom adr supersede`'s edge case (re-superseding an already
      superseded ADR by a different new ADR) traced, decided, and fixed if
      needed.
- [x] `axiom-decision.ts`'s lack of `supersede` confirmed as a documented,
      deliberate design choice, not an oversight.
- [x] `index-cmd.ts`'s `SCANNED_KINDS` confirmed to include `adr`/
      `decision`.
- [x] Doctor-check decision made (new check vs. extend existing generic
      mechanism) based on reading `checks.ts` directly, with the finding
      acted upon.
- [x] `npm run typecheck`, `npm run build`, `npm test` run independently,
      fresh, full monorepo; results compared against the stated baseline
      (12 failed files / 13 failed tests / 1408 passed) and cli-implementer's
      claimed post-implementation numbers (1436 passed).
- [x] INC-11 closed as a whole if all of the above hold, with deferred
      items named explicitly.
- [x] `general-spec.md` updated with a consolidated "ADR/Decision artifact
      kinds" section.

## Open questions

None blocking. The two open questions left by schema-writer for
cli-implementer (creation-time-required-fields, atomic-vs-two-call link
commands) were resolved by cli-implementer and independently confirmed
here as reasonable, consistent choices — not re-opened.

## Assumptions

- The `Axiom`-repo-as-implementation-location convention established by
  the two prior increments in this chain is intentional/accepted, not a
  process violation to correct as part of this validation pass.
- The `supersede` reassignment fix below (warn-and-proceed, not
  error-and-block) is the right default because re-pointing a
  supersession is a legitimate corrective action an operator might
  deliberately want (e.g. fixing a wrong prior `supersede` call), and
  nothing in either source document specifies ADR supersession should be
  immutable once set. If a stricter (block-by-default,
  require-a-force-flag) policy is wanted later, it is a small, additive
  change to `runSupersedeAdr`, not a redesign.

## Implementation notes

### 1. `BaseArtifactMetadataFields` extraction — independently verified correct

Read `Axiom/packages/workflow/src/artifact-store.ts` directly (not from
either prior spec's summary). Confirmed:

- `BaseArtifactMetadataFields` holds exactly `id`, `kind`, `title`,
  `createdAt`, `updatedAt`, `externalRefs` — no `status`.
- `BaseArtifactMetadata extends BaseArtifactMetadataFields` and adds only
  `status: WorkflowState`. `IncrementMetadata`/`BugMetadata`/`PlanMetadata`
  still extend `BaseArtifactMetadata` unchanged — their effective shape is
  identical to pre-INC-11 (verified independently via `npm run typecheck`
  returning clean, confirming no downstream consumer broke).
- `AdrMetadata`/`DecisionMetadata` extend `BaseArtifactMetadataFields`
  directly (bypassing `BaseArtifactMetadata`) and declare their own
  `status: AdrStatus` / `status: DecisionStatus`. This is exactly the
  schema-writer's design, landed without deviation.

### 2. `parseStatusForKind` — independently traced, not just trusted by test name

Read the function body directly (`artifact-store.ts` lines ~295-320).
Confirmed the actual narrowing logic: it checks `obj.status` is a string,
then branches on `kind === 'adr'` against `ADR_STATUSES` (4 values) or
`kind === 'decision'` against `DECISION_STATUSES` (3 values); for
increment/bug/plan it returns the raw string unchecked (pre-existing
behavior, unchanged — `WorkflowState` was never closed-enum-validated
here). Traced both cross-kind rejection directions concretely:

- An ADR `metadata.yml` with `status: in-progress` (a `WorkflowState`
  value, not in `ADR_STATUSES`) is rejected by the `kind === 'adr'`
  branch — confirmed via a new doctor-level test added in this increment
  (see Validation section) hitting exactly this case end-to-end through
  `IX-001`.
- A Decision `metadata.yml` with `status: superseded` (a valid `AdrStatus`
  value, but not in `DECISION_STATUSES`) is rejected by the
  `kind === 'decision'` branch — already covered by cli-implementer's own
  `artifact-store.test.ts` suite, re-run and confirmed green in this
  increment (not re-implemented, per the impl spec's own hand-off note
  that shape validation was already exercised there).

Both directions are real, non-overlapping vocabulary enforcement, not
"any string works" — confirmed by direct trace, not by trusting the test
names alone.

### 3. The three narrowing-guard fixes — independently verified, with the corruption scenario made concrete

Read all three call sites directly:
`Axiom/apps/cli/src/commands/axiom-increment.ts:516-518`,
`axiom-bug.ts:419-421`, `axiom-plan.ts:353-360`. Each does:

```ts
const existing = loadArtifactMetadata(projectRoot, 'increment', metadataId);
if (!existing.ok || existing.value === undefined || existing.value.kind !== 'increment') return;
saveArtifactMetadata(projectRoot, { ...existing.value, status: toState, updatedAt: now });
```

(bug/plan mirror this with their own literal kind — plan's guard is
written as `existingMetadata.value.kind === 'plan'` (positive form) rather
than `!== 'plan'` (negated form used by increment/bug), but both are
logically equivalent single-branch narrowing before the spread — not a
functional inconsistency.)

**Concrete scenario traced (per the task brief's explicit instruction not
to just trust "the types compile"):** `loadArtifactMetadata(projectRoot,
kind, id)` resolves the *file path* using the caller-supplied `kind`
parameter (via `resolveArtifactDir` -> `ARTIFACT_FOLDER[kind]`), but
`parseMetadata`/`parseBaseFields` derive the *returned* `ArtifactMetadata`
value's `kind` field from the file's own YAML content (`obj.kind`, read
verbatim at `artifact-store.ts:282`), not from the caller's `kind`
argument. This means `loadArtifactMetadata(root, 'increment', id)` reads
from `increments/<id>/metadata.yml` (path resolved by the caller's
`'increment'` argument) but would return a value with `kind: 'adr'` if
that specific file's YAML content had `kind: adr` written into it (e.g. a
hand-edited or corrupted file placed in the wrong folder, or a future bug
elsewhere that writes the wrong `kind` field into the wrong folder).
**Before the fix**, `syncIncrementMetadata` would spread that
`AdrMetadata`-shaped value and force `status: toState` (a `WorkflowState`
value) onto it, producing a `metadata.yml` that is a `kind: adr` object
carrying a `WorkflowState` value in its `status` field — invalid per
`AdrMetadata`'s own shape, and exactly the kind of `IX-001`-catchable
corruption this whole increment's status-vocabulary work is designed to
prevent. **After the fix**, the `kind !== 'increment'` guard makes this
function return early (no write) instead of producing that corrupted
file. This is not hypothetical type-theater: it is a real (if
narrow/rare) file-content-vs-folder-location mismatch the guard
concretely closes. Confirmed by direct code trace, not by assumption.

### 4. `axiom adr supersede` — traced happy path and edge case; fixed the edge case

Read `Axiom/apps/cli/src/commands/axiom-adr.ts`'s `runSupersedeAdr` in
full. Happy path (both ADRs exist, `old-id` not previously superseded):
confirmed correct — sets `old-id`'s `status: 'superseded'` and
`links.supersededBy: new-id`, appends `old-id` to `new-id`'s
`links.supersedes` (idempotently, already covered by cli-implementer's
own test), leaves `new-id`'s own `status` untouched. Confirmed by direct
trace and by running the existing test suite.

**Edge case found (not covered by cli-implementer's own test suite,
not called out in their "Assumptions" section, which only discussed
same-pair idempotency):** before this increment's fix, calling
`supersede <old-id> <new-id-2>` on an ADR that was already
`supersededBy: <new-id-1>` (a *different* ADR) silently overwrote
`links.supersededBy` from `new-id-1` to `new-id-2` and set `status:
'superseded'` again (no-op on status, already superseded) — with **no
warning, no error, and no trace of the prior relationship having
existed**. `new-id-1`'s own `links.supersedes` array is not touched by
this reassignment (it still lists `old-id`), which would leave the
ADR graph in a state where two ADRs (`new-id-1` and `new-id-2`) both
claim `old-id` in `supersedes`, while `old-id` itself only points back to
one of them (`new-id-2`) — an inconsistency an operator would have no way
to discover without manually diffing both files.

**Decision: this is a real gap worth fixing, not acceptable as-is.**
Silently overwriting a prior architectural relationship with zero signal
to the caller is exactly the kind of "corruption you don't find out about
until much later" this whole increment chain exists to prevent for
adr/decision metadata. **Fix applied**: `runSupersedeAdr` now detects
this case (`oldAdr.links.supersededBy` is non-null and different from the
requested `new-id`) and still performs the reassignment (treated as a
legitimate, deliberate correction — not blocked, no new required flag,
consistent with this codebase's general preference for informative
non-blocking behavior over hard failures for still-valid operations), but
returns an explicit warning message naming the prior superseder and
noting that the prior superseder's own `links.supersedes` was not
automatically cleaned up (semantic graph cleanup across an unbounded
number of possibly-stale entries was explicitly out of scope for
cli-implementer's increment per its own non-goals, and remains out of
scope here — the warning surfaces the fact so a human can act on it,
which is proportionate to a schema-only/CLI-ergonomics increment).

Added two new tests in `Axiom/apps/cli/tests/axiom-adr.test.ts`:
reassignment emits the warning and correctly updates both sides;
re-invoking with the *same* `new-id` (already covered by
cli-implementer's idempotency test) does NOT emit the warning (confirms
the warning is precise to the "different superseder" case, not a blanket
notice on every `supersede` call).

### 5. `axiom-decision.ts`'s lack of `supersede` — confirmed sound and documented

Read `Axiom/apps/cli/src/commands/axiom-decision.ts` in full: no
`supersede` function or command is exported or registered. The file's own
header comment states this is deliberate ("Consequently this file has no
`supersede` command; that operation is ADR-specific only"), consistent
with the schema-writer design's explicit choice to exclude `superseded`
from `DecisionStatus` and `supersedes`/`supersededBy` from
`DecisionLinks`. cli-implementer's own test file
(`axiom-decision.test.ts`) includes an explicit assertion that no
`supersede` function is exported — confirmed by reading that test
directly. This is a sound, intentional, well-documented asymmetry, not an
oversight.

### 6. `index-cmd.ts`'s `SCANNED_KINDS` — independently confirmed

Read `Axiom/apps/cli/src/commands/index-cmd.ts` line 43 directly:
`SCANNED_KINDS: ReadonlyArray<ArtifactKind> = ['increment', 'bug', 'plan',
'adr', 'decision']`, used identically by both `runIndexRebuild` and
`runIndexValidate`'s scan loops (lines 102 and 142). Confirmed `axiom
index rebuild`/`validate` genuinely enumerate ADR/decision folders — not
assumed from the impl spec's own claim.

### 7. Doctor-check decision: extend existing generic check, no new check needed — but a real gap was found and fixed

Read `Axiom/packages/doctor/src/checks.ts` directly, specifically the
`TC-012`/`TC-013` checks (skills-role-index-validity /
technical-context-index-validity) to establish the actual precedent
before deciding. Finding: `TC-012`/`TC-013` are **not** generic
artifact-kind-covering checks — they are specific to the
`SkillsRoleIndex`/`TechnicalContextIndex` curated-index mechanisms
(reusing `validateSkillsRoleIndex`/`validateTechnicalContextIndex` from
their respective packages), which is an entirely different kind of
artifact (curated index files, not per-instance `metadata.yml`). They are
not the right precedent to mirror for `adr`/`decision`, which per the
schema-writer's own decision explicitly do NOT get a curated index.

The actually-relevant existing check is **`IX-001`**
(`runArtifactIndexChecks`, categoría `artifact-index`), which already
generically reuses `listArtifacts` (the same kind-parameterized primitive
`index-cmd.ts` uses) to validate that every `metadata.yml` under the spec
scope parses correctly, regardless of kind. Reading it directly revealed
a real, concrete gap: its own `ARTIFACT_INDEX_KINDS` constant (a
*separate* hardcoded array from `index-cmd.ts`'s `SCANNED_KINDS`) was
still `['increment', 'bug', 'plan']` — **cli-implementer updated
`SCANNED_KINDS` in `index-cmd.ts` but missed this second, independent
hardcoded array in `checks.ts`**, so `axiom index rebuild`/`validate`
covered adr/decision but `@axiom/doctor`'s `IX-001` health check did not.

**Decision and fix**: no new `@axiom/doctor` check is warranted — `IX-001`
is already the correct, generic, kind-agnostic mechanism and duplicating
it with an `AD-001`-style check would be redundant, unrequested
complexity. Instead, extended `ARTIFACT_INDEX_KINDS` to `['increment',
'bug', 'plan', 'adr', 'decision']` (one-line change,
`Axiom/packages/doctor/src/checks.ts`), matching `SCANNED_KINDS`'s own
extension, plus updated the two `desc`/pass-message strings that
literally said "increments/bugs/plans" to also mention adr/decisions.
Added two new tests to `Axiom/packages/doctor/tests/checks.test.ts`:
`IX-001` passes when valid `adr`/`decision` entries exist alongside
increment/bug/plan, and `IX-001` fails when an ADR has a
`WorkflowState`-vocabulary status value (`in-progress`) instead of a
valid `AdrStatus` — this closes the loop end-to-end from the
artifact-store-level status-vocabulary rejection (item 2 above) through
to the doctor-level health check a real CI run would execute.

### Files changed (this increment)

- `Axiom/packages/doctor/src/checks.ts` — `ARTIFACT_INDEX_KINDS` extended
  to include `'adr'`/`'decision'`; two description/pass-message strings
  updated to mention adr/decisions; explanatory comment added.
- `Axiom/packages/doctor/tests/checks.test.ts` — two new tests for
  `IX-001` covering adr/decision (pass case, status-vocabulary-rejection
  case); import list extended with `makeInitialAdrMetadata`/
  `makeInitialDecisionMetadata`.
- `Axiom/apps/cli/src/commands/axiom-adr.ts` — `runSupersedeAdr` now
  detects reassignment of an already-`supersededBy`-set ADR to a
  different superseder and returns an explicit warning message (exit code
  still `0` — this remains a successful, deliberate operation) instead of
  silently overwriting with no signal.
- `Axiom/apps/cli/tests/axiom-adr.test.ts` — two new tests: reassignment
  warning case, and confirmation the warning does NOT fire on the
  already-covered same-pair idempotent re-invocation case.

No changes were made to `artifact-id.ts`, `artifact-store.ts`'s schema
(items 1-2 confirmed correct as-is), `index-cmd.ts` (item 6 confirmed
correct as-is), or `axiom-decision.ts` (item 5 confirmed correct as-is).

## Validation

Validation commands discovered via `package.json` (root, `Axiom`
monorepo, same as both prior increments in this chain): `npm run
typecheck` (`tsc -b`), `npm run build` (`tsc -b`), `npm test` (`vitest
run`, full monorepo).

Results (fresh, independent runs performed in this increment, not copied
from either prior spec):

- `npm run typecheck` (full monorepo): **clean, zero errors.**
- `npm run build` (full monorepo, `tsc -b`): **clean, zero errors.**
- `npm test` (full monorepo, `vitest run`): **12 failed test files, 13
  failed tests, 1440 passed / 1453 total.**

Compared against the chain's stated baseline (12 failed files / 13 failed
tests / 1408 passed, captured by cli-implementer before any INC-11 change)
and cli-implementer's own post-implementation numbers (1436 passed): this
increment's fresh run shows **+4 passed over cli-implementer's number**
(1436 -> 1440), which reconciles exactly to the 4 new tests added in this
increment (2 in `checks.test.ts` for `IX-001`/adr-decision coverage, 2 in
`axiom-adr.test.ts` for the supersede-reassignment edge case). The same
12 failing test files/13 failing tests as both the original baseline and
cli-implementer's post-implementation run were confirmed present and
unchanged by diffing the `FAIL` lines from this run against both prior
reports — all pre-existing, environment-dependent (TUI terminal-snapshot
tests, tests reading real repo config files from the dev machine,
retention-sweep timing, toolchain repair, a `resolveProject(cwd)`
ambiguous-vs-resolved classification test). None touch `artifact-store`,
`artifact-id`, or any of the CLI artifact commands. Zero regressions.

Full new/modified test files also run in isolation to confirm precisely
which tests are new/green (not relying on the aggregate full-suite number
alone): `packages/doctor/tests/checks.test.ts` (37 tests, 36 passed / 1
pre-existing unrelated failure — a `resolveProject(process.cwd())`
`'ambiguous'`-vs-`'resolved'`/`'not-found'` cwd-classification assertion,
confirmed identical to the pre-existing failure signature from the
full-suite run), `apps/cli/tests/axiom-adr.test.ts` (11/11, was 9 before
this increment), `apps/cli/tests/axiom-decision.test.ts` (6/6, unchanged),
`packages/workflow/tests/artifact-store.test.ts` (29/29, unchanged),
`apps/cli/tests/index-cmd.test.ts` (7/7, unchanged),
`apps/cli/tests/artifact-metadata-cli.test.ts` (17/17, unchanged, confirms
no regression from cli-implementer's `LinkFieldName` extraction).

## Result

Independently re-verified every claim made by the schema-writer and
cli-implementer increments in this chain by reading the actual source
code and tests directly, not by trusting either prior spec's prose.
Confirmed as genuinely correct, with no deviation: the
`BaseArtifactMetadataFields` extraction; `parseStatusForKind`'s
non-overlapping per-kind vocabulary enforcement (both rejection
directions traced concretely); the three narrowing-guard fixes in
`axiom-bug.ts`/`axiom-increment.ts`/`axiom-plan.ts` (with a concrete,
non-hypothetical corruption scenario identified and confirmed closed by
the guards); `axiom-decision.ts`'s deliberate omission of `supersede`;
and `index-cmd.ts`'s `SCANNED_KINDS` coverage of `adr`/`decision`.

Found and fixed two real gaps neither prior increment surfaced: (1)
`@axiom/doctor`'s `IX-001` check had its own separate
`ARTIFACT_INDEX_KINDS` constant that was not updated in step with
`index-cmd.ts`'s `SCANNED_KINDS`, meaning `axiom index rebuild`/`validate`
covered adr/decision but the doctor health check did not — fixed with a
one-line extension plus two new tests, no new check needed since `IX-001`
is already the correct generic mechanism; (2) `axiom adr supersede` had
an untested, undocumented silent-overwrite edge case when re-superseding
an already-superseded ADR by a different new ADR — fixed by adding an
explicit, non-blocking warning that names the prior superseder, plus two
new tests.

Full monorepo `typecheck`/`build` are clean; `test` shows zero
regressions against both the original baseline and cli-implementer's own
numbers, with 4 additional tests (all passing) added by this increment on
top of cli-implementer's 28.

## General spec integration

Added a new "ADR/Decision artifact kinds" subsection to
`Axiom.Spec/general-spec.md`, alongside the existing "Folder-per-artifact
convention," "Skills role-index," and "Technical-context index" sections,
per the schema-writer's own forward note and this repo's `AGENTS.md`
documentation rules (own file first, then `general-spec.md`, stable
knowledge only, no full implementation history copied). States: the two
new kinds and their folder names, their own non-`WorkflowState` status
vocabularies, the `supersede` operation and its reassignment-warning
behavior, the derived (not curated) ADR-index decision, and the `IX-001`
doctor-check coverage — in that section's existing terse, one-paragraph
style.

## INC-11 closure summary (whole chain: schema-writer -> cli-implementer -> validator-reviewer)

All roadmap-assigned roles for INC-11 have completed:

1. **schema-writer** (`INC-20260702-adr-decision-reconcile-design`):
   produced the `AdrMetadata`/`DecisionMetadata` schema, resolved the
   `status`-vocabulary typing conflict via `BaseArtifactMetadataFields`,
   and recommended a derived (not curated) ADR index.
2. **cli-implementer** (`INC-20260702-adr-decision-reconcile-impl`):
   implemented the schema exactly as designed, added `axiom-adr`/
   `axiom-decision` CLI commands, wired both kinds into `index rebuild`/
   `validate`, and surfaced/fixed three latent narrowing-safety gaps in
   existing increment/bug/plan status-sync code.
3. **validator-reviewer** (this increment): independently re-verified
   every claim above against the actual code (not the prior specs'
   prose), found and fixed two additional real gaps (the `IX-001`
   `ARTIFACT_INDEX_KINDS` omission and the `supersede` silent-reassignment
   edge case), and ran a fresh full-monorepo validation pass showing zero
   regressions.

**Closure rule check** (per `Axiom.SDD/AGENTS.md`):

- Goal/expected behavior clear: yes, across all three roles.
- Acceptance criteria exist: yes, in all three specs.
- Changes implemented (or no-code rationale explicit): yes — schema-writer
  is design-only by explicit scope; cli-implementer and validator-reviewer
  both implemented real, tested code.
- Available validation executed: yes, independently, twice (cli-implementer
  and validator-reviewer each ran their own fresh full-suite pass).
- Review against intent and acceptance criteria: done, explicitly, in this
  file's "Implementation notes" section item-by-item.
- Stable knowledge integrated into `general-spec.md`: done (see above).
- Result documented clearly: yes, in all three specs.

**All closure conditions are met. INC-11 as a whole is CLOSED.**

Deferred items (explicitly, not silently dropped):

- Curated/versioned ADR index (obligatoriedad, priority, groupings) —
  deliberately deferred by schema-writer's own design, reaffirmed here,
  pending a concrete future consumer (e.g. an MCP tool needing "ADR-000x
  is mandatory reading"). Not part of this or any INC-11 role's scope.
- `axiom.yml` `paths` configurability for `adr`/`decisions` folder names —
  deliberately out of scope per schema-writer's non-goals; would touch
  every existing artifact kind, not just adr/decision.
- Semantic graph cleanup of stale `supersedes` entries when a `supersede`
  reassignment happens (item 4's fix surfaces the inconsistency via a
  warning but does not auto-clean the prior superseder's own
  `links.supersedes` array) — a deliberate, proportionate scope
  boundary for a CLI-ergonomics fix, not a design gap; documented in this
  file's implementation notes as a known, surfaced (not hidden)
  limitation.
- Relocating this increment chain's implementation from `Axiom` to
  `Axiom.SDD` to match `Axiom.SDD/AGENTS.md`'s stated repository-role
  split — pre-existing convention across all three roles in this chain
  (and, per direct inspection, across the entire `Axiom` product's actual
  codebase), not something this validation pass should change as a side
  effect. Flagged here for whoever eventually reconciles the roadmap's
  repository-role documentation with its actual practice.

## Next step recommendation

INC-11 is closed. Per the parent roadmap
(`INC-20260702-axiom-redesign-roadmap/README.md`), the next queued
increment is **INC-12 ("Human-readable generated views, optional")**,
which the roadmap itself marks low-priority/optional.

**Recommendation: do not default to INC-12 next; ask the user/roadmap
owner to pick the next highest-value increment instead**, for these
concrete reasons (not a generic "always deprioritize optional work"
heuristic):

- The roadmap explicitly flags INC-12 as optional, which is a signal from
  the roadmap's own author that it is not on the critical path — INC-11's
  closure does not create new urgency for it.
- This validator-reviewer pass surfaced a real, if narrow, pattern risk
  worth naming: two increments in a row (cli-implementer here, and
  per this chain's own citation, INC-06/07's original migration-engineer
  audit) each found "one more hardcoded kind-array that needed updating"
  after the fact (`SCANNED_KINDS` here, and analogous findings in earlier
  audits). If there is a next increment in the pipeline touching
  `ArtifactKind`, `WorkflowState`, or any other closed-union "grows over
  time" type, doing that increment's own migration-engineer/
  validator-reviewer pass with this exact "search all hardcoded arrays of
  this union type across the whole monorepo" check baked in up front would
  likely be higher-value than a human-readable-views feature.
- If the user has no other explicit priority, INC-12 remains a reasonable
  default (it is still the next item in sequence and nothing blocks it),
  but this is a judgment call the user should confirm rather than an
  automatic next action, given the roadmap's own "optional" framing.
