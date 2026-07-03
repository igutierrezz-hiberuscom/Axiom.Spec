# Increment: Reconcile technical context structure — implementer

Status: closed
Date: 2026-07-02

Closed as part of INC-10's overall closure by
`INC-20260702-technical-context-index-reconcile-validator/README.md`
(validator-reviewer, final step of the chain), which independently
re-verified this increment's implementation and ruled on both flagged
design/live-code mismatches. See that spec's "INC-10 closure assessment"
section for the full closure rationale covering all four increments in
this chain.

## Goal

Execute the **implementer** step of INC-10's subagent sequence
(migration-engineer -> schema-writer -> **implementer** -> next step),
following on from
`Axiom.Spec/specs/increments/INC-20260702-technical-context-index-reconcile/README.md`
(migration-engineer audit) and
`Axiom.Spec/specs/increments/INC-20260702-technical-context-index-reconcile-design/README.md`
(schema-writer design, the exact implementation contract for this
increment). This role combines the docs-skills-writer + cli-implementer
roles the design's own next-step recommendation named, per that
recommendation's own rationale ("skip docs-skills-writer for now...
recommend going directly to a combined implementer role"), and
additionally builds the `@axiom/doctor` `TC-013` check the design left
as a forward note for whichever role picks up validator-reviewer's step
next — collapsing that hop too, since the check is a small, mechanical,
directly-specified extension of TC-012's already-established pattern
with no open design questions left to resolve.

## Context

Parent chain: `INC-20260702-axiom-redesign-roadmap` (parent roadmap) ->
`INC-20260702-technical-context-index-reconcile` (migration-engineer
audit, `pending`) -> `INC-20260702-technical-context-index-reconcile-design`
(schema-writer design, `pending`) -> this increment (implementer).

The schema-writer design is treated as the authoritative, exact
implementation contract: type family, `resolveMandatoryDocuments`
signature/semantics, `loadTechnicalContextIndex`/
`validateTechnicalContextIndex` error-handling contract, file-path
convention (`<specScope>/technical-context/indexes/<role-or-kind>.index.yml`),
and the worked `backend` example. Nothing in this increment revisits any
of the three resolved open questions (Q-context-1/2/3) — they are
implemented as designed, not re-litigated.

## Scope

- Create new package `Axiom/packages/technical-context/` (`package.json`,
  `tsconfig.json`, `src/index.ts` barrel), registered in the npm
  workspace (already covered by the root `packages/*` glob) and in the
  root `tsconfig.json`'s TypeScript project references.
- Implement `TechnicalContextIndex` and its nested types
  (`TechnicalContextDocumentRef`, `TechnicalContextWhenTagsGroup`,
  `TechnicalContextMandatory`, `TechnicalContextAvailableEntry`) plus
  hand-written validation (no Zod) in
  `packages/technical-context/src/technical-context-index.ts`, mirroring
  `@axiom/skills/src/role-index.ts`'s exact style and error-shape
  conventions.
- Implement `loadTechnicalContextIndex(specScopeAbsolutePath, roleOrKind)`
  (never-throws, `{kind: 'ok'|'absent'|'malformed'}`) and
  `technicalContextIndexPath`/`TECHNICAL_CONTEXT_INDEX_RELATIVE_DIR`.
- Implement `validateTechnicalContextIndex(parsed)` as a standalone,
  disk-independent function reusable by `doctor`.
- Implement `resolveMandatoryDocuments(index, taskTags)` with the
  designed ALL-tags-present (subset) semantics, deduping by `id`.
- Author a worked `backend` example fixture under
  `packages/technical-context/tests/fixtures/technical-context/indexes/backend.index.yml`,
  byte-for-byte matching the design's own worked example.
- Write tests: type/validation (valid + malformed cases), loader (happy
  path, absent file, malformed file), `resolveMandatoryDocuments`
  (matching group, non-matching group due to partial tag overlap,
  multiple groups, dedup, empty-tags-group, empty-taskTags).
- Add `@axiom/doctor` check `TC-013` (next free id after `TC-012`,
  confirmed by reading `checks.ts` fresh), reusing
  `validateTechnicalContextIndex`, following `TC-012`'s exact
  `DoctorCheck`/skip-when-absent conventions, and wire it into
  `runDoctorChecks`/`packages/doctor/src/index.ts`.
