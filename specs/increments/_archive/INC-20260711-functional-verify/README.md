# Increment: Functional verify per role/increment

> **Código**: INC-20260711-functional-verify
> **Estado**: Implementado y gate-verde (build + suite vitest completa + typecheck + doctor PASS).
> **Fecha**: 2026-07-12
> **Tipo de cambio**: Nueva capacidad (verify funcional, no sólo transición de estado)
> **Epic padre**: `INC-20260711-sdd-launcher-core-port`

---

## 1. Goal

Turn the SDD **verify** step from a pure STATE TRANSITION into a FUNCTIONAL gate.
Today `axiom-increment verify` (plan-approved → verifying) and `axiom-role
complete` only move the workflow state; they never run the project's own
tests/build/lint, and `axiom-qa-e2e` is inline-noop by default. This increment
wires functional verification: **discover** the target repo's validation commands
(the AGENTS.md discovery order), **run** them via an injectable command runner,
and **GATE** the transition on the result.

## 2. Scope

### Incluido

- **Validation discovery + runner** (`apps/cli/src/commands/_functional-verify.ts`):
  discovers a repo's validation commands (README hints → `package.json` scripts →
  task runners/Makefile) and runs them through an injectable spawn-based
  `ValidationRunner` seam (mirrors `GitRunner`). No command found → the exact
  AGENTS.md best-effort statement, non-blocking.
- **`axiom-increment verify`** — runs the discovered validation for the repo(s)
  the ACTIVE plan targets; on failure BLOCKS the transition (exit 1, names the
  failing command) unless `--no-verify`/`--force`. `--preview`/`--dry-run`
  reports what would run without running or transitioning.
- **`axiom-role complete`** — runs the ROLE repo's validation as part of
  completion, composed with the existing per-role write-scope review; a failure
  blocks completion. Bypass with `--no-verify` (or `--force`, which also bypasses
  the review).
- **`axiom-qa-e2e verify --run-validation`** — the QA lane can run a real
  validation; inline-noop remains the default (explicit opt-out).

### Excluido (Non-goals)

- No new command runner is invented where one already exists in spirit — the
  `ValidationRunner` mirrors `GitRunner`/`GitDiffRunner` (`spawnSync` + injection).
- No change to the DEFAULT behavior of existing lifecycle steps beyond the gate:
  the functional run is detection-gated (only runs when a command is discovered),
  so fixture repos with no test script behave exactly as before.
- No auto-fix / no writing to the target repo. Verify is read-only except for
  whatever the project's own validation command does.
- No MCP surface in this increment (the CLI gate is the deliverable).

## 3. Validation model

### 3.1 Discovery order (AGENTS.md)

The discovery order mirrors AGENTS.md "Validation Rules":

1. **README hints** — validation commands declared in `README(.md)` inside code
   spans/fences (`npm (run) test|build|lint|typecheck`, `pnpm …`, `yarn …`,
   `make test|build|lint|check`).
2. **package scripts** — `package.json` `scripts` keys `test` / `build` / `lint`
   / `typecheck`, run as `npm run <script>`.
3. **task runners** — a `Makefile` with `test` / `build` / `lint` / `check`
   targets, run as `make <target>`.
4. **test/build configs** — presence-only (a config with no runnable script is
   NOT a command); contributes to the no-command fallback if nothing above is
   found.

The FIRST source that yields commands wins (highest priority). Discovery returns
the ordered list of commands to run.

### 3.2 What gates what

- **`axiom-increment verify`**: resolves the ACTIVE approved plan's `targetRepos`
  to absolute paths (topology + `LocalBindings`, the same primitives as
  `validate changes --all-repos`), runs the discovered validation in each, and
  aggregates a per-repo report. Any RESOLVED repo whose validation FAILS →
  transition BLOCKED (exit 1). Unresolvable target repos are REPORTED (never
  silently skipped) but do not block. No active plan / no targetRepos / topology
  unresolved → skip (non-blocking) — this keeps single-repo/dogfooding green.
- **`axiom-role complete`**: runs the ROLE repo's own validation (the role repo
  is `projectRoot`). Fail → completion blocked (state stays `in-progress`).
  Composed AFTER the write-scope review (both must pass).
