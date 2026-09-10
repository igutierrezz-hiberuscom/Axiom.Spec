# Plan del estado persistente y contrato CLI de self-update

> **Código**: PLAN-INC-20260909-r13-self-update-state-cli-contract
> **Estado**: draft
> **Artefacto origen**: INC-20260909-r13-self-update-state-cli-contract
> **Versión de spec**: v1
> **Versión de plan**: p1

## Resumen ejecutivo

Este plan prepara la implementación de ACC-079 en el runtime Axiom. Entregará el contrato `install.json` v2, compatibilidad/migración segura desde v1, primitives de lectura y escritura durable y la nueva gramática CLI exclusiva. Es el **segundo incremento de una secuencia de cinco** y no puede comenzar sin la identidad de release entregada por `INC-20260909-r13-self-update-release-identity`.

El plan es prospectivo: no presupone que los archivos, símbolos, tests o comportamientos descritos ya existan. Las rutas propuestas son candidatas que el builder deberá confirmar contra el monorepo antes de editar. No se ejecutará una transición de lifecycle desde este plan.

## Secuencia global R13

1. **Release identity** — `INC-20260909-r13-self-update-release-identity`: define la identidad canónica y es dependencia obligatoria.
2. **Estado y contrato CLI — este incremento**: añade receipt v2, migración, atomic writer y operaciones CLI estables sin motor mutante.
3. **Motor real de apply/recovery**: consumirá reader/migrator/writer y reemplazará el fallo cerrado por mutación real verificada.
4. **Integración de Launcher**: consumirá outcomes/envelopes ya estabilizados, sin redefinirlos.
5. **Release CI y certificación**: automatizará publicación/canales sobre los contratos anteriores.

Este incremento habilita 3–5 mediante contratos, pero no implementa ninguna de esas capacidades.

## Objetivo técnico

Construir una frontera de estado y CLI que sea segura ante ausencia, corrupción, procesos concurrentes y crashes, y que pueda ser consumida por el motor futuro sin cambios incompatibles. La implementación deberá:

- resolver identidad real con el contrato del incremento 1;
- tratar `install.json` como receipt/cache no autoritativa;
- leer v1 y v2, escribir solo v2 y migrar v1 con evidencia real;
- reutilizar el lock user-level y capacidades atómicas de `@axiom/core`;
- evitar lost updates mediante revisión y fingerprint esperados;
- separar check, attempt, success y recovery;
- exponer `status/check/plan/apply/recover` como operaciones exclusivas;
- demostrar que `status/check/plan` y todas las rutas `--dry-run` permitidas no mutan;
- emitir envelope JSON v1 y exit codes uniformes;
- mantener `apply/recover` en fallo cerrado `engine_unavailable` hasta el incremento 3.

## Alcance incluido

- Discovery del reader/writer/lock/CLI actuales y fixtures reales v1.
- Tipos, validators y tests del schema cerrado `InstallStateV2`.
- Resultado discriminado de lectura, sin fallback `0.0.0`.
- Reader v1/v2 y migrador v1→v2 idempotente basado en instalación real.
- Writer atómico, protocolo de recuperación y protección compare-and-swap lógica.
- Router/parser CLI de operación única y opciones por operación.
- Handlers read-only de `status`, `check` y `plan`.
- Handlers fail-closed de `apply` y `recover`.
- Renderer humano y envelope JSON v1 compartiendo un resultado de dominio.
- Taxonomía de outcomes, errores y exit codes; uso de `process.exitCode`.
- Pruebas unitarias, integración, concurrencia, fault injection, no-mutación y wrapper compilado.
- Review independiente, evidencia y handoff al incremento 3.

## Alcance excluido

- Fetch/checkout/reset/rebase o sustitución Git real.
- Descarga/instalación real de releases y rollback de producto.
- Mutación funcional desde `apply` o `recover` antes del incremento 3.
- Launcher UI, IPC o lógica visual.
- Release CI, publicación, firma o promoción de canales.
- Un nuevo framework de locks, journal o filesystem paralelo a `@axiom/core`.
- Telemetría, scheduler, daemon o actualización automática.
- Cambios manuales de metadata/status/índices de lifecycle.

