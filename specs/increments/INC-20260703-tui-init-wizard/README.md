# Increment: TUI init wizard (Etapa 2 of the init redesign)

Status: closed
Date: 2026-07-03

## Goal

Replace the current no-question `axiom` bootstrap ("Inicializar Axiom en esta
carpeta" → `runInit` with `slugify(folder)` as the only input) with a GUIDED
WIZARD, inside `axiom tui`'s `setup` screen, that asks the user for each
init field in order, pre-filling a default per field (inferred where
possible) that the user can accept or override, then calls `runInit` with
all fields explicit and opens the operational TUI on the new project.

This is Etapa 2 of the init redesign. Etapa 1 (`INC-20260703-init-enums-and-role`,
already closed) added the canonical enums (`REPO_ROLES`, `PROJECT_LAYOUTS`,
`FUNCTIONAL_PROFILES`, `OPERATIONAL_OVERLAYS`, `ADAPTER_TARGETS`) and the
`role` field to `runInit`/`InitArgs`/`InitResult` in
`apps/cli/src/commands/init.ts`. This increment builds on that; it does not
redo it.

## Context

Today, `packages/tui/src/driver.ts`'s `setup` screen (shown by
`apps/cli/src/commands/tui.ts`'s `runNoProjectBootstrap` when `axiom tui`/
`axiom` opens in a folder with no Axiom project) only offers a fixed menu
(`SETUP_ITEMS`: init / self-update / view registered projects / exit).
Choosing "Inicializar Axiom en esta carpeta" calls `runInit` directly with
`name: slugifyProjectName(basename(cwd))` and every other field left to its
default — no questions asked.

`@axiom/tui` (`packages/tui`) is a package; `apps/cli` is an app. The tui
package must not import from `apps/cli` (layering rule already enforced
elsewhere in this codebase, e.g. `@axiom/cli-commands` exists specifically
as the seam so the TUI never imports `apps/cli/src/...` directly). The
wizard's selectable option values (`RepoRole`, `ProjectLayout`,
`FUNCTIONAL_PROFILES`, `OPERATIONAL_OVERLAYS`, `ADAPTER_TARGETS`) are
exported as `as const` arrays from `apps/cli/src/commands/init.ts` (app
layer) — they must not be duplicated inside `@axiom/tui`.

`packages/tui/src/driver.ts` (~1900 lines) has selection-menu screens only
(`menu`, `setup`, `projects`, `model-list`); confirm/summary/preview are
internal driver states (not `ScreenId`s). There is no free-text input
screen.

## Scope

- Add two GENERIC, reusable screens to `@axiom/tui` (package layer, no
  enum/business knowledge inside them):
  - a "select one option from a caller-supplied list" screen
    (`wizard-select`), reusing `printMenu`/`printHeader`/`printFooter`/
    `printSectionBanner` for visual consistency with the rest of the TUI;
  - a "free-text input" screen (`wizard-text`), consistent styling, with a
    pre-filled default the user can accept (Enter) or overwrite (type +
    Enter).
- Add a driver-level "wizard" internal state (mirrors the existing
  `pendingConfirmFlow`/`pendingSummary` pattern: NOT a new router
  `ScreenId`, an internal state machine over an ordered list of steps
  passed in via `RunTuiArgs`) that walks a caller-supplied ordered list of
  steps (`select` | `text`), collects answers, then shows a confirmation
  summary of all choices, and on confirm resolves `runTui` with the
  collected `wizardResult` (on cancel, returns to the `setup` menu).
- Wire `apps/cli/src/commands/tui.ts`'s `runNoProjectBootstrap` (`init`
  action) to launch the wizard instead of calling `runInit` directly: it
  supplies the step list + option arrays (imported from `init.ts`'s
  exported enums) + inferred defaults, receives the `wizardResult`, and
  calls `runInit` with every field explicit
  (`name`/`role`/`layout`/`profile`/`overlay`/`target`).
