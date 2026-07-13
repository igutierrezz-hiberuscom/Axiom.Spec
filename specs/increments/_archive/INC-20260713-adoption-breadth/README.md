# INC-20260713-adoption-breadth — Adoption breadth (FIX-B)

## Resumen

Two confirmed gaps found while adopting a real KVP25 spec repo now block breadth
of adoption. This increment widens the two bootstrap adopters so a foreign spec
repo shaped like KVP (`Kvp.Spec/specs/{increments,bugs,e2e}` +
`Kvp.Spec/context/{TECHNICAL_CONTEXT.md, architecture/, backend/, ...}`) is fully
adopted when the CLI is pointed at the repo root.

## Contexto y motivación

- **Gap 1 — legacy-sdd detectors don't descend + missing e2e mapping + flat-registry
  false-positive.** Detectors matched folder names ONLY at the scanned root, so
  pointing at the KVP repo root (artifacts nested under `specs/`) found nothing.
  KVP's `e2e/` folder mapped to no detector (silently ignored), and a flat
  `REGISTRO_INCREMENTOS.md` under `increments/` was migrated as a false-positive
  increment.
- **Gap 2 — from-context ingest didn't recognize KVP's context taxonomy.** It
  recognized only a root `ARCHITECTURE.md`, a README architecture section, and
  `docs`/`doc`/`adr` subtrees, so KVP's `context/` tree yielded 0 docs.

## Alcance

1. **Descend (bounded).** `orchestrateScan` retries the whole detector registry
   against `<root>/specs` (then `<root>/spec`) when NO detector matched at the
   pointed root. One level deep, fixed subdir list. Root-level layouts are
   unaffected (root probe wins; descend never runs).
2. **e2e mapping.** `e2e/` is added to the `axiom-native` folder map as
   `increment`. Migrated e2e READMEs carry a distinct `AXIOM:E2E-ORIGIN` marker.
3. **Ignore registry/index files.** The shared folder-map walk skips kind-folder
   entries named (case-insensitive) `REGISTRO_INCREMENTOS.md`, `README.md`,
   `INDEX.md`, `_index.md`, on a dedicated `ignored` channel the command reports.
4. **from-context taxonomy.** Recognize a root `TECHNICAL_CONTEXT.md`
   (-> `architecture`) and ALL immediate subfolders of the pointed context dir as
   categories (classified by `classifyByRelPath`; unknown names -> `references`).

## Non-goals

- No new `ArtifactKind` (`e2e` is not a kind; it maps to `increment`).
- No widening of the closed `ContextCategory` union (unknown context subfolders
  fall back to `references`, not a new folder).
- No deep-walk: descend is one level; nested `archive/` trees are not migrated.
- No changes to `migrator.ts` idempotency/provenance semantics beyond carrying the
  new optional `originFolder` field.

## e2e-mapping decision (D-002)

`e2e/<item>` folders are QA/E2E work items (they carry `README.md` +
`metadata.yml` + a QA plan), so they map to `increment` — the closest work-item
kind. Each migrated e2e README gets an `AXIOM:E2E-ORIGIN` HTML comment (alongside
the AXIOM:MIGRATED banner that names the `.../e2e/...` source), so a reviewer can
find and reclassify them. Origin is tracked via a new optional
`ScannedLegacyItem.originFolder` set by the shared folder-map walk; the verbatim
body and its content hash are untouched (the marker lives only in the README).

## Unknown-context-category decision (D-004)

Immediate subfolders whose name matches the `technical-context-template.md`
taxonomy (`architecture`, `conventions`, `operations`, `testing`, `integrations`,
`references`) map to that category. Unknown names (`backend`, `frontend`,
`functional`, `shared`) fall back to the `references` catch-all. Rationale: the
closed `ContextCategory` union keeps the generated index and MCP consumers within
the known six folders; the path-encoded slug (`deriveSlug`) preserves the origin
folder in the filename, so `references/backend-*.md` and `references/frontend-*.md`
never collide. `TECHNICAL_CONTEXT.md` maps to `architecture` (the architectural
gateway doc).

