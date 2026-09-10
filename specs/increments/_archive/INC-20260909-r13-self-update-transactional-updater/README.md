# r13-self-update-transactional-updater

> **Código**: INC-20260909-r13-self-update-transactional-updater
> **Estado**: En Borrador
> **Fecha de creación**: 2026-09-09
> **Tipo de cambio**: Incremento funcional y técnico de actualización global transaccional

## Resumen

Este incremento, tercero de cinco en la secuencia R13, especifica la sustitución del `apply` simulado por un actualizador Git global, transaccional y recuperable. Cubre ACC-078 y completa el comportamiento operativo de `apply` y `recover` requerido por ACC-079: selección exclusiva de una release publicada y validada, plan inmutable común a `preview` y `apply`, exclusión mutua global, preflight no destructivo, preparación y build aislados, verificación del candidato por rutas absolutas, activación atómica del único entry global, persistencia de la versión realmente observada y recuperación determinista ante interrupciones.

Este artefacto define comportamiento futuro y criterios de ejecución; no afirma que el runtime actual ya los implemente.

## Contexto y motivación

## Diagnóstico de cierre (2026-09-10)

El foco actual pasa, pero el incremento permanece en `verifying` y con cierre
pendiente. La revisión independiente conserva como evidencia histórica el
receipt `fd492abb0bc0859d4adf348bd9dc12bc0cf4567dc1556d62f802f82198f8e447`:
declaró un GO focal mientras dejó pendientes gates que no autorizan archive.
La ejecución focal vigente debe reportarse por suite: `packages/core/tests/self-update.test.ts` (35 tests), `apps/cli/tests/self-update-contract.test.ts` (13 tests) y `apps/cli/tests/self-update-status.test.ts` (28 tests), para un total de 76 tests cuando las tres pasan. La validación independiente actual pasa las tres suites, además de `npm run typecheck` y `npm run build`. El gate global sigue rojo con 28 archivos y 65 tests fallidos. El resultado "sin delta frente al baseline" no se considera un gate verde.

Los blockers técnicos reparados en esta pasada, aún pendientes de evidencia
multiplataforma/completa, fueron:

1. `FaultBoundary` tipado y hooks dentro de las operaciones reales de manifest/activation;
2. terminación acotada y diagnosticada del árbol padre/hijo;
3. política branch/upstream completa y obligatoria;
4. señal operativa separada de `recoverySignal`;
5. preflight no destructivo de paths, permisos, rename y herramientas;
6. cleanup seguro del sibling preparado y recovery conservador ante preflight fallido;
7. pruebas focales ampliadas para faults, Git, concurrencia y recovery.

La evidencia todavía pendiente no es un blocker funcional reproducido en estas
pruebas focales, sino el gate de cierre: ejecutar la matriz completa de faults,
permisos y recovery en Windows y POSIX, demostrar G8 con un consumidor fake que
use solo exports públicos, y resolver la baseline global fuera del diff
self-update.

Hasta que esos puntos, el baseline global y G8 estén verdes, el gate es
`STOP`, no se ejecuta `increment-archive` y los receipts anteriores se
conservan únicamente como historial.

Un `apply` que solo simula éxito puede dejar al operador con una versión declarada que no coincide con el binario, el commit, las dependencias o el build realmente activos. Una actualización global también amplía el riesgo: un fallo durante `fetch`, instalación, build, activación o escritura del manifest puede inutilizar el único comando `axiom` disponible. Por ello, el cambio debe tratar release, checkout/candidato, dependencias, build, entry activo y manifest como una sola transacción observable y recuperable.

La secuencia global es obligatoria: **1 → 2 → 3 → 4 → 5**.

- Los incrementos 1 y 2 de R13 son precondiciones obligatorias. Deben aportar y validar los contratos de instalación global, versión/release y descubrimiento que este incremento consume; si su evidencia no está aceptada, el gate es `STOP`.
- Este incremento es el **3/5** y no absorbe trabajo de sus dependencias.
- El incremento 4 consumirá la API headless de runner, journal y eventos aquí definida para el Launcher y será propietario del helper externo, claim/ACK/fencing, espera del PID, reattach y relaunch.
- El incremento 5 consumirá los resultados y evidencias de este incremento para cerrar el pipeline final de release y la documentación, sin que este incremento los implemente.

## Alcance

### Incluido

