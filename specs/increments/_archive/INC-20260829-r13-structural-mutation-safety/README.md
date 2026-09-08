# Seguridad transaccional de mutaciones estructurales R-13

> **Código**: INC-20260829-r13-structural-mutation-safety
> **Estado documental**: especificación refinada; lifecycle gestionado por Axiom Core
> **Fecha**: 2026-08-29
> **Acciones**: ACC-065..ACC-067
> **Dependencias**: A-D

## Objetivo

Garantizar que setup, adopción y operaciones incrementales prevalidan sin escribir, calculan un estado deseado válido y aplican los archivos estructurales como una unidad recuperable, preservando contenido humano de `axiom.yaml`.

## Revalidación

Setup, repo add y role add deben rechazar una identidad `axiom.yaml` foránea. Adopt puede encontrar en el destino una identidad válida de otro proyecto y preservarla explícitamente como `skipped`/no-clobber; un documento inválido, ambiguo o conflictivo dentro del proyecto activo sigue siendo rechazo. La mutación estructural no puede degradar fallos de recursos requeridos a warnings/éxito ni re-renderizar el YAML humano completo.

## Alcance

- ACC-065: preflight común para workspace setup/adopt/repo add/role add, canonicalización, ownership y cero mutaciones ante rechazo.
- ACC-066: desired-state planner + coordinador con journal/staging y recuperación determinista para axiom.yaml, topology, bindings, registro solicitado y estado estructural.
- ACC-067: `axiom.yaml` gestionado por secciones, preservación byte a byte de contenido humano y fallo fail-closed si no es reconciliable.
- Resultados/envelopes tipados y exit codes honestos; outputs derivados quedan fuera del commit estructural y se reportan como warnings.

## No objetivos

- No duplicar locks/writers de A/C; el coordinador solo los compone.
- ACC-068 se implementa después reutilizando este preflight/catálogo.
- No rollback de archivos humanos ajenos ni de legacy sources.
- No seguridad general del launcher ni R-13.4/ACC-070..076.

## Decisiones cerradas

1. `planStructuralMutation` es pura respecto al filesystem: puede leer/stat/realpath, pero no crea dirs, locks, temporales, logs ni telemetría.
2. Preflight valida project/role/repo IDs, reservados `axiom|sdd|spec` para roles de code, unicidad global, ownership de paths, flags create y solapamientos bidireccionales entre targets escribibles y legacy/context sources. Setup/repo/role rechazan YAML foráneo; adopt preserva una identidad válida de otro proyecto como no-clobber explícito. YAML inválido, ambiguo o conflictivo con el proyecto activo siempre rechaza.
3. `ENOENT` es la única prueba de ausencia para planificación y canonicalización. `ENOTDIR`, `EACCES`, `EIO` y cualquier observación desconocida se convierten en error tipado con path, causa, operación y código; nunca autorizan un target inventado.
4. Ownership cubre todo output previsto: axiom.yaml, topology, bindings, registro, workspace state, AGENTS, config, MCP, skills/rules/adapters/process surfaces. El plan contiene paths exactos y owner esperado.
5. Tras preflight, un lock de proyecto serializa la operación. Locks de registry y recursos múltiples se adquieren en orden canónico, se deduplican por identidad física y reutilizan `@axiom/core`. Toda excepción durante adquisición parcial libera lo ya adquirido o devuelve cleanup/recovery diagnosticado.
6. Apply mantiene locks de proyecto, registry y recursos durante recovery, replan y comprobación de precondiciones. El conjunto de recursos no puede cambiar; bytes, mtime, tipo, realpath e identidad física observados deben seguir coincidiendo antes del journal.
7. Commit estructural usa journal persistido bajo `.axiom-state/<projectKey>/structural-transactions/<operationId>/`: intent, hashes previos, staging, estado y acciones de recuperación. Cada replace es atómico; fallo provoca rollback exacto o deja estado `recovery-required` visible, nunca éxito.
8. `--no-register` omite cualquier persistencia del registry, pero no desactiva el guard read-only de ownership. Apply puede adquirir un lock compartido temporal sobre `projects.yml`; al terminar o rechazar no deja registry, lock ni directorio home creado por el protocolo. Si el registro fue solicitado, cualquier fallo revierte la unidad.
9. Outputs derivados (adapters, docs generadas, caches) se ejecutan solo después del commit; fallos son warnings tipados y no revierten estructura válida. Su gate usa la identidad/membresía observada, no padres creados temporalmente por locks.
10. Resultados por recurso: `created|updated|unchanged|skipped`; warnings separados; `ok:false` y exit no cero para cualquier fallo estructural.
11. `axiom.yaml` contiene un bloque delimitado `AXIOM:MANAGED:START/END` con identidad, repoId/kind y puntero al axiomRepo. Reemplazar el bloque preserva exactamente prefijo/sufijo, comentarios y extensiones humanas.
12. Documento legacy actual puede convertirse una vez solo si el parser demuestra identidad coincidente y no hay campos conflictivos; schema topológico 1 no se conserva. Documento inválido/ambiguo falla en preflight.
13. Recuperación al iniciar inspecciona journals incompletos y hace rollback/roll-forward determinista antes de aceptar otra mutación. Errores de lectura/escritura conservan path, causa, clasificación y código; entradas no reconocidas se preservan y bloquean fail-closed.

## Riesgos

Crash entre replaces, transacciones cross-volume, deadlocks con home/user registry, contenido YAML ambiguo y symlinks/junctions. Staging debe residir junto a cada destino; el journal registra cada frontera y los tests inyectan fallo.

## Compatibilidad

No se preservan shapes topológicos v1. Solo se admite reconciliación controlada del `axiom.yaml` actual cuando identidad y ownership son inequívocos; no hay migración de instalaciones externas inexistentes.

## Validación prevista

Matriz de preflight y cero mutación; EACCES/EIO/ENOTDIR tipados; adquisición parcial y release de locks de apply/recovery; fault injection antes/después de staging, journal, write y rename para cada recurso publicable; crash/recovery/retry; dos writers realmente solapados; drift de bytes/mtime/tipo/realpath/identidad física; preservación byte a byte; JSON/exit codes; wrappers CLI/launcher; suites setup/adopt/incremental/YAML/MCP/steps; build/typecheck/doctor/readiness/index/diff-check.

## Resultado, revisión y validación

ACC-065..067 quedaron implementadas con observación fail-closed, preflight común,
locks/replan, journal por operación, staging adyacente, rollback/recovery y
reconciliación seccional de `axiom.yaml`. `ENOENT` es la única ausencia; los
fallos de observación preservan path/operación/código/causa. Los fallos derivados
post-commit conservan `state: committed`, recursos y `exitCode: 0` con warning
tipada. Dos intentos de `axiom-review` agotaron timeout; el fallback inline revisó
B1/B2/B3 y los P0 y concluyó GO sin blockers.

Evidencia final: incremental `42/42`; seis suites estructurales `121/121`;
setup/adopt `25/25`; YAML/MCP/steps/envelopes/bindings `32/32`; workspace command
`19/19`; app/launcher `53/53`; `npm run build`, doctor (`48/61`, 0 fallos),
readiness, `axiom index validate` y `git diff --check` en PASS.

## Integración estable

Integrada en `specs/00..08`, `context/TECHNICAL_CONTEXT.md`, arquitectura,
operaciones y el plan integral. Se sustituyeron directamente los claims activos
de warning estructural, bindings/workspace legacy y ausencia de planner/journal;
la auditoría previa queda marcada únicamente como historia fechada.
