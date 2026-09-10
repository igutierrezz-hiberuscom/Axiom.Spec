# r13-self-update-state-cli-contract

> **Código**: INC-20260909-r13-self-update-state-cli-contract
> **Estado**: En Borrador
> **Fecha de creación**: 2026-09-09
> **Tipo de cambio**: Contrato de estado persistente y superficie CLI

## Resumen

Este incremento especifica ACC-079: el contrato de estado local y de línea de comandos sobre el que se apoyará el self-update de Axiom. Es el **segundo de cinco incrementos** de la secuencia R13 y depende obligatoriamente de `INC-20260909-r13-self-update-release-identity`.

El resultado esperado es un lector, migrador y escritor seguro para `install.json` v2, junto con una gramática CLI inequívoca para `status`, `check`, `plan`, `apply` y `recover`. El manifiesto se tratará siempre como cache/receipt derivado de la instalación real, nunca como autoridad de versión o procedencia.

Este documento describe estado deseado y criterios de aceptación. **No afirma que el runtime, la migración, el writer ni el contrato CLI estén ya implementados o validados.**

## Contexto y motivación

El incremento previo establece la identidad canónica de release. ACC-079 debe consumir ese contrato para evitar que un archivo local desactualizado, ausente o corrupto decida qué versión está realmente instalada. También debe eliminar selecciones ambiguas por flags, separar operaciones de observación de operaciones mutantes y dar a automatizaciones un envelope y códigos de salida estables.

Sin esta base, un motor de actualización podría perder escrituras concurrentes, confundir una comprobación con una instalación exitosa, fabricar `0.0.0` ante ausencia de estado o presentar como aplicada una operación que todavía no existe.

## Alcance

### Incluido

- Contrato cerrado de `install.json` v2, con validación estricta de SemVer, timestamps ISO-8601 UTC, paths absolutos normalizados, refs de release y provenance.
- Estados explícitos para archivo ausente, v1 migrable, v2 válido, corrupto, versión de schema no soportada e identidad real no reconciliable.
- Lectura compatible de v1 y migración idempotente v1→v2 basada en la instalación real y en la identidad de release del incremento 1; nunca basada únicamente en datos declarados por v1.
- Distinción entre observaciones/receipts de `check`, `attempt`, `success` y `recovery`.
- Escritura atómica con temporales únicos, flush/fsync, recuperación determinista, lock de ámbito de usuario reutilizado desde `@axiom/core` y protección contra lost updates.
- Gramática canónica `axiom self-update <status|check|plan|apply|recover>` con exactamente una operación.
- Garantía de no mutación para `status`, `check` y `plan`; `--dry-run` no podrá atravesar la frontera mutante.
- Envelope JSON v1 uniforme, stdout JSON limpio, diagnósticos humanos por stderr, asignación mediante `process.exitCode` y taxonomía estable de outcomes/errores/códigos de salida.
- Fallo cerrado y tipado de `apply` y `recover` mientras no exista el motor del incremento 3.
- Pruebas de schema, corrupción, migración, concurrencia, recuperación, no mutación, envelopes y wrapper compilado.

### Excluido

- Descarga, checkout, reemplazo o actualización Git real.
- Ejecución real de `apply` o `recover`; corresponde al incremento 3.
- Launcher UI o cualquier interfaz gráfica.
- Release CI, publicación de artefactos o certificación del canal de releases.
- Cambios al contrato canónico de identidad de release definido por el incremento 1.
- Convertir `install.json` en fuente de verdad, inventar una versión por defecto o persistir resultados de comandos declarados read-only.

## Documentos del incremento

- [01_Requisitos.md](./01_Requisitos.md): requisitos funcionales, no funcionales y reglas de negocio.
- [02_Cambios_Modelo.md](./02_Cambios_Modelo.md): modelo v2, migración, writer, estados, envelopes y errores.
- [03_Criterios_Aceptacion.md](./03_Criterios_Aceptacion.md): escenarios verificables y evidencia mínima.
- [04_Interacciones_UI.md](./04_Interacciones_UI.md): contrato de interacción de la CLI; no define Launcher UI.
- [context/README.md](./context/README.md): límites, fuentes y handoff técnico del contexto local.
- Plan asociado: `PLAN-INC-20260909-r13-self-update-state-cli-contract`.