## Roles impactados

- **builder**: implementa cambios focalizados en el runtime y tests; detalle en [role-builder.md](./role-builder.md).
- **reviewer**: revisión independiente de autoridad, atomicidad, no-mutación, envelopes y exclusiones.
- **owner de release identity**: confirma que el contrato del incremento 1 está estable y consumible.
- **owner del incremento 3**: valida el handoff de APIs sin pedir mutación anticipada.

No se necesita un rol de Launcher ni de release CI en este plan.

## Plan de implementación detallado

### Fase 0 — Preflight, baseline y dependencia

1. Confirmar que el incremento 1 tiene contrato cerrado, implementación disponible y pruebas verdes.
2. Localizar la fuente actual de identidad instalada y verificar que no dependa de `install.json`.
3. Localizar todos los readers/writers de `install.json`, su path resolver y las variantes v1 observadas.
4. Localizar el lock user-level, atomic write/replace, fsync y recovery existentes en `@axiom/core`.
5. Localizar command registry, parser, renderer JSON/humano, entrypoint y wrapper compilado de CLI.
6. Capturar baseline de build y suites antes de editar; clasificar fallos preexistentes.

**Gate G0 — Dependency/primitive readiness**

- **GO**: release identity es importable y estable; existen primitives Core reutilizables o una extensión focalizada compatible; hay fixtures suficientes para describir v1; wrapper compilado identificable.
- **STOP**: identidad sigue implícita/derivada del receipt, no hay forma segura de reconciliar instalación real, el primitive Core no puede ofrecer exclusión/durabilidad requerida, o v1 real es desconocido. No se inventará un contrato alternativo.

### Fase 1 — Contratos ejecutables y fixtures

1. Definir tipos/uniones para `InstallStateV2`, receipts, resultados de lectura, envelope v1, outcomes y errores.
2. Implementar schema cerrado con `additionalProperties: false` en todos los niveles.
3. Reutilizar los validators SemVer/ref/provenance del incremento 1; añadir validators ISO UTC y path absoluto donde corresponda.
4. Crear fixtures mínimos: v1 soportados, v2 válido, campos extra, SemVer/ISO/path/ref/provenance inválidos, corruptos y schema futuro.
5. Escribir primero tests de parse/serialize y tabla outcome→exit.

**Gate G1 — Contract lock**

- **GO**: owner/reviewer aprueban schema, ausencia distinta de `0.0.0`, receipts separados, envelope y taxonomía; todos los fixtures tienen una clasificación única.
- **STOP**: el schema duplica o contradice release identity, permite campos abiertos/defaults o no distingue attempt de success/recovery.

### Fase 2 — Reader y resolución read-only

1. Implementar lectura por bytes y fingerprint sin crear directorios ni archivos.
2. Distinguir ENOENT de permisos/I/O; parsear v1/v2 por discriminator explícito.
3. Resolver la identidad real independientemente del receipt y producir una unión discriminada de reconciliación.
4. Asegurar que v1 válido devuelve `migration_required`, no una migración automática.
5. Añadir tests de snapshots filesystem para ausencia, v1, v2, corrupción, mismatch e I/O.

**Gate G2A — Reader purity**

- **GO**: todos los casos de lectura conservan bytes, mtimes y árbol; ausencia nunca devuelve `0.0.0`.
- **STOP**: el reader crea carpetas, normaliza/escribe documentos o toma un lock mutante.

### Fase 3 — Atomic writer y recuperación

1. Adaptar, no duplicar, el lock user-level de `@axiom/core`; derivar clave con target canónico.
2. Implementar precondición `expectedRevision + expectedFingerprint` verificada bajo lock.
3. Serializar v2 determinísticamente y validar antes de staging.
4. Crear temporales hermanos exclusivos y únicos; nunca un `.tmp` fijo.
5. Aplicar write completo, flush/fsync de archivo, close, revalidación, replace atómico y fsync de directorio soportado.
6. Implementar cleanup/recovery usando la evidencia del primitive Core; no promover por mtime.
7. Liberar recursos en `finally`; normalizar errores de lock/I/O/durabilidad.
8. Probar dos o más procesos reales, no solo promesas en un proceso.
9. Inyectar crashes/fallos en cada frontera y comprobar que target previo o estado recuperable permanecen inequívocos.

