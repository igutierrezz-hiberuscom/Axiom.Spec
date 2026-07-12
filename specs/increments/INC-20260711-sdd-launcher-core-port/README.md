# Increment: Port sdd-launcher capabilities into Axiom core

> **Código**: INC-20260711-sdd-launcher-core-port
> **Estado**: Design/epic ACTIVO (umbrella). Todas las fases entregadas 2026-07-11 — P0 (`INC-20260711-sdd-launcher-p0-core-generator`, archivado), P1 (`INC-20260711-sdd-launcher-p1-cli-subcommands`, archivado), P2 (`INC-20260711-sdd-launcher-p2-tracker`, archivado), P4 (`INC-20260711-sdd-launcher-p4-launcher`, archivado), PX (`INC-20260711-cross-repo-mcp-wiring`, archivado); P3 (`INC-20260711-sdd-launcher-p3-front-server`, archivado) core + largo plazo mantenido entregado — el push channel (SSE) y el mapeo execute `plan-new`/`plan-execute` se entregaron en `INC-20260711-front-longtail`; **cost-dashboard + deployment-trace DESCARTADOS (fuera de Axiom, decisión de owner 2026-07-11 — no quedan como pendientes)**. **Restante**: git-services/`script/action` side-effect + superficie ADO periférica + paneles del front ADO-suggestions/role-branch/commit-sync.
> **Fecha de creación**: 2026-07-11
> **Tipo de cambio**: Nueva capacidad + Reconciliación de arquitectura
> **Referencias externas**: `C:/repos/KVP25 Workspace/Kvp.Sdd/tools/sdd-launcher` (read-only source)
> **RF afectados**: consolidar en `04_Flujos_SDD_y_Ciclo_de_Vida.md`, `05_Interfaces_Operativas.md`, `06_Integraciones_y_Capacidades.md`

This is a **design spec + phased plan only**. No production code is written, no
package is scaffolded, and the source extension is treated as read-only. The
executable work is decomposed in `PLAN.md`; the convergence strategy with the
existing `@axiom/workflow` lifecycle is in `RECONCILIATION.md`.

---

## 1. Resumen

The owner wants the capabilities of `sdd-launcher` (a VSCode extension: scripted
management of increments/bugs/plans, validation, archiving, and Azure DevOps
integration) **re-homed into Axiom** so that:

1. Everything executes from the **core** (a CLI/library), not the editor.
2. The **Azure DevOps** integration is a **decoupled, optional plugin** that
   works with or without it.
3. The **front-end runs from anywhere** (not VSCode-bound) and helps **craft
   prompts** to run against whichever adapter's commands/skills the user chose.
4. Increments/bugs/plans get a **uniform structure generated mostly by scripts**,
   so the AI invents nothing structural.
5. There is **explicit state management**.

The central finding of this design — established by reading the actual Axiom
`packages/*` and `apps/cli/*` code, not the package names — is that **Axiom has
already re-homed roughly 70% of this architecture**, but as thinner, stubbed, or
differently-shaped variants. Therefore this is a **reconciliation-based port**
(extend what exists, fill the stubs, build only the two genuinely-absent
capabilities), **not a greenfield build of a new `@axiom/sdd-core`**. See
`RECONCILIATION.md` for the head-to-head and the recommendation.

## 2. Contexto y motivación

### 2.1 What sdd-launcher is today (ground truth, spot-verified)

- ~31k LOC total. It is a **thin VSCode shell over ~90% portable logic**; only
  ~2.4k LOC is genuinely editor-bound (`extension.ts` webview host,
  `workflowStateService` memento, `launchBridge.ts`).
- The real surface is two `WebviewPanel`s driven by a clean JSON message protocol
  (~26 message types), not the 3 registered VSCode commands.
- The capability catalog is **data-driven**:
  `src/adapters/common/launcherDefinitions.ts` (606 LOC) builds a cross-product of
  families `{incremento,bug,plan,back,front,e2e}` × modes `{new,execute,close,archive}`
  with declarative form-field definitions (`LauncherActionDefinition` +
  `LauncherFieldDefinition`).
