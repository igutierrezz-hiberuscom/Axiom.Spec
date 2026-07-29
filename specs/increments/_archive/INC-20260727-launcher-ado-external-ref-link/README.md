# INC-20260727-launcher-ado-external-ref-link

Status: closed

## Goal

When the launcher creates an Azure DevOps work item for a just-created increment/bug
(the tracker-suggestion "Crear work item en Azure DevOps" flow), persist the resulting
external reference — **ADO id + URL** — into the artifact's local `metadata.yml`
`externalRefs`, so the local spec records which external work item it maps to.

## Context / finding (review of the user's question)

`metadata.yml` already carries `externalRefs: ReadonlyArray<{ provider, type, id, url? }>`
(`packages/workflow/src/artifact-store.ts`). The field **supports id + url**. But:
- The increment/bug **create** path hardcodes `externalRefs: []` and never contacts the tracker.
- The launcher's `apiAdoCreateWorkItem` **creates** the work item and **returns** its id+url to
  the front, but does NOT persist them locally.
- Persisting is only done by a **separate explicit** action: `apiAdoLinkAxiom` / the
  `external-ref add` CLI subcommand → `runExternalRefAdd`.

So today the external id+url are recorded in local metadata **only if the user also runs the
explicit link step**. This increment closes that loop for the launcher create flow.

## Scope

- `apiExecuteLauncherAction` captures the created artifact folder id
  (`exec.result.record.vars['metadataId']`) and passes it to `buildTrackerSuggestion`.
- `buildTrackerSuggestion` includes `artifactId` (the metadataId) in the returned suggestion.
- Front (`onTrackerSuggestionClick`) forwards `artifactKind` + `artifactId` in the
  `adoWorkItemCreate` message values.
- `apiAdoCreateWorkItem` accepts optional `artifactKind` + `artifactId`; on a **confirmed,
  successful** create with both present, it calls the EXISTING `runExternalRefAdd` with the
  created ref's `provider:type:id:url` to persist the external ref. Best-effort: a link failure
  never fails the create; the response reports the link outcome.

## Non-goals

- No change to the increment/bug **create** path (still emits `externalRefs: []`; linking stays
  a post-create step).
- No auto-link for the generic ADO-workflow "Crear work item" form (not tied to an artifact —
  it sends no `artifactId`, so the persist path no-ops).
- No new endpoint (reuses `runExternalRefAdd`).

## Acceptance criteria

- After a confirmed launcher increment/bug create + "Crear work item en Azure DevOps", the
  artifact's `metadata.yml` `externalRefs` contains an entry `{ provider: 'azure-devops',
  type, id: <WI id>, url: <WI url> }`.
- Creating a WI via the generic ADO-workflow form (no artifactId) still works and writes no ref.
- A link failure (e.g. artifact folder missing) is reported but does not fail the WI create.
- Build + unit tests green.

## Result (closed 2026-07-27)

Implemented in `apps/cli/src/commands/app-launcher.ts` (`apiExecuteLauncherAction` captures
`record.vars.metadataId`; `buildTrackerSuggestion` + `TrackerSuggestion.artifactId`;
`TrackerSuggestionWorkItem`), `app-launcher-ado.ts` (`apiAdoCreateWorkItem` persists the ref via
`runExternalRefAdd` when `artifactKind`+`artifactId` present; `repro` added to the body), and the
launcher front (`onTrackerSuggestionClick` forwards artifact context; result handler reports the
local link). **Validation:** `tsc -b` clean; new tests in `launcher-ado-workflow.test.ts`
(auto-link persists id+url into `metadata.yml`; generic form → no link) + `launcher-ado-bridge.test.ts`
(artifactId threaded); full launcher/tracker suites green. Integrated into
`06_Integraciones_y_Capacidades.md`.

