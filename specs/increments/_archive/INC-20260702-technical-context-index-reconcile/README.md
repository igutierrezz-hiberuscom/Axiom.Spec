# Increment: Reconcile technical context structure — audit

Status: closed
Date: 2026-07-02

Closed as part of INC-10's overall closure by
`INC-20260702-technical-context-index-reconcile-validator/README.md`
(validator-reviewer, final step of the chain). See that spec's "INC-10
closure assessment" section for the full closure rationale covering all
four increments in this chain.

## Goal

Audit the `Axiom` monorepo against the two external decision documents'
technical-context model (source doc section 10, addendum reinforcements)
and produce a confirmed, code-grounded diff: does any per-role,
per-repo-kind curated document index matching 10.2's nested
`mandatory.always`/`mandatory.whenTags`/`available` shape exist today,
under any name? This increment is **audit-only**: no code changes.

This is INC-10 of the parent roadmap
(`Axiom.Spec/specs/increments/INC-20260702-axiom-redesign-roadmap/README.md`,
Phase D). It follows INC-01 through INC-09 (all closed) and reads
`Axiom.Spec/general-spec.md` first, including the "Skills role-index"
section, which already flags the exact domain distinction this increment
must respect: 9.4 (skills, flat `mandatory`/`available`) is a different
domain from 10.2 (technical-context, nested
`mandatory.always`/`mandatory.whenTags`/`available`). INC-09 built the
former; this increment audits whether the latter exists.

## Context

Roadmap's pre-audit hypothesis for INC-10 (roadmap README, Phase D): no
dedicated `technical-context/` package was identified among the 26
audited packages; likely represented today via `axiom.spec/` docs and the
`skills`/`document-bootstrap` packages rather than a first-class indexed
structure. Needs explicit audit before scoping.

Source documents re-read directly for this audit:

- `axiom_decisiones_sesion_prompt_implementacion.md` section 10 (Contexto
  técnico): 10.1 canonical technical context always lives in the spec
  repo, never in code repos. 10.2's full example
  (`technical-context/indexes/backend.index.yml`): `schemaVersion: 1`,
  `projectId`, `role`, `repoKinds: [...]`, then a **nested**
  `mandatory: { always: [{id, path, reason}], whenTags: [{tags: [...],
  documents: [{id, path, reason}]}] }`, and a flat `available: [{id, path,
  summary, tags}]`. This nested `mandatory.always`/`mandatory.whenTags`
  split (conditional-by-tag mandatory documents, not just an
  unconditional flat list) is the structural feature that has no analog
  anywhere in the currently-shipped `SkillsRoleIndex` (INC-09's flat
  `mandatory: [...]`). 10.3: if a role's index declares `architecture.md`
  + `commands.md` as `mandatory.always`, `get_implementation_context`
  must return both documents directly, in addition to the full index —
  i.e. mandatory-document resolution is itself a rule the future
  MCP-tool-implementer (INC-14) will need, but is out of scope to build
  here.
- `axiom_decisiones_sesion_addendum_revision.md`: section 6 (ADR index
  derivado o curado) reinforces the general "if the index decides agent
  behavior, it must be versioned" rule (source doc 5.5) that also governs
  the technical-context index — consistent with 10.2 already showing a
  versioned, curated shape, not a generated cache. Section 15 (bootstrap
  documental en draft/review) is relevant to a later increment
  (INC-18/INC-19, bootstrap-from-code / bootstrap-from-legacy-SDD), not to
  this audit. No other addendum section adds new technical-context
  structural content beyond these two reinforcements.

## Scope

- Grep/search the whole `Axiom` monorepo for any existing technical
  context concept: package names, folder names (`technical-context`,
  `context/`), function/type names (`technicalContext`, `TechContext`,
  `TechnicalContext`), and CLI commands.
- Read `Axiom/apps/cli/src/commands/context.ts` in full and confirm what
  `axiom context refresh`/`status` actually operate on today.
