# Ciclo de vida CLI y orquestación

Fuente: `Axiom/docs/cli/*.md`, `@axiom/orchestrator`, `@axiom/cli-commands`, `Axiom/apps/cli/src/commands/`.

## Secuencia de lifecycle (7 comandos base + upgrade)

```
init → join → configure → sync → start → audit → doctor
                                              ↓
                                          upgrade
```

`@axiom/orchestrator` implementa una state machine con 7 comandos de lifecycle + 15 "intent commands" (comandos de intención de más alto nivel que en MVP devuelven `not-implemented`), un predicado `gateFor` por comando y un `runCommand` que corre con telemetría y hooks.

## Detalle por comando (contrato lee/escribe)

### `axiom init`
1. Valida nombre (`^[a-z0-9][a-z0-9-]{0,62}$`).
2. Determina layout: `self-hosted` (si detecta markers `_builder/`, `Axiom/`) o `installed-multi-repo` (default).
3. Genera `axiom.yaml` con profile triple.
4. Crea `.sdd/local/` y `.sdd/<projectName>/`, `.gitignore`.
5. Si layout `installed-multi-repo`, resuelve `topology.yaml`.
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
Ejecuta familias de checks: boundaries, policies, manifests, isolation, capability model, gateway. Soporta `--json`. No escribe nada.

### `axiom upgrade`
`--dry-run` | `--from-checkpoint <id>` | `--target-version <v>` | `--no-sync` | `--no-doctor`. Calcula migraciones aplicables, crea checkpoint pre-upgrade (`init.json`, `install-profile.json`, `managed-state.json`), aplica migraciones en orden con rollback automático si alguna falla, persiste nuevo `ManagedState`, y por defecto encadena `sync` + `doctor` post-upgrade.

## TUI (`axiom tui`)

Requiere TTY. Menú de 6 items: configurar, sincronizar, diagnóstico, upgrade, model routing, salir. Para mutaciones (configure/sync/upgrade): preview read-only → confirmación (Y/n) → ejecución real vía `@axiom/cli-commands` → post-run summary con restore point y follow-ups. Doctor corre directo (read-only). Model routing tiene submenú por slot (teclas 1-4 = clases, 0 = unset).

## `axiom model` (routing de modelos por slot)

- `show [--target] [--slot]`: routing efectivo por slot (read-only).
- `set <slot> <class>` / `unset <slot>` / `reset`: mutación de `.sdd/config/<projectName>/model-assignments.json`.
- `validate [--target]`: corre 4 checks de drift (MRC-001 a MRC-004) y proyecta a `.opencode/model-routing.json` si el target es `opencode`.

Slots: `increment`, `bug`, `plan`, `implementation`, `qa-e2e`, `review`, `archive`. Clases: `cheap`, `medium`, `strong`, `local`.

## `axiom components` y `axiom skills`

- `components list/show/install/uninstall/restore`: catálogo derivado de `integrations.yaml`; install/uninstall mutan `components-state.json` con checkpoint (uninstall siempre crea uno salvo `--from-checkpoint`).
- `skills list/refresh/drift`: fuente de verdad es `.opencode/skills-lock.yaml` (generado por el adapter opencode, read-only desde `axiom skills`); `refresh --recompute-hashes` recalcula `bundleHash` contra `skills-catalog.yaml`.

## Comandos reales sin documentación operativa dedicada

`apps/cli/src/commands/` tiene 36 ficheros. Además de los 12 documentados arriba (incluyendo `model`/`components`/`skills`/`tui`), existen: `app`, `app-api`, `app-plugins`, `app-plugins-azure-devops`, `axiom-bug`, `axiom-increment`, `axiom-plan`, `axiom-qa-e2e`, `axiom-role`, `capability`, `context`, `gateway`, `intent`, `mcp`, `memory`, `projects`, `qa-archive-gate`, `repo`, `roles`, `self-update`, `toolchain`, `topology`. Se mencionan de forma dispersa en documentos de incremento (`0019`-`0030`) pero carecen de página propia en `docs/cli/`. Antes de tratarlos como contrato estable, verificar en código.

## `@axiom/cli-commands` (barrel)

Re-exporta funciones `runX` (`runConfigure`, `runSync`, `runModel`, `runComponents`, `runSkills`, `runMemory`, `runMcp`, etc.) desde `apps/cli/src/commands/*`, para que `@axiom/tui` no dependa directamente de `apps/cli`. Es re-export trivial, sin lógica propia. Nota conocida: bug pre-existente de resolución de paths por configuración de `tsconfig`/`rootDir`, documentado en `Axiom/docs/installation.md`.