**Gate G2B — Storage safety**

- **GO**: no hay lost updates, temporales colisionantes, targets parciales ni falsos commits; la matriz de fault injection pasa en plataformas soportadas.
- **STOP**: una plataforma requiere delete-then-rename, un stale writer pisa cambios, un temporal puede confundirse con commit o la recuperación depende de heurística. Escalar al owner de Core antes de seguir.

### Fase 4 — Migración v1→v2

1. Separar decode v1, reconciliación real y commit v2 en funciones comprobables.
2. Calcular fingerprint de bytes v1 y provenance `legacy-migration`.
3. Validar identidad/path contra la instalación real; no copiar una versión v1 como autoridad.
4. Construir y validar v2; usar el writer de Fase 3 con expected fingerprint.
5. Releer el commit antes de devolver éxito.
6. Probar idempotencia, v1 cambiado concurrentemente, mismatch, identidad no disponible, permisos y crash.
7. Exponer el migrador solo a una frontera mutante explícita. No conectarlo a `status/check/plan`; mientras no exista el motor 3, la CLI solo informa `migration_required`.

**Gate G3 — Migration fidelity**

- **GO**: cada v1 soportado migra desde identidad real, cada caso dudoso conserva bytes fuente y reintentos no reescriben.
- **STOP**: se necesita fabricar versión/provenance, se pierde evidencia v1 o una lectura dispara migración.

### Fase 5 — Gramática, handlers y renderers CLI

1. Sustituir selección por flags por subcomando obligatorio y exclusivo.
2. Definir opciones por operación y rechazar combinaciones/legacy flags antes de side effects.
3. Implementar `status` local read-only, `check` con red pero sin persistencia y `plan` como cálculo puro/read-only.
4. Aceptar `--dry-run` solo con `plan`; hacer que cualquier combinación mutante falle en parse/validation antes del dispatcher.
5. Implementar `apply` y `recover` como capacidades no disponibles: envelope/error `engine_unavailable`, exit `69`, cero lock/escritura/receipt.
6. Centralizar resultado de dominio y renderers. En JSON: un objeto por stdout; texto humano/errores por stderr.
7. Devolver al entrypoint y asignar `process.exitCode`; eliminar `process.exit()` de esta ruta.
8. Probar que logs de dependencias no contaminan stdout JSON.

**Gate G4 — CLI contract**

- **GO**: matriz completa operation/options/outcome/streams/exit pasa y snapshots prueban no-mutación.
- **STOP**: hay operación implícita, precedencia entre flags, dry-run alcanza un adapter mutante, apply/recover simulan éxito o JSON mezcla texto.

### Fase 6 — Integración y wrapper compilado

1. Compilar el workspace.
2. Ejecutar el bin/wrapper distribuible contra sandbox temporal.
3. Cubrir al menos `state_ready`, `state_absent`, `migration_required`, `update_available`, `plan_ready`, uso inválido, corrupción, fallo remoto y `engine_unavailable`.
4. Capturar stdout/stderr como bytes y exit code; parsear stdout JSON y comprobar un único valor/newline.
5. Repetir pruebas no-mutación contra el proceso compilado.
6. Ejecutar suites focalizadas y suite completa.

**Gate G5 — Release-facing behavior**

- **GO**: build y pruebas nuevas pasan; wrapper preserva contratos; cualquier fallo preexistente está documentado con baseline reproducible.
- **STOP**: falla una prueba introducida, el wrapper transforma códigos/streams o solo funciona mediante imports de test.

### Fase 7 — Review, rollback readiness e integración canónica

