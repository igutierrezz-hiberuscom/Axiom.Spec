# MCP unificado moderno de Axiom

> **Código**: INC-20260810-mcp-unified-modern  
> **Estado**: Archivado e integrado  
> **Fecha de creación**: 2026-08-10  
> **Tipo de cambio**: Migración  
> **Acción de auditoría**: ACC-030  
> **Plan padre**: `PLAN-REVISION-INTEGRAL-AXIOM`

## Resumen

Axiom pasa de varios brokers MCP gestionados por separado a un único broker
project-scoped: `axiom-mcp-broker`. El broker expondrá en una sola superficie
las capabilities `sdd.*`, `spec.*`, `axiom.*` y `memory.*`, incluyendo lecturas
y mutaciones controladas de estados de incrementos, bugs, planes y workflow.

El transporte seguirá siendo stdio con mensajes JSON-RPC delimitados por líneas,
pero el protocolo será únicamente MCP `2026-07-28`. No se mantendrá una rama de
compatibilidad con `initialize`, `notifications/initialized`, `ping` ni otras
semánticas legacy porque todavía no existen clientes ni proyectos Axiom reales
que deban conservarlas.

## Contexto y motivación

La implementación actual conserva restos de varias decisiones históricas:

- `sdd-mcp-server` expone capabilities del dominio `sdd`.
- `spec-mcp-broker` expone capabilities del dominio `spec`.
- `memory-mcp-server` es un tercer kind para memoria local.
- `axiom-mcp-broker` existe como broker experimental, pero no cubre toda la
  superficie y no es el broker materializado por el setup normal.
- El protocolo actual usa MCP `2024-11-05` con handshake legacy.

La arquitectura objetivo del producto es más simple: un repo Axiom debe tener
un MCP gestionado y project-scoped, con una única lista de tools y una única
identidad de proceso. Los prefijos de capability siguen siendo útiles para
clasificar handlers, documentación y routing interno, pero dejan de representar
servidores MCP independientes.

## Alcance

### Incluido

- Un único `McpServerKind`: `axiom`.
- Una única entrada gestionada: `axiom-mcp-broker`.
- Unión completa de las capabilities registradas en los dominios `sdd`,
  `spec`, `axiom` y `memory`.
- Lecturas y escrituras en el mismo broker.
- Protocolo MCP `2026-07-28` sin compatibilidad dual.
- `server/discover`, metadata `_meta` por petición, negociación de versión,
  `UnsupportedProtocolVersionError` (`-32022`), `resultType`, resultados
  cacheables y orden determinista de `tools/list`.
- Cancelación mediante `notifications/cancelled` y aceptación de
  `progressToken`; el servidor puede no emitir progreso cuando una operación
  es corta.
- Preview y confirmación para mutaciones ya existentes, incluyendo cambios de
  estado workflow, ramas y sincronización Git.
- Materialización en `mcp-manifest.yaml`, `.axiom/mcp.yml`, configuraciones
  nativas y `member-install` de una sola entrada `axiom-mcp-broker`.
- Tests unitarios, tests de transporte, tests de configuración y E2E con el
  dispatcher real y un proceso stdio lanzado.
- Actualización de la documentación activa y del contexto técnico una vez
  verificado el resultado.

### Excluido

- Compatibilidad con MCP `2024-11-05` o cualquier handshake legacy.
- Transporte HTTP o Streamable HTTP en esta entrega.
- MCP Tasks; se deja para un incremento posterior solo si aparece una
  operación realmente larga o con interacción prolongada.
- Un nuevo backend de memoria: Engram continúa siendo un backend/provider
  local opcional consumido por las tools `memory.*`.
- Shell arbitrario, mutaciones sin confirmación o bypass del aislamiento
  project-scoped.
- Cambios de producto no relacionados con MCP.

## Decisiones cerradas

| ID | Decisión | Motivo |
|----|----------|--------|
| D-001 | Un MCP gestionado por repo Axiom | El usuario considera innecesario mantener brokers SDD y Spec separados. |
| D-002 | Los prefijos `sdd.*`, `spec.*`, `axiom.*` y `memory.*` permanecen como dominios internos | Permiten conservar el routing y la trazabilidad sin crear procesos MCP separados. |
| D-003 | El broker expone lectura y escritura | El agente debe poder consultar contexto y solicitar cambios de estado controlados. |
| D-004 | Solo MCP `2026-07-28` | No hay clientes ni proyectos reales que exijan compatibilidad legacy. |
| D-005 | Tasks no forma parte de la primera entrega | No existe todavía una operación Axiom que necesite un handle asíncrono durable. |

## Dependencias

- `ACC-025` a `ACC-029` finalizadas y validadas.
- `@axiom/mcp-tools` como registry y dispatcher de handlers.
- `@axiom/mcp-server` como transporte y protocolo.
- `@axiom/providers` para el cliente stdio local usado por providers.
- `@axiom/doctor` y los writers de configuración MCP.

