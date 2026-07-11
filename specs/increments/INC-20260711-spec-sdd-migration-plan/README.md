# Increment: Migrate foreign spec/SDD repos into Axiom-apt form

Status: pending (design spec + phased plan — no code in this artifact)
Date: 2026-07-11
Type: New capability
Owner: migration-engineer

> This document is a DESIGN SPEC and a PHASED PLAN. It specifies what a future
> implementation increment must build. It contains no production code and
> mandates no build. Every reference to current behavior is grounded in the
> real code paths cited inline.

## Goal

At install time, a user may point Axiom at an **existing spec repo** and/or an
**existing SDD/tooling repo** that predate Axiom and do not follow Axiom
conventions. This increment specifies how to **adopt** (migrate) such foreign
repos into an Axiom-apt, MCP-queryable form:

- a foreign **spec repo** becomes canonical `specs/{increments,bugs,plans,adr,decisions}/<id>/{metadata.yml,README.md}` plus a navigable `technical-context/` served over MCP;
- a foreign **SDD/tooling repo** is brought into conformance with the layout Axiom's adapters + MCP expect (`AGENTS.md`, skills/agents/commands, MCP config), or the gaps are reported precisely enough to close them.

The through-line requirement: after adoption, the migrated content is
**queryable over MCP** (e.g. `spec.technicalContextIndexRead`,
artifact listing) with the same guarantees as freshly-created Axiom content,
and every migrated item carries visible provenance and is safe-by-default
(draft/proposed, never auto-approved).

## Scope

- A **pluggable format-detector layer** that replaces the folder-name-only map in `bootstrap-from-legacy-sdd/scanner.ts` with adapters (OpenSpec `openspec/changes/`, `docs/adr/*`, front-matter status, alternate folder names).
- A **structural conversion** pass that reshapes foreign bodies toward the canonical templates instead of copying verbatim, reusing `@axiom/document-bootstrap`'s existing renderer/guarded-write primitives.
- A new **`from-context` / ingest-context** mode that reads an existing technical context (`ARCHITECTURE.md`, `docs/`, ADRs) and maps it into `technical-context/*` + a `TechnicalContextIndex`, wired through the same guarded-write + index path `from-code` already uses (so MCP-queryability is free).
- **Install-time integration**: `axiom workspace setup` / init offers "adopt existing spec/sdd repo" and invokes migration, preserving no-clobber + provenance-banner + dry-run + collision-skip guarantees.
- Idempotency, provenance (`AXIOM:MIGRATED`), and correct **multi-repo placement** (write into the RESOLVED spec scope, assuming the known scope bug — below — is fixed first).

## Non-goals

- **Not** a general-purpose importer for every SDD tool in existence. This iteration ships a small, named detector set; the registry is designed to grow, but exhaustive coverage is explicitly out of scope.
- **Not** a semantic/architectural understanding pass over foreign code. `from-context` ingests *existing docs*; it does not re-derive architecture from source (that stays the Level-2+ boundary `bootstrap-from-code/analyzer.ts` already declines).
- **Not** an auto-approval path. Everything migrated lands draft/proposed and requires human review, matching today's `from-legacy-sdd`/`from-code` posture.
- **Not** a rewrite of the artifact-store creation primitives. Migration continues to reuse `generateUniqueArtifactId`/`saveArtifactMetadata`/`makeInitial*Metadata` (`@axiom/workflow`) and `writeGuardedFile`/`technicalContextIndexPath`.
- **No** folder/path renames in the target workspace; **no** enterprise lifecycle constructs.

## Two migration subjects, treated separately

Migration has two distinct subjects with different targets, primitives, and
success criteria. They share the detector/conversion/provenance machinery but
must not be conflated.

### A. Foreign SPEC repo → canonical Axiom spec + technical context

**Input**: a repo of specification content in some non-Axiom shape (OpenSpec
`openspec/changes/*`, a Jira/Markdown export, `docs/adr/*`, differently-named
folders like `proposals/`, `rfcs/`, `stories/`).

**Target**:
- Per-artifact canonical folders `specs/{increments,bugs,plans,adr,decisions}/<generated-id>/` each with a `metadata.yml` (via the `makeInitial*Metadata` factories) and a `README.md` (converted body + provenance banner). This is exactly what `bootstrap-from-legacy-sdd/migrator.ts` produces today via `resolveArtifactDir`/`saveArtifactMetadata`/`ensureArtifactReadmeWithContent`.
- Any prose describing the *system/product* (as opposed to a discrete change item) routed into `technical-context/` + a `TechnicalContextIndex` — the subject-B mechanism below, reused.

