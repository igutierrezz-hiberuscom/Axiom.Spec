# Increment: schema reconciliation (P0-1)

Status: closed
Date: 2026-07-10

## Goal

Make `axiom mcp list/validate/repair/inventory` and `axiom toolchain
show/validate/add` work against the REAL committed
`Axiom/axiom.config/mcp-manifest.yaml` and the toolchain global
catalogue, without breaking `@axiom/doctor`'s existing checks
(TC-004/TC-005/TC-007) or any existing test suite.

## Context

Two canonical YAML config files had schemas that disagreed with
their readers — a mismatch invisible to the test suite because
existing unit tests use rich-shape synthetic fixtures instead of the
real committed artifacts:

1. `Axiom/axiom.config/mcp-manifest.yaml` ships the minimal shape
   `{id, server, projectBinding}` (schemaVersion 1, Spec 0024 Lote
   A / TC-007). `apps/cli/src/commands/mcp.ts`'s `isMcpEntryLike`
   required the rich shape `{id, displayName, capabilities[],
   installMode, projectBinding, readonly}` — so `axiom mcp list`
   failed live with "mcp-manifest.yaml no tiene la forma McpManifest
   esperada" (exit 1), while `@axiom/doctor`'s TC-007 (which has its
   own independent, already-tolerant inline parser) passed against
   the same file.
2. `Axiom/axiom.config/integrations.yaml` ships `{mcp_servers,
   allowed_tools}` (Spec 0013 PC-001, the MCP allowlist) — it never
   declared `axiomManagedDefaults.install` /
   `installationDecisions.optionalPostMvp`. But
   `apps/cli/src/commands/toolchain.ts`, `packages/doctor/src/checks.ts`,
   and `packages/components/src/loader.ts` all read the toolchain /
   components global catalogue from exactly those (absent) keys, so
   `axiom toolchain add --id serena` failed live with "El id 'serena'
   no existe en el catálogo global. IDs disponibles: []" (exit 1),
   and `@axiom/components`'s D1 catalog was silently empty.

## Scope

- `apps/cli/src/commands/mcp.ts`: relax `isMcpEntryLike`/
  `isMcpManifestLike` to accept the real minimal committed shape
  `{id, server?, projectBinding}` and derive rich-field defaults
  (`displayName` ← `id`, `capabilities` ← `[]`, `installMode` ←
  `'project-scoped'`, `readonly` ← `false`). Rich-shape entries
  (explicit fields) still parse unchanged (back-compat).
- New canonical file `axiom.config/toolchain-catalog.yaml`: dedicated
  toolchain/utility global catalogue (`{schemaVersion: 1, tools:
  [{id, kind, mvp: false}, ...]}`), shipped with the declared
  utility set: `serena`, `codegraph`, `graphify`, `engram`,
  `context7`, `rtk`, `caveman`, `autoskills` — all declared/optional,
  none required.
- `apps/cli/src/commands/toolchain.ts`: `loadGlobalCatalogue` reads
  the new dedicated file first; falls back to the legacy
  `integrations.yaml#axiomManagedDefaults` /
  `installationDecisions.optionalPostMvp` reading if the dedicated
  file is absent (back-compat for existing installs/tests).
  `deriveKindForId` extended to also recognise bare ids (`"serena"`)
  in addition to the existing project-scoped suffix convention
  (`"serena-this-project"`).
- `packages/doctor/src/checks.ts`: `readToolchainCatalogue` (feeds
  TC-004/TC-005) split into `readToolchainCatalogFile` (new dedicated
  source) + `readIntegrationsYamlCatalogue` (legacy, unchanged
  logic), tried in that order.
- `packages/components/src/loader.ts`: `loadComponentsCatalog` tries
  the dedicated `toolchain-catalog.yaml` first (deriving the
  `installList` from `tools[].id`); falls back to the existing
  `integrations.yaml` + `validateIntegrationsYaml` path unchanged if
  absent.
- Regression tests (vitest) that load the ACTUAL committed
  `mcp-manifest.yaml` / `toolchain-catalog.yaml` (not synthetic
  fixtures) in `apps/cli/tests/`, `packages/doctor/tests/`, and
  `packages/components/tests/` — the guard that would have caught
  this class of bug.

## Non-goals

- Do NOT remove or repurpose `integrations.yaml`'s MCP allowlist
  (Spec 0013 PC-001, `mcp_servers` / `allowed_tools`). It stays
  untouched.
- Do NOT unify `axiom components` (Spec 0019 D1, install/uninstall
  lifecycle with checkpoints) and `axiom toolchain` (Spec 0023) into
  one concept — they remain separate features. The change here is
  only about WHICH file `@axiom/components` reads its `installList`
  from, not about merging the two command surfaces.
