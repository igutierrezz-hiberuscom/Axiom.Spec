# Ciclo de vida CLI y orquestación

Fuente: `Axiom/docs/cli/*.md`, `@axiom/orchestrator` (README + `src/`), `@axiom/cli-commands`, `Axiom/apps/cli/src/commands/`.

## Secuencia de lifecycle (7 comandos base + upgrade)

```
init → join → configure → sync → start → audit → doctor
                                              ↓
                                          upgrade
```

`@axiom/orchestrator` implementa una state machine con **8 comandos de lifecycle** (`init`, `join`, `configure`, `sync`, `start`, `audit`, `doctor`, `upgrade`) **+ 19 "intent commands"** (`axiom-*-command`, stubs que siempre devuelven `not-implemented`), un predicado `gateFor` por comando y un `runCommand` que corre con telemetría y hooks (verificado: `packages/orchestrator/README.md`, actualiza el conteo "7+15" del baseline 2026-07-02).

**Matiz de honestidad documentado por el propio package** (`INC-20260710-honesty-and-toolchain-states`): de los 8 lifecycle commands declarados, solo **7 pasan realmente por este gate** desde `apps/cli` (vía `runOrchestrated` en `init.ts`/`join.ts`/`configure.ts`/`sync.ts`/`start.ts`/`audit.ts`/`upgrade.ts`). `doctor-command` está declarado en la matriz pero el comando real `axiom doctor` NO invoca este gate — llama directo a `@axiom/doctor`. Los 19 intent ids son stubs reservados: hoy solo los ejercitan los tests propios de `@axiom/orchestrator`, ningún caller real de `apps/cli` los invoca — no son la vía de ejecución real del workflow SDD (esa vive en `@axiom/workflow`, ejercitada directamente por `axiom-increment`/`axiom-bug`/`axiom-plan`/`axiom-role`).

## Detalle por comando (contrato lee/escribe)

### `axiom init`
1. Valida nombre (`^[a-z0-9][a-z0-9-]{0,62}$`).
2. Determina layout: `self-hosted` (si detecta markers `_builder/`, `Axiom/`) o `installed-multi-repo` (default).
3. Genera `axiom.yaml` con profile triple.
4. Crea `.axiom-state/local/` y `.axiom-state/<projectName>/`, `.gitignore` (renombrado desde el prefijo antiguo por `INC-20260703-config-folder-renames`, cerrado).
5. Si layout `installed-multi-repo`, ya **no** escribe `topology.yaml` (`INC-20260703-config-dedup`, cerrado): `@axiom/topology#loadTopology` deriva un manifest de fallback desde `axiom.yaml` cuando el fichero está ausente; `topology.yaml` se materializa perezosamente solo cuando el proyecto corre `axiom roles assign`.
6. Persiste `init.json`.
Flags: `--name`, `--profile`, `--overlay`, `--layout`, `--target`, `--path`, `--yes`, `--force`.

### `axiom join`
Registra `--member <id>` (`user:alice`, `agent:sdd`, etc.) en `members.yaml`, deduplicando por igualdad exacta.

### `axiom configure`
Lee `init.json` + `profiles.yaml` + `providers.yaml` (+ `provider-overrides.yaml` local opcional), compone `ResolvedInstallProfile`, persiste `install-profile.json`, y materializa surfaces del target activo (p. ej. Copilot instructions vía `@axiom/document-bootstrap`).

### `axiom sync`
Lee `init.json`, determina overlay, **valida el gate de telemetría antes de escribir nada**; si falla, aborta sin tocar `last-sync.json`. Si pasa, invoca el materializador del adapter y persiste `last-sync.json`.

### `axiom start`
Resuelve modo de discovery según overlay + flags `--gateway`/`--no-gateway` (`local-only` → siempre filesystem; `standard` → filesystem por defecto, gateway opt-in; `enterprise` → exige gateway). Ejecuta un `routeTool` sintético (`verify-startup`) y persiste `last-start.json`.

