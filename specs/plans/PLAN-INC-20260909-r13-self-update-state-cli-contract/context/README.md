# Context

## Propósito

Concentrar el contexto operativo que el builder y reviewer necesitan para ejecutar el plan de ACC-079 sin redescubrir sus invariantes ni confundir diseño con implementación existente. Este es el incremento **2/5**; su dependencia obligatoria es `INC-20260909-r13-self-update-release-identity` y su consumidor inmediato será el motor del incremento 3.

La información siguiente es un mapa de trabajo. Los paths y símbolos runtime son probables y deben confirmarse en el gate G0.

## Qué puede vivir aquí

- Inventario confirmado de módulos/símbolos runtime y sus owners.
- Shapes v1 encontrados y matriz de compatibilidad aprobada.
- Capacidades reales del lock/atomic writer de `@axiom/core` por plataforma.
- Comando exacto y comportamiento del wrapper compilado.
- Evidencia resumida de gates, no-mutación, concurrencia y fault injection.
- Handoff contractual al incremento 3.

## Qué no debe vivir aquí

- Implementación TypeScript, outputs compilados o dependencias vendorizadas.
- Metadata/status/índices/receipts de lifecycle editados manualmente.
- Secretos, tokens, paths personales o manifests de usuarios reales.
- Diseño de Git update, Launcher UI o release CI.
- Copia integral de la spec/incremento o afirmaciones de éxito sin evidencia.

## Mapa de búsqueda inicial

El builder deberá resolver estas preguntas en una sola pasada de discovery:

1. ¿Dónde se calcula hoy la versión instalada y qué fuentes consulta?
2. ¿Quién lee/escribe `install.json` y con qué shapes v1?
3. ¿Cuál es el path canónico por instalación y cómo se evita depender de cwd?
4. ¿Qué primitive de `@axiom/core` implementa lock user-level y cuál es su clave/timeout/stale policy?
5. ¿Qué primitive implementa staging, fsync, replace y recovery en Windows/POSIX?
6. ¿Dónde se registran comandos/subcomandos y dónde se renderiza JSON/humano?
7. ¿Qué código llama a `process.exit()` o escribe directo en stdout?
8. ¿Qué archivo `bin`/wrapper se distribuye tras `npm run build`?
9. ¿Qué tests/fixtures existentes cubren release identity, self-update y filesystem?

Rutas candidatas a contrastar:

- `packages/core/src/**/self-update*` para estado, identidad, migración y writer;
- `packages/core/src/**` para lock y atomic filesystem existentes;
- `apps/cli/src/**/self-update*` para command/parser/handlers;
- entrypoint/renderer/bin declarados por `apps/cli` o su package;
- tests co-localizados o configuración Vitest existente.

No se crearán módulos paralelos solo porque estos nombres no coincidan con el árbol real.

## Invariantes contractuales

### Autoridad y estado

- Instalación real + `ReleaseIdentityV1` son autoridad.
- `install.json` v2 es receipt/cache.
- Ausencia = `state_absent`; nunca `0.0.0`.
- v1 legible = `migration_required` durante lecturas.
- Corrupción, schema no soportado, I/O y mismatch son ramas distintas.

### Mutación

- Solo una frontera mutante explícita puede invocar migrator/writer.
- En este incremento la CLI no tiene una operación mutante funcional: `apply/recover` fallan cerrados.
- `status`, `check`, `plan` y `plan --dry-run` no escriben nada.
- `--dry-run` en cualquier otra operación se rechaza antes de adapters mutantes.

### Atomicidad

- Lock user-level Core + target canónico.
- Expected revision y fingerprint comprobados bajo lock.
- Temporal hermano exclusivo/único.
- Write completo, fsync file, replace atómico, fsync dir cuando se soporte.
- Relectura antes de éxito.
- Temporal no equivale a commit.
- Recovery bajo lock y basada en evidencia, no mtime.

### CLI

- Exactamente una operación: `status|check|plan|apply|recover`.
- Envelope JSON v1 único y cerrado.
- stdout JSON limpio; stderr humano.
- `process.exitCode`, no `process.exit()`.
- Exit 0/10/11 para resultados contractuales; 64/65/69/73/74/75/78 para clases de error definidas.

## Matriz de compatibilidad a completar en G0/G1

