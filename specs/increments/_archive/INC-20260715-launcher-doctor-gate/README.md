# Increment: launcher-doctor-gate

Status: closed
Date: 2026-07-15

## Goal

Before launching or executing anything from the web launcher (`axiom app`), always run
that project's doctor and show what is missing/failing on screen. The user must see the
project's health (pass/warn/fail checks) and be warned before performing a mutation when
doctor reports failures.

## Context

`axiom app` already exposes a launcher (P3/P4, front-longtail, epic-close-panels,
launcher-ux-ado increments) that crafts prompts, previews CLI commands, and executes
mutating transitions (confirmed-gated) or launches prompts to an adapter. None of that
flow surfaces the project's `axiom doctor` health today — a user can launch/execute
against a project with failing doctor checks (missing config, broken topology, etc.)
without any visible warning. `runDoctorChecks` (from `@axiom/doctor`, given a
`resolveProject` resolution from `@axiom/project-resolution`) already exists, is
synchronous, tolerant of unresolved projects (checks `skip` cleanly), and is already a
dependency of `apps/cli` (used by the existing `axiom doctor` CLI command).

## Scope

- A new read-only GET endpoint `/api/projects/:id/launcher/doctor` that runs
  `runDoctorChecks` against the resolved project and returns a compact summary + the
  non-passing checks (`fail`/`warn`).
- Wiring that endpoint into the existing `axiom app` HTTP route matcher/dispatcher.
- Front (`static/launcher/`): request doctor data whenever a project is selected, render
  a compact doctor panel (health line + issue list) above the guided wizard in the
  "Crear" view, with a manual refresh button.
- A visible (non-blocking) warning gate on the two mutating front actions (`Ejecutar` /
  `Lanzar`): when the doctor reports at least one `fail`, the first click warns and
  requires a second click to proceed — mirrors the existing execute arm-to-confirm
  pattern, stacked before it.

## Non-goals

- Not re-implementing any doctor check logic — this is a thin wrapper over
  `runDoctorChecks`.
- Not hard-blocking execute/launch when doctor fails — the gate is a visible warning the
  user can still deliberately override (Axiom does not add new enterprise-lifecycle
  approval gates here).
- Not touching `@axiom/doctor` or `@axiom/project-resolution` internals, `@axiom/tui`, or
  the `axiom doctor` CLI command itself.
- Not adding a new push (SSE) event for doctor changes — the front pulls doctor data on
  project selection and via manual refresh only.

## Acceptance criteria

- [x] `apiGetLauncherDoctor` exists in `app-launcher.ts`, returns
      `{ ok: true, doctor: { projectStatus, summary, issues, ok } }` for a resolvable
      project and a 404 for an unknown project id; never throws (doctor errors are
      caught and surfaced as a best-effort failure, not a server crash).
- [x] `GET /api/projects/:id/launcher/doctor` is routed in `app-api.ts` and dispatches to
      `apiGetLauncherDoctor`.
- [x] The front requests doctor data on project selection (alongside launcher data +
      registry) and renders a doctor panel (health line + issues) before the 3-step
      wizard in the "Crear" view; a manual "Revisar (doctor)" button re-requests it.
- [x] `doExecute()` and `doLaunch()` surface a doctor warning (armed, second-click-to-
      proceed) gate when the last-known doctor report has `ok === false`, stacked before
      the existing execute confirm-arm flow.
- [x] `npm run build` and the launcher-focused vitest suite stay green; a new focused
      test covers the doctor endpoint's ok/404 shape.

## Open questions

none — resolved by orchestrator

## Assumptions

- `runDoctorChecks` is safe to call synchronously per-request (no caching needed at this
  scale; matches the existing `axiom doctor` CLI command's usage).
- The front's "last-known doctor report" is refreshed on project selection and via the
  manual refresh button only (no SSE push channel for doctor in this increment).

## Implementation notes

- Server: `apps/cli/src/commands/app-launcher.ts` — new `apiGetLauncherDoctor` function
  (mirrors `apiGetLauncherRegistry`'s resolve-then-respond shape), wrapped in a
  try/catch so a doctor failure returns a 500 with a message rather than crashing the
  server.
- Routing: `apps/cli/src/commands/app-api.ts` — new `projects.launcherDoctor` route kind,
  a `GET /api/projects/:id/launcher/doctor` regex matcher next to `launcher/registry`,
  and a dispatch `case` mirroring the registry case.
- Front transport: `apps/cli/static/launcher/transport.js` — new `requestDoctor` message
  case mirroring `requestRegistry`.
- Front UI: `apps/cli/static/launcher/launcher.js` + `index.html` + `launcher.css` — new
  `S.doctor` state, `renderDoctor()`, a `#doctor` panel in the "Crear" view, a
  `#btn-doctor` refresh button, and a `S.launchArmed` flag (new) alongside the existing
  `S.confirmArmed` to gate `doExecute`/`doLaunch` on doctor failures.

## Validation

Executed from `C:/repos/Axiom Workspace/Axiom`:
- `npm run build` (tsc -b, full monorepo) — clean.
- `npx vitest run apps/cli/tests -t launcher` — launcher-tagged tests green.
- New focused test file `apps/cli/tests/launcher-doctor.test.ts` covering
  `apiGetLauncherDoctor` (ok shape + 404) added and green.

See the orchestrator's report for the verbatim command tails.

## Result

Implemented as scoped: a new best-effort, non-crashing doctor endpoint wired into the
existing launcher route table, and a front doctor panel that loads on project selection
and gates the two mutating actions (execute/launch) behind a visible, dismissible
warning when doctor reports failures. No doctor logic was reimplemented; no new
dependencies were introduced.

## General spec integration

Integrado por el pase final de la orquestación (autopilot, 2026-07-15):
- `01_Requisitos_Funcionales.md` → **RF-AXM-030 Gate de doctor pre-lanzamiento en el launcher**.
- `05_Interfaces_Operativas.md` → sección "tanda INC-20260715-*" (gate de doctor: panel de salud + segundo clic ante fallos).
- `04_Flujos_SDD_y_Ciclo_de_Vida.md` → sección "Onboarding guiado desde el launcher" (doctor antes de lanzar).
- `08_Glosario.md` → término "Gate de doctor (pre-lanzamiento)".

Archivado en `Axiom.Spec/specs/increments/_archive/`.
