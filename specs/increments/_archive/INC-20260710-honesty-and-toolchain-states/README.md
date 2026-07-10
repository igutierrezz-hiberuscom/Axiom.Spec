# Increment: honesty and toolchain states (P1-8 + toolchain quality)

Status: closed
Date: 2026-07-10

## Goal

Three independent, small, audit-confirmed fixes: (1) make
`@axiom/orchestrator`'s "15 intent commands" claim honest in product
docs and decide whether the dead intent surface should be removed or
annotated; (2) make the toolchain's `present`/`absent` detection
honest by introducing differentiated states backed by a real,
best-effort probe; (3) fix the `upgradeFailed:` double-prefix bug in
`axiom upgrade`, the sibling of the already-fixed `gateFailure:` bug.

## Context

Audit findings from the brief, verified against the real code:

1. `@axiom/orchestrator`'s `STATE_MACHINE` (`packages/orchestrator/
   src/state-machine.ts`) declares 8 lifecycle commands + **19**
   (not 15 — the design.md always listed 19; `state-machine.test.ts`
   already documented "el spec/proposal subestimaron el conteo")
   `axiom-*-command` intent ids, all with a synthetic `notImplemented`
   precondition. `Axiom/README.md` described this package as
   "operativo" with "7 lifecycle + 15 intent commands" — stale on
   both counts (8 lifecycle since 0019-A, 19 intent always).
2. **Correction to the brief's own premise**: the brief also named
   `apps/cli/src/commands/intent.ts` (`axiom intent`) as "surfacing"
   the orchestrator's not-implemented intents. That is factually
   inaccurate and conflates two UNRELATED things:
   - `intent.ts`'s `INTENT_CHAINS` declares only **3** intents
     (`increment-new`, `bug-new`, `implement-role`), not 15/19, and
     its ids (`'increment-new'`) don't even match the orchestrator's
     `CommandId` ids (`'axiom-increment-new-command'`).
   - `runIntentCommand` (`intent.ts`) never calls
     `runOrchestrated`/`gateFor`/`runCommand` at all — it chains REAL
     sub-commands directly (`runIncrementSubcommand`/
     `runBugSubcommand`/`runRoleSubcommand`).
   - There is no `axiom intent` CLI command: `intent.ts` has no
     `registerIntent` function and is never imported by
     `apps/cli/src/index.ts` (confirmed via repo-wide grep). It is
     ONLY imported by its own test (`intent.test.ts`), and that test
     only exercises the static `INTENT_CHAINS` catalog — never
     `runIntentCommand` itself.
   - The orchestrator's 19 intent `CommandId`s ARE exercised, but
     ONLY by the orchestrator package's own tests
     (`gates.test.ts`/`runner.test.ts`/`state-machine.test.ts`) — no
     real CLI command ever passes one of them to `runOrchestrated`.
3. `detectToolState` (`packages/toolchain/src/detect.ts`) was
   `fs.existsSync` on a marker path, reported as `'present'`.
   `repairTool` (`repair.ts:~73-113`) creates an EMPTY marker
   directory when a tool is `'absent'`, so `axiom toolchain
   validate` could report a required tool as fully satisfied with
   no real binary installed anywhere. `instruction-only` tools were
   ALSO hardcoded to `'present'` in `repair.ts` — a second, separate
   instance of the same dishonesty.
4. `apps/cli/src/commands/upgrade.ts`'s `formatUpgradeErrorLine` had
   the exact same double-prefix bug for `upgradeFailed:` that was
   already fixed for `gateFailure:` in
   `INC-20260710-lifecycle-correctness-fixes` — flagged there as a
   known, out-of-scope sibling bug, never fixed.

## Scope

**Part 1 — Orchestrator honesty:**
- `Axiom/README.md`: corrected the orchestration-layer row and the
  `packages/orchestrator` table row (8 lifecycle, 19 intent, explicit
  "NOT the execution path" language, cross-reference to
  `packages/workflow` as the real execution path). Also strengthened
  the `packages/workflow` table row to state it IS the real
  execution path.
- `packages/orchestrator/README.md`: rewritten — states the real
  scope (7 of 8 lifecycle commands actually wired from `apps/cli`;
  `doctor-command` declared but not invoked by the real `axiom
  doctor`, a separate pre-existing gap, documented not fixed), and
  that the 19 intent ids are not the SDD workflow's execution path.
- `packages/orchestrator/src/state-machine.ts` /
  `packages/orchestrator/src/types.ts`: corrected stale "15
  intent"/"22 entries" comments to the accurate 19/27; added an
  explicit "not the execution path, confirmed by audit" annotation
  on `CommandId`, the intent-command block, and `notImplemented`'s
  `remediation` string.
- **Removed** (genuinely dead code): `apps/cli/src/commands/
  intent.ts` and `apps/cli/tests/intent.test.ts`. See "Dead-code
  decision" below.
- **Kept, annotated** (not removed): the orchestrator's 19 intent
  `CommandId` stubs in `state-machine.ts`/`types.ts`. See same
  section.

**Part 2 — Toolchain honest states:**
- `packages/toolchain/src/types.ts`: `ToolState` changed from
  `'present' | 'absent' | 'drift' | 'unknown-version'` to `'declared'
  | 'absent' | 'marker' | 'installed-working' | 'drift' |
  'unknown-version'` (full doc comment explains the honesty model).
  New `ToolchainFinding` variant `required-tool-not-verified`.
- `packages/toolchain/src/detect.ts`: `detectToolState` (sync,
  unchanged signature) now returns `declared` for
  `supportLevel: 'instruction-only'` tools (centralizing that special
  case, previously duplicated and hardcoded wrong in `repair.ts`),
  and `marker` (was `'present'`) when a `detectionPaths` entry
  exists on disk.
- `packages/toolchain/src/probe.ts` (**new**): best-effort, never-
  throwing, short-timeout (default 2s) REAL probe —
  `resolveProbeCommand`/`probeCommandVersion` (generic `spawn(cmd,
  ['--version'])`, mirrors the never-throw spawn discipline of
  `packages/providers/src/code-intel/shared.ts`/`stdio-mcp-client.ts`
  without the full MCP handshake), `hasRealCodegraphDb` (CodeGraph-
  specific: real non-empty `.codegraph/codegraph.db` fallback),
  `probeToolInstalled`, `detectToolStateWithProbe`/
  `detectAllToolsWithProbe` (upgrade `marker`/`absent` to
  `installed-working` ONLY on positive confirmation), and an
  injectable `ProbeFn` (mirrors the existing `mcpLaunchCommandOverride`
  pattern in `workspace-mcp.ts`/`member-install.ts`) for deterministic
  tests.
- `packages/toolchain/src/repair.ts`: updated to the new state names;
  the `instruction-only` early return now correctly reports
  `declared` (was hardcoded `present`).
- `packages/toolchain/src/validate.ts`: `validateToolchain` gained an
  OPTIONAL 6th param `detections?: ReadonlyMap<string, ToolDetection>`.
  Omitted (every pre-existing caller, incl. `@axiom/doctor`'s TC-004):
  100% unchanged behaviour (falls back to the cheap sync
  `detectToolState`). Supplied (the CLI's new default path): a
  required tool whose state is `marker`/`declared` now produces a
  `required-tool-not-verified` WARNING instead of being silently
  treated as fully satisfied. `ok` is unaffected by this warning —
  it stays `false` only for genuine `absent`.
- `apps/cli/src/commands/toolchain.ts`: `runToolchainShow`/
  `runToolchainValidate` are now `async` and run the real probe
  (`detectAllToolsWithProbe`) by default; `--json` for both now emits
  `{manifest, detections}` / `{validation, detections}` (previously
  bare manifest / bare validation) so the differentiated state is
  visible in JSON too. Both gained optional `probeTimeoutMs`/
  `probeFn` args (test-only injection points).
- `apps/cli/src/commands/app-api.ts`: `apiGetToolchain` is now
  `async` (follows `runToolchainShow`); its one call site
  (`handleApiRequest`'s `'projects.toolchain'` case, already an
  `async` function) now `await`s it.
- `apps/cli/src/commands/member-install.ts`: `axiomStateActivated`
  changed from `=== 'present'` to `!== 'absent'` (repair never runs
  the real probe, so this is `declared | absent | marker` here —
  the check means "some local state exists", not "verified working",
  which is exactly what this field always meant).

**Part 3 — Fast-follow:**
- `apps/cli/src/commands/upgrade.ts`: `formatUpgradeErrorLine`'s
  `upgradeFailed` branch no longer re-prepends `'upgradeFailed: '`
  onto a message that already starts with it (mirrors the
  `gateFailure` fix from `INC-20260710-lifecycle-correctness-fixes`
  verbatim — same helper, same root cause, same solution).
- `apps/cli/tests/upgrade.test.ts`: Scenario 2 updated from
  "documents the existing (buggy) doubled-prefix behaviour" to
  "asserts the single-prefix fix", mirroring Scenario 1's assertions.

## Non-goals

- Did **not** remove the orchestrator's 19 intent `CommandId` stubs
  (`state-machine.ts`/`types.ts`) — see "Dead-code decision" below
  for the full rationale. Only annotated them more clearly and fixed
  the stale counts.
- Did **not** touch `doctor-command`'s pre-existing gap (declared in
  `STATE_MACHINE` but the real `axiom doctor` CLI command never
  invokes the gate — it calls `@axiom/doctor` directly). Discovered
  during the audit, documented in both READMEs, but out of the
  brief's literal scope (which named the 15/19 intent commands, not
  the lifecycle ones) — flagged as a future consideration, not
  implemented.
- Did **not** touch `axiom.config/command-protocol.yaml` or
  `docs/configuration/files/command-protocol.md`. The doc file
  references a `command-protocol.yaml` that **does not exist
  anywhere in the repo** (confirmed via `find`) — a separate,
  pre-existing documentation/config drift, unrelated to this
  increment's two literal asks (README correction + intent dead-code
  decision). Flagged here as a discovered-but-out-of-scope finding.
- Did **not** touch `docs/0028-workflow-ux-and-archive-safety-
  completion.md` (the historical increment-completion doc that
  originally introduced `intent.ts`). It is an archival record of
  what was true when it was written; rewriting history there was not
  requested and risks contradicting AGENTS.md's "do not modify
  unrelated files".
- Did **not** add a `--probe`/`--no-probe` CLI flag to `axiom
  toolchain show`/`validate` — the probe runs by default (per the
  brief's own live-validation script, which invokes `show` with no
  extra flag and expects the differentiated state). `probeTimeoutMs`/
  `probeFn` exist only as programmatic options for tests, not
  exposed via commander.
- Did **not** change `validateToolchain`'s required-tool-missing
  (error) semantics — a bare `marker`/`declared` for a required tool
  still does not flip `ok` to `false`; it now surfaces as a WARNING
  (`required-tool-not-verified`) instead of a silent pass. Making it
  a hard failure would turn `@axiom/doctor`'s TC-004 red for
  legitimate, already-existing installations that were never wired
  to run a probe (out of scope, and arguably wrong given many tools
  have no automatable probe contract at all).
- Did **not** implement a real per-tool version-drift/`unknown-version`
  detector — those two `ToolState` values remain reserved/not yet
  produced by the built-in detector (pre-existing scope boundary,
  unchanged by this increment).

## Acceptance criteria

- [x] `Axiom/README.md` and `packages/orchestrator/README.md` state
      that `@axiom/orchestrator`'s intent commands are NOT the real
      SDD workflow execution path, and that the real path is
      `@axiom/workflow` via `axiom-increment/bug/plan/role`.
- [x] Investigated whether the 15(19) intent stubs + `axiom intent`
      are used by anything other than their own tests; made and
      documented the call (remove `intent.ts`, annotate the
      orchestrator stubs).
- [x] `axiom intent` does not exist in `axiom --help` (confirmed: it
      never did — `intent.ts` had no `registerIntent`); the dead
      module + its test were removed; `axiom --help` output has no
      `intent` entry.
- [x] The 7 lifecycle commands / gate that the real CLI DOES use
      remain intact — full suite green, zero regressions to
      `init`/`join`/`configure`/`sync`/`start`/`audit`/`upgrade`.
- [x] Toolchain: differentiated states `declared`/`absent`/`marker`/
      `installed-working` exist, backed by a real best-effort probe
      that never throws and degrades honestly.
- [x] `axiom toolchain show`/`validate` (and `--json`) surface the
      differentiated state; `validate` does not silently count a bare
      marker as satisfied for a required tool (surfaces
      `required-tool-not-verified`).
- [x] LIVE proof (scratch project, real compiled CLI): `toolchain add
      --id serena` → `show` → `absent` (nothing installed, no marker
      yet) → `toolchain repair` → `show` → `marker` (marker exists,
      still honestly NOT `installed-working`) → marking serena
      required → `validate` → `required-tool-not-verified` warning.
- [x] `upgradeFailed:` double-prefix fixed, mirroring the
      `gateFailure:` fix; test updated to assert the fix (not the
      old bug).
- [x] `npm run build` exits 0.
- [x] Targeted + full `npx vitest run` green, reconciled against the
      pre-increment baseline.
- [x] Status closed/pending with rationale — see "Result" below.

## Open questions

None blocking. Two discovered-but-out-of-scope findings are
documented under "Non-goals" for a possible future fast-follow:
`doctor-command`'s gate-bypass, and the phantom
`axiom.config/command-protocol.yaml` referenced by docs but absent
from the repo.

## Assumptions

- "Used by anything other than their own tests" (the brief's litmus
  test for the dead-code decision) was interpreted literally via a
  repo-wide grep for each symbol, not just a visual code read —
  confirmed for both `intent.ts` (zero non-test importers) and the
  orchestrator's 19 `CommandId`s (only the orchestrator package's
  own 3 test files touch them).
- The orchestrator's 19 intent stubs were judged "risky to remove for
  no real benefit" rather than "genuinely safe to remove", because
  removing them means rewriting/deleting real test coverage of the
  state machine's own declarative contract (`state-machine.test.ts`'s
  `INTENT_COMMANDS` list, `gates.test.ts`'s Scenario 4, `runner.test.
  ts`'s not-implemented test) for a change that has ZERO effect on
  any real CLI behavior (nothing consumes these ids either way) —
  this is a larger, riskier diff than the "trivial dead-code removal"
  framing implied, for a benefit (fewer type-union entries) that
  documentation alone already delivers (honesty was the actual ask).
- `validateToolchain`'s new `detections` param defaults to `undefined`
  (not an empty `Map`) specifically so `detectionFor`'s `??` fallback
  triggers per-tool, not just when the whole map is absent — this
  preserves the exact pre-existing behaviour for every caller that
  doesn't know about the new param.
- The real probe's tool dispatch (`serena --version`, `codegraph
  --version`, `python -m graphify --version`, `engram --version`) is
  a best-effort guess at each tool's actual CLI contract, grounded in
  what the codebase already documents for `serena`
  (`packages/providers/src/code-intel/serena-client.ts`) and `engram`
  (`packages/memory/src/engram-backend.ts`'s `ENGRAM_COMMAND`), but
  NOT independently verified against the `codegraph`/`graphify`
  binaries themselves (Axiom has never installed real third-party
  binaries — see `INC-20260710-per-member-install`'s Non-goals). If
  any of these four commands turns out not to support a bare
  `--version` flag, the probe still degrades honestly (never crashes,
  never falsely claims `installed-working`) — it would just always
  report `marker`/`absent` for that one tool until corrected, which
  is the same (safe) failure mode as "unknown tool kind".

## Implementation notes

Files changed (grouped, with the 1-line rationale each):

**Part 1 — Orchestrator honesty**
1. `Axiom/README.md` — corrected orchestration-layer + `packages/
   orchestrator` table rows; strengthened the `packages/workflow` row
   to state it's the real execution path.
2. `packages/orchestrator/README.md` — rewritten from scratch:
   accurate scope (7/8 lifecycle wired, `doctor-command` gap
   documented), explicit "NOT the execution path" for the 19 intent
   ids.
3. `packages/orchestrator/src/state-machine.ts` — corrected "15
   intent"/"22 entries" → "19 intent"/"27 entries" in 3 comment
   blocks; added the honesty annotation to the module header, the
   intent-command block, and `notImplemented`'s `remediation` text.
4. `packages/orchestrator/src/types.ts` — same correction on
   `CommandId`'s doc comment ("22 commands" → "27 (8 lifecycle + 19
   intent)"), with the full audit rationale for the annotate-not-
   remove decision.
5. **Removed**: `apps/cli/src/commands/intent.ts`,
   `apps/cli/tests/intent.test.ts` — confirmed dead (no
   `registerIntent`, never imported by `apps/cli/src/index.ts`, only
   consumed by its own test, which itself never exercised
   `runIntentCommand`).

**Part 2 — Toolchain honest states**
6. `packages/toolchain/src/types.ts` — `ToolState` differentiated
   model + `required-tool-not-verified` finding kind.
7. `packages/toolchain/src/detect.ts` — `detectToolState` returns
   `declared`/`marker`/`absent` (was `present`/`absent`); centralizes
   the `instruction-only` special case.
8. `packages/toolchain/src/probe.ts` (**new**) — the real,
   best-effort, injectable probe + `detectToolStateWithProbe`/
   `detectAllToolsWithProbe`.
9. `packages/toolchain/src/repair.ts` — updated state names; fixed
   the hardcoded-`present` instruction-only branch.
10. `packages/toolchain/src/validate.ts` — optional `detections`
    param + `required-tool-not-verified` warning branch.
11. `packages/toolchain/src/index.ts` — barrel-exported the new
    `probe.ts` surface.
12. `apps/cli/src/commands/toolchain.ts` — `runToolchainShow`/
    `runToolchainValidate` async + probe-backed; `--json` shape for
    both now includes `detections`.
13. `apps/cli/src/commands/app-api.ts` — `apiGetToolchain` async
    (follows `runToolchainShow`); one call site updated.
14. `apps/cli/src/commands/member-install.ts` — `axiomStateActivated`
    check updated (`!== 'absent'` instead of `=== 'present'`).
15. Tests: `packages/toolchain/tests/toolchain.test.ts` (renamed
    "present" test to "marker", +1 new test for `declared`);
    `apps/cli/tests/toolchain.test.ts` (Scenarios 1-3 made async,
    Scenario 2 updated to assert the new honest warning with an
    injected deterministic `probeFn`); `apps/cli/tests/
    toolchain-catalog-real.test.ts` (made async, injected `probeFn`).

**Part 3 — Fast-follow**
16. `apps/cli/src/commands/upgrade.ts` — `formatUpgradeErrorLine`'s
    `upgradeFailed` branch fixed (same pattern as `gateFailure`).
17. `apps/cli/tests/upgrade.test.ts` — Scenario 2 updated to assert
    the fix.

## Validation

- `cd Axiom && npm run build` (`tsc -b`): **exit 0**, clean, run
  twice (immediately after Part 1+2+3 source changes, and again
  after all test updates).
- Targeted `npx vitest run`:
  - `packages/toolchain`: **3 files / 28 tests, all green**
    (`p1-tools.test.ts` 4, `toolchain.test.ts` 12 [+1 new], `repair-
    add-gitignore.test.ts` 12).
  - `packages/orchestrator`: **4 files / 44 tests, all green**
    (`delegation.test.ts` 13, `state-machine.test.ts` 15,
    `gates.test.ts` 7, `runner.test.ts` 9 — unchanged; the
    `[orchestrator] FAIL: command="audit-command"` lines in stderr
    are the suite's own EXPECTED negative-path assertions, not
    failures).
  - `apps/cli`: **68 files / 667 tests, all green** (one fewer file
    than the pre-increment baseline's `apps/cli` count, matching the
    removed `intent.test.ts`).
- Full `npx vitest run` (`npm test`): **211 files / 2249 tests, all
  green.** Reconciles exactly against the stated pre-increment
  baseline (212 files / 2254 tests): −1 file / −6 tests for the
  removed `intent.test.ts` (which had 6 tests: the 3-closed-ids
  check, the ≥2-steps check, the 3 per-intent ordering checks, and
  the unknown-id check), +1 test for the new `declared`-for-
  instruction-only case in `packages/toolchain/tests/
  toolchain.test.ts` → 212 − 1 = 211 files; 2254 − 6 + 1 = 2249
  tests. Zero unexplained deltas.
- LIVE e2e proof, real compiled `apps/cli/dist/index.js`, scratch
  project (`AppData/Local/Temp/.../scratchpad/inc-honesty-live`,
  removed afterward; the one incidental `~/.axiom/projects.yml` entry
  it created via `axiom init`'s registry auto-add was manually
  removed afterward — no transient `axiom.config/toolchain.yaml` was
  ever created in the REAL `Axiom/` repo, since the CLI was invoked
  with cwd pointed at the scratch project throughout):
  - `toolchain add --id serena` → `✓ Tool 'serena' agregada al
    toolchain.` (against a copy of the real
    `axiom.config/toolchain-catalog.yaml`).
  - `toolchain show` (no marker yet) → `STATE: absent` — NOT
    `installed-working` (confirms `serena` is not on this machine's
    `PATH` as a raw CLI binary; the probe genuinely ran and failed
    honestly).
  - `toolchain repair` → creates the marker dir → `toolchain show`
    again → `STATE: marker` — still honestly NOT
    `installed-working` (this is exactly the case the OLD code
    called `present`).
  - `toolchain show --json` → `{"manifest": {...}, "detections":
    [{"toolId": "serena", "state": "marker", "detectedPath": "..."}]
    }` — differentiated state visible in JSON.
  - Manifest edited to mark `serena` as `mvp: true` (required) in
    the catalog → `toolchain validate` → `✓ Toolchain válido con 1
    warning(s): ⚠ [required-tool-not-verified] Tool requerida
    "serena" está declarada (estado: marker) pero un probe real no
    confirmó una instalación funcional (installed-working).` — the
    exact honest signal the brief asked for.
  - `axiom --help` → no `intent` entry (was already true before this
    increment — `axiom intent` never existed as a registered CLI
    command; the removed `intent.ts` module was dead independent of
    `--help`).

## Result

Closed. Part 1: corrected `Axiom/README.md` and rewrote
`packages/orchestrator/README.md` for honesty; on investigation, the
brief's own premise about `axiom intent` was inaccurate (conflated
the orchestrator's 19 never-invoked `CommandId` stubs with an
unrelated, genuinely dead 3-intent CLI module) — removed the
genuinely dead `intent.ts` + its test, and annotated (did not remove)
the orchestrator's 19 stubs, because removing them would mean
deleting/rewriting legitimate internal state-machine contract tests
for zero real-behavior change (nothing consumes those ids either
way), which exceeds a "trivial dead-code" removal for no benefit
beyond what the documentation fix already delivers. Part 2:
introduced a real, best-effort, injectable, never-throwing probe
(`packages/toolchain/src/probe.ts`) and a differentiated
`declared`/`absent`/`marker`/`installed-working` state model,
replacing the dishonest binary `present`/`absent` that let an empty
marker directory satisfy `axiom toolchain validate`; wired into
`axiom toolchain show`/`validate` (incl. `--json`) as the new
default, backward-compatible for every pre-existing caller
(`@axiom/doctor`'s TC-004 keeps its exact prior behavior since it
never opts into the `detections` param). Part 3: fixed the
`upgradeFailed:` double-prefix bug, the exact sibling of the
`gateFailure:` bug already fixed in
`INC-20260710-lifecycle-correctness-fixes`, reusing the same helper
and the same one-line fix pattern. Full suite green throughout
(211 files / 2249 tests, reconciling exactly against the 212/2254
baseline). Two out-of-scope findings discovered during the audit
(`doctor-command`'s gate-bypass; the phantom
`command-protocol.yaml`) are documented, not fixed, per AGENTS.md's
scope discipline.

## General spec integration

- `general-spec.md` does not exist in this repo's structure (same
  situation already noted by `INC-20260710-lifecycle-correctness-
  fixes`, `INC-20260710-dynamic-team-roles`, and
  `INC-20260710-workspace-command-parity`). The equivalent canonical
  location used here is `Axiom.Spec/specs/05_Interfaces_Operativas.md`
  (CLI surface) and `Axiom.Spec/specs/03_Modelo_Operativo_y_Datos.md`
  (data/state model) — both updated below.
- `Axiom.Spec/specs/03_Modelo_Operativo_y_Datos.md`: added a note
  under the toolchain section documenting the new `ToolState`
  differentiated model (`declared`/`absent`/`marker`/
  `installed-working`) replacing the old dishonest `present`/`absent`
  binary, and that `axiom toolchain validate` now distinguishes
  "declared/marker but unverified" (warning) from "confirmed
  installed-working" for required tools.
- `Axiom.Spec/specs/05_Interfaces_Operativas.md`: updated the
  pre-existing "naming collision" section (which, independently of
  this increment, had ALREADY documented the exact same finding —
  `intent.ts` unreachable, the orchestrator's 19 stubs unreachable)
  to record the outcome: `intent.ts` removed, the orchestrator's
  stubs kept-and-annotated, and the real SDD execution path
  (`@axiom/workflow`) named explicitly. Also removed `intent` from
  the "commands without deep documentation" list (line 16) since the
  file no longer exists. Did NOT add a `toolchain --json` shape note
  here — `toolchain` is explicitly listed in this doc as "not yet
  documented in depth"; the `--json` shape detail lives in the
  `03_Modelo_Operativo_y_Datos.md` addition instead, alongside the
  rest of the differentiated-state model.
