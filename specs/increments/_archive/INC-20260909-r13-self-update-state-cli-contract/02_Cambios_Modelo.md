# 02 Cambios de Modelo

## Objetivo del documento

Definir el modelo objetivo de `install.json` v2, su compatibilidad con v1, las fronteras de lectura/escritura y los contratos CLI observables de ACC-079. Los nombres aquí son normativos a nivel de concepto; los símbolos TypeScript exactos deberán alinearse con el runtime existente durante la implementación.

## Entidades o estructuras afectadas

### 1. `InstallStateV2`

El documento persistido será un objeto JSON cerrado. Su representación conceptual, expresada como pseudotipo para no duplicar el shape de la dependencia, es:

```ts
type InstallStateV2 = {
  schemaVersion: 2;
  revision: PositiveSafeInteger;
  installation: {
    path: AbsoluteNormalizedPath;
    release: ReleaseIdentityV1;
  };
  receipts: {
    check: CheckReceipt | null;
    attempt: AttemptReceipt | null;
    success: SuccessReceipt | null;
    recovery: RecoveryReceipt | null;
  };
  provenance: {
    kind: "installer" | "legacy-migration" | "self-update";
    recordedAt: CanonicalUtcIsoTimestamp;
    sourceSchemaVersion: 1 | null;
    sourceFingerprint: Sha256Fingerprint | null;
  };
};
```

Reglas cerradas:

- `schemaVersion` es el literal entero `2`.
- `revision` es un entero seguro positivo y aumenta una vez por commit durable; no aumenta en lecturas ni en escrituras fallidas.
- `installation.path` es absoluto, normalizado y reconciliable con la instalación inspeccionada. No se resuelve de forma relativa al cwd.
- `installation.release` usa directamente el contrato cerrado `ReleaseIdentityV1` del incremento 1. Debe aportar SemVer canónico, ref canónica y provenance verificable; este incremento no redefine ni debilita ese contrato.
- `receipts` contiene siempre las cuatro claves; cada valor es `null` o el objeto cerrado de su tipo.
- `provenance.kind` pertenece a `installer | legacy-migration | self-update`.
- `provenance.recordedAt` es ISO-8601 UTC canónico con milisegundos y sufijo `Z`.
- `sourceSchemaVersion` solo puede ser `1` en una migración o `null`.
- `sourceFingerprint` es `sha256:<64-hex-lowercase>` para migración o `null`; identifica los bytes fuente, no una versión instalada.
- `additionalProperties: false` aplica a la raíz y a cada objeto anidado, incluidos los contratos importados.

`install.json` no es autoridad. El reader devolverá por separado la identidad real inspeccionada y el receipt, para que una coincidencia o discrepancia sea explícita.

### 2. Receipts y correlación

Los cuatro tipos no son intercambiables y sus shapes exactos son:

```ts
type CheckReceipt = {
  checkedAt: CanonicalUtcIsoTimestamp;
  outcome: "up_to_date" | "update_available" | "check_failed";
  installedRef: CanonicalReleaseRef;
  candidateRef: CanonicalReleaseRef | null;
};

type AttemptReceipt = {
  operationId: OperationId;
  kind: "apply" | "recover";
  startedAt: CanonicalUtcIsoTimestamp;
  fromRef: CanonicalReleaseRef;
  targetRef: CanonicalReleaseRef | null;
  outcome: "started" | "failed" | "cancelled";
};

type SuccessReceipt = {
  operationId: OperationId;
  completedAt: CanonicalUtcIsoTimestamp;
  installedRef: CanonicalReleaseRef;
  installedPath: AbsoluteNormalizedPath;
};

type RecoveryReceipt = {
  operationId: OperationId;
  recoveredAt: CanonicalUtcIsoTimestamp;
  outcome: "restored" | "no_action";
  restoredRef: CanonicalReleaseRef | null;
};
```

| Tipo | Significado | ¿Prueba instalación exitosa? |
|---|---|---|
| `CheckReceipt` | Observación de disponibilidad | No |
| `AttemptReceipt` | Inicio, fallo o cancelación de una mutación | No |
| `SuccessReceipt` | Commit durable de apply real | Sí, sujeto a reconciliación con instalación real |
| `RecoveryReceipt` | Resultado durable de recuperación real | No se etiqueta como apply |