- Do NOT enrich derived `McpEntry.readonly` from
  `integrations.yaml#mcp_servers[server].read_only` — kept simple
  (default `false`) to avoid a new cross-file coupling in `mcp.ts`;
  `readonly` is display/inventory-only, not used for any gating
  logic.
- No migration tooling: no real adopting projects exist yet that
  ship a rich-shape `mcp-manifest.yaml` or a populated
  `axiomManagedDefaults` in `integrations.yaml` that would need
  migrating — both old shapes continue to parse via the back-compat
  paths described above.

## Acceptance criteria

- [x] `axiom mcp list` and `axiom mcp validate` succeed (exit 0)
      against the real committed `mcp-manifest.yaml`.
- [x] `axiom toolchain add --id serena` succeeds (exit 0) and
      `axiom toolchain show` lists it afterwards, with a correctly
      derived `kind`.
- [x] `@axiom/doctor` TC-004/TC-005/TC-007 are not newly broken
      (verified via `doctor --json` diff against a pre-change
      baseline: identical summary, zero status changes).
- [x] New regression tests load the ACTUAL committed
      `mcp-manifest.yaml` / `toolchain-catalog.yaml` (not synthetic
      fixtures) and pass.
- [x] `npm run build` is clean.
- [x] `npx vitest run` (full suite) shows only the 1 known
      pre-existing failure (`packages/doctor/tests/agents.test.ts`
      TC-011, an unrelated stale `bundleHash` on `axiom-reviewer`);
      zero new failures.

## Open questions

None blocking.

## Assumptions

- No real adopting project ships a populated
  `axiomManagedDefaults`/`installationDecisions.optionalPostMvp` in
  `integrations.yaml` yet that this change could silently shadow —
  the new dedicated file is tried FIRST, so if a project already has
  a real `toolchain-catalog.yaml` (none do today) its content takes
  priority over the legacy keys, per the explicit design in the
  original brief.
- `axiom.config/toolchain.yaml` (the project's own installed-tools
  manifest, written by `toolchain add`) is a tracked config file, not
  gitignored `.axiom-state/`. The live validation for this increment
  created one transiently while probing `toolchain add --id serena`
  against the real repo, then removed it — it is not part of this
  increment's deliverable.

## Implementation notes

### Part 1 — `mcp-manifest.yaml` reader tolerance

`apps/cli/src/commands/mcp.ts`: replaced the strict `isMcpEntryLike`/
`isMcpManifestLike` guards (which required the full rich shape) with
`isRawMcpEntryLike`/`isRawMcpManifestLike` (tolerant: only `id` +
`projectBinding` required; `server`, `displayName`, `capabilities`,
`installMode`, `readonly` all optional but type-checked if present)
plus a new `normalizeMcpEntry`/`normalizeMcpManifest` pair that
derives the rich `McpEntry` the rest of the file (formatting,
inventory, repair) already depends on. Public types (`McpEntry`,
`McpManifest`) and every downstream consumer (`app-api.ts`,
`repair.ts`) are unchanged — only the internal parse path changed.

### Part 2 — toolchain catalog source

Created `Axiom/axiom.config/toolchain-catalog.yaml` as the new
canonical, dedicated source for the toolchain/utility catalogue.
Updated the three readers named in the brief:

- `toolchain.ts#loadGlobalCatalogue`: tries
  `loadToolchainCatalogFile` (new dedicated file) first; falls back
  to the exact previous `integrations.yaml` logic (renamed inline,
  behavior byte-for-byte identical) if the dedicated file is absent.
  This is what keeps the 5 existing `toolchain.test.ts` scenarios
  green — none of their tmp dirs write a `toolchain-catalog.yaml`, so
  they all exercise the (unchanged) fallback path.
- `packages/doctor/src/checks.ts#readToolchainCatalogue`: split into
  `readToolchainCatalogFile` (new) and `readIntegrationsYamlCatalogue`
  (previous body, renamed, unchanged), composed with `??`.
- `packages/components/src/loader.ts#loadComponentsCatalog`: tries
  `tryLoadToolchainCatalogInstallList` (new) first; falls back to the
  original `integrations.yaml` + `validateIntegrationsYaml` path
  (extracted into `buildCatalogFromInstallList` to avoid duplicating
  the `enforcementByProfile`/`required` derivation between the two
  sources). All 6 existing `loader.test.ts` scenarios only write
  `integrations.yaml` in their tmp dirs, so they exercise the
  unchanged fallback and stay green.

