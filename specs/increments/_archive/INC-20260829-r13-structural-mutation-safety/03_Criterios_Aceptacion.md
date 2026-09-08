# 03 Criterios de Aceptación

## Preflight

- **CA-E1 / ACC-065**: cada conflicto listado en la acción produce error tipado y comparación before/after idéntica, incluso ausencia de nuevos directorios/locks.
- Cubre symlink/junction/8.3/case-fold, espacios, role `sdd/spec`, path duplicado, nesting target/source, YAML inválido/ambiguo, identidad foránea y reejecución legítima.
- Setup/repo/role rechazan identidad foránea; adopt preserva byte a byte una identidad válida de otro proyecto como no-clobber explícito y rechaza conflicto dentro del proyecto activo.
- Solo `ENOENT` se trata como ausencia; `ENOTDIR`, `EACCES`, `EIO` y observaciones desconocidas devuelven path/cause/operation/classification/errorCode y no mutan.

## Transacción

- **CA-E2 / ACC-066**: éxito aplica identity/topology/bindings/workspace state y registro solicitado con outcomes correctos.
- Fallo inyectado antes/después de staging, journal, write y rename, para cada recurso publicable, revierte bytes o deja `recovery-required`; nunca exit 0.
- Crash simulado se recupera determinísticamente; retry es idempotente; dos writers realmente solapados se serializan sin lost update/deadlock.
- Toda adquisición parcial de locks de apply/recovery libera lo ya adquirido ante retorno o excepción; fallo de cleanup queda tipado y preserva artifacts.
- Replan y precondiciones se ejecutan bajo locks de proyecto/registry/recursos y detectan drift de bytes, mtime, tipo, realpath e identidad física.
- `--no-register` puede leer y bloquear temporalmente el registry para ownership/coordinación, pero no persiste ni deja home/registry/lock creados por el protocolo; registro solicitado que falla revierte estructura.

## Preservación

- **CA-E3 / ACC-067**: comentarios, campos/extensiones y tail humano sobreviven byte a byte; solo cambia bloque gestionado.
- YAML inválido/markers ambiguos/identidad conflictiva no se reemplazan. Rollback no pisa una edición humana post-start detectada por hash.

## Evidencia

Suites setup/adopt/repo/role, planner/coordinator/recovery, YAML/MCP/steps, wrappers CLI/launcher, matriz de faults y concurrencia real, build/typecheck/doctor/readiness/index y diff-check.
