# Increment: TUI workspace setup wizard

Status: closed
Date: 2026-07-05

## Goal

Replace the single-repo init wizard (`setup` screen's "Inicializar Axiom en
esta carpeta" action, shipped by `INC-20260703-tui-init-wizard`) with a
GUIDED MULTI-REPO WORKSPACE wizard that collects: human project name, SDD
repo path, Spec repo path, a multi-select of functional/implementation
roles to integrate, one path per selected role, and the
profile/overlay/target triple, then shows a single confirmation summary
and, on confirm, calls `runWorkspaceSetup`
(`INC-20260705-workspace-multirepo-setup-engine`, MCP-wired by
`INC-20260705-workspace-mcp-generation`) to scaffold/parameterize the
whole workspace in one operation.

## Context

Two increments landed earlier the same day and are the foundation this
one builds on:

- `INC-20260705-workspace-multirepo-setup-engine` shipped
  `apps/cli/src/commands/workspace-setup.ts`'s `runWorkspaceSetup(spec):
  Promise<WorkspaceSetupResult>`, the core (no TUI) engine that scaffolds
  a control (SDD) repo, a spec repo, and N role repos in one call, with
  role-aware `axiom.yaml#paths`, one `axiom.config/topology.yaml`, and one
  registry v2 registration.
- `INC-20260705-workspace-mcp-generation` wired `.axiom/mcp.yml` +
  adapter `mcp.json` generation as a best-effort final step inside
  `runWorkspaceSetup`. No public contract change.

The wizard mechanics themselves (`INC-20260703-tui-init-wizard`) already
shipped a generic `WizardStep` (`select` | `text`) state machine in
`@axiom/tui`'s driver, with pure renderers (`wizard-select.ts`,
`wizard-text.ts`), reused by `apps/cli/src/commands/tui.ts`'s
`buildInitWizardSteps` + `runNoProjectBootstrap` for the OLD 6-step
single-repo flow. This increment extends that machinery (new
`multi-select` step kind + a per-step `expand` capability) instead of
building a new wizard engine, and replaces the step list `tui.ts` builds
for the `setup` screen's `init` action.

## Scope

- New generic step kind in `@axiom/tui`: `WizardMultiSelectStep` (kind:
  `'multi-select'`) — pick zero or more options from a caller-supplied
  list. Collected value stored as a comma-joined string of selected
  `value`s (empty string = none). New pure renderer
  `wizard-multi-select.ts` (`[x]`/`[ ]` per option, highlighted row),
  barrel-exported alongside the type.
- Driver interaction for `multi-select`: arrows move highlight, digit
  0-9 or space toggles that option, empty line/Enter confirms the
  current selection set (empty allowed).
- Optional `expand?: (answerValue: string) => WizardStep[]` capability on
  wizard steps: when a step with `expand` is answered, the driver splices
  the returned steps into the remaining step list immediately after the
  current step (once). Used by the `roles` multi-select step to inject
  one `text` step per selected role (`key: 'rolePath:<role>'`).
