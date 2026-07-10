# Increment: incremental-operations

Status: closed
Date: 2026-07-08

## Goal

Close (part of) the documented "7-operation gap" (NFR-002): today you cannot
ADD a repo / adapter / provider / role to an EXISTING Axiom installation
without re-running full workspace setup. Expose idempotent, no-clobber
INCREMENTAL operations (CLI commands + reusable functions) that operate on
an already-initialized project resolved from cwd, reusing the existing
multi-repo engine (`runWorkspaceSetup` + its helpers) instead of
duplicating scaffolding logic.

## Context

The multi-repo engine and all its pieces already exist and are reusable:

- `runWorkspaceSetup` + `writeOneRepo` (reciprocal `axiom.yaml#paths`,
  topology.yaml, `.axiom-state`) — `apps/cli/src/commands/workspace-setup.ts`.
- `upsertProjectReposV2` (`@axiom/user-workspace`) — already merges repos
  into an existing project entry (idempotent).
- `generateWorkspaceAdapters` (`workspace-adapters.ts`) + native MCP
  (`native-mcp-config.ts`) + `writeWorkspaceMcpConfig`/
  `writeWorkspaceNativeMcpConfigs` (`workspace-mcp.ts`).
- `scaffoldCodeRepoSkills`/`scaffoldSddSkills` + `scaffoldRules`
  (`workspace-skills.ts`/`workspace-rules.ts`).
- `buildProjectProviderRegistry` (`@axiom/providers`) reads
  `workspace.json#providers` directly — no separate "registration" step
  needed beyond persisting the id there.
- Topology writer (`buildTopologyManifest`/`writeTopologyManifest` in
  `workspace-setup.ts`) + `loadTopology`/`RoleAssignment`/`RepoRef`
  (`@axiom/topology`).

`axiom configure --providers <csv>` already exists but is a FULL-REPLACE,
single-repo (`init.json`-based) surface, not additive/multi-repo-aware —
does not fit "add one provider to an existing multi-repo project".
`axiom repo attach` only registers a path in the user-level registry; it
does not scaffold `axiom.yaml`/topology/adapters/skills for the new repo.

## Scope

Implement 4 ADD-only incremental operations, each idempotent + merge/
no-clobber, resolving the target project from cwd:

1. **`axiom repo add`** — add a new repo (any `kind`: `role` by default,
   or `control`/`spec`) to an existing project. Writes the new repo's
   `axiom.yaml` with reciprocal `paths` to existing siblings AND updates
   every EXISTING sibling's `axiom.yaml#paths` to reference the new repo;
   updates `axiom.config/topology.yaml` (`roleCodeRepositories`/
   `assignments` for `role` repos); `upsertProjectReposV2` to register it;
   generates adapters/native-MCP/skills/rules for the project's currently
   enabled adapters (read from `workspace.json`).
2. **`axiom adapter add <target>`** — add a new adapter target: persists
   into `workspace.json#adapters` (append, dedup) and generates that
   adapter's output set (incl. native MCP) across ALL repos of the project
   (read from the registry).
3. **`axiom provider add <id>`** — add a provider id: persists into
   `workspace.json#providers` (append, dedup). Best-effort local
   validation via `buildProjectProviderRegistry` (confirms the id
   registers/resolves); no external tool is spawned.
4. **`axiom role add <roleId> --path <path>`** — shorthand combining
   `repo add --kind role --role-id <roleId>` with the topology role
   assignment and per-repo skills/rules; thin wrapper over op #1.

Remove-operations (`repo remove`, `adapter remove`, `provider remove`,
`role remove`) are explicitly OUT OF SCOPE — documented as a follow-up
below, not implemented.

## Non-goals

- No `remove`/`detach` operations (deferred; symmetric follow-up).
- No changes to `runWorkspaceSetup`'s full-setup behavior/contract.
- No changes to `axiom configure`/`axiom repo attach` behavior.
- No new enterprise lifecycle constructs.
- No external tool invocation beyond what already exists
  (`buildProjectProviderRegistry` never spawns; it only registers clients).

## Acceptance criteria

