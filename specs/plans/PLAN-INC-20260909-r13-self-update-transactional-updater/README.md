# Plan del actualizador global transaccional y recuperable

> **Código**: PLAN-INC-20260909-r13-self-update-transactional-updater
> **Estado**: draft
> **Artefacto origen**: [INC-20260909-r13-self-update-transactional-updater](../../increments/INC-20260909-r13-self-update-transactional-updater/README.md)
> **Versión de spec**: v1
> **Versión de plan**: p1

## Resumen ejecutivo

El plan implementará el incremento 3/5 de R13 para reemplazar el `apply` simulado por un actualizador Git global que solo active releases publicadas y validadas. La solución se organiza alrededor de un plan inmutable compartido por preview/apply, lock global, preflight fail-closed, staging gestionado, `npm ci` + build completo, verificación por rutas absolutas, swap atómico del único entry, manifest basado en versión observada y journal recuperable.

La prioridad es proteger la única instalación global y cualquier intención Git local. Ninguna rama ejecuta stash/reset/clean/clobber ni elimina paths por edad o nombre. Un fallo se clasifica como `failed` solo cuando puede probarse que el estado previo sigue sano; cualquier ambigüedad termina `recovery-required`.

Este plan describe trabajo y validación futuros. No afirma implementación actual ni cambia estado, metadata, receipts o índices.

## Objetivo técnico

Entregar un core headless y adapters de CLI/plataforma que satisfagan ACC-078 y cierren `apply/recover` de ACC-079 con estas invariantes:

1. **Origen legítimo:** target = release publicada validada → ref/tag exacto → commit exacto.
2. **Fidelidad:** apply consume el mismo payload/digest producido por preview.
3. **Exclusión:** un solo apply/recover por instalación global.
4. **No clobber:** estados Git o paths ambiguos bloquean; el updater no “arregla” intención local.
5. **Preparación aislada:** fetch/staging, dependencias y build ocurren antes de tocar el entry.
6. **Observación:** candidato y entry activo se verifican por rutas absolutas; solo la versión observada se persiste como instalada.
7. **Commit transaccional:** entry + manifest + journal determinan `installed`; cualquier fallo anterior revierte o requiere recovery.
8. **Paridad:** Windows y POSIX comparten contratos y difieren solo en adapters atómicos/path/process.
9. **Separación:** instalación inicial no es una rama implícita de self-update.
10. **Extensibilidad controlada:** el Launcher 4 consume runner/helpers; el pipeline/docs 5 consume outcomes/evidencia, sin implementarse aquí.

## Estrategia de implementación por fases

### Fase 0 — Readiness y baseline

- Probar que los incrementos 1 y 2 están aceptados y localizar sus contratos de instalación, manifest y release.
- Identificar el símbolo que hoy ejecuta/simula apply, el único entry global y las convenciones de tests/build del repo Axiom.
- Ejecutar build y tests baseline; clasificar fallos preexistentes.
- Producir el mapeo de rutas reales frente al mapa previsto de `role-builder.md`.
- Gate: `STOP` ante dependencia no aceptada, entry múltiple/ambiguo o falta de primitiva atómica viable.

### Fase 1 — Contratos, paths y procesos

- Añadir tipos cerrados de plan, snapshot, outcome, diagnósticos y eventos.
- Implementar canonicalización/ownership de paths y process runner sin shell con timeout/signal/spawn tipados.
- Separar por contrato `InitialInstaller` de `GlobalSelfUpdater`.
- Validar en unitarias Windows/POSIX antes de tocar Git.

### Fase 2 — Resolver target y sellar preview

- Adaptar el proveedor de releases de los incrementos 1 y 2.
- Resolver release/ref/commit, observar instalación/Git y construir serialización canónica + digest.
- Implementar revalidación que compara observaciones sin mutar el plan.
- Probar tampering, drift y target no publicado.

### Fase 3 — Lock, journal y preflight

- Implementar lock global con creación exclusiva, ownership y reclaim conservador.
- Implementar journal atómico, monotónico, con checksum y snapshots previos.
- Añadir preflight de remoto, branch/upstream, tag/ref, commit, dirty, detached, ahead/diverged y ancestor.
- Ejecutar pruebas con dos updaters y repos Git locales antes de habilitar staging.

### Fase 4 — Fetch y staging seguro

- Ejecutar fetch no interactivo del remoto autorizado y verificar ref → commit.
- Crear candidato bajo una raíz gestionada, con marker/transaction id y protección frente a symlink/reparse escapes.
- Materializar el commit exacto sin modificar el checkout activo. Si una ruta ff-only in-place no demuestra reversibilidad sin reset/clobber, queda descartada y se usa staging.
- Registrar ownership y fingerprints en journal.

