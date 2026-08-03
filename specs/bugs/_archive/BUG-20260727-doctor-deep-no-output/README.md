# Bug: axiom doctor --deep produces no output and exits 0

Status: closed
Date: 2026-07-27

## Symptom

`axiom doctor --deep` (and `axiom doctor --deep --json`) prints nothing to
stdout and exits with code 0, even in a project where the sync `axiom
doctor` correctly reports failures. Only a stderr telemetry `WARN` line
(unrelated, printed at CLI boot regardless of the subcommand) is visible.

## Current behavior

Before the fix: `axiom doctor --deep` silently abandons the doctor report.
The CLI process exits early (code 0) before the `await
runDoctorChecksDeep(...)` continuation in `apps/cli/src/index.ts`'s
`doctor` action ever writes the report to stdout.

## Expected behavior

`axiom doctor --deep` must print the SAME report as sync `axiom doctor`
PLUS the opt-in runtime probes (TC-018 toolchain functional probe,
TC-019 MCP server liveness handshake). `--deep --json` must emit valid
JSON including those checks. Exit code must follow `summary.failed`
(deep probes never produce `fail`, so `--deep`'s exit code always equals
sync doctor's exit code for the same project). In a sandbox where the
declared MCP server's launch `command` (e.g. `axiom`) is not on PATH,
TC-019 must resolve to `warn` (never crash, never hang, never silently
drop the whole run) and the report must still print in full.

## Impact

`axiom doctor --deep` was completely non-functional whenever any declared
MCP server's launch command failed to spawn (ENOENT) — the single most
common real-world case (a project adopted/cloned onto a machine where
`axiom` isn't yet on PATH, exactly the sandbox condition that surfaced
this bug). Any CI/tooling depending on `--deep`'s exit code or JSON output
silently got nothing instead of a report.

## Reproduction steps

```
cd "C:/repos/_pruebas/KVP25-copia-1/KVP25.axiom"
node "C:/repos/Axiom Workspace/Axiom/apps/cli/dist/index.js" doctor            # prints the full 52-check report, exit 1
node "C:/repos/Axiom Workspace/Axiom/apps/cli/dist/index.js" doctor --deep     # (before fix) prints ONLY the telemetry WARN, no report, exit 0
node "C:/repos/Axiom Workspace/Axiom/apps/cli/dist/index.js" doctor --deep --json  # (before fix) empty stdout
```

Minimal isolation (no CLI, no telemetry noise) that proves the exact
mechanism, run from the same sandbox project:

```
node --input-type=module -e "
import { resolveProject } from 'file:///.../packages/project-resolution/dist/index.js';
import { runMcpServerLivenessCheck } from 'file:///.../packages/doctor/dist/deep-checks.js';
const resolution = resolveProject(process.cwd());
const checks = await runMcpServerLivenessCheck(resolution);
console.error(checks);
"
```

This prints `Warning: Detected unsettled top-level await` and exits with
code 13 — proving the check's own promise NEVER settles (it is not merely
"not awaited by the CLI"; it genuinely hangs forever unless something
forces the process to end first).

## Acceptance criteria

- [x] `axiom doctor --deep` prints the full report (sync checks + TC-018 +
      TC-019), never just the telemetry boot `WARN`.
- [x] `axiom doctor --deep --json` emits valid, parseable JSON including
      the runtime-probes checks.
- [x] `--deep`'s exit code equals sync doctor's exit code for the same
      project (deep probes never change `summary.failed`).
- [x] In a sandbox where the declared MCP server's `command` is not on
      PATH, TC-019 resolves to `warn` (not a crash, not a hang) and the
      report still prints in full.
- [x] Sync `axiom doctor` and other CLI commands (`--help`, `init --help`)
      are unaffected.
- [x] A regression test exists that fails without the fix and passes with it.

## Suspected cause

Two hypotheses were listed for investigation: (a) `apps/cli/src/index.ts`
uses `program.parse(process.argv)` instead of `program.parseAsync(...)`,
so nothing awaits the async `doctor --deep` action; (b) an unref'd/early
exit inside the deep/MCP probe path. (a) turned out to be a red herring —
other async CLI commands work fine under plain `.parse()` because Node's
event loop naturally keeps running as long as any real handle (timer,
socket, child process) is pending, regardless of whether the top-level
caller awaits the returned promise. (b) is the confirmed root cause, see
below.

