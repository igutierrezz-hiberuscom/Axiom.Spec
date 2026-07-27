# Increment: Export/eject — leaving Axiom without touching legacy

Status: closed
Date: 2026-07-24

## Goal

Give users a safe way to "leave Axiom" without losing what Axiom created.
Since adoption/migration (INC-A2/A3) NEVER writes to legacy repos, rollback
from Axiom's perspective cannot mean "restore legacy" — legacy already has
whatever it always had. It means: stop using Axiom, and take WITH you
everything Axiom itself created or changed (axiom-native content, and
migrated content Axiom has since modified), so it can be manually
re-integrated into the user's own repo(s). This is **INC-A4** of Cluster A in
the Axiom evolution plan (`giggly-imagining-moore.md`, "Arquitectura
objetivo" / "Provenance: ... como legacy está intacto, el rollback es 'dejar
de usar Axiom' + exportar lo que Axiom creó"), building on INC-A1 (topology
`axiomRepo`/`legacyRepos`), INC-A2 (adoption creates a NEW `<project>.axiom`
repo, legacy untouched), and INC-A3 (`origin`/`lifecycle`/`managedBy`/
`exportPolicy` metadata + the global migration manifest). Explicit,
user-approved graduation to the full product lifecycle (per the plan) — this
feature is NOT down-scoped.

## Context

Before this increment, `axiom rollback` (`apps/cli/src/commands/rollback.ts`,
`INC-20260714-op-rollback-restore`) already existed but means something
completely unrelated: restoring `.axiom-state`/managed-state from an
`axiom upgrade` checkpoint. There was no command that read INC-A3's
`origin`/`lifecycle`/`exportPolicy` metadata to produce a dump of
Axiom-owned content for a user leaving the platform.

## Scope

- NEW `apps/cli/src/commands/eject.ts`: the `axiom eject` command — selector
  (`selectEjectableArtifacts`), report builder (`buildExportReport`),
  manifest builder (`buildExportManifest`), export-id generator
  (`generateExportId`), the physical folder-copy step, `runEject` (pure,
  commander-free), and `registerEject` (commander wrapper).
- `apps/cli/src/index.ts`: register `registerEject`, right after
  `registerBootstrap` (its semantic inverse) and before `registerRepair`.
- `packages/workflow/src/artifact-store.ts` + `src/index.ts`: one small,
  additive reader, `resolveArtifactRollbackEligible`, mirroring
  `resolveArtifactOrigin`/`resolveArtifactLifecycleState`'s exact pattern —
  reads the persisted `exportPolicy.rollbackEligible` when present, falling
  back to the same derivation formula `buildOriginBundle` already uses
  (`lifecycle.state !== 'migrated'`) for back-compat metadata missing
  `exportPolicy` entirely.
- Tests: `packages/workflow/tests/artifact-store.test.ts` (new describe
  block for the reader), NEW `apps/cli/tests/eject.test.ts` (unit —
  selector, dry-run, write mode, report/manifest shape, internal-artifacts
  exclusion, id generation), NEW `apps/cli/tests/e2e/eject.e2e.test.ts`
  (adopt → native create + migrated edit → eject dry-run + write, legacy
  byte-unchanged).

## Non-goals

- Touching or overloading the existing `axiom rollback` (upgrade-checkpoint
  restore) — a different command, a different name (`eject`), no shared code.
- The unified `<project>.axiom` MCP broker (INC-A5).
- Any `--apply-to-legacy` / automatic re-integration into a legacy repo — the
  export folder is the deliverable; re-integration is manual, out of scope.
- Migrating Axiom's OWN `Axiom.SDD`/`Axiom.Spec` repos to the managed model
  (deferred to the user, per the plan).
- Building new tracking machinery for `bootstrap from-context` (technical-
  context documents) — see Assumptions for why these are out of scope.

## Acceptance criteria

- [x] New command registered in `apps/cli/src/index.ts`; name does not
      collide with the existing `axiom rollback`.
- [x] Selector exports exactly the artifacts with
      `exportPolicy.rollbackEligible === true` (axiom-native +
      migrated-and-modified); excludes unmodified `migrated`; NEVER scans or
      copies internal config/state/skills/commands/hooks/capabilities/
      telemetry surfaces.