- Wizard steps, in order: `name` (free text, default = folder basename),
  `role` (select, default `sdd`), `layout` (select, default =
  `inferInitLayout`), `profile` (select, default `builder`), `overlay`
  (select, default `local-only`), `target` (select, default `opencode`),
  then a confirmation summary of all six choices.
- On confirm: call `runInit` with all fields explicit, then open the
  operational TUI on the new project (same behavior `runNoProjectBootstrap`
  already has for the non-wizard path).
- Focused tests for the new screens (`wizard-select`, `wizard-text`) and
  the new driver flow, mirroring `packages/tui/tests/driver.test.ts`'s
  harness (`FakeReadline.emitLine`, `makeTtyInput`, `waitForOutput`), plus
  an `apps/cli` test exercising the wizard end-to-end through
  `runNoProjectBootstrap`.

## Non-goals

- No folder/path renames (`.sdd` → `.axiom-state`, `axiom.spec/config` →
  `axiom.config`, etc.) — explicitly out of scope, deferred to a separate
  increment.
- No changes to `runInit`'s validation, `axiom.yaml` schema, or the
  registry — Etapa 1 already delivered the enums/role field this wizard
  consumes.
- No enterprise lifecycle constructs, no speculative multi-step undo/redo,
  no persisted wizard-in-progress state across process restarts.
- No change to `axiom init`'s non-interactive CLI flags (`--name`,
  `--role`, etc.) — the wizard is additive UI, not a replacement for
  scriptable `axiom init`.

## Acceptance criteria

- [x] `axiom` (or `axiom tui`) opened in a folder with no Axiom project,
      choosing "Inicializar Axiom en esta carpeta", walks the user through
      all 6 fields in order (`name`, `role`, `layout`, `profile`,
      `overlay`, `target`), each with a pre-selected/pre-filled default the
      user can accept with Enter or change.
- [x] Defaults match the spec: `name` = folder basename (free text, not
      slugified — the wizard's `name` step is the human-readable name;
      `runInit`/`buildAxiomYaml` derive the `projectId`/`repoId` slug
      internally via `slugifyProjectId`); `role` = `sdd`; `layout` =
      `inferInitLayout(cwd)`; `profile` = `builder`; `overlay` =
      `local-only`; `target` = `opencode`.
- [x] After the last step, a confirmation summary lists all 6 chosen
      values before anything is written.
- [x] On confirm, `runInit` is called with all 6 fields explicit (not
      relying on `runInit`'s own internal defaults) and the operational
      TUI opens on the freshly initialized project.
- [x] On cancel (from the confirmation step), no `runInit` call happens
      and the user returns to the `setup` menu.
- [x] `@axiom/tui` (package) does not import from `apps/cli` (app), and
      does not hardcode/duplicate the enum value lists — they flow in from
      `apps/cli/src/commands/tui.ts` via `RunTuiArgs`.
- [x] `npm run build` stays green from `Axiom/`.
- [x] `npx vitest run apps/cli packages/tui` passes, including new
      focused tests for the wizard screens/flow.

## Open questions

None blocking. Two narrow scope calls were made explicitly (see
Assumptions) rather than left open, to keep the increment shippable in one
pass per the "prioritize a working end-to-end flow" instruction.

## Assumptions

- The wizard is scoped to the `setup` (no-project bootstrap) screen's
  `init` action only. `axiom init` (the non-interactive CLI command) and
  `axiom repo attach`'s own flows are untouched.
- `name` free-text step accepts the raw human name (e.g. "SGL Workspace");
  `runInit`'s existing `validateProjectName`/`slugifyProjectId` machinery
  already derives a valid `projectId` from it (Etapa 1 territory,
  unchanged). If the user's typed name normalizes to an invalid
  `projectName` for `runInit`'s legacy `PROJECT_NAME_REGEX` path (the
  `args.name` passed to `runInit` must still satisfy
  `PROJECT_NAME_REGEX`), the wizard passes the slugified form (same
  `slugifyProjectName` helper `tui.ts` already exports) as `runInit`'s
  `name`, and shows the original human text only in the confirmation
  summary label — avoiding a second, undocumented validation surface
  inside `@axiom/tui`.
