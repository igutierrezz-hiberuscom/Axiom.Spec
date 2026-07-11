# Increment: P4 — Adapter-agnostic prompt engine + routing table + MCP/skill integration

> **Código**: INC-20260711-sdd-launcher-p4-launcher
> **Estado**: archived (gate green — build / 2499 tests / typecheck / doctor PASS)
> **Fecha de creación**: 2026-07-11
> **Tipo de cambio**: Nueva capacidad (net-new package)
> **Epic**: `INC-20260711-sdd-launcher-core-port` — implementa la **Fase P4** del `PLAN.md`.
> **Referencias externas (READ-ONLY)**: `C:/repos/KVP25 Workspace/Kvp.Sdd/tools/sdd-launcher/src/`
> — `services/promptBuilder.ts`, `adapters/common/launcherDefinitions.ts`, `services/agentRoutingService.ts`.

---

## 1. Goal

Craft prompts against **whichever adapter's commands/skills the user chose**. Deliver the two
genuinely-absent capabilities identified in the epic gap list (§2.3 items 1–3):

1. A **pure prompt engine** (`buildPrompt`) — Axiom had no prompt-crafting capability.
2. A **families × modes action catalog** — the declarative form-field cross-product.
3. An **adapter-agnostic routing table** — generalizing `agentRouting.actions` so the same
   action produces a correct prompt for Claude Code (skill / slash-command), GitHub Copilot
   (chat participant) or plain CLI (`axiom <cmd>`), differing **only** in the header/mention
   line and the launch verb.

Wire the routing table to Axiom's **real** surfaces (`@axiom/skills`, `@axiom/mcp-tools`,
`@axiom/workflow`) so a crafted prompt targets something that actually exists.

## 2. Scope

### Incluido

- New package `@axiom/launcher` (`packages/launcher`), wired into root `package.json`
  (workspaces), `tsconfig.json` (refs) and `vitest.config.ts` (alias), plus `npm install`.
