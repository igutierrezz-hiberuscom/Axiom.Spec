# Phased plan — sdd-launcher → Axiom core port

> **Artefacto origen**: `INC-20260711-sdd-launcher-core-port` (increment)
> **Estado**: epic activo (umbrella) — P0/P1/P2/P3/P4/PX shipped y archivados; P3 core + largo plazo mantenido entregado (push channel + mapeo execute `plan-new`/`plan-execute` en `INC-20260711-front-longtail`); cost-dashboard + deployment-trace DESCARTADOS (fuera de Axiom, no pendientes)
> **Versión de spec**: v1 · **Versión de plan**: p1
> **E2E**: `parallel` (materialized in P3)

Each phase is **independently shippable** (own increment, own acceptance gate) and
follows this workspace's reconciliation convention: the first task of every phase is
a **migration-engineer audit** of the named existing package(s) before any contract
is (re)written. "Additive, no legacy package" phases skip the audit.

**Effort key (rough, net-new):** S ≈ ≤300 LOC · M ≈ 300–800 · L ≈ 800–2000 ·
XL ≈ >2000. LOC counts are *net-new/ported* code, not the sdd-launcher totals
(most of the lifecycle target already exists in Axiom).

---

## Phase P0 — Core reconciliation + canonical generator + explicit state (local-only) — ✅ DONE (2026-07-11)

**Shipped** as `INC-20260711-sdd-launcher-p0-core-generator` (archived): canonical
skeleton generator (`artifact-skeleton.ts` + `scaffoldArtifact`, plans get
`plan.metadata.yml` + per-role files), declared per-transition side-effects
(`{ localYaml, tracker }` + `transition-effects.ts` runner), pure `recommendNext`,
lifecycle-vocabulary table. Gate green (2375 tests, typecheck, doctor PASS, DF-001
green). P0.5 (`FileSystem` port) deferred. See that increment for details.

**Goal.** Make `@axiom/workflow` (+ `@axiom/core`) the canonical **sdd-core**: one
structure generator, declared per-transition side-effects, guided next-step. No ADO,
no front-end.

**Work.**
- **P0.1 — Canonical structure generator** (extend `artifact-store.ts`). Replace the
  `# title`-only `ensureArtifactReadme` with a template-driven skeleton emitter that
  pre-fills **every structural key** of `metadata.yml`, the README sections, the
  conditional docs (`02_Cambios_Modelo`, `04_Interacciones_UI`), the `context/`
  folder, and — new vs. sdd-launcher — the **plan `plan.metadata.yml` + role-plan
  MD files**. Single source of truth = `Axiom.Spec/templates/*`. AI fills prose only.
- **P0.2 — Declared per-transition side-effects.** Extend the `workflows.yaml` /
  `WorkflowStep` shape so each transition declares its local-YAML mutation and its
  **optional tracker call** (a named hook, not an inline ADO call). Move the lockstep
  logic out of the CLI wrappers into the declared graph (mirrors sdd-launcher's
  `guidedWorkflowService` but data-driven).
- **P0.3 — Guided next-step.** Port `nextFamily`/`nextMode` as a **pure function**
  over the workflow config (`recommendNext(config, state) → transition[]`).
- **P0.4 — Lifecycle vocabulary mapping.** A static, non-breaking table mapping
  sdd-launcher's emoji-prefixed Spanish literals (En Borrador…Integrado, Descartado)
  onto Axiom's 9 `WorkflowState`s.
- **P0.5 (optional, Q-004) — `FileSystem` port** in `@axiom/core`; NodeFs impl wraps
  the raw-`fs` calls in `artifact-store.ts`.

**Dependencies.** None (foundation). Q-001 resolved (EXTEND vs. parallel).
**Effort.** L (generator M–L; side-effects M; next-step S; mapping S; port S).
**Acceptance.**
- `axiom-increment/bug/plan create` emit the **full** template skeleton (all
  structural keys present, prose placeholders only); a golden-file test asserts the
  generated tree matches `Axiom.Spec/templates/*`.
- `recommendNext` and per-transition side-effects are unit-tested against
  `DEFAULT_WORKFLOWS`; existing `@axiom/workflow` tests still pass.
- No editor dependency introduced (dogfooding boundary check stays green).

---

## Phase P1 — CLI generator subcommands (scaffold / normalize / integrate / validate / state) — ✅ DONE (`INC-20260711-sdd-launcher-p1-cli-subcommands`, archivado)

**Goal.** Expose the P0 generator and state model as first-class, script-invocable
CLI subcommands so the AI invokes structure, never invents it.

**Work.**
- `axiom scaffold increment|bug|plan|e2e` — thin wrappers over the P0 generator
  (reconcile with the existing `axiom-increment create`, which becomes a `scaffold`
  alias).
- `axiom normalize` — port `normalize-kvp25-tracking-metadata.mjs` (640 LOC:
  lifecycle state machine + `syncStatus` rule) as a **pure normalizer** over
  `metadata.yml` / `plan.metadata.yml`.