### Fase 5 — Dependencias, build y verificación candidata

- Exigir lockfile y ejecutar `npm ci` en cwd absoluto del candidato.
- Ejecutar build completo, sin reutilizar outputs previos.
- Verificar por ruta absoluta `--version` exacta, `--help`, load/import y smoke/doctor.
- Inyectar spawn, timeout, signal, exit no cero y mismatch en cada frontera.

### Fase 6 — Activación, manifest y commit

- Snapshotear entry y manifest previos.
- Preparar un sibling del único entry y ejecutar reemplazo atómico específico de plataforma sin unlink-first.
- Verificar de nuevo el entry activo por ruta absoluta.
- Persistir manifest por temp + flush + replace usando la versión observada, no la deseada.
- Escribir `committed` durable y solo entonces devolver `installed`.

### Fase 7 — Rollback y recovery

- Implementar rollback inverso de manifest, entry y candidato (release/commit + dependencias/build), con prueba del estado restaurado.
- Implementar recovery bajo el mismo lock, basado en journal + realidad, idempotente y sin promoción silenciosa de transacciones no comprometidas.
- Rechazar cleanup sin marker, contención, transaction id y no-actividad demostrados.
- Cubrir interrupciones en todas las fronteras y retry con plan nuevo.

### Fase 8 — Integración, revisión y handoff

- Sustituir el falso apply en el adapter existente por el runner; ningún fallback puede seguir declarando éxito simulado.
- Exportar API headless y demostrar un fake consumer sin dependencias de Launcher.
- Ejecutar matriz completa, revisión semántica independiente, blast-radius review y comandos de validación.
- Preparar evidencia para integrar conocimiento estable y habilitar 4/5 y 5/5, manteniendo fuera de alcance UI y pipeline final.

## Alcance incluido

- Core transaccional de preview/apply/recover y composición `runGlobalSelfUpdate`.
- Adapters para release validada, Git, filesystem/path/atomic replace y procesos.
- Lock, journal, manifest observado y máquina de rollback/recovery.
- Sustitución del apply simulado en el punto de entrada existente, sin crear un comando paralelo.
- Tests unitarios, integración y E2E con Git local, fault injection y concurrencia.
- Validación Windows/POSIX y paths con espacios/escapes.
- Contrato público mínimo para el consumidor Launcher posterior.
- Evidencia de review, rollback de implementación e integración estable propuesta.

## Alcance excluido

- Instalación inicial o migración/adopción de instalaciones no gestionadas.
- Vistas y comportamiento del Launcher, auto-relaunch o UX visual del incremento 4.
- Pipeline final de release, promoción, publicación, firma y documentación final del incremento 5.
- Cambios al proveedor/política canónica de releases más allá de consumir su contrato aceptado.
- Reparación automática de dirty/detached/diverged, stash/reset/clean/merge/rebase.
- Actualización de repositorios de proyectos del usuario.
- Transiciones Core, cambios de estado, metadata, receipts o índices durante la edición documental de este plan.

## Roles impactados

- **builder:** implementa contratos, adapters, runner, wiring y tests en el repo runtime Axiom siguiendo `role-builder.md`.
- **reviewer/validator independiente:** reconstruye comportamiento, ejecuta gates y compara evidencia con ACC-078/079; no confía solo en happy path.
- **mantenedor de release/instalación (dependencias 1 y 2):** confirma contratos consumidos antes de `GO` sin ampliar su alcance.
- **consumidor Launcher (incremento 4):** recibe API/eventos; no participa en esta implementación.
- **responsable de release/docs (incremento 5):** recibe outcomes y evidence packet una vez aceptado el incremento 3.

Solo se materializa el archivo de rol builder en este plan. Las revisiones y aprobaciones son gates, no una autorización para crear artifacts estructurales manualmente.

## Estrategia E2E

### Harness Git local

Cada caso crea dentro del temp del test:

1. bare remote local;
2. repositorio publicador con commits, lockfile y tags/releases fixture;
3. instalación global gestionada con entry/manifest conocidos;
4. staging, journal y lock aislados;
5. ejecutables fixture para npm/build/CLI cuando la prueba no requiera el toolchain real.

No se usa el checkout de desarrollo ni red pública. Los tests capturan hashes de todo path fuera de staging para demostrar no clobber.

### Matriz mínima

