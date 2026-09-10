# 03 Criterios de Aceptación

## Criterios de aceptación

Los criterios siguientes son normativos y deberán aportar evidencia automatizada. «Sin mutación» significa igualdad del árbol relevante antes/después: contenido, existencia, tamaño y mtime de `install.json`, ausencia de temporales/locks nuevos y ausencia de receipts persistidos. La especificación no presupone que estos criterios ya pasen.

### Happy path

- **AC-079-01 — Gate de dependencia.** Dado un build donde el contrato validado de release identity del incremento 1 no está disponible o es incompatible, cuando se intenta implementar/integrar ACC-079, entonces el gate es STOP y no se introduce un modelo local alternativo.
- **AC-079-02 — Lectura v2 válida.** Dada una instalación real verificable y un v2 cerrado que coincide con ella, cuando se ejecuta `status`, entonces el outcome es `state_ready`, se devuelve la identidad real y el archivo se mantiene byte por byte.
- **AC-079-03 — Ausencia.** Dada una instalación sin `install.json`, cuando se ejecuta `status --json`, entonces el outcome es `state_absent`, exit code `11`, no aparece `0.0.0` y no se crea ningún archivo.
- **AC-079-04 — Migración basada en realidad.** Dado un v1 soportado y una instalación real cuya release identity puede verificarse, cuando el migrador se invoca desde una frontera mutante autorizada, entonces produce v2 a partir de la identidad real, registra provenance `legacy-migration` y conserva el fingerprint fuente.
- **AC-079-05 — Idempotencia.** Dado un v1 ya migrado con éxito, cuando se reintenta la misma solicitud, entonces se devuelve el v2 existente sin segunda escritura ni incremento adicional de revision.
- **AC-079-06 — Check disponible.** Dadas identidades local/remota válidas, cuando `check` compara ambas, entonces devuelve `up_to_date` con exit `0` o `update_available` con exit `10` y no persiste el resultado.
- **AC-079-07 — Plan disponible.** Dado un update candidato válido, cuando se ejecuta `plan`, entonces se obtiene `plan_ready` y exit `10`, con from/to refs explícitas, sin invocar apply ni modificar estado.
- **AC-079-08 — Operaciones exclusivas.** Para cada subcomando `status`, `check`, `plan`, `apply`, `recover`, cuando se proporciona exactamente uno y solo opciones permitidas, entonces el parser selecciona una única rama discriminada.
- **AC-079-09 — Envelope uniforme.** Para cada operación y clase de resultado, cuando se usa `--json`, entonces stdout contiene un único envelope v1 cerrado seguido de newline y parseable mediante `JSON.parse`.
- **AC-079-10 — Wrapper real.** Cuando los escenarios de contrato se ejecutan mediante el wrapper compilado, entonces stdout, stderr y exit code coinciden con los obtenidos en la entrada TypeScript soportada.

### Validaciones y errores

- **AC-079-11 — Schema cerrado.** Para cada nivel de v2 y envelope v1, dado un objeto con una propiedad adicional, cuando se valida, entonces se rechaza con el error tipado correspondiente y no se reescribe.
- **AC-079-12 — SemVer.** Versiones vacías, `v1.2.3`, parciales, con ceros inválidos o fuera del contrato SemVer se rechazan; versiones canónicas soportadas se aceptan sin normalización silenciosa.
- **AC-079-13 — ISO.** Timestamps sin zona, con offset no canónico, fecha imposible o texto libre se rechazan; el formato UTC canónico con milisegundos y `Z` se acepta.
- **AC-079-14 — Path.** Paths relativos, vacíos, no normalizados o no reconciliables con la instalación se rechazan sin resolverlos contra cwd.
- **AC-079-15 — Ref/provenance.** Ref ausente, provenance incompleta o identidad que no satisface `ReleaseIdentityV1` se rechaza con `release_identity_invalid`/`provenance_invalid`.
- **AC-079-16 — Corrupción.** JSON vacío, truncado, sintácticamente inválido, shape parcial o checksum/fingerprint inconsistente produce `state_corrupt`, exit `65`, sin stack trace en el envelope y sin reparación automática.
- **AC-079-17 — Schema futuro.** Un `schemaVersion` no soportado produce `schema_unsupported`; no se interpreta como v1 ni v2 por heurística.
- **AC-079-18 — Mismatch de migración.** Dado que v1 declara una versión pero la instalación real entrega otra identidad, cuando se intenta migrar, entonces gana la instalación real solo si la reconciliación y provenance son demostrables; en otro caso la migración se rechaza y v1 queda byte por byte intacto.
- **AC-079-19 — Lost update.** Dados dos writers que leyeron la misma revision/fingerprint, cuando el primero compromete y el segundo obtiene el lock, entonces el segundo devuelve `state_conflict`/exit `73` y no pisa la escritura ganadora.
- **AC-079-20 — Temporales únicos.** Dados writers concurrentes o PIDs reutilizados, cuando crean staging, entonces ningún proceso abre o trunca el temporal de otro y no existe nombre temporal fijo compartido.
- **AC-079-21 — Fallos durables.** Para fallos inyectados después de create, durante write, antes/después de fsync y antes/durante replace, el último target válido permanece legible o se obtiene `recovery_required`; nunca se publica JSON parcial ni un falso success.
- **AC-079-22 — Recuperación inequívoca.** Dado un temporal huérfano sin evidencia durable suficiente, cuando se inspecciona/recupera, entonces no se promueve por fecha o nombre; se limpia de forma segura bajo lock o se devuelve `recovery_required`.
- **AC-079-23 — Flags ambiguos.** `self-update` sin operación, con varios selectores, con flags legacy o con opciones ajenas devuelve `cli_usage_error`/exit `64` y un hint canónico, sin I/O mutante.
- **AC-079-24 — Dry-run.** `plan --dry-run` conserva exactamente la semántica read-only de `plan`; cualquier `--dry-run` con `apply`, `recover`, `status` o `check` se rechaza antes de alcanzar lock, migrador, writer o motor.
- **AC-079-25 — Apply cerrado.** En este incremento, toda invocación sintácticamente válida de `apply` devuelve `engine_unavailable`, exit `69`, `ok: false` y no produce attempt/success ni cambios en disco.
- **AC-079-26 — Recover cerrado.** En este incremento, toda invocación sintácticamente válida de `recover` devuelve `engine_unavailable`, exit `69`, `ok: false` y no informa `recovered` ni modifica artefactos.
- **AC-079-27 — Fallo remoto.** Si `check` no puede consultar el proveedor por una causa temporal, devuelve `release_check_unavailable`, exit `75`, `retryable: true` y no altera estado local.
- **AC-079-28 — Process lifecycle.** Instrumentando el entrypoint, todas las ramas asignan `process.exitCode`, retornan y permiten flush; ninguna llama a `process.exit()`.

