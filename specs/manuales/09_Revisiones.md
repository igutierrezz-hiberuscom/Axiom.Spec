# 09. Revisiones

Cómo se revisa el trabajo antes de darlo por cerrado — write-scope, salud del proyecto, y el flujo `axiom-review`.

## Review de write-scope (al cierre de rol y agregado)

Todo `allowedWriteScope` declarado en un plan aprobado (ver [07_Planes.md](07_Planes.md)) se valida contra el `git diff` real mediante UNA misma primitiva compartida (`validateWriteScope`), en dos superficies complementarias:

- **Por rol, en `axiom-role complete`**: gate explícito que bloquea la finalización de ESE rol ante cualquier violación del scope permitido para su repo (ver [08_Implementacion.md](08_Implementacion.md)). Degrada a un SKIP explícito (nunca un crash) cuando el repo no es identificable o no hay plan aprobado local.
- **Agregado, desde el repo de spec**: `axiom validate changes --plan <id> --all-repos` resuelve cada repo destino del plan, diffea y valida cada uno, y emite un reporte consolidado per-repo (✓/✗) más los repos no resueltos, con exit 1 ante cualquier violación.

## Verificación funcional

`axiom-increment verify` / `axiom-bug verify` / `axiom-role complete` descubren y CORREN la validación propia del repo destino (test/build/lint/typecheck) y bloquean la transición ante un fallo real — no son solo un cambio de estado. `--no-verify`/`--force` saltan el gate deliberadamente; `--preview`/`--dry-run` solo reporta lo descubierto.

## El flujo `axiom-review`

`axiom-review` opera sobre un incremento, un bug, o los cambios actuales del working tree, y produce una recomendación de cierre acompañada de un **ledger de hallazgos** estructurado: cada hallazgo con `id` (`{LENS}-{NNN}`), `lens`, `location` (`file:line`), `severity` (BLOCKER/CRITICAL/WARNING/SUGGESTION) y `status` (open/fixed/verified/wont-fix/info). La primera pasada es exhaustiva (barre el scope hasta que dos barridos consecutivos no arrojen hallazgos nuevos); una re-review posterior solo verifica el ledger + el diff del fix, no relee todo el scope original. El ledger se persiste dentro de la carpeta del artefacto (`review-ledger.md`) cuando existe, o como memoria si no.

## El doctor como review de salud

El pre-launch gate de `axiom doctor` que corre el launcher antes de ejecutar/lanzar cualquier acción es, en esencia, una revisión de salud del proyecto (boundaries/policies/manifests/isolation/capability model/gateway) — ver [11_Launcher_Visual.md](11_Launcher_Visual.md). Es complementario, no un sustituto, del review de write-scope: uno valida "¿está sano el proyecto?", el otro valida "¿este cambio se mantuvo dentro de lo permitido?".

## Reglas de cierre (recordatorio)

Ningún incremento/bug se marca `closed` sin: goal claro, acceptance criteria, implementación (o justificación no-code), validación ejecutada, revisión contra el intent, y conocimiento estable integrado en la spec cuando aplica. Ver [05_Incrementos.md](05_Incrementos.md) / [06_Bugs.md](06_Bugs.md).

## Relacionado

- [07_Planes.md](07_Planes.md)
- [08_Implementacion.md](08_Implementacion.md)
- [10_Archivado.md](10_Archivado.md)
- [11_Launcher_Visual.md](11_Launcher_Visual.md)
