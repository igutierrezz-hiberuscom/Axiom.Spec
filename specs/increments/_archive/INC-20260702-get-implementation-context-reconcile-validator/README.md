# Increment: `get_implementation_context` (flagship MCP tool) — validator review

Status: closed
Date: 2026-07-02

## Goal

Independently verify `implementation-context-handler.ts` (INC-14's
`mcp-tool-implementer` output) against `@axiom/mcp-tools`'s "thin
registration layer" architecture rule, hand-trace the `confidence`/
`missingMetadata` heuristic and the `contextBudget` tier logic, decide
the `repositories.target` ambiguity question on concrete evidence, close
any real documentation gap for `mandatory.rules`/`mandatory.commands`,
run real validation independently, and close INC-14 as a whole if all
its acceptance criteria are met.

## Context

Roadmap: `Axiom.Spec/specs/increments/INC-20260702-axiom-redesign-roadmap/
README.md`, Phase E, INC-14. Role in INC-14's sequence:
**validator-reviewer**, the final role, following:

1. `INC-20260702-get-implementation-context-reconcile` (migration-engineer
   audit, closed).
2. `INC-20260702-get-implementation-context-reconcile-impl`
   (mcp-tool-implementer, pending on this review).

Reviewed code: `Axiom/packages/mcp-tools/src/implementation-context-handler.ts`,
`Axiom/packages/mcp-tools/src/registry.ts`, and the sibling handler files
it composes (`registry-handlers.ts`, `artifact-handlers.ts`,
`skills-handlers.ts`, `technical-context-handlers.ts`).

## Scope

- Trace all 8 composed sibling-handler calls in `buildImplementationContext`
  against their actual signatures to confirm no domain-package logic is
  duplicated.
- Hand-trace `confidence`/`missingMetadata` on 2 concrete scenarios.
- Hand-trace the 3 `contextBudget` tiers' conditional content-inlining
  logic directly in the source (not via test names).
- Evaluate the `repositories.target` ambiguity-resolution assumption
  against how `@axiom/topology`/registry v2 actually models multi-repo
  projects.
- Close the `mandatory.rules`/`mandatory.commands` "documented gap"
  visibility question — confirm it is visible to an MCP client at the
  response level, not just in source comments.
- Confirm the content-inlining file reads degrade gracefully on a
  missing/unreadable `README.md`.
- Run `npm run typecheck`, `npm run build`, `npm test` at the monorepo
  root and confirm the implementer's reported numbers independently.
- Close INC-14 as a whole if warranted.

## Non-goals

- No new capability ids, no new handler files, no change to
  `contextBudget`'s 3-tier design or to the `relatedSpec` placeholder
  decision (both already closed by the audit).
- No implementation of any deferred INC-13b tool
  (`get_workspace_equivalent`, `get_related_specs`,
  `list_recommended_skills`, `get_technical_context_document`).
- No new "rules"/"commands" backing artifact kind — the documented-empty
  decision stands; only its *visibility* is in scope.

## Acceptance criteria

- [x] Confirmed `buildImplementationContext` calls only already-exported
      sibling handler functions (`getProjectRegistry`, `getProjectRepos`,
      `getPlan`, `getIncrement`, `getAdrIndex`, `getSkillIndex` x2,
      `getTechnicalContextIndex`, `getAllowedWriteScope`) — zero
      re-imported backing domain functions beyond pure types/helpers
      (`resolveMandatoryDocuments`, which is itself a pure function, not
      I/O).
- [x] `confidence`/`missingMetadata` heuristic traced on 2 concrete
      scenarios (fully-resolved -> `high`; one missing mandatory doc ->
      `medium` with the field named) and confirmed to match the actual
      code path, not just test assertions.
- [x] All 3 `contextBudget` tiers' conditional logic traced directly in
      source: `small` has zero inlined `content` anywhere; `medium`
      inlines exactly `plan`+`relatedSpec`+`mandatory.*`; `large`
      additionally inlines `relatedAdrs`; `recommended.*` never inlines
      content at any tier.