- Run real validation (`typecheck`, `build`, `test`) at package scope
  and full-monorepo scope, and confirm the current baseline via a
  clean-`HEAD` git worktree comparison (git stash was not viable — the
  working tree already carries substantial unrelated in-progress work
  from other increments; a worktree comparison achieves the same
  baseline-confirmation goal without touching that work).

## Non-goals

- No re-litigation of Q-context-1/2/3 — implemented exactly as the
  schema-writer design resolved them.
- No change to `@axiom/skills`'s `SkillsCatalog`/`SkillsRoleIndex` or to
  `@axiom/document-bootstrap`'s renderers/writer.
- No real `architecture.md`/`commands.md`/`persistence.md` content
  authored for any concrete project — the fixture reuses the design's
  own placeholder-style paths verbatim, exactly as 10.2's own worked
  example does. No docs-skills-writer content generation performed, per
  the design's own explicit rationale for skipping that role.
- No implementation of `get_implementation_context` or any of source doc
  8.3's response-envelope fields — remains INC-14, unchanged from both
  prior increments' non-goals.
- No fix attempted for any of the pre-existing, unrelated test failures
  found during validation (see "Validation" below) — they predate this
  increment's changes and are out of scope to fix here.

## Acceptance criteria

- [x] `Axiom/packages/technical-context/` exists with `package.json`,
      `tsconfig.json`, `src/index.ts`, and is registered in the root
      `tsconfig.json`'s references (workspace registration via the
      existing `packages/*` glob required no change).
- [x] `TechnicalContextIndex` and its nested types are implemented
      field-for-field per the schema-writer design, with hand-written
      validation (no Zod) mirroring `role-index.ts`'s style.
- [x] `loadTechnicalContextIndex`/`technicalContextIndexPath` implemented
      exactly per the design's signature and never-throws contract.
- [x] `validateTechnicalContextIndex` implemented as a standalone,
      disk-independent function.
- [x] `resolveMandatoryDocuments` implemented with ALL-present (subset)
      semantics and id-based dedup, matching the design's worked-example
      traces.
- [x] A worked `backend` fixture exists under `tests/fixtures/`,
      matching the design's own worked example.
- [x] Tests cover: valid load, malformed shape (multiple sub-cases),
      absent file, standalone validation reuse, and
      `resolveMandatoryDocuments` (matching, non-matching/partial
      overlap, multiple groups, dedup, empty-tags-group, empty
      taskTags).
- [x] `@axiom/doctor` gained a new `TC-013` check reusing
      `validateTechnicalContextIndex`, following `TC-012`'s exact
      pass/fail/skip conventions, wired into `runDoctorChecks`, with its
      own test coverage mirroring `TC-012`'s test scenarios.
- [x] Real validation was executed (`typecheck`, `build`, `test`) at both
      package scope and full-monorepo scope, with results compared
      against a confirmed clean-`HEAD` baseline.
- [x] This spec documents files/package created, real test results, and
      any flagged design/live-code mismatches.
- [ ] Independent review by validator-reviewer (see "Next step
      recommendation") — not yet performed, which is why this increment
      stays `pending` rather than `closed`.

## Open questions

None at this increment's own scope. All three of the audit's open
questions were already resolved by schema-writer; this increment
implements those resolutions without introducing new ones.

## Assumptions

- "Technical context" continues to mean source doc section 10's concept
  specifically, unchanged from both prior increments' Assumptions
  sections.
- The design's exact type field names, optionality choices, and
  function signatures are implemented verbatim — no deviation was judged
  necessary during implementation (see "Design/live-code mismatches"
  below for the one filename-matching nuance that required an
  implementation-level decision not explicitly pinned down by the
  design).
- Per the task's explicit instruction, `git commit` was not run at any
  point; all changes are left uncommitted in the `Axiom` working tree.

## Implementation notes

### Package created: `Axiom/packages/technical-context/`

```
packages/technical-context/
  package.json          — name @axiom/technical-context, deps: js-yaml only
                           (no @axiom/skills, no @axiom/core — matches the
                           audit/design's "no shared executable code"
                           conclusion)
  tsconfig.json          — extends ../../tsconfig.base.json, no `paths`/
                           `references` needed (no internal package imports)
  src/
    index.ts              — barrel, re-exports all public types/functions
    technical-context-index.ts
                           — TechnicalContextIndex type family,
                             validateTechnicalContextIndex,
                             loadTechnicalContextIndex,
                             resolveMandatoryDocuments,
                             technicalContextIndexPath,
                             TECHNICAL_CONTEXT_INDEX_RELATIVE_DIR
  tests/
    technical-context-index.test.ts   — 26 tests (5 scenarios)
    fixtures/technical-context/indexes/backend.index.yml
                                       — worked example, byte-for-byte
                                         matching the design/source-doc
                                         10.2 example
```

Registered in `Axiom/tsconfig.json`'s `references` array (sibling to
`packages/skills`). No npm workspace config change was needed —
`Axiom/package.json`'s `workspaces` already includes the `packages/*`
glob, so `npm install` linked the new package automatically.

