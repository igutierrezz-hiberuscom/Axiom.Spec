# Increment: Delegation triggers (non-blocking signal) + curated delegation-agent roster

Status: closed
Date: 2026-07-08

## Goal

Give Axiom a data-driven, NON-BLOCKING delegation-trigger signal
(GentleAI's "thin orchestrator, delegate by default" discipline,
adapted honestly to what Axiom actually is: a CLI + orchestrator
state machine, not an agent runtime with live tool-call
instrumentation) plus a small curated roster of specialized
delegation agents that the triggers point at. This was ranked the
#1 GentleAI-inspired opportunity in the prior cross-repo analysis.

## Context

- `packages/orchestrator/src/{types,state-machine,gates,runner,index}.ts`:
  a pure 22-command state machine (`STATE_MACHINE`, `gateFor`,
  `runCommand`). `runCommand` already has an `onTransition?:
  (trace: TraceEntry) => void` seam (AD-07) invoked once,
  synchronously, after a successful command — the natural extension
  point for a non-blocking post-command signal, but it fires per
  command, not per "session," and the brief's metrics (files-read,
  files-touched, sequential-command count, task-scope size) are NOT
  tracked anywhere in this codebase today — Axiom has no live
  tool-call instrumentation layer (it is a CLI, not an in-process
  agent loop). Fabricating that instrumentation would be exactly the
  kind of speculative architecture `Axiom.SDD/AGENTS.md` forbids.
- `packages/agents/src/{types,catalog,materialize,index}.ts`: an
  "agent" here is a MATERIALIZABLE CONTRACT (D1 of spec 0033) — a
  catalog entry (YAML, `axiom.config/agents-catalog.yaml`) with a
  `.md` source under `axiom.spec/target-axiom-agents/`, rendered by
  `materializeAgentSet` to `.opencode/agents/<id>/AGENT.md`. Roles
  are a CLOSED union (`AgentRole`): `sdd-orchestration | planning |
  product-review`. Only 3 official agents exist today (thin roster).
- `.claude/agents/{axiom-increment,axiom-bug,axiom-review}.md`: the
  actual bootstrap agents this workspace's own orchestrator spawns
  today (increment executor, bug executor, review executor) — the
  vocabulary a curated delegation roster should align with.
- `packages/skills/src/*` + `axiom.config/skills-catalog.yaml` +
  `06_Integraciones_y_Capacidades.md` §"Baseline de Axiom SKILLS...":
  the established "bundled seed" pattern this codebase already uses
  — a small canonical catalog seeded as data (YAML + TS source
  constants), NOT invented at increment time from nothing. The
  delegation roster reuses this exact pattern rather than a new one.
- `apps/cli/src/commands/context.ts` (`runContextStatus`,
  `buildRecentLessonsBlock`): the existing, real "what does the
  project look like right now" status surface. AB10
  (`INC-20260708-continuous-learning`) already added a best-effort,
  never-throws "recentLessons" block here — this increment follows
  the identical wiring pattern for a "delegationSuggestions" block:
  same command, same best-effort/never-fail discipline, no new
  command invented.
- GentleAI's Hermes delegation rules
  (`C:\repos\GentleAI\docs\agents.md`, "Hermes Ephemeral Delegation"
  section) are the closest REAL precedent for the brief's threshold
  language: "Delegate when work needs broad exploration (4+ files),
  multi-file implementation, test/build execution, or fresh
  adversarial review." These are honest, human-facing heuristics,
  not live-instrumented triggers — Hermes' own docs describe them as
  guidance for the orchestrator's judgment, not a wired numeric
  trigger evaluated by code. This increment's evaluator formalizes
  the SAME heuristics as a pure, testable function, but — because
  Axiom has no instrumentation to auto-populate the metrics — the
  metrics are an explicit, caller-supplied input (see Decisions).

### Critical scoping finding: no live metrics source exists

