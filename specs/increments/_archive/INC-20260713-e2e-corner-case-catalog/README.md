# INC-20260713 — E2E corner-case catalog + prioritized new e2e tests

This increment is **the catalog**: a per-area inventory of Axiom platform corner
cases, each with its **expected behavior**, its **current coverage** (unit /
integration / e2e / NONE + the exact file), and a **priority** (P0/P1/P2). It also
adds a **bounded set of new e2e tests** for the genuinely-uncovered P0/P1
cross-cutting gaps, and a prioritized **Known gaps to build** list for the owner.

The coverage column is not a guess: it is the result of a full read-only survey of
the existing suites this session (274 test files under `apps/cli/tests/**` and
`packages/*/tests/**`), across five areas run in parallel. Where a case is
`COVERED`, the exact `file::'test'` is named so the reader can jump to it.

## Legend

- **Coverage kinds** — `unit` (a single function/module), `integration` (a command
  or handler through its real I/O), `e2e` (a full chained flow, often spawning the
  built CLI), `NONE` (uncovered).
- **Priority** — `P0` (core correctness this session built; must not regress),
  `P1` (real gap/limitation worth building or covering), `P2` (edge/nice-to-have).
- Paths are relative to `Axiom/` unless noted.

---

## Area A — Adoption / migration

