# 03 Criterios de Aceptación

## Criterios de aceptación

- [x] `ProjectMode` solo contiene `local-only`.
- [x] v1 y v2 normalizan `gateway` y `hybrid` a `local-only`.
- [x] No quedan consumidores activos que ramifiquen sobre esos modos del
	resolver.
- [x] Los modos de topología, worktree y routing permanecen intactos.
- [x] Build, doctor, readiness y pruebas focalizadas pasan.
- [x] La revisión independiente se registra en `review-ledger.md` y sus
	hallazgos quedan resueltos o clasificados como no aplicables.

### Happy path

Un `axiom.yaml` válido se resuelve con `status: resolved`, `mode: local-only`
y el mismo comportamiento efectivo tanto si el documento omite el modo como
si trae `gateway` o `hybrid`.

### Validaciones y errores

El schema puede aceptar la forma raw legacy para permitir migración. El
resolver no debe lanzar ni devolver un modo remoto por esa entrada. El build
debe exponer una declaración TypeScript cerrada.

### Permisos y visibilidad

No se crean permisos, providers, endpoints ni superficies UI nuevas. Doctor
solo muestra la política local-only efectiva.

### Estados y efectos observables

La única diferencia observable es que los consumidores tipados ya no pueden
tratar `gateway` o `hybrid` como estados activos. `rawConfig` permanece
disponible para los consumidores estructurales legacy.
