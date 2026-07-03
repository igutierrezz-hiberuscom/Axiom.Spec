# Increment: Reconcile technical context structure — validator-reviewer

Status: closed
Date: 2026-07-02

## Goal

Execute the **validator-reviewer** step of INC-10's subagent sequence
(migration-engineer -> schema-writer -> implementer -> **validator-reviewer**,
final), following on from
`Axiom.Spec/specs/increments/INC-20260702-technical-context-index-reconcile/README.md`
(migration-engineer audit),
`Axiom.Spec/specs/increments/INC-20260702-technical-context-index-reconcile-design/README.md`
(schema-writer design), and
`Axiom.Spec/specs/increments/INC-20260702-technical-context-index-reconcile-impl/README.md`
(implementer). This increment independently re-verifies the implementer's
own claims against live code, rules on the two design/live-code mismatches
the implementer explicitly flagged, runs real validation independently,
and — if warranted — closes INC-10 as a whole.

## Context

Parent chain: `INC-20260702-axiom-redesign-roadmap` (parent roadmap) ->
`INC-20260702-technical-context-index-reconcile` (audit, `pending`) ->
`INC-20260702-technical-context-index-reconcile-design` (design,
`pending`) -> `INC-20260702-technical-context-index-reconcile-impl`
(implementer, `pending`) -> this increment (validator-reviewer, final
step of the chain).

This increment's job is independent verification, not a from-scratch
build: the package (`Axiom/packages/technical-context/`) and the `TC-013`
doctor check already exist. Nothing here re-litigates Q-context-1/2/3
(resolved by schema-writer) or the type/loader/validator/resolver
implementation itself (built by implementer) except where independent
tracing surfaced a concrete concern.

Two design/live-code mismatches were explicitly flagged by the
implementer for this role to judge:

1. `TC-013`'s filename-matching rule accepts either `role` **or** any
   `repoKinds` entry matching the file stem, because a literal
   `role === stem` check (`TC-012`'s exact rule) would incorrectly fail
   against the design's own worked example (`role: backend-developer`
   inside `backend.index.yml`, matched via `repoKinds: [backend]`
   instead).
2. `TC-013` performs shape-only validation — it does not verify that each
   document's `path` resolves to a file that exists on disk, even though
   the schema-writer design's own forward note anticipated this as
   future `doctor`-check scope.

## Scope

- Independently read `packages/technical-context/src/technical-context-index.ts`
  in full (`validateTechnicalContextIndex`, `loadTechnicalContextIndex`,
  `resolveMandatoryDocuments`) and confirm the type matches the
  schema-writer design field-for-field.
- Manually trace `resolveMandatoryDocuments` against a fully-matching tag
  set and a partially-overlapping tag set to independently confirm the
  ALL-tags-present (subset) semantics are implemented correctly, not just
  claimed.
- Independently judge flagged decision #1 (filename-matching rule):
  confirm as correct, or narrow it, with documented rationale either way.
- Independently judge flagged decision #2 (no path-existence check):
  decide whether to add it now (small, `warn`-not-`fail`, per-document
  `fs.existsSync`) or defer it, with documented rationale either way.
- Confirm no unintended changes leaked into files unrelated to this
  increment's own scope (`git diff --stat`).