| ID | Corner case | Expected behavior | Coverage | Pri |
|----|-------------|-------------------|----------|-----|
| CC-ADOPT-01 | Nested `specs/` layout (artifacts under `<root>/specs`) | Descend once into `specs/` (then `spec/`) when no detector matched at root; root layout still wins | unit+e2e — `bootstrap-from-legacy-sdd/detectors.test.ts::'D-001: desciende a specs/…'`; `.../adopter-e2e.test.ts::'descends into specs/, migrates nested items…'` | P0 |
| CC-ADOPT-02 | Foreign format: openspec | Detect + convert `changes/*` → increment (front-matter status; `changes/archive/` → archived) | unit+integ — `detectors.test.ts` (openspec detector, 4 tests); `adopter-e2e.test.ts` | P0 |
| CC-ADOPT-03 | Foreign format: docs-adr / MADR | Detect `docs/adr` (and `doc/adr`); parse `## Status`; migrate as ADR | unit+integ — `detectors.test.ts` (docs-adr detector); `adopter-e2e.test.ts` | P0 |
| CC-ADOPT-04 | Foreign format: generic-folders | `proposals/rfcs/stories` → increment; `defects/issues` → bug | unit — `detectors.test.ts::'mapea proposals/rfcs/stories…'`; `convert.test.ts` | P1 |
| CC-ADOPT-05 | `e2e/` folder mapping | `e2e/<item>` → increment; migrated README carries `AXIOM:E2E-ORIGIN` | unit+e2e — `detectors.test.ts::'D-002: e2e/ mapea a increment…'`; marker asserted in `adopter-e2e.test.ts` **+ NEW `adoption-breadth.e2e.test.ts`** | P0 |
| CC-ADOPT-06 | Ignore registry/index files (kind-folder level) | Skip `REGISTRO_INCREMENTOS.md`, `README.md`, `INDEX.md`, `_index.md` (case-insensitive), report on `ignored` channel | unit+e2e — `detectors.test.ts::'D-003…'` (asserts REGISTRO+README); `INDEX.md`/`_index.md` by NAME were untested → **NEW `adoption-breadth.e2e.test.ts`** closes it | P1 |
| CC-ADOPT-07 | Mixed formats in one repo (openspec + docs-adr) | Union of non-overlapping detectors; both migrate | unit+e2e — `detectors.test.ts::'repo mixto…'`; `adopter-e2e.test.ts`; `e2e/workspace-adopt.e2e.test.ts` (openspec + MADR in one adopt) | P0 |
| CC-ADOPT-08 | Empty repo (nothing migrable) | Clean exit 0, "no migrable items" reported, zero writes | integ — `bootstrap.test.ts::'reporta que no se encontraron items migrables, exit 0'`; `workspace-adopt.test.ts` | P1 |
| CC-ADOPT-09 | Partially-Axiom repo (already-migrated untouched, only foreign migrated) | Existing artifacts/docs preserved; only foreign items migrated | integ — `workspace-adopt.test.ts::'no clobbea un technical-context doc pre-existente; sólo migra lo foráneo'`; `bootstrap-idempotency-e2e.test.ts` (delete-one/re-create-one) | P1 |
| CC-ADOPT-10 | Own pre-existing spec repo — no-clobber | Hand-written artifacts / `.axiom/mcp.yml` / foreign-project `axiom.yaml` preserved; reconcile + warn | integ — `workspace-adopt.test.ts` (3 no-clobber tests); `migrator.test.ts::'nunca sobreescribe un artefacto ya existente…'` | P0 |
| CC-ADOPT-11 | adopt-sdd conformance report | Report present/added/reconcile for `axiom.yaml`/topology/skills-index/`.axiom/mcp.yml` | integ — `workspace-adopt.test.ts::'reporta conformidad present/added/reconcile…'` (+ dry-run) | P1 |
| CC-ADOPT-12 | Provenance idempotent re-run | 2nd run creates 0; all skipped as already-migrated; stable map | unit+integ+e2e — `bootstrap-from-legacy-sdd/idempotency.test.ts`, `bootstrap-from-context/idempotency.test.ts`, `bootstrap-idempotency-e2e.test.ts`, `e2e/workspace-adopt.e2e.test.ts` | P0 |
| CC-ADOPT-13 | Provenance file corrupt / absent | `loadProvenance` → empty map, no throw; **run-level**: no crash + self-heal. **Limitation: provenance is the SOLE idempotency key → corruption re-migrates (duplicate)** | unit — `bootstrap-shared/provenance.test.ts` (ausente/corrupto/shape). Run-level was NONE → **NEW `adoption-breadth.e2e.test.ts`** (no-crash + self-heal + duplication locked/catalogued) | P1 |
| CC-ADOPT-14 | Source moved/renamed, content-hash match | Re-run skips by content hash (not path) | unit+integ — `provenance.test.ts::'matchea por HASH…'`; `idempotency.test.ts` (legacy-sdd + context) | P1 |
| CC-ADOPT-15 | Huge repo bounds (scan/size caps) | Bounded descend (one level, fixed subdir list); appendix cap `MAX_APPENDIX_CHARS` w/ `appendixTruncated` | **NONE** — cap constants exist (`convert.ts`, `detectors/registry.ts`) but no test triggers truncation/depth cap | P2 |
| CC-ADOPT-16 | from-context taxonomy recognition | Recognize `TECHNICAL_CONTEXT.md` (→ architecture) + all immediate subfolders as categories | unit+**e2e** — `bootstrap-from-context/ingest.test.ts`; **NEW `adoption-breadth.e2e.test.ts`** (KVP shape, MCP-queryable) | P0 |
| CC-ADOPT-17 | from-context unknown-category fallback | Unknown subfolder name → `references` (D-004 catch-all) | unit+e2e — `ingest.test.ts::'…subcarpetas arbitrarias…'`; **NEW `adoption-breadth.e2e.test.ts`** (`backend/` → references) | P1 |
| CC-ADOPT-18 | Unreadable doc continues | One bad doc recorded as failure, scan continues | unit — `bootstrap-from-context/ingest.test.ts::'registra un doc ilegible como failure y CONTINÚA…'`. (legacy-sdd read-error path not covered; only folder-with-no-`.md`) | P2 |
| CC-ADOPT-19 | Role-repo baseline-commit warning (no git mutation) | Warn on unborn/non-git role repo; ZERO git mutations | integ — `workspace-adopt.test.ts::'advierte sobre un repo de rol SIN commits…; sólo lee git'` | P1 |
| CC-ADOPT-20 | Preview reflects provenance map (no over-report) | Re-run dry-run predicts `would-migrate: 0` (== real run) | integ+e2e — `workspace-adopt.test.ts::'re-run PREVIEW…would-migrate: 0'`; `bootstrap-idempotency-e2e.test.ts` | P1 |

**Area A: 20 cases — 19 covered (some newly), 1 NONE (CC-ADOPT-15).**

---

## Area B — Spec-scope path

