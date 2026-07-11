# Increment: Cross-repo MCP wiring (control-plane slice)

> **Código**: INC-20260711-cross-repo-mcp-wiring
> **Estado**: Archived (gate verde) — ver §Resultado
> **Fecha de creación**: 2026-07-11
> **Tipo de cambio**: Nueva capacidad (control-plane) + reconciliación multi-repo
> **Epic padre**: `INC-20260711-sdd-launcher-core-port` (D-004; PLAN.md "Cross-cutting")

---

## 1. Goal

Make the role-side lifecycle commands work correctly in a **true multi-repo
workspace**, where an increment/bug/plan's approved **workflow-state** and
**`allowedWriteScope`** live in the **SPEC repo**, but role-implementation runs
from each **ROLE repo**.

Today the role-side commands read state **local-only**
(`.axiom-state/local/workflow-state.json` in the role repo), so:

- `axiom-role start`'s plan-approval gate (`blockRoleStartIfPlanIsNotApproved`)
  cannot see the plan and blocks even when the plan IS approved in the spec repo.
- The per-role write-scope review (`INC-20260711-per-role-review`) degrades to
  **SKIP** because the plan is not mirrored into the role repo.

This increment delivers the **near-term slice of the bidirectional MCP control
plane** (D-004): role commands **READ** plan state from the spec repo, and a
role/external client can **PUSH** a workflow transition via a new,
confirm-gated MCP mutation tool `sdd.transitionApply`.

## 2. Scope

**Included**

- **A. Cross-repo plan-state resolution.** A small helper resolves the spec repo
  path from a role repo (topology `specRepo` + `loadLocalBindings` /
  `resolveRepoPath`) and reads the plan's workflow-state + `allowedWriteScope`
  from **there**. Wired into `axiom-role`'s `checkPlanIsApproved` and the per-role
  review (`_role-review.ts`).
- **B. `sdd.transitionApply` MCP tool.** A MUTATION tool in the `sdd` tool-set
  that applies a workflow-state transition, gated by the same
  preview/execute/`confirmed` contract as `axiom app`.

**Non-goals (explicitly excluded)**

- **NO git/script side-effects.** `roleBranch` / `commitSync` / `gitSync` and the
  `script/action` side-effect variant are **P3** — not built here.
- No live stdio MCP client wired into the CLI (see Control-plane model §5).
- No change to the workflow state-machine graph, the artifact schema, or the
  existing read tools' contracts.
- No general-spec (`06_*.md`) rewrite here — that happens at the epic's end.

## 3. Acceptance criteria

- **AC1** — Multi-repo, plan APPROVED in the spec repo: `axiom-role start` from a
  role repo **PASSES** its plan-approval gate (reads spec state, not local).
- **AC2** — Multi-repo, plan NOT approved in the spec repo: the gate **BLOCKS**
  with the existing SAFETY GATE message.
- **AC3** — Multi-repo, per-role review at `complete` reads the plan's
  `allowedWriteScope` from the spec repo (no longer an automatic SKIP when the
  plan is not mirrored locally).
- **AC4** — Single-repo / no topology / spec repo not present on disk: behavior is
  **unchanged** (local read) — the 2409-test suite stays green.
- **AC5** — `sdd.transitionApply` without `confirmed:true` returns a **PREVIEW**
  (workflowId, command, from→to, declared side-effects) and mutates nothing.
- **AC6** — `sdd.transitionApply` with `confirmed:true` **applies** the transition
  (persists the new workflow-state record). An illegal transition returns the
  typed `invalid-transition` error. A cross-project `projectRoot` arg is rejected
  by `pinned()`.
- **AC7** — `axiom mcp serve --kind sdd` lists `sdd.transitionApply` in
  `tools/list`.
- **Gate** — `npm run build && npm test && npm run typecheck && npm run doctor` in
  `Axiom/` all green.

## 4. Corner cases

- Spec repo unresolvable (no binding/path, or the resolved dir does not exist) →
  clean fallback to local, no crash.
- Role repo IS the spec repo (degenerate) → local read (identical result).
- `sdd.transitionApply` on a terminal state / unknown command → typed
  `invalid-transition`; unknown workflow → `unknown-workflow`.

## 5. Control-plane model

The MCP surface is a **bidirectional control plane** (D-004), not read-only.

### 5.1 Reading plan state from the spec repo — resolution order

1. **MCP endpoint (external clients).** The reads are exposed as MCP tools
   `spec.planRead` and `sdd.allowedWriteScopeRead`. An external agent pointed at a
   spec-scoped server reads plan state over JSON-RPC. (No change to these tools'
   contracts is needed — they already read the spec scope.)
2. **Bindings direct-read (CLI primary).** The Axiom CLI itself uses the topology
   **bindings direct-read** path: resolve the spec repo (`topology.specRepo` +
   `loadLocalBindings` / `resolveRepoPath`) and read its
   `.axiom-state/local/workflow-state.json` + plan `metadata.yml` directly.
   *Why bindings, not a live MCP client, as the CLI's primary:* wiring a live
   stdio MCP client into `checkPlanIsApproved` / `runRoleReview` would force those
   (and their callers `runRoleSubcommand` / `app-api.executeSubcommand`) to become
   async — a large blast radius that risks the green suite. Bindings direct-read
   is synchronous, robust, and reuses the exact resolution primitives the MCP
   server context itself uses (`resolveMcpServerContext`). **D-CRMW-001.**