- The generic `wizard-select`/`wizard-text` screens are internal driver
  state (like `confirm`/`summary`/`preview`), not new `ScreenId` union
  members — consistent with the existing pattern for multi-step
  interactions that don't need router stack tracking.

## Implementation notes

Layering approach chosen: **(A)** from the brief — `@axiom/tui` provides
generic reusable screens; `apps/cli/src/commands/tui.ts` (which can import
`init.ts`'s enums) drives the wizard sequence.

Files touched:

- `packages/tui/src/screens/wizard-select.ts` (new) — generic "select one
  option from a caller-supplied list" screen. Reuses `printHeader`/
  `printFooter`/`printMenu`/`printSectionBanner`. No enum/business
  knowledge; options come in as `{ value, label, hint? }[]`.
- `packages/tui/src/screens/wizard-text.ts` (new) — generic free-text
  input screen with a pre-filled default shown as `[default]`. This is
  the first free-text screen in `@axiom/tui`; previously only
  selection-menu screens existed.
- `packages/tui/src/driver.ts` — added public types (`WizardStep` union of
  `WizardSelectStep`/`WizardTextStep`, `WizardResult`), `RunTuiArgs.
  wizardSteps`, `RunTuiResult.wizardResult`, and an internal `pendingWizard`
  driver state (same pattern as `pendingConfirmFlow`/`pendingSummary`/
  `pendingPreview` — NOT a router `ScreenId`). The `setup` screen's `init`
  item now starts the wizard when `wizardSteps` is provided (falls back to
  the old `selectedSetupAction: 'init'` propagation when it is not, so
  other callers of the driver are unaffected). A `handleWizardLine`
  function walks steps sequentially (`text` step: any non-empty line
  replaces the default, empty accepts it; `select` step: digits 0-9 jump
  to that option, arrow keys move the highlighted option, empty confirms
  the highlighted one), then renders a confirmation summary
  (`renderWizardConfirmScreen`) before resolving. `b`/`atras`/`volver`/
  `salir` cancel the wizard from any step and return to `setup`, EXCEPT
  that `q`/`b` typed as the first character of free text in a `text` step
  are treated as literal text, not commands (checked via keypress-handler
  guard), since a project name could start with those letters.
- `packages/tui/src/index.ts` — barrel-exports the two new screens/types
  and `WizardStep`/`WizardSelectStep`/`WizardTextStep`/`WizardResult`.
- `apps/cli/src/commands/init.ts` — exported `inferInitLayout` (was
  private) so the wizard can use it as the `layout` step's default.
- `apps/cli/src/commands/tui.ts` — added `buildInitWizardSteps(folder,
  layoutDefault)`, which builds the 6-step list with option labels sourced
  from `init.ts`'s `REPO_ROLES`/`PROJECT_LAYOUTS`/`FUNCTIONAL_PROFILES`/
  `OPERATIONAL_OVERLAYS`/`ADAPTER_TARGETS`. `runNoProjectBootstrap`'s `init`
  branch now passes `wizardSteps` into `runTuiDriver` and, on
  `result.wizardResult.confirmed === true`, normalizes the free-text
  `name` with the existing `slugifyProjectName` (kept for `runInit`'s
  `PROJECT_NAME_REGEX` requirement) and calls `runInit` with all 6 fields
  explicit. On `confirmed === false`, it loops back to `setup` (existing
  bounded-reentry loop, unchanged).

Tests added:

- `packages/tui/tests/wizard-screens.test.ts` (5 tests) — pure renderer
  tests for `renderWizardSelectScreen`/`renderWizardTextScreen` (title,
  subtitle, step label, default value, selection highlight, last message).
- `packages/tui/tests/driver-wizard.test.ts` (5 tests) — end-to-end driver
  tests using the same harness as `driver.test.ts`
  (`FakeReadline.emitLine`, `makeTtyInput`, `waitForOutput`): accept all
  defaults + confirm; override the free-text value and pick a non-default
  option + confirm; cancel at the final confirmation (`n`); cancel from a
  mid-wizard step (`b`); and a regression check that omitting
  `wizardSteps` preserves the pre-wizard `selectedSetupAction` behavior.
- `apps/cli/tests/tui.test.ts` — one new end-to-end scenario
  (`runTui — wizard de init end-to-end`) that scripts all 6 wizard answers
  plus confirmation through the real CLI wrapper (`runTui` in
  `apps/cli/src/commands/tui.ts`), then asserts the written `axiom.yaml`
  contains the wizard-chosen `role` and `name`.

Gotcha found and fixed during implementation (documented here for future
maintainers): because the wizard's steps are 100% synchronous (no flow
runner `await`s in between, unlike `configure`/`sync`/etc.), two
`emitLine` calls fired back-to-back in a test can race the driver's
`busy` mutex — the `finally { busy = false }` of the previous line's
handler may not have run yet when the next `emitLine` arrives, so the
second line gets silently dropped by the re-entrancy guard. This is the
same class of issue `driver.test.ts` already works around with
`await new Promise((resolve) => setImmediate(resolve))` between rapid
`emitLine` calls; the new wizard tests use the identical idiom (a local
`flush()` helper). Also found and fixed a real driver ordering bug during
this work: the wizard's internal state must be checked and handled
*before* the router-stack-based `setup` branch, because the wizard does
not push anything onto the router stack (same as `confirm`/`preview`), so
`state.stack` still reports `setup` on top while the wizard is active —
without the earlier check, every wizard input line was being
mis-handled as "Enter/digit on the `setup` menu."