**Primitives to reuse (already present)**:
- `@axiom/workflow`: `generateUniqueArtifactId`, `artifactExists`, `saveArtifactMetadata`, `resolveArtifactDir`, `makeInitialIncrementOrBugMetadata` / `makeInitialPlanMetadata` / `makeInitialAdrMetadata` / `makeInitialDecisionMetadata`.
- Templates: `templates/increment-template.md`, `templates/increment-metadata-template.yaml`, `templates/migration-template.md` (the last one is the natural home for a per-run migration report: source system, artifacts detected, proposed mapping, risks, open questions).

**Success criteria**: every migrated item is listable/queryable exactly like a natively-created artifact; each carries the `AXIOM:MIGRATED` banner naming its source path; statuses are resolved safely (never trust an unrecognized legacy status).

### B. Foreign SDD/tooling repo → Axiom-conformant control repo

**Input**: an existing repo that plays the "control / process" role (skills,
agent definitions, prompt/command libraries, CI, its own MCP config) but was
not scaffolded by `axiom workspace setup`.

**What it must conform to** (derived from what `runWorkspaceSetup`
materializes and what the adapters/MCP consume):

| Concern | Axiom-apt shape | Produced/consumed by |
|---|---|---|
| Structural contract | repo-root canonical `AGENTS.md` (with the `AXIOM:GENERATED` block + preserved `TEAM:CUSTOM`) | `@axiom/document-bootstrap`'s `writeCanonicalAgentsMd` / `canonical-agents-md.ts` |
| Repo identity + wiring | `axiom.yaml` schemaVersion 2 with role + role-aware `paths` map | `workspace-setup.ts` `buildRoleAwareAxiomYaml` / `tryReadExistingProjectId` |
| Topology | `axiom.config/topology.yaml` (control repo only) | `buildTopologyManifest` / `writeTopologyManifest` |
| Skills layout | `skills-index/*.yaml` + materialized skills baseline | `scaffoldSddSkills` / `scaffoldCodeRepoSkills` |
| MCP config | `.axiom/mcp.yml` (canonical) + native per-adapter configs | `workspace-mcp.ts` `buildWorkspaceMcpServers` / `writeWorkspaceMcpConfig` / `writeWorkspaceNativeMcpConfigs` |
| Architect declarations | `axiom.config/mcp-manifest.yaml`, `toolchain-catalog.yaml`, `profiles.yaml` | `scaffoldArchitectDeclarations` / `scaffoldProfilesYamlIfMissing` |

**Migration approach for subject B is additive parametrization, not
conversion**: `runWorkspaceSetup` already supports adopting an existing
directory in-place via `WorkspaceRepoSpec.create: false` and a per-file
no-clobber guard (`writeOneRepo` refuses to overwrite an `axiom.yaml` owned by
a different `projectId`, and `scaffoldSpecRepoBase`/`scaffoldRules` guard
per-file). So subject B is mostly: **run setup in adopt mode + emit a
conformance report of what was already present, what was added, and what
must be reconciled by hand** (e.g. a hand-written MCP config that conflicts
with the generated one). No verbatim body copy is involved for subject B —
the "content" here is config/layout, which is regenerated idempotently.

## Gap analysis vs current `from-legacy-sdd` / `from-code`

Grounded in the real code:

1. **Detection is folder-name-only.** `scanner.ts` hardcodes
   `LEGACY_FOLDER_TO_KIND = { increments, bugs, plans, adr, decisions }` and
   iterates only those literal names (`LEGACY_FOLDER_NAMES`). Any foreign
   layout — OpenSpec `openspec/changes/`, `docs/adr/`, `proposals/`, `rfcs/`
   — is silently skipped (`scanLegacyRepo` `continue`s on an absent kind
   folder). There is no content sniffing and no front-matter awareness.

2. **Status parsing is one narrow regex.** `STATUS_LINE_REGEX =
   /^Status:\s*(\w[\w-]*)/im` matches only a literal `Status:` line.
   YAML front-matter (`status: accepted`), OpenSpec change status, or a
   `**Status**: Done` variant are missed, and `extractLegacyStatus` then
   returns `undefined`, so everything falls back to `draft` via
   `resolveWorkflowStateForMigration`.

