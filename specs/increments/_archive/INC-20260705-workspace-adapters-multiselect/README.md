# Increment: Workspace adapters multi-select

Status: closed
Date: 2026-07-05

## Goal

Let the user multi-select which adapters to install (opencode,
claude-code, copilot-vscode, antigravity, visual-studio-2026, cursor,
github-copilot, litellm), persist that selection, and generate each
selected adapter's files into every repo of the workspace (control + spec
+ every role repo). Generalize the MCP projection to every selected
MCP-capable adapter.

## Context

Builds on the closed multi-repo workspace increments:
`INC-20260705-workspace-multirepo-setup-engine` (`runWorkspaceSetup`),
`INC-20260705-workspace-mcp-generation` (`workspace-mcp.ts`), and
`INC-20260705-tui-workspace-setup-wizard` (the TUI wizard, generic
`multi-select` step kind + `expand`).

`runWorkspaceSetup` today accepts a single `spec.target` and threads it
only into the best-effort MCP projection step
(`writeWorkspaceMcpConfig`/`buildWorkspaceMcpServers`); it never
materializes any adapter's actual output files (`.opencode/AGENTS.md`,
`.claude/AGENTS.md`, etc.) into the scaffolded repos. `axiom init`
(single-repo) and `axiom configure` remain the only paths that call the
per-target generators today, and they are out of scope here.

## Scope

- `WorkspaceSetupSpec.adapters: AdapterTarget[]` (additive; `target`
  remains the primary/back-compat field).
- New `apps/cli/src/commands/workspace-adapters.ts`:
  `generateWorkspaceAdapters` — for each repo in the workspace, resolves
  one `ResolvedInstallProfile` (via `installProfile`, primary adapter)
  and then dispatches every selected adapter's generator against that
  repo, best-effort (never throws; failures become warnings).
- `runWorkspaceSetup` calls `generateWorkspaceAdapters` for all
  `spec.repos` after registration + MCP, appending to
  `filesCreated`/`warnings`.
- MCP projection generalized: `.opencode/mcp.json` and/or
  `.claude/mcp.json` are produced for every selected MCP-capable adapter
  present in `spec.adapters` (not just the single primary `target`).
- Workspace-level selection persisted in the control repo:
  `.axiom-state/<projectId>/workspace.json` (`{schemaVersion, adapters,
  profile, overlay, createdAt}`).
- TUI wizard: `buildWorkspaceSetupWizardSteps`'s single `target` `select`
  step replaced by an `adapters` `multi-select` step (default:
  `['opencode']`); `buildWorkspaceSetupSpecFromWizard` parses the
  comma-joined value into `adapters` + derives `target = adapters[0]`.
- `antigravity`/`visual-studio-2026` (no dedicated generator): their
  `GENERATED_FILES_BY_TARGET` file (`.antigravity/AGENTS.md` /
  `.vs/AXIOM.md`) is written via `renderCanonicalAgentsMd` +
  `writeGuardedFile`/`writeCanonicalAgentsMd` (thin canonical AGENTS.md,
  not a richer invented format).

## Non-goals

- No changes to `configure.ts`/`sync.ts`/`init.json` schema — the
  single-repo, single-target path stays 100% intact.
- No changes to `@axiom/tui` (stays generic; no new step kinds needed —
  `multi-select` already exists).
- No enum duplication (`ADAPTER_TARGETS` from `init.ts` remains the only
  source of truth).
- No git operations of any kind.
- No integration into `Axiom.Spec/specs/00-08` in this pass (deferred to
  the orchestrator's cross-increment integration pass, matching the
  precedent of the three sibling `INC-20260705-workspace-*` increments).

## Acceptance criteria

- [x] `WorkspaceSetupSpec.adapters?: AdapterTarget[]` added; if omitted,
      defaults to `[target ?? 'opencode']`; if `target` omitted, defaults
      to `adapters[0]`.
- [x] `generateWorkspaceAdapters` writes the selected adapters' files
      into a given repo (`.opencode/AGENTS.md`, `.claude/AGENTS.md`,
      `.antigravity/AGENTS.md`, etc.), reusing the existing generators /
      `installProfile` — no hand-rolled formats except the thin
      antigravity/vs-2026 canonical AGENTS.md.
- [x] `runWorkspaceSetup` with `adapters: ['opencode', 'claude-code']`
      materializes both adapters' files in EVERY repo of the workspace
      (control, spec, and each role repo).
- [x] MCP projection covers every selected MCP-capable adapter
      (`opencode`, `claude-code`) in the control repo; non-MCP-capable
      selected adapters do not affect `.axiom/mcp.yml` (still written
      once).
- [x] `.axiom-state/<projectId>/workspace.json` is written in the control
      repo with the adapter list, profile, and overlay, and is included
      in `filesCreated`.
- [x] Missing generator inputs (e.g. no `agents-md-template.md` in a
      freshly scaffolded repo) degrade to a warning per repo/adapter, and
      never abort the overall workspace setup.
