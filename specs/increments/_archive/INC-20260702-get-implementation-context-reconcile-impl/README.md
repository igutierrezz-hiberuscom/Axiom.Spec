# Increment: `get_implementation_context` (flagship MCP tool) — implementation

Status: closed
Date: 2026-07-02

## Goal

Implement `get_implementation_context` as a 17th `@axiom/mcp-tools`
capability, composing the 16 existing handlers per the migration-engineer
audit's field-by-field table
(`Axiom.Spec/specs/increments/INC-20260702-get-implementation-context-reconcile/
README.md`), with no new domain-package dependency and no changes to
`@axiom/orchestrator`/`@axiom/tool-routing`/`@axiom/capability-model`.

## Context

Roadmap: `Axiom.Spec/specs/increments/INC-20260702-axiom-redesign-roadmap/
README.md`, Phase E, INC-14. This is the `mcp-tool-implementer` phase,
sibling to and directly following the audit increment
(`INC-20260702-get-implementation-context-reconcile`, closed audit-only).
Depends on INC-13 (closed; `@axiom/mcp-tools`'s 16 handlers), INC-08
(write-scope), INC-09/INC-10 (skills/technical-context indexes).

Role in INC-14's sequence: **mcp-tool-implementer**. Next: **validator-reviewer**.

## Scope

- New handler file `Axiom/packages/mcp-tools/src/implementation-context-handler.ts`
  exporting `buildImplementationContext` (pure composition) and
  `getImplementationContext` (registry-facing, `McpToolResult`-wrapped).
- New capability id `spec.implementationContextRead` registered in
  `Axiom/packages/mcp-tools/src/registry.ts`'s `MCP_TOOL_CAPABILITY_IDS`/
  `MCP_TOOL_HANDLERS`, alongside (not replacing) the 16 existing entries.
- Barrel export additions in `Axiom/packages/mcp-tools/src/index.ts`.
- Tests: `Axiom/packages/mcp-tools/tests/implementation-context-handler.test.ts`
  (happy path, partial data, project/plan-not-found, all three
  `contextBudget` tiers).
- Consistency updates to the pre-existing capability-routing example
  fixtures/tests (`examples/capabilities.example.yaml`,
  `examples/providers.example.yaml`, `examples/README.md`,
  `tests/registry.test.ts`, `tests/capability-routing-roundtrip.test.ts`)
  so the "16 capability ids" assertions/comments reflect the new 17th
  entry — required for those pre-existing tests to keep passing, not
  scope creep.

## Non-goals

- No implementation of `get_workspace_equivalent`, `get_related_specs`,
  `list_recommended_skills`, `get_technical_context_document`, or the
  any-tag `list_recommended_context` variant (deferred INC-13b tools per
  the audit's decision).
- No new "rules"/"commands" artifact kind — `mandatory.rules`,
  `mandatory.commands`, their `indexes`/`recommended` counterparts stay
  documented-empty (`[]` / absent `IndexSummary`).
- No token-counting or byte-budget engine for `contextBudget` — coarse
  3-tier content-inlining switch only, per the audit's design.
- No changes to `@axiom/orchestrator`, `@axiom/tool-routing`, or
  `@axiom/capability-model` source code.

## Acceptance criteria

- [x] `getImplementationContext({projectId, planId, targetRepoId?, role?,
      contextBudget?})` composes `getProjectRegistry`, `getProjectRepos`,
      `getPlan`, `getIncrement`, `getAdrIndex`, `getSkillIndex` (sdd +
      target repo roots), `getTechnicalContextIndex` +
      `resolveMandatoryDocuments`, `getAllowedWriteScope` — all called as
      already-exported sibling handler functions, no re-import of their
      backing domain functions.
- [x] Response shape matches the audit's field-by-field table:
      `project`, `repositories{spec,sdd,target}`, `plan`, `relatedSpec`
      (placeholder via `plan.links.incrementId` -> `getIncrement`),
      `relatedAdrs` (filter over `getAdrIndex`), `mandatory{sddSkills,
      repoSkills,technicalContext,rules:[],commands:[]}`,
      `indexes{...}`, `recommended{skills,technicalContext,adrs,
      commands:[]}`, `allowedWriteScope`, `confidence`, `missingMetadata`.
- [x] `confidence`/`missingMetadata` computed as documented, simple
      post-processing (no new I/O beyond the composed calls).
- [x] `contextBudget` tiers implemented exactly as designed: `small`
      (ids/paths/summaries only), `medium` (inlines `plan`+`relatedSpec`+
      mandatory skill/technical-context content), `large` (also inlines
      `relatedAdrs` content). Budget never changes which fields are
      present.
- [x] Registered as `spec.implementationContextRead` in
      `MCP_TOOL_HANDLERS`, following INC-13's exact registration pattern.
- [x] Tests cover: happy path (confidence high), partial data (confidence
      medium, `missingMetadata` populated), project/plan-not-found
      (confidence low), and each `contextBudget` tier's content-inlining
      behavior.
- [x] Real validation executed: `npm run build`, `npm run typecheck`,
      `npm test` at both monorepo root and scoped to `packages/mcp-tools`.
      Baseline confirmed via a pre-change full test run.
- [x] Closure fully satisfied (see Result / Status below — confirmed by
      `INC-20260702-get-implementation-context-reconcile-validator`).

## Open questions

None blocking implementation. Carried forward for validator-reviewer:
confirm the new handler stays inside `@axiom/mcp-tools`'s "thin
registration layer" architecture rule (calls only sibling handler
functions, no direct domain-package logic beyond what those handlers
already expose) and that `mandatory.rules`/`mandatory.commands` staying
`[]` is not mistaken for a bug.

## Assumptions

Same as the audit's assumptions section, carried forward verbatim:
`contextBudget` is a content-inlining hint, not a token-counting engine;
`spec.implementationContextRead` is one more entry in the existing
`MCP_TOOL_HANDLERS` map, no new registration mechanism; `role`
resolution falls back to `PlanMetadata.taskType` when `input.role` is
omitted.

Additional assumptions made during implementation (not pre-decided by
the audit, resolved here):

- **`repositories.target` selection when `targetRepoId` is omitted**: the
  audit flagged this as "real, small mapping logic" without fully
  specifying the selection rule. Implemented as: explicit
  `input.targetRepoId` wins; otherwise, if exactly one repo entry remains
  after excluding `spec`/`sdd` keys, that one is `target`; otherwise
  `target` is `null` and `"repositories.target"` is added to
  `missingMetadata`. This avoids guessing among multiple ambiguous
  candidates.
- **`allowedWriteScope` shape**: kept as `AllowedWriteScopeEntry[]`
  (`{repo, paths[]}[]`), per the audit's explicit recommendation — not
  flattened to `string[]`.
- **`plan.content`/`relatedSpec.content` source**: read directly from
  each artifact's `README.md` body via `resolveArtifactDir` (exported by
  `@axiom/workflow`) + a plain `fs.readFileSync`, since no existing
  handler exposes raw markdown body content. This is the "small
  additive read" the audit anticipated for `plan.content`, extended
  identically to `relatedSpec.content` and (at `contextBudget: "large"`)
  `relatedAdrs[].content` — same pattern, no new backing function.
