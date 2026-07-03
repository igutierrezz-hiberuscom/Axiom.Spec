# Increment: Reconcile SDD/role skills (`@axiom/skills` catalog) — schema-writer design

Status: closed
Date: 2026-07-02

Closed as part of INC-09's overall closure. See
`INC-20260702-skills-role-index-reconcile-validator/README.md` ("INC-09
closure assessment") for the chain-wide closure rationale and deferred
items.

## Goal

Execute the **schema-writer** step of INC-09's subagent sequence
(migration-engineer -> **schema-writer** -> docs-skills-writer ->
validator-reviewer), following on from
`Axiom.Spec/specs/increments/INC-20260702-skills-role-index-reconcile/README.md`
(the migration-engineer audit). This increment resolves the audit's three
open questions (Q-skills-1, Q-skills-2, Q-skills-3) and produces the exact
TypeScript/YAML schema for the new per-role skills-index artifact (source
doc section 9.4's shape), including its file placement, naming, and
versioning. It does **not** implement any of this in `@axiom/skills` or
anywhere else in `Axiom/packages/*` — that is a later step (docs-
skills-writer produces the role-catalog markdown content next;
validator-reviewer wires a `doctor` check afterward; actual code/YAML
authoring in `@axiom/skills` is not assigned to either of those roles
either, per the audit's own brief — see "Next step recommendation").

## Context

Parent chain: `INC-20260702-axiom-redesign-roadmap` (parent roadmap) ->
`INC-20260702-skills-role-index-reconcile` (migration-engineer audit,
status `pending`) -> this increment (schema-writer design).

The migration-engineer audit is treated as authoritative input and is not
re-litigated here except where it explicitly delegated a decision to
schema-writer (its three open questions) or where its own brief needed a
concrete shape (its `Brief for schema-writer`, items 1-2). Source
documents were re-read directly for this design, not paraphrased from the
audit:

- `axiom_decisiones_sesion_prompt_implementacion.md` section 9 (Skills),
  read in full again: 9.1 (SDD skills live in the SDD repo — examples are
  all slash-command-style skill names, e.g. `create-increment`,
  `review-plan`), 9.2 (repo-specific technical skills live in each code
  repo, example `skills/backend-developer.md`, `skills/ef-core.md`, etc.,
  installable/enrichable via autoskills), 9.3 (one role-catalog markdown
  per role, human-readable "what/when/why", worked example is "Backend
  Developer Skill Catalog"), 9.4 (one flat per-role skills-index YAML,
  worked example below, confirmed byte-for-byte against the actual
  document this session).
- Section 10.1/10.2, read for contrast only (confirmed, not re-litigated):
  canonical technical context lives in the SPEC repo, never in code repos;
  10.2's per-role/task/repo-kind index example
  (`technical-context/indexes/backend.index.yml`) has a **nested**
  `mandatory.always`/`mandatory.whenTags`/`available` shape — a different
  artifact, a different domain (technical-context, not skills), and a
  different increment (INC-10). This design does not use 10.2's shape.
- Live code read fresh this session (not reused from the audit's prose):
  `Axiom/packages/skills/src/{types.ts,catalog.ts,apply.ts,materialize.ts}`,
  `Axiom/packages/skills/tests/catalog.test.ts`,
  `Axiom/packages/doctor/src/checks.ts` (TC-010 region, lines ~1897-2089),
  `Axiom.Spec/general-spec.md` (repository-roles section) and
  `Axiom.Spec/specs/increments/INC-20260702-axiom-redesign-roadmap/README.md`
  (lines 140-190, the `axiom.spec/` colocation discussion).

### Source doc 9.4's worked example (verbatim, re-confirmed)

```yaml
role: backend-developer

mandatory:
  - id: backend.repo-rules
    path: skills/backend-developer.md

available:
  - id: backend.ef-core
    path: skills/ef-core.md
    tags: [ef-core, database, persistence]

  - id: backend.migrations
    path: skills/migrations.md
    tags: [migration, schema, database]

  - id: backend.api-controller
    path: skills/api-controller.md
    tags: [api, controller, endpoint]

  - id: backend.testing
    path: skills/testing.md
    tags: [testing, unit-test, integration-test]
```

Flat, no `mandatory.always`/`mandatory.whenTags` nesting (that is 10.2's
shape, a different domain). No `schemaVersion` field is shown in the
source doc's example itself — that omission is addressed by Q-skills-2
below, since section 5.5 separately mandates versioning for
`skills.index` regardless of what the illustrative YAML snippet shows.

### Ground truth confirmed from live code (not re-derived, re-verified)

- `SkillsCatalog { schemaVersion: 1; skills: CatalogEntry[] }`,
  `CatalogEntry { id, name, version, source, status, securityCheckStatus,
  bundleHash }` (`Axiom/packages/skills/src/catalog.ts`) — flat, one
  global list, no role/grouping concept anywhere in `@axiom/skills`.
  Confirmed unchanged; this design does not touch this type.
- Physical catalog path is resolved as
  `path.join(rootPath, 'axiom.spec', 'config', 'skills-catalog.yaml')`
  (`apply.ts`'s exported `SKILLS_CATALOG_RELATIVE_PATH` constant, reused
  verbatim by `materialize.ts`'s caller and `doctor`'s TC-010 check).
  `rootPath` here is **the product project's own repo root** — confirmed
  independently by `catalog.test.ts`'s integration test, which computes
  `repoRoot = path.resolve(__dirname, '..', '..', '..', '..')` starting
  from `Axiom/packages/skills/tests/` and lands on `Axiom/` itself, then
  reads `Axiom/axiom.spec/config/skills-catalog.yaml`. There is no
  sibling-repo indirection anywhere in `@axiom/skills`'s source or tests:
  `rootPath` is always "the one repo this tool is running against,"
  resolved by `@axiom/project-resolution`/topology elsewhere in the
  monorepo, never a separate "SDD repo" path.
- `materialize.ts`'s `materializationPath(rootPath, id)` writes to
  `<rootPath>/.opencode/agents/<id>/SKILL.md` — again rooted at the same
  single `rootPath`, confirming there is exactly one repo-root concept in
  play for this whole package, not two.
- `Axiom.Spec/general-spec.md`'s "Repository roles (as actually
  practiced)" section states explicitly: `Axiom` is "the actual
  implementation repository for the product code described by this spec
  chain," and `Axiom.SDD` "holds the canonical lightweight-SDD workflow
  rules... Not itself a target for product code from this increment
  chain." This is this **workspace's own** meta-level convention (how the
  Axiom-building work itself is organized), which is a distinct question
  from what a project **built and managed by** the finished Axiom product
  does with its own `sddRepo`/`specRepo`/code-repo topology (per
  `TopologyManifest`, also documented in `general-spec.md`). Both facts
  are load-bearing for Q-skills-3 below, for two different reasons.

## Scope

- Resolve Q-skills-1 (file placement) with a concrete path/filename
  pattern, grounded in `@axiom/skills`'s actual `rootPath` resolution.
- Resolve Q-skills-2 (versioning scope): confirm or override the audit's
  own recommendation (independent `schemaVersion` on the new artifact,
  existing catalog untouched).
- Resolve Q-skills-3 (SDD-repo naming ambiguity): produce a documented,
  reusable answer for what source doc 9.1's "SDD repo" maps to, for (a)
  this workspace's own dogfooding practice and (b) a real Axiom-managed
  product project (the fictional `kvp25`-style case the source docs use).
- Design the full TypeScript/YAML shape of the new role-index artifact,
  adapting source doc 9.4's field names to this codebase's existing
  conventions where they differ.
- Provide one fully worked example (`backend-developer`), matching source
  doc 9.3/9.4's own worked example, expressed in the new shape.
- Hand off an unambiguous, implementation-ready contract to
  docs-skills-writer (and, per the audit's brief, eventually
  validator-reviewer for the `doctor` check).

## Non-goals

- No implementation of this schema as TypeScript types, a Zod schema, a
  loader function, or a real YAML file inside `Axiom/packages/skills/src/`
  or `Axiom/axiom.spec/`. This is a design-only increment, same posture as
  the sibling `INC-20260702-registry-manifest-schema-v2-design` increment
  this design's style follows.
- No change to `SkillsCatalog`, `CatalogEntry`, `loadSkillsCatalog`,
  `applySkillSet`, `materializeSkillSet`, or the `axiom skills` CLI
  command. Confirmed additive per the audit; nothing here revises that
  finding.
- No design of source doc 9.3's role-catalog markdown *content* — that is
  docs-skills-writer's job next; this increment only fixes where 9.3's
  markdown physically lives (via the same Q-skills-1/Q-skills-3
  reasoning) so docs-skills-writer does not have to re-derive it.
- No design of source doc 10.2's technical-context per-role index
  (`mandatory.always`/`whenTags`) — that remains INC-10, a different
  increment for a different domain, per the audit's own non-goals.
- No `doctor` check code (`checks.ts` edits) — validator-reviewer's job,
  per the audit's brief item 6. This design states what such a check
  would need to verify (for validator-reviewer's future benefit) but does
  not write it.

## Acceptance criteria

- [x] Q-skills-1 resolved with an exact path/filename pattern and
      rationale grounded in `@axiom/skills`'s actual `rootPath`
      resolution (not a guess).
- [x] Q-skills-2 resolved (confirmed or overridden) with explicit
      reasoning.
- [x] Q-skills-3 resolved with a documented mapping usable by future
      increments/docs, distinguishing this workspace's own dogfooding
      practice from a real Axiom-managed product project's topology.
- [x] A complete TypeScript type (and equivalent YAML shape) for the
      role-index artifact exists, with field-by-field justification for
      any deviation from source doc 9.4's literal field names.
- [x] One fully worked example (`backend-developer`) is included, using
      the resolved file-placement decision and the resolved schema.
- [x] No implementation code or real YAML file was written under
      `Axiom/packages/*` or `Axiom/axiom.spec/*`.
- [x] Next step (docs-skills-writer) is named with its exact required
      inputs.

## Open questions

None remain open at this increment's own scope. Q-skills-1, Q-skills-2,
and Q-skills-3 are resolved below. One residual item is carried forward,
not newly opened: per the audit's "Implementation notes," the physical
`axiom.spec/config/skills-catalog.yaml` file was not found on disk in the
`Axiom` repo at audit time despite being referenced consistently by three
independent code paths. This design's file-placement decision (below)
places the new role-index directory as a sibling under the same
not-yet-materialized `axiom.spec/config/` tree; whoever first authors
either file (existing catalog or new role-index) should create the parent
directory structure as needed — this is a pre-existing gap, not something
this increment causes or must fix.

## Assumptions

- "Role" continues to mean the source doc's product/task role concept
  (e.g. `backend-developer`), not `@axiom/agents`' `AgentRole` closed enum
  — unchanged from the audit's own Assumptions section.
- The existing catalog's `status`/`securityCheckStatus`/`bundleHash`
  fields remain untouched and are not referenced or duplicated by the new
  role-index shape (per the audit's field-by-field diff, "must be
  preserved, not dropped" — preserved here by simply not touching
  `CatalogEntry` at all).
- Following `INC-20260702-registry-manifest-schema-v2-design`'s own
  established style for this codebase (re-read in full this session as
  the explicit style/rigor reference this increment was asked to match),
  every resolution below is grounded in a direct re-read of live source,
  not inference from prior prose, and every open question gets an
  explicit decision plus rationale rather than being deferred again.

## Implementation notes

### Q-skills-1 — File placement

**Decision: sibling directory to the existing catalog, one file per
role, under `axiom.spec/config/skills-index/<role>.yaml` (relative to the
same `rootPath` the existing catalog already resolves against).**

Full resolved path pattern:

```
<rootPath>/axiom.spec/config/skills-index/<role>.yaml
```

Example: `<rootPath>/axiom.spec/config/skills-index/backend-developer.yaml`.

Rationale, grounded in the actual code (not a stylistic preference):

- `@axiom/skills`'s only notion of "where do I read project-scoped skill
  config from" is `rootPath`, and the **only** existing anchor for that is
  `SKILLS_CATALOG_RELATIVE_PATH = path.join('axiom.spec', 'config',
  'skills-catalog.yaml')` (`apply.ts`). There is no second root, no
  sibling-repo indirection, and no existing "SDD-scoped vs
  product-scoped" split anywhere in `@axiom/skills`'s source or tests
  (confirmed again this session, not just carried over from the audit).
  Placing the new artifact under a different repo (a literal "SDD repo"
  distinct from `rootPath`) would require inventing a second
  root-resolution mechanism that nothing in `@axiom/skills` currently
  has, has any extension point for, or was asked to add — that would be
  exactly the kind of speculative architecture `Axiom.SDD/AGENTS.md`
  prohibits ("do not create speculative architecture," "do not introduce
  mandatory indexes" beyond what's asked).
- `axiom.spec/config/` is already the established convention directory
  for versioned, curated (non-generated) project configuration in this
  codebase — the existing skills catalog lives there, and
  `Axiom.Spec/specs/increments/INC-20260702-axiom-redesign-roadmap/README.md`
  (line 119) independently corroborates that "section 9's skill
  index/catalog concept" is expected to be "colocated under `axiom.spec/`
  inside the same repo." A new `skills-index/` subdirectory next to
  `config/skills-catalog.yaml` (both under `axiom.spec/config/`) is the
  minimal, most-consistent placement — no new top-level convention is
  introduced.
- One file per role (`<role>.yaml`), not one combined multi-role file,
  matches source doc 9.4's own example directly (it shows a single
  document with a top-level `role: backend-developer` scalar field, which
  only makes sense for a one-role-per-document convention — a combined
  file would need to nest multiple `role` blocks under some wrapper key
  that 9.4 never shows). One file per role also mirrors 9.3's own
  "one MD per role" convention (9.3 and 9.4 are clearly meant to be
  parallel, one-document-per-role artifacts for the same role, just two
  different formats — human-readable catalog vs. machine-readable index).
- This directly resolves the audit's own phrasing of Q-skills-1
  ("sibling to catalog... `axiom.spec/config/skills-index/<role>.yaml`")
  — that was the audit's own first-listed option, and this design
  confirms it rather than choosing the second option (a separate
  SDD-repo-scoped folder), because `@axiom/skills`'s code gives no
  evidence such a second root exists or is intended for this package.

### Q-skills-2 — Versioning scope

**Decision: confirmed, override not needed. The new role-index artifact
gets its own independent `schemaVersion: 1`. The existing
`skills-catalog.yaml`'s `schemaVersion: 1` is untouched.**

Rationale:

- The two artifacts are structurally and behaviorally independent: the
  catalog answers "what skill content exists and is it
  approved/security-checked," the role-index answers "which of those
  skills does a given role need, and how (mandatory vs available)." They
  are read by different (future) loaders, serve different callers, and
  neither's shape constrains the other's — the role-index *references*
  catalog `id`s (see schema below) but does not embed or duplicate
  catalog fields, so a schema change to one has no forced ripple to the
  other. This mirrors the exact reasoning already used and accepted in
  this codebase for `projects.yml`'s `schemaVersion: 2` vs `axiom.yaml`'s
  independent `schemaVersion: 2`
  (`INC-20260702-registry-manifest-schema-v2-design/README.md`: "two
  independently versioned artifacts... they happen to share the numeral
  by coincidence... not because they are coupled") — same pattern,
  applied here with the numbers *not* coinciding (existing catalog stays
  at `1`, new artifact starts at its own `1`), which is the even simpler
  case.
- Section 5.5's rule ("`skills.index` decides what an agent must read/do,
  therefore must be versioned") is about the role-index specifically —
  the audit already established that the *existing* catalog's
  `schemaVersion: 1` field was not something the two source documents
  ever discuss at all (the catalog's review/security-gate fields are a
  pre-existing, source-doc-independent addition). Bumping the existing
  catalog's version would imply the catalog's own shape changed, which it
  has not (confirmed unchanged, see Non-goals) — a version bump with no
  shape change would be a false signal to any future reader/loader.
  `loadSkillsCatalog` today hard-rejects any `schemaVersion !== 1`
  (`catalog.ts` line ~113-118); bumping it without a real shape change
  would immediately break every existing valid catalog document for zero
  benefit.
- Net effect: this confirms the audit's own recommendation exactly as
  stated in its Q-skills-2 text ("this audit recommends the latter... new
  artifact, independent version") — no override.

### Q-skills-3 — "SDD repo" naming resolution

**Decision: source doc 9.1's "SDD repo" maps to different concrete things
depending on which of two distinct situations is being discussed, and
both must be named explicitly and separately going forward — they are not
the same referent and must not be conflated:**

1. **This workspace's own meta-level practice** (how the Axiom-building
   work itself — this three-repo `Axiom.SDD`/`Axiom.Spec`/`Axiom` parent
   workspace — is organized): source doc 9.1's "SDD repo" here maps to
   **`Axiom.SDD`**, per `Axiom.Spec/general-spec.md`'s own stated
   convention (`Axiom.SDD` "holds the canonical lightweight-SDD workflow
   rules"). This is the sense used by, e.g., `Axiom.SDD/AGENTS.md` itself
   and every increment spec in this chain when they say "the SDD repo" in
   the context of *this* workspace. **However — and this is the
   important qualifier — no product-code SDD-skill artifacts (9.1's
   `create-increment`/`review-plan`-style skill examples) are stored
   inside `Axiom.SDD` today, because `Axiom.SDD` is explicitly "not
   itself a target for product code from this increment chain."** This
   sense of "SDD repo" is about *this workspace's own governance*, not
   about a schema/file-placement target — it has no bearing on where
   `@axiom/skills`' role-index or catalog files physically go, because
   those are product artifacts of the `Axiom` product, not artifacts of
   this workspace's own SDD process.

2. **A real Axiom-managed product project's own topology** (the fictional
   `kvp25`-style case, and — per Axiom's own dogfooding practice —
   `Axiom` itself, since `Axiom` is built through, and validated against,
   its own product model): here, source doc 9.1's "SDD repo" maps to
   whichever repo `TopologyManifest.sddRepo` resolves to for that project
   (`Axiom/packages/topology/src/types.ts`, documented in
   `Axiom.Spec/general-spec.md`'s "Project topology model" section). In
   the **default, MVP `single-repo` mode** — which is what `Axiom`'s own
   dogfooded topology currently uses, and what `catalog.test.ts`'s
   integration test confirms in practice — `sddRepo` and `specRepo` both
   resolve to the same `rootPath` as the code repo itself. This is why
   `@axiom/skills`'s `rootPath` resolution shows only **one** root, not
   two: for `Axiom`'s own topology today, "the SDD repo" and "the code
   repo" and "the spec repo" are the literal same filesystem path. In a
   **`multi-repo`** topology (schema-supported, not yet validated against
   a real non-trivial project per `general-spec.md`'s own caveat),
   `sddRepo` could resolve to a genuinely separate path from the code
   repo — but nothing in `@axiom/skills`'s current code reads
   `TopologyManifest` or resolves a second root; it always takes a single
   `rootPath` parameter from its caller. This means: **today, for any
   project (including `Axiom` dogfooding itself), 9.1's "SDD skills live
   in the SDD repo" is satisfied trivially because `sddRepo === rootPath`
   in single-repo mode** — there is no separate physical location to
   design for yet. If/when a project actually runs in `multi-repo` mode
   with a distinct `sddRepo`, `@axiom/skills`'s callers (not
   `@axiom/skills`'s own internals) would need to pass `sddRepo`'s path
   as `rootPath` for 9.1-classified skills and the code repo's path for
   9.2-classified skills — a caller-side routing decision, not a schema
   change to anything designed in this increment.

**Practical consequence for this increment and for docs-skills-writer
next**: the role-index and role-catalog-markdown placement decided under
Q-skills-1 (`<rootPath>/axiom.spec/config/skills-index/<role>.yaml`, and
by the same reasoning, role-catalog markdown under
`<rootPath>/axiom.spec/target-axiom-skills/catalogs/<role>.md` — see
"Sibling decision for 9.3 markdown" below) is **not** affected by which
sense of "SDD repo" applies, because both artifacts are rooted at
whatever `rootPath` the caller already resolved (today, always a single
path per project) — no new repo-splitting logic is being introduced by
either artifact. The distinction above matters for **prose clarity in
future specs**, not for this increment's schema: future documents should
say "this workspace's own SDD repo (`Axiom.SDD`)" when talking about this
workspace's governance, and "the project's `sddRepo` topology role (today
== code repo root in single-repo mode)" when talking about a
product-managed project's skill placement — never bare "the SDD repo"
without qualifying which of the two is meant, since they answer different
questions and, in single-repo mode, coincidentally share a physical path
while remaining conceptually distinct.

#### Sibling decision for 9.3 markdown (placement only, not content)

Not required by the audit's acceptance criteria, but blocking for
docs-skills-writer per the audit's own Q-skills-3 framing ("blocking
before docs-skills-writer decides where new role-catalog markdown files
should physically go"), so resolved here using the same reasoning as
Q-skills-1: role-catalog markdown (9.3) lives at
`<rootPath>/axiom.spec/target-axiom-skills/catalogs/<role>.md`, colocated
under the same `axiom.spec/target-axiom-skills/` tree the existing
catalog's `source` fields already point into (confirmed by
`catalog.test.ts`'s fixture data, e.g. `source:
axiom.spec/target-axiom-skills/axiom-sdd-orchestrator.md`) — a new
`catalogs/` subfolder there, parallel to `skills-index/` under
`config/`, keeps both new 9.3/9.4 artifacts next to the existing
convention they extend rather than introducing a third top-level
location.

### Role-index schema design

```ts
// packages/skills/src/role-index.ts (proposed location; not created by
// this increment — see Non-goals)

/** Schema version of a per-role skills-index document. Independent from
 *  SkillsCatalog.schemaVersion (see Q-skills-2 resolution above) — the
 *  two artifacts are versioned separately because their shapes evolve
 *  independently. */
export type SkillsRoleIndexSchemaVersion = 1;

/** One entry in a role-index's `mandatory` list: a skill every agent
 *  acting in this role must load, unconditionally. */
export interface SkillsRoleIndexMandatoryEntry {
  /** References an existing CatalogEntry.id (@axiom/skills' catalog).
   *  Not a new identifier space — see "id reuses catalog id" below. */
  readonly id: string;
  /**
   * Optional direct path to the skill's source markdown, relative to
   * rootPath. Only needed for a skill that is NOT (yet) present in
   * skills-catalog.yaml (e.g. a 9.2-style repo-technical skill with no
   * catalog entry today). When the id already exists in the catalog,
   * omit `path` and resolve it via the catalog's own `source` field
   * instead of duplicating it here — see "path is a fallback pointer,
   * not a duplicate" below.
   */
  readonly path?: string;
  /**
   * Optional human-readable reason this skill is mandatory for the
   * role. Not present in source doc 9.4's own example, but a small,
   * additive, self-documenting field with no existing-code conflict —
   * included because it costs nothing and makes a `mandatory` list
   * self-explanatory without cross-referencing the 9.3 markdown catalog.
   * Optional so a minimal entry (`{ id }` or `{ id, path }`) is always
   * valid and matches 9.4's example exactly when omitted.
   */
  readonly reason?: string;
}

/** One entry in a role-index's `available` list: a skill an agent in
 *  this role MAY load on demand, selected via `tags`. */
export interface SkillsRoleIndexAvailableEntry {
  /** Same id-reuse convention as SkillsRoleIndexMandatoryEntry.id. */
  readonly id: string;
  /** Same fallback-pointer convention as
   *  SkillsRoleIndexMandatoryEntry.path. */
  readonly path?: string;
  /** Tags used for semantic/topic-based selection (source doc 9.4's own
   *  field, e.g. ['ef-core', 'database', 'persistence']). Required on
   *  `available` entries (not optional) because tags are the only
   *  selection mechanism 9.4 defines for this list — an available entry
   *  with no tags could never be found by the (future) selection logic
   *  this index exists to support. */
  readonly tags: ReadonlyArray<string>;
  /** Optional one-line human summary, same additive rationale as
   *  `reason` above. Kept separate from `reason` (mandatory-only) and
   *  `tags` (machine-selection) since summary is prose for a human or
   *  an LLM deciding whether to request the skill. */
  readonly summary?: string;
}

/** Document persisted at
 *  `<rootPath>/axiom.spec/config/skills-index/<role>.yaml`. One file
 *  per role (see Q-skills-1 resolution). */
export interface SkillsRoleIndex {
  readonly schemaVersion: SkillsRoleIndexSchemaVersion;
  /** Role this index applies to (e.g. 'backend-developer'). Matches the
   *  filename stem by convention (<role>.yaml), not enforced by the
   *  type itself — a loader-level invariant, same posture as
   *  SkillsCatalog's own id-uniqueness check being loader-enforced, not
   *  type-enforced. */
  readonly role: string;
  /**
   * Optional repo-kind scoping, additive beyond source doc 9.4's literal
   * example (9.4 shows no such field). Included because 9.1/9.2 already
   * establish two distinct skill populations (SDD-repo skills vs
   * per-repo-code technical skills) that a single role could plausibly
   * need to distinguish (e.g. a 'backend-developer' role's mandatory
   * list might differ between an 'sdd' repo context and a 'code' repo
   * context). Left optional and unconstrained (string, not a closed
   * enum) because @axiom/topology's own RepoRoleKey is already an
   * open-ended string (per registry-manifest-schema-v2-design's own
   * RepoRoleKey type) — this index must not be stricter than the
   * topology model it may eventually be cross-referenced against. Not
   * required by 9.4; omit entirely for a role-index that applies
   * uniformly regardless of repo kind (the common case, and the shape
   * used in the worked example below).
   */
  readonly repoKinds?: ReadonlyArray<string>;
  readonly mandatory: ReadonlyArray<SkillsRoleIndexMandatoryEntry>;
  readonly available: ReadonlyArray<SkillsRoleIndexAvailableEntry>;
}
```

Field-naming deviations from source doc 9.4's literal example, each
justified:

| 9.4 literal field | This schema | Deviation kind | Reason |
|---|---|---|---|
| `role` | `role` | none | kept verbatim |
| `mandatory[].id` | `mandatory[].id` | none | kept verbatim, and reuses `CatalogEntry.id` (see below) |
| `mandatory[].path` | `mandatory[].path?` (optional) | narrowed to optional | 9.4's example always shows `path`, but this codebase already has a stable `id` -> `source` resolution via the catalog (`getSkillById`) for any skill that is cataloged; requiring `path` on every entry would force duplicating a fact the catalog already owns for the common case. `path` remains available, required in practice only for the *uncommon* case (a skill not yet cataloged) |
| `available[].id` | `available[].id` | none | kept verbatim |
| `available[].path` | `available[].path?` (optional) | narrowed to optional | same reasoning as `mandatory[].path` |
| `available[].tags` | `available[].tags` (required, not optional) | none (kept required) | 9.4 always shows `tags` on `available` entries; made required by this design, not just kept as shown, because `available` without `tags` has no selection mechanism at all in either source doc |
| *(absent in 9.4)* | `mandatory[].reason?`, `available[].summary?` | new, optional, additive | small self-documentation fields with no existing-code conflict; both optional so a minimal entry matching 9.4's literal example exactly (no `reason`/`summary`) is always valid |
| *(absent in 9.4)* | `repoKinds?` | new, optional, additive | see rationale in the type's own doc comment above; not required by 9.4, omitted in the worked example below since the 9.4 example itself does not scope by repo kind |
| *(absent in 9.4)* | `schemaVersion` | new, required | resolves Q-skills-2; 9.4's own example is illustrative prose and does not show every required production field, same posture the audit itself took toward the existing catalog's `status`/`securityCheckStatus`/`bundleHash` fields having "no analog in the new docs" without that being a contradiction |

**`id` reuses catalog id, does not invent a new identifier space.** A
`SkillsRoleIndexMandatoryEntry.id`/`SkillsRoleIndexAvailableEntry.id`
value is expected to match an existing `CatalogEntry.id`
(`Axiom/packages/skills/src/catalog.ts`) whenever the referenced skill is
already cataloged. This is the audit's own brief item 2 ("Prefer
referencing existing `skills-catalog.yaml` `id`s over duplicating `path`
when the skill is already cataloged"), confirmed and formalized here as
part of the schema rather than left as a loose recommendation: a future
loader for this artifact should resolve `path` via
`getSkillById(catalog, entry.id)?.source` when `entry.path` is absent,
and only fall back to a literal `entry.path` for skills with no catalog
entry (e.g. 9.2-style repo-technical skills that have not been onboarded
into the catalog yet). This is a design note for the future loader
implementer, not code shipped by this increment.

**`path` is a fallback pointer, not a duplicate.** The schema does not
make `path` mandatory precisely so that a fully-cataloged role's
role-index stays a pure reference list (`{id, tags}` only, no path
duplication drifting out of sync with the catalog's own `source` field) —
this avoids introducing a second, competing source of truth for a path
that already has one.

### Worked example — `backend-developer`

File: `<rootPath>/axiom.spec/config/skills-index/backend-developer.yaml`

```yaml
schemaVersion: 1
role: backend-developer

mandatory:
  - id: backend.repo-rules
    path: skills/backend-developer.md
    reason: Repo-wide conventions every backend change must follow.

available:
  - id: backend.ef-core
    path: skills/ef-core.md
    tags: [ef-core, database, persistence]
    summary: Entities, DbContext, repositories, and query patterns.

  - id: backend.migrations
    path: skills/migrations.md
    tags: [migration, schema, database]
    summary: EF Core migration authoring and review checklist.

  - id: backend.api-controller
    path: skills/api-controller.md
    tags: [api, controller, endpoint]
    summary: REST endpoint, controller, and request/response contract rules.

  - id: backend.testing
    path: skills/testing.md
    tags: [testing, unit-test, integration-test]
    summary: Unit/integration test conventions for backend changes.
```

This example intentionally mirrors source doc 9.4's own worked example
field-for-field (same `role`, same four `available` entries with the same
`id`/`path`/`tags`), plus the two additive optional fields (`reason` on
the mandatory entry, `summary` on each available entry) and the required
`schemaVersion: 1`. None of these five skill ids
(`backend.repo-rules`/`backend.ef-core`/`backend.migrations`/
`backend.api-controller`/`backend.testing`) exist in the current
7-entry `skills-catalog.yaml` fixture (`axiom-sdd-orchestrator`,
`axiom-install-profile`, `axiom-capability-router`,
`axiom-context-persistence`, `axiom-token-optimization`,
`axiom-code-intelligence`, `axiom-telemetry`) — consistent with the
audit's own finding that the existing catalog covers the 9.1 population
(SDD-orchestration-flavored skills) only, with no 9.2-style backend
technical skills onboarded yet. This is why every entry in the worked
example carries an explicit `path` (the fallback case, per the schema
design above) rather than omitting it in favor of catalog resolution —
today, resolving purely by `id` against the live catalog would fail for
all five, since none are catalogued yet. When/if these five skills are
added to `skills-catalog.yaml` in a future increment, `path` becomes
redundant and may be dropped from the role-index entries at that time
without a schema change (a loader-behavior change only, per the
`id`-reuse design above).

### What validator-reviewer will need later (not designed here)

Per the audit's brief item 6, a future `doctor` check (working name:
TC-012, next available number after TC-011/agents-catalog-coverage)
should verify, per role-index file: (a) it parses and matches this
schema, (b) every `mandatory`/`available` entry's `id` either resolves in
`skills-catalog.yaml` or has an explicit `path` that exists on disk, and
(c) no duplicate `id` within the same file's combined
`mandatory`+`available` lists (mirroring the existing catalog's own
duplicate-id rejection in `loadSkillsCatalog`). This paragraph is a
forward note for validator-reviewer, not a check implementation.

## Validation

`No validation command was found. Performed best-effort validation by
inspecting changed files and checking consistency against the requested
behavior.`

This increment is design/spec-only (no implementation code or YAML file
changed in `Axiom/packages/*` or `Axiom/axiom.spec/*`). Best-effort
validation performed:

- Re-read source doc section 9 (all four subsections) directly from
  `axiom_decisiones_sesion_prompt_implementacion.md` this session, byte-
  for-byte, rather than relying on the audit's paraphrase — the 9.4
  worked example reproduced above was copy-checked against the live
  document.
- Re-read `Axiom/packages/skills/src/{types.ts,catalog.ts,apply.ts,
  materialize.ts}` and `Axiom/packages/skills/tests/catalog.test.ts` in
  full this session to confirm the `rootPath`/single-root finding
  underlying Q-skills-1 and Q-skills-3, rather than trusting the audit's
  prior summary of the same files.
- Re-read `Axiom/packages/doctor/src/checks.ts`'s TC-010 region to
  confirm the existing check's exact scope (source-file-existence +
  bundleHash-consistency, not role/index-aware), grounding the "what
  validator-reviewer will need later" forward note.
- Re-read `Axiom.Spec/general-spec.md` and
  `Axiom.Spec/specs/increments/INC-20260702-axiom-redesign-roadmap/README.md`
  (the `axiom.spec/` colocation and topology-model sections) to ground
  Q-skills-3's two-senses resolution in this workspace's own documented
  conventions rather than assumption.
- Cross-checked every new/deviating field in the proposed
  `SkillsRoleIndex` shape against `@axiom/skills`'s existing types for
  naming collisions or contradictions (none found: `schemaVersion`,
  `id`, `path`, `tags` are all reused verbatim from existing
  `CatalogEntry`/`SkillsCatalog` naming conventions; `reason`, `summary`,
  `repoKinds` are new names chosen to not collide with any existing field
  anywhere in `@axiom/skills`, `@axiom/agents`, or `@axiom/topology`).
- Followed `INC-20260702-registry-manifest-schema-v2-design/README.md`'s
  established style (explicit per-question resolution with rationale
  grounded in live code, a field-origin/deviation table, a fully worked
  example, an explicit next-step handoff) as the rigor bar this increment
  was asked to match.
- `Axiom/package.json`'s real validation commands (build/test/doctor/
  typecheck) were not run, because no code was changed in `Axiom/` for
  this increment; running them would only assert pre-existing state.

## Result

Resolved all three of the migration-engineer audit's open questions:

- **Q-skills-1**: role-index files live at
  `<rootPath>/axiom.spec/config/skills-index/<role>.yaml`, one file per
  role, sibling to the existing `axiom.spec/config/skills-catalog.yaml`
  — confirming the audit's own first-listed option, grounded in
  `@axiom/skills` having exactly one root-resolution mechanism
  (`rootPath`), never a second sibling-repo path.
- **Q-skills-2**: confirmed the audit's own recommendation — the new
  role-index artifact gets its own independent `schemaVersion: 1`; the
  existing catalog's `schemaVersion: 1` is untouched, since its shape has
  not changed and a version bump with no shape change would be a false
  signal `loadSkillsCatalog` would also hard-reject today.
- **Q-skills-3**: source doc 9.1's "SDD repo" resolves to two distinct
  referents that must not be conflated going forward — this workspace's
  own `Axiom.SDD` (a governance-only referent, irrelevant to
  `@axiom/skills`' file placement since `Axiom.SDD` holds no product
  code) versus a real Axiom-managed project's `TopologyManifest.sddRepo`
  role (a schema-level referent that, in today's default `single-repo`
  mode, is the literal same path as the code repo root — which is exactly
  why `@axiom/skills` only ever resolves one `rootPath`). Neither sense
  changes this increment's file-placement decision, since both new
  artifacts (role-index, role-catalog markdown) are rooted at whatever
  single `rootPath` the caller already resolves today.

Produced the full `SkillsRoleIndex`/`SkillsRoleIndexMandatoryEntry`/
`SkillsRoleIndexAvailableEntry` TypeScript design (schema-versioned,
`id`-reuse against the existing catalog, optional `path` as a
fallback-only pointer, required `tags` on `available` entries, two small
additive optional fields `reason`/`summary`, and an optional
`repoKinds` scoping field), plus one fully worked `backend-developer`
example matching source doc 9.3/9.4's own worked example. Also resolved,
as a non-blocking side effect requested by the audit's own Q-skills-3
framing, where source doc 9.3's role-catalog markdown should physically
live (`<rootPath>/axiom.spec/target-axiom-skills/catalogs/<role>.md`),
so docs-skills-writer does not have to re-derive a placement decision
from scratch next.

No implementation code or YAML file was written under
`Axiom/packages/*` or `Axiom/axiom.spec/*`, per this increment's
non-goals. Status is `pending`, not `closed` (see Closure rationale
below).

## General spec integration

No integration into `Axiom.Spec/general-spec.md` performed yet.
Rationale, consistent with the sibling design increment
`INC-20260702-registry-manifest-schema-v2-design`'s own closure
reasoning: this increment finalizes a *design* (schema, placement,
naming, versioning) that has not yet survived contact with
implementation (no loader exists yet, no real YAML file has been
authored and validated against a live `doctor` check). Locking this into
`general-spec.md` now would consolidate a design that could still shift
once docs-skills-writer and validator-reviewer's downstream work
surfaces a mismatch. The correct point to integrate this into
`general-spec.md` is after validator-reviewer closes out the full INC-09
chain (migration-engineer -> schema-writer -> docs-skills-writer ->
validator-reviewer), when the shape is both designed and proven — same
threshold already used for INC-01's schema-writer step.

## Next step recommendation

Hand off to **docs-skills-writer** next, per INC-09's subagent sequence.
Exact inputs it needs from this document:

1. **The role-index schema** (`SkillsRoleIndex` and its two entry types)
   and the worked `backend-developer` example above, as the machine-
   readable artifact its markdown catalog (9.3) must stay consistent
   with — every skill `id` docs-skills-writer documents in prose should
   have a corresponding entry in the same role's role-index YAML (or be
   flagged as a gap if the role-index has not been authored yet for that
   role).
2. **The Q-skills-1 file-placement decision** for the role-index
   (`axiom.spec/config/skills-index/<role>.yaml`) and the "Sibling
   decision for 9.3 markdown" sub-section's placement for the markdown
   catalog itself (`axiom.spec/target-axiom-skills/catalogs/<role>.md`)
   — docs-skills-writer should write to that exact path pattern, not
   invent a different one.
3. **The Q-skills-3 two-senses resolution** — so any prose
   docs-skills-writer writes about "the SDD repo" is precise about which
   of the two referents it means, avoiding the exact ambiguity this
   increment resolved.
4. **The audit's own 9.3 worked example** (`Axiom/packages/skills`'
   parent audit README, "Backend Developer Skill Catalog" reproduced
   there) as the prose-content template — this increment does not
   redesign 9.3's content shape, only where it lives.

After docs-skills-writer, continue the audit's stated sequence:
**validator-reviewer** extends `doctor`'s skills category (alongside
TC-010) with a new check for the role-index file (working name TC-012,
per "What validator-reviewer will need later" above), following the
existing `pass/fail/warn/skip` + evidence convention — validator-reviewer
should not invent a new check-result shape, consistent with the audit's
own brief item 6.
