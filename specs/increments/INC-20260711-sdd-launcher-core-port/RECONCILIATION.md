# Reconciliation — sdd-launcher port vs. the existing `@axiom/workflow` lifecycle

## The decision (Q-001)

**Recommendation: EXTEND `@axiom/workflow` (+ `@axiom/core`) as the "sdd-core".
Do NOT build a parallel `@axiom/sdd-core` and migrate.**

The owner's brief describes a target `@axiom/sdd-core` "pure Node, zero editor deps,
behind a `FileSystem` port: action catalog, config/project model, registry discovery,
prompt engine, YAML state engine, guided state machine". Reading the actual code
shows that **most of that already exists and is already pure-Node with zero editor
deps** — it is `@axiom/workflow` plus a handful of sibling packages. Creating a second
package with the same responsibilities would duplicate the artifact lifecycle the
task explicitly warns against duplicating.

## Head-to-head

| sdd-core responsibility (as requested) | Already in Axiom? | Where | Verdict |
|---|---|---|---|
| YAML state engine (kill the triplicated hand-rolled upsert) | **Yes** | `artifact-store.ts` uses `js-yaml`, atomic tmp+rename, one code path | Already collapsed — the triplication is a *KVP25* problem, not Axiom's |
| Guided state machine (minus tracker calls) | **Yes** | `state-machine.ts` (generic) + `default-workflows.ts` (5 workflows, 9 states) | Extend: add declared per-transition side-effects + `recommendNext` |
| Action catalog (families × modes × form fields) | **No** | — | New (`@axiom/launcher`, ported from `launcherDefinitions.ts`) |
| Prompt engine | **No** | — | New (`@axiom/launcher`, ported from `promptBuilder.ts`) |
| Config/project model + registry discovery | **Yes** | `@axiom/project-resolution`, `@axiom/user-workspace`, `@axiom/topology` | Reuse as-is |
| `FileSystem` port (NodeFs) | **Partial** | raw `fs` in `artifact-store.ts`; `@axiom/filesystem-truth` does discovery only | Add small port (P0.5, optional) |
| Canonical structure generator | **Partial** | `makeInitial*Metadata` (full) + `ensureArtifactReadme` (`# title` only) | Extend to skeleton the full template set + role plans |
| Stable local IDs (tracker-less allocator) | **Yes** | `generateArtifactId` / `generateUniqueArtifactId` | Reuse — this *already solves* the "local ID allocator" the brief flags as needed |
| External tracker refs | **Yes** | `ExternalRef { provider, type, id, url }` on every artifact | Reuse as the tracker persistence shape |
| Write-scope validation | **Yes** | `write-scope.ts` + `validate-changes.ts` + doctor WS checks | Reuse; extend `validate` with transition-graph checks |
| Local server + front-end host | **Yes** | `axiom app` (`app-api.ts` + `apps/cli/static/*`), preview/execute/confirm | Reuse the server; re-home the richer front onto it |
| Decoupled optional plugin model | **Yes** | `app-plugins.ts` (`AppPlugin` schema + loader) | Reuse |
| Azure DevOps plugin | **Stub** | `app-plugins-azure-devops.ts` — declarative only, API is TODO | **Fill** with the ported `adoWorkflowService.ts` behind `IWorkItemTracker` |
| MCP surface | **Yes** | `axiom mcp serve` + `@axiom/mcp-tools/*` handlers | Reuse; wire launcher/skills in P4 |

**Score: ~9 reuse, ~2 extend, ~2 fill-a-stub, ~2 genuinely new.** That distribution is
what makes "extend" the correct call over "build parallel".

## Where the two models differ (and how they converge)

1. **Lifecycle vocabulary.** Axiom uses 9 machine-friendly `WorkflowState`s
   (`draft, specifying, planned, plan-approved, in-progress, verifying, archived,
   failed, cancelled`). sdd-launcher uses ranked emoji-prefixed Spanish literals
   (`En Borrador → Especificado → Planificando → Planificado → En Desarrollo →
   En Validación → Implementado → Integrado`; terminal `Descartado`), compared
   string-exact. **Converge via a static mapping table** (P0.4), not by changing
   either enum. `Especificado`/`Planificando` collapse onto `specifying`/`planned`;
   `Integrado` maps to Axiom's separate `integration.status: integrated` field
   (already present in `metadata.yml`), keeping the workflow state and the
   integration state on distinct axes — which is *more* correct than sdd-launcher's
   single ranked string.

2. **State machine location of side-effects.** sdd-launcher's
   `guidedWorkflowService` mutates local YAML **and** the ADO work-item **in
   lockstep, inline**, inside each workflow branch. Axiom's state machine is a **pure**
   function (`applyTransition`), with side-effects pushed to the CLI wrappers and a
   `hooks.ts` engine. **Converge by declaring the side-effect** (local mutation +
   optional named tracker hook) **on the transition** (P0.2), so the pure engine
   stays pure and the tracker call becomes an injected, optional hook — which is
   exactly the decoupling the owner asked for.

3. **Richness.** sdd-launcher is ADO-aware and prompt-aware; Axiom is
   lifecycle/isolation/capability-aware. The union is: **Axiom's backbone + the two
   sdd-launcher capabilities Axiom lacks (prompt engine, launcher catalog) + the one
   Axiom stubbed (real ADO client)**. Nothing in Axiom's richer capability/provider/
   telemetry model is regressed.

4. **Operational-canonical `plan.metadata.yml`.** Both agree plans carry an
   operational-canonical metadata file with `roles.<family>.{applies,status,planFile,
   adoTaskId}`. Axiom already models `PlanMetadata.roles` (`required` + `roleFiles[]`
   with per-role `status`/`file`) and `ExternalRef`; the `adoTaskId` becomes an
   `ExternalRef` on the role entry rather than a bespoke field. Keep
   `plan.metadata.yml` operational-canonical, as the owner requires.

## Migration posture

No data migration is required for Axiom itself (it is the target, and its
`metadata.yml` schema is a superset). The only "migration" is the **one-time port**
of code from the read-only KVP25 extension, plus the sdd-launcher→Axiom **vocabulary
mapping table** applied by `axiom normalize` when/if a KVP25 project is ever brought
into Axiom (out of scope here — this increment does not migrate KVP25 content, only
ports capabilities).

## Net recommendation

- Treat `@axiom/workflow` (+ `@axiom/core`) as the sdd-core. Extend it (P0–P1).
- Add **two** net-new packages only: `@axiom/launcher` (prompt engine + action
  catalog + adapter routing) and `@axiom/tracker` (`IWorkItemTracker` + `NullTracker`)
  with `@axiom/tracker-ado` as its first impl.
- **Fill** the declarative ADO plugin stub rather than inventing a new integration
  surface.
- **Reuse** the `axiom app` server + static-PWA host + plugin/preview/execute/confirm
  protocol as the transport for the re-homed front-end.
- Keep VSCode as one optional `Launcher`/adapter target — never a requirement.