- [x] `repositories.target` ambiguity resolution reviewed against
      `@axiom/topology`'s actual multi-repo model
      (`roleCodeRepositories: readonly RepoRef[]`) and a decision made
      (kept as-is, with rationale — see Implementation notes).
- [x] `mandatory.rules`/`mandatory.commands` documented-empty status
      confirmed visible at the response level (not just source comments)
      — gap found and closed with a minimal, additive field.
- [x] Content-inlining file reads (`resolveArtifactDir` + `fs.readFileSync`)
      confirmed to degrade gracefully (no throw) on a missing/unreadable
      `README.md`, traced directly in `readArtifactBody`/`inlineContent`.
- [x] `npm run typecheck`, `npm run build`, `npm test` executed
      independently at the monorepo root; results compared against the
      implementer's reported baseline/post-change numbers.
- [x] INC-14 as a whole reviewed for closure; deferred items named
      explicitly.

## Open questions

None blocking closure.

## Assumptions

None beyond what the audit and implementation increments already
recorded.

## Implementation notes

### 1. Composition-only check (no domain-package logic duplicated)

Traced `buildImplementationContext` line-by-line against the 4 sibling
handler files it imports from:

| Call in handler | Sibling function | File | Confirmed |
|---|---|---|---|
| `getProjectRegistry({homeDir})` | same | `registry-handlers.ts` | thin wrap of `listProjectsV2` |
| `getProjectRepos({homeDir, projectId})` | same | `registry-handlers.ts` | thin wrap of `getProjectV2` |
| `getPlan({projectRoot, id})` | same | `artifact-handlers.ts` | thin wrap of `loadArtifactMetadata('plan', ...)` |
| `getIncrement({projectRoot, id})` | same | `artifact-handlers.ts` | thin wrap of `loadArtifactMetadata('increment', ...)` |
| `getAdrIndex({projectRoot})` | same | `artifact-handlers.ts` | thin wrap of `listArtifacts('adr', ...)` |
| `getSkillIndex({rootPath, role})` (x2: sdd + target) | same | `skills-handlers.ts` | thin wrap of `loadSkillsRoleIndex` |
| `getTechnicalContextIndex({specScopeAbsolutePath, roleOrKind})` | same | `technical-context-handlers.ts` | thin wrap of `loadTechnicalContextIndex` |
| `getAllowedWriteScope({projectRoot, id})` | same | `artifact-handlers.ts` | thin wrap of `loadArtifactMetadata('plan', ...).allowedWriteScope` |

