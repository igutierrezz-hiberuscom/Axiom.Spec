# Registro user-level de proyectos R-13

> **Código**: INC-20260829-r13-user-project-registry
> **Estado documental**: implementado y verificado; el estado lifecycle y el archivado los gestiona Axiom Core
> **Fecha de creación**: 2026-08-29
> **Acciones**: ACC-050..ACC-056

## Objetivo

Dejar `~/.axiom/projects.yml` como único catálogo user-level, estricto y concurrentemente seguro, con identidad sin colisiones, disponibilidad precisa y contratos CLI/JSON honestos.

## Contexto

La revalidación inicial del 2026-08-29 encontró `registry.json`, el migrador v1, un estado `stale` ambiguo, un temporal fijo y `projects use --print-cwd`. Ese baseline motivó ACC-050..056; el resultado implementado ya no conserva ninguna de esas superficies activas.

## Alcance incluido

- ACC-050: retirada completa del registro/migrador v1 en código, tests y documentación activa.
- ACC-051: `projects use` solo actualiza recencia; `--print-path` sustituye a `--print-cwd`; listado determinista por recencia.
- ACC-052: validación estricta de todo `projects.yml` y errores de dominio tipados.
- ACC-053: contrato de ID, Unicode, ownership y comparación canónica de paths sin merges accidentales.
- ACC-054: mutaciones coordinadas con lock local acotado, temporal único y rename atómico.
- ACC-055: disponibilidad física precisa y resolubilidad Axiom separada.
- ACC-056: envelope JSON v1 uniforme en éxito/error para `list|add|join|use`, stdout limpio y exit code honesto.
- CLI `projects`, consumidores launcher/MCP y documentación operativa afectada.
- Primitive reutilizable de lock/escritura segura en `@axiom/core`, propietario también de ACC-062/066 posteriores.

## No objetivos

- No cambiar todavía el schema topológico ni resolver el workspace desde `axiomRepo` (B-D).
- No introducir transacciones estructurales multiarchivo (E).
- No tocar self-update/R-13.5.
- No conservar ni migrar v1; no hay instalaciones que lo requieran.
- No registrar implícitamente un proyecto al comprobar resolubilidad.

## Riesgos

- Colisiones de slug o equivalencias de path dependientes de plataforma.
- Lock abandonado o proceso interrumpido; se mitiga con timeout, owner/edad verificables, limpieza segura y errores accionables.
- Consumidores que dependan del shape JSON o de `stale`; se exige barrido de compilación y tests E2E del wrapper.
- Solapamiento con `app-api.ts` modificado localmente por R-12; se preservarán esos cambios.

## Decisiones cerradas

1. ID explícito: slug lowercase ASCII válido; ID derivado: nombre normalizado NFKC, diacríticos eliminados y separadores colapsados. Derivación vacía exige ID explícito. Una colisión nunca se fusiona.
2. Paths: `resolve` + canonicalización física cuando existe; comparación case-insensitive en Windows y sensible en plataformas que lo sean. Symlinks/junctions equivalentes comparten ownership.
3. Reejecución idempotente solo si coinciden ID canónico y todos los repos ya poseídos; cualquier evidencia divergente falla sin escribir.
4. `projects use` no selecciona contexto. `--print-path` emite exactamente un path y salto de línea. La elección por defecto prefiere un repo marcado `axiom`; si aún no existe esa metadata, usa la key lexicográficamente menor. `--role` explícito prevalece.
5. Orden de lista: `lastUsedAt` descendente, después `id` ascendente.
6. Envelope: `{schemaVersion:1, ok, command, data? , error?}`; un único JSON en stdout cuando `--json` está activo.
7. Disponibilidad de repo: `present-directory | missing | not-directory`; resolubilidad Axiom es un campo separado. Rollup: `available` (todos directorios), `partial` (mezcla) o `unavailable` (ningún directorio).

## Resultado implementado

- `@axiom/user-workspace` conserva únicamente `ProjectsFile` schema 2, valida el YAML completo y devuelve errores discriminados para schema, identidad, ownership, I/O y timeout de lock.
- Las altas y upserts comparan paths canónicos, rechazan colisiones sin mutar y serializan el read-modify-write completo.
- `@axiom/core` aporta `acquireLocalFileLockSync`, `withLocalFileLockSync` y `atomicWriteFileSync`. El lock usa un lease previo a `mkdir`, publica y fuerza a disco `owner.json` antes del handoff, y liga cada reclaim claim a la generación/epoch observada. Esto cierra tanto el borrado tardío de una generación sucesora como la ventana de publicación del owner.
- CLI, launcher/MCP y consumidores usan disponibilidad y resolubilidad separadas; `projects use` solo actualiza recencia y `--print-path` reemplaza a `--print-cwd`.
- `list|add|join|use --json` comparten el envelope v1 y mantienen stdout con un único documento JSON.

## Estrategia de compatibilidad

Rechazo explícito de compatibilidad legacy: se eliminan `registry.json`, tipos v1, migrador y error de migración. Un archivo v1 residual no se lee ni se modifica; `projects.yml` ausente significa catálogo nuevo vacío.

## Validación ejecutada

- Dos ejecuciones frescas de `npx.cmd vitest run packages/core/tests/local-file.test.ts packages/user-workspace/tests/registry-concurrency.test.ts packages/user-workspace/tests/registry-v2.test.ts --reporter=basic`: 3/3 archivos y 40/40 tests en ambas.
- `npx.cmd tsc -b --pretty false`: exit 0.
- `npm run build`: exit 0 antes de los tests y de nuevo al final.
- `git diff --check`: exit 0; solo avisos de normalización LF/CRLF.
- Parse de `Axiom.Pruebas/verificar-adopcion.ps1`: exit 0; barrido de fixtures sin referencias activas a registry v1.
- El freeze `a27f7afcf598a7b09126d50efbb669562bd8bf401568915ff47b74e645ea68fc` se verificó inmediatamente antes de cada apply delegado. Los receipts apply válidos tienen hashes `d32c68eea90b7fe90b9a384a58ecf24eceffbf25b98b1e9cbb19690d0ad26218` y `347174a14f953edb4e0bb880c3c164b97c9a50f6956fef746bd39cd91f04ff4c`; el receipt verify tiene hash `42cfa2a4d3d6e5b2fcc77a622bd0f392026a8f0011e927178b23e23755d415ec`.
- La primera revisión independiente detectó la ventana `mkdir`→owner; tras la remediación, la revisión final no encontró blockers y devolvió `ready_for_verify: yes`.

La actualización documental final de este README cambia el candidate después de los applies; no se presenta ese freeze previo como vigente ni se ejecuta un apply adicional.

## Integración documental

El conocimiento estable se consolidó una sola vez en `specs/01`, `02`, `03`, `05`, `07`, `08`, `specs/manuales/02_Configuracion.md` y los owners de `context/{architecture,operations,references}`. `specs/00`, `04` y `06` se revisaron y no requerían cambios: no poseían claims obsoletos de registry v1 ni del contrato CLI. ACC-050..056 quedan reconciliadas en `PLAN-REVISION-INTEGRAL-AXIOM.md`.

El movimiento a `_archive`, el estado terminal y el registro de integración se ejecutan exclusivamente mediante `axiom integrate`; no se editan manualmente metadata, links ni índices.

## Dudas abiertas

Ninguna bloqueante. El enriquecimiento estructural de repos en el catálogo pertenece a D y debe conservar este contrato sin inferencias por keys `sdd/spec`.