- [x] `axiom repo add` on an existing multi-repo project: new repo gets
      `axiom.yaml` + reciprocal `paths`; existing siblings' `axiom.yaml`
      gain a path entry for the new repo; `topology.yaml` updated;
      registry (`projects.yml`) has the new repo key; adapters/skills/
      rules generated for the new repo per the project's enabled
      adapters. Re-running with the same args is a no-op/merge (no dup
      entries, no clobbered files).
- [x] `axiom adapter add <target>`: `workspace.json#adapters` gains the
      target (deduped); that adapter's output set is generated across all
      registered repos of the project. Idempotent re-run does not
      duplicate the entry and does not clobber already-customized adapter
      files (best-effort generators already guard writes).
- [x] `axiom provider add <id>`: `workspace.json#providers` gains the id
      (deduped); `buildProjectProviderRegistry` includes it. Idempotent
      re-run does not duplicate.
- [x] `axiom role add`: end-to-end — new code repo + topology role
      assignment + per-repo skills/rules, built on top of `repo add`.
- [x] Full existing suite stays green (0 regressions vs. baseline).

## Open questions

None blocking. Ambiguities resolved under Assumptions below.

## Assumptions

- **Project resolution**: incremental ops resolve the existing project
  from cwd via `resolveProject` (same as `axiom repo attach`/`roles`),
  then read the registry entry (`getProjectV2`/`findByRepoPathV2`) to
  discover sibling repo paths, and read `workspace.json` (scanned per
  `readEnabledProviders`'s pattern) to discover currently-enabled
  adapters/providers. If no `workspace.json` exists yet (project was
  never through `runWorkspaceSetup`/wizard), `adapter add`/`provider add`
  create a minimal one (schemaVersion 1, empty arrays) rather than
  failing — additive, no prerequisite full-setup run required.
- **`repo add` default kind**: `role` (the common case — "add a code
  repo for a new role/service"). `--kind control|spec` is accepted for
  completeness but a project already has exactly one of each per
  `runWorkspaceSetup`'s invariant; adding a second `control`/`spec` repo
  is rejected with a clear error (that invariant is NOT relaxed by this
  increment).
- **Reciprocal wiring on existing siblings**: reused `buildRoleAwareAxiomYaml`
  (exported, not duplicated) is re-run for EVERY existing repo of the
  project (not just the new one) so its `paths` map picks up the new
  repo — this is the "existing siblings' `paths` gain a path entry for
  the new repo" requirement. This is safe/no-clobber because
  `buildRoleAwareAxiomYaml` fully re-derives the `paths` block from the
  full repo set deterministically (same function `runWorkspaceSetup`
  itself uses on every repo, every run) — it does not touch any other
  section of an existing `axiom.yaml`.
- **Adapters/skills/rules generation scope for `repo add`**: only the
  NEW repo receives adapters/skills/rules generation (existing repos are
  untouched beyond their `axiom.yaml#paths` refresh) — mirrors
  `runWorkspaceSetup`'s own gate-on-`created` semantics, just applied to
  one repo instead of a batch.
- **`provider add` "registration"**: per `buildProjectProviderRegistry`'s
  own design (see `packages/providers/src/project-registry.ts` doc
  comment), there is no separate init/registration step beyond persisting
  the id into `workspace.json#providers` — the registry is rebuilt from
  that file on every read. "Best-effort init" for `provider add` means
  calling `buildProjectProviderRegistry` once after persisting, to
  surface a warning if the id doesn't resolve to a known provider
  (typo-catching), never spawning anything.
- **CLI ownership**: new commands are app-owned
  (`apps/cli/src/commands/workspace-incremental.ts`), registered directly
  in `apps/cli/src/index.ts` — mirrors `mcp-serve.ts`'s pattern (not
  re-exported through `@axiom/cli-commands`) to avoid a circular
  reference back into `workspace-setup.ts` internals.

## Implementation notes

- New file `apps/cli/src/commands/workspace-incremental.ts` holds
  `runRepoAdd`, `runAdapterAdd`, `runProviderAdd`, `runRoleAdd` +
  `registerWorkspaceIncremental(program)`. Registered in
  `apps/cli/src/index.ts` alongside the other command registrations.
- Shared-helper refactor in `workspace-setup.ts`: exported
  `buildRoleAwareAxiomYaml`, `writeOneRepo`, `buildTopologyManifest`,
  `writeTopologyManifest`, `axiomYamlPathFor`, `tryReadExistingProjectId`,
  `relativeRef` (previously module-private) so the incremental ops reuse
  the EXACT SAME per-repo scaffolding path `runWorkspaceSetup` uses —
  avoids any full-setup vs. incremental divergence. No behavior change to
  `runWorkspaceSetup` itself (pure export widening).
- `workspace.json` read/write for the incremental ops reuses the same
  scan pattern as `@axiom/providers`'s `readEnabledProviders`
  (`.axiom-state/<anyProjectId>/workspace.json`) since the incremental
  commands do not always know `effectiveProjectId` up front (mirrors how
  `buildProjectProviderRegistry` itself resolves it).

## Validation

`npm run build`:

```
> axiom-product@0.1.0 build
> tsc -b
```

(clean exit, no diagnostics)

`npx vitest run apps/cli packages/user-workspace packages/topology packages/providers`:

```
 Test Files  79 passed (79)
      Tests  785 passed (785)
```

(includes the 11 new tests in `apps/cli/tests/workspace-incremental.test.ts`)

`npx vitest run --no-file-parallelism` (full suite):

```
 Test Files  199 passed (199)
      Tests  2130 passed (2130)
```

Baseline was 2119/2119 (per this batch's brief); 2130 = 2119 + 11 new
tests. 0 regressions.

## Result

Implemented `axiom repo add`, `axiom adapter add`, `axiom provider add`,
`axiom role add` as thin CLI commands over the existing multi-repo
engine's exported helpers (`writeOneRepo`, `buildRoleAwareAxiomYaml`,
`buildTopologyManifest`, `writeTopologyManifest`, `relativeRef`, all
widened from module-private to exported in `workspace-setup.ts` — no
behavior change to `runWorkspaceSetup` itself). All four operations are
idempotent and no-clobber (re-running with the same args merges/no-ops;
never overwrites unrelated content; verified by dedicated idempotency
tests for each op). Remove-operations deliberately deferred (documented
above as a follow-up, not implemented).

New/changed files:
- `apps/cli/src/commands/workspace-incremental.ts` (new) — `runRepoAdd`,
  `runAdapterAdd`, `runProviderAdd`, `runRoleAdd` +
  `registerWorkspaceIncremental`.
- `apps/cli/src/commands/workspace-setup.ts` — widened exports only
  (`axiomYamlPathFor`, `tryReadExistingProjectId`, `relativeRef`,
  `buildRoleAwareAxiomYaml`, `writeOneRepo`, `buildTopologyManifest`,
  `writeTopologyManifest`).
- `apps/cli/src/index.ts` — registers `registerWorkspaceIncremental`
  after `registerRepo`.
- `apps/cli/tests/workspace-incremental.test.ts` (new) — 11 focused
  tests covering all 4 ops + idempotency + rejection paths.

## General spec integration

Integrated into `Axiom.Spec/specs/00-08` by the batch's final
consolidation pass (autopilot orchestrator). The canonical structure is
the numbered `specs/00_..08_*.md` set (there is no `general-spec.md`
file). Landed in:

- **02_Requisitos_No_Funcionales.md** — new NFR-AXM-015 "Operaciones
  incrementales sobre instalaciones existentes (hueco de 7 operaciones
  — parcialmente cerrado)": documents the gap that `01`/`04` already
  referenced and records that the 4 ADD ops ship (REMOVE deferred).
- **05_Interfaces_Operativas.md** — the 4 CLI commands (`axiom
  repo/adapter/provider/role add`) in the new INC-20260708 command
  subsection.
- **03_Modelo_Operativo_y_Datos.md** — idempotency/no-clobber semantics
  and the exported `workspace-setup.ts` helpers reused, in the
  INC-20260708 data subsection.
- **01_Requisitos_Funcionales.md** — RF-AXM-028 (incremental ADD ops);
  the RF-AXM-022 cross-reference updated to note partial closure.
- **04_Flujos_SDD_y_Ciclo_de_Vida.md** — the `configure` gap note
  updated to point at the ADD ops.
- **08_Glosario.md** — new term: incremental operations (ADD).
