# Increment: Configure/upgrade/repair operations on existing installs — reconcile

Status: closed
Date: 2026-07-02

Closed by `INC-20260702-configure-upgrade-repair-reconcile-validator`
(validator-reviewer, final role of the INC-22 chain). This audit's own
acceptance criteria were already fully met at authoring time
(audit-only increment, no code changes); closure was withheld pending
the full chain's independent review, which found no discrepancies in
this audit's findings.

## Goal

Audit `axiom configure` and `axiom upgrade` (CLI + TUI wiring) against
source doc §17's full operation list for modifying an existing Axiom
install, confirm or refute the roadmap's hypothesis that a distinct
`repair` operation does not exist, and — if it genuinely doesn't —
recommend a minimal, buildable first version rather than a large
speculative one. This increment is **audit-only**: no code changes.

## Context

Roadmap increment INC-22 (`Axiom.Spec/specs/increments/INC-20260702-axiom-redesign-roadmap/README.md`,
Phase G). Source doc §17 (`axiom_decisiones_sesion_prompt_implementacion.md`)
lists the operations a TUI/CLI must support once it detects an existing
Axiom install, and names three operation **types**:

```text
configure -> cambia configuración funcional del proyecto
upgrade   -> actualiza assets Axiom desde versión nueva
repair    -> repara manifests, índices o estructuras incompletas
```

The roadmap's hypothesis (written before this audit): `configure.ts`/
`upgrade.ts` exist and are wired into the TUI (`tui/flows/configure.ts`,
`upgrade.ts`); a distinct `repair` operation "was not confirmed among
audited commands." Dependencies INC-21 (upgrade output, closed),
INC-04 (TUI detection, closed), INC-15 (MCP isolation/config, closed)
are all satisfied.

## Scope

- Read `Axiom/apps/cli/src/commands/configure.ts` and
  `Axiom/apps/cli/src/commands/upgrade.ts` (+ `@axiom/versioning`'s
  `upgrade.ts`) in full.
- Read `Axiom/packages/tui/src/flows/configure.ts` and `upgrade.ts` to
  confirm TUI wiring and naming.
