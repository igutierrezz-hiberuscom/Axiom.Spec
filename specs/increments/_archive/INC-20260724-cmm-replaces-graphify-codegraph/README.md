# Increment: `cmm` sustituye a `graphify` y `codegraph` (ADR-0031)

Status: closed
Date: 2026-07-24

## Goal

Make `codebase-memory-mcp` (`cmm`) the SOLE structural code-intelligence
provider, REPLACING both `graphify` and `codegraph` (which must stop being
selectable/registered/routable anywhere in Axiom). `serena` remains the
symbolic provider (def/refs/rename), strongly separated from `cmm`
(structural/graph/blast-radius/dependencies/traces). Fallback ALWAYS applies
(`cmm → filesystem`, `serena → filesystem`, unchanged).

This is **INC-T1** of Cluster T in the Axiom evolution plan
(`giggly-imagining-moore.md`, "Decisiones tomadas" §1 and "Cluster T →
INC-T1"): independent of Cluster A, and a dependency of INC-W4 (per-worktree
provider isolation, not implemented here). The provider-set change is
explicitly user-approved ("cmm sustituye a graphify y codegraph, estos dos ya
no tienen que estar como seleccionables") — the closed set's own
"changeable only by ADR" rule (ADR-0021) is honored by authoring ADR-0031
rather than bypassing it.

## Context

Before this increment: `packages/capability-model/src/constants.ts` fixed
`CANONICAL_PROVIDER_IDS` at 7 ids (`filesystem`, `axiom-gateway`, `serena`,
`codegraph`, `graphify`, `engram`, `generated-snapshots`), enforced by
`ProviderRegistrySchema.providers.length(7)` and doctor's `CC-001`.
`codegraph` (`code.knowledgeGraph`) and `graphify` (`code.structureAnalysis`)
were two real, wired `ProviderClient`s
(`packages/providers/src/code-intel/{codegraph,graphify}-client.ts`), each
spawning its own local MCP server and maintaining its own local index format
(`.codegraph/`, `graphify-out/graph.json`), chained
`graphify → codegraph → serena` in `providers.yaml`. Neither had any
freshness/staleness concept — once `initializeCodeIntelIndexes`
(`apps/cli/src/commands/workspace-code-intel.ts`) built a local index at
setup time, nothing ever re-checked it.

`codebase-memory-mcp` has no local checkout/README in this environment (unlike
serena/codegraph/graphify, each confirmed against a real README when they were
wired). Its exact binary name and MCP tool surface are therefore a documented
ASSUMPTION (`codebase-memory-mcp`, subcommands `mcp`/`sync`, tools
`explore`/`query_structure`), deliberately kept overridable end-to-end
(launch command AND per-capability tool names) so a project can correct them
once the real CLI is confirmed, without a code change. This is recorded here
rather than silently asserted as fact.

## Scope

- **Provider-set change (ADR-0031)**: `packages/capability-model/src/
  {constants,schemas,types}.ts` — `CANONICAL_PROVIDER_IDS` 7→6
  (`codegraph`+`graphify` removed, `cmm` added); `ProviderRegistrySchema`
  closed to exactly 6; new `ProviderKind` value `'structural-code-intel'`
  (legacy `'knowledge-graph'`/`'repository-graph'` kept in the enum for
  backward-compat, unused by any current provider).
- **ADR**: `docs/0031-adr-cmm-replaces-graphify-and-codegraph.md` (new),
  following the repo's `docs/NNNN-*.md` convention (next number after 0030).
- **`cmm` provider client**: `packages/providers/src/code-intel/cmm-client.ts`
  (new) + `cmm-freshness.ts` (new, freshness/auto-sync) — mirrors
  `serena-client.ts`'s shape via the shared `createCodeIntelProviderClient`
  helper, serving BOTH `code.knowledgeGraph` and `code.structureAnalysis`.
  `codegraph-client.ts`/`graphify-client.ts` DELETED (clean removal, per the
  task's stated preference over "present-but-unwired").
- **Registration/selection/native-launch**: `register.ts`, `selectable.ts`
  (`SELECTABLE_PROVIDER_IDS`/`CODE_INTEL_PROVIDER_IDS` 3→2 code-intel ids: `cmm`
  + `serena`), `native-mcp-launch.ts` (`cmmMcpArgs` replaces
  `codegraphMcpArgs`/`graphifyMcpArgs`), `project-registry.ts`,
  `packages/providers/src/index.ts` barrel.
- **Config**: `axiom.config/providers.yaml` (registry + fallback:
  `cmm → filesystem`), `axiom.config/toolchain-catalog.yaml`,
  `axiom.config/profiles.yaml`, root `axiom.yaml` (Axiom's own dogfood
  narrative `capabilities.tools` list).
- **Routing**: no source changes to `packages/tool-routing/src/*`
  (dispatcher/select/fallback) — routing is entirely data-driven from
  `providers.yaml`/install profiles; only the DATA changed. New unit tests
  added proving `code.knowledgeGraph`/`code.structureAnalysis` resolve to
  `cmm`.
- **Freshness**: `packages/providers/src/code-intel/cmm-freshness.ts` (new) —
  a per-project `.cmm/sync-state.json` marker, `evaluateCmmFreshness`
  (fresh/stale/unknown vs. a max-age window), `ensureCmmFreshness`
  (best-effort auto-sync before a structural invocation, wired into
  `cmm-client.ts`). `apps/cli/src/commands/workspace-code-intel.ts`'s
  init-on-setup hook writes the SAME marker on a successful sync
  (`initializeCodeIntelIndexes`'s `cmm` branch replaces the old
  `codegraph`/`graphify` branches). `packages/toolchain/src/probe.ts`:
  `hasRealCmmIndex` replaces `hasRealCodegraphDb`.
- **No git hooks**: confirmed — sync only happens from the setup-time init
  hook or the per-invocation freshness pre-check; nothing is wired to a git
  hook anywhere in this change.
- **Doctor/toolchain lockstep**: `packages/doctor/src/checks.ts` (`CC-001`
  provider count 7→6); ALL guard tests across
  `packages/{capability-model,providers,tool-routing,doctor,install-profiles,
  installer,document-bootstrap,toolchain}/tests` and
  `apps/cli/tests/{capability,configure,context,document-bootstrap,gateway,
  workspace-code-intel,workspace-config-scaffold,workspace-mcp,
  workspace-setup}.test.ts` updated to the new 6-provider set / 2 code-intel
  ids.
- **Skill/agent catalog bundleHash lockstep (gotcha honored)**: 5 skill
  source `.md` files (`axiom.spec/target-axiom-skills/
  {axiom-capability-router,axiom-code-intelligence,axiom-role-implementer,
  axiom-role-planner,axiom-tech-context}.md`) and 3 agent source `.md` files
  (`axiom.spec/target-axiom-agents/{axiom-role-implementer,axiom-role-planner,
  axiom-tech-context}.md`) mention `codegraph`/`graphify` in prose;
  updated to `cmm`, and their `bundleHash` in `axiom.config/
  {skills,agents}-catalog.yaml` recomputed (sha256 of the new content) and
  verified programmatically against doctor's `TC-010`/`TC-011` formula.
- **Prose/docs consistency**: `README.md`, `docs/first-project-readiness.md`,
  `packages/mcp-tools/examples/{providers.example.yaml,README.md}`,
  `axiom.spec/templates/{discovery-provider-overview-template.md,
  increment-metadata-template.yaml}` updated. `docs/0027-toolchain-provider-
  expansion-and-repair.md` (an ARCHIVED, dated increment summary) deliberately
  LEFT UNCHANGED — it is historical record of what 0027 did at the time, not
  a living doc.
- **Tests**: guard-test lockstep (see above) + new tests: `cmm-freshness.test.ts`,
  `cmm-fallback-integration.test.ts` (e2e-style, real `createCmmProviderClient`
  + real `createFilesystemProviderClient` + real `invokeCapabilityLive`,
  proving the fallback), `route-tool.test.ts` additions (structural
  capabilities route to `cmm`; removed ids yield `env-unsupported`),
  `workspace-code-intel.test.ts` additions (`cmm` not-installed path;
  successful sync persists `.cmm/sync-state.json`).

## Non-goals

- Per-worktree provider isolation (INC-W4) — not implemented; `cmm`'s local
  index (`.cmm/`) is project-root-scoped exactly like the removed
  `.codegraph/`/`graphify-out/` were.
- RTK runtime wiring (INC-T2), skills concision policy (INC-T3), AutoSkills
  lock hygiene (INC-S1), the unified `<project>.axiom` MCP broker (Cluster A,
  done), worktrees (Cluster W) — untouched.
- Migrating Axiom's own repos (`Axiom.SDD`/`Axiom.Spec`) to any new model —
  out of scope (deferred to the user per the plan). The leftover
  `.codegraph/.gitignore` at the repo root (evidence of a past real
  `codegraph` local install against this very repo) was deliberately left
  untouched — removing it is unrelated repo housekeeping, not part of the
  product's provider-set change.
- Introducing new canonical capability ids (e.g. a distinct
  `code.blastRadius`/`code.dependencies`/`code.traces`) — "blast-radius/
  dependencies/traces" describe what `cmm`'s tool surface can internally
  analyze under the EXISTING `code.knowledgeGraph`/`code.structureAnalysis`
  capability ids, not new capability ids to mint.
- Fixing the pre-existing, unrelated gap that `code.impactAnalysis`/
  `code.symbolSearch`/`code.referenceSearch` are not mapped to ANY provider in
  `providers.yaml` — unrelated to the provider-set swap, not touched.

## Acceptance criteria

- [x] `codegraph` and `graphify` are no longer selectable, registered, or
      routable anywhere (`SELECTABLE_PROVIDER_IDS`, `CODE_INTEL_PROVIDER_IDS`,
      `CANONICAL_PROVIDER_IDS`, `providers.yaml`, the TUI wizard's provider
      step, `axiom configure --providers`, the toolchain catalog).
- [x] `cmm` is the sole structural provider, serving `code.knowledgeGraph` +
      `code.structureAnalysis`, with a real `ProviderClient`
      (`cmm-client.ts`) mirroring the serena-client pattern.
- [x] `serena` unchanged (`code.semanticNavigation`, fallback `filesystem`).
- [x] Fallback chain: `cmm → filesystem` (physical-path read/search/glob);
      proven end-to-end by a real (non-mocked) integration test.
- [x] Routing stays governed by the EXISTING `@axiom/tool-routing` machinery
      (`DispatchEvent`, `profile.preferredProviders`, `walkFallbackChain`) —
      zero source changes to `dispatcher.ts`/`select.ts`/`fallback.ts`; only
      data changed.
- [x] Freshness: a minimal auto-sync + staleness check exists for `cmm`
      (there was none before), never a mandatory git hook.
- [x] `packages/doctor`'s `CC-001..006`/`TR-001..004`/`PS-001` and
      `packages/toolchain`'s probe are updated in lockstep with the new set.
- [x] `axiom doctor` capability-model/skills/agents checks stay green with
      the new set (verified via the full test suite, incl.
      `capability-model.test.ts`, `skills.test.ts`, `agents.test.ts`,
      `toolchain-catalog-real.test.ts`).
- [x] An ADR documents the rationale and the constants reference it.
- [x] `npm run build` (tsc -b) passes.
- [x] Targeted `vitest run` (providers, capability-model, tool-routing,
      doctor, toolchain, install-profiles, installer, document-bootstrap,
      mcp-tools, apps/cli) passes, 0 new failures.
- [x] New unit tests: structural capability routes to `cmm`; `cmm→filesystem`
      fallback when `cmm` unavailable; `serena→filesystem` unchanged;
      `codegraph`/`graphify` no longer selectable; freshness check behaves
      (fresh/stale/unknown, auto-sync on stale, no-op when fresh).
- [x] An integration test proves a structural query resolves via `cmm` with
      the physical-path (`filesystem`) fallback, end-to-end.

## Open questions

None blocking. The `codebase-memory-mcp` binary name / exact MCP tool
surface is an ASSUMPTION (see Context) — resolved as "make it fully
overridable, document the assumption prominently, do not invent a real-looking
GitHub URL for it" rather than blocking on external verification this
environment cannot perform.

## Assumptions

Each is a deliberate, narrowest-change resolution of an ambiguity the task
left open — see the executor's final report for the one-line reasoning:

1. Clean removal of `codegraph-client.ts`/`graphify-client.ts` (not
   present-but-unwired) — the brief's preferred option, and the actual blast
   radius (contained to `packages/providers/src/code-intel/*` + data/test
   fixtures) made it tractable.
2. `cmm`'s assumed local index directory convention is `.cmm/` at the project
   root, mirroring the removed `.codegraph/`/`graphify-out/` convention.
3. `cmm`'s assumed launch shape: binary `codebase-memory-mcp`, subcommand
   `mcp` (stdio server; `--project <path>` when statically pinned, mirroring
   codegraph's `-p` convention) and `sync` (index build/refresh, mirroring
   `codegraph init`/`graphify extract .`). Both fully overridable.
4. `cmm`'s assumed MCP tool names: `explore` (`code.knowledgeGraph`, mirrors
   codegraph's single "ask almost anything" tool) and `query_structure`
   (`code.structureAnalysis`, mirrors graphify's BFS/DFS `query_graph`).
   Overridable per-capability (a first for a code-intel client in this
   codebase — justified by the lack of a verified README, unlike
   serena/codegraph/graphify).
5. Freshness model kept deliberately minimal per the task's "don't
   over-build" instruction: a single timestamp marker + max-age window, not
   a real content/mtime diff against the source tree. Auto-sync is
   best-effort and NEVER blocks or fails the actual capability call.
6. `ProviderKind` gets a new `'structural-code-intel'` value rather than
   reusing `'knowledge-graph'`/`'repository-graph'` — `cmm`'s scope
   (structural + graph + impact) is broader than either removed kind alone,
   and a distinct kind avoids implying `cmm` IS one of the removed tools
   under a new name. The two legacy kind strings are kept in the
   type/schema enums (harmless, backward-compat) though no current provider
   uses them.
7. `packages/installer/src/registry.ts`'s `EXTERNAL_DEPS_BY_CAPABILITY`
   (`code.knowledgeGraph → cmm`) is a narrower, pre-existing, slightly
   divergent mapping from `providers.yaml`'s real fallback chain (e.g. it
   already mapped `code.structureAnalysis → serena`, not the removed
   `graphify`) — left structurally as-is, just renamed the surviving
   `codegraph` reference to `cmm`; not "fixed" to match `providers.yaml`
   exactly, since that pre-existing divergence is unrelated to this
   increment's scope.
8. No fabricated GitHub URL for `codebase-memory-mcp`: `apps/cli/src/
   commands/member-install.ts`'s `EXTERNAL_INSTALL_GUIDANCE` map has NO
   `cmm` entry (falls through to the existing generic "no automated install
   command known" message) — the removed `codegraph`/`graphify` entries HAD
   real, verifiable URLs; inventing one for `cmm` would be a false claim.
9. `packages/toolchain/src/commands` free-form toolchain ids
   (`apps/cli/src/commands/toolchain.ts`'s `deriveKindForId`) keep
   `codegraph`/`graphify` prefix-matching ALONGSIDE the new `cmm` one, for
   backward-compat classification of a pre-existing project's
   `toolchain.yaml` entry — this is a decoupled, free-form string namespace,
   not the closed provider set, so nothing required removing them there.
10. Toolchain-package tests using `codegraph-this-project`/
    `graphify-this-project-local-only` as arbitrary example tool ids
    (`packages/toolchain/tests/{p1-tools,repair-add-gitignore}.test.ts`,
    `apps/cli/tests/toolchain.test.ts`, `packages/doctor/tests/
    toolchain.test.ts`) were LEFT UNCHANGED — self-contained fixtures
    exercising generic toolchain-manifest CRUD/repair logic, unrelated to
    `CANONICAL_PROVIDER_IDS`.
11. `packages/providers/tests/registry.test.ts`'s generic `ProviderRegistry`
    test using `'codegraph'` as an arbitrary placeholder id was LEFT
    UNCHANGED — same reasoning (tests a generic container, not the real
    provider set).
12. `docs/0027-toolchain-provider-expansion-and-repair.md` (archived
    increment summary, dated 2026-06-30) was NOT rewritten to retroactively
    describe `cmm` — it accurately records what increment 0027 did AT THE
    TIME (CodeGraph/Graphify); rewriting history would be inaccurate.
13. ADR numbered `0031` — the next available number after the highest
    existing `docs/NNNN-*.md` (0030), following this repo's established
    convention/location exactly as instructed.

## Implementation notes

All files changed live in `Axiom/` (the product monorepo) unless noted.
Full file-level detail is in the Scope section above and the executor's
final report. Key new files:
`packages/providers/src/code-intel/{cmm-client,cmm-freshness}.ts`,
`packages/providers/tests/{cmm-freshness,cmm-fallback-integration}.test.ts`,
`docs/0031-adr-cmm-replaces-graphify-and-codegraph.md`.

## Validation

Ran from `C:\repos\Axiom Workspace\Axiom`:

- `npm run build` (`tsc -b`) — PASSED, clean, exit 0, no output.
- Targeted `npx vitest run packages/providers packages/capability-model
  packages/tool-routing packages/toolchain` — **17 files, 163 tests passed**.
- Targeted `npx vitest run packages/doctor` — **18 files, 182 tests passed**
  (includes `capability-model.test.ts`, `skills.test.ts`, `agents.test.ts`,
  `toolchain-catalog-real.test.ts`, `provider-selection.test.ts`,
  `tool-routing.test.ts`, `gateway.test.ts`).
- Targeted `npx vitest run packages/install-profiles packages/installer
  packages/document-bootstrap packages/mcp-tools` — **23 files, 210 tests
  passed** (includes `capability-routing-roundtrip.test.ts`, the real
  `loadCapabilityModel`/`routeTool` round-trip against the example YAMLs).
- Targeted `npx vitest run apps/cli/tests` — **116 files, 1074 tests
  passed** (includes the previously-flagged-as-sometimes-flaky
  `context.test.ts` and `workspace-setup.test.ts`, both green here).
- Full-suite `npx vitest run` (whole repo) — **297 files, 2990 tests
  passed. Zero failures.**
- bundleHash lockstep verified programmatically: a script recomputed
  sha256 for every `skills-catalog.yaml`/`agents-catalog.yaml` entry's
  `source` file and confirmed it matches the stored `bundleHash` for
  ALL entries (not just the 8 touched by this increment) — "ALL bundleHash
  entries verified OK (skills + agents)".

No pre-existing failures were observed to classify — the suite was fully
green before and after in every scope run.

## Result

Implemented. `cmm` (`codebase-memory-mcp`) is now the sole structural
code-intelligence provider in the closed 6-id `CANONICAL_PROVIDER_IDS` set,
cleanly replacing the removed `codegraph`/`graphify`. `serena` is unchanged.
Fallback (`cmm → filesystem`) is proven end-to-end. A minimal freshness/
auto-sync model was added for `cmm` (none existed before), on-demand only,
never a git hook. Doctor and toolchain checks were updated in lockstep, and
every downstream guard test (routing, capability-model, doctor, install
profiles, installer, document-bootstrap, toolchain, apps/cli commands) was
updated to the new set. The skills/agents catalog bundleHash gotcha was
honored: 8 source `.md` files were edited and their hashes recomputed and
verified programmatically. Full build + full test suite are green
(297/297 files, 2990/2990 tests).

## General spec integration

Integrado en la spec canónica al cierre del batch INC-20260724-* (integración
única de toda la tanda). Ficheros que recibieron el conocimiento de este
incremento:

- `06_Integraciones_y_Capacidades.md` — `cmm` único proveedor estructural
  (ADR-0031), `serena` simbólico, fallbacks, freshness/auto-sync sin hooks;
  supersede la sección de code-intel providers (codegraph/graphify) y
  `SELECTABLE_PROVIDER_IDS`.
- `07_Gobierno_y_Seguridad.md` — sin hooks git obligatorios (auto-sync
  on-demand).
- `08_Glosario.md` — `cmm` (codebase-memory-mcp) / ADR-0031.
- `00_Resumen_Ejecutivo.md` — eje "stack externo reordenado".
- `01_Requisitos_Funcionales.md` — RF-AXM-044.
