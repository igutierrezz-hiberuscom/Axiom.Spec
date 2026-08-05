# Plan de revisión integral de Axiom

## Objetivo

Entender el runtime actual de `Axiom/` de arriba abajo antes de decidir qué simplificar, retirar, corregir o ampliar. La revisión se realizará en sesiones independientes y este documento será el registro único de áreas, hallazgos y acciones propuestas.

Durante esta fase no se modifica el producto. Las decisiones de cambio se registran aquí y se ejecutan después mediante incrementos o bugs separados, con su propia validación.

## Preparación de cada sesión

1. Abrir el workspace completo: `Axiom.SDD`, `Axiom.Spec`, `Axiom` y `Axiom.Pruebas`.
2. Iniciar una conversación limpia y seleccionar `Axiom Runtime Auditor` en Copilot, o escribir `Auditor Axiom: R-XX` en los otros targets.
3. Indicar el identificador del área, por ejemplo:

   ```text
   Auditor Axiom: R-04. Explícame esta área desde cero y no cambies nada fuera del plan de revisión.
   ```

4. Resolver dudas durante la conversación. El agente explica primero y solo anota una acción cuando se le pide expresamente.
5. Cerrar la sesión con un resumen breve. El siguiente área empieza en una sesión limpia, no con un contexto acumulado.

## Método de revisión

Para cada área el agente debe:

1. Localizar las fuentes reales: código, configuración, tests y documentación.
2. Explicar en castellano claro qué es cada pieza, por qué existe, cómo funciona, cómo se usa y con qué está conectada.
3. Trazar al menos un flujo concreto de entrada a salida cuando aplique.
4. Distinguir hechos verificados de documentación histórica o afirmaciones no confirmadas.
5. Registrar únicamente las acciones que el usuario solicite conservar.

La prioridad de evidencia es: comportamiento ejecutable y tests, código fuente, configuración activa, documentación operativa y, por último, documentación histórica. Una discrepancia se registra como investigación, no como eliminación automática.

## Áreas y sesiones