Also fixed `deriveKindForId` (`toolchain.ts`) to recognise bare ids
(`"engram"`, `"context7"`, `"rtk"`, `"caveman"`, `"autoskills"`) in
addition to the pre-existing project-scoped suffix convention
(`"engram-this-project"`). Without this fix, every catalog id without
a matching `-this-project`-style suffix would have silently fallen
through to the default `'code-intelligence'` kind — verified live
(`engram` → `library-context`, `caveman` → `output-optimizer` after
the fix, both previously would have been mis-derived as
`code-intelligence`).

### Part 3 — regression tests against REAL artifacts

Added 4 new test files, each loading the byte-for-byte real committed
YAML (no synthetic fixture):

- `apps/cli/tests/mcp-real-manifest.test.ts` — points `loadMcpManifest`/
  `runMcpList`/`runMcpValidate` directly at the real
  `axiom.config/mcp-manifest.yaml` (read-only, safe to reference
  directly).
- `apps/cli/tests/toolchain-catalog-real.test.ts` — copies the real
  `toolchain-catalog.yaml` content into a tmp project (since
  `runToolchainAdd` writes `toolchain.yaml` in its `cwd`, and must not
  mutate the real repo tree) and exercises `runToolchainAdd`/
  `runToolchainShow`.
- `packages/components/tests/loader-real-catalog.test.ts` — same
  copy-to-tmp pattern, exercises `loadComponentsCatalog`.
- `packages/doctor/tests/toolchain-catalog-real.test.ts` — same
  pattern, exercises `runToolchainP0CoverageCheck`/
  `runToolchainP1CoverageCheck`.

## Validation

- `npm run build` (`tsc -b`): clean, exit 0, both before and after
  the change.
- `npx vitest run` (full workspace suite):
  - Baseline (before this increment's code changes): 201 test files
    (200 passed / 1 failed), 2150 tests (2149 passed / 1 failed). The
    1 failure is `packages/doctor/tests/agents.test.ts` TC-011 (stale
    `axiom-reviewer` `bundleHash`), pre-existing and unrelated.
  - After (with the 4 new regression test files added): 205 test
    files (204 passed / 1 failed), 2159 tests (2158 passed / 1
    failed) — the same single pre-existing TC-011 failure, zero new
    failures, +9 new passing tests across the 4 new files.
- Live probes (from `Axiom/`, using the built
  `apps/cli/dist/index.js`):
  - `node apps/cli/dist/index.js mcp list` → exit 0, prints the
    `sdd`/`spec` table (previously: exit 1, "mcp-manifest.yaml no
    tiene la forma McpManifest esperada").
  - `node apps/cli/dist/index.js mcp validate` → exit 0, "✓ MCPs
    válidos. Proyecto activo: 'Axiom'. IDs bound: [sdd, spec]."
  - `node apps/cli/dist/index.js toolchain show` (before add) → exit
    0, "toolchain.yaml está vacío o no existe." (expected/unchanged —
    no tools added yet).
  - `node apps/cli/dist/index.js toolchain add --id serena` → exit 0,
    "✓ Tool 'serena' agregada al toolchain." (previously: exit 1,
    "El id 'serena' no existe en el catálogo global. IDs
    disponibles: [].").
  - `node apps/cli/dist/index.js toolchain show` (after add) → exit
    0, lists `serena` with `kind: code-intelligence`,
    `support: partial`, `mvp: no`, `state: absent`.
  - `node apps/cli/dist/index.js doctor --json`: summary identical
    before/after (`total: 55, passed: 41, failed: 1, warnings: 3,
    skipped: 10`); zero status changes across all 55 check ids
    (diffed programmatically) — no new doctor failures.
  - The transient `axiom.config/toolchain.yaml` created by the
    `toolchain add --id serena` live probe was removed afterwards
    (not part of the deliverable; see Assumptions).

## Result

Closed. Both bugs described in the brief are fixed at the root
(reader tolerance + a real dedicated canonical source, not a
one-off patch), with explicit back-compat fallbacks so no existing
test or hypothetical existing install breaks. All 3 originally-named
readers (`mcp.ts`, `toolchain.ts`, `packages/doctor/src/checks.ts`,
`packages/components/src/loader.ts`) were updated. New regression
tests against the real committed artifacts close the gap that let
this class of bug ship silently (rich-fixture tests passing while the
CLI failed live).

## General spec integration

Integrated into `Axiom.Spec/general-spec.md` (see "MCP manifest y
catálogo global de toolchain" section): documented
`axiom.config/mcp-manifest.yaml`'s real minimal shape and its
reader-side defaulting, the new `axiom.config/toolchain-catalog.yaml`
canonical file and its role as the primary source for the toolchain
global catalogue (with `integrations.yaml#axiomManagedDefaults` as a
back-compat fallback, not a duplicate source of truth), and the
`integrations.yaml` MCP allowlist (Spec 0013 PC-001) as a distinct,
unrelated concern that was not touched.