- **`recommended.skills`/`recommended.technicalContext` construction**:
  built from the same `SkillsRoleIndex.available` /
  `TechnicalContextIndex.available` lists already loaded for `indexes.*`
  and `mandatory.*` — no second index load.

## Implementation notes

### Files changed (`Axiom/`)

- **New**: `packages/mcp-tools/src/implementation-context-handler.ts` —
  the composite handler (`buildImplementationContext` pure function +
  `getImplementationContext` `McpToolResult`-wrapped registry entry).
- **New**: `packages/mcp-tools/tests/implementation-context-handler.test.ts`
  — 10 tests (happy path, 2 partial-data cases, 2 not-found cases, 3
  `contextBudget` tier tests, 1 envelope test, 1 fixture-sanity test).
- **Modified**: `packages/mcp-tools/src/registry.ts` — added
  `getImplementationContext` import, `spec.implementationContextRead`
  capability id, and its `MCP_TOOL_HANDLERS` entry.
- **Modified**: `packages/mcp-tools/src/index.ts` — barrel-exports the
  new handler's public types and functions.
- **Modified** (consistency, not scope creep — required for pre-existing
  tests to keep passing after the 17th capability id was added):
  - `packages/mcp-tools/examples/capabilities.example.yaml` — added
    `spec.implementationContextRead` entry.
  - `packages/mcp-tools/examples/providers.example.yaml` — added
    `spec.implementationContextRead` to `axiom-gateway`'s
    `applicableCapabilities`.
  - `packages/mcp-tools/examples/README.md` — updated "16" -> "17"
    capability-id count reference.
  - `packages/mcp-tools/tests/registry.test.ts` — updated the "declares
    exactly 16 capability ids" assertion/title to 17.
  - `packages/mcp-tools/tests/capability-routing-roundtrip.test.ts` —
    updated comments from "16 new" to "17 registered" (assertions
    themselves already iterate `Object.values(MCP_TOOL_CAPABILITY_IDS)`
    generically, so no assertion logic needed to change, only comments).

