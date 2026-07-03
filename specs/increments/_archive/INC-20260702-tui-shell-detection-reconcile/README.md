# Increment: Reconcile contextual TUI shell and detection (`@axiom/tui`)

Status: pending
Date: 2026-07-02

## Goal

Audit `@axiom/tui` (shell, router, screens) and the project-detection path
it depends on (`@axiom/filesystem-truth#discoverAxiomRoot`,
`@axiom/project-resolution#resolveProject`) against the parent roadmap's
INC-04 hypothesis and against the addendum's section 14 detection
heuristic, now that INC-01 (registry v2), INC-02 (structural command/event
model), and INC-03 (installer wizard reconciliation) are closed. Produce a
concrete gap analysis (detection heuristic + menu diff) and an
implementation brief for the next subagent (**tui-developer**). This
increment is audit-only; it does not implement code.

## Context

This is INC-04 of
`Axiom.Spec/specs/increments/INC-20260702-axiom-redesign-roadmap/README.md`.
The roadmap's original (pre-INC-01/02/03) hypothesis for INC-04 stated:

- `@axiom/tui` already renders a real menu (`configure/sync/doctor/
  upgrade` + `mcp-inventory`/`memory-inventory`/`topology`/`model-list`/
  `projects` screens).
- Detection is driven by `project-resolution` (`axiom.yaml` in cwd via
  `discoverAxiomRoot`), not yet the addendum-14 heuristic (parent folders,
  registered-path membership, git root).
- The shell, the `@axiom/cli-commands` layering seam, and the "TUI has no
  business logic of its own" rule (source doc 1.5, 12.4) are already
  correctly implemented.
- Diff is additive: extend `discoverAxiomRoot`/`project-resolution` with
  the addendum-14 heuristic, extend the menu with items for new INC-01
  commands; **no shell rewrite needed**.

This audit re-verifies every one of those claims against the live code
(`Axiom/packages/tui/src/*`, `Axiom/packages/tui/tests/*`,
`Axiom/packages/cli-commands/src/index.ts`,
`Axiom/packages/filesystem-truth/src/discovery.ts`,
`Axiom/packages/project-resolution/src/resolver.ts`) and against addendum
section 14 read directly from
`axiom_decisiones_sesion_addendum_revision.md`, since the roadmap's INC-04
entry was written before INC-01 landed the v2 registry
(`listProjectsV2`/`findByRepoPathV2` in `@axiom/user-workspace`) that
section 14's "registered-path membership" check depends on.

## Scope

- Read `Axiom/packages/tui/src/` in full (`driver.ts`, `router.ts`, every
  `screens/*.ts` file) and `Axiom/packages/tui/tests/`.
- Read `Axiom/packages/cli-commands/src/index.ts` (the layering seam).
- Read `Axiom/packages/filesystem-truth/src/discovery.ts`
  (`discoverAxiomRoot`, `AXIOM_CONFIG_FILENAME`) and
  `Axiom/packages/project-resolution/src/resolver.ts` (`resolveProject`,
  now version-aware per INC-01).
- Re-read addendum section 14 verbatim and check feasibility of each of
  its four detection checks against what INC-01 actually shipped in
  `@axiom/user-workspace`.
- Diff the roadmap's proposed menu content (source doc sections 12.2/12.3)
  against the live `MENU_ITEMS` array and the full set of reachable
  screens (menu-reachable vs CLI-flag-only).
- Produce a gap analysis and an implementation brief for the
  **tui-developer** subagent.

## Non-goals

- No code implementation in this increment (migration-engineer audit
  only).
- No decision on whether `axiom.yaml` schemaVersion:2 re-enablement in
  `init.ts` happens here (explicitly deferred per parent context; INC-04
  only affects detection/menu, not manifest emission).
- No Roles model or Tools/MCPs selection work (explicitly deferred per
  parent context, unrelated to this increment).
- No implementation of `mcp-inventory`/`memory-inventory` beyond what
  already exists (their current stub state is documented as a finding,
  not fixed here).
- No rewrite of the router/driver shell architecture.

## Acceptance criteria

- [x] Every claim in the roadmap's INC-04 entry is checked directly
      against live code, not assumed.
- [x] The addendum-14 detection heuristic is reproduced verbatim and each
      of its 4 checks is individually assessed for current feasibility.
