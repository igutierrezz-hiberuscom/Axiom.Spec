# BUG-20260727-adopt-telemetry-sinks-warn

Status: closed

## Expected behavior

After `axiom workspace setup` (adoption) or `axiom init`, running any CLI command in the new
project should not emit a noisy `WARN` about a missing `axiom.config/telemetry-sinks.yaml` on a
local-only project. Either the file is scaffolded with a safe default, or its absence is treated
as the (silent) local-only default.

## Actual behavior (found in E2E, 2026-07-27)

Every CLI invocation in the freshly-adopted `kvp25-e2e.axiom` project prints:

```
[axiom] WARN: loadEnabledSinks falló (missing-file: No se pudo leer telemetry-sinks.yaml en
<proj>\axiom.config\telemetry-sinks.yaml: ENOENT ...); continuando con bus vacío.
```

`axiom workspace setup` scaffolds the rest of `axiom.config/` (topology, mcp-manifest,
toolchain-catalog, skills-index, profiles, policy-as-code, integrations, agents-catalog) but NOT
`telemetry-sinks.yaml`, and `loadEnabledSinks` logs a `WARN` (not a debug/info) when the file is
absent. It is non-fatal (bus continues empty) but noisy on every command.

## Repro

1. `axiom workspace setup --name X --adopt-spec <legacy> --control-path <dest> ...`
2. `cd <dest> && axiom index validate` (or any command) → the WARN appears.

## Scope / non-goals

- Scope: silence the WARN for the legitimate local-only "no sinks declared" case (log at debug),
  OR scaffold a default `telemetry-sinks.yaml` in the setup/init path.
- Non-goal: no change to telemetry semantics — the empty-bus fallback is correct; only the log
  level / scaffolding gap is the defect.

## Notes

Found by the 2026-07-27 end-to-end adoption test (KVP25 → kvp25-e2e.axiom).

## Fix (closed 2026-07-27)

Chosen approach: treat a **missing** file as the legitimate local-only default (silent), warn only
on a real config error. Nothing scaffolds `telemetry-sinks.yaml`, and `loadEnabledSinks` returns a
distinct `missing-file` error kind (vs `invalid-yaml`/`malformed-shape`). The two callers now skip
the WARN when `error.kind === 'missing-file'`:
- `apps/cli/src/index.ts` (`installTelemetryBus`, per-command global bus install).
- `apps/cli/src/commands/sync.ts` (gate sink load).
`audit.ts` already handled a missing file silently (no change).

**Validation:** `tsc -b` clean; existing `loader.test.ts` (`missing-file` err) + `sync.test.ts`
green; live re-check — `axiom index validate` in the adopted `kvp25-e2e.axiom` project no longer
prints the telemetry WARN. Genuine config errors (`invalid-yaml`/`malformed-shape`) still warn.

