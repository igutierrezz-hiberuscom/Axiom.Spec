# Increment: MCP tool-routing reconcile — implementation (INC-13, Phase E, mcp-tool-implementer)

Status: closed
Date: 2026-07-02

Closure note (validator-reviewer, 2026-07-02): independently verified in
`INC-20260702-mcp-tool-routing-reconcile-validator/README.md`. All
findings confirmed accurate (handler-to-backing-function traces,
`ProviderRegistrySchema`/`CANONICAL_PROVIDER_IDS` constraint, `TR-001..004`
unmodified, `@axiom/cli-commands` barrel pattern preserved); test/build/
typecheck results independently reproduced with zero regressions. The
one open item this increment left (`TR-005`+ scope) is resolved there:
deferred, with rationale. See that file for the full independent
verification and INC-13-as-a-whole closure assessment.

## Goal

Implement the 15 "trivial" MCP tools identified by the migration-engineer
audit (`INC-20260702-mcp-tool-routing-reconcile/README.md`) as new
`capabilityId`s dispatched through the existing, unmodified
`@axiom/tool-routing`/`@axiom/capability-model` pipeline, plus resolve the
two concrete decisions the audit left open for this subagent: the
`sdd`/`spec` provider-id naming gap, and where the new handler functions
physically live in the dependency graph.

## Context

Roadmap: `Axiom.Spec/specs/increments/INC-20260702-axiom-redesign-roadmap/
README.md`, Phase E, INC-13. Prior increment in this chain:
`INC-20260702-mcp-tool-routing-reconcile` (migration-engineer audit,
audit-only, no code). This increment is `mcp-tool-implementer`'s pass.

Source docs (per the audit, not re-read line-by-line here — the audit's
per-tool table is this increment's brief):
`axiom_decisiones_sesion_prompt_implementacion.md` §7.3 (tool list),
`axiom_decisiones_sesion_addendum_revision.md` §4/§17.

## Scope

- Pre-work: read `@axiom/install-profiles` in full
  (`types.ts`/`constants.ts`/`composer.ts`) and
  `@axiom/capability-model/src/schemas.ts` in full before writing any
  capability/provider YAML.
- Resolve the `sdd`/`spec` provider-id naming gap explicitly (see
  "sdd/spec provider-id decision" below).
- Resolve the handler-package-placement question explicitly (see
  "Package placement decision" below).
- Implement the 15 "trivial" tools from the audit's table (rows 1, 2, 5,
  6, 7, 9, 10, 11, 12, 13, 15, 18, 19, 21, 22) as capability ids + handler
  functions in a new package, `@axiom/mcp-tools`.
- Attempt the "mostly trivial" row 17 (`list_recommended_context`) as a
  bonus if genuinely quick — it was (reuses `resolveMandatoryDocuments`
  verbatim, no new logic needed for the ALL-tags-present semantics the
  audit already flagged as the closest existing match).
- Write tests for the registration layer and each of the 16 implemented
  handlers (happy path + not-found/empty cases).
- Run real validation (`npm run typecheck`, `npm run build`, `npm test`,
  scoped + full monorepo) and compare against the audit's stated baseline.

## Non-goals