1. Obtener review independiente contra todos los AC-079, con foco en autoridad, durabilidad y fail-closed.
2. Revisar diff para excluir Git update, Launcher, release CI y cambios no relacionados.
3. Preparar rollback técnico y comprobar que no requiere down-migration destructiva.
4. Documentar evidencia de validación y handoff al incremento 3.
5. Solo después de GO, consolidar conocimiento estable en las specs canónicas propietarias del modelo operativo y CLI; no copiar el historial.
6. Las transiciones/status se gestionarán por Axiom/Core fuera de este trabajo documental.

**Gate G6 — Final GO**

- **GO**: criterios cubiertos, review sin bloqueantes, rollback seguro, integración canónica identificada y owner acepta el contrato.
- **STOP**: queda un criterio sin evidencia, se requiere una excepción de seguridad/durabilidad o el incremento 3 necesitaría romper schema/grammar/envelope.

## Archivos y símbolos probables

Las rutas son hipótesis de trabajo y deberán confirmarse en Fase 0. Se preferirá extender el ownership existente en vez de crear duplicados.

| Área | Rutas candidatas | Símbolos/conceptos probables |
|---|---|---|
| Modelo/schema | `packages/core/src/self-update/install-state*.ts` o módulo existente equivalente | `InstallStateV2`, `InstallStateReadResult`, `parseInstallState`, `serializeInstallState` |
| Release identity | módulo entregado por incremento 1 | `ReleaseIdentityV1`, `resolveInstalledReleaseIdentity`, validators SemVer/ref/provenance |
| Reader | `packages/core/src/self-update/install-state-reader.ts` o equivalente | `readInstallState`, `reconcileInstallState` |
| Writer | `packages/core/src/self-update/install-state-writer.ts` o equivalente | `writeInstallStateAtomic`, `recoverInstallStateWrite`, `StateConflictError` |
| Migración | `packages/core/src/self-update/install-state-migration.ts` o equivalente | `decodeInstallStateV1`, `migrateInstallStateV1ToV2` |
| Lock/fs | módulos existentes bajo `packages/core/src/**` | primitive user-level lock, atomic replace, fsync helpers; **reutilizar, no clonar** |
| CLI command | `apps/cli/src/commands/self-update*.ts` o registro existente | parser discriminado, `runSelfUpdateStatus/Check/Plan/Apply/Recover` |
| Renderer/entrypoint | `apps/cli/src/**` y wrapper/bin declarado por package | `SelfUpdateEnvelopeV1`, renderer JSON/humano, mapper outcome→exit, `process.exitCode` |
| Tests | tests co-localizados o árbol de test existente | schema/migration/concurrency/fault/no-mutation/CLI/wrapper suites |

Si el repositorio usa otros paquetes o nombres, el builder actualizará el mapa de contexto; no moverá archivos solo para hacer coincidir estas hipótesis.

## Estrategia de migración y compatibilidad

- **Reader**: v1 y v2; schemas desconocidos fallan explícitamente.
- **Writer**: solo v2.
- **Trigger**: API mutante explícita del instalador/motor; nunca startup o comandos read-only.
- **Fuente**: identidad real del incremento 1; v1 aporta bytes/fingerprint y evidencia legacy.
- **Commit**: lock Core + expected fingerprint/revision + writer durable.
- **Fallo**: v1 intacto y error tipado; no best-effort destructivo.
- **Reintento**: idempotente; si otro proceso ganó, releer y clasificar.
- **CLI legacy**: flags selectores devuelven uso inválido con hint; no hay precedencia implícita.
- **Dry-run**: `plan` es la operación canónica; `--dry-run` fuera de `plan` se rechaza antes de mutación.
- **Forward compatibility**: envelope v1 y schema v2 evolucionan solo mediante versionado explícito, nunca aceptando campos desconocidos.

## Estrategia del atomic writer

La implementación deberá demostrar estas invariantes:

1. un solo writer por usuario/instalación;
2. cada staging file tiene ownership inequívoco;
3. el target previo sigue válido hasta el replace;
4. los bytes staged se validan y fsync antes del replace;
5. la precondición se comprueba bajo lock;
6. un commit cambia revision exactamente una vez;
7. el éxito solo se informa tras releer el target comprometido;
8. un temporal no es éxito;
9. recuperación/cleanup se hace bajo lock y con evidencia durable;
10. las limitaciones de plataforma producen STOP o error explícito, no garantías ficticias.