| Entrada | Reader | Status | Migrador autorizado | Writer |
|---|---|---|---|---|
| archivo ausente | `state_absent` | read-only, exit 11 | no hay fuente v1 | solo una frontera mutante futura |
| v1 soportado y reconciliable | `migration_required` | read-only, exit 11 | sí, con identidad real | emite v2 |
| v1 desconocido/inválido | `state_corrupt` | read-only, exit 65 | no | no |
| v2 válido/reconciliado | `state_ready` | read-only, exit 0 | no-op/idempotente | v2 con CAS lógico |
| v2 válido con mismatch | `identity_mismatch` | read-only, exit 65 | no automático | no hasta resolución explícita |
| schema futuro | `schema_unsupported` | read-only, exit 65 | no | no |
| I/O/permiso | `state_io_error` | error tipado | no | no |
| transacción dudosa | `recovery_required` | no repara | solo recovery mutante futura | no commit hasta resolver |

Cada variante v1 real encontrada deberá añadir una fila/fixture antes de GO; los datos desconocidos se rechazan.

## Diseño de pruebas de no mutación

Para cada invocación read-only o fail-closed:

1. crear home/instalación sandbox con fixture;
2. capturar árbol, hashes, tamaños, mtimes y permisos relevantes;
3. instrumentar adapters de lock, writer, migrator y cache;
4. ejecutar mediante handler y wrapper compilado;
5. esperar flush/fin del proceso;
6. comparar snapshot exacto y verificar cero llamadas mutantes;
7. comprobar stdout, stderr y exit code.

Casos obligatorios: ausencia, v1, v2, corrupto, mismatch, up-to-date, update available, provider error, plan empty/ready, `plan --dry-run`, dry-run inválido, apply y recover no disponibles.

## Diseño de pruebas de concurrencia y recovery

- Lanzar workers como procesos separados con el mismo user-home sandbox.
- Barrera después de lectura para forzar dos writers con la misma precondición.
- Liberar ambos: uno compromete; el otro debe obtener `state_conflict`.
- Repetir con aliases del mismo path y con targets distintos.
- Inyectar fallo en create, write parcial, flush, close, replace, dir-fsync y re-read.
- Tras cada fallo, abrir en un proceso nuevo y clasificar: target previo válido o `recovery_required`.
- Verificar que un temporal huérfano nunca se promueve solo por ser reciente.
- Verificar unicidad de temporales y cleanup exclusivamente bajo lock.

Si el primitive Core no ofrece hooks de fault injection, se deberá introducir un adapter mínimo alrededor del filesystem existente; no un segundo writer productivo.

## Contrato de envelope para el harness

El harness validará un único objeto por stdout:

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

Validaciones raw:

- exactamente un JSON y un newline final;
- ninguna línea previa/posterior, BOM, spinner o debug;
- stderr separado;
- schema cerrado y campos específicos por outcome;
- exit code coincidente con outcome;
- errores sin stack, token ni excepción serializada.

## Gates operativos

| Gate | Evidencia mínima para GO | Motivos de STOP |
|---|---|---|
| G0 | dependency/API/Core/v1/wrapper confirmados | autoridad o primitives no demostrables |
| G1 | schema/outcomes/fixtures aprobados | defaults/campos abiertos/contradicción |
| G2A | snapshots read-only verdes | cualquier side effect de lectura |
| G2B | concurrency + fault matrix verde | lost update, parcial o recovery heurística |
| G3 | migración real/idempotente | v1 usado como autoridad o destruido en fallo |
| G4 | grammar/no-mutation/streams/exits verdes | ambigüedad, dry-run mutante o éxito fingido |
| G5 | build + wrapper compilado verdes | divergencia bin vs handler |
| G6 | review/rollback/handoff aceptados | AC sin evidencia o contrato inestable |

## Rollback y preservación

- No existe down-migration v2→v1 automática.
- Un fallo pre-commit conserva v1/target previo.
- Un v2 comprometido no se borra para facilitar rollback.
- Un writer defectuoso se deshabilita; no se sustituye por write directo.
- Un estado dudoso se preserva y reporta.
- El wiring CLI puede revertirse antes de release, pero no debe restaurar flags ambiguos con acceso mutante.

## Validación prevista

En el runtime Axiom se ejecutarán como base:

```text
npm run build
npx vitest run
```

El gate G5 añadirá la suite focalizada confirmada durante discovery y ejecutará el bin real declarado por el package compilado con `self-update status --json` y `self-update apply --json`. La evidencia deberá registrar cada comando exacto, resultado, stdout/stderr/exit y clasificación de fallos.

## Handoff e integración canónica

El handoff al incremento 3 incluirá APIs confirmadas de reader/migrator/writer, precondiciones CAS, receipts y taxonomía; el motor deberá producir attempt/success/recovery reales sin cambiar schema/grammar/envelope. Los incrementos 4 y 5 consumirán esos contratos después.

Tras validación y review, el conocimiento estable se consolidará en las specs canónicas propietarias del modelo operativo y la interfaz CLI. Este plan no modifica esas specs, no cambia status y no ejecuta transiciones Core.