### Permisos y visibilidad

- **AC-079-29 — Lock user-level.** Dos procesos del mismo usuario y distinto cwd que apuntan a la misma instalación compiten por la misma clave de lock de `@axiom/core`.
- **AC-079-30 — Scope correcto.** Instalaciones canónicamente distintas no se bloquean entre sí por una clave global accidental; aliases del mismo target sí convergen en la misma clave.
- **AC-079-31 — Permiso denegado.** Un error de lectura se distingue de archivo ausente; un error al adquirir lock o escribir devuelve código tipado, mantiene el target previo y no filtra secretos ni stack traces.
- **AC-079-32 — No escalado.** Ninguna operación intenta cambiar permisos, elevar privilegios o escribir fuera del target/área user-level autorizada para sortear un error.
- **AC-079-33 — Provenance visible.** `status` expone provenance sanitizada suficiente para explicar la identidad; no expone tokens, credenciales ni datos internos no contractuales.
- **AC-079-34 — Canales.** En modo JSON, cualquier explicación humana se dirige a stderr; stdout queda reservado al único envelope. En modo humano, los errores se escriben en stderr y preservan el mismo exit code.

### Estados y efectos observables

- **AC-079-35 — Matriz de lectura.** Fixtures para `state_absent`, `migration_required`, `state_ready`, `identity_mismatch`, `state_corrupt`, `schema_unsupported`, `state_io_error` y `recovery_required` seleccionan exactamente una rama.
- **AC-079-36 — Receipts distintos.** Fixtures y pruebas de tipos demuestran que check, attempt, success y recovery no son asignables ni serializables como el tipo incorrecto y que comparten `operationId` solo donde el contrato lo exige.
- **AC-079-37 — No mutación en status.** Ejecutar `status` sobre ausencia, v1, v2, corrupto y mismatch deja inalterado el snapshot completo del filesystem relevante.
- **AC-079-38 — No mutación en check.** Ejecutar `check` con resultado positivo, negativo o error remoto no crea `lastCheck`, cache, lock, temporal ni cambio de mtime.
- **AC-079-39 — No mutación en plan.** Ejecutar `plan` y `plan --dry-run` deja el mismo snapshot antes/después y no llama al adapter mutante.
- **AC-079-40 — Outcome no es éxito mutante.** `update_available`, `plan_ready`, `attempt_started`, `engine_unavailable` y `recovery_required` nunca serializan outcome `applied` ni `recovered`.
- **AC-079-41 — Exit taxonomy.** Cada outcome documentado se prueba contra la tabla de exit codes, incluida su propagación por wrapper; un mensaje localizado no cambia la clasificación.
- **AC-079-42 — Calidad global.** `npm run build`, pruebas focalizadas del área y `npx vitest run` finalizan correctamente o los fallos se clasifican con evidencia como introducidos o preexistentes. No se declara GO con un fallo nuevo.
- **AC-079-43 — Review de intención.** Una revisión independiente confirma que no se implementaron Git update, Launcher UI, release CI ni una fuente de autoridad paralela.
- **AC-079-44 — Habilitación.** El handoff al incremento 3 identifica APIs de reader/migrator/writer, grammar, envelope y errores estables, sin afirmar que los incrementos 3–5 estén implementados.
