# Increment: MCP tool-routing reconcile (INC-13, Phase E)

Status: pending
Date: 2026-07-02

## Goal

Audit `@axiom/tool-routing`, `@axiom/capability-model`, `@axiom/isolation`,
and `@axiom/doctor`'s `TR-001..004` checks in full, and produce a concrete,
per-tool mapping from the new decision documents' MCP tool list (source doc
§7.3) onto the existing capability/provider dispatch mechanism — so that
`mcp-tool-implementer` (the next subagent in the roadmap's INC-13 sequence)
has a grounded, non-speculative brief instead of the roadmap's original
"needs an audit" placeholder. This increment is audit-only: no code is
written.

## Context

Roadmap: `Axiom.Spec/specs/increments/INC-20260702-axiom-redesign-roadmap/
README.md`, Phase E, INC-13. INC-01 through INC-12 are closed.

**Q4 is resolved (user decision, binding for this increment)**: the
existing `capability-model`/`tool-routing`/`isolation`/`telemetry`/
`gateway`/`orchestrator` system stays completely intact and orthogonal to
the new decision documents. The new tools
(`get_project_registry`, `get_implementation_context`, etc.) are added as
**new `capabilityId`s** dispatched through the **existing** `routeTool`
mechanism, keeping `sdd`/`spec`/`serena` as the allowed providers per
`@axiom/isolation`'s existing allowlist (ADR-0021). This increment does not
redesign the capability model — it registers new capabilities on top of it.

Source docs (read directly, not summarized from the roadmap):
`C:\Users\igutierrezz\Downloads\axiom_decisiones_sesion_prompt_implementacion.md`
§7.3 (full tool list) and §8 (`get_implementation_context` signature/shape —
flagship tool, its own increment, INC-14);
`C:\Users\igutierrezz\Downloads\axiom_decisiones_sesion_addendum_revision.md`
§4 (filesystem-mode vs MCP-mode coexistence) and §17 (security/isolation).

## Scope

- Read `Axiom/packages/tool-routing/src/*` in full (types, dispatcher,
  select, fallback, schemas, events, index) plus
  `tests/route-tool.test.ts`. Confirm exactly how a new `capabilityId` is
  registered and dispatched today.
- Read `Axiom/packages/capability-model/src/*` in full (types, constants,
  loader, index, matrix, policies, resolver) plus its tests. Confirm the
  `CapabilityId`/`CapabilityDefinition`/`ProviderDefinition` shapes and the
  provider-registration mechanism.
- Read `Axiom/packages/isolation/src/p0.ts` (MCP allowlist,
  `DEFAULT_ALLOWED_MCP_SERVERS`) to confirm no change is needed there.
- Read `Axiom/packages/doctor/src/checks.ts`'s `runToolRoutingChecks`
  (`TR-001..004`) to document exactly what they smoke-test today.
- Produce a per-tool mapping table (source doc §7.3's full list) against
  what already exists as a CLI command or library function from INC-01
  through INC-12.
- Recommend the minimal, concrete registration mechanism for adding N new
  `capabilityId`s, grounded in `capability-model`'s real extension points.
- Note `get_implementation_context` (source doc §8) is out of scope here —
  it is INC-14, and depends on most of this increment's mapped tools
  existing first.

## Non-goals

- No code implementation. This is audit-only, per the user's explicit
  instruction and the roadmap's own subagent sequencing
  (migration-engineer audits before mcp-tool-implementer writes anything).
- No implementation of `get_implementation_context` (INC-14).
- No change to `@axiom/isolation`'s MCP allowlist (`sdd`/`spec`/`serena`) —
  confirmed unnecessary, not attempted.
- No re-litigation of Q4 — treated as resolved per the user's decision
  above.
- No new MCP server process, no new transport. "Wiring a new capability"
  in this spec means adding it to the capability/provider data model that
  `routeTool` already consumes, not standing up new infrastructure.

## Acceptance criteria

- [x] `tool-routing`'s dispatcher contract is documented precisely enough
      that a mechanical reader can state what a new `capabilityId`
      requires without re-reading the source.
- [x] `capability-model`'s exact type shapes
      (`CapabilityDefinition`/`ProviderDefinition`/`CapabilityModel`) and
      its loader's extension point are documented.
- [x] `isolation`'s allowlist is confirmed unchanged and the reason stated.
- [x] `TR-001..004`'s exact smoke-test behavior is documented so
      validator-reviewer can extend (not replace) them later.
- [x] A concrete per-tool table exists for source doc §7.3's full list,
      each row stating: tool name, existing backing
      function/package (if any), and trivial-exposure vs.
      needs-new-logic classification.
- [x] `get_implementation_context` is explicitly flagged out of scope
      (INC-14) with its dependency on this increment's table noted.
- [x] A registration-mechanism recommendation is given, grounded in
      `capability-model`'s actual loader/schema code, not speculation.
- [x] Recorded in `Axiom.Spec`; no code changed in `Axiom.SDD` or `Axiom`.

## Open questions

None blocking. Q4 (the only open question that gated this increment per
the roadmap) is resolved by the user's decision recorded above.

## Assumptions

- "New `capabilityId`s" means new entries in the same `CANONICAL_CAPABILITY_IDS`-style
  taxonomy (currently hardcoded in `constants.ts`, 16 ids across 4 domains,
  changeable only "via ADR" per that file's own header comment) plus matching
  `ProviderDefinition.applicableCapabilities` entries on `sdd`/`spec`/`serena`.
  Whether that taxonomy literally stays at 16 fixed ids or grows is itself a
  decision `mcp-tool-implementer`/an ADR must make explicitly when it writes
  code — this increment surfaces the constraint, does not resolve it.
- No `capabilities.yaml`/`providers.yaml` files exist on disk anywhere in
  `Axiom/` today (confirmed by search) — `loadCapabilityModel` is exercised
  only by unit tests building in-memory fixtures and by the doctor's
  `TR-001..004` in-memory smoke context. Any real capability/provider
  registration for the new tools will be the **first** real YAML-backed
  registration this loader has ever done in this repo, not an edit to an
  existing file.
- The per-tool table's "existing backing function" column names the actual
  exported function/package confirmed by direct source reads in this audit
  (see Implementation notes), not inferred from package names.

## Implementation notes (audit findings)

### 1. `@axiom/tool-routing` — dispatcher contract

Files read in full: `types.ts`, `dispatcher.ts`, `select.ts`, `fallback.ts`,
`schemas.ts`, `events.ts`, `index.ts`, `tests/route-tool.test.ts`.

- `routeTool(call: ToolCall, context: RouteToolContext): ResolvedDispatch`
  is a **pure function** — no filesystem/network I/O. `RouteToolContext`
  (`{ projectScope, profile, capabilityModel }`) is built entirely
  upstream by the caller from `@axiom/project-resolution`,
  `@axiom/install-profiles`, and `@axiom/capability-model`'s
  `loadCapabilityModel()`.
- `ToolCall.capabilityId` must match `/^[a-z]+\.[a-zA-Z]+$/`
  (`<domain>.<verbOrNoun>`, enforced both by `ToolCallSchema` in
  `schemas.ts` and duplicated informally in `dispatcher.ts`'s
  `isWellFormedCall`). **Any new tool name must be expressed as a
  capabilityId matching this exact regex** — e.g. `get_skill_index` would
  need to become something like `spec.skillIndexRead` or
  `sdd.skillIndexRead`, not the literal snake_case tool name from the
  source doc.
- The 9-step dispatch algorithm (as implemented, not just as commented):
  1. `isWellFormedCall` (id/capabilityId/envTarget non-empty).
  2. `capabilityId` must be in `profile.enabledCapabilities`.
  3. `capabilityId` must exist in `capabilityModel.capabilities`.
  4. If `operationId` starts with `axiom.` and
     `profile.gatewayExpectation === 'forbidden'` → degraded.
  5. `walkFallbackChain` builds the candidate list: dedup
     `profile.preferredProviders` + `profile.discoveryOrder` (normalizing
     the `'gateway'` token to `'axiom-gateway'`), filtered by
     `provider.applicableCapabilities.includes(capabilityId)`, then by
     `envTargetSupportsCapability` (matrix lookup; unknown = allowed) and
     `gatewayExpectation === 'forbidden'` (drops `axiom-gateway`).
  6. Empty chain → `env-unsupported`.
  7. Walk the chain; drop `workspace-native`/`semantic-navigation`
     providers when `workspaceIsAvailable(projectScope)` is false; drop
     `axiom-gateway` again if forbidden; first survivor wins.
  8. No survivor → `provider-failed`.
  9. Emit exactly one `DispatchEvent` (resolved or degraded) via
     `emitResolvedEvent`/`emitDegradedEvent`, which also forward to the
     active `@axiom/telemetry` `TelemetryBus`.
- **What registering a genuinely new capability requires, mechanically**
  (no filesystem changes to `tool-routing` itself — this package has zero
  hardcoded capability names):
  1. A new capability id string (matching the domain-dot-camelCase regex)
     added wherever `capabilityModel.capabilities` is populated
     (`capability-model`'s loader/data, see §2 below) — the dispatcher has
     no compile-time list to edit.
  2. At least one `ProviderDefinition.applicableCapabilities` entry naming
     that new capability id, on one of the existing providers
     (`sdd`/`spec`/`serena` per the Q4 decision — `filesystem`,
     `axiom-gateway`, `codegraph`, `graphify`, `engram`,
     `generated-snapshots` are the other 4 of the 7 canonical provider ids
     and are out of scope for this increment's tools).
  3. `profile.enabledCapabilities` (an install-profile array, owned by
     `@axiom/install-profiles`, not read in this pass) must include the
     new id for any profile that should be able to call it — otherwise
     `routeTool` degrades with `capability-missing` at step 2, deliberately.
  4. Nothing in `tool-routing` needs new code paths — same `routeTool`,
     same event shape, same doctor smoke pattern. This is the basis for
     the "additive, not breaking" call in the roadmap.
- Tests confirm purity (`same call + context → same result`, byte-for-byte
  including `emittedAt` derived from `call.requestedAt`, not
  `Date.now()`), and cover all 4 `DegradedReason`s
  (`capability-missing`, `provider-failed`, `gateway-forbidden-by-profile`,
  `env-unsupported`) plus the dedup/normalization behavior of
  `walkFallbackChain`. No test exercises a real
  `capabilities.yaml`/`providers.yaml` file — everything is in-memory
  fixtures (confirmed: no such YAML files exist anywhere under `Axiom/`,
  see Assumptions).

### 2. `@axiom/capability-model` — types, taxonomy, provider registration

Files read in full: `types.ts`, `constants.ts`, `loader.ts`, `index.ts`
(barrel), `matrix.ts`/`policies.ts`/`resolver.ts` referenced but not the
focus since `routeTool` does not call `resolveCapability` directly.

- `CapabilityId` is **not a closed TypeScript union/enum** — capability ids
  are plain `string` everywhere (`CapabilityDefinition.id: string`,
  `Record<string, CapabilityDefinition>`). The only closed set is the
  **runtime constant** `CANONICAL_CAPABILITY_IDS` in `constants.ts`: 16
  literal strings across 4 domains (`sdd.*` x4, `spec.*` x4, `code.*` x6,
  `memory.*` x2), with a header comment stating "HARD-CODED and can only
  change via ADR." This is the real governance gate for adding new
  capability ids — not a type-system constraint, a documented policy one.
- `CapabilityDomain = 'sdd' | 'spec' | 'code' | 'memory'` (4 domains,
  matches `CAPABILITY_DOMAINS` constant) — a real closed TS union. New
  tools' capability ids must fall into one of these 4 domains; none of
  the source doc §7.3 tools obviously need a 5th domain (they are
  spec-reading, skill-reading, context-reading, and validation
  operations — all fit `spec`/`sdd`/`code` as domains, arguably mostly
  `spec` and `sdd`).
- `CANONICAL_PROVIDER_IDS`: exactly 7 (`filesystem`, `axiom-gateway`,
  `serena`, `codegraph`, `graphify`, `engram`, `generated-snapshots`).
  `MVP_DEFAULT_PROVIDERS` is only `filesystem` + `serena`.
  `FORBIDDEN_DEFAULT_PROVIDERS` is `backend-mcp`/`frontend-mcp` (ADR-0021)
  — confirms this decision predates and is unrelated to the new tools;
  nothing here conflicts with keeping `sdd`/`spec`/`serena` as allowed.
  **Naming note**: `@axiom/isolation`'s allowlist names are `sdd`/`spec`/
  `serena` (MCP server names); `@axiom/capability-model`'s canonical
  provider ids are `filesystem`/`axiom-gateway`/`serena`/`codegraph`/
  `graphify`/`engram`/`generated-snapshots` (capability-routing provider
  ids). Only `serena` is spelled identically in both lists. `sdd` and
  `spec` as MCP server names in the isolation allowlist do not have an
  identically-named entry in `CANONICAL_PROVIDER_IDS` today — the closest
  analogs are `axiom-gateway` (project-scoped-broker, i.e. an
  `sdd`/`spec`-flavored broker) and `filesystem` (workspace-native reads).
  This is a real terminology gap the mcp-tool-implementer brief (§7
  below) must resolve explicitly, not paper over.
- `loadCapabilityModel(specScopePath)` reads
  `<specScopePath>/config/capabilities.yaml` and
  `<specScopePath>/config/providers.yaml`, validates each against a Zod
  schema (`CapabilityModelSchema`/`ProviderRegistrySchema` in
  `schemas.ts`, not read line-by-line in this pass but referenced by the
  loader), and builds the `CapabilityModel` aggregate. **This is the real
  extension point**: capabilities and providers are data (YAML), not code.
  Confirmed no such YAML file exists anywhere in `Axiom/` yet (`Glob` for
  `**/capabilities.yaml` and `**/providers.yaml` under `Axiom/` returned
  zero results) — every existing consumer (tests, doctor smoke checks)
  builds the `CapabilityModel` object literal directly in memory rather
  than loading real files.
- `loader.ts` never throws; returns `CapabilityModelLoadError` on
  file-not-found/YAML-parse/schema-validation failure. Doctor-friendly by
  design (matches the project's "skip cleanly, don't crash" convention).

### 3. `@axiom/isolation` — MCP allowlist confirmed unchanged

File read in full: `p0.ts` (+ `index.ts` barrel).

- `DEFAULT_ALLOWED_MCP_SERVERS = ['sdd', 'spec', 'serena']` — exactly the
  3 names the user's Q4 decision requires to stay as the allowed
  providers. `checkMcpAllowed(serverName, context)` does a simple
  `includes()` check against this list (or a caller-supplied override
  list) and returns a reasoned `McpCheckResult`.
- No structural change is needed here for INC-13's scope: the new tools
  route through `sdd`/`spec`/`serena` (per Q4), all three already present
  in `DEFAULT_ALLOWED_MCP_SERVERS`. Confirmed, not assumed, by direct
  read — this file has no dependency on `capability-model`'s provider ids
  at all (it is a separate, simpler string-list check operating on MCP
  server names, not `ProviderDefinition.id`s). The two lists (isolation's
  MCP server allowlist vs. capability-model's `CANONICAL_PROVIDER_IDS`)
  are confirmed to be two independent mechanisms today, not one derived
  from the other — worth flagging for `mcp-tool-implementer` so it does
  not assume a shared source of truth that does not exist.

### 4. `@axiom/doctor`'s `TR-001..004` — exact smoke-test behavior

File read: `Axiom/packages/doctor/src/checks.ts`,
`runToolRoutingChecks(resolution)` and its helpers
(`tryLoadToolRouting`, `buildSmokeContext`), lines ~859-1092. Wired into
`runAllChecks` via `Axiom/packages/doctor/src/index.ts`.

- **TR-001**: `require('@axiom/tool-routing')` at runtime (not a static
  import) and check `typeof mod.routeTool === 'function'`. If the
  resolution status isn't `'resolved'`, or the package fails to load,
  **all 4 checks `skip`** (not fail) — this check family is explicitly
  provisional/soft-fail by design, "skip cleanly until the package
  ships," per its own comment (now moot since `tool-routing` exists and
  builds).
- **TR-002**: builds an in-memory `standard` overlay profile + a 3-provider
  in-memory capability model (`filesystem`/`serena`/`axiom-gateway`, all
  only declaring `code.semanticNavigation`) via `buildSmokeContext`, calls
  `routeTool`, and asserts the result is non-degraded with a non-empty
  `providerEffective`.
- **TR-003**: same fixture but `overlay: 'local-only'`,
  `gatewayExpectation: 'forbidden'`; calls `routeTool` with an
  `operationId: 'axiom.*'`-prefixed call; asserts
  `degraded && degradedReason === 'gateway-forbidden-by-profile'`.
- **TR-004**: same `local-only` fixture but a **non**-`axiom.*` call;
  asserts it still resolves (not degraded).
- All 4 checks are entirely **in-memory** — no real `capabilities.yaml`/
  `providers.yaml` file is read, no new capability id from source doc
  §7.3 is exercised. **Extension implication for validator-reviewer
  (later, per roadmap)**: adding new capability ids per this increment's
  table does NOT require editing `TR-001..004` at all, since those checks
  only smoke-test `code.semanticNavigation` via a synthetic fixture and
  never touch a live capability registry. A genuinely useful extension
  would be a **new** `TR-005`+ check (or a new category) that loads the
  real `capabilities.yaml`/`providers.yaml` once they exist and confirms
  the new tool capability ids round-trip through `routeTool` — but that
  is new coverage, not a modification of `TR-001..004`'s existing
  contract. This confirms the roadmap's "extend, do not replace" framing
  is achievable with zero risk to the 4 existing checks.

### 5. Per-tool mapping table (source doc §7.3)

Legend: **Trivial** = an existing exported function/CLI command already
does this; wiring as an MCP capability is "expose the existing function
behind a new capabilityId + provider entry," no new business logic.
**New logic** = no existing backing function; the tool's behavior must be
written from scratch (composition, aggregation, or a capability that
genuinely doesn't exist yet). **Composite/deferred** = explicitly out of
scope for this increment (INC-14).

| # | Tool (source doc §7.3) | Existing backing function/package | Classification |
|---|---|---|---|
| 1 | `get_project_registry` | `listProjectsV2(homeDir, opts)` — `@axiom/user-workspace/registry.ts` | Trivial |
| 2 | `get_project_repos` | `getProjectV2(homeDir, id)` — same file, returns `repos` map per project (registry v2 shape, see `general-spec.md`) | Trivial |
| 3 | `get_workspace_equivalent` | No exact equivalent found. Closest: `@axiom/project-resolution`'s `discoverAxiomRoot`/`ProjectResolution.scopes`, but that resolves scopes *within* one repo, not a registry-wide "equivalent path across the registered repos of a project" concept the source doc implies. | New logic |
| 4 | `get_implementation_context` | None (flagship composite) | Composite/deferred — **INC-14**, depends on rows 1, 2, 5-19 below existing first |
| 5 | `get_plan` | `loadArtifactMetadata(projectRoot, 'plan', id)` — `@axiom/workflow/artifact-store.ts` | Trivial |
| 6 | `get_increment` | `loadArtifactMetadata(projectRoot, 'increment', id)` — same file | Trivial |
| 7 | `get_bug` | `loadArtifactMetadata(projectRoot, 'bug', id)` — same file | Trivial |
| 8 | `get_related_specs` | No exact equivalent. `listArtifacts`/`loadArtifactMetadata` return one artifact's own metadata (including `links`), but nothing today resolves a "related specs" graph/traversal across artifacts. | New logic |
| 9 | `get_adr_index` | `listArtifacts(projectRoot, 'adr')` — `@axiom/workflow/artifact-store.ts`, already exposed via `axiom-adr.ts`'s `list` subcommand | Trivial |
| 10 | `get_decisions` | `listArtifacts(projectRoot, 'decision')` — same file, exposed via `axiom-decision.ts`'s `list` subcommand | Trivial |
| 11 | `get_skill_index` | `loadSkillsRoleIndex(rootPath, role)` — `@axiom/skills/role-index.ts` | Trivial |
| 12 | `get_skill_catalog` | `loadSkillsCatalog(path)` — `@axiom/skills/catalog.ts` | Trivial |
| 13 | `get_skill` | `getSkillById(...)` — `@axiom/skills/catalog.ts` | Trivial |
| 14 | `list_recommended_skills` | `getRoleIndexEntryById` (`role-index.ts`) resolves one entry by id, but nothing today filters/ranks the role-index's `available` list by task tags the way `resolveMandatoryDocuments` does for technical-context (row 17). No direct "recommend by tags" function exists for skills. | New logic (thin — likely a small pure function mirroring `resolveMandatoryDocuments`'s pattern, not a full subsystem) |
| 15 | `get_technical_context_index` | `loadTechnicalContextIndex(specScopeAbsolutePath, roleOrKind)` — `@axiom/technical-context/technical-context-index.ts` | Trivial |
| 16 | `get_technical_context_document` | No exact equivalent. `loadTechnicalContextIndex` returns the index (paths + metadata), but no function reads the referenced document *content* at a given `path`. Likely a thin file-read wrapper, not genuinely new logic — flagged separately from row 3/8/14 because the gap is this small. | New logic (thin) |
| 17 | `list_recommended_context` | `resolveMandatoryDocuments(index, taskTags)` — `@axiom/technical-context/technical-context-index.ts` (note: resolves *mandatory*, ALL-tags-present docs; the source doc's "recommended" language maps most closely to this existing ALL-semantics helper, though a literal "recommended/available-by-any-tag" variant does not exist — same class of gap as row 14) | Mostly trivial (reuse `resolveMandatoryDocuments`; an "available"-side any-tag variant would be new logic, small) |
| 18 | `get_allowed_write_scope` | `PlanMetadata.allowedWriteScope` field, already loaded by `loadArtifactMetadata(..., 'plan', ...)` — `@axiom/workflow/write-scope.ts` defines the shape; no separate accessor function needed beyond reading the loaded metadata | Trivial |
| 19 | `validate_changes` | `runValidateChanges(args)` — `Axiom/apps/cli/src/commands/validate-changes.ts`, backed by `validateWriteScope` (`@axiom/workflow/write-scope.ts`) | Trivial |
| 20 | `validate_project` | No single exact equivalent. Closest: `@axiom/doctor`'s `runAllChecks`/full check suite (`axiom doctor`), which is a superset validating far more than write-scope alone — likely the correct backing, but needs an explicit decision on which check categories `validate_project` surfaces (all of them vs. a curated subset) | Mostly trivial (reuse `@axiom/doctor`'s existing check runner; the "new logic" is a thin filter/summary layer, not new checks) |
| 21 | `rebuild_indexes` | `runIndexRebuild(args)` — `Axiom/apps/cli/src/commands/index-cmd.ts` | Trivial |
| 22 | `validate_indexes` | `runIndexValidate(args)` — same file | Trivial |

**Tally**: of 22 tools (excluding `get_implementation_context`, counted
separately as composite/deferred), **15 are trivial** (exposing an
existing function/command as a new capability, no new business logic),
**5 need new logic** (`get_workspace_equivalent`, `get_related_specs`,
`list_recommended_skills`, `get_technical_context_document`,
`list_recommended_context`'s any-tag variant — though 2 of these 5 are
"thin" gaps, not substantial builds), and **2 are "mostly trivial"**
(`list_recommended_context` reusing `resolveMandatoryDocuments` almost
as-is; `validate_project` reusing `@axiom/doctor`'s runner with a thin
summary layer) — arguably closer to trivial than new-logic once counted
precisely. `get_implementation_context` (INC-14) is excluded from this
tally as explicitly out of scope.

### 6. `get_implementation_context` — explicitly out of scope (INC-14)

Confirmed per the user's instruction and the roadmap's own INC-14 entry:
this is the flagship composite tool (source doc §8's signature/response
shape). It is **not** built in this increment. It depends on most of the
table above existing first (it composes plan, spec, ADRs, mandatory
skills, mandatory technical context, allowed write scope, and
recommendations — i.e. rows 1, 2, 5, 8, 9, 11, 15, 17, 18 at minimum).
Noted here only so `mcp-tool-implementer` does not accidentally start
building it while working this increment's table.

### 7. Registration-mechanism recommendation

Grounded directly in `capability-model`'s real extension points (§2
above), not speculation:

1. **Capability ids**: add new entries to whatever populates
   `capabilities.yaml`'s `capabilities` map — since no such file exists
   yet in `Axiom/`, this is the **first real file** to create, not an
   edit. Recommend one file,
   `axiom.spec/config/capabilities.yaml`, matching `loadCapabilityModel`'s
   expected path (`<specScopePath>/config/capabilities.yaml`) and
   `CapabilityModelSchema`'s shape (schema itself not read line-by-line in
   this pass; `mcp-tool-implementer` must read `schemas.ts` before writing
   this file). Each new tool's capability id must satisfy
   `<domain>.<verbOrNoun>` and pick one of the 4 existing domains
   (`sdd`/`spec`/`code`/`memory`) — `spec.*` and `sdd.*` are the natural
   fits for nearly every row in the table above (reading plans/increments/
   bugs/ADRs/decisions/skills/technical-context is `spec`-domain reading;
   validation/index maintenance leans `sdd`-domain per the existing
   `sdd.workflow`/`sdd.commandResolution` naming precedent).
2. **Provider registration**: add matching entries to
   `axiom.spec/config/providers.yaml`'s `providers[].applicableCapabilities`
   arrays for whichever of `sdd`/`spec`/`serena`'s **capability-model**
   analogs should serve each new capability. This surfaces the naming gap
   noted in §2: `capability-model`'s canonical provider id list has no
   literal `sdd`/`spec` provider id today, only `axiom-gateway`
   (project-scoped-broker) and `filesystem` (workspace-native). Recommend
   `mcp-tool-implementer`'s first concrete decision be: does `sdd`/`spec`
   (the MCP server names from `@axiom/isolation`'s allowlist) map onto
   `axiom-gateway` as the capability-model provider id (a broker serving
   `spec.*`/`sdd.*` capabilities), or does capability-model need two new
   canonical provider ids literally named `sdd`/`spec`? Given
   `CANONICAL_PROVIDER_IDS` is explicitly "hard-coded, changeable only via
   ADR" (constants.ts), the lower-friction, ADR-avoiding path is mapping
   onto the existing `axiom-gateway` provider id — but this is a real
   decision to make explicitly, not silently assume.
3. **No new file format, no new loader**: `loadCapabilityModel` already
   handles arbitrary capability/provider lists from YAML; adding N new
   tools this way requires zero changes to `loader.ts`, `dispatcher.ts`,
   `select.ts`, or `fallback.ts`. This is what makes the roadmap's
   "additive, not breaking" classification correct.
4. **Per-profile enablement**: whichever `@axiom/install-profiles`
   profile(s) should expose the new tools need their
   `enabledCapabilities` arrays extended — this package was not read in
   this pass (out of the 3-package scope the user specified) and is
   flagged as `mcp-tool-implementer`'s second concrete pre-work item.
5. **Doctor extension** (validator-reviewer's later task, not this
   increment's): a new `TR-005`+ check loading the real
   `capabilities.yaml`/`providers.yaml` and confirming each new tool's
   capabilityId round-trips through `routeTool` for at least one MVP
   target — additive to `runToolRoutingChecks`, no edit to `TR-001..004`.

## Validation

`No validation command was found. Performed best-effort validation by inspecting changed files and checking consistency against the requested behavior.`

This increment is audit-only (no code changed under `Axiom/` or
`Axiom.SDD/`); the "changed files" inspected are the increment spec file
itself. Best-effort validation performed: every claim above about
`tool-routing`/`capability-model`/`isolation`/`doctor` behavior is sourced
from a direct, full read of the named files (`types.ts`, `dispatcher.ts`,
`select.ts`, `fallback.ts`, `schemas.ts`, `events.ts`, `index.ts`, and
`tests/route-tool.test.ts` for tool-routing; `types.ts`, `constants.ts`,
`loader.ts`, `index.ts` for capability-model; `p0.ts`/`index.ts` for
isolation; the `runToolRoutingChecks` block of `checks.ts` for the doctor
checks), not inferred from package/file names. The per-tool table's
"existing backing function" column was verified against actual exported
symbols via targeted greps of `artifact-store.ts`, `write-scope.ts`,
`index-cmd.ts`, `validate-changes.ts`, `skills/*.ts`,
`technical-context-index.ts`, and `user-workspace/registry.ts` — not
assumed from the general-spec's package descriptions alone, though those
descriptions (from INC-01 through INC-12's closed specs) were consistent
with what was found.

## Result

Produced a grounded, non-speculative audit of the three packages the user
named, confirmed `TR-001..004`'s exact behavior for future extension, and
built a 22-row per-tool mapping table classifying 15 tools as trivial
exposures of existing functions, 5 as needing new (mostly thin) logic, and
2 as "mostly trivial" reuses with a thin new layer.
`get_implementation_context` is explicitly deferred to INC-14 per the
roadmap and the user's instruction. The registration mechanism is
data-driven (YAML capability/provider entries consumed by the existing,
unmodified `loadCapabilityModel`/`routeTool` pipeline) — no new package,
no new loader, no new dispatcher logic. One real naming gap was
surfaced and left as an explicit decision for `mcp-tool-implementer`
rather than silently resolved: whether `sdd`/`spec` (isolation's MCP
server names) map onto the existing `axiom-gateway` capability-model
provider id, or need two new canonical provider ids.

## General spec integration

No integration into `Axiom.Spec/general-spec.md` was performed in this
increment. Rationale: this increment's findings are audit conclusions and
a per-tool classification table meant to seed the next (code-writing)
increment's brief — they are not yet "stable, consolidated knowledge"
about a shipped contract, since no capability/provider YAML file exists
yet and no new capabilityId has actually been registered. Per
`Axiom.SDD/AGENTS.md`'s documentation rules, integration belongs at the
point a concrete, working contract lands (i.e. when `mcp-tool-implementer`
actually registers the new capabilities and `validator-reviewer` confirms
them), not at the audit stage. The one exception worth flagging for that
future integration: `general-spec.md`'s existing "Registry v2" and
"ADR/Decision artifact kinds" sections already document the backing
functions this table reuses (`listProjectsV2`, `listArtifacts`, etc.) —
future integration should cross-reference those sections rather than
re-describe them.

## Next step recommendation

Proceed to `mcp-tool-implementer` with this increment's per-tool table and
§7's registration-mechanism recommendation as its starting brief. Its
first concrete pre-work items, in order: (1) read `@axiom/install-profiles`
in full (not covered by this audit's 3-package scope) to confirm
`enabledCapabilities`/`preferredProviders` extension points; (2) read
`capability-model/src/schemas.ts` in full before writing the first real
`capabilities.yaml`/`providers.yaml`; (3) resolve explicitly whether
`sdd`/`spec` map onto `axiom-gateway` as the capability-model provider id
or need two new canonical provider ids (flagged in §7.2, not resolved
here). Only after those three items should it begin registering the 15
"trivial" tools from the table (lowest risk, highest immediate value),
deferring the 5 "new logic" tools and treating `get_implementation_context`
as INC-14's separate scope entirely.
