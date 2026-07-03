# Increment: Reconcile increment/bug/plan commands (migration-engineer audit)

Status: pending
Date: 2026-07-02

## Goal

Execute INC-06 of the parent roadmap
(`Axiom.Spec/specs/increments/INC-20260702-axiom-redesign-roadmap/README.md`):
audit `axiom-increment.ts`, `axiom-bug.ts`, `axiom-plan.ts`, and
`axiom-role.ts` (the roadmap explicitly lists `axiom-role.ts` alongside the
other three) against source doc section 5.3 (`metadata.yml` per artifact),
section 6.1 (increment/bug/plan command list), section 6.2 (event model),
and addendum section 11 (internal IDs + `externalRefs`).

This specific increment covers only the **migration-engineer** role: read
every relevant file in full, determine the actual on-disk format these
commands write today, compare field-by-field against the two source
documents' target shapes, and produce a brief for the next role. It does
**not** write new schema types or implementation code.

## Context

Parent increment: `INC-20260702-axiom-redesign-roadmap`, Phase C / INC-06.
The parent roadmap's diff hypothesis for INC-06 was written before any
line-by-line read of these files and said: "likely partial overlap
(commands exist) but possibly different metadata shape than `metadata.yml`
... and no confirmed `externalRefs` support yet." This increment re-verifies
that hypothesis against the actual code, per the parent roadmap's own
instruction not to trust it.

**Filenames confirmed**: `Axiom/apps/cli/src/commands/axiom-increment.ts`,
`axiom-bug.ts`, `axiom-plan.ts`, `axiom-role.ts` all exist exactly as named
in the roadmap (`Glob` confirmed each individually).

## Scope

- Read `axiom-increment.ts`, `axiom-bug.ts`, `axiom-plan.ts`, `axiom-role.ts`
  in full, plus their test files
  (`Axiom/apps/cli/tests/axiom-{increment,bug,plan,role}.test.ts`).
- Read `@axiom/workflow`'s `state-store.ts`, `types.ts`, `state-machine.ts`,
  and `branch-naming.ts` — the shared engine these four commands wrap.
- Determine the actual persisted format (file, shape, fields) these
  commands write today.
- Grep the monorepo for `metadata.yml`/`metadata.yaml`, `externalRef`,
  and `link-plan`/`link-increment`/`link-bug`-style commands to confirm
  or refute the roadmap's "possibly different shape" / "no confirmed
  externalRefs" hypotheses.
- Produce a field-by-field diff against source doc 5.3's `metadata.yml`
  shape and addendum 11's `externalRefs` shape, using the same
  deprecation-warning / hard-break / needs-decision convention as INC-01's
  migration-engineer audit
  (`INC-20260702-registry-manifest-schema-v2/README.md`).
- Determine whether internal IDs are stable/slug-based today (addendum 11).
- Confirm or refute whether linking commands (`increment -> plan`,
  `bug -> plan`, `plan -> increment/bug`) exist today, and under what exact
  names.
- Recommend whether `externalRefs` support is additive or needs a
  schema/version bump, and whether a schema-writer role is needed at all
  for INC-06 (per INC-05's precedent of skipping unneeded roles).

## Non-goals

- No new TypeScript types, Zod schemas, or CLI flags — that is the
  cli-implementer's job (and schema-writer's, only if this audit finds a
  real version-bump need).
- No implementation of `externalRefs`, linking commands, or a
  `metadata.yml` writer.
- No decision on the still-open Q1–Q5 from the parent roadmap; this
  increment does not depend on them being resolved because increments/
  bugs/plans as they exist today are not yet metadata-file-based at all
  (see Result below), so the multi-repo/registry axis is orthogonal to
  this specific finding.
- No code changes anywhere in `Axiom/` or `Axiom.SDD/` in this increment
  (audit + spec authoring only).

## Acceptance criteria

- [x] `axiom-increment.ts`, `axiom-bug.ts`, `axiom-plan.ts`, `axiom-role.ts`
      read in full, plus their four test files.
- [x] `@axiom/workflow`'s `state-store.ts`, `types.ts`, `state-machine.ts`,
      `branch-naming.ts` read in full.
- [x] Monorepo-wide grep performed for `metadata.yml`/`metadata.yaml`,
      `externalRef`, and `link-plan`/`link-increment`/`link-bug` (and
      camelCase variants).
