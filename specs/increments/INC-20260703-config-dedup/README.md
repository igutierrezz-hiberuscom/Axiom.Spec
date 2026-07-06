# Increment: config duplication consolidation (dedup)

Status: closed
Date: 2026-07-03

## Goal

Make `axiom.yaml` the single source of truth for project identity + repo
map, and remove or derive redundant copies of that data WHERE SAFE, without
breaking `npm run build`, the full `npx vitest run` suite (beyond the 11
known pre-existing failures), or `@axiom/doctor`'s drift-detection
(especially GW-001).

## Context

Today the same data is written to more than one file:

1. `init.json` (`.axiom-state/<projectName>/init.json`) carries a
   `projectName` field that duplicates the directory name it already lives
   under (which itself is derived from `axiom.yaml` via `resolveProject`).
2. `axiom.config/topology.yaml` (repo map: `sddRepo`/`specRepo`/
   `roleCodeRepositories`/`assignments`) duplicates information that is
   already declared in `axiom.yaml#paths` (v2) — both describe "what repos
   make up this project and where do they live".
3. `init.json.profileTriple` (initial user choice) and
   `install-profile.json` (resolved install profile) both carry
   `overlay`/`target`-shaped data, but at different lifecycle stages.

## Scope

- Remove `projectName` from the `init.json` written by `axiom init`.
- Stop `axiom init` from writing a standalone `axiom.config/topology.yaml`
  for the `installed-multi-repo` layout (the only layout that used to write
  one). Rely on `@axiom/topology`'s existing derive-on-read fallback
  (`loadTopology` → `tryLoadTopologyHint` → `defaultInstalledMultiRepoManifest`)
  so `axiom.yaml` remains the only authored source for the repo map until a
  project opts into an explicit `topology.yaml` (e.g. via `axiom roles
  assign`, which still lazily materializes the file on first mutation).
- Update the `InitRecord`-shaped local interfaces in the CLI commands that
  read `init.json` (`configure.ts`, `context.ts`, `model.ts`, `sync.ts`,
  `audit.ts`) to drop the now-absent `projectName` field, so the type
  matches what is actually persisted.
- Update `apps/cli/tests/init.test.ts` fixtures/assertions accordingly.
- Update `Axiom.Spec/specs/03_Modelo_Operativo_y_Datos.md` to describe the
  consolidated model.

## Non-goals

- Do NOT unify `axiom.yaml#paths` and `TopologyManifest` into a single
  schema/type. `TopologyManifest` has structure (`assignments`,
  `roleCodeRepositories`, `qaLane`) that `axiom.yaml#paths` does not
  express, and multiple commands (`roles assign/remove`, `topology
  show/validate`) read/write `TopologyManifest` directly. Full schema
  unification is deferred (see "What was deferred" below).
- Do NOT touch `init.json.profileTriple` vs `install-profile.json`. These
  are intentionally different lifecycle stages (see Resolution below), not
  duplication.
- Do NOT change `GW-001`'s hashed inputs (`providers.yaml`, `profiles.yaml`,
  `install-profile.json`) — confirmed unrelated to this dedup by reading
  `collectGatewayDigests` in `packages/doctor/src/checks.ts`.
- Do NOT change the `single-repo` default manifest path (unaffected by this
  increment; it never involved `topology.yaml` I/O).
- No migration path needed: no real adopting projects exist yet.

## Acceptance criteria

- [x] `init.json` written by `axiom init` no longer has a `projectName`
      field (`profileTriple` + `createdAt` + `version` only).
- [x] `axiom init --layout installed-multi-repo` no longer writes
      `axiom.config/topology.yaml`.
- [x] `axiom topology show` / `axiom roles assign` / doctor's `TC-001`,
      `TC-003`, and other `loadTopology` consumers still behave correctly
      for a freshly-`init`-ed `installed-multi-repo` project (derived
      fallback manifest, verified by existing `topology.test.ts` Scenario
      2b which already covers this derivation path).
  - `axiom roles assign` still materializes `topology.yaml` on first write
      (unaffected — it already calls `loadTopology` then
      `writeTopologyYaml`).
