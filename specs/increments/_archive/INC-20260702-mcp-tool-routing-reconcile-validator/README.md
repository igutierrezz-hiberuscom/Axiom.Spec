# Increment: MCP tool-routing reconcile — validation and closure (INC-13, Phase E, validator-reviewer)

Status: closed
Date: 2026-07-02

## Goal

Independently verify `mcp-tool-implementer`'s implementation
(`INC-20260702-mcp-tool-routing-reconcile-impl/README.md`) against the
migration-engineer audit's claims
(`INC-20260702-mcp-tool-routing-reconcile/README.md`), decide the open
`TR-005`+ doctor-extension question left by the implementer, run real
validation independently, and close INC-13 as a whole if all of its
acceptance criteria are met.

## Context

Roadmap: `Axiom.Spec/specs/increments/INC-20260702-axiom-redesign-roadmap/
README.md`, Phase E, INC-13. This is the third and final role in the
INC-13 chain:

1. migration-engineer audit (`-reconcile/README.md`) — audit only, no code.
2. mcp-tool-implementer (`-reconcile-impl/README.md`) — implemented
   `@axiom/mcp-tools`, 16 capability ids, `Status: pending` (explicitly
   deferring closure to this role).
3. This increment (validator-reviewer) — independent verification and
   INC-13-as-a-whole closure decision.

## Scope

- Independently trace at least 4 of the 16 MCP tool handlers in
  `packages/mcp-tools/src/*.ts` against their claimed backing functions,
  confirming genuine delegation (not reimplementation/stubs).
- Independently re-read `capability-model/src/schemas.ts`'s
  `ProviderRegistrySchema` and `constants.ts`'s `CANONICAL_PROVIDER_IDS`
  to confirm the `.length(7)` + closed-provider-id-set constraint is real,
  and that mapping `sdd`/`spec` onto `axiom-gateway` was a genuine
  constraint-driven decision, not just convenient.
- Confirm `TR-001..004` were not modified by INC-13 and still reflect
  their pre-existing contract.
- Verify the `@axiom/cli-commands` barrel extension (used by the 3
  CLI-backed tools) follows the same re-export pattern as the
  pre-existing `configure`/`sync`/`upgrade`/`model`/`components` entries,
  preserving the "TUI/consumers never import `apps/cli/src` directly"
  layering rule.
- Decide the implementer's open question: should a `TR-005`+ doctor check
  consume `@axiom/mcp-tools`'s example fixtures / round-trip test.
- Run `npm run typecheck`, `npm run build`, `npm test` for the full
  monorepo independently and compare against both the audit's stated
  baseline and the implementer's stated post-implementation numbers.
- Close INC-13 as a whole if acceptance criteria are met, naming deferred
  items explicitly.
- Integrate stable knowledge into `Axiom.Spec/general-spec.md`.

## Non-goals

- No new tool implementation. The 5 deferred "need new logic" tools
  (`get_workspace_equivalent`, `get_related_specs`,
  `list_recommended_skills`, `get_technical_context_document`, and the
  any-tag "recommended" variant) are not attempted here — confirmed as a
  candidate follow-up increment (INC-13b) instead.
- `get_implementation_context` (INC-14) is not attempted here.
- No modification to `@axiom/tool-routing`, `@axiom/capability-model`, or
  `@axiom/isolation` source — re-confirmed unmodified, not re-litigated.

## Acceptance criteria

- [x] At least 4 of the 16 handlers independently traced to their real
      backing functions with no reimplementation/stub found.
- [x] `ProviderRegistrySchema`'s `.length(7)` constraint and
      `CANONICAL_PROVIDER_IDS`'s closed-set/ADR-gated nature independently
      confirmed by direct source read, validating the `axiom-gateway`
      mapping decision was constraint-driven.
- [x] `TR-001..004` confirmed unmodified (no `TR-005` present in
      `checks.ts`; last commit touching the file predates INC-13's
      uncommitted work entirely).
- [x] `@axiom/cli-commands` barrel extension confirmed to follow the
      pre-existing re-export pattern, no layering violation.
