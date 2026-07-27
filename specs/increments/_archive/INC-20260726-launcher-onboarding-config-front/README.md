# Increment: Launcher onboarding config front (role/repo/adapters/tools/execution-mode)

Status: closed
Date: 2026-07-26

## Goal

Make the web launcher (`axiom app`) the EASY way to install/join an Axiom
project entirely from the browser — role selection, repo path(s), adapters
(primary + additional), tools to install, execution mode, profile/overlay/
layout — with a preview → confirm gate, HONESTLY wiring every parameter that
has a real supported path and clearly surfacing (never faking) the ones that
don't yet.

## Context

`apps/cli/src/commands/app-onboarding.ts` (INC-20260715-launcher-onboarding)
already exposed THIN, confirmed-gated wrappers: `apiGetOnboardingOptions`,
`apiBrowseDirectory`, `apiLauncherInstall` (→ `runInit`), `apiLauncherJoin`
(→ `runProjectsJoin`), `apiLauncherRolesRegister`/`apiLauncherRolesAssign`
(→ `runRolesRegister`/`runRolesAssign`). Before this increment:

- `runInit`'s `InitArgs` only accepted a SINGLE `target` adapter, no
  `adapters[]`, no `tools`, no `executionMode` — and `apiLauncherInstall`
  called `runInit` ALONE, so a confirmed install never materialized any
  per-adapter output (only the generic, adapter-agnostic root `AGENTS.md`).
- Execution mode (`'in-place'|'worktree'`) lived in `axiom configure
  --execution-mode` (INC-20260724-worktree-mode-selection), never reachable
  from the launcher's onboarding forms.
- `generateWorkspaceAdapters` (`apps/cli/src/commands/workspace-adapters.ts`)
  already materializes a full `adapters[]` set (best-effort, bundled
  templates, never throws) for `axiom workspace setup`/`configure`/`sync` —
  but `runInit`/`apiLauncherInstall` never called it.
- The toolchain sub-system (`apps/cli/src/commands/toolchain.ts`,
  `runToolchainAdd`) requires a per-project `axiom.config/toolchain-
  catalog.yaml` (or the legacy `integrations.yaml` fallback) to already exist
  — neither is scaffolded by `runInit`, so "install tool X" has NO cleanly-
  callable path for a brand-new project.
- `ADAPTER_LABELS` (friendly adapter display names) existed ONLY in
  `app-launcher.ts`, scoped to the 9-id `AXIOM_ADAPTER_ROUTING` set — not the
  full 10-id `ADAPTER_TARGETS` set `runInit --target` actually accepts.

## Scope

- `apps/cli/src/commands/_adapter-labels.ts` (NEW): single shared
  `ADAPTER_LABELS` map covering the UNION of `ADAPTER_TARGETS` (init/
  workspace-adapters) and `AXIOM_ADAPTER_ROUTING.adapters` (launcher routing)
  ids — `app-launcher.ts` now imports it instead of declaring its own local
  copy (values unchanged for the 9 pre-existing ids).
- `apps/cli/src/commands/init.ts`: exported the previously-private
  `DEFAULT_PROFILE`/`DEFAULT_OVERLAY` constants (additive, no behavior
  change) so `app-onboarding.ts`'s join path can reuse `runInit`'s OWN
  (profile, overlay) defaults without re-declaring them.