## Root cause (confirmed)

`packages/providers/src/stdio-mcp-client.ts`'s `createStdioMcpClient(...).close()`
used an **unref'd** 2-second safety-net timer as its ONLY fallback path to
resolve the `close()` promise:

```ts
setTimeout(finish, 2000).unref?.();
```

When the declared MCP server's `command` (e.g. `axiom`) is not on PATH,
`child_process.spawn` fails with `ENOENT`. On this platform/Node version,
an ENOENT spawn failure emits the child's `'error'` event exactly once and
**never** emits `'exit'` (proven with an isolated `spawn()` repro). By the
time `defaultMcpProbeFn`'s `finally` block calls `client.close()`, that
one-time `'error'` event has already fired and been consumed — so
`close()`'s freshly-attached `.once('error', finish)` /
`.once('exit', finish)` listeners will never fire again. The **only**
remaining way to resolve the `close()` promise is the safety-net
`setTimeout`. Because that timer was `.unref()`'d, Node considers it "not
a reason to keep the process alive" — and since, in the real CLI, this is
the LAST pending handle in the whole process by that point, Node drops
the timer and exits the process (code 0, no error, no warning in CommonJS)
**before the timer ever fires**, permanently abandoning the still-pending
`close()` promise and, transitively, `runMcpServerLivenessCheck`'s
`Promise.all(...)`, `runDoctorChecksDeep(...)`'s `await`, and the CLI
action's continuation that would have written the report.

Confirmed via:
1. Direct ESM repro of `runMcpServerLivenessCheck(resolution)` in the
   sandbox project → Node's "Detected unsettled top-level await" diagnostic
   (exit 13), proving the promise never settles at all (not merely
   "abandoned by a non-awaiting caller").
2. A minimal `spawn('nonexistent-binary', ...)` + `.once('exit', ...)`
   repro showing `'exit'` never fires for an ENOENT spawn failure — only
   `'error'` does, and only once.
3. Reverting the fix and re-running the new regression tests (below):
   all 4 fail exactly as expected (empty stdout / exit 0 instead of the
   report / exit matching sync).

## Fix

`packages/providers/src/stdio-mcp-client.ts`, `close()`:

Before:
```ts
await new Promise<void>((resolve) => {
  let settled = false;
  const finish = (): void => {
    if (settled) return;
    settled = true;
    resolve();
  };
  activeChild.once('exit', finish);
  activeChild.once('error', finish);
  try { activeChild.stdin.end(); } catch { /* ignore */ }
  try { activeChild.kill(); } catch { /* ignore */ }
  // Safety net: if the process ignores the signal, do not hang the
  // caller forever waiting for `exit`.
  setTimeout(finish, 2000).unref?.();
});
```

After:
```ts
await new Promise<void>((resolve) => {
  let settled = false;
  let safetyTimer: NodeJS.Timeout;
  const finish = (): void => {
    if (settled) return;
    settled = true;
    clearTimeout(safetyTimer);
    resolve();
  };
  activeChild.once('exit', finish);
  activeChild.once('error', finish);
  try { activeChild.stdin.end(); } catch { /* ignore */ }
  try { activeChild.kill(); } catch { /* ignore */ }
  // Safety net: NOT `.unref()`'d — this timer must be able to keep the
  // event loop alive on its own when it is the LAST pending handle (a
  // `spawn` ENOENT failure never emits `exit`). `finish()` clears the
  // timer on the normal `exit`/`error` path, so a healthy close never
  // waits the full 2s.
  safetyTimer = setTimeout(finish, 2000);
});
```

Removing `.unref()` alone would reintroduce a different regression (an
already-resolved `close()` leaving a live 2s timer behind, delaying
process exit on the healthy path); adding `clearTimeout(safetyTimer)`
inside `finish()` is what keeps the happy path exit-time unchanged.

## Scope

- Changed: `Axiom/packages/providers/src/stdio-mcp-client.ts` (`close()` only).
- No changes to `apps/cli/src/index.ts`'s `program.parse` — investigated
  and ruled out as the root cause; changing it would not have fixed the
  underlying hang (a promise stuck on an unref'd-timer-as-last-handle
  never settles regardless of whether the top-level caller awaits it).
- No changes to `packages/doctor` (TC-018/TC-019 check logic was already
  correct — it correctly never produces `fail`, and correctly reports
  `warn` with the real ENOENT reason once `close()` actually resolves).

