# ADR-0031 — `cmm` replaces `graphify` and `codegraph` in the canonical provider set

> **Estado**: accepted, implemented 2026-07-24
> **Spec**: `Axiom.Spec/specs/increments/INC-20260724-cmm-replaces-graphify-codegraph/README.md`
> **Amends**: ADR-0021 (closed set of 7 canonical provider ids, "changeable only by ADR")

## Contexto

`packages/capability-model/src/constants.ts`'s `CANONICAL_PROVIDER_IDS` has been a
HARD-CODED, closed set since ADR-0021: `filesystem`, `axiom-gateway`, `serena`,
`codegraph`, `graphify`, `engram`, `generated-snapshots` (7 ids). `codegraph`
(`code.knowledgeGraph`) and `graphify` (`code.structureAnalysis`) were two
separate, real, wired `ProviderClient`s (`packages/providers/src/code-intel/
{codegraph,graphify}-client.ts`) covering related-but-split structural
code-intelligence concerns: a SQLite-backed knowledge graph (codegraph) and a
multi-source (code+docs) repository graph (graphify), each with its own local
index format, its own CLI, and its own fallback (`graphify → codegraph → serena`
per `providers.yaml`).

The user explicitly decided to consolidate both into a single external tool,
`codebase-memory-mcp` (`cmm`), as part of a larger evolution plan (see
`giggly-imagining-moore.md`, "Decisiones tomadas" §1 and "Cluster T → INC-T1"):
one structural/graph provider instead of two, with `serena` unchanged as the
symbolic (def/refs/rename) provider. Running two structural code-intel tools
side by side duplicated local index maintenance (two separate on-disk indexes
to keep in sync, two separate binaries to install) for capabilities
(`code.knowledgeGraph`, `code.structureAnalysis`) that a single tool can serve
coherently, with a clearer boundary against `serena` (symbolic navigation).

## Decisión

1. `codegraph` and `graphify` are REMOVED from `CANONICAL_PROVIDER_IDS` (and
   from every selectable/registered/routable surface — see the increment's
   file-level changes). They are no longer selectable via the TUI wizard,
   `axiom configure --providers`, or any provider registry.
2. `cmm` (`codebase-memory-mcp`) is ADDED to `CANONICAL_PROVIDER_IDS` as the
   sole structural code-intelligence provider, serving BOTH capabilities the
   two removed providers used to split: `code.knowledgeGraph` and
   `code.structureAnalysis` (graph queries, blast-radius/impact-style
   structural analysis, dependency and call/reference traces at the
   structural — not symbolic — level).
3. The canonical set shrinks from 7 to 6 ids: `filesystem`, `axiom-gateway`,
   `serena`, `cmm`, `engram`, `generated-snapshots`.
4. Fallback: `cmm → filesystem` (physical-path read/search/glob), matching the
   "fallback ALWAYS" rule already established for every other provider.
   `serena → filesystem` is unchanged.
5. Freshness: unlike `codegraph`/`graphify` (which had no freshness concept),
   `cmm` gets a minimal freshness check (`packages/providers/src/code-intel/
   cmm-freshness.ts`) backed by a small per-project sync-state marker
   (`<projectRoot>/.cmm/sync-state.json`), consulted before a structural
   capability call and used to trigger a best-effort auto-sync when the index
   is stale or has never been synced. This is on-demand/self-healing, never a
   mandatory git hook (see Non-goals).
6. `ProviderKind` gains `'structural-code-intel'` (cmm's kind); the two legacy
   kinds `'knowledge-graph'`/`'repository-graph'` are kept in the type/schema
   enums for backward-compatibility with historical fixtures/docs, but no
   provider in the canonical set uses them anymore.

## Alternativas consideradas

- **Keep `codegraph`+`graphify` selectable, add `cmm` as a third structural
  option.** Rejected: the user explicitly asked for a REPLACEMENT ("estos dos
  ya no tienen que estar como seleccionables"), not a superset. A 3-way
  structural split would also reintroduce the exact duplication problem this
  ADR resolves.
- **Keep the client files registered but drop them from the selectable/TUI
  surface only.** Considered as the increment's documented fallback if clean
  removal proved too invasive (see the increment's Scope boundaries). Not
  needed in practice: the codegraph/graphify surface was fully removed
  (`codegraph-client.ts`/`graphify-client.ts` deleted, `cmm-client.ts` added)
  because the actual blast radius was tractable (contained to
  `packages/providers/src/code-intel/*` plus data/test fixtures).
- **Give `cmm` the existing `'knowledge-graph'` kind (reuse, no new enum
  value).** Rejected in favor of a dedicated `'structural-code-intel'` kind:
  `cmm`'s scope is broader than a single knowledge-graph tool (it also serves
  `code.structureAnalysis`), and a distinct kind avoids implying it IS the old
  `codegraph` tool under a new name.

## Consecuencias

- Every guard test asserting the closed 7-provider set, the `codegraph`/
  `graphify` fallback chain, or their doctor/toolchain wiring needed a
  lockstep update to the new 6-provider set (see the increment's Implementation
  notes for the full file list). This was the highest-churn part of the
  increment.
- Any project that had `codegraph`/`graphify` explicitly enabled in
  `workspace.json#providers` (AB6 selection) keeps that string persisted
  (workspace.json is not schema-validated), but it is silently ignored by
  `buildProjectProviderRegistry`/`isCodeIntelProviderId` from this point on —
  no automatic migration is performed (out of scope; `axiom configure
  --providers cmm` re-selects the new id).
- `packages/toolchain`'s global catalog (`axiom.config/toolchain-catalog.yaml`)
  and its bundled default (`workspace-config-scaffold.ts`) drop `codegraph`/
  `graphify` entries in favor of one `cmm` entry.
- Future ADR required to change the set again (unchanged rule from ADR-0021).

## Referencias

- `packages/capability-model/src/constants.ts` (`CANONICAL_PROVIDER_IDS`).
- `packages/capability-model/src/schemas.ts` (`ProviderRegistrySchema`, closed
  to exactly 6 providers).
- `axiom.config/providers.yaml` (registry + fallback chains).
- `packages/providers/src/code-intel/cmm-client.ts` (the `cmm` `ProviderClient`).
- `packages/providers/src/code-intel/cmm-freshness.ts` (freshness/auto-sync).
- `Axiom.Spec/specs/increments/INC-20260724-cmm-replaces-graphify-codegraph/README.md`.
