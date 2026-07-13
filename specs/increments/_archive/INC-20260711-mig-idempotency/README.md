# Increment: Idempotent re-runs + source→artifact provenance map

Status: archived
Date: 2026-07-13
Type: New capability
Owner: migration-engineer

> Implements **Phase 4** of the parent design/plan
> `specs/increments/INC-20260711-spec-sdd-migration-plan/README.md`
> (§Proposed design (5) Idempotency, provenance, and multi-repo placement).
> Builds on the two SHIPPED phases: INC-MIG-SPEC (subject A — spec artifacts)
> and INC-MIG-CONTEXT (subject B — technical context).

## Goal

Make adoption **idempotent**. Today a second adoption run over the same
foreign repo DUPLICATES: `from-legacy-sdd` mints a fresh timestamped id every
time (subject A), and `from-context` only relies on a per-doc non-existence
gate (subject B). This increment records a **source→artifact provenance map**
so a re-run **skips** any source already migrated and reports it distinctly
("N already migrated, skipped"), for BOTH subjects.

## Scope

- A shared **provenance store** (`apps/cli/src/bootstrap-shared/provenance.ts`)
  that loads/saves a provenance map at the **resolved spec scope** and offers
  pure `has`/`record` helpers plus a robust (never-throw) loader.
- **Subject A** (`bootstrap-from-legacy-sdd/migrator.ts` +
  `runBootstrapFromLegacySdd`): before migrating each `ScannedLegacyItem`,
  compute its content hash + normalized source path; if already recorded (and
  its target still exists) → **skip** as already-migrated; else migrate (fresh
  id per Q-004) and **record** the mapping. The pre-existing collision-skip +
  `artifactExists` guard remains as the belt-and-suspenders fallback.
- **Subject B** (`bootstrap-from-context/map.ts` +
  `runBootstrapFromContext`): same skip-and-record, keyed by the ingested doc's
  normalized source path + content hash. The pre-existing non-existence gate
  (skip-if-exists, no clobber of human edits) remains as fallback.
- **Reporting**: both commands' output (and `--dry-run`) report
  `migrated: N` and `already-migrated (skipped): M` distinctly.

## Non-goals

- **No install-time adoption UX.** Wiring adoption into `axiom workspace setup`
  / the init wizard is **Phase 5**, out of scope here.
- **No new detector formats / conversion depth.** Those shipped in Phases 1-2.
- **No ID-reuse-based idempotency as the primary mechanism.** Q-004 resolves to
  a provenance map; a foreign Axiom-style id is preserved only opportunistically
  (format match + no collision).
- **No change to the safe-status / provenance-banner / dry-run / collision-skip
  guarantees** established by the prior phases — this is purely additive.

## Provenance model (Q-004 resolution — implemented exactly)

Idempotency is keyed by a **source→artifact provenance map**:

- **Key** = **normalized source path** (forward-slashed, relative to the scanned
  root) **+ content hash** (`sha256` hex of the source bytes — for subject A the
  scanned `item.body`, for subject B the ingested `doc.body`).
- **Default = mint FRESH Axiom ids** (keep `generateUniqueArtifactId`) and
  RECORD `source → {artifactId, artifactRelPath, contentHash, migratedAtUtc,
  artifactKind, subject}`.
- A **foreign Axiom-style id is preserved ONLY** if `item.childName` already
  matches Axiom's id format (`<PREFIX>-YYYYMMDD-HHMMSS-<6 base36>`) for the
  item's kind **AND** does not collide (`!artifactExists`); otherwise mint fresh
  + record.
- **On re-run, SKIP** any source whose (**path OR content-hash**) is already in
  the map for the same subject **AND** whose recorded target still exists on
  disk. If the recorded target is gone, the source is re-migrated (so deleting a
  migrated artifact recreates only that one).
- **Timestamp is threaded from the CLI layer** (`nowUtc`); no impure `now()` is
  invented in the migration core. Tests inject a fixed timestamp.

**Store location (documented):**
`<resolvedSpecScope>/technical-context/.migration-provenance.yml` — a single
dot-prefixed YAML covering BOTH subjects. It is NOT picked up by the
`technical-context` ingest, index-builder, or `spec.technicalContextIndexRead`
MCP surfaces (those only read `<category>/*.md` and `indexes/*.index.yml`).