- [x] A concrete menu diff table is produced: roadmap-proposed items vs.
      live `MENU_ITEMS`, marking each as `exists (same)` /
      `exists (different label/order)` / `missing`.
- [x] The "no shell rewrite needed" judgment is explicitly confirmed or
      revised based on the fresh read.
- [x] A tui-developer implementation brief is written with concrete file
      targets, not a wholesale menu rewrite.
- [x] No code changed in `Axiom.SDD` or `Axiom`.
- [x] Validation statement recorded per `Axiom.SDD/AGENTS.md` (no
      validation command applicable; no code changed).

## Open questions

- **OQ1**: Should `mcp-inventory` and `memory-inventory` be promoted from
  CLI-flag-only entry points to first-class `MENU_ITEMS` in this same
  increment's follow-up (tui-developer), or is that a separate concern
  from addendum-14 detection? Not blocking — flagged for tui-developer to
  decide scope with the user before implementing, since the parent
  roadmap only asked for "menu content/labels and detection heuristics,"
  not new screen wiring.
- **OQ2**: Addendum-14 item 3 ("current path is inside a path registered
  in `~/.axiom/projects.yml`") requires ancestor-path matching (`cwd`
  starts with or is a descendant of a registered repo path), not exact
  match. `findByRepoPathV2` today does exact `path.resolve` equality
  only. Should the ancestor-walk be implemented as a new
  `@axiom/user-workspace` function (e.g. `findByRepoPathV2` extended with
  an `ancestorMatch` option, or a new `findByAncestorRepoPathV2`), or
  inline in `@axiom/filesystem-truth`/`@axiom/project-resolution` without
  touching `@axiom/user-workspace`'s public surface? Recommend the former
  (keep registry-shape knowledge inside `@axiom/user-workspace`) but this
  is tui-developer's call to make explicit before writing code.
- **OQ3**: What should the "no Axiom detected, but a registered project
  matches via addendum-14 detection" UX be? Today `axiom tui` with no
  resolved project throws `projectNotFound` and exits 1 (see
  `apps/cli/src/commands/tui.ts` lines 212-219) — there is no "no Axiom
  detected" menu screen at all today, contrary to what source doc
  12.2 assumes. Confirm with the user whether INC-04's tui-developer step
  should build this screen, or whether it is deferred to a later
  increment (this changes the diff from "additive relabeling" to "new
  screen + new driver branch").

## Assumptions

- "Existing package(s) audited" per the parent roadmap's own convention:
  this migration-engineer pass reads every file in `packages/tui/src/`
  once each, not a byte-for-byte diff against git history.
- The parent roadmap's dependency ordering (INC-04 depends on INC-01,
  INC-02, INC-03, all closed) is taken as satisfied; this audit does not
  re-verify INC-01/02/03 closure claims beyond what was directly needed
  to confirm `listProjectsV2`/`findByRepoPathV2` exist.
- **Correction (post-audit review)**: this migration-engineer pass
  incorrectly claimed `Axiom.Spec/specs/increments/` had no individual
  spec files for INC-01/02/03. That was wrong — the full chain exists as
  sibling folders (`INC-20260702-registry-manifest-schema-v2*` for INC-01,
  `INC-20260702-structural-commands-events*` for INC-02,
  `INC-20260702-installer-wizard-reconcile*` for INC-03), each with its
  own `README.md` per role. This increment's own naming
  (`INC-20260702-tui-shell-detection-reconcile`) already matches that
  precedent, so no other content in this spec depended on the incorrect
  claim above — it is corrected here for the record.

## Implementation notes

Audit-only. No files were changed under `Axiom/packages/`, `Axiom/apps/`,
or `Axiom.SDD/` for this increment.

### 1. Roadmap claims vs. live code