| ID | Corner case | Expected behavior | Coverage | Pri |
|----|-------------|-------------------|----------|-----|
| CC-SPEC-01 | Dedicated spec repo (`role: spec`, rel-path `.`) — CLI create | Artifact at `<scope>/increments` (no `axiom.spec`) | integ — `spec-scope-convergence.test.ts::'axiom-increment create writes to <scope>/increments…'` | P0 |
| CC-SPEC-02 | Self-hosted / single-repo (rel-path `axiom.spec`) | Unchanged — artifact at `<root>/axiom.spec/…` | integ — `spec-scope-convergence.test.ts::'regression: single-repo…'` | P0 |
| CC-SPEC-03 | CLI create + list + read converge (dedicated) | All resolve the same tree | integ — `spec-scope-convergence.test.ts` (create+list); `mcp-server/…spec-scope-convergence.test.ts` (read) | P0 |
| CC-SPEC-04 | Migrated artifact co-located with CLI-created + MCP-readable | Both under `<scope>/increments`, both listed | integ — `spec-scope-convergence.test.ts::'a migrated…artifact is co-located…'` **+ NEW `adoption-breadth.e2e.test.ts`** (migrated + listed + MCP loader) | P0 |
| CC-SPEC-05 | MCP `spec.incrementRead`/`planRead` on dedicated repo, DEFAULT args | Resolves scope-root artifact without explicit `specRelPath` | integ — `incrementRead` in `mcp-server/…spec-scope-convergence.test.ts`; `planRead`-on-dedicated was NONE → **NEW test** added to that file | P0 |
| CC-SPEC-06 | `spec.implementationContextRead` bundle on dedicated repo | Non-null `plan` + `relatedSpec` + non-empty `allowedWriteScope` at scope root | unit — `mcp-tools/implementation-context-handler.test.ts` (FIX-D, hand-crafted fixture). Through the REAL setup engine → **NEW `north-star-bundle.e2e.test.ts`** | P0 |
| CC-SPEC-07 | bundle populates repoSkills + technicalContext | `mandatory.repoSkills`/`technicalContext` non-empty | unit — `implementation-context-handler.test.ts::'…confidence "high"'` (non-dedicated fixture only). repoSkills on a DEDICATED repo via real setup → **NEW `north-star-bundle.e2e.test.ts`** | P0 |
| CC-SPEC-08 | Regression: self-hosted bundle reads `<root>/axiom.spec` | Byte-identical to pre-FIX-D | unit — `implementation-context-handler.test.ts::'REGRESSION: a non-"spec" role keeps the axiom.spec default'`; `mcp-server` regression test | P0 |

**Area B: 8 cases — all covered (5 strengthened by new tests).**

---

## Area C — Lifecycle

| ID | Corner case | Expected behavior | Coverage | Pri |
|----|-------------|-------------------|----------|-----|
| CC-LIFE-01 | Full increment lifecycle create→…→archive | State machine walks to `archived` | integ — `axiom-increment.test.ts::'completa el ciclo completo y deja state archived'` | P0 |
| CC-LIFE-02 | Bug lifecycle | create→fix-plan→verify→archive | integ — `axiom-bug.test.ts::'completa el ciclo completo…'` | P1 |
| CC-LIFE-03 | Plan role-split → one plan per role + targetRepos | Derives roles from `topology.yaml#roles`; per-role files + `allowedWriteScope` | integ — `axiom-plan-create.test.ts::'sin --roles: deriva backend/frontend…'` (+3) | P0 |
| CC-LIFE-04 | scaffold | dry-run preview; real create; no-clobber | integ — `scaffold.test.ts` | P2 |
| CC-LIFE-05 | normalize | canonicalize + idempotent; axis-split | integ — `normalize-cmd.test.ts` | P2 |
| CC-LIFE-06 | integrate | verifying→archived+integrated+move; refuses early | integ — `integrate.test.ts` | P1 |
| CC-LIFE-07 | validate-transition (legal/illegal) | accept legal; illegal → invalid-transition + legal list | integ — `validate-transition.test.ts` | P1 |
| CC-LIFE-08 | state-cmd | workflow + artifact + integration-axis read | integ — `state-cmd.test.ts` | P2 |
| CC-LIFE-09 | qa-e2e workflow | inline vs parallel start; pass→archived; fail→failed; archive-gate | integ — `axiom-qa-e2e.test.ts`, `qa-archive-gate.test.ts` | P1 |

**Area C: 9 cases — all covered.**

---

## Area D — Repo-affinity (all in `repo-affinity.test.ts`)