- Resolver el target únicamente desde una release publicada cuya identidad, versión, ref/tag y commit hayan sido validados mediante los contratos de los incrementos 1 y 2.
- Crear un `UpdatePlan` canónico, sellado e inmutable que `preview` devuelve y `apply` consume sin recalcular ni sustituir silenciosamente el target.
- Adquirir un lock global por identidad de instalación antes de cualquier mutación y aplicar el mismo lock a `apply` y `recover`.
- Revalidar, bajo lock, remoto, upstream, branch, tag/ref, commit, limpieza del checkout, estado detached y relación de fast-forward/divergencia.
- Rechazar estados dirty, detached, ahead/diverged, remotos inesperados, refs móviles o commits distintos sin ejecutar `stash`, `reset`, merge, checkout destructivo ni sobrescritura de trabajo del usuario.
- Preparar el target mediante Git con semántica fast-forward verificable en un espacio gestionado o, como estrategia preferente cuando la reversibilidad lo exija, en staging seguro y aislado.
- Ejecutar `npm ci` exclusivamente desde el lockfile versionado del commit target y después el build completo; no se acepta `npm install` ni reutilización opaca de dependencias.
- Verificar el candidato invocando rutas absolutas: `--version` con igualdad exacta respecto de la release esperada, `--help`, carga del entry y smoke/doctor.
- Activar de forma atómica el único shim/binlink autorizado, sin borrarlo primero ni crear un segundo entry competitivo.
- Escribir de forma atómica el manifest de instalación y persistir como versión instalada únicamente la salida observada y validada de `--version`.
- Mantener journal durable por fases y rollback de commit/release candidato, dependencias/build, entry activo y bytes previos del manifest.
- Implementar recuperación idempotente sin borrado ciego, con inspección de realidad y ownership antes de completar rollback o reconocer un commit.
- Producir exclusivamente los outcomes `installed`, `unchanged`, `failed` y `recovery-required`, con diagnóstico estructurado de timeout, signal, spawn y exit no cero.
- Unificar semántica de paths y operaciones atómicas para Windows y POSIX.
- Separar explícitamente instalación inicial de actualización: un entorno sin instalación global gestionada no se convierte implícitamente en instalación durante `apply`.
- Exponer runner headless, journal, eventos de progreso/cancelación y API pública para consumo posterior por CLI y Launcher. Los helpers internos del runner pueden permanecer privados; el helper externo de handoff pertenece al incremento 4.

### Excluido

- La UI, navegación, botones, auto-relaunch o experiencia visual del Launcher, que pertenecen al incremento 4.
- El pipeline final de publicación, firma/promoción de releases y la documentación final para operadores, que pertenecen al incremento 5.
- Crear una instalación global desde cero, migrar instalaciones no gestionadas o adoptar directorios arbitrarios del usuario.
- Autorreparar checkouts dirty/detached/diverged, hacer `stash`, `reset`, merge, force checkout, force push o eliminar contenido no demostrado como propiedad de la transacción.
- Cambiar la política canónica de publicación de releases definida por los incrementos 1 y 2.
- Ejecutar ahora implementación, transición de estado, archivado o consolidación fuera de los dos directorios autorizados.

## Documentos del incremento

- [01_Requisitos.md](./01_Requisitos.md): requisitos normativos y reglas de negocio.
- [02_Cambios_Modelo.md](./02_Cambios_Modelo.md): contratos, estados, journal, lock, activación y recovery.
- [03_Criterios_Aceptacion.md](./03_Criterios_Aceptacion.md): escenarios verificables para ACC-078 y ACC-079.
- [04_Interacciones_UI.md](./04_Interacciones_UI.md): superficie headless/CLI y frontera explícita con Launcher.
- [context/README.md](./context/README.md): reglas para evidencia contextual local del incremento.
- [Plan de implementación](../../plans/PLAN-INC-20260909-r13-self-update-transactional-updater/README.md): fases, tareas, gates y rollback de implementación.

## Dudas abiertas

No quedan dudas funcionales que permitan degradar las garantías de este incremento. Durante el inventario inicial de implementación deberán fijarse las rutas reales de los módulos existentes, los timeouts por comando y la primitiva atómica soportada por cada plataforma. Esas decisiones son de adaptación técnica: no pueden relajar la exclusión mutua, la inmutabilidad del plan, la verificación absoluta, el rollback ni la prohibición de borrado destructivo. Si una plataforma no ofrece una activación atómica demostrable para el entry configurado, el resultado debe ser `failed` antes de activarlo y el gate permanece `STOP`.

## Decisiones funcionales cerradas