| ID | Área | Alcance principal | Pregunta guía |
| --- | --- | --- | --- |
| R-00 | Mapa y reglas de revisión | Raíces del workspace, ownership, entradas de documentación y límites de la auditoría | ¿Qué repositorio es responsable de cada cosa y qué fuentes describen el estado actual? |
| R-01 | Estructura, build y empaquetado | `package.json`, TypeScript project references, workspaces, scripts, `dist/`, configuración de tests | ¿Qué se compila, qué se publica o ejecuta y qué artefactos pueden ser residuales? |
| R-02 | Shell de la CLI y registro de comandos | `apps/cli/src/index.ts`, bindings, bootstrap, helpers compartidos y registro de comandos | ¿Cómo entra una orden al runtime y qué comandos están realmente conectados? |
| R-03 | Lifecycle básico de proyecto | `init`, `join`, `configure`, `sync`, `start` y sus dependencias directas | ¿Qué crea o modifica cada paso y cuál es la secuencia mínima útil? |
| R-04 | Configuración y modelo de capacidades | `axiom.config/`, `config-validation`, `capability-model`, `providers`, `install-profiles` | ¿Qué YAML decide el comportamiento y qué configuración se usa realmente? |
| R-05 | Estado, filesystem y aislamiento | `core`, `persistence`, `filesystem-truth`, `project-resolution`, `isolation`, `installer` | ¿Dónde persiste Axiom el estado, cómo encuentra un proyecto y cómo evita mezclar proyectos? |
| R-06 | Scaffolding y superficies generadas | Comandos de scaffold, `document-bootstrap`, plantillas y archivos emitidos | ¿Qué archivos se generan, cuáles se pueden editar y quién los regenera? |
| R-07 | Targets y adapters | `packages/adapters/*`, support matrix y proyección a OpenCode, Claude Code, Copilot, Antigravity y otros targets | ¿Qué soporte es real por target, qué archivos recibe y qué queda en fallback? |
| R-08 | Agents, skills y componentes | `agents`, `skills`, `components`, catálogos y materializadores | ¿Qué contratos se instalan para los agentes, qué hacen de verdad y qué puede sobrar o estar duplicado? |
| R-09 | Salud, auditoría y telemetría | `doctor`, `audit`, `telemetry`, políticas y comandos asociados | ¿Qué se valida, qué bloquea operaciones y qué datos se registran o emiten? |
| R-10 | Workflow SDD y gobierno de cambios | `workflow`, `orchestrator`, `cavekit-discipline` y comandos de incrementos, bugs, planes, roles, fases y freeze | ¿Cuál es el flujo SDD que se ejecuta realmente y qué rutas son solo contratos o stubs? |
| R-11 | Contexto técnico e integraciones de herramientas | `toolchain`, `tool-routing`, `mcp-server`, `mcp-tools`, `technical-context`, `knowledge*` | ¿Cómo obtiene y entrega contexto Axiom, y qué integraciones son necesarias frente a aspiracionales? |
| R-12 | Memoria, trackers y sincronización externa | `memory`, `tracker`, `tracker-ado`, `external-sync`, `learn` | ¿Qué información mantiene Axiom, dónde queda y qué dependencias externas añade? |
| R-13 | Workspace de usuario, topología y launcher | `user-workspace`, `topology`, `launcher`, `projects`, `repo`, `gateway`, `self-update` | ¿Qué vive fuera del proyecto, cómo se conectan varios repositorios y qué automatismos se activan? |
| R-14 | Control de operador e interfaces | `model-routing`, `versioning`, `upgrade`, `tui`, `app*` y comandos relacionados | ¿Qué controles ya son operativos, qué interfaz los expone y qué partes son solo prototipo? |
| R-15 | Documentación, spec e historial | `docs/`, `axiom.spec/`, `Axiom.Spec/`, referencias legacy y fuentes duplicadas | ¿Qué documentación está vigente, cuál es histórica y dónde hay contradicciones que puedan confundir? |
| R-16 | Pruebas, coherencia y cierre del inventario | Vitest, scripts de readiness y doctor, artefactos `dist/` y síntesis de los hallazgos | ¿Qué está cubierto, qué no se puede verificar y qué acciones deben priorizarse? |

## Orden recomendado

Completar las áreas en orden. Si surge una dependencia, se puede consultar puntualmente otra área, pero la decisión se registra en la sesión propietaria. R-15 y R-16 se reservan para contrastar las conclusiones contra el estado real y priorizar la lista final.

## Registro de sesiones

El agente agrega aquí una entrada por sesión terminada. No debe reemplazar entradas anteriores.

