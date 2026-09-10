# Rol de plan: builder

## Rol

- nombre del rol: builder
- repositorio(s) asignado(s): `Axiom` (runtime TypeScript: `packages/*`, `apps/cli`); `Axiom.Spec` solo para evidencia/integración mediante el workflow autorizado
- posición en la secuencia: incremento **2 de 5**

## Objetivo del rol en este plan

Implementar ACC-079 sin anticipar el motor mutante: schema y estado v2, reader/migrator/writer seguros, gramática CLI exclusiva, operaciones read-only verificables y `apply/recover` fail-closed. El builder debe consumir la identidad de release del incremento 1 y dejar APIs estables para los incrementos 3–5.

No debe considerar este plan como evidencia de que el comportamiento ya existe. Antes de editar código deberá confirmar las rutas y símbolos reales.

## Alcance incluido

1. **Discovery y baseline**
   - Confirmar implementación/contrato de `INC-20260909-r13-self-update-release-identity`.
   - Buscar todos los usos de `install.json`, versión instalada, flags self-update, `process.exit`, stdout/stderr y wrappers.
   - Identificar fixtures v1 reales y el path resolver soportado.
   - Identificar y reutilizar lock user-level, atomic write/replace, fsync y recovery de `@axiom/core`.
   - Ejecutar baseline de build/tests y registrar fallos preexistentes.

2. **Modelo ejecutable**
   - Añadir `InstallStateV2`, receipts separados, unión de resultados de lectura, envelope v1 y errores/outcomes.
   - Cerrar schemas en cada nivel y reutilizar validators del incremento 1.
   - Mantener archivo ausente como rama `state_absent`, nunca `0.0.0`.

3. **Reader/migrator/writer**
   - Leer v1/v2 sin side effects.
   - Migrar v1 solo desde una frontera mutante explícita y usando identidad real.
   - Escribir solo v2 con revision/fingerprint esperados.
   - Usar temporales únicos, fsync/replace/recovery y lock user-level Core.
   - Rechazar stale writers y preservar target válido en fallos.

4. **CLI**
   - Implementar `self-update <status|check|plan|apply|recover>` con una operación obligatoria.
   - Rechazar flags legacy y opciones cruzadas; `--dry-run` solo con `plan`.
   - Mantener `status`, `check` y `plan` read-only.
   - Devolver `engine_unavailable`/69 para `apply` y `recover`, sin receipt ni lock.
   - Emitir envelope JSON único por stdout, texto humano por stderr y usar `process.exitCode`.

5. **Pruebas y evidencia**
   - Schema/corrupción/compatibilidad, concurrencia real, fault injection, no-mutación, envelopes, exit codes y wrapper compilado.
   - Build, suite focalizada y suite completa.
   - Handoff documentado al owner del incremento 3.

## Archivos y símbolos probables

Estos candidatos orientan la búsqueda inicial; no son autorización para crear módulos duplicados:

| Candidato | Símbolos esperados |
|---|---|
| módulo self-update/state bajo `packages/core/src/**` | `InstallStateV2`, `InstallStateReadResult`, schema/parser/serializer |
| módulo release identity del incremento 1 | `ReleaseIdentityV1`, resolver de identidad instalada, validators |
| módulo de filesystem/locking existente en `@axiom/core` | lock user-level, atomic replace/write, fsync/recovery |
| módulo de migración self-update | `decodeInstallStateV1`, `migrateInstallStateV1ToV2` |
| comando bajo `apps/cli/src/commands/**` | handlers `status/check/plan/apply/recover`, parser discriminado |
| renderer/entrypoint CLI existente | `SelfUpdateEnvelopeV1`, mapper outcome→exit, asignación `process.exitCode` |
| bin/wrapper declarado en package | proceso compilado usado por E2E |
| tests co-localizados o suite existente | fixtures, concurrency workers, fault adapter, CLI harness |

Si la búsqueda descubre ownership distinto, el builder mantendrá la arquitectura existente y actualizará el mapa de contexto; no moverá código por estética.

## Orden de trabajo obligatorio

1. G0: dependencia, primitives, v1 y wrapper confirmados.
2. G1: contratos/tests de schema cerrados antes del writer.
3. G2A: reader puro.
4. G2B: writer/lock/fault/concurrencia.
5. G3: migración real e idempotente sobre el writer aprobado.
6. G4: grammar/handlers/renderers/no-mutación/fail-closed.
7. G5: build y wrapper compilado.
8. G6: review, rollback y handoff.

