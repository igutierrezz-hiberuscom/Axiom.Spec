# Increment: Axiom redesign — reconciled increment roadmap

Status: closed (roadmap complete — see "Roadmap closure summary" below)
Date: 2026-07-02

## Goal

Decompose the Axiom architectural redesign (repo separation, local registry,
`axiom.yml` manifests, derived vs curated indexes, structural commands,
semantic MCP, contextual TUI, documentary bootstrap, versioning/migrations)
into an ordered sequence of increments — **reconciled against the existing
`Axiom/packages/*` implementation**, not planned as greenfield. Each
increment states the concrete package(s) it migrates/refactors, the diff
between current and target contract, and whether a compatibility path or
breaking cutover is required.

This increment is planning-only. It does not implement code.

## Correction from the initial draft of this increment

An earlier version of this roadmap (produced before inspecting
`Axiom/packages/`) treated the redesign as greenfield and proposed building
registry, manifests, structural commands, MCP, TUI, etc. from scratch. That
was wrong: `Axiom` already contains a mature, operational monorepo (~26
packages, 29+ CLI subcommands, real tests, closed increments up to at least
`0036` per its own `README.md`, last commit 2026-07-02). This revision
replaces that draft with a reconciliation-based plan per explicit user
decision.

## Context

Two decision documents from an external architecture session define the
target model:

- `axiom_decisiones_sesion_prompt_implementacion.md` — repo separation
  (sdd/spec/code), global local registry (`~/.axiom/projects.yml`),
  per-repo `axiom.yml` manifests, derived vs curated indexes, mandatory
  structural CLI commands, single project-scoped MCP with
  `get_implementation_context`, contextual TUI, documentary bootstrap,
  versioning/migrations. 7-phase priority order (source doc section 20).
- `axiom_decisiones_sesion_addendum_revision.md` — 16 reinforcements:
  dogfooding boundary, `AGENTS.md` as canonical contract, filesystem-mode
  vs MCP-mode coexistence, cache rebuild/fallback, ADR index
  derived-or-curated, generated human views are not sources of truth,
  per-repo commit separation, write-scope validation, external
  integrations as optional plugins, internal IDs + `externalRefs`,
  optional command router, project-scoped MCP config per adapter, TUI
  detection from subfolders, bootstrap output in draft/review state,
  optional Workbench, per-project security isolation.

### What already exists in `Axiom/packages/*` (verified by direct inspection)

The existing implementation is built around a **single-repo-first** model,
not the new docs' **separated-repos-first** model:

- **Topology** (`@axiom/topology`, spec 0021): `TopologyManifest` with
  `mode: 'single-repo' | 'multi-repo'`, `sddRepo`/`specRepo`/
  `roleCodeRepositories` + `assignments`, stored at
  `axiom.spec/config/topology.yaml` (versioned) plus per-user
  `.sdd/local/topology-bindings.yaml` (gitignored path overrides). Default
  when absent: `defaultSingleRepoManifest` — sdd and spec both resolve to
  `projectRoot` (`ref: '.'`). Multi-repo exists only as an "installed"
  fallback (`defaultInstalledMultiRepoManifest`, spec repo assumed at
  `../${projectName}.spec`), not as the default working mode.
- **User-level registry** (`@axiom/user-workspace/registry.ts`, spec 0020):
  `~/.axiom/registry.json` (JSON, not YAML), schema-versioned
  (`schemaVersion: 1`), atomic tmp+rename writes, `addProject` /
  `removeProject` / `getProject` / `listProjects` / `useProject` /
  `findByRootPath`, lazy `stale` flag computed from `fs.existsSync`. This
  already **is** a working equivalent of the new docs' `projects.yml`,
  minus two things: (a) JSON vs YAML, (b) one `rootPath` per project entry,
  not a `repos: { sdd, spec, code }` map — multi-repo-per-project is not
  represented at the registry level today, only at the topology level
  inside a single project root.
- **Project/manifest resolution** (`@axiom/project-resolution/resolver.ts`,
  spec 0013): reads `axiom.yml` (singular `.yml`, matching the new docs)
  from the current root via `discoverAxiomRoot`, resolves `project.name`,
  `project.mode` (`local-only | gateway | hybrid`), and a `scopes` map
  (`{ path, product_runtime }`) — this is the closest existing analog to
  the new docs' per-repo `axiom.yml` + `role` field, but scopes are
  sub-paths inside one repo, not references to independent repos at
  arbitrary filesystem locations.
- **Isolation** (`@axiom/isolation/p0.ts`, spec 0014 P0): project-scoped
  paths under `<rootPath>/.sdd/{memory,mcp,config,outputs}/<projectName>/`,
  an MCP allowlist (`DEFAULT_ALLOWED_MCP_SERVERS = ['sdd', 'spec',
  'serena']`), and `pathsAreIsolated` cross-project collision checks. This
  already implements most of addendum section 17, but scoped to
  sub-directories of one repo root, not to independently-located repos
  tied together by a global registry.
- **Doctor** (`@axiom/doctor/checks.ts`, spec 0013/0014/0004/0003/0008/
  0021/0022): 8+ check categories (boundaries, policies, manifests,
  isolation, capability-model, install-profiles, gateway, tool-routing,
  topology, QA-lane coherence), each producing `pass | fail | warn | skip`
  with an evidence string. This is a working, more granular superset of
  the new docs' `axiom doctor` concept — already exists, does not need to
  be built, only extended with new checks as new contracts land.
- **Versioning** (`@axiom/versioning`): `managed-state.ts`, `checkpoints.ts`,
  `migrations.ts`, `upgrade.ts` — `axiom upgrade` already supports
  `--dry-run`, `--from-checkpoint`, `--target-version`, rollback-first
  design, and shipped a real MVP migration (`0.0.0 -> 0.1.0`). This is
  materially ahead of the new docs' section 16 description, which reads as
  if migrations do not exist yet.