- **Prompt engine** — `buildPrompt(action, values, options)` ported from `promptBuilder.ts`
  as a pure, side-effect-free, deterministic function set (`buildReviewPrompt`,
  `buildTechnicalRoleDocPrompt` included). `ResolvedLauncherContext` and `OperationGuide`
  ported as plain **data** interfaces (the FS-resolving services are NOT ported — that
  resolution belongs to P3's server layer).
- **Action catalog** — `createLauncherActions(projectLabel, roles)` + `AXIOM_LAUNCHER_ROLES`
  ported from `launcherDefinitions.ts` (families `{incremento,bug,plan,back,front,e2e}` ×
  modes `{new,execute,close,archive}` with `requiredFields`/`optionalFields`/conditional
  `visibleWhen`), plus a pure `isFieldVisible` / `visibleFields` evaluator.
- **Adapter-agnostic routing table** — `AXIOM_ADAPTER_ROUTING` +
  `craftPrompt(action, values, context, adapterId)` returning a structured
  `CraftedPrompt { adapterId, header, launchMechanism, launchVerb, target, prompt, ... }`.

### Excluido (Non-goals)

- **No front-end** (that is P3 — the re-homed `launcher.js` + transport shim).
- **No ADO / tracker** work (that is P2 — already shipped).
- No FS path resolution service (P3 consumes the pure engine; context arrives pre-resolved).
- No mutation of the KVP25 source extension (read-only reference).

## 3. Routing model

### 3.1 Family/mode → Axiom workflow reconciliation (D4-004)

The ported KVP25 catalog literals are preserved verbatim (no source rename). Reconciliation
to Axiom's 5 workflows (`increment`, `bug`, `plan`, `role`, `qa-e2e`) lives in a
`FAMILY_TO_WORKFLOW` map + a reconciliation table — it never mutates the port.

| Launcher family | Axiom workflow | Note |
|---|---|---|
| `incremento` | `increment` | Spanish literal preserved; mapped, not renamed. |
| `bug` | `bug` | direct |
| `plan` | `plan` | direct |
| `back` | `role` | the single generic `role` workflow; repo-role distinguishes back/front |
| `front` | `role` | idem |
| `e2e` | `qa-e2e` | direct |

Mode → transition (monotonic; only an exact `DEFAULT_WORKFLOWS` transition is recorded as
`workflowCommand`, so the cross-check test can prove it is real):

| Action key | workflowCommand | CLI invocation |
|---|---|---|
| `increment-new` | `increment-create` | `axiom-increment create` |
| `increment-execute` | `increment-refine` | `axiom-increment refine` |
| `increment-close` | `increment-specify` | `axiom-increment specify` |
| `bug-new` | `bug-create` | `axiom-bug create` |
| `bug-execute` | `bug-fix-plan` | `axiom-bug fix-plan` |
| `bug-close` | `bug-verify` | `axiom-bug verify` |
| `plan-new` | *(none — CLI scaffold)* | `axiom-plan create` |
| `plan-execute` | *(none — CLI scaffold)* | `axiom-plan create` |
| `plan-close` | `plan-approve` | `axiom-plan approve` |
| `back-new` / `front-new` | `role-start` | `axiom-role start` |
| `back-execute` / `front-execute` | `role-apply` | `axiom-role apply` |
| `back-close` / `front-close` | `role-complete` | `axiom-role complete` |
| `e2e-new` | `qa-e2e-start` | `axiom-qa-e2e start` |
| `e2e-execute` | `qa-e2e-verify` | `axiom-qa-e2e verify` |
| `e2e-close` | `qa-e2e-pass` | `axiom-qa-e2e pass` |
| `e2e-archive` | `qa-e2e-pass` | `axiom-qa-e2e pass` (launcher's separate "archive" collapses into pass→archived) |

`plan-new`/`plan-execute` carry a CLI target with **no** `workflowCommand`: plan authoring is
an `axiom-plan create` scaffold, not a workflow state transition.

### 3.2 Per-adapter table shape (D4-002)

```
AdapterRoutingTable {
  defaultAgentMention: string;                 // global fallback (adapter id absent)
  adapters: AdapterRoutingEntry[];
}
AdapterRoutingEntry {
  id: string;                                  // adapter id (matches @axiom/adapters-*)
  promptDialect: 'slash-command'|'chat-mention'|'shell';
  launchMechanism: 'skill-invocation'|'chat-participant'|'cli-exec'|'clipboard';
  defaultAgentMention: string;                 // per-adapter fallback (action unmapped)
  routingMap: Record<actionKey, RoutingTarget>;
}
RoutingTarget { kind:'skill'|'command'|'mcp-tool'; id: string; workflowCommand?: string; mcpTool?: string }
```

Adapters covered: **`claude-code`** (slash-command → `/axiom-sdd-orchestrator`,
skill-invocation), **`github-copilot`** (chat-mention → `@axiom`, chat-participant), and
**`cli`** (shell → `$ axiom <cmd>`, cli-exec). Every target additionally records the
`sdd.transitionApply` MCP tool for the eventual confirmed-mutation launch path (P2/P3).

### 3.3 Wiring to real ids (D4-003)

- **CLI** → `workflowCommand` cross-checked against `@axiom/workflow`'s `DEFAULT_WORKFLOWS`.
- **Claude Code / Copilot** → catalogued `@axiom/skills` skill id (`axiom-sdd-orchestrator`),
  resolvable via `getSkillById`.
- **MCP** → `@axiom/mcp-tools` `MCP_TOOL_CAPABILITY_IDS` (`sdd.transitionApply`, `spec.*Read`).

## 4. Acceptance (from PLAN.md P4)

- **A1** — The SAME action produces a correct prompt for ≥2 adapters (Claude Code skill
  invocation vs. Copilot chat participant) differing **only** in the header/mention line
  and the launch verb; snapshot-tested per adapter (mirrors the `routing-snapshot` pattern).
- **A2** — Selecting an adapter with **no routing entry** falls back to `defaultAgentMention`
  (behavior preserved from `agentRoutingService`); unit-tested.
- **A3** — A conditional field appears **only** when its `visibleWhen` holds; unit-tested.
- **A4** — The prompt engine is **pure/deterministic** (same inputs → same output; snapshot).
- **Gate** — `npm install` → `npm run build` → `npm test` → `npm run typecheck` →
  `npm run doctor` in `Axiom/` all green; no existing test weakened; new package trips no
  doctor check.

## 5. Result

**Green.** `@axiom/launcher` created and wired (workspaces + tsconfig refs + vitest alias +
`npm install`). Prompt engine + action catalog ported pure; adapter-agnostic routing table
covers `claude-code`, `github-copilot`, `cli`, wired to real workflow/skill/MCP ids. All
acceptance criteria met with dedicated tests (snapshot per adapter, adapter fallback,
`visibleWhen`, determinism). Gate green — see this increment's execution report for totals.

## 6. Trazabilidad

- Source (read-only): `KVP25/.../sdd-launcher/src/services/promptBuilder.ts`,
  `adapters/common/launcherDefinitions.ts`, `services/agentRoutingService.ts`.
- Axiom evidence: `packages/launcher/src/*`; `packages/workflow/src/default-workflows.ts`
  (`DEFAULT_WORKFLOWS`); `packages/mcp-tools/src/registry.ts` (`MCP_TOOL_CAPABILITY_IDS`);
  `packages/skills/src/catalog.ts` (`getSkillById`); `Axiom/axiom.config/skills-catalog.yaml`
  (`axiom-sdd-orchestrator`).
