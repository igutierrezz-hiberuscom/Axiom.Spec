# 00 Resumen Ejecutivo

## Visión

Axiom es una plataforma de Spec-Driven Development (SDD) con un runtime MVP+post-MVP ya operativo. No es un simple reempaquetado de una herramienta existente: es un CLI Node/TypeScript (`axiom`) que, para un proyecto concreto, coordina estructura, configuración, resolución de profiles, materialización de archivos para herramientas externas (IDEs/CLIs de IA), validaciones y checks de salud.

Modelo mental del producto (`Axiom/docs/overview.md`):
- la spec dice qué reglas existen y qué contrato debe cumplirse;
- la configuración (YAML por proyecto) define cómo se activa ese contrato en un proyecto concreto;
- el CLI materializa, valida y opera sobre ese contrato.

## Registro histórico: snapshot del runtime (verificado en el código, 2026-07-02)

Axiom NO está en fase de diseño: tiene un monorepo npm workspaces (`Axiom/`) con `apps/cli` + 28 packages bajo `packages/*` (ver [context/references/01-inventario-de-packages.md](../context/references/01-inventario-de-packages.md)), 36 ficheros de comandos en `apps/cli/src/commands/` y un histórico de al menos 40 commits (2026-06-05 a 2026-07-02) e incrementos numerados (0008 a 0039+) que fueron cerrando MVP y post-MVP en oleadas.

Según `Axiom/README.md` (2026-06-25/30), todas las capas declaradas están "implementadas y testeadas": dominio, aplicación (CLI), adapters (6 reales), telemetría, orquestación y persistencia.

Este bloque conserva una fotografía inicial del repositorio y no es el contrato cuantitativo vigente. La baseline actual de adapters, providers y MCP se consolida en la sección siguiente y en [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md).

## Estado vigente del runtime

El runtime actual reconoce 8 targets activos (`opencode`, `claude-code`, `github-copilot`, `vscode`, `cursor`, `antigravity`, `visual-studio-2026` y `codex`), con 8 paquetes de adapter dedicados. `copilot-vscode` no forma parte del vocabulario público: solo `axiom configure` puede migrar ese literal cuando está persistido en `init.json#profileTriple.adapterTarget`, a `github-copilot` y antes de instalar o despachar. LiteLLM fue retirado. El launcher mantiene 9 entradas de routing (8 adapters headline más `cli`). La configuración efectiva es `builder` + `local-only`, ambas implícitas; los roles de equipo y las fases SDD son ejes separados. La selección project-scoped ofrece `cmm`, `serena` y `engram`; el registry canónico contiene 4 providers locales (`filesystem`, `serena`, `cmm`, `engram`).

El runtime expone un único broker MCP gestionado, `axiom-mcp-broker`, con un registro global de 25 capability ids distribuidos internamente entre los dominios `sdd`, `spec`, `memory` y `axiom`. La configuración MCP nativa escribe superficies de proyecto solo después de validar un binding único `axiom` y el manifest; los targets user-globales `codex` y `antigravity` no reciben escritura ni recomendación automática sin binding seguro. El broker implementa MCP `2026-07-28` sobre stdio, rechaza el handshake legacy y referencias cross-project, y permite lecturas y mutaciones confirmadas en la misma superficie.

### Versionado reproducible del toolchain (`INC-20260730-toolchain-versioning`)

El toolchain externo mantiene tres responsabilidades separadas: el catálogo global de IDs permitidos, el manifest de tools habilitadas por el proyecto y el estado local de versiones fijadas. `axiom.config/toolchain-catalog.yaml` usa schema 2 y puede declarar `versionExtractor`, versiones por canal (`stable`, `candidate`, `edge`) y compatibilidad mínima de Axiom. El catálogo vigente contiene `serena`, `cmm`, `engram`, `context7`, `rtk`, `caveman` y `autoskills`; ninguna está marcada como `mvp` por defecto.

Las versiones fijadas se persisten en `.axiom-state/<projectKey>/toolchain.lock`, schema 1, con escritura atómica. El lockfile es estado operativo local y no instala ni reemplaza binarios externos. `axiom toolchain plan` calcula, sin escribir, el diff entre las tools declaradas o ya lockeadas y el canal solicitado; `--id` limita el subconjunto. `axiom toolchain upgrade` muestra una vista previa salvo que reciba `--yes` y, cuando escribe, protege el lockfile con checkpoint y rollback ante un fallo de persistencia o de verificación posterior.