| Fecha | Área | Resumen verificado | Dudas abiertas | Referencias revisadas |
| --- | --- | --- | --- | --- |
| 2026-08-03 | R-00 | Se verificó el ownership del workspace: `Axiom.Spec/` es el repo canónico, `Axiom/` contiene el runtime y `Axiom.SDD/` contiene el workflow; se migraron los ocho ADR a `Axiom.Spec/decisions/`, se alineó la documentación y se conservó `Axiom/axiom.spec/` como baseline product-owned por sus consumidores activos. | Ninguna bloqueante. La similitud nominal queda mitigada y documentada por ADR-0032. | `Axiom.SDD/AGENTS.md`, `Axiom.Spec/README.md`, `Axiom.Spec/decisions/`, `Axiom.Spec/decisions/0032-axiom-spec-boundary-and-runtime-baseline.md`, `Axiom/axiom.yaml`, `Axiom/axiom.config/topology.yaml`, consumidores de `Axiom/axiom.spec/`. |
| 2026-08-04 | R-01 | Se verificó que el build raíz usa TypeScript project references: cada paquete genera su propio `dist/`, la CLI termina en `apps/cli/dist/index.js` y los artefactos generados están ignorados por Git. `@axiom/cli-commands` y `@axiom/tui` usan hoy `rootDir: "../.."`; `cli-commands` además incluye fuentes que viven bajo `apps/cli`, por lo que emite rutas anidadas y declara `dist/packages/...` como `main/types`. TUI solo incluye fuentes bajo su propio `src` y pasa una comprobación con raíz local. El build completo terminó correctamente y el test nativo de instalación pasó 18/18. | La salida anidada de `cli-commands` no se puede corregir solo cambiando `rootDir` a `src`: TypeScript falla porque sus fuentes y dependencias directas están fuera de esa raíz. Hay que decidir cómo conservar el ownership único sin mezclar raíces. TUI parece una normalización independiente y más sencilla. La suite completa de Vitest produjo una salida excesiva y no se obtuvo un resumen final limpio en esta sesión; las pruebas focalizadas y el build sí fueron comprobados. | `Axiom/package.json`, `Axiom/tsconfig.json`, `Axiom/tsconfig.base.json`, `Axiom/vitest.config.ts`, `Axiom/apps/cli/package.json`, `Axiom/apps/cli/tsconfig.json`, `Axiom/apps/cli/README.md`, `Axiom/packages/cli-commands/package.json`, `Axiom/packages/cli-commands/tsconfig.json`, `Axiom/packages/cli-commands/src/index.ts`, `Axiom/packages/tui/package.json`, `Axiom/packages/tui/tsconfig.json`, `Axiom/scripts/install-global.test.mjs`, `Axiom/.gitignore`. |
| 2026-08-04 | R-02 | Se verificó que `Axiom/apps/cli/src/index.ts` crea la CLI, registra la superficie de comandos en un orden explícito, define `doctor` directamente, inicializa la telemetría de forma no bloqueante, abre la TUI cuando no se indica subcomando y finalmente entrega los argumentos al enrutador de comandos. La ayuda compilada expone los comandos registrados. Se confirmó la separación entre el registro (`registerX`) y el trabajo (`runX`) en `init`, `join`, `bindings` y `bootstrap`; las pruebas enfocadas pasaron 24/24. | No se verificó el comportamiento completo de cada subcomando. `phase`, `freeze` y `doctor` contienen parte de su lógica directamente en el registro de la CLI. Un comentario antiguo de `@axiom/cli-commands` dice que no se reexporta `registerX`, pero el código vigente sí lo hace y declara que esa decisión fue superada. | `Axiom/apps/cli/src/index.ts`, `Axiom/apps/cli/src/commands/_shared.ts`, `Axiom/apps/cli/src/commands/init.ts`, `Axiom/apps/cli/src/commands/join.ts`, `Axiom/apps/cli/src/commands/bindings.ts`, `Axiom/apps/cli/src/commands/bootstrap.ts`, `Axiom/packages/cli-commands/src/index.ts`, `Axiom/apps/cli/tsconfig.json`, `Axiom/apps/cli/tests/bindings.test.ts`, `Axiom/apps/cli/tests/bootstrap.test.ts`. |
| 2026-08-04 | R-03 | Se verificó el flujo básico: `init` crea `axiom.yaml`, el estado local y `init.json`; `join` solo registra miembros y es opcional; `configure` compone `install-profile.json`; `sync` regenera outputs solo después de pasar su gate; `start` comprueba un primer ruteo y escribe `last-start.json`. Las pruebas focalizadas pasan: 5 archivos, 50 pruebas. | El overlay `enterprise` exige `lineage` y `verification`, pero el `telemetry-sinks.yaml` revisado declara otros tipos de señal; la documentación de `init` aún afirma que crea `topology.yaml`, mientras código y pruebas actuales indican que no lo hace. No se registró ninguna acción. | `Axiom/apps/cli/src/commands/init.ts`, `join.ts`, `configure.ts`, `sync.ts`, `start.ts`, `_shared.ts`, `Axiom/packages/orchestrator/src/state-machine.ts`, `Axiom/packages/telemetry/src/loader.ts`, pruebas focalizadas en `Axiom/apps/cli/tests/`, `Axiom/axiom.config/profiles.yaml`, `telemetry-sinks.yaml`, `providers.yaml`, `Axiom/docs/cli/`. |
| 2026-08-04 | R-13/R-14 (consulta cruzada) | Se verificó que `axiom app` ya abre por defecto el launcher web local bajo `/launcher/`, con onboarding de instalación y unión, catálogo de acciones, previews, ejecución confirmada, registro de incrementos/bugs/planes, doctor y paneles ADO/Git. También se verificó que `axiom tui` sigue registrado como superficie independiente y que el paquete `@axiom/tui` mantiene su propio driver, pantallas, flujos y pruebas. | El launcher actual no cubre todavía de forma demostrada todo el recorrido de migración de spec/contexto desde su interfaz; el registro expone `id`, `title` y `status`, pero no la relación completa entre plan, roles, repositorio e implementación. La ejecución genérica de plugins declarativos todavía no está conectada a un endpoint de comandos/scripts; Azure DevOps sí tiene rutas especializadas. La retirada de la TUI depende de completar y probar esa paridad. | `Axiom/apps/cli/src/index.ts`, `Axiom/apps/cli/src/commands/app.ts`, `app-api.ts`, `app-launcher.ts`, `app-onboarding.ts`, `app-plugins.ts`, `app-plugins-azure-devops.ts`, `Axiom/apps/cli/static/launcher/`, `Axiom/packages/launcher/src/`, `Axiom/packages/tui/src/`, `Axiom/packages/tui/tests/`, `Axiom/apps/cli/tests/app-launcher.test.ts`, `Axiom/apps/cli/tests/tui.test.ts`, `Axiom.Spec/specs/05_Interfaces_Operativas.md`, `Axiom.Spec/specs/06_Integraciones_y_Capacidades.md`. |