- install target normal y target ya activo;
- remote/ref/commit inválidos o tag movido;
- dirty tracked/staged/untracked, detached, branch/upstream inesperados, ahead/diverged/unrelated;
- dos updaters y lock incierto;
- lockfile ausente, `npm ci` y build con spawn/timeout/signal/exit;
- version mismatch, fallos de help/load/doctor;
- atomic replace antes/durante/después del swap;
- manifest temp/write/flush/replace failure;
- interrupción en cada transición de journal;
- rollback íntegro, rollback interrumpido, recover repetido y retry con nuevo plan;
- cleanup con marker válido, marker ajeno, path activo y escape;
- Windows y POSIX, paths con espacios, case/drive/UNC/reparse o symlink/permisos según plataforma.

### Oráculos

Cada test valida outcome, fase terminal, orden de eventos, entry absoluto, versión observada, bytes del manifest, HEAD/status de clones, journal/checksum, locks y fingerprints de paths no gestionados. No basta comprobar el mensaje CLI.

## Riesgos y dependencias

| Riesgo/dependencia | Impacto | Mitigación / gate |
|---|---|---|
| Incrementos 1 o 2 no aceptados o contrato ambiguo | target/identidad no confiables | `STOP`; no duplicar ni inferir contratos |
| Primitiva de swap no atómica en Windows/POSIX | entry ausente o parcial | adapter probado; `STOP` antes del swap si no se demuestra |
| Drift entre preview y apply | se instala algo no aprobado | digest + revalidación bajo lock; exigir preview nuevo |
| Tag/ref mutable o remote spoofing | commit incorrecto | resolver dos veces y comparar remote/ref/commit exactos |
| Git local dirty/detached/diverged | pérdida de trabajo | fail-closed; jamás stash/reset/clean |
| Dos updaters | corrupción de journal/entry | lock único, timeout y test concurrente |
| Proceso hijo no termina | lock retenido/estado ambiguo | process-tree termination acotada; rollback/recovery-required |
| Build válido pero binario equivocado | manifest falso | verificación por ruta absoluta y versión exacta pre/post swap |
| Fallo entre entry y manifest | instalación incoherente | journal + snapshot + rollback inverso |
| Journal/manifest corrupto | recovery inseguro | schema/checksum, observación de realidad y no borrado |
| Escape por symlink/reparse | borrado fuera de raíz | realpath/ownership/marker/no-activity antes de cleanup |
| Wiring conserva fallback simulado | falso `installed` | test de contrato y búsqueda/review del antiguo path |
| Tests dependientes de red/host | flakiness y riesgo | repos Git locales y temp roots aislados |

## Blast radius

El blast radius intencional de runtime queda limitado a:

- una identidad de instalación global;
- su raíz gestionada de releases/staging;
- un lock y un journal por operación;
- el único entry global autorizado;
- su manifest;
- fetch de un remoto de release autorizado;
- procesos `npm ci`, build y verificaciones dentro del candidato.

No se modifican workspaces, repositorios de usuario, configuración Git global, PATH del sistema, más de un entry, instalaciones iniciales ni artifacts de Launcher/pipeline. Los tests deben probar esa frontera con fingerprints externos. Cualquier necesidad de ampliar el radio exige `STOP` y revisión de alcance.

## Rollback de implementación

Este apartado es distinto del rollback transaccional en runtime.

1. **Antes de publicar:** los cambios se organizan en commits focalizados por contratos, mecánica transaccional y wiring. Un fallo de review se revierte en Git sin tocar artifacts de instalación reales.
2. **Después de integrar pero antes de release:** revertir el wiring debe dejar self-update explícitamente deshabilitado con outcome/error no exitoso; nunca restaurar el falso apply como éxito.
3. **Después de que una versión haya emitido journals/manifests:** cualquier reversión debe conservar lectura diagnóstica de schema v1 y una ruta de recovery compatible. Si no puede hacerlo, se prepara una corrección forward; no se eliminan journals/manifests manualmente.
4. **Instalaciones ya activadas:** no se fuerzan a volver de versión como rollback de código. Se verifica la versión activa y se usa el protocolo transaccional/release permitido en una operación separada.
5. **Criterio de activación del rollback:** regresión que pueda romper el entry global, falsear versión, clobber Git o impedir recovery. El responsable registra evidencia, detiene nuevos apply y preserva journals.

El pipeline para publicar una corrección pertenece al incremento 5; este plan solo exige que el código sea reversible y que un rollback no reintroduzca falso éxito.

## Cambios en contexto técnico

Durante implementación puede generarse, mediante el flujo permitido, evidencia local de:

- mapeo de responsabilidades a archivos/símbolos reales;
- matriz de primitivas atómicas y paths por plataforma;
- matriz de fault injection con outcomes y snapshots;
- revisión independiente y fallos baseline.

No se crearán índices paralelos ni se copiará la especificación. El directorio `context/` de este plan define qué evidencia es admisible. Cualquier documento estable se consolida después de la aceptación, no durante la codificación inicial.

## Consolidación y archivado

Tras obtener `GO` técnico y revisión humana:

1. actualizar primero el propio incremento mediante Axiom/Core, sin editar status manualmente;
2. integrar solo invariantes aceptadas en las specs generales propietarias (flujo operativo, interfaces, integraciones y seguridad);
3. registrar la API que habilita al incremento 4 y la evidencia que habilita al 5;
4. evitar copiar tareas, logs o historial de fallos a la spec canónica;
5. archivar/cerrar únicamente cuando todas las reglas de cierre se satisfagan y mediante comandos Axiom/Core.

Esta tarea documental no ejecuta consolidación, archivado ni transición.

## Validaciones y gates

### Comandos previstos en el repo Axiom

```text
npm run build
npx vitest run packages/core/src/self-update
npx vitest run apps/cli
npx vitest run
```

Si el inventario de Fase 0 demuestra que el paquete o configuración usa rutas de test distintas, se ejecutará el comando equivalente ya existente y se registrará el mapping; no se crearán scripts ficticios solo para satisfacer el plan.

### Smokes absolutos del harness

POSIX:

```text
"$AXIOM_ABSOLUTE_BIN" --version
"$AXIOM_ABSOLUTE_BIN" --help
"$AXIOM_ABSOLUTE_BIN" doctor
node -e "import(process.env.AXIOM_ABSOLUTE_ENTRY_URL).then(() => process.exit(0))"
```

PowerShell:

```text
& $env:AXIOM_ABSOLUTE_BIN --version
& $env:AXIOM_ABSOLUTE_BIN --help
& $env:AXIOM_ABSOLUTE_BIN doctor
node -e "import(process.env.AXIOM_ABSOLUTE_ENTRY_URL).then(() => process.exit(0))"
```

El harness asigna variables a rutas absolutas del candidato o entry activo; no usa PATH para resolver `axiom`.

### Gates STOP/GO

| Gate | GO exige | STOP inmediato |
|---|---|---|
| G0 Readiness | incrementos 1/2 aceptados, baseline y mapping real | dependencia o entry ambiguos |
| G1 Contratos | tipos/outcomes cerrados, paths/process tests | fallback a instalación inicial o shell strings |
| G2 Plan/preflight | plan tamper-proof y matriz Git local | target no publicado o cualquier clobber |
| G3 Transacción | lock/journal/staging y dos updaters | lock reclaim por edad o journal no durable |
| G4 Build/verify | npm ci, build y smokes absolutos | versión no exacta o PATH-dependent |
| G5 Activation/recovery | swap atómico, manifest observado, fault matrix | unlink-first, rollback ambiguo sin escalation |
| G6 Plataformas | suite Windows/POSIX con paths adversos | plataforma sin atomicidad probada |
| G7 Review | build/full tests, revisión ACC y blast radius | falso éxito, fallo nuevo o evidencia ausente |
| G8 Handoff | API 4 y evidence 5 identificadas, estable consolidable | UI/pipeline incluidos o secuencia rota |

El `GO` final requiere todos los gates. Un fallo preexistente debe reproducirse en baseline y demostrarse no agravado; un fallo nuevo mantiene `STOP`.

## Fuentes y supuestos

- Fuente funcional: ACC-078 y cierre de `apply/recover` de ACC-079 según el incremento origen.
- Orden obligatorio: 1 → 2 → 3 → 4 → 5; este plan ejecuta el tercero.
- Fuente de target/instalación: contratos aceptados de incrementos 1 y 2, no reescritos aquí.
- Repo de implementación: Axiom runtime; las rutas de `role-builder.md` son destinos previstos y se mapean a la estructura real en Fase 0.
- Build y tests conocidos por contrato de repositorio: `npm run build` y `npx vitest run`.
- Supuesto de seguridad: staging y sibling de activación pueden ubicarse en filesystems compatibles con operación atómica; si no, no se degrada a borrado primero.
- Supuesto operativo: el updater posee únicamente recursos marcados dentro de la instalación global; cualquier contenido no atribuible se conserva.
- Restricción de esta edición: solo se modifican los nueve Markdown del incremento/plan autorizados; no se ejecutan transiciones ni se afirma ejecución de código.