Investigation confirmed Axiom has no in-process agent loop, no
tool-call log, and no session-scoped file-read/file-touch counter
anywhere in the codebase (`grep` for
`filesRead|sequentialCommand|taskScope|SessionMetrics` across
`packages/` returned zero hits). GentleAI's own triggers are
evaluated by the LLM orchestrator's judgment inside a live
conversation loop — GentleAI does not "wire" them into Go code
either (`internal/` has no threshold-evaluator; `docs/agents.md`'s
table is operator-facing documentation, not a called function).

**Decision (resolved without stopping to ask):** build
`evaluateDelegation` as a PURE function over an explicit
`DelegationMetrics` input (the caller supplies real counts it
already has — e.g. a future CLI wrapper, a TUI session tracker, or a
human/operator filling in a `--metrics-file`). This is honest: the
function is real, tested, and useful the moment ANY caller has real
counts, without Axiom fabricating a synthetic "files read" counter
that doesn't correspond to anything real today. For the "surfaced in
status output" requirement, `axiom context status` reads an
OPTIONAL, best-effort `.axiom-state/<project>/session-metrics.json`
file if present (never required, never fabricated, never blocks) —
if a future increment or an external harness (e.g. the Claude Code
skill/agent loop itself) writes real counts there, the suggestions
appear; if absent, the block is silently omitted, exactly like
AB10's `recentLessons` degrades to `[]` when memory is unavailable.

## Scope

1. **`packages/orchestrator/src/delegation.ts`** (new): pure
   `evaluateDelegation(metrics: DelegationMetrics, thresholds?:
   DelegationThresholds): DelegationSuggestion[]`.
   - `DelegationMetrics`: `{ filesRead: number; filesTouched:
     number; sequentialCommands: number; taskScopeSize: 'small' |
     'medium' | 'large' }` — 4 real, minimal signals named exactly
     per the brief.
   - `DelegationThresholds` (bundled `DEFAULT_DELEGATION_THRESHOLDS`,
     configurable via the optional 2nd arg): `filesReadThreshold`
     (default 4, mirrors Hermes' "broad exploration (4+ files)"),
     `filesTouchedThreshold` (default 2, mirrors the brief's
     "touching 2+ non-trivial files → fresh review"),
     `sequentialCommandsThreshold` (default 5, a fresh-context-window
     heuristic), plus a `taskScopeSize === 'large'` structural
     trigger (no numeric threshold needed — it is already a
     3-value enum).
   - `DelegationSuggestion`: `{ trigger: DelegationTriggerId;
     agentId: string; reason: string; thresholdCrossed: string }` —
     each suggestion NAMES which roster agent to delegate to, why,
     and the exact threshold value crossed (e.g.
     `"filesRead=6 >= filesReadThreshold=4"`).
   - `DelegationTriggerId` closed union: `'broad-exploration' |
     'multi-file-touch' | 'long-sequential-run' | 'large-task-scope'`.
   - Below every threshold → `[]` (empty, not an error).
   - `recommendAgentForTrigger(trigger: DelegationTriggerId): string`
     — the trigger→agent-id mapping, exported standalone so callers
     can look up the mapping without re-running the evaluator.
   - NON-BLOCKING by construction: pure function, returns data, never
     throws for valid input, has no side effects, cannot fail a
     command (it is not called from `gateFor`/`runCommand`'s control
     flow at all — see Non-goals).
2. **`packages/orchestrator/src/index.ts`**: re-export
   `evaluateDelegation`, `recommendAgentForTrigger`,
   `DEFAULT_DELEGATION_THRESHOLDS`, `DELEGATION_TRIGGER_AGENTS`, and
   the 4 new types.
3. **`packages/agents/src/roster.ts`** (new): curated delegation
   roster as bundled TS constants (mirrors the skills-catalog seed
   pattern, §"Catálogo semilla bundleado" of
   `06_Integraciones_y_Capacidades.md`), 6 agents:
   - `axiom-explorer` (role: reuse `sdd-orchestration`) — broad
     read-only search/exploration across the workspace.
   - `axiom-spec-planner` — already exists in the catalog (role
     `planning`); reused as the roster's planner (no duplicate
     entry).
   - `axiom-reviewer` (role: `product-review`) — read-only quality
     review of an increment/bug/change set (mirrors
     `.claude/agents/axiom-review.md`'s real, already-shipped
     bootstrap agent).
   - `axiom-security-reviewer` (role: `product-review`) — read-only
     security-focused review (auth/secrets/injection surface).
   - `axiom-tester` (role: `sdd-orchestration`) — runs/interprets
     validation (build, tests, lint) for a change set.
   - `axiom-bug-executor` — already exists conceptually as
     `.claude/agents/axiom-bug.md`; NOT duplicated into the catalog
     in this increment (no roster entry created for it) because the
     brief's roster is for the ORCHESTRATOR's triggers
     (explore/review/security/test/plan), not for increment/bug
     execution itself, which already has its own dedicated
     `.claude/agents` entries outside `@axiom/agents`' catalog scope.
   - Total NEW catalog entries: 4 (`axiom-explorer`,
     `axiom-reviewer`, `axiom-security-reviewer`, `axiom-tester`) +
     1 reused (`axiom-spec-planner`, already catalogued) = 5 agents
     available to the roster mapping, well inside the "~5-8, not
     ECC's 67" bound.
   - `AgentRole` union: UNCHANGED (`sdd-orchestration | planning |
     product-review` already covers all 4 new agents — no new role
     value needed, avoiding an unnecessary closed-set expansion).