## Validación prevista

- Tests de protocolo y dispatcher en `packages/mcp-server/tests/`.
- Tests del cliente stdio en `packages/providers/tests/`.
- Tests de configuración y manifest en `apps/cli/tests/` y
  `packages/user-workspace/tests/`.
- E2E de workspace y proceso MCP real sobre stdio.
- `npm run build` y los checks MCP/doctor relevantes.
- Revisión independiente contra estos criterios y contra el plan padre.

## Integración de spec y contexto

Al cerrar el incremento, el orquestador actualizará una sola vez los documentos
canónicos `Axiom.Spec/specs/00..08` y los documentos aplicables de
`Axiom.Spec/context/**`. La integración debe retirar afirmaciones activas que
presenten `sdd-mcp-server`, `spec-mcp-broker` o `memory-mcp-server` como brokers
gestionados independientes; las referencias históricas pueden permanecer solo
en archivos archivados.

## Implementation notes

- El broker gestionado se materializa exclusivamente como `axiom-mcp-broker` y
  se lanza con `--kind axiom`.
- `McpServerKind` queda cerrado en `axiom`; `sdd`, `spec` y `memory` siguen
  siendo prefijos internos del registry, no procesos ni aliases de servidor.
- La superficie de `axiom` se deriva de todas las entradas actuales de
  `@axiom/mcp-tools`, de forma que las lecturas y las mutaciones registradas
  (`sdd.transitionApply`, `sdd.gitRoleBranch` y `sdd.gitCommitSync`) comparten
  el mismo dispatcher y sus guards existentes.
- Para evitar una topología ambigua, la entrada única apunta al repo de
  control/Axiom que materializa y resuelve la topología del proyecto. No se
  generan entradas separadas para los repos SDD o Spec.
- El wire protocol se implementa como MCP `2026-07-28` sobre stdio
  newline-delimited, sin handshake legacy ni compatibilidad dual. `server/discover`
  y `tools/list` usan envelopes completos y metadatos deterministas.

## Assumptions and resolved ambiguity

- La proyección nativa mantiene servidores externos opcionales (por ejemplo,
  `engram`) únicamente cuando son providers/backend de memoria; no se
  consideran brokers gestionados de Axiom y no se escriben en `.axiom/mcp.yml`.
- La cancelación se trata como estado transitorio por petición: se acepta
  `notifications/cancelled`, se suprime la respuesta pendiente y no se
  persiste estado MCP. `progressToken` se acepta como metadata de la petición;
  no se introduce progreso sintético ni Tasks para operaciones cortas.

## Estado del resultado

**Status: archived; integración canónica completada**

El broker único y el protocolo MCP moderno están implementados y verificados. La revisión
independiente encontró y se corrigió un bypass de paths, un binding de arranque
demasiado permisivo y un `targetRepo` incorrecto en worktrees. La integración
activa en `Axiom.Spec/specs/00..08` y `Axiom.Spec/context/**` ya fue
reconciliada; el candidato fue re-congelado, verificado y archivado mediante
`axiom integrate`.

## Archivos y decisiones implementadas

- `Axiom/packages/mcp-server`: protocolo MCP `2026-07-28`, `server/discover`,
  error `-32022`, envelopes `resultType: complete`, metadata/cache, cancelación,
  binding fail-closed y unión completa del registry.
- `Axiom/packages/providers`: cliente stdio sin handshake legacy, metadata
  moderna en cada request/notification, `discover`, cancelación y consumidores
  de code-intel; `Axiom/packages/memory` mantiene Engram como backend opcional y
  también usa `discover`.
- `Axiom/apps/cli/src/commands`: `mcp-serve`, `workspace-mcp`,
  `workspace-config-scaffold`, `native-mcp-config`, `workspace-setup`,
  `workspace-incremental` y `member-install` convergen en una entrada
  `axiom-mcp-broker` con `--kind axiom` y root de control/Axiom.
- `Axiom/packages/doctor` verifica el único broker mediante
  `server/discover`; `Axiom/axiom.config/{mcp-manifest,integrations}.yaml`
  declaran solo `axiom-mcp-broker`.
- `Axiom/packages/isolation` permite por defecto únicamente `axiom-mcp-broker`;
  `serena`, `cmm` y `engram` siguen siendo providers/backend externos y no
  brokers gestionados de gobierno.
- Las pruebas MCP, providers, memory, doctor, configuración, member-install e
  E2E fueron migradas a la superficie única. Los ids legacy solo aparecen en
  una lista explícita de limpieza de configuración nativa stale; nunca se
  emiten como servidores deseados.

## Receipts de validación