- [x] No consumer treats `init.json.projectName` or a missing
      `topology.yaml` as authoritative/required in a way that breaks.
- [x] `npm run build` is clean.
- [x] `npx vitest run` (full) shows only the 11 known pre-existing
      failures; no new failures.
- [x] GW-001 and the manifest/config-hash-based doctor checks are
      unaffected (verified: they hash `providers.yaml`/`profiles.yaml`/
      `install-profile.json`, none of which changed).
- [x] `Axiom.Spec/specs/03_Modelo_Operativo_y_Datos.md` reflects the
      consolidated model.

## Open questions

None blocking. `Q3` (survival of `axiom.spec/` prefix) and other pre-existing
open questions from `03_Modelo_Operativo_y_Datos.md` are out of scope here.

## Assumptions

- No real adopting projects exist yet, so no migration/back-compat handling
  is required for projects that already have a written `topology.yaml` or
  an `init.json` with `projectName` — `loadTopology` already tolerates a
  present, valid `topology.yaml` (reads it verbatim) and `readInitJson`-style
  helpers never dereferenced `.projectName`, so old and new shapes coexist
  without special-casing.

## Implementation notes

### 1. `init.json.projectName` — REMOVED (safe win)

Traced every reader of `init.json` in `apps/cli/src/commands/`
(`configure.ts`, `context.ts`, `model.ts`, `sync.ts`, `audit.ts`,
`start.ts`, `init.ts` itself). All of them declare a local `InitRecord`-like
interface with a `projectName: string` field, but **none dereference
`.projectName`** on the parsed object — they all resolve the project
directory name from `ctx.projectName` (`@axiom/project-resolution`'s
`resolveProject(cwd).name`, read from `axiom.yaml`), passed in as a
parameter to locate `<root>/.axiom-state/<projectName>/init.json` itself.
`start.ts`'s inline type didn't even declare the field. This confirms the
field was dead data, not an authoritative source — safe to drop.

Changed `buildAxiomYaml`-adjacent `InitRecord` in
`apps/cli/src/commands/init.ts` to no longer include `projectName`, and
updated the five downstream local `InitRecord` interface copies
(`configure.ts`, `context.ts`, `model.ts`, `sync.ts`, `audit.ts`) to match
(drop the field so the type doesn't lie about what's on disk).

### 2. `topology.yaml` vs `axiom.yaml#paths` — PARTIALLY CONSOLIDATED (documented fallback, not full schema unification)

Investigated `@axiom/topology`'s `loadTopology` (`packages/topology/src/loader.ts`):
it ALREADY derives a fallback `TopologyManifest` from `axiom.yaml` when
`axiom.config/topology.yaml` is absent, via `tryLoadTopologyHint` (version
-aware: reads `schemaVersion: 2`'s top-level `name`/`mode`, or v1's
`project.name`/`project.mode`) → `defaultInstalledMultiRepoManifest`. Every
doctor check that inspects the repo map already goes through `loadTopology`
(confirmed by reading all 4 call sites in `packages/doctor/src/checks.ts`),
never reading the raw YAML file directly — so they transparently benefit
from the fallback. `axiom roles assign`/`remove` also call `loadTopology`
first and only write `topology.yaml` on the first actual mutation
(`writeTopologyYaml`), i.e. the file is lazily materialized, not required
upfront. `qa-archive-gate.ts` explicitly treats an absent `topology.yaml`
as "no warning" (tolerant).

Given this, the safe consolidation was: **stop `axiom init` from writing
`axiom.config/topology.yaml` for the `installed-multi-repo` layout** (it
was the only place that authored one from a fresh `init`). `axiom.yaml`
(with its `paths` map) becomes the only artifact `init` writes describing
the repo map; `topology.yaml` is now purely opt-in/derived, materialized
lazily only when a project actively assigns roles.

