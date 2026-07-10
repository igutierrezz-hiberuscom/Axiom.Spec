# Increment: Workspace wizard UX fixes + custom roles

Status: closed
Date: 2026-07-07

## Goal

Fix four reported UX/behavior issues in the TUI workspace-setup wizard
(`axiom tui` → "Inicializar Axiom en esta carpeta"):

1. Text-step answers give no visible confirmation of what was captured.
2. Path defaults (`sddPath`/`specPath`/role paths) are not clearly
   sibling-style and the "type an absolute path" escape hatch is
   undocumented in the UI.
3. Functional roles are limited to a fixed backend/frontend/qa-e2e
   multi-select; the user needs arbitrary role names, any count.
4. `profile`/`overlay` select steps don't explain what they actually do.

## Context

Wizard code lives in `packages/tui/src/driver.ts` (generic wizard state
machine + step kinds `text`/`select`/`multi-select`, with an `expand`
closure hook), `packages/tui/src/screens/wizard-{text,select,multi-
select}.ts` (pure renderers), and `apps/cli/src/commands/tui.ts`
(`buildWorkspaceSetupWizardSteps` / `buildWorkspaceSetupSpecFromWizard`,
which own all business enums per the layering rule — `@axiom/tui` stays
business-knowledge-free).

Prior increment `INC-20260705-tui-workspace-setup-wizard` built the
current flow: name → sddPath → specPath → roles (multi-select of
`DEFAULT_PROFILES` builder `activatesImplementationRoles`, `expand` to
per-role path steps) → profile → overlay → adapters (multi-select,
sibling increment `INC-20260705-workspace-adapters-multiselect`,
untouched here).

## Scope

- `@axiom/tui` (`packages/tui/src`): generic "last captured value" echo
  in the wizard driver + all three step renderers (`text`, `select`,
  `multi-select`). No business knowledge added to the package.
- `apps/cli/src/commands/tui.ts`: sibling-style path defaults derived
  from the `name` answer (fallback to folder-basename slug before
  `name` is known); updated subtitles documenting the absolute-path
  escape hatch; `roles` step converted from `multi-select` to `text`
  (comma-separated, arbitrary names) with an `expand` that injects one
  path step per parsed role; `buildWorkspaceSetupSpecFromWizard` parses
  and sanitizes arbitrary role names into `WorkspaceRepoSpec`; clearer
  `profile`/`overlay` subtitles.
- Tests: `packages/tui/tests/driver-wizard.test.ts`,
  `wizard-screens.test.ts`, `wizard-multi-select.test.ts` updated for the
  echo; `apps/cli/tests/tui.test.ts` updated end-to-end scenario +
  focused unit coverage for arbitrary-role parsing and absolute paths.

## Non-goals

- No engine changes in `apps/cli/src/commands/workspace-setup.ts` —
  `roleKey`/`functionalRoleId` are already open `string` fields; verified,
  not modified.
- No changes to `runInit`/`axiom init` (single-repo wizard,
  `buildInitWizardSteps`).
- No change to the `adapters` multi-select step (sibling increment).
- No removal of the generic `multi-select` step kind from `@axiom/tui`
  (still used by `adapters` and available for other flows).
- No `Axiom.Spec/specs/00-08` integration (orchestrator does the final
  cross-increment pass, per instruction).

## Acceptance criteria

- [x] After answering a `text` step (and, generically, any step kind),
      the wizard renders a captured-value confirmation before/while
      showing the next step, generically in `@axiom/tui` (not hardcoded
      per field).
- [x] `sddPath`/`specPath`/role-path steps default to sibling-style
      paths derived from the slug of the entered (or so-far-known)
      project name (`../<slug>-sdd`, `../<slug>-spec`,
      `../<slug>-<role>`), and their subtitles state the path can be
      relative to the current folder OR absolute.
- [x] `path.resolve(cwd, <absolute path>)` returns the absolute path
      unchanged on Windows and this is exercised by a test.
- [x] The `roles` step accepts an arbitrary comma-separated list of role
      names (any count, including zero), and its `expand` injects one
      path step per parsed role.
