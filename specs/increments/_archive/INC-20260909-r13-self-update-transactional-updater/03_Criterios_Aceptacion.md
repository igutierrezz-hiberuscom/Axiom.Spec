# 03 Criterios de Aceptación

## Criterios de aceptación

Los criterios siguientes son la evidencia requerida para ACC-078 y para cerrar `apply/recover` de ACC-079. Se evalúan sobre código futuro del runtime; este documento no afirma que estén satisfechos hoy. Todo escenario debe comprobar outcome, fase, diagnóstico, estado Git, entry activo, manifest, journal y ausencia de mutaciones fuera de la raíz gestionada.

### Happy path

**AC-078.1 — Dependencias y secuencia.** Dado el incremento 3, cuando se evalúa el gate de inicio, entonces existe evidencia aceptada de los incrementos 1 y 2 y se registra la secuencia 1 → 2 → 3 → 4 → 5. Si falta una dependencia, el gate es `STOP` y no se modifica código/runtime.

**AC-078.2 — Target legítimo.** Dada una release local fixture publicada y validada con versión, tag/ref y commit exactos, cuando se ejecuta preview, entonces el plan referencia exclusivamente ese descriptor. Una versión/ref aportada sin evidencia de publicación es rechazada.

**AC-078.3 — Plan único e inmutable.** Dado un preview válido, cuando se serializa/deserializa el plan y se entrega a apply, entonces digest y payload son idénticos y apply usa ese mismo target/estrategia. Alterar cualquier campo, cambiar el target remoto o presentar drift de instalación devuelve `failed` antes de mutar.

**AC-078.4 — Instalación transaccional.** Dada una instalación gestionada limpia y una release posterior válida, cuando apply concluye, entonces:

1. el candidato procede del commit exacto;
2. `npm ci` consumió el lockfile del candidato;
3. el build completo terminó correctamente;
4. `--version`, `--help`, load y smoke/doctor se ejecutaron sobre rutas absolutas candidatas;
5. el único entry global fue reemplazado atómicamente;
6. el entry activo volvió a verificarse por ruta absoluta;
7. el manifest contiene la versión observada exacta, release id y commit;
8. el journal termina `committed`;
9. el outcome es `installed`.

**AC-078.5 — Ya actualizado.** Dado que release, commit, entry y versión observada ya coinciden y los smokes son sanos, cuando apply consume un plan válido, entonces no ejecuta `npm ci`, build ni swap, no reescribe el manifest sin necesidad y devuelve `unchanged`.

**AC-078.6 — Retry tras rollback.** Dado un primer intento que falla y completa rollback probado, cuando se corrige la causa, se genera un nuevo preview y se reintenta, entonces el segundo apply puede terminar `installed`; no reutiliza un plan stale ni residuos no acreditados del intento anterior.

**AC-079.1 — Journal de apply.** Durante una instalación exitosa, cada fase mutable deja una transición monotónica, durable y con checksum. Ningún estado posterior se publica antes de que su efecto sea verificable.

**AC-079.2 — Recover idempotente.** Dada una interrupción en cualquier frontera no terminal, cuando recover se ejecuta una o más veces, entonces adquiere el lock, observa realidad y restaura el snapshot previo en ausencia de commit. Si la restauración queda probada devuelve `unchanged`; si existe commit válido reconoce `installed`; si hay ambigüedad devuelve `recovery-required` sin borrado ciego.

### Validaciones y errores

**AC-078.7 — Remoto/ref/commit.** Con remote URL/identidad incorrecta, tag ausente o movido, ref que no resuelve al commit, commit no alcanzable o descriptor no publicado, preview/apply devuelve `failed`, no abre una transacción mutable y mantiene entry/manifest byte a byte.

**AC-078.8 — Matriz Git local.** Usando un bare remote y clones locales se prueban por separado:

| Estado | Resultado obligatorio |
|---|---|
| clean, attached y behind con ancestor válido | puede continuar a staging/fast-forward seguro |
| dirty tracked | `failed`, sin stash/reset/clean |
| dirty staged | `failed`, sin stash/reset/clean |
| dirty untracked | `failed`, sin borrar archivos |
| detached HEAD | `failed`, sin switch/checkout |
| branch/upstream inesperados | `failed` |
| ahead | `failed`, sin push/reset |
| diverged/unrelated | `failed`, sin merge/rebase/reset |

En todos los casos bloqueantes, `git status`, HEAD y archivos locales quedan idénticos.

**AC-078.9 — Dos updaters.** Dadas dos operaciones sobre el mismo `installId`, cuando la primera posee el lock, entonces la segunda no entra en fases mutables, termina en tiempo acotado con `failed/update-lock-held` y no borra el lock. Tras liberación, un nuevo plan puede ejecutarse. Un lock de ownership dudoso produce `recovery-required`, no reclamación por edad.

**AC-078.10 — Fallo de dependencias o build.** Al inyectar fallo de spawn, timeout, signal o exit no cero en `npm ci` o build, el outcome es `failed`, el error conserva tipo/fase/cwd/ejecutable sanitizados y entry/manifest previos siguen sanos. El candidato solo se limpia si ownership y contención están demostrados.

**AC-078.11 — Mismatch y smokes.** Si `--version` candidata difiere incluso cuando el build terminó, o fallan `--help`, load o doctor, apply devuelve `failed` antes del swap. PATH puede apuntar a otro `axiom` sin alterar la prueba: siempre se ejecuta la ruta absoluta candidata.