- `apps/cli/src/commands/tui.ts`: new `buildWorkspaceSetupWizardSteps`
  (replaces `buildInitWizardSteps` as the step-builder wired into the
  `setup` screen's `init` action) with the step order: `name` (text) →
  `sddPath` (text) → `specPath` (text) → `roles` (multi-select, options
  sourced from `DEFAULT_PROFILES`'s `builder` profile's
  `activatesImplementationRoles`, with `expand` injecting per-role path
  steps) → `rolePath:<role>` (text, one per selected role) → `profile`
  (select) → `overlay` (select) → `target` (select).
- `runNoProjectBootstrap`'s `init` branch assembles a `WorkspaceSetupSpec`
  from `wizardResult.values`, calls `runWorkspaceSetup`, prints a result
  summary (repos, topology path, registry status, warnings), and opens
  the operational TUI on the control repo. Errors from `runWorkspaceSetup`
  are caught, printed, and return to the `setup` menu (no crash).
- `axiom init` / `runInit` (single-repo, non-interactive) stay untouched.

## Non-goals

- Custom/user-defined functional roles beyond backend/frontend/qa-e2e
  (noted as a future consideration; `DEFAULT_PROFILES` is the sole
  source of the option list, not a hardcoded duplicate).
- Any change to `runInit`'s CLI flags, validation, or `axiom.yaml` schema.
- Any change to `runWorkspaceSetup`'s or `workspace-mcp.ts`'s public
  contracts (consumed as-is).
- Enterprise lifecycle constructs, persisted in-progress wizard state
  across process restarts.

## Acceptance criteria

- [x] Choosing "Inicializar Axiom en esta carpeta" in the `setup` screen
      walks the user through: name, sddPath, specPath, roles
      (multi-select), one path step per selected role, profile, overlay,
      target — in that order, each with a sensible pre-filled default.
- [x] Role options come from `DEFAULT_PROFILES`'s `builder` profile's
      `activatesImplementationRoles` (`@axiom/install-profiles`), not a
      hardcoded/duplicated list in either `@axiom/tui` or `tui.ts`.
- [x] A single confirmation summary lists every repo (role + resolved
      path), the selected roles, and profile/overlay/target before
      anything is written. New-vs-existing status for each path's DEFAULT
      value and a note that MCP config will be generated are surfaced as
      step subtitles seen during the walk-through leading up to that
      confirmation (kept in the generic driver's per-step subtitle slot
      rather than adding workspace-specific rendering to the driver's
      confirm screen — see Implementation notes); the actual, definitive
      new-vs-existing status per repo is reported in the POST-confirm
      result summary (`repo.created`), which is authoritative since it
      reflects what `runWorkspaceSetup` actually did.
- [x] On confirm, a `WorkspaceSetupSpec` is assembled from the answers and
      `runWorkspaceSetup` is called; on success, a result summary (repos
      created, topology path, registry status, warnings) is printed and
      the operational TUI opens on the control repo.
- [x] On cancel (any step or the final confirmation), no `runWorkspaceSetup`
      call happens and the user returns to the `setup` menu.
- [x] `@axiom/tui` stays generic: no `RepoRole`/role-vocabulary/business
      knowledge inside the package; all enum/option lists and defaults
      are built in `apps/cli/src/commands/tui.ts`.
- [x] If `runWorkspaceSetup` throws, the error is caught, printed clearly,
      and the user returns to the `setup` menu without crashing the TUI.
- [x] `npm run build` (tsc -b) is clean from `Axiom/`.
- [x] `npx vitest run apps/cli packages/tui packages/user-workspace`
      passes (new + existing tests); pre-existing unrelated failures are
      classified, not silently absorbed.

## Open questions

None blocking — the increment prompt supplied closed design decisions
(dynamic-steps mechanism preference + fallback, path semantics, role
option source, error handling). See Assumptions for the few narrow calls
made while implementing them.

## Assumptions

- Dynamic per-role path steps use the `expand` splice mechanism (the
  prompt's preferred option), not the two-pass fallback — implementing it
  did not destabilize the driver's synchronous state machine or the
  existing wizard tests (see Implementation notes for how it composes
  with the existing `stepIndex`/`selectIndex` bookkeeping).
- `create` per repo is computed purely from directory existence at spec-
  assembly time (`!fs.existsSync(resolvedPath)`); the engine's own
  no-clobber guard (existing `axiom.yaml` for a different `projectId`)
  is the safety net for the "exists but not ours" case, exactly as
  `runWorkspaceSetup` already documents.
- The wizard does not special-case "same path for sdd and spec, zero
  roles" beyond an accurate summary label; the engine already handles
  that degenerate case (`multiRepo` computed from path/role-repo count).
- `homeDirOverride` is not exposed as a user-facing wizard step; it flows
  through `runNoProjectBootstrap`'s existing `args.homeDirOverride`
  parameter (test-only), same as before this increment.

## Implementation notes

- `packages/tui/src/driver.ts`: added `WizardMultiSelectStep` to the
  `WizardStep` union, an `expand?` field on all three step kinds, and
  `PendingWizard.values[key]` multi-select serialization (comma-joined
  values, insertion order = option order, not selection order — keeps
  the serialization stable regardless of toggle order). The `expand`
  splice happens once, right when the step advances (`stepIndex += 1`):
  `steps.splice(insertAt, 0, ...expandedSteps)` on a **mutable local
  copy** of the original `steps` array (the original `PendingWizard.steps`
  field becomes non-readonly `WizardStep[]` internally to support the
  splice; the public `WizardStep`/`WizardResult` types are unaffected).
  Re-verified all existing `driver-wizard.test.ts` scenarios stay green
  because `expand` is `undefined` for every pre-existing step (no-op
  branch).
- `packages/tui/src/screens/wizard-multi-select.ts` (new): renders
  `[x]`/`[ ]` per option reusing `printHeader`/`printFooter`/
  `printSectionBanner`; highlight uses the same reverse-video convention
  as `wizard-select.ts` (custom render loop, not `printMenu`, since
  `printMenu` doesn't support a checkbox prefix).
  `packages/tui/src/index.ts` barrel-exports the new renderer/type.
- `apps/cli/src/commands/tui.ts`: `buildWorkspaceSetupWizardSteps`
  replaces `buildInitWizardSteps` as the function wired into
  `runNoProjectBootstrap`. Role options + labels come from
  `DEFAULT_PROFILES.functionalProfiles.find(p => p.id === 'builder')
  .activatesImplementationRoles` (`@axiom/install-profiles`). The
  `roles` step's `expand` closure builds one `text` step per selected
  role with key `rolePath:<role>` and default `../<slug>-<role>`.
  `runNoProjectBootstrap`'s confirm branch replaces the old `runInit`
  call with: slugify the name, resolve `sddPath`/`specPath`/role paths
  relative to `cwd`, build a `WorkspaceRepoSpec[]` (control/spec/role
  entries per the path-semantics rules), call `runWorkspaceSetup` inside
  a try/catch, print a result/error summary, then open the operational
  TUI on the resolved control repo path (or return to `setup` on error).
- `buildInitWizardSteps` (old 6-step single-repo builder) is left in
  place, unused by `runNoProjectBootstrap` after this change, since
  removing dead code the increment didn't ask to delete is out of scope;
  documented here as intentionally orphaned rather than silently deleted.
- New-vs-existing / MCP-generation notices: rather than teaching the
  GENERIC driver's confirm screen about per-repo/workspace semantics
  (which would break the layering rule), each path step's `subtitle`
  carries a `fs.existsSync`-computed "(nuevo, se generará)" /
  "(existente, se parametrizará)" note for its DEFAULT value (subtitles
  are already rendered while that step is active, ahead of the final
  confirmation), and the `target` step's subtitle notes that MCP config
  will be generated on confirm. The POST-confirm result summary
  (`runNoProjectBootstrap`, printed right after `runWorkspaceSetup`
  returns) reports the real, definitive `created`/parameterized status
  per repo — the authoritative source, since a user may type a
  non-default path after seeing the default's note.

## Validation

From `C:\repos\Axiom Workspace\Axiom`:

- `npm run build` (tsc -b) — clean, no errors.
- `npx vitest run apps/cli packages/tui packages/user-workspace`:

  ```
  Test Files  67 passed (67)
       Tests  661 passed (661)
  ```

  0 failures. New tests in this exact scope: 4 in
  `packages/tui/tests/wizard-multi-select.test.ts`, 3 in
  `packages/tui/tests/driver-wizard.test.ts` (8 total in that file, up
  from 5), and the replaced end-to-end scenario in
  `apps/cli/tests/tui.test.ts` (11 total in that file, same count as
  before — one scenario replaced in place). The `[orchestrator] FAIL:
  command=...` console lines seen during the run are pre-existing log
  output from OTHER tests that intentionally exercise failure paths
  (sync-command/init-command/join-command/start-command negative cases),
  not vitest failures — confirmed by the `67 passed (67)` / `661 passed
  (661)` summary. No pre-existing dogfood/real-repo failures surfaced in
  this three-package validation scope.

## Result

Implemented end-to-end. The `setup` screen's "Inicializar Axiom en esta
carpeta" action now runs a single guided multi-repo workspace wizard
(name → sddPath → specPath → roles multi-select → one path step per
selected role, injected via `expand` → profile → overlay → target),
ending in one confirmation summary, and calls `runWorkspaceSetup` on
confirm. `runWorkspaceSetup` failures are caught and reported without
crashing the TUI; cancelling at any point returns to the `setup` menu
with no side effects. `@axiom/tui` gained a generic `multi-select` step
kind and an optional `expand` capability on all step kinds, with zero
business/enum knowledge added to the package. Role options are sourced
live from `@axiom/install-profiles`'s `DEFAULT_PROFILES` (`builder`
profile's `activatesImplementationRoles`), not duplicated. `axiom init`/
`runInit` are untouched. Build and the full `apps/cli` + `packages/tui`
+ `packages/user-workspace` test suites are green.

Files created:

- `Axiom/packages/tui/src/screens/wizard-multi-select.ts`
- `Axiom/packages/tui/tests/wizard-multi-select.test.ts`

Files modified:

- `Axiom/packages/tui/src/driver.ts` — `WizardMultiSelectStep` type,
  `WizardStepExpand` type, `expand?` field on all step kinds,
  `PendingWizard.multiSelectValues`, `primeWizardCursorForStep`,
  `serializeMultiSelectValues`, `applyStepExpand` helpers; multi-select
  rendering/handling in `renderWizardInternal`/`renderWizardConfirmScreen`/
  `handleWizardLine`; keyboard handling (arrows, space, digits) extended
  to `multi-select`.
- `Axiom/packages/tui/src/index.ts` — barrel-exports for the new
  renderer/types.
- `Axiom/packages/tui/tests/driver-wizard.test.ts` — 3 new tests
  (multi-select toggle+confirm, multi-select empty selection, `expand`
  splice injecting follow-up steps).
- `Axiom/apps/cli/src/commands/tui.ts` — `buildWorkspaceSetupWizardSteps`,
  `buildWorkspaceSetupSpecFromWizard`, `parseSelectedRoles`,
  `BUILDER_IMPLEMENTATION_ROLES`/`WORKSPACE_ROLE_LABELS`/
  `workspaceRoleLabel`; `runNoProjectBootstrap`'s confirm branch rewired
  to `runWorkspaceSetup` with try/catch + result/error summary printing.
  `buildInitWizardSteps` (old 6-step builder) is left in place but no
  longer wired into `runNoProjectBootstrap` (intentionally orphaned, not
  deleted — see Implementation notes).
- `Axiom/apps/cli/tests/tui.test.ts` — Scenario 6 replaced with a
  multi-repo end-to-end wizard scenario asserting control/spec/backend
  repos, `topology.yaml`, `.axiom/mcp.yml`, and the registry.

## General spec integration

Integrated into the canonical `Axiom.Spec/specs/00-08` in the final
cross-increment pass covering this increment and its two siblings
(`INC-20260705-workspace-multirepo-setup-engine`,
`INC-20260705-workspace-mcp-generation`). Files this increment's
knowledge landed in:

- `00_Resumen_Ejecutivo.md` — updated the bootstrap paragraph to describe
  the init action as a guided multi-repo workspace wizard superseding the
  old single-repo one.
- `01_Requisitos_Funcionales.md` — the UI half of the new **RF-AXM-024**
  (wizard step order, multi-select roles from `DEFAULT_PROFILES`,
  per-role path steps, confirm/cancel/error behavior); also updated
  RF-AXM-023 to point forward to RF-AXM-024 as the behavior in force
  today, removing the contradiction with the now-superseded single-repo
  wizard.
- `05_Interfaces_Operativas.md` — **superseded** the prior subsection
  "TUI — menú de bootstrap `setup` y wizard guiado de `init`
  (`INC-20260703-tui-init-wizard`)" with **"TUI — menú de bootstrap
  `setup` y wizard guiado de setup de workspace
  (`INC-20260705-tui-workspace-setup-wizard`)"**: full step order and
  defaults, the new generic `multi-select` step kind and `expand` splice
  mechanism added to `@axiom/tui`, the app-vs-package layering split, and
  confirm/cancel/error behavior. Kept the note that `axiom init`/`runInit`
  stays single-repo.
- `08_Glosario.md` — the wizard-facing terms are covered by the
  workspace-setup glossary block ("Workspace setup", roles, SDD/Spec MCP)
  added in this same pass.