- **Structure is duplicated**: live creation uses hardcoded TS string templates
  (`guidedWorkflowService.ts` `scaffoldIncrement`/`scaffoldBug`), while
  `scripts/*.mjs` re-encode the same schema (`migrate-spec-metadata` 1059 LOC =
  best schema reference; `normalize-kvp25-tracking-metadata` 640 = lifecycle state
  machine + `syncStatus` rule; `integrate-spec-item` 338 = archive+transition). A
  hand-rolled YAML upsert primitive is **triplicated**. Plans are **not**
  skeleton-generated today (the AI authors them).
- **ADO**: `adoWorkflowService.ts` (2211 LOC, ~2140 VSCode-free) — PAT/Basic auth,
  raw `https` to `dev.azure.com` `_apis` v7.1 (WIT/Build/Git). It touches VSCode in
  exactly 2 places (`context.secrets`, `showInputBox`). Local artifact IDs are
  currently derived from the ADO work-item id (so a tracker-less mode needs a local
  ID allocator).
- **Front**: framework-free vanilla JS/CSS (`media/launcher.js` ~5359 LOC). It
  composes and previews the prompt locally (`promptBuilder.ts` 579 LOC) but
  delegates the actual send to the host (`launchBridge.ts` → clipboard +
  `vscode.editorChat.start`). ~90-95% reusable behind a ~100-line transport shim
  (replace `acquireVsCodeApi()` with fetch/WebSocket to a local server).
- **Guided chaining**: `guidedWorkflowService.ts` (1982 LOC) mutates local YAML +
  ADO work-item in lockstep and returns a `nextFamily`/`nextMode` recommendation
  to chain the next step.

### 2.2 What Axiom already has (verified by reading the code)

