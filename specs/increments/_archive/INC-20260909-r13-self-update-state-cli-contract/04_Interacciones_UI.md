# 04 Interacciones UI

## Objetivo del documento

Definir la interacción observable de ACC-079 en la CLI. En este incremento, «UI» significa exclusivamente terminal y contrato machine-readable; no incluye Launcher ni interfaz gráfica. Los ejemplos representan el comportamiento requerido, no evidencia de que ya esté disponible.

## Superficie UI afectada

La superficie canónica será:

```text
axiom self-update status  [--json]
axiom self-update check   [--json]
axiom self-update plan    [--json] [--dry-run]
axiom self-update apply   [--json]
axiom self-update recover [--json]
```

Reglas de entrada:

- El subcomando es obligatorio y exclusivo.
- `--json` cambia representación, no semántica ni exit code.
- `--dry-run` solo se admite con `plan`; en cualquier otra operación es error de uso antes de la frontera mutante.
- Flags selectores legacy no se combinan ni resuelven por precedencia. El error indicará el subcomando sustituto.
- Opciones desconocidas o pertenecientes a otra operación se rechazan sin side effects.
- `--help` podrá mostrar gramática y exit codes sin inspeccionar ni modificar la instalación.

## Flujo de interacción

### `status`

1. Resuelve la instalación local y el path canónico del receipt.
2. Inspecciona la identidad real mediante el contrato del incremento 1.
3. Lee y clasifica `install.json` si existe.
4. Compara receipt e identidad real sin corregirlos.
5. Presenta uno de los estados discriminados y termina.

No usa red, no toma lock mutante y no migra v1. Un archivo ausente se muestra como «estado local no registrado»/`state_absent`, no como versión `0.0.0`.

### `check`

1. Obtiene la identidad instalada real.
2. Consulta la identidad candidata por el provider soportado.
3. Compara identidades según release identity.
4. Devuelve `up_to_date`, `update_available` o error tipado.

No persiste el momento del check ni ningún candidate receipt. Un fallo remoto no afecta el estado local.

### `plan`

1. Valida estado e identidades de entrada.
2. Calcula un plan declarativo con from/to refs, precondiciones y acciones que corresponderán al motor futuro.
3. Devuelve `plan_empty` o `plan_ready`.

El plan no se ejecuta, no reserva lock y no produce attempt. `plan --dry-run` es equivalente en efectos a `plan`; el flag no habilita una ruta de apply.

### `apply`

Hasta el incremento 3:

1. valida gramática y opciones;
2. devuelve `engine_unavailable` y exit `69`;
3. no toma lock, no migra, no escribe attempt/success y no presenta «applied».

### `recover`

Hasta el incremento 3:

1. valida gramática y opciones;
2. devuelve `engine_unavailable` y exit `69`;
3. no limpia/promueve temporales, no modifica estado y no presenta «recovered».

La recuperación de bajo nivel será una capacidad del writer/migrator probada en aislamiento, pero su operación CLI mutante no quedará fingida antes del motor.

## Estados visibles

| Operación | Outcome | Exit | Presentación humana mínima |
|---|---|---:|---|
| status | `state_ready` | 0 | Identidad instalada y provenance reconciliadas |
| status | `state_absent` | 11 | Receipt ausente; no se muestra versión sintética |
| status | `migration_required` | 11 | v1 legible; requiere frontera mutante futura |
| status | `state_corrupt` | 65 | Estado inválido; no se reparó |
| check | `up_to_date` | 0 | No hay actualización |
| check | `update_available` | 10 | Release candidata disponible |
| plan | `plan_empty` | 0 | Ninguna acción prevista |
| plan | `plan_ready` | 10 | Plan declarativo disponible, no ejecutado |
| apply/recover | `engine_unavailable` | 69 | Capacidad pendiente del incremento 3 |
| cualquier operación | `cli_usage_error` | 64 | Uso inválido y forma canónica sugerida |

### Envelope JSON

Éxito de lectura:

```json
{"schemaVersion":1,"command":"self-update","operation":"status","ok":true,"outcome":"state_ready","data":{"state":"ready","receiptSchemaVersion":2,"reconciled":true},"error":null,"warnings":[]}
```

El objeto de datos definitivo incluirá la identidad sanitizada mediante el shape cerrado `ReleaseIdentityV1`; no se replica aquí para evitar que este incremento diverja de su dependencia obligatoria.

Archivo ausente:

```json
{"schemaVersion":1,"command":"self-update","operation":"status","ok":true,"outcome":"state_absent","data":{"state":"absent","receipt":null},"error":null,"warnings":[]}
```

Motor no disponible:

```json
{"schemaVersion":1,"command":"self-update","operation":"apply","ok":false,"outcome":"engine_unavailable","data":null,"error":{"code":"SELF_UPDATE_ENGINE_UNAVAILABLE","message":"Self-update apply is not available in this increment.","retryable":false,"details":{"requiredIncrement":3}},"warnings":[]}
```

Cada ejemplo se emitirá como una sola línea más newline. En modo `--json`, ningún banner, spinner, hint, debug log o stack trace puede aparecer en stdout.

### Modo humano

- Los resultados normales podrán imprimirse de forma concisa por stdout.
- Errores, warnings y hints se imprimirán por stderr.
- El mensaje debe distinguir «hay actualización» de «se aplicó actualización».
- La indicación de v1 debe decir que la migración está pendiente, no que ocurrió.
- La indicación de corrupción debe preservar el archivo y ofrecer una acción futura segura, no recomendar editarlo manualmente.
- El texto humano no forma parte del contrato estable; outcome, error code y exit code sí.

## Cascadas y comportamiento reactivo

- `--json` solo selecciona renderer. El handler de dominio no escribe directamente en streams.
- El renderer emite exactamente una vez y el entrypoint asigna `process.exitCode`; no se usa `process.exit()`.
- Un error del provider durante `check` cambia outcome/exit, pero no dispara recovery, migración ni persistencia.
- Detectar v1 en `status/check/plan` solo añade estado/warning machine-readable; no dispara el migrador.
- Detectar mismatch o corrupción bloquea cualquier plan que requiera confianza en el receipt y devuelve error tipado.
- `apply --dry-run` y `recover --dry-run` se rechazan; nunca se reinterpretan tarde después de entrar al dispatcher mutante.
- Hasta el incremento 3, ninguna combinación de opciones puede convertir `engine_unavailable` en `applied` o `recovered`.
- El Launcher futuro deberá consumir el mismo envelope/outcome sin alterar este contrato, pero no se implementa aquí.