- [x] `buildWorkspaceSetupSpecFromWizard` parses arbitrary role names
      into `WorkspaceRepoSpec` entries with sanitized `roleKey`/
      `functionalRoleId`, and backend/frontend/qa-e2e keep working
      exactly as before.
- [x] `profile` and `overlay` step subtitles concisely explain their
      real effect per the codebase (`code.*` gating, implementation
      roles; local-only/standard/enterprise gateway + governance/
      compliance semantics — worded from actual `default-profiles.ts`
      fields, not an invented "telemetry" feature).
- [x] `npm run build` clean from `Axiom/`.
- [x] `npx vitest run apps/cli packages/tui packages/user-workspace` —
      all relevant tests pass (pre-existing unrelated failures
      classified, not silently ignored).

## Open questions

None blocking — the task brief resolved the open design points up
front. Narrow implementation choices are recorded under Assumptions.

## Assumptions

- "Generic" text-feedback is implemented as a single `lastCaptured:
  {title, value} | null` slot on the driver's internal `PendingWizard`
  state, rendered as a `✓ <title>: <valor>` line by all three step
  renderers (right after the header, before the step banner) when
  present. This keeps the feature step-kind-agnostic and adds no
  business knowledge to `@axiom/tui` (it only echoes back whatever
  `title`/serialized value the caller already supplied).
- For `select`, the echoed value is the option's `label` (human
  readable); for `multi-select`, the comma-joined labels (or "(ninguno)"
  when empty); for `text`, the raw captured string.
- Sibling-style defaults for `sddPath`/`specPath` use the `name` step's
  answer when the wizard has already advanced past it (steps are built
  once per wizard invocation, before `name` is known, so the FIRST
  render of `sddPath`/`specPath` necessarily uses the folder-basename
  slug — the wizard step list is static, built by
  `buildWorkspaceSetupWizardSteps` before the driver runs). This is a
  pre-existing structural constraint (steps are plain data, not
  re-evaluated against later answers) and is called out explicitly
  rather than silently worked around; it matches the existing
  `existingNote` pattern which already has this same limitation for
  role paths (computed against the folder slug, not the name answer).
- Role name sanitization for arbitrary roles reuses the same
  lowercase/collapse-to-hyphen algorithm as `slugifyProjectName`
  (extracted as `sanitizeRoleName`), so "Data Team!" → "data-team",
  consistent with how `slugifyProjectName` already normalizes the
  project name.
- Duplicate role names (after trim + sanitize) are de-duplicated
  preserving first-occurrence order; empty entries (from consecutive
  commas or leading/trailing commas) are dropped.
- Backend/frontend/qa-e2e typed by the user sanitize to themselves
  (already lowercase, hyphen-only), so existing behavior/paths/labels
  for those three names are unchanged.

## Implementation notes