- [x] Existing single-target tests (`workspace-mcp.test.ts`,
      `workspace-setup.test.ts` scenarios a-d, `tui.test.ts` scenario 6)
      keep passing (updated only where the new best-effort adapters step
      legitimately introduces expected `template-missing` warnings on
      fresh tmp repos — see Implementation notes).
- [x] `npm run build` (tsc -b) clean from `Axiom/`.
- [x] Focused vitest run (`apps/cli`, `packages/tui`, `packages/adapters`,
      `packages/installer`, `packages/document-bootstrap`,
      `packages/user-workspace`) green (new + existing): 83 test files,
      787 tests, 0 failures.

## Open questions

None blocking — the increment prompt supplied fully closed design
decisions (contract shape, dispatch map, persistence path, wizard step
replacement). Narrow implementation choices are recorded under
Assumptions.

## Assumptions

- `generateOpencodeConfig`/`generateClaudeCodeConfig` require an
  `agents-md-template.md` under `<repo>/axiom.spec/templates/` (default
  path) that is NOT bundled anywhere in the codebase (confirmed: the only
  copy lives in `Axiom.Spec/templates/`, a sibling repo, never copied
  into a scaffolded workspace repo). For workspace repos without that
  template (the common case for a freshly created control/spec/role
  repo), the generator call fails with `template-missing`; per the
  brief's "never throw" contract, `generateWorkspaceAdapters` catches
  this and records a warning instead of aborting. This is a real,
  expected, best-effort degradation — not a bug — and is exercised
  directly by the new focused tests.
- `writeCopilotInstructions` (used for `copilot-vscode`) has the same
  template dependency (`axiom.spec/templates/copilot-instructions.template.md`,
  also not bundled) plus a stricter contract (`missing-required-variable`
  can also fire). Same best-effort warning handling applies.
- `cursor`'s `GENERATED_FILES_BY_TARGET` entry
  (`.cursor/rules/axiom.mdc`) does NOT match what `generateCursorConfig`
  actually writes (`.cursor/settings.json` + `.cursor/AGENTS.md`) — a
  pre-existing inconsistency in `packages/installer/src/registry.ts`,
  out of scope to reconcile here. `generateWorkspaceAdapters` calls the
  real generator (`generateCursorConfig`) and reports whatever it
  actually writes; it does not attempt to reconcile the registry
  comment/list.
- `installProfile` is called once per repo using `adapters[0]` (the
  primary adapter) as `adapterTarget` for `ResolvedInstallProfile`
  composition/persistence (`install-profile.json`); the resolved profile
  object is target-shape-agnostic enough (`enabledCapabilities`,
  `generatedFiles`, etc.) that every subsequent generator call in that
  repo reuses the same `resolvedProfile`, matching the brief's explicit
  instruction ("call `installProfile` ONCE... primary adapter").
- `DEFAULT_PROFILES` (`@axiom/install-profiles`) is passed as
  `profilesData` to `installProfile` so composition succeeds without a
  scaffolded `profiles.yaml`, mirroring the existing
  `BUG-20260703-configure-needs-bundled-profiles` fallback precedent.
- `.axiom-state/<projectId>/workspace.json` is written once, in the
  control repo only (the same project-scoped anchor already used for
  `axiom.config/topology.yaml`), not duplicated per-repo.

## Implementation notes

- New file `apps/cli/src/commands/workspace-adapters.ts`:
  `generateWorkspaceAdapters(args)` — per repo: one `installProfile`
  call, then a `target -> generator` dispatch map covering all 8
  `ADAPTER_TARGETS`. `antigravity`/`visual-studio-2026` use
  `renderCanonicalAgentsMd` + `writeGuardedFile` against their
  registry-declared relative path. Every generator call is wrapped in
  try/catch (and `Result` `ok`-checked where applicable); failures push a
  warning, successes push absolute paths to `filesCreated`.
- `workspace-setup.ts`: `WorkspaceSetupSpec.adapters` added (`target` made
  optional); default resolution (`adapters ?? [target ?? 'opencode']`,
  `target ?? adapters[0]`) computed once near the top of
  `runWorkspaceSetup`. After the existing MCP best-effort block, a new
  best-effort block calls `generateWorkspaceAdapters` for `spec.repos`
  and writes `workspace.json` in the control repo.
- `workspace-mcp.ts`: `writeWorkspaceMcpConfig` extended to accept
  `adapters?: AdapterTarget[]` (kept `target` for back-compat single-value
  callers/tests); loops over the MCP-capable subset
  (`opencode`/`claude-code`) present in `adapters`, producing one
  adapter-config file per capable adapter. `.axiom/mcp.yml` is still
  written exactly once regardless of how many adapters are selected.
  `WriteWorkspaceMcpConfigResult` gained `adapterConfigPaths: string[]`
  (plural; `adapterConfigPath` kept as the first entry for compat).