## Taxonomía de errores a implementar

| Clase | Error/outcome | Exit | Mutación permitida en incremento 2 |
|---|---|---:|---:|
| Uso | `cli_usage_error`, `conflicting_operation`, `invalid_option`, `dry_run_forbidden` | 64 | No |
| Datos | `state_corrupt`, `schema_unsupported`, `identity_mismatch`, `migration_rejected` | 65 | No tras fallo |
| Capacidad | `engine_unavailable` | 69 | No |
| Concurrencia | `lock_unavailable`, `state_conflict` | 73 | No commit perdedor |
| I/O/durabilidad | `state_io_error`, `atomic_write_failed`, `recovery_required` | 74 | Solo cleanup seguro bajo lock |
| Externo temporal | `release_check_unavailable` | 75 | No |
| Config/provenance | `release_identity_invalid`, `provenance_invalid` | 78 | No |

Resultados no-error: exit `0` (`state_ready`, `up_to_date`, `plan_empty`), `10` (`update_available`, `plan_ready`) y `11` (`state_absent`, `migration_required`).

## Matriz de pruebas

| Familia | Casos mínimos | Evidencia |
|---|---|---|
| Schema | válido, extra props en cada nivel, SemVer/ISO/path/ref/provenance inválidos | tests de tabla/property cuando aplique |
| Reader | ausente, v1, v2, corrupto, futuro, permisos, mismatch | unión exacta + snapshot sin mutación |
| Migración | variantes v1 reales, mismatch, identidad ausente, idempotencia, conflicto | bytes antes/después + provenance/fingerprint |
| Writer | serialización, revision, temp único, fsync/replace, cleanup | fault injection y relectura |
| Concurrencia | dos procesos, stale revision, aliases de path, instalaciones distintas | winner único; perdedor `state_conflict` |
| CLI grammar | cada op, ninguna, múltiples, legacy flags, opciones cruzadas, dry-run | outcome/error/exit + adapter spies |
| Read-only | status/check/plan en todos los outcomes | árbol/bytes/mtime/locks idénticos |
| Fail-closed | apply/recover con/sin JSON y opciones válidas | exit 69, cero receipts/locks/writes |
| Envelope | éxitos, accionables y errores | JSON único, schema cerrado, stderr separado |
| Wrapper | matriz representativa sobre build | stdout/stderr raw + exit code |
| Regresión | build, focalizadas, suite completa | logs y clasificación de fallos |

## Estrategia E2E

El E2E usará un home y una instalación temporales aislados, provider fake determinista y el wrapper compilado. Antes de cada comando se capturará un inventario con hashes, mtimes y nombres; después se comparará. Las pruebas mutantes del writer/migrador se harán mediante API/harness explícito, no fingiendo `apply/recover`. Los procesos concurrentes serán procesos del sistema independientes para ejercitar el lock user-level real.

Validaciones previstas desde el runtime Axiom:

```text
npm run build
npx vitest run
```

Además, G5 ejecutará las suites focalizadas identificadas en Fase 0 y el bin exacto declarado por el package compilado para cada operación `self-update`, incluida una invocación `status --json` y otra `apply --json`. La evidencia registrará los comandos concretos descubiertos; no se adivinará una ruta de wrapper.

## Riesgos y dependencias

| Riesgo/dependencia | Impacto | Mitigación/gate |
|---|---|---|
| Release identity incompleta | Estado vuelve a ser autoridad implícita | G0 STOP |
| v1 tiene variantes desconocidas | Migración destructiva o incompleta | inventario/fixtures; rechazar lo desconocido |
| Replace/fsync difiere por OS | Falso durability guarantee | adapter Core + fault tests; G2B STOP |
| Lock keyed por path no canónico | carreras o bloqueo excesivo | canonicalización + tests aliases/distintos targets |
| Check persiste cache por dependencia | viola read-only | inyectar provider sin cache o desactivar persistencia; snapshots |
| Logs contaminan JSON | rompe automatización | renderer único + captura raw wrapper |
| Exit codes truncados/transformados | wrappers inconsistentes | E2E compilado |
| Compatibilidad legacy ambigua | operaciones inesperadas | error tipado, no precedence |
| Incremento 3 fuerza cambios de contrato | retrabajo | handoff y review del owner antes de G6 |

