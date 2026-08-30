# Seguridad transaccional de mutaciones estructurales R-13

> **Código**: INC-20260829-r13-structural-mutation-safety
> **Estado documental**: especificación refinada; lifecycle gestionado por Axiom Core
> **Fecha**: 2026-08-29
> **Acciones**: ACC-065..ACC-067
> **Dependencias**: A-D

## Objetivo

Garantizar que setup, adopción y operaciones incrementales prevalidan sin escribir, calculan un estado deseado válido y aplican los archivos estructurales como una unidad recuperable, preservando contenido humano de `axiom.yaml`.

## Revalidación

Los guards actuales pueden preservar un `axiom.yaml` foráneo pero continuar escribiendo otros outputs. Setup/repo add escriben por etapas y degradan fallos estructurales a warnings/éxito. `writeOneRepo` re-renderiza el YAML completo y pierde comentarios/extensiones.

## Alcance

- ACC-065: preflight común para workspace setup/adopt/repo add/role add, canonicalización, ownership y cero mutaciones ante rechazo.
- ACC-066: desired-state planner + coordinador con journal/staging y recuperación determinista para axiom.yaml, topology, bindings, registro solicitado y estado estructural.
- ACC-067: `axiom.yaml` gestionado por secciones, preservación byte a byte de contenido humano y fallo fail-closed si no es reconciliable.
- Resultados/envelopes tipados y exit codes honestos; outputs derivados quedan fuera del commit estructural y se reportan como warnings.

## No objetivos

- No duplicar locks/writers de A/C; el coordinador solo los compone.
- ACC-068 se implementa después en I reutilizando este preflight/catálogo.
- No rollback de archivos humanos ajenos ni de legacy sources.
- No seguridad launcher ni R-13.5.

## Decisiones cerradas

1. `planStructuralMutation` es pura respecto al filesystem: puede leer/stat/realpath, pero no crea dirs, locks, temporales, logs ni telemetría.
2. Preflight valida project/role/repo IDs, reservados `axiom|sdd|spec` para roles de code, unicidad global, ownership de paths, foreign/invalid axiom.yaml, flags create y solapamientos bidireccionales entre targets escribibles y legacy/context sources.
3. Ownership cubre todo output previsto: axiom.yaml, topology, bindings, registro, workspace state, AGENTS, config, MCP, skills/rules/adapters/process surfaces. El plan contiene paths exactos y owner esperado.
4. Tras preflight, un lock de proyecto serializa la operación. Locks de recursos múltiples se adquieren en orden canónico para evitar deadlock y reutilizan `@axiom/core`.
5. Commit estructural usa journal persistido bajo `.axiom-state/<projectKey>/structural-transactions/<operationId>/`: intent, hashes previos, staging, estado y acciones de recuperación. Cada replace es atómico; fallo provoca rollback exacto o deja estado `recovery-required` visible, nunca éxito.
6. Registro omitido por `--no-register` es una decisión explícita. Si fue solicitado, cualquier fallo de registro revierte la unidad.
7. Outputs derivados (adapters, docs generadas, caches) se ejecutan solo después del commit; fallos son warnings tipados y no revierten estructura válida.
8. Resultados por recurso: `created|updated|unchanged|skipped`; warnings separados; `ok:false` y exit no cero para cualquier fallo estructural.
9. `axiom.yaml` contiene un bloque delimitado `AXIOM:MANAGED:START/END` con identidad, repoId/kind y puntero al axiomRepo. Reemplazar el bloque preserva exactamente prefijo/sufijo, comentarios y extensiones humanas.
10. Documento legacy actual puede convertirse una vez solo si el parser demuestra identidad coincidente y no hay campos conflictivos; schema topológico 1 no se conserva. Documento inválido/ambiguo falla en preflight.
11. Recuperación al iniciar inspecciona journals incompletos y hace rollback/roll-forward determinista antes de aceptar otra mutación.

## Riesgos

Crash entre replaces, transacciones cross-volume, deadlocks con home/user registry, contenido YAML ambiguo y symlinks/junctions. Staging debe residir junto a cada destino; el journal registra cada frontera y los tests inyectan fallo.

## Compatibilidad

No se preservan shapes topológicos v1. Solo se admite reconciliación controlada del `axiom.yaml` actual cuando identidad y ownership son inequívocos; no hay migración de instalaciones externas inexistentes.

## Validación prevista

Matriz de preflight y cero mutación, fault injection en cada frontera, crash/recovery/retry/concurrencia, preservación byte a byte, JSON/exit codes, suites setup/adopt/incremental y build/diff-check.

## Integración estable

Diferida al final; el worker no modifica `specs/00..08` ni `context/**`.
