# Increment: Reconcile increment/bug/plan commands (schema-writer design)

Status: pending
Date: 2026-07-02

## Goal

Execute the **schema-writer** step of INC-06 of the parent roadmap
(`Axiom.Spec/specs/increments/INC-20260702-axiom-redesign-roadmap/README.md`),
following directly from the migration-engineer audit
(`Axiom.Spec/specs/increments/INC-20260702-increment-bug-plan-commands-reconcile/README.md`).

Design the exact `metadata.yml` schema (increment/bug/plan), the stable
internal-ID generation scheme, the folder layout, and the interaction
between the new per-artifact metadata files and `@axiom/workflow`'s
existing lifecycle state machine — precise enough that **cli-implementer**
can implement it next without re-deciding anything.

This increment does not write any code. It is a design/spec artifact only.

## Context

**User-resolved decision (given, not re-litigated in this document):**
adopt the folder-per-artifact model from source doc section 5.3
(`increments/INC-xxx/{README.md, metadata.yml}`, same pattern for bugs and
plans), replacing `workflow-state.json` as the source of truth for
artifact *metadata/identity*. This is a genuine new capability: multiple
concurrent increments/bugs/plans/roles per project, which today's single
shared `workflow-state.json` (one record per workflow ID, not per
instance) cannot support.

Source documents consulted directly for this design (both re-read in
full for the relevant sections, not taken from the parent roadmap's
paraphrase):

- `axiom_decisiones_sesion_prompt_implementacion.md` section 5.3 (source
  of truth: folder + `metadata.yml` per increment/bug/plan instance) and
  section 6.1 (command list, including `link-plan`/`link-increment`/
  `link-bug`).
- `axiom_decisiones_sesion_addendum_revision.md` section 9 (`targetRepos`
  / `allowedWriteScope` on plan metadata, with a full YAML example) and
  section 11 (`externalRefs` shape + stable internal ID example
  `INC-20260702-183012-a7f3c9`).

**Codebase facts this design is built on** (from the migration-engineer
audit, re-verified by reading `@axiom/workflow/src/*` in full for this
increment):

- `@axiom/workflow` is a generic, config-driven state machine
  (`WorkflowConfig` + `applyTransition`) with no knowledge of "increment"
  vs. "bug" vs. "plan" — those are just `WorkflowId` values materialized
  from `axiom.spec/config/workflows.yaml`.
- The state machine (`state-machine.ts`) is pure (no I/O) and only
  answers "given `(config, state, command)`, what's the next state (or
  error)?" It has zero knowledge of persistence — that is entirely
  `state-store.ts`'s job.
- `state-store.ts` persists exactly one `WorkflowStateRecord` per
  `workflowId` (not per instance) to
  `<projectRoot>/.sdd/local/workflow-state.json`, and that file is
  explicitly documented as project-local, gitignored-by-convention state
  (`.sdd/local/`), not spec-repo content.
- `axiom.yaml` (both schema v1's `scopes.spec.path` and v2's
  `paths.spec.path`) already designates a `spec` path scope for the
  current repo, defaulting to `axiom.spec` — this is the exact same root
  `axiom-increment.ts` already uses for
  `axiom.spec/config/workflows.yaml` (see
  `defaultWorkflowsYamlPath()` in `axiom-increment.ts`). This is the
  correct, already-established anchor for the new artifact folders (no
  new parallel root is being introduced).
- `hooks.ts`'s hook engine is orthogonal to both the state machine and to
  metadata; it fires lifecycle hook points (`pre-start`/`post-start`/
  `pre-complete`/`post-complete`) that are currently no-op. Nothing in
  this design changes the hook engine.

## Scope

- Design `metadata.yml` schema shapes for increments, bugs, and plans
  (shared base + type-specific fields).
- Design the stable ID-generation function (signature, format,
  collision-avoidance).
- Decide how `metadata.yml`'s `status` field interacts with
  `@axiom/workflow`'s existing `WorkflowState`/state machine (keep both,
  fold one into the other, or something else) — an explicit,
  minimal-disruption call.