3. **Body copy is verbatim.** `migrator.ts` `buildReadmeContent` does
   `[provenanceMarker, '', body.trimEnd(), '']` — the foreign body is carried
   as-is, wrapped only with the `AXIOM:MIGRATED` banner. Nothing reshapes it
   into the canonical `increment-template.md` structure (Resumen / Alcance /
   Criterios de aceptación / Dudas abiertas). A migrated artifact is
   listable but not template-conformant.

4. **No context ingest.** `bootstrap-from-code` generates a **fresh**
   technical-context index from *mechanical facts only* (`analyzer.ts`:
   package manifests, top-level folder names, `package.json#scripts`). It
   does **not** read an existing `ARCHITECTURE.md`, `docs/` tree, or ADR set
   and adapt it. "Adopt an existing context" is entirely missing.

5. **Known scope bug (must be fixed first).** `runBootstrapFromLegacySdd`
   sets `const projectRoot = resolution.rootPath` and passes that straight to
   `migrateLegacyItems`, which writes via `resolveArtifactDir(projectRoot,
   …)` / `saveArtifactMetadata(projectRoot, …)`. Unlike
   `runBootstrapFromCode`, it never calls `resolveSpecScopeAbsolutePath`.
   In a multi-repo topology this writes migrated artifacts into the **control
   repo root**, not the resolved **spec scope**. This plan assumes a
   prerequisite fix that routes writes to the resolved spec scope (Phase 0).

## Proposed design

### 1. Pluggable format-detector layer (replaces the hardcoded folder map)

Introduce a detector registry in a new `bootstrap-from-legacy-sdd/detectors/`
module. A detector is a pure, read-only function:

```
interface SpecFormatDetector {
  id: string;                       // 'axiom-native' | 'openspec' | 'docs-adr' | 'generic-folders'
  detect(repoRoot): DetectionScore; // cheap probe: does this repo look like my format?
  scan(repoRoot): ScannedLegacyItem[]; // produce the SAME ScannedLegacyItem shape scanner.ts already emits
}
```

- Keep the existing `ScannedLegacyItem` contract (`kind`, `childName`,
  `sourceFilePath`, `title`, `rawStatus`, `body`) as the **stable seam** so
  `migrator.ts` is unchanged by the detector work.
- `scanLegacyRepo` becomes an orchestrator: run each detector's `detect`,
  pick the highest-confidence one (or union non-overlapping detectors for a
  mixed repo — see corner cases), then delegate to `scan`.
- Ship these detectors first:
  - **axiom-native** — the current `LEGACY_FOLDER_TO_KIND` behavior, extracted verbatim (guarantees zero regression for repos that already match).
  - **openspec** — maps `openspec/changes/<name>/` → increment, `openspec/specs/` prose → technical-context candidate; reads OpenSpec change status.
  - **docs-adr** — maps `docs/adr/*.md` / `doc/adr/*.md` → `adr`, parsing MADR-style front-matter/status.
  - **generic-folders** — configurable alias map (`proposals|rfcs|stories → increment`, `defects|issues → bug`) so alternate names are recognized.
- **Status extraction becomes pluggable too**: add a front-matter reader (YAML
  fence) and broaden the synonym table already in `scanner.ts`
  (`STATUS_SYNONYMS` / `KNOWN_STATUS_VALUES`). Detectors provide a `rawStatus`;
  `migrator.ts`'s safe-resolution (`resolveWorkflowStateForMigration` /
  `resolveDisplayStatusForMigration`) is unchanged and remains the single
  source of truth for the final status.

### 2. Structural conversion pass (reshape, don't copy verbatim)

Insert a conversion step between scan and write, in a new
`bootstrap-from-legacy-sdd/convert.ts`:

- Input: a `ScannedLegacyItem` + its target kind. Output: a canonical-shaped
  README body (and, where the template mandates, the sibling docs referenced
  by `increment-template.md` — `01_Requisitos`, `03_Criterios_Aceptacion`).
