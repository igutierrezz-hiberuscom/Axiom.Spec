# Increment: launcher-onboarding

Status: closed
Date: 2026-07-15

## Goal

Make the web launcher (`axiom app`) a guided visual front for ONBOARDING, so a team
member can start working WITHOUT the terminal/TUI: install Axiom in a NEW project, join
an EXISTING Axiom project, and register roles + associate them to repos — all from the
browser, with a folder-picker helper for choosing paths. Under the hood the launcher
calls the SAME core run-functions the CLI/TUI use (`runInit`, `runProjectsJoin`,
`runRolesRegister`, `runRolesAssign`).

## Context

`axiom app` already exposes a project-scoped launcher (families/actions catalog, craft,
execute, registry, doctor, launch, ADO/git operator panels — P3/P4, front-longtail,
epic-close-panels, launcher-ux-ado, launcher-doctor-gate increments). All of that assumes
a project is ALREADY selected in the project dropdown (populated from
`GET /api/projects`, the user-level registry at `state.homeDir`). There was no
browser-native way to get a project INTO that registry in the first place, or to
provision team/code roles for it — a user had to drop to the CLI/TUI (`axiom init`,
`axiom projects join`, `axiom roles register/assign`) first. This increment adds a
pre-project "Instalar / Unirse" onboarding tab that wraps those exact run-functions.

## Scope

- New server module `apps/cli/src/commands/app-onboarding.ts` with six thin wrappers,
  all confirmed-gated for mutations:
  - `apiGetOnboardingOptions` — static select-option catalog (targets/profiles/
    overlays/layouts/repoRoles) sourced from `init.ts`'s canonical const arrays.
  - `apiBrowseDirectory` — a folder-picker backend (`fs.readdirSync`, directories only),
    never throws.
  - `apiLauncherInstall` — wraps `runInit` (new project).
  - `apiLauncherJoin` — wraps `runProjectsJoin` (existing project).
  - `apiLauncherRolesRegister` — wraps `runRolesRegister` (project-scoped).
  - `apiLauncherRolesAssign` — wraps `runRolesAssign` (project-scoped).
- New server-level routes (pre-project, no project id): `GET /api/launcher/options`,
  `GET /api/launcher/browse`, `POST /api/launcher/install`, `POST /api/launcher/join`.
- New project-scoped routes: `POST /api/projects/:id/launcher/roles/register`,
  `POST /api/projects/:id/launcher/roles/assign`.
- Front: a new "Instalar / Unirse" tab (`#view-onboard`) with three cards (install /
  join / roles), a shared folder-picker widget, and transport wiring
  (`transport.js`) for the six new message types.
- After a successful install/join, the front re-requests `/api/projects` so the new/
  joined project appears in the existing project selector immediately.

## Non-goals

- Not re-implementing `init`/`join`/`roles register`/`roles assign` logic — only calling
  the existing `run*` functions.
- Not widening the `@axiom/cli-commands` barrel — run-functions are imported directly
  from `apps/cli/src/commands/*` (same package).
- Not touching `@axiom/tui`.
- Not adding a native OS file-picker dialog — the folder picker is a simple server-
  backed directory listing rendered in the browser.
- Not adding any new npm dependency.

## Acceptance criteria

- [x] `apiGetOnboardingOptions`, `apiBrowseDirectory`, `apiLauncherInstall`,
      `apiLauncherJoin`, `apiLauncherRolesRegister`, `apiLauncherRolesAssign` exist in
      `app-onboarding.ts` with the exact result shapes described in the brief.
- [x] Install/join/browse are routed as SERVER-LEVEL endpoints (`/api/launcher/...`,
      no project id); roles register/assign are routed as PROJECT-SCOPED endpoints
      (`/api/projects/:id/launcher/roles/...`).
- [x] Every mutating endpoint (install/join/roles-register/roles-assign) is
      confirmed-gated: without `confirmed:true` it returns a preview only, with zero
      filesystem/registry mutation.
- [x] `apiBrowseDirectory` never throws — an unreadable path returns
      `{ ok:false, status:400 }`.
- [x] The front has a new "Instalar / Unirse" tab with install/join/roles forms and a
      folder-picker helper wired through `transport.js`.
- [x] After a successful (confirmed) install or join, the front asks the transport to
      refresh `/api/projects` so the new/joined project shows up in the existing
      project selector.
- [x] `npm run build` passes clean; the launcher-focused vitest suite (including the new
      `launcher-onboarding.test.ts`) is green; the two edited front JS files pass
      `node --check`.

## Open questions

none — resolved by orchestrator

## Assumptions

- The launcher's user-level registry (`state.homeDir`) is the SAME home the CLI/TUI
  use — install/join must be called with `homeDirOverride`/`homeDir` set to
  `state.homeDir` so the newly-registered project is visible to the SAME project
  selector.
- `apiLauncherRolesRegister`/`Assign` operate against the resolved primary repo root
  (`resolveProjectRoot`), same primary-repo resolution the rest of the project-scoped
  launcher endpoints already use — no new multi-repo-aware API surface introduced.
- Testing the confirmed (mutating) path of every onboarding endpoint end-to-end would
  require heavier fixtures (a real resolvable `axiom.yaml` for join, a full topology
  fixture for roles); the test suite covers the confirmed path for `roles register`
  (reusing the existing `app-launcher.test.ts` topology fixture) and for `join`
  (using a real `axiom.yaml` produced by a direct `runInit` call as the fixture), plus
  preview/validation-level coverage for install, browse, and roles-assign. See
  `## Validation` for the exact list.