- **CLI commands** (`apps/cli/src/commands/*`): `init`, `join`, `configure`,
  `sync`, `start`, `audit`, `upgrade`, `tui`, `doctor`, plus
  `axiom-increment.ts`, `axiom-bug.ts`, `axiom-plan.ts`, `axiom-role.ts`,
  `axiom-qa-e2e.ts`, `context.ts`, `topology.ts`, `repo.ts`, `roles.ts`,
  `skills.ts`, `mcp.ts`, `gateway.ts`, `model.ts`, `capability.ts`,
  `components.ts`, `memory.ts`, `projects.ts`, `app*.ts` (API/plugins
  including an Azure DevOps plugin already present),
  `qa-archive-gate.ts`, `self-update.ts`. Structural commands for
  increments/bugs/plans/roles/context/topology/skills/MCP **already exist**
  as CLI surface; the question per increment below is whether their
  current contract matches the new docs' exact shape, not whether they
  exist at all.
- **Skills** (`@axiom/skills`, spec 0032): `axiom.spec/config/
  skills-catalog.yaml` catalog with `id/name/version/source/status/
  securityCheckStatus/bundleHash`, `apply.ts`, `materialize.ts`,
  `refresh.ts` (drift detection). This already covers much of source doc
  section 9's skill index/catalog concept, colocated under `axiom.spec/`
  inside the single repo.
- **TUI** (`@axiom/tui`, spec 0019 B1-B3): `driver.ts`, `router.ts`, a real
  `menu` screen plus `configure/sync/doctor/upgrade` flows, previews and
  post-run summaries, `mcp-inventory`/`memory-inventory`/`topology`/
  `model-list` screens. `cli-commands` package exists specifically as a
  shared seam so the TUI never imports `apps/cli/src/...` directly
  (layering rule already enforced). This already substantially implements
  source doc section 12 and the "TUI has no business logic of its own"
  rule (source doc 1.5, 12.4) — the gap is menu content/labels and
  detection heuristics, not the shell or the layering discipline.
- **Adapters** (`packages/adapters/*`, spec 0031): 6 real adapters
  (opencode, claude-code, github-copilot, vscode, cursor, litellm), each
  declaring `GENERATED_FILES_BY_TARGET` and consumed by
  `@axiom/installer`. This already implements source doc section 2.3's
  adapter folders and addendum section 3's "adapters derived from a
  contract" idea, though the canonical source today is `axiom.yml` +
  `install-profile.json`, not literally `AGENTS.md`.
- **Document bootstrap** (`@axiom/document-bootstrap`, spec 0012): renders
  `.github/copilot-instructions.md` (and by extension other adapter docs)
  from `ResolvedVariables`, with path-guard, team-block preservation, and
  atomic writes. This is a narrower, single-purpose ancestor of source doc
  section 15's "bootstrap documental" — it materializes *instructions*
  from already-known variables; it does not *analyze* a codebase or a
  legacy SDD repo to draft new technical-context documents from scratch.
- **Capability/tool-routing/telemetry/orchestrator/toolchain/memory**:
  further packages (`capability-model`, `tool-routing`, `telemetry`,
  `orchestrator`, `toolchain`, `memory`, `model-routing`,
  `install-profiles`, `config-validation`, `persistence`,
  `filesystem-truth`, `cavekit-discipline`, `components`) implement a
  capability/provider/profile model (`local-only | standard | enterprise`
  overlays, MCP provider allowlists, gateway-first vs filesystem-first
  discovery) that has **no equivalent at all** in the two new decision
  documents. The new docs do not mention capability profiles, telemetry,
  or a gateway concept. This is a real gap in the *other* direction: the
  existing product is more advanced in these areas than what the new docs
  ask for, and the roadmap must not regress or silently drop this work
  while reconciling the topology/registry/manifest layer.

### The fundamental architectural tension

The new decision documents mandate: **always separate repos** (sdd/spec/
code), resolved through a **global local registry** with no default
"single-repo" mode, and technical context/increments/bugs/plans always
physically outside the code repo. The existing implementation defaults to
**single-repo mode** (`sdd-repo` and `spec-repo` both resolve to
`projectRoot`), with multi-repo as an opt-in fallback, and colocates
`axiom.spec/` (config, plans, increments references) inside the same repo
as the runtime by default. This is not a naming gap — it is a genuine
architectural decision conflict that must be resolved explicitly (Open
Question Q1 below) before any migration increment can be scoped precisely,
because it determines whether `single-repo` mode is deprecated, kept as an
opt-in convenience (per source doc 1.1: "comodidad local, pero no
requisito"), or redefined.

## Scope

- Reconcile every phase of the original 7-phase sequence (base estructural
  -> instalación de proyecto -> artefactos estructurales -> skills/
  contexto/adapters -> MCP -> bootstrap documental -> versionado/upgrades)
  against the concrete existing package(s) that already cover that
  concern.
- For each increment: existing package(s) audited, concrete diff, whether
  the change is additive/compatible or breaking, migration/shim strategy
  if breaking, and the subagent sequence (migration-engineer first in most
  cases) that would execute it.
- Preserve the capability/provider/telemetry/orchestrator work that has no
  equivalent in the new docs — the roadmap must not plan its removal.

## Non-goals

- No code implementation in this increment.
- No decision here on whether `single-repo` mode is deprecated — that is
  Open Question Q1, explicitly blocking for INC-01.
- No re-litigation of the capability/provider/profile model; it is treated
  as out-of-scope-to-change unless a specific increment below says
  otherwise.
- No Workbench implementation (addendum section 16) — remains deferred.
- No speculative rewrite of packages that already satisfy the new docs'
  intent structurally, even if naming differs (e.g. `registry.json` vs
  `projects.yml` is a format/rename decision inside INC-01, not a rebuild).

## Acceptance criteria

- [x] Every increment in the sequence names the existing package(s) it
      audits/migrates, not a "build from scratch" scope.
- [x] Every increment states a concrete diff sourced from actually reading
      the package's code (not inferred from the package name alone).
- [x] Every increment states whether the change is breaking and, if so,
      what migration/shim path is proposed, respecting
      `Axiom.SDD/AGENTS.md`'s explicit bootstrap limits (no speculative
      architecture, no heavy constructs beyond what is explicitly asked).