| 2026-08-04 | R-13/R-14 -> ACC-004..ACC-008 | Se ejecutó el lote solicitado mediante cinco incrementos SDD secuenciados: ACC-004 normalizó `@axiom/cli-commands`; ACC-006 completó onboarding/adopción del launcher; ACC-007 completó el ciclo SDD; ACC-008 integró plugins allowlisted y seguros; ACC-005 retiró la TUI solo después de verificar la paridad. Se integraron los hechos estables en `specs/00..08` y `context/**`, con review independiente, freeze/receipts y archive completados. | R-04 quedó como siguiente área; se mantienen como riesgos residuales la cobertura DOM del launcher y el redactor de plugins basado en patrones. | `Axiom.Spec/specs/increments/INC-20260804-*`, `Axiom.Spec/specs/00..08`, `Axiom.Spec/context/**`, `Axiom/` build/doctor/readiness y suites focalizadas. |
| 2026-08-05 | R-04 | Se verificó la ruta real de configuración: `profiles.yaml` y `DEFAULT_PROFILES` coinciden en functional profiles, overlays, targets, bindings y aliases; `loadCapabilityModel` carga 19 capabilities, 6 providers y sus fallbacks desde `axiom.config`; la batería focalizada de validación/composición pasa 4 archivos y 70 tests. | Quedan como hallazgos de auditoría, sin acción de producto registrada: `config-validation` rechaza el kind vigente `structural-code-intel` de `cmm`; `installProfile` cae silenciosamente a `DEFAULT_PROFILES` ante un `profiles.yaml` malformado; la taxonomía/type de capability aún dice 4 dominios mientras el runtime incluye `axiom.*`; `CC-004` solo comprueba el conjunto ya declarado por providers y no detecta capabilities canónicas sin provider. | `Axiom/axiom.config/{profiles,capabilities,providers}.yaml`, `packages/config-validation`, `packages/capability-model`, `packages/install-profiles`, `packages/installer`, `packages/doctor`, `Axiom.Spec/specs/03_Modelo_Operativo_y_Datos.md`. |
| 2026-08-05 | R-02 (cierre) | Se corrigió el comentario obsoleto de `@axiom/cli-commands`: ahora documenta que el barrel reexporta tanto `runX` como los wrappers `registerX`, y que `apps/cli/src/index.ts` usa estos últimos para registrar Commander. R-02 queda cerrado sin cambios de lógica. | Ninguna. | `Axiom/packages/cli-commands/src/index.ts`, `Axiom/apps/cli/src/index.ts`, `Axiom/packages/cli-commands/tsconfig.json`, `Axiom/packages/cli-commands/package.json`. |