- `packages/tui/src/driver.ts` (generic, no business knowledge added):
  - `PendingWizard` gained `lastCaptured: { title: string; value: string }
    | null`, set every time a `text`/`select`/`multi-select` step is
    answered (all three "advance step" branches in `handleWizardLine`),
    using the step's own `title` and a new `describeWizardAnswer(step,
    rawValue)` helper (select → chosen option's `label`; multi-select →
    comma-joined labels or `'(ninguno)'`; text → raw value).
  - `renderWizardInternal` builds `capturedNote = '✓ <title>: <valor>'`
    from `lastCaptured` (or `undefined` on the wizard's first step) and
    passes it to all three renderers via a new optional `capturedNote`
    arg.
  - `packages/tui/src/screens/wizard-{text,select,multi-select}.ts`: each
    renderer prints `capturedNote` (when present) right after
    `printHeader`, before the step's banner.
  - The final confirmation screen (`renderWizardConfirmScreen`) was left
    unchanged — it already lists every answer, which independently
    satisfies "make it obvious what was recorded" at that point.
- `apps/cli/src/commands/tui.ts`:
  - Added `sanitizeRoleName` (exported) — same lowercase/collapse-to-
    hyphen algorithm as `slugifyProjectName`, without the
    `"axiom-project"` fallback (empty results are simply dropped by the
    caller).
  - Added `parseRolesInput` (exported) — splits on comma, trims, drops
    empties, sanitizes, and de-duplicates preserving first-occurrence
    order. `''`/`undefined` → `[]`.
  - `buildWorkspaceSetupWizardSteps`: removed `BUILDER_IMPLEMENTATION_ROLES`
    / `WORKSPACE_ROLE_LABELS` / `workspaceRoleLabel` and the `roles`
    multi-select entirely. Replaced with a `text` step (`key: 'roles'`,
    default `''`) whose subtitle suggests backend/frontend/qa-e2e as
    examples and states "dejalo vacío para ninguno"; its `expand`
    delegates to `parseRolesInput` and injects one `rolePath:<role>` text
    step per parsed role — mechanically unchanged from the prior
    `expand`-splice mechanism, just driven by free text instead of a
    multi-select serialization.
  - `sddPath`/`specPath` step defaults changed from `'.'` /
    `` `../${slug}.spec` `` to sibling-style `` `../${folderSlug}-sdd` ``
    / `` `../${folderSlug}-spec` ``, where `folderSlug =
    slugifyProjectName(humanNameDefault)`. Both steps' subtitles now
    state `"Relativa a esta carpeta o una ruta absoluta (e.g.
    C:\repos\Foo\Bar)."` — verified `path.resolve(cwd, absolutePath)`
    returns the absolute path unchanged (native `path.resolve`
    semantics; exercised by the rewritten end-to-end test using a real
    Windows tmp dir as the typed `sddPath` answer).
  - Documented explicitly (in the function's doc comment) the
    pre-existing structural limitation carried over unchanged: the step
    list is static data built once, before the wizard runs, so
    `sddPath`/`specPath` defaults use the folder-basename slug
    (`humanNameDefault`), not the `name` step's actual answer (which is
    only known once the wizard is already running). The `roles` step's
    per-role path defaults DO use the live `name` answer, because
    `expand` is a closure invoked at confirm-time, after `name` (and
    `roles` itself) are already in `pendingWizard.values`.
  - `buildWorkspaceSetupSpecFromWizard`: `parseSelectedRoles` now
    delegates to `parseRolesInput` (kept as a thin wrapper for minimal
    diff / call-site stability); default `sddPath`/`specPath` fallbacks
    updated to match the new sibling scheme; each parsed role is used
    verbatim (already sanitized) as `roleKey`/`functionalRoleId`.
    Verified: `apps/cli/src/commands/workspace-setup.ts`'s
    `WorkspaceRepoSpec.roleKey`/`functionalRoleId` are plain `string`
    (no enum), and `runWorkspaceSetup` never calls `validateTopology`
    (that only runs from the read-only `topology` TUI screen, where an
    undeclared role would surface as a non-blocking `unknown-role`
    finding) — so arbitrary role names flow through the engine
    unmodified, exactly as the increment brief anticipated.
  - `profile`/`overlay` steps in `buildWorkspaceSetupWizardSteps` gained
    `subtitle` text (the shared `PROFILE_LABELS`/`OVERLAY_LABELS` used by
    both this wizard and `buildInitWizardSteps` were left untouched, per
    the non-goal of not touching `runInit`): profile subtitle states
    builder enables `code.*` + implementation roles vs. product-owner
    limited to spec/SDD with `code.*` blocked (verified against
    `packages/install-profiles/src/default-profiles.ts`:
    `activatesImplementationRoles: []` + no `code.*` capability for
    product-owner vs. `code.semanticNavigation`/etc. + `['backend',
    'frontend', 'qa-e2e']` for builder). Overlay subtitle states
    local-only = no gateway (recommended for dev), standard = gateway
    optional, enterprise = gateway required for the primary path +
    expanded governance/compliance (verified against the same file's
    `operationalOverlays[].gatewayExpectation`/`complianceRisk`/`focus`
    fields — worded around actual fields rather than inventing an
    "audit telemetry" feature that does not exist in code).
  - Removed the now-unused `DEFAULT_PROFILES` import (the fixed-role
    derivation it fed was deleted).
