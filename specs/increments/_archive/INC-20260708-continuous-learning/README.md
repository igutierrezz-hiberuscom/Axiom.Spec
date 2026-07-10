# Increment: Continuous learning (AB10)

Status: closed
Date: 2026-07-08

## Goal

Give Axiom a MINIMAL, CONCRETE continuous-learning capability grounded in
REAL systems already shipped: AB5's engram-backed `@axiom/memory`
(topic-keyed upsert, `resolveMemoryBackend` fallback) and the REAL
telemetry/audit run records (`.axiom-state/local/audit.log`). NOT a
speculative "instinct engine" (ECC's `homunculus`/instinct-import/
instinct-export machinery was read and explicitly rejected as
out-of-bootstrap-scope — see Non-goals). Add opt-in, LIGHT hooks so a
lesson can be captured at session end and recalled at session/work start.

## Context

- `Axiom` (`C:\repos\Axiom Workspace\Axiom`) now has a REAL memory
  backend: `packages/memory/src/engram-backend.ts` +
  `resolve-backend.ts` (`INC-20260708-memory-real-local-backend`, AB5).
  `resolveMemoryBackend(projectRoot, scope, opts)` probes a LOCAL
  `engram mcp` process and falls back to the JSON file backend
  (`createInMemoryBackend`) — never throws, always returns a `note`.
  `MemoryEntry.topicKey` gives UPSERT semantics (same `(projectId,
  topicKey)` updates in place) on BOTH backends.
- `MemoryKind` (`packages/memory/src/types.ts`) is a CLOSED union:
  `'decision' | 'bug' | 'learning' | 'pattern' | 'context'`. There is no
  `'lesson'`/`'instinct'` kind, and this increment does not add one
  (closed-set discipline, GATE 0024 spirit) — lessons are stored as
  `kind: 'pattern'` (a recurring, reusable behavior) tagged `'lesson'`.
- `packages/telemetry/src/sinks/audit-trail.ts` appends one JSON
  `TelemetryEvent` line per accepted signal to
  `<projectRoot>/.axiom-state/local/audit.log` (+ a `.sha256` sidecar).
  `packages/telemetry/src/audit-trail-verify.ts`'s `auditTrailVerify` is
  the ONLY sanctioned read path for that file, but it is
  integrity/aggregate-only (`sha256`, `lineCount`, `eventCount`,
  `oldest/newestEvent`, retention violations) — it does NOT expose
  individual parsed events. Reading individual audit records for
  learning purposes therefore requires a small, local, read-only reader
  (same `fs.readFileSync` + JSON-line-parse pattern as
  `auditTrailVerify`, kept INSIDE the new learning module, not added to
  `@axiom/telemetry`'s public surface — out of this increment's scope
  per the Non-goals below).
- `apps/cli/src/commands/memory.ts` is the CLI-surface pattern to mirror
  (`show`/`add`/`query`/`inventory`, `resolveMemoryScope` +
  `createInMemoryBackend`, Spanish UI, `run*`/`register*` split for
  testability without commander/process.exit). It is registered
  APP-SIDE in `apps/cli/src/index.ts` (`import { registerMemory } from
  './commands/memory'`), NOT via `@axiom/cli-commands` — confirming
  memory-adjacent CLI surfaces are app-owned, not migrated into the
  shared `@axiom/cli-commands` package. `axiom learn` follows the SAME
  app-side placement (avoids the `@axiom/cli-commands` single-ownership
  gotcha: that package compiles a fixed set of shared command files and
  must not be extended piecemeal from `apps/cli`).
- ECC (`C:\repos\ECC`) was read for inspiration only
  (`.claude/homunculus/instincts/`, `commands/{evolve,instinct-import,
  instinct-export,instinct-status,learn}.md`, `.kiro/steering/
  lessons-learned.md`). Its instinct engine (confidence scoring,
  inheritance, promotion pipelines) is explicitly the kind of
  speculative/enterprise architecture `Axiom.SDD/AGENTS.md`'s "Explicit
  Bootstrap Limits" rules out. This increment borrows ONLY the
  recall-at-start / capture-at-stop shape, re-grounded in Axiom's real
  memory + telemetry, with none of ECC's scoring/inheritance/promotion
  machinery.