## Registro de acciones

Cada entrada representa una petición explícita del usuario. Una propuesta no autoriza cambios en el código ni en la estructura del repositorio.

| ID | Área | Tipo | Estado | Propuesta | Evidencia o motivo | Dependencias |
| --- | --- | --- | --- | --- | --- | --- |
| ACC-001 | R-00 | modificar | validado | Crear `Axiom.Spec/decisions/` y mover ahí los ADR que estaban almacenados en `Axiom/docs/` (`0015-cavekit-discipline-post-mvp.md`, `0019-operator-control-plane-runtime.md`, `0026-integration-hardening-and-target-parity.md`, `0027-toolchain-provider-expansion-and-repair.md`, `0028-workflow-ux-and-archive-safety-completion.md`, `0029-memory-recall-and-context-repair.md`, `0030-operator-app-plugins-and-external-bridge.md`, `0031-adr-cmm-replaces-graphify-and-codegraph.md`). | `Axiom.Spec/decisions/` contiene los ocho ADR byte-a-byte; los ocho orígenes fueron eliminados después de verificar igualdad y SHA-256; las referencias activas usan el destino canónico. La secuencia final de freeze y apply quedó probada con receipts válidos. | Resuelta: ACC-002 se ejecutó después de poblar `decisions/`; el barrido posterior confirmó que las rutas antiguas no aparecen como referencias activas. |
| ACC-002 | R-00 | documentar | validado | Actualizar `AGENTS.md` raíz del workspace, `Axiom.SDD/AGENTS.md` y `Axiom.Spec/README.md` para que reflejen la estructura real de `Axiom.Spec` (incluir `decisions/` una vez creada en ACC-001, y las carpetas reales `bugs/`, `increments/`, `plans/`, `templates/`, `technical-context/` que hoy no se mencionan en el README). | Los tres documentos describen ahora `specs/`, `context/`, `bugs/`, `increments/`, `technical-context/`, `plans/`, `templates/`, `prompts/` y `decisions/`, distinguiendo raíces top-level existentes de las specs canónicas bajo `specs/`. | Resuelta después de ACC-001; la estructura documentada y la ruta canónica numerada fueron revalidadas. |
| ACC-003 | R-00 | investigar | validado | Analizar el contenido y propósito real de `Axiom/axiom.spec/` (contiene hoy `increments/`, `plans/`, `target-axiom-agents/`, `target-axiom-skills/`, `templates/`) para decidir si su contenido es útil (y en tal caso moverlo/reintegrarlo donde corresponda o eliminarlo si no aporta valor) o si es infraestructura legítima de runtime/instalación de Axiom que debe renombrarse para no confundirse con el repo sibling `Axiom.Spec/`. | ADR-0032 fija `Axiom.Spec/` como repo canónico y `Axiom/axiom.spec/` como baseline product-owned materializable; inventario, consumidores activos y `specRepo.ref: ../Axiom.Spec` verifican que debe conservarse en su ubicación actual. | Resuelta: la investigación y la revalidación de consumidores confirman conservar la ubicación actual y distinguirla de `Axiom.Spec/`. |
| ACC-004 | R-01 | modificar | validado | Normalizar la compilación de `@axiom/cli-commands` para que su API pública se emita en la ruta convencional `packages/cli-commands/dist/index.js` y `dist/index.d.ts`, eliminando la dependencia de `rootDir: "../.."` y de salidas bajo `dist/apps/` o `dist/packages/`. Para conseguirlo habrá que reorganizar o separar bajo la raíz del paquete la implementación compartida que hoy se selecciona desde `apps/cli/src/commands/`, actualizar `main`, `types`, referencias, aliases, consumidores (`apps/cli`, `@axiom/tui`, `@axiom/mcp-tools`), pruebas y documentación. Como trabajo relacionado, revisar y normalizar la salida de `@axiom/tui` si se mantiene el mismo criterio de empaquetado. | Validado en `INC-20260804-cli-commands-package-output`: 14 módulos trasladados al owner físico, entrypoints convencionales, build/runtime y suite de consumidores verdes; la salida histórica TUI quedó reconciliada antes de ACC-005. | Dependencia completada antes de ACC-006/007/008 y ACC-005; build, runtime, readiness, tests y review independiente documentados en el README y receipt `verify`. |
| ACC-005 | R-14 | eliminar | validado | Retirar la TUI como superficie operativa pública: quitar el registro de `axiom tui`, eliminar `@axiom/tui` y su driver, pantallas, flujos, pruebas y documentación dedicada cuando la paridad del launcher esté demostrada, y limpiar referencias, aliases, dependencias, project references y textos históricos. Mantener los comandos CLI/Core y los `run*` compartidos que sigan siendo usados por la CLI o el launcher; retirar solo la interfaz TUI y no la lógica de negocio subyacente. | Validado en `INC-20260804-tui-retirement`: paridad completa sin capacidad exclusiva, registro/dependencia retirados, dist stale limpiado, regresiones directas de `axiom tui` y ausencia de subcomando, build/doctor/readiness y 23 archivos/221 tests verdes. | Depende de ACC-006 y ACC-007; ejecutado al final del lote, después de ACC-004 y con ACC-008 integrado, preservando runners compartidos y marcando la historia TUI como histórica. |
| ACC-006 | R-13 | modificar | validado | Convertir el launcher servido por `axiom app` y distribuido con la CLI en la única interfaz guiada de Axiom. Debe leer el registry, los ficheros de configuración, specs, planes y estado real, y lanzar los comandos canónicos de Axiom con preview, confirmación y resultado visible, sin duplicar la lógica de negocio. La superficie debe cubrir instalación en proyecto nuevo, unión a proyecto existente, selección de proyecto, configuración, workspace setup, adopción/migración de spec y contexto técnico, y creación desde cero. | Validado en `INC-20260804-launcher-onboarding-migration`: setup/adopt, preview/dry-run, ownership/overlap guards, partial success, provenance, registry/doctor refresh e idempotencia sobre primitives canónicos; 7 archivos/90 tests focalizados verdes. | Depende de ACC-004 y de `workspace setup/adopt`, `bootstrap from-legacy-sdd`, `bootstrap from-context`, `user-workspace` y `project-resolution`; habilita ACC-007 y precede ACC-005. |
| ACC-007 | R-10 | modificar | validado | Completar la superficie del launcher para el ciclo SDD: crear y listar incrementos y bugs abiertos, crear y aprobar planes, iniciar y completar implementaciones por rol, ejecutar validaciones, archivar artefactos y mostrar relaciones entre spec, plan, roles, repositorios, implementación y estado. Cada operación debe leer el estado de ficheros existente y delegar en los comandos/workflows canónicos con preview y confirmación cuando haya mutación. | Validado en `INC-20260804-launcher-sdd-lifecycle`: registry aditivo con provenance, acciones canónicas, gates preview/confirm, worktree async, validate/archive y rollback con `inconsistent`; 12 archivos/104 tests focalizados y E2E launcher verdes. | Depende de ACC-006 y del workflow real; precedió ACC-008 y se integró antes de ACC-005. `increment-change` y `plan-archive` permanecen honestamente no ejecutables donde no existe wrapper canónico. |
| ACC-008 | R-12 | modificar | validado | Integrar plugins externos en el launcher mediante un contrato declarativo y seguro: descubrirlos, mostrar sus acciones, distinguir lectura de mutación local/externa, generar preview, exigir confirmación y delegar en comandos o scripts permitidos. Azure DevOps debe usar las rutas existentes de `tracker`/`tracker-ado` y `external-sync`, conservando credenciales fuera de la UI y mostrando resultados, errores y enlaces externos. No se debe habilitar ejecución arbitraria de shell por una entrada de plugin. | Validado en `INC-20260804-launcher-safe-plugins`: schema/discovery tolerante, DTO explícita, handler allowlisted, validación de campos, redacción de secretos/URLs, NullTracker local-only y gates de confirmación; 20 archivos/178 tests y E2E verdes. | Depende de ACC-006 y ACC-007; ejecutado después de ambos y antes de ACC-005, con `tracker`/`tracker-ado`/`external-sync` existentes y sin shell arbitrario. |