3. **Local-only fallback.** Single-repo, no topology, or a spec repo that does not
   resolve / does not exist on disk → read local (today's behavior). **D-CRMW-002.**

Engagement rule (D-CRMW-002): cross-repo read is used **only** when topology mode
is `multi-repo` AND the resolved spec repo path exists on disk AND differs from
the role repo's own root; otherwise local.

### 5.2 Pushing a transition — `sdd.transitionApply`

- **Contract** mirrors `axiom app` preview/execute/`confirmed`:
  - Input: `{ workflowId, command, confirmed?, vars? }` (+ `projectRoot`, pinned to
    the server's bound project by `pinned()`).
  - Without `confirmed:true` → **PREVIEW**: `{ mode: 'preview', workflowId, command,
    fromState, toState, requiresApproval, declaredEffects, persisted: false }`.
    **Nothing is mutated.**
  - With `confirmed:true` → **APPLY**: reuses the workflow engine (`applyTransition`
    to validate/derive `to`, then `saveWorkflowState` to persist the new
    `WorkflowStateRecord`). `declaredEffects` surfaces the P0 side-effect descriptor
    (`{ localYaml, tracker }`) for transparency.
  - Illegal transition → typed `invalid-transition` error (from the state machine);
    unknown workflow → `unknown-workflow`.
- **Scope (D-CRMW-003):** transition-apply ONLY. It persists the workflow-state
  record (exactly what the role-side gate reads). It does NOT itself mutate artifact
  `metadata.yml` nor run git/script side-effects (P3).
- **Isolation:** the input builder resolves `projectRoot` through the existing
  `pinned()` chokepoint, so a caller-supplied foreign `projectRoot` is rejected with
  the standard cross-project message.

## 6. Files touched

- **A. Cross-repo read**
  - `apps/cli/src/commands/_cross-repo-plan.ts` (new) — `resolveSpecRepoPlanRoot`,
    `planStateReadRoot`.
  - `apps/cli/src/commands/axiom-role.ts` — `checkPlanIsApproved` reads from the
    resolved spec-repo root.
  - `apps/cli/src/commands/_role-review.ts` — `loadActivePlan` reads from the
    resolved spec-repo root; skip message reflects the read source.
- **B. `sdd.transitionApply`**
  - `packages/mcp-tools/src/transition-handlers.ts` (new) — the handler.
  - `packages/mcp-tools/src/registry.ts` — register `sdd.transitionApply`.
  - `packages/mcp-server/src/{tool-sets.ts,input-builders.ts}` — description +
    `buildTransitionApply` (pinned).
  - `packages/mcp-tools/examples/{capabilities,providers}.example.yaml` — add the
    new capability id (round-trip routing test).
- **Tests**: `apps/cli/tests/_cross-repo-plan.test.ts`,
  `apps/cli/tests/axiom-role-cross-repo.test.ts`,
  `packages/mcp-tools/tests/transition-handlers.test.ts`,
  `packages/mcp-server/tests/server.test.ts` (transitionApply cases),
  `packages/mcp-tools/tests/registry.test.ts` (count 19 → 20).

## 7. Resultado

**Green.** Gate in `Axiom/`: `npm run build` clean · `npm test` **2431 passed / 0
failed** (229 files; +22 new tests) · `npm run typecheck` clean · `npm run doctor`
**PASS** (0 failures).

Verified end-to-end in a real 3-repo scratch workspace:

- **AC1** — Plan approved in the spec repo; `axiom-role start` run from the
  backend role repo returned **exit 0** ("transitioned draft → in-progress"). The
  backend repo's local `workflow-state.json` held only the `role` record (no
  `plan`), proving the plan-approval gate read the spec repo cross-repo.
- **AC5/AC6/AC7** — Over a live `axiom mcp serve --kind sdd`: `tools/list`
  returned 8 tools including `sdd.transitionApply`; a preview call
  (`increment-create`, no `confirmed`) returned `{ mode: 'preview', draft →
  specifying, persisted: false }` and wrote nothing; the confirmed call persisted
  the transition (the spec repo's `workflow-state.json` gained the `increment`
  record at `specifying`); an illegal `plan-approve` on an already-approved plan
  returned the typed `invalid-transition` error (`isError: true`).

The CLI's actual cross-repo primary is the topology **bindings direct-read** path
(D-CRMW-001); the same reads are exposed over MCP for external clients. Single-repo
/ unresolvable / absent-spec-repo cases fall back to local — no regression.

## 8. Human validation

- Responsable: `igutierrezz@hiberus.com`.
- Fecha: 2026-07-11.
- Veredicto: implementación con gate `build && test && typecheck && doctor` verde.
