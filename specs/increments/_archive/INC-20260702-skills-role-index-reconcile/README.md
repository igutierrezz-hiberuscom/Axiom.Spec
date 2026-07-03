# Increment: Reconcile SDD/role skills (`@axiom/skills` catalog) — audit

Status: closed
Date: 2026-07-02

Closed as part of INC-09's overall closure. See
`INC-20260702-skills-role-index-reconcile-validator/README.md` ("INC-09
closure assessment") for the chain-wide closure rationale and deferred
items.

## Goal

Audit `@axiom/skills` (the whole package, not just the roadmap's
hypothesis) against the two external decision documents' skills model
(source doc section 9, addendum section 2's SDD-vs-technical split) and
produce a confirmed, code-grounded diff, so a later schema-writer can add
a per-role skills index layer without guessing at the existing contract.
This increment is **audit-only**: no code changes.

This is INC-09 of the parent roadmap
(`Axiom.Spec/specs/increments/INC-20260702-axiom-redesign-roadmap/README.md`,
Phase D). It follows INC-01 through INC-08 (all closed) and reads
`Axiom.Spec/general-spec.md` for cross-increment stable facts before
diving into package detail.

## Context

Roadmap's pre-audit hypothesis for INC-09 (roadmap README, Phase D):
`@axiom/skills` already has a catalog
(`axiom.spec/config/skills-catalog.yaml`, `schemaVersion: 1`, flat list of
entries with `id/name/version/source/status/securityCheckStatus/
bundleHash`), plus `apply.ts`/`materialize.ts`/`refresh.ts` (drift
detection). Diff hypothesis: existing catalog is flat, not split into
SDD-repo skills vs per-repo technical skills vs a curated per-role index
with `mandatory.always`/`mandatory.whenTags`/`available` (source doc 5.5,
9.4, 10.2 — note 10.2 is the technical-context domain, not skills; the
roadmap's own citation conflated two different structures across two
different domains).

Source documents re-read directly for this audit:

- `axiom_decisiones_sesion_prompt_implementacion.md` section 9 (Skills):
  9.1 SDD skills live in the `sdd` repo; 9.2 repo-specific technical
  skills live in each code repo; 9.3 one markdown catalog per role
  (human-readable "what/when/why" doc); 9.4 one **flat** per-role skills
  index YAML with a `mandatory: [...]` list and an `available: [{id, path,
  tags}]` list — no `whenTags`/nested structure at this section. Section
  5.5 (Índices versionados): `skills.index` is named explicitly as one of
  the indexes that "decide qué debe leer/hacer un agente" and therefore
  **must be versioned**.
- Section 10 (Contexto técnico), read for contrast only: 10.2's example
  (`technical-context/indexes/backend.index.yml`) is the one that has
  `mandatory.always` / `mandatory.whenTags` / `available` with `reason`/
  `summary`/`tags` fields — this is the **technical-context** domain's
  index shape, a different artifact from the skills index in 9.4. The
  roadmap's hypothesis incorrectly attributed 10.2's nested
  `mandatory.always`/`whenTags` shape to the skills domain. This
  increment's diff below uses 9.4's actual flat `mandatory`/`available`
  shape as the skills-index target, not 10.2's.
- `axiom_decisiones_sesion_addendum_revision.md`: reinforces the SDD-vs-
  technical-skills split (already in section 9.1/9.2, listed among the
  items the addendum says the main doc "already covers correctly") and
  the dogfooding boundary (section 2: `axiom/` product must not embed the
  SDD/spec factory used to build it). No new skills-index structural
  content is added by the addendum beyond this reinforcement.

## Scope

- Read every file in `Axiom/packages/skills/src/` (`types.ts`,
  `catalog.ts`, `loader.ts`, `apply.ts`, `materialize.ts`, `refresh.ts`,
  `index.ts`) and `Axiom/packages/skills/tests/*.test.ts`.
- Confirm the catalog's actual `schemaVersion`/fields and the physical
  file path against live code and tests (not the roadmap's paraphrase).
- Produce a field-by-field / structure-by-structure diff between the
  existing catalog and source doc 9.4's exact role-index shape
  (`mandatory: [...]`, `available: [{id, path, tags}]`).
- Check whether any per-role or per-repo-kind grouping concept already
  exists anywhere in `@axiom/skills`, `@axiom/agents`, `@axiom/doctor`, or
  `@axiom/config-validation`.
- Check whether `docs-skills-writer`-style role-catalog markdown (source
  doc 9.3's "Backend Developer Skill Catalog" example) already exists
  anywhere in the repo.
- Determine whether adding the role-index layer is additive (new optional
  file/section, or a `schemaVersion` bump on a *new* artifact) versus a
  restructuring of the existing flat catalog.
- Write this spec file with goal/scope/non-goals/acceptance criteria, the
  confirmed diff, and a brief for the next subagent (schema-writer).

## Non-goals

- No code implementation. No new schema, YAML file, or TypeScript type is
  written in this increment.
- No decision on the exact final field names of the role-index schema —
  that is schema-writer's job in a follow-up increment, using this
  audit's diff as the starting brief.
- No change to the existing `SkillsCatalog`/`CatalogEntry` shape,
  `apply.ts`/`materialize.ts`/`refresh.ts` behavior, or the `axiom
  skills` CLI command.
- No re-litigation of `@axiom/agents`' separate `role`-per-entry catalog
  (a different, already-shipped pattern — flat catalog where each entry
  declares one role, closed-enum `VALID_ROLES`) — noted for contrast only,
  not in scope to change.
- No implementation of source doc 10's technical-context per-role index
  (`mandatory.always`/`whenTags`) — that is INC-10 in the roadmap, a
  different increment for a different domain.

## Acceptance criteria

- [x] Every file in `Axiom/packages/skills/src/` and `tests/` was read
      directly (not inferred from the package name or the roadmap's prior
      paraphrase).
- [x] The catalog's actual `schemaVersion` (1), physical path
      (`axiom.spec/config/skills-catalog.yaml`, relative to a project's
      `rootPath` — confirmed by `apply.ts`'s
      `SKILLS_CATALOG_RELATIVE_PATH` constant, `doctor`'s TC-010 check,
      and the catalog test's integration case), and exact field set
      (`id/name/version/source/status/securityCheckStatus/bundleHash`)
      are confirmed against live code, not assumed.
- [x] A field-by-field diff against source doc 9.4's role-index shape is
      produced (see below), using 9.4's actual flat shape, not 10.2's
      nested `mandatory.always`/`whenTags` shape.
- [x] Confirmed whether any per-role/per-repo-kind grouping already
      exists in the codebase (none was found in `@axiom/skills`; a
      related-but-different `role`-per-entry pattern exists in
      `@axiom/agents`, noted for contrast).
- [x] Confirmed whether a role-catalog markdown (source doc 9.3 style)
      exists anywhere in the repo (none was found).
- [x] Additive-vs-restructuring determination made and justified (see
      "Additive or restructuring" below).
- [x] A brief for schema-writer is included, ready to hand off.
- [x] Recorded in `Axiom.Spec`; no code changed in `Axiom.SDD` or
      `Axiom`.

## Open questions

- **Q-skills-1**: Should the per-role skills index (source doc 9.4) live
  physically alongside the existing catalog
  (`axiom.spec/config/skills-catalog.yaml`) as sibling files (e.g.
  `axiom.spec/config/skills-index/<role>.yaml`), or inside a different
  directory the new docs imply (e.g. under `sdd`/role-scoped skills
  folders per 9.1/9.2's SDD-repo-vs-code-repo split)? Not blocking this
  audit; blocking for schema-writer's file-placement decision.
- **Q-skills-2**: Per section 5.5, `skills.index` must be versioned
  because it "decides what an agent must read." Does this mean the
  **existing** `skills-catalog.yaml` also needs an explicit
  `schemaVersion` bump when the role-index is added (since the two are
  closely related artifacts), or does only the **new** role-index file
  get its own independent `schemaVersion: 1`, leaving the existing
  catalog's `schemaVersion: 1` untouched? This audit recommends the
  latter (new artifact, independent version) per the "additive" finding
  below, but flags it for schema-writer/validator-reviewer confirmation.
- **Q-skills-3**: Source doc 9.1 states SDD skills "live in the SDD
  repo." In this workspace's actual practice (per
  `Axiom.Spec/general-spec.md`), `Axiom.SDD` holds the lightweight-SDD
  workflow rules (`AGENTS.md`) but is explicitly "not itself a target for
  product code from this increment chain" — while `@axiom/skills`'
  physical catalog lives inside the `Axiom` repo's own in-repo
  `axiom.spec/config/` path (the product's own dogfooding convention, per
  `Axiom/packages/skills/tests/catalog.test.ts`'s integration test
  resolving `repoRoot` as `Axiom/`, not this workspace's sibling
  `Axiom.Spec`). Does source doc 9.1's "SDD repo" map to `Axiom.SDD`
  (this workspace) or to `Axiom`'s own internal `axiom.spec/` (the
  product's dogfooded spec home)? These are two different repos with
  similar names in two different senses (workspace-level vs
  product-internal). Not blocking this audit (both existing artifacts —
  the skills catalog and the 7 skill `.md` sources — already live inside
  `Axiom/axiom.spec/`), but blocking before docs-skills-writer decides
  where new role-catalog markdown files should physically go.

## Assumptions

- "Role" in this increment means the source doc's product/task role
  concept (e.g. `backend-developer`), not `@axiom/agents`' `AgentRole`
  closed enum (`sdd-orchestration | planning | product-review`), which is
  a different, already-shipped, unrelated classification.
- The existing catalog's `status`/`securityCheckStatus` review fields
  (which the two source documents do not mention at all) must be
  preserved verbatim by any future schema-writer work — they are a real,
  useful addition specific to this codebase's review/security-gate
  workflow (enforced today by `doctor`'s TC-010 check).
- Following the parent roadmap's stated dependency, this increment
  assumes INC-06 (increment/bug/plan command reconciliation) is stable
  enough that any commands a future role-index feature might reference
  will not shift under it. This audit did not re-verify INC-06's closure
  status as part of its scope (out of scope; INC-06 is a separate closed
  increment per the roadmap listing).

## Implementation notes

Audit-only. No files were changed under `Axiom/packages/`, `Axiom/apps/`,
or `Axiom.SDD/` for this increment.

### Files read (exhaustive for `@axiom/skills`)

`Axiom/packages/skills/src/types.ts`, `catalog.ts`, `loader.ts`,
`apply.ts`, `materialize.ts`, `refresh.ts`, `index.ts`;
`Axiom/packages/skills/tests/catalog.test.ts` (all 4 test scenarios read
in full); `Axiom/apps/cli/src/commands/skills.ts` (the `axiom skills`
CLI: `list`, `refresh`, `drift`, `apply` sub-commands); the relevant
`TC-010` check in `Axiom/packages/doctor/src/checks.ts`. Also read
`Axiom/packages/agents/src/catalog.ts` for the contrast case (a
different, already-shipped per-entry `role` field pattern).

### Confirmed catalog structure (ground truth, not roadmap paraphrase)

`SkillsCatalog` (`Axiom/packages/skills/src/catalog.ts`):

```ts
interface SkillsCatalog {
  schemaVersion: 1;
  skills: ReadonlyArray<CatalogEntry>;
}

interface CatalogEntry {
  id: string;
  name: string;
  version: string;
  source: string;               // path relative to rootPath
  status: string;                // review status, e.g. "approved"
  securityCheckStatus: string;   // e.g. "ok"
  bundleHash: string;            // "sha256:<hex>"
}
```

- Physical file: `<rootPath>/axiom.spec/config/skills-catalog.yaml`
  (confirmed three independent ways: `apply.ts`'s
  `SKILLS_CATALOG_RELATIVE_PATH` constant, `doctor/checks.ts`'s TC-010
  `SKILLS_CATALOG_FILENAME` + spec-scope resolution, and
  `catalog.test.ts`'s integration test that reads the real file at
  `Axiom/axiom.spec/config/skills-catalog.yaml` and asserts exactly 7
  entries by id). **This file was not found to physically exist at the
  time of this audit** via direct filesystem search (`find` for
  `*/axiom.spec/config/skills-catalog.yaml` under `Axiom/` returned no
  result) — the path and shape are confirmed by the loader/constant/test
  contract, but the test's own docstring frames it as the canonical
  7-skill catalog, so its absence at audit time is worth a flag: either
  the file is gitignored/generated and regenerated on demand, or it was
  removed/not yet committed. This does not change the schema diff below,
  but a schema-writer should re-check its presence before assuming it can
  edit an existing file in place versus needing to also (re)create it.
- Loader is pure/never-throws: `{ kind: 'ok' | 'absent' | 'malformed' }`
  discriminated union — same defensive pattern as
  `@axiom/agents`' `loadAgentsCatalog`.
- Nothing in `types.ts`, `catalog.ts`, `loader.ts`, `apply.ts`,
  `materialize.ts`, or `refresh.ts` contains any `role`, `mandatory`,
  `available`, `tags`, or per-repo-kind field or grouping concept. The
  catalog is genuinely one global flat list, exactly as the roadmap
  hypothesized — confirmed, not revised.
- `@axiom/skills` also owns a **second**, unrelated concept:
  `SkillRegistry`/`Skill` (from `loader.ts`/`types.ts`), which is a
  derived, read-only view of `.opencode/skills-lock.yaml` (the installed/
  materialized skill state, with `status: 'ok'|'missing'|'moved'|
  'out-of-version'|'unknown'`), consumed by `axiom skills list/refresh/
  drift`. This is a different artifact from the catalog (`SkillsCatalog`)
  and must not be conflated with it when a role-index layer is designed —
  the role-index is a curated/authored artifact (source doc 9.4's
  `mandatory`/`available`), while `SkillRegistry` is a derived/observed
  one (source doc 5.5's "generated list, not versioned" category).

### Field-by-field diff vs. source doc 9.4's role-index shape

Source doc 9.4 example (role-index YAML, flat):

```yaml
role: backend-developer

mandatory:
  - id: backend.repo-rules
    path: skills/backend-developer.md

available:
  - id: backend.ef-core
    path: skills/ef-core.md
    tags: [ef-core, database, persistence]
```

| Aspect | Existing `skills-catalog.yaml` | Target role-index (9.4) | Diff kind |
|---|---|---|---|
| Top-level shape | `{ schemaVersion, skills: [...] }`, one global list | `{ role, mandatory: [...], available: [...] }`, one list per role | **New artifact** (not a reshape of the existing one) |
| Grouping | None — flat, all skills for the whole project | Per-role: one index file/document per role | **Additive**: a new file/section keyed by role, layered on top |
| `id` | Present, stable id, uniqueness-checked | Present (`mandatory[].id`, `available[].id`) | Same concept, reusable — a role-index entry's `id` should reference an existing catalog `id` rather than duplicate its metadata |
| `path`/`source` | `source` (path relative to `rootPath`) | `path` (relative, presumably to the skill file directly) | Naming differs (`source` vs `path`); role-index `path` can likely just be a pointer, or could resolve indirectly via the catalog's `source` — decision for schema-writer |
| `name`, `version` | Present in catalog | Absent in the 9.4 example (index only lists `id`/`path`, sometimes `tags`) | Role-index is intentionally a thin pointer list; it does not duplicate the catalog's descriptive metadata — additive, no conflict |
| `status`, `securityCheckStatus`, `bundleHash` | Present in catalog (review/security-gate fields) | Not mentioned anywhere in source doc 9 | Existing-only fields with no analog in the new docs; **must be preserved**, not dropped, since they are load-bearing for `doctor`'s TC-010 check |
| `mandatory` | Not present as a concept at all | A flat list of `{id, path}` (no nested `always`/`whenTags` at 9.4 — that nesting is 10.2's *technical-context* shape, a different domain) | **New concept**, additive |
| `available` | Not present as a concept (the whole catalog is implicitly "available") | A flat list of `{id, path, tags}}`, `tags` used for semantic selection | **New concept**, additive |
| `tags` | Not present anywhere in `@axiom/skills` | Present on `available[]` entries | **New field**, additive |
| Versioning | `schemaVersion: 1` (unversioned-in-spirit per 5.5's rule since it's a generated-looking flat list, but already has a schema version field in practice) | Per 5.5, `skills.index` explicitly **must** be versioned (it "decides what an agent must read/do") | The **new** role-index artifact needs its own explicit `schemaVersion`; whether it is `1` (new, independent) or must also bump the existing catalog's version is Q-skills-2, open |
| SDD-repo vs per-repo split (9.1/9.2) | Not modeled at all — one catalog, no repo-kind or SDD/technical distinction | Two conceptually separate skill populations (SDD skills vs per-repo technical skills), each potentially with its own role-index | Existing catalog's 7 entries (`axiom-sdd-orchestrator`, `axiom-install-profile`, `axiom-capability-router`, `axiom-context-persistence`, `axiom-token-optimization`, `axiom-code-intelligence`, `axiom-telemetry`) read as Axiom-product/SDD-orchestration-flavored skills, not per-repo technical skills (no backend/frontend-specific entries exist yet) — the existing catalog today covers roughly the 9.1 population only; the 9.2 population (technical skills per code repo) has no existing catalog entries or separate file to reconcile against |
| Role-catalog markdown (9.3) | None found anywhere in the repo | One MD per role explaining what/why/when per skill | **Additive, no legacy content** — nothing to reconcile against; a pure new-content task for docs-skills-writer |

### Additive or restructuring

**Additive.** The existing flat `SkillsCatalog` (`schemaVersion: 1`,
`id/name/version/source/status/securityCheckStatus/bundleHash`) does not
need to change shape, gain new fields, or be migrated to support a
role-index layer. The role-index is a structurally distinct, new artifact
(source doc 9.4's `{role, mandatory, available}` shape) that references
existing catalog `id`s rather than replacing or nesting inside them. This
confirms the roadmap's own hypothesis ("additive if role-based
mandatory/available grouping is layered on top of the existing catalog
schema... rather than replacing it") — no revision needed on this point.
The one correction this audit makes to the roadmap's framing is narrower:
the target shape to layer on top is section 9.4's **flat**
`mandatory`/`available` structure, not section 10.2's **nested**
`mandatory.always`/`mandatory.whenTags` structure — the roadmap's own
diff text cited both sections together as if they described one shape,
but they are two different domains (skills vs technical-context) with two
different index shapes, only one of which (9.4) applies to `@axiom/skills`.

### Brief for schema-writer (next subagent)

1. Design a new, standalone YAML artifact for the per-role skills index
   (working name: `<role>.skills-index.yaml` or similar — exact naming
   and directory placement depend on Q-skills-1), independent from
   `skills-catalog.yaml`.
2. Shape: `{ schemaVersion: 1, role: string, mandatory: Array<{id: string,
   path?: string}>, available: Array<{id: string, path?: string, tags:
   string[]}> }`. Prefer referencing existing `skills-catalog.yaml` `id`s
   over duplicating `path` when the skill is already cataloged; only use
   a direct `path` for skills that are not (yet) in the catalog (e.g. a
   role-specific technical skill per 9.2 that has no catalog entry today).
3. Explicitly version this new artifact per source doc 5.5's rule (it
   decides what an agent reads) — do not treat it as a derived/
   generated-and-unversioned index.
4. Do not modify `CatalogEntry`, `SkillsCatalog`, `loadSkillsCatalog`,
   `apply.ts`, `materialize.ts`, or `refresh.ts` — none of them need to
   change for this layer to exist. Preserve `status`/`securityCheckStatus`/
   `bundleHash` untouched.
5. Before writing the schema, re-check whether
   `axiom.spec/config/skills-catalog.yaml` physically exists in the
   `Axiom` repo (it did not at the time of this audit — see
   "Implementation notes" above) — if it is still absent, flag that as a
   separate, pre-existing gap unrelated to this increment's scope, not
   something to silently create as part of the role-index work.
6. Loop in a validator-reviewer afterward to extend `doctor`'s skills
   category (alongside TC-010) with a new check for the role-index file,
   following the existing `pass/fail/warn/skip` + evidence convention —
   do not invent a new check-result shape.

## Validation

`No validation command was found. Performed best-effort validation by
inspecting changed files and checking consistency against the requested
behavior.`

No code was changed by this increment, so there is nothing to build/test/
lint. Best-effort validation performed:

- Directly read every source file in `Axiom/packages/skills/src/` and its
  full test suite (`tests/catalog.test.ts`) to ground the diff in real
  code rather than the roadmap's paraphrase.
- Cross-checked the catalog's physical path claim against three
  independent sources in the codebase (`apply.ts` constant, `doctor`
  TC-010 check, and the catalog test's integration case) and found they
  agree with each other, even though the file itself was not found on
  disk at audit time (flagged above, not silently assumed away).
- Re-read source doc sections 5.5, 9 (all four subsections), and 10
  (contrast only) directly rather than relying on the roadmap's
  paraphrase, which surfaced the 9.4-vs-10.2 conflation corrected above.
- Searched the addendum document for any skills-specific reinforcement
  beyond the SDD/technical split already known; found none beyond the
  dogfooding-boundary reinforcement (section 2), which does not add new
  skills-index structure.
- Grepped the full `Axiom/packages` tree for `mandatory`/`whenTags`/
  `repoKind`/role-index-like terms to confirm no partial implementation
  exists elsewhere (`config-validation`'s `mandatory: boolean` and
  `@axiom/agents`' per-entry `role` field are the only matches, both
  confirmed unrelated).

## Result

Confirmed the roadmap's hypothesis on the load-bearing points (flat
catalog, `schemaVersion: 1`, exact field set, physical path, additive
direction) and corrected one framing error (the target role-index shape
is source doc 9.4's flat `mandatory`/`available`, not 10.2's nested
`mandatory.always`/`whenTags`, which belongs to the separate
technical-context domain / a later, different roadmap increment, INC-10).
Found no existing per-role or per-repo-kind grouping anywhere in
`@axiom/skills`, and no existing role-catalog markdown (source doc 9.3
style) anywhere in the repo — both are genuinely additive, no-legacy-
content work. Flagged that the physical `skills-catalog.yaml` file itself
was not found on disk at audit time despite being referenced consistently
by three independent code paths, which the next subagent should re-check
before assuming an in-place edit is possible.

## General spec integration

No integration into `Axiom.Spec/general-spec.md` performed. Rationale:
this increment is an audit with no shipped schema or code — `general-
spec.md`'s documentation rules call for consolidating stable, load-bearing
knowledge, and the role-index shape/placement is not yet decided (Q-skills-1
through Q-skills-3 remain open). The existing `@axiom/skills` catalog facts
confirmed here (path, schema, fields) were already implicitly assumed
correct by the parent roadmap and are not new cross-increment knowledge
requiring consolidation; they will be worth adding to `general-spec.md`
once schema-writer's role-index artifact actually lands and its shape is
final, alongside the corrected 9.4-vs-10.2 domain distinction (skills
index vs technical-context index), which is a genuinely reusable
clarification for any future increment touching either domain.

## Next step recommendation

Hand off to **schema-writer** using the brief above, scoped narrowly to
designing the new standalone role-index YAML artifact (not touching the
existing catalog). Before schema-writer starts, resolve Q-skills-1 (file
placement) since it affects where the new schema's file-path constants
will live; Q-skills-2 and Q-skills-3 can be resolved in parallel or
deferred to schema-writer/validator-reviewer without blocking the start
of schema design. After schema-writer, continue the roadmap's stated
subagent sequence for INC-09: docs-skills-writer (role-catalog markdown,
source doc 9.3 style) -> validator-reviewer (extend `doctor`'s skills
category with a new check for the role-index, following TC-010's
pass/fail/warn/skip convention).
