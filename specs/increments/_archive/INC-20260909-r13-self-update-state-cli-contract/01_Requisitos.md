# 01 Requisitos

## Objetivo del documento

Definir los requisitos normativos de ACC-079 para el estado persistente de self-update y su contrato CLI. Todos los puntos describen comportamiento que deberá implementarse y probarse; no constituyen evidencia de implementación actual.

## Requisitos del incremento

### Precondición y autoridad

- **REQ-079-01 — Dependencia obligatoria.** La implementación no comenzará hasta que `INC-20260909-r13-self-update-release-identity` exponga un contrato consumible y validado de identidad de release. Un cambio incompatible en esa dependencia obliga a detener este incremento.
- **REQ-079-02 — Autoridad real.** La versión, ref y provenance instaladas se resolverán desde la instalación real mediante el contrato de release identity. `install.json` solo será una cache/receipt y nunca podrá corregir, sustituir ni sobreescribir la identidad real.
- **REQ-079-03 — Ausencia explícita.** La inexistencia de `install.json` producirá el estado `state_absent`, separado de versión desconocida, corrupción y cualquier SemVer. Queda prohibido sintetizar `0.0.0`.

### `install.json` v2 y compatibilidad

- **REQ-079-04 — Schema cerrado.** v2 rechazará propiedades desconocidas en todos sus niveles y validará, como mínimo, `schemaVersion`, `revision`, identidad instalada, path, refs, timestamps, receipts y provenance.
- **REQ-079-05 — Tipos estrictos.** Las versiones usarán SemVer canónico sin prefijo `v`; los timestamps serán ISO-8601 UTC canónicos; los paths serán absolutos y normalizados para la plataforma; las refs y provenance cumplirán el contrato cerrado de release identity.
- **REQ-079-06 — Estados separados.** El modelo distinguirá `check`, `attempt`, `success` y `recovery`. Un intento no implica éxito; una recuperación no se presentará como instalación; un check no se convertirá en receipt de éxito.
- **REQ-079-07 — Reader compatible.** El reader reconocerá v1 soportado, v2 válido, schema no soportado, corrupción sintáctica y violación semántica como resultados distintos. No corregirá silenciosamente bytes inválidos.
- **REQ-079-08 — Writer v2-only.** Toda escritura nueva emitirá exclusivamente v2. El reader v1 es una compatibilidad transitoria, no autorización para seguir produciendo v1.
- **REQ-079-09 — Migración real.** La migración v1→v2 reconstruirá la identidad desde la instalación real, verificará que el path pertenece a esa instalación y conservará provenance explícita de migración. No confiará en la versión declarada por v1 como autoridad.
- **REQ-079-10 — Migración segura.** La migración será idempotente y usará el mismo lock, expected revision/fingerprint y writer atómico que las escrituras normales. Si no puede reconciliar identidad, path o provenance, fallará tipadamente y dejará v1 byte por byte intacto.
- **REQ-079-11 — Sin migración en lecturas.** `status`, `check` y `plan` podrán informar `migration_required`, pero no migrarán, repararán ni reescribirán estado. El migrador quedará disponible únicamente detrás de una frontera explícitamente mutante del instalador o del motor futuro.

### Escritura, lock y recuperación

- **REQ-079-12 — Reutilización de Core.** El lock será el primitive de ámbito de usuario ya provisto por `@axiom/core`. No se creará una implementación, namespace o semántica de locking paralela.
- **REQ-079-13 — Exclusión entre procesos.** La clave de lock combinará ámbito de usuario e identidad canónica de la instalación/target, de modo que procesos distintos que apunten al mismo estado se serialicen aunque sus cwd sean diferentes.
- **REQ-079-14 — Temporales únicos.** Cada intento usará un temporal hermano creado con exclusividad y nombre no reutilizable (por ejemplo, PID más nonce criptográfico). No se compartirá un nombre fijo `.tmp`.
- **REQ-079-15 — Durabilidad.** El protocolo escribirá el documento completo, validará el payload, hará flush/fsync del archivo, cerrará el descriptor, realizará replace atómico y hará fsync del directorio cuando la plataforma lo soporte. Las limitaciones de plataforma deberán quedar explícitas y cubiertas por un adapter probado.
- **REQ-079-16 — Lost-update protection.** Toda escritura read-modify-write comprobará bajo lock una revisión y fingerprint esperados. Si el target cambió desde la lectura, devolverá `state_conflict`; queda prohibido last-writer-wins silencioso.
- **REQ-079-17 — Recuperación determinista.** Tras interrupción, el reader nunca tratará un temporal como commit exitoso. La recuperación se ejecutará bajo lock, usará evidencia durable del primitive de Core y solo promoverá/limpiará artefactos cuando el estado sea inequívoco; en caso contrario devolverá `recovery_required` sin inventar éxito.
- **REQ-079-18 — Preservación ante fallo.** Error de serialización, permiso, write, fsync, replace, validación o conflicto conservará el último target válido y eliminará o dejará identificables los artefactos recuperables sin publicar un receipt parcial.

### Gramática y comportamiento CLI

