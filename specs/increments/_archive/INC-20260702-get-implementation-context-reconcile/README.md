# Increment: `get_implementation_context` (flagship MCP tool) — audit

Status: closed
Date: 2026-07-02

## Goal

Audit whether `get_implementation_context` (source doc
`axiom_decisiones_sesion_prompt_implementacion.md` §8) can be built as a
composition over the 16 MCP tools already registered by INC-13
(`@axiom/mcp-tools`), map §8.3's response shape field-by-field onto those
existing tools, identify exactly which fields are blocked on deferred
INC-13b tools, and decide a pragmatic unblocking path for the one
confirmed gap (`relatedSpec` / `get_related_specs`) so implementation
(`mcp-tool-implementer`) can proceed without waiting on INC-13b in full.
This increment is audit-only — no code is written.

## Context

Roadmap: `Axiom.Spec/specs/increments/INC-20260702-axiom-redesign-roadmap/
README.md`, Phase E, INC-14. Depends on INC-13 (closed;
`Axiom.Spec/general-spec.md`'s "MCP tool registration" section is the
canonical summary), INC-08 (write-scope, closed), INC-09/INC-10
(skills/technical-context indexes, closed).

Source doc sections re-read in full for this audit:
`C:\Users\igutierrezz\Downloads\axiom_decisiones_sesion_prompt_implementacion.md`
§8.1 (objective), §8.2 (input signature), §8.3 (response shape), §8.4
(14-step behavior).

INC-13's own validator explicitly flagged: *"`get_related_specs` is the
one gap INC-14 must explicitly address (placeholder vs. co-dependency)."*
This increment resolves that flag.

## Scope

- Re-read `Axiom/packages/mcp-tools/src/*` in full (all 7 files) to confirm
  the exact current signatures/return shapes of the 16 registered tools.
- Re-read `Axiom/packages/orchestrator/src/*` in full to confirm or refute
  the roadmap's claim that it has a "profile/capability/provider
  resolution pattern" worth reusing for a composite tool.
- Field-by-field mapping of source doc §8.3's response shape against the
  16 existing tools plus other already-existing backing functions
  (`@axiom/workflow`, `@axiom/topology`, `@axiom/user-workspace`).
- Decide the `relatedSpec` placeholder-vs-co-dependency question.
- Design (coarse, first-pass) what `contextBudget` (`small|medium|large`)
  actually controls.
- Produce the field-by-field composition table and a brief for
  `mcp-tool-implementer`.

## Non-goals

- No implementation of `get_implementation_context` itself in this
  increment (that is `mcp-tool-implementer`'s job, next in the sequence).
- No implementation of the full `get_related_specs` tool (deferred to
  INC-13b) — only a documented placeholder/best-effort decision for
  `relatedSpec` inside `get_implementation_context`'s response.
- No implementation of `get_workspace_equivalent`,
  `list_recommended_skills`, `get_technical_context_document`, or the
  any-tag `list_recommended_context` variant — confirmed non-blocking per
  INC-13's close, re-confirmed here.
- No changes to `@axiom/orchestrator`, `@axiom/tool-routing`, or
  `@axiom/capability-model`.

## Acceptance criteria

- [x] `Axiom/packages/mcp-tools/src/` read in full (7 files:
      `types.ts`, `registry.ts`, `registry-handlers.ts`,
      `artifact-handlers.ts`, `skills-handlers.ts`,
      `technical-context-handlers.ts`, `cli-backed-handlers.ts`).
- [x] `Axiom/packages/orchestrator/src/` read in full (5 files:
      `index.ts`, `types.ts`, `gates.ts`, `state-machine.ts`,
      `runner.ts`) and a explicit confirm/refute verdict recorded.
- [x] Every field in source doc §8.3's response shape is mapped to
      exactly one of: existing-tool composition, new composition logic,
      or blocked-on-deferred-tool, with the concrete existing
      function/type cited.
- [x] `relatedSpec`'s placeholder-vs-co-dependency question is answered
      with a concrete recommendation grounded in code already read.
- [x] `contextBudget`'s effect is designed at a first-pass, coarse level
      (no over-engineering).
- [x] A brief for `mcp-tool-implementer` is recorded (signature, response
      shape, composition table, budget rules, relatedSpec approach).
- [x] Recorded in `Axiom.Spec`; no code changed in `Axiom.SDD` or `Axiom`.

## Open questions

None blocking. One naming note carried into the brief: source doc §8.3's
`repositories.target: RepoRef` assumes a single fixed target repo, but
the actual registry (`ProjectEntryV2.repos: Record<RepoRoleKey,
RegistryRepoEntry>`, `RepoRoleKey = string`) is an open string-keyed map,
not a closed `{spec, sdd, target}` triple — see composition table row for
`repositories`.

## Assumptions

- `contextBudget` is a hint the handler uses to decide how much document
  *content* to inline vs. reference by id/path — not a token-counting or
  truncation engine. Kept coarse per `AGENTS.md`'s bootstrap limits (no
  speculative complex metadata systems).