- `packages/workflow/src/hooks.ts` (`createHookEngine`) is Axiom's
  existing hook mechanism, but it is scoped to WORKFLOW step lifecycle
  (`pre-start`/`post-start`/`pre-complete`/`post-complete` keyed by
  `workflowId`) — a different concept from an AGENT SESSION lifecycle
  hook (ECC's `SessionStart`/`SessionStop`). There is no existing
  Axiom-native session-level hook to wire into; inventing one is
  out-of-scope speculative infrastructure. Session-level recall/capture
  is instead surfaced as: (a) a real CLI command (`axiom learn
  capture`/`list`) any adapter or human can invoke, (b) a small, real
  integration point (`axiom context status` now includes a
  recent-lessons recall block), and (c) a documented, OPT-IN
  `.claude/settings.json` hooks snippet that shells out to `axiom learn`
  — never auto-applied, never written to disk by Axiom itself.

## Scope

- `Axiom/packages/memory/src/learning.ts` (new) — three PURE, testable
  functions (`extractLessons`, `persistLessons`, `recallLessons`) plus a
  small local, read-only audit-log reader
  (`readRecentAuditEvents(projectRoot, limit)`, same pattern as
  `auditTrailVerify`: `fs.readFileSync` + JSON-line parse, best-effort,
  never throws). Lives in `@axiom/memory` (not a new `packages/learning`
  package — see Assumptions for the "why here, not a new package"
  justification) because its entire persistence surface IS `@axiom/
  memory`'s `MemoryBackend`/topic-key/scope model; a separate package
  would just re-import all of `@axiom/memory`'s types for no isolation
  benefit, which the AGENTS.md "no speculative architecture" guardrail
  argues against.
- `Axiom/packages/memory/src/index.ts` — barrel export of the three
  functions + supporting types (`LessonCandidate`, `LessonRecord`,
  `AuditEventSample`, `ExtractLessonsInput`, `RecallLessonsOptions`).
- `Axiom/packages/memory/tests/learning.test.ts` (new) — hermetic tests
  for all three functions (pure-function tests for `extractLessons`;
  `createInMemoryBackend`-backed tests for `persistLessons`/
  `recallLessons`, topic-key upsert-in-place, project isolation).
- `Axiom/apps/cli/src/commands/learn.ts` (new) — `axiom learn capture
  [--from-audit] [--text "..."] [--limit N]` and `axiom learn list
  [--query ...] [--limit N]`, mirroring `memory.ts`'s `run*`/`register*`
  split, `resolveMemoryScope`, Spanish UI. Uses `resolveMemoryBackend`
  (AB5) with `forceJson` NOT set by default (same fallback behavior as
  any other real caller) so a real engram install is used transparently
  when present.
- `Axiom/apps/cli/tests/learn.test.ts` (new) — CLI-level tests for
  `capture`/`list`, forcing `forceJson: true` via a `--no-engram`-style
  test seam (mirrors how `engram-backend.test.ts` avoids spawning a real
  engram probe) — hermetic, no real engram or telemetry event needed
  (uses a small hand-written `audit.log` fixture).
- `Axiom/apps/cli/src/index.ts` — one import + one `registerLearn(program)`
  call, registered AFTER `registerMemory`/`registerMcp` and BEFORE the
  TUI registration (same convention as every other 0024-adjacent
  command).
- `Axiom/apps/cli/src/commands/context.ts` — `runContextStatus` gains a
  best-effort "recent lessons" recall block (top 3 `pattern`/`lesson`-
  tagged memory entries for the active project, via
  `resolveMemoryBackend` + `recallLessons`), printed under the existing
  status lines. This is the "recall surfaced where an agent begins
  work" integration point: `axiom context status` is the existing,
  real, already-tested command an agent/operator runs to see project
  state at the start of a session. Never fails the command if memory
  resolution fails (best-effort, degrades to "no lessons available").
- `Axiom/apps/cli/tests/context.test.ts` — extended with a scenario
  covering the new recall block (present when lessons exist, silently
  absent/best-effort when memory is empty or unavailable).
- Documentation-only, NOT written to any repo by code: a short, opt-in
  `.claude/settings.json` hooks snippet documented inline in
  `learn.ts`'s header comment (SessionStart → `axiom learn list
  --query "" --limit 3`; SessionEnd/Stop → `axiom learn capture
  --from-audit`) — an adapter/operator may copy it manually into their
  own `.claude/settings.json`. No new template constant is added to
  `workspace-adapter-templates.ts` (that file bundles CONTENT actually
  written by `generateWorkspaceAdapters`; this snippet is deliberately
  NOT auto-written, per the brief's "NOT auto-enabled").

## Non-goals

- No new `MemoryKind` value (no `'lesson'`/`'instinct'` enum member).
  Lessons are `kind: 'pattern'` + `tags: ['lesson']` (optionally
  `['lesson', 'instinct']` when the operator explicitly requests the
  ECC-flavored tag) — fits the EXISTING closed set.
- No ML, embeddings, confidence scoring, decay, promotion pipelines, or
  inheritance across projects (ECC's `homunculus` concepts). `extract
  Lessons` is deterministic pattern-matching over real records (repeat
  counts, explicit operator text) — nothing probabilistic.
- No new `packages/learning` workspace package. See Scope's placement
  rationale.
- No changes to `@axiom/telemetry`'s public API (no new exported
  "read individual audit events" function) — the audit-log reader used
  by `extractLessons`'s `--from-audit` path is a small private helper
  local to `packages/memory/src/learning.ts`, reading the SAME on-disk
  format (`.axiom-state/local/audit.log`, one JSON `TelemetryEvent` per
  line) `auditTrailVerify` already reads, via the same read-only
  `fs.readFileSync` pattern. This avoids widening `@axiom/telemetry`'s
  sanctioned-read-path contract (`audit-trail-access-control`'s "sink
  has no read API" scenario governs the SINK; this increment does not
  touch the sink or add a second sink-level read API).
- No new Axiom-native session-lifecycle hook engine (no `SessionStart`/
  `SessionStop` construct added to `@axiom/workflow` or anywhere else).
  `packages/workflow/src/hooks.ts`'s existing `HookEngine` is workflow-
  step-scoped and is NOT extended or repurposed here — doing so would
  be exactly the kind of "heavy multi-agent orchestration framework"
  `Axiom.SDD/AGENTS.md` rules out. The `.claude/settings.json` snippet
  is documentation, not code.
- No auto-invocation of `axiom learn capture` from any existing command
  (no command silently starts writing memory entries as a side effect).
  Capture is always an explicit `axiom learn capture` invocation (by a
  human, a script, or a manually-configured adapter hook).
- No git commits. No integration into `Axiom.Spec/specs/00-08` (reserved
  for the orchestrator's final consolidation pass per the batch brief).
- No new npm dependencies.

## Acceptance criteria

- [x] `extractLessons(input)` exists, is PURE (no I/O), and derives
      `LessonCandidate[]` from (a) an explicit operator-provided
      `text`, and/or (b) a sample of REAL `AuditEventSample` records
      (repeated failures on the same `capabilityId`, recurring
      `tool-calls`/`dispatch` signal patterns) — deterministic, no
      randomness, no ML.
- [x] `persistLessons(lessons, backend, {projectId, scope})` stores each
      lesson as a `MemoryEntry` (`kind: 'pattern'`, tag `'lesson'`) via
      the given `MemoryBackend`, using a DERIVED `topicKey` (stable hash
      or slug of the lesson's normalized text) so a recurring lesson
      UPDATES in place (topic-key upsert, reusing AB5's mechanism)
      rather than duplicating.
- [x] `recallLessons(query, backend, scope, options?)` retrieves lesson-
      tagged entries relevant to `query` (reuses `@axiom/memory`'s
      existing `rankEntries`/`buildRecallResult`, filtered to
      `tags: ['lesson']`) — no new ranking algorithm invented.
- [x] `axiom learn capture [--from-audit] [--text "..."] [--limit N]`
      and `axiom learn list [--query ...] [--limit N]` exist, mirror
      `memory.ts`'s CLI conventions, and are wired into
      `apps/cli/src/index.ts`.
- [x] `axiom context status` surfaces a best-effort recent-lessons
      block (never fails the command if memory/lessons are unavailable).
- [x] GATE 0024 project isolation is preserved: lessons are persisted
      and recalled strictly through `resolveMemoryScope`-bound backends
      (same cross-project-blocked guarantee as every other memory
      caller) — no new isolation mechanism invented, no isolation
      bypass introduced.
- [x] Best-effort: a learning failure (audit log missing/corrupt, memory
      backend unavailable) never raises the exit code of an unrelated
      command; `axiom learn` itself reports a clear message and exit 1
      only for its OWN operation failures.
- [x] No new `MemoryKind` added; no speculative scoring/ML/instinct
      engine introduced.
- [x] Hermetic tests: `extractLessons` pure-function tests (sample audit
      records + explicit text; dedupe/topic-key derivation);
      `persistLessons`/`recallLessons` against `createInMemoryBackend`
      (topic-key upsert-in-place on a repeated lesson; project isolation
      preserved); `axiom learn capture/list` CLI thin tests; `axiom
      context status` recall-block test. None depend on a real engram
      binary.
- [x] `npm run build` (`tsc -b`) clean.
- [x] `npx vitest run apps/cli packages/memory packages/telemetry`
      passes.
- [x] `npx vitest run` (full, parallel) and `npx vitest run
      --no-file-parallelism` (full, serial) both pass with 0 regressions
      against the AB9 closing baseline.

## Open questions

None blocking. Ambiguities resolved under Assumptions.

## Assumptions

- **Lessons live in `@axiom/memory`, not a new `packages/learning`
  package**: `extractLessons` is pure and could technically live
  anywhere, but `persistLessons`/`recallLessons` are thin wrappers over
  `@axiom/memory`'s own `saveMemory`/`queryMemory`/`rankEntries`/
  `buildRecallResult` and `MemoryBackend`/`ResolvedMemoryScope` types.
  Splitting them into a new package would only add an import boundary
  with no isolation benefit, contradicting AGENTS.md's "do not create
  speculative architecture." `packages/memory/src/learning.ts` (new
  file) + barrel re-export is the minimal placement.
- **`kind: 'pattern'` + `tags: ['lesson']`, not a new `MemoryKind`**:
  `pattern` is defined in `types.ts` as "a recurring pattern that
  worked (or didn't)" — the closest existing semantic fit for a learned
  lesson. Adding a `'lesson'`/`'instinct'` enum member would touch a
  closed, deliberately small union for a single increment's benefit;
  tagging achieves the same discoverability (`axiom learn list` filters
  by tag) without widening the type.
- **`extractLessons`'s audit-derived candidates use a SIMPLE repeat-
  count heuristic**: given a window of `AuditEventSample` records
  (`capabilityId`, `signalType`, `payload` fields already on
  `TelemetryEvent`), a lesson candidate is emitted when the SAME
  `capabilityId` appears >= a small threshold (default 3) within the
  sample — phrased as a neutral, human-readable observation ("capability
  X was exercised N times in recent activity"). This is intentionally
  NOT failure/success classification (the wire `TelemetryEvent` payload
  shape is caller-defined free-form `Record<string, unknown>`, so a
  generic "failed"/"succeeded" field cannot be assumed to exist across
  every signal type without inventing a schema) — deliberately
  conservative rather than guessing at semantics the type system
  doesn't guarantee.
- **Topic key derivation**: `topicKey` for an audit-derived lesson is
  `learning/<capabilityId>` (stable per capability, so repeated runs
  upsert the SAME entry); for an explicit `--text` lesson, `topicKey` is
  `learning/manual/<slug-of-first-40-chars>` — stable enough to dedupe
  near-identical repeated manual entries without being so aggressive
  that unrelated manual lessons collide.
- **`readRecentAuditEvents` is best-effort and PRIVATE**: mirrors
  `auditTrailVerify`'s read-only guarantee (`fs.readFileSync`/
  `existsSync` only) but is not exported from `@axiom/memory`'s public
  barrel — it is an internal implementation detail of `extractLessons`'s
  `--from-audit` path, invoked by the CLI layer (`learn.ts`) which reads
  the file and passes the resulting `AuditEventSample[]` into the PURE
  `extractLessons` function. This keeps `extractLessons` pure/testable
  (fixture arrays, no filesystem in the pure-function tests) while still
  wiring a REAL file read for the `axiom learn capture --from-audit`
  path.
- **`axiom context status`'s recall block, not `axiom start`**: `axiom
  start` (`start.ts`) is a narrow, single-purpose runtime bootstrap
  command (writes `last-start.json`, samples one `routeTool` call) with
  no existing "status/summary for a human/agent" output shape. `axiom
  context status` already prints a multi-line project-state summary
  (profile triple, capabilities, install profile) — the natural,
  minimal place to append a "recent lessons" block without inventing a
  new command or a new "agent work start" concept.
- **No adapter-hook file is written by Axiom**: the `.claude/
  settings.json` snippet is documented as a comment/example in
  `learn.ts`'s header (verbatim JSON), not emitted by any
  `generateWorkspaceAdapters`-style writer, per the brief's explicit
  "best-effort, documented, NOT auto-enabled."

## Implementation notes

**Files created:**
- `Axiom/packages/memory/src/learning.ts` — `extractLessons` (pure),
  `persistLessons`, `recallLessons`, `LESSON_TAG`, plus the supporting
  types listed in Scope.
- `Axiom/packages/memory/tests/learning.test.ts` — 16 hermetic tests.
- `Axiom/apps/cli/src/commands/learn.ts` — `registerLearn`,
  `runLearnCapture`, `runLearnList`, `formatLessonHits`, and the private,
  read-only `readRecentAuditEvents` (audit-log projection reader).
- `Axiom/apps/cli/tests/learn.test.ts` — 9 hermetic CLI-level tests
  (all pass `forceJson: true`).

**Files edited:**
- `Axiom/packages/memory/src/index.ts` — barrel export of the new
  learning surface.
- `Axiom/apps/cli/src/index.ts` — `import { registerLearn } from
  './commands/learn'` + `registerLearn(program)`, registered right
  after `registerMcpServe` and before `registerApp`.
- `Axiom/apps/cli/src/commands/context.ts` — `runContextStatus` now
  calls `buildRecentLessonsBlock` (best-effort, never throws) and
  appends a `recentLessons` block to its existing multi-line output.
  Added a `forceJsonMemory?` test seam to `ContextStatusArgs` (the real
  CLI never sets it — only tests do, for the same hermetic reason as
  every `resolveMemoryBackend` caller's test).
- `Axiom/apps/cli/tests/context.test.ts` — 2 new scenarios ("recall
  block present when lessons exist", "absent/best-effort when empty"),
  both using `forceJsonMemory: true`.

**Real-environment discovery (worth flagging for the batch's final
consolidation pass):** this dev machine has a REAL `engram` binary on
PATH (`C:\Users\<user>\AppData\Local\engram\bin\engram.exe`). The first
draft of `context.test.ts`'s new scenario called `resolveMemoryBackend`
without forcing JSON in `buildRecentLessonsBlock`, while the test's
`runLearnCapture` call used `forceJson: true` — the two calls picked
DIFFERENT backends (engram vs JSON), so the lesson written by one was
invisible to the other, and the test failed non-deterministically
depending on whether engram happens to be installed on the machine
running the suite. Fixed by adding the `forceJsonMemory` test seam to
`runContextStatus`, mirroring the exact pattern `engram-backend.test.ts`
already established for `resolveMemoryBackend` callers. This is a
general gotcha for ANY future caller of `resolveMemoryBackend` from
CLI-level tests: always force JSON explicitly rather than relying on
"engram is probably not installed on CI" — it may well be installed on
a developer's own machine.

**Design confirmations made during implementation** (all pre-decided in
Assumptions, applied as planned, no deviations):
- `LessonCandidate`/`LessonRecord` are new types in `learning.ts`, not
  additions to `types.ts`'s closed unions — `MemoryKind` was NOT
  touched.
- `persistLessons`'s `PersistLessonsScope` is structurally identical to
  `ResolvedMemoryScope` (kept as a separate named type for call-site
  clarity at the `learn.ts` boundary; assignable directly).
- `recallLessons` intentionally reuses `buildRecallResult` verbatim
  (no new ranking code) — confirmed by the "ranked by recency + kind
  boost" test (`recallLessons with empty query still returns recent
  lessons`).

## Validation

- `npm run build` (`tsc -b`, from `Axiom/`): clean, no errors.
- `npx vitest run apps/cli/tests/learn.test.ts
  packages/memory/tests/learning.test.ts`:
  ```
  Test Files  2 passed (2)
       Tests  25 passed (25)
  ```
- `npx vitest run apps/cli packages/memory packages/telemetry`:
  ```
  Test Files  69 passed (69)
       Tests  635 passed (635)
  ```
- `npx vitest run` (full suite, parallel — default config):
  ```
  Test Files  195 passed (195)
       Tests  2064 passed (2064)
  ```
- `npx vitest run --no-file-parallelism` (full suite, serial):
  ```
  Test Files  195 passed (195)
       Tests  2064 passed (2064)
  ```
  0 regressions in either mode. Baseline (stated in the batch brief) was
  `2037/2037`; delta is `+27` new tests, all new, all passing (16 in
  `packages/memory/tests/learning.test.ts`, 9 in
  `apps/cli/tests/learn.test.ts`, 2 in `apps/cli/tests/context.test.ts`'s
  new "Scenario 2b" describe block).

## Result

`@axiom/memory` gained a `learning.ts` module (`extractLessons`/
`persistLessons`/`recallLessons`) that is a thin, deterministic layer
over AB5's REAL memory backend infrastructure — no new `MemoryKind`, no
new persistence mechanism, no ranking algorithm invented; lessons are
`kind: 'pattern'` entries tagged `'lesson'`, upserted via the SAME
topic-key mechanism every other memory entry already uses.
`extractLessons` derives candidates from an explicit operator text
and/or a deterministic repeat-count heuristic over a sample of REAL
audit-trail records (`.axiom-state/local/audit.log`, read by a small,
private, read-only helper in the CLI layer — `@axiom/telemetry`'s public
surface was not touched). `axiom learn capture [--from-audit] [--text]`
and `axiom learn list [--query]` are a real, working, app-owned CLI
surface (mirroring `axiom memory`'s exact placement and conventions),
wired into `apps/cli/src/index.ts`. `axiom context status` — the
existing, real "what does this project look like" command — now
surfaces a best-effort "recentLessons" block, the concrete "recall at
the start of work" integration point requested by the brief; it never
fails the command if memory is unavailable or empty. GATE 0024 project
isolation is preserved for free (every persist/recall call is routed
through the caller-supplied, scope-bound `MemoryBackend`, exactly like
every other `@axiom/memory` caller — confirmed by a dedicated
cross-project-blocked test). Session-level hooks (SessionStart/
SessionStop) remain a documented, OPT-IN `.claude/settings.json`
snippet in `learn.ts`'s header comment — Axiom does not ship or invoke
any session-lifecycle hook engine; nothing is auto-applied, and no
existing command silently starts writing memory as a side effect of
this increment. No ML, confidence scoring, decay, or promotion pipeline
was introduced (ECC's `homunculus` instinct-engine concepts were
explicitly read and rejected as speculative/out-of-bootstrap-scope). A
real-environment gotcha was discovered and fixed during test
implementation: this dev machine has a real `engram` binary installed,
which made an early draft of the `context.test.ts` scenario
non-deterministic until a `forceJsonMemory` test seam (mirroring
`engram-backend.test.ts`'s established pattern) was added.

## General spec integration

Integrated into `Axiom.Spec/specs/00-08` by the batch's final
consolidation pass (autopilot orchestrator). Landed in:

- **04_Flujos_SDD_y_Ciclo_de_Vida.md** — new subsection "Aprendizaje
  continuo (captura al cierre, recall al inicio del trabajo)": the
  recall-at-start (`axiom context status` block) and explicit-capture
  (`axiom learn`) shape in the lifecycle; hooks are a documented opt-in
  snippet, never auto-applied.
- **06_Integraciones_y_Capacidades.md** — new subsection "Aprendizaje
  continuo (módulo `learning.ts` de `@axiom/memory`)":
  `extractLessons`/`persistLessons`/`recallLessons`, deterministic, no
  ML, no new `MemoryKind`.
- **05_Interfaces_Operativas.md** — `axiom learn capture|list` CLI.
- **01_Requisitos_Funcionales.md** — RF-AXM-027 (continuous learning).
- **03_Modelo_Operativo_y_Datos.md** — lessons stored as
  `kind: 'pattern'` tag `'lesson'`, topic-key derivation, in the
  INC-20260708 data subsection.
- **08_Glosario.md** — new term: lesson / continuous learning.