- Read `@axiom/doctor`'s `specScope` resolution logic
  (`Axiom/packages/doctor/src/checks.ts`) to understand the codebase's
  current "where is the spec content for X" pattern, and confirm whether
  it is reusable for a future technical-context index resolver.
- Check `@axiom/skills`'s role-index (INC-09,
  `Axiom/packages/skills/src/role-index.ts`) and `@axiom/document-bootstrap`
  for any content that is actually technical-context in nature (real
  architecture docs, patterns, examples) hiding under a different name.
- Determine the most natural package placement for a future
  technical-context index artifact: `@axiom/skills` (adjacent, similar
  shape to the role-index just built), `@axiom/document-bootstrap`
  (adjacent to content generation), or a new small package (e.g.
  `@axiom/technical-context`).
- Write this spec file with goal/scope/non-goals/acceptance criteria, the
  "nothing exists today" finding, the package-placement recommendation,
  and a brief for the next subagent (schema-writer).

## Non-goals

- No code implementation. No new schema, YAML file, or TypeScript type is
  written in this increment.
- No decision on the exact final field names of the technical-context
  index schema — that is schema-writer's job in a follow-up increment,
  using this audit's diff as the starting brief.
- No change to `@axiom/skills`' existing `SkillsCatalog` or
  `SkillsRoleIndex` shapes, or to `@axiom/document-bootstrap`'s existing
  renderers/writer.
- No implementation of `get_implementation_context` (source doc 10.3's
  mandatory-document-resolution rule) — that is INC-14 in the roadmap, a
  different increment, dependent on this one's index existing first.
- No re-litigation of INC-09's skills-index scope; this increment treats
  9.4 vs 10.2 as settled (per `general-spec.md`'s "Skills role-index"
  section) and does not revisit that distinction beyond restating it for
  self-containment.

## Acceptance criteria

- [x] The whole `Axiom` monorepo was searched (grep, not assumption) for
      any existing technical-context concept under any name.
- [x] `Axiom/apps/cli/src/commands/context.ts` was read in full and its
      actual behavior (not the roadmap's or a prior pass's secondhand
      description) is confirmed and documented.
- [x] `@axiom/doctor`'s `specScope` resolution logic was read directly
      and its reuse potential for a technical-context index resolver is
      assessed.
- [x] `@axiom/skills`'s role-index and `@axiom/document-bootstrap` were
      checked for hidden technical-context content; result documented.
- [x] A clear finding is produced: does any per-role, per-repo-kind
      curated document index matching 10.2's nested shape exist today,
      under any name.
- [x] A package-placement recommendation is made, grounded in actual code
      cohesion (imports, shared helpers, physical file layout), not
      speculation.
- [x] A brief for schema-writer is included, ready to hand off.
- [x] Recorded in `Axiom.Spec`; no code changed in `Axiom.SDD` or
      `Axiom`.

## Open questions

- **Q-context-1**: Should the technical-context index (10.2) live inside
  `@axiom/skills` as a structurally parallel sibling module to
  `role-index.ts` (e.g. `technical-context-index.ts`, reusing the same
  `axiom.spec/config/` area), or as an independent new package (e.g.
  `@axiom/technical-context`)? Not blocking this audit; blocking for
  schema-writer's package-placement decision. This audit's recommendation
  (see "Package placement" below) is a new small package, but the
  alternative is viable and cheap to reverse since both options reuse the
  same `specScope`-resolution pattern either way.