- The 5 "need new logic" tools (`get_workspace_equivalent`,
  `get_related_specs`, `list_recommended_skills`,
  `get_technical_context_document`, and the any-tag "recommended" variant
  underlying row 17's full gap) are explicitly deferred — not attempted in
  this pass. They may become their own follow-up increment.
- `get_implementation_context` (source doc §8) remains INC-14's scope,
  untouched here, per the roadmap and the audit's own framing.
- No change to `@axiom/isolation`'s MCP allowlist (`sdd`/`spec`/`serena`)
  — confirmed unnecessary by the audit, re-confirmed here (see "sdd/spec
  provider-id decision").
- No new MCP server process, no new transport, no wiring into an actual
  running MCP server binary — this increment registers the
  capability/handler data model and proves it round-trips through
  `routeTool`; a real MCP transport that calls `invokeMcpTool` at runtime
  is future work, not attempted here.
- `@axiom/tool-routing` and `@axiom/capability-model` source code is
  **not modified** — zero lines changed in either package, confirming the
  audit's "additive, not breaking" classification.
- Doctor `TR-005`+ extension (validator-reviewer's task per the audit) is
  not attempted here.

## Acceptance criteria

- [x] Pre-work reads completed: `@axiom/install-profiles` (`types.ts`,
      `constants.ts`, `composer.ts`) and `@axiom/capability-model/src/
      schemas.ts` read in full before writing any capability/provider
      YAML.
- [x] The `sdd`/`spec` provider-id naming gap is resolved explicitly, with
      rationale grounded in real schema constraints (not silently
      assumed).
- [x] The handler-package-placement decision is made explicitly, grounded
      in avoiding a dependency-graph mess.
- [x] All 15 "trivial" tools from the audit's table are implemented as
      capability ids + handler functions.
- [x] Row 17 (`list_recommended_context`, "mostly trivial") is also
      implemented — 16 total capability ids registered.
- [x] The 5 "need new logic" tools are explicitly listed as deferred, not
      implemented.
- [x] Tests exist for the registration layer (`registry.ts`) and every
      handler (happy path + not-found/empty cases where relevant).
- [x] A round-trip test proves the new capability ids load through the
      real, unmodified `loadCapabilityModel` and resolve through the
      real, unmodified `routeTool` (not just handler-level unit tests).
- [x] `npm run typecheck`, `npm run build`, and `npm test` were run for
      the full monorepo; results are compared against the audit's stated
      baseline (~12 failed files / 13 failed tests / ~1440 passed) with
      any delta explained.
- [x] `@axiom/tool-routing` and `@axiom/capability-model` source files are
      unmodified (confirmed by scoping all changes to new/other files).
- [ ] validator-reviewer confirms the registration against the doctor's
      `TR-001..004` contract and decides on `TR-005`+ extension (next
      step, not this increment's closure gate).

## Open questions

None blocking further implementation. One item is explicitly left for
validator-reviewer (not a blocker for closing this increment as
`pending`, since the roadmap's own subagent sequencing places doctor
extension after `mcp-tool-implementer`): should `TR-005`+ load the real
`examples/*.example.yaml` fixtures from `@axiom/mcp-tools` (renamed to
real `.yaml` in a real project scope) to smoke-test the round-trip in
`@axiom/doctor` itself, or keep that coverage purely in
`@axiom/mcp-tools`'s own test suite? This increment does not decide it —
flagged for validator-reviewer.

## Assumptions

- The `capabilities.yaml`/`providers.yaml` pair this increment produces
  are documented **examples** under `packages/mcp-tools/examples/`, not
  live files under `Axiom/axiom.spec/config/` — because `axiom.spec/`
  does not exist at the `Axiom` monorepo root at all, and
  `loadCapabilityModel(specScopePath)`'s only real runtime call site
  (`apps/cli/src/commands/start.ts`) always resolves `specScopePath`
  against an **end-user project's** `<rootPath>/axiom.spec/`, never this
  monorepo's own root. No installer/scaffold code path
  (`@axiom/installer`, `@axiom/document-bootstrap`) writes
  `capabilities.yaml`/`providers.yaml` content to a real project either
  (confirmed by direct read — `document-bootstrap`'s `ProvidersYaml`
  interface only consumes `providers[*].id` for `AGENTS.md` variable
  rendering). Writing a "real" file at a path nothing in this monorepo
  reads would be inert, speculative infrastructure — against
  `Axiom.SDD/AGENTS.md`'s explicit bootstrap limits ("no scripts that do
  not exist yet"). The example files are proven correct by a real
  round-trip test that copies them into a temp `specScopePath` and calls
  the actual `loadCapabilityModel`/`routeTool` (see
  `packages/mcp-tools/tests/capability-routing-roundtrip.test.ts`), which
  is the strongest validation available without inventing a fake
  production call site.
- `CANONICAL_CAPABILITY_IDS` (currently 16 ids, "HARD-CODED, changeable
  only via ADR" per `constants.ts`'s own header) is **not edited** by
  this increment. The 16 new MCP-tool capability ids are NOT added to
  that constant. Rationale: `CapabilityModelSchema` validates capability
  ids by regex only (`CAPABILITY_ID_REGEX`), not against
  `CANONICAL_CAPABILITY_IDS` — the loader's explicit-map branch
  (`loader.ts`'s `else` branch, used whenever `capabilities.yaml` is the
  explicit-map shape rather than the compliance-lists shape) builds
  `CapabilityDefinition`s directly from whatever ids the YAML declares,
  with no cross-check against the constant. `CANONICAL_CAPABILITY_IDS` is
  only consulted by the loader's OTHER branch (compliance-lists shape)
  and by `apps/cli/src/commands/capability.ts`'s `axiom capability
  list`/`resolve` (hardcoded against the 16-id constant, confirmed by
  direct read). Editing that constant would be a real, ADR-gated
  taxonomy change with a hardcoded-16 assumption baked into
  `capability.ts`'s docstrings/behavior — out of scope for "register new
  MCP tools." The new ids are real, functional, and route correctly
  (proven by the round-trip test); they are simply not yet surfaced by
  `axiom capability list` (a pre-existing tool that would need its own,
  separate update to stop hardcoding "16" if this taxonomy is meant to
  grow — flagged as a finding, not fixed here since it is unrelated to
  MCP tool routing itself).
- `invokeMcpTool`/`MCP_TOOL_HANDLERS` (the handler-dispatch layer) is
  deliberately decoupled from `routeTool` (the provider-resolution
  layer) — see `packages/mcp-tools/src/types.ts`'s header comment. No
  real MCP server transport exists yet in this monorepo to wire the two
  together end-to-end at the process level; that wiring is future work
  once an actual MCP server binary exists to receive tool calls.

## Implementation notes

### sdd/spec provider-id decision

**Decision: map `sdd`/`spec` (the MCP server names from `@axiom/isolation`'s
allowlist) onto the existing `axiom-gateway` capability-model provider
id. No new canonical provider ids are created.**

Rationale, grounded in code read during pre-work (not the audit's
speculation):

- `packages/capability-model/src/schemas.ts`'s `ProviderRegistrySchema`
  enforces `providers: z.array(ProviderDefinitionSchema).length(7, ...)`
  — **exactly 7 providers**, no more, no fewer. It also `.refine()`s that
  every provider `id` is a member of `CANONICAL_PROVIDER_IDS` (closed,
  hard-coded per `constants.ts`'s own header: "changeable only via ADR").
  Minting `sdd`/`spec` as two new literal provider ids would require (a)
  an ADR to grow `CANONICAL_PROVIDER_IDS` from 7 to 9, AND (b) changing
  the schema's `.length(7)` to `.length(9)` — a real, non-trivial change
  to `capability-model`'s locked contract, not a "just add a row" data
  change.
- `axiom-gateway`'s declared `kind: 'project-scoped-broker'` is exactly
  the right semantic: a broker for spec/sdd-domain reads, which is what
  the `sdd`/`spec` MCP servers do per `@axiom/isolation`'s own comments
  (`'sdd'` = "solo lectura", `'spec'` = "escritura brokerizada"). Every
  existing fixture in this repo
  (`packages/capability-model/tests/loader.test.ts`) already routes
  `spec.read`-class capabilities through `axiom-gateway` — this
  increment's new capability ids follow the exact same, already-proven
  pattern rather than inventing a new one.
- `@axiom/isolation`'s `checkMcpAllowed` and `@axiom/tool-routing`'s
  provider-id resolution are two independently-operating mechanisms
  today (confirmed by the audit's own direct read of `p0.ts`): the
  isolation allowlist checks MCP **server names** (`sdd`/`spec`/`serena`)
  against a caller-supplied override list; `routeTool` resolves
  capability-model **provider ids** against `applicableCapabilities`.
  Nothing requires them to share literal string values — `serena` being
  spelled identically in both lists is a coincidence of naming, not a
  structural coupling. Mapping `sdd`/`spec` onto `axiom-gateway` does not
  touch or weaken the isolation allowlist at all; `DEFAULT_ALLOWED_MCP_SERVERS`
  remains `['sdd', 'spec', 'serena']`, unmodified.
- Lower-friction path, per `constants.ts`'s own stated governance policy:
  adding to `applicableCapabilities` on an existing provider is a pure
  data change requiring no ADR; minting new provider ids is an ADR-gated
  taxonomy change. Given the tools being registered are read/validation
  operations with no distinct routing behavior of their own, there is no
  functional reason to pay the ADR cost.

See `packages/mcp-tools/examples/README.md` and
`providers.example.yaml` for the concrete registration this decision
produces.

### Package placement decision

**Decision: new package `@axiom/mcp-tools`, depending on
`@axiom/tool-routing`, `@axiom/capability-model`, `@axiom/workflow`,
`@axiom/skills`, `@axiom/technical-context`, `@axiom/user-workspace`, and
`@axiom/cli-commands`. Neither `@axiom/tool-routing` nor
`@axiom/capability-model` gained any new dependency.**

Rationale:

- Confirmed by reading both packages' `package.json`s during pre-work:
  `@axiom/capability-model` depends on `@axiom/filesystem-truth`,
  `@axiom/config-validation`, `@axiom/isolation`, `@axiom/project-resolution`
  — zero domain packages. `@axiom/tool-routing` depends on
  `@axiom/capability-model`, `@axiom/install-profiles`, `@axiom/isolation`,
  `@axiom/project-resolution`, `@axiom/telemetry` — also zero domain
  packages. Making either depend on `@axiom/workflow`/`@axiom/skills`/
  `@axiom/technical-context`/`@axiom/user-workspace` to reach the backing
  functions for the 16 new tools would invert this clean separation for
  no benefit — `routeTool` never needs to call a handler function itself,
  it only needs a capability id string to exist in the loaded model.
- A NEW package above both, importing from the domain packages and
  registering with the (unmodified) `@axiom/tool-routing`/
  `@axiom/capability-model` surface, keeps the dependency graph a strict
  DAG with `mcp-tools` as a new top-level consumer, not a modification to
  any existing package's dependency set.
- For the three CLI-backed tools (`validate_changes`, `rebuild_indexes`,
  `validate_indexes`), the backing functions live in
  `apps/cli/src/commands/{validate-changes,index-cmd}.ts` — an **app**,
  not a library (`apps/cli/src/index.ts` has zero exports; apps are not
  meant to be imported as dependencies). The monorepo already has an
  established seam for exactly this situation:
  `@axiom/cli-commands` — a barrel package that re-exports specific `runX`
  functions from `apps/cli/src/commands/*` verbatim (see its own
  `index.ts` header comment, "AD-01: el barrel NO añade lógica"),
  originally built so `@axiom/tui` never imports from `apps/cli/src/...`
  directly. This increment extends that exact seam (adding
  `runIndexRebuild`/`runIndexValidate`/`runValidateChanges` to the
  existing barrel, same pattern as the pre-existing
  `configure`/`sync`/`upgrade`/`model`/`components` re-exports) rather
  than inventing a second, parallel mechanism. `@axiom/mcp-tools` depends
  on `@axiom/cli-commands` for those three tools only; it never imports
  `apps/cli/src/...` directly.
- `invokeMcpTool`(capabilityId, input)` in `@axiom/mcp-tools` is the
  generic dispatch entry point a future MCP server transport would call;
  each handler is also individually exported and typed for direct,
  non-generic use.

## Files changed

New package `packages/mcp-tools/`:

- `package.json`, `tsconfig.json` — new package scaffold, dependencies:
  `@axiom/capability-model`, `@axiom/cli-commands`, `@axiom/core`,
  `@axiom/skills`, `@axiom/technical-context`, `@axiom/tool-routing`,
  `@axiom/user-workspace`, `@axiom/workflow`.
- `src/types.ts` — `McpToolResult`/`McpToolError`/`McpToolHandler`/
  `McpToolRegistryEntry`.
- `src/registry.ts` — `MCP_TOOL_CAPABILITY_IDS` (16 ids),
  `MCP_TOOL_HANDLERS` (built registry), `invokeMcpTool` (generic
  dispatch).
- `src/registry-handlers.ts` — `getProjectRegistry`, `getProjectRepos`
  (backing: `@axiom/user-workspace`'s `listProjectsV2`/`getProjectV2`).
- `src/artifact-handlers.ts` — `getPlan`, `getIncrement`, `getBug`,
  `getAdrIndex`, `getDecisions`, `getAllowedWriteScope` (backing:
  `@axiom/workflow`'s `loadArtifactMetadata`/`listArtifacts`).
- `src/skills-handlers.ts` — `getSkillIndex`, `getSkillCatalog`,
  `getSkill` (backing: `@axiom/skills`'s `loadSkillsRoleIndex`/
  `loadSkillsCatalog`/`getSkillById`).
- `src/technical-context-handlers.ts` — `getTechnicalContextIndex`,
  `listRecommendedContext` (backing: `@axiom/technical-context`'s
  `loadTechnicalContextIndex`/`resolveMandatoryDocuments`).
- `src/cli-backed-handlers.ts` — `rebuildIndexes`, `validateIndexes`,
  `validateChanges` (backing: `@axiom/cli-commands`'s re-exported
  `runIndexRebuild`/`runIndexValidate`/`runValidateChanges`).
- `src/index.ts` — barrel.
- `tests/*.test.ts` (7 files, 41 tests) — registry contract tests,
  per-handler happy-path + not-found/empty tests, and one end-to-end
  round-trip test through the real `loadCapabilityModel`/`routeTool`.
- `examples/README.md`, `examples/capabilities.example.yaml`,
  `examples/providers.example.yaml` — documented reference registration
  (see "Assumptions" for why these are examples, not live files).

Modified (existing files, additive changes only):

- `packages/cli-commands/package.json` — added `@axiom/topology`,
  `@axiom/workflow` dependencies.
- `packages/cli-commands/tsconfig.json` — added `paths`/`references` for
  `@axiom/topology`/`@axiom/workflow`; added
  `apps/cli/src/commands/index-cmd.ts` and
  `apps/cli/src/commands/validate-changes.ts` to `include`.
- `packages/cli-commands/src/index.ts` — added re-exports of
  `runIndexRebuild`/`runIndexValidate`/`runValidateChanges` and their
  arg/result types, following the file's own established `AD-01..04`
  pattern (comment block added documenting the new seam usage).
- `tsconfig.json` (root) — added `{ "path": "packages/mcp-tools" }` to
  `references`.
- `vitest.config.ts` (root) — added `@axiom/mcp-tools` and
  `@axiom/technical-context` (pre-existing package, was missing its
  alias entry — needed transitively since `@axiom/mcp-tools` imports it)
  to `resolve.alias`.

Not modified: `packages/tool-routing/**`, `packages/capability-model/**`,
`packages/isolation/**`, `packages/doctor/**` — zero lines changed in any
of the four packages the audit examined.

## Validation

Ran directly (not the AGENTS.md fallback statement — real commands
exist and were run):

- `npx tsc -b packages/mcp-tools` (scoped typecheck of the new package):
  clean, zero errors.
- `npx tsc -b packages/cli-commands` (scoped typecheck of the extended
  barrel): clean, zero errors.
- `npx tsc --noEmit` against a temp project covering
  `packages/mcp-tools/src/**` + `packages/mcp-tools/tests/**` with all
  transitive `@axiom/*` deps aliased (vitest's esbuild transform does not
  type-check test files by itself, so this step exists specifically to
  catch type errors `vitest run` would silently pass through — it caught
  one real issue, an unexported `CAPABILITY_ID_REGEX` import, fixed by
  inlining the regex in the test with a comment explaining why): clean
  after the fix.
- `npx vitest run packages/mcp-tools` (scoped): **7 test files, 41 tests,
  all passing.**
- `npm run build` (full monorepo, `tsc -b`): clean, zero errors.
- `npm run typecheck` (full monorepo, `tsc -b`): clean, zero errors.
- `npm test` (full monorepo, `vitest run`), baseline established first
  via a clean run on the untouched tree: **baseline = 12 failed files /
  13 failed tests / 1440 passed** (matches the audit brief's stated
  recent baseline exactly). After this increment's changes: **12 failed
  files / 13 failed tests / 1481 passed** — same 12 pre-existing failing
  files (confirmed by name: `apps/cli/tests/start.test.ts`,
  `packages/agents/tests/catalog.test.ts`,
  `packages/doctor/tests/checks.test.ts`,
  `packages/model-routing/tests/{assignments,loader,opencode-projection,resolver}.test.ts`,
  `packages/project-resolution/tests/resolver.test.ts`,
  `packages/skills/tests/catalog.test.ts`,
  `packages/telemetry/tests/audit-trail-sink.test.ts`,
  `packages/toolchain/tests/repair-add-gitignore.test.ts`,
  `packages/tui/tests/driver.test.ts` — all pre-existing, tied to missing
  real `axiom.spec/` files at the monorepo root / environment flakiness,
  none touched by this increment), zero regressions, +41 new passing
  tests (1440 → 1481, exactly this increment's new test count).

## Result

Implemented `@axiom/mcp-tools`, a new thin registration/handler-dispatch
package with 16 registered capability ids (15 "trivial" tools from the
audit's table plus row 17's "mostly trivial" `list_recommended_context`,
attempted as a genuinely-quick bonus per the brief's discretion) backed
by existing, unmodified functions across `@axiom/workflow`,
`@axiom/skills`, `@axiom/technical-context`, `@axiom/user-workspace`, and
(via the pre-existing `@axiom/cli-commands` re-export seam)
`apps/cli/src/commands/{index-cmd,validate-changes}.ts`. Neither
`@axiom/tool-routing` nor `@axiom/capability-model` were modified — a
round-trip test proves all 16 new capability ids load through the real
`loadCapabilityModel` and resolve, non-degraded, through `routeTool` via
the existing `axiom-gateway` provider id. Two decisions the audit
explicitly left open were resolved with concrete rationale: `sdd`/`spec`
map onto `axiom-gateway` (not new provider ids), and handler functions
live in a new package above both `tool-routing` and every domain package
(not inside either). The 5 "need new logic" tools remain deferred, listed
explicitly for a future increment. Full monorepo build/typecheck are
clean; the full test suite shows zero regressions against the audit's
stated baseline, plus 41 new passing tests.

## General spec integration

No integration into `Axiom.Spec/general-spec.md` was performed in this
increment. Rationale: per `Axiom.SDD/AGENTS.md`'s documentation rules,
integration belongs at the point a concrete, working contract is fully
closed — this increment is `Status: pending` specifically because
validator-reviewer has not yet confirmed the registration against the
doctor's `TR-001..004` contract (the roadmap's own subagent sequencing
gate). Once validator-reviewer closes its pass, the stable knowledge
worth integrating is: (1) the `sdd`/`spec` → `axiom-gateway` provider-id
mapping decision (a reusable precedent for any future new capability
needing an `sdd`/`spec`-flavored provider), and (2) the
`@axiom/mcp-tools` package's existence and role as the handler-dispatch
layer above `tool-routing`/`capability-model` (a reusable extension
point for the 5 deferred tools and any future MCP tool). Both should be
added to `general-spec.md` as part of validator-reviewer's closure, not
duplicated here first.

## Next step recommendation

Proceed to `validator-reviewer` (the next subagent in the roadmap's
INC-13 sequence) with this increment's spec as its input. Concrete inputs
for that pass:

1. `packages/mcp-tools/src/registry.ts`'s `MCP_TOOL_CAPABILITY_IDS` (16
   ids) and `MCP_TOOL_HANDLERS` registry.
2. `packages/mcp-tools/examples/{capabilities,providers}.example.yaml`
   and the round-trip test
   (`packages/mcp-tools/tests/capability-routing-roundtrip.test.ts`) as
   the concrete artifact to validate against `TR-001..004`'s existing
   contract (per the audit's own finding: adding new capability ids
   requires zero edits to `TR-001..004`, since those checks only
   smoke-test `code.semanticNavigation` via a synthetic fixture).
3. The explicit open question above (whether a new `TR-005`+ should read
   `@axiom/mcp-tools`'s example fixtures or stay independent) for
   validator-reviewer to decide.
4. The 5 deferred "need new logic" tools
   (`get_workspace_equivalent`, `get_related_specs`,
   `list_recommended_skills`, `get_technical_context_document`, and the
   any-tag "recommended" variant) as a candidate list for a future,
   separate increment — validator-reviewer should confirm whether any of
   these are now unblocked or still correctly deferred.
