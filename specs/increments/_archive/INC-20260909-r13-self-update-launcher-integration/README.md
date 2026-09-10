# r13-self-update-launcher-integration

> **Código**: INC-20260909-r13-self-update-launcher-integration
> **Estado**: En Borrador
> **Fecha de creación**: 2026-09-09
> **Tipo de cambio**: Integración funcional y UX del self-update global en el Launcher

## Resumen

Este incremento especifica la incorporación futura de las superficies de self-update de ACC-077 y ACC-078 al Launcher. Es el cuarto de cinco incrementos de la secuencia global R-13.5:

1. `INC-20260909-r13-self-update-release-identity`;
2. `INC-20260909-r13-self-update-state-cli-contract`;
3. `INC-20260909-r13-self-update-transactional-updater`;
4. **este incremento**, integración en Launcher;
5. `INC-20260909-r13-self-update-release-certification`.

El Launcher mostrará, sin confundirlas, las versiones publicada, descargada e instalada, junto con freshness, provenance y el estado tipado derivado. Al arrancar programará un refresh del remoto canónico best-effort, no bloqueante y acotado. El refresh podrá actualizar únicamente evidencia de descubrimiento y refs remotas autorizadas; nunca ejecutará `checkout`, `merge`, `pull`, instalación ni activación.

Las operaciones `status`, `check`, `plan`, `apply` y `recover` se expondrán mediante los mismos runners headless consumidos por la CLI. `apply` siempre partirá de un preview inmutable y exigirá confirmación explícita, single-use y vinculada al plan. La mutación se delegará a un helper externo para no reemplazar módulos cargados por el Launcher. Este artefacto especifica comportamiento futuro: **no afirma que el Launcher actual ya ofrezca self-update**.

## Contexto y motivación

R-13.4 dejó como baseline un control plane local con sesión por proceso, same-origin, schemas cerrados, límites de body/tiempo/SSE y grants de confirmación ligados a sesión, acción y payload. Esa baseline excluyó expresamente self-update.

Los incrementos 1–3 separan identidad de release, estado/contrato CLI y motor transaccional. La integración visual no debe reimplementar esas reglas ni introducir un segundo motor Git. Su responsabilidad es orquestar el runner compartido, representar su estado de forma honesta, proteger acciones mutantes y coordinar el handoff a un proceso externo con cierre/reinicio explícitos.

### Ownership del helper y gate G2

R13-3 entrega el runner headless, journal, eventos tipados y API pública de
progreso/cancelación. R13-4 es propietario de la integración del helper
externo y de su protocolo de claim/ACK/fencing, espera del PID del Launcher,
reattach, cierre y relaunch. Por tanto, G2 de este incremento queda en
`STOP` mientras esos contratos y su evidencia no estén implementados y
verificados aquí; no se presume que fueron entregados por R13-3.

La superficie es global a la instalación user-level de Axiom, no al proyecto seleccionado en el Launcher. El runtime project-scoped puede mostrarse como contexto adicional, pero nunca sustituye ni redefine `installedVersion`.

## Alcance

### Incluido

