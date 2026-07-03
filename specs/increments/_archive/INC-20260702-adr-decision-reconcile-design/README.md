# Increment: ADR/decision handling — schema design (INC-11)

Status: closed
Date: 2026-07-02

Closed as part of INC-11 as a whole (schema-writer -> cli-implementer ->
validator-reviewer). See
`Axiom.Spec/specs/increments/INC-20260702-adr-decision-reconcile-validator/README.md`
for the chain's closure summary, independent verification of this
design's claims, and the `general-spec.md` integration performed on
closure.

## Goal

Design `adr` and `decision` as two new, additive artifact kinds inside the
existing folder-per-artifact `metadata.yml` mechanism landed by INC-06
(`Axiom/packages/workflow/src/artifact-store.ts`, `artifact-id.ts`), and
settle the ADR-index derived-vs-curated question (addendum §6) with a
recommendation. This increment is design-only: it produces the schema and
a brief for `cli-implementer`; it does not write product code.

## Context

Parent roadmap: `Axiom.Spec/specs/increments/INC-20260702-axiom-redesign-roadmap/README.md`,
INC-11 ("Reconcile ADR/decision handling"), marked **additive, no legacy
package**, depending on INC-07 (derived cache/index-rebuild mechanism,
closed) and INC-09/INC-10 (curated index mechanism, closed). Roadmap's own
assigned subagent sequence: schema-writer -> cli-implementer ->
validator-reviewer (no migration-engineer, per the roadmap's own additive
marking).

Source documents (re-read directly for this increment):

- `axiom_decisiones_sesion_prompt_implementacion.md` §2.2: the SPEC repo's
  suggested structure lists `adr/` and `decisions/` as two **separate**
  top-level folders, siblings of `increments/`, `bugs/`, `plans/`. §6.1
  lists the structural commands: `axiom adr create`, `axiom adr link`,
  `axiom decision create`, `axiom decision link`.
