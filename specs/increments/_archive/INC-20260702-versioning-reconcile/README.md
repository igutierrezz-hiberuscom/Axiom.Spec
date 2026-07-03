# Increment: Versioning reconcile (INC-20, Phase G)

Status: closed
Date: 2026-07-02

## Goal

Audit `@axiom/versioning` (`managed-state.ts`, `checkpoints.ts`,
`migrations.ts`, `upgrade.ts`, `version.ts`) in full, confirm the exact
on-disk shape it persists against source doc §16.2's `version.yml` shape,
confirm the "MVP migration `0.0.0 -> 0.1.0` with rollback" and
`--dry-run`/`--from-checkpoint`/`--target-version` claims directly, and
give an honest scope assessment for whether this increment is "confirm/
align an existing shape" (the roadmap's hypothesis) or something else now
that INC-01 through INC-19 have landed. This increment is audit-only: no
code is written.

## Context

Roadmap: `Axiom.Spec/specs/increments/INC-20260702-axiom-redesign-roadmap/
README.md`, Phase G, INC-20. INC-01 through INC-19 are closed.

Source doc (read directly, not summarized from the roadmap):
`C:\Users\igutierrezz\Downloads\axiom_decisiones_sesion_prompt_implementacion.md`
§16.1 (two distinct versions: global Axiom version vs. project installation
version), §16.2 (`version.yml` example shape), §16.3 (`axiom upgrade
global`, `axiom project upgrade --project X`), §16.4 (migration flow:
detect installed version -> detect pending migrations -> show diff ->
apply with backup -> validate -> report per-repo changes for commit).

## Scope

- Read `Axiom/packages/versioning/src/{managed-state,checkpoints,
  migrations,upgrade,version}.ts` in full, plus all four test files
  (`tests/{managed-state,checkpoints,migrations,upgrade}.test.ts`). Confirm
  the exact on-disk shape `managed-state.ts` persists.
- Produce a field-by-field diff: source doc §16.2's `version.yml` shape vs.
  the actual persisted `ManagedState` shape.