| ID | Corner case | Expected behavior | Coverage | Pri |
|----|-------------|-------------------|----------|-----|
| CC-AFF-01 | increment create from a role repo | REJECT (exit 1); allow from spec | integ — `::'increment create desde un repo de rol → REJECT'` | P0 |
| CC-AFF-02 | bug create from a role repo | REJECT | integ — `::'bug create desde un repo de rol → REJECT'` | P0 |
| CC-AFF-03 | plan create/approve from a role repo | REJECT | integ — `::'plan create…REJECT'`, `::'plan approve…REJECT'` | P0 |
| CC-AFF-04 | role command from the wrong role repo | REJECT (frontend from backend repo) | integ — `::'start del rol frontend desde el repo backend → REJECT'` | P0 |
| CC-AFF-05 | single-repo / self-hosted | no-op (allow even if "wrong" repo) | integ — `::'single-repo → allow (no-op)…'` (+v1, empty-assignments, skip variants) | P1 |
| CC-AFF-06 | unresolvable repo | allow (no-op) | integ — `::'sin axiom.yaml (no resoluble) → allow'` | P1 |

**Area D: 6 cases — all covered.**

---

## Area E — Cross-repo MCP + git

| ID | Corner case | Expected behavior | Coverage | Pri |
|----|-------------|-------------------|----------|-----|
| CC-XREPO-01 | Role reads plan-approved from spec via topology ref (NO local bindings) | Gate passes only by reading the spec repo; blocks when not approved | integ — `axiom-role-cross-repo.test.ts` AC1/AC2/AC2b (topology ref `./spec`, no bindings; proves no local plan state) | P0 |
| CC-XREPO-02 | `sdd.transitionApply` preview vs confirmed vs illegal | preview no-persist; confirmed persists; illegal → isError; foreign root rejected | integ — `mcp-server/transition-apply.test.ts`; `mcp-tools/transition-handlers.test.ts` | P0 |
| CC-XREPO-03 | `sdd.gitRoleBranch` preview/confirm/no-push | preview no-mutate; confirmed creates branch in real temp repo; never pushes | integ — `mcp-tools/git-handlers.test.ts`; `axiom-role-git.test.ts` | P0 |
| CC-XREPO-04 | `sdd.gitCommitSync` preview/confirm/no-push | preview; commit-local-no-push; push only with `push:true`+confirm | integ — `mcp-tools/git-handlers.test.ts` | P0 |
| CC-XREPO-05 | git tool on a non-git dir | typed error, no crash | integ (git-service layer) — `workflow/git-services.test.ts::'errors on a non-repo path'`. **At the mcp-tools git-handler layer: NONE** (handlers always use a repo-ready runner/`git init`) | P1 |
| CC-XREPO-06 | All MCP targets materialize native MCP config | 5 verified targets write their native shape; every repo gets it | integ+e2e — `native-mcp-config.test.ts`; `e2e/workspace-mcp.e2e.test.ts` Part A | P0 |

**Area E: 6 cases — 5 covered; CC-XREPO-05 partial (service layer only).**

---

## Area F — North-star flow (generated surfaces + code-intel)

| ID | Corner case | Expected behavior | Coverage | Pri |
|----|-------------|-------------------|----------|-----|
| CC-NS-01 | Per-adapter per-role surfaces generated | spec → author+planner; code → implementer; sdd → orchestrator | integ — `workspace-process-surfaces.test.ts`. **Note: sdd-orchestrator materialization only unit-mapped, not asserted materialized in control repo** | P0 |
| CC-NS-02 | AGENTS.md role-diff, NO raw `{placeholders}` | spec AGENTS.md ≠ code AGENTS.md; no `{role}`/`{projectId}` left | integ — `workspace-process-surfaces.test.ts` (`NO_RAW_PLACEHOLDER`, slice-diff) | P0 |
| CC-NS-03 | skills-index / repoSkills non-empty | implementer registered in `skills-index/<role>.yaml` (mandatory) | integ — `workspace-process-surfaces.test.ts`; **+ NEW `north-star-bundle.e2e.test.ts`** (flows into the bundle) | P0 |
| CC-NS-04 | opencode `skills-lock.yaml installed` non-empty | contains process skills | integ — `workspace-process-surfaces.test.ts` (`not.toMatch(/installed:\s*\[\s*\]/)`) | P1 |
| CC-NS-05 | code-intel enabled (serena/codegraph pinned) | native MCP config gets serena+codegraph pinned to repo path (code repos only) | integ+e2e — `workspace-mcp.test.ts` (unit + `runWorkspaceSetup` e2e) | P0 |
| CC-NS-06 | code-intel disabled (default) | only sdd/spec brokers (byte-identical); implementer body advertises physical-path fallback | integ — `workspace-mcp.test.ts` (no-regression); `workspace-process-surfaces.test.ts` | P0 |
| CC-NS-07 | Adapter breadth | claude-code+opencode FULL; cursor/gh-copilot/copilot-vscode instruction files; portable `.axiom/` everywhere | integ+e2e — `workspace-process-surfaces.test.ts`, `workspace-adapters.test.ts`, `e2e/adapters.e2e.test.ts` | P0 |
| CC-NS-08 | Implementer surface references the WHAT bundle | body cites `spec.implementationContextRead` + repo-local skills + code-intel fallback | integ — `workspace-process-surfaces.test.ts` | P0 |

