# INC-20260727-cli-doctor-ado-check

Status: closed

## Goal

Make the CLI `axiom doctor` surface the ADO connectivity check too — not just the launcher's
Doctor panel — so both entry points validate the ADO plugin (PAT / organization / project) when
it is the active tracker. The user's expectation: "the doctor is the same, the launcher just
calls the CLI".

## Context / finding

Both `axiom doctor` (`apps/cli/src/index.ts`) and the launcher (`apiGetLauncherDoctor`) call the
SAME `runDoctorChecks` / `runDoctorChecksDeep` from `@axiom/doctor`. The ADO probe
(`apiLauncherAdoDoctorCheck`, a real `tags.listTags()` authenticated read) is spliced into the
response **only** in the launcher dispatch (`app-api.ts`), so `axiom doctor` never runs it.
`@axiom/doctor` deliberately has NO tracker dependency (its sync checks are offline; network
work belongs to the opt-in probes). So the fix must be injected at the CLI layer (where
`@axiom/tracker-ado` is already available), mirroring the launcher pattern — NOT inside
`@axiom/doctor`.

## Scope

- Factor the ADO probe into a shared `projectRoot`-based helper
  `runAdoConnectivityDoctorCheck(projectRoot, deps)` returning a `@axiom/doctor`-compatible
  check object (`{ id, category, description, status, evidence? }`) or `null` when ADO is NOT
  the active tracker (local-only projects → no check, no network).
- `apiLauncherAdoDoctorCheck` becomes a thin wrapper over the shared helper (launcher dispatch
  unchanged in behavior).
- CLI `axiom doctor` (`index.ts`): after building the report, call the shared helper with the
  resolved project root; if non-null, append it to `report.checks` and recompute the summary,
  then print (text + `--json`) and honor exit code. Because the helper returns `null` unless
  ADO is configured, plain `axiom doctor` stays offline for local-only projects and CI.

## Non-goals

- No `@axiom/tracker` / `@axiom/tracker-ado` dependency added to `packages/doctor`.
- No change to the offline sync checks or the existing deep probes.

## Acceptance criteria

- For a project with `tracker.kind: ado` (enabled, org, project, PAT), `axiom doctor` shows an
  `ado-connectivity` check (pass when reachable, fail with evidence otherwise).
- For a local-only project (NullTracker), `axiom doctor` output is unchanged (no ADO check, no
  network) — existing doctor tests still pass.
- The launcher Doctor panel keeps showing the same check (shared helper).
- Build + unit tests green.

## Result (closed 2026-07-27)

Factored `runAdoConnectivityDoctorCheck(projectRoot, deps)` in `app-launcher-ado.ts`
(`apiLauncherAdoDoctorCheck` now delegates to it); the CLI `axiom doctor` (`apps/cli/src/index.ts`)
appends the check to the report + recomputes the summary. No tracker dependency added to
`@axiom/doctor`. **Validation:** `tsc -b` clean; unit test of the shared helper (null local / pass
configured) in `launcher-ado-peripherals.test.ts`; existing `launcher-doctor.test.ts` unchanged.
**Real E2E (2026-07-27):** `axiom doctor --deep` against the adopted KVP25 project returned
`ado-connectivity: pass` (real KVPHiberus/KVP25 + `ADO_PAT_KVP25`), 56 checks / 0 fail. Integrated
into `06_Integraciones_y_Capacidades.md`.

