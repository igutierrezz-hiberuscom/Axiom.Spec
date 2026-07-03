# Increment: Structural command foundation and event model (migration-engineer audit)

Status: pending
Date: 2026-07-02

## Goal

Execute the **migration-engineer** step of INC-02's subagent sequence
(migration-engineer -> registry-engineer -> validator-reviewer), per the
parent roadmap
(`Axiom.Spec/specs/increments/INC-20260702-axiom-redesign-roadmap/README.md`,
"INC-02 — Structural command foundation and event model"): audit
`@axiom/orchestrator` (`state-machine.ts`, `gates.ts`, `runner.ts`,
`types.ts`) and the CLI's `intent.ts` surface in full, map the source
decision document's 9 structural events onto the existing `CommandId`
union and intent-command stubs, and determine — grounded in actual code,
not the roadmap's own paraphrase — whether "wire INC-01 registry/topology
writes through `runCommand`/gates instead of direct writes" is genuinely
additive or a larger architectural change than the roadmap assumed.

This increment is audit-only. No code changes in `Axiom/`.

## Context

Parent chain: `INC-20260702-axiom-redesign-roadmap` (parent roadmap) ->
`INC-20260702-registry-manifest-schema-v2` chain (INC-01, five roles,
closed) -> this increment (INC-02, migration-engineer, first of three
roles).

INC-01 landed `*V2` registry functions (`addProjectV2`/`saveRegistryV2`/
etc.) in `@axiom/user-workspace`, `AxiomYamlSchemaV2` + version-dispatch
in `@axiom/config-validation`, a version-aware `resolveProject` in
`@axiom/project-resolution`, and doctor's MC-001 three-way branch — all
read in full before this audit (see
`INC-20260702-registry-manifest-schema-v2-impl`,
`INC-20260702-registry-manifest-schema-v2-cli`,
`INC-20260702-registry-manifest-schema-v2-resolver-fix`,
`INC-20260702-registry-manifest-schema-v2-validator`). Five CLI command
files (`init.ts`, `join.ts`, `projects.ts`, `repo.ts`, `app-api.ts`) now
call the `*V2` registry API directly, outside any orchestration layer.

The parent roadmap's own diff hypothesis for INC-02 (unverified prior to
this audit): "`@axiom/orchestrator` already implements a 7-lifecycle-state
machine and 15 intent commands... the gap is that several intent commands
are stubs... additive — implement the currently-stubbed intent commands
relevant to the new registry/manifest operations from INC-01 rather than
inventing a parallel event system." This audit confirms, corrects, and
sharpens that hypothesis against the real code.

