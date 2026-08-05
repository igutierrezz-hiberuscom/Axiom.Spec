# Historial de incrementos (0015–0030, post-MVP)

Fuente: ADR migrados a `Axiom.Spec/decisions/` (0015, 0019, 0026–0031) y evidencia histórica de los incrementos post-MVP 0015–0030. El MVP (incrementos previos a 0015, incluyendo 0008/0010/0018) cerró el 2026-06-25; este documento cubre la ola post-MVP cerrada el 2026-06-30.

Las rutas y nombres de toolchain que aparecen en las entradas históricas
(`axiom.spec/config/`, `.sdd/`, `CodeGraph` y `Graphify`) describen el estado
del runtime en el momento de aquellos incrementos. El estado actual usa
`axiom.config/`, `.axiom-state/` y `cmm`, como se documenta en los documentos
de arquitectura e integraciones reconciliados el 2026-07-29.

## 0015 — Cavekit Discipline and Optional GGA Adoption

Adopta contratos de capacidad nativos (disciplina Cavekit) en vez de copiar tooling externo: predicados puros con invariantes (`Invariants<T>`: `id`, `level`, `message`, `predicate`), workflow declarativo de backprop (clasifica fallos por severidad: critical/high → `bug-create`, medium/low → `spec-update`, vacío → `noop`), check workflow read-only, GGA opcional (`advisory-first` o `strict`, rollback configurable). Sin comandos nuevos (runtime interno). Package nuevo: `@axiom/cavekit-discipline`. Métricas: 1029/1029 tests verdes, `tsc -b` verde. Archivado 2026-06-30.

## 0019 — Operator Control Plane Runtime

Cierra la brecha entre MVP y una superficie operativa comparable a GentleAI: configuración declarativa versionada en `axiom.spec/config/*.yaml`, estado mutable project-scoped en `.sdd/<project>/`, proyección por adapter target (primer target con cobertura completa: `opencode`). Decisiones: multi-mode routing solo en `opencode` en MVP; fallback `medium` para el resto. Comandos nuevos: `axiom upgrade`, `axiom tui`, `axiom model show/set/unset/reset/validate`, `axiom components list/show/install/uninstall/restore`, `axiom skills list/refresh/drift`. Packages nuevos/extendidos: `@axiom/versioning`, `@axiom/tui`, `@axiom/cli-commands`, `@axiom/model-routing`, `@axiom/components`, `@axiom/skills`. Métricas: 681/681 tests, doctor 40/0 estable. Cerrado 2026-06-30 (5 gaps diferidos al roadmap).

## 0026 — Integration Hardening and Target Parity

Cierra paridad operativa de integraciones project-scoped: elimina flake de `tool-routing`, integra routing real con el adapter opencode, hace `axiom skills apply` idempotente, añade integración TUI para `model validate` y `components show`. Decisiones: `routeTool`/emitters derivan `emittedAt` de `call.requestedAt` para pureza byte a byte; `skills apply` materializa `skills-pending.json` sin tocar el lockfile (D3 read-only respetado); `loadRoutingSnapshot` tolera ausencia/malformación; support matrix extendida con `antigravity` y `visual-studio-2026`. Comando nuevo: `axiom skills apply [--skill <id>...] [--dry-run]`. Métricas: 963/963 tests (+19), `tsc -b` verde, 40/40 doctor-contracts. Archivado 2026-06-30.

## 0027 — Toolchain Provider Expansion and Repair

Completa el lote B de 0023: tools P1 (CodeGraph, Graphify, Headroom/RTK, Caveman, Autoskills) con `mvp: true`, detección dinámica, repair idempotente, selección por repo, políticas `.gitignore` por tool. Bug corregido: `resolveDetectionPath` usaba rutas relativas al `cwd` en vez de a `projectRoot`. Comandos nuevos: `axiom toolchain repair [--id <id>]`, `axiom toolchain add --id <id> --path <repoId>`, `axiom toolchain gitignore [--write <file>]`. Métricas: 983/983 tests (+20). Archivado 2026-06-30.

## 0028 — Workflow UX and Archive Safety Completion

Completa el lote B de 0022: intent commands como chain wrappers, archive-gate visible (warning, no bloqueo), parser centralizado de `workflows.yaml`. Decisiones: intent commands encadenan sub-comandos existentes declarados en `command-protocol.yaml#intentCommandBehavior`; archive-gate retorna warning si `qaLane=parallel` y QA pendiente (GATE 0028, no bloquea); parser tolerante (malformación → warnings, no abort). Ficheros nuevos: `@axiom/workflow/workflows-loader.ts`, `apps/cli/src/commands/intent.ts`, `apps/cli/src/commands/qa-archive-gate.ts`. Métricas: 1015/1015 tests (+15). Archivado 2026-06-30.

## 0029 — Memory Recall and Context Repair

Completa el lote B de 0024: recall útil de memoria en workflows, repair operativa de MCP, inventory CLI/TUI. Decisiones: ranking de recall (text match + recencia 90 días + kind boost: `decision=1.5`, `bug=1.4`, `pattern=1.2`, `learning=1.1`, `context=1.0`), cada hit con `reason` explicativo; repair registra en `.sdd/local/mcp-bindings.json` (escritura atómica); recall opt-in (wire-up al state machine diferido); cross-project access bloqueado (GATE 0024). Comandos nuevos: `axiom mcp repair --id <mcpId>`, `axiom memory inventory`, `axiom mcp inventory`. Métricas: 992/992 tests (+9). Archivado 2026-06-30.