## Registry/index ignore-list (D-003)

`REGISTRO_INCREMENTOS.md`, `README.md`, `INDEX.md`, `_index.md` (case-insensitive)
at the kind-folder level are skipped and reported on the `ignored` channel. The
guard is at the kind-folder ENTRY level, so a per-artifact folder's own inner
`README.md` (the artifact's body) is never affected.

## Criterios de aceptación

- A repo with `specs/{increments,bugs}` (nested) is detected + scanned; a
  root-level layout still works (`axiom-native` byte-identical for the pre-existing
  root case).
- `e2e/` items are mapped to `increment` with the `AXIOM:E2E-ORIGIN` marker, never
  silently ignored.
- `REGISTRO_INCREMENTOS.md` / `README.md` at a kind folder are ignored (reported,
  not migrated).
- A context dir shaped like KVP yields `technical-context/*` docs + a valid draft
  index; unknown subfolders -> `references`; no-clobber re-run; unreadable doc ->
  failure recorded, scan continues.
- E2E: copying the real KVP `Kvp.Spec` to a throwaway dir and running
  `from-legacy-sdd` at the REPO ROOT now migrates the nested artifacts;
  `from-context` on `<copy>/context` now ingests the context tree.
- Gate green: `npm run build` -> `npm test` -> `npm run typecheck` -> `npm run doctor`.

## Dudas abiertas

Q-001 (deferred): whether nested `archive/` items should be migrated. Out of scope;
the descend stays bounded to one level.

## Result

**Archived — complete, gate green.** All four scope items shipped and verified by
unit + command tests and a real KVP25 e2e run.

Changes (surgical):
- `bootstrap-from-legacy-sdd/scanner.ts` — `e2e -> increment` in the folder map;
  new optional `ScannedLegacyItem.originFolder`; new `IgnoredEntry` type; `ignored`
  + `scannedSubPath` on `ScanLegacyRepoResult`.
- `detectors/shared.ts` — `IGNORED_REGISTRY_FILENAMES` guard at the kind-folder
  entry level (reports on the `ignored` channel); sets `originFolder`.
- `detectors/registry.ts` — bounded descend: `orchestrateScan` retries the whole
  registry against `<root>/specs` then `<root>/spec` only when nothing matched at
  root; merges `ignored`; reports `scannedSubPath`.
- `detectors/types.ts` — optional `ignored` on `DetectorScanResult`.
- `bootstrap-from-legacy-sdd/migrator.ts` — `AXIOM:E2E-ORIGIN` marker on e2e-origin
  READMEs (README only; body hash / idempotency unchanged).
- `commands/bootstrap.ts` — reports the descend + the ignored entries; broadened
  from-context no-op message.
- `bootstrap-from-context/ingest.ts` — root `TECHNICAL_CONTEXT.md` -> architecture;
  walk ALL immediate subfolders (ignore-list + dot-dir aware) excluding work-item
  folders (`increments`/`bugs`/`plans`/`e2e`); unknown names -> `references`.

Tests: +10 (2 detector describes, 3 ingest, 1 command e2e). Full suite 2749 passed
(2739 baseline + 10). build / typecheck exit 0; doctor PASS.

Real KVP25 e2e (throwaway copy of `Kvp.Spec`, pointed at REPO ROOT):
- `from-legacy-sdd <root>`: descended to `specs/`; detected axiom-native (400
  items); created 229 artifacts (167 increments — 59 of them e2e-origin carrying
  `AXIOM:E2E-ORIGIN` — + 62 bugs); `REGISTRO_INCREMENTOS.md` and `e2e/README.md`
  reported as ignored (0 false-positives); re-run idempotent (already-migrated 400).
  `specs/{increments,bugs,e2e}/archive` surfaced as reported failures (Q-001,
  deferred — nested archive walk out of scope, never silent).
- `from-context <root>/context`: ingested 162 docs (was 0 before) — architecture 11,
  testing 3, operations 1, references 147 (backend/frontend/functional/shared with
  origin-preserving slugs); `TECHNICAL_CONTEXT.md` -> architecture; valid draft
  index emitted.