- **REQ-079-19 — Operación exclusiva.** La gramática canónica será `axiom self-update <status|check|plan|apply|recover> [opciones]`. Debe existir exactamente una operación. La ausencia, repetición o combinación de selectores será `cli_usage_error`.
- **REQ-079-20 — Flags antiguos.** Selectores ambiguos como `--check`, `--apply`, `--recover` o combinaciones equivalentes no se interpretarán por precedencia. Se rechazarán con error tipado y una sustitución canónica.
- **REQ-079-21 — Status local y read-only.** `status` inspeccionará identidad y estado locales sin red y sin crear/modificar archivo, lock, temporal, cache, receipt o timestamp.
- **REQ-079-22 — Check read-only.** `check` podrá consultar el proveedor de releases, pero no modificará el filesystem local ni persistirá `lastCheck`; su observación vivirá en la respuesta de esa invocación.
- **REQ-079-23 — Plan read-only.** `plan` calculará una intención a partir de identidades y estado válidos, sin invocar el motor, adquirir lock mutante ni persistir ningún dato.
- **REQ-079-24 — Dry-run seguro.** `plan` será el dry-run canónico. Si `--dry-run` se conserva por compatibilidad, solo será válido con `plan` y será rechazado antes del dispatcher mutante en cualquier otra operación. Ninguna ruta con `--dry-run` alcanzará writer, migrador, lock mutante, apply o recover.
- **REQ-079-25 — Motor no disponible.** Hasta integrar el incremento 3, `apply` y `recover` terminarán con `engine_unavailable`, exit code estable y envelope tipado. No adquirirán locks, no migrarán y no emitirán outcomes `applied`/`recovered`.
- **REQ-079-26 — Opciones acotadas.** Cada operación aceptará solo sus opciones declaradas. Opciones desconocidas o propias de otra operación fallarán antes de cualquier I/O mutante.

### Salida, errores y proceso

- **REQ-079-27 — Envelope JSON v1.** En modo `--json`, éxito, resultado accionable y error emitirán exactamente un objeto JSON cerrado con `schemaVersion: 1`, comando, operación, `ok`, outcome, data/error y warnings.
- **REQ-079-28 — Canales limpios.** En modo JSON, stdout contendrá únicamente el envelope y un newline final. Logs, hints, warnings humanas y stack traces permitidos irán a stderr. Ninguna dependencia podrá contaminar stdout.
- **REQ-079-29 — Errores sanitizados.** El envelope no expondrá secretos, tokens, paths no solicitados, stack traces ni objetos de excepción sin normalizar. Los errores tendrán `code`, `message`, `retryable` y detalles cerrados por tipo.
- **REQ-079-30 — Códigos de salida.** Outcomes equivalentes tendrán el mismo código en la entrada TypeScript y en el wrapper compilado. Resultados accionables se distinguirán de errores sin confundirlos con aplicación exitosa.
- **REQ-079-31 — Finalización.** La ruta CLI asignará `process.exitCode` y retornará; no llamará a `process.exit()`, para permitir flush de stdout/stderr y cleanup controlado.

### Calidad y evidencia

- **REQ-079-32 — Tests contractuales.** Se probarán schema, migración, corrupción, concurrencia, lost updates, fallos en cada frontera durable, no mutación, envelope/canales, taxonomía y wrapper compilado.
- **REQ-079-33 — Build real.** La validación incluirá `npm run build`, pruebas focalizadas y `npx vitest run` en el runtime. El E2E del wrapper se ejecutará contra los artefactos compilados, no solo mediante imports de test.
- **REQ-079-34 — Habilitación, no anticipación.** Los contratos deberán permitir los incrementos 3–5 sin implementar Git update, Launcher UI ni release CI.

## Reglas de negocio relevantes

1. La identidad de la instalación real gana siempre frente al receipt local.
2. Ausente, inválido, no soportado, migrable y válido son estados distintos.
3. La única evidencia de éxito es un commit durable asociado a una operación mutante real; ni `check`, ni `plan`, ni `attempt` prueban éxito.
4. Un `attempt` y su eventual `success` se correlacionan por `operationId`; `recovery` mantiene identidad y outcome propios.
5. Las operaciones read-only deben ser comprobables por comparación de árbol, bytes, mtimes y ausencia de locks/temporales antes y después.
6. Un conflicto concurrente se informa; nunca se resuelve pisando silenciosamente la escritura ganadora.
7. Un archivo corrupto no se repara de oficio. La recuperación exige una operación mutante explícita y un protocolo seguro.
8. `apply`/`recover` no podrán reportar éxito hasta que el motor del incremento 3 entregue una evidencia durable verificable.

## Fuera de alcance funcional

- Mecanismo real de update o rollback de Git.
- Descarga y verificación de artefactos remotos más allá del consumo del contrato de release identity.
- UI de Launcher, notificaciones visuales o interacción gráfica.
- Pipelines de release, firma en CI o publicación multicanal.
- Telemetría nueva, daemon de actualización o scheduler.
- Edición manual de manifests como flujo soportado.
- Limpieza indiscriminada de temporales sin lock o sin evidencia de ownership.
