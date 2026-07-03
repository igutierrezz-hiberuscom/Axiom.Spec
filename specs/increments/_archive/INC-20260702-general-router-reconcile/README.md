# Increment: Optional general command router — reconcile

Status: closed
Date: 2026-07-02

## Goal

Audit whether anything in `Axiom` today violates addendum §12's rule
("comandos específicos primero, router general opcional después; el router
no debe sustituir la API/CLI estructurada"), reconciling the roadmap's
hypothesis that `apps/cli/src/commands/intent.ts` plus
`@axiom/orchestrator`'s intent commands are "the closest existing analog to
a general router" (`INC-20260702-axiom-redesign-roadmap/README.md`,
INC-17). Close with no code changes if already compliant; otherwise
document the exact violation and recommend a fix.

## Context

This is INC-17 of the reconciled roadmap
(`Axiom.Spec/specs/increments/INC-20260702-axiom-redesign-roadmap/README.md`),
the last increment of Phase E ("MCP"). INC-01 through INC-16 are closed.
Addendum §12 (`axiom_decisiones_sesion_addendum_revision.md`) states:

> Comandos específicos primero. Router general opcional después.
> Preferido: `axiom increment create`, `axiom plan status`, `axiom bug
> link-plan`. Opcional: `axiom run create-increment`, `axiom run
> implement-plan`. El router no debe sustituir la API/CLI estructurada.

The roadmap's original hypothesis named two things as candidate "router"
analogs: (1) `apps/cli/src/commands/intent.ts`, and (2)
`@axiom/orchestrator`'s intent commands (originally counted as 15, then
corrected to 19 by `general-spec.md`'s "MCP tool registration" section,
sourced from INC-02). Planned subagent sequence: migration-engineer (audit)
-> validator-reviewer, no implementer, per the roadmap's own framing that
this was likely already compliant.

## Scope

- Read `Axiom/apps/cli/src/commands/intent.ts` in full and determine what
  it actually does.
- Re-verify the exact current count and implementation status of
  `@axiom/orchestrator`'s intent commands in
  `Axiom/packages/orchestrator/src/state-machine.ts`.
- Determine whether either mechanism is reachable by a user today, and
  whether either steers a user toward itself instead of the structured
  `axiom increment`/`axiom bug`/`axiom plan`/`axiom role` commands.
- Assess addendum §12 compliance and close with no code changes if
  already satisfied, per the established precedent of INC-05/INC-12/INC-16.

## Non-goals

- Implementing any of the 19 stubbed intent commands (that is a separate,
  unscheduled future increment per `general-spec.md`'s existing "Deferred
  tools" / external-plugins notes on `axiom-external-sync-command`).
- Wiring `intent.ts` into `apps/cli/src/index.ts` (would be new product
  surface, not requested, and not necessary for compliance — see Result).
- Renaming or restructuring either mechanism to resolve the naming
  collision described below (documented as a finding only).

## Acceptance criteria

- [x] `intent.ts`'s actual behavior is read and described precisely (not
      assumed from the file name or roadmap hypothesis).
- [x] The exact current count and implementation status of
      `@axiom/orchestrator`'s intent commands is re-verified directly from
      `state-machine.ts`, not carried over from a stale prior count.
- [x] A concrete finding is recorded on whether any code path steers users
      toward a router/intent mechanism instead of the structured commands.
- [x] Closure decision (code change vs. none) is justified by the audit,
      not assumed in advance.
- [x] Recorded in `Axiom.Spec`; no code changed in `Axiom.SDD` or `Axiom`
      unless a genuine violation was found (none was).

## Open questions

None blocking. One non-blocking naming-clarity finding is recorded below.

## Assumptions

- "Router" in addendum §12's sense means a generic dispatch surface
  (`axiom run <action>`) that could compete with or replace structured
  commands — not any internal chain-wrapper utility that happens to share
  the word "intent."

## Implementation notes

No files were changed under `Axiom/packages/`, `Axiom/apps/`, or
`Axiom.SDD/`. This increment is audit/verification-only, per the roadmap's
own subagent sequence (migration-engineer -> validator-reviewer, no
implementer) and per the established closed-with-zero-changes precedent of
INC-05, INC-12, and INC-16.

### Finding 1 — `intent.ts` and the orchestrator's "intent commands" are two unrelated mechanisms, despite the shared name

The roadmap's hypothesis (written before this audit) assumed `intent.ts`
and `@axiom/orchestrator`'s 15/19 `axiom-*-command` entries were the same
thing, or closely related. They are **not**:

- **`intent.ts`** (`Axiom/apps/cli/src/commands/intent.ts`, spec 0028/"E1")
  implements a small, closed catalog of exactly **3** `IntentChain` entries
  (`increment-new`, `bug-new`, `implement-role`). Each is a literal
  **chain wrapper**: it calls the existing structured subcommand functions
  (`runIncrementSubcommand`, `runBugSubcommand`, `runRoleSubcommand`) in a
  fixed sequence (e.g. `increment-new` = `create` -> `refine` ->
  `specify`), aborting on the first non-zero exit code. Its own doc
  comment is explicit about this: "los intents son chain wrappers, NO
  semántica nueva... GATE 0022 honrada." It is not a natural-language or
  generic-action router in the source-doc sense; it is a convenience
  macro over already-structured commands.
- This file is **not imported or registered anywhere in
  `apps/cli/src/index.ts`** (confirmed: no `intent` reference exists in
  `index.ts`, and no other command file imports from `./intent`). It has
  unit tests (`apps/cli/tests/intent.test.ts`) exercising the module
  directly, but there is **no CLI verb, flag, or code path a user can
  invoke to reach `runIntentCommand`/`INTENT_CHAINS` today.** It is
  orphaned library code, not a reachable "optional secondary path" —
  it is not reachable at all.
- **`@axiom/orchestrator`'s intent commands** (`state-machine.ts`) are a
  completely different mechanism from a different spec (0018-C1): a
  closed `CommandId` union of **19** `axiom-*-command` entries (re-counted
  directly from `types.ts`/`state-machine.ts`, confirming INC-02's
  correction from 15 to 19 still holds — no increment since INC-02 added
  or removed an entry). Every one of the 19 carries `notImplemented` as
  its sole, always-failing precondition (`reason: 'not-implemented'`).
  **None have been wired to real logic** by any increment through INC-16,
  including INC-06 (increment/bug/plan commands), INC-11 (ADR/decision
  commands), or INC-13 (MCP tool registration) — those increments added
  real CLI commands and MCP capabilities, but none of them touch
  `state-machine.ts`'s `CommandId` union or its `notImplemented`
  preconditions. Grepping the entire `apps/cli/src` tree for any of the 19
  literal `axiom-*-command` ids returns zero matches outside
  `packages/orchestrator/` itself — they are referenced by no CLI command,
  no TUI screen, and no MCP tool handler. They are dead, unreachable stub
  data, not a working alternate command surface.

Both mechanisms share the word "intent" and both are unreachable by an
end user today, which is why the roadmap's pre-audit hypothesis treated
them as one thing. They are unrelated in spec origin (0028 vs 0018-C1),
shape (a 3-entry chain-wrapper catalog vs. a 19-entry gated state-machine
stub list), and purpose (macro-over-structured-commands vs.
future-intent-command placeholders). This is a naming-clarity gap worth
recording in `general-spec.md`, not a functional defect.

### Finding 2 — Addendum §12 compliance: satisfied, trivially

Addendum §12's rule is that specific structured commands must remain
primary and a general router, if it exists, must be optional and must not
replace them. Applying this to what was found:

- The **actually reachable, actually primary** command surface is exactly
  the structured one addendum §12 prefers: `axiom increment ...`, `axiom
  bug ...`, `axiom plan ...`, `axiom role ...` (registered in
  `apps/cli/src/index.ts` via `registerAxiomIncrement`, etc., confirmed by
  direct inspection), matching the addendum's own preferred-form examples
  almost verbatim.
- **Neither candidate "router" is reachable at all** — `intent.ts` is
  unregistered, and the orchestrator's 19 intent commands are gated by an
  always-failing precondition and are not invoked from any command,
  screen, or tool. A mechanism that cannot be invoked cannot steer a user
  toward itself, and cannot substitute for (`sustituir`) the structured
  CLI, because there is no path by which a user ever reaches it instead of
  the structured commands.
- There is no code path anywhere in `Axiom` today where a user would be
  offered, defaulted to, or redirected toward either "router" candidate in
  place of a structured command. The structured commands are not merely
  "primary among options" — they are the *only* option a user can
  actually execute for increment/bug/plan/role workflows.

Conclusion: addendum §12's rule is satisfied today, and satisfied more
strongly than "router exists as genuinely optional secondary path" —
there is currently no user-facing router at all, optional or otherwise.
This closes clean with no code changes, consistent with INC-05, INC-12,
and INC-16's precedent.