4. **`axiom.config/agents-catalog.yaml`**: append the 4 new agent
   entries (schema-compatible with the existing 3), each pointing at
   a new `.md` source under `axiom.spec/target-axiom-agents/`.
5. **`axiom.spec/target-axiom-agents/{axiom-explorer,axiom-reviewer,
   axiom-security-reviewer,axiom-tester}.md`** (new, short,
   single-purpose source files, same shape/length as the 3 existing
   sources) — real materializable content, not placeholder stubs.
6. **`packages/orchestrator/src/delegation.ts`**'s
   `DELEGATION_TRIGGER_AGENTS` mapping (the concrete
   `recommendAgentForTrigger` table):
   - `'broad-exploration'` → `axiom-explorer`
   - `'multi-file-touch'` → `axiom-reviewer`
   - `'long-sequential-run'` → `axiom-explorer`
   - `'large-task-scope'` → `axiom-spec-planner`
7. **Surfacing in `axiom context status`**
   (`apps/cli/src/commands/context.ts`): a new
   `buildDelegationSuggestionsBlock(rootPath, projectName)` helper,
   wired identically to AB10's `buildRecentLessonsBlock` (best-effort,
   try/catch-all, returns `[]` on any failure, appended after the
   `recentLessons` block). Reads an OPTIONAL
   `.axiom-state/<projectName>/session-metrics.json` (shape:
   `DelegationMetrics`); if absent or malformed, the block is empty
   (silently, matching the `recentLessons` degrade contract) — NEVER
   fabricates metrics, NEVER blocks or fails `axiom context status`.
   If present and valid, calls `evaluateDelegation` and renders each
   suggestion as one line:
   `  delegationSuggestions:` /
   `    - [broad-exploration] delegate to axiom-explorer (filesRead=6 >= filesReadThreshold=4)`.
8. Add `@axiom/agents` as a dependency of `apps/cli` (needed only if
   `context.ts` looks up roster agent metadata for the status line —
   see Implementation notes for the exact minimal need).

## Non-goals

- No live tool-call/file-read instrumentation layer added anywhere
  (no wrapping of `Read`/`Grep`/`Bash` tool calls, no session
  tracker, no new persisted "activity log"). `DelegationMetrics` is
  an explicit input; populating it from a real running agent loop
  (e.g. a Claude Code hook, a TUI session counter) is a documented
  future integration, not built here.
- No wiring into `gateFor` or `runCommand`'s pass/fail control flow.
  The evaluator is NEVER called from the state machine and CANNOT
  block, veto, or alter any command's outcome — "non-blocking" is
  structural (the function is simply never in that call path), not a
  flag that could accidentally be flipped to `enforce: true` later
  without a new increment.