- [x] `TR-005`+ decision made explicitly (deferred, with rationale) —
      see "TR-005 decision" below.
- [x] `npm run typecheck`, `npm run build`, `npm test` run independently;
      results compared against stated baselines with any delta explained.
- [x] INC-13-as-a-whole closure assessed against its own roadmap entry,
      with deferred items named explicitly.
- [x] `Axiom.Spec/general-spec.md` updated with the MCP tool registration
      pattern and the Q4 resolution.

## Open questions

None blocking closure. The one open item inherited from the prior
increment (`TR-005`+ scope) is resolved below (deferred).

## Assumptions

- The workspace has no baseline commit isolating "before INC-13" from
  "before INC-01" — `Axiom`'s last commit (`4426241`) predates the entire
  INC-01..13 sequence, and all of INC-01 through INC-13's work is
  uncommitted working-tree state. `git diff`/`git log` were used only to
  confirm `packages/doctor/src/checks.ts`'s last real commit predates any
  of this sequence's work (i.e., it was not touched by any increment in
  this chain, including INC-13) and that `packages/mcp-tools` has zero
  commits (pure new working-tree addition) — not to isolate INC-13's diff
  specifically from INC-01..12's.

## Implementation notes (independent verification findings)

### 1. Handler-to-backing-function trace (4 of 16, plus spot checks on the rest)

Read directly: `packages/mcp-tools/src/registry.ts`,
`artifact-handlers.ts`, `cli-backed-handlers.ts`, `registry-handlers.ts`,
`skills-handlers.ts`, `technical-context-handlers.ts`.

- **`getPlan`** (`spec.planRead`) -> `artifact-handlers.ts`'s shared
  `readArtifact('plan', input)` -> `loadArtifactMetadata(projectRoot,
  'plan', id, specRelPath)`, imported from `@axiom/workflow`. Confirmed
  `loadArtifactMetadata` is a real exported function in
  `packages/workflow/src/artifact-store.ts` (line 511) with the exact
  signature the handler calls. No reimplementation — the handler is a
  thin translation of the `Result` shape into `McpToolResult`, nothing
  more.
- **`getAdrIndex`** (`spec.adrIndexRead`) -> `listArtifacts(projectRoot,
  'adr', specRelPath)`, imported from `@axiom/workflow`. Confirmed
  `listArtifacts` is real (`artifact-store.ts` line 631), scans the ADR
  folder and returns `{ entries, failures }`; the handler returns
  `result.entries` verbatim. No new logic.
- **`rebuildIndexes`** (`sdd.indexesRebuild`) -> `runIndexRebuild(args)`,
  imported from `@axiom/cli-commands`. Confirmed `runIndexRebuild` is a
  real exported function in `apps/cli/src/commands/index-cmd.ts` (line
  97), re-exported verbatim by the barrel. The handler is a one-line
  wrapper (`{ ok: true, data: runIndexRebuild(input) }`) — genuinely
  trivial, not a stub (it calls the real command logic; the "always
  ok:true" is documented and matches the command's own contract of
  reporting failures via `data`, not a thrown/rejected result).
- **`validateChanges`** (`sdd.changesValidate`) -> `runValidateChanges(args)`,
  imported from `@axiom/cli-commands`, itself backed by
  `validateWriteScope` (`@axiom/workflow/write-scope.ts`, the INC-08
  write-scope validator). Confirmed `runValidateChanges` is real
  (`apps/cli/src/commands/validate-changes.ts` line 80) and its body
  calls `validateWriteScope` (line 111) — the same primitive INC-08
  built, not a reimplementation.