- Gate obligatorio de compatibilidad con los contratos y evidencia aceptada de los incrementos 1, 2 y 3.
- Carga inmediata de `status` local al abrir el Launcher, sin esperar red.
- `git fetch` del remoto canónico al iniciar como tarea best-effort, cancelable y con deadline; fallo o ausencia de red no impide arrancar ni usar capacidades no mutantes.
- Prohibición expresa de `checkout`, `merge`, `pull`, `reset`, `stash`, activación o instalación durante el arranque y los flujos `status`, `check` y `plan`.
- Presentación separada de `publishedVersion`, `downloadedVersion` e `installedVersion`, con timestamp, procedencia y freshness.
- Representación de los estados tipados de identidad: `updated`, `update-available`, `checkout-behind`, `ahead`, `installed-misaligned`, `unknown` y `offline`.
- Acciones `status`, `check`, `plan`, `apply` y `recover` sobre runners compartidos con CLI, sin comandos Git ni reglas de negocio duplicadas en HTTP o frontend.
- Preview de `UpdatePlan` sellado, confirmación explícita vinculada a instalación, operación, digest del plan, target y política de reinicio.
- Consumo atómico y single-use del grant antes de lanzar cualquier side effect.
- Implementación del helper externo y supervisado para apply/recover, incluido claim/ACK/fencing, espera del PID, reattach, cierre/reinicio y recovery, consumiendo el runner headless de R13-3.
- Elección explícita entre cerrar sin reiniciar o cerrar y reiniciar al finalizar; ninguna de las dos se infiere de forma silenciosa.
- Recuperación honesta de estado offline, stale, fallo de helper, plan invalidado, `updateOperationLock` ocupado y `recovery-required`.
- Reutilización de sesión, auth, Host/Origin, headers, schemas, límites y token de confirmación de R-13.4.
- Pruebas herméticas del control plane, frontend y handoff al helper con repositorios Git, homes y prefixes temporales, sin red externa.

### Excluido

- Implementar o modificar motor Git, resolución de release, manifest v2, `stateWriteLock`, `updateOperationLock`, staging, activación, rollback transaccional o journal; pertenecen a los incrementos 1–3, especialmente engine/operation lock al 3.
- Añadir `checkout`, `merge`, `pull`, `reset`, `stash` o clobber implícito como atajo de integración.
- Crear una segunda taxonomía, comparador SemVer, `UpdatePlan`, writer de estado o recovery distinto del compartido con CLI.
- Aplicar una actualización automáticamente al arrancar, tras un check, al detectar una release o al cerrar el navegador.
- Convertir el Launcher en servicio remoto, relajar loopback/same-origin o introducir tokens en URLs.
- Definir release CI, matriz final multiplataforma, packaging/publicación y documentación operativa final; corresponden al incremento 5.
- Declarar npm como canal soportado mientras los paquetes sigan privados.
- Cambiar metadata, índices, receipts o estado de lifecycle como parte de esta especificación.

## Documentos del incremento

- [`01_Requisitos.md`](./01_Requisitos.md): requisitos funcionales, de seguridad, no-mutación y paridad.
- [`02_Cambios_Modelo.md`](./02_Cambios_Modelo.md): view models, estados tipados, bindings y transiciones.
- [`03_Criterios_Aceptacion.md`](./03_Criterios_Aceptacion.md): criterios observables y evidencia exigida.
- [`04_Interacciones_UI.md`](./04_Interacciones_UI.md): comportamiento detallado de UI, confirmación, progreso, cancelación, reinicio y recovery.
- [`context/README.md`](./context/README.md): límites del contexto de trabajo local al incremento.
- Plan asociado: `PLAN-INC-20260909-r13-self-update-launcher-integration`.

## Dudas abiertas

No quedan dudas funcionales bloqueantes para planificar. Los nombres finales de módulos, símbolos y rutas HTTP son decisiones de implementación condicionadas por la API realmente entregada por el incremento 3. Si esa API headless, sus contratos de progreso/cancelación o su helper no existen o son incompatibles, el resultado es `STOP`: no se suplen con lógica local del Launcher.

## Decisiones funcionales cerradas

1. La integración ocupa la posición 4/5 y depende obligatoriamente de los incrementos 1, 2 y 3; habilita el 5.
2. La actualización es global user-level aunque el Launcher esté mostrando un proyecto.
3. El primer render usa estado local; el refresh remoto se agenda después y nunca bloquea el bootstrap.
4. Solo el remoto canónico validado por el incremento 1 puede aportar `publishedVersion`.
5. El refresh de arranque es best-effort: timeout/unavailable marca offline; cancelación conserva estado previo; provenance inválida produce unknown/error tipado. Ningún resultado es fatal ni inicia apply.
6. Ningún refresh ejecuta `checkout`, `merge` o `pull`; `status`, `check` y `plan` no cambian la instalación ni abren una transacción.
7. HTTP y UI son adaptadores del mismo runner headless que usa la CLI; no contienen un motor alternativo.
8. `apply` requiere plan inmutable más grant single-use vinculado. Cambiar target, payload, instalación o política de reinicio invalida la confirmación.
9. `recover` también es explícito, se basa en el journal y usa preview/grant; nunca se dispara solo al arrancar.
10. El helper externo espera el cierre del proceso que cargó los módulos antes de activar. La opción de relanzar es una elección del operador ligada al plan.
11. Cerrar la pestaña no equivale a cancelar. Una operación aceptada continúa según su journal y se reconcilia al volver a abrir el Launcher.
12. El incremento 5 es responsable de certificar la secuencia completa y publicar documentación final; este incremento aporta la superficie y sus pruebas focales.

