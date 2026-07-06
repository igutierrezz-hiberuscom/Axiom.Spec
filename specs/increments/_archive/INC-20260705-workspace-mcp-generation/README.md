# Increment: Workspace MCP generation

Status: closed
Date: 2026-07-05

## Goal

Generate the SDD MCP and Spec MCP configuration during workspace setup,
writing it to both the canonical `.axiom/mcp.yml` project config and the
adapter-target-specific MCP config (`.opencode/mcp.json` or
`.claude/mcp.json`), wired as a best-effort step at the end of
`runWorkspaceSetup` (INC-20260705-workspace-multirepo-setup-engine), plus
a standalone, independently testable function.

## Context

The prior increment (`INC-20260705-workspace-multirepo-setup-engine`)
shipped `runWorkspaceSetup`, which scaffolds a multi-repo Axiom workspace
(control/SDD repo, spec repo, N role repos), wires per-repo `axiom.yaml`,
writes `axiom.config/topology.yaml`, and registers the project in the
user-level registry v2. By the time registration succeeds, the project id
resolves via `getProjectV2`, which is a hard prerequisite for
`validateMcpProjectConfig` (`packages/user-workspace/src/mcp-config.ts`).

The canonical MCP project config (`McpProjectConfig` /
`McpServerEntry`, schema/loader/validator in
`packages/user-workspace/src/mcp-config.ts`) already exists with
`loadMcpProjectConfig` + `validateMcpProjectConfig`, but **no writer
function and no path-resolution helper exist yet** — this increment adds
both, scoped to the workspace-setup path only (no changes to
`axiom configure`/`sync`).

The adapter projection generators `generateOpencodeMcpJson`
(`@axiom/adapters-opencode`) and `generateClaudeCodeMcpJson`
(`@axiom/adapters-claude-code`) already exist, are tested, and are
already exported from their package barrels — they were simply never
wired into any flow. `GENERATED_FILES_BY_TARGET`
(`packages/installer/src/registry.ts`) intentionally omits
`.opencode/mcp.json` / `.claude/mcp.json` per the TR-005 deferral for the
`configure`/`sync` path; that deferral is left intact. This increment
wires the generators only into the new workspace-setup path.

## Scope

- `buildWorkspaceMcpServers(repos)` in
  `apps/cli/src/commands/workspace-mcp.ts`: derives the `sdd-mcp-server` +
  `spec-mcp-broker` repo-scoped `McpServerEntry[]` from a
  `WorkspaceSetupResult['repos']` list.
- `writeWorkspaceMcpConfig(args)` in the same file: builds
  `McpProjectConfig`, validates it (`validateMcpProjectConfig`, reused,
  not re-implemented), writes `.axiom/mcp.yml` in the control repo (new
  atomic writer, since none existed), and projects to
  `.opencode/mcp.json` / `.claude/mcp.json` via the existing generators
  when `target` is `'opencode'` / `'claude-code'`; for any other target,
  skips adapter projection and records a warning.
- Wiring into `runWorkspaceSetup` (`apps/cli/src/commands/workspace-setup.ts`):
  best-effort call after successful registration; appends produced paths
  to `filesCreated`, issues to `warnings`; never fails the overall setup.
- Focused tests: `apps/cli/tests/workspace-mcp.test.ts` (unit tests for
  `buildWorkspaceMcpServers` + `writeWorkspaceMcpConfig`) and new cases in
  `apps/cli/tests/workspace-setup.test.ts` covering the end-to-end wiring.

## Non-goals

- No changes to `axiom configure`/`sync`, `GENERATED_FILES_BY_TARGET`, or
  the TR-005 deferral for the non-workspace path.
- No changes to `axiom.config/mcp-manifest.yaml` or `axiom mcp` (a
  separate, human-facing capability catalog, unrelated to this machine
  config — confirmed via `packages/user-workspace/src/mcp-config.ts`'s own
  header comment on "Q-mcp-1").
- No MCP entries for role/code repos (SDD + Spec MCP only).
- No TUI wizard surface (INC-C, a sibling increment, is out of scope
  here).
- No git operations of any kind.

## Acceptance criteria