Cada objeto aplica `additionalProperties: false`. `OperationId` será ASCII opaco no vacío de 1–128 caracteres y patrón `[A-Za-z0-9][A-Za-z0-9._:-]*`; los timestamps, paths y refs usarán los validators canónicos indicados. `SuccessReceipt.operationId` deberá corresponder al attempt del apply que se comprometió. Un recovery fallido se representa por el attempt/error, no por un `RecoveryReceipt` de éxito, y un error previo al comienzo mutante no crea `AttemptReceipt`.

Invocar `check` en este incremento no persistirá `CheckReceipt`. El campo permite conservar evidencia producida por una futura transacción mutante o migrar evidencia legacy verificable, pero no convierte un comando read-only en writer.

### 3. Resultado de lectura

El reader modelará una unión discriminada y no devolverá defaults engañosos:

- `state_absent`: el path objetivo no existe;
- `migration_required`: v1 reconocido y semánticamente legible;
- `state_ready`: v2 válido y reconciliado con la instalación real;
- `identity_mismatch`: v2 válido, pero receipt e instalación real discrepan;
- `state_corrupt`: JSON truncado/inválido o shape v1/v2 inválido;
- `schema_unsupported`: `schemaVersion` conocido como número pero no soportado;
- `state_io_error`: fallo de acceso no equivalente a ausencia;
- `recovery_required`: existen señales de una transacción interrumpida que no pueden resolverse de forma inequívoca en una lectura.

`state_absent` no contendrá `version: "0.0.0"`. Una identidad no disponible será `null` o una rama discriminada, nunca una SemVer fabricada.

### 4. `InstallStateMigrationV1ToV2`

La migración seguirá estas fases:

1. leer bytes v1 y fingerprint desde un descriptor estable;
2. decodificar solo variantes v1 explícitamente soportadas;
3. localizar e inspeccionar la instalación real;
4. obtener `ReleaseIdentityV1` mediante la dependencia obligatoria;
5. reconciliar path, ref, versión y provenance; los valores v1 sirven como evidencia legacy, no como autoridad;
6. construir v2 con `provenance.kind = legacy-migration`, `sourceSchemaVersion = 1` y fingerprint de los bytes fuente;
7. validar v2 completo antes de escribir;
8. adquirir el lock user-level, releer y verificar fingerprint/revisión esperados;
9. publicar con el writer atómico;
10. releer y validar el target comprometido antes de informar éxito.

La migración será idempotente: v2 válido no se vuelve a migrar; reintentar después de un commit devolverá el estado v2 existente; una fuente cambiada causará `state_conflict`. Todo fallo anterior al commit preservará los bytes v1.

### 5. `AtomicInstallStateWriter`

El writer deberá componerse sobre primitives existentes de `@axiom/core`:

1. canonicalizar target y derivar una clave user-level estable;
2. adquirir el lock compartido por todos los procesos Axiom del usuario;
3. ejecutar recuperación/limpieza segura de una transacción anterior conforme al primitive Core;
4. releer target bajo lock y comparar `expectedRevision` más `expectedFingerprint`;
5. serializar de manera determinista, con UTF-8 y newline final;
6. crear un temporal hermano mediante `open(..., "wx")` y sufijo único (PID + nonce o primitive equivalente);
7. escribir todos los bytes y comprobar longitud;
8. hacer flush/fsync del archivo y cerrarlo;
9. validar el temporal desde bytes persistidos;
10. reemplazar el target de forma atómica sin ventana de documento parcial;
11. hacer fsync del directorio cuando esté soportado;
12. releer target, confirmar revision/fingerprint y liberar lock en `finally`.

No se promoverá «el temporal más nuevo» por heurística. Un temporal por sí solo nunca significa commit. Si las capacidades de Core no permiten demostrar replace durable o recuperación inequívoca en una plataforma soportada, el plan entra en gate **STOP** antes de crear una solución paralela.

### 6. Envelope JSON v1

Toda operación en modo JSON devolverá exactamente este shape cerrado:

```json
{
  "schemaVersion": 1,
  "command": "self-update",
  "operation": "status",
  "ok": true,
  "outcome": "state_ready",
  "data": {},
  "error": null,
  "warnings": []
}
```

Reglas:

- `operation` pertenece a `status | check | plan | apply | recover`.
- `ok` indica que la operación produjo su resultado contractual; no implica que se haya aplicado una actualización.
- `outcome` es un literal estable de la taxonomía.
- `data` es un objeto cerrado específico del outcome o `null`.
- `error` es `null` o `{ code, message, retryable, details }`, también cerrado.
- `warnings` es una lista de objetos cerrados `{ code, message }` y no sustituye al error principal.
- No se incluirán campos volátiles no requeridos ni excepciones crudas.

### 7. Taxonomía de outcomes y exit codes

| Exit | Clase | Outcomes/códigos representativos |
|---:|---|---|
| `0` | Completado sin acción requerida | `state_ready`, `up_to_date`, `plan_empty` |
| `10` | Resultado accionable, no error | `update_available`, `plan_ready` |
| `11` | Estado local requiere atención, no versión fabricada | `state_absent`, `migration_required` |
| `64` | Uso CLI inválido | `cli_usage_error`, `conflicting_operation`, `invalid_option`, `dry_run_forbidden` |
| `65` | Datos inválidos | `state_corrupt`, `schema_unsupported`, `identity_mismatch`, `migration_rejected` |
| `69` | Capacidad todavía no disponible | `engine_unavailable` |
| `73` | Exclusión/conflicto | `lock_unavailable`, `state_conflict` |
| `74` | I/O o durabilidad | `state_io_error`, `atomic_write_failed`, `recovery_required` |
| `75` | Fallo temporal externo | `release_check_unavailable` |
| `78` | Configuración/provenance inválida | `release_identity_invalid`, `provenance_invalid` |

El wrapper compilado deberá preservar estos códigos. Un outcome `update_available` o `plan_ready` nunca se llamará `applied`. Los mensajes humanos pueden evolucionar; `schemaVersion`, `operation`, `outcome`, `error.code` y exit code forman el contrato automatizable.

## Contratos o estados afectados

### Gramática

```text
axiom self-update status  [--json]
axiom self-update check   [--json]
axiom self-update plan    [--json] [--dry-run]
axiom self-update apply   [--json]
axiom self-update recover [--json]
```

- Exactamente un subcomando es obligatorio.
- `--dry-run` solo es aceptable con `plan`, donde confirma semántica read-only; en `apply`/`recover` se rechaza antes del dispatcher.
- Los antiguos flags selectores no tienen precedencia ni fallback silencioso.
- Hasta el incremento 3, `apply` y `recover` siempre terminan en `engine_unavailable` después de validar sintaxis y antes de lock/migración/escritura.

### Efectos por operación

| Operación | Red | Lectura local | Escritura/lock mutante | Estado en incremento 2 |
|---|---:|---:|---:|---|
| `status` | No | Sí | No | Disponible |
| `check` | Sí, si el proveedor lo requiere | Sí | No | Disponible |
| `plan` | Solo a través de input/check explícito definido | Sí | No | Disponible |
| `apply` | No antes del motor | Validación mínima | No | Fallo cerrado tipado |
| `recover` | No antes del motor | Validación mínima | No | Fallo cerrado tipado |

### Frontera de proceso

El handler retornará un resultado normalizado al entrypoint. El entrypoint serializará una sola vez, escribirá al canal correspondiente y asignará `process.exitCode`. Ninguna rama de dominio llamará a `process.exit()` ni escribirá directamente en stdout.

## Notas de compatibilidad

- v1 se soporta únicamente para lectura y migración controlada; no para nuevas escrituras.
- Un v1 que no pueda reconciliarse con la instalación real permanece intacto y se reporta como `migration_rejected` o `identity_mismatch`.
- Un v2 con campos extra no se «limpia»: se rechaza como inválido.
- Una versión con prefijo `v`, timestamp no UTC, path relativo o ref/provenance inválida no se normaliza silenciosamente en el reader.
- Los flags legacy reciben error de uso con hint hacia el subcomando; no se mantiene una matriz de precedencias histórica.
- `--dry-run` no conserva semántica de «apply que promete no escribir»; la operación contractual es `plan`.
- La versión del envelope (`1`) es independiente de la versión de `install.json` (`2`).
- La integración futura del motor deberá extender implementaciones detrás de `apply/recover` sin cambiar la gramática, los envelopes ni la separación attempt/success/recovery.