- Design the exact folder layout and its location relative to
  `axiom.spec/` (the same root `workflows.yaml` already uses).
- Specify `externalRefs` shape (already fixed by addendum 11 — this
  document places it precisely inside the new schema).
- Specify the `link-plan`/`link-increment`/`link-bug` cross-reference
  fields (not the CLI commands themselves — that is cli-implementer's
  job — but the metadata fields those commands will read/write).

## Non-goals

- No TypeScript implementation (types, Zod schemas, parsers, CLI flags).
- No implementation of `link-plan`/`link-increment`/`link-bug` commands.
- No change to `@axiom/workflow`'s public API surface (`applyTransition`,
  `state-machine.ts`, `hooks.ts` are not modified by this design; see
  "State-machine interaction decision" below for why).
- No decision on multi-repo/registry topics (`allowedWriteScope`
  validation *execution*, `axiom validate changes`) beyond stating the
  metadata field shape needed to support them later (per addendum 9's own
  scope, which only asks for the field to exist on plan metadata, not for
  the validator command to be built now).
- No migration script for existing `.sdd/local/workflow-state.json` data
  (today's single in-flight increment/bug/plan/role, if any, is not
  auto-migrated into the new folder format by this design — flagged as an
  open question below for cli-implementer/product-owner).

## Acceptance criteria

- [x] Source doc 5.3 and addendum 11 re-read directly for this design
      (not taken from the migration-engineer's paraphrase alone).
- [x] `@axiom/workflow/src/` read in full to decide the state-machine
      interaction question.
- [x] `metadata.yml` schema defined for increment, bug, and plan, with a
      shared base shape and explicit type-specific extensions.
- [x] ID-generation function designed with exact signature, format, and
      collision-avoidance approach.
- [x] State-machine-interaction decision made explicitly, with rationale
      referencing what the state machine actually offers.
- [x] Folder layout designed precisely, anchored to the same root
      `axiom-increment.ts` already resolves (`axiom.spec/`), not a new
      parallel structure.
- [x] `externalRefs` and cross-artifact link fields specified with exact
      field names and types.
- [x] Design is unambiguous enough for cli-implementer to implement
      without further decisions (self-reviewed against this criterion).

## Open questions

1. **Existing in-flight `workflow-state.json` data migration.** If a
   project currently has an in-flight increment/bug/plan/role recorded in
   `.sdd/local/workflow-state.json`, this design does not specify an
   automatic migration into a new `metadata.yml` folder. Recommended
   default (stated in Implementation notes below, not requiring a
   decision to proceed): treat it as a one-time manual/best-effort
   migration performed by cli-implementer's first release (create one
   folder from the existing single record, if present, on first use of
   any new command), rather than a blocking prerequisite. This is
   additive-safe because `workflow-state.json` is not deleted by this
   design.