- No new IDE/harness adapter IDE targets (Gemini CLI, Windsurf,
  Codex, Kilo, etc.) — flagged explicitly by the brief as a separate,
  bounded, on-demand follow-up. This increment touches ZERO files
  under `packages/adapters/*`.
- No speculative multi-agent-orchestration engine (no agent-to-agent
  message bus, no automatic spawning, no runtime that "executes" a
  suggestion). Axiom's job stops at SUGGESTING + providing roster
  metadata; the human/harness performs the actual delegation exactly
  as today.
- No new `AgentRole` enum value — the 4 new roster agents fit inside
  the existing closed union.
- No duplication of `.claude/agents/{axiom-increment,axiom-bug,
  axiom-review}.md` into `@axiom/agents`' catalog; those remain the
  workspace-level bootstrap agents for THIS repo's own increment
  workflow, a different concern from the product-level delegation
  roster this increment adds.
- No changes to `00`–`05`, `07`, `08` canonical spec docs beyond what
  is described under "General spec integration" below.
- No git commit (per orchestrator instructions).

## Acceptance criteria

- [x] `evaluateDelegation(metrics, thresholds?)` is pure, returns
      `DelegationSuggestion[]`, empty when all metrics are below
      threshold, and each suggestion names the correct
      `agentId`/`reason`/`thresholdCrossed` for the trigger it
      represents.
- [x] `DEFAULT_DELEGATION_THRESHOLDS` is bundled and overridable via
      the optional 2nd argument (a test passes custom thresholds and
      observes different trigger behavior for the same metrics).
- [x] `recommendAgentForTrigger` returns the correct roster agent id
      for each of the 4 `DelegationTriggerId` values.
- [x] `packages/agents/src/roster.ts` exports the curated roster (4
      new + 1 reused = 5 agent ids); the 4 new ids are present in
      `axiom.config/agents-catalog.yaml` and materialize correctly
      via the existing `materializeAgentSet` (new
      `packages/agents/tests/roster.test.ts`; `materialize.ts` itself
      is unchanged since the contract is catalog-driven).
  - Note: the 5th id (`axiom-spec-planner`) already existed in the
    catalog before this increment; the roster module correctly
    references it without a duplicate entry (verified by a test).
- [x] `axiom context status` prints a `delegationSuggestions` block
      when `.axiom-state/<project>/session-metrics.json` exists and
      is valid; prints nothing extra when it is absent, malformed,
      or has all-below-threshold metrics — and NEVER throws or
      changes the command's exit code in any of those cases.
- [x] `npm run build` clean.
- [x] `npx vitest run packages/orchestrator packages/agents apps/cli`
      passes.
- [x] `npx vitest run --no-file-parallelism` (full repo): 0
      regressions vs the 2064/2064 baseline.

## Open questions

None blocking. The absence of a live metrics source was resolved by
making `DelegationMetrics` an explicit, honest input rather than
fabricating instrumentation (see "Critical scoping finding" above).

## Assumptions

- "Files-read count" / "files-touched count" / "sequential-command
  count" / "task-scope size" are read literally as the brief's own
  minimal signal set; `taskScopeSize` is modeled as a 3-value enum
  (`'small' | 'medium' | 'large'`) rather than a numeric LOC/file
  count, because no such continuous metric exists anywhere in this
  codebase to threshold against, and inventing one would be
  speculative. A caller that has a numeric proxy (e.g. changed-line
  count) is expected to bucket it into this enum before calling
  `evaluateDelegation`.
- `session-metrics.json`'s exact producer is intentionally left
  external/future (a Claude Code hook, a TUI counter, or manual
  operator input) — this increment only defines the CONSUMER
  contract (`DelegationMetrics` shape) and the read-if-present
  wiring, per the "small real integration, not a framework" brief
  constraint.
- The roster's `axiom-spec-planner` reuse (rather than a fresh
  `axiom-planner` id) avoids a near-duplicate catalog entry; the
  existing agent already serves the exact "plan a large task" role
  the `'large-task-scope'` trigger needs.
