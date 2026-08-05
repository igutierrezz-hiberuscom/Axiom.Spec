# Doctor, troubleshooting y telemetría

Fuente: `Axiom/docs/cli/doctor.md`, `Axiom/docs/troubleshooting.md`, `Axiom/docs/configuration/telemetry-and-isolation.md`, `Axiom/docs/configuration/files/telemetry-sinks.md`, `Axiom/packages/doctor/src/{checks,governance-checks,deep-checks,write-scope}.ts`.

> Reconciliado 2026-07-29: `@axiom/doctor` creció mucho desde el baseline 2026-07-02 (6 familias → ~19 categorías con IDs propios, más un modo `--deep` opt-in nuevo). Conteo de familias/IDs verificado por grep directo sobre `packages/doctor/src/*.ts`; no inventar IDs no encontrados en el código.

## `axiom doctor`

Familias de checks verificadas (prefijo de check ID entre paréntesis; el prefijo `TC-` NO es específico de una sola categoría — se reutiliza entre topology/toolchain/memory/adapters/skills/agents/workflow-config/deep-probes, ver tabla):

| Categoría | Prefijo(s) de ID | Qué valida |
|---|---|---|
| boundaries | `BC-001..003` | separación entre scopes documentales y runtime |
| policies | `PC-001/002` | `axiom.config/integrations.yaml`, `axiom.config/policy-as-code.yaml` presentes |
| manifests | `MC-001/002` | `axiom.yaml` válido, `.axiom-state/local/` protegida por `.gitignore` |
| isolation | `IC-001..003` | invariantes project-scoped, restricciones MCP |
| capability-model | `CC-001..006` | consistencia del modelo declarativo y cobertura provider-routed de capabilities |
| install-profiles | `IP-001..004` | consistencia del profile triple resuelto |
| gateway | `GW-001` | `gateway-state.json` vs drift contra `providers.yaml`/`profiles.yaml`/`install-profile.json` |
| tool-routing | `TR-001..004` | consistencia del dispatcher de `ToolCall` |
| topology | `TC-001..003` | `axiom.config/topology.yaml` (o su derivación por fallback) |
| toolchain | `TC-004..006` | `toolchain-catalog.yaml`, detección/registro de tools |
| toolchain versioning | `TC-020..023` | lockfile, compatibilidad de versiones, drift y canales |
| memory | `TC-007/008` | backend de memoria, invariantes de recall |
| adapters | `TC-009` | los 9 packages `@axiom/adapters-<target>` tienen `src/generator.ts` + `dist/index.js` materializados |
| skills | `TC-010/012/013` | `skills-catalog.yaml`, lockfile, `bundleHash` |
| agents | `TC-011` | `axiom.config/agents-catalog.yaml`, cada entry con agent válido |
| workflow-config | `TC-014/015` | config de workflow/lifecycle |
| artifact-index | `IX-001` | índices de increment/bug/adr/decision |
| write-scope | `WS-001` | scope de escritura permitido por un plan |
| dogfooding-boundary | `DF-001` | el propio repo `Axiom/` no se auto-referencia de forma inconsistente |
| provider-selection | `PS-001` | selección de provider efectivo consistente con `providers.yaml` |
| gobernanza | `GC-001..013` | lockfile de skills, `AGENTS.md`, `repo.manifest`/`product.manifest`, contenido materializado en `.axiom/*` |

Salida `--json` disponible. No muta nada en el modo síncrono. Detalle de MRC-001..004 (drift de model routing) en `../architecture/04-adapters-y-model-routing.md` — corren vía `axiom model validate`, NO como parte del `axiom doctor` síncrono (viven en el package separado `@axiom/model-routing`, no en `@axiom/doctor`).

### `axiom doctor --deep` (probes opt-in de runtime)

`runDoctorChecksDeep` (`packages/doctor/src/deep-checks.ts`) añade probes reales que **nunca** pueden fallar (solo `pass`/`warn`/`skip`): **TC-018** (tool functional: `--version` real para `serena`/`cmm`/`engram`; `skip` honesto para `rtk`/`caveman`), **TC-019** (MCP liveness: handshake `initialize` JSON-RPC real contra `sdd-mcp-server`/`spec-mcp-broker`, leyendo `.axiom/mcp.yml`) y el drift profundo de toolchain (**TC-022**). Para TC-022, `runToolchainVersionDriftDeepCheck` compara la observación de versión contra `toolchain.lock`; una versión no observable o distinta produce `warn`, nunca `fail`. La ruta síncrona deja TC-022 como indicación informativa de que hace falta `--deep`. Ver `../architecture/04-adapters-y-model-routing.md` para el detalle completo.

