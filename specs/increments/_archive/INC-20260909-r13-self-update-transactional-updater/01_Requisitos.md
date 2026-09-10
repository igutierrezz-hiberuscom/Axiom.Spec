# 01 Requisitos

## Objetivo del documento

Definir el comportamiento normativo del incremento 3/5 para cumplir ACC-078 y cerrar las ramas operativas `apply` y `recover` de ACC-079. Los términos **DEBE**, **NO DEBE** y **SOLO** expresan requisitos obligatorios. El documento describe el estado objetivo; no certifica que exista en el runtime actual.

## Requisitos del incremento

### Dependencias, identidad y target

**R3-001 — Gate de secuencia.** El trabajo de implementación DEBE comenzar con un gate que demuestre la aceptación de los incrementos 1 y 2 de R13 y los contratos que aportan. Sin esa evidencia el resultado del gate es `STOP`. La secuencia completa es 1 → 2 → 3 → 4 → 5.

**R3-002 — Separación de instalación inicial.** El updater SOLO DEBE operar sobre una instalación global gestionada cuya identidad, raíz, entry y manifest puedan validarse. Si no existe, DEBE devolver `failed` con causa `initial-install-required`, sin crear/adoptar una instalación ni escribir versión.

**R3-003 — Target publicado.** El target DEBE resolverse desde el descriptor de una release publicada y validada proporcionado por los contratos de los incrementos 1 y 2. El descriptor DEBE vincular al menos release id, versión esperada, remoto autorizado, ref/tag inmutable y commit exacto. Texto libre, branch móvil sin evidencia o datos aportados solo por UI NO son targets válidos.

**R3-004 — Coherencia ref/commit.** Antes de planificar y de nuevo al aplicar, el updater DEBE demostrar que la ref/tag remota resuelve exactamente al commit del descriptor y que el commit pertenece al remoto autorizado. Un tag movido, una ref ausente o un commit distinto invalidan el plan antes de mutar la instalación.

### Plan común a preview y apply

**R3-005 — Plan inmutable.** `preview` DEBE producir un `UpdatePlan` canónico, serializable, profundamente inmutable y sellado con digest. El plan DEBE contener target validado, snapshot observado de la instalación, estrategia, rutas absolutas gestionadas, comandos declarados y precondiciones.

**R3-006 — Mismo plan.** `apply` DEBE recibir y verificar exactamente el plan emitido por `preview`. NO DEBE reconstruirlo, sustituir target, cambiar estrategia ni completar campos desde estado mutable. Cualquier alteración del payload o digest produce `failed` sin mutación.

**R3-007 — Revalidación sin mutación del plan.** Bajo lock, `apply` DEBE comparar las precondiciones actuales con el snapshot del plan y revalidar la release. Si cambian instalación, entry, manifest, remoto, ref, commit o estado Git, DEBE exigir un nuevo preview; el plan original permanece intacto.

### Lock y preflight

**R3-008 — Lock global.** `apply` y `recover` DEBEN adquirir de forma exclusiva un lock por identidad canónica de instalación antes de cualquier cambio. El lock DEBE registrar operation id, plan digest cuando aplique, pid, host, inicio y raíz canónica sin secretos.

**R3-009 — Contención.** Un segundo updater NO DEBE esperar indefinidamente ni mutar estado. Debe respetar el timeout configurado y devolver `failed` con diagnóstico `update-lock-held`. Un lock potencialmente abandonado SOLO puede reclamarse tras demostrar que no existe propietario vivo y mediante compare-and-swap/reemplazo atómico; si la prueba no es concluyente, el resultado es `recovery-required`. Nunca se borra solo por antigüedad.

**R3-010 — Preflight Git completo.** El preflight DEBE observar y registrar: URL e identidad del remoto, upstream, branch actual, estado detached, HEAD, ref/tag target, commit target, cambios tracked, staged y untracked, relación ancestor/fast-forward y estados ahead/diverged.

**R3-011 — Estados bloqueantes.** Dirty, detached, branch/upstream inesperados, ahead/diverged, remoto no autorizado, tag/ref inválido o commit no alcanzable DEBEN producir `failed` sin cambiar checkout, dependencias, build, entry ni manifest.

**R3-012 — Prohibición destructiva.** El updater NO DEBE ejecutar `git stash`, `git reset`, merge no fast-forward, checkout/switch que sobrescriba trabajo, clean destructivo, force, ni eliminación de archivos cuya propiedad no esté demostrada.

### Preparación, dependencias y build

