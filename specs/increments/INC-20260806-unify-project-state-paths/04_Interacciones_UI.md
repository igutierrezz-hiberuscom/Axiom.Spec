# 04 Interacciones UI

## Objetivo del documento

Describir lo que un operador ve al ejecutar comandos que leen o migran estado.

## Superficie UI afectada

CLI, doctor, readiness, toolchain show/validate/repair, rollback y
provisioning de worktrees. No se añade una pantalla nueva.

## Flujo de interacción

Los comandos resuelven el proyecto, derivan `projectKey`, consultan el
namespace canónico y solo si falta consultan aliases legacy. Una migración
exitosa continúa la operación desde el destino canónico.

## Estados visibles

Las salidas pueden informar que se encontró o migró estado legacy y pueden
emitir warnings de conflicto. Las rutas nuevas se muestran como
`.axiom-state/<projectKey>/...`; no se presenta `config/<projectName>` como
destino vigente.

## Cascadas y comportamiento reactivo

Restore y repair son idempotentes. El provisioning de worktree no vuelve a
copiar datos locales ni selecciona providers de otro proyecto.