- Spot-checked the remaining 12: `getProjectRegistry`/`getProjectRepos`
  call `listProjectsV2`/`getProjectV2` from `@axiom/user-workspace`
  (confirmed exported); `getIncrement`/`getBug`/`getDecisions`/
  `getAllowedWriteScope` share the same `artifact-store.ts` backing as
  `getPlan`/`getAdrIndex`; `getSkillIndex`/`getSkillCatalog`/`getSkill`
  call `loadSkillsRoleIndex`/`loadSkillsCatalog`/`getSkillById` from
  `@axiom/skills`; `getTechnicalContextIndex`/`listRecommendedContext`
  call `loadTechnicalContextIndex`/`resolveMandatoryDocuments` from
  `@axiom/technical-context`; `validateIndexes` calls `runIndexValidate`
  from the same `index-cmd.ts` file as `rebuildIndexes`. All follow the
  identical "thin translation of a real function's result into
  `McpToolResult`" shape — no handler reimplements or stubs its backing
  function.

### 2. `ProviderRegistrySchema` / `CANONICAL_PROVIDER_IDS` — independently confirmed

Read directly: `packages/capability-model/src/schemas.ts` (lines 65-124),
`packages/capability-model/src/constants.ts` (lines 1-45).

- `ProviderRegistrySchema.providers` is `z.array(ProviderDefinitionSchema)
  .length(7, { message: 'Registry must contain exactly 7 providers' })` —
  confirmed verbatim, not paraphrased.
- `ProviderDefinitionSchema` `.refine()`s `CANONICAL_PROVIDER_IDS.includes(val.id)`
  with message `'Provider id must be one of the 7 canonical ids'`.
- `constants.ts`'s `CANONICAL_PROVIDER_IDS` is exactly 7 strings
  (`filesystem`, `axiom-gateway`, `serena`, `codegraph`, `graphify`,
  `engram`, `generated-snapshots`), under a file-level comment: "These
  sets are HARD-CODED and can only change via ADR."