La detección de versiones usa probes locales best-effort y la regex del catálogo. `axiom doctor` incorpora TC-020..TC-023 para existencia/validez del lockfile, compatibilidad, drift y canales; la comparación real de versión instalada contra la locked se ejecuta de forma opt-in en `axiom doctor --deep` y nunca convierte un fallo de probe en un fallo duro del doctor.

La resolución pública solo expone `ProjectMode: 'local-only'`. Los valores raw
históricos `gateway` y `hybrid` se aceptan únicamente al leer manifiestos v1/v2
para normalizarlos a local-only; no activan providers, permisos, discovery ni
comandos remotos. Todo estado project-bound comparte el namespace físico
`.axiom-state/<projectKey>/`; `local/` y `executions/` conservan sus fronteras.

## Qué hace el producto hoy

El punto de entrada guiado vigente es `axiom app`, que abre el launcher web local bajo `/launcher/`. Desde allí se puede instalar o unir un proyecto, configurar un workspace, adoptar spec/SDD/contexto, consultar el registry y operar el ciclo SDD con preview y confirmación. La misma capacidad está disponible de forma headless mediante los comandos CLI y sus handlers MCP. `axiom` sin subcomando ya no abre una TUI y `axiom tui` dejó de ser un comando registrado; `axiom init` sigue disponible como comando no-interactivo/scriptable single-repo.

Para un proyecto que adopta Axiom, el ciclo de vida real es:

```
axiom init → axiom join → axiom configure → axiom sync → axiom start → axiom audit → axiom doctor → axiom upgrade
```

Cada comando lee/escribe un conjunto concreto de artefactos (`axiom.yaml`, `.axiom-state/<projectKey>/*.json`, `.axiom-state/local/*`) y, según el profile y los adapters seleccionados, genera la superficie portable `.axiom/{agents,commands,skills}/`, salidas específicas como `.opencode/`, `.claude/`, `.codex/`, `.antigravity/`, `.cursor/`, `.vscode/` y `.github/copilot-instructions.md`, además de `.axiom/mcp.yml` y la configuración MCP nativa aplicable (`opencode.json`, `.mcp.json`, `.cursor/mcp.json` o `.vscode/mcp.json`). Visual Studio reutiliza `.github/copilot-instructions.md` y no genera `.vs/AXIOM.md`; `codex` y `antigravity` no reciben configuración MCP global automática. El detalle completo vive en [04_Flujos_SDD_y_Ciclo_de_Vida.md](04_Flujos_SDD_y_Ciclo_de_Vida.md) y en `context/architecture/`.

## Reconciliación de superficies operativas (2026-08-04)

La salida pública de `@axiom/cli-commands` se publica desde `dist/index.js` y
`dist/index.d.ts`, con ownership único de los comandos compartidos; la TUI
pública se retiró y ya no forma parte del runtime. El launcher delega onboarding y adopción en
`runWorkspaceSetup`/`runWorkspaceAdopt`, rechaza destinos foráneos o paths de
roles solapados y conserva resultados parciales con warnings y provenance.

El registry del launcher deriva metadata, paths, workflow-state y relaciones
solo de archivos reales. Las acciones SDD se delegan a los `run*` canónicos,
propagan `executionMode` al handler async de worktrees y archivan mediante
`runIntegrate` con preflight y rollback. Los plugins externos usan handlers
allowlisted: `command` es una etiqueta, no una autoridad de ejecución; el
catálogo y los resultados se proyectan sin secretos. Estas reglas quedaron
verificadas en ACC-004, ACC-006, ACC-007 y ACC-008.

## Dos modelos distintos que conviven bajo el nombre "Axiom" (evitar confundirlos)

1. **El producto Axiom** (`Axiom/`): el CLI/runtime descrito arriba, que cualquier proyecto externo puede adoptar. Su unidad de configuración es `axiom.yaml` + estado runtime en `.axiom-state/<projectKey>/` dentro del proyecto que lo instala, más un catálogo de ~20 YAML de política/capacidad que el propio Axiom espera encontrar en una carpeta `axiom.config/` (minúsculas) en la raíz del proyecto; el contenido de spec vive aparte en `axiom.spec/`. Ambas carpetas fueron renombradas en `INC-20260703-config-folder-renames` (ver [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md) y [08_Glosario.md](08_Glosario.md)).
2. **El workspace de desarrollo de Axiom** (esta raíz: `Axiom/`, `Axiom.SDD/`, `Axiom.Spec/`): el modelo de tres repos desacoplados que `Axiom.SDD/AGENTS.md` establece para construir el propio producto con disciplina SDD ligera (spec en `Axiom.Spec/`, implementación en `Axiom.SDD/`, runtime opcional en `Axiom/`). Este es el "dogfooding" de Axiom sobre sí mismo, y es un concepto de nivel distinto al `axiom.spec/` interno de un proyecto cualquiera.