- [x] The primary subagent for each increment is a migration-engineer that
      audits before any contract is (re)written, except where an
      increment is genuinely additive with no existing package to
      reconcile against.
- [x] The roadmap explicitly preserves the existing capability/provider/
      telemetry/orchestrator investment rather than planning its
      replacement.
- [x] Recorded in `Axiom.Spec` per `Axiom.SDD/AGENTS.md`; no code changed
      in `Axiom.SDD` or `Axiom`.

## Open questions

- **Q1 (blocking for INC-01)**: Does the redesign deprecate `single-repo`
  mode as the default, or keep it as an explicit opt-in "comodidad local"
  per source doc 1.1, with `multi-repo` + global registry becoming the
  primary supported path? This determines whether `defaultSingleRepoManifest`
  in `@axiom/topology` is kept, deprecated with a migration warning, or
  removed. Must be answered before INC-01 write work starts.
- **Q2 (blocking for INC-01)**: Is `~/.axiom/registry.json` (JSON, one
  `rootPath` per project) renamed/reshaped to `~/.axiom/projects.yml`
  (YAML, `repos: { role: path }` map per project), or does the new
  per-repo model get layered on top of the existing registry without
  renaming the file (e.g. `registry.json` gains a `repos` array per
  project entry)? Changing the file format/location is a breaking change
  for every existing installed project.
- **Q3**: Should `axiom.spec/` remain the conventional in-repo path used
  pervasively across existing specs/docs/code (`axiom.spec/config/*.yaml`,
  `axiom.spec/plans/*.md`) even after multi-repo separation becomes
  primary, or does the spec repo drop the `axiom.spec/` prefix once it is
  its own repo root? Affects every package that hardcodes `axiom.spec/...`
  path segments (topology, doctor, document-bootstrap, skills, and more).
- **Q4**: Do the capability/provider/profile/telemetry/gateway packages
  (which have no equivalent in the two source documents) get: (a) left
  untouched and orthogonal to this redesign, (b) explicitly folded into
  the new `mcp.yml`/tools-per-project model (source doc section 14), or
  (c) something else? Not blocking the roadmap sequence, but blocking
  before INC-13/INC-15 (MCP phase) can be scoped precisely, since the
  existing `@axiom/tool-routing` + `@axiom/capability-model` already do
  significant MCP-adjacent work.
- **Q5**: Target format for `axiom.yml` — the existing schema uses
  `project: { name, mode }` + `scopes: { name: { path, product_runtime } }`.
  The new docs use `schemaVersion / projectId / repoId / role / indexes /
  paths`. Is this a versioned schema migration (`schemaVersion: 1 -> 2`
  with a migration script) or an additive superset? Blocking for INC-01.

## Assumptions

- "Subagent types" remain logical task-type roles, not new infrastructure,
  consistent with `AGENTS.md`'s prohibition on heavy multi-agent
  orchestration frameworks unless explicitly requested (it is, for
  planning purposes only).
- The existing test suites (`vitest`) and `axiom doctor` remain the
  validation backbone; migration increments are expected to extend them,
  not replace them.
- Each increment below will get its own spec file under
  `Axiom.Spec/specs/increments/` when work actually starts on it, following
  this roadmap's sequence and diff analysis as the starting brief.
- Because `Axiom.Spec` and `Axiom.SDD` (the repos this workspace treats as
  canonical for spec/method) currently contain no record of the `Axiom`
  product's own increments 0001-0036+, this roadmap treats `Axiom`'s
  in-repo `axiom.spec/` history as the authoritative implementation record
  for "what already exists," while `Axiom.Spec` (capital, sibling repo)
  remains the canonical home for *this* cross-repo redesign's own spec
  going forward, per `Axiom.SDD/AGENTS.md`. This dual-spec-location
  reality is itself a candidate finding for Q3.

## Implementation notes

Planning-only. No files were changed under `Axiom/packages/`, `Axiom/apps/`,
or `Axiom.SDD/` for this increment.

## Reconciled increment roadmap

Legend (unchanged from prior draft, task-type roles not tool/product
names): **schema-writer**, **cli-implementer**, **registry-engineer**,
**index-engineer**, **tui-developer**, **mcp-tool-implementer**,
**adapter-engineer**, **docs-skills-writer**, **migration-engineer**,
**bootstrap-analyzer**, **validator-reviewer**.

New convention for this reconciled version: the primary subagent listed
first for each increment is a **migration-engineer** whose first task is
always "audit the existing package(s) named in Diff, confirm the diff
still holds, and produce a compatibility/cutover note" before any other
subagent writes new contract code. Increments with no existing package to
reconcile against (genuinely additive) are marked **(additive, no legacy
package)** and skip the audit step.

### Phase A — Base estructural (Fase 1)

**INC-01 — Reconcile registry + manifest schema (`user-workspace` +
`project-resolution` + `topology` -> new contract)**
Existing packages: `@axiom/user-workspace` (`registry.ts`,
`registry-types.ts`), `@axiom/project-resolution` (`resolver.ts`),
`@axiom/topology` (`types.ts`, `loader.ts`).
Diff: existing registry is JSON at `~/.axiom/registry.json`, one
`rootPath` per project, no repo-role map. New docs want YAML at
`~/.axiom/projects.yml`, `repos: { roleKey: { role, path } }` per project.
Existing `axiom.yml` uses `project.mode` (`local-only|gateway|hybrid`) +
`scopes` (sub-paths); new docs want `schemaVersion/projectId/repoId/role/
indexes/paths` per independent repo. Existing topology defaults to
`single-repo` (sdd/spec both = projectRoot); new docs assume repos are
never co-located by default.
Breaking/compatible: breaking at the file-format and schema level (Q2,
Q5); the `mode` enum itself (`single-repo|multi-repo` vs `local-only|
gateway|hybrid`) encodes different axes (repo topology vs execution
topology) and both may need to coexist rather than merge. Needs a
migration script (`registry.json` -> `projects.yml` or additive
extension, per Q2 resolution) and an `axiom.yml` schema version bump with
a `config-validation` update. Must not silently break the 26 packages
that already read `axiom.yml`/`registry.json` shapes.
Dependencies: none (first increment), but blocked on answering Q1, Q2, Q5
before write work.
Subagents (in order): migration-engineer (audit `user-workspace`,
`project-resolution`, `topology`; produce compatibility note answering
whether Q1/Q2/Q5 allow an additive path) -> schema-writer (new/extended
`axiom.yml` + registry schema, versioned) -> registry-engineer (extend
`@axiom/user-workspace` + `@axiom/topology` rather than replace) ->
cli-implementer (extend existing `repo.ts`/`topology.ts`/`projects.ts`
commands) -> validator-reviewer (extend `@axiom/doctor` topology checks
TC-001/TC-002 and manifest checks MC-001/MC-002 for the new schema
version; run existing vitest suites to confirm no regression).