- `axiom integrate` — port `integrate-spec-item.mjs` (338 LOC: archive + transition);
  reconcile with existing `archiveArtifactDir` + `qa-archive-gate.ts`.
- `axiom validate` — extend existing `validate-changes.ts` to also **validate a
  requested transition against the declared graph** (uses `applyTransition`).
- `axiom state` — inspector over `workflow-state.json` (`loadWorkflowState`) +
  per-artifact `metadata.yml`; prints current state, available + recommended next.

**Dependencies.** P0.
**Effort.** M (mostly wiring; `normalize` is the largest single port, M).
**Acceptance.**
- Each subcommand has a `--dry-run`/preview that prints the intended change without
  writing (consistent with `axiom app`'s preview contract).
- `axiom validate` rejects an illegal transition with the typed
  `invalid-transition` error and lists legal commands.
- Round-trip test: `scaffold → normalize → integrate` on a fixture increment
  produces a schema-valid archived artifact; `@axiom/doctor` reports no new failures.

---

## Phase P2 — `IWorkItemTracker` port + `NullTracker` + `@axiom/tracker-ado` — ✅ DONE (`INC-20260711-sdd-launcher-p2-tracker`, archivado; superficie ADO periférica build/git/sprint/attachment pendiente)

**Goal.** Make the tracker a decoupled, optional plugin. Axiom runs **local-only by
default**; ADO is opt-in and fills the existing declarative stub.

**Work.**
- **P2.1** — Define `IWorkItemTracker` (~25 methods) + `SecretStore` +
  `promptForSecret` ports in a new `@axiom/tracker` package; ship `NullTracker`
  (total no-op). Confirm `ExternalRef` (already persisted) is the only state the
  port needs.
- **P2.2** — Port `adoWorkflowService.ts` (2211 LOC, ~2140 VSCode-free) into
  `@axiom/tracker-ado`, replacing the 2 VSCode touches with the injected ports.
- **P2.3** — Wire `@axiom/tracker-ado` **behind** the existing declarative
  `AZURE_DEVOPS_PLUGIN` + the `axiom external-sync azure-devops …` command names
  (TODO stubs today). Config: `tracker: { kind: 'ado'|'none', enabled, org, project,
  mappings }`.
- **P2.4** — Confirm local-only ID allocation: `generateArtifactId` already yields
  tracker-independent IDs, so no ADO id is required in `NullTracker` mode (removes
  sdd-launcher's ADO-id-derived-local-id coupling).

**Dependencies.** P0 (side-effect hook point), P1 (`integrate` calls the tracker).
**Effort.** XL (the ADO client is the single largest port).
**Acceptance.**
- With `tracker.kind: none`, every workflow completes end-to-end with **no network
  and no ADO id**; artifacts carry an empty `externalRefs`.
- With `tracker.kind: ado`, `increment→User Story`, `bug→Bug`, `roles→child Tasks`,
  `e2e→US+Task` mappings create/link real work items (verified against a test ADO
  project or a recorded-fixture double); `externalRefs` is populated.
- Swapping trackers requires **only** config; no core code path changes.

---

## Phase P3 — Local server + front-end transport shim + `Launcher` — ✅ CORE + KEPT LONG-TAIL SHIPPED (`INC-20260711-sdd-launcher-p3-front-server`, archivado; push channel SSE + mapeo execute `plan-new`/`plan-execute` entregados en `INC-20260711-front-longtail`; cost-dashboard + deployment-trace DESCARTADOS — fuera de Axiom, no pendientes; ADO-suggestions/role-branch/commit-sync git panels PARCIAL)

**Goal.** Run the prompt-crafting front-end **from anywhere**, served by Axiom's
**existing** `axiom app` HTTP server. Reuse sdd-launcher's front-end; do not rebuild.

**Work.**
- **P3.1** — Extend `app-api.ts` to expose sdd-launcher's ~26 message types as
  HTTP/WS endpoints (mapped onto the existing GET-read / POST-execute-with-`confirmed`
  contract; reuse `loadAppPlugins`).
- **P3.2** — Re-home `media/launcher.js` (~5359 LOC, ~90% reusable) behind a
  ~100-line **transport shim** that replaces `acquireVsCodeApi()` with fetch/WebSocket.
  Serve from `apps/cli/static/` (or a second view — Q-002).
- **P3.3** — `Launcher` abstraction replacing `launchBridge.ts`: `ClipboardLaunch`,
  `HttpLaunch(agentEndpoint)`, optional `VSCodeLaunch`.
- **P3.4** — E2E lane (`parallel`): open action → craft prompt → preview → confirm →
  local-only artifact created, with **no live ADO**.

**Dependencies.** P1 (endpoints wrap the CLI subcommands), P4's prompt engine is
consumed here but P4 can land first or concurrently (see note).
**Effort.** L (front-end port is bulk-but-mechanical; server + Launcher are M).
**Acceptance.**
- The front-end runs in a plain browser against `axiom app` with **zero VSCode APIs**
  present; preview shows the exact prompt/command without executing.
- `Launcher` selection is user-visible and testable (clipboard path asserted without
  a running agent).
- The earlier discardable HTML front is **not** carried forward (D-002).

---

## Phase P4 — Adapter-agnostic prompt engine + routing table + MCP/skill integration — ✅ DONE (`INC-20260711-sdd-launcher-p4-launcher`, archivado)

**Goal.** Craft prompts against whichever adapter's commands/skills the user chose.

**Work.**
- **P4.1** — New `@axiom/launcher` package: port `promptBuilder.ts` (579) +
  `launcherDefinitions.ts` (606, the families×modes catalog) + `agentRoutingService.ts`
  (27).
- **P4.2** — Generalize `agentRouting.actions` into a **per-adapter routing table**
  `{ id, promptDialect, routingMap, launchMechanism }` (Copilot chat participant,
  Claude Code skill/slash-command, plain CLI). `promptBuilder` stays; only the
  header/mention + launch verb vary per selected adapter.
- **P4.3** — Wire the launcher to `@axiom/mcp-tools` + `@axiom/skills` so a crafted
  prompt can target the selected adapter's real commands/skills.

**Dependencies.** P0 (action catalog references the state model). Independent of P2.
Can ship before or concurrently with P3 (P3 consumes it; a stub prompt engine
unblocks P3 if sequenced first).
**Effort.** M–L.
**Acceptance.**
- The same action produces a correct prompt for ≥2 adapters (e.g. Claude Code skill
  invocation vs. Copilot chat participant) differing only in header + launch verb;
  snapshot-tested per adapter (mirrors the existing `routing-snapshot` test pattern).
- Selecting an adapter with no routing entry falls back to `defaultAgentMention`
  (behavior preserved from `agentRoutingService`).

---

## Cross-cutting — MCP control plane + git scripts (owner clarification, 2026-07-11) — 🟡 CROSS-REPO WIRING SHIPPED (`INC-20260711-cross-repo-mcp-wiring`, archivado: `sdd.transitionApply` confirm-gated + role gate/review leen estado del spec); git-services/`script/action` side-effect PENDIENTE

The MCP surface is a **bidirectional control plane**, not read-only. This threads through P2/P3:
- **P2** additionally exposes **MCP action tools** that apply a workflow transition and
  invoke a tracker/side-effect — behind the `axiom app` preview/execute/`confirmed`
  contract (nothing runs unconfirmed). The transition side-effect taxonomy (shipped in
  P0 as `{ localYaml, tracker }`) gains a **`script/action` variant**.
- **P3** ports KVP25's **git services** (`roleBranchService` / `commitSync` /
  `gitSyncService`) into the core behind a small `GitRunner` seam, exposed as the
  concrete `script/action` implementations and as MCP action tools, so a role repo can
  **push its transition and launch its own git script cross-repo** (read state from the
  spec repo via MCP; topology bindings as a no-server fallback). See README §5
  (bidirectional MCP, git services) and D-004.
- The near-term **cross-repo wiring** (role commands read plan state from the spec repo
  instead of local-only) is the smallest slice of this and can ship as its own increment
  ahead of the full front-end.

## Sequencing summary

```
P0 ──▶ P1 ──▶ P2         (tracker; optional, ships whenever)
        │
        ├──▶ P4          (prompt engine + routing; independent of P2)
        │
        └──▶ P3          (front-end + server; consumes P4)
```

- **Critical path**: P0 → P1 → P3 (a usable, local-only, prompt-crafting Axiom with
  no ADO). P2 and P4 are parallelizable off P1/P0 respectively.
- **Ship value early**: P0+P1 alone already deliver "scripts generate structure, AI
  invents nothing" + explicit state — the owner's requirements (4) and (5) — with no
  front-end and no ADO.

## Riesgos y dependencias

| ID | Sev | Riesgo | Mitigación |
|----|-----|--------|------------|
| RISK-001 | high | Building a parallel `@axiom/sdd-core` duplicates the tested `@axiom/workflow` lifecycle | Resolve Q-001 = EXTEND before P0 write work (see `RECONCILIATION.md`) |
| RISK-002 | med | ADO client port drags VSCode types in transitively | Port behind `IWorkItemTracker` + `SecretStore`; CI check that `@axiom/tracker-ado` has no `vscode` import |
| RISK-003 | med | Lifecycle vocab mismatch corrupts state on normalize | P0.4 mapping table + `axiom validate` transition gate before any write |
| RISK-004 | low | Front-end coupling to VSCode theming/messaging leaks through the shim | Shim is the only seam; assert no `acquireVsCodeApi`/`--vscode-*` remain (sdd-launcher already avoids theme vars) |

## Validaciones y gates

- **Pre-start (each phase)**: migration-engineer audit note published; Q-001 resolved
  (P0 only).
- **Pre-done**: `npm run build && npm test && npm run typecheck && npm run doctor` in
  `Axiom/` green; new unit/snapshot tests for the phase; dogfooding boundary check
  (no editor dep in core) green.
- **Pre-archive**: E2E lane (P3) passing for the local-only path; integration map in
  `metadata.yml` applied to the three canonical spec docs.