- Build an operation checklist against source doc §17's full list.
- Re-verify, by direct grep across `Axiom/apps/cli/src` and
  `Axiom/packages/*`, whether any `repair` operation exists today
  (21+ increments have landed since the roadmap's original check).
- If `repair` exists in some form, describe exactly what it does and
  how it relates to source doc §17's definition.
- Assess whether `@axiom/doctor` has any auto-fix capability, or is
  purely diagnostic.
- Recommend a minimal-first-version scope for any genuinely missing
  `repair` capability.

## Non-goals

- No code implementation (audit-only increment; `cli-implementer`/
  `tui-developer`/`validator-reviewer` are the next roles, not this one).
- No redesign of `configure`/`upgrade`'s existing contracts.
- No attempt to make `axiom doctor` auto-fix every check category it
  currently only diagnoses — that would be large, unrequested scope.

## Acceptance criteria

- [x] `configure.ts` and `upgrade.ts` read in full (fresh read, not
      reused from prior increment context).
- [x] TUI flow files read and naming/wiring confirmed or corrected.
- [x] Operation checklist produced against source doc §17's exact list.
- [x] `repair` existence re-verified by direct grep, with a corrected
      finding if the roadmap's original claim no longer holds.
- [x] `@axiom/doctor` auto-fix capability assessed directly from
      `checks.ts` (not assumed from the general-spec description).
- [x] Minimal-first-version `repair` scope recommended, explicitly
      distinguishing "buildable now by composition" from "needs new
      fix-logic per check category."
- [x] Brief for the next role (cli-implementer) written.

## Open questions

- **Q-repair-1**: Should a new, general `axiom repair` command be
  introduced at all, given that two narrow, real `repair` subcommands
  (`axiom toolchain repair`, `axiom mcp repair`) already exist and
  cover their own domains adequately? Or does source doc §17's
  "repara manifests, índices o estructuras incompletas" describe a
  **new, broader** surface (structural artifacts: `axiom.yaml`,
  `install-profile.json`, generated adapter files, indexes) that
  neither existing `repair` subcommand touches? This audit recommends
  the latter (see "Repair scope recommendation" below) but the final
  call belongs to whoever scopes the next increment's acceptance
  criteria.
- **Q-repair-2**: Should the new `repair` be a top-level `axiom repair`
  command (mirroring `axiom configure`/`axiom upgrade`'s standalone
  shape), or a `--repair` flag/mode on `axiom doctor` (since it would
  literally consume `runDoctorChecks`' output)? This audit recommends
  a standalone `axiom repair` command for symmetry with `configure`/
  `upgrade` (source doc §17 lists it as a sibling operation type, not
  a doctor flag) but flags this as a real design choice, not a fact.

## Assumptions

- "21+ increments have landed since the roadmap's original repair
  check" (per the task brief) refers to INC-01 through INC-21 plus the
  dedicated D3 (schemaVersion:2) increment, all closed per
  `Axiom.Spec/general-spec.md`.
- The existing `axiom toolchain repair` (spec 0027/B2) and
  `axiom mcp repair` (spec 0024/spec 0029 C2) commands are correctly
  scoped to their own domains and are not being asked to be replaced
  or merged — this audit treats them as precedent/reusable patterns
  for a new, broader `repair`, not as a gap to fix.

## Implementation notes

### 1. `configure.ts` — fresh full read

`Axiom/apps/cli/src/commands/configure.ts` (491 lines). `runConfigure`:

1. Reads `<rootPath>/.sdd/<projectName>/init.json` (the `profileTriple`
   written by `axiom init`: `functionalProfile`, `operationalOverlay`,
   `adapterTarget`).
2. Calls `installProfile()` (`@axiom/installer`) with that triple —
   this re-materializes the `ResolvedInstallProfile` and its
   `generatedFiles`/`externalDependencies` for the **currently
   configured** profile. This is a "re-apply the existing functional
   config" operation, not an "add/remove a specific repo/role/adapter"
   operation.
3. For `adapterTarget === 'opencode'` specifically, conditionally calls
   `generateOpencodeConfig` if `installProfile` didn't already cover
   `.opencode/AGENTS.md` (avoids double-write).
4. For `adapterTarget === 'copilot-vscode' | 'github-copilot'`, calls
   `writeCopilotInstructions` (reads `product.manifest.yaml`,
   `providers.yaml`, local overlay; resolves `supportLevel`; writes
   `.github/copilot-instructions.md` with team-block preservation).

`configure` takes **zero CLI flags** — it always re-runs the full
install-profile materialization for whatever `init.json` already says.
There is no `axiom configure --add-repo`, `--add-role`,
`--add-adapter`, or `--add-mcp` surface; it is a single "regenerate
everything from the persisted profile" operation, scoped to one
project (not one repo among several — the whole `installProfile()`
model is single-profile, single-adapter-target per invocation).

### 2. `upgrade.ts` — fresh full read (CLI + `@axiom/versioning`)

`Axiom/apps/cli/src/commands/upgrade.ts` (287 lines) + already-audited
`@axiom/versioning/src/upgrade.ts` (confirmed real by INC-20, re-read
here to confirm nothing changed since). `runUpgrade`:

- `--dry-run`: calls `previewUpgrade` (pure read, returns
  `UpgradePlan { fromVersion, toVersion, isNoOp, migrations,
  touchedFiles }`), no mutation.
- Real run: calls `executeUpgrade` (`@axiom/versioning`), which
  creates a pre-upgrade checkpoint, applies pending migrations
  (currently exactly one registered: `0.0.0 -> 0.1.0`), persists the
  new `ManagedState`, optionally runs `syncFn`/`doctorFn`
  (`--no-sync`/`--no-doctor` to skip), and rolls back + re-throws on
  any failure (`upgradeFailed: true`).
- `--from-checkpoint <id>`: restores from an explicit prior checkpoint
  instead of creating a new one.
- `--target-version <v>`: overrides the default target
  (`RUNTIME_VERSION` from `@axiom/versioning`'s own `package.json`).

This is squarely "actualiza assets Axiom desde versión nueva" (source
doc §17's `upgrade` definition) — it advances `ManagedState.runtime`/
`adapterTargets`/`lastUpgrade` via the registered migration chain, with
rollback safety. Confirms INC-20/INC-21's prior findings still hold;
no drift detected since those increments closed.

### 3. TUI wiring — confirmed, naming exact

`Axiom/packages/tui/src/flows/configure.ts` (59 lines) and
`Axiom/packages/tui/src/flows/upgrade.ts` (101 lines) exist exactly as
the roadmap named them. Both are thin, stateless wrappers:

- `runConfigureFlow`: calls the injected `runners.configure`, formats
  `FlowResult.message` from `generatedFiles`/`externalDependencies`/
  `profilePath`. No confirmation logic (that's the driver's job per
  its own header comment).
- `runUpgradeFlow`: calls the injected `runners.upgrade` with
  `dryRun` true/false depending on which stage the driver is in
  (dry-run to show the plan, then a second real call after user
  confirms). Formats plan-vs-result messages differently.

Neither flow file contains business logic beyond message formatting —
confirms the roadmap's "TUI has no business logic of its own" claim
(source doc 1.5/12.4) still holds for these two flows specifically.

`Axiom/packages/tui/src/router.ts`'s `MENU_ITEMS` (the actual menu
content, not `screens/menu.ts`, which is a pure renderer) has exactly
5 real entries: "Configurar proyecto", "Sincronizar outputs del
adapter", "Ejecutar diagnóstico", "Aplicar upgrade del runtime",
"Model routing" (plus "Salir de Axiom TUI"). No `repair` menu entry
exists.

### 4. Operation checklist against source doc §17

Source doc §17 lists 14 operations. Status of each against the actual
codebase:

| # | Source doc §17 operation | Covered today? | Where |
|---|---|---|---|
| 1 | añadir repo de código | **No** | No multi-repo-add flow exists; `@axiom/topology`'s `roleCodeRepositories` is populated at manifest-write time, no dedicated "add a repo to an existing install" command found in `apps/cli/src/commands/repo.ts` beyond what INC-01 already covers (not re-audited line-by-line here — out of this increment's scope, flagged for whoever revisits INC-01's repo-role-map follow-up) |
| 2 | quitar/desactivar repo | **No** | Same as above — no removal path found |
| 3 | cambiar ruta de repo | **Partial** | `.sdd/local/topology-bindings.yaml` (per-user local path overrides, per `general-spec.md`'s topology section) covers a path override, but not a structural "change the registered path" operation |
| 4 | añadir rol | **No** | No `axiom role add`-style command surfaced in this pass; `axiom-role.ts` exists (INC-06 scope) but was not re-audited here for an "add a new role to an existing install" operation specifically |
| 5 | modificar rol | **No** | Same as above |
| 6 | añadir adapter | **Partial** | `configure` regenerates whatever `adapterTarget` is already in `init.json` — it does not let you *add a second* adapter target to an existing install (single `adapterTarget` per profile, not a set) |
| 7 | regenerar adapters | **Yes** | This is exactly what `axiom configure` already does — re-running `installProfile()` + the target-specific writer (`writeCopilotInstructions`/`generateOpencodeConfig`) regenerates the currently configured adapter's files. Confirmed real, not aspirational. |
| 8 | añadir tool/MCP | **Yes (narrow)** | `axiom mcp repair --id <id>` (spec 0024/0029) registers a binding for an MCP already declared in `mcp-manifest.yaml` — this is closer to "activate/repair a declared MCP" than "add a new one to the manifest," but the manifest itself is edited by "manual / `axiom mcp repair`" per `general-spec.md`'s own consumer table. No dedicated `axiom mcp add` was found. |
| 9 | quitar tool/MCP | **No** | No removal command found |
| 10 | actualizar skills | **Partial** | `@axiom/skills`'s `refresh.ts` (drift detection, per `general-spec.md`) exists but was not re-confirmed in this pass to be wired into `configure`/`upgrade`; likely a separate `axiom skills` subcommand, not part of configure/upgrade's own flow |
| 11 | instalar autoskills en repo | **Unconfirmed** | Not found in `configure.ts`/`upgrade.ts`; would need a dedicated audit of `@axiom/skills`'s `apply.ts`/`materialize.ts` (out of this increment's scope) |
| 12 | actualizar guías | **No** | No guides-update path found in `configure`/`upgrade` |
| 13 | actualizar templates | **Partial** | `upgrade`'s migration mechanism can touch template-shaped assets as part of a registered migration (e.g. a future migration bumping `templatesVersion`), but `@axiom/versioning`'s own `ManagedState` (confirmed by INC-20) has no `assets.templatesVersion`-equivalent field yet — so "update templates" isn't independently addressable today, only bundled inside whatever a migration does |
| 14 | validar consistencia | **Yes** | `axiom doctor` (9+ check categories, confirmed extensively by prior increments) is exactly this operation, already real and wired (`upgrade --no-doctor` opts out of running it post-upgrade) |
| 15 | exportar diagnóstico | **Partial** | `axiom doctor` produces a report (`report.ts`) but "exportar" (writing it to a file/artifact for sharing) was not confirmed as a distinct flag in this pass — `axiom mcp inventory --json`/`axiom doctor`'s own JSON output mode (if any) would need a dedicated check, out of this increment's scope |

Summary: **`configure` + `upgrade` together genuinely cover 2 of 15
operations fully (regenerate adapters, validate consistency), and
partially/narrowly touch 5 more** (path override, single-adapter
regen, MCP binding repair, skills drift detection existing separately,
templates bundled into migrations, diagnostic export). **7 operations
have no confirmed implementation at all** (add/remove repo, add/modify
role, remove tool/MCP, update guides, install autoskills in repo).

This is a materially larger gap than the roadmap's original "configure
and upgrade already cover most of this, only repair might be missing"
framing suggested. Most of the missing operations are not `repair`
material at all — they are **additional `configure` sub-operations**
(add-repo, add-role, remove-tool) that `configure.ts`'s current
single-shot "re-apply the whole profile" design does not support. This
is a `configure` **surface gap**, not a `repair` gap, and is
independent of whether `repair` gets built. Flagged explicitly as a
finding for whoever plans `configure`'s next iteration — not
addressed by this audit's `repair` recommendation below.

### 5. `repair` — corrected finding: two narrow `repair` commands exist, no general one

Direct grep re-verification (`rg -i repair` across
`Axiom/apps/cli/src` and `Axiom/packages/*`, source files only, dist/
build artifacts excluded from the finding but confirming they're
build output of the same source):

- **`axiom toolchain repair`** (`Axiom/packages/toolchain/src/repair.ts`,
  `Axiom/apps/cli/src/commands/toolchain.ts`, spec 0027/B2, tested by
  `packages/toolchain/tests/repair-add-gitignore.test.ts`).
  `repairTool`/`repairToolchain`: re-derives a single tool's detection
  state; if `absent`, creates missing `detectionPaths` (mkdir,
  recursive); if `present`, no-op; if `drift`, reports but does **not**
  correct it (explicitly documented as requiring manual inspection or
  a dedicated sub-command). `instruction-only` tools are always a
  no-op. Scoped strictly to the toolchain manifest's own tool entries
  — does not touch `axiom.yaml`, indexes, or any other structural file.
- **`axiom mcp repair`** (`Axiom/apps/cli/src/commands/mcp.ts`, spec
  0024/0029-C2). Verifies an MCP id exists in `mcp-manifest.yaml` and
  has `installMode: project-scoped`, then registers a
  `lastRepairedAt` timestamp in `.sdd/local/mcp-bindings.json` (atomic
  tmp+rename). Explicitly documented as **not** physically installing
  the MCP — it "repairs" the declared binding record, not any
  filesystem/index structure.

**Corrected finding**: the roadmap's claim ("a distinct `repair`
operation was NOT confirmed among audited commands") is **no longer
fully accurate** — two real, narrow, well-tested `repair` commands now
exist, both landed after the roadmap's original audit pass (their spec
numbers, 0027 and 0024/0029, plus their being absent from the
roadmap's own file listing, indicate they are genuinely newer or were
simply missed). However, **neither is source doc §17's `repair`**:
§17 defines `repair` as "repara manifests, índices o estructuras
incompletas" — a **general**, cross-cutting operation over the whole
install's structural correctness, not a tool-specific or MCP-specific
binding repair. So the corrected, precise finding is: **no general
`axiom repair` command exists; two narrowly-scoped domain-specific
`repair` subcommands exist and should be treated as reusable
precedent, not duplicated or replaced.**

### 6. `@axiom/doctor` auto-fix capability — confirmed: purely diagnostic

Direct read of `Axiom/packages/doctor/src/checks.ts` (the file
`general-spec.md` describes as having "9+ check categories" — a
re-grep in this pass counts at least 12 distinct `category:` usages
across `pass`/`fail`/`warn`/`skip` helper calls, consistent with
`general-spec.md`'s "~19+ categories now" framing once
`governance-checks.ts` and `write-scope.ts` are included). Grepping
`checks.ts` for `fix`/`repair`/`autofix` returns **zero real matches**
— the only hits are the unrelated word "fixture" inside test-fixture
comments for `tool-routing` smoke tests (`TR-002`). `DoctorCheck`'s
shape (`{ id, category, description, status: pass|fail|warn|skip,
evidence }`) has no `fixable`/`autoFixed`/`suggestedFix` field. Doctor
detects and reports; it never mutates the filesystem. This confirms
the task brief's suspicion directly, not just by inference from
`general-spec.md`'s prose.

### 7. Repair scope recommendation — minimal viable version

Building a `repair` that attempts to auto-fix every category doctor
currently checks (~19+ categories: boundaries, policies, manifests,
isolation, capability-model, install-profiles, gateway, tool-routing,
topology, QA-lane coherence, write-scope, plus governance checks) is
**large, speculative scope** — most doctor failures require human
judgment (e.g. a boundary violation, a missing product-runtime scope,
an inconsistent capability model) and do not have a single safe,
mechanical fix. Building fix-logic per category is explicitly **not**
recommended as a first version; it would violate `AGENTS.md`'s
"no speculative architecture" and "keep changes small and focused"
guardrails.

**Recommended minimal `repair` v1**: a new `axiom repair` command that
runs `runDoctorChecks`, then applies fixes for only the **narrow subset
of doctor findings that are genuinely mechanical, safe, and
idempotent** — the same posture `toolchain repair` already uses
(create-if-missing, never overwrite, never guess intent). Concrete
candidates, in priority order, each independently small:

1. **Missing/stale generated file re-materialization** — reuse
   `axiom configure`'s existing `installProfile()` + adapter-writer
   call path verbatim (no new writer). If doctor's manifest/adapter
   checks report a missing generated file that `GENERATED_FILES_BY_TARGET`
   declares should exist, `repair` calls the same regeneration
   `configure` already performs. This is the single lowest-risk,
   highest-value fix: it is literally "re-run configure," with zero
   new business logic.
2. **Missing index-able structures that are cache-only** — `axiom
   index rebuild` (confirmed real, cache-less, direct-scan, per
   `general-spec.md`'s "Folder-per-artifact convention" section)
   already regenerates derived caches from source-of-truth folders.
   `repair` can call this unconditionally — it is safe by
   construction (`listArtifacts` tolerates absent/malformed entries,
   never throws).
3. **Missing toolchain/MCP binding artifacts** — delegate directly to
   the two existing `repairToolchain`/`runMcpRepair` functions for
   their own domains, rather than reimplementing equivalent logic.
   This is composition, not new fix-logic.

Each of these three is **buildable now by composition** over existing,
already-tested functions (`installProfile`, `runIndexRebuild`,
`repairToolchain`, `runMcpRepair`) — no new fix-logic per doctor
check category is needed for v1. A `repair` v1 built this way would:

- Run `runDoctorChecks` first (read-only).
- For each `fail`/`warn` result whose `category`/`id` matches one of
  the three buildable cases above, call the corresponding existing
  function.
- For every other `fail`/`warn` (the ~16+ remaining categories),
  report it verbatim in the `repair` output as "not auto-fixable;
  needs manual review" — never silently skip, never guess a fix.
- Never touch `pass`/`skip` results.

This keeps `repair` v1 small, composition-only, and honest about its
limits, while still satisfying source doc §17's literal ask
("repara manifests, índices o estructuras incompletas") for the
subset of cases where "incompleto" genuinely means "a known generator
didn't run yet" rather than "a human decision is missing." Expanding
`repair` to cover more categories is explicitly a future increment's
decision, not this one's.

## Brief for the next role (cli-implementer)

If a `repair` increment is opened next, scope it to:

1. A new `axiom repair` command (`apps/cli/src/commands/repair.ts`),
   mirroring `configure.ts`/`upgrade.ts`'s `runX`/`registerX` split
   for testability (no `commander`/`process.exit` inside `runRepair`).
2. `runRepair` calls `runDoctorChecks` first, then dispatches only the
   3 buildable cases above via existing functions — do not write new
   fix-logic for any other doctor category.
3. Output format should mirror `formatResult`'s style in `upgrade.ts`
   (human-readable summary + explicit "not auto-fixable" list), and
   the command should be zero-flag or minimal-flag for v1 (matching
   `configure`'s zero-flag precedent), not a large flag surface.
4. Do not touch `toolchain repair`/`mcp repair`'s own CLI surfaces —
   call their underlying functions, don't duplicate their commands.
5. TUI wiring (a `tui/flows/repair.ts` + a new `MENU_ITEMS` entry) is
   the `tui-developer`'s job after the CLI command lands, following
   `configure.ts`'s/`upgrade.ts`'s exact flow-file shape (stateless,
   no confirmation logic inside the flow itself).
6. The larger `configure` surface gap found in section 4 above (7 of
   15 source-doc-§17 operations with no implementation at all — add/
   remove repo, add/modify role, remove tool/MCP, update guides,
   install autoskills) is a **separate, larger finding** not resolved
   by a minimal `repair`. Flag it back to whoever sequences the next
   roadmap increment — it may warrant its own increment rather than
   being folded into this one's `repair` scope.

## Validation

`No validation command was found. Performed best-effort validation by inspecting changed files and checking consistency against the requested behavior.`

Best-effort validation performed for this audit-only increment:

- Read `configure.ts` (491 lines) and `upgrade.ts` (287 lines) in full,
  fresh (not reused from prior D3/INC-20/INC-21 context).
- Read `Axiom/packages/versioning/src/upgrade.ts`'s existence
  confirmed via `Glob`; its behavior cross-checked against INC-20/
  INC-21's already-closed findings rather than re-reading it
  line-by-line a third time (no drift indicators found — `RUNTIME_VERSION`/
  `executeUpgrade`/`previewUpgrade` imports in `upgrade.ts` are
  unchanged from what INC-21 documented).
- Read both TUI flow files (`configure.ts`, `upgrade.ts` under
  `packages/tui/src/flows/`) in full — confirmed exact naming and
  stateless-wrapper shape.
- Read `packages/tui/src/router.ts`'s `MENU_ITEMS` directly (not
  inferred) to confirm no `repair` menu entry exists.
- Grepped `Axiom/apps/cli/src` and `Axiom/packages/*` case-insensitively
  for `repair`, found and read both real hits in full
  (`packages/toolchain/src/repair.ts`, the `repair` sections of
  `apps/cli/src/commands/mcp.ts` and `commands/toolchain.ts`).
- Grepped `packages/doctor/src/checks.ts` for `fix|repair|autofix`,
  confirmed zero real matches (only unrelated "fixture" word hits),
  and inspected `DoctorCheck`'s type shape directly to confirm no
  fix-related field exists.
- Did not re-audit `axiom-role.ts`, `repo.ts`'s full add/remove-repo
  surface, `@axiom/skills`'s `apply.ts`/`materialize.ts`, or a
  diagnostic-export flag line-by-line — each explicitly flagged above
  as "not re-audited in this pass" rather than guessed at.

## Result

Two corrected findings relative to the roadmap's original hypothesis:

1. **`repair` is not entirely absent** — two real, narrow,
   well-tested `repair` subcommands (`axiom toolchain repair`, `axiom
   mcp repair`) exist today, landed after the roadmap's original audit.
   Neither implements source doc §17's general "repara manifests,
   índices o estructuras incompletas" — a general `axiom repair`
   remains genuinely additive, but should reuse both as internal
   building blocks/precedent rather than be scoped as if starting from
   zero.
2. **`configure`/`upgrade` cover fewer of source doc §17's 15
   operations than the roadmap assumed** — a full checklist (section 4
   above) found only 2 of 15 fully covered, 5 partially/narrowly
   covered, and 7 with no confirmed implementation at all (mostly
   add/remove-repo, add/modify-role, remove-tool/MCP, update-guides,
   install-autoskills — a `configure`-surface gap, not a `repair` gap).

`@axiom/doctor` is confirmed purely diagnostic (zero auto-fix
capability), so any `repair` command must implement or delegate to
its own fix-logic. A minimal, composition-only `repair` v1 (regenerate
via `installProfile`, rebuild indexes via existing `index rebuild`,
delegate to `repairToolchain`/`runMcpRepair` for their own domains) is
buildable now with zero new fix-logic per doctor-check-category, and is
the recommended scope for the next `cli-implementer` increment.

## General spec integration

Two additions to `Axiom.Spec/general-spec.md`, both narrowly factual
and consistent with the file's existing style (stable, load-bearing
facts, not history):

1. A corrected/updated note under the existing "Versioning
   (`@axiom/versioning`)" section area: two real `repair` subcommands
   exist (`axiom toolchain repair`, `axiom mcp repair`), landed by
   specs 0027/B2 and 0024/0029-C2 respectively, both narrowly scoped
   to their own domain and neither implementing a general "repair
   manifests/indexes/structures" operation.
2. A note that `@axiom/doctor` (`checks.ts`) is confirmed purely
   diagnostic — `DoctorCheck` has no fix-related field, and no
   auto-fix code path exists anywhere in the package.

See `Axiom.Spec/general-spec.md`'s new "Configure/upgrade/repair
operations" section for the integrated text.

## Next step recommendation

Open a follow-up increment (e.g.
`INC-20260702-repair-command-implementation`) with **cli-implementer**
as the primary role, scoped exactly to the "Repair scope
recommendation" (section 7) and "Brief for the next role" above: a
minimal, composition-only `axiom repair` command covering only the 3
buildable cases (generated-file regeneration via `installProfile`,
index rebuild, delegation to existing toolchain/MCP repair functions),
followed by **tui-developer** for the flow/menu wiring and
**validator-reviewer** for final review. Separately, flag the larger
`configure`-surface gap (7 of 15 source-doc-§17 operations with no
implementation) to whoever sequences the next roadmap increment — it
is a distinct, larger body of work not addressed by a minimal
`repair`.