Las checks de lockfile **TC-020..TC-023** son project-scoped: TC-020 valida existencia/forma del lockfile y trata su ausencia como información; TC-021 valida que la versión locked satisface el canal del catálogo; TC-022 representa el drift diferido a probe real; TC-023 valida versión y canales `stable`/`candidate`/`edge`. Las checks de versión no instalan ni reparan binarios.

## Troubleshooting por comando (categorías documentadas)

**`axiom init`**
- `Cannot find module '@axiom/orchestrator'`: binario compilado en `apps/cli/dist/` no resuelve packages workspace → revisar `workspaces` en `package.json`, `npm install`, rebuild.
- `invalid-config`: YAML de `axiom.yaml` malformado → corregir sintaxis/indentación.
- Conflicto con `axiom.yaml` existente sin `--force`.
- Nombre de proyecto inválido (no cumple regex).

**`axiom join`**: falta `--member` en modo no interactivo.

**`axiom configure`**
- No encuentra `init.json` → ejecutar `axiom init` primero.
- Falla al escribir surfaces de Copilot → revisar `axiom.spec/templates/copilot-instructions.template.md`, `product.manifest.yaml`, `axiom.yaml`, `providers.yaml`, overlay local.

**`axiom sync`**
- `telemetrySinkMissing`: overlay exige señales mínimas sin sink habilitado → revisar `telemetry-sinks.yaml`, overlay activo, re-ejecutar `configure`.
- `adapterGenerationFailed`: falta `install-profile.json` (se corrió `sync` antes de `configure`).

**`axiom start`**
- Conflicto overlay/flags de gateway (`enterprise` prohíbe `--no-gateway`; `standard` lo trata como opt-in; `local-only` siempre filesystem).
- Warning por capability model no cargado (`capabilities.yaml`/`providers.yaml` faltantes o incompletos).

**`axiom audit`**
- `violation`: rewrite externo detectado del audit trail → tratar como señal de integridad rota.
- Warnings de retención: revisar política espejo en `telemetry-sinks.yaml`.

**`axiom doctor`**: correr con `--json`, corregir primero fallos estructurales, dejar warnings para una segunda pasada.

`CC-004` usa las 16 capabilities provider-routed canónicas, no solo las que
ya aparecen en `providers.yaml`. Lee su clase y estado desde
`capabilities.yaml`: requeridas activas sin provider son `fail`, opcionales o
post-MVP son `warn`, y `disabled`/`unavailable` quedan visibles sin fallo
activo. Las tres capabilities MCP-only `axiom.*` quedan fuera de esta
obligación. Si `capabilities.yaml` falta o no es válido, `CC-004` devuelve un
fallo explícito con la ruta y la causa.

## Telemetría (`telemetry-sinks.yaml`)

Impacta: `sync` (valida gate antes de regenerar outputs), `audit` (usa mirror de retención), boot del CLI (carga sinks habilitados).

- `dataSensitivityBoundaries` por overlay: tags permitidos, tags redactados, nivel de redacción, flujo cross-project permitido, ventana de retención.
- `sinks`: `null-sink`, `log-sink`, `remote-sink`, `audit-trail-sink`.
- Regla práctica: si un overlay exige señales mínimas (`minimumSignals`) y no hay sink habilitado que las cubra, `sync` aborta **antes** de escribir.
- No documentado explícitamente: un mecanismo formal de opt-out completo de telemetría. Tratar como ausente, no como implementado silenciosamente.

## Aislamiento (`@axiom/isolation`, doctor)

`doctor` valida: que las rutas project-scoped contengan el `projectId`; que `backend-mcp`/`frontend-mcp` sigan bloqueados en MVP; que el modo local pueda operar sin gateway cuando corresponde.

`integrations.yaml` declara: si el sistema está scoped por proyecto, cómo se resuelve el proyecto, catálogo y modo de política, defaults gestionados por Axiom, decisiones de instalación aprobadas, y el bloque `projectIsolation` (qué contaminación cross-project debe bloquearse).

## Gobernanza mínima verificada por doctor

`integrations.yaml` existe; `policy-as-code.yaml` existe; `axiom.yaml` es válido; `.axiom-state/local/` no queda expuesta accidentalmente al versionado. Ver también la fila "gobernanza" (`GC-001..013`) de la tabla arriba para las verificaciones de gobernanza más amplias (lockfile de skills, `AGENTS.md`, manifests) añadidas desde el baseline.