| Roadmap claim (pre-INC-01/02/03 pass) | Live-code verification | Verdict |
|---|---|---|
| "menu... already renders configure/sync/doctor/upgrade" | `router.ts` `MENU_ITEMS` has `configure`, `sync`, `doctor`, `upgrade`, `model-list`, `exit` (6 items) | Confirmed, plus one item the roadmap didn't mention (`model-list`, added in increment 0019-C3b, predates this pass) |
| "+ more screens: mcp-inventory, memory-inventory, topology, model-list, projects" | All 4 named screens (`mcp-inventory`, `memory-inventory`, `topology`, `projects`) exist as renderer files and as `ScreenId` union members, but **none are `MENU_ITEMS` entries** | Partially wrong: they exist as *screens*, reachable only via CLI flags (`axiom tui --topology`, `--projects`, etc.) or, for `mcp-inventory`/`memory-inventory`, apparently not even wired to a CLI flag in `tui.ts` (see below) — not reachable from the interactive menu at all |
| "cli-commands package exists specifically as a seam so the TUI never imports apps/cli/src/... directly" | Confirmed: `packages/cli-commands/src/index.ts` re-exports `runConfigure`, `runSync`, `runUpgrade`, `runModelValidate`, `runComponentsShow` from `apps/cli/src/commands/*`, plus `runDoctorChecks` from `@axiom/doctor` and model-routing mutations from `@axiom/model-routing`. `driver.ts` imports exclusively from `@axiom/cli-commands`, `@axiom/topology`, `@axiom/installer`, `@axiom/core`, `@axiom/persistence`, `@axiom/model-routing`, `@axiom/user-workspace`, `@axiom/project-resolution` — zero imports from `apps/cli/src/...` | Confirmed, layering rule intact |
| "TUI has no business logic of its own" (1.5, 12.4) | `router.ts`/`reduce` is a pure state machine; screens are pure renderers (`out.write` only); `driver.ts` does orchestrate some read-only data loading (`loadModelListData`, `loadTopologyData`, `loadProjectsData`) directly against `@axiom/topology`/`@axiom/model-routing`/`@axiom/user-workspace`, which is arguably driver-level "glue," not new business logic — it re-uses existing pure loaders rather than reimplementing rules | Confirmed with a caveat: driver-level orchestration exists but does not duplicate business rules |
| "Detection today is driven by project-resolution (axiom.yaml in cwd via discoverAxiomRoot)" | Confirmed: `apps/cli/src/commands/tui.ts` calls `resolveProject(args.cwd)` for every mode; `resolveProject` calls `discoverAxiomRoot` which only walks parent directories looking for `axiom.yaml` | Confirmed |
| "no shell rewrite needed" | See Section 4 below | Confirmed, with one caveat (OQ3) |

New finding not in the original roadmap pass: `packages/tui/src/screens/
projects.ts`'s own doc comment still says `~/.axiom/registry.json`
(v1 JSON registry) as its data source in its header comment, which is
now stale relative to INC-01's `~/.axiom/projects.yml` (v2 YAML,
`listProjectsV2`). This is a documentation/comment drift, not a functional
bug (the driver's `loadProjectsData` calls `listProjects`/`useProject`,
the v1 functions, from `@axiom/user-workspace` — it has not yet been
migrated to call the v2 equivalents at all). This is a genuine functional
gap for tui-developer: the `projects` screen still reads the v1 registry,
not the v2 `projects.yml` registry that INC-01 introduced.

### 2. Addendum section 14 — detection heuristic feasibility

Verbatim from `axiom_decisiones_sesion_addendum_revision.md` section 14:

> La TUI no debe depender solo de encontrar `axiom.yaml` en el directorio
> actual. Debe detectar:
> 1. si el directorio actual contiene `axiom.yaml`;
> 2. si algún padre contiene `axiom.yaml`;
> 3. si la ruta actual está dentro de una ruta registrada en
>    `~/.axiom/projects.yml`;
> 4. si el repo es Git y su root tiene `axiom.yaml`.