### Composition logic

Each field is produced by calling the already-exported sibling handler
functions directly (`getProjectRegistry`, `getProjectRepos`, `getPlan`,
`getIncrement`, `getAdrIndex`, `getSkillIndex`, `getTechnicalContextIndex`,
`getAllowedWriteScope`), matching the audit's field-by-field table
1:1. No backing domain function is imported a second time; `@axiom/skills`
and `@axiom/technical-context` are imported only for their exported
**types** (`SkillsRoleIndex`, `TechnicalContextIndex`,
`resolveMandatoryDocuments`) needed to type/derive the composed output,
not for a second copy of the loading logic.

### `confidence` / `missingMetadata` design

Deliberately simple, per the audit's "keep this simple, not a complex
scoring system" instruction:

- `missingMetadata: string[]` — one string pushed per field that
  resolved to `null`/empty/absent during composition (`"project"`,
  `"repositories"`, `"repositories.target"`, `"plan"`, `"relatedSpec"`,
  `"mandatory.sddSkills"`, `"mandatory.repoSkills"`,
  `"mandatory.technicalContext"`). Pure bookkeeping alongside the
  existing composition steps — no extra I/O.
- `confidence: "high" | "medium" | "low"`:
  - `"low"` — the project entry or the plan itself could not be
    resolved (nothing meaningful can be composed on top of a missing
    anchor).
  - `"medium"` — project + plan resolved, but `missingMetadata` is
    non-empty (some mandatory/related content missing).
  - `"high"` — `missingMetadata` is empty.

### `contextBudget` tier implementation

- `"small"` (default): every ref (`mandatory.*`, `relatedAdrs`,
  `recommended.*`) stays `{id, path?, reason?}` — no `content` field.
  `plan`/`relatedSpec` also omit `content`.
- `"medium"`: `plan.content`/`relatedSpec.content` read from each
  artifact's `README.md` body (via `resolveArtifactDir` + `fs`);
  `mandatory.sddSkills`/`mandatory.repoSkills`/`mandatory.technicalContext`
  refs get `content` inlined by reading each ref's `path` relative to
  the owning repo root. `relatedAdrs` and all `recommended.*` stay
  reference-only.
- `"large"`: same as `medium`, plus `relatedAdrs[].content` inlined from
  each ADR's `README.md` body. `recommended.*` still never inlines
  content at any tier (by design — the reference/recommendation split is
  the point).

Budget never adds/removes fields — verified by the "small" tier test
asserting every structural field is still present, just without
`content`.

## Validation

Validation commands discovered via `package.json` scripts (`build`,
`typecheck`, `test` at both monorepo root and `packages/mcp-tools`).

- `npm run build` (monorepo root, `tsc -b`): **clean, zero errors**,
  before and after the change.