**Area F: 8 cases — all covered (CC-NS-03 strengthened; CC-NS-01 has a materialization sub-gap).**

---

## Area G — Functional verify (all in `functional-verify.test.ts` unless noted)

| ID | Corner case | Expected behavior | Coverage | Pri |
|----|-------------|-------------------|----------|-----|
| CC-VERIFY-01 | Discovery order | README hints > package scripts > task-runner | integ — `discoverValidation` (3 tests) | P1 |
| CC-VERIFY-02 | No-command fallback (exact statement) | Emit the exact `NO_VALIDATION_COMMAND_FALLBACK`, non-blocking | integ — `discoverValidation::'no command…exact AGENTS.md fallback'`, `runFunctionalVerify::'…exact fallback statement'` | P1 |
| CC-VERIFY-03 | Multi-repo aggregation | per-repo results aggregated; one fails → blocked | integ — `runIncrementFunctionalVerify (aggregate)::'one repo fails → blocked…'` | P1 |
| CC-VERIFY-04 | Unresolvable repo reported (never silently skipped) | reported unresolved; does NOT block | integ — `resolveIncrementVerifyRepos::'…reported unresolved…'`, `runIncrementFunctionalVerify::'unresolved…does NOT block'` | P1 |
| CC-VERIFY-05 | Blocks on failure | transition BLOCKED (exit 1), stays plan-approved; role complete blocked | integ — increment verify + role complete + qa-e2e verify | P0 |
| CC-VERIFY-06 | `--no-verify` / `--force` bypass | `--no-verify` proceeds with warning | integ (`--no-verify` covered). **`--force` verify bypass: NONE** (only `--no-verify`/`--preview` exist) | P1 |

**Area G: 6 cases — 5 covered; `--force` bypass NONE (may not exist as a flag).**

---

## Area H — Versioning / upgrade