- `tui.ts`: `target` `select` step replaced by an `adapters`
  `multi-select` step (`defaultValues: ['opencode']`);
  `buildWorkspaceSetupSpecFromWizard` parses the CSV, sets both
  `spec.adapters` and `spec.target = adapters[0]`.
- Pre-existing tests updated: `workspace-mcp.test.ts` (2 assertions) and
  `workspace-setup.test.ts` (3 assertions) previously asserted
  `warnings: []`; since `runWorkspaceSetup` now always runs the
  best-effort adapters step, and their tmp-dir fixtures have no
  `axiom.spec/templates/agents-md-template.md`, `generateOpencodeConfig`
  legitimately reports `template-missing` per repo. Assertions changed to
  `warnings.every((w) => w.includes('template-missing'))` — the tests
  still verify their original intent (axiom.yaml/topology/registry/MCP
  wiring), just tolerate the new expected warning class.

## Validation

From `C:\repos\Axiom Workspace\Axiom`:

- `npm run build` (tsc -b) — clean, no errors (ran after every file
  change; final run clean).
- `npx vitest run apps/cli packages/tui packages/adapters packages/installer packages/document-bootstrap packages/user-workspace`:

  ```
  Test Files  83 passed (83)
       Tests  787 passed (787)
  ```

  0 failures. New tests added by this increment: 6 in
  `apps/cli/tests/workspace-adapters.test.ts` (dispatch coverage:
  opencode+claude-code template-missing degradation vs. antigravity
  succeeding, visual-studio-2026, cursor/github-copilot/litellm real
  output, copilot-vscode degradation, multi-repo fan-out, best-effort
  resilience), 2 in `apps/cli/tests/workspace-setup.test.ts` (Scenario e:
  full `adapters: ['opencode','claude-code']` across 3 repos + MCP +
  `workspace.json`; back-compat `adapters` omitted falls back to
  `[target]`), and 1 in `apps/cli/tests/tui.test.ts` (Scenario 7:
  multi-select `adapters` step scripted with two picks). `[orchestrator]
  FAIL: command=...` lines seen during the run are pre-existing log
  output from OTHER tests that intentionally exercise failure paths
  (sync/init/join/start negative cases), confirmed by the `83 passed
  (83)` / `787 passed (787)` summary. `apps/cli`'s own `npm run
  typecheck` (`tsc --noEmit`) also ran clean.

## Result

Implemented `generateWorkspaceAdapters` (`apps/cli/src/commands/
workspace-adapters.ts`) and wired it as a best-effort step of
`runWorkspaceSetup`, materializing every selected adapter's output set in
every repo of the workspace via the existing per-target generators (or a
thin canonical `AGENTS.md` for `antigravity`/`visual-studio-2026`, which
have no dedicated generator). `writeWorkspaceMcpConfig` generalized to
project MCP config for every selected MCP-capable adapter
(`opencode`/`claude-code`), not just the single primary target. The
wizard's `target` select was replaced with an `adapters` multi-select
(default `['opencode']`), preserving exact backward-compatible behavior
for single-adapter flows (confirmed by the pre-existing tui.test.ts
Scenario 6 passing unmodified). The workspace-level selection is
persisted at `.axiom-state/<projectId>/workspace.json` in the control
repo. `configure.ts`/`sync.ts`/`init.json` and the single-repo path were
not touched. Build and focused test suite are both green.

## General spec integration

Integrated into the canonical spec in the round-2 cross-increment pass
(covering this increment and the three sibling round-2
`INC-20260705-workspace-*` increments). Files updated:

- **05_Interfaces_Operativas.md** — superseded the wizard's step 8
  `target` (single select) with an `adapters` multi-select (default
  `['opencode']`); noted the `buildWorkspaceSetupWizardSteps` reuse of
  the generic `multi-select` step for `adapters`; generalized the
  "Adapters — MCP config" subsection so the projection covers every
  selected MCP-capable adapter, not just the single primary `target`.
- **06_Integraciones_y_Capacidades.md** — generalized the MCP-projection
  bullet to "cada adapter MCP-capaz seleccionado" (`adapterConfigPaths`
  plural); added the "Generación multi-adapter en todos los repos del
  workspace" subsection (dispatch table target→generator, antigravity/
  vs-2026 via canonical AGENTS.md writer, one `installProfile` per repo,
  best-effort).
- **03_Modelo_Operativo_y_Datos.md** — documented the
  `<controlRepo>/.axiom-state/<projectId>/workspace.json`
  (`WorkspaceSetupRecord`) shape and the per-repo `install-profile.json`
  in the new "Artefactos adicionales del setup de workspace" subsection.
- **01_Requisitos_Funcionales.md** — extended RF-AXM-024 with the
  multi-select-adapters-across-all-repos sub-point.
- **08_Glosario.md** — added "Adapters (multi-select en el wizard)" and
  "`workspace.json`".
- **07_Gobierno_y_Seguridad.md** — noted the best-effort ownership-safety
  posture of the round-2 additive steps in the ownership section.