- `get_implementation_context` will be registered as one more
  `capabilityId` (`spec.implementationContextRead` or similar,
  `mcp-tool-implementer`'s call) dispatched through the existing
  `MCP_TOOL_HANDLERS` map — no new registration mechanism.
- `role`/`taskType` resolution (§8.4 step 7) reuses `PlanMetadata.taskType`
  (already a field on `PlanMetadata`) when `input.role` is not given,
  rather than inventing a new resolution algorithm.

## Implementation notes (audit findings)

### 1. `@axiom/mcp-tools` — confirmed current shape (16 tools, 7 files)

`Axiom/packages/mcp-tools/src/`:

- `types.ts`: `McpToolResult<T> = {ok:true,data:T} | {ok:false,error}`,
  `McpToolHandler<TInput,TOutput>`, `McpToolRegistryEntry
  {capabilityId, domain, handler}`. Untyped `unknown`-in/`unknown`-out at
  the registry-map level by design (documented: 16 distinct signatures
  cannot share one generic map type).
- `registry.ts`: `MCP_TOOL_CAPABILITY_IDS` (16 entries, `sdd.*`/`spec.*`),
  `MCP_TOOL_HANDLERS` (built once at module load, pure data),
  `invokeMcpTool(capabilityId, input)` — generic dispatch, returns
  `undefined` for unregistered ids, never throws.
- `registry-handlers.ts`: `getProjectRegistry(homeDir, includeStale?)` ->
  `ProjectEntryV2WithStatus[]`; `getProjectRepos(homeDir, projectId)` ->
  `ProjectEntryV2WithStatus['repos'] | null` (a `Record<RepoRoleKey,
  RegistryRepoEntryWithStatus>`, i.e. an **open string-keyed map**, not a
  fixed `{spec,sdd,target}` triple).
- `artifact-handlers.ts`: `getPlan`/`getIncrement`/`getBug` all delegate
  to one shared `readArtifact(kind, input)` wrapping
  `loadArtifactMetadata(projectRoot, kind, id, specRelPath?)`;
  `getAdrIndex`/`getDecisions` wrap `listArtifacts(projectRoot, kind,
  specRelPath?)` -> `{entries: ArtifactListEntry[], failures: [...]}`
  (only `entries` surfaced); `getAllowedWriteScope` re-reads the plan and
  projects `.allowedWriteScope` (`AllowedWriteScopeEntry{repo,
  paths[]}[]`).
- `skills-handlers.ts`: `getSkillIndex(rootPath, role)` ->
  `SkillsRoleIndex | null` (flat `mandatory: [{id,path?,reason?}]`,
  `available: [{id,path?,tags,summary?}]`); `getSkillCatalog(path)` ->
  `SkillsCatalog | null`; `getSkill(path, skillId)` -> `CatalogEntry |
  null`.
- `technical-context-handlers.ts`: `getTechnicalContextIndex
  (specScopeAbsolutePath, roleOrKind)` -> `TechnicalContextIndex | null`
  (nested `mandatory.always`/`mandatory.whenTags`/`available`);
  `listRecommendedContext(specScopeAbsolutePath, roleOrKind, taskTags)`
  reuses `resolveMandatoryDocuments(index, taskTags)` (ALL-tags-present
  semantics, explicitly **not** a true any-tag "recommended" matcher —
  same caveat as INC-13's close).
- `cli-backed-handlers.ts`: `rebuildIndexes`/`validateIndexes`/
  `validateChanges`, thin re-exports of `@axiom/cli-commands`'s
  `runIndexRebuild`/`runIndexValidate`/`runValidateChanges`. Not directly
  used by `get_implementation_context` (maintenance-domain tools, out of
  this tool's read-only scope).

Confirms `Axiom.Spec/general-spec.md`'s "MCP tool registration" section
verbatim — no drift found between that summary and the actual code.

### 2. `@axiom/orchestrator` — roadmap hint refuted

Read in full: `index.ts`, `types.ts`, `gates.ts`, `state-machine.ts`,
`runner.ts`. Verdict: **the roadmap's hint that `@axiom/orchestrator` has
"a profile/capability/provider resolution pattern worth reusing" is
stale/inaccurate.**

What `@axiom/orchestrator` actually is: a 22-entry (7 lifecycle + 15
intent) declarative lifecycle **state machine** (`STATE_MACHINE: Record
<CommandId, StateMachineEntry>`), a pure gate function (`gateFor`) that
evaluates filesystem-existence preconditions
(`axiom.yaml`/`init.json` presence) against a minimal
`ProjectContextLike {rootPath, projectName}`, and an async runner
(`runCommand`) that calls the gate before invoking a caller-supplied
`fn()`. There is no concept of "capability," "provider," or "profile"
anywhere in this package — those terms and that resolution logic
actually live in `@axiom/tool-routing` (`routeTool`, fallback walk,
provider resolution) and `@axiom/capability-model` (16 capability ids, 4
domains, provider registry), which is exactly what `@axiom/mcp-tools`
(INC-13) already sits on top of. `@axiom/orchestrator` is not the
"closest adjacent piece" for `get_implementation_context`; the domain
packages `get_implementation_context` needs to compose
(`@axiom/workflow`, `@axiom/skills`, `@axiom/technical-context`,
`@axiom/user-workspace`, `@axiom/topology`) are already fully wired
through `@axiom/mcp-tools`'s existing 16 handlers, and that is the real
adjacent piece to reuse. `@axiom/orchestrator` is not touched by, and has
no bearing on, this increment.

### 3. Field-by-field composition table (source doc §8.3)

| Response field | Status | Source |
|---|---|---|
| `project.id`, `project.name` | **Trivial composition** | `getProjectRegistry`/`getProjectRepos` input `projectId` echoed back; `name` from `ProjectEntryV2WithStatus` (registry-handlers.ts) if present. |
| `repositories.spec` / `.sdd` / `.target` | **New composition logic (thin)** | `getProjectRepos` returns `Record<RepoRoleKey, RegistryRepoEntry>` (open string-keyed map, `RepoRoleKey = string`), not a closed `{spec,sdd,target}` triple. Composition must select entries by role/id convention (e.g. role key `'spec'`/`'sdd'`, and `input.targetRepoId` or the sole remaining entry for `target`). No existing tool returns a fixed 3-key triple — this is real, small mapping logic, not a new backing function. |
| `plan.id/path/status/summary/content/metadata` | **Trivial composition** | `getPlan(projectRoot, planId)` -> `PlanMetadata`. `path` derives from the same folder-per-artifact convention (`<specPath>/plans/<id>/`) already used by `loadArtifactMetadata`; `content` (raw markdown body, if any) is not currently returned by `getPlan` — needs a small additive read of the artifact folder's body file if one exists, or omit `content` when absent (matches "absent is not an error" convention). |
| `relatedSpec.{id,path,summary}` | **Placeholder decision (see below)** | No `get_related_specs` tool exists. Placeholder: derive from `plan.links.incrementId` (already on `PlanMetadata.links: PlanLinks`) via `getIncrement(projectRoot, incrementId)`. |
| `relatedAdrs: ContextRef[]` | **New composition logic (thin)** | No direct `plan -> adr` link field exists. `getAdrIndex(projectRoot)` returns all ADRs (`ArtifactListEntry[]` with `metadata.links: AdrLinks {incrementId?, planId?}`); composition filters entries where `links.planId === planId` or `links.incrementId === plan.links.incrementId`. A filter over an existing list, not a new backing function. |
| `mandatory.sddSkills` | **Trivial composition** | `getSkillIndex(sddRepoRootPath, role)` -> `SkillsRoleIndex.mandatory` (role-index loaded against the **sdd** repo's root). |
| `mandatory.repoSkills` | **Trivial composition** | `getSkillIndex(targetRepoRootPath, role)` -> same function, loaded against the **target** repo's root instead. Same handler, different repo-root input — not new logic, just two calls. |
| `mandatory.technicalContext` | **Trivial composition** | `getTechnicalContextIndex(specScopeAbsolutePath, roleOrKind)` -> `.mandatory.always` plus `resolveMandatoryDocuments(index, taskTags)`'s `whenTags` matches (same helper `listRecommendedContext` already wraps). |
| `mandatory.rules` | **New composition logic OR documented empty** | No existing tool/index models a "rules" document set distinct from technical-context/skills. Recommend: return `[]` for a first pass, with a comment noting no backing artifact kind exists yet (do not invent a new artifact kind — out of `AGENTS.md`'s bootstrap limits). |
| `mandatory.commands` | **New composition logic OR documented empty** | Same situation as `rules` — no existing "commands doc" artifact kind. Recommend `[]` for first pass. |
| `indexes.sddSkills` / `.repoSkills` / `.technicalContext` / `.commands` / `.rules` / `.adr` | **Trivial composition (partial)** | `sddSkills`/`repoSkills`/`technicalContext` summarize the same `SkillsRoleIndex`/`TechnicalContextIndex` objects already loaded above (id/count/available-count, not full content) — trivial. `adr` summarizes `getAdrIndex`'s entry count. `commands`/`rules` have no backing index (same gap as the `mandatory.*` fields above) — return an empty/absent `IndexSummary` for first pass. |
| `recommended.skills` | **Trivial composition** | `SkillsRoleIndex.available` filtered/echoed (or full list if no tag-matching implemented yet — see `contextBudget` design). |
| `recommended.technicalContext` | **Trivial composition** | `TechnicalContextIndex.available`, same treatment. |
| `recommended.adrs` | **New composition logic (thin)** | No existing "recommend an ADR" logic. Recommend: reuse the same `relatedAdrs` filter result (an ADR already related to the plan/increment is definitionally "recommended") rather than inventing a second algorithm. |
| `recommended.commands` | **Documented empty** | No backing artifact kind (same as `mandatory.commands`). `[]` for first pass. |
| `allowedWriteScope: string[]` | **Trivial composition (shape change)** | `getAllowedWriteScope(projectRoot, planId)` -> `AllowedWriteScopeEntry[] | null` (`{repo, paths[]}[]`), richer than source doc's flat `string[]`. Composition must flatten to `"<repo>:<path>"`-style strings or similar to match §8.3's literal shape, or the brief should recommend keeping the richer shape and treating §8.3's `string[]` as illustrative, not literal (see brief's recommendation). |
| `confidence: "high"\|"medium"\|"low"` | **New composition logic** | No existing tool computes this. Simple, coarse heuristic (see brief): degrade based on `missingMetadata.length` and whether `repositories`/`relatedSpec` resolved. |
| `missingMetadata: string[]` | **New composition logic** | No existing tool computes this. Trivially derivable: collect a string per field above that resolved to `null`/absent (e.g. `"relatedSpec"`, `"repositories.target"`). |

Summary count: **9 fields/groups are trivial composition** (direct call to
an existing tool, or two calls to the same tool with different args), **5
are thin new composition logic** (a filter, a flatten, or a small
selection over already-available data — not new backing functions or new
domain packages), **4 are documented-empty for a first pass** (`rules`,
`commands` under `mandatory`/`indexes`/`recommended` — no backing
artifact kind exists and inventing one is out of scope), and **2 are new,
genuinely computed fields** (`confidence`, `missingMetadata`) that exist
nowhere today and must be written as part of this tool. **Zero fields are
hard-blocked** on a deferred INC-13b tool — `relatedSpec` was the one
candidate blocker and is resolved via the placeholder below, not left
blocked.

### 4. `relatedSpec` placeholder decision

**Recommendation: placeholder now, not a co-dependency.** Fill
`relatedSpec` from `plan.links.incrementId` (a field `PlanMetadata`
already has today, landed by INC-06) via a direct `getIncrement(
projectRoot, incrementId)` call — already-registered tool, zero new
capability ids, zero new backing functions. When `plan.links.incrementId`
is `null` (a plan with no linked increment, which the schema explicitly
allows), `relatedSpec` is `null` and `"relatedSpec"` is added to
`missingMetadata`. This is a strictly narrower capability than the full
`get_related_specs` tool (which per source doc §7.3 would presumably also
search/rank specs not already linked by id), but it exactly matches this
codebase's established minimalism precedent: INC-13 shipped 16 tools as
"thin translation of an existing, unmodified backing function... no new
business logic was written for any of them," and this placeholder follows
that same discipline. Building full `get_related_specs` search/ranking
logic now would be exactly the kind of speculative complex-metadata-system
`AGENTS.md` prohibits for a tool with no current caller needing more than
"the one spec this plan already declares it implements." Revisit only if
a future increment surfaces a real need for `relatedSpec` beyond the
already-linked increment (e.g. cross-referencing specs that mention the
plan without a formal link).

### 5. `contextBudget` design (first pass, coarse)

Three tiers, each controlling exactly one thing — how much document
**content** is inlined vs. referenced by id/path — deliberately not
touching which fields are present (all fields in the response shape are
always present at `small`; budget never removes structure, only content
weight):

- `"small"` (default when omitted): every `mandatory.*` and `related*`
  entry is returned as a reference only (`{id, path, reason?}` — no file
  content read). `recommended.*` returns ids only. This matches §8.1's
  "minimal reliable context, not everything" objective as the safe
  default.
- `"medium"`: `mandatory.*` entries have their document content inlined
  (small read per mandatory doc, since "mandatory" is by definition a
  short, curated list); `recommended.*` stays id-only.
  `relatedSpec.content`/`plan.content` inlined if the underlying artifact
  body is reasonably sized (no hard byte cap in this first pass —
  revisit only if a real oversized-document case appears).
  `relatedAdrs` stays reference-only (ADR bodies can be long; inlining N
  ADRs is exactly the "saturar el contexto" §8.1 warns against).
- `"large"`: same as `medium`, plus `relatedAdrs` content inlined too.
  Still does not inline `recommended.*` content (recommended-but-not-
  mandatory documents are, by construction, the ones the caller should
  explicitly fetch if actually needed — inlining them defeats the
  reference/recommendation split that is the entire point of §8.1's
  "no debe devolver todo").

Kept intentionally coarse: no per-document byte budgets, no token
counting, no per-field overrides. If `contextBudget` is omitted, behave
as `"small"`. This is a first-pass policy `mcp-tool-implementer` should
treat as the starting contract, not a locked spec — refine only if a
concrete oversized-response case is found in practice.

### 6. Brief for `mcp-tool-implementer`

- Add one new capability id (suggest `spec.implementationContextRead`,
  following the existing `spec.*`-for-read-only-composite-reads
  convention) registered in `@axiom/mcp-tools`'s `registry.ts`, alongside
  (not replacing) the 16 existing entries.
- New handler file, e.g. `implementation-context-handler.ts`, exporting
  `getImplementationContext(input: {projectId, planId, targetRepoId?,
  role?, contextBudget?})`. Implement by calling the **existing**
  handlers from `registry-handlers.ts`, `artifact-handlers.ts`,
  `skills-handlers.ts`, `technical-context-handlers.ts` directly (same
  package, already importable) — do not re-import their backing domain
  functions a second time; call the already-exported handler functions.
- Follow the composition table above field-by-field. Write `confidence`/
  `missingMetadata` computation as pure post-processing over the already-
  collected pieces (no new I/O beyond what the composed calls already do).
- `mandatory.rules`/`mandatory.commands` and their `indexes`/
  `recommended` counterparts: return empty arrays / absent summaries with
  an inline code comment citing this increment's finding (no backing
  artifact kind exists yet) — do not invent a new artifact kind to fill
  them.
- `allowedWriteScope`: recommend keeping `AllowedWriteScopeEntry[]`
  (`{repo, paths[]}[]`) as the actual returned shape rather than
  force-flattening to `string[]`, since the richer shape is strictly more
  useful and already what `getAllowedWriteScope` returns — treat source
  doc §8.3's literal `string[]` as illustrative shorthand, not a binding
  wire-format requirement, consistent with how this codebase has already
  treated other source-doc shapes as intent-level rather than
  byte-literal (e.g. general-spec.md's ADR/decision folder naming
  discussion).
- Do not touch `@axiom/orchestrator`, `@axiom/tool-routing`, or
  `@axiom/capability-model` — confirmed orthogonal per Finding 2 above.
- Add tests mirroring the existing `packages/mcp-tools/tests/` pattern
  (per INC-13's own test suite, not inspected file-by-file in this audit
  but assumed present given INC-13's closure required validation).

## Validation

`No validation command was found. Performed best-effort validation by inspecting changed files and checking consistency against the requested behavior.`

Best-effort validation performed for this audit-only increment:

- Read all 7 files in `Axiom/packages/mcp-tools/src/` in full.
- Read all 5 files in `Axiom/packages/orchestrator/src/` in full.
- Read the exact type shapes composed against
  (`Axiom/packages/workflow/src/artifact-store.ts`,
  `Axiom/packages/topology/src/types.ts`,
  `Axiom/packages/user-workspace/src/registry.ts`/`registry-types.ts`,
  `Axiom/packages/skills/src/role-index.ts`,
  `Axiom/packages/technical-context/src/technical-context-index.ts`) to
  ground every composition-table row in actual code, not inferred names.
- Re-read source doc §8 in full (§8.1-§8.4) directly from
  `axiom_decisiones_sesion_prompt_implementacion.md` rather than relying
  on the roadmap's paraphrase.
- Grepped the whole `Axiom` repo for `get_related_specs`/`relatedSpec` to
  confirm zero existing implementation before recommending the
  placeholder.
- No code was changed; no build/test run applies to a planning-only
  increment.

## Result

Confirmed `get_implementation_context` can be built entirely as
composition over already-existing `@axiom/mcp-tools` handlers and
already-existing domain-package functions, with no field hard-blocked on
a deferred INC-13b tool. Of 21 response fields/groups in source doc
§8.3: 9 are trivial composition, 5 are thin new composition logic
(filters/selections over existing data), 4 are documented-empty pending a
future "rules"/"commands" artifact kind that does not exist yet, and 2
(`confidence`, `missingMetadata`) are genuinely new, small computed
fields. The roadmap's `@axiom/orchestrator`-as-adjacent-piece hint is
confirmed stale: that package is a lifecycle state machine with no
capability/provider/profile resolution concept; the real adjacent
pieces are `@axiom/mcp-tools`'s own 16 handlers, which already compose
cleanly. `relatedSpec` resolves via a documented placeholder
(`plan.links.incrementId` -> `getIncrement`), not a blocking
co-dependency on full `get_related_specs`. `contextBudget` is designed as
a coarse 3-tier content-inlining control, not a token-budgeting engine.

## General spec integration

Not yet integrated — this increment is audit-only and the composition
plan is a starting brief for `mcp-tool-implementer`, not yet-implemented
stable fact. Once `mcp-tool-implementer` and `validator-reviewer` close
their phases and `get_implementation_context` is real, working code,
`Axiom.Spec/general-spec.md`'s "MCP tool registration" section should be
extended with: the new capability id, the confirmed final response shape
(especially whether `allowedWriteScope` stayed structured or was
flattened), and the `relatedSpec` placeholder's documented limitation.
Recording it now would describe a plan, not a fact, which the general
spec's own conventions elsewhere (e.g. the ADR-index "derived, not
curated" entry) avoid doing.

## Next step recommendation

Hand this spec to **mcp-tool-implementer** with the brief in
"Implementation notes" section 6 as its concrete starting task: add one
new capability id + handler file to `@axiom/mcp-tools`, composing the
already-listed existing handlers, following the field-by-field table
and the `relatedSpec`/`contextBudget` decisions above. After
implementation, run **validator-reviewer** to confirm the composition
introduced no new domain-package dependency violations (the new handler
must stay inside `@axiom/mcp-tools`, calling only already-exported
handler functions from sibling files in the same package, per the
package's own "thin registration layer" architecture rule), and that
`mandatory.rules`/`mandatory.commands` staying empty is not silently
mistaken for a bug by tests expecting non-empty arrays.
