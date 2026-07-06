# Increment: Workspace multi-repo setup engine

Status: closed
Date: 2026-07-05

## Goal

Build the CORE ENGINE (no TUI, no MCP yet) that scaffolds and wires a
MULTI-REPO Axiom workspace in one operation: an SDD/control repo, a Spec
repo, and N functional-role code repos (backend/frontend/qa-e2e/custom),
each parameterized so the repos "know each other" via role-aware
`axiom.yaml#paths`, one `axiom.config/topology.yaml`, and one registration
in the user-level registry v2 (`~/.axiom/projects.yml`). This is the
foundation two later increments build on (MCP generation and a TUI
wizard), so the exported public contract must be stable.

## Context

Single-repo scaffolding (`apps/cli/src/commands/init.ts`'s `runInit`)
already writes `axiom.yaml`, `.gitignore`, `.axiom-state/`, and registers
ONE repo via `addProjectV2`. It has no notion of a multi-repo workspace:
`buildAxiomYaml` hardcodes `paths.control`/`paths.specification` for the
`installed-multi-repo` layout regardless of `role`, and there is no way to
scaffold or parameterize N additional role-bound code repos, nor to
generate `axiom.config/topology.yaml` describing them.

Registry v2 (`@axiom/user-workspace`) already supports multi-repo
projects in its data model (`addProjectV2` accepts a `repos` map keyed by
an open `RepoRoleKey` string) but only `init.ts` calls it, always with a
single repo. Topology (`@axiom/topology`) already models `sddRepo` /
`specRepo` / `roleCodeRepositories` / `assignments`, with loaders
(`loadTopology`, `saveLocalBindings`) and a known atomic-YAML-write
pattern (`apps/cli/src/commands/roles.ts`'s local `writeTopologyYaml`).

This increment wires these existing primitives together behind one new
entry point, `runWorkspaceSetup`, without touching `runInit`'s behavior.

## Scope

- New file `apps/cli/src/commands/workspace-setup.ts` exporting
  `runWorkspaceSetup(spec: WorkspaceSetupSpec): Promise<WorkspaceSetupResult>`
  and the public types `WorkspaceRepoKind`, `WorkspaceRepoSpec`,
  `WorkspaceSetupSpec`, `WorkspaceSetupResult` (exact shapes below).
- Role-aware `axiom.yaml` generation per repo (control / spec / role),
  with reciprocal relative `paths` entries computed via `path.relative`.
- Guard against clobbering a pre-existing `axiom.yaml` that was not
  authored for this `projectId`.
- One `axiom.config/topology.yaml` written in the control repo
  (`mode: multi-repo` when there are role repos or a distinct spec repo,
  else `single-repo`), plus `.axiom-state/local/topology-bindings.yaml`
  via `saveLocalBindings`.
- One `addProjectV2` call registering all repos under one project id, with
  an idempotent merge path (new `upsertProjectReposV2` helper in
  `@axiom/user-workspace`) for the case where the control repo (or any
  repo) is already registered.
- Scaffolding for brand-new repo directories (`create: true`) and
  in-place parameterization for existing directories (`create: false`).
- Focused tests in `apps/cli/tests/workspace-setup.test.ts` and
  `packages/user-workspace/tests/` for the new helper.

## Non-goals

- No TUI wizard surface (a later increment, INC-C, consumes this engine).
- No MCP generation (a later increment, INC-B, consumes this engine).
- No `--workspace` CLI flag / `axiom` command surface in this increment.
- No changes to `runInit`'s behavior, its CLI, or its tests.
- No closed enum for functional/implementation roles (`backend`,
  `frontend`, `qa-e2e` stay open strings, per existing
  `install-profiles` vocabulary).
- No git operations of any kind.

## Acceptance criteria

- [x] `runWorkspaceSetup` scaffolds a fresh 3-repo workspace
      (control + spec + one role repo), each with a correct, reciprocal
      `axiom.yaml#paths` map, in one call.
- [x] `axiom.config/topology.yaml` is written once, in the control repo,
      with `sddRepo`/`specRepo`/`roleCodeRepositories`/`assignments`
      matching the input spec, and round-trips through `loadTopology`.
- [x] `.axiom-state/local/topology-bindings.yaml` is written in the
      control repo mapping every `topologyId` to its absolute path.
- [x] Exactly one `addProjectV2` (or idempotent merge) call registers all
      repos under one project id in `projects.yml`.
- [x] An existing repo directory passed with `create:false` is
      parameterized in place (its `axiom.yaml` written/updated) without
      being otherwise scaffolded as if new, and without clobbering a
      pre-existing `axiom.yaml` that belongs to a different project.
- [x] Re-running `runWorkspaceSetup` with the same spec is idempotent: it
      merges into the existing registry entry instead of throwing on
      `duplicate-id`.
- [x] `runInit`/`axiom init` behavior and its existing test suite are
      unchanged.
- [x] `npm run build` (tsc -b) is clean from `Axiom/`.
- [x] New tests in `apps/cli` and `packages/user-workspace` pass; no
      regressions beyond the known pre-existing failing tests.

## Open questions

None blocking — the increment prompt supplied fully closed design
decisions (public contract, wiring rules, registry merge strategy).
Narrow implementation choices are recorded under Assumptions.

## Assumptions

- `projectId` = `slugifyProjectId(projectName)`; `axiom.yaml#name` uses
  the human `projectName` when it satisfies `PROJECT_NAME_REGEX`,
  otherwise falls back to the slug (mirrors `tui.ts`'s
  `slugifyProjectName` pattern referenced in the prompt).
- "Axiom-managed file for THIS project" guard is implemented by parsing
  the existing `axiom.yaml` (schemaVersion 2) and comparing its
  `projectId` field to the one being written; if it parses but the id
  differs (or is v1 / unparseable-but-present), the write is skipped and
  recorded as a warning, not an error.
- `upsertProjectReposV2` is added to `@axiom/user-workspace` as a small
  reusable helper (load → merge repos map (new repo keys win on
  conflict, since a re-run means "this workspace now knows this path") →
  save), with its own focused unit test, rather than inlining the
  load/merge/save in the engine — reusable for future callers (`repo
  attach`, INC-B/INC-C) per the prompt's stated preference.
- Registration remains best-effort (never throws the whole setup): any
  registry failure sets `registryRegistered:false` and the engine still
  returns the files/topology it wrote, mirroring `runInit`'s existing
  contract.
- `qaLane` is always written as `'inline'` (matches the existing
  `defaultSingleRepoManifest`/`defaultInstalledMultiRepoManifest`
  default; the prompt does not ask for a QA lane option here).
- For the "two repos share the same dir" degenerate collapse case: if
  `control` and `spec` (or any two repos) resolve to the same absolute
  path, the reciprocal path for that pair is written as `.` instead of a
  computed relative path (avoids emitting a self-referential non-`.`
  relative path).

## Implementation notes

See the increment's own `Implementation notes` maintained inline in code
comments in `apps/cli/src/commands/workspace-setup.ts` (function-level
docblocks) — this spec file summarizes the outcome, not a duplicate
narrative, per `Axiom.SDD/AGENTS.md`'s rule against copying implementation
history into canonical specs at excessive detail. Key points:

- `buildRoleAwareAxiomYaml` is a new, separate builder from `init.ts`'s
  `buildAxiomYaml` (which stays untouched for the single-repo path);
  it accepts the full repo list so it can compute reciprocal paths.
- `upsertProjectReposV2` (new export, `packages/user-workspace/src/registry.ts`)
  performs load → merge (new repo keys overwrite existing keys with the
  same name; other existing repos are preserved) → save, and creates the
  project if it does not exist yet (mirrors `addProjectV2`'s empty-repos
  guard).
- The control repo is the only repo that receives `axiom.config/topology.yaml`
  and `.axiom-state/<projectId>/init.json`-style metadata; role/spec repos
  only get `axiom.yaml` + `.gitignore` (new dirs) / `axiom.yaml` (existing
  dirs) + best-effort `AGENTS.md` + `.axiom-state/local/` +
  `.axiom-state/<projectId>/`.

## Validation

From `C:\repos\Axiom Workspace\Axiom`:

- `npm run build` (tsc -b) — clean, no errors.
- `npx vitest run apps/cli packages/user-workspace packages/topology`:

  ```
  Test Files  57 passed (57)
       Tests  515 passed (515)
  ```

  0 failures within this exact validation scope (10 of the 515 are the
  new tests added by this increment: 5 in
  `apps/cli/tests/workspace-setup.test.ts`, 5 in
  `packages/user-workspace/tests/upsert-project-repos-v2.test.ts`). The
  ~11 pre-existing failing tests mentioned in the increment brief (real-
  repo/dogfood integration tests expecting `axiom.config/*.yaml`) live
  outside this three-path filter and were not exercised by this
  validation scope; `apps/cli/tests/init.test.ts` (16 tests, `runInit`'s
  own suite) stayed green and unmodified in behavior.

Verbatim tails captured in the final delivery message of this increment's
implementation session.

## Result

Implemented `runWorkspaceSetup` in `apps/cli/src/commands/workspace-setup.ts`
plus `upsertProjectReposV2` in `packages/user-workspace/src/registry.ts`,
per the closed design in this spec. `runInit`/`axiom init` untouched and
its test suite stays green. New focused tests added and passing. Build
clean.

## General spec integration

Integrated into the canonical `Axiom.Spec/specs/00-08` in the final
cross-increment pass covering this increment and its two siblings
(`INC-20260705-workspace-mcp-generation`,
`INC-20260705-tui-workspace-setup-wizard`). Files this increment's
knowledge landed in:

- `00_Resumen_Ejecutivo.md` — updated the "Qué hace el producto hoy"
  bootstrap paragraph to say the init action now prepares a full
  multi-repo workspace (control/Spec/role repos) in one operation.
- `01_Requisitos_Funcionales.md` — added **RF-AXM-024 Setup de workspace
  multi-repo desde el wizard de la TUI**, whose data-model half describes
  `runWorkspaceSetup`'s reciprocal per-repo `axiom.yaml#paths`, the
  control-repo `topology.yaml` + `topology-bindings.yaml`, the one-call
  registry v2 registration via `upsertProjectReposV2`, and the no-clobber
  guard.
- `03_Modelo_Operativo_y_Datos.md` — added the subsection **"Setup de
  workspace multi-repo en una operación (`runWorkspaceSetup`)"** under
  "Topología de repos y registro global", describing the engine's written
  data: role-aware reciprocal `axiom.yaml#paths`, the single control-repo
  `axiom.config/topology.yaml`, `.axiom-state/local/topology-bindings.yaml`,
  and one-call idempotent registry v2 registration via
  `upsertProjectReposV2`.
- `07_Gobierno_y_Seguridad.md` — added the no-clobber guard as an
  ownership-safety rule under "Trazabilidad de write-scope y ownership".
- `08_Glosario.md` — added the terms "Workspace setup", "Repo de control
  (SDD)", "Repo de código de rol", and "`topology-bindings.yaml`".