- **`axiom-qa-e2e verify --run-validation`**: runs validation in the repo and
  gates exit code; without the flag the lane keeps its current inline-noop /
  parallel-transition behavior.

### 3.3 No-command fallback (exact statement)

When discovery finds no validation command, the runner emits AGENTS.md's EXACT
statement and treats it as a **non-blocking pass-with-note**:

```
No validation command was found. Performed best-effort validation by inspecting changed files and checking consistency against the requested behavior.
```

### 3.4 Bypass + preview

- `--no-verify` (and `--force`) skip the functional run entirely (proceed with a
  warning). At `axiom-role complete`, `--force` bypasses BOTH the write-scope
  review and the functional verify.
- `--preview` / `--dry-run` on `axiom-increment verify` reports the discovered
  validation per target repo WITHOUT running it and WITHOUT transitioning.

### 3.5 Safety

- The `ValidationRunner` always runs with `cwd = <resolved target repo root>`,
  never `process.cwd()` implicitly.
- Unit tests inject a FAKE `ValidationRunner` that records the requested command
  and returns a scripted result — no real sub-build ever runs in the suite. The
  real spawn seam is proven only against a lightweight `node <script>` in a
  throwaway temp dir.
- Detection-gated: no discovered command → no spawn.

## 4. Acceptance

- **AC1 (discovery)**: picks up `package.json` `test`/`build`/`lint`/`typecheck`
  scripts; README-declared commands take precedence; a `Makefile` provides
  targets; nothing found → the exact AGENTS.md fallback statement (non-blocking).
- **AC2 (`axiom-increment verify`)**: fake runner returns failure → transition
  BLOCKED (exit 1, names the failing command); success → proceeds; `--no-verify`
  → proceeds with a warning; `--preview` → reports without running/transitioning.
- **AC3 (`axiom-role complete`)**: role-repo validation failure → completion
  blocked (stays `in-progress`); success + write-scope OK → completes.
- **AC4 (multi-repo)**: per-repo validation results are aggregated; an
  unresolvable target repo is reported, not silently skipped.
- **AC5 (green)**: existing lifecycle tests stay green (detection-gated + injected
  runner, no real build spawned). A real `node <script>` run proves the spawn
  seam gates pass/fail.
- **AC6 (gate)**: build + full vitest suite + typecheck + doctor all green.

## 5. Result

Implemented and gate-green. See `metadata.yml` decisions D-FV-001..005.

- **Discovery + runner seam**: `apps/cli/src/commands/_functional-verify.ts` —
  `discoverValidation` (README → package scripts → Makefile), the injectable
  `ValidationRunner` (`createValidationRunner` via `spawnSync`), `runFunctionalVerify`
  (detection-gated; `pass` | `fail` | `no-command` | `preview`), and the
  multi-repo resolver/aggregator `runIncrementFunctionalVerify` (reuses
  `resolveRepoPath` + `loadLocalBindings` + `loadActivePlan`). The exact AGENTS.md
  fallback lives in `NO_VALIDATION_COMMAND_FALLBACK`.
- **Gated transitions**: `axiom-increment verify` (blocks on failure, `--no-verify`
  bypass, `--preview`), `axiom-role complete` (role-repo verify composed with the
  write-scope review, `--no-verify`/`--force` bypass), `axiom-qa-e2e verify
  --run-validation` (real-run opt-in; inline-noop default preserved).
- **Existing tests stayed green**: the functional run is detection-gated (no
  discovered command → no spawn) and every unit/integration test injects a fake
  `ValidationRunner`; fixture lifecycle repos have no package.json/README/Makefile
  so they short-circuit to `no-command` and never build.

## 6. Trazabilidad

- Parent epic: `INC-20260711-sdd-launcher-core-port` (functional verify slice).
- Siblings: `INC-20260711-git-services` (the injectable command-runner seam +
  declared side-effect model reused in spirit), `INC-20260711-per-role-review`
  (the write-scope review functional verify composes with), and
  `INC-20260711-cross-repo-mcp-wiring` (cross-repo plan/topology resolution).
- Canonical validation-discovery order: `Axiom.SDD/AGENTS.md` "Validation Rules".