## Validation

`npm run build` (tsc -b) from `Axiom/Axiom` — green, no errors.

`npx vitest run apps/cli packages/tui` — 547 passed, 1 failed (58 files, 1
failed file), verbatim tail:

```
 Test Files  1 failed | 57 passed (58)
      Tests  1 failed | 547 passed (548)
```

The 1 failure (`apps/cli/tests/start.test.ts > axiom start > Scenario 4:
start sin install-profile.json lo deriva de init.json`, `ENOENT ... axiom.
spec/config/profiles.yaml`) is pre-existing and unrelated: reproduced
identically on a clean `git stash` of this increment's changes (same
error, same line, same test). It depends on a file
(`axiom.spec/config/profiles.yaml`) that does not exist in this workspace's
own dogfooding `Axiom/` repo root — an environment gap, not a regression
introduced here.

## Result

Implemented end-to-end. All 6 fields are asked in order with pre-selected
defaults; a confirmation summary lists all choices before anything is
written; confirming calls `runInit` with every field explicit and opens
the operational TUI on the new project; cancelling (from any step or the
final confirmation) returns to the `setup` menu with no side effects.
Layering rule respected: `@axiom/tui` has no knowledge of `RepoRole`/
`ProjectLayout`/etc. — it only renders whatever step list it receives.
15 new tests added across 3 files; build and existing test suites stay
green (aside from the one pre-existing, unrelated failure noted above).

## General spec integration

- `Axiom.Spec/specs/05_Interfaces_Operativas.md` — added a new subsection
  "TUI — wizard guiado de `init` en la screen `setup`
  (`INC-20260703-tui-init-wizard`)" describing the wizard's steps, defaults,
  confirm/cancel behavior, and the layering split between `@axiom/tui`'s
  generic screens and `apps/cli`'s enum-aware step builder.
- `Axiom.Spec/specs/03_Modelo_Operativo_y_Datos.md` — added one paragraph
  under the `axiom.yaml schemaVersion: 2` section cross-referencing that
  `role`/`layout`/profile-triple enums are now also the source for the
  TUI wizard, pointing to the new subsection in `05_Interfaces_
  Operativas.md` for the UI-level description (kept the data-model file
  focused on data, not UI flow, to avoid duplicating content across
  files).
