# Increment: launcher-ado-bridge

Status: closed
Date: 2026-07-15

## Goal

When the Azure DevOps plugin (tracker) is CONFIGURED for the project, creating an
increment or bug from the launcher's "Crear" flow should also OFFER to create the
corresponding work item in the plugin, pre-filled and one-click — "si tiene plugin
instalado, crea también lo propio en el plugin". This must NOT couple the core
lifecycle: increment/bug creation is unchanged; the launcher only DETECTS the
configured plugin and surfaces a pre-filled, confirmed-gated ADO create step that
reuses the EXISTING ADO create endpoint.

## Context

The launcher execute flow (`apiExecuteLauncherAction` in `app-launcher.ts`) already
handles `increment-new`/`bug-new` via `executeSubcommand`, returning
`{ executed:true, executable:true, command, result }` on a successful confirmed
transition. Separately, `app-launcher-ado.ts` already ships a config-gated,
confirmed-gated ADO work-item-create endpoint (`apiAdoCreateWorkItem`), and the
front already has a wired transport message (`adoWorkItemCreate` in
`transport.js` → `POST /launcher/ado/work-items` → `adoWorkItemCreateResult`),
today consumed only by the "ADO & Git" operator panel (`panels.js`). Tracker
configuration lives at `.axiom-state/<project>/tracker.json`, normalized by
`@axiom/tracker`'s `resolveTrackerConfig`/`isRealTrackerRequested` (real ADO
requires `kind:'ado'`, `enabled:true`, `organization`, `project`).

`app-launcher-ado.ts` already imports from `app-launcher.ts` (`launcherEventHub`),
so the reverse import would create a cycle — the tracker-status detection needed
by `app-launcher.ts` had to live in a new, neutral module.

## Scope

- New neutral module `apps/cli/src/commands/_tracker-status.ts`:
  `resolveLauncherTrackerStatus(projectRoot)` — a network-free, non-throwing
  inspection of `tracker.json` (via `resolveProject` + `defaultLoadTrackerConfig`
  + `isRealTrackerRequested`), returning `{ configured, provider, trackerKind }`.
- `app-launcher.ts`: a new pure helper `buildTrackerSuggestion(state, projectId,
  actionKey, values)` that, for `increment-new`/`bug-new` only, maps the create
  action to a suggested ADO work item (`type`: `'User Story'`/`'Bug'`, `title`
  from the form values, falling back to `id` then a default label) and reports
  whether ADO is configured. `apiExecuteLauncherAction` attaches this as
  `body.trackerSuggestion` ONLY when the confirmed transition succeeded
  (`exec.result.exitCode === 0`) and only for the two create actions — purely
  additive to the existing response shape.
- Front (`static/launcher/`): a new `#tracker-suggestion` card in the "Crear"
  view, below the prompt-preview panel. Renders a muted "not configured" note
  when `configured:false`, or a highlighted card with an editable title and a
  "Crear work item en Azure DevOps" button when `configured:true`. The button
  reuses the arm-to-confirm idiom (first click previews via the EXISTING
  `adoWorkItemCreate` transport message with `confirmed:false`, second click
  confirms with `confirmed:true`). A new, independent `adoWorkItemCreateResult`
  handler in `launcher.js` updates the status bar (coexists with `panels.js`'s
  own handler for its ADO workflow panel).

## Non-goals

- Does NOT auto-create the work item — it offers a confirmed, one-click step;
  the user must explicitly confirm.
- Does NOT modify the core increment/bug lifecycle (`runIncrementSubcommand`,
  `runBugSubcommand`, `executeSubcommand`) — the suggestion is attached only
  after a successful, unchanged transition.
- Does NOT change `@axiom/tracker` or `@axiom/tracker-ado` — reuses
  `isRealTrackerRequested`/`resolveTrackerConfig` and the existing
  `apiAdoCreateWorkItem` endpoint verbatim; no new ADO create path was added.
- Does NOT add a new transport message — the front reuses the existing
  `adoWorkItemCreate` message end-to-end.

## Acceptance criteria

- [x] `resolveLauncherTrackerStatus` in `_tracker-status.ts` never touches the
      network, never throws, and returns `configured:true` only for a real
      (`kind:'ado'`, `enabled:true`, `organization`+`project` present) tracker
      config; `configured:false` (provider `'none'`, trackerKind `'none'`) for
      any missing/incomplete/absent config or unresolved project.
- [x] `buildTrackerSuggestion` returns `undefined` for any action other than
      `increment-new`/`bug-new`, and for those maps to
      `{ artifactKind:'increment', workItem.type:'User Story' }` /
      `{ artifactKind:'bug', workItem.type:'Bug' }` respectively, with `title`
      from `values.title` → `values.id` → a default label, in that order.
- [x] `apiExecuteLauncherAction`'s confirmed + transition branch attaches
      `body.trackerSuggestion` only when the execute succeeded AND the action
      is a create action; preview responses (`confirmed` absent/false) and
      non-create confirmed executes never carry it.