- **Q-context-2**: Source doc 10.2's example nests `mandatory` into
  `always`/`whenTags`, where `whenTags` entries carry their own
  `documents: [...]` array conditioned on a `tags` match. Should
  `whenTags` matching be evaluated by the index-loading library itself
  (i.e. `getTechnicalContextEntries(index, {tags})`-style resolution
  function), or is tag-matching entirely `get_implementation_context`'s
  (INC-14's) responsibility, with this increment's artifact only
  providing the raw parsed shape? This audit recommends the library
  provide a pure resolution helper (mirroring
  `getRoleIndexEntryById`'s existing pure-helper precedent in
  `role-index.ts`), but leaves the exact function signature to
  schema-writer.
- **Q-context-3**: Source doc 4.2's example places
  `paths.technicalContext: technical-context/` and
  `indexes.technicalContext: technical-context/indexes/` inside a repo's
  own `axiom.yml`. The existing codebase's convention (per INC-09) is
  `<rootPath>/axiom.spec/config/skills-index/<role>.yaml` — i.e.
  `axiom.spec/config/`, not a top-level `technical-context/` folder.
  Should the new index follow the existing `axiom.spec/config/` sibling
  convention (consistent with `skills-catalog.yaml` and
  `skills-index/`), or introduce the source doc's own top-level
  `technical-context/indexes/` path? This audit recommends following the
  existing `axiom.spec/config/` sibling convention for consistency with
  every other config-scoped artifact `specScope` resolves today, but
  flags it as open pending schema-writer/validator-reviewer confirmation.

## Assumptions

- "Technical context" in this increment means source doc section 10's
  concept specifically: curated, versioned, per-role/per-repo-kind
  document indexes with conditional mandatory-document rules — not any
  other use of the phrase "context" in the codebase (e.g. `axiom
  context refresh/status`'s install-profile/capability context, which is
  a materially different, unrelated domain — confirmed below).
- Following the parent roadmap's stated dependencies (INC-06, INC-08),
  this increment assumes both are stable; it did not re-verify their
  closure status as part of its own scope (out of scope; both are
  separate closed increments per the roadmap listing).
- The existing `@axiom/skills` role-index (INC-09) is the closest
  in-repo precedent for how a future technical-context index should be
  designed, loaded, and validated — same defensive
  never-throws/discriminated-union pattern, same reuse-by-`doctor`
  convention (TC-012-style, not Zod).

## Implementation notes

Audit-only. No files were changed under `Axiom/packages/`, `Axiom/apps/`,
or `Axiom.SDD/` for this increment.

### Files/areas read for this audit

- `Axiom/apps/cli/src/commands/context.ts` (full file, 376 lines).
- `Axiom/packages/doctor/src/checks.ts`: every `specScope` resolution
  site (12+ call sites, e.g. `PC-001`, `CC-001`, `IP-001`, `GW-001`,
  `readTopologyManifest`, `readOptionalStructuralCapabilities`,
  `resolveSkillsRoleIndexDir`, `TC-012`'s `runSkillsRoleIndexCheck`).
- `Axiom/packages/skills/src/role-index.ts` (full file, 348 lines).
- `Axiom/packages/document-bootstrap/src/canonical-agents-md.ts` (first
  100 lines, including the verbatim source-doc §18.1/§18.2 boilerplate
  blocks) and `variables.ts` (grepped for architecture/pattern/example
  content).
- `Axiom/packages/topology/src/types.ts` (`RepoRef`, `RoleAssignment` —
  confirmed `roleId` is an open string, not a closed enum, consistent
  with `SkillsRoleIndex.repoKinds`' own open-string design choice).
- Full-repo grep for `technical.?context|technicalContext|TechContext|
  TechnicalContext` (case-insensitive) and a filesystem search for any
  `*technical-context*` file or directory anywhere under `Axiom/`
  (excluding `node_modules`).
- `Axiom.Spec/specs/increments/INC-20260702-skills-role-index-reconcile/README.md`
  (INC-09's own audit spec) for the settled 9.4-vs-10.2 domain
  distinction and the sub-agent-chain precedent to follow.

### Finding 1 — `axiom context refresh`/`status` is NOT technical-context

Confirmed by direct read of `Axiom/apps/cli/src/commands/context.ts`
(376 lines, in full): this command is a thin wrapper around
`installProfile` (`@axiom/installer`) and reads/writes
`.sdd/<project>/install-profile.json`. `refresh` re-derives a
`ResolvedInstallProfile` (`functionalProfile`, `operationalOverlay`,
`adapterTarget`, `enabledCapabilities`, `overlay`,
`degradedModeProvider`, `gatewayExpectation`, `generatedFiles`,
`externalDependencies`) from `init.json` and persists it atomically.
`status` prints that same profile's current state. None of this reads or
writes anything resembling a document index, a per-role mandatory-docs
list, or anything under a `technical-context/` path. The command's own
description string even says "Gestiona el contexto técnico del proyecto
Axiom activo (install profile, capabilities, overlay)" — it uses the
Spanish phrase "contexto técnico," but the concept it actually implements
is capability/profile state, not source doc section 10's document-index
concept. This is a naming coincidence, not a code overlap: an earlier
roadmap pass's citation of `context.ts` as a "precedent pattern" for
INC-10 is corrected here — it is not a working precedent for anything in
scope for this increment; it is a different domain entirely (matching
this workspace's own already-documented pattern of INC-09 correcting a
similar cross-domain citation error in the roadmap for 9.4 vs 10.2).

### Finding 2 — `specScope` resolution is a real, reusable precedent

Confirmed by direct read of `Axiom/packages/doctor/src/checks.ts`: the
`specScope` lookup

```ts
const specScope =
  Object.values(resolution.scopes).find(
    (s) => !s.isProductRuntime && s.name === 'specification'
  ) ??
  Object.values(resolution.scopes).find(
    (s) => !s.isProductRuntime && s.relativePath.includes('spec')
  ) ??
  Object.values(resolution.scopes).find((s) => !s.isProductRuntime);
```

is repeated verbatim at 12+ call sites in `checks.ts` (policy checks,
capability-model checks, install-profile checks, gateway checks, topology
manifest reading, toolchain reading, integrations reading, MCP manifest
reading, the skills catalog check, and `resolveSkillsRoleIndexDir` for
TC-012). It resolves "where is the spec content for X" by preferring a
scope literally named `specification`, falling back to a scope whose
relative path contains `spec`, falling back to the first non-product
scope. This is genuinely the correct, working precedent to reuse for
resolving where a technical-context index directory lives (e.g.
`<specScope.absolutePath>/config/technical-context-index/<role>.yaml`,
mirroring `resolveSkillsRoleIndexDir`'s exact pattern) — not a new
resolution mechanism to invent. `resolveSkillsRoleIndexDir` is the
closest direct template, being the newest (INC-09) sibling artifact using
this same lookup for its own optional-index directory.

### Finding 3 — no hidden technical-context content in `@axiom/skills` or `@axiom/document-bootstrap`

- `@axiom/skills`'s `role-index.ts` (INC-09) is confirmed **flat**:
  `{ schemaVersion: 1, role, repoKinds?, mandatory: [{id, path?,
  reason?}], available: [{id, path?, tags, summary?}] }` — no nested
  `always`/`whenTags` split anywhere in the type, the parser, or the
  validator. This is source doc 9.4's shape, not 10.2's, exactly as
  `general-spec.md`'s "Skills role-index" section already states. It
  contains no architecture/pattern/example document content of its own —
  it is a pointer index, not a content repository.
- `@axiom/document-bootstrap`'s `canonical-agents-md.ts` renders two
  verbatim boilerplate blocks lifted directly from source doc §18.1/§18.2
  into every generated `AGENTS.md`: one of them explicitly lists "indexed
  technical context documents" among things that must not be manually
  created/renamed/moved, and the other states "Canonical technical
  context never lives in code repositories." These are **prose rule
  statements rendered as static text**, not an implementation of an
  index, a schema, or a loader — grepping confirmed no corresponding
  type, loader, or validator exists anywhere near this file.
  `variables.ts` (the `ResolvedVariables` placeholder set consumed by the
  separate copilot-instructions renderer) has zero architecture/pattern/
  example content. No hidden or partially-built technical-context feature
  exists under either package.
- A test-fixture-only capability id literal, `technical-context-extended`,
  appears once in `Axiom/packages/doctor/tests/qa-lane.test.ts`, alongside
  an equally arbitrary `general-context-root` id, both used purely to
  exercise `readOptionalStructuralCapabilities`' generic array-parsing
  logic for TC-003 (`qa-lane-coherence`). Confirmed by reading
  `runQaLaneCoherenceCheck`/`readOptionalStructuralCapabilities` in full:
  only the literal string `e2e-parallel-lane` is ever checked for; the
  other two ids in the fixture are inert decoys with no associated
  behavior anywhere in `checks.ts`. This is not a real
  technical-context feature or capability — it is an unrelated fixture
  string that happens to contain the substring "technical-context."

### Finding 4 — no `technical-context` file, folder, or package exists anywhere

A full-repo grep for `technical.?context|technicalContext|TechContext|
TechnicalContext` (case-insensitive) across `Axiom/` returned exactly
four files, all already covered by Findings 1-3 above
(`canonical-agents-md.ts` + its test, `qa-lane.test.ts`,
`memory/src/types.ts`'s comment referencing "the technical context" as a
conceptual layer distinct from memory — itself just a comment, no code).
A separate filesystem search for any file or directory literally named
`*technical-context*` anywhere under `Axiom/` (excluding
`node_modules`) returned zero results. This confirms the roadmap's
hypothesis without qualification: **no dedicated `technical-context/`
package, folder, or first-class indexed structure exists today, under
any name.**

### Package placement recommendation

**New small package: `@axiom/technical-context`.**

Reasoning, grounded in actual code cohesion rather than speculation:

- Placing it inside `@axiom/skills` would work mechanically (both are
  small, config-scoped, `specScope`-resolved YAML index loaders with the
  same defensive validation style), but `@axiom/skills`'s existing name,
  its `SkillsCatalog`/`SkillRegistry`/`SkillsRoleIndex` exports, and its
  `apply.ts`/`materialize.ts`/`refresh.ts` drift-detection machinery are
  all specifically about *skills* (installable, materializable,
  drift-checked units). A technical-context document index is a
  different kind of artifact — static markdown documents plus a curated
  pointer index over them, with no install/materialize/drift lifecycle
  at all. Folding it into `@axiom/skills` would blur that package's
  single, currently-clean responsibility rather than share real code:
  the only thing actually shared is the *pattern* (flat/nested
  discriminated-union YAML loader, `specScope`-resolved path,
  `validate*`-reusable-by-doctor convention), not any executable code
  path, type, or constant.
- Placing it inside `@axiom/document-bootstrap` would also work
  mechanically (both eventually touch generated project documentation),
  but that package's actual responsibility today is narrowly
  *rendering/writing* files from already-known variables (idempotent
  `AXIOM:GENERATED`/`TEAM:CUSTOM` block preservation, atomic writes) — it
  does not *read back* or *index* content for consumption by an agent.
  A technical-context index is consumed (read, resolved, filtered by
  tags) far more than it is rendered; its natural lifecycle is closer to
  `@axiom/skills`' catalog/role-index reading side than to
  `document-bootstrap`'s writing side.
- A new `@axiom/technical-context` package cleanly owns: the index type
  (`TechnicalContextIndex`, mirroring `SkillsRoleIndex`'s nested
  `mandatory.always`/`mandatory.whenTags`/`available` shape from 10.2),
  its loader/validator (same never-throws, discriminated-union pattern
  as `loadSkillsRoleIndex`/`validateSkillsRoleIndex`), and a pure
  resolution helper for tag-based `whenTags` matching (Q-context-2). It
  can freely reuse the `specScope`-resolution snippet (Finding 2) without
  needing to import anything from `@axiom/skills` or
  `@axiom/document-bootstrap`, keeping the dependency graph simple and
  consistent with how small, single-purpose packages already exist in
  this monorepo (e.g. `@axiom/write-scope`-style precedent from INC-08).
- This also directly serves 10.1's structural intent: canonical technical
  context is a first-class concept, not a byproduct of skills tooling or
  document rendering — giving it its own package name makes that
  intent legible in the codebase, consistent with how `@axiom/topology`
  and `@axiom/isolation` are already separate small packages for their
  own first-class concepts rather than folded into a nearby package for
  convenience.

### Brief for schema-writer (next subagent)

1. Design a new package `@axiom/technical-context` (or confirm final
   naming) with a `TechnicalContextIndex` type matching source doc 10.2's
   nested shape:
   `{ schemaVersion: 1, projectId?, role: string, repoKinds?: string[],
   mandatory: { always: Array<{id, path, reason?}>, whenTags:
   Array<{tags: string[], documents: Array<{id, path, reason?}>}> },
   available: Array<{id, path, summary?, tags: string[]}> }`.
2. Follow `@axiom/skills`'s `role-index.ts` defensive pattern exactly:
   never-throws loader (`{kind: 'ok'|'absent'|'malformed'}`), a pure
   `validateTechnicalContextIndex(parsed: unknown)` reusable by a future
   `doctor` check without re-reading the file, hand-written narrowing
   validation (no Zod), one file per role at
   `<specScope>/config/technical-context-index/<role>.yaml` pending
   Q-context-3's resolution (default recommendation: follow this path
   under `axiom.spec/config/`, sibling to `skills-index/`, not source
   doc's own literal `technical-context/indexes/` path).
3. Reuse the `specScope`-resolution snippet from Finding 2 verbatim
   (mirroring `resolveSkillsRoleIndexDir`'s exact pattern) rather than
   inventing a new lookup.
4. Provide a pure resolution helper (naming and exact signature is
   schema-writer's call, per Q-context-2) that, given an index and a set
   of task tags, returns the resolved `mandatory` document list
   (`always` entries plus every `whenTags` entry whose `tags` intersect
   the input tags) — this is preparatory groundwork for INC-14's
   `get_implementation_context`, not `get_implementation_context` itself.
5. Do not modify `@axiom/skills`' `SkillsCatalog`, `SkillsRoleIndex`, or
   any of its loaders — this is a structurally independent artifact, no
   shared types, no cross-import required in either direction beyond
   optionally referencing skill ids by convention (out of scope to
   enforce, same posture INC-09 took for its own catalog-id
   cross-reference).
6. Loop in a docs-skills-writer next (per the roadmap's stated sequence)
   to draft example technical-context documents
   (`architecture.md`/`commands.md`-style, source doc 10.1's
   `backend/frontend/cross-repo` folder layout) only if/when a concrete
   project needs them — do not speculatively generate content for
   `Axiom` itself as part of schema design.
7. Loop in a validator-reviewer afterward to extend `@axiom/doctor`'s
   `CATEGORY_TOPOLOGY` or a new/existing skills-adjacent category with a
   new check (working id `TC-013` or next free id) for the
   technical-context index, following TC-012's exact skip-if-absent +
   reuse-validator convention — do not invent a new check-result shape.
8. Defer 10.3's mandatory-document-direct-return rule
   (`get_implementation_context` must return `architecture.md` +
   `commands.md` directly when they are `mandatory.always`) entirely to
   INC-14 (mcp-tool-implementer) — this increment's index and resolution
   helper are necessary preparation for that rule, not an implementation
   of it.

## Validation

`No validation command was found. Performed best-effort validation by
inspecting changed files and checking consistency against the requested
behavior.`

No code was changed by this increment, so there is nothing to build/test/
lint. Best-effort validation performed:

- Directly read `Axiom/apps/cli/src/commands/context.ts` in full (376
  lines) rather than trusting an earlier pass's "precedent pattern"
  description, and found it describes a materially different domain
  (install-profile/capability state, not a document index).
- Directly read every `specScope` resolution call site in
  `Axiom/packages/doctor/src/checks.ts` to confirm the pattern's shape
  and its 12+ real usages, rather than assuming a single example
  generalizes.
- Directly read `Axiom/packages/skills/src/role-index.ts` in full (348
  lines) to confirm its shape is genuinely flat (9.4), not nested
  (10.2), consistent with `general-spec.md`'s existing statement.
- Grepped the full `Axiom/` tree (case-insensitive) for every
  technical-context-related term and cross-checked all four resulting
  file hits individually, plus a separate filesystem search for any
  `*technical-context*`-named file or directory, confirming zero
  first-class structure exists.
- Read `Axiom/packages/topology/src/types.ts`'s `RoleAssignment` to
  confirm `roleId` is an open string (not a closed enum), which grounds
  the recommendation that a new `TechnicalContextIndex.role`/`repoKinds`
  should follow the same open-string convention `SkillsRoleIndex` already
  uses, rather than introducing a new closed-enum constraint.
- Cross-checked this audit's findings against INC-09's own closed audit
  spec (`INC-20260702-skills-role-index-reconcile/README.md`) to ensure
  the 9.4-vs-10.2 domain distinction is restated consistently, not
  re-litigated or contradicted.

## Result

Confirmed the roadmap's hypothesis without qualification: no dedicated
`technical-context/` package, folder, function, or CLI command exists
anywhere in the `Axiom` monorepo today, under any name. `axiom context
refresh`/`status` (an earlier pass's cited "precedent pattern") is
confirmed, by direct full-file read, to be an unrelated domain
(install-profile/capability state), not technical-context-adjacent code —
this corrects that earlier characterization. `@axiom/doctor`'s
`specScope` resolution logic is confirmed to be the real, directly
reusable precedent for "where is the spec content for X," already used
12+ times including by INC-09's own `resolveSkillsRoleIndexDir`, which is
the closest structural template for a future technical-context index
resolver. `@axiom/skills`'s role-index and `@axiom/document-bootstrap`
were checked directly and contain no hidden technical-context content:
the former is confirmed flat (9.4's shape), the latter only renders
static prose boilerplate mentioning the phrase "technical context," with
no backing schema or loader. This work is genuinely additive. Recommended
package placement: a new small package, `@axiom/technical-context`,
rather than folding into `@axiom/skills` or `@axiom/document-bootstrap`,
because the artifact's actual lifecycle (curated document index, read/
resolved/tag-filtered, no install/materialize/drift concept, no
render/write concept) does not share real code with either existing
package — only a reusable *pattern*, not a dependency.

## General spec integration

No integration into `Axiom.Spec/general-spec.md` performed yet.
Rationale: this increment is an audit with no shipped schema or code —
per `Axiom.SDD/AGENTS.md`'s documentation rules, `general-spec.md` should
hold stable, consolidated, cross-increment knowledge, and the
technical-context index shape/placement is not yet decided (Q-context-1
through Q-context-3 remain open, and no package/type has been created).
Once schema-writer's `@axiom/technical-context` package actually lands
with a final shape, that is the right point to add a "Technical-context
index" section to `general-spec.md`, mirroring the existing "Skills
role-index" section's structure and explicitly cross-referencing it (the
two sections should state the 9.4-vs-10.2 distinction consistently in
both directions, so a future reader lands on the correct one regardless
of which section they read first). Flagging this explicitly now avoids
the same ambiguity INC-09 had to correct after the fact.

## Next step recommendation

Hand off to **schema-writer** using the brief above, scoped narrowly to
designing the new `@axiom/technical-context` package's index type,
loader, validator, and resolution helper (not touching
`@axiom/skills` or `@axiom/document-bootstrap`). Before schema-writer
starts, resolve Q-context-3 (file placement: `axiom.spec/config/`
sibling convention vs. source doc's literal `technical-context/indexes/`
path) since it affects the schema's path constants; Q-context-1
(package-vs-sibling-module) is effectively pre-resolved by this audit's
recommendation but can be revisited by schema-writer if a concrete
cohesion concern surfaces during design; Q-context-2 (where `whenTags`
matching logic lives) can be resolved during schema design without
blocking its start. After schema-writer, continue the roadmap's stated
subagent sequence for INC-10: docs-skills-writer (only if/when a concrete
project needs example technical-context documents) -> cli-implementer
(only if a CLI surface is actually needed beyond what `doctor`/a future
MCP tool require — not assumed necessary by this audit) ->
validator-reviewer (extend `@axiom/doctor` with a new check following
TC-012's exact convention).