All 8 (9 counting the double `getSkillIndex` call) are direct calls to
already-exported sibling functions in the same package, matching every
call's actual parameter shape 1:1 — no parameter mismatch, no
reimplementation. `@axiom/skills`, `@axiom/technical-context`,
`@axiom/user-workspace`, `@axiom/workflow` are imported in
`implementation-context-handler.ts` **only** for types
(`SkillsRoleIndex`, `TechnicalContextIndex`, `RegistryRepoEntryWithStatus`,
`PlanMetadata`, `ArtifactMetadata`, `AllowedWriteScopeEntry`,
`ArtifactListEntry`) and for `resolveArtifactDir`/`resolveMandatoryDocuments`,
both pure functions with no I/O of their own (`resolveArtifactDir` is a
path-join helper; `resolveMandatoryDocuments` is a pure filter over an
already-loaded index — the exact same helper `listRecommendedContext`
already wraps). **Verdict: confirmed clean composition, no architecture-
rule violation.** The only genuinely new logic is `selectRepositories`
(a small selection over an already-loaded map — exactly the "thin new
composition logic" the audit called for) and the `confidence`/
`missingMetadata`/inlining helpers, all pure post-processing with no new
I/O beyond what the composed calls already perform (`readArtifactBody`/
`inlineContent` do add new `fs.readFileSync` calls, but these read the
same artifact folders the composed handlers already resolved paths for
— an additive read of already-known locations, not a new backing
function or a second copy of loading logic).

### 2. `confidence`/`missingMetadata` — hand-traced scenarios

**Scenario A (fully resolved)**: project found; `repos` map has
`spec`/`sdd`/`target` keys, `targetRepoId` omitted, `remainingKeys` after
excluding `spec`/`sdd` is exactly `['target']` (length 1) so `target`
resolves non-null, no `repositories.target` push; `getPlan` succeeds;
`plan.links.incrementId` set and `getIncrement` succeeds so `relatedSpec`
resolves; skill/technical-context indexes all load. Result:
`missingMetadata = []` -> `confidence = 'high'`. Matches
`implementation-context-handler.test.ts`'s "happy path" test exactly.

**Scenario B (one missing mandatory document)**: project + plan resolve;
no skills/technical-context YAML written for the role, so
`getSkillIndex`/`getTechnicalContextIndex` each return `ok:true, data:null`
(their own `absent` contract, not an error) -> `sddSkillIndex`,
`repoSkillIndex`, `technicalContextIndex` are all `null` ->
`'mandatory.sddSkills'`, `'mandatory.repoSkills'`,
`'mandatory.technicalContext'` all pushed to `missingMetadata`. Since
`projectEntry !== null` and `plan !== null` but `missingMetadata.length > 0`,
`confidence = 'medium'`. Matches the "partial data" test and the
acceptance-criteria's literal scenario. **Verdict: heuristic is sound —
`confidence` degrades exactly where a real gap exists, and every
degradation is traceable to a named field in `missingMetadata`.**

### 3. `contextBudget` tiers — traced directly in source

- **`small`**: `plan`/`relatedSpec` construction uses
  `contextBudget !== 'small' ? readArtifactBody(...) : undefined`, so at
  `small` the ternary always yields `undefined` and the object-spread
  `...(body !== undefined ? {content: body} : {})` adds nothing — the
  `content` key is genuinely absent from the JSON, not merely `undefined`
  as a value. `mandatory.*` inlining (`sddSkillsMandatoryOut` etc.) is
  gated on `contextBudget !== 'small'`, so at `small` all three stay the
  un-inlined ref arrays. `relatedAdrs` inlining is gated on
  `contextBudget === 'large'` only, so `small` never touches it.
  **Confirmed: `small` has zero inlined content anywhere.**
- **`medium`**: the three `mandatory.*Out` variables all inline via
  `contextBudget !== 'small' && <repoRoot> !== undefined`, true at
  `medium`. `plan.content`/`relatedSpec.content` populate via the same
  `contextBudget !== 'small'` ternary. `relatedAdrs`'s inlining block is
  strictly `if (contextBudget === 'large')`, false at `medium`, so
  `relatedAdrs` stays reference-only. **Confirmed: `medium` inlines
  exactly `plan` + `relatedSpec` + `mandatory.*`, nothing else.**
- **`large`**: same `medium` behavior (both conditions above still use
  `!== 'small'`, true at `large` too) plus the `relatedAdrs` block's
  `contextBudget === 'large'` branch now fires, mapping each ADR ref
  through `readArtifactBody(specRoot, 'adr', ref.id)`. **Confirmed:
  `large` additionally inlines `relatedAdrs`.**
- **`recommended.*`** (`recommendedSkills`, `recommendedTechnicalContext`,
  and `recommended.adrs` which reuses the `relatedAdrs` array reference)
  are never passed through `inlineContent` or any conditional at any
  tier — traced directly, confirmed by the "large" test asserting
  `recommended.skills[0]?.content` is `undefined` even at the richest
  tier. **Confirmed: the reference/recommendation split holds at every
  budget level, matching the audit's design intent.**

### 4. `repositories.target` resolution — reviewed against real topology model

The implementer's rule (explicit `targetRepoId` wins; else the sole
non-`spec`/`sdd` entry; else `null` + `missingMetadata`) was checked
against `@axiom/topology`'s actual multi-repo model, not just the
audit's abstract "open string-keyed map" description:

- `RegistryProjectManifest` (`@axiom/topology/src/types.ts`) models
  `sddRepo: RepoRef`, `specRepo: RepoRef`, and **`roleCodeRepositories:
  readonly RepoRef[]`** — an explicit array, not a single slot.
- `RepoRoleKey` (`@axiom/user-workspace/src/registry-types.ts:105`) is
  deliberately typed as an open `string`, not a closed union, with an
  explicit comment: *"Open-ended string, not a closed union, because
  `roleCodeRepositories` in `@axiom/topology` already allows arbitrary
  role-code repo ids beyond sdd/spec — the registry must not be stricter
  than the topology model it mirrors."*