**AC-078.12 — Fallo de activación.** Si la primitiva atómica no está disponible o falla antes de commit, no se ejecuta unlink-first. Si no hubo swap, el estado previo permanece; si el resultado de la primitiva es ambiguo, se inspecciona y se hace rollback o se devuelve `recovery-required`.

**AC-078.13 — Fallo de manifest.** Al fallar temp write, flush o rename del manifest después de activar, apply restaura atómicamente entry y manifest previos y los verifica. Devuelve `failed` si la restauración se prueba; si no, `recovery-required`. Nunca deja el target activo con una versión previa declarada como éxito.

**AC-079.3 — Interrupción por frontera.** Existe fault injection determinista antes y después de lock, journal, preflight, fetch, stage, `npm ci`, build, verificación, preparación/swap del entry, verificación activa, manifest, commit, rollback y cleanup. Cada caso cumple una de estas reglas:

- antes del swap: activo intacto y `failed`;
- después del swap pero antes de commit: rollback probado y `failed`, o `recovery-required`;
- después de commit: `installed`; un cleanup no seguro se omite y se diagnostica sin revertir una instalación sana.

**AC-079.4 — Rollback integral.** En fallos post-swap, la evidencia demuestra restauración de los cuatro dominios: release/commit candidato, dependencias/build candidato, entry activo y manifest. Ninguna restauración usa `reset`, `stash`, `clean` o borrado recursivo sin marcador.

**AC-079.5 — Recovery con journal incompleto/corrupto.** Si el journal está truncado, checksum/schema no es válido o un path escapa de la raíz mediante symlink/reparse point, recover no ejecuta esa instrucción, preserva evidencia y devuelve `recovery-required`.

**AC-079.6 — Señales y procesos hijos.** Una cancelación/timeout intenta terminar el árbol de procesos de forma acotada. Si ocurre antes de mutación y el activo se verifica, el outcome es `failed`; tras swap se ejecuta rollback. Una segunda señal o hijo no controlable que impida probar consistencia produce `recovery-required`.

**AC-079.7 — Separación del instalador.** Sin `installId`, manifest gestionado o entry reconocido, preview/apply devuelve `failed/initial-install-required`. No crea la raíz global, no adopta un clone y no escribe una versión.

**AC-079.8 — Paridad Windows/POSIX.** La suite pasa en Windows y POSIX con paths que contienen espacios. Incluye case/drive/UNC y reparse points donde aplique, symlinks/permisos en POSIX, mismo-volumen para swap y prohibición de escape. Los outcomes e invariantes son iguales aunque cambie la primitiva de plataforma.

### Permisos y visibilidad

**AC-078.14 — Permisos preflight.** Antes de fetch/staging, el updater verifica lectura/escritura/rename en raíces gestionadas y ejecución de herramientas requeridas. No solicita elevación de forma implícita. Un permiso insuficiente produce `failed` antes del swap.

**AC-078.15 — Mínimo privilegio.** Lock, journal, candidatos temporales y manifest se crean con permisos no más amplios que la instalación. El updater no cambia permisos de repositorios o workspaces ajenos.

**AC-078.16 — Diagnóstico seguro.** Eventos y errores muestran fase, outcome, release id, commit abreviado y paths gestionados necesarios, pero redactan credenciales embebidas en remotos, tokens, variables sensibles y output ilimitado.

**AC-079.9 — API visible para consumidores.** Tests de contrato pueden importar runner/helpers y observar eventos `planning`, `locked`, `preflight`, `fetching`, `installing`, `building`, `verifying`, `activating`, `persisting`, `rolling-back`, `recovering` y `completed`. La API no importa paquetes del Launcher.

**AC-079.10 — Sin UI del incremento 4.** El diff del incremento 3 no añade vistas, botones, ventanas, auto-relaunch ni selector visual. Un fake consumer headless demuestra que el Launcher futuro puede invocar preview/apply/recover sin acceder a internals.

### Estados y efectos observables

**AC-078.17 — Outcomes exhaustivos.** Tests de tipos y runtime demuestran que solo se emiten `installed`, `unchanged`, `failed` y `recovery-required`; cada uno cumple su invariante y ningún fallo se traduce a éxito genérico.

**AC-078.18 — Versión observada.** El valor de versión instalada del manifest se compara con stdout normalizado de la invocación absoluta post-swap. Un test hace diferir la versión esperada y la observada y confirma que no se persiste la esperada.

**AC-078.19 — Entry único.** Antes y después de apply hay un solo entry autoritativo. La prueba observa el directorio durante la frontera de activación y confirma que no existe ventana de borrado ni un segundo shim/binlink utilizable.

**AC-079.11 — Evidencia de no clobber.** Snapshots/hash de archivos fuera de staging y de cambios Git locales permanecen iguales en todos los fallos. Cleanup rechaza paths sin marker, paths activos, escapes y transaction ids distintos.

**AC-079.12 — Matriz de recovery.** Se cubren al menos: interrupción preactivación, post-swap/premanifest, postmanifest/precommit, commit durable/precleanup, fallo durante rollback, reintento de recover y retry posterior de apply.

**AC-079.13 — Gate GO/STOP.** El gate final solo es `GO` cuando pasan build, unitarias, integración con Git local, fault matrix completa, dos updaters, pruebas Windows/POSIX, revisión de blast radius y revisión independiente contra ACC-078/079. Cualquier ausencia mantiene `STOP`; no se declara cierre.

**AC-079.14 — Habilitación posterior.** La evidencia identifica la API estable consumible por el incremento 4 y los outcomes/evidencias consumibles por el 5, sin implementar ninguno. La secuencia 1 → 2 → 3 → 4 → 5 queda preservada.
