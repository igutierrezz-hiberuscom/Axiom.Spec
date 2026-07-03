# Increment: Reconcile technical context structure — schema-writer design

Status: closed
Date: 2026-07-02

Closed as part of INC-10's overall closure by
`INC-20260702-technical-context-index-reconcile-validator/README.md`
(validator-reviewer, final step of the chain). See that spec's "INC-10
closure assessment" section for the full closure rationale covering all
four increments in this chain.

## Goal

Execute the **schema-writer** step of INC-10's subagent sequence
(migration-engineer -> **schema-writer** -> next step, resolved below),
following on from
`Axiom.Spec/specs/increments/INC-20260702-technical-context-index-reconcile/README.md`
(the migration-engineer audit). This increment resolves the audit's three
open questions (Q-context-1, Q-context-2, Q-context-3) and produces the
exact TypeScript type/schema for the new `@axiom/technical-context`
index artifact (source doc section 10.2's nested shape), including its
file placement, loader/validator design, and the `resolveMandatoryDocuments`
tag-matching helper. It does **not** implement any of this in
`Axiom/packages/*` — this is a design-only increment, same posture as
`INC-20260702-skills-role-index-reconcile-design` (INC-09's own
schema-writer step, which this design follows structurally).

## Context

Parent chain: `INC-20260702-axiom-redesign-roadmap` (parent roadmap) ->
`INC-20260702-technical-context-index-reconcile` (migration-engineer
audit, status `pending`) -> this increment (schema-writer design).

The migration-engineer audit is treated as authoritative input and is not
re-litigated here except where it explicitly delegated a decision to
schema-writer (its three open questions) or where its own brief needed a
concrete shape (its "Brief for schema-writer," items 1-4). Source
documents were re-read directly for this design, not paraphrased from the
audit:

- `axiom_decisiones_sesion_prompt_implementacion.md` section 10
  (`Contexto técnico`), re-read in full this session, byte-for-byte:
  - **10.1** (`Decisión principal`): canonical technical context always
    lives in the SPEC repo, never in code repos.
  - **10.2** (`Índices por rol`): "Debe haber índices detallados por
    rol/tarea/repo kind." The worked example's own file-path comment is
    literally `# project.spec/technical-context/indexes/backend.index.yml`
    (reproduced verbatim below, in "Source doc 10.2's worked example").
  - **10.3** (`Regla de mandatory documents`): "Si un índice por rol
    define documentos obligatorios, el MCP debe devolverlos
    directamente." Worked example (prose, not YAML): if
    `architecture.md` + `commands.md` are `mandatory always`,
    `get_implementation_context` must return both documents directly, in
    addition to the full index. Out of scope to implement here (INC-14),
    but its exact wording is load-bearing for confirming `always` +
    `whenTags` are additive, not exclusive — see "10.3 cross-check"
    below.
- Section 4.2 (`Manifests axiom.yml`, `Ejemplo para repo SPEC`), read
  fresh this session (not previously quoted by the audit) because it is
  the only other place in the source doc that names a technical-context
  path convention:
  ```yaml
  paths:
    technicalContext: technical-context/
  indexes:
    technicalContext: technical-context/indexes/
  ```
  This is a **spec-repo's own `axiom.yml` manifest**, declaring where
  *that specific repo* keeps its technical-context tree and index
  subfolder — a per-project, per-repo-manifest-declared path, not a
  hardcoded convention `@axiom/technical-context` can assume for every
  caller. This distinction is the crux of Q-context-3's resolution below.
- Section 8.3 (`get_implementation_context`'s output shape, read for
  contrast only, not re-litigated): shows `mandatory.technicalContext:
  Document[]`, `indexes.technicalContext: IndexSummary`,
  `recommended.technicalContext: string[]` as fields of the *tool's own
  response envelope* — a different, larger artifact (INC-14's job) that
  *consumes* this increment's index/resolution output as one input among
  several (skills, rules, ADRs). Confirms this increment's artifact is
  correctly scoped as a narrower building block, not the full tool
  response.
- `Axiom.Spec/specs/increments/INC-20260702-skills-role-index-reconcile-design/README.md`
  (INC-09's own schema-writer design), read in full this session as the
  explicit style/rigor reference this increment was asked to match: every
  open question gets an explicit decision with rationale grounded in live
  source (not inference), a field-origin/deviation table, one fully
  worked example, and an unambiguous next-step handoff.
- `Axiom/packages/skills/src/role-index.ts` (348 lines, full file),
  re-read fresh this session as the structural template this design must
  mirror for loader/validator error-handling style (never-throws,
  discriminated-union `{kind: 'ok'|'absent'|'malformed'}` results, a
  standalone `validate*` function reusable by a future `doctor` check
  without re-reading the file, hand-written narrowing with no Zod).
- `Axiom.Spec/general-spec.md`'s "Skills role-index" section (confirmed
  present, lines 148+), read to ensure this design's own eventual
  `general-spec.md` entry (deferred — see "General spec integration"
  below) will sit consistently alongside it.

### Source doc 10.2's worked example (verbatim, re-confirmed)

```yaml
# project.spec/technical-context/indexes/backend.index.yml

schemaVersion: 1
projectId: kvp25
role: backend-developer
repoKinds:
  - backend

mandatory:
  always:
    - id: backend.architecture
      path: ../backend/architecture.md
      reason: Required for any backend implementation

    - id: backend.commands
      path: ../backend/commands.md
      reason: Required before build/test/migration commands

  whenTags:
    - tags:
        - persistence
        - database
        - ef-core
      documents:
        - id: backend.persistence
          path: ../backend/persistence.md
          reason: Persistence task requires persistence context

available:
  - id: backend.api-patterns
    path: ../backend/api-patterns.md
    summary: API controller, route and DTO conventions
    tags:
      - api
      - controller
      - dto

  - id: backend.validation
    path: ../backend/validation.md
    summary: Business validation patterns
    tags:
      - validation
      - rules

  - id: backend.testing
    path: ../backend/testing.md
    summary: Unit/integration testing conventions
    tags:
      - testing
      - regression

  - id: backend.examples
    path: ../backend/examples.md
    summary: Real project examples for common backend changes
    tags:
      - examples
      - patterns
```

Every field this design's schema exposes is present in this example
already (`schemaVersion`, `projectId`, `role`, `repoKinds`,
`mandatory.always[].{id,path,reason}`,
`mandatory.whenTags[].{tags,documents[].{id,path,reason}}`,
`available[].{id,path,summary,tags}`) — no field is invented beyond it,
and no field shown here is dropped (see the field-table in "Technical-
context index schema design" below).

### 10.3 cross-check — `always` and `whenTags` are additive, not exclusive

Re-reading 10.3's own prose confirms the semantics needed for
`resolveMandatoryDocuments` (Q-context-2): "if `architecture.md` +
`commands.md` are `mandatory always`, `get_implementation_context` must
return both documents directly **and also the full index**." This
establishes two things load-bearing for this design:

1. `mandatory.always` entries are unconditionally mandatory — resolution
   never excludes them regardless of task tags.
2. The rule is phrased entirely in terms of `always`; it says nothing
   about `whenTags` being mutually exclusive with `always`, or about
   `whenTags` groups being mutually exclusive with each other. The
   natural reading — confirmed by 10.2's own example structure, where
   `whenTags` is a *sibling* array under the same `mandatory` object,
   not an alternative to `always` — is that a full mandatory-document
   resolution for a given task is the **union** of `always` plus every
   `whenTags` group whose condition is satisfied by that task's tags.
   This is the basis for Q-context-2's resolution below.

## Scope

- Resolve Q-context-1 (package placement): confirm or override the
  audit's own `@axiom/technical-context` new-package recommendation.
- Resolve Q-context-2 (`whenTags` matching ownership and semantics):
  decide whether the loader package or the caller owns tag-matching, and
  — the part the audit explicitly left open — whether a `whenTags`
  group's `tags` match via ALL-present or ANY-present against a task's
  tags.
- Resolve Q-context-3 (file-path convention): decide the exact resolved
  path pattern, reconciling source doc 10.2's own literal comment
  (`project.spec/technical-context/indexes/backend.index.yml`) and
  4.2's `axiom.yml`-manifest-declared `paths.technicalContext`/
  `indexes.technicalContext` fields against this codebase's actual
  `specScope`-resolution mechanism (audit Finding 2).
- Design the full `TechnicalContextIndex` TypeScript type family, field-
  for-field against 10.2's worked example, adapted only where this
  codebase's existing conventions (from `role-index.ts`) require it,
  with every deviation justified in a table.
- Design `resolveMandatoryDocuments(index, taskTags)` with its exact
  signature, return shape, and matching semantics, justified against
  10.3's own wording.
- Design `loadTechnicalContextIndex(rootPath, roleOrKind)` and
  `validateTechnicalContextIndex(parsed)`, mirroring
  `loadSkillsRoleIndex`/`validateSkillsRoleIndex`'s exact structure and
  error-handling style.
- Provide one fully worked example (`backend`), matching source doc
  10.2's own worked example as closely as possible, expressed under the
  resolved file-path convention.
- Hand off an unambiguous, implementation-ready contract, and recommend
  the next step in INC-10's sequence.

## Non-goals

- No implementation of this schema as TypeScript source, a real YAML
  file, or a `doctor` check inside `Axiom/packages/*` or
  `Axiom/axiom.spec/*`. Design-only, same posture as
  `INC-20260702-skills-role-index-reconcile-design` and
  `INC-20260702-registry-manifest-schema-v2-design`.
- No change to `@axiom/skills`'s `SkillsCatalog`/`SkillsRoleIndex` or to
  `@axiom/document-bootstrap`'s renderers/writer — confirmed
  structurally independent by the audit; nothing here revisits that.
- No implementation of `get_implementation_context` or any of source doc
  8.3's response-envelope fields (`mandatory.technicalContext`,
  `indexes.technicalContext`, `recommended.technicalContext`) — that
  remains INC-14, a different increment, per the audit's own non-goals.
  This design's `resolveMandatoryDocuments` is preparatory groundwork
  INC-14 will call, not a re-implementation of INC-14's own tool.
- No `doctor` check code (`checks.ts` edits) — deferred to whichever
  role picks up validator-reviewer's role next (see "Next step
  recommendation"). This design states what such a check would need to
  verify but does not write it.
- No speculative multi-tag-set matching modes (e.g. a configurable
  ALL-vs-ANY switch per index or per `whenTags` group). Source doc 10.2/
  10.3 show one concrete example and one concrete rule; this design
  picks the one semantics that fits both without adding a
  configuration surface nothing has asked for (`Axiom.SDD/AGENTS.md`'s
  "do not create speculative architecture").

## Acceptance criteria

- [x] Q-context-1 resolved (confirmed or overridden) with explicit
      reasoning grounded in the audit's own cohesion analysis and this
      design's independent re-check.
- [x] Q-context-2 resolved: matching ownership (library helper vs.
      caller) AND matching semantics (ALL vs. ANY), both decided with
      rationale grounded in 10.2/10.3's literal wording, not assumption.
- [x] Q-context-3 resolved with an exact, unambiguous resolved-path
      pattern and rationale reconciling 10.2's literal comment, 4.2's
      manifest fields, and the audit's Finding 2 (`specScope`
      resolution).
- [x] A complete TypeScript type family for `TechnicalContextIndex`
      exists, field-for-field justified against 10.2's worked example,
      with no invented fields and no dropped fields.
- [x] `resolveMandatoryDocuments`'s exact signature, return shape, and
      matching semantics are specified with rationale.
- [x] `loadTechnicalContextIndex`/`validateTechnicalContextIndex` are
      designed mirroring `loadSkillsRoleIndex`/`validateSkillsRoleIndex`'s
      structure and error-handling style.
- [x] One fully worked example (`backend`) is included, matching source
      doc 10.2's own worked example as closely as possible, under the
      resolved file-path convention.
- [x] No implementation code or real YAML file was written under
      `Axiom/packages/*` or `Axiom/axiom.spec/*`.
- [x] Next step is named with explicit rationale for which role picks it
      up (not a default assumption of docs-skills-writer).

## Open questions

None remain open at this increment's own scope. Q-context-1,
Q-context-2, and Q-context-3 are resolved below.

## Assumptions

- "Technical context" continues to mean source doc section 10's concept
  specifically (curated, versioned, per-role/per-repo-kind document
  indexes with conditional mandatory-document rules) — unchanged from
  the audit's own Assumptions section, not re-litigated here.
- `role`/`repoKinds` remain open strings (not closed enums), consistent
  with `SkillsRoleIndex.repoKinds`'s existing design and
  `@axiom/topology`'s `RoleAssignment.roleId` — confirmed already by the
  audit; this design does not deviate.
- Following `INC-20260702-skills-role-index-reconcile-design`'s
  established style bar for this codebase (re-read in full this
  session), every resolution below is grounded in a direct re-read of
  live source or the source doc itself, not inference from prior prose,
  and every open question gets an explicit decision plus rationale
  rather than being deferred again.

## Implementation notes

### Q-context-1 — Package placement

**Decision: confirmed. New small package, `@axiom/technical-context`.
No override.**

The audit's own reasoning (package-placement recommendation section) is
re-checked here against the same three candidates and holds without
qualification:

- Folding into `@axiom/skills` would blur that package's clean,
  single responsibility (installable/materializable/drift-checked skill
  units) with a fundamentally different artifact kind: a technical-
  context index has no install, no materialize, no drift-detection
  lifecycle at all — it is a static, versioned pointer index over
  markdown documents, read and resolved, never applied to a project's
  `.opencode/agents/` tree the way a skill is. The two packages would
  share only a *pattern* (defensive discriminated-union YAML loader,
  `specScope`-resolved path, `validate*`-reusable-by-doctor convention),
  never an executable code path, type, or constant — confirmed again
  this session by re-reading `role-index.ts` in full: nothing in it
  imports from or exports to anything a technical-context loader would
  need to share.
- Folding into `@axiom/document-bootstrap` would also not share real
  code: that package's actual responsibility is idempotent
  rendering/writing of generated files from known variables
  (`AXIOM:GENERATED`/`TEAM:CUSTOM` block preservation), not reading back
  or indexing content for agent consumption. A technical-context index
  is read-and-resolved far more than rendered — its lifecycle is closer
  to `@axiom/skills`'s catalog/role-index reading side than to
  `document-bootstrap`'s writing side, but "closer to" is not "shares
  code with," which is why the answer is still a new package, not
  folding into `@axiom/skills` either.
- A new `@axiom/technical-context` package cleanly owns exactly three
  things with no cross-import requirement in either direction: the
  `TechnicalContextIndex` type family, its loader/validator (mirroring
  `loadSkillsRoleIndex`/`validateSkillsRoleIndex`), and the pure
  `resolveMandatoryDocuments` helper. This keeps the dependency graph
  simple, consistent with how `@axiom/topology` and `@axiom/isolation`
  already exist as separate small packages for their own first-class
  concepts.

No new evidence surfaced during this design that would change the
audit's conclusion; confirming rather than re-deriving from scratch is
the correct posture here (same as INC-09's schema-writer design
confirming Q-skills-2 rather than manufacturing a reason to override).

### Q-context-2 — `whenTags` matching ownership and semantics

**Decision: the package itself exposes a pure
`resolveMandatoryDocuments(index, taskTags)` helper that performs the
matching. A `whenTags` group matches when its `tags` are a subset of
`taskTags` — i.e. ALL of the group's tags must be present in
`taskTags`, not just ANY one of them.**

**Ownership (library helper, not caller-side):**

- The audit already recommended this ("this audit recommends the
  library provide a pure resolution helper"), citing
  `getRoleIndexEntryById`'s existing pure-helper precedent in
  `role-index.ts`. This design confirms it rather than pushing
  resolution to the future `get_implementation_context` caller (INC-14),
  for the same reason `getRoleIndexEntryById` exists at all: the
  resolution logic (walking `always` + filtering `whenTags` by a set
  intersection) is a pure function of data the package itself owns the
  shape of (`TechnicalContextIndex`) — there is no reason to force every
  future caller (INC-14's tool, a `doctor` check, a CLI command, a test)
  to reimplement the same tag-matching loop. Centralizing it in the
  package that owns the type is strictly more useful and less
  error-prone than leaving it to each caller, and it costs nothing extra
  to design now since the type is already fully specified.
- This mirrors `role-index.ts`'s own established pattern precisely:
  `getRoleIndexEntryById` is a "pure, no I/O" helper co-located with the
  type it operates on, exported for reuse. `resolveMandatoryDocuments`
  is the same kind of artifact for a more complex (but still pure, still
  no I/O) resolution.

**Matching semantics (ALL, not ANY) — grounded in 10.2/10.3's own
wording, not assumed:**

- Source doc 10.2's own example is unambiguous about *why* a group's
  tags exist together: the single `whenTags` entry lists three tags —
  `persistence`, `database`, `ef-core` — under one `documents` array
  containing one document (`backend.persistence`, "Persistence task
  requires persistence context"). If this matched on ANY of the three
  tags being present, a task tagged only `database` (e.g. a schema
  documentation task with no actual persistence-layer code change) would
  trigger the same mandatory `backend.persistence` document as a task
  tagged all three — that reading makes the grouping of three tags under
  one condition meaningless, since any single one of them alone would
  already fire it. Grouping multiple tags under one condition only makes
  semantic sense if the group represents a *combined* signal — this task
  is persistence work AND touches a database AND specifically uses
  EF Core — which is exactly the ALL-of-the-tags (subset) reading.
- 10.3's own reinforcement of the "mandatory must be conservative and
  precise" intent (documents are returned "directly," not just
  suggested, when mandatory) supports requiring a stronger match
  condition (all tags present) rather than a looser one (any tag
  present) before forcing a document into the mandatory set — a looser
  ANY match would risk over-triggering mandatory documents from
  incidental single-tag overlaps, undermining the entire point of
  `mandatory` being a small, deliberate, high-confidence list (as
  opposed to `available`, which already exists specifically for the
  broader, tag-filterable, non-mandatory case).
- This also creates a clean, principled boundary between `mandatory.
  whenTags` and `available`: `available` entries already do ANY-style
  selection informally (a caller picks entries whose `tags` overlap
  what it needs, e.g. "show me anything tagged `api`"), so if
  `mandatory.whenTags` used the same ANY semantics it would functionally
  duplicate `available`'s own selection mechanism with no distinguishing
  behavior. Requiring ALL-of-group for `mandatory.whenTags` gives it a
  materially different (stricter, combined-condition) selection
  mechanism, which justifies the two lists existing as structurally
  distinct concepts rather than one being a stricter copy of the other
  in name only.

### Q-context-3 — File-path convention

**Decision: `<specScope>/technical-context/indexes/<role-or-kind>.index.yml`
— i.e. keep source doc 10.2's own literal subpath and filename
convention (`technical-context/indexes/<name>.index.yml`), resolved
against this codebase's actual `specScope` mechanism (audit Finding 2)
rather than a hardcoded, unconditional path assumption. This is
source doc 10.2's own path, NOT the `axiom.spec/config/skills-index/`
sibling convention `@axiom/skills` uses.**

Full resolved path pattern:

```
<specScope.absolutePath>/technical-context/indexes/<role-or-kind>.index.yml
```

Example: `<specScope.absolutePath>/technical-context/indexes/backend.index.yml`.

Rationale, reconciling all three inputs directly rather than picking one
without addressing the other two:

- **Why not literally follow 4.2's `axiom.yml`-declared
  `paths.technicalContext`/`indexes.technicalContext` fields as the
  resolution mechanism.** Section 4.2 shows those fields inside a
  **spec repo's own `axiom.yml` manifest** — they are a per-project,
  per-repo declaration of where *that project's* spec repo happens to
  keep its technical-context tree (`technical-context/`) and index
  subfolder (`technical-context/indexes/`), not a description of how a
  generic loader like this package's `loadTechnicalContextIndex` should
  compute a path without first reading and parsing that project's
  `axiom.yml`. Building `@axiom/technical-context` to *require* parsing
  an `axiom.yml` manifest before it can resolve anything would import a
  dependency on `axiom.yml`-manifest-reading logic this package does not
  currently have and was not asked to add (`Axiom.SDD/AGENTS.md`'s "do
  not introduce mandatory indexes" / "do not create speculative
  architecture" — inventing manifest-driven path resolution now, before
  any `axiom.yml` reader exists in this monorepo for
  `@axiom/technical-context` to call, is exactly that kind of
  speculative addition). The audit's Finding 2 already identified the
  actual, working, in-repo mechanism for "where is the spec content for
  X" — `specScope` resolution — and this design reuses it rather than
  inventing a second one.
- **Why not follow `@axiom/skills`'s `axiom.spec/config/skills-index/`
  sibling convention instead.** That convention is specific to
  `@axiom/skills`'s own artifact (source doc 9.4's flat shape) and its
  own established `axiom.spec/config/` colocation with
  `skills-catalog.yaml`. Source doc 10.2 is a **different, unrelated
  concept with its own explicit path in its own worked example**
  (`technical-context/indexes/backend.index.yml`, not nested under any
  `config/` folder) — there is no existing code, no existing convention,
  and no instruction from the audit that says 10.2's index should be
  relocated under `axiom.spec/config/` the way 9.4's was. Doing so would
  silently overwrite 10.2's own explicit path decision with 9.4's
  unrelated one, for no reason stronger than "they are both YAML
  indexes" — not a strong enough reason, given 10.2's own example
  already specifies a path and nothing in the audit's Finding 2 or brief
  asked for consolidating the two conventions.
- **Why `specScope` resolution is still the right mechanism (not a
  literal, unconditional `technical-context/indexes/` path from
  `rootPath`).** The audit's Finding 2 confirms `specScope` resolution
  (preferring a scope named `specification`, falling back to a scope
  whose relative path contains `spec`, falling back to the first
  non-product scope) is the real, 12+-times-reused, working precedent
  for "where is the spec content for X" in this monorepo — most recently
  reused by `resolveSkillsRoleIndexDir` (INC-09) for its own optional
  index directory. 10.1 states technical context "always lives in the
  SPEC repo, never in code repos" — `specScope` resolution is precisely
  the existing mechanism that already encodes "find the spec repo/scope
  among the resolved topology scopes," so reusing it here directly
  satisfies 10.1's own placement rule without inventing a new lookup.
  The alternative (a literal path computed straight from a bare
  `rootPath`, the way `@axiom/skills`'s `SKILLS_CATALOG_RELATIVE_PATH`
  does) would be wrong for this specific artifact, because 10.1
  explicitly requires the technical-context tree to live in the spec
  repo/scope, which — unlike `@axiom/skills`'s single-`rootPath`-is-
  always-enough case confirmed by INC-09's own Q-skills-3 analysis — is
  not guaranteed to be the same path as the code repo whenever a
  project's topology is not `single-repo`. `specScope` resolution
  already handles exactly this distinction (it explicitly filters out
  `isProductRuntime` scopes), so it is the correct, not merely
  convenient, mechanism.
- **Why keep 10.2's own `technical-context/indexes/<name>.index.yml`
  subpath and filename suffix verbatim, rather than renaming to match
  `@axiom/skills`'s `<role>.yaml` filename convention.** 10.2's own
  worked example already shows the exact intended subpath
  (`technical-context/indexes/`) and filename suffix
  (`backend.index.yml`, not bare `backend.yml`) — there is no
  ambiguity to resolve here beyond *where* `technical-context/` is
  rooted (resolved above via `specScope`), so this design does not
  introduce a third naming variant. The `.index.yml` suffix (rather than
  a bare `.yml`) is also a small but real disambiguator once a project's
  `technical-context/` tree contains both this index file and the
  actual content documents it points to (e.g. `backend/architecture.md`
  sitting near `indexes/backend.index.yml`) — keeping 10.2's own naming
  avoids a same-directory collision between an index and a content file
  that could otherwise both be named `backend.*`.
- **Net effect on Q-context-1's audit-stated recommendation.** The
  audit's own text ("this audit recommends following the existing
  `axiom.spec/config/` sibling convention... but flags it as open") is
  explicitly overridden here, not confirmed — the deciding factor is
  that 10.2 already has its own explicit, unambiguous path in its own
  worked example, which is a stronger signal than "consistency with an
  unrelated artifact's convention." This is a case where the audit's own
  hedge ("flags it as open pending schema-writer... confirmation") is
  resolved by schema-writer choosing the other option, with the
  reasoning made explicit here rather than defaulting to the audit's own
  tentative lean.

### Technical-context index schema design

```ts
// packages/technical-context/src/technical-context-index.ts
// (proposed location; not created by this increment — see Non-goals)

/** Schema version of a technical-context index document. Independent
 *  from SkillsRoleIndexSchemaVersion and from SkillsCatalog's own
 *  schemaVersion (three separately versioned artifacts, same posture
 *  already established between SkillsCatalog and SkillsRoleIndex by
 *  INC-09's Q-skills-2 resolution) — this artifact's shape evolves on
 *  its own timeline. */
export type TechnicalContextIndexSchemaVersion = 1;

/** One entry in `mandatory.always` or inside a `whenTags` group's
 *  `documents` array: a document reference with an id, a path, and an
 *  optional human-readable reason. Source doc 10.2 shows `reason` on
 *  every such entry in its own worked example (unlike 9.4, which never
 *  shows a `reason` field at all) — kept required-shaped-as-optional
 *  here (optional in the type, but present in every 10.2 example entry)
 *  for the same reason `role-index.ts` keeps its own additive fields
 *  optional: a minimal entry without `reason` must still be valid, even
 *  though the worked example always includes it. */
export interface TechnicalContextDocumentRef {
  /** Stable identifier for this document within the index (e.g.
   *  'backend.architecture'). Not required to reference any external
   *  catalog — unlike @axiom/skills' id-reuse convention against
   *  SkillsCatalog, technical-context documents have no equivalent
   *  external catalog to reference in this codebase today, so `id` here
   *  is a self-contained namespace scoped to this index. */
  readonly id: string;
  /** Path to the document, relative to the index file's own location
   *  (matches 10.2's own example exactly: `../backend/architecture.md`
   *  is relative to `technical-context/indexes/backend.index.yml`, i.e.
   *  it resolves to `technical-context/backend/architecture.md`). Kept
   *  required (not optional), unlike @axiom/skills' `path?`, because
   *  there is no external catalog a technical-context document could
   *  fall back to resolving `path` from — `path` is this schema's only
   *  source of truth for where the document lives. */
  readonly path: string;
  /** Human-readable reason this document matters. Optional: present on
   *  every entry in 10.2's own worked example, but not made required at
   *  the type level so a minimal `{id, path}` entry remains valid. */
  readonly reason?: string;
}

/** One `mandatory.whenTags` group: a set of tags whose combined presence
 *  (see resolveMandatoryDocuments, Q-context-2) makes `documents`
 *  mandatory for a task carrying those tags. */
export interface TechnicalContextWhenTagsGroup {
  /** Tags that must ALL be present in a task's tags for this group's
   *  `documents` to become mandatory (Q-context-2's ALL-match
   *  resolution; see rationale above — grouping only makes sense as a
   *  combined-condition signal, not an any-of-these shortcut). */
  readonly tags: ReadonlyArray<string>;
  /** Documents that become mandatory when this group's `tags` condition
   *  is satisfied. Same shape as `mandatory.always` entries — both are
   *  TechnicalContextDocumentRef, since 10.2's own example shows
   *  identical `{id, path, reason}` shape in both places. */
  readonly documents: ReadonlyArray<TechnicalContextDocumentRef>;
}

/** The full `mandatory` object: unconditional `always` entries plus
 *  conditional `whenTags` groups. Nested exactly as 10.2's own example
 *  shows it — not flattened, not restructured. */
export interface TechnicalContextMandatory {
  readonly always: ReadonlyArray<TechnicalContextDocumentRef>;
  readonly whenTags: ReadonlyArray<TechnicalContextWhenTagsGroup>;
}

/** One entry in the top-level `available` list: a document an agent MAY
 *  request on demand, selected by `tags`. Flat, one level, matching
 *  10.2's own `available` array exactly (this is NOT the same list as
 *  @axiom/skills' `available` — different domain, coincidentally
 *  similar shape). */
export interface TechnicalContextAvailableEntry {
  readonly id: string;
  readonly path: string;
  /** One-line human summary. Present on every entry in 10.2's own
   *  example; kept optional at the type level for the same
   *  minimal-valid-entry reason as `reason` above. */
  readonly summary?: string;
  /** Tags for selection. Required (not optional), mirroring
   *  SkillsRoleIndexAvailableEntry.tags' own required-field reasoning:
   *  an `available` entry with no tags has no selection mechanism at
   *  all, in either 9.4 or 10.2's design. */
  readonly tags: ReadonlyArray<string>;
}

/** Document persisted at
 *  `<specScope>/technical-context/indexes/<role-or-kind>.index.yml`
 *  (see Q-context-3 resolution above). One file per role/repo-kind
 *  combination, matching 10.2's own single-document, single-`role`-
 *  scalar-field example. */
export interface TechnicalContextIndex {
  readonly schemaVersion: TechnicalContextIndexSchemaVersion;
  /** Optional project identifier, present in 10.2's own worked example
   *  (`projectId: kvp25`). Kept optional at the type level because a
   *  single-project deployment (this workspace's own dogfooding case
   *  today) has no second project to disambiguate against — but the
   *  field is not dropped, since 10.2's example includes it and a
   *  multi-project registry (per `~/.axiom/projects.yml`, already
   *  documented elsewhere in the source doc) will need it. */
  readonly projectId?: string;
  /** Role this index applies to (e.g. 'backend-developer'), matching
   *  10.2's own field verbatim. Open string, not a closed enum — same
   *  posture as SkillsRoleIndex.role and @axiom/topology's
   *  RoleAssignment.roleId. */
  readonly role: string;
  /** Repo-kind scoping (e.g. ['backend']), matching 10.2's own field
   *  verbatim. Unlike SkillsRoleIndex.repoKinds (optional, added there
   *  because 9.4's own example never shows it), 10.2's own example DOES
   *  show `repoKinds` explicitly — so this field is kept required here,
   *  not optional, to stay faithful to what 10.2 actually demonstrates
   *  rather than importing @axiom/skills' optionality choice into a
   *  different artifact's schema by default. */
  readonly repoKinds: ReadonlyArray<string>;
  readonly mandatory: TechnicalContextMandatory;
  readonly available: ReadonlyArray<TechnicalContextAvailableEntry>;
}
```

Field-naming/shape deviations from source doc 10.2's literal example,
each justified:

| 10.2 literal field | This schema | Deviation kind | Reason |
|---|---|---|---|
| `schemaVersion` | `schemaVersion` | none | kept verbatim |
| `projectId` | `projectId?` (optional) | narrowed to optional | 10.2's example includes it, but a single-project deployment has no second project to disambiguate; field is not dropped, only made non-mandatory |
| `role` | `role` (required) | none | kept verbatim, required, matching 10.2's example exactly |
| `repoKinds` | `repoKinds` (required array) | none | kept verbatim and required, since 10.2's own example shows it explicitly (unlike SkillsRoleIndex.repoKinds, which is optional because 9.4 never shows it) |
| `mandatory.always[].{id,path,reason}` | `TechnicalContextDocumentRef{id,path,reason?}` | `reason` narrowed to optional | every 10.2 example entry includes `reason`, but a minimal `{id, path}` entry should remain valid; `path` kept required (no external catalog to fall back to, unlike @axiom/skills' `path?`) |
| `mandatory.whenTags[].{tags,documents}` | `TechnicalContextWhenTagsGroup{tags,documents}` | none | kept verbatim, nested exactly as shown |
| `mandatory.whenTags[].documents[]` | `ReadonlyArray<TechnicalContextDocumentRef>` | none | same shape as `always` entries, since 10.2's example shows identical `{id,path,reason}` structure in both places |
| `available[].{id,path,summary,tags}` | `TechnicalContextAvailableEntry{id,path,summary?,tags}` | `summary` narrowed to optional | every 10.2 example entry includes `summary`, but kept optional for the same minimal-valid-entry reason as `reason`; `tags` kept required (only selection mechanism `available` has) |
| *(absent in 10.2's YAML, but implied by 4.2/5.5)* | none added | — | no additive fields were introduced beyond what 10.2 already shows — unlike `SkillsRoleIndex`, which added `reason?`/`summary?`/`repoKinds?` because 9.4's example omitted them; 10.2's example already includes every field this design needs, so nothing new was invented |

**No id-reuse-against-an-external-catalog convention, unlike
`@axiom/skills`.** `SkillsRoleIndexMandatoryEntry.id`/`.path?` follow an
"id reuses catalog id, path is a fallback-only pointer" convention
because `@axiom/skills` already has a `SkillsCatalog` to resolve `id`s
against. No equivalent catalog exists for technical-context documents in
this codebase today (confirmed by the audit's Finding 3 — no hidden
technical-context content anywhere), so
`TechnicalContextDocumentRef.path` is kept required, not optional: it is
the only source of truth for where a document lives, with no fallback
mechanism to omit it in favor of.

### `resolveMandatoryDocuments` design

```ts
/**
 * Resolves the full set of mandatory documents for a task carrying
 * `taskTags`, given a `TechnicalContextIndex`. Pure, no I/O — mirrors
 * `getRoleIndexEntryById`'s existing pure-helper precedent in
 * `role-index.ts`.
 *
 * Returns `index.mandatory.always` in full, plus the `documents` of
 * every `whenTags` group whose `tags` are a SUBSET of `taskTags` (i.e.
 * every tag in the group must be present in `taskTags` — see
 * Q-context-2 for the ALL-vs-ANY rationale). A group with an empty
 * `tags` array is treated as never matching (an empty condition cannot
 * be meaningfully "all present"), which also protects against a
 * malformed/empty `tags` group unconditionally firing.
 *
 * Deduplicates by `id` across `always` and every matched `whenTags`
 * group's `documents` (same document `id` could in principle appear in
 * more than one whenTags group, or already be in `always`) — first
 * occurrence wins, in `always`-then-`whenTags`-declaration-order.
 *
 * Never throws.
 */
export function resolveMandatoryDocuments(
  index: TechnicalContextIndex,
  taskTags: ReadonlyArray<string>,
): ReadonlyArray<TechnicalContextDocumentRef> {
  const tagSet = new Set(taskTags);
  const seen = new Set<string>();
  const result: TechnicalContextDocumentRef[] = [];

  for (const doc of index.mandatory.always) {
    if (!seen.has(doc.id)) {
      seen.add(doc.id);
      result.push(doc);
    }
  }

  for (const group of index.mandatory.whenTags) {
    if (group.tags.length === 0) continue;
    const allPresent = group.tags.every((tag) => tagSet.has(tag));
    if (!allPresent) continue;
    for (const doc of group.documents) {
      if (!seen.has(doc.id)) {
        seen.add(doc.id);
        result.push(doc);
      }
    }
  }

  return result;
}
```

This signature and behavior directly fulfills the audit's brief item 4
("given an index and a set of task tags, returns the resolved
`mandatory` document list — `always` entries plus every `whenTags`
entry whose `tags` intersect the input tags"), refined from "intersect"
(the audit's own provisional phrasing, left open) to the more precise
"subset" (ALL-present) semantics resolved and justified under
Q-context-2 above.

### Loader and validator design

Mirrors `loadSkillsRoleIndex`/`validateSkillsRoleIndex`'s exact
structure (`role-index.ts`, lines 213-332) — same discriminated-union
result types, same never-throws contract, same file-absent-is-not-an-
error posture, same standalone-validator-for-doctor-reuse convention.

```ts
export type TechnicalContextIndexResult =
  | { readonly kind: 'ok'; readonly value: TechnicalContextIndex }
  | { readonly kind: 'absent' }
  | { readonly kind: 'malformed'; readonly error: string };

export type TechnicalContextIndexValidationResult =
  | { readonly kind: 'ok'; readonly value: TechnicalContextIndex }
  | { readonly kind: 'malformed'; readonly error: string };

/** Relative subpath, from a resolved specScope root, where
 *  technical-context indexes live (see Q-context-3). */
export const TECHNICAL_CONTEXT_INDEX_RELATIVE_DIR =
  path.join('technical-context', 'indexes');

/**
 * Resolves the absolute path of a technical-context index for
 * `roleOrKind` (a role id, e.g. 'backend-developer', or a shorter
 * repo-kind-style name, e.g. 'backend' — 10.2's own filename
 * (`backend.index.yml`) uses the shorter form, so this helper accepts
 * whatever string the caller already resolved as the file stem;
 * @axiom/technical-context does not enforce a specific naming
 * convention between role id and repo-kind beyond "matches the
 * filename stem," same posture as SkillsRoleIndex.role):
 *
 *   `<specScopeAbsolutePath>/technical-context/indexes/<roleOrKind>.index.yml`
 */
export function technicalContextIndexPath(
  specScopeAbsolutePath: string,
  roleOrKind: string,
): string {
  return path.join(
    specScopeAbsolutePath,
    TECHNICAL_CONTEXT_INDEX_RELATIVE_DIR,
    `${roleOrKind}.index.yml`,
  );
}

/**
 * Validates an already-parsed value (e.g. yaml.load's result) against
 * TechnicalContextIndex's shape. No disk I/O. Reusable by a future
 * doctor check (working id: next free TC-0xx after TC-012) that already
 * has the parsed YAML, and internally by loadTechnicalContextIndex.
 *
 * Shape rules:
 *   - `schemaVersion` must be `1`.
 *   - `role` must be a non-empty string.
 *   - `repoKinds` must be an array of strings (required, non-optional —
 *     see schema table above).
 *   - `projectId`, if present, must be a non-empty string.
 *   - `mandatory` must be an object with `always` (array) and
 *     `whenTags` (array) — both required, may be empty.
 *   - Each `always`/whenTags-group-`documents` entry requires
 *     non-empty `id` and `path`; `reason` optional if present.
 *   - Each `whenTags` group requires a `tags` array of strings
 *     (may be empty, though resolveMandatoryDocuments treats an empty
 *     group as never-matching) and a `documents` array.
 *   - `available` must be an array; each entry requires non-empty `id`,
 *     `path`, and a `tags` array of strings (may be empty); `summary`
 *     optional if present.
 *   - No duplicate `id` across `mandatory.always`, all
 *     `mandatory.whenTags[].documents`, and `available` combined — same
 *     spirit as SkillsRoleIndex's own cross-list id-uniqueness check.
 *
 * Never throws.
 */
export function validateTechnicalContextIndex(
  parsed: unknown,
): TechnicalContextIndexValidationResult {
  /* implementation deferred — design only, per Non-goals */
}

/**
 * Loads the technical-context index for `roleOrKind` from
 * `<specScopeAbsolutePath>/technical-context/indexes/<roleOrKind>.index.yml`.
 *
 *   - `{ kind: 'absent' }` if the file does not exist.
 *   - `{ kind: 'malformed', error }` if the YAML does not parse or the
 *     shape violates the contract.
 *   - `{ kind: 'ok', value }` if the index is valid.
 *
 * Never throws. Caller is responsible for resolving
 * `specScopeAbsolutePath` via the existing specScope lookup (audit
 * Finding 2) before calling this function — this function does not
 * perform specScope resolution itself, matching loadSkillsRoleIndex's
 * own posture of accepting an already-resolved root rather than
 * re-deriving it.
 */
export function loadTechnicalContextIndex(
  specScopeAbsolutePath: string,
  roleOrKind: string,
): TechnicalContextIndexResult {
  /* implementation deferred — design only, per Non-goals */
}
```

`loadTechnicalContextIndex` intentionally takes an already-resolved
`specScopeAbsolutePath`, not a bare `rootPath`, unlike
`loadSkillsRoleIndex(rootPath, role)`. This is a deliberate, justified
deviation: `@axiom/skills`'s `rootPath` is always the single product
repo root (confirmed by INC-09's own Q-skills-3 analysis, single-repo
mode today), whereas 10.1 requires technical context to live in the spec
repo/scope specifically, which is not always the same path as a code
repo's root. Pushing `specScope` resolution to the caller (reusing the
audit's Finding 2 snippet, e.g. mirroring
`resolveSkillsRoleIndexDir`'s call site) keeps
`@axiom/technical-context` itself free of any dependency on
`@axiom/doctor` or topology-resolution machinery — it only needs a path,
already resolved by whichever caller (a future `doctor` check, INC-14's
tool, a CLI command) has access to the topology/scope resolution
context.

### Worked example — `backend`

File:
`<specScopeAbsolutePath>/technical-context/indexes/backend.index.yml`

```yaml
schemaVersion: 1
projectId: kvp25
role: backend-developer
repoKinds:
  - backend

mandatory:
  always:
    - id: backend.architecture
      path: ../backend/architecture.md
      reason: Required for any backend implementation

    - id: backend.commands
      path: ../backend/commands.md
      reason: Required before build/test/migration commands

  whenTags:
    - tags:
        - persistence
        - database
        - ef-core
      documents:
        - id: backend.persistence
          path: ../backend/persistence.md
          reason: Persistence task requires persistence context

available:
  - id: backend.api-patterns
    path: ../backend/api-patterns.md
    summary: API controller, route and DTO conventions
    tags:
      - api
      - controller
      - dto

  - id: backend.validation
    path: ../backend/validation.md
    summary: Business validation patterns
    tags:
      - validation
      - rules

  - id: backend.testing
    path: ../backend/testing.md
    summary: Unit/integration testing conventions
    tags:
      - testing
      - regression

  - id: backend.examples
    path: ../backend/examples.md
    summary: Real project examples for common backend changes
    tags:
      - examples
      - patterns
```

This example is byte-for-byte identical in content to source doc 10.2's
own worked example, placed under the resolved `Q-context-3` path
(`technical-context/indexes/backend.index.yml`, relative to a resolved
`specScopeAbsolutePath` rather than the source doc's own bare
`project.spec/` prefix, which was itself just an illustrative repo-name
prefix, not a literal instruction to hardcode `project.spec/`).

Two `resolveMandatoryDocuments` calls against this example illustrate
the ALL-match semantics decided in Q-context-2:

- `resolveMandatoryDocuments(index, ['persistence'])` → returns only
  `backend.architecture` and `backend.commands` (the `always` entries).
  The `whenTags` group does NOT fire, because `database` and `ef-core`
  are not present in the task's tags — only one of the group's three
  required tags is present.
- `resolveMandatoryDocuments(index, ['persistence', 'database',
  'ef-core'])` → returns `backend.architecture`, `backend.commands`,
  AND `backend.persistence` — all three tags of the `whenTags` group are
  present, so the group fires and its `documents` are added to the
  `always` entries (union, not replacement, per the 10.3 cross-check
  above).

### What comes after this design (not designed here)

A future `doctor` check (working id: next free TC-0xx after TC-012, per
the audit's brief item 7) should verify, per index file: (a) it parses
and matches `TechnicalContextIndex`'s schema via
`validateTechnicalContextIndex`, (b) every document `id`'s `path`
resolves to a file that exists on disk relative to the index file's own
location, and (c) no duplicate `id` across `mandatory.always`, every
`mandatory.whenTags[].documents`, and `available` combined (mirroring
`validateTechnicalContextIndex`'s own uniqueness rule). This paragraph
is a forward note for whichever role picks up validator-reviewer's step
next (see "Next step recommendation"), not a check implementation.

## Validation

`No validation command was found. Performed best-effort validation by
inspecting changed files and checking consistency against the requested
behavior.`

This increment is design/spec-only (no implementation code or YAML file
changed in `Axiom/packages/*` or `Axiom/axiom.spec/*`). Best-effort
validation performed:

- Re-read source doc section 10 (10.1, 10.2, 10.3) directly from
  `axiom_decisiones_sesion_prompt_implementacion.md` this session,
  byte-for-byte — the 10.2 worked example reproduced above (both in
  "Context" and in the final "Worked example" section) was copy-checked
  against the live document, confirming every field and every value.
- Re-read section 4.2's `Ejemplo para repo SPEC` fresh this session
  (not previously quoted by the audit) to ground Q-context-3's
  reconciliation between the `axiom.yml`-manifest-declared path fields
  and 10.2's own literal file-path comment.
- Re-read section 8.3's response-envelope shape to confirm this
  increment's artifact is correctly scoped as a narrower input to a
  larger, later (INC-14) tool, not a re-implementation of that tool.
- Re-read `Axiom/packages/skills/src/role-index.ts` in full this session
  (348 lines) to confirm the exact structural/error-handling pattern
  this design's loader/validator mirrors, rather than trusting a
  secondhand description.
- Re-read `INC-20260702-skills-role-index-reconcile-design/README.md`
  in full this session as the explicit style/rigor bar this increment
  was asked to match, and followed its structure (per-question
  resolution with rationale, field-origin/deviation table, fully worked
  example, explicit next-step handoff).
- Cross-checked every field in the proposed `TechnicalContextIndex`
  shape against 10.2's own worked example line-by-line (see the
  field-table above) to confirm zero invented fields and zero dropped
  fields — the only intentional shape changes are widening required
  fields to optional where a minimal valid entry needs to exist without
  them, never removing or renaming a field 10.2 shows.
- Manually traced `resolveMandatoryDocuments` against the worked
  example with two different `taskTags` inputs (see "Worked example"
  section above) to confirm the ALL-match semantics produce the
  expected, non-degenerate result in both the matching and
  non-matching case.
- `Axiom/package.json`'s real validation commands (build/test/doctor/
  typecheck) were not run, because no code was changed in `Axiom/` for
  this increment; running them would only assert pre-existing state.

## Result

Resolved all three of the migration-engineer audit's open questions:

- **Q-context-1**: confirmed, no override — new small package,
  `@axiom/technical-context`, because the artifact shares only a
  *pattern* with `@axiom/skills`/`@axiom/document-bootstrap`, never
  executable code, and giving it its own package name keeps 10.1's
  first-class-concept intent legible in the codebase.
- **Q-context-2**: the package exposes a pure
  `resolveMandatoryDocuments(index, taskTags)` helper (library-owned,
  not caller-owned), and matching is **ALL-present (subset)**, not
  ANY-present — grounded directly in 10.2's own example (a
  three-tag group only makes sense as a combined-condition signal) and
  in 10.3's "mandatory must be conservative and precise" intent, and
  chosen specifically to keep `mandatory.whenTags` behaviorally distinct
  from `available`'s already-existing any-of-these-tags selection
  pattern.
- **Q-context-3**: `<specScope>/technical-context/indexes/<role-or-
  kind>.index.yml` — keeping 10.2's own literal subpath and filename
  suffix verbatim, resolved against the audit's Finding 2 `specScope`
  mechanism rather than a hardcoded `axiom.spec/config/` sibling
  convention or a manifest-driven `axiom.yml`-parsing requirement that
  does not exist in this codebase yet. This is an explicit override of
  the audit's own tentative lean toward the `axiom.spec/config/`
  sibling convention.

Produced the full `TechnicalContextIndex`/`TechnicalContextDocumentRef`/
`TechnicalContextWhenTagsGroup`/`TechnicalContextAvailableEntry`
TypeScript design (schema-versioned, no external-catalog id-reuse
convention since none exists for this domain, required `path` on every
document reference, ALL-match `whenTags` resolution), the
`resolveMandatoryDocuments` pure helper with worked-example traces for
both a matching and non-matching case, and the
`loadTechnicalContextIndex`/`validateTechnicalContextIndex`/
`technicalContextIndexPath` function signatures mirroring
`role-index.ts`'s exact structure — plus one fully worked `backend`
example matching source doc 10.2's own worked example byte-for-byte.

No implementation code or YAML file was written under
`Axiom/packages/*` or `Axiom/axiom.spec/*`, per this increment's
non-goals. Status is `pending`, not `closed` (see Closure rationale
below).

## General spec integration

No integration into `Axiom.Spec/general-spec.md` performed yet.
Rationale, consistent with `INC-20260702-skills-role-index-reconcile-design`'s
own closure reasoning: this increment finalizes a *design* (schema,
placement, naming, versioning, resolution semantics) that has not yet
survived contact with implementation — no loader exists yet, no real
YAML file has been authored and validated against a live `doctor` check.
Locking this into `general-spec.md` now would consolidate a design that
could still shift once implementation surfaces a mismatch. The correct
point to integrate this into `general-spec.md` is once the
`@axiom/technical-context` package actually lands with a final,
implemented shape — at that point, add a "Technical-context index"
section mirroring the existing "Skills role-index" section's structure,
explicitly cross-referencing it in both directions (the two sections
should each restate the 9.4-vs-10.2 domain distinction consistently, so
a future reader lands on the correct one regardless of which section
they read first) — the same threshold already used for INC-09's own
schema-writer step.

## Next step recommendation

**Skip docs-skills-writer for now; recommend going directly to a
combined implementer role (schema + loader + validator authored
together in `@axiom/technical-context`), with validator-reviewer's
`doctor`-check step following after.**

Rationale for not inserting docs-skills-writer next, using INC-09's own
precedent of "not inventing unneeded role-hops": docs-skills-writer's
job in INC-09 was to author 9.3's role-catalog *markdown content*
(human-readable "what/when/why" prose for skills that already existed
conceptually, e.g. `backend-developer.md`, `ef-core.md`). For
technical-context, the audit's own brief (item 6) already flags this
explicitly: "loop in a docs-skills-writer... only if/when a concrete
project needs them — do not speculatively generate content for `Axiom`
itself as part of schema design." No concrete project exists yet that
needs real `architecture.md`/`commands.md`/`persistence.md` content
written — the worked example above deliberately reuses 10.2's own
placeholder-style content (paths that do not point to real files),
exactly as 10.2 itself does. Authoring real backend architecture/
commands/persistence documents for a project that does not yet exist
would be speculative content generation this increment (and the
audit before it) explicitly declined to do. Inserting a
docs-skills-writer step now would produce either (a) more placeholder
content indistinguishable from what this design increment already
shows, or (b) content invented for a hypothetical project — neither is
useful, and both would need to be redone once a real project exists.

Instead, the more valuable next step is implementation: author
`@axiom/technical-context`'s actual TypeScript source
(`technical-context-index.ts` or equivalent), following this design
exactly — type family, `loadTechnicalContextIndex`,
`validateTechnicalContextIndex`, `resolveMandatoryDocuments`,
`technicalContextIndexPath` — the same way INC-09's own chain eventually
needed a real `role-index.ts` file regardless of docs-skills-writer's
separate markdown-content work. This is analogous to schema-writer's own
brief item 5 already stating "no cross-import required... out of scope
to enforce" for skill-id cross-referencing — the schema is
self-contained enough to implement without first waiting on markdown
content that does not exist yet.

Exact inputs the next role needs from this document:

1. **The full type family** (`TechnicalContextIndex` and its four
   nested/sibling types) and the worked `backend` example above, to
   implement as real TypeScript source plus one real (still
   placeholder-content) YAML fixture for tests.
2. **The Q-context-3 file-placement decision**
   (`<specScopeAbsolutePath>/technical-context/indexes/<role-or-
   kind>.index.yml`) and the note that `loadTechnicalContextIndex`
   accepts an already-resolved `specScopeAbsolutePath`, not a bare
   `rootPath` — the caller (test harness, future `doctor` check, INC-14)
   is responsible for `specScope` resolution via the audit's Finding 2
   snippet.
3. **The `resolveMandatoryDocuments` pure-function design** (signature,
   ALL-match semantics, dedup-by-id, empty-`tags`-group-never-matches
   edge case) to implement and unit-test against the two traced
   examples above (`['persistence']` vs. `['persistence', 'database',
   'ef-core']`).
4. **The loader/validator error-handling contract**
   (`{kind: 'ok'|'absent'|'malformed'}`, never-throws) to implement
   identically to `loadSkillsRoleIndex`/`validateSkillsRoleIndex`'s
   existing style, for consistency across both packages.

After implementation, continue with **validator-reviewer**, extending
`@axiom/doctor` with a new check (working id: next free TC-0xx after
TC-012) for the technical-context index, following TC-012's exact
skip-if-absent + reuse-validator convention — per the "What comes after
this design" forward note above and the audit's brief item 7. Deferred
entirely: 10.3's mandatory-document-direct-return rule
(`get_implementation_context` must return `architecture.md` +
`commands.md` directly when `mandatory.always`), which remains INC-14 —
this design and its eventual implementation are necessary preparation
for that rule, not an implementation of it.
