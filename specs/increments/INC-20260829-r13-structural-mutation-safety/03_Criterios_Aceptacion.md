# 03 Criterios de Aceptación

## Preflight

- **CA-E1 / ACC-065**: cada conflicto listado en la acción produce error tipado y comparación before/after idéntica, incluso ausencia de nuevos directorios/locks.
- Cubre symlink/junction/8.3/case-fold, espacios, role `sdd/spec`, path duplicado, nesting target/source, foreign/invalid YAML y reejecución legítima.

## Transacción

- **CA-E2 / ACC-066**: éxito aplica identity/topology/bindings/workspace state y registro solicitado con outcomes correctos.
- Fallo inyectado antes/después de cada write/rename revierte bytes o deja `recovery-required`; nunca exit 0.
- Crash simulado se recupera determinísticamente; retry es idempotente; dos escritores se serializan sin lost update/deadlock.
- `--no-register` no toca home; registro solicitado que falla revierte estructura.

## Preservación

- **CA-E3 / ACC-067**: comentarios, campos/extensiones y tail humano sobreviven byte a byte; solo cambia bloque gestionado.
- YAML inválido/markers ambiguos/identidad conflictiva no se reemplazan. Rollback no pisa una edición humana post-start detectada por hash.

## Evidencia

Suites setup/adopt/repo/role, planner/coordinator/recovery, wrappers CLI/launcher, build/typecheck y diff-check.