| Owner requirement | Axiom already ships | File(s) |
|---|---|---|
| Core executes outside the editor | `@axiom/workflow` is **pure Node, zero editor deps** | `packages/workflow/src/*` |
| Explicit state management | Generic declarative state machine + 5 workflows + 9 states + `workflow-state.json` | `state-machine.ts`, `default-workflows.ts`, `workflows-loader.ts`, `state-store.ts` |
| Uniform artifact structure | Folder-per-instance `metadata.yml` store (increment/bug/plan/adr/decision), atomic writes, archive, list | `artifact-store.ts` |
| Local IDs independent of tracker | `generateArtifactId` / `generateUniqueArtifactId` (`INC-YYYYMMDD-HHMMSS-<suffix>`) | `artifact-id.ts` |
| External tracker references | `ExternalRef { provider, type, id, url }` persisted on every artifact | `artifact-store.ts` |
| Validation / write-scope | `validateWriteScope` + `axiom validate-changes` + `@axiom/doctor` WS checks | `write-scope.ts`, `validate-changes.ts` |
| CLI structural commands | `axiom-increment / -bug / -plan / -role / -qa-e2e / -adr / -decision` | `apps/cli/src/commands/*` |
| Front-end running from anywhere | `axiom app`: local HTTP server + **static PWA** (no framework, no Chromium) + preview/execute/**confirm** protocol | `app-api.ts`, `apps/cli/static/{index.html,app.js,style.css,sw.js,manifest.json}` |
| Decoupled optional plugins | Declarative `AppPlugin` loader + form/action schema | `app-plugins.ts` |
| Azure DevOps as a plugin | **Declarative plugin exists** (form + `axiom external-sync azure-devops …` commands) | `app-plugins-azure-devops.ts` |
| MCP surface | `axiom mcp serve` (JSON-RPC/stdio) + `@axiom/mcp-tools` handlers (artifact/registry/skills/context) | `mcp-serve.ts`, `packages/mcp-tools/src/*` |

### 2.3 The gaps (what the port actually adds)

1. **Prompt engine** (`promptBuilder.ts`) — Axiom has **no** prompt-crafting
   capability. The grep for prompt/launcher hits only TUI wizard prompts and
   *model*-routing; none of it composes a prompt for an agent.
2. **Launcher action catalog** (`launcherDefinitions.ts`) — the families×modes
   cross-product with form-field defs has no Axiom equivalent.
3. **Adapter-agnostic prompt routing** (`agentRoutingService.ts`) — Axiom's
   "adapters" generate files and route *models*, not prompt dialects / launch verbs.
4. **Rich prompt-crafting front-end** — Axiom's `axiom app` PWA is a thin operator
   console, far simpler than sdd-launcher's 5359-line `launcher.js`.
5. **Real ADO client** — Axiom's ADO plugin is explicitly a **declarative stub**
   ("la ejecución real con la API de Azure DevOps queda como TODO").
   sdd-launcher's `adoWorkflowService.ts` is the real, working implementation.
6. **Skeleton generation of full artifact structure** — `axiom-increment create`
   today writes a complete `metadata.yml` but only a `# title` README
   (`ensureArtifactReadme`), and **does not** skeleton the template sections
   (`01_Requisitos`, `03_Criterios_Aceptacion`, conditional `02`/`04`, `context/`)
   nor role-plan files. The "scripts generate structure, AI invents nothing"
   requirement is only half-met.
7. **Guided next-step recommendation** — Axiom's state machine exposes
   `availableTransitions` but no `nextFamily`/`nextMode` chaining UX.
8. **FileSystem port** — `artifact-store.ts` uses raw `fs`. A `FileSystem` port
   (NodeFs impl) does not exist; it is polish, not a blocker (writes are already
   atomic tmp+rename).

## 3. Alcance

### Incluido

- Formalize `@axiom/workflow` (+ `@axiom/core`) as the canonical **sdd-core** and
  close its structural gaps (canonical generator, declared per-transition
  side-effects, guided next-step).
- A **single canonical structure generator** that pre-fills every structural key
  of `metadata.yml` / `plan.metadata.yml` / role-plan files from the
  `Axiom.Spec/templates/*` — one source of truth, AI fills prose only.
- An **`IWorkItemTracker` port** with a **`NullTracker`** (local-only default) and
  an **`@axiom/tracker-ado`** implementation ported from `adoWorkflowService.ts`,
  wired **behind the existing declarative ADO plugin surface**.
- A **`@axiom/launcher`** package (prompt engine + action catalog + adapter-agnostic
  routing table) — the two genuinely-absent capabilities.
- **Re-home sdd-launcher's front-end** behind a transport shim, served by the
  **existing `axiom app` HTTP server** (reuse, do not rebuild), with a `Launcher`
  abstraction (`ClipboardLaunch` / `HttpLaunch` / optional `VSCodeLaunch`).
- Explicit **lifecycle-vocabulary reconciliation** (mapping table, non-breaking).

### Excluido

- No production code, no package scaffolding, no build run.
- No modification of the source extension (read-only).
- No re-litigation of Axiom's capability/provider/model-routing/telemetry model.
- No new editor extension. VSCode is one optional `Launcher`/adapter target, not a
  requirement.
- No rewrite of the existing `@axiom/workflow` state machine — it is extended.

## 4. Target architecture (reconciled)

Owner-requested names on the left; where they live after reconciliation on the right.

| Requested target | Reconciled home | New or existing |
|---|---|---|
| `@axiom/sdd-core` (pure core) | **Extend `@axiom/workflow` + `@axiom/core`** | Existing (extend) |
| `FileSystem` port + NodeFs | `@axiom/core` (or `@axiom/filesystem-truth`) | New (small) |
| Canonical structure generator | New module in `@axiom/workflow` (`scaffold`) driving `Axiom.Spec/templates/*` | Existing store + new generator |
| Explicit state model + `axiom state` | Extend `workflows.yaml` engine with declared per-transition side-effects + guided next-step; add `axiom state` inspector | Existing (extend) |
| `IWorkItemTracker` + `NullTracker` | New `@axiom/tracker` (port + NullTracker) | New |
| `@axiom/tracker-ado` | New package; port `adoWorkflowService.ts`; inject `SecretStore` + `promptForSecret` | New (ported) |
| Front-end + local server | **Reuse `axiom app` (`app-api.ts` + `apps/cli/static/`)**; re-home `launcher.js` behind a transport shim | Existing server + ported front |
| `Launcher` abstraction | New module in `@axiom/launcher` (`ClipboardLaunch`/`HttpLaunch`/`VSCodeLaunch`) | New (replaces `launchBridge`) |
| Prompt engine (`promptBuilder`) | New `@axiom/launcher` | New (ported) |
| Adapter-agnostic routing table | New `@axiom/launcher`; leans on existing adapter packages | New |

Net-new packages: **two** — `@axiom/launcher` and `@axiom/tracker` (+ its
`@axiom/tracker-ado` impl). Everything else is an extension of an existing
package or a fill-in of an existing stub.

## 5. Key ports and interfaces (to be specified in the phase increments)

- **`FileSystem`** — `readFile / writeFileAtomic / exists / mkdirp / readdir /
  rename`. NodeFs impl wraps the current raw-`fs` calls in `artifact-store.ts`.
- **`IWorkItemTracker`** (~25 methods) — `createWorkItem / getWorkItem /
  updateWorkItem / linkWorkItems / listWorkItems / createChildTask / mapIncrement→US /
  mapBug→Bug / mapRole→Task / …`. `NullTracker` is a total no-op that lets Axiom run
  **local-only** (IDs already come from `generateArtifactId`, so no ADO id is needed).
- **`SecretStore`** + **`promptForSecret`** — remove the 2 VSCode touches in the ADO
  client (`context.secrets`, `showInputBox`). CLI impl reads env/keychain and
  prompts on the terminal; the `axiom app` server impl prompts over HTTP.
- **`Launcher`** — `launch(prompt, adapterTarget): Promise<void>`. Implementations:
  `ClipboardLaunch`, `HttpLaunch(agentEndpoint)`, optional `VSCodeLaunch`.
- **Adapter routing table** — generalize `agentRouting.actions` into
  `{ id, promptDialect, routingMap, launchMechanism }` per adapter (Copilot chat
  participant, Claude Code skill/slash-command, plain CLI). `promptBuilder` stays;
  only the header/mention and launch verb vary per selected adapter.
- **Transition side-effect descriptor** — extend the `workflows.yaml` step shape so
  each transition declares its local-YAML mutation and its optional tracker call,
  moving that logic out of the CLI wrappers into the declared graph. **The
  side-effect taxonomy is extensible to a named *script/action* variant** (owner
  clarification 2026-07-11) — e.g. a transition can declare "run the `role-branch`
  or `commit-sync` git action in the target repo". P0 shipped the `{ localYaml,
  tracker }` shape; the `action`/`script` variant + its runner (git services below)
  land with P2/P3 when there is a consumer (no speculative widening in P0).
- **MCP as a bidirectional control plane** (owner clarification 2026-07-11) — the MCP
  surface must not be read-only. In addition to today's read/validation tools
  (`spec.*Read`, `sdd.allowedWriteScopeRead`), add **action/mutation tools** that (a)
  apply a workflow state transition and (b) **launch a declared script/action in the
  target repo** (git: role-branch, commit, sync). All mutations run **behind the
  existing `axiom app` preview/execute/`confirmed` safety contract** — nothing
  executes without confirmation. This is how a role repo both *reads* plan state from
  the spec repo and *pushes* its transition / triggers its git script cross-repo
  (topology bindings remain a no-server fallback).
- **Git services port** — bring KVP25's `roleBranchService` / `commitSync` /
  `gitSyncService` (currently in the extension) into the core as the concrete
  implementations behind the `script/action` side-effect variant and the MCP action
  tools. Axiom already shells `git` (`gitSyncService` uses `execFile git`), so this is
  a mechanical port behind a small `GitRunner` seam.

## 6. Dudas abiertas

- **Q-001 (blocking)** — Extend `@axiom/workflow` as sdd-core, or build a parallel
  `@axiom/sdd-core`? `RECONCILIATION.md` recommends **EXTEND**.
- **Q-002** — Replace the thin `axiom app` PWA with the re-homed front, or run both?
- **Q-003** — Lifecycle vocabulary: mapping table vs. widening `WorkflowState`.
- **Q-004** — Add the `FileSystem` port now, or defer (it is polish).

## 7. Decisiones funcionales cerradas

| ID | Decisión | Resultado |
|----|----------|-----------|
| D-001 | Design/epic only — no code, no scaffolding | Execution deferred to `PLAN.md`; each phase opens its own increment |
| D-002 | Reuse sdd-launcher's front-end (re-home backend), **not** rebuild, and **not** keep the earlier discardable HTML front | Front-end is ported behind a transport shim onto `axiom app` |
| D-003 | ADO stays optional and decoupled; `NullTracker` is the default | Axiom runs local-only out of the box; ADO is opt-in via `tracker.kind: ado` |
| D-004 | MCP is a **bidirectional control plane**, not read-only (owner 2026-07-11) | Add MCP action tools (apply transition + launch script in target repo) behind the `axiom app` preview/execute/`confirmed` model; the transition side-effect taxonomy gains a `script/action` variant; KVP25 git services (`roleBranch`/`commitSync`/`gitSync`) are ported into core. Scoped into P2/P3. |

## 8. Consolidación en la spec general

On integration, this increment updates: `04_Flujos_SDD_y_Ciclo_de_Vida.md`
(declared per-transition side-effects + guided next-step), `05_Interfaces_Operativas.md`
(re-homed front-end + `Launcher`), and `06_Integraciones_y_Capacidades.md`
(real `IWorkItemTracker`/ADO plugin + `NullTracker`). No standalone `general-spec.md`
is created (per `Axiom.Spec/specs/README.md`).

## 9. Estrategia E2E

`parallel` — Phase P3 (front-end + local server) warrants an E2E lane that drives
`axiom app` end-to-end (open action → craft prompt → preview → confirm →
local-only artifact created), independent of any live ADO instance.

## 10. Effort honesty

- **~11.6k LOC "moves mechanically"** in principle, but much of the lifecycle/state/
  artifact target **already exists in Axiom**. The *net-new* mechanical move is
  smaller and concentrated in: `promptBuilder.ts` (579), `launcherDefinitions.ts`
  (606), `agentRoutingService.ts` (27), the non-ADO parts of `guidedWorkflowService.ts`
  (~1982), and the schema logic from `scripts/*.mjs` (~2037) collapsed into the one
  canonical generator.
- **~3k LOC becomes the ADO plugin** — `adoWorkflowService.ts` (2211) plus supporting
  services, ported behind `IWorkItemTracker` with 2 VSCode touches removed.
- **~7.5k front, ~90% reusable** — `media/launcher.js` (5359) + CSS + webview HTML,
  behind a ~100-line transport shim.

## 11. Trazabilidad y fuentes

- Source (read-only): `C:/repos/KVP25 Workspace/Kvp.Sdd/tools/sdd-launcher/`.
- Axiom evidence: `packages/workflow/src/{types,state-machine,default-workflows,artifact-store,artifact-id,write-scope,index}.ts`;
  `apps/cli/src/commands/{app-api,app-plugins,app-plugins-azure-devops,axiom-increment,mcp-serve,validate-changes}.ts`;
  `apps/cli/static/*`; `packages/core/src/result.ts`; `packages/mcp-tools/src/*`.
- Templates aligned to: `Axiom.Spec/templates/{increment-template,plan-template,role-plan-template,plan-metadata-template,increment-metadata-template}.*`.

## 12. Estado de validación humana

- Responsable: `igutierrezz@hiberus.com`.
- Fecha: 2026-07-11.
- Veredicto: **pendiente de aprobación** (design/epic). No code validation run —
  no code changed. When phases execute, `npm run build && npm test && npm run doctor`
  in `Axiom/` is the validation gate.