No fix is required. If a future increment wants an actual optional router
(e.g. wiring `intent.ts` into `index.ts` as `axiom run <action>`, per
addendum §12's own optional example), that is new, currently-unrequested
product surface — not a compliance requirement — and should be its own
increment if a concrete need arises.

## Validation

`No validation command was found` does not apply here — a real validation
command exists (`Axiom/package.json`'s `npm test` -> `vitest run`) and was
used narrowly, consistent with this being an audit/verification-only
increment with no code changes to validate broadly.

Ran the directly relevant existing test files to confirm the audit's
factual claims about current behavior (not asserting anything new):

```
npx vitest run apps/cli/tests/intent.test.ts \
  packages/orchestrator/tests/state-machine.test.ts \
  packages/orchestrator/tests/gates.test.ts
```

Result: 3 files, 28 tests, all passing (`apps/cli/tests/intent.test.ts`: 6,
`packages/orchestrator/tests/state-machine.test.ts`: 15,
`packages/orchestrator/tests/gates.test.ts`: 7). This confirms
`intent.ts`'s 3-entry `INTENT_CHAINS` catalog and `state-machine.ts`'s
gate behavior (including the `notImplemented`/`reason: 'not-implemented'`
precondition on all 19 `axiom-*-command` entries) match what this
increment describes, with no drift from a stale assumption.

Additional evidence gathered by direct source inspection (not test-based):
- `grep`-equivalent search of `apps/cli/src/index.ts` and every file under
  `apps/cli/src/commands/` for any reference to `intent.ts`'s exports or
  any of the 19 literal `axiom-*-command` ids: zero matches outside
  `intent.ts`/`state-machine.ts`/`types.ts` themselves.
- Confirmed `apps/cli/src/index.ts` registers `registerAxiomIncrement`,
  `registerAxiomRole`, etc. (the structured commands) as the real CLI
  surface.

## Result

Audit confirms the roadmap's cautious hypothesis ("likely already
compliant") was correct, but for a different and stronger reason than
assumed: `intent.ts` and the orchestrator's intent commands are two
unrelated, both-unreachable mechanisms, not one shared "router" analog.
Addendum §12 is satisfied because there is currently no reachable general
router of any kind competing with the structured commands — not because a
reachable-but-secondary router happens to defer correctly to them. Zero
code changes were made, following the same closure shape as INC-05,
INC-12, and INC-16.

## General spec integration

Added a new "Optional general command router" section to
`Axiom.Spec/general-spec.md`, recording: `intent.ts`'s actual 3-entry
chain-wrapper behavior and its unregistered/unreachable status; the
orchestrator's 19-entry (re-confirmed) `notImplemented` intent-command
stub list and its unreachable status; the naming collision between the
two "intent" concepts (spec 0028 vs. 0018-C1) as a documentation-clarity
note for future readers; and the addendum §12 compliance conclusion. This
is genuinely reusable cross-increment knowledge (a future increment
touching either `intent.ts` or `state-machine.ts` needs to know they are
unrelated) and belongs in the stable spec, not only in this increment's
own history.

## Next step recommendation

Phase E (INC-13 through INC-17) is now complete — all five increments are
closed:

- INC-13: MCP tool registration (`@axiom/mcp-tools`, 16 capability ids).
- INC-14: `get_implementation_context` flagship composite tool (17th
  capability id).
- INC-15: `mcp.yml` project-scoped MCP config + per-adapter generation.
- INC-16: External plugins (Azure DevOps) verified as optional, isolated,
  and correctly non-blocking of Axiom Core.
- INC-17 (this increment): General command router reconciled — confirmed
  no violation of addendum §12, no reachable router exists at all today.

Per the roadmap's own phase ordering, **Phase F ("Bootstrap documental")
starts next**, beginning with **INC-18 — Reconcile bootstrap-from-code**.
The roadmap already scoped INC-18's starting hypothesis:
`@axiom/document-bootstrap` (`Axiom/packages/document-bootstrap/`) renders
already-known `ResolvedVariables` into adapter files (e.g.
`.github/copilot-instructions.md`) with path-guard, team-block
preservation, and atomic writes — but it does not *analyze* a codebase or
a legacy SDD repo to *draft* new technical-context content from scratch.
INC-18's migration-engineer should confirm `document-bootstrap/writer.ts`
is reusable as-is for the final write step, before a bootstrap-analyzer
subagent scopes the actual repo/backend/frontend/qa/architecture analysis
chain (source doc 15.3 "Vía A") that does not exist yet. INC-18 depends on
INC-06 and INC-09/INC-10 (already closed) and INC-07 (also closed per the
roadmap's dependency graph).