**R3-013 — Estrategia Git reversible.** La preparación DEBE usar `fetch` más validación explícita de fast-forward en un espacio gestionado o staging seguro. `git pull` solo sería admisible con semántica `--ff-only`, sin hooks/interacción y con reversibilidad demostrada; si no puede garantizarse sin clobber, DEBE usarse staging. El checkout activo del operador no se usa como área temporal.

**R3-014 — Staging propiedad de la transacción.** Toda ruta candidata DEBE ser absoluta, estar contenida en la raíz gestionada de staging, residir en el filesystem requerido para activación atómica y portar un marcador no falsificable accidentalmente con transaction id. No se siguen symlinks/reparse points fuera de la raíz.

**R3-015 — Lockfile obligatorio.** El candidato DEBE contener el lockfile esperado. Las dependencias DEBEN instalarse con `npm ci` desde ese lockfile y con working directory absoluto del candidato. La ausencia o incoherencia del lockfile produce `failed`.

**R3-016 — Build completo.** Tras `npm ci`, el updater DEBE ejecutar el build completo del runtime. No puede activar salidas parciales, heredadas de otra release o generadas antes del commit target.

**R3-017 — Aislamiento.** Fallos de fetch, preparación, `npm ci` o build NO DEBEN alterar entry o manifest activos. Los restos solo se limpian si ownership y contención están demostrados; de otro modo se conservan con diagnóstico para recuperación.

### Verificación y activación

**R3-018 — Rutas absolutas.** Todas las verificaciones DEBEN invocar el entry construido mediante ruta absoluta y cwd explícito; PATH, alias, shell lookup o el shim actualmente activo NO pueden decidir qué candidato se valida.

**R3-019 — Suite mínima.** Antes de activar, el candidato DEBE superar: `--version`, `--help`, carga/import del entry y smoke/doctor. Cada proceso DEBE tener timeout, captura estructurada de spawn/exit/signal y salida limitada/sanitizada.

**R3-020 — Versión exacta.** La salida normalizada de `--version` DEBE ser exactamente la versión esperada del descriptor de release. Cualquier mismatch produce `failed` antes de activar.

**R3-021 — Entry único y activación atómica.** El updater DEBE resolver el único shim/binlink autorizado, preparar su reemplazo como sibling en el mismo filesystem y activar con una primitiva atómica específica de plataforma. NO DEBE borrar primero el entry activo ni dejar dos entries autoritativos. Si la atomicidad no puede demostrarse, DEBE fallar antes del swap.

**R3-022 — Verificación posterior.** Después del swap y antes del commit, el updater DEBE ejecutar por la ruta absoluta activa al menos `--version` y un smoke de carga/doctor. El resultado observado debe seguir coincidiendo con el candidato verificado.

**R3-023 — Manifest observado.** El manifest DEBE escribirse mediante temp + flush + reemplazo atómico. El único valor que representa la versión instalada DEBE provenir del `--version` observado sobre el entry activo y validado. La versión deseada no se copia como si fuera observada. Release id y commit pueden persistirse en campos propios como evidencia factual.

### Journal, rollback y recovery

**R3-024 — Journal durable.** Tras adquirir el lock y superar el preflight, `apply` DEBE abrir un journal versionado antes de la primera mutación transaccional de release/candidato, dependencias/build, entry o manifest, y persistir atómicamente cada transición. El lock es un artefacto de coordinación previo y separado; su ownership se valida aunque todavía no exista journal. El journal debe contener transaction id, digest del plan, fases, paths canónicos, snapshots del estado previo, candidatos y errores; NO debe contener tokens, credenciales ni salidas ilimitadas.

**R3-025 — Fronteras inyectables.** Cada frontera entre lock, preflight, fetch, staging, `npm ci`, build, verificación, activación, manifest, commit, rollback y cleanup DEBE aceptar fault injection mediante una dependencia solo de test. Las ramas reales no pueden depender de variables ocultas de producción.

**R3-026 — Rollback completo.** Ante fallo anterior al commit, el rollback DEBE abarcar, en orden inverso y según lo tocado: bytes/existencia previa del manifest, entry activo anterior, release/commit candidato y dependencias/build candidatos. El estado previo se restaura desde snapshots, nunca con `reset` ni borrado genérico.

**R3-027 — Prueba posterior al rollback.** Un rollback solo se considera completo si el entry anterior responde por ruta absoluta, su versión observada coincide con el snapshot previo y el manifest restaurado coincide byte a byte o en ausencia documentada. Si no puede probarse, el outcome DEBE ser `recovery-required`.