En este workspace, la topología identifica `Axiom.Spec/` como `specRepo` y fuente documental canónica. `Axiom/axiom.spec/` se conserva como baseline product-owned materializable del runtime dogfoodeado; sus contenidos no sustituyen el repositorio canónico ni deben fusionarse con él (ADR-0032).

Ver el glosario ([08_Glosario.md](08_Glosario.md)) para no mezclar `Axiom.Spec/` (este repo) con `axiom.spec/` (carpeta que el producto espera dentro de cualquier proyecto que lo adopte, incluido potencialmente `Axiom/` mismo).

## Baseline histórica de auto-validación (cierre 2026-07-08; estado actual verificado aparte)

El cierre histórico de `INC-20260708-product-repo-self-bootstrap` y `INC-20260708-fix-longstanding-test-failures` registró la baseline canónica de `Axiom/` (`axiom.config/`, `axiom.spec/target-axiom-skills|agents/`, `axiom.spec/templates/`, `AGENTS.md`, `axiom.skills.lock`), corrigió el `repoRoot` de `verify-first-project-readiness.mjs` y dejó la suite en verde (`178/178` ficheros, `1859/1859` tests en ese momento). Ese resultado describe el cierre de aquella tanda y no debe leerse como una garantía vigente.

**Registro histórico de validación (2026-07-30; superado por la verificación del 2026-08-02):** `npm run doctor` y `npm run readiness:first-project` fallaban en `TC-011` por un `bundleHash` stale de `axiom-reviewer`. La ejecución global independiente del review reportó 328/330 archivos y 3425/3427 tests, con dos fallos fuera del alcance de `INC-20260730-toolchain-versioning`; después se añadió una prueba focalizada de `extractVersion()` y ambos fallos se reprodujeron aisladamente. La verificación posterior devuelve `npm run doctor` en PASS (46/61 OK, 0 fallos, 3 advertencias, 12 omitidos) y `npm run readiness:first-project` en PASS. Las carpetas `axiom.config/` y `axiom.spec/` sí existen en la raíz actual de `Axiom/`.

**Revalidación R-04 (2026-08-05):** la corrección de `CC-004` hace visible que el
repo dogfooded sirve 13 de las 16 capabilities provider-routed. Las tres
capabilities opcionales restantes producen `warn` no bloqueante; el check no
presenta un fallo de cobertura activa. La batería dirigida de R-04 y
`npm run build` pasan.

## Ola de endurecimiento 2026-07-10 (auditoría + 10 incrementos, cerrada)

Tras una auditoría de premisas contra el código real (build limpio, suite en verde), se corrió una ola de 10 incrementos de reconciliación (`specs/increments/_archive/INC-20260710-*`) que cerró las brechas entre "arquitectura testeada" y "camino feliz operativo". Suite final: **212 ficheros / 2268 tests en verde**. Resumen por área:

- **Schemas duplicados reconciliados** (`schema-reconciliation`): `axiom mcp list/validate/repair` y `axiom toolchain add/show/validate` estaban rotos contra los YAML reales committeados (dos lectores incompatibles del mismo artefacto, ocultado por fixtures). Lectores tolerantes + `axiom.config/toolchain-catalog.yaml` dedicado + tests que cargan los artefactos reales.
- **Dogfooding del propio workflow** (`dogfooding-workflow-configs`): faltaban `axiom.config/workflows.yaml` y `topology.yaml`, así que `axiom-increment create` fallaba en el propio repo. `DEFAULT_WORKFLOWS` bundled con fallback + ambos ficheros materializados + checks de doctor TC-014/TC-015. Reparada también la deriva TC-011.
- **Roles dinámicos** (`dynamic-team-roles`): los roles dejan de ser un catálogo fijo. Registro de roles de equipo de primera clase en `topology.yaml#roles` (`axiom roles register/unregister`, 1..N arbitrarios), desacoplado del eje de perfiles de instalación; el validador de topología acepta los roles registrados y la topología generada por el wizard ya pasa su propio `topology validate`.
- **Planes por rol** (`plan-role-split`): `PlanMetadata.roles` + `axiom-plan create` deriva la separación por rol desde los roles/asignaciones de topología, puebla `targetRepos`/`allowedWriteScope` (ahora `validate changes` sí enforce) y genera un `role-<slug>.md` por rol.
- **Contexto técnico servido por MCP** (`technical-context-served`): las tools MCP devolvían `null` porque el contexto curado (`context/`) no estaba indexado. Generador de `technical-context/indexes/<rol>.index.yml` (`axiom context index`), índice `repo` materializado, y corregido el `inlineContent` (resolución relativa al índice + path-guard). El MCP ahora sirve el contexto real.
- **Paridad de comandos del antiguo wizard interactivo** (`workspace-command-parity`): todo lo que hacía el wizard es ahora ejecutable por comando — `axiom workspace setup` headless + subcomandos granulares (`spec-base`, `adapters`, `skills`, `rules`, `mcp-config`, `config-scaffold`) para reparar o reutilizar partes de una instalación.
- **Instalación por miembro** (`per-member-install` + `architect-member-handoff`): separación explícita responsabilidad **compartida/committeada** (arquitecto: `axiom.config/*`, roles, MCPs, catálogo de utilidades, spec, contexto, skills) vs **personal/gitignored** (miembro: rutas físicas en `.axiom-state/local/topology-bindings.yaml`, config MCP nativa per-máquina, estado local de utilidades). Comandos `axiom member install` y `axiom bindings`; launch MCP resoluble (`axiom` en PATH o `node <cli>`); `.gitignore` protege `.axiom-state/`. El `workspace setup` scaffoldea las declaraciones committeadas que `member install` consume.
- **Correctitud y honestidad** (`lifecycle-correctness-fixes` + `honesty-and-toolchain-states`): `archive` reubica físicamente la carpeta a `_archive/`; `self-update` lee la versión del paquete instalado; registry v1→v2 auto-migra en `init`/`repo attach`; toolchain con estados diferenciados (`declared`/`marker`/`installed-working` con probe real, ya no "present" falso); código muerto del orchestrator (`axiom intent`) eliminado y README corregido; bugs menores (doble prefijo `gateFailure:`/`upgradeFailed:`, preview dry-run adr/decision).

Ver el detalle por área en `03`/`04`/`05`/`06`/`07` (actualizados por cada incremento) y las brechas resueltas en [../context/references/03-riesgos-y-brechas-conocidas.md](../context/references/03-riesgos-y-brechas-conocidas.md).

## Ola de endurecimiento 2026-07-11 (correctness + affinity + review, cerrada)

Tanda de 3 incrementos implementados y verificados (build + suite en verde; `specs/increments/_archive/INC-20260711-audit-bug-fixes|repo-affinity-guard|per-role-review`): 7 fixes de correctitud/consistencia auditados, un **guard de repo-affinity** (cada comando de ciclo de vida enforced contra el repo correcto en multi-repo) y un **review de write-scope por rol** en `axiom-role complete` más un agregado `axiom validate changes --all-repos` desde el spec. Detalle por área en `03`/`04`/`05`/`06`. En la misma oleada quedaron **totalmente entregados y archivados** el epic `INC-20260711-sdd-launcher-core-port` (re-home del sdd-launcher de KVP25 a Axiom core: P0-P4 + PX + los tres paneles del front, cerrados en `INC-20260711-epic-close-panels`) y el **adopter de migración de repos foráneos** (`from-legacy-sdd` format-aware + ingesta de contexto + UX de adopción); ver `05`/`06`.

## Cierre de brechas post-KVP25 + front launcher/ADO (2026-07-14, cerrada)

Tras el test de integración controlado adoptando KVP25 (2026-07-13), que dejó 5 brechas de producto catalogadas (ver [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md) "Brechas conocidas (2026-07-13)"), se corrió esta tanda de 6 incrementos (`specs/increments/_archive/INC-20260714-*`), cada uno spec-first con gate re-verificado (build/test/typecheck/doctor). Suite **2791→2832 (+41 tests, cero regresiones)**; los timeouts 5000ms observados bajo carga son el flake I/O conocido, confirmados verdes en aislamiento. Las 5 brechas quedan **cerradas** y el front del launcher **rediseñado + con workflow ADO**:

- **`axiom rollback <id>`** — restore de checkpoint invocable por operador (`INC-20260714-op-rollback-restore`).
- **`axiom upgrade` cross-repo** — fan-out sobre la topología desde el repo de control, `--repo-only` conserva el modo per-repo (`INC-20260714-cross-repo-upgrade-fanout`).
- **`member install` / `adapter add` a paridad** — materializan process-surfaces + code-intel como `repo add` (`INC-20260714-member-adapter-surface-parity`).
- **Honestidad de toolchain** — `validate` sin verde-falso (marker vs installed-working), UX declare/scaffold (`INC-20260714-toolchain-install-honesty`).
- **Dedup de `configure`** — comparte el resolvedor de plantilla on-disk→bundled con `sync` (`INC-20260714-configure-generation-dedup`).
- **Front launcher rediseñado + workflow ADO** — flujo guiado de 3 pasos para crear increment/bug/plan/implementación con prompt pregenerado + selección de herramienta, y superficie ADO (crear WI, estado, estimación, horas, rama/enlace) sobre `@axiom/tracker-ado`, todo en el navegador vía `axiom app`, cero VSCode APIs (`INC-20260714-launcher-ux-ado`). Detalle en `04`/`05`/`06`.