- [x] `buildWorkspaceMcpServers` returns exactly two entries
      (`sdd-mcp-server` targeting the control repo's `roleKey`,
      `spec-mcp-broker` targeting the spec repo's `roleKey`), both
      `type: 'axiom'`, `scope: 'repo'`, `enabled: true`.
- [x] When control and spec collapse to the same `roleKey` (degenerate
      single-repo workspace), only the `sdd-mcp-server` entry is emitted
      plus a warning explaining the collision — no invalid
      `duplicate-type-target-repo` pair is produced.
- [x] `writeWorkspaceMcpConfig` writes a `.axiom/mcp.yml` in the control
      repo that round-trips through `loadMcpProjectConfig` and passes
      `validateMcpProjectConfig` with zero issues for the standard
      (non-degenerate) case.
- [x] For `target: 'opencode'`, `.opencode/mcp.json` is written in the
      control repo with both enabled servers mirrored verbatim (no
      invented transport fields).
- [x] For `target: 'claude-code'`, `.claude/mcp.json` is written
      analogously.
- [x] For any other target, `.axiom/mcp.yml` is still written, no adapter
      file is produced, and a warning is recorded naming the
      unsupported target.
- [x] `runWorkspaceSetup` calls this wiring only after a successful
      registration (`register !== false` and `registryRegistered: true`);
      if registration was skipped or failed, MCP generation is skipped
      and a warning explains why; overall setup never fails because of
      this step (try/catch, warnings only).
- [x] `npm run build` (tsc -b) is clean from `Axiom/`.
- [x] New/extended tests in `apps/cli` pass; no regressions in
      `packages/user-workspace` or `packages/adapters`.

## Open questions

None blocking — the increment prompt supplied fully closed design
decisions (function contracts, wiring point, server ids/types,
unsupported-target behavior). Narrow implementation choices (exact
`.axiom/mcp.yml` path resolution root, atomic-write helper reuse) are
recorded under Assumptions since no writer/path-helper pre-existed to
just call.

## Assumptions

- `.axiom/mcp.yml` is resolved relative to the **control repo path**
  (`path.join(controlRepoPath, '.axiom', 'mcp.yml')`) — the control repo
  is the one place `runWorkspaceSetup` already treats as the
  project-scoped anchor for generated, non-per-repo artifacts
  (`axiom.config/topology.yaml`, `.axiom-state/local/topology-bindings.yaml`
  are both written there, not in the spec repo). No existing code path
  reads `mcp.yml` today (only the loader/validator exist, uncalled), so
  there is no established root to defer to; this choice is consistent
  with the existing project-scoped-artifact convention and is stated
  explicitly here as the authoritative resolution for future callers.
- No writer function existed for `mcp.yml` (`loadMcpProjectConfig` /
  `validateMcpProjectConfig` are load/validate only). `writeWorkspaceMcpConfig`
  serializes `McpProjectConfig` with `js-yaml` and writes it via the same
  atomic tmp+rename helper already exported from `init.ts`
  (`atomicWriteFile`), matching every other writer in this file — no new
  YAML-writing pattern introduced.
- `buildWorkspaceMcpServers` takes `WorkspaceSetupResult['repos']` (not
  the full `WorkspaceRepoSpec[]`) since that is the shape already
  available at the call site inside `runWorkspaceSetup` after step 1
  completes, and is sufficient (only `roleKey` + structural role are
  needed) — the control/spec `roleKey`s are identified by re-deriving
  `kind` from the original `spec.repos` list passed alongside.
- Validation failures from `validateMcpProjectConfig` are surfaced as
  warnings (one per issue, prefixed for traceability) rather than thrown,
  per the increment prompt's explicit instruction; the `.axiom/mcp.yml`
  file is still written even if validation reports issues, since the
  prompt frames this whole step as best-effort and the write already
  happened by the time validation runs (validate-after-write, matching
  how `runWorkspaceSetup`'s own no-clobber checks work: report, don't
  roll back).
- `homeDirOverride` is threaded through to `writeWorkspaceMcpConfig` (via
  `resolveHomeDir`) so `validateMcpProjectConfig`'s registry lookup uses
  the same home directory the registration step itself just used.

## Implementation notes

- New file `apps/cli/src/commands/workspace-mcp.ts` (sibling to
  `workspace-setup.ts`, imported by it) hosts `buildWorkspaceMcpServers`
  and `writeWorkspaceMcpConfig`, keeping `workspace-setup.ts` focused on
  orchestration.
- `writeWorkspaceMcpConfig` reuses `McpServerEntry`/`McpProjectConfig`
  types and `validateMcpProjectConfig` from `@axiom/user-workspace`
  verbatim (re-exported already), and `generateOpencodeMcpJson` /
  `generateClaudeCodeMcpJson` from `@axiom/adapters-opencode` /
  `@axiom/adapters-claude-code` verbatim (both already declared
  dependencies of `apps/cli` and already exported from their barrels —
  no new package.json edits needed).
- Wiring point in `runWorkspaceSetup`: after the existing
  `upsertProjectReposV2` success branch, inside a `try/catch` that can
  never escape and fail the overall setup.

## Validation

From `C:\repos\Axiom Workspace\Axiom`:

- `npm run build` (tsc -b) — clean, no errors (ran twice: once right
  after wiring `workspace-mcp.ts`, once after adding the test file; both
  clean).
- `npx vitest run apps/cli packages/user-workspace packages/adapters`:

  ```
  Test Files  65 passed (65)
       Tests  563 passed (563)
  ```

  0 failures in this exact filtered scope. 14 of the 563 are new tests
  from this increment (9 in `apps/cli/tests/workspace-mcp.test.ts`
  covering `buildWorkspaceMcpServers`, standalone
  `writeWorkspaceMcpConfig`, and end-to-end `runWorkspaceSetup` wiring
  for `opencode`/`claude-code`/unsupported-target/`register:false`; 5
  pre-existing in `apps/cli/tests/workspace-setup.test.ts`, unmodified and
  still green). Console `[orchestrator] FAIL: command=...` lines seen
  during the run are pre-existing log output from OTHER tests that
  intentionally exercise failure paths (e.g. `sync-command`,
  `init-command`, `join-command`, `start-command` negative cases) — not
  actual vitest test failures; confirmed by the `65 passed (65)` /
  `563 passed (563)` summary. The ~11 known pre-existing dogfood/
  real-repo integration failures mentioned in the brief were not
  exercised (they live outside this three-package filter) and did not
  surface.

## Result

Implemented `buildWorkspaceMcpServers` + `writeWorkspaceMcpConfig` in
`apps/cli/src/commands/workspace-mcp.ts`, wired as a best-effort final
step of `runWorkspaceSetup`. `.axiom/mcp.yml` is written in the control
repo; `.opencode/mcp.json` / `.claude/mcp.json` are projected for
supported targets; unsupported targets get `.axiom/mcp.yml` only plus a
warning. Registration-skipped/failed cases skip MCP generation with an
explanatory warning. Build clean; new focused tests pass.

## General spec integration

Integrated into the canonical `Axiom.Spec/specs/00-08` in the final
cross-increment pass covering this increment and its two siblings
(`INC-20260705-workspace-multirepo-setup-engine`,
`INC-20260705-tui-workspace-setup-wizard`). Files this increment's
knowledge landed in:

- `03_Modelo_Operativo_y_Datos.md` — updated the `mcp.yml` subsection to
  supersede the stale "no está aún cableado a producción" claim: recorded
  `runWorkspaceSetup` as the first/only live generator call site, and
  fixed the file-location claim from `<specRoot>/.axiom/mcp.yml` to
  `<controlRepo>/.axiom/mcp.yml` (authoritative resolution), while keeping
  the accurate note that `configure`/`sync` still don't call the
  generators (TR-005 deferral intact).
- `06_Integraciones_y_Capacidades.md` — added the subsection **"Generación
  de config MCP durante el setup de workspace (SDD MCP + Spec MCP)"**:
  `.axiom/mcp.yml` shape with the two servers (`sdd-mcp-server` /
  `spec-mcp-broker`), `targetRepo` binding to registry repo keys, adapter
  projection (`.opencode/mcp.json` / `.claude/mcp.json`), best-effort
  semantics, and the `.axiom/` gitignore caveat. Also reconciled the
  TR-005+ doctor-check deferral wording in `07` (see below).
- `05_Interfaces_Operativas.md` — updated the "Adapters — MCP config por
  proyecto" subsection heading/body from "(no cableado a producción)" to
  "(cableada por el setup de workspace)".
- `07_Gobierno_y_Seguridad.md` — reconciled the stale clause claiming no
  scaffold writes `mcp.yml`; noted that workspace setup now writes a real
  `mcp.yml` validated in line via `validateMcpProjectConfig` (not via a
  doctor check), so the future `mcp.yml` doctor-check remains deferred.
- `08_Glosario.md` — added the terms "`mcp.yml`" and "SDD MCP / Spec MCP".