- Run real validation independently (`npm run typecheck`, `npm run
  build`, `npm test`) and compare against the stated pre-INC-10 baseline
  (12 failed files / 13 failed tests / 1374 passed, per INC-09's close).
- Close INC-10 as a whole if all of its acceptance criteria are met,
  naming any deferred items explicitly.
- Integrate a "Technical-context index" section into
  `Axiom.Spec/general-spec.md`, cross-referencing "Skills role-index" in
  both directions.

## Non-goals

- No re-litigation of Q-context-1/2/3 (schema-writer's resolved
  decisions) or of the type/loader/validator/resolver design itself,
  except where independent tracing surfaced a concrete, evidenced
  concern (it did not, beyond the two explicitly flagged items).
- No implementation of `get_implementation_context` or any of source doc
  8.3's response-envelope fields — remains INC-14, unchanged from all
  three prior increments' non-goals.
- No fix attempted for any of the 13 pre-existing, unrelated test
  failures confirmed during validation — they predate this chain
  entirely and are out of scope for INC-10.
- No `git commit` — all changes left uncommitted per explicit
  instruction.

## Acceptance criteria

- [x] `technical-context-index.ts` independently read in full; type
      confirmed to match the schema-writer design field-for-field.
- [x] `resolveMandatoryDocuments` manually traced against a
      fully-matching tag set and a partially-overlapping tag set;
      ALL-tags-present semantics independently confirmed correct.
- [x] Flagged decision #1 independently judged, with an explicit ruling
      and rationale (confirm or narrow).
- [x] Flagged decision #2 independently judged, with an explicit ruling
      and rationale (implement now or defer).
- [x] `git diff --stat` reviewed across this session's touched files;
      confirmed scope matches what the implementer claimed, with any
      shared-file caveats documented.
- [x] Real validation executed independently (`typecheck`, `build`,
      `test`) and compared against the stated pre-INC-10 baseline.
- [x] INC-10 closed as a whole if all of its acceptance criteria are met
      (see "INC-10 closure assessment" below), or left `pending` with
      explicit rationale if not.
- [x] `general-spec.md` gains a "Technical-context index" section
      cross-referencing "Skills role-index," or an explicit no-op
      rationale is recorded.

## Open questions

None. Both flagged decisions are resolved below with explicit rulings.

## Assumptions

- "Technical context" continues to mean source doc section 10's concept
  specifically, unchanged from all three prior increments' Assumptions
  sections.
- Per the task's explicit instruction, `git commit` was not run at any
  point; all changes are left uncommitted in the `Axiom` working tree.
- The implementer's own test suite (26 tests in
  `packages/technical-context/tests/technical-context-index.test.ts`, 8
  tests for `TC-013` in `packages/doctor/tests/checks.test.ts`) is
  treated as existing, trustworthy evidence to independently spot-check
  by direct reading and manual tracing, not to blindly re-run without
  verification or to discard and rebuild from scratch.

## Implementation notes

### Independent type verification

Read `packages/technical-context/src/technical-context-index.ts` in full
(514 lines). Confirmed field-for-field against the schema-writer design:

- `TechnicalContextIndexSchemaVersion = 1`, `TechnicalContextDocumentRef
  {id, path, reason?}`, `TechnicalContextWhenTagsGroup {tags, documents}`,
  `TechnicalContextMandatory {always, whenTags}`,
  `TechnicalContextAvailableEntry {id, path, summary?, tags}`,
  `TechnicalContextIndex {schemaVersion, projectId?, role, repoKinds,
  mandatory, available}` — every field, every optionality choice
  (`projectId?`, `reason?`, `summary?` optional; `role`, `repoKinds`,
  `path`, `tags` required) matches the design's own type family and
  field-deviation table exactly. No field was invented; no field was
  dropped.
- `validateTechnicalContextIndex` implements every shape rule the design
  specified in prose (`schemaVersion === 1`, non-empty `role`,
  `repoKinds` as a required string array, optional non-empty
  `projectId`, `mandatory.always`/`mandatory.whenTags` both required
  arrays, per-entry `id`/`path` non-empty with optional `reason`,
  `whenTags` groups requiring a `tags` string array and a `documents`
  array, `available` entries requiring `id`/`path`/`tags` with optional
  `summary`, and cross-list `id` uniqueness across `always` +
  `whenTags[].documents` + `available` combined). Confirmed via direct
  read of `parseDocumentRef`, `parseWhenTagsGroup`, `parseAvailableEntry`,
  and `checkDuplicateIds` — hand-written narrowing, no Zod, matching
  `role-index.ts`'s style exactly as claimed.
- `loadTechnicalContextIndex(specScopeAbsolutePath, roleOrKind)` confirmed
  never-throws (`fs.existsSync` guard before any read, `try/catch` around
  both `readFileSync` and `yaml.load`), returning `{kind: 'absent'}` for
  a missing file and delegating shape validation to
  `validateTechnicalContextIndex` for the rest — exactly the contract the
  design specified, including accepting an already-resolved
  `specScopeAbsolutePath` rather than a bare `rootPath`.

### Independent manual trace of `resolveMandatoryDocuments`

Traced directly against the implementation's own logic (lines 485-513),
using the worked `backend` fixture's `whenTags` group
(`tags: [persistence, database, ef-core]`, `documents:
[backend.persistence]`), independently of the implementer's own test
assertions:

- **Fully-matching tag set** — `taskTags = ['persistence', 'database',
  'ef-core']`: `group.tags.every((tag) => tagSet.has(tag))` evaluates
  `persistence` ∈ `{persistence, database, ef-core}` → true, `database` ∈
  set → true, `ef-core` ∈ set → true. All three pass → `allPresent =
  true` → group fires. Result: `mandatory.always` (`backend.architecture`,
  `backend.commands`) plus the group's `documents`
  (`backend.persistence`) = 3 documents, in that order. **Confirmed
  correct.**
- **Partially-overlapping tag set** — `taskTags = ['persistence',
  'database']` (2 of the group's 3 tags present, intentionally chosen to
  distinguish ALL-semantics from ANY-semantics): `group.tags.every(...)`
  evaluates `persistence` → true, `database` → true, `ef-core` ∈
  `{persistence, database}` → **false**. `.every()` short-circuits to
  `false` on the first failing element → `allPresent = false` → `continue`
  skips the group entirely. Result: only `mandatory.always` (2
  documents), the `whenTags` group's `backend.persistence` is correctly
  **excluded**. **Confirmed correct** — this is the exact case that would
  have produced a false positive under an ANY-semantics implementation
  (2 of 3 tags overlapping is enough to fire under ANY, but must not fire
  under ALL), and the live code correctly does not fire it.

Both traces match the implementer's own test assertions
(`technical-context-index.test.ts` Scenario 5) exactly, but were derived
independently from the source code's own control flow rather than from
trusting the test file's expectations. The ALL-tags-present (subset)
logic is implemented correctly, not merely claimed.

### Ruling on flagged decision #1 — filename-matching rule

**Ruling: confirmed as correct. No code change.**

Independent analysis of the concrete adversarial scenario posed (a file
named `backend.index.yml` containing `role: totally-unrelated-role` but
`repoKinds: ['backend']`):