`@axiom/doctor` was updated to depend on the new package:
`packages/doctor/package.json` (dependency), `packages/doctor/tsconfig.json`
(`paths`/`references` entries), mirroring exactly how it already
depends on `@axiom/skills`.

### Type/validation implementation

`TechnicalContextIndex`/`TechnicalContextDocumentRef`/
`TechnicalContextWhenTagsGroup`/`TechnicalContextMandatory`/
`TechnicalContextAvailableEntry` implemented exactly per the design's
type family (`packages/technical-context/src/technical-context-index.ts`).
`validateTechnicalContextIndex` performs hand-written narrowing (no Zod),
following `role-index.ts`'s helper-function decomposition style
(`isObject`, `isNonEmptyString`, `isStringArray`, per-shape parse
helpers, a duplicate-id check across all three document lists combined).

One small addition to the design's stated validation rules, needed for
type-checking to actually enforce them via TypeScript strictness (the
design's own prose rules already implied this, but the design's example
validator body left it as `/* implementation deferred */`): each
`TechnicalContextDocumentRef` requires both `id` and `path` non-empty
(design confirms `path` is required, not optional, unlike
`@axiom/skills`'s `path?`); this is implemented identically in both
`mandatory.always` entries and `mandatory.whenTags[].documents` entries,
since the design states both share the same shape.

### Loader implementation

`loadTechnicalContextIndex(specScopeAbsolutePath, roleOrKind)` implemented
exactly per the design: accepts an already-resolved `specScopeAbsolutePath`
(not a bare `rootPath`), resolves
`<specScopeAbsolutePath>/technical-context/indexes/<roleOrKind>.index.yml`
via `technicalContextIndexPath`, and returns the same
`{kind: 'ok'|'absent'|'malformed'}` discriminated union as
`loadSkillsRoleIndex`. The `specScope`-resolution snippet itself (Finding
2 from the audit) is **not** duplicated inside `@axiom/technical-context`
— per the design's own explicit intent, that responsibility stays with
the caller (a `doctor` check, a future CLI command, INC-14's tool). This
increment's `TC-013` doctor check is the one caller implemented so far,
and it performs the `specScope` lookup itself
(`resolveTechnicalContextIndexDir` in `checks.ts`), inlined the same way
`resolveSkillsRoleIndexDir` already is for `TC-012` — no shared helper
was factored out of `checks.ts` for this lookup, matching the audit's
own explicit permission not to force a refactor of `checks.ts` for this
increment ("don't force a refactor of `checks.ts` itself if it's out of
scope"). This is a duplication of a 6-line snippet across two sibling
check functions in the same file, not a duplication across packages —
judged acceptable at this scale, consistent with how the snippet is
already duplicated 12+ times within `checks.ts` itself for other checks.

### `resolveMandatoryDocuments` implementation

Implemented verbatim per the design's own reference implementation:
`index.mandatory.always` unconditionally, plus every `whenTags` group
whose `tags` are a full subset of `taskTags` (ALL-present, not
ANY-present), deduped by `id` (first occurrence wins, `always`-then-
`whenTags`-order), with an empty-`tags` group treated as never-matching.
Verified against both of the design's own worked-example traces
(`['persistence']` → only `always`; `['persistence', 'database',
'ef-core']` → `always` + the `whenTags` group) as explicit test cases,
plus additional cases the design named as needed test coverage but did
not itself trace: partial overlap (`['persistence', 'database']`, one
tag short of the group's three — correctly does NOT fire, confirming
ALL-vs-ANY), multiple `whenTags` groups firing independently, id dedup
between `always` and a matching `whenTags` group, and empty `taskTags`.

### Doctor check: `TC-013` (technical-context-index-validity)

Confirmed by a fresh read of `Axiom/packages/doctor/src/checks.ts` that
`TC-012` is the highest existing `TC-0xx` id; `TC-013` is next-free.
`runTechnicalContextIndexCheck` (new function, `checks.ts`) mirrors
`runSkillsRoleIndexCheck`/`TC-012` exactly:

- SKIP if `resolution.status !== 'resolved'`.
- SKIP if the spec scope cannot be resolved.
- SKIP if `<specScope>/technical-context/indexes/` does not exist
  (optional structure, same posture as `TC-012`).
- PASS with `0 technical-context index encontrados` if the directory
  exists but is empty.
- For each `*.index.yaml`/`*.index.yml` file found: read, YAML-parse,
  and call `validateTechnicalContextIndex` directly (no
  re-implementation of shape validation) — FAIL with per-file details on
  any read/parse/shape failure.

One filename-convention decision needed during implementation that the
design did not pin down explicitly (`TC-012`'s equivalent rule is
"`role` must match the filename stem," but a technical-context index's
own worked example is named `backend.index.yml` while its `role` field
is `backend-developer` — a `role`-vs-filename mismatch that is *correct*
per 10.2's own worked example, not an error): `TC-013` accepts a file as
correctly named if **either** `role` **or** any entry of `repoKinds`
matches the filename stem (`backend.index.yml` → stem `backend` matches
`repoKinds: [backend]`, even though `role: backend-developer` does not
match). This is documented explicitly below under "Design/live-code
mismatches" since it is a real gap the design left implicit, not a
deviation from anything the design stated outright.

`CATEGORY_SKILLS` was reused for `TC-013` (not a new category) — the
design's own forward note offered `CATEGORY_TOPOLOGY` or "a new/existing
skills-adjacent category" as options; `CATEGORY_SKILLS` was chosen
because `TC-013` is presented in the doctor report immediately adjacent
to `TC-012` (both cover optional, per-role index files with the same
skip-if-absent posture), which is more discoverable for a human reading
`doctor`'s categorized output than placing it under the unrelated
`topology` category. This is a naming/grouping choice for the `doctor`
report, not a code dependency — `@axiom/technical-context` itself still
has zero dependency on `@axiom/skills`.

### Design/live-code mismatches flagged for validator-reviewer

1. **Filename-matching rule for `TC-013` was not explicit in the
   design.** The design specified the schema and the loader/validator,
   but its own "What comes after this design" forward note for the
   future doctor check only says "every document `id`'s `path` resolves
   to a file that exists on disk" and "no duplicate `id`" — it does not
   mention a `role`/`repoKinds`-vs-filename convention at all (unlike
   `TC-012`'s spec, which explicitly required `role === filename stem`).
   Since 10.2's own worked example uses `role: backend-developer` in a
   file named `backend.index.yml` (a `repoKinds`-based match, not a
   `role`-based one), a literal `role === filename stem` check
   (`TC-012`'s exact rule) would have **failed the design's own worked
   example** if applied verbatim to `TC-013`. This increment resolved it
   by accepting either match (see above), which is `TC-013`'s only
   implementation-level decision not directly dictated by prior specs —
   flagged explicitly for validator-reviewer to confirm or override.
2. **`TC-013` does not yet implement path-existence-on-disk checking for
   each document reference**, which the design's forward note does
   mention explicitly ("every document `id`'s `path` resolves to a file
   that exists on disk relative to the index file's own location"). This
   increment's `TC-013` validates shape only (parses + matches
   `TechnicalContextIndex`, no duplicate ids), same scope as what
   `validateTechnicalContextIndex` itself checks — it does NOT resolve
   and stat every relative `path` on disk. This was a deliberate scope
   decision consistent with `TC-012`'s own precedent (`TC-012` also does
   not verify that `mandatory`/`available` `path`s exist on disk when
   present, only that catalog `id`s resolve or a `path` string is
   present) but is flagged here explicitly as a gap versus the design's
   forward note, for validator-reviewer to judge whether it should be
   added in a follow-up increment or left as a known, accepted scope
   limit.

## Validation

Validation command discovery: `Axiom/package.json` defines
`build` (`tsc -b`), `typecheck` (`tsc -b`), `test`/`test:watch`
(`vitest run`/`vitest`), and package-level `package.json`s define their
own scoped `build`/`typecheck`/`test` scripts (e.g.
`packages/technical-context/package.json`,
`packages/doctor/package.json`). These were run directly.

### Package-scoped results

- `npx tsc -b packages/technical-context` — clean, no output, exit 0.
- `packages/technical-context` tests (`npx vitest run`): **26/26
  passed** (5 scenarios: valid load incl. fixture integration test,
  malformed shape x10 sub-cases, absent file + path resolution, standalone
  validation reuse x3, `resolveMandatoryDocuments` x7 sub-cases).
- `npx tsc -b packages/doctor` — clean, no output, exit 0 (confirms
  `@axiom/doctor`'s new dependency on `@axiom/technical-context`
  type-checks correctly).
- `packages/doctor` tests (`npx vitest run`): **146/147 passed** in this
  package. The 1 failure
  (`runDoctorChecks — repositorio Axiom real > encuentra y resuelve el
  proyecto Axiom real`) is a pre-existing, environment-path-dependent
  test unrelated to this increment's changes — confirmed below. The new
  `runTechnicalContextIndexCheck` (`TC-013`) test suite: **8/8 passed**
  in isolation (`-t "runTechnicalContextIndexCheck"`).

### Full-monorepo results

- `npm run typecheck` (`tsc -b` from repo root): clean, no output, exit
  0.
- `npm run build` (`tsc -b` from repo root): clean, no output, exit 0.
- `npm test` (`vitest run` from repo root): **1408/1421 passed**, 13
  failing tests across 12 files. All 13 failures are pre-existing,
  environment/real-path-dependent integration tests unrelated to any
  file this increment touched (`apps/cli/tests/start.test.ts`,
  `packages/agents/tests/catalog.test.ts`,
  `packages/doctor/tests/checks.test.ts`'s "repositorio Axiom real" case,
  `packages/model-routing/tests/*` x4, `packages/skills/tests/catalog.test.ts`,
  `packages/telemetry/tests/audit-trail-sink.test.ts`,
  `packages/toolchain/tests/repair-add-gitignore.test.ts`,
  `packages/project-resolution/tests/resolver.test.ts`,
  `packages/tui/tests/driver.test.ts` x2).

### Baseline confirmation (git worktree comparison, not stash)

`git stash` was not used to confirm baseline: the actual `Axiom` working
tree already carries substantial unrelated, uncommitted work from other
in-progress increments (`git status --porcelain` showed ~30 modified/new
files spanning `apps/cli`, `packages/workflow`, `packages/skills`,
`packages/project-resolution`, `packages/filesystem-truth`, `packages/tui`,
`packages/user-workspace`, `packages/installer`, and `packages/doctor`
itself, none of it produced by this increment). Stashing that work risked
disrupting other sessions' in-progress state for no benefit. Instead, a
disposable `git worktree` checked out at `HEAD` (`4426241`, unrelated to
and untouched by this increment) was used to run the identical
`npm test` at a clean baseline, then removed:

- At clean `HEAD` (before any of this increment's or any other pending
  session's changes): **1091/1105 passed**, 14 failing tests across 21
  files — a strict superset of failure names, including all 13 seen in
  the actual working tree (`agents/catalog`, `skills/catalog`,
  `doctor/checks "repositorio Axiom real"`, `model-routing/*`,
  `project-resolution/resolver`, `start`, `telemetry`, `toolchain repair`,
  `tui/driver`), confirming these are pre-existing, not introduced by
  this increment or by the other pending work already in the tree.
- The working tree (with this increment's changes plus other pending
  work) shows **fewer** failures than clean `HEAD` (13 vs. 14 tests, 12
  vs. 21 files) — consistent with other pending increments' fixes
  already being layered in, not with this increment introducing any
  regression.
- The worktree was removed after the comparison
  (`git worktree remove`/`prune`); no residual state was left behind.

## Result

Implemented `@axiom/technical-context` exactly per the schema-writer
design: the full `TechnicalContextIndex` type family, hand-written
validation (no Zod), `loadTechnicalContextIndex`/
`technicalContextIndexPath` (never-throws, caller-resolved `specScope`),
`validateTechnicalContextIndex` (standalone, disk-independent, reused
directly by the new `doctor` check), and `resolveMandatoryDocuments`
(ALL-present/subset `whenTags` matching, id-deduped). Authored a
byte-for-byte worked `backend` fixture matching the design's own
example, with 26 passing tests covering load/validation/loader/
resolution behavior including edge cases the design named but did not
itself fully enumerate (partial tag overlap, multiple groups, dedup,
empty-tags-group). Extended `@axiom/doctor` with `TC-013`
(`technical-context-index-validity`), reusing `validateTechnicalContextIndex`
directly and mirroring `TC-012`'s exact skip-if-absent/pass/fail
conventions, with 8 passing tests of its own, wired into
`runDoctorChecks`. Real validation was executed at both package and
full-monorepo scope (`typecheck`, `build`, `test`), and the resulting 13
pre-existing test failures were confirmed, via a disposable git-worktree
comparison against clean `HEAD`, to predate and be unrelated to every
file this increment touched — no regression was introduced. Two
design/live-code gaps were flagged explicitly for validator-reviewer
(the `role`-vs-`repoKinds` filename-matching rule `TC-013` had to decide
on its own, and the not-yet-implemented on-disk path-existence check the
design's forward note mentioned).

## General spec integration

No integration into `Axiom.Spec/general-spec.md` performed yet.
Rationale, consistent with the equivalent step in INC-09's own chain
(`INC-20260702-skills-role-index-reconcile-impl`'s closure rationale):
even though this increment's `TC-013` doctor check is more complete than
INC-09's implementer step was at the equivalent point (that step had not
yet built `TC-012` at all; this one has), the check has not yet received
independent review — no second pair of eyes has confirmed the
`role`/`repoKinds` filename-matching decision, the deliberate
path-existence-on-disk scope limit, or the `CATEGORY_SKILLS` reuse
choice are sound. Locking the technical-context index shape into
`general-spec.md` now would consolidate an artifact whose full
implementation has not yet survived independent review, the same
threshold INC-09's chain used consistently across all of its own
non-final increments. The correct integration point remains after
validator-reviewer's review closes out the full INC-10 chain, at which
point `general-spec.md` should gain a concise "Technical-context index"
section — mirroring the existing "Skills role-index" section's
structure and explicitly cross-referencing it in both directions (the
9.4-vs-10.2 domain distinction should be restated consistently in both
places), documenting: the file placement pattern
(`technical-context/indexes/<role-or-kind>.index.yml`, not the
`axiom.spec/config/` sibling convention `@axiom/skills` uses), the
nested `mandatory.always`/`mandatory.whenTags` shape, the ALL-present
`whenTags` matching semantics, and `TC-013` as its validation gate.

## Next step recommendation

Hand off to **validator-reviewer** next, scoped as an independent review
of this increment's implementation (not a from-scratch build, since
`TC-013` already exists) — following the same final-step role INC-09's
own chain used, adapted to this chain having already produced the check
itself. Exact inputs it needs:

1. **This increment's full diff** under `Axiom/packages/technical-context/`
   (new package) and `Axiom/packages/doctor/{package.json,tsconfig.json,
   src/checks.ts,src/index.ts,tests/checks.test.ts}` (new dependency +
   `TC-013`), plus the root `Axiom/tsconfig.json` reference addition, as
   the artifact to review against both prior specs in this chain.
2. **The two flagged design/live-code mismatches** above (filename-
   matching rule, path-existence-on-disk scope limit) as the two
   concrete judgment calls needing confirmation or override.
3. **The real validation results and baseline-comparison method**
   documented above (package-scoped + full-monorepo `typecheck`/`build`/
   `test`, plus the git-worktree-based baseline confirmation) as the
   evidence trail to independently spot-check rather than re-derive from
   scratch.
4. **This increment's own test files**
   (`packages/technical-context/tests/technical-context-index.test.ts`,
   the `TC-013` additions to `packages/doctor/tests/checks.test.ts`) as
   the existing test corpus to extend if validator-reviewer's review
   surfaces a gap (e.g. the path-existence-on-disk check, if judged
   worth adding now rather than deferring further).

After validator-reviewer closes this chain, integrate the consolidated
"Technical-context index" section into `Axiom.Spec/general-spec.md`
(cross-referencing "Skills role-index"), closing out the full INC-10
chain per the parent roadmap.