- [x] A field-by-field diff exists comparing the actual persisted shape to
      source doc 5.3's `metadata.yml` shape.
- [x] The `externalRefs` gap is explicitly classified as additive or
      needs-schema-writer, with reasoning.
- [x] The ID-generation/stability finding is stated with the exact file
      and function (or explicit absence) backing it.
- [x] The linking-commands finding is stated with exact current command
      names compared to the roadmap's assumed list.
- [x] Next-role recommendation is explicit (schema-writer vs. skip to
      cli-implementer), per INC-05's precedent of not inventing unneeded
      roles.

## Open questions

None blocking this increment. One question is raised for the
cli-implementer/product-owner to resolve before write work starts (see
"Open question for next role" below), but it does not block this audit.

## Assumptions

- The cli-implementer subagent (next step) will use this document's
  field-by-field diff, ID-scheme finding, and linking-commands finding as
  its starting brief, consistent with the parent roadmap's INC-06 subagent
  sequence (migration-engineer -> schema-writer [conditional] ->
  cli-implementer -> validator-reviewer).
- "Additive" here means: a new optional field/command can be added without
  changing the meaning or validity of any existing persisted state file or
  breaking any existing test, consistent with `Axiom.SDD/AGENTS.md`'s
  prohibition on speculative architecture and unnecessary version-bump
  machinery.

## Implementation notes

### What these four commands actually are

All four files (`axiom-increment.ts`, `axiom-bug.ts`, `axiom-plan.ts`,
`axiom-role.ts`) are thin CLI wrappers over a single shared generic engine,
`@axiom/workflow` (spec 0022, "Lote B"). Each file:

1. Loads a `WorkflowConfig` by parsing `<projectRoot>/axiom.spec/config/
   workflows.yaml` (or a test-injected override path) and finding the
   entry whose `id` matches the workflow (`increment`, `bug`, `plan`, or
   `role`).
2. Loads the current `WorkflowStateRecord` for that workflow via
   `loadWorkflowState(projectRoot, workflowId)`.
3. Calls `applyTransition(config, currentState, command)` — a pure state
   machine lookup (`state-machine.ts`) that finds a transition matching the
   `from` state and `command` name, returns the `to` state or a typed
   `invalid-transition` error.
4. Fires two no-op hooks (`pre-start`/`post-start` or `pre-complete`/
   `post-complete`) via `createHookEngine()` — hooks have no registered
   handlers yet in this MVP.
5. Persists via `saveWorkflowState(projectRoot, newRecord)`.
6. Prints a human message or, with `--json`, the raw `WorkflowStateRecord`.

**There is no per-artifact folder, no `README.md`, and no `metadata.yml`
file written by any of these four commands.** The only file any of them
writes is a single shared state file per project:

```
<projectRoot>/.sdd/local/workflow-state.json
```

whose shape (from `@axiom/workflow/src/state-store.ts`) is:

```json
{
  "schemaVersion": 1,
  "workflows": {
    "increment": {
      "workflowId": "increment",
      "state": "specifying",
      "vars": { "id": "0019", "slug": "operator-control-plane" },
      "updatedAt": "2026-07-02T12:00:00.000Z"
    },
    "bug": { "...": "..." },
    "plan": { "...": "..." },
    "role": { "...": "..." }
  }
}
```