**Source material caveat**: the two source decision documents
(`axiom_decisiones_sesion_prompt_implementacion.md` §6.2 and the addendum)
are not present in `Axiom.Spec` or anywhere else in this workspace — only
the parent roadmap's paraphrase of their content is available. This audit
works from that paraphrase (the 9 named events and the "internal events
update local caches, validate metadata, enable traceability" framing) and
flags this as a real source-fidelity gap, not a silent assumption.

## Scope

- Read `Axiom/packages/orchestrator/src/{state-machine.ts,gates.ts,
  runner.ts,types.ts,index.ts}` and their tests in full.
- Read `Axiom/apps/cli/src/commands/intent.ts` in full.
- Read every CLI command file that calls `runOrchestrated`
  (`audit.ts`, `configure.ts`, `init.ts`, `join.ts`, `start.ts`, `sync.ts`,
  `upgrade.ts`) and `_shared.ts`'s `runOrchestrated`/`withProjectContext`.
- Read `projects.ts`'s and `repo.ts`'s registry-write call sites and their
  own in-code rationale for not using `runOrchestrated`.
- Map the 9 source-doc events onto the `CommandId` union: existing
  equivalent / stub needing implementation / no counterpart.
- Determine the actual shape of "7 comandos CLI pasan por el gate"
  (`README.md`'s own claim) against live code: name the 7, confirm what
  `doctor-command` is, and confirm what the gate concretely does for each.
- Determine whether routing INC-01's registry/topology writes through
  `runCommand`/gates is additive or a bigger change, grounded in the
  project-scoped-vs-user-scoped distinction already documented in the
  live code (`join.ts`'s 0020-C1 decision comment, `projects.ts`'s "El
  CLI NO usa `runOrchestrated`" comment).
- Produce a concrete implementation brief for registry-engineer (next
  role): which events/stubs to implement, where they hook in, which
  INC-01 write paths (if any) get wired through `runCommand`, what stays
  out of scope.

## Non-goals

- No code changes in `Axiom/packages/orchestrator`, `Axiom/apps/cli`, or
  any other package.
- No redesign of the state machine, the `CommandId` union, or the gate
  mechanism.
- No implementation of any of the 15/19 stubbed intent commands.
- No resolution of the parent roadmap's Q3/Q4 (untouched, out of scope
  for this chain).
- No new event-bus framework, message queue, or pub/sub abstraction —
  the source doc's "simple internal events" intent is explicitly a
  lightweight concept (update caches, validate metadata, enable
  traceability), not infrastructure, per `Axiom.SDD/AGENTS.md`'s bootstrap
  limits ("no heavy multi-agent orchestration frameworks", "no complex
  metadata systems" unless explicitly requested).

## Acceptance criteria

- [x] Every file in `@axiom/orchestrator/src/` and its tests was read in
      full; findings are grounded in the actual code, not the package
      name or the roadmap's paraphrase.
- [x] `apps/cli/src/commands/intent.ts` was read in full and its actual
      relationship (or lack thereof) to `@axiom/orchestrator`'s `CommandId`
      union is stated explicitly.
- [x] The exact count and identity of `CommandId` members is confirmed
      from the union itself (not from `README.md`'s summary line), and
      any discrepancy between `README.md`'s "7 lifecycle + 15 intent" and
      the live code is flagged.
- [x] The 9 source-doc events are mapped 1:1 to an existing equivalent, a
      stub, or "no counterpart", with the exact `CommandId` name cited
      where applicable.
- [x] The 7 CLI commands that call `runOrchestrated` are named exactly,
      with file:line references, and what the gate concretely does for
      each (precondition list, target state) is stated.
- [x] The `doctor-command` state-machine entry's actual (non-)usage from
      the CLI is confirmed by direct code search, not assumed.
- [x] The additive-vs-bigger-change judgment for "wire INC-01 writes
      through runCommand/gates" is grounded in specific, cited code
      (the `join.ts` 0020-C1 comment and the `projects.ts` "no usa
      runOrchestrated" comment), not asserted without evidence.
- [x] A concrete registry-engineer brief is produced: exact stubs/events
      to implement, exact hook points, exact scope boundary.
- [x] No files were modified under `Axiom/`.
- [x] Validation followed `Axiom.SDD/AGENTS.md`'s fallback rule (no code
      changed, so the exact "no validation command" statement applies).

## Open questions

None blocking this increment's own closure. One judgment call is made
explicitly below (the roadmap's "additive" framing needs a scope
correction, not a full reversal) rather than left as an open question,
since the audit has enough evidence to answer it directly.

Carried forward, unresolved, not this increment's job: parent roadmap's
Q3 (`axiom.spec/` prefix survival) and Q4 (capability/provider/MCP
folding) — both untouched, consistent with the entire INC-01 chain.

New open question for **registry-engineer** (next role), not blocking
this increment: should the new local-cache/traceability behavior for
increment/bug/plan/repo events live as a small, dedicated addition inside
`@axiom/orchestrator` (e.g. an optional `onTransition` hook parameter to
`runCommand`), or as a thin wrapper package/module outside it that
subscribes to `RunResult`/`TraceEntry` values the CLI already receives?
This audit recommends the latter (see "Registry-engineer brief" below)
but does not mandate it, since it is an implementation-shape decision, not
an audit finding.

## Assumptions

- The parent roadmap's own "7 comandos CLI pasan por el gate" claim
  (sourced from `Axiom/README.md`) is treated as a hypothesis to verify
  against code, not as ground truth — verified below, and found accurate
  by name-count but incomplete in one respect (see Finding 3).
- The source doc's 9 event names are taken verbatim from the user's task
  description (`IncrementCreated`, `IncrementStatusChanged`,
  `PlanCreated`, `PlanLinkedToIncrement`, `BugCreated`, `AdrLinked`,
  `TechnicalContextUpdated`, `SkillInstalled`, `RepoRegistered`), since
  the source documents themselves are not present in this workspace (see
  "Source material caveat" above).
- "Wiring a write through `runCommand`/gates" is interpreted as: the write
  becomes the `fn` callback of a `runCommand(commandId, ctx, fn)` call, so
  it only executes if `gateFor` passes and is recorded in a `TraceEntry`.
  This is the only mechanism the existing runner exposes; no alternate
  interpretation was assumed.

## Implementation notes

### Finding 1 — `@axiom/orchestrator`'s actual shape: 27 `CommandId` values, not "7 + 15 = 22"

`Axiom/packages/orchestrator/src/types.ts`'s own header comment (line 14)
literally says "el gate retorna `reason: 'not-implemented'`... AD-02...
`CommandId` usa el sufijo `-command`" and separately claims "22 commands"
nowhere explicit in prose, but the state machine's own test file
(`tests/state-machine.test.ts`, lines 6-10) states directly: **"la matriz
tiene 7 lifecycle + 19 intent entries (el design lista 19 intent
commands; el proposal/spec mencionan 15 pero la lista explícita del
design tiene 19 — seguimos la lista del design, que es la fuente
autoritativa del shape)."**

Counting the literal `CommandId` union in `types.ts`: **27 total members**
— 8 lifecycle-shaped entries (`init-command`, `join-command`,
`configure-command`, `sync-command`, `start-command`, `audit-command`,
`doctor-command`, `upgrade-command`) + 19 intent-shaped entries
(`axiom-increment-new-command` through `axiom-external-sync-command`).
`STATE_MACHINE` in `state-machine.ts` has exactly one entry per
`CommandId` (27 entries, confirmed by direct read, not inferred).

**Correction to the parent roadmap and to `Axiom/README.md`**: both say
"7 lifecycle + 15 intent commands." The lifecycle count of 7 undercounts
by one (`doctor-command` exists in the matrix as an 8th lifecycle-shaped
entry — see Finding 3 for why it is plausibly omitted from the "7" count
in practice). The intent count of 15 is stale; the orchestrator's own
test suite already documents this exact discrepancy and resolves it in
favor of 19, treating the design doc as authoritative over the
proposal/spec and over `README.md`'s summary line. This is a real,
pre-existing doc-vs-code drift internal to `Axiom`, not something INC-01
or this audit introduced — flagged here because the parent roadmap's own
diff hypothesis repeats the stale "15" figure.

### Finding 2 — the CLI's `intent.ts` is architecturally unrelated to `@axiom/orchestrator`'s intent-command stubs

`Axiom/apps/cli/src/commands/intent.ts` (`INTENT_CHAINS`,
`runIntentCommand`) implements **chain wrappers** per spec 0028: each
"intent" (`increment-new`, `bug-new`, `implement-role`) is a fixed
sequence of calls into `runIncrementSubcommand`/`runBugSubcommand`/
`runRoleSubcommand` (defined in `axiom-increment.ts`/`axiom-bug.ts`/
`axiom-role.ts`). This file has **zero imports from `@axiom/orchestrator`**
(confirmed by direct read and grep) and its `IntentChain`/`IntentArgs`
types share no structural relationship with `CommandId` or `GateResult`.

Despite the naming coincidence (`intent.ts`'s intent id `increment-new`
vs. the orchestrator's `CommandId` `axiom-increment-new-command`), these
are two **independent, unconnected mechanisms** that happen to describe
similar user-facing concepts:

- `@axiom/orchestrator`'s `axiom-increment-new-command` etc. are inert
  stubs in a `Record<CommandId, StateMachineEntry>` — calling
  `gateFor('axiom-increment-new-command', ctx)` always returns
  `pass: false, reason: 'not-implemented'`. Nothing in `apps/cli` ever
  calls `gateFor`/`runCommand` with any of the 19 intent `CommandId`
  values (confirmed: `grep` for `'axiom-increment-new-command'` etc.
  across `apps/cli/src` returns zero call sites; the only appearances are
  inside `@axiom/orchestrator`'s own source and tests).
- `axiom-increment.ts`'s `runIncrementSubcommand` (called by `intent.ts`'s
  chain) is a fully working, non-stub implementation today — it creates
  real increment artifacts on disk. It does **not** go through
  `runOrchestrated`/`runCommand`/`gateFor` at all (confirmed: zero
  `@axiom/orchestrator` imports in `axiom-increment.ts`).

**Consequence for the parent roadmap's diff hypothesis**: "implement the
currently-stubbed intent commands relevant to the new registry/manifest
operations from INC-01" is not a meaningful instruction as literally
written, because the 19 stubbed `CommandId` intent entries have no
relationship to INC-01's registry/manifest write paths at all — none of
them are `RepoRegistered`-shaped or registry-shaped; they are all
increment/bug/role lifecycle actions (`axiom-increment-new-command`,
`axiom-bug-create-command`, `axiom-role-apply-command`, etc.). The
roadmap's INC-17 (a later, separate increment) already flagged
`intent.ts` for its own audit against the "general command router"
concept — that is the correct home for reconciling `intent.ts` against
the orchestrator's stubs, not INC-02. This audit does not expand INC-02's
scope to cover that; it only reports the finding so INC-02 is not
misdirected into "materializing" one of the 19 stubs under a false
assumption that doing so relates to INC-01's registry work.

### Finding 3 — the 7 gated CLI commands, named exactly, and what the gate does for each

Confirmed by direct search (`runOrchestrated(` call sites) across
`Axiom/apps/cli/src/commands/*.ts`:

| # | File:line | `CommandId` | Preconditions (`gateFor` checks) | `targetState` |
|---|---|---|---|---|
| 1 | `audit.ts:158` | `audit-command` | `projectResolved`, `hasInitJson` | `null` (read-only) |
| 2 | `configure.ts:327` | `configure-command` | `projectResolved`, `hasInitJson` | `'specified'` |
| 3 | `init.ts:484` | `init-command` | *(none — `preconditions: []`)* | `'draft'` |
| 4 | `join.ts:179` | `join-command` | `projectResolved`, `hasInitJson` | `null` |
| 5 | `start.ts:100` | `start-command` | `projectResolved`, `hasInitJson` | `'in_progress'` |
| 6 | `sync.ts:390` | `sync-command` | `projectResolved`, `hasInitJson` | `null` |
| 7 | `upgrade.ts:123` | `upgrade-command` | `projectResolved`, `hasInitJson` | `null` |

**What the gate concretely does** (`gates.ts`'s `gateFor`, pure function,
no I/O of its own beyond what the precondition predicates do): looks up
`STATE_MACHINE[command]`, runs each `Precondition.evaluate(ctx)` in order,
short-circuits on the first failure. `projectResolved` and `hasInitJson`
are the only two non-trivial preconditions in live use by these 7 — both
do a single `fs.existsSync` check (`axiom.yaml` presence;
`.sdd/<projectName>/init.json` presence, respectively). `runCommand`
(`runner.ts`) wraps this: if the gate fails, it throws a tagged
`GateFailureError` **before** invoking the command's actual logic
(`fn`); if the gate passes, it invokes `fn`, times it, and returns
`{ result, trace }`. The CLI's `runOrchestrated` wrapper
(`_shared.ts:181-188`) is a one-line pass-through that discards the
`TraceEntry` and returns only `result`. There is **no persistence** of
the trace today (`runner.ts`'s own comment, AD-06: "in-process only...
future increment persists to `.sdd/local/orchestrator-trace.jsonl`") and
**no state tracking** (`gates.ts:52`'s own comment: "MVP: el estado
actual NO se rastrea; el matrix declara target pero no source" —
`sourceState` is always `null` in every `GateResult`, even on `pass`).

**`doctor-command` is declared but never invoked from the CLI.** No
`doctor.ts` command file exists; `axiom doctor` is registered directly in
`apps/cli/src/index.ts` (`.command('doctor')`, line 287) and calls
`runDoctorChecks`/`formatReport` from `@axiom/doctor` **directly**, with
zero `@axiom/orchestrator` involvement. `'doctor-command'` as a string
literal appears only inside `@axiom/orchestrator`'s own source and test
files — never in `apps/cli`. This means the state machine's own 8th
lifecycle-shaped entry is dead code from the CLI's perspective, and the
practical, currently-true count of "CLI commands wired through the gate"
is **7**, matching `README.md`'s number by coincidence of a different
undercount (`README.md` never lists `doctor-command` as the 8th
lifecycle entry at all, so its "7" is arguably the count of gated CLI call
sites, which is correct, not the count of lifecycle-shaped
`StateMachineEntry` values, which is 8). This nuance was not previously
documented anywhere in the chain and is worth carrying forward as a
precise, disambiguated fact rather than the ambiguous "7 lifecycle"
phrasing repeated by both `README.md` and the parent roadmap.

### Finding 4 — the additive judgment for "wire INC-01 writes through runCommand/gates" needs a scope correction, not a reversal

The parent roadmap's registry-engineer instruction ("wire INC-01
registry/topology writes through `runCommand`/gates instead of direct
writes") is evaluated against two pieces of direct, already-existing
evidence in the live code, both predating this audit:

1. **`join.ts`'s own 0020-C1 design comment** (lines 161-171, read in
   full): *"Decisión (0020-C1): el write de `members.yaml` corre dentro
   de `runOrchestrated` (gate + trace + state project-scoped); el
   auto-registro user-level corre FUERA del orquestrador, en
   best-effort (mismo patrón que `init.ts`)."* This is not a gap or an
   oversight — it is an explicit, already-shipped architectural decision
   that **project-scoped** writes (`members.yaml`, inside `.sdd/`) go
   through the gate, while **user-level** writes (the global
   `~/.axiom/projects.yml` registry) deliberately do not, and are
   best-effort so a registry hiccup never blocks the underlying project
   operation.
2. **`projects.ts`'s own header comment** (lines 16-19, read in full):
   *"El CLI NO usa `runOrchestrated` (mismo patrón que `audit`, `model`,
   `components`, `skills`). El registry es user-level, no
   project-scoped."* (Note: this comment is itself stale on one point —
   `audit.ts` **does** call `runOrchestrated`, per Finding 3's table
   above — but its structural point about the registry being user-level
   is independently correct and consistent with `join.ts`'s comment.)

**Why this matters for "additive or bigger than assumed":** the
orchestrator's `gateFor`/`runCommand` machinery is fundamentally
**project-scoped** — every existing `Precondition` (`notAlreadyInitialized`,
`projectResolved`, `hasInitJson`) reads from `ProjectContextLike.rootPath`/
`projectName`, i.e. a single resolved project. INC-01's registry writes
(`addProjectV2`, `saveRegistryV2`, etc., in `@axiom/user-workspace`) are
**not** project-scoped — they mutate `~/.axiom/projects.yml`, a
single file shared across every project on the machine, keyed by
`homeDir`, not `rootPath`. `repo.ts`'s `runRepoAttach` and `projects.ts`'s
`runProjectsAdd`/`runProjectsJoin` can be called from **outside** any
resolved project directory at all (attaching a brand-new repo to a
project entry does not require a pre-existing `axiom.yaml` at the repo
being attached). Feeding these operations through `gateFor` would require
either: (a) inventing a *new* precondition class that operates on
`homeDir`/registry state instead of `rootPath`/project state — a genuine
new axis the state machine does not have today, not "implementing an
existing stub"; or (b) constructing a synthetic/fake `ProjectContextLike`
just to satisfy the gate's shape, which would make the gate's
`projectResolved`/`hasInitJson` preconditions meaningless noise for these
call sites (they would need to be skipped or trivially satisfied, since a
repo being newly attached usually is not yet "resolved").

**Verdict: the roadmap's "additive" framing is correct at the level of
"do not build a parallel event system," but the specific instruction
"wire INC-01 registry/topology writes through runCommand/gates instead of
direct writes" is not additive as literally stated — it contradicts an
already-shipped, explicit design decision (0020-C1) that these two
concerns (project lifecycle gate vs. user-level registry) are
deliberately kept separate.** Reversing that decision to force registry
writes through the project-scoped gate would be a real, non-trivial
architecture change (new precondition class, new `ProjectContextLike`
variant or a second gate axis), not a matter of "implementing a stub,"
and is exactly the kind of unrequested cross-package expansion
`Axiom.SDD/AGENTS.md`'s bootstrap limits caution against. The roadmap's
own INC-01 chain (all five closed increments) never suggested this either
— `join.ts`'s auto-registration and `init.ts`'s auto-registration were
both implemented, reviewed, and closed as "outside the orchestrator,
best-effort," with no dissent from any of the five roles.

**What genuinely is additive**, consistent with the source doc's actual
framing ("simple internal events... update local caches, validate
metadata, enable traceability") and requiring no new precondition axis:
using the `RunResult<T>`/`TraceEntry` values the 7 already-gated commands
already produce today (currently discarded by `runOrchestrated`) as the
trigger point for lightweight, in-process notifications when a
project-scoped lifecycle transition happens (e.g. `configure-command`
succeeding could notify "config materialized," `init-command` succeeding
could notify "project drafted"). This is additive because it only
observes commands that already pass through the gate — it does not ask
the gate to cover commands that structurally do not fit its
project-scoped shape.

### Mapping table — 9 source-doc events onto existing `CommandId`/code

| Source-doc event | Existing equivalent (exact name) | Status | Notes |
|---|---|---|---|
| `IncrementCreated` | `axiom-increment-new-command` (`CommandId` stub) *or* `axiom-increment.ts`'s `runIncrementSubcommand('create', ...)` (live, working code) | **Stub exists, but the live equivalent is a separate, unrelated code path (Finding 2)** | The stub is inert; the real "increment created" event source, if built, should hook `axiom-increment.ts`'s `create` subcommand, not the orchestrator stub — implementing the stub alone would not fire on any real `axiom increment create` invocation today. |
| `IncrementStatusChanged` | No `CommandId` equivalent. Closest stubs: `axiom-increment-refine-command`, `axiom-increment-change-command`, `axiom-increment-plan-command`, `axiom-increment-plan-approve-command`, `axiom-increment-verify-command`, `axiom-increment-archive-command` (each a distinct lifecycle transition, not one generic "status changed" event) | **No 1:1 counterpart** | The orchestrator models each transition as its own stub `CommandId`, not a single generic status-change event. Live code: `axiom-increment.ts`'s own subcommand dispatch already changes status on disk today, independent of these stubs. |
| `PlanCreated` | No `CommandId` equivalent found. No `axiom-plan.ts`-equivalent command file was located under `apps/cli/src/commands/` in this pass (the roadmap's own INC-06 already flags plan commands as needing audit — not re-audited here, out of scope) | **No counterpart confirmed in this pass** | Flagged, not resolved — INC-06's job, per the parent roadmap's own phase sequencing. |
| `PlanLinkedToIncrement` | Same as above | **No counterpart confirmed in this pass** | Same caveat. |
| `BugCreated` | `axiom-bug-new-command` (`CommandId` stub) *or* `axiom-bug.ts`'s `runBugSubcommand('create', ...)` (live) | **Same Finding-2 split as `IncrementCreated`** | Identical structural situation: stub inert, real create path separate and unrelated. |
| `AdrLinked` | No `CommandId` equivalent; no `adr.ts`/`decision.ts` command file found under `apps/cli/src/commands/` (consistent with the parent roadmap's own INC-11 finding: "not found among audited packages/commands") | **No counterpart, additive** | Matches the parent roadmap's own INC-11 classification exactly; nothing new found in this audit. |
| `TechnicalContextUpdated` | No dedicated command/package found (consistent with the parent roadmap's own INC-10 finding) | **No counterpart confirmed in this pass** | Not independently re-audited here (out of INC-02's scope per the roadmap's own phase sequencing — INC-10's job). |
| `SkillInstalled` | No `CommandId` equivalent. `@axiom/skills`'s `apply.ts`/`materialize.ts` (not read in this pass) are the likely real hook point per the parent roadmap's INC-09 finding | **No counterpart in the orchestrator; likely lives in `@axiom/skills` instead** | Not independently re-audited here — INC-09's job, out of INC-02's scope. |
| `RepoRegistered` | No `CommandId` equivalent. Closest conceptual match: `repo.ts`'s `runRepoAttach` / `projects.ts`'s `runProjectsAdd`/`runProjectsJoin` / `init.ts`/`join.ts`'s auto-registration — all call INC-01's `addProjectV2` directly, **outside** `runOrchestrated` by explicit design (Finding 4) | **No stub exists for this at all; the live write path exists but is deliberately ungated** | This is the one event with a directly relevant, already-implemented live write path (INC-01's registry writes) — and Finding 4 shows gating it the way the roadmap suggested is not a clean fit. See "Registry-engineer brief" below for the recommended minimal alternative. |

Six of the nine events have no `CommandId` stub to "implement" at all
(`PlanCreated`, `PlanLinkedToIncrement`, `AdrLinked`,
`TechnicalContextUpdated`, `SkillInstalled`, `RepoRegistered`). Two
(`IncrementCreated`, `BugCreated`) have a same-named-in-spirit stub that
is structurally disconnected from the real, live write path (Finding 2).
One (`IncrementStatusChanged`) maps to six separate stubs, not one. This
substantially narrows what "implement the currently-stubbed intent
commands relevant to INC-01" can mean in practice: **none of the 19
existing stubs are actually about registry/topology operations** — the
one event that is (`RepoRegistered`) has no stub at all, and its live
write path is intentionally outside the gate.

## Validation

`No validation command was found. Performed best-effort validation by inspecting changed files and checking consistency against the requested behavior.`

No code was changed in this increment (audit-only), so the
`Axiom/package.json` validation commands (`npm run build`, `npm test`,
`npm run typecheck`) were not run, consistent with `Axiom.SDD/AGENTS.md`'s
own guidance and the same posture the parent roadmap and every
migration-engineer-only step in the INC-01 chain took.

Best-effort validation performed for this audit:
- Read every file in `Axiom/packages/orchestrator/src/` (`state-machine.ts`,
  `gates.ts`, `runner.ts`, `types.ts`, `index.ts`) and
  `Axiom/packages/orchestrator/tests/state-machine.test.ts` in full.
- Read `Axiom/apps/cli/src/commands/intent.ts` in full.
- Read `Axiom/apps/cli/src/commands/_shared.ts` in full (the sole
  `@axiom/orchestrator` consumer outside the package itself, confirmed by
  a workspace-wide grep).
- Directly greped every `runOrchestrated(` call site across
  `apps/cli/src/commands/*.ts` and inspected the `CommandId` literal
  passed at each (7 confirmed: `audit`, `configure`, `init`, `join`,
  `start`, `sync`, `upgrade`).
- Directly greped for `'doctor-command'` across the entire `Axiom/`
  workspace and confirmed zero call sites outside
  `@axiom/orchestrator`'s own source/tests, and confirmed `axiom doctor`'s
  actual CLI registration (`apps/cli/src/index.ts`) calls `@axiom/doctor`
  directly.
- Read `join.ts`'s 0020-C1 design comment and `runJoin`'s implementation
  (lines 150-230) to confirm the project-scoped-vs-user-level write split
  is real, shipped behavior, not a stale comment.
- Read `projects.ts`'s header comment and confirmed (by grep) it has zero
  `runOrchestrated` calls.
- Read `repo.ts` and confirmed (by grep) it has zero `runOrchestrated`
  calls either.
- Read `axiom-increment.ts`'s subcommand dispatch and confirmed (by grep)
  zero `@axiom/orchestrator` imports.
- Counted the literal `CommandId` union members in `types.ts` (27) against
  `README.md`'s "7 lifecycle + 15 intent" claim (22) and the test file's
  own "7 + 19 = 26" comment (which itself omits `doctor-command` from its
  explicit lifecycle list before immediately listing it as an 8th entry
  two lines later — the test's own `LIFECYCLE_COMMANDS` array has 8
  entries including `doctor-command`, confirming 8 + 19 = 27 is the
  correct total, matching the union count exactly).
- Did not re-read `@axiom/skills`, `@axiom/topology`'s full loader, or any
  `axiom-plan.ts`/ADR/technical-context equivalent — those are explicitly
  the parent roadmap's INC-09/INC-10/INC-11's own scope, not INC-02's, and
  are reported here as "not counterpart confirmed in this pass" rather
  than guessed.

## Result

Audited `@axiom/orchestrator` and the CLI's `intent.ts` surface in full
and produced four grounded findings that revise the parent roadmap's
INC-02 diff hypothesis:

1. The `CommandId` union has **27** members (8 lifecycle-shaped + 19
   intent-shaped), not "7 lifecycle + 15 intent" as `README.md` and the
   parent roadmap both state — the orchestrator's own test suite already
   documents and resolves this exact discrepancy in favor of 19 intent
   commands, which this audit independently confirms by direct count.
2. The CLI's `apps/cli/src/commands/intent.ts` (`INTENT_CHAINS`,
   spec 0028) is **architecturally unrelated** to `@axiom/orchestrator`'s
   19 intent-command stubs, despite sharing similar names — it is a chain
   wrapper over already-working `axiom-increment.ts`/`axiom-bug.ts`/
   `axiom-role.ts` subcommands that never touch `@axiom/orchestrator`.
   "Implementing a stub" would not make any real `axiom increment
   create`/`axiom bug new` invocation start emitting anything, because
   those real invocations do not run through the stub at all.
3. Exactly **7** CLI commands call `runOrchestrated`
   (`audit`, `configure`, `init`, `join`, `start`, `sync`, `upgrade`,
   with file:line references above); `doctor-command` is an 8th
   lifecycle-shaped `STATE_MACHINE` entry that is never invoked from the
   CLI at all (`axiom doctor` bypasses the orchestrator entirely, calling
   `@axiom/doctor` directly). The gate itself is a pure, stateless
   precondition check (`fs.existsSync` on `axiom.yaml`/`init.json`); the
   runner adds timing/trace, currently discarded, never persisted.
4. The roadmap's instruction to "wire INC-01 registry/topology writes
   through `runCommand`/gates instead of direct writes" contradicts an
   already-shipped, explicit design decision (0020-C1, documented in
   `join.ts`'s own comments and independently corroborated by
   `projects.ts`'s header comment): project-scoped writes go through the
   gate, user-level registry writes (which is what all of INC-01's
   `*V2` functions are) deliberately do not, because the gate's
   preconditions are inherently project-scoped (`rootPath`/`projectName`)
   and the registry is keyed by `homeDir`, shared across every project.
   Forcing registry writes through the gate would require a genuinely new
   precondition axis, which is a real architecture change, not "additive."

The 9-event mapping table shows 6 of 9 source-doc events have no
`CommandId` counterpart at all, 2 have a same-named-in-spirit stub that is
structurally disconnected from the real write path (Finding 2), and only
`RepoRegistered` has a directly relevant, already-implemented live write
path — which is precisely the one Finding 4 shows should **not** be
routed through the gate as literally instructed.

### Registry-engineer brief (next role)

Recommended minimal, additive scope for the next role, consistent with
`Axiom.SDD/AGENTS.md`'s bootstrap limits:

1. **Do not implement any of the 19 `CommandId` intent stubs.** None of
   them are registry/topology-shaped (Finding 2, mapping table); doing so
   would not advance INC-01's registry work and would be speculative,
   disconnected scope.
2. **Do not route `addProjectV2`/`saveRegistryV2`/etc. through
   `runCommand`/`gateFor`.** This would contradict the shipped 0020-C1
   decision and requires a new precondition axis the state machine does
   not have (Finding 4) — treat this as explicitly out of scope, not a
   deferred task.
3. **What is genuinely additive and satisfies the source doc's actual
   intent** ("simple internal events... update local caches, validate
   metadata, enable traceability" — not a parallel event-bus): add a
   single, optional, synchronous notification hook that fires on the
   already-existing, already-gated `RunResult`/`TraceEntry` produced by
   the 7 commands that already call `runOrchestrated` — e.g. an optional
   4th parameter to `runCommand`/`runOrchestrated`,
   `onTransition?: (trace: TraceEntry) => void`, called after a
   successful `fn()` execution, before returning. This lets a future,
   equally small consumer (not built by this brief) log/cache "config
   materialized" (`configure-command`) or "project drafted"
   (`init-command`) without inventing new infrastructure — it reuses data
   the runner already computes and currently discards.
4. **For `RepoRegistered` specifically**: the minimal additive step is
   *not* gating the write, but making the existing, already-working
   `addProjectV2` call sites (`init.ts`, `join.ts`, `repo.ts`,
   `projects.ts`) emit the same kind of lightweight, synchronous
   notification directly at their own call site (no gate needed, since
   they already run outside the gate by design) — e.g. a plain callback
   or a one-line console/trace log matching the existing best-effort
   pattern (`init.ts`'s `registryAttempt.outcome`), not a new abstraction.
   This satisfies "structural commands emit events that update
   caches/validate metadata" without touching the state machine at all.
5. **Six of the nine events (`PlanCreated`, `PlanLinkedToIncrement`,
   `AdrLinked`, `TechnicalContextUpdated`, `SkillInstalled`, and the
   generic `IncrementStatusChanged`) have no live write path confirmed in
   this audit's scope.** Do not build speculative hooks for code paths
   that were not confirmed to exist — those six belong to the parent
   roadmap's own INC-06/INC-09/INC-10/INC-11, not INC-02. If
   registry-engineer wants to extend the notification hook from item 3
   to `axiom-increment.ts`/`axiom-bug.ts`'s already-working `create`
   subcommands (the live equivalent of `IncrementCreated`/`BugCreated`),
   that is a reasonable, small extension of the same pattern — but it is
   a judgment call for registry-engineer to make explicitly, not a
   requirement of this brief.
6. **Explicit non-goals to carry forward**: no new event-bus/pub-sub
   package, no persistence of `TraceEntry` (that remains AD-06's own
   named future increment), no change to `CommandId`, `STATE_MACHINE`,
   `gateFor`, or any existing precondition.

## General spec integration

No integration into `Axiom.Spec/general-spec.md` was performed — that
file does not exist in this repo, consistent with every increment in the
INC-01 chain. This audit's stable finding worth flagging for eventual
consolidation (once `general-spec.md` exists, per the same reasoning the
INC-01 chain used): **`@axiom/orchestrator`'s `runCommand`/`gateFor` gate
is project-scoped by construction, and user-level registry writes
(`~/.axiom/projects.yml`) are deliberately kept outside it — this is a
load-bearing architectural boundary (0020-C1), not an implementation gap,
and any future structural-command/event work must respect it.**

## Closure rationale

`Status: pending`, not `closed`, because:

- This increment's own scope (migration-engineer audit: read the
  orchestrator and `intent.ts` in full, map the 9 events, determine the
  additive-vs-bigger-change question, produce the registry-engineer
  brief) is fully complete, evidenced, and documented above.
- However, INC-02 as a whole (per the parent roadmap's own three-role
  sequence: migration-engineer -> registry-engineer -> validator-reviewer)
  has two more roles pending. Neither the registry-engineer's
  implementation nor the validator-reviewer's review has happened yet.
- Consistent with the INC-01 chain's own closure discipline (every
  migration-engineer/schema-writer/registry-engineer/cli-implementer step
  in that chain stayed `pending` until the final role closed the whole
  parent increment), this increment follows the same pattern: `pending`
  for the sub-step, not `closed`, even though this sub-step's own work is
  finished and validated to the extent audit-only work can be.

## Next step recommendation

Launch **registry-engineer** next, per INC-02's subagent sequence. Exact
inputs it needs from this document:

1. **The corrected `CommandId` count and shape** (Finding 1): 27 total
   (8 lifecycle-shaped, including the CLI-unused `doctor-command`; 19
   intent-shaped stubs) — do not plan around the stale "15 intent
   commands" figure.
2. **Finding 2 in full**: `apps/cli/src/commands/intent.ts` is unrelated
   to the orchestrator's stubs; do not treat "materializing a stub" as
   equivalent to "wiring up the real `axiom increment create` flow."
3. **Finding 3's exact 7-command table** (file:line, `CommandId`,
   preconditions, target state) as the precise, current definition of
   "commands that pass through the gate today."
4. **Finding 4's verdict**: do not route INC-01's `*V2` registry writes
   through `runCommand`/`gateFor` — this contradicts the shipped 0020-C1
   decision and would require a new precondition axis, which is
   out-of-scope architecture expansion, not additive implementation.
5. **The "Registry-engineer brief" section above**, items 1-6, as the
   literal scope boundary: implement the optional `onTransition`-style
   hook (or an equivalent, equally minimal shape registry-engineer judges
   better) on the 7 already-gated commands, and a matching lightweight
   notification at the existing, ungated `addProjectV2` call sites for
   `RepoRegistered` — nothing else.
6. **The 9-event mapping table**, as the authoritative record of which
   events have no live counterpart in this audit's scope (six of nine),
   so registry-engineer does not attempt to build hooks for code paths
   that were not confirmed to exist.

After registry-engineer, **validator-reviewer** is the final role in
INC-02's sequence, responsible for confirming the new hook (if
implemented) does not alter any existing `GateResult`/`RunResult`
contract, and for extending relevant `@axiom/doctor` checks or vitest
coverage as appropriate, following the same "extend, don't replace"
discipline the INC-01 chain's own validator-reviewer step used for
MC-001/BC-001.