- `@axiom/agents` becomes a new dependency of `apps/cli` (previously
  absent) strictly to let `context.ts` validate/label suggested
  agent ids against the real catalog when rendering the status
  block; this is a one-line `package.json` addition, not a layering
  violation (agents metadata is read-only, analogous to how
  `context.ts` already reads `@axiom/memory`).

## Implementation notes

- `packages/orchestrator/src/delegation.ts`: 4 pure functions/consts
  — `DEFAULT_DELEGATION_THRESHOLDS`, `DELEGATION_TRIGGER_AGENTS`,
  `recommendAgentForTrigger`, `evaluateDelegation`. Each of the 4
  triggers is checked independently (a single `DelegationMetrics`
  input can yield 0–4 suggestions, not mutually exclusive) — e.g. a
  metrics object with both `filesRead: 10` and `taskScopeSize:
  'large'` yields 2 suggestions (`broad-exploration` +
  `large-task-scope`), which is correct: both conditions are
  independently true and both are useful signals to surface.
- `packages/agents/src/roster.ts`: exports `DELEGATION_ROSTER:
  ReadonlyArray<{ id: string; oneLine: string }>` (5 entries, ids
  matching the catalog) — a thin, catalog-referencing list, NOT a
  duplicate of `Agent` (avoids two sources of truth for the same
  agent metadata; the catalog YAML remains authoritative for
  materialization).
- `axiom.config/agents-catalog.yaml`: 4 new entries appended after
  the existing 3, same shape (`id, name, version, role,
  responsibility, inputs, outputs, relatedSkills, relatedCommands,
  relatedWorkflows, source, status, securityCheckStatus,
  bundleHash`). `bundleHash` computed with the SAME sha256 convention
  as the existing 3 (`sha256:<hex>` of the source `.md` content) —
  computed via `computeAgentBundleHash` against each new source file
  and pasted into the YAML (matches how the existing 3 entries were
  authored; `@axiom/doctor`'s TC-011 re-validates it against the live
  source, unchanged by this increment).
- `apps/cli/src/commands/context.ts`: new
  `buildDelegationSuggestionsBlock` mirrors
  `buildRecentLessonsBlock`'s try/catch-all shape exactly; reads
  `session-metrics.json` with `fs.existsSync` + `JSON.parse` in a
  try/catch, validates the 4 required fields are present with
  correct types (defensive — a malformed file degrades to `[]`, same
  as a missing one), then calls `evaluateDelegation` and formats each
  suggestion as one line under a `  delegationSuggestions:` header
  (omitted entirely when the resulting suggestion list is empty,
  matching how `recentLessons:` is only pushed when `hits.length >
  0`).
- `apps/cli/package.json`: added `"@axiom/agents": "*"` next to the
  existing dependencies (single-line diff).

## Validation

### `npm run build` (tsc -b), from `Axiom/`

```
> axiom-product@0.1.0 build
> tsc -b

(clean exit, no errors)
```

### `npx vitest run packages/orchestrator packages/agents apps/cli`

```
 Test Files  67 passed (67)
      Tests  631 passed (631)
```