## Roadmap de rediseño (cerrado, parcialmente implementado)

El incremento de planificación `specs/increments/_archive/INC-20260702-axiom-redesign-roadmap/` secuenció un rediseño (separación de repos por rol `sdd`/`spec`/`code`, registro global `~/.axiom/projects.yml`, manifiestos `axiom.yml` por repo, índices derivados/curados, MCP único por proyecto, la interfaz TUI contextual histórica y el sistema de versionado/migraciones) en 24 incrementos (INC-01 a INC-24). A fecha 2026-07-03, los 23 incrementos de infraestructura (Fases A-G más INC-23, dogfooding) están **cerrados** — cada uno auditó primero el paquete `Axiom/packages/*` existente (modelo de reconciliación, no greenfield) antes de extenderlo. INC-24 (Workbench) permanece explícitamente diferido, sin empezar. El detalle de qué quedó resuelto, qué quedó pendiente y el registro de las 5 preguntas de arquitectura (Q1-Q5) vive repartido en las secciones correspondientes de `01` a `08` de esta carpeta (topología y registro en [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md), modelo de artefactos y bootstrap en [01_Requisitos_Funcionales.md](01_Requisitos_Funcionales.md) y [04_Flujos_SDD_y_Ciclo_de_Vida.md](04_Flujos_SDD_y_Ciclo_de_Vida.md), capa MCP en [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md), gobierno en [07_Gobierno_y_Seguridad.md](07_Gobierno_y_Seguridad.md)). El histórico completo de las 71 carpetas de incremento de esta cadena vive archivado en `specs/increments/_archive/`; el índice/resumen de cierre del roadmap está en `specs/increments/_archive/INC-20260702-axiom-redesign-roadmap/README.md`. No confundir el modelo de `axiom.yaml schemaVersion: 1` (vigente por defecto, ver "Dos modelos distintos" arriba) con el `schemaVersion: 2` opt-in que este roadmap añadió de forma aditiva.

## Principios del producto

1. Spec first, implementación second.
2. Separación clara entre builder tooling y product runtime.
3. Axiom model primero; generated config después.
4. La trazabilidad local append-only es transversal y usa audit trail, sidecar SHA-256 y retención `P365D`.
5. El runtime opera con la política `local-only` implícita; no existe una selección activa de overlays ni gateway.
6. Las capabilities se materializan con `builder` implícito más un adapter target; el runtime separa 16 capabilities provider-routed de las tres lecturas MCP-only `axiom.*`.
7. Las integraciones de tooling son adapters opcionales, con niveles de soporte explícitos (`multi-mode`, `single-mode`, `fallback-only`).
8. Minimizar el gasto de tokens sin reducir la auditabilidad.
9. Mantener una arquitectura modular y reemplazable (`Result<T,E>` sin excepciones, escritura atómica, path-guard por proyecto).
10. Evitar vendor lock-in.

## Roadmap y estado operativo

No existe hoy `plans/PLAN-PRODUCT-ROADMAP.md` en `Axiom.Spec/plans/` (la carpeta solo tiene un `README.md` de propósito, sin contenido de roadmap). El estado operativo consolidado y verificable vive en:
- `Axiom/README.md` (tabla de capas y paquetes operativos);
- `Axiom.Spec/decisions/0015-…`, `0019-…`, `0026-…` a `0031-…` (ocho ADR migrados) y `0032-…` (frontera entre `Axiom.Spec/` y `Axiom/axiom.spec/`);
- `specs/increments/_archive/INC-20260702-axiom-redesign-roadmap/README.md` (roadmap de rediseño archivado — ver sección anterior);
- `specs/increments/_archive/` (histórico de las 71 carpetas de incremento del roadmap de rediseño).

## Front visual de onboarding y manuales (2026-07-15) — tanda INC-20260715-*