## Rollback

- Antes de publicación, el rollback de código será revertir únicamente el cambio focalizado tras conservar fixtures/evidencia para diagnóstico.
- Nunca se hará down-migration automática v2→v1.
- Si una migración falla antes del commit, los bytes v1 deben permanecer como target válido.
- Si el commit v2 fue durable, un rollback de runtime no podrá borrar ni reinterpretar el receipt; se deberá desplegar un reader compatible o detener el rollback.
- Ante un defecto CLI, se podrá retirar el wiring nuevo antes de release, pero no reactivar flags ambiguos que alcancen mutación.
- Ante un defecto del writer/lock, se deshabilitarán fronteras mutantes y se conservarán `status/check/plan` solo si se demuestra que siguen read-only y seguras.
- Cualquier artefacto de recuperación dudoso queda preservado y reportado; no se limpia automáticamente para «volver a verde».

## Cambios en contexto técnico

Durante la implementación se documentarán, dentro del artefacto autorizado o mediante operaciones Axiom apropiadas:

- paths/símbolos reales y ownership de paquetes;
- variantes v1 realmente soportadas;
- garantías y limitaciones de atomic replace/fsync por plataforma;
- clave y lifecycle del lock Core;
- comando exacto del wrapper compilado;
- evidencia de no-mutación y concurrencia.

No se copiará prosa canónica ni se crearán índices manuales.

## Consolidación y archivado

Tras G6, el owner decidirá la integración estable en las specs canónicas del modelo operativo/datos y de interfaces operativas. La integración resumirá únicamente: autoridad real frente a receipt, schema/versionado, operaciones CLI, outcomes/exit y garantías de atomicidad. No se archivará ni cambiará status desde este plan; esas acciones corresponden a Axiom/Core y quedan fuera de la ejecución solicitada.

## Validaciones y gates

Resumen de gates:

| Gate | GO | STOP |
|---|---|---|
| G0 dependencia | identity/Core/v1/wrapper identificados | autoridad o primitive inseguros/desconocidos |
| G1 contrato | schemas/outcomes aprobados y fixtures unívocos | defaults, shapes abiertos o duplicación |
| G2A reader | pureza y estados separados probados | cualquier escritura/default engañoso |
| G2B storage | concurrencia/fault matrix verde | parcial, lost update o recovery heurística |
| G3 migración | real, idempotente, preserva v1 en fallo | identidad fabricada o auto-migración read-only |
| G4 CLI | grammar/streams/exits/no-mutación verdes | ambigüedad, contaminación o éxito fingido |
| G5 wrapper | build y E2E compilado verdes | divergencia del bin distribuible |
| G6 final | review, rollback, handoff e integración aceptados | AC sin evidencia o contrato inestable |

Un STOP no se convierte en GO reduciendo cobertura, omitiendo una plataforma soportada o relajando el schema.

## Fuentes y supuestos

- Fuente funcional: ACC-079 y el incremento origen.
- Dependencia: `INC-20260909-r13-self-update-release-identity`.
- Runtime propietario: monorepo TypeScript Axiom (`packages/*`, `apps/cli`).
- Validación conocida: `npm run build` y `npx vitest run`.
- Supuesto a confirmar: `@axiom/core` ya contiene primitives reutilizables de lock/atomic filesystem con garantías suficientes.
- Supuesto a confirmar: el contrato del incremento 1 permite inspeccionar la instalación real sin confiar en `install.json`.
- Restricción: no hay motor del incremento 3; por tanto apply/recover deben fallar cerrados.
- Restricción de este trabajo documental: no se modifica nada fuera de los dos directorios del incremento y plan.