| Check | Current implementation | Feasible now (post-INC-01)? |
|---|---|---|
| 1. cwd has `axiom.yaml` | `discoverAxiomRoot`'s first loop iteration (depth 0) | Already implemented |
| 2. a parent has `axiom.yaml` | `discoverAxiomRoot`'s parent-walk loop (up to `MAX_TRAVERSAL_DEPTH = 10`) | Already implemented (checks 1+2 are the same function) |
| 3. cwd is inside a path registered in `~/.axiom/projects.yml` | Not implemented. `findByRepoPathV2` exists in `@axiom/user-workspace` (confirmed at `packages/user-workspace/src/registry.ts:848`) but does **exact** `path.resolve` equality against each registered repo's `path`, not ancestor/prefix matching. `listProjectsV2` (confirmed at line 783) returns all registered projects and could be walked manually for a prefix match. | **Now feasible** (was not before INC-01, since `projects.yml`/`listProjectsV2`/`findByRepoPathV2` did not exist pre-INC-01), but requires new ancestor-matching logic — not a drop-in call to an existing function. See OQ2. |
| 4. git root has `axiom.yaml` | Not implemented anywhere in `filesystem-truth`/`project-resolution`. No git-root detection exists in either package (confirmed by reading both files in full; no `git` or `.git` references). | Feasible, additive: requires walking up from cwd looking for `.git` (or shelling out to `git rev-parse --show-toplevel`), then checking that root for `axiom.yaml`. This is architecturally the same shape as parent-walk (check 2) with a different stop condition. |

Conclusion: addendum-14 was **not fully feasible** before INC-01 (check 3
had no registry v2 to query), and remains **partially unimplemented**
today for checks 3 and 4 specifically. Checks 1 and 2 are already covered
by the existing `discoverAxiomRoot`. This confirms the roadmap's framing
("not yet the addendum's fuller heuristic") and sharpens it: the missing
work is exactly checks 3 and 4, now unblocked by INC-01's registry v2.

### 3. Menu diff: roadmap-proposed vs. live `MENU_ITEMS`