El launcher web (`axiom app`) evoluciona de superficie de ejecución a **front visual completo de onboarding**: un equipo recién instalado puede instalar Axiom en un proyecto nuevo, unirse a uno existente, registrar roles y asociarlos a repos (con explorador de carpetas), ver el registro y ejecutar incrementos/bugs/planes/implementación — todo desde el navegador, sin terminal ni TUI, con un **gate de doctor** que verifica el proyecto antes de lanzar y con el **prompt pregenerado por adapter seleccionado** (enriquecido con tuning de agente `low`/`pragmatic`). Si el plugin de Azure DevOps está configurado, la creación ofrece además el work item correspondiente. La spec incorpora `specs/manuales/` (guías de operación cruzadas). Detalle funcional en [01_Requisitos_Funcionales.md](01_Requisitos_Funcionales.md) (RF-AXM-029..033), superficies en [05_Interfaces_Operativas.md](05_Interfaces_Operativas.md), manuales en [manuales/README.md](manuales/README.md).

## Alineación del ciclo SDD instalado con sistemas role-specialized (2026-07-15) — tanda INC-20260715-*

Tras revisar Axiom contra un sistema SDD role-specialized (KVP25 `.github`), el set que Axiom **instala** en un proyecto pasa de "spec/plan/impl sólidos, review/QA/consolidación finos" a un ciclo completo con **gates y disciplinas instaladas**: revisión por fase spec/plan/código (`axiom-phase-reviewer`, OK/KO), QA-validation desde criterios (`axiom-qa-validator`), revisión de seguridad con cuerpo real (`axiom-security-reviewer`), consolidación a spec canónica + archivado (`axiom-spec-integrator`), autoría de contexto técnico + spec-drift (`axiom-tech-context`), un análisis de alcance opcional en el planner, y las 4 disciplinas transversales como skills reutilizables (dudas estructuradas, cobertura `CF-xx`, drift, cierre documental). La genericidad se preserva: lo específico de cada stack se inyecta por proyecto (`skills-index/<role>.yaml` + contexto técnico + skills de rol), no se hornea en el producto. Detalle en [01_Requisitos_Funcionales.md](01_Requisitos_Funcionales.md) (RF-AXM-034..039) y [manuales/13_Skills_Agentes_y_Roles.md](manuales/13_Skills_Agentes_y_Roles.md).

## Graduación a full lifecycle: repo único `<project>.axiom`, worktrees y stack externo (2026-07-24) — tanda INC-20260724-*

Graduación consciente de Axiom del modo *bootstrap* al *full product lifecycle* (excepción explícita, user-approved, a los "Explicit Bootstrap Limits"): un batch de 18 incrementos que cambia lo que Axiom hace **en los proyectos donde se instala**. Los repos propios de Axiom (`Axiom.SDD`/`Axiom.Spec`) **no** se migran a este modelo (dogfooding diferido al usuario). Ejes:

- **Modelo de repo único `<project>.axiom`** (Cluster A): el par `sddRepo`+`specRepo` se sustituye por un único repo gestionado que concentra todo el "cerebro". La adopción **crea** ese repo nuevo y migra la spec/context legacy dentro, dejando los repos legacy **intactos** (read-only). Metadata de provenance/lifecycle (`migrated`/`migrated-and-modified`/`axiom-native`) + manifest de migración habilitan `axiom eject` para salir de Axiom sin tocar legacy. Un **MCP unificado** (broker `axiom`) reemplaza los dos brokers para los code repos. Topología `schemaVersion: 2` (aditiva, retrocompatible).
- **Stack externo reordenado** (Clusters T/S): `cmm` (ADR-0031) pasa a ser el único proveedor estructural de code-intel (fuera `graphify`/`codegraph`), `serena` sigue simbólico; RTK y la concisión (filosofía de Caveman) se adoptan como **skills** (sin hooks ni runtime); AutoSkills gana provenance + fecha + policy allow/deny/licencia.
- **Worktrees como modo de ejecución** (Cluster W): un incremento/bug/plan puede ejecutarse en un git worktree aislado (default elegido por el arquitecto en la instalación, overridable por run), con entidad `Execution` de primera clase, provisioning portable, aislamiento de providers/índices por worktree, harvest+cleanup seguro y freshness de artefactos SDD (auto-fetch + aviso de desfase, sin bloquear).
- **Transversal** (Cluster X): barrido e2e + revisión adversarial que encontró y arregló 2 defectos HIGH de cierre en worktree.