- This independently confirms the implementer's decision rationale is
  real, not asserted: minting `sdd`/`spec` as two new literal provider
  ids would require growing `CANONICAL_PROVIDER_IDS` from 7 to 9 (an
  ADR-gated change per the file's own header) AND changing the schema's
  `.length(7)` to `.length(9)`. Mapping onto the existing `axiom-gateway`
  id (`kind: project-scoped-broker`) avoids both changes and is a pure
  data addition (`applicableCapabilities` growing). The genuinely
  required, not merely convenient, framing holds up under independent
  re-read.
- The round-trip test
  (`packages/mcp-tools/tests/capability-routing-roundtrip.test.ts`) was
  re-run independently (see "Validation" below) and confirmed to load
  the example YAML through the real `loadCapabilityModel`, then resolve
  all 16 new capability ids through the real `routeTool`, non-degraded,
  with `providerEffective: 'axiom-gateway'` — this is not a unit test
  against a mock; it exercises the actual, unmodified pipeline.

### 3. `TR-001..004` unmodified — confirmed

- `git log --oneline -1 -- packages/doctor/src/checks.ts` returns
  `4ba8abe feat: implement registry v2 with YAML support and migration
  handling`, a commit that predates the entire INC-01..13 uncommitted
  working-tree sequence (the repository's most recent real commit,
  `4426241`, is even later but touches unrelated files; `checks.ts`
  itself has had no commit since `4ba8abe`, i.e. no increment in this
  workspace's session — including INC-13 — has committed a change to it).
- `git log --all --oneline -- packages/mcp-tools` returns nothing —
  `@axiom/mcp-tools` has zero commits; it is pure uncommitted new
  working-tree content, consistent with "new package, not a modification
  to any existing one."
- Direct read of `checks.ts` lines 859-1094 confirms exactly `TR-001`
  through `TR-004` exist (no `TR-005` present), matching both the
  audit's and the implementer's description of the check family:
  `require()`-based provisional loading, in-memory
  `buildSmokeContext` fixture exercising only `code.semanticNavigation`,
  zero references to `@axiom/mcp-tools`, `capabilities.yaml`, or
  `providers.yaml`. The checks were not touched, and their existing
  contract has no dependency on the new capability ids — confirming the
  audit's "extend, do not replace" framing was accurate.

### 4. `@axiom/cli-commands` barrel extension — layering rule preserved

Read directly: `packages/cli-commands/src/index.ts` in full.

- The file's own header comments (AD-01 through AD-04) establish the
  barrel's rule: "el barrel NO añade lógica" (trivial re-export only),
  built so `@axiom/tui` never imports `apps/cli/src/...` directly.
- The pre-existing entries (`runConfigure`, `runSync`, `runUpgrade`,
  `runModelValidate`, `runComponentsShow`) are all
  `export { runX } from '../../../apps/cli/src/commands/x'` plus a
  parallel `export type { ... }` block.
- The new entries added by this increment
  (`runIndexRebuild`/`runIndexValidate` from `index-cmd.ts`,
  `runValidateChanges` from `validate-changes.ts`) are byte-for-byte the
  same shape: `export { ... } from '../../../apps/cli/src/commands/...'`
  plus a matching `export type { ... }` block, under a new comment
  section explicitly citing "Same seam as
  `configure|sync|upgrade|model|components` above (AD-01)". No new
  mechanism was introduced; `@axiom/mcp-tools` depends on
  `@axiom/cli-commands` for these three tools and does not import
  `apps/cli/src/...` anywhere (confirmed by reading
  `cli-backed-handlers.ts`'s import block, which imports only from
  `@axiom/cli-commands`). The INC-04 layering rule is preserved, not
  bypassed.

### 5. TR-005 decision: deferred

**Decision: do not add a `TR-005`+ doctor check in this increment. Defer
it, with explicit rationale below.**

Considered concretely: a `TR-005` that requires `@axiom/mcp-tools` as a
new `packages/doctor/package.json` dependency, loads
`packages/mcp-tools/examples/{capabilities,providers}.example.yaml` (or
imports the round-trip test's fixture-building logic), and asserts the
16 new capability ids resolve through `routeTool` — mirroring
`TR-001..004`'s `require()`-based, in-memory, skip-cleanly pattern.

Why this does not qualify as "small, clearly-scoped, consistent with
`TR-001..004`'s existing pattern" on closer inspection:

- `TR-001..004` smoke-test packages `@axiom/doctor` **already depends on
  for real runtime behavior** (`@axiom/tool-routing` is a genuine
  `dependencies` entry, loaded provisionally because the doctor's own
  `runAllChecks` pipeline needs to reason about tool-routing health for
  real projects). `@axiom/mcp-tools` is not consumed by any real runtime
  path in this monorepo yet — confirmed by the implementer's own
  `examples/README.md`: "These files are examples, not consumed by any
  code in this monorepo," and no installer/scaffold writes
  `capabilities.yaml`/`providers.yaml` to any real project
  (`@axiom/document-bootstrap` only reads `providers[*].id` for
  `AGENTS.md` template variables). Adding a doctor check for a package
  with no live production call site inverts `TR-001..004`'s own
  justification (checking something the doctor's real pipeline depends
  on) into checking something nothing yet depends on.
- It would duplicate, not extend, existing coverage: the exact assertion
  a `TR-005` would make (all 16 ids load through `loadCapabilityModel`
  and resolve non-degraded through `routeTool` via `axiom-gateway`) is
  already made, in full, by
  `packages/mcp-tools/tests/capability-routing-roundtrip.test.ts` — reindependently re-run in this increment's validation and confirmed
  passing. Re-running the same assertion inside `@axiom/doctor` adds a
  new package dependency edge (`doctor -> mcp-tools`) for zero new
  signal.
- It would introduce a real, if small, architectural coupling
  (`@axiom/doctor` depending on `@axiom/mcp-tools`) that the roadmap and
  `Axiom.SDD/AGENTS.md`'s "no speculative architecture" / "no scripts
  that don't exist yet" bootstrap limits caution against, given the
  underlying YAML files this check would load are themselves explicitly
  documented as inert examples with no real consumer.

Correct trigger to revisit: once a real project scaffold or installer
path actually writes `capabilities.yaml`/`providers.yaml` content
including these 16 ids to a live `axiom.spec/config/` directory (i.e.,
once `@axiom/mcp-tools`'s registration stops being example-only), a
`TR-005`+ smoke-testing that real file against `routeTool` would become
directly analogous to `TR-001..004`'s justification and should be added
then — not before. This is noted as a candidate item for whichever
future increment first wires a real MCP server transport or scaffold
(the same "future work" boundary the implementer's own non-goals already
drew).

### 6. Independent test/build/typecheck validation

Ran directly, from a clean invocation (not copied from prior increments'
output):

- `npx vitest run packages/mcp-tools` (scoped): **7 test files, 41 tests,
  all passing**, including
  `capability-routing-roundtrip.test.ts`'s single end-to-end test.
- `npm run typecheck` (full monorepo, `tsc -b`): clean, zero errors.
- `npm run build` (full monorepo, `tsc -b`): clean, zero errors.
- `npm test` (full monorepo, `vitest run`): **12 failed files / 13 failed
  tests / 1481 passed (out of 1494 total)** — independently reproduces
  the implementer's stated post-implementation numbers exactly, and the
  delta against the audit's stated baseline (12 failed / 13 failed /
  1440 passed) is exactly +41 passed, matching `@axiom/mcp-tools`'s test
  count with zero new failures.
- The 12 failing files were enumerated directly from this run's output
  and confirmed to be the same pre-existing set named in the
  implementer's spec (`apps/cli/tests/start.test.ts`,
  `packages/agents/tests/catalog.test.ts`,
  `packages/doctor/tests/checks.test.ts`,
  `packages/model-routing/tests/{assignments,loader,opencode-projection,resolver}.test.ts`,
  `packages/project-resolution/tests/resolver.test.ts`,
  `packages/skills/tests/catalog.test.ts`,
  `packages/telemetry/tests/audit-trail-sink.test.ts`,
  `packages/toolchain/tests/repair-add-gitignore.test.ts`,
  `packages/tui/tests/driver.test.ts`) — none touch `mcp-tools`,
  `tool-routing`, `capability-model`, `isolation`, `cli-commands`, or
  `doctor`'s tool-routing section; all are tied to missing real
  `axiom.spec/` config at the monorepo root or environment/snapshot
  flakiness, pre-existing and unrelated to INC-13.
- Zero regressions confirmed independently, not merely re-asserted from
  the implementer's report.

## Validation

Commands run (package scripts, discovered via `package.json`):

- `npx vitest run packages/mcp-tools` — 7 files / 41 tests passed.
- `npm run typecheck` — clean.
- `npm run build` — clean.
- `npm test` — 12 failed files / 13 failed tests / 1481 passed / 1494
  total, zero regressions against the audit's stated baseline.

## Result

Independently verified `mcp-tool-implementer`'s implementation is
accurate and non-speculative: all 16 handlers genuinely delegate to real,
pre-existing backing functions (4 traced in full depth, 12 spot-checked);
the `sdd`/`spec` -> `axiom-gateway` provider-id mapping is grounded in a
real, independently-confirmed `ProviderRegistrySchema`/
`CANONICAL_PROVIDER_IDS` constraint, not convenience; `TR-001..004` are
confirmed untouched (no `TR-005` exists, no commit touches `checks.ts`
within this increment chain); the `@axiom/cli-commands` barrel extension
is byte-for-byte consistent with the pre-existing re-export pattern,
preserving the INC-04 layering rule. The `TR-005`+ open question is
resolved: deferred, because the underlying YAML files it would check are
explicitly inert examples with no real production consumer today, making
a new `doctor -> mcp-tools` dependency edge premature and duplicative of
existing round-trip test coverage. Full monorepo `typecheck`/`build` are
clean; `npm test` independently reproduces 1481 passed / 13 failed / 12
failed files, a net +41 over the audit's stated 1440-passed baseline with
zero regressions.

## General spec integration

Added a new "MCP tool registration" section to
`Axiom.Spec/general-spec.md` documenting: the `@axiom/mcp-tools` package
and its role as a handler-dispatch layer above `tool-routing`/
`capability-model`; the `sdd`/`spec` -> `axiom-gateway` provider-id
mapping precedent (reusable for any future capability needing an
`sdd`/`spec`-flavored provider); Q4's resolution (capability-model stays
completely orthogonal to the new decision documents; MCP tools are
additive `capabilityId`s, not a parallel system); and the current
16-capability-id registration list with a note on the 5 deferred tools.
This is now stable, working, independently-validated knowledge — the
correct point for integration per `Axiom.SDD/AGENTS.md`'s documentation
rules (a concrete, working contract has landed and been confirmed, not
merely proposed).

## INC-13 closure summary (whole increment, all 3 roles)

**Status: closed.**

All of INC-13's roadmap-level acceptance criteria are met:

- The existing `@axiom/tool-routing`/`@axiom/capability-model`/
  `@axiom/isolation` system was audited in full (migration-engineer),
  confirmed orthogonal to the new decision documents per Q4, and left
  completely unmodified (re-confirmed independently in this role).
- 16 of the MCP tool list's entries (15 "trivial" + 1 "mostly trivial"
  bonus) are implemented as new capability ids dispatched through the
  existing, unmodified `routeTool` mechanism, in a new package
  (`@axiom/mcp-tools`) that does not invert or pollute the existing
  packages' dependency graphs.