## 0030 — Operator App Plugins and External Bridge

Completa el lote B de 0025: sistema de plugins project-scoped para la app portable, bridge Azure DevOps declarativo. Decisiones: plugin loader con discovery en `.sdd/<project>/app-plugins/*.json`; schema guard tolerante; ID único por proyecto; plugins sin lógica de negocio paralela; mutación externa exige `confirmed: true`. Endpoints nuevos: `GET /api/projects/:id/plugins` + `azure-devops` builtin (11 endpoints totales). Métricas: 1000/1000 tests (+8). Archivado 2026-06-30.

## Línea temporal consolidada

1. **0015** — Capability contracts Cavekit nativos (disciplina interna).
2. **0019** — Control plane operativa (upgrade, TUI, model routing, components, skills).
3. **0026** — Hardening: fix tool-routing, skills apply idempotente, routing snapshot real, TUI integration.
4. **0027** — Toolchain P1 operativo (repair, add, gitignore).
5. **0028** — Intent chains, archive-gate visible, parser workflows centralizado.
6. **0029** — Memory recall (opt-in), MCP repair, inventory CLI/TUI.
7. **0030** — App plugins project-scoped, Azure DevOps bridge declarativo.

Bloques por categoría: Control Plane (0019, 0026, 0027) · Workflows & UX (0028) · Memory & Context (0029) · Extensiones (0030) · Disciplina (0015).

## Nota de alcance

Este documento cubre 0015-0030. El git log real de `Axiom/` llega hasta commits posteriores a 0030 (menciones a 0031-0039 aparecen en `Axiom/README.md` y en el listado de comandos de `apps/cli/src/commands/`, p. ej. adapters 0031, skills registry 0032, agents 0033, capability/repo/context/gateway 0034). Esos incrementos no tenían documento `00XX-*.md` propio en `Axiom/docs/` al momento de este relevamiento — su detalle vive solo en el código y en el resumen de `Axiom/README.md`.

## Evolución posterior — `INC-20260730-toolchain-versioning`

Este incremento extiende el toolchain posterior a la ventana histórica 0015–0030. El catálogo dedicado `axiom.config/toolchain-catalog.yaml` pasa a schema 2 con `versionExtractor`, canales `stable`/`candidate`/`edge` y compatibilidad opcional. El estado de versiones fijadas se persiste como `toolchain.lock` schema 1 bajo `.axiom-state/<project>/`; `axiom toolchain plan` es read-only y `axiom toolchain upgrade` solo escribe el lockfile con `--yes`, checkpoint y rollback. El catálogo funciona como allowlist/política y no ordena instalar todas sus tools.

La implementación también añade observaciones best-effort de versión, checks TC-020..TC-023 y drift profundo en `axiom doctor --deep`. No instala binarios, no descubre releases upstream y no trata las versiones del catálogo como releases upstream verificadas. Los probes locales conocen `serena`, `cmm` y `engram`; `context7`, `rtk`, `caveman` y `autoskills` permanecen sin contrato local de probe. La validación focalizada del paquete de toolchain cubre 10 tests; el estado global y los fallos preexistentes se documentan en el README del incremento.

## Tanda `INC-20260730-*` — gobierno verificable (2026-08-02, 6 incrementos)

| Incremento | Entrega | Nota |
|---|---|---|
| `INC-20260730-typed-recovery` | `AxiomError` + catálogo `AXIOM_ERROR_CODES` (12 códigos); 45 sitios tipados (antes 3) | Corrigió un `instanceof` roto en toda subclase de `AxiomError` (`setPrototypeOf` hardcodeado) |
| `INC-20260730-engram-evidence` | `rationale`/`source` fail-closed en runtime, enforced en los 3 puntos de entrada del backend | Ruta de lectura y session-summary exentas por diseño |
| `INC-20260730-phase-receipts` | Emisión automática de receipts por transición, éxito y fallo | Encontró y corrigió una carpeta fantasma por `metadataId` y una race con el move de `archive` |
| `INC-20260730-candidate-freeze` | Cobertura de tests (antes cero) + `JSON.parse` endurecido en `checkCandidateFreeze` | El grueso ya estaba implementado y cableado antes de esta tanda |
| `INC-20260730-exact-scope` | 3 loaders de `profiles.yaml` convergidos en el schema canónico ya existente | El Scope original ("instalar zod") estaba obsoleto: zod y `@axiom/config-validation` ya existían |
| `INC-20260730-autopilot-integration` | Las 3 directivas de gobierno propagadas a las 7 superficies de `axiom-autopilot` | Defecto de propagación: solo la copia local del workspace las tenía |

Suite al cierre de la tanda: **3489 tests, 3483 verdes, 6 rojos preexistentes** (5 deterministas en `install-profiles/composer.test.ts`, 1 flake por contención en `memory/engram-backend.test.ts`). Baseline antes de la tanda: 3428 tests. Neto: **+61 tests, cero regresiones**.