- `npm run build` desde `Axiom/`: **PASS**.
- Typechecks de `@axiom/mcp-server`, `@axiom/providers`, `@axiom/memory`,
  `@axiom/doctor` y `@axiom/cli`: **PASS**.
- Batería directa de protocolo, guards, transición, scope y broker E2E:
  **32 tests PASS**; batería de worktree y aislamiento: **48 tests PASS**.
- E2E de proceso real: CLI compilada lanzada como hijo, `server/discover`,
  `tools/list` completo, lectura y transición preview/confirmación por stdio:
  **PASS**.
- Doctor/liveness MCP moderno y E2E `doctor --deep`: **PASS**.
- Batería amplia final: **71 archivos / 709 tests PASS**, con dos fallos
  preexistentes de `ACC-025` (`TC-009` e `IP-003`) por la retirada de LiteLLM.
- Probe contra `packages/mcp-server/dist`: un junction que apunta fuera del
  proyecto es rechazado por el guard real de symlinks.
- `packages/isolation/tests/p0.test.ts` y las suites MCP de aislamiento:
  **45 tests PASS** tras retirar la allowlist lógica legacy `sdd`/`spec`.
- `git diff --check`: **PASS**; solo quedaron avisos informativos de normalización
  de finales de línea CRLF del worktree de Windows.

## Revisión y clasificación

- Review independiente: los findings P1 de aislamiento, binding y worktree
  quedaron corregidos y verificados; los residuos documentales activos de
  brokers separados también quedaron reconciliados.
- Fallos encontrados durante la migración: expectativas históricas de
  `sdd/spec/memory` y handshake `initialize`, además de `dist` desactualizado;
  se clasificaron como contratos/artefactos preexistentes y fueron migrados o
  regenerados.
- Persisten dos fallos ajenos en la batería amplia de doctor sobre adapters y
  perfiles (`TC-009`/`IP-003`), originados por la retirada de LiteLLM de
  `ACC-025`; no son regresiones de ACC-030.
- No se ejecutó la suite completa a ciegas. El worktree contiene cambios de
  acciones anteriores y no se revirtieron ni se incorporaron fuera del alcance
  de ACC-030.

## Post-cierre (2026-08-10): corrección de TC-009 e IP-003

Los dos fallos preexistentes de `ACC-025` fueron corregidos en
`Axiom/packages/doctor/tests/{adapters,install-profiles}.test.ts`:

- **TC-009** (`adapters.test.ts`): la expectativa pasó de `9/9` a `8/8`
  adapters, reflejando la retirada de LiteLLM (ACC-025). El check real ya
  reportaba `8/8`; el test quedó alineado con el contrato activo.
- **IP-003** (`install-profiles.test.ts`): el fixture "canonical" usaba el
  alias legacy `copilot-vscode` en `allowedTargets`, que no es un id
  canónico. Se sustituyó por `github-copilot`, el id canónico real.

Validación post-corrección: `packages/doctor` pasa completo (20 archivos /
212 tests), `npm run build` PASS y `doctor --deep` PASS con 0 fallos. La
suite amplia queda en `711/711` salvo el E2E de worktree
(`worktree-provider-isolation.e2e.test.ts`), que falla por entorno (el
proyecto no se registra en el registry MCP v2 de `~/.axiom` porque `runInit`
usa `register: false`), no por estos cambios — se confirmó que falla igual
sin ellos.

### Corrección adicional del E2E de worktree (2026-08-10)

El E2E `worktree-provider-isolation.e2e.test.ts` también fue corregido. El
fallo no era de entorno sino del propio test, que no estaba alineado con el
gate MCP project-bound introducido por `ACC-029`/`ACC-030`:

- No registraba el proyecto en el registry v2 (`addProjectV2`), por lo que
  `filterProjectBoundMcpServers` no podía confirmar la identidad del
  proyecto.
- No sembraba el binding MCP project-scoped (`.axiom/mcp.yml` +
  `axiom.config/mcp-manifest.yaml`), por lo que no se proyectaba ningún
  broker.
- Usaba el home real (`~/.axiom`) en vez de un `homeDirOverride` temporal.

Se alineó con el patrón de `worktree-provision.e2e.test.ts` (que sí pasaba):
`homeDirOverride` temporal, `addProjectV2` y `seedProjectMcpBinding`. Con
esto la suite completa queda en **326 archivos / 3248 tests PASS**, build
PASS y doctor PASS con 0 fallos.

## Integración realizada

Se actualizaron las superficies canónicas aplicables en `Axiom.Spec/specs/00..08`
y `Axiom.Spec/context/**`: el contrato vigente es `axiom-mcp-broker`, los
dominios de capability no son procesos, MCP usa `2026-07-28` y Engram queda
como backend/provider opcional. Las referencias legacy permanecen únicamente en
artefactos históricos archivados.