| ACC-014 | R-02 | documentar | validado | Corregir el comentario antiguo de `@axiom/cli-commands` para que describa la exportación actual de `runX` y `registerX`. | El barrel exporta wrappers `registerX` para que `apps/cli/src/index.ts` registre los comandos de Commander; la corrección elimina la contradicción documental sin cambiar comportamiento. | Ninguna. |

Tipos permitidos: `eliminar`, `modificar`, `agregar`, `investigar`, `documentar` y `riesgo`. Estados permitidos: `propuesto`, `validado`, `descartado` y `pospuesto`.

## Ejecución y cierre del lote ACC-004..ACC-008

El lote se ejecutó en este orden de dependencia: `ACC-004` -> `ACC-006` ->
`ACC-007` -> `ACC-008` -> `ACC-005`. La TUI no se retiró hasta completar y
validar la paridad del launcher y las superficies CLI/MCP. Las cinco acciones
quedan `validado`; sus incrementos están `closed`, con metadata `archived`,
integración canónica `integrated`, review independiente resuelta y carpetas
físicamente archivadas en `specs/increments/_archive/`.

Evidencia consolidada del lote:

- `npm run build` pasa; `npm run doctor` devuelve PASS (46/61 OK, 0 fallos,
   3 warnings y 12 omitidos); `npm run readiness:first-project` pasa.