This is a **single shared file with one record per workflow ID** (not one
record per increment/bug/plan instance). A second `axiom-increment create
--id 0020 --slug other` before the first is archived would overwrite the
`increment` key's record entirely (`saveWorkflowState` merges at the
workflow-ID level, not at an artifact-instance level) — confirmed by
reading `saveWorkflowState`'s merge logic (`state-store.ts:303-367`, `next.
workflows = { ...existing.value.workflows, [record.workflowId]: record }`).
This is a materially different data model from source doc 5.3's "one
folder + `metadata.yml` per artifact instance" — not a naming difference,
a structural one: today's model can only track **one in-flight
increment, one in-flight bug, one in-flight plan, and one in-flight role
per project at a time**, because `vars.id`/`vars.slug` live inside the
single per-workflow record, not inside a per-instance file. (`axiom-plan.ts`
partially works around this for `approve` by keying re-init on
`vars.planId !== args.planId`, but this only supports one *active* plan
at a time — approving a second plan overwrites the first plan's record.)

### `README.md` / content files

No test, source file, or grep hit shows any of these four commands writing
prose content (a spec body, a bug description, a plan body) anywhere. The
`--id`/`--slug`/`--title`-type content that source doc 5.3's `README.md`
would hold is not written by these commands at all today. Combined with
the absence of `metadata.yml`, this means **the entire per-artifact
filesystem structure from source doc 5.3 does not exist yet** — this is
not a shape mismatch to reconcile, it is a missing layer.

### Field-by-field diff: target `metadata.yml` (source doc 5.3) vs. actual `workflow-state.json` record

| Source doc 5.3 target (`metadata.yml` per artifact, inferred fields from addendum 11's example + section 5.3's structure) | Actual today (`WorkflowStateRecord`, one shared record per workflow ID) | Classification |
|---|---|---|
| File: `project.spec/increments/INC-.../metadata.yml` (+ sibling `README.md`), one folder per artifact instance | File: `<projectRoot>/.sdd/local/workflow-state.json`, one JSON record per **workflow ID**, not per artifact instance | Hard structural gap — no equivalent exists; this is additive net-new work, not a field rename |
| `id: INC-20260702-183012-a7f3c9` (addendum 11 example — internal ID, timestamp+random-suffix based) | `vars.id` — free-text string supplied by the caller via `--id <id>` (e.g. `0019`); no generator, no stability guarantee, no collision check | Needs-decision: current "ID" is caller-typed, not system-generated |
| `title: ...` | Not persisted anywhere in the workflow record; only `--slug` is captured (used for branch naming, not as a title) | Missing field — additive |
| `status: draft` (artifact-level status) | `state` field exists, but it is the workflow's lifecycle state (`draft\|specifying\|planned\|plan-approved\|in-progress\|verifying\|archived\|failed\|cancelled`), which is conceptually close but scoped to the single shared workflow record, not an artifact instance | Conceptually additive/reusable once artifact-instance scoping exists, but cannot be reused as-is without the instance-scoping fix above |
| `externalRefs: [{provider, type, id, url}]` (addendum 11) | No field of this name or shape anywhere in `WorkflowStateRecord`, `WorkflowStateFile`, or any of the four command files | Confirmed absent — see dedicated finding below |
| (implicit) created/updated timestamps | `updatedAt` (ISO 8601) exists; no `createdAt` | Minor additive gap |
| (implicit) links to other artifacts (plan -> increment/bug) | No link field exists; `axiom-plan.ts`'s only cross-workflow awareness is `axiom-role.ts`'s safety gate, which reads the `plan` workflow's `state` (not an ID/link) to block `role start` if no plan is `plan-approved` | See Linking-commands finding below |

### `externalRefs` finding (addendum 11)

Grepped the entire `Axiom` monorepo for `externalRef` (case-sensitive,
covers `externalRef`/`externalRefs`). Two files match, both unrelated to
addendum 11's shape:

- `apps/cli/src/commands/app-plugins-azure-devops.ts` — a form-field
  definition object for a work-item-creation UI plugin. Fields like
  `title`, `description`, `workItemType`, `assignedTo`, `iterationPath`,
  `tags`, `workItemId` are each tagged `externalRef: true`. This is a
  **boolean UI-form flag** meaning "this form field maps to an Azure
  DevOps work-item field," not the addendum 11 array-of-objects shape
  (`externalRefs: [{ provider, type, id, url }]`) attached to an
  increment/bug's own metadata.
- `app-plugins.ts` — generic plugin registration; matched only because it
  imports/re-exports the Azure DevOps plugin module above.

**Conclusion: `externalRefs` (addendum 11 shape) does not exist anywhere
in the codebase today**, not under this name, not under a differently
named but structurally equivalent field. The Azure DevOps plugin already
proves the *intent* (linking Axiom artifacts to external work items) is
recognized in the product, but it operates entirely inside a UI-plugin
form schema, disconnected from `axiom-increment.ts`/`axiom-bug.ts`/
`axiom-plan.ts`'s actual persisted state.

**Additive vs. schema-writer classification**: Adding `externalRefs`
itself would be trivially additive **once an artifact-instance metadata
file exists to add it to** — a new optional array field on a
`metadata.yml`-equivalent breaks nothing by construction (it is new,
optional, and no current reader/writer touches it). However, this
increment finds that **no such per-instance metadata file exists yet**
(see structural gap above), so `externalRefs` cannot be bolted onto
today's `workflow-state.json` in a way that matches source doc 5.3's
model without first solving the instance-scoping gap. This is not a
`schemaVersion` bump situation in the classic sense (`workflow-state.json`
already has `schemaVersion: 1` and nothing here requires incrementing it,
because we are not changing that file's existing meaning) — it is a
**net-new artifact layer** that needs to be designed, not a migration of
an existing versioned schema. Recommendation: **no schema-writer role
needed for a version bump**, but the cli-implementer step must first
decide (in consultation with a schema-writer only if the shape is
non-trivial) what the new per-instance file looks like before
`externalRefs` has anywhere to live. See Next step recommendation below.

### ID-generation / stability finding (addendum 11)

Addendum 11 states: *"Axiom usa IDs internos por defecto... El ID interno
debe ser estable aunque cambie la integración externa."*

Grepped for `slugifyProjectId` (the pattern named in the task, from
`@axiom/user-workspace`) and any local slugify/ID-generation function
reachable from the four command files:

- `slugifyProjectId` exists only in `@axiom/user-workspace/src/
  registry-id.ts` and is used for **project** IDs in the user-level
  registry (`~/.axiom/registry.json`), unrelated to increments/bugs/plans.
- The only ID/slug-adjacent utility reachable from the four commands is
  `@axiom/workflow/src/branch-naming.ts`'s `slugify(input)` — but this is
  used exclusively to build **Git branch names** from `vars.id`/
  `vars.slug` (e.g. `increment/0019-operator-control-plane`), not to
  generate or validate an artifact's own ID.
- All four command files read `--id`/`--slug` as **plain caller-supplied
  strings** with no format validation beyond "non-empty" (see
  `addIncrementSubcommand`'s `requiresId`/`requiresSlug` checks — length
  `> 0`, nothing else). There is no timestamp+random-suffix generator, no
  collision detection, and no persistence of an ID once assigned beyond
  the single shared `vars.id` field in `workflow-state.json`.

**Conclusion: internal IDs for increments/bugs/plans/roles are not
system-generated today.** They are free-text, caller-typed values with no
stability guarantee beyond "whatever string the user or script passed to
`--id` last." This is a materially different posture from addendum 11's
"Axiom debe ser el sistema de registro, no la IA" principle (source doc
1.4) — nothing currently prevents a caller (human or AI-driven script)
from typing an inconsistent or colliding ID. This finding is a
prerequisite for `externalRefs` to be meaningful: an `externalRefs` array
attached to an unstable, caller-typed ID would defeat the addendum's own
stated purpose ("el ID interno debe ser estable").

### Linking-commands finding (source doc 6.1)

Source doc 6.1 lists these commands as part of the minimum command set:

```
axiom increment link-plan
axiom bug link-plan
axiom plan link-increment
axiom plan link-bug
```

Grepped the entire monorepo for `link-plan`, `link-increment`, `link-bug`,
and the camelCase equivalents (`linkPlan`, `linkIncrement`, `linkBug`):
**zero matches**. None of these commands exist under any name.

The closest existing analog is **not a link, but a gate**:
`axiom-role.ts`'s `checkPlanIsApproved` function (lines 257-298) reads the
`plan` workflow's single shared state record and blocks `axiom-role start`
with a `SAFETY GATE` error message unless that record's `state ===
'plan-approved'`. This is a one-way, hardcoded dependency check (role ->
plan), not a general-purpose link/reference mechanism between arbitrary
increment/bug/plan instances, and it only works because there is
currently at most one active plan per project (see structural gap above).

**Confirmed exact current command surface** for the four workflows (all
subcommands, no aliases):

| Workflow | Current subcommands | Roadmap/source-doc-6.1 assumed names |
|---|---|---|
| `axiom-increment` | `create`, `refine`, `specify`, `change`, `plan`, `plan-approve`, `verify`, `archive` | `create`, `update`, `status`, `link-plan`, `archive` |
| `axiom-bug` | `create`, `fix-plan`, `verify`, `archive` | `create`, `update`, `status`, `link-plan`, `archive` |
| `axiom-plan` | `approve` (only) | `create`, `update`, `status`, `link-increment`, `link-bug`, `add-target-repo`, `set-scope` |
| `axiom-role` | `start`, `apply`, `complete`, `sync-graph` | (not in source doc 6.1's list at all — `role` is not one of the source docs' structural artifact types; it is this codebase's own lifecycle-execution concept) |

None of the current command names match the source doc's assumed names
1:1. This is a genuine command-surface gap, not just a metadata-shape gap:
even `create`/`archive` (which exist under the same name in both lists)
have entirely different semantics today (workflow state transitions on a
single shared record) versus the source doc's implied semantics (creating/
archiving a `metadata.yml`-backed artifact instance, with `update`/
`status` as separate read/write operations on that file). `axiom-plan`
today has no `create` at all (per its own header comment: "el plan se
crea implícitamente cuando se necesita aprobar algo") — this is the
widest gap of the three (`increment`/`bug`/`plan`), since source doc 6.1
expects `plan create` as a first-class command.

### Event model finding (source doc 6.2)

Source doc 6.2 asks for internal events (`IncrementCreated`,
`IncrementStatusChanged`, `PlanCreated`, `PlanLinkedToIncrement`,
`BugCreated`, `AdrLinked`, `TechnicalContextUpdated`, `SkillInstalled`,
`RepoRegistered`) that "actualizan caches locales, validan metadata y
permiten trazabilidad." No event of any of these names, or any generic
event-emission mechanism, was found in any of the four command files or in
`@axiom/workflow`. The closest analog is the hook engine
(`createHookEngine()`/`runHook`), which fires `pre-start`/`post-start` (or
`pre-complete`/`post-complete`) — but these are lifecycle hook points, not
named domain events, and per the file header comments they are
**currently no-op** ("Los hooks son no-op hasta que se registren specs
adicionales"). This matches the parent roadmap's INC-02 finding (already
audited and closed) that `@axiom/orchestrator`'s 15 intent commands are
the closer analog to an event model and are mostly stubs — INC-06's four
commands do not add a competing event mechanism, they use a narrower,
already-identified-as-partial hook mechanism. No new finding beyond what
INC-02 already established; flagged here only for completeness since
source doc 6.2 was explicitly in this increment's scope.

## Validation

`No validation command was found. Performed best-effort validation by
inspecting changed files and checking consistency against the requested
behavior.`

This increment is audit-only (no code changed), so the applicable
best-effort validation was:
- Every claim above is sourced from a direct, full read of
  `axiom-increment.ts`, `axiom-bug.ts`, `axiom-plan.ts`, `axiom-role.ts`,
  their four test files, and `@axiom/workflow`'s `state-store.ts`,
  `types.ts`, `state-machine.ts`, `branch-naming.ts`.
- The `metadata.yml`/`externalRef`/linking-command absence claims are each
  backed by a monorepo-wide grep (not inferred from file/package names
  alone), with the two `externalRef` hits inspected directly to confirm
  they are unrelated (Azure DevOps plugin form-field flags).
- The ID-generation claim is backed by reading `branch-naming.ts`'s
  `slugify` (branch-name-only) and confirming `slugifyProjectId` is scoped
  to `@axiom/user-workspace`'s project registry, unrelated to increment/
  bug/plan IDs.
- `Axiom/package.json`'s real validation commands (`npm run build`, `npm
  test`, `npm run doctor`, `npm run typecheck`) were not run, because no
  code was changed in `Axiom/` for this increment — running them would
  only assert pre-existing state, consistent with the same reasoning
  INC-01's and the parent roadmap's audit increments used.

## Result

Completed the migration-engineer audit for INC-06. The parent roadmap's
diff hypothesis ("likely partial overlap... possibly different metadata
shape... no confirmed externalRefs support") undersold the actual gap:

1. **No `metadata.yml`, no `README.md`, no per-artifact folder exists at
   all.** `axiom-increment.ts`/`axiom-bug.ts`/`axiom-plan.ts`/
   `axiom-role.ts` are thin wrappers around a single shared lifecycle
   state machine (`@axiom/workflow`) that persists one JSON record **per
   workflow ID** (not per artifact instance) to
   `<projectRoot>/.sdd/local/workflow-state.json`. This means today's
   implementation can track at most one in-flight increment, one
   in-flight bug, one in-flight plan, and one in-flight role per project
   simultaneously — a structural limitation, not a naming or field-shape
   difference.
2. **`externalRefs` (addendum 11) does not exist anywhere**, under any
   name. The only `externalRef`-named construct in the codebase is an
   unrelated boolean UI-form-field flag inside the Azure DevOps plugin.
3. **Internal IDs are not system-generated.** They are free-text
   `--id`/`--slug` CLI arguments with no format validation, generator, or
   collision check — contradicting addendum 11's "el ID interno debe ser
   estable" expectation, since nothing prevents ID reuse/collision today.
4. **None of source doc 6.1's `link-plan`/`link-increment`/`link-bug`
   commands exist**, under any name. The closest analog
   (`axiom-role.ts`'s `checkPlanIsApproved` safety gate) is a one-way
   hardcoded state check, not a link/reference mechanism.
5. **Command surface names differ from source doc 6.1 across the board**,
   including different semantics even where names coincide (`create`,
   `archive`). `axiom-plan` has no `create` subcommand at all today.
6. Source doc 6.2's event model has no new finding beyond what INC-02
   already established (hooks exist but are no-op; `@axiom/orchestrator`
   remains the closer, still-partial analog).

`externalRefs` support is **additive in isolation** (a new optional field
breaks nothing), but it has **no artifact-instance metadata file to attach
to today** — building that file is net-new work, not a migration of an
existing schema, and does not require a schema-version bump on the
existing `workflow-state.json` (`schemaVersion: 1` stays valid and
unchanged; a new, separate metadata file would carry its own initial
version).

## General spec integration

No integration into a `general-spec.md` was performed — that file does
not exist in this repo (confirmed by the parent roadmap and by INC-01's
audit; the closest equivalents are `Axiom.Spec/specs/00_Resumen_
Ejecutivo.md` through `08_Glosario.md`). Those documents were not
modified. This increment's findings (no per-instance metadata file, no
externalRefs, no linking commands, non-generated IDs) are specific enough
to INC-06's own scope that consolidating them into a general spec now
would be premature — they belong in the cli-implementer's design once the
next step below is executed, consistent with INC-01's own reasoning for
deferring `03_Modelo_Operativo_y_Datos.md` updates until its schema-writer
step lands.

## Open question for next role

Not blocking this audit, but must be answered before the cli-implementer
step designs the new per-instance file: should the new artifact-instance
metadata (title, `externalRefs`, per-instance status/links) be (a) a new
file colocated per artifact instance as source doc 5.3 describes
(`project.spec/increments/INC-.../metadata.yml` + `README.md`), which
requires deciding where these folders live relative to today's
`axiom.spec/config/workflows.yaml`-based project layout, or (b) an
extension of the existing single shared `workflow-state.json` to become
instance-keyed (e.g. `vars` plus a new per-instance array) while keeping
the lifecycle state machine as-is. Option (a) matches source doc 5.3
exactly but is a larger structural change; option (b) is a smaller,
more incremental change but diverges from the source docs' explicit
per-folder model. This increment does not recommend one over the other —
it is a genuine design decision for the next role, not a fact this audit
can resolve.

## Next step recommendation

Skip the schema-writer role for the `externalRefs` field itself (it would
be trivially additive once a target file exists), consistent with INC-05's
precedent of not inventing unneeded roles. However, **do not go straight
to cli-implementer without first resolving the open question above**,
because "add `externalRefs`, align flags" (the parent roadmap's INC-06
one-line brief) presupposes a per-instance metadata file that does not
exist yet. Concretely:

1. If the product owner/user picks option (a) or (b) above via a short
   decision (same style as INC-01's D1/D2/D3), launch **schema-writer**
   next specifically to design the new per-instance metadata shape
   (fields: id, title, status, `externalRefs`, links, timestamps) — this
   is a real, non-trivial shape decision, unlike the `externalRefs` field
   alone.
2. Then **cli-implementer**: extend `axiom-increment.ts`/`axiom-bug.ts`/
   `axiom-plan.ts` to read/write the new per-instance file (or extended
   `workflow-state.json`), add the missing `link-plan`/`link-increment`/
   `link-bug` commands, add `plan create` (currently absent), and reconcile
   ID generation to be system-stable rather than free-text per addendum 11
   — all four gaps found in this audit are inputs to this step.
3. Then **validator-reviewer**: run `Axiom/package.json`'s real test suite
   (`npm test`) plus `npm run doctor` against the new implementation, and
   confirm the four existing test files
   (`axiom-{increment,bug,plan,role}.test.ts`) still pass or are updated
   consistently with the new per-instance model.

Exact inputs the next role needs from this document: the structural gap
finding (no per-instance file exists), the four field-by-field/command
gap tables above, the `externalRefs`/ID-generation/linking-commands
findings, and the explicit open question on file-shape option (a) vs (b).