Detalle: modelo/datos en [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md), flujos en [04_Flujos_SDD_y_Ciclo_de_Vida.md](04_Flujos_SDD_y_Ciclo_de_Vida.md), superficies en [05_Interfaces_Operativas.md](05_Interfaces_Operativas.md), capacidades en [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md), gobierno en [07_Gobierno_y_Seguridad.md](07_Gobierno_y_Seguridad.md), requisitos en [01_Requisitos_Funcionales.md](01_Requisitos_Funcionales.md) (RF-AXM-040..048).

## Estado reconciliado de adapters y onboarding (2026-07-26 en adelante)

La tanda original de 7 incrementos llevo el conjunto de adapters a paridad de primera clase y convirtió el launcher web (`axiom app`) en un front de onboarding config-rich, con probes de runtime opt-in en `doctor`. Las decisiones posteriores de R-07 dejan este estado vigente:

- **Targets activos**: se mantienen 8 (`opencode`, `claude-code`, `github-copilot`, `vscode`, `cursor`, `antigravity`, `visual-studio-2026` y `codex`). `copilot-vscode` no pertenece a la API pública: `axiom configure` únicamente migra ese literal si ya estaba persistido en `init.json#profileTriple.adapterTarget`, antes de instalar o despachar. LiteLLM fue retirado.
- **Instrucciones comunes**: GitHub Copilot, Copilot CLI, VS Code y Visual Studio reutilizan `.github/copilot-instructions.md`; Visual Studio no genera `.vs/AXIOM.md`.
- **MCP project-bound**: la proyeccion valida proyecto, `mcp.yml` y manifest antes de escribir; limpia solo IDs gestionados por Axiom, preserva servidores custom y no escribe ni recomienda configuracion global para Codex/Antigravity sin binding seguro.
- **Launcher a paridad**: rutea y ofrece los 8 adapters headline (+cli) con routing real (skill + `sdd.transitionApply`), y cada prompt pregenerado ahora lleva DÓNDE leer (rutas de spec/plan/metadata) + QUÉ MCP/tool + QUÉ skill — adapter-neutral.
- **Doctor con probes de runtime (opt-in)**: `axiom doctor --deep` (y `?deep=1` en el launcher) añade un probe funcional de tool (`TC-018`) y un descubrimiento `server/discover` MCP `2026-07-28` real de liveness (`TC-019`); aditivo y never-fail (nunca cambia el veredicto del doctor síncrono).
- **Front de onboarding config-rich**: install/join exponen profile/overlay/layout/role/adapter-primario/adapters-adicionales/execution-mode (todos cableados a rutas reales) y muestran las tools a instalar honestamente (diferido: sin ruta limpia de instalación de toolchain desde init). Gate de confirmación preservado.

Detalle: requisitos en [01_Requisitos_Funcionales.md](01_Requisitos_Funcionales.md) (RF-AXM-049..055), no funcionales en [02_Requisitos_No_Funcionales.md](02_Requisitos_No_Funcionales.md) (NFR-AXM-020/021), capacidades en [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md), glosario en [08_Glosario.md](08_Glosario.md).

## Scaffolding de config de adopción + launcher por defecto (2026-07-27) — tanda INC-20260727-*

Cierre e integración (pasada de consolidación 2026-07-29) de los incrementos `INC-20260727-*` que quedaban cerrados pero sin volcar a esta spec. Los de plugin ADO en el launcher ya estaban integrados (ver [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md) §"Plugin ADO en el launcher"); estos dos cerraban huecos de adopción/onboarding:

- **Scaffolding de config de adopción** (`INC-20260727-adoption-config-scaffolding`): `runWorkspaceSetup` (motor compartido por `axiom workspace setup` y `axiom workspace adopt`) siembra ahora, best-effort y no-clobber, los 4 artefactos de config que a un proyecto recién adoptado le faltaban — `axiom.config/integrations.yaml` (**PC-001**), `axiom.config/policy-as-code.yaml` (**PC-002**), `axiom.config/agents-catalog.yaml` (**TC-011**) y el `axiom.skills.lock` raíz (**GC-001/GC-002/GC-007**) — con `bundleHash` calculado contra el mismo fichero que el doctor recomputa, de modo que un proyecto scaffoldeado de cero pasa esas 5 checks out-of-the-box. Detalle en [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md).

## Knowledge Harvest y memoria entre fases (R-11/R-12)

La memoria de fase se persiste únicamente en el Engram local del proyecto. `MemoryEntry` conserva la metadata trazable (`increment`, `phase`, `actorRole`, `knowledgeKind`, `stability`, `visibility`, `sourceArtifact`) codificada como frontmatter en el contenido de Engram; la lectura decodifica el envelope real de `mem_get_observation`. El proceso queda fijado por `engram mcp --project=<projectId> --tools=agent`, con handshake MCP estándar `initialize` y guards contra referencias cross-project. No hay backend, archivo, flag ni fallback JSON: si Engram no está disponible, cada operación devuelve un error visible y TC-024 de Doctor falla con la guía `engram --version`; los JSON históricos permanecen intactos pero sin consumo runtime.