No avanzar después de un STOP sin resolver la causa con el owner correspondiente.

## Fuera de alcance

- Implementar fetch, checkout, replace o rollback Git.
- Marcar `apply`/`recover` como éxito o escribir attempts ficticios.
- Launcher UI, IPC, release CI o publicación.
- Crear un lock, journal o atomic writer paralelo si Core no cumple; eso requiere STOP y decisión explícita.
- Persistir checks, planes o dry-runs.
- Auto-migrar al ejecutar `status`, `check`, `plan` o al cargar la CLI.
- Usar `0.0.0`, cwd o datos v1 como sustituto de identidad real.
- Cambiar metadata, status, índices, receipts o ejecutar transiciones lifecycle manualmente.
- Refactors no necesarios para ACC-079.

## Dependencias

- **Obligatoria**: `INC-20260909-r13-self-update-release-identity` implementada y validada.
- **Runtime**: contrato/validators de release identity; provider usado por `check`; command framework y renderer existentes.
- **Core**: lock user-level y primitives de escritura durable/replace/recovery.
- **Filesystem**: garantías comprobables en cada plataforma soportada.
- **Consumidor futuro**: incremento 3, que debe aceptar grammar/schema/envelope sin romperlos.

## Validaciones y evidencias esperadas

### Contrato y compatibilidad

- Tests positivos/negativos de cada campo y `additionalProperties: false` en todos los niveles.
- Fixtures v1 reales con resultado de migración esperado.
- Prueba de que v1 mismatch/corrupto queda byte por byte intacto.
- Prueba de que ausencia no produce versión ni crea archivo.

### Atomicidad y concurrencia

- Procesos independientes compitiendo por el mismo target y por targets distintos.
- Stale revision/fingerprint produce `state_conflict`, sin overwrite.
- Temporales exclusivos y únicos.
- Fallos inyectados en create/write/fsync/close/replace/dir-fsync/re-read.
- Recovery bajo lock sin promoción por heurística.

### CLI

- Matriz completa de subcomando/opciones/legacy/dry-run.
- Snapshot filesystem para `status`, `check`, `plan`, errores y fail-closed.
- Envelope único parseable, stdout sin texto y stderr humano.
- Error sanitizado sin stack/secreto.
- `process.exitCode` y ausencia de `process.exit()` en la ruta.
- `apply/recover`: `engine_unavailable`, exit 69 y cero mutación.

### Comandos

Desde el runtime se ejecutarán como mínimo:

```text
npm run build
npx vitest run
```

Tras el discovery, el builder añadirá las suites focalizadas reales y ejecutará el bin declarado por el package compilado con `self-update status --json` y `self-update apply --json`. La evidencia registrará esos comandos exactos junto con exit/stdout/stderr. No basta una prueba mediante imports: el wrapper compilado es gate.

### Review

- Revisión independiente contra AC-079-01..44.
- Diff audit de exclusiones y de escrituras indirectas.
- Confirmación del owner del incremento 3 sobre el handoff.
- Clasificación explícita de cualquier fallo preexistente frente a regresiones nuevas.

## Rollback del builder

- Mantener cambios pequeños y separables por contrato, storage y CLI.
- Antes de release, revertir wiring/código focalizado si falla un gate; no down-migrar receipts v2.
- Ante fallo de migración, preservar v1 como target.
- Ante defecto de writer, deshabilitar fronteras mutantes; no degradar a write no atómico.
- Ante defecto de CLI, no restaurar flags ambiguos con precedencia.
- No eliminar temporales/estado dudoso fuera del protocolo de recovery.

## Bloqueos y observaciones

- **STOP** si release identity depende circularmente de `install.json`.
- **STOP** si no se pueden enumerar las variantes v1 soportadas.
- **STOP** si el lock no es realmente cross-process/user-level para el target canónico.
- **STOP** si una plataforma soportada no puede mantener target válido durante replace y no hay primitive Core probado.
- **STOP** si `check` o dependencias escriben cache implícita que no puede desactivarse.
- **STOP** si dry-run se evalúa después de entrar al dispatcher mutante.
- **STOP** si el wrapper compilado no conserva streams/exit codes.
- **STOP** ante cualquier outcome `applied`/`recovered` antes del incremento 3.

GO final exige build y suites verdes, review sin bloqueantes, rollback seguro y evidencia suficiente para integrar conocimiento estable. El builder no cambia status ni ejecuta transiciones Core como parte de este plan.