- `axiom repo attach --role <role>` (`apps/cli/src/commands/repo.ts`)
  lets an operator attach multiple named "code" repos under distinct
  role keys for the same project id, confirming this is a real,
  designed-for, reachable scenario — not a hypothetical edge case.

**Decision: the current `null` + `missingMetadata` degrade is correct
and is kept as-is; no code change made.** Reasoning:

1. A project with 2+ non-`spec`/`sdd` repos and no explicit
   `targetRepoId` is exactly the scenario `RepoRoleKey`'s open-ended
   design was built to support — treating it as resolvable-by-guessing
   would contradict the very reason the type is open-ended.
2. There is no semantic ordering in the `repos` map. Its iteration order
   reflects YAML key-write order (parsed via `js-yaml`'s `yaml.load`),
   an implementation accident of *when* a repo was attached, not a
   declared "primary" repo. "Pick the first one deterministically" would
   be stable per-file but arbitrary and could silently select the wrong
   repo relative to what the caller actually means by "the target".
3. `target` feeds directly into `mandatory.repoSkills`/`recommended.skills`
   (skills loaded from the guessed repo's filesystem root). Silently
   guessing wrong here produces a *plausible-looking but wrong* mandatory
   skill set with `confidence: 'high'` — a correctness bug hiding behind
   false confidence. The current behavior instead produces
   `confidence: 'medium'` and a named `missingMetadata` entry, which is
   honest and pushes the caller to disambiguate via `targetRepoId`.
4. This matches the codebase's own established convention of not
   resolving genuine ambiguity silently (e.g. `getSkillIndex`'s
   `absent`/`malformed` distinction; `selectRepositories`'s own existing
   doc comment already states "falls back to `null` when that is
   ambiguous").

No cheaper-but-safer alternative was found; the reviewed behavior is
sound and was left unchanged.

### 5. `mandatory.rules`/`mandatory.commands` visibility — gap found and closed

The documented-empty decision was already visible in **source-level**
JSDoc/comments (file header, field-level `/** Documented empty... */`
comments on `MandatoryOutput.rules`/`.commands`, `IndexesOutput.commands`/
`.rules`, `RecommendedOutput.commands`). This is invisible to an MCP
client that only ever sees the emitted JSON — it would see
`"rules": [], "commands": []` and `"commands": {"present": false,
"count": 0}`, indistinguishable in shape from a per-project "nothing
found this time" case (e.g. `indexes.sddSkills` when a project simply
has no skills index yet also resolves to `{present: false, count: 0}`).
`missingMetadata` never mentions `rules`/`commands` either (correctly,
per the audit's decision — these are not "missing metadata" in the
sense the other fields are; they are a permanent product gap, not a
per-project data-completeness issue), so there was no existing
response-level signal a caller could use to distinguish "this project
has no skills index" from "this feature does not exist yet."

**Fix applied** (minimal, additive, non-breaking): added an optional
`unsupported?: boolean` field to `IndexSummary`
(`Axiom/packages/mcp-tools/src/implementation-context-handler.ts`), set
to `true` only on the two hardcoded `indexes.commands`/`indexes.rules`
entries. `sddSkills`/`repoSkills`/`technicalContext`/`adr` never set it
(stays `undefined`), preserving the existing per-project "not found"
semantics for those four. A new test
(`implementation-context-handler.test.ts`, "happy path" describe block)
asserts both the new field's value on `commands`/`rules` and its absence
on `sddSkills`, closing the ambiguity at the type and response level
without inventing a new artifact kind or a general-purpose metadata
system (stays within `AGENTS.md`'s bootstrap limits — one optional
boolean on an existing, already-small type).

### 6. Content-inlining file-read graceful degradation — traced directly

`readArtifactBody(projectRoot, kind, id)`
(`implementation-context-handler.ts`): wraps `resolveArtifactDir` +
`fs.existsSync` + `fs.readFileSync` in a `try/catch`; returns `undefined`
on any exception, and also explicitly returns `undefined` when
`fs.existsSync(filePath)` is `false` before ever attempting the read
(avoids relying on the exception path for the common "file legitimately
absent" case). Every call site treats `undefined` as "omit `content`",
never as an error.

`inlineContent(refs, baseDir)`: same pattern per-ref — `fs.existsSync`
check first, `try/catch` around the read, returns the original
reference unchanged (no `content` key added) on any failure. Confirmed:
**neither function can throw out of the composite handler; a missing or
unreadable `README.md` degrades to "no content inlined for this ref",
never a thrown exception or an `ok:false` envelope.** This matches
`getImplementationContext`'s own doc comment that the composite "has no
genuine I/O/parse error path of its own."

### 7. Independent validation run

Validation commands discovered via `package.json` scripts (`build`,
`typecheck`, `test`), same discovery as the prior two increments.

- `npm run typecheck` (monorepo root, `tsc -b`): **clean, zero errors**
  (confirmed before and after this review's own edit).
- `npm run build` (monorepo root, `tsc -b`): **clean, zero errors**
  (confirmed before and after this review's own edit).
- `npm test` (monorepo root, vitest), run independently, **before** this
  review's edit: **12 failed files / 13 failed tests / 1491 passed / 1504
  total** — matches the implementer's reported post-change numbers
  exactly. The 12 failing files were listed and diffed against the
  documented baseline list; **byte-identical** (`apps/cli/tests/start.test.ts`,
  `packages/agents/tests/catalog.test.ts`, `packages/doctor/tests/checks.test.ts`,
  `packages/model-routing/tests/{assignments,loader,opencode-projection,
  resolver}.test.ts`, `packages/project-resolution/tests/resolver.test.ts`,
  `packages/skills/tests/catalog.test.ts`,
  `packages/telemetry/tests/audit-trail-sink.test.ts`,
  `packages/toolchain/tests/repair-add-gitignore.test.ts`,
  `packages/tui/tests/driver.test.ts`) — all pre-existing,
  environment/fixture-dependent failures unrelated to `@axiom/mcp-tools`.
- `npm test`, run again **after** this review's `unsupported` field +
  test addition: **12 failed files / 13 failed tests / 1492 passed / 1505
  total** — +1 passed test is exactly the new "marks indexes.commands/.rules
  as unsupported" test. Same 12 failing files, same 13 failing tests.
  **No regressions.**
- `npx vitest run` scoped to `packages/mcp-tools`: **8 test files, 51
  tests, all passing**, confirmed before this review's edit (matching the
  implementer's reported count exactly); **8 files, 52 tests, all
  passing** after adding the one new test.

## Result

Independently confirmed `implementation-context-handler.ts` is a clean
composition over the 16 pre-existing `@axiom/mcp-tools` handlers with no
domain-package logic duplicated — every one of the 8 distinct sibling
calls (9 counting the double `getSkillIndex` call) traced against its
actual exported signature. `confidence`/`missingMetadata` hand-traced on
2 concrete scenarios and confirmed sound. All 3 `contextBudget` tiers'
content-inlining conditionals traced directly in source (not inferred
from test names) and confirmed to match the documented design exactly,
including that `recommended.*` never inlines content at any tier.

`repositories.target`'s `null`+`missingMetadata` degrade was reviewed
against `@axiom/topology`'s actual multi-repo model
(`roleCodeRepositories: readonly RepoRef[]`, `RepoRoleKey` deliberately
open-ended for this exact reason) and confirmed to be the correct,
safer choice over guessing a default among genuinely ambiguous
candidates — kept unchanged.

`mandatory.rules`/`mandatory.commands` being empty was previously
documented only in source comments, invisible to an MCP client reading
the JSON response. Closed this gap with one minimal, additive,
optional `IndexSummary.unsupported` field (`true` only on the two
permanently-empty `commands`/`rules` index summaries), plus one new
test. This is the only code change made during this review.

Content-inlining file reads confirmed to degrade gracefully (never
throw) on a missing/unreadable `README.md`, traced directly in
`readArtifactBody`/`inlineContent`.

All validation independently re-run and confirmed: `typecheck`/`build`
clean; full monorepo test suite at 1492/1505 passing (was 1491/1504
before this review's one added test), 12 pre-existing failing files
byte-identical to the documented baseline, zero regressions;
`packages/mcp-tools` scoped suite at 52/52 passing (was 51/51 before
this review's one added test).

## General spec integration

Integrated into `Axiom.Spec/general-spec.md`'s "MCP tool registration"
section: updated capability-id count from 16 to 17, added
`spec.implementationContextRead` to the id list with a one-line summary
of its composite nature and response shape, documented the
`relatedSpec` placeholder limitation, the `contextBudget` 3-tier
content-inlining behavior, the `repositories.target` ambiguity-resolution
rule, and the `indexes.commands`/`indexes.rules` `unsupported` marker.
This records confirmed, working fact now that validator-reviewer has
closed — consistent with the audit's and implementer's own stated
precedent of not recording it as stable fact before this review.

## INC-14 closure summary (as a whole)

INC-14 (`get_implementation_context`, flagship MCP tool) is **closed**.
All three roles in its sequence are complete:

1. **migration-engineer** (`INC-20260702-get-implementation-context-reconcile`):
   audited the 16 existing `@axiom/mcp-tools` handlers, field-mapped
   source doc §8.3's response shape, resolved the `relatedSpec`
   placeholder-vs-co-dependency question, and designed `contextBudget`'s
   3-tier behavior. Closed, audit-only.
2. **mcp-tool-implementer** (`INC-20260702-get-implementation-context-reconcile-impl`):
   implemented the 17th capability id (`spec.implementationContextRead`),
   composing the 16 existing handlers with zero new domain-package
   dependencies, per the audit's field-by-field table. All builds/
   typechecks/tests passed at implementation time (1491/1504, +10 new
   tests, no regressions).
3. **validator-reviewer** (this increment): independently re-traced the
   composition, the `confidence`/`missingMetadata` heuristic, the
   `contextBudget` tiers, the `repositories.target` ambiguity rule
   against real topology evidence, and the `mandatory.rules`/`commands`
   visibility question; found and closed one real (if minor) gap
   (`IndexSummary.unsupported`); independently re-ran and confirmed all
   validation with zero regressions.

All of INC-14's acceptance criteria (across all three constituent specs)
are satisfied: goal is clear, acceptance criteria existed and are met,
changes are implemented, available validation was executed and
independently re-confirmed, review against intent/acceptance criteria
was completed, and stable knowledge was integrated into
`Axiom.Spec/general-spec.md`.

**Deferred items, named explicitly** (unchanged from the audit's and
implementer's own non-goals, carried forward as INC-13b candidates, not
part of INC-14's scope):

- `get_workspace_equivalent`
- `get_related_specs` (full implementation — `get_implementation_context`
  ships only the narrower `relatedSpec` placeholder via
  `plan.links.incrementId` -> `getIncrement`, not the full search/ranking
  tool)
- `list_recommended_skills`
- `get_technical_context_document`
- An any-tag "recommended" variant of `list_recommended_context` (the
  currently-registered tool reuses `resolveMandatoryDocuments`'s
  ALL-tags-present semantics, not a true any-tag matcher)
- A "rules"/"commands" backing artifact kind (no product decision yet on
  whether these should exist as a first-class artifact type at all —
  `mandatory.rules`/`mandatory.commands` stay documented-empty/
  `unsupported: true` until such a decision is made)

None of these block INC-14's own closure; none were silently dropped —
each is named here and was already named in the audit's and
implementer's non-goals sections.

## Next step recommendation

INC-14 is closed. Recommended next step: **INC-15, "Project-scoped MCP
isolation + per-adapter MCP config"**, per the parent roadmap
(`Axiom.Spec/specs/increments/INC-20260702-axiom-redesign-roadmap/
README.md`, Phase E). If a future increment revisits any of INC-14's
deferred items (most likely `get_related_specs` given it is the one
`get_implementation_context` already leans on via placeholder), it
should re-read this increment's "relatedSpec placeholder decision"
section (in the audit spec) before designing the full tool, to preserve
the placeholder's existing behavior for callers already depending on it.