- Tests updated/added:
  - `packages/tui/tests/wizard-screens.test.ts`: added `capturedNote`
    coverage for both `renderWizardSelectScreen` and
    `renderWizardTextScreen` (present + absent cases, plus an ordering
    assertion that the note appears before the step banner).
  - `packages/tui/tests/wizard-multi-select.test.ts`: added a
    `capturedNote` coverage case.
  - `packages/tui/tests/driver-wizard.test.ts`: added an end-to-end
    driver test asserting the `✓ <title>: <valor>` echo appears on the
    step following a `text` answer and again following a `select`
    answer, and that no echo appears on the wizard's very first step.
  - `apps/cli/tests/tui.test.ts`: rewrote both existing end-to-end
    workspace-wizard scenarios for the new flow (sibling path defaults,
    `roles` as free text, arbitrary role `data` alongside `backend`,
    absolute `sddPath` answer honored verbatim, generic text-echo
    assertion); added new unit-test blocks for `sanitizeRoleName`,
    `parseRolesInput`, and `buildWorkspaceSetupWizardSteps` defaults/
    subtitles/`expand` behavior.

## Validation

From `C:\repos\Axiom Workspace\Axiom`:

- `npm run build` (tsc -b) — clean, no output (success).
- `npx vitest run apps/cli packages/tui packages/user-workspace` — 71
  test files, 716 tests, all passing. `packages/tui`'s 11 test files
  (165 → 171 tests after the new echo coverage) all green; `apps/cli`'s
  `tui.test.ts` (26 tests, including the 2 rewritten end-to-end
  scenarios + new unit blocks) green; `packages/user-workspace` fully
  green (unaffected by this increment, included per the requested
  scope). The `[orchestrator] FAIL: command=...` lines visible in the
  run output are expected stderr from OTHER pre-existing tests that
  intentionally exercise failure paths (e.g. `sync.test.ts`,
  `join.test.ts`, `start.test.ts`) — not failures of this increment's
  changes.

## Result

All four reported UX/behavior issues are fixed:

1. Every wizard step now echoes `✓ <title>: <valor>` for the
   immediately-preceding answer, generically in `@axiom/tui` (works for
   `text`, `select`, and `multi-select` steps alike, with zero business
   knowledge added to the package).
2. `sddPath`/`specPath`/role-path steps default to sibling-style paths
   (`../<slug>-sdd`, `../<slug>-spec`, `../<slug>-<role>`) and their
   subtitles state the absolute-path escape hatch explicitly;
   `path.resolve(cwd, absolutePath)` is confirmed (by test, using a real
   Windows tmp path) to return the absolute path unchanged.
3. The `roles` step is now a free-text comma-separated list accepting
   ANY role name and ANY count (including zero); `buildWorkspaceSetup
   SpecFromWizard` sanitizes and forwards them unchanged to
   `WorkspaceRepoSpec.roleKey`/`functionalRoleId`, which the engine
   already accepts as open strings — confirmed with an end-to-end test
   creating a `data` role repo alongside `backend`.
4. `profile`/`overlay` step subtitles now concisely explain their real
   effect, worded directly from `default-profiles.ts`'s actual fields.

## General spec integration

Integrated into the canonical spec in the round-3 cross-increment pass
(covering this increment and the two sibling round-3
`INC-20260705-workspace-*` increments: setup-registry-robustness and
code-repo-skills). Files updated:

- **05_Interfaces_Operativas.md** (PRIMARY) — added the "Mejoras de UX y
  roles custom del wizard" subsection: superseded the `roles` step
  description (fixed multi-select → free-text arbitrary comma-separated
  roles) and the `sddPath`/`specPath`/role-path defaults (`.`/`../<slug>.spec`
  → sibling `../<slug>-sdd`/`../<slug>-spec`/`../<slug>-<role>`, absolute
  paths accepted); added the generic `✓ <título>: <valor>` capture-echo
  mechanism (`@axiom/tui`); noted the `profile`/`overlay` explanatory
  subtitles. Also updated the layering paragraph to stop calling `roles`
  a `multi-select` reuse (only `adapters` now is).
- **01_Requisitos_Funcionales.md** — added a custom-roles sub-bullet to
  RF-AXM-024 (arbitrary free-text comma-separated roles, forwarded
  verbatim as open `roleKey`/`functionalRoleId`).
- **08_Glosario.md** — added the term "roles custom (nombre arbitrario)".