| ID | Corner case | Expected behavior | Coverage | Pri |
|----|-------------|-------------------|----------|-----|
| CC-VER-01 | Dry-run zero-write | `previewUpgrade` read-only, returns plan/`isNoOp` | unit — `versioning/upgrade.test.ts` (asserts plan shape; no explicit fs-unchanged assert) | P1 |
| CC-VER-02 | Checkpoint + migrate + persist | checkpoint created; `runtime.version` = target | unit — `versioning/upgrade.test.ts::'corre la migración 0.0.0 → 0.1.0, persiste…'` | P0 |
| CC-VER-03 | Older → current migration | 0.0.0 → 0.1.0 stamped | unit — `versioning/migrations.test.ts`, `upgrade.test.ts` | P0 |
| CC-VER-04 | Rollback-on-failure | failing post-step restores checkpoint + rethrows `upgradeFailed` | unit — `versioning/upgrade.test.ts` (via `syncFn`/`doctorFn` fail). **A migration-fn itself throwing is NOT tested** | P1 |
| CC-VER-05 | **Operator-invokable restore** of a pre-upgrade checkpoint | (desired) a first-class `axiom` command restores a checkpoint | **NONE — GAP.** `restoreCheckpoint` exists + unit-tested but no CLI invokes it; `--from-checkpoint` only reuses a rollback point | **P1 (build)** |
| CC-VER-06 | self-update preview no-mutation | preview prints, no install, suggests `--apply` | integ — `self-update.test.ts::'…sin --apply → preview puro…'` | P1 |
| CC-VER-07 | Upgrade idempotent (re-run no-op) | 0 migrations but still checkpoints | unit — `versioning/upgrade.test.ts::'idempotente…'` | P1 |
| CC-VER-08 | init.json gate | missing → `missing-init-json`; present → pass | unit (gate config + shared precondition) — `orchestrator/state-machine.test.ts`, `gates.test.ts`. **Driven through the REAL `runUpgrade` → NEW `adopt-upgrade.e2e.test.ts`** | P0 |
| CC-VER-09 | Per-repo scope of upgrade | one resolved project (cwd); no multi-repo fan-out | **LIMITATION** — `runUpgrade` runs `withProjectContext(cwd)` on one repo; no fan-out exists/tested | **P1 (build)** |
| CC-VER-10 | Checkpoint churn | keep N most recent, prune rest | unit — `versioning/checkpoints.test.ts` (keep=2/4; default 5) | P2 |
| CC-VER-11 | Upgrade on an ADOPTED project (default flags) | init.json materialized + sync bundled-template fallback → upgrade completes | integ (split) — `workspace-setup.test.ts` (FIX-E init.json) + `sync.test.ts` Scenario 7/7b (bundled). **The adopt→upgrade chain → NEW `adopt-upgrade.e2e.test.ts`** | P0 |
| CC-VER-12 | Upgrade error-line formatting | no duplicated `gateFailure:`/`upgradeFailed:` prefix | unit — `apps/cli/tests/upgrade.test.ts` | P2 |

**Area H: 12 cases — 10 covered (2 strengthened), 2 GAPS to build (CC-VER-05, CC-VER-09).**

---

## Area I — Isolation

| ID | Corner case | Expected behavior | Coverage | Pri |
|----|-------------|-------------------|----------|-----|
| CC-ISO-01 | Per-project state isolation | project name in isolated paths; two projects don't collide | unit — `isolation/p0.test.ts`; `persistence/filesystem-store.test.ts::'…dos proyectos no se ven entre sí'` | P0 |
| CC-ISO-02 | Hermetic home (`homeDirOverride`) | registry lands in tmp home; real home untouched | integ — `init.test.ts::'con homeDirOverride…NO toca el home real'` (+ projects/self-update) | P0 |
| CC-ISO-03 | Cross-project access blocked (persistence guard) | `assertCrossProjectBlocked` throws outside scope; path-traversal blocked | unit — `persistence/filesystem-store.test.ts` (`assertCrossProjectBlocked`, traversal, `validateKey`); `isolation/p0.test.ts`. **"TC-008" is a doctor memory check (`doctor/tests/memory.test.ts`), not a persistence test name** | P0 |
| CC-ISO-04 | Toolchain markers convention-only | a marker dir is a declaration, never proof of install (`status: 'marker'`; unverified → warning) | integ — `toolchain.test.ts` Scenario 2 (`marker`, `required-tool-not-verified`) | P1 (known) |

**Area I: 4 cases — all covered; toolchain-install-is-declare confirmed as a documented convention.**

---

## Area J — Adapters / misc

| ID | Corner case | Expected behavior | Coverage | Pri |
|----|-------------|-------------------|----------|-----|
| CC-ADP-01 | All adapter targets materialize native MCP | Only the **5 verified** targets emit native config; other 3 write nothing + warn | integ — `native-mcp-config.test.ts` (`NATIVE_MCP_TARGETS` = 5; unsupported → no file + warn) | P0 |
| CC-ADP-02 | `member install` surface breadth | emits native MCP (brokers) + toolchain activation; **does NOT emit code-intel MCP nor process-surfaces** | integ (positive path) — `member-install.test.ts`. **The absence is UNGUARDED** (KNOWN GAP; source-confirmed in `member-install.ts`) | **P1 (build/guard)** |
| CC-ADP-03 | `adapter add` (`runAdapterAdd`) surface breadth | updates workspace.json + AGENTS.md across repos; **omits process-surfaces + code-intel** (unlike `repo add`/`role add`) | integ (positive path) — `workspace-incremental.test.ts`. **The gap is UNGUARDED** (KNOWN GAP; source-confirmed) | **P1 (build/guard)** |
| CC-ADP-04 | `configure.ts` near-duplicate generator call | (finding) `configure` may call the surface generator twice | **NONE** — flagged this session; not covered | P2 |
| CC-ADP-05 | model-routing projections per adapter | opencode multi-mode (7 slots); claude-code single-mode fallback | integ+e2e — `e2e/adapters.e2e.test.ts` Part B | P2 |