- The `sdd`/`spec` provider-id naming gap the audit surfaced is resolved
  with a concrete, schema-constraint-grounded decision (map onto
  `axiom-gateway`), independently re-verified in this role.
- `TR-001..004` are confirmed unmodified and still pass; the doctor
  extension question is explicitly decided (deferred, with rationale),
  not left open indefinitely.
- Real validation (`typecheck`/`build`/`test`) was run independently by
  this role and matches both prior roles' reported numbers, with zero
  regressions.
- Stable knowledge is integrated into `general-spec.md`.

**Deferred items, named explicitly (not silently dropped):**

1. `get_workspace_equivalent` — no exact backing function exists; needs
   new logic resolving a registry-wide "equivalent path" concept.
2. `get_related_specs` — no exact backing function exists; needs a new
   cross-artifact relationship traversal.
3. `list_recommended_skills` — no exact backing function exists; needs a
   small new function mirroring `resolveMandatoryDocuments`'s pattern for
   skills rather than technical-context documents.
4. `get_technical_context_document` — thin gap; needs a new function
   reading a technical-context document's content at a given path (the
   index loader only returns paths/metadata today).
5. `list_recommended_context`'s any-tag "recommended" variant — the
   implemented `listRecommendedContext` reuses `resolveMandatoryDocuments`
   (ALL-tags-present semantics); a true any-tag "recommended" matcher
   remains a small, separate gap.