- `apps/cli/src/commands/app-onboarding.ts`:
  - `OnboardingOptions` gained `adapters` (`{id,label}[]`, full
    `ADAPTER_TARGETS` catalog), `executionModes` (`@axiom/install-profiles`'
    `EXECUTION_MODES`), `tools` (`{id,label}[]`, a fixed snapshot mirroring
    this project's own `axiom.config/toolchain-catalog.yaml` — the only
    canonical tool-id source found).
  - `LauncherInstallBody`/`LauncherInstallPreview` gained `adapters`
    (additional, beyond the primary `target`), `executionMode`, `tools`.
    `apiLauncherInstall`'s CONFIRMED path now additionally calls (both real,
    thin, already-shipped functions — never re-implemented):
    1. `generateWorkspaceAdapters` with the full adapter set (primary +
       additional) — closes the "runInit never materializes per-adapter
       output" gap.
    2. `runConfigure({ cwd, executionMode })` when `executionMode` is
       provided (best-effort: failure surfaced in
       `executionMode.warning`, never rolls back init/adapters).
    `tools` is echoed in the response (`tools.applied: false` + an honest
    note) — NOT wired (see Assumption 3).
  - `LauncherJoinBody` gained `adapters`/`executionMode` (same real,
    best-effort wiring as install, using `runInit`'s own profile/overlay
    defaults since `join` has no functional-profile concept of its own).
    `apiLauncherJoin` became `async` (it now `await`s the two calls above).
  - New helpers: `sanitizeAdapterList`/`sanitizeExecutionMode` (defensive
    filtering of unknown ids/values — never crash the request over a stray
    value), `ONBOARDING_WIRING_NOTES` (static, honest wiring-approach text
    surfaced in BOTH preview and executed responses).
- `apps/cli/src/commands/app-api.ts`: `apiLauncherJoin` call site now
  `await`s it (route handler was already `async`).
- `apps/cli/static/launcher/index.html`: install/join cards gained an
  execution-mode `<select>`, an additional-adapters checkbox-group, and (
  install only) a tools checkbox-group, each with a Spanish help line
  stating exactly how it's applied (or that it's pending).