**Area J: 5 cases — 3 partially covered (positive path only), 2 NONE/gap.**

---

## Catalog totals

| Area | Cases | Covered | NONE / gap |
|------|-------|---------|-----------|
| A — Adoption/migration | 20 | 19 | 1 (huge-repo bounds) |
| B — Spec-scope | 8 | 8 | 0 |
| C — Lifecycle | 9 | 9 | 0 |
| D — Repo-affinity | 6 | 6 | 0 |
| E — Cross-repo MCP + git | 6 | 5 | 1 partial (non-git at mcp-tools layer) |
| F — North-star | 8 | 8 | 0 (1 materialization sub-gap) |
| G — Functional verify | 6 | 5 | 1 (`--force` bypass) |
| H — Versioning/upgrade | 12 | 10 | 2 gaps to build |
| I — Isolation | 4 | 4 | 0 |
| J — Adapters/misc | 5 | 3 | 2 gaps |
| **Total** | **84** | **77** | **7** |

Coverage is high because this session shipped and tested FIX-A..E + NS-1..3
extensively. The remaining uncovered items are mostly **product gaps to build**
(rollback restore, upgrade fan-out, member-install / adapter-add surface parity)
rather than test omissions.

---

## New e2e tests added

Six new test cases (all verified green in isolation), added only for
genuinely-uncovered P0/P1 **cross-cutting** gaps. Each was checked against the
survey to confirm it is not a duplicate.

1. **`apps/cli/tests/e2e/north-star-bundle.e2e.test.ts`** (1 test, P0) — chains
   FIX-A + NS-2 + FIX-D through the REAL engine: `runWorkspaceSetup` (dedicated
   multi-repo, register + generated surfaces) → CLI `increment create` + `plan
   create` + `link-plan` in the dedicated spec repo → `buildImplementationContext`
   with **default args**. Proves the composite bundle resolves `plan`/`relatedSpec`
   at the scope ROOT (no `axiom.spec`), a non-empty `allowedWriteScope`, AND that
   the repo skills the real setup wrote (`axiom-role-implementer`) flow into
   `mandatory.repoSkills`. The existing FIX-D unit test hand-crafts the registry and
   does not assert repoSkills on a dedicated repo — this closes that end-to-end.
2. **`apps/cli/tests/e2e/adoption-breadth.e2e.test.ts` → "corrupt provenance"**
   (1 test, P1) — drives a full `runBootstrapFromLegacySdd` re-run against a
   **corrupt** `.migration-provenance.yml`. Proves the run does NOT crash and
   SELF-HEALS the file to a valid map, and LOCKS IN the catalogued limitation that
   provenance is the sole idempotency key (corruption → re-migration/duplicate).
   Previously only `loadProvenance` was unit-tested in isolation.
3. **`apps/cli/tests/e2e/adoption-breadth.e2e.test.ts` → "absent provenance"**
   (1 test, P1) — same run-level robustness for a **deleted** provenance file
   (no crash; file rewritten valid).
4. **`apps/cli/tests/e2e/adoption-breadth.e2e.test.ts` → "KVP-shaped breadth"**
   (1 test, P0/P1) — legacy-sdd (descend into nested `specs/`, `e2e/`→increment
   with `AXIOM:E2E-ORIGIN`, ignore `REGISTRO_INCREMENTOS.md`/`INDEX.md`/`_index.md`
   **by name** — closing the untested-by-name case) AND from-context taxonomy
   (`TECHNICAL_CONTEXT.md`→architecture, arbitrary `backend/`→references) converge
   into ONE dedicated spec scope, both read back through the loaders that back
   `spec.incrementRead` / `spec.technicalContextIndexRead`.