- Confirm directly (not re-trusted from the roadmap's own claim) that a
  real MVP migration `0.0.0 -> 0.1.0` exists with rollback support, and
  that `--dry-run`/`--from-checkpoint`/`--target-version` (plus
  `--no-sync`/`--no-doctor`) are real, wired CLI flags, not aspirational
  documentation.
- Check whether the redesign's other landed increments (registry v2's
  `schemaVersion`, `TechnicalContextIndex.schemaVersion`,
  `SkillsRoleIndex.schemaVersion`, `McpProjectConfig.schemaVersion`,
  `axiom.yaml`'s attempted-and-reverted `schemaVersion: 2`) create enough
  real per-asset-category versioned surface to justify reconciling against
  source doc §16.2's `assets: {sddSkillsVersion, adaptersVersion,
  guidesVersion, templatesVersion}` tracking now, or whether that
  granularity is premature.
- Give an honest scope assessment: is the real remaining work here
  "confirm/align an existing on-disk shape," or has most of it already
  been superseded/blocked by other increments' outcomes?

## Non-goals

- No code changes to `@axiom/versioning` or any other package.
- No schema-writer work in this increment (the roadmap's own sequencing
  makes schema-writer conditional on "only if a real shape gap is found";
  this increment's finding is that no such gap exists — see Result).
- No re-opening of INC-01's `axiom.yaml schemaVersion: 2` blocker; it is
  referenced here as evidence, not re-litigated.

## Acceptance criteria

- [x] `managed-state.ts`'s exact persisted shape is confirmed by direct
      read (field names, structure, types), not inferred from the
      roadmap's prior guess.
- [x] A field-by-field diff table exists comparing source doc §16.2's
      `version.yml` shape against the real `ManagedState` shape.
- [x] The MVP migration (`0.0.0 -> 0.1.0`) and rollback mechanism are
      confirmed by reading `migrations.ts`/`checkpoints.ts`/`upgrade.ts`
      directly, not re-trusted from the roadmap.
- [x] `--dry-run`/`--from-checkpoint`/`--target-version`/`--no-sync`/
      `--no-doctor` are confirmed as real, wired, tested CLI flags.
- [x] The `assets`-per-category versioning question is answered with
      concrete evidence (cites INC-01's `schemaVersion: 2` blocker as the
      controlling fact), not a generic "maybe premature" hedge.
- [x] A clear scope verdict is recorded: confirm-only vs. something else.
- [x] Recorded in `Axiom.Spec` per `Axiom.SDD/AGENTS.md`; no code changed
      in `Axiom.SDD` or `Axiom`.

## Open questions

None blocking. One non-blocking note carried into "Result": if/when the
redesign's `assets`-per-category tracking becomes wanted, it should be
scoped as a new, explicitly-additive increment once `axiom.yaml
schemaVersion: 2` actually ships (see Result for why this increment does
not propose it now).

## Assumptions

- The roadmap's own framing (`Axiom.Spec/specs/increments/
  INC-20260702-axiom-redesign-roadmap/README.md`, INC-20 entry) is the
  starting hypothesis to confirm or correct, not a settled fact.
- `Axiom/packages/versioning`'s `RUNTIME_VERSION` (`0.1.0`) is the "global
  Axiom version" analog referenced by source doc §16.1; there is currently
  no separate, distinct "project installation version" field in
  `ManagedState` beyond `runtime.version` itself (see diff below).

## Implementation notes

Audit-only. No files were changed under `Axiom/packages/`, `Axiom/apps/`,
or `Axiom.SDD/` for this increment.

### Confirmed on-disk shape (`ManagedState`, `managed-state.ts`)

Read in full: `Axiom/packages/versioning/src/managed-state.ts`,
`migrations.ts`, `checkpoints.ts`, `upgrade.ts`, `version.ts`, and all four
test files. The persisted shape (JSON, at
`<root>/.sdd/config/<projectName>/managed-state.json`, via the
`FilesystemStore`'s `config` scope) is:

```ts
interface ManagedState {
  schemaVersion: 1;
  runtime: {
    package: string;   // fixed: '@axiom/versioning'
    version: string;   // e.g. '0.1.0' — the ONE version field tracked
  };
  adapterTargets: {
    id: string;
    version: string;
    lastSyncedAt: string | null;
  }[];
  lastUpgrade: {
    fromVersion: string;
    toVersion: string;
    at: string;
    checkpointId: string | null;
  } | null;
  lastCheckpointId: string | null;
}
```

This is confirmed **structurally different** from source doc §16.2's
`version.yml`, not a close variant and not a naming-only difference.

### Field-by-field diff: source doc §16.2 `version.yml` vs. real `ManagedState`

| Source doc §16.2 field | Real `ManagedState` equivalent | Verdict |
|---|---|---|
| `axiom.projectContractVersion` (a *contract* version, distinct from runtime version) | **none** — no field represents "which contract version this project was installed against," only `runtime.version` | Missing concept, not just a missing field |
| `axiom.installedWithAxiomVersion` (frozen at install time) | **none** — `runtime.version` is mutated in place on every upgrade; no separate "as-installed" snapshot is kept | Missing concept |
| `axiom.lastUpgradedWithAxiomVersion` | `lastUpgrade.toVersion` (only populated after at least one upgrade; `null` before) | Partial analog — close in spirit, absent by default, not distinct from `runtime.version` after upgrade (both end up equal) |
| `assets.sddSkillsVersion` | **none** | Missing — no per-asset-category version tracked at all |
| `assets.adaptersVersion` | **none** | Missing |
| `assets.guidesVersion` | **none** | Missing |
| `assets.templatesVersion` | **none** | Missing |
| `appliedMigrations: [...]` (a list of every migration id ever applied, e.g. `0001-initial-contract`, `0002-add-mcp-manifest`) | **none** — `ManagedState` records only `lastUpgrade` (a single most-recent transition: `fromVersion`/`toVersion`/`at`/`checkpointId`), not a cumulative applied-migrations log | Missing — real gap: no way to answer "which migrations has this project ever received," only "what was the last upgrade" |
| (no equivalent in source doc) | `adapterTargets: [{id, version, lastSyncedAt}]` | Real, additive field the source doc does not describe — tracks per-adapter drift, not covered by §16.2 at all |
| (no equivalent in source doc) | `lastCheckpointId` | Real, additive field — points at the rollback mechanism (`checkpoints.ts`), which has no analog in the source doc's `version.yml` example at all |

**Summary of the diff**: this is not "additive/renaming only" as the
roadmap's initial hypothesis guessed. Four real concepts from §16.2 are
entirely absent (`projectContractVersion`, `installedWithAxiomVersion`,
the four `assets.*` fields, and `appliedMigrations` as a cumulative list).
Two real concepts exist in `ManagedState` with no analog in §16.2
(`adapterTargets`, `lastCheckpointId`) — evidence of the roadmap's own
observation that `@axiom/versioning` is "ahead" of the source doc in the
rollback/checkpoint dimension, while being genuinely behind it in the
per-asset-category and cumulative-migration-history dimensions.

### MVP migration and rollback: confirmed real, not aspirational

- `migrations.ts`: exactly one migration is registered,
  `migration_0_0_0_to_0_1_0` (`MIGRATIONS: ReadonlyArray<Migration>` has
  length 1). It is pure (`apply: (state, ctx) => Promise<ManagedState>`,
  never mutates its input), idempotent (re-running it over an
  already-migrated state preserves `adapterTargets`/`lastUpgrade`/
  `lastCheckpointId` rather than resetting them), and its final
  `runtime.version` stamp is applied once, centrally, by
  `applyMigrations` at the end of the pipeline — not per-migration. This
  matches the roadmap's claim exactly.
- `listApplicableMigrations(from, to)` walks the registry as a chain
  (`from` -> next matching `from` -> ... -> `to`), throws with a clear,
  actionable message on any gap or cycle, and `compareVersions` guards
  against a migration overshooting the requested target.
- `checkpoints.ts`: `createCheckpoint`/`restoreCheckpoint`/
  `pruneCheckpoints`/`listCheckpoints` are a complete, real snapshot/
  restore service — atomic restore (`<path>.restore.tmp` + `fs.rename`),
  ISO-timestamp+random-suffix checkpoint ids (Windows-safe, `:` replaced
  with `-`), skip-and-warn on malformed manifests (never aborts the whole
  list), default retention of 5 checkpoints via `pruneCheckpoints`.
- `upgrade.ts`'s `executeUpgrade` is genuinely rollback-first: it creates
  a checkpoint **before** applying migrations (or accepts an explicit
  `fromCheckpointId`), and any failure at any later step (migration apply,
  `saveManagedState`, injected `syncFn`, injected `doctorFn`) triggers
  `failWithRollback`, which restores the checkpoint and re-throws an
  `Error & { upgradeFailed: true }` — confirmed exactly by
  `upgrade.test.ts`'s two dedicated rollback tests (sync failure, doctor
  failure), both asserting the persisted state reverts to the pre-upgrade
  version.
- CLI flags: `apps/cli/src/commands/upgrade.ts` (`AD-04` in its own header
  comment) registers `--dry-run`, `--from-checkpoint <id>`,
  `--target-version <v>`, `--no-sync`, `--no-doctor` as real Commander
  options, each mapped 1:1 onto `runUpgrade`'s `UpgradeArgs`
  (`dryRun`/`fromCheckpoint`/`targetVersion`/`noSync`/`noDoctor`), which in
  turn maps onto `previewUpgrade`/`executeUpgrade`'s real parameters. Not
  stubs, not documentation-only — every flag has a corresponding branch in
  `runUpgrade` and a corresponding test exercising it
  (`upgrade.test.ts`'s `fromCheckpointId` tests explicitly verify no new
  checkpoint is created when an explicit one is supplied).
- All 36 tests across the 4 versioning test files pass (`vitest run
  packages/versioning`), confirmed by direct execution as part of this
  audit, not assumed from the presence of test files.

**Verdict**: the roadmap's claim ("a real MVP migration `0.0.0 -> 0.1.0`
already shipped with rollback support," "`axiom upgrade` already supports
`--dry-run`, `--from-checkpoint`, `--target-version`, rollback-first
design") is fully confirmed, field for field, by direct source and test
inspection. This part of the roadmap's hypothesis was accurate.

### The `assets`-per-category question: premature, not "already covered"

The task asked whether enough new versioned artifacts now exist (registry
v2's `schemaVersion`, `TechnicalContextIndex.schemaVersion`,
`SkillsRoleIndex.schemaVersion`, `McpProjectConfig.schemaVersion`) to
justify reconciling against source doc §16.2's `assets: {sddSkillsVersion,
adaptersVersion, guidesVersion, templatesVersion}` granularity.

Direct evidence says no, and for a specific, concrete reason — not a
generic "still mid-rollout" hedge:

- `Axiom/apps/cli/src/commands/init.ts`'s own `buildAxiomYaml` has a
  documented, reverted attempt at exactly this kind of version-surface
  expansion. Its inline comment (confirmed by direct read, lines ~207-239)
  states plainly: emitting `axiom.yaml schemaVersion: 2` for new projects
  was attempted in the `INC-20260702-registry-manifest-schema-v2-cli`
  increment and **reverted** because `@axiom/project-resolution`'s
  `resolveProject` only reads the v1 shape (`config.project.name`); a v2
  manifest makes `resolveProject` return `status: 'ambiguous'`, which
  cascades to break `axiom join`, `axiom projects join`, `axiom repo
  attach`, and three `axiom tui` modes. That increment
  (`Axiom.Spec/specs/increments/INC-20260702-registry-manifest-schema-v2-cli/README.md`)
  is still `Status: pending` today.
- The four `SchemaVersion`-bearing artifacts named in the task
  (`TechnicalContextIndex`, `SkillsRoleIndex`, `McpProjectConfig`,
  registry v2) are each **independent, narrow, additive** schema versions
  — none of them is the project's own top-level contract version. None of
  them addresses, or was designed to address, "which Axiom contract
  version is this whole project installed against," which is what source
  doc §16.2's `axiom.projectContractVersion` and `assets.*` fields are
  actually for.
- Building `assets: {sddSkillsVersion, adaptersVersion, guidesVersion,
  templatesVersion}` tracking today would mean inventing four new version
  counters with no real generator or consumer wired to bump them (no
  package currently stamps a discrete "skills version," "adapters
  version," "guides version," or "templates version" anywhere in the 26
  packages) — this would be exactly the kind of speculative, ungrounded
  metadata `Axiom.SDD/AGENTS.md`'s bootstrap limits warn against
  ("complex metadata systems," "deep autogenerated folder hierarchies").
- The correct trigger for revisiting this is explicit and already known:
  once `axiom.yaml schemaVersion: 2` actually ships (i.e. once
  `INC-20260702-registry-manifest-schema-v2-cli`'s blocker is resolved),
  a top-level project contract version becomes meaningful, and *then*
  deciding whether to also track per-asset-category versions underneath
  it is a grounded question with a real consumer. Today it is not.

## Validation

`No validation command was found. Performed best-effort validation by
inspecting changed files and checking consistency against the requested
behavior.`

Note: `Axiom/package.json` does define real validation commands (`npm run
build`, `npm test`, `npm run doctor`, `npm run typecheck`), and this
increment did run the relevant one directly as part of the audit:

```text
npx vitest run packages/versioning
```

Result: 4 test files, 36 tests, all passing (`migrations.test.ts`: 7,
`managed-state.test.ts`: 8, `checkpoints.test.ts`: 9, `upgrade.test.ts`:
12). This is real evidence for the "migration/rollback mechanism already
works" claim above, not just a documentation read.

Best-effort validation performed beyond the test run:
- Directly read all 5 source files in `Axiom/packages/versioning/src/`
  and all 4 test files in `Axiom/packages/versioning/tests/` in full
  (not excerpted).
- Cross-checked `Axiom/apps/cli/src/commands/upgrade.ts` end to end for
  the CLI flag wiring, confirming each flag maps to a real
  `UpgradeArgs`/`runUpgrade` branch, not just a Commander `.option()`
  declaration with no effect.
- Grepped the whole `Axiom` monorepo for source doc §16.2's literal field
  names (`projectContractVersion`, `installedWithAxiomVersion`,
  `appliedMigrations`, `sddSkillsVersion`) — zero matches anywhere in
  source, confirming the diff table above is not missing an
  already-renamed equivalent hiding under a different file.
- Read `Axiom/apps/cli/src/commands/init.ts`'s `buildAxiomYaml` and the
  `INC-20260702-registry-manifest-schema-v2-cli` increment spec directly
  to confirm the `schemaVersion: 2` blocker claim (`Status: pending`,
  confirmed by direct grep of its own file), rather than inferring it
  from the roadmap's summary alone.

## Result

The roadmap's INC-20 hypothesis was **half right**. It correctly predicted
that the migration/checkpoint/rollback *mechanism* is real, working, and
ahead of source doc §16 — fully confirmed here by direct source read and
by actually running all 36 tests. It was **not correct** that the
increment's work is limited to "confirming/aligning the on-disk shape" in
the sense of a near-identical shape needing a light touch-up: the real
diff is a structural gap, not a naming gap. Four real concepts from source
doc §16.2 do not exist anywhere in `ManagedState` today
(`projectContractVersion`, `installedWithAxiomVersion`, the four
`assets.*` fields, and `appliedMigrations` as a cumulative list — only the
single most-recent `lastUpgrade` is kept).

Despite that real gap, this increment closes with **no code changes**,
for a reason grounded in evidence rather than convenience: building the
missing fields today would be speculative. There is no live consumer for
`projectContractVersion`/`assets.*`/`appliedMigrations` anywhere in the 26
packages, and the one concrete artifact that would give
`projectContractVersion` real meaning — `axiom.yaml`'s own
`schemaVersion: 2` — was already attempted and explicitly reverted in
`INC-20260702-registry-manifest-schema-v2-cli` (still `Status: pending`)
because of a live, cascading breakage in `@axiom/project-resolution`.
Writing new version-tracking fields against a project-contract concept
that does not yet exist in a stable form would be exactly the kind of
premature, ungrounded schema `Axiom.SDD/AGENTS.md`'s bootstrap limits
warn against.

This mirrors the precedent set by INC-05/INC-12/INC-16/INC-17: a
migration-engineer audit that finds real, evidenced gaps but no actionable
next step yet, and closes clean with an explicit no-code rationale rather
than leaving a `pending` status with no forward motion.

**Honest scope verdict**: this increment's real scope was not "confirm/
align an existing on-disk shape" (the roadmap's framing) — it was "confirm
the migration engine is real and working (yes), and determine whether the
`version.yml`/`assets` shape gap is worth closing now (no, and here is the
specific blocking fact why)." The next real trigger for a schema-writer
pass on this package is `INC-20260702-registry-manifest-schema-v2-cli`
resolving its `resolveProject` blocker — not a standalone follow-up to
this increment.

## General spec integration

Integrated into `Axiom.Spec/general-spec.md` (new "Versioning
(`@axiom/versioning`)" section): the confirmed `ManagedState` shape, the
field-by-field diff verdict against source doc §16.2, and the explicit
dependency of any future `assets`-per-category work on
`INC-20260702-registry-manifest-schema-v2-cli`'s `schemaVersion: 2`
blocker being resolved first. This is stable, load-bearing knowledge any
future increment touching versioning needs, not implementation history.

## Next step recommendation

No new increment is opened by this audit. The next concrete trigger for
revisiting `@axiom/versioning`'s shape is
`INC-20260702-registry-manifest-schema-v2-cli` resolving its
`@axiom/project-resolution` `resolveProject` blocker (i.e. `axiom.yaml`
actually shipping `schemaVersion: 2`). Once that lands, a future
increment can reopen the `assets`-per-category and
`projectContractVersion`/`appliedMigrations` question with a real
contract version to anchor it to, using this increment's field-by-field
diff as its starting brief instead of re-auditing from scratch.

Separately, per the roadmap's own sequence, **INC-21** (migration engine
and upgrade commands' per-repo change reporting) and **INC-22**
(configure/upgrade/repair operations) remain the next roadmap items in
Phase G; neither is blocked by this increment's findings.