- [x] `--dry-run` is the default (zero writes); `--write-export-folder`
      performs the dump; if both are passed, `--dry-run` wins.
- [x] Export folder shape: `<project>.axiom/exports/<exportId>/
      {increments,bugs,plans,adr,decisions}/<id>/` (only kinds with ≥1
      included artifact) + `EXPORT_REPORT.md` + `export-manifest.yaml`.
      Never writes to any legacy repo (no `--apply-to-legacy`).
- [x] `EXPORT_REPORT.md` clearly lists Included and Excluded (with the
      excluded split into "migrated, unmodified" and "Axiom-internal
      surfaces, never exported").
- [x] Unit tests: selector picks only rollback-eligible; dry-run writes
      nothing; `--write-export-folder` produces folder + report + manifest
      with correct Included/Excluded; internal/clutter artifacts excluded
      (proven via a non-wholesale-copy assertion).
- [x] e2e: adopt a sandbox → create a native increment + modify a migrated
      one → `axiom eject --write-export-folder` → export contains the
      native + modified, excludes the unmodified-migrated artifact +
      internals; legacy repo byte-unchanged.
- [x] `npm run build` (tsc -b) passes.
- [x] Targeted `vitest run` (`apps/cli/tests`, `packages/workflow`) passes;
      classified pre-existing vs new (1 pre-existing flaky, 0 new failures).

## Open questions

None blocking. Two ambiguities in the brief's illustrative prose were
resolved narrowly (existing-precedent-first, per the executor's guardrails)
and are recorded under Assumptions: (1) whether "plan" (a 5th `ArtifactKind`
not named in the brief's Included/Excluded examples) is in scope, and (2)
whether "context" in the brief's phrasing means literal technical-context
documents or is loose phrasing for "artifact content".

## Assumptions

1. **Command name: `axiom eject`, not `axiom export axiom-native`.** The
   CLI's existing top-level vocabulary
   (init/join/configure/sync/start/audit/upgrade/rollback/repair/bootstrap)
   is uniformly single, short, imperative verbs. `eject` fits that shape and
   captures the Goal's own "leave Axiom" metaphor directly (eject a
   disc/drive before removing it), with zero collision risk against any
   existing or reasonably-foreseeable command — unlike a two-token phrase
   like "export axiom-native", which also risks later colliding with a
   plain, more general `axiom export ...` surface for something unrelated.
2. **Selector scope: all FIVE `ArtifactKind`s, including `plan`.** The
   brief's illustrative Included/Excluded lists only name
   increments/bugs/adr/decisions, but bullet 2 ("Selector") says "the
   artifacts" without a kind restriction and explicitly says "Use the A3
   helpers/manifest" — and INC-A3's own `computeMigrationArtifactsSnapshot`
   already scans all 5 kinds uniformly. Arbitrarily excluding `plan` would
   be an unjustified, undocumented inconsistency (a rollback-eligible plan
   silently never exported). Resolved in favor of full-kind coverage,
   per the brief's own "do NOT down-scope citing bootstrap limits"
   instruction.
3. **"context" in the brief's Included/export-folder wording is loose
   phrasing for "artifact content", NOT literal technical-context
   documents.** `bootstrap from-context` output carries NO
   origin/lifecycle/exportPolicy concept at all — INC-A3 explicitly scoped
   this out (its own Assumption #5: "a wholly separate subsystem ... no
   metadata.yml/origin/lifecycle concept applies to them at all"). There is
   no way to tell a migrated-untouched technical-context doc from a
   migrated-and-then-edited one without building NEW tracking machinery,
   which this increment's own Scope excludes too ("ONLY: the new
   export/eject command + selector + export folder/report/manifest +
   tests"). Resolved via existing-precedent-first: technical-context is
   excluded from the selector, exactly as INC-A3 itself excluded it from the
   migration manifest.
4. **New reader added to `@axiom/workflow` rather than re-deriving the
   formula locally or reusing `computeMigrationArtifactsSnapshot`.**
   `resolveArtifactRollbackEligible` reads the actual persisted
   `exportPolicy.rollbackEligible` field first (the brief literally names
   this field), falling back — only for back-compat metadata missing
   `exportPolicy` entirely — to the SAME derivation formula
   `buildOriginBundle` already uses. This is more faithful to the brief than
   deriving purely from `lifecycleState` (which `computeMigrationArtifactsSnapshot`
   would have made possible without touching `packages/workflow` at all,
   since today `exportPolicy.rollbackEligible` always equals
   `lifecycleState !== 'migrated'` per INC-A3's own invariant) — reading the
   field directly is more robust against any FUTURE divergence between the
   two. The brief's own validation section anticipates touching
   `packages/workflow`, confirming this is within scope.
5. **`generateExportId` duplicates `generateMigrationId`'s ~15-line body
   (new prefix) instead of generalizing the migration-manifest module.**
   Keeps this increment's diff scoped to ONLY the new export command,
   rather than modifying a sibling module's public API for a cosmetic
   reason (prefix parameterization).
6. **`eject.ts` is a plain `apps/cli`-owned command file, NOT added to
   `@axiom/cli-commands`'s single-ownership include list.** It is not
   consumed by the TUI (`@axiom/tui`) today, so it follows the SAME pattern
   as `bootstrap.ts`/`workspace-adopt.ts`/`axiom-increment.ts` (compiled
   directly by `apps/cli`'s own tsconfig, registered via a plain relative
   import in `index.ts`) rather than the smaller set of commands
   (`rollback`/`configure`/`sync`/`upgrade`/`model`/...) that
   `@axiom/cli-commands` compiles on `apps/cli`'s behalf for TUI reuse. This
   is also precisely why touching `rollback.ts` was never necessary or
   attempted.
7. **Physical copy is the FULL artifact instance folder** (`README.md` +
   `metadata.yml` + any sub-docs), preserving the exact
   `<kindFolder>/<id>/` relative path it had under the spec scope (so the
   export folder shape matches what a legacy repo would expect for manual
   re-integration). Uses `fs.cpSync` (Node ≥16.7 stdlib) — no existing
   recursive-copy utility was found in the repo to reuse (verified by
   search).
8. **An empty selection in write mode still produces a valid (near-empty)
   export folder + report/manifest** — each `eject` invocation is a fresh,
   self-contained folder (new `exportId` every time, never an accumulating
   list like the migration manifest), so there is no pollution concern in
   documenting "nothing was rollback-eligible yet".
9. **Archived artifacts (`<kindFolder>/_archive/<id>/`) are not
   included.** This is an existing, pre-existing limitation inherited from
   `listArtifacts` (which only scans one level deep and therefore already
   excludes archived artifacts from INC-A3's own manifest snapshot too) —
   out of scope to fix in this increment.
10. **Placement in `index.ts`:** `registerEject` is registered immediately
    after `registerBootstrap` — `eject` is bootstrap's semantic inverse
    (bootstrap brings legacy content IN; eject dumps Axiom's own content
    OUT) — and before `registerRepair`.

## Implementation notes

Files changed (all in `Axiom/`, the product monorepo, except this spec):

- `packages/workflow/src/artifact-store.ts` — new
  `resolveArtifactRollbackEligible(metadata)` reader, placed directly after
  `resolveArtifactLifecycleState`.
- `packages/workflow/src/index.ts` — barrel export for the new reader.
- `packages/workflow/tests/artifact-store.test.ts` — new
  `describe('INC-A4: resolveArtifactRollbackEligible', ...)` block: respects
  a persisted `exportPolicy` verbatim; derives from lifecycle state when
  `exportPolicy` is absent (axiom-native/migrated-and-modified → `true`,
  migrated → `false`); matches the real `exportPolicy` produced by
  `makeInitial*Metadata` (no drift).
- NEW `apps/cli/src/commands/eject.ts` — the whole command: types
  (`EjectArtifactSummary`, `EjectSelection`, `EjectArgs`, `EjectResult`,
  `ExportManifest`), `generateExportId`, `selectEjectableArtifacts`,
  `buildExportReport`, `buildExportManifest`, `copyArtifactIntoExport`
  (internal), `runEject`, `registerEject`. Extensive header comment
  documents the naming rationale, selector scope, and the deliberate
  technical-context exclusion.
- `apps/cli/src/index.ts` — imports and registers `registerEject`, right
  after `registerBootstrap`.
- NEW `apps/cli/tests/eject.test.ts` — `generateExportId` shape/determinism;
  `selectEjectableArtifacts` classification across all 5 kinds (axiom-native
  + migrated-and-modified → included; migrated-untouched → excluded);
  `runEject` project-not-resolved / no-spec-scope error paths; dry-run
  (default, explicit, and "both flags → dry-run wins"); write mode (export
  folder shape, report Included/Excluded ordering, manifest round-trip,
  empty-selection edge case, two non-clobbering runs); `buildExportReport`
  minimal-content check.
- NEW `apps/cli/tests/e2e/eject.e2e.test.ts` — extends INC-A2/A3's adoption-
  sandbox pattern: adopt (1 increment + 1 ADR migrated) → `axiom-increment
  create` (native) → `external-ref add` on the migrated increment
  (migrated-and-modified) → `axiom eject` dry-run (zero writes) →
  `--write-export-folder` (export contains native + modified increments,
  excludes the untouched ADR) → asserts the legacy sandbox tree and file
  contents are byte-for-byte unchanged throughout the entire flow.

## Validation

Ran from `C:\repos\Axiom Workspace\Axiom`:

- `npm run build` (`tsc -b`) — PASSED, clean, exit 0.
- Targeted `npx vitest run apps/cli/tests/eject.test.ts
  apps/cli/tests/e2e/eject.e2e.test.ts packages/workflow`: 17 files, 220
  tests passed.
- Full required scope `npx vitest run apps/cli/tests packages/workflow`:
  **131 files, 1278 tests passed, zero failures** (includes
  `context.test.ts` and `workspace-setup.test.ts`, both flagged as
  potentially flaky under full-suite load — both passed cleanly at this
  scope).
- Full-suite `npx vitest run` (all 292 files, 2947 tests) as an extra,
  broader regression check (this touched `artifact-store.ts`, a
  foundational module used across `mcp-tools`/`mcp-server`/`doctor`): 291/292
  files, 2946/2947 tests passed. One failure:
  `apps/cli/tests/context.test.ts > Scenario 2: runContextStatus con
  proyecto resuelto > exit 0; ... capabilities` — `Error: Test timed out in
  5000ms`. Classified **pre-existing, unrelated**: re-run in isolation, all
  12 tests in that file pass in ~1.1s; nothing in that test touches
  `artifact-store.ts` or `eject.ts`; this exact test/timeout is
  independently documented as known full-suite-parallel-load flakiness in
  both this increment's own brief and INC-A3's prior validation report
  (verbatim: "a 5000ms timeout under full-suite parallel load"). No new
  failures.

## Result

Implemented. All acceptance criteria met. `axiom eject` dumps every
rollback-eligible artifact (`axiom-native` + `migrated-and-modified`, across
all 5 `ArtifactKind`s) into a self-contained
`<project>.axiom/exports/<exportId>/` folder with an `EXPORT_REPORT.md`
(Included/Excluded, human-readable) and an `export-manifest.yaml`
(machine-readable), defaulting to a zero-write `--dry-run` report and never
writing to any legacy repo. The existing `axiom rollback` (upgrade-checkpoint
restore) is completely untouched — no shared code, no name collision. One
small, additive reader (`resolveArtifactRollbackEligible`) was added to
`@axiom/workflow` to expose the persisted `exportPolicy.rollbackEligible`
field (with a safe back-compat fallback), reusing INC-A3's metadata/manifest
foundation rather than re-deriving eligibility ad hoc.

## General spec integration

Integrado en la spec canónica al cierre del batch INC-20260724-* (integración
única de toda la tanda). Ficheros que recibieron el conocimiento de este
incremento:

- `04_Flujos_SDD_y_Ciclo_de_Vida.md` — flujo "Salida de Axiom: `axiom eject`"
  (selección rollback-eligible, dry-run default, nunca escribe legacy).
- `05_Interfaces_Operativas.md` — comando `axiom eject`
  (`--dry-run`/`--write-export-folder`), sin colisión con `axiom rollback`.
- `07_Gobierno_y_Seguridad.md` — `eject` nunca toca legacy; volcado de lo
  rollback-eligible.
- `08_Glosario.md` — `axiom eject` (con aviso de no confundir con
  `axiom rollback`).
- `01_Requisitos_Funcionales.md` — RF-AXM-042.