**What was deferred and why:** full schema unification (making
`TopologyManifest`'s shape a projection computed 1:1 from
`AxiomYamlConfigV2.paths` at every read, rather than a separate derived
default) was NOT done. `defaultInstalledMultiRepoManifest` still hardcodes
`ref: '../${projectName}.spec'` for `specRepo` instead of reading
`axiom.yaml#paths.specification.path` — this works today only because
`buildAxiomYaml` happens to hardcode the identical convention
independently. This is a parallel-but-consistent convention, not a
guaranteed-can't-diverge derivation. Making `tryLoadTopologyHint` read
`paths.specification.path` instead of re-deriving the convention would
touch `@axiom/topology`'s public default-manifest builders
(`defaultInstalledMultiRepoManifest`'s signature takes `projectName`, not
a `paths` map) and its test suite (`packages/topology/tests/topology.test.ts`,
11 tests including exact-shape assertions on `defaultInstalledMultiRepoManifest`
output) plus `apps/cli/tests/topology.test.ts` Scenario 2b's exact-shape
assertion. That is real, non-trivial surface for a schema-shape change with
no adopting project depending on it yet, so it is deferred rather than
risked in this lightweight increment. Documented here as a known future
consolidation opportunity, not scheduled to a specific increment.

`roleCodeRepositories`/`assignments`/`qaLane` have no `axiom.yaml`
equivalent at all (v2's `paths` only expresses `sddRepo`/`specRepo`-shaped
entries) — these fields are NOT candidates for full unification regardless,
confirming the guardrail decision to keep `TopologyManifest` as a
distinct, richer type rather than trying to fold it entirely into
`axiom.yaml`.

### 3. `init.json.profileTriple` vs `install-profile.json` — NOT duplication, left as-is

Confirmed by design intent already documented in the codebase
(`configure.ts`'s header comment, `context.ts`): `init.json.profileTriple`
records the user's INITIAL choice at `axiom init` time (functional
profile / overlay / target). `install-profile.json` is the FULLY RESOLVED
install profile computed later by `axiom configure` (enabled capabilities,
degraded-mode provider, gateway expectation, generated files, etc.) — a
strict superset derived from the triple plus `profiles.yaml`/
`capabilities.yaml`/`providers.yaml`. They answer different questions
("what did the user ask for" vs "what did that resolve to") and have
different lifecycles (`init.json` never changes after `init`;
`install-profile.json` is regenerated by `configure`/`context refresh`).
Removing either would lose information the other doesn't carry. No change.

### GW-001 coherence

`GW-001` (`packages/doctor/src/checks.ts`, `collectGatewayDigests`) hashes
exactly three files: `axiom.config/providers.yaml`,
`axiom.config/profiles.yaml`, and `<projectDir>/install-profile.json`.
None of those changed in this increment (only `init.json`'s shape and
whether `topology.yaml` gets authored by `init`), so `GW-001`'s
`configHash` computation and drift-detection semantics are unaffected —
verified by direct code reading, not just by running tests.

## Validation

- `npm run build` — clean, no errors (see closure report for verbatim tail).
- `npx vitest run` (full suite) — same 11 known pre-existing failures as
  documented in the increment prompt; zero new failures.

## Result

Implemented both safe consolidations (#1 `init.json.projectName` removal,
#2 stop authoring a duplicated `topology.yaml` from `init` for
`installed-multi-repo`, relying on the existing derive-on-read fallback).
Documented and deferred full `topology.yaml`/`axiom.yaml#paths` schema
unification with an explicit rationale (real, non-trivial blast radius on
`@axiom/topology`'s public builders and their tests, no adopting project
forcing the need yet). Documented #3 (`init.json.profileTriple` vs
`install-profile.json`) as intentional, not duplication — no change.

## General spec integration

Integrated into `Axiom.Spec/specs/03_Modelo_Operativo_y_Datos.md`: updated
the "Ficheros generados por comando" table (`init` no longer writes
`axiom.config/topology.yaml`), the `init.json` description (no
`projectName` field), and added a short note under the topology section
documenting `axiom.yaml` as the authoritative repo-map source with
`topology.yaml` as an opt-in/derived/lazily-materialized artifact, plus the
deferred full-unification note.