- ACC-004: consumidores focalizados 31 archivos/402 tests y CLI completa
   133 archivos/1268 tests; entrypoint `dist/index.js` y declaración
   `dist/index.d.ts` resolubles.
- ACC-006: batería final 7 archivos/90 tests; preview/dry-run, ownership,
   solapamiento, adopción parcial, provenance e idempotencia cubiertos.
- ACC-007: batería focalizada 12 archivos/104 tests, más E2E de launcher y
   regresiones de worktree/archive; rollback inconsistente señalizado.
- ACC-008: batería focalizada 20 archivos/178 tests y E2E; DTO explícitas,
   allowlist de handlers, validación estricta, redacción y `NullTracker`
   local-only verificados.
- ACC-005: batería final dirigida 23 archivos/221 tests, regresiones directas
   de `axiom tui` y de la invocación sin subcomando, dist stale limpiado y
   claims activos de TUI reconciliados como launcher/CLI/MCP o históricos.

La integración canónica única se aplicó en `specs/00..08` y `context/**`.
Se preservaron los registros históricos y no se ejecutaron mutaciones Git.
La revisión de auditoría se reanudó en **R-04: Configuración y modelo de
capacidades**; la siguiente área es **R-05: Estado, filesystem y aislamiento**.

## Cierre de la auditoría

Tras R-16, agrupar las acciones `propuesto` y `validado` por dependencia y riesgo. Cada grupo que se vaya a ejecutar se convierte en un incremento o bug en `Axiom.Spec`; este plan conserva el razonamiento de la auditoría, no sustituye la especificación de implementación.