### `axiom audit`
Solo lectura. Calcula SHA-256 del audit trail, cuenta líneas, valida retención, detecta rewrite externo. Estados: `compliant` (exit 0), `absent` (exit 0), `violation` (exit 1).

### `axiom doctor`
Ejecuta familias de checks (creció bastante desde el baseline 2026-07-02: boundaries, policies, manifests, isolation, capability model, install-profiles, gateway, tool-routing, topology, workflow-config, toolchain, memory, adapters, skills, agents, artifact-index, write-scope, dogfooding, provider-selection, más gobernanza GC-00x). Soporta `--json` y, opt-in, `--deep` (probes reales de tool/MCP, ver `../architecture/04-adapters-y-model-routing.md`). No muta nada (los probes `--deep` solo `pass`/`warn`/`skip`, nunca `fail`). Detalle completo en `../operations/02-doctor-troubleshooting-y-telemetria.md`.

### `axiom upgrade`
`--dry-run` | `--from-checkpoint <id>` | `--target-version <v>` | `--no-sync` | `--no-doctor`. Calcula migraciones aplicables, crea checkpoint pre-upgrade (`init.json`, `install-profile.json`, `managed-state.json`), aplica migraciones en orden con rollback automático si alguna falla, persiste nuevo `ManagedState`, y por defecto encadena `sync` + `doctor` post-upgrade.

## Histórico: TUI (`axiom tui`)

La TUI fue retirada como superficie pública en ACC-005 (2026-08-04). Esta
sección se conserva únicamente para explicar decisiones históricas y no es
una instrucción operativa vigente.

Requiere TTY. Menú de 6 items: configurar, sincronizar, diagnóstico, upgrade, model routing, salir. Para mutaciones (configure/sync/upgrade): preview read-only → confirmación (Y/n) → ejecución real vía `@axiom/cli-commands` → post-run summary con restore point y follow-ups. Doctor corre directo (read-only). Model routing tiene submenú por slot (teclas 1-4 = clases, 0 = unset).

**Wizard genérico de `init` (histórico, `INC-20260703-tui-init-wizard`)**: fue
retirado junto con la TUI en ACC-005. El contrato vigente equivalente es el
launcher de onboarding y `axiom init` headless; no quedan screens ni driver
TUI en el runtime.

## `axiom model` (routing de modelos por slot)

- `show [--target] [--slot]`: routing efectivo por slot (read-only).
- `set <slot> <class>` / `unset <slot>` / `reset`: mutación de `.axiom-state/config/<projectName>/model-assignments.json`.
- `validate [--target]`: corre 4 checks de drift (MRC-001 a MRC-004) y proyecta a `.opencode/model-routing.json` si el target es `opencode`.

Slots: `increment`, `bug`, `plan`, `implementation`, `qa-e2e`, `review`, `archive`. Clases: `cheap`, `medium`, `strong`, `local`.

## `axiom components` y `axiom skills`

- `components list/show/install/uninstall/restore`: catálogo derivado de `integrations.yaml`; install/uninstall mutan `components-state.json` con checkpoint (uninstall siempre crea uno salvo `--from-checkpoint`).
- `skills list/refresh/drift`: fuente de verdad es `.opencode/skills-lock.yaml` (generado por el adapter opencode, read-only desde `axiom skills`); `refresh --recompute-hashes` recalcula `bundleHash` contra `skills-catalog.yaml`.

## Comandos reales sin documentación operativa dedicada

`apps/cli/src/commands/` tiene **81 ficheros** (verificado: `ls apps/cli/src/commands/*.ts | wc -l` → 81; de esos, 10 son helpers internos con prefijo `_` — `_adapter-labels.ts`, `_cross-repo-plan.ts`, `_execution-mode.ts`, `_functional-verify.ts`, `_repo-affinity.ts`, `_role-review.ts`, `_shared.ts`, `_spec-scope.ts`, `_tracker-status.ts`, `_worktree-execution.ts` — no comandos invocables por sí mismos). Muy por encima del baseline 2026-07-02 (36). Familias completas que no existían entonces:

- `workspace*` (16 ficheros): `workspace.ts`, `workspace-setup.ts`, `workspace-adopt.ts`, `workspace-adapters.ts`, `workspace-adapter-templates.ts`, `workspace-autoskills.ts`, `workspace-catalog-scaffold.ts`, `workspace-code-intel.ts`, `workspace-config-scaffold.ts`, `workspace-incremental.ts`, `workspace-mcp.ts`, `workspace-process-surfaces.ts`, `workspace-rules.ts`, `workspace-skills.ts`, `workspace-spec-base.ts`, `workspace-worktree-provision.ts`.
- `app*` / launcher: `app.ts` (abre el launcher como front por defecto, ver más abajo), `app-api.ts`, `app-onboarding.ts`, `app-launcher.ts`, `app-launcher-panels.ts`, `app-launcher-ado.ts`, `app-plugins.ts`, `app-plugins-azure-devops.ts`.
- `member-install.ts` (instalación multi-repo por miembro de equipo).
- `native-mcp-config.ts` (proyección de los servers MCP gestionados a la config nativa de cada tool).
- Comandos backed por `@axiom/tracker`/`@axiom/tracker-ado`: `_tracker-status.ts`, `external-sync.ts`, además de `app-launcher-ado.ts`.
- Otros ficheros nuevos desde el baseline: `axiom-adr.ts`, `axiom-decision.ts`, `artifact-metadata-cli.ts`, `bindings.ts`, `bootstrap.ts`, `eject.ts`, `external-sync.ts`, `index-cmd.ts`, `integrate.ts`, `learn.ts`, `mcp-serve.ts`, `normalize-cmd.ts`, `rollback.ts`, `scaffold.ts`, `state-cmd.ts`, `validate-changes.ts`.

Los comandos documentados en el baseline (`init`, `join`, `configure`, `sync`, `start`, `audit`, `doctor`, `upgrade`, `model`, `components`, `skills`) siguen siendo páginas históricas de `docs/cli/`; `tui` ya no forma parte de la superficie registrada. **`axiom app`** abre `${url}/launcher/` por defecto; el viejo operator UI raíz se eliminó y `GET /`/`GET /index.html` redirigen 302 a `/launcher/`. Antes de tratar el comportamiento de cualquier comando como contrato estable, verificar directamente en el código.

## `@axiom/cli-commands` (barrel)

Reexporta funciones `runX` desde el owner físico `packages/cli-commands/src/commands/*` y publica `dist/index.js`/`dist/index.d.ts` con ownership único para CLI, launcher y MCP. No contiene lógica de negocio ni depende de una interfaz TUI.

## Gobierno verificable en el ciclo (2026-08-02) — tanda `INC-20260730-*`

`runIncrementSubcommand` y `runBugSubcommand` son desde esta tanda **wrappers públicos delgados** sobre un `…Core` privado. El core conserva la lógica de transición byte-idéntica; el wrapper emite un receipt JSON a partir del resultado ya calculado y devuelve ese resultado sin tocarlo. Consecuencia para quien lea el código: el `safeParse` del schema de argumentos vive en el **core**, no en el wrapper, y cualquier lógica nueva de transición debe ir al core — añadirla al wrapper la dejaría fuera del contrato de "no modificar el resultado".

`axiom freeze` (`apps/cli/src/commands/freeze.ts`) expone además `checkCandidateFreeze`, ya consumido por `axiom-increment` como gate previo al apply. Devuelve siempre `{ ok, reason? }` y nunca lanza, incluido el caso de `candidate-freeze.json` corrupto o truncado.

Detalle normativo en [../../specs/07_Gobierno_y_Seguridad.md](../../specs/07_Gobierno_y_Seguridad.md), artefactos en [../../specs/03_Modelo_Operativo_y_Datos.md](../../specs/03_Modelo_Operativo_y_Datos.md).