- `apps/cli/static/launcher/launcher.js`: `fillSelect` gained an optional
  `defaultValue` param (pre-selects a value instead of a blank "(server
  default)" placeholder — used for the primary-adapter and execution-mode
  selects); new `fillCheckboxGroup` helper; `collectFormValues` now detects
  GROUPED checkboxes (several inputs sharing a `name`) and collects them into
  an array (adapters/tools), while a LONE checkbox still collects as a
  boolean (unchanged, e.g. `roles/assign`'s `primary`); `renderOnboardResult`
  now also renders the `adapters`/`executionMode`/`tools` outcome blocks;
  `handleOnboardResult` auto-selects the onboarded project after a
  successful install/join (see Assumption 4 — this is how "roles assignment
  as part of the join flow" is satisfied, with ZERO new backend code).
- `apps/cli/static/launcher/launcher.css`: `.checkbox-group` rule (flex-wrap
  layout for the new checkbox-group widgets).
- `apps/cli/tests/launcher-onboarding.test.ts`: extended — enriched options
  catalog; install preview echoes the full resolved config + honesty notes;
  confirmed install applies primary+additional adapters and execution-mode
  (asserted against real materialized files + `install-profile.json`), tools
  surfaced-not-applied; a minimal confirmed install (no new fields) still
  materializes the PRIMARY adapter (documented behavior change — see
  Assumption 2) with no `executionMode`/`tools` keys; join preview/confirmed
  mirrors the same coverage; a minimal confirmed join asserts the response
  has NO `adapters`/`executionMode` keys (backward-compat safety net). Every
  pre-existing confirm-gate assertion is unchanged.

## Non-goals

- Real tool installation (binary download/setup) — no such function exists
  anywhere in the codebase today (`runToolchainAdd` only declares an id in
  `toolchain.yaml`, and requires a catalog file `runInit` doesn't scaffold).
  Out of scope; documented as a deferred, honestly-surfaced gap.
- A new roles-assignment ENDPOINT for the join flow — reuses the EXISTING
  `apiLauncherRolesAssign` (unchanged) via a front-only auto-select of the
  onboarded project (see Assumption 4).
- Changing `runInit`'s own CLI-facing defaults (`target: 'opencode'`) — the
  launcher FRONT pre-selects `claude-code`/`in-place` as ITS OWN defaults
  (see Assumption 1); a bare `axiom init` (CLI, no `--target`) is unaffected.
- Modifying `generateWorkspaceAdapters`/`runConfigure`/`installProfile`
  internals — reused exactly as shipped by prior increments.
- Adding execution-mode/adapters to `axiom workspace setup`'s TUI wizard —
  out of scope (that wizard doesn't call `installProfile` either, per
  INC-20260724-worktree-mode-selection's own Assumption 2).

## Acceptance criteria

- [x] `GET /api/launcher/options` returns `adapters` (full `ADAPTER_TARGETS`
      + friendly labels), `executionModes`, `tools`, alongside the 5
      original fields.
- [x] Install card surfaces name/path/profile/overlay/layout/role/primary
      adapter/additional adapters/tools/execution-mode; the preview
      (unconfirmed) echoes the FULL resolved config + honesty notes and
      performs NO mutation.
- [x] Confirmed install applies: `runInit` (base scaffold, unchanged) →
      `generateWorkspaceAdapters` (primary + additional adapters, real
      per-adapter file output) → `runConfigure` when `executionMode` is
      given (real `install-profile.json` persistence, best-effort).
- [x] Tools selection is surfaced (never dropped, never faked as applied) —
      response includes `tools.applied: false` + an explicit note pointing
      at `axiom toolchain add --id <id>`.
- [x] Join card surfaces path/role/adapters/execution-mode with the same
      preview→confirm gate; confirmed join applies adapters/execution-mode
      via the SAME real functions as install, best-effort.
- [x] Team/code role assignment is reachable as part of the join flow (the
      pre-existing "Roles" card, via the EXISTING `apiLauncherRolesAssign`,
      auto-activated for the just-onboarded project).
- [x] `npm run build` (`tsc -b`) passes clean.
- [x] `apps/cli/tests/launcher-onboarding.test.ts` covers the enrichment;
      full `apps/cli/tests` suite green.

## Open questions

None blocking — every ambiguity was resolved per "existing-precedent-first"
(see Assumptions).

## Assumptions

1. **Front defaults: primary adapter `claude-code`, execution-mode
   `in-place`.** Both are PRE-SELECTED (not a blank "(server default)"
   placeholder) in the two forms, per the brief's explicit instruction.
   `runInit`'s OWN internal fallback (`opencode`, used by any CLI caller
   that omits `--target`) is UNCHANGED — this is a front-only default,
   implemented via `fillSelect`'s new `defaultValue` param.
2. **A confirmed install ALWAYS calls `generateWorkspaceAdapters` for at
   least the primary adapter**, even when the front sends no `adapters[]`
   at all (a deliberate, documented behavior IMPROVEMENT over the
   pre-increment gap where `runInit` alone never produced per-adapter
   output) — locked in by a dedicated test
   (`'confirmed, no adapters/executionMode/tools selected → still
   materializes the PRIMARY (default) adapter'`).
3. **Tools are surfaced, not wired**, because `runToolchainAdd` requires a
   per-project `axiom.config/toolchain-catalog.yaml` (or the legacy
   `integrations.yaml`) that `runInit` never scaffolds — a project just
   created by this launcher literally cannot satisfy that dependency yet.
   `ONBOARDING_TOOL_CATALOG` is a fixed snapshot mirroring THIS project's
   own catalog file (7 ids); the response always says so explicitly
   (`tools.note`), per the brief's "surface, don't fake" directive.
4. **"Roles assignment as part of the join flow" is a FRONT-ONLY
   composition, not a new endpoint.** `apiLauncherRolesAssign` (project-
   scoped, confirmed-gated, already fully tested) is untouched; after a
   successful install/join, `launcher.js`'s `handleOnboardResult` calls the
   existing `selectProject(id)` with the onboarded project's id, which
   `configure`s the bridge for that project — making the pre-existing
   "Roles del proyecto seleccionado" card (register/assign) immediately
   usable without an extra manual selector click. Zero new backend risk;
   100% reuse.
5. **Join's adapter materialization reuses `runInit`'s OWN
   (`DEFAULT_PROFILE`, `DEFAULT_OVERLAY`) = (`'builder'`, `'local-only'`)**
   — `join` has no functional-profile concept of its own; these two values
   only steer discovery-mode/template variables inside the generators, they
   never gate correctness. Exported (additive) from `init.ts` for reuse
   rather than re-declared.
6. **`runConfigure` — not a lower-level direct `installProfile` call — is
   the execution-mode application seam**, per the brief's explicit "reuse
   what configure.ts does" instruction. This means a primary target of
   `copilot-vscode`/`github-copilot` on a brand-new project (no
   `axiom.spec/templates/copilot-instructions.template.md` on disk yet) can
   make this ONE step fail — caught, non-fatal, surfaced in
   `executionMode.warning`; the `runInit` + `generateWorkspaceAdapters`
   steps that already ran are NEVER rolled back. This mirrors the exact
   risk a CLI user would hit running `axiom init --target github-copilot &&
   axiom configure` on the same fresh project — a pre-existing product gap,
   not one this increment introduces.
7. **`ADAPTER_LABELS` moved to a new shared file, `_adapter-labels.ts`**,
   rather than importing `app-onboarding.ts` → `app-launcher.ts` (which
   would work but reads backwards for a file the increment brief called
   "already large") or duplicating the map. Values for the 9 ids
   `app-launcher.ts` already had are byte-identical; `copilot-vscode`/
   `litellm` were added (present only in `ADAPTER_TARGETS`, not in
   `AXIOM_ADAPTER_ROUTING.adapters`).

## Implementation notes

### A real regression caught and fixed during this increment

The first implementation imported `runConfigure` via a RELATIVE path
(`from './configure'`). `npm run build` (`tsc -b`) passed clean — `tsc`
type-checks the sibling `.ts` source file fine — but `configure.ts` is a
SINGLE-OWNED file (INC-20260703-cli-commands-single-ownership:
`apps/cli/tsconfig.json`'s `exclude` list keeps `apps/cli`'s OWN build from
ever emitting `dist/commands/configure.js`; only `@axiom/cli-commands`'s
build does). The relative import compiled fine but threw
`MODULE_NOT_FOUND` at RUNTIME the moment any command loaded
`app-onboarding.ts` transitively — which includes `axiom mcp serve`,
spawned by `apps/cli/tests/e2e/workspace-mcp.e2e.test.ts`. That is exactly
why that e2e file's two "spawned server" tests started timing out (the
child process crashed at require-time before it could ever answer
`initialize`). Fixed by importing `runConfigure` from `@axiom/cli-commands`
instead (the same seam every other apps/cli-owned file already uses for
`configure.ts`'s exports) — confirmed via a manual `node dist/index.js mcp
serve ...` spawn (crashed with the `MODULE_NOT_FOUND` stack before the fix,
started cleanly after) and by re-running the full `apps/cli/tests` suite
green. `workspace-adapters.ts`/`init.ts`/the new `_adapter-labels.ts` are
NOT single-owned (not in that exclude list), so their relative imports are
fine.

### Wiring summary (per parameter)

| Field (install/join) | Wired via | Real path? |
|---|---|---|
| name/path/profile/overlay/layout/role/primary adapter | `runInit` | Yes (pre-existing) |
| additional adapters | `generateWorkspaceAdapters` | Yes (NEW this increment) |
| execution-mode | `runConfigure` | Yes, best-effort (NEW) |
| tools | — | NO — surfaced + honest note only |
| team/code role assignment (join) | `apiLauncherRolesAssign` via front auto-select | Yes (pre-existing endpoint, NEW front reachability) |

## Validation

Ran from `C:\repos\Axiom Workspace\Axiom`:

- `npm run build` (`tsc -b`) — PASSED, clean, exit 0 (re-verified after the
  `configure` import fix above).
- `npx vitest run apps/cli/tests/launcher-onboarding.test.ts` — **22 tests
  passed** (was 12 before this increment; +10 new: enriched options,
  install preview/confirmed with adapters+executionMode+tools, install
  minimal-confirmed default-materialization, join preview/confirmed with
  adapters+executionMode, join minimal-confirmed backward-compat).
- `npx vitest run apps/cli/tests/app-launcher.test.ts
  packages/launcher/tests/adapter-routing.test.ts` — **28 tests passed**
  (confirms the `ADAPTER_LABELS` move to `_adapter-labels.ts` is byte-
  identical for every pre-existing id).
- `npx vitest run apps/cli/tests/init.test.ts apps/cli/tests/configure.test.ts
  apps/cli/tests/workspace-adapters.test.ts` — **33 tests passed** (the
  three modules this increment reuses/extends exports from).
- `npx vitest run apps/cli/tests` (full suite, safety pass) — **125 files /
  1180 tests passed**, zero failures. Includes
  `apps/cli/tests/e2e/workspace-mcp.e2e.test.ts` (the file that exposed the
  single-ownership regression above — green after the fix, re-confirmed
  both standalone and inside the full run).

No pre-existing failures were encountered in any touched or adjacent scope
in the FINAL run (the transient `MODULE_NOT_FOUND`-induced timeout above was
this increment's own bug, caught and fixed before closing — not a residual
flake).

## Result

Implemented. The launcher's install/join cards now surface every requested
onboarding parameter (name, path, profile, overlay, layout, this-repo role,
primary + additional adapters, tools, execution-mode) with clear Spanish
labels, a preview that echoes the FULL resolved config plus explicit
wiring-honesty notes, and a confirm gate preserved exactly as before. Every
parameter with a real, cleanly-callable path is genuinely wired
(`runInit` → `generateWorkspaceAdapters` → `runConfigure`, all pre-existing,
never re-implemented); the one parameter without such a path (tools) is
surfaced for review with an explicit, honest "pending" note — never faked as
applied. Team/code role assignment is reachable as part of the join flow via
front-only auto-selection of the onboarded project, reusing the existing,
untouched `apiLauncherRolesAssign` endpoint. A real single-ownership import
regression (caught via the full-suite e2e safety pass) was found and fixed
before closing.

## General spec integration

Not integrated into `Axiom.Spec/specs/00..08` directly, per the brief
("Do NOT edit `Axiom.Spec/specs/00..08`") — consistent with the sibling
`INC-20260726-*` increments in this same batch, which were also left for a
later consolidated integration pass. Stable facts worth carrying into that
future pass:

- The web launcher's onboarding (install/join) forms now cover the full
  (profile, overlay, layout, role, adapters[], tools, executionMode)
  configuration surface, with adapters/executionMode REALLY wired
  (`generateWorkspaceAdapters`/`runConfigure`) and tools explicitly deferred
  pending a real toolchain installer.
- `ADAPTER_LABELS` (friendly adapter display names) now lives in
  `apps/cli/src/commands/_adapter-labels.ts`, shared by the launcher's
  prompt-crafting adapter picker (`app-launcher.ts`) AND the onboarding
  catalog (`app-onboarding.ts`) — any future adapter id addition should
  update that ONE map.
- Reconfirmed the single-ownership convention
  (INC-20260703-cli-commands-single-ownership): `apps/cli`-owned files must
  import `configure.ts`/`sync.ts`/`upgrade.ts`/`rollback.ts`/
  `workspace-adapter-templates.ts`/`model.ts`/`components.ts`/
  `index-cmd.ts`/`validate-changes.ts`/`repair.ts`/`toolchain.ts`/`mcp.ts`
  exports from `@axiom/cli-commands`, NEVER by relative path — `tsc -b`
  does NOT catch the violation (it type-checks fine); only a real spawn of
  the built CLI does.
