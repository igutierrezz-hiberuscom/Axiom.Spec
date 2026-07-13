# Increment: Foreign spec-repo adopter — format detectors + structural conversion

> **Código**: INC-20260711-mig-spec-adopter
> **Estado**: Archivado (Phases 1+2 implementados; gate verde — 2658 tests — see Result)
> **Fecha de creación**: 2026-07-12
> **Tipo de cambio**: Nueva capacidad
> **Implements**: Phases 1 & 2 of `INC-20260711-spec-sdd-migration-plan`

## Goal

Turn `axiom bootstrap from-legacy-sdd` from a folder-name-only migrator into a
**pluggable, format-aware adopter** for foreign spec repos, and reshape migrated
bodies toward the canonical increment template instead of copying verbatim —
without losing any information and without changing the safe status semantics.

This increment implements the parent plan's **§Proposed design (1) Pluggable
format-detector layer** and **(2) Structural conversion pass**, satisfying its
**Phase 1** and **Phase 2** acceptance + corner cases. Phase 0 (spec-scope
routing) already shipped (INC-20260711-audit-bug-fixes); this builds on it —
migration writes to the resolved spec scope.

## Scope

### Incluido

- **Pluggable format-detector layer** (`bootstrap-from-legacy-sdd/detectors/`):
  a `SpecFormatDetector` interface (`id` / `detect(repoRoot): DetectionScore` /
  `scan(repoRoot): DetectorScanResult`), a detector registry, and an
  orchestrator that turns `scanLegacyRepo` into a dispatcher: run each
  detector's `detect`, pick the highest-confidence one, or **union
  non-overlapping detectors** for a mixed repo; overlaps resolve by confidence
  and are surfaced.
- **Four detectors**: `axiom-native` (today's `LEGACY_FOLDER_TO_KIND`, extracted
  verbatim → zero regression), `openspec` (`openspec/changes/<n>/` → increment),
  `docs-adr` (`docs/adr/*.md` / `doc/adr/*.md` → adr, MADR front-matter), and
  `generic-folders` (alias map `proposals|rfcs|stories → increment`,
  `defects|issues → bug`).
- **Broadened, pluggable status extraction**: a YAML front-matter reader plus a
  widened synonym/known-value table for the new detectors. The narrow
  `extractLegacyStatus` used by `axiom-native` is left untouched. Detectors
  provide a best-effort `rawStatus`; `migrator.ts`'s safe resolver stays
  authoritative for the final status.
- **Structural conversion pass** (`bootstrap-from-legacy-sdd/convert.ts`), wired
  between scan and write: maps recognizable sections (Summary/Scope/Acceptance →
  canonical anchors) into a template-filled README (reusing
  `@axiom/document-bootstrap`'s `PLACEHOLDER_REGEX` `{{var}}` engine against the
  `increment-template.md` anchors), drops unmapped content into a clearly-labeled
  "Contenido original migrado" appendix (no data lost), and **downgrades
  gracefully to today's verbatim-with-banner output** when the body has no
  recognizable structure.
- **Per-run migration report** rendered from `templates/migration-template.md`
  and written to each migrated item's `context/migration-report.md` (source
  system/detector, artifacts detected, proposed mapping, risks, open questions).
- **Dry-run** surfaces the detected format(s), per-kind counts, and the resolved
  status each item would receive.

### Excluido (Non-goals)

- **No context ingest** (`from-context` / `ARCHITECTURE.md` → technical-context)
  — that is the parent plan's Phase 3.
- **No idempotency / source→artifact provenance map** — Phase 4. Re-runs keep
  today's behavior (fresh minted IDs, never overwrite).
- **No install-time adoption UX** (`axiom workspace setup` branch) — Phase 5.
- **No change to status semantics**: `resolveWorkflowStateForMigration` /
  `resolveDisplayStatusForMigration` are untouched and authoritative.
- **No general-purpose importer** for every SDD tool (D-001 of the parent plan).

## Acceptance (copied from the parent plan's Phase 1 & 2)

**Phase 1** — an OpenSpec repo (`openspec/changes/*`) and a `docs/adr/*` repo are
detected and scanned into `ScannedLegacyItem[]`; front-matter `status:` is
honored via the safe resolver; a repo that already matches the legacy folder
names produces **byte-identical** results to today; `--dry-run` lists the
detected format(s). Corner cases: mixed formats (union non-overlapping
detectors; overlaps by confidence, surfaced); a repo matching no detector →
empty result + a clear "no recognized format" line (today's "no migrable items"
message, extended).

**Phase 2** — a migrated increment's README follows `increment-template.md`
structure when the source had recognizable sections; no source content is lost
(unmapped content appears in the appendix); unstructured sources still migrate
(verbatim fallback) with no error; a migration report is written to the item's
`context/`. Corner cases: body with only a title; conflicting/multiple `Status:`
lines (first-match wins, documented); enormous single body (cap the appendix,
respect ignore lists, keep dry-run affordable).

## Dudas abiertas

- **Q-004 (ID policy)** — blocking for Phase 4, not for this increment. Deferred;
  current mint-fresh-never-overwrite behavior retained.

## Decisiones funcionales cerradas

| ID | Decisión | Resultado |
|----|----------|-----------|
| D-001 | `ScannedLegacyItem` is the stable seam; status semantics unchanged | `axiom-native` reproduces today's scan verbatim; safe resolver authoritative |
| D-002 | Conversion is graceful-downgrade | Unstructured bodies → byte-identical verbatim-with-banner output; no data loss/invention |

## Result

**Phases 1 & 2 implemented and green.** See the parent plan for the full design.

- Detector registry + orchestrator: `apps/cli/src/bootstrap-from-legacy-sdd/detectors/`
  (`types.ts`, `status.ts`, `shared.ts`, `axiom-native.ts`, `openspec.ts`,
  `docs-adr.ts`, `generic-folders.ts`, `registry.ts`, `index.ts`).
- Conversion + migration report: `apps/cli/src/bootstrap-from-legacy-sdd/convert.ts`,
  wired via `migrator.ts` (`buildReadmeContent` now delegates to `convert`, banner
  ownership unchanged; report written to `context/migration-report.md`).
- Orchestration + dry-run format reporting: `apps/cli/src/commands/bootstrap.ts`.
- Regression proof: `axiom-native` shares the exact folder-scan logic and the
  narrow status extractor, so a legacy repo yields byte-identical README/metadata;
  the entire pre-existing `from-legacy-sdd` suite stays green.
- Gate: `npm run build` / `npm test` / `npm run typecheck` / `npm run doctor` all
  green (see the increment execution report).

## Trazabilidad y fuentes

- Parent plan: `specs/increments/INC-20260711-spec-sdd-migration-plan/README.md`.
- Grounding code: `apps/cli/src/bootstrap-from-legacy-sdd/{scanner,migrator}.ts`,
  `apps/cli/src/commands/bootstrap.ts`, `@axiom/document-bootstrap`
  (`renderer.ts` / `guarded-write.ts`).
- Templates: `templates/increment-template.md`, `templates/migration-template.md`.