## Validation

`npm run build` (from `Axiom/`) — passes, no TypeScript errors.

Sandbox repro (`C:\repos\_pruebas\KVP25-copia-1\KVP25.axiom`), before/after:
- `doctor` (sync): unchanged, 52 checks, exit 1 (`Resultado: FAIL`).
- `doctor --deep`: before → empty stdout, exit 0. After → full 55-check
  report (52 sync + TC-018 pass + 2x TC-019 warn with `spawn axiom ENOENT`
  reasons), exit 1, same `failed` count as sync (6).
- `doctor --deep --json`: before → empty stdout. After → valid JSON
  (parsed successfully), `checks.length === 55`, includes
  `TC-018`/`TC-019-sdd-mcp-server`/`TC-019-spec-mcp-broker`.
- Spot-checked `--help` and `init --help` still exit 0 (no regression to
  other sync/async commands).

Regression tests added (all spawn either an isolated Node subprocess or
the real built CLI as a child process — the bug only reproduces when
nothing else keeps the event loop alive, which an in-process vitest run
cannot simulate):
1. `Axiom/packages/providers/tests/stdio-mcp-client.test.ts` — new test
   spawns an isolated `node` subprocess that requires the built
   `dist/stdio-mcp-client.js`, creates a client against a deterministically
   nonexistent command (`axiom-definitely-not-a-real-local-binary-xyz`,
   the codebase's own established ENOENT-simulation convention), and
   asserts the subprocess prints a `CLOSED` marker and exits 0 within a
   bounded 8s timeout (well above the fix's 2s internal safety net).
   Skips gracefully (matching the existing `distAvailable` convention)
   if the package hasn't been built yet.
2. `Axiom/packages/doctor/tests/deep-checks.test.ts` — new test asserts
   `runDoctorChecksDeep(...)`'s report feeds `formatReport`/`JSON.stringify`
   to a non-empty, well-formed result including the TC-018/TC-019 ids
   (the doctor-package "at least" fallback).
3. `Axiom/apps/cli/tests/e2e/doctor-deep.e2e.test.ts` (new file) — 3 tests
   spawn the real built CLI (`dist/index.js`) against a minimal fixture
   project declaring the two managed MCP servers with the same
   deterministically-nonexistent command, asserting: (a) `doctor --deep`
   prints a non-empty report containing TC-018/TC-019/`ENOENT`/`Resultado:`;
   (b) `--deep --json` parses as valid, non-empty JSON with the
   runtime-probes checks; (c) `--deep`'s exit code equals sync doctor's
   exit code. Guarded by a `distAvailable` check (skips with a console
   warning if `npm run build` hasn't run), mirroring
   `apps/cli/tests/e2e/workspace-mcp.e2e.test.ts`'s established pattern.

Confirmed the tests are real regressions (not vacuous): temporarily
reverted the fix and reran — all 4 new tests failed with exactly the
expected symptoms (empty stdout / exit 0 / exit-code mismatch); restored
the fix and reran — all pass.

Full targeted suite after the fix: `npx vitest run packages/doctor
packages/providers apps/cli/tests/e2e/doctor-deep.e2e.test.ts` → 33 test
files, 307 tests, all passing.

## Result

Fixed. `axiom doctor --deep` and `axiom doctor --deep --json` now reliably
print the full report (sync checks + TC-018 + TC-019) in every case,
including the exact sandbox condition that surfaced the bug (declared MCP
server `command` not on PATH). Exit code semantics are unchanged and
verified to match sync doctor. No regression observed in sync doctor or
other CLI commands (`--help`, `init --help` spot-checked; full targeted
suite green).

## General spec integration

None. This task's explicit instructions scope canonical spec edits to this
bug file only ("Do NOT edit specs/00..08") — this repository's
`Axiom.Spec/specs/` uses the numbered `00_..08_` files in place of a single
`general-spec.md`, and those are out of scope for this fix. The durable,
reusable knowledge worth consolidating there in a future pass — "any local
process-spawning helper's cleanup/close path must not rely on an
`.unref()`'d timer as its only fallback settlement path, since an unref'd
timer is dropped the instant it becomes the last pending handle in the
process" — is recorded here for a maintainer to fold into
`06_Integraciones_y_Capacidades.md` (or equivalent) in a dedicated pass,
scoped to spec-repo edits only.