## Dudas abiertas

No hay una duda funcional que autorice a relajar ACC-079. Antes de implementar deberán resolverse por inspección del runtime, sin cambiar el comportamiento aquí definido:

1. nombres y ubicaciones exactas de los primitives de lock y atomic write existentes en `@axiom/core`;
2. shape exacto importado de `ReleaseIdentityV1`, para referenciarlo y no duplicarlo;
3. variantes reales de `install.json` v1 presentes en fixtures o instalaciones soportadas;
4. garantías de fsync de directorio y replace atómico que ofrece cada plataforma soportada;
5. ubicación exacta del wrapper compilado que debe probarse end-to-end.

Si cualquiera de estos puntos invalida una garantía normativa, el gate correspondiente será **STOP** y se refinará el contrato antes de continuar.

## Decisiones funcionales cerradas

- `install.json` v2 es cache/receipt y no autoridad.
- Archivo ausente significa `state_absent`; no equivale a SemVer `0.0.0`.
- Un v1 solo se migra tras reconciliar path e identidad de la instalación real.
- Lectores pueden entender v1 durante la transición; todo writer nuevo emite únicamente v2.
- `status`, `check` y `plan` no escriben estado, locks, temporales, caches ni receipts.
- Solo puede seleccionarse una operación CLI; ausencia o combinación de operaciones es error de uso.
- `plan` es la operación canónica de simulación. `--dry-run` solo podrá aceptarse en una forma que termine antes del dispatcher mutante; nunca modificará estado.
- Hasta disponer del motor del incremento 3, `apply` y `recover` devuelven `engine_unavailable`, no `applied`, `recovered` ni éxito simulado.
- Las escrituras mutantes futuras usarán el lock user-level de `@axiom/core`, compare-and-swap lógico y reemplazo durable; no se implementará un lock paralelo.
- En modo JSON se emitirá exactamente un envelope v1 por stdout; el texto humano y los diagnósticos irán por stderr.
- El proceso se completará asignando `process.exitCode`; la ruta de comando no llamará a `process.exit()`.

## Consolidación en la spec general

La consolidación canónica queda pendiente hasta que la implementación y las pruebas satisfagan los criterios. En cierre, el conocimiento estable deberá integrarse de forma resumida en las especificaciones generales que posean el modelo operativo y la interfaz CLI, sin copiar el historial del plan. En esta preparación no se modifica ningún documento externo a este artefacto y su plan.

## Estrategia E2E

La validación deberá combinar:

1. pruebas unitarias y contractuales del schema v2 y del envelope v1;
2. fixtures v1/v2, archivos ausentes, truncados, corruptos y con campos extra;
3. migraciones sobre instalaciones temporales cuya identidad real sea verificable;
4. procesos concurrentes para lock, temporales únicos y rechazo de revisiones obsoletas;
5. inyección de fallos en write, fsync y replace para comprobar recuperación sin falsos éxitos;
6. snapshots del árbol antes/después de `status`, `check`, `plan` y casos `--dry-run` para demostrar no mutación;
7. ejecución del wrapper compilado, capturando stdout, stderr y exit code en éxitos, resultados accionables y errores;
8. `npm run build`, pruebas focalizadas y `npx vitest run` desde el repositorio runtime cuando se implemente.

## Trazabilidad y fuentes

- Criterio rector: ACC-079.
- Dependencia obligatoria: `INC-20260909-r13-self-update-release-identity`.
- Secuencia global R13: (1) release identity; **(2) este incremento de estado/CLI**; (3) motor real de apply/recovery; (4) integración de Launcher; (5) release CI/certificación.
- Contratos heredados: identidad de release y primitives compartidos de `@axiom/core`, pendientes de inspección durante la implementación.
- Restricción de secuencia: este incremento habilita los incrementos 3, 4 y 5, pero no implementa sus capacidades.

## Estado de validación humana

Pendiente. Requiere revisión explícita del owner sobre schema v2, compatibilidad v1, gramática CLI, taxonomía de errores/códigos y garantías de atomicidad antes del gate de implementación.