`axiom memory add` es el guardado explícito. Las skills de incremento y bug deben ejecutarlo y comunicar el resultado al confirmar una decisión o un bug: `project-shared` se reserva para conocimiento confirmado, reutilizable por el equipo y libre de secretos; `private` conserva contexto local legible y excluido de Knowledge Sync, pero no admite secretos. Ningún audit trail, workflow o launcher infiere ni persiste automáticamente una decisión, bug o lección; `axiom learn` y sus lecciones derivadas del audit trail fueron retirados.

`axiom knowledge harvest --increment <id>` lee Engram, clasifica por `stability` y produce propuestas de promoción sin mutar contexto ni skills. `axiom knowledge sync --increment <id> --phase <phase>` sigue siendo preview por defecto y, con `--confirm`, exporta solo `project-shared`, preservando evidencia y metadata y excluyendo privadas, entradas sin visibilidad o secretos. `pull` importa todos los chunks pendientes y solo los marca después de persistencia íntegra. El marker personal sigue en `.axiom-state/<projectKey>/knowledge/imported-chunks.json`; los chunks/manifest JSON versionables y la base de Engram gitignored conservan responsabilidades separadas.

La consulta y el listado de memoria usan límite predeterminado de `10`. La evidencia `rationale`/`source` se valida antes de I/O cuando el caller la aporta al backend; la CLI mantiene valores explícitos por defecto para una acción humana/agent directa.

- **Launcher como front por defecto de `axiom app`** (`INC-20260727-launcher-default-and-old-ui-removal`): `axiom app` abre el launcher (`/launcher/`) por defecto y `GET /` / `GET /index.html` redirigen `302` a `/launcher/`; la vieja UI de operador raíz (`static/index.html`+`app.js`+`style.css`+`sw.js`+`manifest.json`) y sus 6 endpoints backend muertos se eliminaron (confirmados sin consumidores antes de borrar). Detalle en [05_Interfaces_Operativas.md](05_Interfaces_Operativas.md).
### Tanda `INC-20260730-*` — gobierno verificable de la ejecución desatendida (2026-08-02)

Seis incrementos que cierran el bloque de gobierno de los flujos desatendidos, con la propiedad común de que un orquestador o subagente pueda **comprobar** el estado sobre el que actúa en vez de confiar en él:

- **Errores tipados** (`INC-20260730-typed-recovery`): `AxiomError` + `AXIOM_ERROR_CODES` (12 códigos anclados a throw sites reales); 45 sitios tipados frente a los 3 previos; recovery por `error.code`, no por texto de mensaje. Corrige de paso un `instanceof` roto en toda subclase de `AxiomError`.
- **Evidencia en memoria** (`INC-20260730-engram-evidence`): `rationale`/`source` fail-closed en runtime antes de cualquier I/O, enforced desde los tres puntos de entrada del backend.
- **Recibos de fase** (`INC-20260730-phase-receipts`): emisión automática en cada transición, en éxito y en fallo, best-effort/never-block, vía wrappers delgados que no alteran el resultado del core.
- **Freeze de candidate** (`INC-20260730-candidate-freeze`): gate de `apply` ya cableado al ciclo; endurecido el parseo del artefacto congelado.
- **Validación de config** (`INC-20260730-exact-scope`): tres loaders de `profiles.yaml` convergidos en el schema canónico ya existente y hasta entonces sin consumidores.
- **Propagación al orquestador** (`INC-20260730-autopilot-integration`): las tres directivas pasan a ser requisito formal en las 7 superficies de `axiom-autopilot`, incluidas las fuentes distribuibles que reciben los proyectos adoptantes.

Dos límites se registran explícitamente en vez de declararse cubiertos: el freeze hashea memoria filtrada + `README.md`, **no** lockfiles ni `metadata.yml`; y el receipt cubre los fallos con `exitCode` distinto de cero pero no una excepción que escape del core de la transición.

Detalle en [01_Requisitos_Funcionales.md](01_Requisitos_Funcionales.md) (RF-AXM-057..061), propiedades en [02_Requisitos_No_Funcionales.md](02_Requisitos_No_Funcionales.md) (NFR-AXM-023), artefactos en [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md), gates en [07_Gobierno_y_Seguridad.md](07_Gobierno_y_Seguridad.md).