- [x] The front's "Crear" flow shows the tracker-suggestion card after a
      successful create; the "not configured" case is informational only (no
      button); the "configured" case offers an editable title + a
      confirm-gated button that reuses `adoWorkItemCreate` end-to-end (no new
      ADO create endpoint).
- [x] `npm run build` (tsc -b) is clean; the launcher-focused vitest suite
      (`-t launcher` + the new `launcher-ado-bridge.test.ts`) is green; the
      edited front JS passes `node --check`.

## Open questions

none — resolved by orchestrator

## Assumptions

- `isRealTrackerRequested` (already exported by `@axiom/tracker`) is the
  correct, canonical "is this a real ADO config" guard — reused verbatim
  instead of re-deriving the same field checks in `_tracker-status.ts`.
- A single project-scoped `tracker.json` (no per-action override) is
  sufficient to decide whether to surface the suggestion; multi-tracker setups
  are out of scope.
- Reusing the existing `adoWorkItemCreate` transport message (rather than
  adding a dedicated one) is acceptable UX: the front's status bar reports the
  create result, and the operator can still see the created work item via the
  existing "ADO & Git" panel/ADO suggestions.

## Implementation notes

- `apps/cli/src/commands/_tracker-status.ts` (new): `resolveLauncherTrackerStatus`
  wraps `resolveProject` + `defaultLoadTrackerConfig` (from `external-sync.ts`)
  + `isRealTrackerRequested`/`resolveTrackerConfig` (from `@axiom/tracker`) in a
  try/catch, defaulting to the safe local-only status on any error. Deliberately
  imports neither `app-launcher.ts` nor `app-launcher-ado.ts` to avoid the
  circular import `app-launcher-ado.ts` → `app-launcher.ts` already creates.
- `apps/cli/src/commands/app-launcher.ts`: `TrackerSuggestion` interface +
  `buildTrackerSuggestion` (exported for unit testing), called from
  `apiExecuteLauncherAction`'s confirmed-transition branch and spread into the
  response body only when defined.
- `apps/cli/static/launcher/index.html`: new `#tracker-suggestion` panel inside
  `#view-create`, below the preview panel, `hidden` by default.
- `apps/cli/static/launcher/launcher.js`: `S.trackerSuggestion` state,
  `renderTrackerSuggestion()` + `onTrackerSuggestionClick()` (arm-to-confirm),
  reset on project/family/action change, populated from `executeResult`'s
  `trackerSuggestion` field, and a new `adoWorkItemCreateResult` case (status
  bar only — independent of `panels.js`'s own handler for the same message
  type, which continues to drive its ADO workflow panel).
- `apps/cli/static/launcher/launcher.css`: minimal `.tracker-suggestion`
  card styles reusing the existing `--ax-*` custom properties.
- No changes to `transport.js` (the `adoWorkItemCreate` case already existed),
  `@axiom/tracker`, `@axiom/tracker-ado`, or any `run*Subcommand`/
  `executeSubcommand` lifecycle code.

## Validation

Executed from `C:/repos/Axiom Workspace/Axiom`:
- `npm run build` (tsc -b, full monorepo) — clean, no errors.
- `node --check apps/cli/static/launcher/launcher.js` (+ `transport.js`,
  `panels.js`, unchanged but re-checked) — all pass.
- `npx vitest run apps/cli/tests/launcher-ado-bridge.test.ts` — 16/16 new tests
  green (`resolveLauncherTrackerStatus` incl. never-throws + real/disabled/
  incomplete ado configs; `buildTrackerSuggestion` mapping/fallbacks/undefined
  cases; `POST /launcher/execute` `trackerSuggestion` wiring over a real HTTP
  server, offline throughout).
- `npx vitest run apps/cli/tests -t launcher` — 55/55 passed (7 files: the new
  file plus `app-launcher`, `launcher-onboarding`, `launcher-push`,
  `launcher-doctor`, `launcher-front-no-vscode`, `launcher-panels`), 0 failed.

See the orchestrator's report for verbatim command tails.

## Result

Implemented as scoped: a purely additive `trackerSuggestion` on successful
increment/bug creates, surfaced by a new front card that offers a one-click,
confirmed-gated Azure DevOps work-item create via the already-shipped
`apiAdoCreateWorkItem` endpoint and transport message. No core lifecycle code,
tracker packages, or existing ADO create path were touched; no new npm
dependencies or new transport messages were introduced.

## General spec integration

Integrado por el pase final de la orquestación (autopilot, 2026-07-15):
- `01_Requisitos_Funcionales.md` → **RF-AXM-032 Puente Azure DevOps en creación desde el launcher**.
- `06_Integraciones_y_Capacidades.md` → sección "tanda INC-20260715-*" (`trackerSuggestion` + detección network-free `_tracker-status.ts`, sin acoplar el ciclo de vida).
- `07_Gobierno_y_Seguridad.md` → nota "PAT nunca en el repo" + sin arquitectura especulativa (no auto-crea).
- `05_Interfaces_Operativas.md` → bullet "puente ADO en creación" en la sección de la tanda.
- `08_Glosario.md` → término "`trackerSuggestion` (puente ADO en creación)".

Archivado en `Axiom.Spec/specs/increments/_archive/`.