2. **`role` artifacts.** Source doc 5.3/6.1 do not mention `role` as a
   structural artifact type at all (confirmed by the migration-engineer
   audit: "`role` is not one of the source docs' structural artifact
   types; it is this codebase's own lifecycle-execution concept"). This
   design does **not** give `role` a `metadata.yml`/folder — it stays on
   `@axiom/workflow`'s existing per-workflow `workflow-state.json` engine
   unchanged, because a `role` in this codebase represents *executing* an
   already-approved plan (a runtime/session concept), not a
   spec-repo-tracked artifact with its own identity, history, or
   `externalRefs`. If a future increment needs multiple concurrent
   `role` executions, that is out of scope here and should be raised as
   its own increment.

## Assumptions

- "Minimal disruption" (per this increment's own instructions) means:
  reuse `@axiom/workflow`'s state machine and hook engine exactly as they
  are today (no API changes), and add the new per-instance metadata layer
  *alongside* them rather than replacing the state machine's role.
- The cli-implementer will extend the existing four CLI files
  (`axiom-increment.ts`, `axiom-bug.ts`, `axiom-plan.ts`) plus add new
  read/write logic for `metadata.yml`, most naturally as new exports from
  `@axiom/workflow` (a sibling module, e.g. `artifact-store.ts`) since
  that package already owns `WorkflowState`/`WorkflowId` and is the
  natural home for "one more persistence concern for the same four
  workflows." This document recommends but does not mandate the exact
  package; cli-implementer may place it in `@axiom/workflow` or a new
  package if `@axiom/workflow`'s existing dependency-free design
  (no `js-yaml` in `state-store.ts` deliberately) argues against adding a
  YAML writer there. See "Package placement" note below.
- `role` is excluded from the new folder-per-artifact model (see Open
  question 2); nothing in this design changes `axiom-role.ts`.

## Implementation notes

### 1. Folder layout

Anchored to the **same** `spec` path scope `axiom-increment.ts` already
resolves today (`axiom.yaml` v1 `scopes.spec.path` / v2 `paths.spec.path`,
defaulting to `axiom.spec`) — not a new parallel root:

```text
<projectRoot>/<specPath>/
├─ config/
│  └─ workflows.yaml              # unchanged, still owned by @axiom/workflow
├─ increments/
│  └─ INC-20260702-183012-a7f3c9/
│     ├─ README.md                # human-authored prose (spec body)
│     └─ metadata.yml             # machine-owned metadata (this design)
├─ bugs/
│  └─ BUG-20260702-184501-f1c0de/
│     ├─ README.md
│     └─ metadata.yml
└─ plans/
   └─ PLAN-20260702-190112-9b7a2e/
      ├─ README.md
      └─ metadata.yml
```

Where `<specPath>` resolves exactly as `axiom-increment.ts`'s
`defaultWorkflowsYamlPath()` already resolves its parent
(`<projectRoot>/axiom.spec` by convention today; the `axiom.yaml`
`scopes.spec.path`/`paths.spec.path` value should be honored the same
way `workflows.yaml`'s own path is, for consistency — cli-implementer
should read the scope path the same way, not hardcode `axiom.spec`
twice in two different places).

This is directly adjacent to, not a replacement of,
`axiom.spec/config/workflows.yaml`: the state machine config file stays
exactly where it is. Only new sibling folders (`increments/`, `bugs/`,
`plans/`) are added under the same `spec` root.

`.sdd/local/workflow-state.json` is **not deleted or renamed** by this
design (see State-machine-interaction decision below for why it still
exists and what it now means).

### 2. Stable ID generation

New, small, focused utility. Naming: `generateArtifactId` (not
`slugifyProjectId` — different domain, different shape: `slugifyProjectId`
turns a human name into a slug; this function generates a *fresh, unique,
system-owned identifier* that has no human input to slugify). Consistent
in *spirit* only: pure, deterministic given its inputs, no I/O, testable
without mocking except for time/randomness injection.

```ts
// New file: @axiom/workflow/src/artifact-id.ts (see Package placement)

/** Closed set of artifact kinds that get a stable, generated ID. */
export type ArtifactKind = 'increment' | 'bug' | 'plan';

/** Maps each kind to its ID prefix, per source doc 5.3 examples. */
const ARTIFACT_ID_PREFIX: Record<ArtifactKind, string> = {
  increment: 'INC',
  bug: 'BUG',
  plan: 'PLAN',
};

export interface GenerateArtifactIdOptions {
  /** Injectable clock for deterministic tests. Default: `() => new Date()`. */
  readonly now?: () => Date;
  /**
   * Injectable random-suffix source. Default: 6 lowercase
   * hex/base36 chars derived from `Math.random()`. Tests should
   * inject a fixed value for determinism.
   */
  readonly randomSuffix?: () => string;
}

/**
 * Generates a stable, system-owned artifact ID:
 *   `<PREFIX>-<YYYYMMDD>-<HHMMSS>-<6-char-suffix>`
 * e.g. `INC-20260702-183012-a7f3c9` (addendum 11's own example format).
 *
 * Collision-avoidance: timestamp resolution is per-second, which is not
 * itself collision-proof for rapid automated creation (e.g. a script
 * creating two increments in the same second) — the 6-char random
 * suffix is the actual collision guard, not the timestamp. Caller
 * (cli-implementer) MUST verify the target folder does not already
 * exist before writing, and retry generation (new suffix) on the rare
 * collision, rather than trusting timestamp+suffix uniqueness
 * unconditionally. This mirrors `slugifyProjectId`'s spirit of "pure,
 * deterministic, no I/O" while adding the retry responsibility to the
 * caller (which does I/O) instead of baking a filesystem check into
 * this pure function.
 */
export function generateArtifactId(
  kind: ArtifactKind,
  options: GenerateArtifactIdOptions = {},
): string {
  const now = (options.now ?? (() => new Date()))();
  const randomSuffix =
    options.randomSuffix ??
    (() => Math.random().toString(36).slice(2, 8).padEnd(6, '0'));

  const yyyy = now.getUTCFullYear().toString().padStart(4, '0');
  const mm = (now.getUTCMonth() + 1).toString().padStart(2, '0');
  const dd = now.getUTCDate().toString().padStart(2, '0');
  const hh = now.getUTCHours().toString().padStart(2, '0');
  const min = now.getUTCMinutes().toString().padStart(2, '0');
  const ss = now.getUTCSeconds().toString().padStart(2, '0');

  return `${ARTIFACT_ID_PREFIX[kind]}-${yyyy}${mm}${dd}-${hh}${min}${ss}-${randomSuffix()}`;
}
```

Rationale for this exact shape over alternatives:

- **Timestamp+hash over pure slug** (rejected): a slug of the title
  (`slugifyProjectId`-style) is not stable if the title changes, and two
  increments can share a title. Addendum 11 explicitly asks for an ID
  that "debe ser estable aunque cambie la integración externa" — title
  independence is required, so slugifying the title is the wrong tool.
- **Timestamp+hash over sequential counter** (rejected): a sequential
  counter (`INC-0001`, `INC-0002`, per today's free-text `--id` style)
  requires a single source of truth for "the next number," which
  reintroduces exactly the single-shared-state bottleneck this whole
  redesign is meant to remove (today's `vars.id` collision risk). A
  timestamp+random-suffix ID needs no shared counter and no lock.
  Trade-off, stated explicitly: system-generated IDs are ugly/unmemorable
  compared to `INC-0019`; this is accepted because addendum 11 prioritizes
  stability and system ownership over human readability, and `title` (a
  separate field) carries the human-friendly label instead.
- Exact format matches addendum 11's own worked example
  (`INC-20260702-183012-a7f3c9`) — no invented format.
- Second-level timestamp (not millisecond) chosen to match the example's
  own precision (`183012` is `HHMMSS`, 6 digits) — matching the source
  doc precisely rather than adding unrequested precision.

### 3. `metadata.yml` schema

**Shared base shape** (used by increment, bug, and plan alike):

```yaml
# Common to increment/bug/plan metadata.yml
id: INC-20260702-183012-a7f3c9     # generateArtifactId() output; immutable after creation
kind: increment                     # 'increment' | 'bug' | 'plan' (redundant with folder + id prefix, kept for self-description when the file is read in isolation)
title: Reconcile increment/bug/plan commands
status: draft                       # see "State-machine interaction decision" for the exact value set
createdAt: "2026-07-02T18:30:12.000Z"   # ISO 8601, set once at creation, immutable
updatedAt: "2026-07-02T19:05:00.000Z"   # ISO 8601, bumped on every metadata.yml write
externalRefs: []                    # ReadonlyArray<ExternalRef>, see shape below; empty by default
links: {}                           # see per-kind links below; empty object by default
```

`ExternalRef` shape (addendum 11, verbatim):

```yaml
externalRefs:
  - provider: azure-devops           # string, plugin-defined vocabulary (e.g. 'azure-devops', 'jira')
    type: user-story                 # string, provider-defined vocabulary (e.g. 'user-story', 'bug', 'task')
    id: "5083"                       # string (always string, even if the provider's native ID is numeric — avoids YAML int-vs-string ambiguity)
    url: "https://dev.azure.com/..." # string, optional in principle but always populated when a plugin creates the ref
```

**Increment-specific fields** (`links`):

```yaml
links:
  planId: PLAN-20260702-190112-9b7a2e   # string | null — set by `axiom increment link-plan`
```

**Bug-specific fields** (`links`) — same shape as increment, by design
(the migration-engineer audit found increment/bug near-identical in
today's command surface, and source doc 6.1 gives bugs the same
`link-plan` command increments get):

```yaml
links:
  planId: PLAN-20260702-190112-9b7a2e   # string | null — set by `axiom bug link-plan`
```

**Plan-specific fields** (`links` plus the addendum-9 fields, verbatim
shape from the addendum's own example, adapted to this design's `links`
convention for symmetry with increment/bug):

```yaml
links:
  incrementId: INC-20260702-183012-a7f3c9   # string | null — set by `axiom plan link-increment`
  bugId: BUG-20260702-184501-f1c0de         # string | null — set by `axiom plan link-bug`
                                             # (both may be null; both may also be set if a plan addresses an increment AND a linked bug — no exclusivity constraint is imposed by this design)
targetRepos:
  - kvp25-back                              # ReadonlyArray<string>, repo IDs matching @axiom/topology's RepoRef.id vocabulary (same vocabulary as axiom.yaml v2's repoId field)
taskType: backend-implementation            # string, free-form classification (addendum 9 does not close this to an enum; not closed here either — avoids speculative enum design)
allowedWriteScope:
  - repo: kvp25-back                        # string, must be one of targetRepos (cli-implementer should validate this invariant; not enforced by this schema alone)
    paths:
      - src/**
      - tests/**
  - repo: kvp25-spec
    paths:
      - plans/PLAN-20260702-190112-9b7a2e/**
```

Note: `targetRepos`/`taskType`/`allowedWriteScope` are **plan-only**,
verbatim from addendum section 9's own example — increment/bug metadata
does not carry them (an increment/bug does not itself declare a
write scope; the plan that implements it does).

**Full worked example — increment `metadata.yml`:**

```yaml
id: INC-20260702-183012-a7f3c9
kind: increment
title: Reconcile increment/bug/plan commands
status: draft
createdAt: "2026-07-02T18:30:12.000Z"
updatedAt: "2026-07-02T18:30:12.000Z"
externalRefs: []
links:
  planId: null
```

**Full worked example — plan `metadata.yml`:**

```yaml
id: PLAN-20260702-190112-9b7a2e
kind: plan
title: Implement increment/bug/plan metadata redesign
status: approved
createdAt: "2026-07-02T19:01:12.000Z"
updatedAt: "2026-07-02T19:10:00.000Z"
externalRefs: []
links:
  incrementId: INC-20260702-183012-a7f3c9
  bugId: null
targetRepos:
  - axiom-cli
taskType: cli-implementation
allowedWriteScope:
  - repo: axiom-cli
    paths:
      - apps/cli/src/commands/**
      - packages/workflow/src/**
  - repo: axiom-spec
    paths:
      - increments/INC-20260702-183012-a7f3c9/**
      - plans/PLAN-20260702-190112-9b7a2e/**
```

### 4. State-machine interaction decision

**Decision: keep `@axiom/workflow`'s state machine and hook engine
completely unchanged. `metadata.yml`'s `status` field is a separate,
coarser, per-instance concept that does NOT replace or wrap
`WorkflowState`. The state machine continues to operate exactly as it
does today (per-workflow-ID, via `workflow-state.json`), and
`metadata.yml`'s `status` is written by the CLI command layer
(cli-implementer's code in `axiom-increment.ts`/etc.), not by
`@axiom/workflow` itself.**

Rationale, based on directly reading `@axiom/workflow/src/*`:

- `state-machine.ts`'s `applyTransition` is a pure function over
  `(config, state, command)`. It has **no concept of an artifact
  instance at all** — it doesn't take an ID, doesn't take a folder path,
  doesn't know anything exists outside the single `state`/`command`
  pair it's given. Making it "operate per-artifact-instance" would
  require threading an instance identifier through the entire engine
  (`WorkflowContext.vars`, `state-store.ts`'s file shape, the CLI
  call sites) — a change to the *core* engine's contract, not an
  additive change. That is a materially larger, riskier change than this
  increment's brief ("minimal disruption") calls for.
- `WorkflowState`'s 9 values (`draft`, `specifying`, `planned`,
  `plan-approved`, `in-progress`, `verifying`, `archived`, `failed`,
  `cancelled`) are **already a superset** of what a simple artifact
  `status` field needs. Rather than inventing a second, competing state
  vocabulary inside `metadata.yml`, this design reuses the same
  `WorkflowState` string union for `metadata.yml`'s `status` field
  (`status: WorkflowState`, imported as a type from `@axiom/workflow`).
  This is additive: `metadata.yml` conceptually *mirrors* the current
  workflow's state at the time the file was last written, without the
  state machine needing to know instances exist.
- Concretely, the write sequence cli-implementer should follow per
  command (e.g. `axiom increment create`) is:
  1. Generate `id` via `generateArtifactId('increment')` (only on
     `create`).
  2. Run `applyTransition` exactly as today (unchanged call).
  3. On success, write/update **two** files instead of one:
     `workflow-state.json` (unchanged shape/call, `saveWorkflowState`) —
     kept because it still gives `@axiom/role`'s `checkPlanIsApproved`-
     style single-active-workflow gates a place to read from without
     redesign — **and** the artifact's own `metadata.yml`
     (`status: <newState>`, `updatedAt: <now>`), which cli-implementer
     writes via the new artifact-store module (see Package placement).
  4. `workflow-state.json`'s per-workflow-ID record becomes, in effect,
     "the currently active/most-recent instance of this workflow type,"
     used only where a single-active-instance gate check is still useful
     (e.g. `axiom-role.ts`'s existing plan-approved gate) — it is **not**
     extended to become instance-keyed. Multi-instance concurrency is
     achieved entirely through the new `metadata.yml` folders, which have
     no shared-file contention at all (each instance is its own file).
- This means `axiom-role.ts`'s existing `checkPlanIsApproved` gate
  (reading the `plan` workflow's single shared `workflow-state.json`
  record) **keeps working unmodified** — it was already a "most recent
  plan" check, and nothing here changes that semantic; it simply becomes
  explicit that it checks "the most recently active plan," which
  cli-implementer should document as a known, accepted limitation (not a
  bug) rather than silently reinterpreting it as "any approved plan."
  Making the gate multi-plan-aware (e.g. "is *some* plan referenced by
  this increment approved") is out of scope for this design and would be
  a genuine behavior change requiring its own increment.

Alternative considered and rejected: extending `WorkflowStateRecord`
itself to carry an array of instances keyed by ID (audit's option (b)).
Rejected because it would require rewriting `state-store.ts`'s merge
logic, its schema-validation code, and every call site's assumption of
"one record per `loadWorkflowState(projectRoot, workflowId)` call" — a
change to `@axiom/workflow`'s tested, documented core contract. The
chosen design (audit's option (a), folders + `metadata.yml`, with the
state machine untouched) achieves the same multi-instance goal with zero
changes to `@axiom/workflow`'s existing, working code — strictly less
disruptive, consistent with `Axiom.SDD/AGENTS.md`'s "do not create
speculative architecture" and "keep changes small and focused" rules.

### 5. Package placement (recommendation, not a hard requirement)

`state-store.ts` deliberately avoids `js-yaml` ("no depende de
`js-yaml`: usa el `JSON.parse` nativo"), by explicit design comment,
because `workflow-state.json` is JSON. `metadata.yml` is, by source doc
5.3's own naming, YAML — and `axiom-increment.ts` already depends on
`js-yaml` directly (for `workflows.yaml`) at the CLI layer, not inside
`@axiom/workflow`. Two options for cli-implementer, stated here so the
choice is made once and not re-litigated per file:

- **Option A (recommended):** add a new sibling module inside
  `@axiom/workflow` (e.g. `src/artifact-store.ts` +
  `src/artifact-id.ts`), taking a `js-yaml` dependency at the package
  level. This keeps all four workflows' persistence concerns (state
  machine config, runtime state, and now instance metadata) in one
  package, consistent with `@axiom/workflow`'s existing role as "the
  shared generic engine these four commands wrap" (per the audit's own
  framing). `state-store.ts`'s "no `js-yaml`" comment is scoped to that
  one file's design rationale (JSON-only state file), not a
  package-wide constraint — the barrel (`index.ts`) already has room to
  export more.
- **Option B:** a new standalone package (e.g. `@axiom/artifact-store`).
  Only worth it if cli-implementer finds a reason `@axiom/workflow`
  shouldn't own YAML I/O (e.g. bundle-size concerns for consumers that
  only need the pure state machine). Not recommended by this design
  absent such a finding — would add a new package for a lightweight
  concern, which is exactly the kind of unnecessary structural growth
  `Axiom.SDD/AGENTS.md`'s bootstrap limits caution against
  ("deep autogenerated folder hierarchies," "complex metadata systems").

This design does not mandate which option cli-implementer picks, but
documents both so the trade-off is explicit rather than rediscovered.

## Validation

`No validation command was found. Performed best-effort validation by
inspecting changed files and checking consistency against the requested
behavior.`

This increment is design-only (no code changed). Best-effort validation
performed:

- Every schema field traced back to an exact source: `id`/`title`/
  `status`/`externalRefs` to addendum 11's worked example, `targetRepos`/
  `taskType`/`allowedWriteScope` to addendum section 9's worked example
  verbatim, folder layout to source doc section 5.3's tree verbatim.
- `@axiom/workflow/src/types.ts`, `state-machine.ts`, `state-store.ts`,
  `hooks.ts`, `index.ts`, `workflows-loader.ts`, and `branch-naming.ts`
  read in full to ground the state-machine-interaction decision in the
  engine's actual, current behavior (not assumed behavior).
- `axiom-increment.ts` read in full to confirm the exact current
  `axiom.spec/config/workflows.yaml` path resolution this design's folder
  layout is anchored to.
- `@axiom/config-validation/src/schemas.ts` and
  `@axiom/project-resolution/tests/resolver.test.ts` read to confirm both
  `axiom.yaml` v1 (`scopes.spec.path`) and v2 (`paths.spec.path`) resolve
  to the same `spec` scope, defaulting to `axiom.spec` — the folder
  layout's anchor point is not invented, it is the already-established
  scope path.
- `slugifyProjectId` (`@axiom/user-workspace/src/registry-id.ts`) read in
  full to justify why `generateArtifactId` is a distinct, new function
  rather than a reuse of that function (different domain: slug-of-a-name
  vs. system-generated-fresh-ID).

## Result

Delivered a complete, unambiguous design for INC-06's per-artifact
metadata model:

1. **Folder layout**: `<specPath>/{increments,bugs,plans}/<ID>/
   {README.md,metadata.yml}`, anchored to the same `spec` path scope
   (`axiom.yaml` `scopes.spec.path`/`paths.spec.path`, default
   `axiom.spec`) that `axiom-increment.ts` already resolves for
   `workflows.yaml` — no new parallel root.
2. **ID generation**: new `generateArtifactId(kind, options?)` in
   `@axiom/workflow` (recommended package), format
   `<PREFIX>-<YYYYMMDD>-<HHMMSS>-<6-char-suffix>` matching addendum 11's
   own worked example exactly, with the random suffix (not the
   timestamp) as the actual collision guard and an explicit
   caller-must-check-and-retry contract for the rare collision case.
3. **`metadata.yml` schema**: a shared base (`id`, `kind`, `title`,
   `status`, `createdAt`, `updatedAt`, `externalRefs`, `links`) plus
   increment/bug identical `links: { planId }` and plan-specific
   `links: { incrementId, bugId }` + `targetRepos` + `taskType` +
   `allowedWriteScope` (verbatim from addendum section 9's example).
4. **State-machine interaction**: `@axiom/workflow`'s state machine and
   `workflow-state.json` are kept **completely unchanged** — no instance
   scoping is added to the core engine. `metadata.yml`'s `status` field
   reuses the existing `WorkflowState` type as its value set and is
   written by the CLI command layer alongside (not instead of) the
   existing `saveWorkflowState` call. Multi-instance concurrency is
   achieved entirely by the new per-instance `metadata.yml` files, which
   have no shared-file contention.
5. **Package placement**: recommended (not mandated) to add the new
   YAML-based artifact-store logic as a sibling module inside
   `@axiom/workflow` rather than a new package, with the trade-off
   documented for cli-implementer to confirm or override.

Two open questions are flagged for cli-implementer/product-owner
(existing `workflow-state.json` data migration; `role`'s explicit
exclusion from the new model) — neither blocks implementation from
starting, both are stated with a recommended default.

## General spec integration

No integration into a `general-spec.md` was performed — as the
migration-engineer audit already noted, that file does not exist in this
repo; the closest equivalents are `Axiom.Spec/specs/00_Resumen_
Ejecutivo.md` through `08_Glosario.md`, which were not modified. This
design's schema is specific to INC-06's own scope and is not yet proven
by an implementation — consolidating it into a general/stable spec now,
before cli-implementer has built and validated it, would risk
documenting a shape that changes during implementation. This is
consistent with the migration-engineer audit's own reasoning for
deferring general-spec updates until implementation lands.

## Next step recommendation

**cli-implementer**, with no further schema-writer or registry-engineer
step needed (per the parent instruction: there is no
`@axiom/user-workspace`-equivalent package for this domain, so
cli-implementer owns both the new `@axiom/workflow` persistence logic
and the CLI command changes in `axiom-increment.ts`/`axiom-bug.ts`/
`axiom-plan.ts`).

Exact inputs cli-implementer needs from this document:

1. The folder layout (section "1. Folder layout" above) — implement
   folder/file creation exactly as specified, anchored to the existing
   `spec` path scope.
2. The `generateArtifactId` function signature and format (section "2.")
   — implement verbatim, including the caller-retries-on-collision
   contract.
3. The three `metadata.yml` worked examples (section "3.") — implement
   the shared base + per-kind extensions exactly as shown, including
   `links` field names (`planId` for increment/bug; `incrementId`/
   `bugId` for plan) and the plan-only `targetRepos`/`taskType`/
   `allowedWriteScope` fields.
4. The state-machine-interaction decision (section "4.") — do NOT modify
   `@axiom/workflow/src/state-machine.ts`, `state-store.ts`, or
   `hooks.ts`. Add new persistence logic alongside the existing
   `saveWorkflowState` call, not instead of it.
5. The package-placement recommendation (section "5.") — Option A
   (extend `@axiom/workflow`) unless cli-implementer finds a concrete
   reason to prefer Option B.
6. The two open questions (existing-data migration default; `role`
   exclusion) — proceed with the stated recommended defaults unless the
   product owner overrides them.
7. Also carry forward, unchanged, the migration-engineer audit's own
   findings that this design does not re-decide: the exact current
   command-surface gap table (`create`/`refine`/`specify`/... vs. source
   doc 6.1's assumed names), and the instruction to add `plan create`
   (currently entirely absent) alongside `link-plan`/`link-increment`/
   `link-bug`.

After cli-implementer, **validator-reviewer** runs `Axiom/package.json`'s
real test suite (`npm test`, `npm run doctor`) against the new
implementation, per the migration-engineer audit's own next-step chain.