**R3-028 — Recovery idempotente.** `recover` DEBE adquirir el mismo lock, seleccionar una transacción no terminal, validar esquema/digest/ownership, observar la realidad y reconciliar. Sin marcador durable de commit, la política por defecto es volver al snapshot previo; solo un estado ya comprometido y verificable puede reconocerse como instalado. Repetir `recover` no debe degradar un estado ya reconciliado.

**R3-029 — Sin borrado ciego.** Cleanup y recovery SOLO pueden eliminar un candidato si transaction id, marcador, realpath, contención, no-actividad y ausencia de escapes por symlink/reparse point están demostrados. Si una comprobación falla, se conserva el path y se emite diagnóstico; nunca se amplía el borrado.

### Outcomes, errores y API

**R3-030 — Outcomes cerrados.** Toda ejecución termina en exactamente uno de estos outcomes:

- `installed`: target activado, manifest observado persistido, journal comprometido y entry activo verificado;
- `unchanged`: la release/commit ya estaban activos y saludables o recovery confirmó/restauró el snapshot previo sin instalar target;
- `failed`: la operación no instaló target y puede demostrarse que el estado activo previo sigue sano;
- `recovery-required`: no puede demostrarse una configuración consistente o el rollback/recovery quedó incompleto.

**R3-031 — Nunca falso éxito.** Ningún write de versión, mensaje de intención, fetch exitoso o build aislado autoriza `installed`. Un error ocurrido después de activar solo puede acabar en `failed` si el rollback queda probado; en caso contrario es `recovery-required`.

**R3-032 — Errores de proceso.** Timeout, señal, fallo de spawn y exit no cero DEBEN distinguirse en un error estructurado con fase, ejecutable, args sanitizados, cwd, duración y causa. El runner debe intentar finalizar el árbol de procesos de forma acotada y continuar con rollback según la fase.

**R3-033 — Cancelación.** Una cancelación antes de mutar termina como `failed`; durante una fase mutable solicita parada segura y rollback. Una segunda señal o imposibilidad de detener hijos que impida verificar consistencia termina en `recovery-required`.

**R3-034 — Paths multiplataforma.** Una única abstracción de paths DEBE regir Windows y POSIX: resolución absoluta/canónica, comparación sensible a reglas de plataforma, validación de raíz, mismo volumen para swap, manejo de espacios, permisos, symlinks y reparse points. No se concatenan comandos shell con paths.

**R3-035 — API headless.** El core DEBE exponer helpers equivalentes a `previewGlobalUpdate`, `applyGlobalUpdate`, `recoverGlobalUpdate` y `runGlobalSelfUpdate`, además de eventos tipados de progreso y cancelación. La API no debe importar UI ni exigir Launcher.

**R3-036 — Consumidor futuro.** El incremento 4 podrá mostrar preview, progreso y outcome consumiendo la API sin replicar validación o rollback. El incremento 5 podrá automatizar release/docs consumiendo evidencia, sin formar parte de este alcance.

## Reglas de negocio relevantes

1. **Legitimidad antes que disponibilidad:** si no puede probarse que el target es una release publicada válida, no hay actualización.
2. **Conservar intención del operador:** cualquier estado Git local no canónico se bloquea; el updater no decide cómo resolverlo.
3. **Una instalación, un escritor:** lock, journal, manifest y entry se derivan de la misma identidad canónica.
4. **Plan visible, ejecución fiel:** lo previsualizado es lo aplicado; un cambio de mundo exige otro plan.
5. **Construir y observar antes de declarar:** la versión instalada es un hecho medido, no un valor solicitado.
6. **Commit al final:** activar no equivale a completar; el commit requiere manifest durable y verificación post-swap.
7. **Rollback verificable:** restaurar se demuestra con ejecución y comparación, no solo por ausencia de excepción.
8. **La ambigüedad se escala:** cuando no puede probarse seguridad, se conserva evidencia y se devuelve `recovery-required`.
9. **Ownership estricto:** solo se modifican la instalación global gestionada y recursos creados por la transacción actual.
10. **Sin prompts destructivos:** el core no pide permiso para stash/reset/clean porque esas acciones están prohibidas.

## Fuera de alcance funcional

- Instalador inicial, adopción o migración de instalaciones preexistentes no gestionadas.
- Selector visual de release, Launcher, notificaciones, relaunch y cualquier UI del incremento 4.
- Producción/promoción final de releases, firma, publicación y manuales del incremento 5.
- Reparación de repositorios Git del usuario, resolución de conflictos o conservación mediante stash.
- Actualización de workspaces de proyectos, dependencias de proyectos consumidores o repositorios distintos de la instalación global.
- Cierre de estado, transición Core, receipts adicionales o edición de índices/metadatos durante esta tarea documental.