**INC-02 — Structural command foundation and event model (audit
`orchestrator` first)**
Existing package: `@axiom/orchestrator` (`state-machine.ts`, `gates.ts`,
`runner.ts`) already implements a 7-lifecycle-state machine and 15 intent
commands (currently `not-implemented` in MVP per README). This is
materially more than "an event model" — it is a governed state machine.
Diff: source doc 6.2 asks for simple internal events
(`RepoRegistered`, etc.). The existing orchestrator's `gateFor` + intent
commands are a superset; the gap is that several intent commands are
stubs (`not-implemented`).
Breaking/compatible: additive — implement the currently-stubbed intent
commands relevant to the new registry/manifest operations from INC-01
rather than inventing a parallel event system.
Dependencies: INC-01.
Subagents: migration-engineer (audit `orchestrator/state-machine.ts`,
confirm which of the 15 intent commands map to `RepoRegistered`-equivalent
events) -> registry-engineer (wire INC-01 registry/topology writes through
`runCommand`/gates instead of direct writes) -> validator-reviewer.

### Phase B — Instalación de proyecto (Fase 2)

**INC-03 — Reconcile project installation wizard (`installer` +
`install-profiles` + `init`/`join`/`configure` commands)**
Existing packages: `@axiom/installer` (`installer.ts`, `registry.ts`
generated-files map, `profiles-loader.ts`), `@axiom/install-profiles`
(10-step composer), CLI commands `init.ts`, `join.ts`, `configure.ts`.
Diff: existing installer asks for `functionalProfile` (`product-owner|
builder`), `overlay` (`local-only|standard|enterprise`), `adapterTarget` —
a capability/profile-driven install, not the new docs' identity/repos/
roles/adapters/tools/bootstrap-level questionnaire (source doc 13.2).
Existing `init` scaffolds a single-repo `axiom.yml` + `.sdd/`; it does not
ask "where is your SDD repo / spec repo" because single-repo is the
default.
Breaking/compatible: the questionnaire shape changes (new questions for
repo roles/paths), but the underlying `installProfile()` machinery
(generated-files map, atomic persistence) is reusable as-is. Not breaking
if the new questions are added as an additional step before the existing
profile composition, gated behind the Q1 decision (do we ask "single or
multi repo" explicitly now).
Dependencies: INC-01, INC-02.
Subagents: migration-engineer (audit `installer.ts` + `init.ts`/`join.ts`
end-to-end flow) -> docs-skills-writer (AGENTS.md generation — check
whether `init` already writes one; if not, add per addendum section 3) ->
registry-engineer (extend installer to call INC-01's repo-registration
path) -> cli-implementer (extend `init`/`configure` command flags) ->
validator-reviewer.

**INC-04 — Reconcile contextual TUI shell and detection (`tui` package)**
Existing package: `@axiom/tui` — `driver.ts`, `router.ts`, `screens/menu.ts`
already render a real menu (`configure/sync/doctor/upgrade` + more
screens: `mcp-inventory`, `memory-inventory`, `topology`, `model-list`,
`projects`). Detection today is driven by `project-resolution` (axiom.yml
in cwd via `discoverAxiomRoot`), not yet the addendum's fuller heuristic
(parent folders, registered-path membership, git root — addendum section
14).
Diff: menu items/labels differ from source doc 12.2/12.3's proposed text,
but the shell, layering (`@axiom/cli-commands` seam so TUI never imports
`apps/cli/src` directly), and "no business logic in the TUI" rule (source
doc 1.5, 12.4) are already correctly implemented.
Breaking/compatible: additive — extend `discoverAxiomRoot`/
`project-resolution` with the addendum-14 heuristic and extend the
existing menu with items for new INC-01 commands; no shell rewrite needed.
Dependencies: INC-01, INC-02, INC-03.
Subagents: migration-engineer (audit `tui/router.ts` + `project-resolution`
detection path) -> tui-developer (extend detection heuristic + menu
items) -> validator-reviewer.

**INC-05 — AGENTS.md as canonical contract + adapter reconciliation
(`document-bootstrap` + `packages/adapters/*`)**
Existing packages: `@axiom/document-bootstrap` (renders
`.github/copilot-instructions.md` from `ResolvedVariables`), 6 real
adapters under `packages/adapters/*` with `GENERATED_FILES_BY_TARGET`.
Diff: today the canonical source for generated files is `axiom.yml` +
`install-profile.json` + per-adapter generator, not `AGENTS.md` as a single
upstream contract (addendum section 3). `document-bootstrap` is scoped to
one target (copilot instructions) with team-block preservation; it is not
yet a general "generate all adapters from one canonical file" pipeline.
Breaking/compatible: potentially breaking if `AGENTS.md` becomes the
single upstream source, since today each adapter generator likely reads
`axiom.yml`/profile data directly. A shim (adapters keep reading their
current inputs, plus a new `AGENTS.md` render step that stays consistent
with them) avoids a hard cutover.
Dependencies: INC-03 (whatever `init` decides to generate).
Subagents: migration-engineer (audit all 6 adapter generators +
`document-bootstrap/renderer.ts` for their current input contract) ->
docs-skills-writer (canonical `AGENTS.md` template) -> adapter-engineer
(wire adapters to also read/derive from `AGENTS.md` without breaking
existing `ResolvedVariables` inputs) -> validator-reviewer (existing
`TC-009` doctor check already enforces adapter package shape — extend it,
do not replace it).

### Phase C — Artefactos estructurales (Fase 3)

**INC-06 — Reconcile increment/bug/plan commands
(`axiom-increment.ts`/`axiom-bug.ts`/`axiom-plan.ts`)**
Existing: CLI commands already exist for increments, bugs, plans, and
roles. Not yet inspected line-by-line in this pass (out of the
minimum-required set the user listed); flagged as needing the same
audit-first treatment before assuming source doc 6.1's exact command
names/flags match.
Diff: unknown until audited — likely partial overlap (commands exist) but
possibly different metadata shape than `metadata.yml` (source doc 5.3)
and no confirmed `externalRefs` support yet (addendum section 11).
Breaking/compatible: unknown until audited; treat as likely additive
(extend existing commands with `externalRefs` field and confirm
`metadata.yml`-equivalent already matches or needs a schema bump).
Dependencies: INC-01, INC-02.
Subagents: migration-engineer (audit `axiom-increment.ts`, `axiom-bug.ts`,
`axiom-plan.ts`, and whatever metadata format they currently write) ->
schema-writer (only if metadata shape needs a version bump) ->
cli-implementer (add `externalRefs`, align flags) -> validator-reviewer.

**INC-07 — Reconcile derived caches / index rebuild
(`@axiom/doctor` + any existing index builder)**
Existing: no dedicated `@axiom/index` package was found among the 26
packages; caching/rebuild behavior may currently live inside `doctor` or
individual commands (unconfirmed — needs audit). `@axiom/doctor` already
implements the fallback pattern in spirit (`skip` when a file is
absent/unparseable rather than crashing), which is the same posture
addendum section 5 asks for.
Diff: unclear whether a generic `axiom index rebuild` exists today; if
not, this is the one sub-item of Phase C that may be genuinely additive.
Breaking/compatible: additive if no equivalent exists.
Dependencies: INC-06.
Subagents: migration-engineer (audit whether any package already does
cache rebuild; report back before assuming additive) -> index-engineer
(build only the missing piece, reusing `doctor`'s read/skip/fail pattern)
-> validator-reviewer.

**INC-08 — Reconcile `axiom doctor` extension + write-scope validation**
Existing package: `@axiom/doctor` already has 9+ check categories and a
`DoctorCheck { id, category, description, status, evidence }` shape used
consistently across all categories. This is the working, more mature
version of source doc's `axiom doctor` — it does not need to be built.
Diff: `allowedWriteScope` (addendum section 9) and `axiom validate
changes` do not appear among the audited checks (`boundaries, policies,
manifests, isolation, capability-model, install-profiles, gateway,
tool-routing, topology`); this is a genuinely new check category to add
in the existing pattern, not a new tool.
Breaking/compatible: additive — new check category following the
existing `pass/fail/warn/skip` + evidence convention.
Dependencies: INC-06, INC-07.
Subagents: migration-engineer (confirm no `allowedWriteScope`-equivalent
exists in `doctor/checks.ts` today) -> schema-writer (allowedWriteScope on
plan metadata) -> cli-implementer/validator-reviewer (new `WS-00x` check
category added to `@axiom/doctor` following existing conventions).

### Phase D — Skills, contexto y adapters (Fase 4)

**INC-09 — Reconcile SDD/role skills (`@axiom/skills` catalog)**
Existing package: `@axiom/skills` already has a catalog
(`axiom.spec/config/skills-catalog.yaml`, `schemaVersion: 1`, entries with
`id/name/version/source/status/securityCheckStatus/bundleHash`), plus
`apply.ts`/`materialize.ts`/`refresh.ts` (drift detection).
Diff: existing catalog is flat (one list), not split into SDD-repo skills
vs per-repo technical skills vs curated per-role index with
`mandatory.always`/`mandatory.whenTags`/`available` (source doc 5.5,
9.4, 10.2). The `status`/`securityCheckStatus` fields are review-oriented,
which is a useful addition the new docs do not mention — must be
preserved.
Breaking/compatible: additive if role-based mandatory/available grouping
is layered on top of the existing catalog schema (bump `schemaVersion` or
add an optional new section) rather than replacing it.
Dependencies: INC-06 (commands referenced by skills must be stable).
Subagents: migration-engineer (audit `skills/catalog.ts` +
`materialize.ts` for extension points) -> schema-writer (role-index
schema, versioned, layered on existing catalog) -> docs-skills-writer ->
validator-reviewer.

**INC-10 — Reconcile technical context structure**
Existing: no dedicated `technical-context/` package was identified among
the 26 audited; likely represented today via `axiom.spec/` docs and the
`skills`/`document-bootstrap` packages rather than a first-class indexed
structure. Needs explicit audit before scoping.
Diff: unknown until audited; source doc 10 wants curated, versioned,
per-role indexes with mandatory-document resolution — if nothing
equivalent exists, this is additive.
Breaking/compatible: likely additive.
Dependencies: INC-06, INC-08.
Subagents: migration-engineer (audit `axiom.spec/` conventions used by
existing packages, e.g. `doctor`'s `specScope` lookup logic, to see if a
context index already exists in practice) -> schema-writer ->
docs-skills-writer -> cli-implementer -> validator-reviewer.

**INC-11 — Reconcile ADR/decision handling**
Existing: not found among audited packages/commands (no `adr.ts` or
`decision.ts` CLI command surfaced in the `apps/cli/src/commands` listing).
Diff: likely additive — the new docs' ADR derived-or-curated policy
(addendum section 6) has no existing equivalent to reconcile against.
Breaking/compatible: additive, marked **(additive, no legacy package)**.
Dependencies: INC-07 (derived cache mechanism), INC-09/INC-10 (curated
index mechanism, once confirmed).
Subagents: schema-writer -> cli-implementer -> validator-reviewer.

**INC-12 — Human-readable generated views (optional)**
Existing: `@axiom/skills`'s `status`/`securityCheckStatus` reporting and
`doctor`'s evidence strings are the closest existing analogs to "views,"
but no dedicated `REGISTRO_INCREMENTOS.md`-style generator was found.
Diff: additive.
Breaking/compatible: additive, low priority — explicitly optional per
addendum section 7.
Dependencies: INC-06, INC-07.
Subagents: index-engineer -> docs-skills-writer -> validator-reviewer.

### Phase E — MCP (Fase 5)

**INC-13 — Reconcile single project-scoped MCP (`tool-routing` +
`capability-model` + `isolation`)**
Existing packages: `@axiom/tool-routing` (dispatcher + fallback,
`routeTool`), `@axiom/capability-model` (16 capability IDs, 4 domains, MVP
providers), `@axiom/isolation` (MCP allowlist: `sdd`, `spec`, `serena`
only — `backend-mcp`/`frontend-mcp` explicitly forbidden per ADR-0021).
This is a substantially built-out MCP-adjacent layer already, more
sophisticated in some respects (capability-to-provider fallback chains,
gateway-vs-filesystem-first discovery order) than the new docs describe.
Diff: the new docs' specific tool names (`get_project_registry`,
`get_skill_index`, `get_technical_context_document`, etc. — source doc
7.3) do not have a confirmed 1:1 mapping to existing capability IDs; needs
an audit to map new-doc tool names onto existing `capabilityId` values
or add new ones. The existing system's "MCP" is actually a
capability/provider abstraction (`sdd`/`spec`/`serena` MCP servers behind
it), which is architecturally adjacent to but not identical to "one MCP
server per project with N tools" (source doc 7.2).
Breaking/compatible: not breaking if the new docs' named tools are added
as new `capabilityId`s dispatched through the existing `routeTool`
mechanism, keeping `sdd`/`spec`/`serena` as the allowed providers. This
requires Q4 to be resolved first (does the new MCP model replace or sit
on top of capability-model).
Dependencies: INC-01 through INC-11, plus Q4 resolved.
Subagents: migration-engineer (audit `tool-routing/index.ts` dispatcher
contract + `capability-model` provider registry in depth — not yet done
in this pass) -> mcp-tool-implementer (map new tool names to
capabilities) -> validator-reviewer (existing `TR-001..004` doctor checks
already smoke-test `routeTool`; extend, do not replace).

**INC-14 — `get_implementation_context` (flagship MCP tool)**
Existing: no direct equivalent found; closest adjacent pieces are
`orchestrator`'s intent-command dispatch and `tool-routing`'s
capability resolution, which already know how to resolve
profile/capability/provider triples.
Diff: additive at the tool level, but must be implemented as a new
capability routed through the existing `tool-routing` dispatcher rather
than a standalone new server, to stay consistent with INC-13's
resolution.
Breaking/compatible: additive.
Dependencies: INC-13, INC-08 (write-scope), INC-09/INC-10 (context/skill
indexes, once confirmed to exist or be built).
Subagents: migration-engineer (confirm no existing capability already
covers most of this) -> mcp-tool-implementer -> validator-reviewer.

**INC-15 — Project-scoped MCP isolation + per-adapter MCP config**
Existing: `@axiom/isolation` already enforces an MCP allowlist and
project-scoped path isolation with cross-project collision checks
(`pathsAreIsolated`) — this is a working superset of addendum section 17's
isolation requirements, scoped to sub-paths of one repo root today.
Diff: extending isolation to cover genuinely separate repo roots (once
INC-01 lands multi-repo-by-default) rather than sub-paths of one root.
Breaking/compatible: additive extension of `buildIsolationContext` once
INC-01's repo model exists.
Dependencies: INC-13, INC-01.
Subagents: migration-engineer (audit `isolation/p0.ts` for what changes
when repos are no longer sub-paths of one root) -> schema-writer (mcp.yml)
-> adapter-engineer -> validator-reviewer.

**INC-16 — Optional external integrations as plugins**
Existing: `apps/cli/src/commands/app-plugins-azure-devops.ts` and
`app-plugins.ts` already exist — an Azure DevOps plugin is present today,
ahead of the new docs' description of this as a future addition.
Diff: confirm the existing plugin already treats Axiom Core as fully
functional without it (addendum section 10's core requirement) — likely
already true given the plugin naming convention, but needs verification,
not a build.
Breaking/compatible: none expected; mostly a documentation/verification
task once audited.
Dependencies: INC-15, INC-06 (`externalRefs`).
Subagents: migration-engineer (audit `app-plugins-azure-devops.ts` against
addendum section 10 and section 11's `externalRefs` model) ->
validator-reviewer.

**INC-17 — Optional general command router**
Existing: `apps/cli/src/commands/intent.ts` plus `orchestrator`'s 15
intent commands already look like the closest existing analog to a
"general router" (addendum section 12) — needs audit to see if it already
satisfies "specific commands remain primary."
Diff: unknown until audited.
Breaking/compatible: likely none; likely already compliant given the
orchestrator's design (intent commands sit behind specific gates, per
README's "7 comandos CLI pasan por el gate; los 15 intent commands
devuelven not-implemented").
Dependencies: INC-06, INC-13.
Subagents: migration-engineer (audit `intent.ts` + `orchestrator` intent
commands) -> validator-reviewer.

### Phase F — Bootstrap documental (Fase 6)

**INC-18 — Reconcile bootstrap-from-code**
Existing: `@axiom/document-bootstrap` renders known variables into
adapter files; it does not analyze a codebase to draft new
technical-context content. No `repo-analyzer`/`architecture-summarizer`
style subagent chain exists today.
Diff: mostly additive — the analysis/drafting pipeline (source doc 15.3
Vía A) does not exist; the rendering/writing tail-end
(path-guard, atomic write, team-block preservation) already exists in
`document-bootstrap/writer.ts` and should be reused for the final write
step instead of building a new writer.
Breaking/compatible: additive for the analysis chain; reuse existing
writer.
Dependencies: INC-06, INC-09/INC-10, INC-07.
Subagents: migration-engineer (confirm `document-bootstrap/writer.ts`
reusability for the final write step) -> bootstrap-analyzer (repo/backend/
frontend/qa/architecture chain) -> docs-skills-writer (drafts to
draft/review state per addendum section 15) -> index-engineer ->
validator-reviewer.

**INC-19 — Reconcile bootstrap-from-legacy-SDD**
Existing: no legacy-SDD migrator found among audited packages.
Diff: additive.
Breaking/compatible: additive, marked **(additive, no legacy package)**.
Dependencies: INC-06, INC-09/INC-10, INC-11.
Subagents: bootstrap-analyzer (legacy-spec-reader) -> migration-engineer
(increment-migrator, bug-migrator, plan-migrator — these ARE migration
tasks in the literal sense, reusing this role) -> docs-skills-writer
(context-normalizer, adr-extractor) -> cli-implementer
(axiom-format-writer, routed through Axiom commands) ->
validator-reviewer.

### Phase G — Versionado y upgrades (Fase 7)

**INC-20 — Reconcile version tracking (`@axiom/versioning`)**
Existing package: `@axiom/versioning` already has `managed-state.ts`,
`checkpoints.ts`, `migrations.ts`, `upgrade.ts`, `version.ts` — this is
already ahead of source doc section 16's description. A real MVP
migration (`0.0.0 -> 0.1.0`) already shipped with rollback support.
Diff: source doc's `version.yml` shape (`projectContractVersion`,
`installedWithAxiomVersion`, per-asset versions, `appliedMigrations`) vs
whatever shape `managed-state.ts` actually persists today — needs a
direct diff read (not yet done in this pass) before assuming compatible.
Breaking/compatible: likely additive/renaming only; the migration
*mechanism* already exists and works, so this increment is about
confirming/aligning the on-disk shape, not building versioning.
Dependencies: INC-01, INC-03.
Subagents: migration-engineer (read `managed-state.ts` in full to compare
against source doc 16.2's `version.yml` shape) -> schema-writer (only if a
real shape gap is found) -> validator-reviewer.

**INC-21 — Reconcile migration engine and upgrade commands**
Existing: `axiom upgrade` already supports `--dry-run`,
`--from-checkpoint`, `--target-version`, `--no-sync`, `--no-doctor`, and
is described in the repo's own `README.md` as "rollback-first design."
This already satisfies source doc 16.4's detect-pending -> show-diff ->
apply-with-backup -> validate flow almost verbatim.
Diff: per-repo change reporting (addendum section 8, "cambios detectados
por repo") may not exist yet since today's model is single-repo-default;
this becomes relevant only once INC-01 makes multi-repo primary.
Breaking/compatible: additive (add per-repo reporting once multi-repo
exists).
Dependencies: INC-20, INC-01.
Subagents: migration-engineer (confirm upgrade.ts's current output format)
-> cli-implementer (add per-repo change summary) -> validator-reviewer.

**INC-22 — Configure/upgrade/repair operations on existing installs**
Existing: `configure.ts` and `upgrade.ts` commands already exist and are
wired into the TUI (`tui/flows/configure.ts`, `upgrade.ts`). A distinct
`repair` operation (source doc section 17) was not confirmed among
audited commands.
Diff: `configure` and `upgrade` already exist; `repair` may be additive.
Breaking/compatible: additive for `repair`; `configure`/`upgrade` are
reconciliation-only (confirm they already cover source doc 17's
add/remove repo, role, adapter, tool/MCP operations).
Dependencies: INC-21, INC-04, INC-15.
Subagents: migration-engineer (audit `configure.ts`/`upgrade.ts` against
source doc 17's full operation list) -> cli-implementer (add `repair` if
missing) -> tui-developer -> validator-reviewer.

### Cross-cutting / not phase-bound

**INC-23 — Dogfooding boundary enforcement**
Existing: `@axiom/doctor`'s `runBoundaryChecks` (BC-001/BC-002/BC-003,
spec 0013) already checks that spec/product-runtime scopes are separated
within one `axiom.yml`. This is the closest existing analog to addendum
section 2's dogfooding rule, but scoped to "scopes inside one repo," not
"the `Axiom` product repo must not depend on `Axiom.SDD`/`Axiom.Spec`."
Diff: the specific cross-repo dogfooding check (this workspace's
`Axiom`/`Axiom.SDD`/`Axiom.Spec` separation) has no existing equivalent
because the existing product doesn't model cross-repo boundaries as
primary yet (see Q1).
Breaking/compatible: additive, dependent on INC-01's repo model landing
first.
Dependencies: INC-08, INC-01.
Subagents: migration-engineer (confirm BC-001/002/003 extension points)
-> validator-reviewer (defines the new check) -> registry-engineer
(implements it) -> validator-reviewer (final review).
Status: closed (2026-07-03). Full 4-role chain complete — see
`INC-20260702-dogfooding-boundary-reconcile-final-validator/README.md`
for the final review and closure summary. `DF-001` is implemented in
`Axiom/packages/doctor/src/checks.ts`, wired into `runDoctorChecks`, and
tested. This workspace's own `Axiom` repo currently `skip`s (no
`topology.yaml` declared) rather than `pass`es — an explicitly deferred
gap, not a defect (see that increment's scope decision).

**INC-24 — Workbench (optional, deferred)**
No existing package found; remains explicitly out of scope until INC-01
through INC-15 are stable, per addendum section 16.
Dependencies: INC-01 through INC-15 at minimum.
Subagents: not assigned — out of scope until explicitly requested.

## Validation

`No validation command was found. Performed best-effort validation by inspecting changed files and checking consistency against the requested behavior.`

Note: `Axiom/package.json` does define real validation commands (`npm run
build`, `npm test`, `npm run doctor`, `npm run typecheck`) and these should
be the actual validation gate once any of the increments above start
writing code. They were not run here because this increment made no code
changes to validate — running the existing test suite would only assert
the pre-existing state, not anything produced by this planning increment.

Best-effort validation performed for this planning increment:
- Directly read source files in `Axiom/packages/{topology,project-resolution,
  installer,doctor,versioning,cli-commands,skills,tui,isolation,
  document-bootstrap,core,user-workspace}` to ground every diff claim in
  actual code rather than package names/assumptions.
- Cross-checked `Axiom/README.md`'s own status table and package list
  against the claims made per increment.
- Confirmed git history (`git log --oneline`) shows this is an actively
  developed repo (last commit same day as this request), not stale
  legacy code — ruling out "ignore it, it's dead" as a valid path.
- Did not exhaustively read every one of the 26 packages line-by-line
  (e.g. `axiom-increment.ts`, `axiom-bug.ts`, `axiom-plan.ts`,
  `technical-context` equivalent, `adr`/`decision` commands, `tool-routing`
  dispatcher internals, `capability-model` provider registry, and
  `managed-state.ts`'s exact persisted shape were not read in full). This
  is flagged explicitly per-increment above ("not yet inspected," "needs
  audit," "unknown until audited") rather than guessed, and is the
  first task of the migration-engineer subagent in each such increment.

## Result

Produced a reconciliation-based 24-increment roadmap. Compared to the
initial greenfield draft, the key changes are: (1) most of Phase A, all of
Phase E's foundational pieces, and all of Phase G are now "audit and
extend an existing package" rather than "build from scratch"; (2) five
blocking open questions (Q1-Q5) were surfaced, all rooted in a genuine
architectural conflict between the existing single-repo-first product and
the new docs' separated-repos-first model; (3) several increments (INC-06,
INC-07, INC-10, INC-13, INC-17, INC-20, INC-21, INC-22) are explicitly
marked as needing a deeper line-by-line audit before their diff can be
called final — this roadmap gives their sequencing and starting hypothesis,
not a closed diff; (4) the existing capability/provider/telemetry/
orchestrator investment, which has no equivalent in the two source
documents, is explicitly called out for preservation rather than being
silently ignored or scoped for replacement.

## General spec integration

Superseded by the final consolidation pass recorded in "Roadmap closure
summary" below: all stable, cross-increment knowledge from this roadmap
was integrated directly into `Axiom.Spec/specs/00_Resumen_Ejecutivo.md`
through `08_Glosario.md`, per `Axiom.Spec/specs/README.md`'s own rule
against maintaining a separate legacy structure "solo por continuidad."
No standalone `general-spec.md` file exists or should be recreated. The
`03_Modelo_Operativo_y_Datos.md` "sin cerrar el naming final" placeholder
has been resolved with the actual shipped registry/manifest schema
(`~/.axiom/projects.yml` `schemaVersion: 2`, `axiom.yaml schemaVersion: 2`,
`TopologyManifest`) — see that file's "Topología de repos y registro
global" section.

## Next step recommendation

Start with **INC-01** (registry + manifest schema reconciliation), because
every other increment in every later phase either directly depends on it
(Phases B, C, most of D) or depends on the architectural direction it
forces (Phases E and G, via Q1/Q4). Before writing any code for INC-01,
resolve Q1, Q2, and Q5 explicitly with the user/product owner — these are
genuine architecture decisions (not implementation details) that determine
whether INC-01 is an additive schema extension or a breaking migration
with a cutover plan. Concretely: hold a short decision session (or async
written decision) on Q1/Q2/Q5 first, then open INC-01 as its own increment
spec in `Axiom.Spec/specs/increments/` with a migration-engineer performing
the full audit of `@axiom/user-workspace`, `@axiom/project-resolution`, and
`@axiom/topology` (reading every remaining file in those three packages,
not just the ones already read in this pass) as its first concrete task.

## Roadmap closure summary (2026-07-03)

All 23 increments in this roadmap (Phases A through G, plus the
cross-cutting INC-23 dogfooding boundary check), plus the D3 side-quest
(`axiom.yaml schemaVersion: 2` re-enablement, itself one of the two items
this roadmap's own original D1/D3 finding named), are now **closed**.
Each chain went through migration-engineer audit -> design/implementation
-> validator-review (or a subset of that sequence for genuinely additive,
no-legacy-package increments), with independent re-verification at the
terminal validator-reviewer step confirming no unresolved regressions.

This roadmap document's own `Status:` line stayed `pending` throughout
execution by convention — it is a planning-only index, not itself a
chain with a terminal validator-reviewer pass — but is updated to
`closed` now that every increment it sequences has closed and this final
consolidation pass has run.

One genuine exception, not a closed chain: `INC-20260702-
tui-menu-promote-inventory-screens`, a **not-started, deferred
placeholder** increment (raised as OQ1 during the TUI shell-detection
audit, explicitly deferred out of that increment's scope). It is archived
alongside the closed chains for record-keeping, but remains open work —
see `Axiom.Spec/specs/05_Interfaces_Operativas.md`'s TUI section.

**`Axiom.Spec/specs/00_Resumen_Ejecutivo.md` through `08_Glosario.md` are
now the canonical, current-state reference for the whole Axiom
architecture**, updated by subject (not by increment number) as part of
this closure pass. Read those files, not this roadmap or the individual
increment specs, to understand the current architecture. (An earlier
version of this closure pass consolidated everything into a standalone
`Axiom.Spec/general-spec.md` file; that was corrected per
`Axiom.Spec/specs/README.md`'s explicit rule against a separate legacy
structure, and the content was redistributed into the 8 topic files
instead. `general-spec.md` no longer exists.)

All 71 non-roadmap increment folders from this roadmap (including the one
not-started placeholder above) have been moved to
`Axiom.Spec/specs/increments/_archive/` — see that folder's own `README.md`
for what it contains and why it is historical detail rather than a
reading requirement. This roadmap file remains at the top level as the
master index and entry point into that history.

**INC-24 (Workbench)** remains explicitly deferred, per its own original
scope note above ("out of scope until INC-01 through INC-15 are stable,"
"Subagents: not assigned — out of scope until explicitly requested").
INC-01 through INC-15 have long since closed, but INC-24 itself was never
requested and is not opened by this closure pass — see "Deferred / not
yet built" in `Axiom.Spec/specs/03_Modelo_Operativo_y_Datos.md`
("Pendientes conocidos") and `Axiom.Spec/specs/02_Requisitos_No_
Funcionales.md`.