Includes this increment's new/updated suites:
`packages/orchestrator/tests/delegation.test.ts` (new, 13 tests:
per-trigger threshold crossing, below-threshold empty array,
configurable thresholds, `recommendAgentForTrigger` mapping for all
4 triggers, multi-trigger simultaneous suggestions, purity),
`packages/agents/tests/roster.test.ts` (new, 7 tests: roster shape
+ curated-bound assertion, lookup helper, all ids present in the
real catalog, `axiom-spec-planner` reuse asserted not duplicated,
materialize round-trip for the 4 new ids against sandboxed copies
of the real sources), `packages/agents/tests/catalog.test.ts`
(updated: the real-catalog integration test now asserts 7 entries
instead of 3, reflecting the roster's 4 new agents),
`apps/cli/tests/context.test.ts` additions (4 tests: suggestions
block appears with a valid, threshold-crossing
`session-metrics.json`; block absent when the file is missing;
block absent when malformed; block absent/omitted when all metrics
are below threshold — never throws or changes exit code in any
case). Also updated (not new): `packages/doctor/tests/agents.test.ts`
TC-011 real-repo smoke test, which asserts the catalog's
source+bundleHash consistency count — now `7/7` instead of `3/3`
(same check logic, extended fixture).

### `npx vitest run --no-file-parallelism` (full repo)

```
 Test Files  197 passed (197)
      Tests  2088 passed (2088)
```

0 regressions vs the 2064/2064 baseline; net +24 tests (13
delegation + 7 roster + 4 context suggestion tests = 24; the
catalog.test.ts and doctor/agents.test.ts updates changed existing
assertions' expected counts, not test counts).

## Result

Implemented as scoped. `packages/orchestrator/src/delegation.ts`
adds a pure, tested `evaluateDelegation`/`recommendAgentForTrigger`
pair over an explicit `DelegationMetrics` input (files-read,
files-touched, sequential-commands, task-scope-size) — deliberately
NOT wired into any live instrumentation (none exists in this
codebase; fabricating it would have been speculative), and
deliberately NEVER called from `gateFor`/`runCommand`'s control flow
(structurally non-blocking, not just non-blocking-by-convention).
`packages/agents/src/roster.ts` adds a curated 5-agent delegation
roster (4 new catalog entries — `axiom-explorer`, `axiom-reviewer`,
`axiom-security-reviewer`, `axiom-tester` — plus the pre-existing
`axiom-spec-planner` reused rather than duplicated), bundled the
same way the skills-catalog seed already is in this codebase
(catalog YAML + real `.md` sources under
`axiom.spec/target-axiom-agents/`), materializing through the
existing, unmodified `materializeAgentSet` contract. `axiom context
status` gained a best-effort `delegationSuggestions` block, wired
identically to AB10's `recentLessons` block (reads an optional
`session-metrics.json`, degrades silently to nothing on
absence/malformed input, never fails the command). All acceptance
criteria met; build clean; targeted suites + full regression suite
green (2088/2088, +24 vs the 2064 baseline, 0 regressions).

## General spec integration

Integrated into the canonical spec
(`Axiom.Spec/specs/06_Integraciones_y_Capacidades.md`) — appended a
new subsection **"Delegation triggers (señal no-bloqueante) y roster
curado de agents de delegación"** stating: (1) `evaluateDelegation`
is a pure, non-blocking suggestion signal over an explicit
`DelegationMetrics` input — Axiom has no live tool-call
instrumentation to auto-populate it, so the metrics are supplied by
the caller (a future harness/hook integration, documented as a
follow-up, not fabricated here); (2) the function is structurally
unreachable from `gateFor`/`runCommand`'s control flow, so it cannot
block or alter any command outcome; (3) the curated 5-agent
delegation roster (`axiom-explorer`, `axiom-spec-planner` (reused),
`axiom-reviewer`, `axiom-security-reviewer`, `axiom-tester`) extends
`axiom.config/agents-catalog.yaml` using the same bundled-seed
pattern as the skills catalog; (4) `axiom context status` surfaces
suggestions best-effort via an optional `session-metrics.json`,
mirroring AB10's `recentLessons` degrade contract; (5) adding more
IDE/harness adapter targets is explicitly out of scope and flagged
as a separate, bounded, on-demand follow-up (not a gap in this
increment). No changes to 00–05, 07, 08 (no contradiction found —
additive to 06's existing agents/skills/adapters sections).

**Reconciliation by the batch's final consolidation pass**: the 06
subsection this increment already appended
("Delegation triggers (señal no-bloqueante) y roster curado de agents
de delegación") was left intact (not duplicated). The final pass
additionally added a short mention in
**04_Flujos_SDD_y_Ciclo_de_Vida.md** ("Señal de delegación
no-bloqueante en el ciclo") that surfaces the `delegationSuggestions`
block in the lifecycle, and a glossary term in **08_Glosario.md**
(delegation triggers / roster). Both cross-reference the existing 06
subsection rather than restating it.