**Recommendation**: these 5 items are a well-scoped candidate for a
follow-up increment, **INC-13b — MCP tool routing reconcile: deferred
new-logic tools**, to be sequenced independently of INC-14 (INC-14 does
not require any of these 5; per the audit's own dependency note, it needs
rows 1, 2, 5, 8, 9, 11, 15, 17, 18 — row 8 (`get_related_specs`) is the
only deferred-tool overlap, and INC-14's scope can proceed without it by
treating "related specs" as an explicit gap in its own composite response
until INC-13b lands).

## Next step recommendation

Proceed to **INC-14 — `get_implementation_context` (flagship MCP tool)**
per the roadmap's Phase E sequencing (`INC-20260702-axiom-redesign-roadmap/
README.md`, dependencies: INC-13 [closed], INC-08 [write-scope, closed],
INC-09/INC-10 [context/skill indexes, closed]). INC-14's composite
response can draw on 9 of this increment's 16 registered capability ids
(`getPlan`, `getIncrement`/`getBug`, `getAdrIndex`, `getSkillIndex`,
`getTechnicalContextIndex`, `listRecommendedContext`,
`getAllowedWriteScope`, plus `getProjectRegistry`/`getProjectRepos` for
project-scoping) as its first concrete building blocks, per the audit's
own dependency note (§6 of the audit spec). `get_related_specs` (row 8,
one of INC-13b's 5 deferred items) is the one INC-14-adjacent gap;
INC-14's own scoping should decide explicitly whether to treat it as a
placeholder/omitted field or to pull INC-13b's `get_related_specs` in as
a co-dependency, rather than silently assuming either.
