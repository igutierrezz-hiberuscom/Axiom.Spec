# Increment: Provenance/lifecycle metadata + global migration manifest

Status: closed
Date: 2026-07-24

## Goal

Add structured, rollback-grade origin/lifecycle metadata to artifacts in
`<project>.axiom`, plus a global migration manifest — so Axiom can later
export/rollback (INC-A4) knowing exactly what was migrated vs created vs
modified.

This is **INC-A3** of Cluster A in the Axiom evolution plan
(`giggly-imagining-moore.md`, "Arquitectura objetivo" / "Provenance:
ficheros que marcan `migrated`/`migrated-and-modified`/`axiom-native`"),
building on **INC-A1** (topology `axiomRepo`/`legacyRepos`, schemaVersion 2)
and **INC-A2** (`INC-20260724-adopt-creates-axiom-repo`, closed — adoption
now creates a NEW `<project>.axiom` repo and migrates legacy content into
it, leaving legacy untouched). Explicit, user-approved graduation to the
full product lifecycle (`Axiom.SDD/AGENTS.md`'s "Explicit Bootstrap Limits"
exception, per the plan) — this feature is NOT down-scoped.

## Context

Before this increment: migrated artifacts carried only an `AXIOM:MIGRATED`
README banner (`bootstrap-from-legacy-sdd/migrator.ts`) plus a HIDDEN
idempotency map (`bootstrap-shared/provenance.ts`,
`technical-context/.migration-provenance.yml`) used only to skip re-migrating
an already-migrated source. `metadata.yml` (`packages/workflow/src/
artifact-store.ts`) had no concept of WHERE an artifact came from or whether
Axiom had modified it since migration — there was no way to later (INC-A4)
distinguish "safe to leave alone" (identical to the intact legacy source)
from "Axiom-owned content that must be exported on rollback" (axiom-native,
or migrated-then-modified).

## Scope

- `packages/workflow/src/artifact-store.ts`: extend `BaseArtifactMetadataFields`
  (shared by all 5 `ArtifactKind`s — increment/bug/plan/adr/decision) with 4
  new OPTIONAL fields (`origin`, `managedBy`, `lifecycle`, `exportPolicy`);
  parse/serialize support; the 4 `makeInitial*Metadata` factories gain an
  optional `origin` override (default: axiom-native) and derive
  `managedBy`/`lifecycle`/`exportPolicy` from it; `saveArtifactMetadata` gains
  an automatic `migrated` → `migrated-and-modified` transition on genuine
  edits (the SINGLE choke point every write path funnels through); new
  exported readers `resolveArtifactOrigin`/`resolveArtifactLifecycleState`
  (safe default: missing → axiom-native).
- `apps/cli/src/bootstrap-from-legacy-sdd/migrator.ts`: thread an `ArtifactOrigin`
  (`source: 'migrated'`, `originalPath`, `migratedAt`, and — when the caller
  opts in — `migrationId`/`repository`) into every migrated artifact via the
  factories' new `origin` option.
- NEW `apps/cli/src/bootstrap-shared/migration-manifest.ts`: the global,
  non-hidden `migration-manifest.yaml` (schema, load/save, `generateMigrationId`,
  a fresh-rescan `computeMigrationArtifactsSnapshot`, and the high-level
  `recordMigrationRun` entry point).
- `apps/cli/src/commands/bootstrap.ts`: `runBootstrapFromLegacySdd` mints a
  `migrationId`, threads it into the migration, and — only when new artifacts
  were actually created (same gate as the existing provenance-map save) —
  records the run + refreshes the manifest.
- Tests: new unit tests in `packages/workflow/tests/artifact-store.test.ts`,
  `apps/cli/tests/bootstrap-from-legacy-sdd/migrator.test.ts`, a new
  `apps/cli/tests/bootstrap-shared/migration-manifest.test.ts`, and a new e2e
  `apps/cli/tests/e2e/provenance-lifecycle-manifest.e2e.test.ts`.

## Non-goals

- The export/rollback command itself (INC-A4) — this increment only POPULATES
  the metadata/manifest `INC-A4` will read; it does not build the `axiom
  export`/`eject` command.
- The unified `<project>.axiom` MCP broker (INC-A5).
- Wiring the manifest into `bootstrap from-context` (technical-context DOCS,
  not `ArtifactKind` items — no `metadata.yml`/origin/lifecycle concept
  applies to them at all; see Assumptions).
- Migrating Axiom's OWN `Axiom.SDD`/`Axiom.Spec` repos to the managed model
  (explicitly deferred to the user, per the plan).
- Auto-refreshing the manifest on every single metadata edit (only on
  migration runs that create something new) — see Assumptions for the
  layering reason.

## Acceptance criteria

- [x] `metadata.yml` schema extended with `origin.source`
      (`'migrated'|'axiom-native'`), `origin.{migrationId,repository,
      originalPath,migratedAt}` (migrated-only), `managedBy.{tool,since,
      lastAxiomModificationAt?}`, `lifecycle.state`
      (`'migrated'|'migrated-and-modified'|'axiom-native'`), and
      `exportPolicy.{rollbackEligible,targetLegacyRepo?,targetLegacyPath?}`.
- [x] Newly-created (non-migrated) artifacts get `origin.source:
      'axiom-native'` + `lifecycle.state: 'axiom-native'` by default (no
      caller change required).
- [x] Migrated artifacts (from `bootstrap-from-legacy-sdd`'s migrator) get
      `origin.source: 'migrated'`, `lifecycle.state: 'migrated'`, and the
      migration fields populated.
- [x] When Axiom edits an artifact whose `lifecycle.state` is `migrated` (or
      already `migrated-and-modified`), it flips to
      `migrated-and-modified` and stamps `managedBy.lastAxiomModificationAt`
      — automatically, at the `saveArtifactMetadata` write path, with no
      separate command required.
- [x] A global, NON-hidden `<project>.axiom>/migration/migration-manifest.yaml`
      enumerates migration runs (`migrationId`, source repo, counts) and a
      fresh per-artifact origin/lifecycle summary (migrated vs axiom-native).
- [x] Backward compatible: `metadata.yml` missing the new fields still loads
      (`origin`/`lifecycle`/`managedBy`/`exportPolicy` all optional; readers
      default missing `origin`/`lifecycle` to axiom-native). Existing
      consumers (CLI list/state commands, MCP handlers, normalize/integrate)
      unaffected.
- [x] Unit tests: axiom-native default, migrated stamping, the
      migrated→migrated-and-modified transition (incl. a 2nd/3rd edit keeping
      the state and advancing the timestamp, and axiom-native/pre-existing
      artifacts NEVER flipping), manifest content, back-compat parsing
      (missing fields load; malformed `origin.source` rejected).
- [x] e2e: adopt a sandbox → migrated artifacts carry `origin.source:
      migrated` + the manifest lists them; create a new increment (real CLI
      `axiom-increment create`) → `axiom-native`; modify a migrated one (real
      CLI `external-ref add`) → `migrated-and-modified`.
- [x] `npm run build` (tsc -b) passes.
- [x] Targeted `vitest run` (`packages/workflow`, `apps/cli/tests` bootstrap/
      adoption/provenance/artifact-metadata/normalize/integrate/state
      surfaces) passes; classified pre-existing vs new (none new).
- [x] Guard-test lockstep: no existing test hard-asserted the exact
      `metadata.yml` key set (verified by search); the two pre-existing
      hand-written fixtures asserting BACK-COMPAT parsing (missing
      origin/lifecycle) were left as-is (they exercise exactly the back-compat
      path this increment must preserve).

## Open questions

None blocking — the brief's "FINAL decisions" section explicitly baked in
the field names/shape; the few remaining implementation-level choices are
resolved narrowly under Assumptions below (existing-precedent-first, per the
executor's guardrails).

## Assumptions

1. **Single choke point over per-command wiring.** The
   migrated→migrated-and-modified transition lives INSIDE
   `saveArtifactMetadata` (keyed off the ON-DISK pre-existing metadata: file
   already exists + its `origin.source === 'migrated'` + its
   `lifecycle.state` is `migrated`/`migrated-and-modified`), rather than
   being duplicated at each of the (at least six) call sites that write
   metadata (`axiom-increment`/`axiom-bug`'s create+transition sync,
   `link-plan`, `external-ref add`, `axiom normalize`, `axiom integrate`, and
   the migrator's own initial write). This is the only way to guarantee "do
   NOT require a separate command" — every current AND future write path
   inherits it for free. Verified safe: a fresh create (no file on disk yet)
   is never touched; an `axiom-native` artifact is never eligible; a
   pre-existing (pre-INC-A3) artifact with no `origin` at all defaults to
   axiom-native and is therefore never eligible either (safe, non-destructive
   default, never guessing "migrated").
2. **Timestamp source for `lastAxiomModificationAt`.** Reuses the incoming
   `metadata.updatedAt` (the timestamp the CALLER already threaded with its
   own clock for this exact write) rather than adding a new clock parameter
   to `saveArtifactMetadata` or invoking `new Date()` internally — this is
   the only "injectable clock the codebase already uses" available at this
   exact choke point, and it satisfies the brief's "never invent
   timestamps" guardrail. One accepted corner case: `axiom normalize`
   deliberately does NOT bump `updatedAt` (idempotency invariant, see
   `normalize.ts`'s own comment) — so a normalize-triggered flip stamps
   whatever `updatedAt` the artifact already had, not "now". Accepted as a
   minor imprecision (still a real timestamp from the artifact, never a
   synthetic clock read) rather than special-casing the rare
   administrative-normalize path.
3. **`managedBy`/`lifecycle`/`exportPolicy` are DERIVED, not
   independently settable by factory callers.** `makeInitial*Metadata`'s
   options gain only `origin` (optional); the other three fields are
   computed deterministically from the resolved origin
   (`buildOriginBundle`), preventing drift between them. `rollbackEligible`
   defaults: `axiom-native` → `true` (nothing in the legacy repo — must be
   exported to leave Axiom); `migrated` (untouched) → `false` (the intact
   legacy repo already has it, byte-identical — nothing new to export);
   flipping to `migrated-and-modified` (edit-time) forces `true` (content now
   diverges from the untouched legacy source).
4. **`migrationId`/`repository` are independent of the existing
   `provenance` idempotency option.** `migrateLegacyItems` gained a new,
   separate, optional `options.migration` (not nested under
   `options.provenance`) so origin-stamping and idempotency-tracking are
   orthogonal concerns; every migrated item still gets `origin.source:
   'migrated'` even when a caller only exercises the 3-arg back-compat form
   (pre-existing unit tests), just without `migrationId`/`repository`
   populated.
5. **Manifest scope: `bootstrap from-legacy-sdd` only, not
   `from-context`.** The manifest's fields (`origin.source`,
   `lifecycle.state`, per-artifact origin summary) are specifically about
   `ArtifactKind` items (increment/bug/plan/adr/decision, each with a
   `metadata.yml`). `bootstrap from-context` migrates technical-context
   markdown DOCS, which have no `metadata.yml`/origin/lifecycle concept at
   all (a wholly separate subsystem: `TechnicalContextIndex` +
   `writeGuardedFile`) — wiring it into this manifest would conflate two
   unrelated content models for no benefit, so it was left out.
6. **Manifest write gate mirrors the existing provenance-map gate.**
   `recordMigrationRun` is only invoked when `migrationResult.created.length
   > 0` (same condition `bootstrap.ts` already uses to gate `saveProvenance`)
   — a 100%-already-migrated re-run never creates/touches the manifest file,
   avoiding a `migrations[]` history entry for a no-op run.
7. **`artifacts[]`/`summary` are a FRESH rescan every time, never
   incremental.** Each `recordMigrationRun` call recomputes the full
   per-artifact snapshot via `listArtifacts` across all 5 kinds (using the
   same back-compat-defaulting readers exposed for this purpose), rather
   than being incrementally maintained — this makes the manifest
   self-healing (reflects archive-moves, edits, or later
   migrated-and-modified transitions as of the last migration run) with zero
   bookkeeping risk of drifting from on-disk truth. Accepted limitation: the
   snapshot is only refreshed AT migration time, not on every subsequent
   metadata edit elsewhere — auto-refreshing on every `saveArtifactMetadata`
   call would coz a layering inversion (`packages/workflow`, a lower-level
   package, would need to know about a CLI/apps-level manifest concept) for
   a concern the brief scopes to "the migration," not "every edit."
   (Deferred to a future explicit recompute, e.g. as part of INC-A4.)
8. **Manifest location mirrors `provenance.ts`'s own convention.**
   `migration/migration-manifest.yaml` is relative to the RESOLVED spec
   scope (same parameter shape as `PROVENANCE_REL_PATH`), so for an
   adopted/dedicated `<project>.axiom` repo (INC-A2, rel-path `.`) it lands
   exactly at `<project>.axiom/migration/migration-manifest.yaml` (the
   brief's literal path); for single-repo/self-hosted/dogfooding project
   layouts it lands at `<projectRoot>/axiom.spec/migration/
   migration-manifest.yaml` (same convention, harmless, never exercised by
   this plan's adoption flow).
9. **`migrationId` shape mirrors `generateArtifactId`'s convention** (prefix
   + UTC timestamp + 6-char random suffix) but is defined in
   `bootstrap-shared/migration-manifest.ts`, NOT added to `@axiom/workflow`'s
   closed `ArtifactKind` union — a migration run is an EVENT id, not an
   artifact kind.
10. **Origin/lifecycle fields live on `BaseArtifactMetadataFields`** (shared
    by all 5 kinds), not on `BaseArtifactMetadata` (increment/bug/plan only,
    sibling to the `integration` axis) — required because ADR/Decision are
    also migratable kinds and need the same fields.
11. **`artifact-skeleton.ts` is untouched.** It generates prose-document
    skeletons (README/sub-docs) and a separate, unrelated `plan.metadata.yml`
    operational template (role-split scheduling) — not the workflow
    `metadata.yml` this increment's schema extends. Read for context per the
    brief, but out of scope for edits.

## Implementation notes

Files changed (all in `Axiom/`, the product monorepo, except this spec):

- `packages/workflow/src/artifact-store.ts` — new types
  (`ArtifactOriginSource`, `ArtifactOrigin`, `ArtifactLifecycleState`,
  `ArtifactLifecycle`, `ArtifactManagedBy`, `ArtifactExportPolicy`);
  `BaseArtifactMetadataFields` gains the 4 optional fields; new
  parse/serialize helpers (`parseOrigin`/`parseLifecycle`/`parseManagedBy`/
  `parseExportPolicy`, wired into `parseBaseFields`; `originToYaml`/
  `managedByToYaml`/`lifecycleToYaml`/`exportPolicyToYaml`/
  `originFieldsToYaml`, wired into all 4 `toYamlObject` branches); new
  exported `resolveArtifactOrigin`/`resolveArtifactLifecycleState`; new
  internal `buildOriginBundle` wired into all 4 `makeInitial*Metadata`
  factories (each gains an optional `origin` in its options); new internal
  `applyLifecycleAutoTransition`, invoked from `saveArtifactMetadata`.
- `packages/workflow/src/index.ts` — barrel exports for the 6 new types +
  `resolveArtifactOrigin`/`resolveArtifactLifecycleState`.
- `apps/cli/src/bootstrap-from-legacy-sdd/migrator.ts` —
  `MigrateLegacyItemsOptions` gains optional `migration: { migrationId,
  legacyRepoPath }`; `buildInitialMetadata` threads an `ArtifactOrigin`
  through to all 5 factories; new `buildItemOrigin` builds it per item
  (`source: 'migrated'` always; `migrationId`/`repository` only when
  `options.migration` is present; `originalPath`/`migratedAt` always).
- NEW `apps/cli/src/bootstrap-shared/migration-manifest.ts` — the manifest
  module (types, `MIGRATION_MANIFEST_REL_PATH`, `generateMigrationId`,
  `loadMigrationManifest`/`saveMigrationManifest`, `upsertMigrationRun`,
  `computeMigrationArtifactsSnapshot`, `recordMigrationRun`).
- `apps/cli/src/commands/bootstrap.ts` — `runBootstrapFromLegacySdd` mints a
  `migrationId`, threads `options.migration` into `migrateLegacyItems`, and
  (gated on `created.length > 0`, same as the provenance-map save) calls
  `recordMigrationRun` + reports the manifest path.
- `packages/workflow/tests/artifact-store.test.ts` — new describe blocks:
  axiom-native default (all 4 factories + round-trip), migrated origin
  (factory + round-trip), `resolveArtifactOrigin`/
  `resolveArtifactLifecycleState` defaults, back-compat parsing (missing
  fields load; invalid `origin.source` rejected), and the
  migrated→migrated-and-modified transition (first edit, second edit keeps
  state + advances timestamp, axiom-native never flips, pre-INC-A3
  no-origin-at-all artifact never flips).
- `apps/cli/tests/bootstrap-from-legacy-sdd/migrator.test.ts` — new describe
  block: origin stamping with/without `options.migration`, and a contrast
  test for the axiom-native default.
- NEW `apps/cli/tests/bootstrap-shared/migration-manifest.test.ts` — id
  generation, path resolution, load robustness (absent/corrupt/malformed
  entries), save/load round-trip, upsert semantics, the artifacts-snapshot
  scan, and `recordMigrationRun` end-to-end (incl. accumulating multiple
  runs).
- NEW `apps/cli/tests/e2e/provenance-lifecycle-manifest.e2e.test.ts` —
  extends INC-A2's adoption-sandbox pattern: adopt → migrated artifacts
  carry `origin.source: migrated` + the manifest lists them; `axiom-increment
  create` → axiom-native; `external-ref add` on the migrated one → flips to
  migrated-and-modified + stamps `managedBy.lastAxiomModificationAt`;
  confirms the axiom-native artifact is unaffected.

## Validation

Ran from `C:\repos\Axiom Workspace\Axiom`:

- `npm run build` (`tsc -b`) — PASSED, clean, exit 0 (run twice, before and
  after the test additions).
- Targeted `npx vitest run` (`packages/workflow` + the bootstrap/adoption/
  provenance/artifact-metadata/normalize/integrate/state/adr/decision/index
  surfaces in `apps/cli/tests`, including all e2e in that scope): 40 files,
  396 tests passed (includes the new/updated test files).
- Broader targeted `npx vitest run packages/workflow apps/cli/tests` (the
  task's full required scope): 129 files, 1260 tests passed.
- Full-suite `npx vitest run` (all 290 files, 2929 tests) as an extra,
  broader regression check (this touched `artifact-store.ts`, a
  foundational module used across `mcp-tools`/`mcp-server`/`doctor`): 289/290
  files, 2928/2929 tests passed. One failure:
  `apps/cli/tests/context.test.ts > Scenario 2 … capabilities` — a 5000ms
  timeout under full-suite parallel load. Classified **pre-existing,
  unrelated**: re-run in isolation, all 12 tests in that file (including
  Scenario 2) pass in ~6.4s; nothing in that test touches `artifact-store.ts`,
  `migrator.ts`, or the new manifest module; this exact
  `context.test.ts`/`workspace-setup.test.ts` full-suite-load flakiness is
  independently documented in INC-A2's own validation report. No new
  failures.

## Result

Implemented. All acceptance criteria met. `metadata.yml` now carries
structured, rollback-grade origin/lifecycle metadata
(`origin`/`managedBy`/`lifecycle`/`exportPolicy`, all additive/optional for
back-compat) for all 5 artifact kinds; newly-created artifacts default to
`axiom-native`, migrated artifacts (from `bootstrap-from-legacy-sdd`) are
stamped `migrated` with their migration/source provenance, and ANY
subsequent Axiom write to a migrated artifact automatically flips it to
`migrated-and-modified` (stamping `managedBy.lastAxiomModificationAt`) at
the single `saveArtifactMetadata` choke point — no new command needed. A
new, non-hidden, global `<project>.axiom>/migration/migration-manifest.yaml`
enumerates migration runs plus a fresh per-artifact origin/lifecycle
summary, additive to (never replacing) the existing hidden
`.migration-provenance.yml` idempotency map. Deferred per Scope: the
export/rollback command itself (INC-A4) and the unified MCP broker (INC-A5).

## General spec integration

Integrado en la spec canónica al cierre del batch INC-20260724-* (integración
única de toda la tanda). Ficheros que recibieron el conocimiento de este
incremento:

- `03_Modelo_Operativo_y_Datos.md` — campos de `metadata.yml`
  (`origin`/`managedBy`/`lifecycle`/`exportPolicy`) + manifest global
  `<project>.axiom/migration/migration-manifest.yaml`.
- `04_Flujos_SDD_y_Ciclo_de_Vida.md` — transición automática
  `migrated → migrated-and-modified` al editar.
- `07_Gobierno_y_Seguridad.md` — provenance como trazabilidad de ownership
  para salir de Axiom.
- `08_Glosario.md` — `origin.source`, `lifecycle.state`,
  `exportPolicy.rollbackEligible`, manifest de migración.
- `01_Requisitos_Funcionales.md` — RF-AXM-041.