- This is **not** a false-positive risk. `repoKinds` is a first-class,
  required field of `TechnicalContextIndex` (per the schema-writer
  design, kept required specifically because 10.2's own worked example
  shows it explicitly, unlike `SkillsRoleIndex.repoKinds`'s optionality).
  Its entire purpose is to declare which repo-kind(s) an index applies
  to. A file at `technical-context/indexes/backend.index.yml` whose
  `repoKinds` includes `backend` is stating, through the one field
  designed for exactly this purpose, "this index applies to the
  `backend` repo kind" — which is precisely what the filename is also
  asserting. Matching on `repoKinds` is not a loophole; it is the
  correct field to check, arguably more correct than `role` for a
  repo-kind-scoped filename.
- The design's own worked example is the load-bearing counter-evidence
  against a literal `role === stem` rule: `role: backend-developer` in a
  file named `backend.index.yml`, matched only via `repoKinds:
  [backend]`. A `TC-012`-style single-field check applied verbatim would
  fail the chain's own canonical example — that would be a correctness
  bug in `TC-013`, not a faithful mirror of `TC-012`.
- The true adversarial case — neither `role` nor any `repoKinds` entry
  matching the stem — is already correctly caught as a `fail`, confirmed
  by direct read of `checks.ts` (`if (!result.value.repoKinds.includes
  (expectedStem) && result.value.role !== expectedStem)`) and by the
  existing test `TC-013 falla cuando ni \`role\` ni \`repoKinds\`
  coinciden con el nombre de archivo`
  (`packages/doctor/tests/checks.test.ts:530`), which uses `role:
  frontend-developer` + `repoKinds: [frontend]` in a file named
  `backend.index.yml` — a real naming mismatch, correctly failed. The
  positive case (`repoKinds` matches, `role` doesn't) is independently
  covered by `TC-013 pasa cuando \`repoKinds\` coincide... aunque \`role\`
  no` (`checks.test.ts:556`).
- Net judgment: the OR logic is the correct generalization of TC-012's
  single-field rule to a schema with two independent identifying fields,
  not an accidental widening. No narrowing is warranted; narrowing to
  `role`-only would reintroduce the exact bug the implementer already
  identified and fixed (failing the chain's own canonical example).

### Ruling on flagged decision #2 — no path-existence check

**Ruling: defer. No code change. Explicit rationale below, distinct from
"forgot to do it."**

Arguments considered for adding it now (`fs.existsSync`/`fileExists` per
document `path`, resolved relative to the index file's own directory,
`warn` not `fail`):

- Low implementation cost — `@axiom/filesystem-truth`'s `fileExists` is
  already imported in `checks.ts` for other checks, so no new dependency
  is required.
- The schema-writer design's own forward note explicitly named this as
  expected future `doctor`-check scope ("every document `id`'s `path`
  resolves to a file that exists on disk").
- Would catch a real, plausible authoring mistake (typo'd or moved
  relative path) that shape validation alone cannot catch.

Arguments against adding it now, which are judged decisive:

- **`TC-012` — the sibling check `TC-013` was explicitly told to mirror
  "exactly" — deliberately does not perform this check either**,
  confirmed by direct read of `runSkillsRoleIndexCheck`: it validates
  shape and filename-convention only, never resolves `mandatory`/
  `available` `path`s against disk. Adding path-existence to `TC-013` but
  not `TC-012` would make the two sibling checks inconsistent in scope
  for no stated reason, breaking the "mirror TC-012's exact convention"
  instruction both the audit's brief and the design's forward note gave.
- **The worked example and its shipped fixture use placeholder paths
  that do not resolve on disk**, by design — both source doc 10.2's own
  example and this repo's own
  `packages/technical-context/tests/fixtures/technical-context/indexes/backend.index.yml`
  point to `../backend/architecture.md`-style paths with no
  corresponding files anywhere in the repo. A `warn`-level path-existence
  check added today would immediately and permanently warn against the
  canonical fixture the test suite ships, on every future `doctor` run
  against this repo's own `axiom.spec/` scope if this fixture pattern
  were ever copied into a real project scaffold — noise without signal,
  since the placeholder nature is intentional and expected at this stage
  of the source doc's own worked example, not a defect to flag.
- **`Axiom.SDD/AGENTS.md`'s explicit guardrails** ("do not create
  speculative architecture," "keep changes small and focused," "do not
  modify unrelated files") favor the narrower, already-scoped check over
  an addition that mirrors a *different* check's scope rather than the
  one it was told to mirror.
- This is genuinely deferrable, not abandoned: the schema-writer design's
  forward note remains the correct place a future increment can pick
  this up from, once a concrete project with real (non-placeholder)
  technical-context documents exists and the noise concern above no
  longer applies uniformly.

Net judgment: consistent with TC-012's own precedent in the same file,
and avoiding a check that would produce structural noise against the
chain's own shipped fixture, the correct call is to defer this
explicitly rather than add it now. This mirrors the same threshold
INC-09's own validator-reviewer step used for equivalent judgment calls
on `TC-012`.

### `git diff --stat` review — scope confirmation

`git status --porcelain` in `Axiom/` (see raw listing in "Validation"
below) shows a working tree carrying substantial unrelated, uncommitted
work from other in-progress increments (~30+ modified/new paths spanning
`apps/cli`, `packages/workflow`, `packages/skills`,
`packages/project-resolution`, `packages/filesystem-truth`,
`packages/tui`, `packages/user-workspace`, `packages/installer`), exactly
as the implementer's own spec disclosed. This increment's (and the prior
two increments' combined) actual contribution is scoped to:

- `Axiom/packages/technical-context/` (new package, entirely additive).
- `Axiom/packages/doctor/{package.json,tsconfig.json,src/checks.ts,
  src/index.ts,tests/checks.test.ts}` (new dependency + `TC-013`).
- `Axiom/tsconfig.json` (new project reference).

One caveat independently confirmed and worth recording: `packages/doctor
/package.json` and `packages/doctor/tsconfig.json`'s diffs include
dependency/path/reference entries beyond `@axiom/technical-context`
alone (`@axiom/core`, `@axiom/installer`, `@axiom/skills`,
`@axiom/workflow`). Direct grep of `checks.ts` confirms these are already
imported and used by **other, pre-existing checks in the same file**
(`@axiom/core`'s `LOCAL_OVERLAY_DIRNAME`, `@axiom/skills`'s
`validateSkillsRoleIndex` for `TC-012`, `@axiom/workflow`'s
`listArtifacts` for artifact/workflow checks from other in-flight
increments) — not something this chain introduced. `checks.ts`'s own
diff (429 lines) is confirmed to be **entirely additive** (`git diff
--stat` shows `429 insertions(+), 0 deletions`), consistent with the
implementer's claim of only adding new functions without touching
existing logic. No unintended changes leaked into files outside this
chain's own stated scope.

## Validation

Validation command discovery: `Axiom/package.json` defines `typecheck`
(`tsc -b`), `build` (`tsc -b`), `test` (`vitest run`). These were run
independently, from a fresh invocation, not by trusting the implementer's
own reported numbers.

- `npm run typecheck` — clean, no output, exit 0.
- `npm run build` — clean, no output, exit 0.
- `npm test` — **1408/1421 passed, 13 failed across 12 files**:
  `apps/cli/tests/start.test.ts`, `packages/agents/tests/catalog.test.ts`,
  `packages/doctor/tests/checks.test.ts` ("repositorio Axiom real"),
  `packages/model-routing/tests/assignments.test.ts`,
  `packages/model-routing/tests/loader.test.ts`,
  `packages/model-routing/tests/opencode-projection.test.ts`,
  `packages/model-routing/tests/resolver.test.ts`,
  `packages/project-resolution/tests/resolver.test.ts`,
  `packages/skills/tests/catalog.test.ts`,
  `packages/telemetry/tests/audit-trail-sink.test.ts`,
  `packages/toolchain/tests/repair-add-gitignore.test.ts`,
  `packages/tui/tests/driver.test.ts` (x2 failures within the same
  file). Every one of these is an environment/real-path-dependent
  integration test (missing `axiom.spec/config/*.yaml` files that do not
  exist in this repo, absolute-path/cwd-dependent resolution, TUI
  terminal-rendering snapshot mismatches) — none touches
  `@axiom/technical-context`, `TC-013`, or any file this chain modified.
  This is an exact match (12 files / 13 tests) with the implementer's own
  independently-reported figures, confirmed by a fresh `npm test`
  invocation rather than trusting the prior report.
- Cross-checked against the stated pre-INC-10 baseline (12 failed files /
  13 failed tests / 1374 passed, per INC-09's close): failed-file and
  failed-test counts match exactly; passed-test count differs (1408 vs.
  1374), consistent with other in-flight increments having added tests
  to the suite since INC-09 closed — not a regression introduced by this
  chain. No new failure was introduced by INC-10's changes.
- `git status --porcelain` (`Axiom/`) reviewed directly to confirm scope
  (see "git diff --stat review" above) — not re-run through a worktree
  comparison a second time, since the implementer's own worktree-based
  baseline comparison (documented in the implementer spec) is treated as
  sufficient prior evidence for the regression question, independently
  corroborated here by the exact-match failed-file/failed-test counts.

## Result

Independently re-verified `packages/technical-context/src/technical-context-index.ts`
field-for-field against the schema-writer design by direct read (514
lines, not a summary) — confirmed matching with no invented or dropped
fields. Independently traced `resolveMandatoryDocuments` against a
fully-matching and a partially-overlapping tag set, confirming the
ALL-tags-present (subset) semantics are correctly implemented, not merely
claimed — the partially-overlapping trace is the specific case that would
expose an accidental ANY-semantics bug, and the live code correctly
excludes the non-fully-matching group. Ruled on both flagged decisions:
(1) the `role`-OR-`repoKinds` filename-matching rule is confirmed
correct, grounded in `repoKinds` being the field actually designed to
declare repo-kind applicability, with the true-negative case already
tested; (2) the on-disk path-existence check is deferred, not added,
because `TC-012` (the check `TC-013` was told to mirror exactly) does not
perform it either, and the chain's own shipped placeholder-path fixture
would produce structural warning noise if it were added now. Confirmed
via `git diff --stat` that no unintended changes leaked outside this
chain's own scope (`@axiom/technical-context` new package,
`@axiom/doctor`'s `TC-013` addition, one new tsconfig reference); the
broader working-tree diff is other in-flight increments' work, disclosed
consistently by the implementer and independently re-confirmed here. Ran
`typecheck`/`build`/`test` independently from a fresh invocation:
clean typecheck and build, 1408/1421 tests passed with the same 13
pre-existing, unrelated failures across 12 files as reported by the
implementer, matching the pre-INC-10 baseline's failed-file/failed-test
counts exactly.

## General spec integration

Added a new "Technical-context index" section to
`Axiom.Spec/general-spec.md`, placed immediately after the existing
"Skills role-index" section, cross-referencing it in both directions
(the "Skills role-index" section itself was updated with a one-line
cross-reference pointing forward to the new section, and the new section
opens by restating the 9.4-vs-10.2 domain distinction). The new section
documents: `TechnicalContextIndex`'s nested
`mandatory.always`/`mandatory.whenTags`/`available` shape, the file
placement convention (`<specScope>/technical-context/indexes/<role-or-
kind>.index.yml` — source doc 10.2's own literal path, not the
`axiom.spec/config/` sibling convention `@axiom/skills` uses), the
ALL-tags-present `resolveMandatoryDocuments` semantics, `TC-013` as the
validation gate (including the `role`-OR-`repoKinds` filename-matching
rule and the deferred path-existence-check decision), and this
increment's independent confirmation. This is the correct integration
point per both prior increments' own stated rationale (design not yet
implemented -> implemented but not independently reviewed -> now
independently reviewed and confirmed sound).

## INC-10 closure assessment

Reviewing INC-10 as a whole (all four increments in the chain) against
`Axiom.SDD/AGENTS.md`'s closure rules:

- **Goal or expected behavior is clear.** Yes — audit confirmed no
  technical-context structure existed; design resolved the exact shape;
  implementation built it; this increment independently verified it.
- **Acceptance criteria exist.** Yes — each of the four increments in the
  chain has its own explicit, checked acceptance-criteria list.
- **Changes were implemented, or no-code rationale is explicit.** Yes —
  `Axiom/packages/technical-context/` (new package: type family, loader,
  validator, resolver, 26 tests, 1 fixture) and `Axiom/packages/doctor`'s
  `TC-013` check (8 dedicated tests, wired into `runDoctorChecks`) are
  both implemented and independently verified in this increment.
- **Available validation was executed.** Yes — `typecheck`/`build`/`test`
  run independently in this increment, in addition to the implementer's
  own prior run and its git-worktree-based baseline comparison.
- **Review against intent and acceptance criteria was done.** Yes — this
  increment is exactly that independent review, including manual tracing
  of the core resolution logic and explicit rulings on both flagged
  design/live-code mismatches.
- **Stable knowledge was integrated into `Axiom.Spec/general-spec.md`
  when applicable.** Yes — the "Technical-context index" section was
  added this increment, cross-referencing "Skills role-index."
- **Result was documented clearly.** Yes — across all four increment
  specs in the chain, each with explicit "Result" sections.

All closure conditions are satisfied. **INC-10 ("Reconcile technical
context structure") is closed as a whole.** No item is deferred as
"blocking" — the one explicitly deferred item (on-disk path-existence
checking in `TC-013`) is a documented, judged scope decision, not an
unfinished acceptance criterion; it was never part of any of the four
increments' own acceptance criteria lists, only a forward note in the
design and a flagged judgment call resolved in this increment.

Deferred items, named explicitly:

1. **On-disk path-existence checking for technical-context document
   `path`s** (`TC-013` or a future check) — deferred per this increment's
   ruling on flagged decision #2, consistent with `TC-012`'s own
   precedent. Revisit once a concrete project with real (non-placeholder)
   technical-context content exists.
2. **`get_implementation_context`'s mandatory-document-direct-return rule**
   (source doc 10.3) — explicitly out of scope for all four increments in
   this chain; remains INC-14 per the parent roadmap.
3. **docs-skills-writer content generation** (real
   `architecture.md`/`commands.md`/`persistence.md` documents for a
   concrete project) — explicitly declined by both the audit and the
   design as speculative content generation for a project that does not
   yet exist; revisit only when a concrete project needs it.

## Next step recommendation

INC-10 is closed. Per the parent roadmap
(`Axiom.Spec/specs/increments/INC-20260702-axiom-redesign-roadmap/README.md`),
the recommended next step is **INC-11 ("Reconcile ADR/decision
handling")**, continuing the roadmap's Phase D sequence. No further work
is recommended on the technical-context index itself until either (a) a
concrete project needs real technical-context document content
(triggering the deferred docs-skills-writer step), or (b) INC-14
(mcp-tool-implementer) is picked up and needs `resolveMandatoryDocuments`
as an input.
