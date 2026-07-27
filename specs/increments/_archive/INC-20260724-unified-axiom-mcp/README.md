# Increment: MCP unificado de `<project>.axiom`

Status: closed
Date: 2026-07-24

## Goal

Give `<project>.axiom` a SINGLE MCP broker (`axiom`) that a code repo binds to
in order to: read spec/increment/bug/adr/technical-context, edit artifact
states, and push those changes — matching the user's model ("un mcp para
traerte info de spec/incremento/bugs/adrs/contexto técnico y poder editar
estados y subirlos"). For everything else the MCP isn't needed (working with
spec/increment/bug/adr/plans directly inside the axiom repo is done via
CLI/files, no MCP required).

This is **INC-A5** of Cluster A in the Axiom evolution plan
(`giggly-imagining-moore.md`, "Arquitectura objetivo" / "Un MCP expuesto por
`<project>.axiom`"), the LAST item of Cluster A, building on **INC-A1**
(topology `axiomRepo`/`codeRepos`/`legacyRepos`, schemaVersion 2), **INC-A2**
(adoption creates `<project>.axiom`, ends up `role: 'spec'`), and **INC-A3**
(provenance/lifecycle metadata + global migration manifest at
`<project>.axiom/migration/migration-manifest.yaml`). Explicit, user-approved
graduation to the full product lifecycle (`Axiom.SDD/AGENTS.md`'s "Explicit
Bootstrap Limits" exception, per the plan) — this feature is NOT down-scoped.

## Context

Today, `axiom.config/mcp-manifest.yaml` declares 2 brokers: `sdd`
(`sdd-mcp-server`, control-plane: registry/skills/write-scope/index reads +
`transitionApply`/`gitRoleBranch`/`gitCommitSync` mutations) and `spec`
(`spec-mcp-broker`, knowledge-plane: plan/increment/bug/adr/decision/
technical-context/implementationContext reads). A third, unlisted `memory`
kind exists in code (`packages/mcp-server/src/tool-sets.ts`) for
`memory.decisionRecall`/`memory.contextRecall`. `packages/mcp-tools/src/
registry.ts` is a CLOSED registry of 22 capability ids (`MCP_TOOL_CAPABILITY_IDS`),
each with a single handler; `packages/mcp-server/src/tool-sets.ts` derives each
server kind's exposed tool set as "every capability id whose domain prefix
matches the kind" (`sdd.*` → `sdd` kind, `spec.*` → `spec` kind, `memory.*` →
`memory` kind) — a strict 1:1 domain↔kind mapping.

For a code repo bound to `<project>.axiom`, needing to launch/bind to TWO
separate broker processes (`sdd` for control + `spec` for knowledge, or a
3rd for memory) is unnecessary ceremony once `<project>.axiom` is a SINGLE
repo (INC-A1/A2): the user's model is one MCP per axiom repo. INC-A2 also
introduced a documented quirk: an adopted `<project>.axiom` repo's merged
control+spec write deterministically ends up `axiom.yaml#role: 'spec'`
(verified as CORRECT, not just tolerable, by INC-A2's own Assumption #2) —
this is what `resolveSpecRelPathForScope`/`packages/mcp-server/src/context.ts`
(`resolveMcpServerContext`'s `specArtifactRelPath` derivation,
INC-20260713-fix-spec-scope-path) already require to resolve artifacts at the
scope's ROOT (no `axiom.spec` nesting) — this increment's unified broker must
compose with that EXISTING resolution unchanged, not reimplement it.

Three read domains are missing entirely from the MCP surface: the topology
manifest itself (`axiomRepo`/`codeRepos`/`legacyRepos`, INC-A1),
the global migration manifest (INC-A3, `<project>.axiom/migration/
migration-manifest.yaml`), and a derived "is this project adopted" summary.

## Scope

- `packages/mcp-tools/src/types.ts`: widen `McpToolRegistryEntry.domain` to
  add `'axiom'` (a 5th capability domain, for the 3 brand-new read tools
  only — NOT a re-classification of any existing `sdd.*`/`spec.*` id).
- NEW `packages/mcp-tools/src/topology-handlers.ts`: `getTopology`
  (`axiom.topologyRead`), backed by `@axiom/topology`'s `loadTopology`. NEW
  dependency: `@axiom/topology` added to `packages/mcp-tools/package.json` +
  `tsconfig.json` (paths/references) — `@axiom/mcp-server` already depends on
  it; this extends `@axiom/mcp-tools`'s existing dependency graph one hop,
  no new architectural layer.
- NEW `packages/mcp-tools/src/migration-manifest-handlers.ts`: a minimal,
  OWN (not reused via `apps/cli`) permissive YAML reader for
  `<specScopeAbsolutePath>/migration/migration-manifest.yaml` (same rel-path
  convention as `apps/cli/src/bootstrap-shared/migration-manifest.ts`'s
  `MIGRATION_MANIFEST_REL_PATH`, deliberately NOT imported — see Assumptions),
  plus `getMigrationManifest` (`axiom.migrationManifestRead`) and
  `getAdoptionState` (`axiom.adoptionStateRead`, composing `loadTopology` +
  the same manifest reader — NO new phase/state machine, pure derivation).
- `packages/mcp-tools/src/registry.ts`: register the 3 new capability ids +
  handlers (22 → 25 total).
- `packages/mcp-tools/src/index.ts`: barrel exports for the 2 new handler
  modules.
- `packages/mcp-tools/examples/{capabilities,providers}.example.yaml` +
  `README.md`: add the 3 new ids (domain `axiom`, mapped onto the existing
  `axiom-gateway` provider id, same as every other non-memory id) so
  `capability-routing-roundtrip.test.ts` keeps proving the full registered
  set round-trips with zero `tool-routing`/`capability-model` source changes.
- `packages/mcp-server/src/tool-sets.ts`: widen `McpServerKind` to
  `'sdd' | 'spec' | 'memory' | 'axiom'`; add `AXIOM_TOOL_CAPABILITY_IDS` (the
  UNION: all `spec.*` reads + the explicit write subset
  `sdd.transitionApply`/`sdd.gitCommitSync` + the 3 new `axiom.*` reads) via a
  dedicated union branch in `capabilityIdsForKindRaw` (NOT the generic
  single-domain filter used by `sdd`/`spec`/`memory`); 3 new `TOOL_DESCRIPTIONS`
  entries.
- `packages/mcp-server/src/server.ts`: `DEFAULT_SERVER_NAME` gains
  `axiom: 'axiom-mcp-broker'`.
- `packages/mcp-server/src/input-builders.ts`: 3 new input builders
  (`axiom.topologyRead` pins `projectRoot`; `axiom.migrationManifestRead`
  pins `specScopeAbsolutePath`; `axiom.adoptionStateRead` pins BOTH), same
  cross-project-pinning contract as every existing builder.
- `apps/cli/src/commands/mcp-serve.ts`: `--kind` accepts `axiom` (CLI-level
  validation + help text); `runMcpServe`'s `kind` param type is
  `@axiom/mcp-server`'s own `McpServerKind` (no local duplication, widens
  automatically).
- `axiom.config/mcp-manifest.yaml`: add a 3rd entry (`id: axiom, server:
  axiom-mcp-broker, projectBinding: required`). `sdd`/`spec` entries
  UNCHANGED (back-compat, decision #4).
- `apps/cli/src/commands/workspace-mcp.ts`: widen `buildMcpServeArgs`'s kind
  param to `'sdd' | 'spec' | 'axiom'`; add `AXIOM_MCP_BROKER_ID` constant +
  a new, additive `buildAxiomMcpBrokerEntry` helper (mirrors
  `buildEngramNativeServer`'s shape) for a future caller that materializes
  the unified broker's native config for a managed `<project>.axiom` repo.
  `buildWorkspaceMcpServers`'s existing sdd+spec derivation is UNCHANGED (see
  Non-goals — NOT rewiring the adoption-time MCP-generation default).
- `apps/cli/src/commands/workspace-config-scaffold.ts`: `DEFAULT_MCP_MANIFEST`
  (the scaffold template written into every NEWLY adopted project, documented
  as intentionally mirroring `Axiom/axiom.config/mcp-manifest.yaml`'s own
  shape) gains the same 3rd `axiom` entry, to stay consistent with its own
  header comment's stated contract.
- `apps/cli/src/commands/member-install.ts`: the per-manifest-id dispatch
  loop (`entry.id === 'sdd' | 'spec'` → native launch entry) gains an
  `'axiom'` branch (`--project-root` = `controlAbsolutePath`, the same
  resolved path the `sdd` branch already uses) — prevents a self-inflicted
  "unknown kind" warning-noise regression once `mcp-manifest.yaml` declares a
  3rd id.
- Tests: guard-test lockstep (`registry.test.ts`'s tool-count/domain-set
  assertions, `workspace-config-scaffold.test.ts`'s 2-id assertion); new unit
  tests for the 2 new handler modules; a new `describe` block in
  `packages/mcp-server/tests/server.test.ts` for the `axiom` kind; a new e2e
  (`packages/mcp-server/tests/axiom-broker.e2e.test.ts`) proving: tools/list
  union, `spec.incrementRead`, `sdd.transitionApply` preview+confirm,
  `sdd.gitRoleBranch` REJECTED (not in the axiom broker's set — proves the
  write subset is exactly 2 tools, not all of `sdd.*`), the 3 new reads, and
  the `role: 'spec'` dedicated-repo convergence (mirrors
  `spec-scope-convergence.test.ts`'s fixture) for the `axiom` kind
  specifically; a new `member-install.test.ts` scenario proving a 3rd
  manifest id materializes an `axiom-mcp-broker` native entry with no
  warning.

## Non-goals

- Removing/deprecating the `sdd`/`spec` brokers — they keep working AS-IS
  (decision #4). No entries removed from `mcp-manifest.yaml`; no capability
  ids removed from `MCP_TOOL_CAPABILITY_IDS`.
- Folding Engram/memory MCP into the unified broker — `memory.decisionRecall`/
  `memory.contextRecall` stay on their own separate `memory` kind/process
  (decision #3).
- Rewiring `runWorkspaceSetup`/`buildWorkspaceMcpServers`'s ADOPTION-TIME
  default MCP-generation to prefer the unified broker for a managed
  (schemaVersion 2) topology — a materially larger, separate change (would
  touch the control/spec `WorkspaceRepoSpec` distinction itself). This
  increment adds the PLUMBING (`buildAxiomMcpBrokerEntry`, widened
  `buildMcpServeArgs`, the manifest entry, `member-install.ts`'s dispatch
  branch) without rewiring that default — a future increment can wire
  adoption to prefer/emit the axiom broker when it revisits that flow.
- A new adoption/migration phase state machine for `axiom.adoptionStateRead`
  — it is a pure DERIVATION from the EXISTING topology (`legacyRepos[]`
  presence) + the EXISTING migration manifest (INC-A3), never a new
  persisted state.
- Worktrees (Cluster W) or provider/code-intel changes (Cluster T) — out of
  this cluster.
- Migrating Axiom's own `Axiom.SDD`/`Axiom.Spec` repos to the managed model
  (explicitly deferred to the user, per the plan).
- Touching `axiom.config/{capabilities,providers}.yaml` (Axiom's OWN, real
  top-level config) — verified these are NOT kept in lockstep with
  `MCP_TOOL_CAPABILITY_IDS` today (the 3 pre-existing git-service/transition
  ids are already absent from them with no doctor failure), so this
  increment does not start a new precedent either; only the
  `packages/mcp-tools/examples/*.example.yaml` TEST fixtures are updated.
- Integration into `Axiom.Spec/specs/00_Resumen_Ejecutivo.md … 08_Glosario.md`
  — per explicit task instruction, deferred to the outer autopilot
  orchestrator's own end-of-Cluster-A consolidation step (see "General spec
  integration" below).

## Acceptance criteria

- [x] A new `axiom` broker/kind exists, exposing exactly the union: all
      `spec.*` reads + `sdd.transitionApply` + `sdd.gitCommitSync` + the 3
      new `axiom.*` reads (25-tool registry total; the `axiom` kind itself
      exposes a strict SUBSET: union size, not 25).
- [x] `axiom.topologyRead`, `axiom.migrationManifestRead`,
      `axiom.adoptionStateRead` are implemented, registered, and return
      correct data against real fixtures.
- [x] `sdd`/`spec`/`memory` kinds are BYTE-IDENTICAL in tool-set membership
      to before this increment (proven by the pre-existing, constant-driven
      `tools/list` assertions in `server.test.ts` continuing to pass
      unchanged).
- [x] `sdd.gitRoleBranch` is NOT exposed by the `axiom` kind (proves the
      write subset is exactly 2 tools, not "all sdd mutations").
- [x] The unified broker resolves the spec scope correctly for a
      `role: 'spec'` dedicated `<project>.axiom` repo (artifacts at the
      scope root, no `axiom.spec` nesting) — proven via a convergence test
      mirroring `spec-scope-convergence.test.ts`, with ZERO changes to
      `context.ts`'s existing `specArtifactRelPath` derivation (already
      correct for this case per INC-A2/INC-20260713-fix-spec-scope-path).
- [x] `axiom.config/mcp-manifest.yaml` declares the 3rd `axiom` id; `sdd`/
      `spec` entries unchanged; `axiom mcp list|validate|inventory` (real
      manifest regression test) keeps passing.
- [x] `npm run build` (tsc -b) passes.
- [x] Targeted `vitest run` (`packages/mcp-tools`, `packages/mcp-server`,
      `apps/cli/tests` MCP/adapter-relevant files) passes; every failure
      classified pre-existing vs new.
- [x] Guard-test lockstep: `registry.test.ts`'s hard-coded tool count/domain
      set, `workspace-config-scaffold.test.ts`'s 2-id assertion, both updated.
- [x] Unit tests: unified broker exposes the union + 3 new tools; sdd/spec
      still expose their original sets; the 3 new reads return correct data
      (topology, migration manifest, adoption-state derived summary),
      including absent-file "not adopted" defaults.
- [x] e2e: from a code-repo-style client bound to the unified broker, read an
      increment + apply a transition (preview then confirm) + read
      topology/migration-manifest/adoption-state, over the REAL JSON-RPC
      dispatcher (`createMcpServer`).

## Open questions

None blocking. All "FINAL decisions" in the task brief are locked in
verbatim (unified broker name `axiom`, union tool-set, 3 new tools' domains,
back-compat via coexistence, memory stays separate). Two genuinely-open
implementation-level choices are resolved narrowly under Assumptions below
(existing-precedent-first, per the executor's guardrails): (1) whether to
reuse `apps/cli/src/bootstrap-shared/migration-manifest.ts`'s richer
load/save module via `@axiom/cli-commands`'s single-ownership seam, or write
a minimal, standalone reader in `@axiom/mcp-tools`; (2) whether/how far to
extend the adoption-time scaffold/member-install plumbing given the brief's
Ficheros list only names `axiom.config/mcp-manifest.yaml` +
`workspace-mcp.ts` explicitly.

## Assumptions

1. **Minimal, standalone migration-manifest reader in `@axiom/mcp-tools`,
   NOT reused via `@axiom/cli-commands`.** `apps/cli/src/bootstrap-shared/
   migration-manifest.ts` is a rich read/write module (`saveMigrationManifest`,
   `recordMigrationRun`, `upsertMigrationRun`, `generateMigrationId`) that
   could be re-exported through `@axiom/cli-commands`'s existing
   single-ownership seam (the SAME pattern `cli-backed-handlers.ts` already
   uses for `index-cmd.ts`/`validate-changes.ts`). Doing so would require
   adding a NEW kind of include-list entry (a non-`commands/`-directory file)
   to `cli-commands/tsconfig.json` + a matching `apps/cli/tsconfig.json`
   exclude entry — a real, cross-cutting, easy-to-get-wrong touch to the
   exact "single-ownership" wiring this increment's own gotchas warn about,
   for a READ-ONLY need. Instead, `migration-manifest-handlers.ts` implements
   its OWN ~15-line permissive YAML load (mirroring the EXACT file-path
   convention, never importing `apps/cli` source), matching this codebase's
   established per-domain-loader precedent (`@axiom/topology`'s own loader,
   `mcp.ts`'s `loadMcpManifest`) — narrowest change, zero risk to the
   cli-commands/apps-cli build wiring, and a nice least-privilege side
   effect (a read tool cannot reach the write functions
   `saveMigrationManifest`/`recordMigrationRun` even in principle).
2. **`axiom.migrationManifestRead`/`axiom.adoptionStateRead` read the
   PERSISTED manifest verbatim, never re-scan artifacts fresh.** The task
   brief says "adoptionStateRead ... migration manifest counts — DERIVE from
   existing topology + manifest; do NOT build a new phase state machine."
   The manifest's own `artifacts[]`/`summary` are already a fresh rescan AS
   OF THE LAST MIGRATION RUN (INC-A3's own accepted limitation, its
   Assumption #7) — re-deriving a THIRD independent live rescan here (via
   `@axiom/workflow`'s `listArtifacts` + `resolveArtifactOrigin`, both
   already available to `@axiom/mcp-tools`) would exceed "derive from ...
   manifest" and duplicate `computeMigrationArtifactsSnapshot`'s logic in a
   3rd place. Absent/malformed manifest → `data: null` (matching the CLI
   reader's own "never throws, defaults to empty" posture and this
   package's "absent is not an error" convention).
3. **`adopted := legacyRepos.length > 0`.** INC-A2's `recordLegacyReposInTopology`
   runs unconditionally after `runWorkspaceSetup` whenever `--adopt-spec`/
   `--adopt-sdd` are supplied (NOT gated on the migration having created
   something new, unlike the manifest's `migrations[]` entries) — making
   `legacyRepos[]` presence the more reliable "was this project adopted"
   signal than `migrations.length > 0`, and it directly matches the brief's
   own wording ("is this project adopted? legacyRepos present").
4. **The 3 new tools get their OWN capability domain (`'axiom'`), not folded
   into `spec.*`/`sdd.*`.** Matches this package's own established precedent
   (`tool-sets.ts`'s header comment on why `memory` got its own kind: "a
   dedicated kind keeps ... aligned with server KIND, matching the
   pre-existing sdd/spec split's own precedent — one kind per coherent
   capability domain"). Topology/migration-manifest/adoption-state form a
   coherent, NEW domain (project-lifecycle-plane reads) distinct from
   `spec.*`'s artifact-content reads and `sdd.*`'s workflow/skill/write-scope
   operations.
5. **The `axiom` KIND's tool set is an explicit UNION, not a domain filter.**
   Unlike `sdd`/`spec`/`memory` (each a straight `entry.domain === kind`
   filter), `axiom` draws from 3 sources (`spec.*` domain in full,
   `axiom.*` domain in full, and an explicit 2-id SUBSET of `sdd.*`) — this
   needs its own branch in `capabilityIdsForKindRaw`, not a change to the
   generic `capabilityIdsForDomain` helper (which stays exactly as-is,
   preserving `sdd`/`spec`/`memory`'s existing behavior byte-for-byte).
6. **`sdd.gitRoleBranch` is deliberately EXCLUDED from the axiom broker's
   write subset.** The brief's user model is "read + edit artifact states +
   push" — branch creation isn't part of that story (`gitCommitSync` already
   covers "push"); `gitRoleBranch` remains reachable via the `sdd` broker
   (back-compat, unchanged).
7. **`axiom.topologyRead`/`axiom.adoptionStateRead` pin `projectRoot`;
   `axiom.migrationManifestRead` pins `specScopeAbsolutePath`;
   `axiom.adoptionStateRead` pins BOTH.** Mirrors the exact per-field
   pinning precedent already established by `sdd.indexesRebuild`
   (`projectRoot`) and `spec.technicalContextIndexRead`
   (`specScopeAbsolutePath`) — no new pinning primitive invented.
8. **Adoption-time scaffold/member-install plumbing IS touched, narrowly.**
   The task's Ficheros list names `axiom.config/mcp-manifest.yaml` +
   `workspace-mcp.ts` explicitly but not `workspace-config-scaffold.ts`/
   `member-install.ts`. However, `workspace-config-scaffold.ts`'s
   `DEFAULT_MCP_MANIFEST` is DOCUMENTED (its own header comment) as
   intentionally mirroring `Axiom/axiom.config/mcp-manifest.yaml`'s shape,
   and `member-install.ts`'s per-id dispatch loop would otherwise emit a
   NEW, self-inflicted "unknown kind" warning for every future adopted
   project the moment the manifest gains a 3rd id — a foreseeable
   regression introduced BY this increment's own manifest change. Fixing
   both is a small (~10 line each), safe, narrowly-scoped consistency fix,
   not scope creep; it stops short of the larger non-goal (rewiring
   `buildWorkspaceMcpServers`'s adoption-time DEFAULT to prefer the axiom
   broker for managed topologies).
9. **`axiom-mcp-broker` is the new broker's `server`/display id**, mirroring
   `spec-mcp-broker`'s "-mcp-broker" suffix convention (the brief's own
   language: "a SINGLE MCP broker").

## Implementation notes

Files changed (all in `Axiom/`, the product monorepo, except this spec) —
see the final executor report for the complete, verbatim list; summarized
here by area:

- **`@axiom/mcp-tools`**: `types.ts` (domain union), NEW
  `topology-handlers.ts`, NEW `migration-manifest-handlers.ts`,
  `registry.ts` (25-tool registry), `index.ts` (barrel), `package.json` +
  `tsconfig.json` (`@axiom/topology` dependency), `examples/*.example.yaml` +
  `examples/README.md`.
- **`@axiom/mcp-server`**: `tool-sets.ts` (widened kind + union branch + 3
  descriptions), `server.ts` (`DEFAULT_SERVER_NAME`), `input-builders.ts` (3
  new builders).
- **`apps/cli`**: `mcp-serve.ts` (kind validation), `workspace-mcp.ts`
  (widened `buildMcpServeArgs` + `buildAxiomMcpBrokerEntry`),
  `workspace-config-scaffold.ts` (`DEFAULT_MCP_MANIFEST`), `member-install.ts`
  (axiom dispatch branch).
- **Config**: `axiom.config/mcp-manifest.yaml` (3rd entry).
- **Tests**: guard-test updates (`registry.test.ts`,
  `workspace-config-scaffold.test.ts`), 2 new handler unit-test files, a new
  `describe` block in `server.test.ts`, a new e2e
  (`axiom-broker.e2e.test.ts`), a new `member-install.test.ts` scenario.

## Validation

Ran from `C:\repos\Axiom Workspace\Axiom`:

- `npm run build` (`tsc -b`) — PASSED, clean, exit 0 (run twice, before and
  after the test additions).
- Targeted `npx vitest run packages/mcp-tools packages/mcp-server`: **18
  files, 151 tests passed** (includes the 2 new handler test files:
  `topology-handlers.test.ts` 3 tests, `migration-manifest-handlers.test.ts`
  10 tests, plus the new `axiom-broker.e2e.test.ts` 2 tests and the new
  `axiom` kind `describe` block inside `server.test.ts`).
- Targeted `npx vitest run` of the MCP/adapter-relevant `apps/cli/tests`
  files (`mcp.test.ts`, `mcp-real-manifest.test.ts`, `mcp-serve.test.ts`,
  `workspace-mcp.test.ts`, `native-mcp-config.test.ts`,
  `member-install.test.ts`, `workspace-config-scaffold.test.ts`,
  `e2e/workspace-mcp.e2e.test.ts`): **8 files, 79 tests passed**.
- Full `apps/cli/tests` sweep (broader required scope): **116 files, 1074
  tests passed** (after fixing one additional pre-existing-shaped guard
  test found by this run — see Result). `context.test.ts` and
  `workspace-setup.test.ts` (the brief's documented occasionally-flaky
  files under full-suite parallel load) both passed cleanly in this run
  (no timeout reproduced).
- Full-suite `npx vitest run` (all 295 files, 2972 tests) as an extra,
  broader regression check (this touched `@axiom/mcp-tools`/
  `@axiom/mcp-server`, foundational packages used across the CLI):
  **295/295 files, 2972/2972 tests passed, zero failures.** No pre-existing
  or new failures this run; no flakiness reproduced either.

## Result

Implemented. All acceptance criteria met. A new `axiom` MCP broker
(`axiom-mcp-broker`) exists as a `McpServerKind` alongside `sdd`/`spec`/
`memory`, exposing the UNION of every `spec.*` read + the explicit 2-tool
`sdd.*` write subset (`transitionApply`/`gitCommitSync`) + 3 brand-new
`axiom.*` project-lifecycle reads (`topologyRead`/`migrationManifestRead`/
`adoptionStateRead`), bringing the registry from 22 to 25 capability ids.
`sdd`/`spec`/`memory` are proven byte-identical to before (the existing,
constant-driven `tools/list` assertions needed zero changes to keep
passing). The `role: 'spec'` dedicated-repo spec-scope convergence
(INC-A2/INC-20260713-fix-spec-scope-path) was verified to already work
correctly for the new `axiom` kind with ZERO changes to `context.ts`. The
3 new reads compose cleanly with the EXISTING topology (INC-A1) and
migration-manifest (INC-A3) artifacts via a minimal, standalone,
read-only YAML loader in `@axiom/mcp-tools` (deliberately not reusing
`apps/cli`'s richer read/write module, to avoid touching the
cli-commands/apps-cli single-ownership build wiring for a read-only need
— see Assumptions #1). `axiom.config/mcp-manifest.yaml` now declares 3
ids; the adoption-time scaffold template (`DEFAULT_MCP_MANIFEST`) and
`member-install.ts`'s per-id native-config dispatch were both updated in
lockstep to avoid a self-inflicted "unknown kind" warning regression.

One additional pre-existing-shaped guard test was found and fixed beyond
what the brief anticipated: `apps/cli/tests/workspace-setup.test.ts`'s
"Scenario (m)" hard-asserted the scaffolded manifest's exact 2-id set
(`['sdd','spec']`) via `runWorkspaceSetup`'s own path (a second guard test
covering the same `DEFAULT_MCP_MANIFEST` constant already updated in
`workspace-config-scaffold.test.ts`) — updated to `['axiom','sdd','spec']`
in lockstep.

Deferred per Scope: rewiring `buildWorkspaceMcpServers`'s adoption-time
DEFAULT MCP generation to prefer the unified broker for managed
(schemaVersion 2) topologies — the plumbing (`buildAxiomMcpBrokerEntry`,
widened `buildMcpServeArgs`) is in place for a future increment to wire
that in without further groundwork.

## General spec integration

Integrado en la spec canónica al cierre del batch INC-20260724-* (integración
única de toda la tanda). Ficheros que recibieron el conocimiento de este
incremento:

- `06_Integraciones_y_Capacidades.md` — broker MCP unificado `axiom` (unión de
  reads `spec.*` + `transitionApply`/`gitCommitSync` + 3 reads `axiom.*`,
  25 tools), superseder aditivo del modelo de 2 brokers.
- `05_Interfaces_Operativas.md` — `axiom mcp serve --kind axiom` + 3er id en
  `mcp-manifest.yaml`.
- `07_Gobierno_y_Seguridad.md` — aislamiento MCP project-scoped preservado en
  el broker unificado; subconjunto de escritura de 2 tools.
- `08_Glosario.md` — broker MCP unificado `axiom` (`axiom-mcp-broker`).
- `01_Requisitos_Funcionales.md` — RF-AXM-043.