1. El target no se acepta desde texto libre, una branch móvil sin evidencia de release o una versión solicitada por la UI; se deriva de una release publicada y validada.
2. `preview` y `apply` comparten exactamente el mismo plan serializado y digest. `apply` revalida precondiciones dinámicas, pero no muta el plan ni cambia de target.
3. Solo una operación `apply` o `recover` puede poseer el lock de una instalación global. La edad del lock, por sí sola, nunca autoriza borrarlo.
4. Todo estado Git que pueda contener intención local no representada por la release provoca `STOP`; el actualizador no intenta repararlo.
5. El candidato se prepara fuera del artefacto activo. Una ruta in-place solo sería admisible si demuestra las mismas garantías sin `reset` ni clobber; en ausencia de esa prueba se usa staging seguro.
6. El build candidato debe superar todas las verificaciones antes de tocar el entry activo.
7. La activación y el manifest son parte de la misma transacción. Un fallo entre ambos obliga a rollback; si no puede probarse el estado restaurado, se devuelve `recovery-required`.
8. `installed` solo se emite tras commit durable y verificación del entry activo; `unchanged` exige coincidencia y salud observadas; nunca se infiere éxito de una intención o de un manifest escrito.
9. La versión instalada persistida procede únicamente de la ejecución absoluta de `--version` y debe coincidir exactamente con la versión de la release validada.
10. La instalación inicial es otro flujo. El updater rechaza la ausencia de identidad/manifest gestionados con un diagnóstico accionable.
11. La recuperación observa filesystem, entry, commit y manifest, contrasta el journal y solo elimina paths con ownership, contención y transaction id demostrados.
12. La API de runner, journal y eventos es estable y desacoplada de UI para habilitar el incremento 4. El helper externo y el protocolo de handoff no forman parte de esta entrega.

## Consolidación en la spec general

Una vez implementado, validado y revisado el incremento, el conocimiento estable deberá integrarse mediante el flujo Axiom/Core correspondiente, no mediante edición incidental en este trabajo:

- ciclo transaccional `preview/apply/recover` y outcomes en la especificación operativa;
- contrato de comando/API headless y estados observables en interfaces operativas;
- validación de release, Git, build y activación multiplataforma en integraciones/capacidades;
- reglas de seguridad de paths, locks, journals y no clobber en gobierno y seguridad.

No se copia el historial de implementación. Se consolidan únicamente invariantes comprobadas y contratos aceptados. Este encargo no modifica esas specs generales ni cambia el estado del incremento.

## Estrategia E2E

La evidencia E2E se construirá con repositorios Git locales reproducibles: un bare remote, una instalación gestionada, tags/releases fixture y candidatos con lockfile. La matriz cubrirá actualización exitosa, target ya activo, dos updaters concurrentes, dirty, detached, ahead/diverged, remote/ref/commit inválidos, fallo de `npm ci`, fallo de build, mismatch de versión, fallo de `--help`/load/doctor, fallo de manifest, interrupción por timeout/signal/spawn, rollback y retry. Cada frontera de fase tendrá fault injection antes y después de cualquier mutación.

Los mismos contratos se ejecutarán en Windows y POSIX, incluyendo paths con espacios, normalización de separadores/case, symlinks o reparse points, reemplazo atómico y procesos hijos. Ninguna prueba E2E dependerá de modificar un repositorio remoto real ni del checkout de desarrollo del operador.

## Trazabilidad y fuentes

- Objetivo de producto: **ACC-078**.
- Cierre operativo incluido: ramas `apply` y `recover` de **ACC-079**.
- Dependencias obligatorias: incrementos **1/5** y **2/5** de R13.
- Dependientes habilitados: incrementos **4/5** y **5/5** de R13.
- Orden canónico: **1 → 2 → 3 → 4 → 5**.
- Fuente de intención: solicitud de planificación del actualizador global transaccional del 2026-09-09.
- Fuente técnica durante implementación: contratos aceptados de los incrementos 1 y 2, código real del runtime Axiom y evidencia automatizada descrita en el plan.

## Estado de validación humana

El contenido queda preparado para revisión humana contra ACC-078/079 y las dependencias R13. No se declara implementación ni ejecución de pruebas de runtime en esta fase documental. La revisión debe confirmar el gate `GO` antes de comenzar código; cualquier ausencia de evidencia de los incrementos 1 y 2, estrategia atómica multiplataforma o recovery determinista mantiene `STOP`. El campo de estado del artefacto permanece sin cambios.