5. **`apps/cli/tests/e2e/adopt-upgrade.e2e.test.ts`** (1 test, P0) — the FIX-E
   adopt→upgrade chain: `runWorkspaceAdopt` materializes `init.json`, then the REAL
   `runUpgrade({dryRun:true})` PASSES the orchestrator `upgrade-command` gate;
   removing `init.json` makes the SAME call FAIL with `gateFailure` + a
   `missing-init-json`-style message. No prior test drove `runUpgrade` through the
   gate after adoption.
6. **`packages/mcp-server/tests/spec-scope-convergence.test.ts` → new
   `spec.planRead` case** (1 test, P0) — `spec.planRead` on a dedicated spec repo
   with DEFAULT MCP args resolves a plan at `<scope>/plans` (no `axiom.spec`).
   Convergence was previously proven only for `spec.incrementRead` on a dedicated
   repo.

**Not added (verified already covered — avoided duplication):**
- Cross-repo role-start reads plan-approved from the spec repo with no local
  bindings → already `axiom-role-cross-repo.test.ts` AC1/AC2/AC2b.
- openspec + docs-adr (MADR) mixed adoption in one repo → already
  `e2e/workspace-adopt.e2e.test.ts` and `adopter-e2e.test.ts`.
- Partially-Axiom context no-clobber → already `workspace-adopt.test.ts`.

---

## Known gaps to build (prioritized)

For the owner to decide on next. These are **product gaps/limitations** surfaced
this session, not test omissions.

### P1

1. **Operator-invokable checkpoint restore** (CC-VER-05). `restoreCheckpoint`
   exists in `@axiom/versioning` and is unit-tested, but no CLI command exposes it.
   After a failed/ regretted upgrade an operator cannot restore a pre-upgrade
   checkpoint. Build `axiom upgrade --restore <id>` (or `axiom restore`).
2. **Cross-repo upgrade fan-out** (CC-VER-09). `runUpgrade` operates on a single
   resolved project (cwd). A multi-repo workspace must be upgraded repo-by-repo.
   Consider a control-repo-driven fan-out over topology repos.
3. **`member install` does not emit code-intel MCP / process-surfaces**
   (CC-ADP-02). A cloned member gets brokers + toolchain activation but never the
   per-role north-star surfaces nor code-intel entries. Thread
   `materializeProcessSurfaces` + `codeIntelProviders` into `member-install.ts`.
   (Minimum: add a **guard test** locking today's behavior so the gap is tracked.)
4. **`adapter add` (`runAdapterAdd`) omits process-surfaces + code-intel**
   (CC-ADP-03). Unlike `repo add`/`role add`, adding an adapter regenerates
   AGENTS.md but not the process surfaces, and its `writeWorkspaceNativeMcpConfigs`
   call drops the `codeIntelProviders` arg. Bring it to parity.
5. **Provenance corruption loses idempotency** (CC-ADOPT-13). A corrupt/absent
   `.migration-provenance.yml` degrades to empty and the migration re-adopts
   (duplicate) — there is no secondary content-idempotency guard for
   non-Axiom-format ids. Consider a content-hash pre-scan or a stronger collision
   policy. (No-crash + self-heal are now covered.)
6. **git tool on a non-git dir at the mcp-tools handler layer** (CC-XREPO-05).
   Covered at the git-service layer only; add a handler-level test (or
   confirm the handler surfaces the typed error).
7. **`--force` functional-verify bypass** (CC-VERIFY-06). Only `--no-verify` /
   `--preview` exist; confirm whether `--force` is intended and cover or drop it.
8. **sdd-orchestrator surface materialization** (CC-NS-01). Only unit-mapped
   (`surfaceIdsForRole('sdd')`), not asserted materialized in the control repo by
   an integration/e2e. Add a materialization assertion.

### P2

9. **Huge-repo scan bounds** (CC-ADOPT-15). The appendix cap + one-level descend
   exist but are untested at their limits; add a bound-triggering test.
10. **`configure.ts` near-duplicate generator call** (CC-ADP-04). Flagged this
    session; verify whether the surface generator is invoked twice and dedupe.
11. **legacy-sdd unreadable-doc continue** (CC-ADOPT-18). from-context covers the
    read-error path; legacy-sdd only covers folder-with-no-`.md`.

---

## Gate

`npm run build` → `npm test` → `npm run typecheck` → `npm run doctor` in `Axiom/`.
Results recorded in the session report. No test was weakened; any 5000ms load
timeout is isolated to confirm the known flake.