- `axiom_decisiones_sesion_addendum_revision.md` §6 ("ADR index: derivado
  o curado según uso"): the ADR index's nature depends on what it does,
  not on being an ADR index per se — verbatim rule: "Si el índice decide
  comportamiento del agente, se versiona. Si solo lista artefactos
  existentes, se regenera localmente." The addendum explicitly frames this
  as project-dependent ("puede ser derivado o curado **según el
  proyecto**"), not as a mandate to build the curated form now.

### Confirmation grep (pre-design check)

Per the task brief, before designing, ran a targeted (not full-audit) grep
across `Axiom/apps/cli/src/commands/` and `Axiom/packages/*` for
ADR/decision artifact-kind evidence, since several "additive" assumptions
elsewhere in this roadmap needed correction once actually checked (e.g.
INC-06/07/09/10's own audits).

Result: **confirmed still empty, as the roadmap expected.**

- No `adr.ts`/`decision.ts` file exists under `Axiom/apps/cli/src/commands/`
  (glob for `*adr*`/`*decision*` returned nothing).
- No `ArtifactKind` value, CLI command, or metadata shape for `adr`/
  `decision` exists anywhere under `Axiom/packages/*`. `ArtifactKind`
  (`Axiom/packages/workflow/src/artifact-id.ts`) is a closed union of
  exactly `'increment' | 'bug' | 'plan'`.
- The only `adr`-matching hits in `Axiom/packages/*` source are three
  **comment references** to architecture decision numbers
  (`ADR-0021`, `ADR-0026`) inside `Axiom/packages/doctor/src/checks.ts` —
  these are citations to this repo's own numbered architecture-decision
  log entries (a documentation convention, not an ADR artifact-kind or
  command), and are unrelated to this increment's scope.
- No finding to report beyond this — safe to proceed with a purely
  additive design, no migration-engineer step needed, matching the
  roadmap's own marking.

### Existing infrastructure read fresh for this design

`Axiom/packages/workflow/src/artifact-store.ts` and `artifact-id.ts`
(INC-06, extended by INC-07's `listArtifacts` consumer in
`Axiom/apps/cli/src/commands/index-cmd.ts`):

- `ArtifactKind = 'increment' | 'bug' | 'plan'` (closed union,
  `artifact-id.ts`), each with an ID prefix (`INC`/`BUG`/`PLAN`) and a
  folder name (`ARTIFACT_FOLDER` map in `artifact-store.ts`:
  `increment -> increments`, `bug -> bugs`, `plan -> plans`).
  `generateArtifactId`/`generateUniqueArtifactId` are pure ID generators,
  parameterized purely by `ArtifactKind` — adding new kinds only requires
  adding new entries to the two `Record<ArtifactKind, string>` maps
  (prefix, folder) plus the ID literal itself; no other change to
  `artifact-id.ts` is needed.
- `BaseArtifactMetadata` (shared shape): `id`, `kind`, `title`, `status`,
  `createdAt`, `updatedAt`, `externalRefs`. Per-kind metadata
  (`IncrementMetadata`, `BugMetadata`, `PlanMetadata`) extends this base
  with a `links` object and, for `plan`, extra fields
  (`targetRepos`/`taskType`/`allowedWriteScope`).
- `ArtifactMetadata` is a **hand-written discriminated union**
  (`IncrementMetadata | BugMetadata | PlanMetadata`), not a generic/open
  shape. `parseMetadata`/`toYamlObject` are **kind-switched functions**
  with an explicit branch per kind (`if (kind === 'plan') {...}`, final
  `else` shared by increment/bug). Adding `adr`/`decision` as new kinds
  means: extending the `ArtifactMetadata` union with two new interfaces,
  adding their branches to `parseMetadata`/`toYamlObject`, and adding
  their factories (mirroring `makeInitialIncrementOrBugMetadata`/
  `makeInitialPlanMetadata`). This is the exact, proven extension point —
  no parallel store, no new file-format, no new I/O primitive.
- `listArtifacts(projectRoot, kind, specRelPath?)` is kind-parameterized
  and folder-driven purely via `ARTIFACT_FOLDER[kind]`; it needs zero
  changes beyond the two kinds being present in that map — it already
  works for any `ArtifactKind` with a folder entry, including future ones.
- `axiom index rebuild`/`validate` (`Axiom/apps/cli/src/commands/index-cmd.ts`,
  INC-07) hardcode `SCANNED_KINDS: ReadonlyArray<ArtifactKind> = ['increment', 'bug', 'plan']`.
  This constant, not `listArtifacts` itself, is the actual place
  `cli-implementer` must touch to make `axiom index rebuild`/`validate`
  cover `adr`/`decision` too — flagged explicitly in the cli-implementer
  brief below.

### A real typing conflict found during this design (not previously flagged)

`BaseArtifactMetadata.status: WorkflowState`, where `WorkflowState`
(`Axiom/packages/workflow/src/types.ts`) is the closed union:

```ts
type WorkflowState =
  | 'draft' | 'specifying' | 'planned' | 'plan-approved'
  | 'in-progress' | 'verifying' | 'archived' | 'failed' | 'cancelled';
```

This is the **workflow state-machine vocabulary** — increment/bug/plan's
`status` field mirrors the state machine's current state for that
workflow instance (see `artifact-store.ts`'s own header comment: "Mirror
del `WorkflowState` del workflow asociado"). ADRs need a **different**
status vocabulary entirely: `proposed | accepted | superseded | rejected`
(standard ADR lifecycle, source docs do not contradict this — neither
document specifies an ADR status enum, so this list is this design's own
minimal, conventional choice). These are not compatible closed unions and
must not be silently coerced into one shared `status: WorkflowState`
field typed against the wrong vocabulary.

**Resolution chosen for this design**: `BaseArtifactMetadata.status` stays
`WorkflowState`-typed and continues to apply only to increment/bug/plan
(kinds that really are driven by the workflow state machine). ADR and
Decision are **not** state-machine-driven artifacts — nothing in either
source document ties them to `workflows.yaml`/`state-machine.ts` gates —
so they should not pretend to have a `WorkflowState`. Concretely:

- Introduce a narrower shared base, `BaseArtifactMetadataFields`, holding
  only the fields that are truly kind-agnostic and typing-safe across all
  five kinds: `id`, `kind`, `title`, `createdAt`, `updatedAt`,
  `externalRefs`. `status` is removed from this narrower base.
  `BaseArtifactMetadata` (the existing exported name) stays exactly as-is
  today (`BaseArtifactMetadataFields & { status: WorkflowState }`) so
  `IncrementMetadata`/`BugMetadata`/`PlanMetadata` and every existing
  consumer (`makeInitialIncrementOrBugMetadata`, `axiom-increment.ts`,
  `axiom-bug.ts`, `axiom-plan.ts`, doctor checks, etc.) compile unchanged
  — this is a strictly additive type-level refactor, not a breaking one.
- `AdrMetadata` and `DecisionMetadata` extend `BaseArtifactMetadataFields`
  directly (not `BaseArtifactMetadata`) and declare their own
  `status: AdrStatus` / `status: DecisionStatus` fields with their own
  vocabularies. This keeps one shared field name (`status`) across all
  five kinds for UI/CLI-output consistency, while keeping the two
  vocabularies type-safe and non-overlapping — no `as WorkflowState`
  casts, no silently-wrong-typed field anywhere.
- `ArtifactMetadata` (the public discriminated union) grows to
  `IncrementMetadata | BugMetadata | PlanMetadata | AdrMetadata | DecisionMetadata`.

This is the one piece of this design that required an actual decision
beyond "copy the plan pattern" — documented here so `cli-implementer`
does not have to re-derive it from scratch, and so a future reader does
not mistake it for an oversight.

## Scope

- New `ArtifactKind` values: `'adr'`, `'decision'`.
- New metadata interfaces: `AdrMetadata`, `DecisionMetadata`, plus their
  own status vocabularies (`AdrStatus`, `DecisionStatus`) and link shapes.
- New ID prefixes and folder names for both kinds
  (`ARTIFACT_ID_PREFIX`, `ARTIFACT_FOLDER`).
- New factory functions (`makeInitialAdrMetadata`,
  `makeInitialDecisionMetadata`) mirroring the existing factories.
- The ADR-index derived-vs-curated recommendation (addendum §6), with
  rationale, as a design decision for this project today — not a new
  schema, since the recommendation is "derived, reuse `listArtifacts`."
- A concrete brief for `cli-implementer`'s next increment (commands,
  files to touch, what NOT to build).

## Non-goals

- No code implementation (no changes under `Axiom/packages/*` or
  `Axiom/apps/*`) — that is `cli-implementer`'s increment.
- No curated/versioned ADR-index schema — recommended against for now
  (see "ADR index" section below); would be speculative infrastructure
  per `Axiom.SDD/AGENTS.md`'s explicit bootstrap limits.
- No `axiom decision link` semantic resolution logic beyond the plain
  `links` field shape (e.g. no graph traversal, no "resolve superseded
  chain" helper) — out of scope for a schema-only increment.
- No changes to `WorkflowState`, `workflows.yaml`, or the state-machine
  engine — ADR/Decision are explicitly designed as non-state-machine
  artifacts (see typing-conflict resolution above).
- No retrofitting of `axiom.yml`'s `paths` field (source doc §4.2's
  `paths: { adr: adr/, decisions: decisions/ }`) — the existing
  artifact-store hardcodes folder names via `ARTIFACT_FOLDER`, matching
  today's convention for increment/bug/plan; making per-kind folder paths
  configurable via `axiom.yml` is a separate, larger concern (would touch
  every existing kind too) and is not requested by this increment.

## Metadata schema

### Shared fields (kind-agnostic, all five kinds)

```ts
interface BaseArtifactMetadataFields {
  readonly id: string;
  readonly kind: ArtifactKind;
  readonly title: string;
  readonly createdAt: string;
  readonly updatedAt: string;
  readonly externalRefs: ReadonlyArray<ExternalRef>;
}
```

`BaseArtifactMetadata` (existing name, unchanged shape) becomes
`BaseArtifactMetadataFields & { readonly status: WorkflowState }`, used
only by `IncrementMetadata`/`BugMetadata`/`PlanMetadata`.

### `AdrMetadata`

```ts
type AdrStatus = 'proposed' | 'accepted' | 'superseded' | 'rejected';

interface AdrLinks {
  readonly incrementId?: string | null;
  readonly planId?: string | null;
  /** IDs of ADRs this one supersedes. Empty array if none. */
  readonly supersedes: ReadonlyArray<string>;
  /** ID of the ADR that supersedes this one, once superseded. */
  readonly supersededBy?: string | null;
}

interface AdrMetadata extends BaseArtifactMetadataFields {
  readonly kind: 'adr';
  readonly status: AdrStatus;
  readonly context: string;
  readonly decision: string;
  readonly consequences: string;
  readonly links: AdrLinks;
}
```

- `context`/`decision`/`consequences` are plain strings (short prose or a
  pointer/summary — the full narrative is expected to live in the
  artifact's `README.md`, consistent with how increment/bug/plan already
  split "metadata" from "content" across `metadata.yml`/`README.md`).
  These three fields exist in `metadata.yml` specifically so they are
  machine-readable without parsing `README.md` (e.g. for a future
  `get_adr_index`/`get_implementation_context` MCP tool per source doc
  §7.3/§8.3 — not built by this increment, but the fields are cheap and
  directly requested by the standard ADR shape referenced in the roadmap).
- `supersedes`/`supersededBy` model the ADR-specific relationship
  addendum §6 and source doc §2.2 both allude to (ADRs referencing other
  ADRs) without inventing a generic graph mechanism — just two plain ID
  fields, consistent with how `PlanLinks`/`IncrementOrBugLinks` are
  already just plain optional ID fields, not a relationship engine.
- ID prefix: `ADR`. Folder: `adr` (matches source doc §2.2's literal
  folder name, singular, not `adrs`).

### `DecisionMetadata`

Source doc §2.2 lists `adr/` and `decisions/` as two distinct folders but
does not explain what structurally distinguishes a "decision" from an
"ADR" beyond the name. This design makes a minimal, explicit call rather
than over-designing a distinction the source docs don't specify:

**Call made**: a `Decision` is a lighter-weight record for a project/
process decision that is not architectural in nature (e.g. "we adopt
`stacked-to-main` as our chain strategy," "we standardize on X naming
convention") — it does not need `context`/`consequences` narrative
fields, only a short rationale. An ADR is reserved for decisions with
lasting architectural/structural impact (the traditional ADR sense:
context, decision, consequences, supersession chain). This mirrors the
same increment/bug distinction already in this codebase (two sibling
kinds, near-identical shared shape, different narrow-purpose fields) —
no new mechanism, just a second minimal narrow-purpose shape.

```ts
type DecisionStatus = 'proposed' | 'accepted' | 'rejected';

interface DecisionLinks {
  readonly incrementId?: string | null;
  readonly planId?: string | null;
  readonly adrId?: string | null;
}

interface DecisionMetadata extends BaseArtifactMetadataFields {
  readonly kind: 'decision';
  readonly status: DecisionStatus;
  readonly rationale: string;
  readonly links: DecisionLinks;
}
```

- No `supersedes`/`supersededBy` on `Decision` — deliberately: nothing in
  either source doc suggests decisions get superseded the way ADRs do;
  adding that speculatively would be exactly the kind of unrequested
  complexity `Axiom.SDD/AGENTS.md` explicitly warns against. If a real
  need for decision supersession appears later, it is a small, additive
  follow-up (add the two optional fields), not a redesign.
- `DecisionStatus` intentionally omits `superseded` (unlike `AdrStatus`)
  for the same reason.
- ID prefix: `DEC` (not `DECISION`, consistent with the existing 3-4
  letter prefix convention: `INC`, `BUG`, `PLAN`, `ADR`). Folder:
  `decisions` (matches source doc §2.2's literal plural folder name —
  intentionally asymmetric with `adr`'s singular folder name because
  that asymmetry is literally what source doc §2.2 specifies; not a
  design inconsistency to "fix").

### Updated `ArtifactKind` and maps

```ts
// artifact-id.ts
type ArtifactKind = 'increment' | 'bug' | 'plan' | 'adr' | 'decision';

const ARTIFACT_ID_PREFIX: Record<ArtifactKind, string> = {
  increment: 'INC',
  bug: 'BUG',
  plan: 'PLAN',
  adr: 'ADR',
  decision: 'DEC',
};
```

```ts
// artifact-store.ts
const ARTIFACT_FOLDER: Record<ArtifactKind, string> = {
  increment: 'increments',
  bug: 'bugs',
  plan: 'plans',
  adr: 'adr',
  decision: 'decisions',
};
```

`parseMetadata`/`toYamlObject` need two new branches (mirroring the
existing `plan` branch), and `parseBaseFields` needs to be split so the
`status` parsing/validation is kind-vocabulary-aware (validate against
`AdrStatus`'s four values for `kind === 'adr'`, `DecisionStatus`'s three
for `kind === 'decision'`, `WorkflowState`'s nine for the other three) —
this is the concrete cost of the typing-conflict resolution above and
should be budgeted into `cli-implementer`'s estimate.

### Factories

```ts
function makeInitialAdrMetadata(
  id: string, title: string, now: string,
  options: { context?: string; decision?: string; consequences?: string } = {},
): AdrMetadata {
  return {
    id, kind: 'adr', title, status: 'proposed',
    createdAt: now, updatedAt: now, externalRefs: [],
    context: options.context ?? '', decision: options.decision ?? '',
    consequences: options.consequences ?? '',
    links: { incrementId: null, planId: null, supersedes: [], supersededBy: null },
  };
}

function makeInitialDecisionMetadata(
  id: string, title: string, now: string,
  options: { rationale?: string } = {},
): DecisionMetadata {
  return {
    id, kind: 'decision', title, status: 'proposed',
    createdAt: now, updatedAt: now, externalRefs: [],
    rationale: options.rationale ?? '',
    links: { incrementId: null, planId: null, adrId: null },
  };
}
```

Both default `status: 'proposed'` — matches the ADR lifecycle convention
and keeps behavior consistent with `makeInitialIncrementOrBugMetadata`,
which also takes its initial `status` as a caller-supplied workflow
state rather than hardcoding one (the difference here: ADR/Decision have
no state machine, so `'proposed'` is a reasonable fixed default rather
than a caller-supplied `WorkflowState`).

## ADR index: derived vs curated — decision

**Recommendation: derived (regenerable), not curated. Reuse INC-07's
`listArtifacts` + `index rebuild`/`validate` mechanism directly. Zero new
versioned index schema for this increment.**

Rationale, applying addendum §6's own test directly:

> "Si el índice decide comportamiento del agente, se versiona. Si solo
> lista artefactos existentes, se regenera localmente."

- Today, nothing in this codebase or either source document specifies an
  ADR index that **decides agent behavior** — no obligatoriedad
  (mandatory-reading) rules, no priority ordering, no semantic groupings
  analogous to `SkillsRoleIndex.mandatory`/`TechnicalContextIndex
  .mandatory.whenTags` (INC-09/INC-10, both closed, both real examples of
  curated indexes in this same codebase — used here as the actual bar for
  "curated," not a hypothetical one). An ADR list that just enumerates
  `id/title/status/supersedes` by scanning `adr/*/metadata.yml` is
  exactly the "solo lista artefactos existentes" case the addendum
  says should regenerate locally.
- `listArtifacts(projectRoot, 'adr')` already returns everything a derived
  ADR index needs (`id`, full `AdrMetadata` including `status`,
  `supersedes`/`supersededBy`) with zero new code once the `adr` kind
  exists in the maps above — this is a direct, immediate consequence of
  reusing the INC-06/07 mechanism rather than a new decision.
  `Axiom/apps/cli/src/commands/index-cmd.ts`'s `SCANNED_KINDS` constant
  is the only place that needs to grow (`['increment', 'bug', 'plan',
  'adr', 'decision']`) for `axiom index rebuild`/`validate` to cover both
  new kinds — no new index-building logic.
- Building a curated/versioned ADR index now (obligatoriedad, priority,
  groupings) would be speculative: no concrete consumer needs it yet (no
  MCP tool exists that would read `mandatory` ADR rules — `INC-13`/`14`,
  the MCP phase, are still open/未-scoped per the roadmap), and
  `Axiom.SDD/AGENTS.md`'s explicit bootstrap limits forbid "mandatory
  indexes" and "complex metadata systems" unless explicitly requested.
  Nothing in this task's brief requested a curated ADR index — only that
  the decision be made deliberately, which this section does.
- This mirrors INC-07's own migration-engineer finding almost exactly:
  "no dedicated index package existed; a cache-less direct-scan wrapper
  closes the gap" — the same posture applies here with even less
  justification for curation, since ADR obligatoriedad/priority has zero
  existing precedent anywhere in this codebase (unlike skills/technical-
  context, which had a closed, delivered curated-index increment each).

**If a curated ADR index becomes genuinely necessary later** (e.g. a
future increment wants "ADR-0021 is mandatory reading for anyone touching
`@axiom/isolation`"), it would follow the exact `SkillsRoleIndex`/
`TechnicalContextIndex` precedent: a new, additive,
`schemaVersion`-carrying file (e.g. `adr.index.yml`) with its own loader/
validator pair and its own `@axiom/doctor` check, layered on top of
(never replacing) the plain `metadata.yml`-per-ADR source of truth
established by this increment. That is future work, not part of this
increment's scope.

`Decision` gets no dedicated index consideration at all — nothing in
either source document raises this for decisions specifically, and the
same derived `listArtifacts('decision')` + `index rebuild`/`validate`
coverage applies identically once `decision` is a scanned kind.

## Acceptance criteria

- [x] Confirmation grep performed and reported (result: still empty, one
      unrelated comment-only false-positive pattern noted).
- [x] `AdrMetadata`/`DecisionMetadata` schemas fully specified, including
      their own status vocabularies, distinct from `WorkflowState`.
- [x] The `BaseArtifactMetadata`/`status: WorkflowState` typing conflict
      identified and resolved with a concrete, backward-compatible type
      design (`BaseArtifactMetadataFields` extraction).
- [x] ADR-vs-Decision structural distinction made explicit and documented
      (not left ambiguous for `cli-implementer` to guess).
- [x] ADR-index derived-vs-curated question answered with rationale tied
      directly to addendum §6's own stated test and to this codebase's
      two real curated-index precedents (INC-09/INC-10).
- [x] Extension points in `artifact-id.ts`/`artifact-store.ts`/
      `index-cmd.ts` named precisely (maps, branches, constants) rather
      than described abstractly.
- [x] No code implementation performed in this increment.
- [x] Brief produced for `cli-implementer` (see "Implementation notes").

## Open questions

- Should `axiom adr create`/`axiom decision create` require `context`/
  `decision`/`consequences` (ADR) or `rationale` (Decision) at creation
  time, or allow empty strings to be filled in later via `README.md` +
  a follow-up `update` command (mirroring how `PlanMetadata`'s
  `taskType`/`targetRepos` default to empty and get filled by dedicated
  `add-target-repo`/`set-scope`-style commands per source doc §6.1)? Left
  to `cli-implementer` to decide against the existing increment/bug/plan
  CLI's own established UX convention, since this is a CLI-ergonomics
  question, not a schema question.
- Should `axiom adr link`/`axiom decision link` support linking two ADRs
  via `supersedes` bidirectionally in one command (updating both
  `metadata.yml` files atomically), or require two separate calls (one
  per side)? Left to `cli-implementer`; the schema in this design
  supports either implementation choice without change.

## Assumptions

- ADR/Decision are not state-machine-driven artifacts (no
  `workflows.yaml` entry, no `state-store.ts` interaction) — confirmed
  as a deliberate design choice above, not an oversight.
- `DecisionStatus` deliberately excludes `superseded`; `AdrStatus`
  includes it. If this asymmetry turns out to be wrong in practice,
  it is a one-line, additive fix (add `'superseded'` to
  `DecisionStatus` and the two link fields to `DecisionLinks`), not a
  breaking schema change.
- The `adr` (singular) vs `decisions` (plural) folder-name asymmetry is
  intentional, taken verbatim from source doc §2.2, not a typo to
  normalize.

## Implementation notes

Design-only increment; no files changed under `Axiom/packages/*` or
`Axiom/apps/*`.

### Brief for `cli-implementer` (next increment in this chain)

Inputs: this spec (schema above), `Axiom/packages/workflow/src/artifact-id.ts`,
`Axiom/packages/workflow/src/artifact-store.ts`,
`Axiom/apps/cli/src/commands/index-cmd.ts`,
`Axiom/apps/cli/src/commands/axiom-increment.ts` (and `axiom-bug.ts`/
`axiom-plan.ts`) as the CLI-shape precedent to follow.

Concrete task list:

1. Extend `ArtifactKind` in `artifact-id.ts` to include `'adr' | 'decision'`;
   add their entries to `ARTIFACT_ID_PREFIX` (`ADR`, `DEC`).
2. In `artifact-store.ts`: add `adr`/`decision` to `ARTIFACT_FOLDER`
   (`adr`, `decisions`); extract `BaseArtifactMetadataFields` per this
   design and re-derive `BaseArtifactMetadata` from it without changing
   its effective shape; add `AdrMetadata`/`DecisionMetadata` interfaces,
   their `AdrLinks`/`DecisionLinks`, and `AdrStatus`/`DecisionStatus`;
   extend the `ArtifactMetadata` union; add parse/serialize branches to
   `parseMetadata`/`toYamlObject` (status-vocabulary-aware per kind); add
   `makeInitialAdrMetadata`/`makeInitialDecisionMetadata` factories.
3. Re-export the five new types plus the two new factories from
   `Axiom/packages/workflow/src/index.ts`'s existing artifact-store
   export block.
4. Add `adr.ts`/`decision.ts` under `Axiom/apps/cli/src/commands/`
   following `axiom-increment.ts`'s `create`/`update`/`status`/`link*`/
   `archive`-equivalent command shape (source doc §6.1's exact command
   names: `axiom adr create`, `axiom adr link`, `axiom decision create`,
   `axiom decision link` — extend with `update`/`status`/`list` only if
   the existing increment/bug precedent already has direct equivalents,
   to stay consistent rather than inventing a divergent command surface).
5. Extend `index-cmd.ts`'s `SCANNED_KINDS` to include `'adr'` and
   `'decision'` so `axiom index rebuild`/`validate` cover both new kinds
   with no other change to that file.
6. Add/extend vitest coverage mirroring
   `Axiom/packages/workflow/tests/artifact-store.test.ts`'s existing
   round-trip pattern (creation, load/save round-trip, folder layout,
   atomic write, externalRefs/links round-trip) for both new kinds,
   plus a status-vocabulary-rejection test (e.g. saving `status:
   'in-progress'` on an `adr` should fail shape validation, since that
   value belongs to `WorkflowState`, not `AdrStatus`).
7. Do NOT build a curated ADR-index schema, `.axiom/cache/adr.index.*`
   file, obligatoriedad/priority model, or a new MCP tool — all
   explicitly out of scope per this design's ADR-index section above.

Then hand off to `validator-reviewer` (existing `@axiom/doctor` extension
posture: confirm whether a new `IX-00x`-style check is warranted for
`adr`/`decision` metadata validity, following the same pattern
`TC-012`/`TC-013` established for skills/technical-context — but only if
`cli-implementer`'s own vitest coverage does not already fully cover
shape validation, to avoid duplicating the same check twice).

## Validation

`No validation command was found. Performed best-effort validation by
inspecting changed files and checking consistency against the requested
behavior.`

This increment made no code changes to validate (design-only, per scope).
Best-effort validation performed:

- Re-read `Axiom/packages/workflow/src/artifact-store.ts` and
  `artifact-id.ts` in full (not from memory/summary) to ground every
  extension-point claim in actual current code.
- Ran the confirmation grep across `Axiom/apps/cli/src/commands/` and
  `Axiom/packages/*` for ADR/decision artifact-kind evidence before
  starting the design, per the task's explicit instruction not to assume
  the roadmap's "additive, no legacy package" marking without checking.
- Cross-checked the new `AdrMetadata`/`DecisionMetadata` shapes against
  `Axiom/packages/workflow/tests/artifact-store.test.ts`'s existing
  round-trip test structure to confirm the proposed shapes are
  test-compatible with the existing harness pattern without requiring a
  new test infrastructure.
- Re-read both source decision documents' relevant sections in full
  (§2.2, §6.1 of the prompt doc; §6 of the addendum) rather than relying
  on the roadmap's paraphrase of them.

## Result

Produced a fully specified, additive `AdrMetadata`/`DecisionMetadata`
schema extending the existing INC-06 folder-per-artifact mechanism with
no parallel infrastructure; identified and resolved a real
`status`-field typing conflict between the workflow-state vocabulary and
ADR/Decision's own status vocabularies; recommended (with rationale) a
derived/regenerable ADR index reusing the INC-07 `listArtifacts`/`index
rebuild` mechanism, explicitly declining to build a curated/versioned ADR
index absent a concrete present need; and produced a concrete,
file-level brief for `cli-implementer`.

## General spec integration

Nothing integrated into `Axiom.Spec/general-spec.md` yet. Per this
increment's own closure status (`pending`, no code implemented), stable
knowledge should be integrated once `cli-implementer` and
`validator-reviewer` close and the schema is confirmed to have landed
without change during implementation — consistent with how the existing
"Folder-per-artifact convention," "Skills role-index," and
"Technical-context index" sections of `general-spec.md` were only added
once their respective increment chains fully closed (schema-writer ->
implementer -> validator-reviewer), not at the schema-writer step alone.
Recording here explicitly so this is not missed: once closed, add a short
"ADR/Decision artifact kinds" subsection to `general-spec.md` alongside
the existing "Folder-per-artifact convention" section, stating the two
new kinds, their status vocabularies, and the derived-ADR-index decision
(one paragraph, following that section's existing terse style — not a
copy of this increment's full rationale).

## Next step recommendation

Hand off to **cli-implementer** with this spec's "Implementation notes"
section as the concrete brief. Inputs: this file, plus direct reads of
`Axiom/packages/workflow/src/artifact-id.ts`,
`Axiom/packages/workflow/src/artifact-store.ts`,
`Axiom/packages/workflow/src/index.ts`,
`Axiom/apps/cli/src/commands/index-cmd.ts`, and
`Axiom/apps/cli/src/commands/axiom-increment.ts` (as the CLI-shape
precedent). After `cli-implementer` closes, run
`validator-reviewer` to confirm the implementation matches this schema
exactly (including the status-vocabulary-rejection behavior) and to
decide whether a new `@axiom/doctor` check is warranted, before this
increment chain as a whole is considered closed and integrated into
`general-spec.md`.