Source doc sections 12.2 ("no Axiom detected") and 12.3 ("Axiom
detected") propose menu content; this table compares against the live
`router.ts` `MENU_ITEMS` array and the full screen set.

| Roadmap/source-doc item | Live status |
|---|---|
| Configure project | `exists (same)` — `MENU_ITEMS[0]`, label "Configurar proyecto" |
| Sync adapter outputs | `exists (same)` — `MENU_ITEMS[1]`, label "Sincronizar outputs del adapter" |
| Run diagnostics (doctor) | `exists (same)` — `MENU_ITEMS[2]`, label "Ejecutar diagnóstico" |
| Apply upgrade | `exists (same)` — `MENU_ITEMS[3]`, label "Aplicar upgrade del runtime" |
| Model routing management | `exists (different origin)` — `MENU_ITEMS[4]`, "Model routing"; not in the roadmap's original 12.2/12.3 text but already live (increment 0019-C3b, predates this pass) |
| **Create increment** | `missing` — `axiom-increment.ts` CLI command exists (`apps/cli/src/commands/axiom-increment.ts`), but no TUI screen or menu entry calls it, and it is not re-exported from `@axiom/cli-commands` |
| **Create bug** | `missing` — same situation, `axiom-bug.ts` exists, no TUI wiring |
| **Create plan** | `missing` — same situation, `axiom-plan.ts` exists, no TUI wiring |
| **Run guided implementation** | `missing` — no equivalent orchestrator-driven "guided implementation" flow found in `tui/src` |
| Inspect topology | `exists (CLI-flag-only, not menu)` — `topology` screen exists, reachable only via `axiom tui --topology`, not from `MENU_ITEMS` |
| Manage/select projects (registry) | `exists (CLI-flag-only, not menu)` — `projects` screen exists, reachable only via `axiom tui --projects`, and still reads the v1 registry (see Section 1 finding) |
| MCP inventory | `exists (stub, CLI-flag-only, not menu)` — `mcp-inventory` screen is a placeholder renderer; no `tui.ts` flag or driver wiring was found connecting it to `runTui`'s `initialScreen` union in the excerpt read (present in the `ScreenId` union and `FLOW_LABELS`, but no `loadMcpInventoryData`-style pre-load call site was found in `driver.ts`, unlike `topology`/`model-validate`/`components-show`) |
| Memory inventory | `exists (stub, CLI-flag-only, not menu)` — same situation as `mcp-inventory` |
| Exit | `exists (same)` — `MENU_ITEMS[5]` |

Net: the roadmap's own INC-04 diff already said "menu items/labels
differ... but shell/layering are correctly implemented" — that holds.
What this fresh read adds precision to: the difference is not just
labels/order, it is that **four proposed first-class menu actions
(create increment/bug/plan, guided implementation) have zero TUI
presence today**, and two "already existing" screens
(`mcp-inventory`/`memory-inventory`) are further from complete than the
roadmap's phrasing ("more screens") suggested — they are placeholder
renderers not fully wired into the driver's dispatch, unlike
`topology`/`projects`/`model-validate`/`components-show`.

### 4. "No shell rewrite needed" — confirmed, with one caveat

The router (`reduce`, pure state machine), the driver's dispatch model
(`dispatchLine` pure / `runTui` impure), and the `@axiom/cli-commands`
layering seam are all sound and require no structural change. Adding menu
items is additive: new `ScreenId` variants + new `MENU_ITEMS` entries +
new flow runners, following the exact pattern already used for
`model-list`, `topology`, `projects`. This confirms the roadmap's
judgment.

The one caveat (OQ3): today there is no "no Axiom detected" menu at all —
`axiom tui` throws and exits 1 outside `--projects`/`--topology`/etc.
modes when `resolveProject` does not resolve. If addendum-14's detection
heuristic finds a project via checks 3 or 4 (registered path or git
root) when check 1/2 (cwd/parent `axiom.yaml`) fail, `tui.ts`'s current
control flow needs a new branch (not a rewrite, but a real code change to
`apps/cli/src/commands/tui.ts`, not just to `discoverAxiomRoot`) to act on
that result instead of throwing `projectNotFound`. This is still additive
to the existing shell, not a rewrite, but it is a control-flow change
beyond "extend a heuristic function," and should be called out explicitly
to tui-developer rather than left implicit.

### 5. Implementation brief for tui-developer

Concrete, scoped tasks (not a wholesale rewrite):

1. **Detection heuristic** (`@axiom/filesystem-truth` and/or
   `@axiom/project-resolution`):
   - Add a git-root check (walk up from cwd looking for `.git`, then test
     that root for `axiom.yaml`) as a new fallback step after
     `discoverAxiomRoot`'s existing parent-walk fails.
   - Add a registered-path check: given the existing `listProjectsV2`
     result set from `@axiom/user-workspace`, test whether cwd is equal
     to or a descendant of any registered repo `path`. Decide (per OQ2)
     whether this lives as a new function in `@axiom/user-workspace`
     (e.g. `findAncestorRepoV2`) or is composed inline from
     `listProjectsV2` in `project-resolution`/`filesystem-truth`. Prefer
     keeping registry-shape knowledge inside `@axiom/user-workspace` to
     avoid leaking `projects.yml` internals into `filesystem-truth`.
   - Compose all four checks into a single ordered resolution function
     (cwd -> parents -> registered path -> git root), consistent with
     addendum-14's stated order.
2. **`apps/cli/src/commands/tui.ts`**: add a control-flow branch so that
   when the primary `resolveProject` fails but the new
   registered-path/git-root check succeeds, the TUI opens against the
   discovered root instead of throwing `projectNotFound` (resolve OQ3
   with the user first — this determines whether this task is in scope
   for this increment's implementation pass or deferred).
3. **Menu additions** (`packages/tui/src/router.ts` `MENU_ITEMS` +
   corresponding flow runners under `packages/tui/src/flows/`): add
   entries for increment/bug/plan creation, wired through
   `@axiom/cli-commands` (which will need new re-exports from
   `axiom-increment.ts`/`axiom-bug.ts`/`axiom-plan.ts`, following the
   exact existing re-export pattern for `runConfigure`/`runSync`/etc.).
   Do not implement "guided implementation" in this pass unless the user
   explicitly confirms scope — it depends on `@axiom/orchestrator`'s
   intent commands (per the roadmap's INC-02 note that several intent
   commands are still stubs), which is outside this increment's audited
   packages.
4. **`projects` screen registry migration**: update
   `loadProjectsData`/`onProjectSelect` in `driver.ts` to call
   `listProjectsV2`/`useProjectV2` instead of the v1
   `listProjects`/`useProject`, and update the screen's stale header
   comment referencing `~/.axiom/registry.json`. This is a direct
   consequence of INC-01 landing and was not caught by the original
   roadmap pass.
5. **Do not** touch `mcp-inventory`/`memory-inventory` wiring unless the
   user explicitly asks (OQ1) — flagged as a finding, not in this
   increment's implementation scope by default.

## Validation

`No validation command was found. Performed best-effort validation by inspecting changed files and checking consistency against the requested behavior.`

No code was changed in this increment, so no test/build run would assert
anything produced by this pass. Best-effort validation performed:

- Read every file in `Axiom/packages/tui/src/` (driver, router, all 16
  screen files, all flow files, prompts, render, index) directly, not
  inferred from names.
- Read all 8 files under `Axiom/packages/tui/tests/` filenames to confirm
  test coverage exists for router/driver/prompts/projects/topology/
  preview/render/flows (contents of `router.test.ts` spot-checked).
- Read `Axiom/packages/cli-commands/src/index.ts` in full.
- Read `Axiom/packages/filesystem-truth/src/discovery.ts` and
  `Axiom/packages/project-resolution/src/resolver.ts` in full.
- Read `axiom_decisiones_sesion_addendum_revision.md` section 14 verbatim
  (and the surrounding sections for context).
- Confirmed `listProjectsV2`/`findByRepoPathV2` exist and read their
  exact implementation and doc comments in
  `Axiom/packages/user-workspace/src/registry.ts` to determine real
  feasibility (not just presence) of addendum-14 check 3.
- Confirmed `axiom-increment.ts`/`axiom-bug.ts`/`axiom-plan.ts` exist as
  files under `apps/cli/src/commands/` via a filesystem search, and
  confirmed via `cli-commands/src/index.ts` that none of them are
  re-exported through the TUI seam today.
- Did not run `Axiom/package.json`'s `npm test`/`npm run build` — no code
  was changed, so running them would only assert pre-existing state, not
  anything produced by this audit.

## Result

Confirmed the roadmap's core INC-04 judgment ("shell/layering intact, gap
is menu content and detection heuristics, no rewrite needed") with two
material refinements: (1) the "more screens" the roadmap credited to the
menu are largely CLI-flag-only or outright placeholder stubs, not
menu-reachable; (2) addendum-14's registered-path check (item 3) is now
feasible thanks to INC-01's `listProjectsV2`/`findByRepoPathV2`, but
requires new ancestor-matching logic, not a drop-in call, and the
`projects` screen itself still reads the pre-INC-01 v1 registry — a
concrete regression-relative-to-current-state finding the original
roadmap pass could not have made before INC-01 landed. A tui-developer
brief with 5 concrete, scoped tasks was produced.

## General spec integration

No integration into `Axiom.Spec/general-spec.md` was performed — this
file does not exist in this repo (confirmed by the parent roadmap
increment's own "General spec integration" note). The closest equivalents
(`Axiom.Spec/specs/00_Resumen_Ejecutivo.md` through `08_Glosario.md`) were
not modified: they describe intent at a level above this increment's
scope (TUI screen inventory and detection heuristic detail), and this
increment's findings are not yet stable/closed knowledge — they are an
audit result feeding a not-yet-executed implementation step. Once
tui-developer implements and validator-reviewer confirms the detection
heuristic and menu changes, that would be the appropriate point to fold
"detection order: cwd -> parent -> registered path -> git root" into a
stable spec document, not before.

## Next step recommendation

Hand off to **tui-developer** with this file as the starting brief.
Concrete inputs for that subagent:

- This spec file:
  `Axiom.Spec/specs/increments/INC-20260702-tui-shell-detection-reconcile/README.md`
- Target files: `Axiom/packages/filesystem-truth/src/discovery.ts`,
  `Axiom/packages/project-resolution/src/resolver.ts`,
  `Axiom/packages/user-workspace/src/registry.ts` (only if OQ2 resolves
  to adding a new exported function there),
  `Axiom/apps/cli/src/commands/tui.ts`,
  `Axiom/packages/tui/src/router.ts`,
  `Axiom/packages/tui/src/driver.ts`,
  `Axiom/packages/cli-commands/src/index.ts`.
- Before writing code, resolve OQ1 (mcp-inventory/memory-inventory scope),
  OQ2 (where the ancestor-match function lives), and OQ3 (whether the
  "no Axiom detected but registered/git-root match found" UX branch is in
  scope for this pass) with the user — these are genuine scope decisions,
  not implementation details.
- After tui-developer's pass, **validator-reviewer** should extend
  `Axiom/packages/tui/tests/` (router/driver tests) and
  `@axiom/project-resolution`'s existing test suite to cover the new
  detection order, and confirm no regression in the existing
  `topology`/`projects`/`model-list` screen tests.