## Implementation notes

- `apps/cli/src/commands/app-onboarding.ts` (new): imports `AppState`,
  `resolveProjectRoot` from `./app-api`; `runInit`, `ADAPTER_TARGETS`, `REPO_ROLES` from
  `./init`; `runProjectsJoin` from `./projects`; `runRolesRegister`, `runRolesAssign`
  from `./roles`. Uses `node:fs`/`node:path`/`node:os` for the folder picker.
- `apps/cli/src/commands/app-api.ts`: imports the six functions; adds
  `launcher.options` / `launcher.browse` / `launcher.install` / `launcher.join` /
  `projects.rolesRegister` / `projects.rolesAssign` to the `RouteMatch` union; adds
  matchers (server-level near the `/api/projects` list matcher, project-scoped near the
  other `/api/projects/:id/launcher/*` matchers); adds dispatch cases reusing the
  existing `readJsonBody`/`sendJson` plumbing (install is `await`ed — the dispatcher was
  already `async`).
- `apps/cli/static/launcher/transport.js`: six new message cases
  (`requestOnboardingOptions`, `browseDirectory`, `installProject`, `joinProject`,
  `registerRole`, `assignRole`). Server-level calls use absolute `/api/launcher/...`
  paths (NOT `base()`, which is project-scoped); roles calls use `base()`.
- `apps/cli/static/launcher/index.html` / `launcher.js` / `launcher.css`: a new
  `data-view="onboard"` tab, `#view-onboard` section with three cards (install / join /
  roles) plus a shared folder-picker list, and the arm-to-confirm preview→execute
  pattern already used elsewhere in the front (`doExecute`'s idiom / `panels.js`'s
  `wireWfForm` idiom).

## Validation

Executed from `C:/repos/Axiom Workspace/Axiom`. See the orchestrator's report for the
verbatim command tails; summary:
- `npm run build` (tsc -b, full monorepo) — clean.
- New test file `apps/cli/tests/launcher-onboarding.test.ts` (harness copied from
  `apps/cli/tests/app-launcher.test.ts`), covering:
  - `apiGetOnboardingOptions` returns the five option arrays.
  - `apiBrowseDirectory` on a known temp dir returns its subdirectories + `parent`;
    on a bogus path returns `{ ok:false }` (never throws).
  - `apiLauncherInstall` unconfirmed → preview-only, no mutation; 400 on missing path.
  - `apiLauncherJoin` unconfirmed → preview-only, no mutation; CONFIRMED path against a
    real `axiom.yaml` fixture (produced by a direct `runInit` call) → project appears in
    a fresh registry.
  - `apiLauncherRolesRegister` unconfirmed → preview-only; CONFIRMED path against the
    `app-launcher.test.ts`-style topology fixture → role persisted in `topology.yaml`.
  - `apiLauncherRolesAssign` unconfirmed → preview-only, no mutation; 400 on missing
    `repoId`/`role`.
- `npx vitest run apps/cli/tests -t launcher` and
  `apps/cli/tests/launcher-onboarding.test.ts` — green.
- `node --check` on `transport.js` and `launcher.js`.

## Result

Implemented as scoped. `apps/cli/src/commands/app-onboarding.ts` (new) wraps `runInit`,
`runProjectsJoin`, `runRolesRegister`, `runRolesAssign` behind six thin, confirmed-gated
functions; `apps/cli/src/commands/app-api.ts` routes them (server-level for
install/join/browse/options, project-scoped for roles register/assign) through the
existing `readJsonBody`/`sendJson` plumbing. The front gained a new "Instalar / Unirse"
tab (`index.html` + `launcher.js` + `launcher.css`) with three cards (install / join /
roles) and a shared folder-picker widget wired through six new `transport.js` message
cases; a successful install/join re-requests `/api/projects` so the new/joined project
appears in the existing selector. `npm run build` is clean; the new
`launcher-onboarding.test.ts` (15 tests) is green, as are the existing launcher-focused
suites and the broader `app.test.ts`/`projects.test.ts`/`init.test.ts`/`roles.test.ts`/
`app-plugins.test.ts` regression check; both edited front JS files pass `node --check`.
No `init`/`join`/`roles` logic was reimplemented; no new dependencies were introduced.

## General spec integration

Integrado por el pase final de la orquestación (autopilot, 2026-07-15):
- `01_Requisitos_Funcionales.md` → **RF-AXM-031 Onboarding visual desde el launcher**.
- `04_Flujos_SDD_y_Ciclo_de_Vida.md` → sección "Onboarding guiado desde el launcher" (install/join/roles + explorador de carpetas).
- `05_Interfaces_Operativas.md` → sección "tanda INC-20260715-*" (pestaña Instalar/Unirse, endpoints server-level + project-scoped).
- `07_Gobierno_y_Seguridad.md` → nota de mutación confirm-gated/best-effort en las superficies nuevas.
- `08_Glosario.md` → término "Onboarding del launcher".
- `00_Resumen_Ejecutivo.md` → mención en la tanda (front visual de onboarding sin terminal/TUI).

Archivado en `Axiom.Spec/specs/increments/_archive/`.