## Consolidación en la spec general

Tras implementación, validación y review, el conocimiento estable deberá reconciliarse de forma concisa en sus owners canónicos candidatos:

| Tema estable | Owner candidato |
|---|---|
| Inventario global, estados y no-auto-apply | `specs/01_Requisitos_Funcionales.md` |
| Preview/confirm/apply/recover, handoff y restart | `specs/04_Flujos_SDD_y_Ciclo_de_Vida.md` |
| Superficie operativa del Launcher | `specs/05_Interfaces_Operativas.md` |
| Runner compartido y helper externo | `specs/06_Integraciones_y_Capacidades.md` |
| Auth/same-origin/grants/limits/fencing | `specs/07_Gobierno_y_Seguridad.md` |
| Límites arquitectónicos/operativos duraderos | `context/**` solo si emerge conocimiento estable nuevo |

Los owners se confirman durante el gate de integración; si un tema ya está cubierto sin cambio, se registra evidencia de “no requiere integración” en vez de duplicarlo. Esta redacción no modifica esos archivos ni copia historia de implementación. No se archivará ni cambiará el estado del incremento sin evidencia de los gates y la integración canónica aplicable.

## Estrategia E2E

La validación del incremento utilizará un servidor Launcher real sobre loopback, un remoto Git bare local que represente la autoridad canónica, dos releases firmadas por fixtures de identidad, clones/homes/prefixes temporales y el helper real detrás de seams inyectables. No se permitirá acceso a red externa.

La matriz cubrirá: bootstrap sin bloqueo, refresh exitoso/timeout/offline, evidencia separada por identidad, plan sin mutación, token válido/ausente/expirado/replay/mismatch, spawn pre-claim, ACK perdido/tardío e idempotencia por `operationId`, cancelación segura/no segura, cierre sin relanzar, cierre con relanzar, reanudación desde journal y recovery bifásico. Cada caso verificará snapshots de worktree, refs no autorizadas, manifest, shim y journal para demostrar qué podía y qué no podía mutar.

El incremento 5 consumirá estos resultados y ampliará la certificación final de release y plataformas; no se adelanta aquí ese cierre.

## Trazabilidad y fuentes

- ACC-077 / R-13.5-A: inventario, descubrimiento, estados tipados, freshness/provenance, offline y separación check/apply.
- ACC-078 / R-13.5-B: plan inmutable, apply transaccional, helper externo, cierre/reinicio y recovery.
- R-13.4 / ACC-070..ACC-076: baseline de control plane, auth, same-origin, schemas, límites, SSE y grants.
- `INC-20260909-r13-self-update-release-identity`: autoridad de release e identidades.
- `INC-20260909-r13-self-update-state-cli-contract`: estado persistente y gramática/runners CLI.
- `INC-20260909-r13-self-update-transactional-updater`: motor, helper, progreso, cancelación, journal, rollback y recovery.
- `PLAN-REVISION-INTEGRAL-AXIOM.md`: secuencia global y reparto de ACC-077..080.

## Estado de validación humana

La intención, alcance, UX y criterios quedan especificados para revisión humana. No hay implementación ni evidencia runtime asociada a este documento y el Launcher actual no debe considerarse capaz de self-update por la sola existencia de esta especificación. El estado de lifecycle se conserva sin cambios.