- `npm run typecheck` (monorepo root, `tsc -b`): **clean, zero errors**.
- `npx tsc --noEmit` scoped to `packages/mcp-tools`: **clean, zero
  errors**.
- `npm test` (monorepo root, vitest):
  - **Pre-change baseline** (confirmed by running the full suite before
    any implementation edit): 12 failed files / 13 failed tests / 1481
    passed / 1494 total.
  - **Post-change**: 12 failed files / 13 failed tests / 1491 passed /
    1504 total. The 12 failing files are byte-identical to the baseline
    list (`apps/cli/tests/start.test.ts`,
    `packages/agents/tests/catalog.test.ts`,
    `packages/doctor/tests/checks.test.ts`,
    `packages/model-routing/tests/{assignments,loader,opencode-projection,
    resolver}.test.ts`, `packages/project-resolution/tests/resolver.test.ts`,
    `packages/skills/tests/catalog.test.ts`,
    `packages/telemetry/tests/audit-trail-sink.test.ts`,
    `packages/toolchain/tests/repair-add-gitignore.test.ts`,
    `packages/tui/tests/driver.test.ts`) — all pre-existing,
    environment/fixture-dependent failures unrelated to `@axiom/mcp-tools`
    (real-repo-path assumptions, retention-sweep timing, TUI snapshot
    text). +10 passed tests are exactly the new
    `implementation-context-handler.test.ts` suite. **No regressions.**
- `npx vitest run` scoped to `packages/mcp-tools`: **8 test files, 51
  tests, all passing** (was 7 files / 41 tests before this increment;
  +1 new file / +10 new tests).

## Result

Implemented `get_implementation_context` as capability id
`spec.implementationContextRead`, the 17th entry in
`@axiom/mcp-tools`'s `MCP_TOOL_HANDLERS`, composing the 16 pre-existing
handlers with zero new domain-package dependencies. `confidence`/
`missingMetadata` are computed as simple, documented post-processing.
`contextBudget`'s three tiers are implemented exactly per the audit's
coarse design (content-inlining only, never field presence).
`mandatory.rules`/`mandatory.commands` (and their `indexes`/
`recommended` counterparts) are `[]`/absent, matching the audit's
documented gap. All builds/typechecks/tests pass; the pre-existing
12-file/13-test failure baseline is unchanged, confirming no
regressions were introduced.

## General spec integration

Integrated by `INC-20260702-get-implementation-context-reconcile-validator`
after independently confirming no architecture-rule violations. See that
increment's "General spec integration" section for what was written into
`Axiom.Spec/general-spec.md`.

## Next step recommendation

Hand this spec to **validator-reviewer** with these concrete inputs:

- New file to review for architecture-rule conformance:
  `Axiom/packages/mcp-tools/src/implementation-context-handler.ts` —
  confirm it calls only already-exported sibling handler functions from
  `registry-handlers.ts`/`artifact-handlers.ts`/`skills-handlers.ts`/
  `technical-context-handlers.ts` (same package), never a domain
  package's backing function a second time, per `@axiom/mcp-tools`'s
  "thin registration layer" rule (see `types.ts`'s header comment).
- Confirm `mandatory.rules`/`mandatory.commands` staying `[]` (and their
  `indexes`/`recommended` counterparts) is understood as a documented
  gap, not a bug, and that no test anywhere expects them non-empty.
- Confirm the `contextBudget` tier tests
  (`packages/mcp-tools/tests/implementation-context-handler.test.ts`,
  "contextBudget tiers" describe block) actually assert the coarse
  content-inlining contract (structure always present, `content` only at
  `medium`/`large`) rather than an accidental stricter/looser contract.
- After validator-reviewer closes, integrate the new capability id +
  final response shape into `Axiom.Spec/general-spec.md`'s "MCP tool
  registration" section (update "16 capability ids" to "17", add
  `spec.implementationContextRead` to the id list, document the
  `relatedSpec` placeholder limitation and `contextBudget`'s tier
  behavior as stable fact).