**Shape:**

```yaml
schemaVersion: 1
entries:
  - sourcePathNormalized: "increments/INC-001-foo/README.md"
    contentHash: "<sha256-hex>"
    artifactKind: "increment"
    artifactId: "INC-20260713-101112-a1b2c3"
    artifactRelPath: "increments/INC-20260713-101112-a1b2c3"
    subject: "spec"          # spec | context
    migratedAtUtc: "2026-07-13T00:00:00.000Z"
```

**Robustness (corner case):** if the file is absent OR fails to parse OR has an
invalid shape, it is treated as an EMPTY map (never crashes) and behavior falls
back to the existing `artifactExists` (subject A) / non-existence (subject B)
guards — so a first run behaves exactly as the prior-phase suites expect, and
nothing duplicates silently on the deterministic-target subject B path.

## Acceptance (Phase 4)

- Running adoption **twice** over the same foreign repo → each artifact/doc
  created **exactly once**; the second run reports "N already migrated,
  skipped".
- **Deleting** one migrated artifact/doc and re-running → recreates **only that
  one** (its recorded target no longer exists → re-migrate; others skipped).
- **Source moved/renamed** between runs (same content at a new path) → skipped
  by content-hash match (no duplicate).
- **Provenance file corrupted/absent** → treated as empty; falls back to
  `artifactExists`/non-existence guards; never duplicates silently, never
  crashes.

## Result

**Shipped and GREEN.** All acceptance criteria + corner cases implemented and
covered by tests; the full gate is green.

**What was built:**
- `apps/cli/src/bootstrap-shared/provenance.ts` — the shared provenance store
  (pure `hashContent`/`normalizeSourcePath`/`findProvenanceMatches`/`hasProvenance`/
  `recordProvenance` + robust `loadProvenance`/`saveProvenance`). Location:
  `<specScope>/technical-context/.migration-provenance.yml`, keyed by normalized
  source path + sha256 content hash, one file for BOTH subjects. Absent/corrupt/
  invalid-shape → empty map, never throws.
- **Subject A** (`bootstrap-from-legacy-sdd/migrator.ts` +
  `runBootstrapFromLegacySdd`): `migrateLegacyItems` gained an optional
  `options` arg (`{ provenance: { specScopeAbsolutePath, legacyRoot, map }, nowUtc }`).
  Skip-already-migrated (path OR hash, live target) + record on create + Q-004
  `extractPreservableAxiomId`. Opt-in and additive: without options it is
  byte-identical to pre-Phase-4.
- **Subject B** (`runBootstrapFromContext`): loads the map, skips docs whose
  source is already migrated (live target), records on write; the pre-existing
  non-existence gate stays as fallback.
- **Reporting**: both commands (and `--dry-run`) print
  `migrated: N, already-migrated (skipped): M` and a distinct
  "Ya migrados, salteados" list. New result fields `alreadyMigratedCount`.
- Timestamp is threaded via `nowUtc` from the CLI layer; tests inject a fixed
  value. No impure `now()` in the migration core.

**Corner cases verified:** twice → exactly-once; delete-one → recreates only
that one (its provenance entry is upserted, not duplicated); moved/renamed
source → skipped by content-hash; corrupt/absent provenance → treated as empty,
falls back to existing guards.

**Gate:** `npm run build` ✓ · `npm test` ✓ **2706 passed** (2681 baseline + 25
new; 1 pre-existing command-level double-run test updated from the old
"re-runs duplicate" premise to Phase 4 exactly-once) · `npm run typecheck` ✓ ·
`npm run doctor` ✓ PASS (0 failures).

**Tests:** `apps/cli/tests/bootstrap-shared/provenance.test.ts`,
`apps/cli/tests/bootstrap-from-legacy-sdd/idempotency.test.ts`,
`apps/cli/tests/bootstrap-from-context/idempotency.test.ts`,
`apps/cli/tests/bootstrap-idempotency-e2e.test.ts`. A real CLI e2e (scratch)
confirmed exactly-once + delete-one-recreates-one on the shipped binary.

**Deferred:** install-time adoption UX (Phase 5).