- Reuse `@axiom/document-bootstrap`'s renderer (`renderCopilotInstructions`'s
  `{{var}}` substitution engine / `PLACEHOLDER_REGEX`) against the canonical
  templates in `Axiom.Spec/templates/`, so conversion is a template-fill, not
  bespoke string assembly. Best-effort mapping: pull a `# heading` into the
  title (already done by `extractLegacyTitle`), map recognizable sections
  (Summary/Scope/Acceptance) into canonical anchors, and drop everything
  unmapped into a clearly-labeled "Migrated original content" appendix so no
  information is lost.
- **Downgrade gracefully**: when a body has no recognizable structure, fall
  back to today's verbatim-with-banner behavior rather than inventing content.
  Conversion is additive quality, never a data-loss risk.
- Emit a per-run **migration report** from `templates/migration-template.md`
  (source system, artifacts detected, proposed mapping, risks, open questions)
  written to the increment's `context/` — this makes the run auditable.

### 3. New `from-context` / ingest-context mode

Add `axiom bootstrap from-context <path>` (or `--ingest-context` on the
adoption flow), in a new `bootstrap-from-context/` module that mirrors the
shape of `bootstrap-from-code/` (analyzer → drafter → index-builder) but
sources from **existing docs** instead of mechanical facts:

- **ingest** (replaces `analyzer.ts`): discover context docs — `ARCHITECTURE.md`, `README.md` architecture sections, `docs/**` trees, `adr/` — and classify them by convention (architecture / conventions / operations / testing / integrations / references, matching the `technical-context-template.md` subfolders).
- **map** (replaces `drafter.ts`): copy/convert each doc into `technical-context/<role-or-kind>/<slug>.md`, prepend an `AXIOM:MIGRATED` provenance banner (distinct from from-code's `AXIOM:DRAFT`), and produce one `DraftedDocument`-shaped entry per doc.
- **index** (reuse `buildTechnicalContextIndex` from `bootstrap-from-code/index-builder.ts` verbatim): emit `available` entries only, `mandatory.always`/`whenTags` left empty (curation is a human judgment — same Q-bootstrap-1 rationale), and `status: 'draft'` on the `TechnicalContextIndex`.
- **write via the exact from-code path**: `writeGuardedFile(specScope, 'technical-context/…', content)` for docs and `technicalContextIndexPath(specScope, role)` for the index. Because the index conforms to `@axiom/technical-context`'s `validateTechnicalContextIndex`, `spec.technicalContextIndexRead` (MCP) serves it immediately — **MCP-queryability is free**, no new MCP surface needed.

This closes gap #4 directly and shares ~all of from-code's write plumbing.

### 4. Install-time integration

Extend the adoption entry point (`axiom workspace setup` / the init wizard)
with an **"adopt existing spec/sdd repo"** branch. `WorkspaceSetupSpec`
already models a repo that exists on disk (`WorkspaceRepoSpec.create: false`,
parametrized in-place). The adoption branch:

1. Collects the foreign repo path(s) and whether each is a spec repo, an SDD/control repo, or both.
2. For an SDD/control repo: run `runWorkspaceSetup` in adopt mode (`create: false`), relying on the existing no-clobber guards (`tryReadExistingProjectId` → skip-with-warning if a foreign `axiom.yaml` belongs to another project; per-file guards in `scaffoldSpecRepoBase`/`scaffoldRules`/`scaffoldArchitectDeclarations`). Produce the subject-B conformance report.
3. For a spec repo: after `scaffoldSpecRepoBase` establishes the base (only when the repo was freshly created — never clobbering an existing base), invoke the detector-driven migration (subject A) and, if requested, `from-context` ingest (subject B/context).
4. **Always offer `--dry-run` first** on the migration step (the existing `runBootstrapFromLegacySdd` dry-run already previews counts + the exact status each item would receive via `resolveDisplayStatusForMigration`). Adoption should default to showing the dry-run summary and requiring confirmation before writes.

**Preserved guarantees** (must hold for every adoption path):
- **No-clobber**: never overwrite an existing artifact (migrator already skips on `artifactExists`), an existing `axiom.yaml` of another project, or an existing technical-context doc/index (guarded-write callers gate on non-existence).
- **Provenance banner**: every migrated README/context doc carries `AXIOM:MIGRATED` naming its source path.
- **Dry-run**: zero writes; report exactly what would be created and at what status.
- **Collision-skip**: one bad/colliding entry is skipped with a clear reason and never aborts the batch (current `migrateLegacyItems` posture).

### 5. Idempotency, provenance, and multi-repo placement

- **Idempotency**: a second adoption run over the same foreign repo must not
  create duplicates. Today's `generateUniqueArtifactId` always mints a fresh
  timestamped ID, so re-runs *duplicate*. Fix: record a **source→artifact
  provenance map** (e.g. a small `technical-context/.migration-provenance.yml`
  or a marker in each migrated `metadata.yml`) keyed by the normalized source
  path/hash, and on re-run **skip** sources already migrated. The
  `AXIOM:MIGRATED` banner already embeds the source path, giving a fallback
  reconciliation signal.
- **Provenance markers**: keep `buildProvenanceMarker` (`AXIOM:MIGRATED`) for
  spec artifacts; use the parallel banner for context docs; keep from-code's
  `AXIOM:DRAFT` distinct (fresh mechanical draft vs. migration of an existing
  document — the two must remain visually distinguishable to a reviewer).
- **Multi-repo placement (depends on Phase 0)**: all writes MUST target the
  **resolved spec scope** via `resolveSpecScopeAbsolutePath(resolution)` (the
  pattern `runBootstrapFromCode` already uses), not `resolution.rootPath`.
  This guarantees migrated artifacts land in the spec repo of the topology,
  where MCP serves them and where `scaffoldSpecRepoBase` already put the base.

## Phased implementation plan

Each phase is independently shippable and ordered by value/risk. Every phase
keeps `from-legacy-sdd`/`from-code` behavior backward-compatible unless the
phase's explicit goal is to change it.

### Phase 0 — Fix spec-scope routing (prerequisite, smallest, highest-risk-if-skipped)

- Route `runBootstrapFromLegacySdd` writes to `resolveSpecScopeAbsolutePath(resolution)` (same helper `from-code` uses), falling back to `resolution.rootPath` only for single-repo/self-hosted where the spec scope is the root.
- **Acceptance**: in a multi-repo topology, `axiom bootstrap from-legacy-sdd` creates artifacts under the spec repo's `specs/…`, not the control repo; existing single-repo behavior unchanged; a regression test asserts the target path resolves to the spec scope.
- **Corner cases**: no spec scope resolvable → exit 1 with the same message `from-code` emits; spec scope == root (self-hosted) → identical to today.

### Phase 1 — Pluggable detector layer + broadened status parsing

- Extract the current folder-map behavior into an `axiom-native` detector (zero-regression), add the detector registry + orchestrator in `scanLegacyRepo`, and add a front-matter/status-synonym reader. Ship `openspec`, `docs-adr`, and `generic-folders` detectors.
- **Acceptance**: an OpenSpec repo (`openspec/changes/*`) and a `docs/adr/*` repo are detected and scanned into `ScannedLegacyItem[]`; front-matter `status:` is honored via the safe resolver; a repo that already matches the legacy folder names produces byte-identical results to today; `--dry-run` lists the detected format(s).
- **Corner cases**: mixed formats (see below); a repo matching no detector → empty result + a clear "no recognized format" line (today's "no migrable items" message, extended).

### Phase 2 — Structural conversion pass

- Add `convert.ts` and wire it between scan and write; convert recognizable sections into canonical template anchors, appendix the rest, and fall back to verbatim on unstructured bodies. Emit the per-run migration report from `migration-template.md`.
- **Acceptance**: a migrated increment's README follows `increment-template.md` structure when the source had recognizable sections; no source content is lost (unmapped content appears in the appendix); unstructured sources still migrate (verbatim fallback) with no error; a migration report is written to the item's `context/`.
- **Corner cases**: body with only a title; body with conflicting/multiple `Status:` lines (first-match wins, documented); enormous single body (stream/limit — see huge repos).

### Phase 3 — `from-context` ingest mode

- Add the `bootstrap-from-context/` module (ingest → map → reuse `buildTechnicalContextIndex`), the `axiom bootstrap from-context <path>` command, and route writes through `writeGuardedFile` + `technicalContextIndexPath`.
- **Acceptance**: pointing at a repo with `ARCHITECTURE.md` + `docs/**` produces `technical-context/<role>/*.md` + a valid `TechnicalContextIndex` (`status: draft`, `available`-only, `mandatory.*` empty) that `spec.technicalContextIndexRead` serves over MCP; every generated doc carries the `AXIOM:MIGRATED` banner; re-run does not clobber human-edited docs.
- **Corner cases**: no context docs found → no-op + explicit message; a doc that fails to read → recorded as a failure, scan continues (mirror `scanLegacyRepo`'s failure posture); doc name collisions → deterministic slug + skip-if-exists.

### Phase 4 — Idempotent re-runs + provenance map

- Add the source→artifact provenance map and skip-already-migrated logic for both subject A (artifacts) and the context ingest.
- **Acceptance**: running adoption twice over the same foreign repo creates each artifact/doc exactly once; the second run reports "N already migrated, skipped"; deleting a migrated artifact and re-running recreates only that one.
- **Corner cases**: source moved/renamed between runs (hash-based match mitigates); provenance file corrupted → treat as absent, fall back to `artifactExists`/non-existence guards (never duplicate silently, never crash).

### Phase 5 — Install-time adoption UX

- Add the "adopt existing spec/sdd repo" branch to `axiom workspace setup` / the init wizard, defaulting to a dry-run preview + confirmation, and emit the subject-B conformance report.
- **Acceptance**: from a fresh install, a user can point Axiom at an existing spec repo and/or SDD repo, see a dry-run summary, confirm, and end with migrated + MCP-queryable content; all four preserved guarantees (no-clobber, provenance, dry-run, collision-skip) verified by tests; a partially-Axiom repo is adopted without clobbering its existing Axiom parts.
- **Corner cases**: adopting a repo that is already a valid Axiom project (idempotent no-op + report); user declines at the confirmation step (zero writes).

### Corner cases (cross-phase checklist)

- **Empty repo** — detectors return nothing; adoption is a no-op with a clear message (no failure).
- **Partially-Axiom repo** — some artifacts already conform; no-clobber + provenance skip means only the non-conforming/foreign items are migrated; existing Axiom items untouched.
- **Conflicting IDs** — a foreign item carrying an Axiom-style ID that collides: current behavior mints a fresh ID and skips on true collision. Q-004 decides whether to preserve the original ID for idempotency. Never overwrite.
- **Mixed formats** — a repo with both `openspec/changes/` and `docs/adr/`: the orchestrator unions non-overlapping detectors (each owns disjoint paths); overlapping claims resolve by confidence, with the conflict surfaced in the migration report.
- **Huge repos** — bound the scan (respect ignore lists like `analyzer.ts`'s `IGNORED_DIR_NAMES`), stream large bodies, cap per-doc size for the appendix, and keep the dry-run affordable so a user can preview before committing to a large write.

## Open questions / decisions for the owner

- **Q-001 (detector set)**: which foreign formats are in-scope for the first pass beyond `axiom-native` (OpenSpec, docs-adr, generic-folders proposed)? Blocking: no.
- **Q-002 (conversion depth)**: reshape into the full multi-doc canonical increment layout, or land a single sectioned README first and defer doc-splitting? Blocking: no.
- **Q-003 (context index status)**: confirm `from-context` emits `status: draft` and never populates `mandatory.*` mechanically (mirrors from-code Q-bootstrap-1/3). Blocking: no.
- **Q-004 (ID policy)**: preserve a foreign Axiom-style ID for idempotent re-runs, or always mint fresh (current behavior)? Blocking: **yes** — it determines the Phase 4 provenance design.
- **Decision D-001 (recorded)**: this iteration is a targeted adopter for one foreign spec repo and one foreign SDD repo, not a general importer; the detector registry grows incrementally.

## Traceability and sources

- Current code grounding: `apps/cli/src/commands/bootstrap.ts`, `apps/cli/src/bootstrap-from-legacy-sdd/{scanner.ts,migrator.ts}`, `apps/cli/src/bootstrap-from-code/{analyzer.ts,drafter.ts,index-builder.ts}`, `apps/cli/src/commands/workspace-setup.ts`.
- Reused primitives: `@axiom/document-bootstrap` (`renderer.ts`, `guarded-write.ts`, `canonical-agents-md.ts`), `@axiom/technical-context` (`technical-context-index.ts`), `@axiom/workflow` artifact-store factories.
- Templates: `Axiom.Spec/templates/{migration-template.md,increment-template.md,increment-metadata-template.yaml,technical-context-template.md}`.
- Prior audits/increments this builds on: `INC-20260702-bootstrap-from-code-reconcile-impl`, `INC-20260702-bootstrap-legacy-sdd-reconcile-impl`, `INC-20260705-workspace-multirepo-setup-engine`.
