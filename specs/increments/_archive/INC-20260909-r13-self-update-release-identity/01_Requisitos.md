# 01 Requisitos

## Objetivo del documento

Definir el comportamiento requerido para identificar releases de Axiom y consultar, sin mutaciones de Git, instalación o proyecto, la relación entre la publicación canónica, el checkout descargado, el CLI global instalado y el runtime gestionado por un proyecto. La consulta solo puede escribir una caché user-level tras una observación remota válida. Todos los requisitos son objetivos del incremento; su presencia aquí no implica que el runtime actual los cumpla.

## Requisitos del incremento

### Identidad e instalación

- **R13-REQ-001 — CLI global único.** Axiom MUST reconocer como instalación soportada un único CLI global por usuario. La resolución del comando efectivo y su versión MUST ser independiente del directorio de proyecto actual. Un binario local, una dependencia de proyecto o un checkout de desarrollo MUST NOT convertirse implícitamente en una segunda autoridad instalada.
- **R13-REQ-002 — Identidad del producto.** El artefacto ejecutable MUST exponer una identidad tipada con, como mínimo, producto, versión de producto, release tag cuando exista, commit de origen cuando esté demostrado e identidad de build/procedencia. Los campos ausentes MUST representarse explícitamente como no disponibles, no mediante valores inventados.
- **R13-REQ-003 — Fuente propia del runtime.** La identidad instalada MUST proceder de metadata embebida/generada junto al CLI o de un módulo propiedad del runtime. MUST eliminarse el literal o lookup de versión cuyo origen sea `@axiom/user-workspace`; ese paquete MUST NOT ser autoridad de versión, producto, tag, commit o build.
- **R13-REQ-004 — Installed version.** `installedVersion` MUST describir la versión del CLI global efectivo que ejecuta el comando. MUST incluir procedencia suficiente para distinguir metadata de artefacto, build de desarrollo y dato no disponible.
- **R13-REQ-005 — Downloaded version.** `downloadedVersion` MUST describir el checkout local sin forzarlo a ser SemVer. Será una unión discriminada: tag SemVer exacto en HEAD, tags exactos ambiguos, ref simbólica acompañada por commit, commit detached o `unavailable`. El commit completo o abreviado de forma no ambigua MUST permanecer disponible siempre que Git pueda resolver HEAD.
- **R13-REQ-006 — Published version.** `publishedVersion` MUST ser la mayor release SemVer válida anunciada por tags del remoto Git canónico conforme a la política de refs de este documento. MUST conservar tag, aliases, target OID remoto y, solo cuando su tipo esté demostrado sin mutación, el commit asociado.
- **R13-REQ-007 — Estado gestionado por proyecto.** `ManagedState.runtime.version` MUST mantener significado project-scoped: versión del runtime que el proyecto declara o tiene materializada. MUST NOT utilizarse para calcular `installedVersion`, sustituir `downloadedVersion`, elegir `publishedVersion` ni almacenar el resultado global de self-update.
- **R13-REQ-008 — No colapso de identidades.** La ausencia de una de las cuatro versiones MUST NOT rellenarse copiando otra. Toda comparación MUST indicar todos sus sujetos, valores saneados, base de comparación y referencias a la procedencia usada.

### Política Git y SemVer

- **R13-REQ-009 — Remoto canónico demostrado.** El descubridor MUST obtener la identidad del repositorio canónico desde una fuente de configuración o identidad de producto inequívoca, validada en la revalidación inicial. MUST NOT asumir que un remote llamado `origin` es canónico. Si no puede demostrar la coincidencia, MUST producir `unknown` sin consultar un remoto arbitrario.
- **R13-REQ-010 — Consulta remota read-only.** La observación remota MUST usar una operación equivalente a `git ls-remote` sobre el remoto/URL canónico con prompting interactivo deshabilitado. Todos los comandos Git MUST ejecutarse con optional locks/refrescos deshabilitados mediante la capacidad equivalente a `GIT_OPTIONAL_LOCKS=0`/`--no-optional-locks`. MUST NOT ejecutar `fetch`, `pull`, `merge`, `checkout`, `switch`, `reset`, mantenimiento, creación/borrado de refs ni comandos que escriban objetos, índice o worktree.
- **R13-REQ-011 — Política de refs.** Solo se considerarán refs bajo `refs/tags/` cuyo nombre completo sea `v` seguido de una SemVer estricta. Los tags anotados MUST resolverse a su target peeled y los tags ligeros a su target directo. El OID se conserva como `targetObjectId`; solo puede llamarse commit cuando su tipo quede demostrado con objetos ya disponibles. Un target demostrado como no-commit MUST invalidar el tag; un target no disponible localmente queda `unverified` y no habilita ancestry.
- **R13-REQ-012 — SemVer estricto.** Parseo y orden MUST seguir SemVer 2.0.0: comparación numérica de major/minor/patch, prerelease inferior a su release final, identificadores prerelease según su tipo y build metadata sin precedencia. MUST NOT usarse orden lexicográfico, coerción de versiones parciales ni aceptación de ceros iniciales inválidos.
- **R13-REQ-013 — Empates sin falsa precedencia.** Si candidatos de máxima precedencia difieren solo en build metadata y apuntan a targets distintos, el resultado MUST quedar `unknown` con evidencia de ambigüedad. Si apuntan al mismo target, MUST tratarse como aliases: se elige el único tag sin build metadata cuando exista y, en otro caso, el tag byte-wise menor; todos se conservan ordenados en `aliasTags`. Esta regla elige representación, no precedencia SemVer.
- **R13-REQ-014 — Relación del checkout.** La relación ahead/behind MUST basarse en versiones SemVer comparables o en ancestry demostrable entre commits con objetos ya presentes. Si el objeto remoto necesario no existe localmente o no se demuestra como commit, MUST devolver evidencia insuficiente y MUST NOT hacer `fetch` para resolverla.

### Descubrimiento, caché y evaluación

- **R13-REQ-015 — Best-effort tipado.** Los fallos esperables (sin repo, HEAD inválido, remoto ausente, timeout, red, autenticación no interactiva, salida remota inválida, target no-commit/no verificable, tags inválidos/ambiguos o caché corrupta) MUST convertirse en observaciones/diagnósticos tipados. Solo un fallo interno no clasificado puede abortar el comando.
- **R13-REQ-016 — Caché offline.** Tras una observación remota válida, Axiom MUST escribir atómicamente una entrada en almacenamiento user-level. La clave MUST incluir identidad canónica del repositorio, política de refs y versión de schema. La entrada MUST guardar timestamp UTC, versión/tag/aliases/target/commit demostrado, método, remoto sanitizado y procedencia. MUST NOT guardar credenciales, tokens ni URLs con secretos.
- **R13-REQ-017 — Uso honesto de caché.** Ante imposibilidad de consultar el remoto, Axiom MAY reutilizar la última entrada válida de la misma clave. El resultado MUST incluir `offline`, `freshness: cached`, `observedAt`, edad calculada y `stale: true`. En este incremento `stale` significa “histórico/no live”, no caducidad por umbral: toda evidencia cacheada es stale; evidencia live/current-process/unavailable no lo es. Una caché inválida o de otra política MUST ignorarse y diagnosticarse; nunca se presenta como dato live.
- **R13-REQ-018 — Estados tipados y trazables.** El evaluador MUST poder emitir los hechos `updated`, `update-available`, `checkout-behind`, `ahead`, `installed-misaligned`, `unknown` y `offline`. Cada hecho MUST portar sujeto, todos los sujetos comparados, base, valores saneados y `provenanceIds`; además, MUST calcular un `primaryState` determinista para clientes que solo admitan un estado.
- **R13-REQ-019 — Estado updated.** Se emite `updated` cuando `installedVersion` y `publishedVersion` son SemVer comparables con igual precedencia e identidad no contradictoria. Describe la instalación global, no garantiza que el checkout esté alineado.
- **R13-REQ-020 — Update available.** Se emite `update-available` cuando `publishedVersion` tiene precedencia SemVer mayor que `installedVersion`.
- **R13-REQ-021 — Checkout behind.** Se emite `checkout-behind` cuando el checkout es demostrablemente anterior a la publicación, por SemVer o ancestry, indicando la base usada.
- **R13-REQ-022 — Ahead.** Se emite `ahead` cuando el checkout o la instalación son demostrablemente posteriores/no publicados respecto de la release remota, indicando `subject: downloaded|installed` y la base usada. No se infiere “más nuevo” del nombre de una rama.
- **R13-REQ-023 — Installed misaligned.** Se emite `installed-misaligned` cuando la identidad efectiva instalada difiere de la identidad descargada y ambas son comparables. Es un diagnóstico de separación, no una autorización para reemplazar ninguna de ellas.
- **R13-REQ-024 — Unknown, offline y ausencia remota.** `unknown` MUST identificar datos ausentes, ambiguos o no comparables; `offline` MUST identificar que no hubo observación remota live. Sin caché, la publicación MUST representarse como `availability: unavailable`, `freshness: unavailable`, nunca como live/cached. Ambos estados pueden coexistir con otros hechos derivados de evidencia independiente.
- **R13-REQ-025 — Orden y estado primario.** `states` MUST ordenarse por `offline` → `unknown` → `installed-misaligned` → `ahead` → `checkout-behind` → `update-available` → `updated`; los empates se ordenan por sujeto y clave estable de evidencia. Solo se eliminan duplicados estructuralmente idénticos. `primaryState` MUST ser el `kind` del primer estado.

### Contrato CLI con consulta no mutante

- **R13-REQ-026 — Consulta explícita.** La superficie planificada `axiom self-update status` MUST limitarse a observar y evaluar. `--json` MUST emitir el contrato estructurado estable; la salida humana MUST derivarse del mismo resultado, incluido el `observedAt` del assessment, sin una segunda lógica de estado.
- **R13-REQ-027 — Sin efectos de actualización.** La consulta MUST NOT descargar, instalar, cambiar PATH, editar el proyecto, modificar `ManagedState`, actualizar el checkout ni pedir confirmación para aplicar cambios. La única escritura permitida es la caché user-level tras una observación remota válida.
- **R13-REQ-028 — Automatización segura.** Estados de dominio, incluida una actualización disponible, MUST poder emitirse con exit code `0` si el contrato se produjo correctamente. Uso inválido MUST usar un código distinto de cero y un fallo interno no clasificado otro distinto; diagnósticos textuales van a stderr y JSON válido permanece aislado en stdout.
- **R13-REQ-029 — Procedencia y privacidad.** Cada identidad MUST enlazar mediante `provenanceId` una entrada que indique sujeto, fuente, frescura, método y timestamp. Cada hecho MUST enlazar todas las evidencias usadas. Remotos y rutas sensibles se sanitizan; la salida MUST NOT revelar credenciales, tokens ni información innecesaria del usuario.
- **R13-REQ-030 — Compatibilidad de consumidores.** El contrato compartido MUST estar versionado y ser consumible por CLI y futuros clientes sin importar UI de Launcher ni tipos de `@axiom/user-workspace`.

## Reglas de negocio relevantes

### Resolución del remoto y release publicada

1. Resolver la identidad canónica configurada para el producto. Normalizar únicamente aspectos que no alteren el repositorio, eliminando credenciales para comparación y display.
2. Si el checkout tiene remotes, aceptar uno solo cuando su fetch URL normalizada coincide con la identidad canónica. El nombre del remote es informativo.
3. Consultar directamente la fuente canónica aunque no exista checkout local; el checkout no es requisito para conocer `publishedVersion`.
4. Enumerar tags y refs peeled sin actualizar refs locales, con prompting y optional locks deshabilitados. Parsear exclusivamente tags con `v` y SemVer canónica.
5. Conservar el OID remoto como target. Si el objeto existe localmente, comprobar su tipo con una operación Git de lectura; rechazar un tipo no-commit y dejar `unverified` si no existe localmente.
6. Elegir la mayor precedencia SemVer. Para candidatos empatados, aplicar la política de aliases/ambigüedad de R13-REQ-013.
7. Mantener separados `publishedVersion`, `publishedTag`, `aliasTags`, `targetObjectId` y `publishedCommit` opcional.

### Resolución del checkout descargado

1. Resolver HEAD mediante operaciones Git de lectura con optional locks deshabilitados.
2. Si HEAD coincide con un único tag de release canónico válido, usar `kind: tag` y conservar tag, SemVer y commit.
3. Si varios tags de distinta precedencia coinciden con HEAD, usar `kind: ambiguous-tag`, conservar candidatos ordenados y emitir `checkout-tag-ambiguous` más `unknown`.
4. Si los tags coincidentes son aliases de igual precedencia y mismo commit, aplicar la regla determinista de aliases.
5. En otro caso, si HEAD tiene ref simbólica, usar `kind: ref` con nombre de ref y commit. Una rama no se transforma en SemVer.
6. En detached HEAD sin tag canónico, usar `kind: commit`.
7. El estado dirty es un atributo diagnóstico, no modifica el commit ni convierte la identidad en otra versión.
8. Si no hay repo o HEAD válido, usar `kind: unavailable` y un diagnóstico tipado.

### Resolución de la instalación

1. La versión se lee de metadata generada/embebida en el artefacto que contiene el entrypoint efectivo.
2. Tag, commit y build pueden ser desconocidos en builds de desarrollo, pero su fuente debe declararse.
3. La ruta de instalación se puede usar para demostrar el CLI efectivo, pero su representación pública debe minimizar o sanear información del usuario.
4. Si se detectan candidatos adicionales en PATH, se informa la ambigüedad; no se selecciona silenciosamente un CLI de proyecto como autoridad global.

### Caché

- Se escribe únicamente una observación remota completa y parseable, mediante archivo temporal y reemplazo atómico o abstracción equivalente.
- No existe una “verdad offline” independiente: la caché es evidencia histórica. Siempre se intenta una observación live cuando la política y el entorno lo permiten.
- Toda evidencia recuperada de caché tiene `freshness: cached`, `stale: true`, timestamp original y edad calculada. No hay TTL en este incremento.
- La antigüedad se calcula con un reloj inyectable. Un reloj regresivo no puede producir edad negativa; se informa `unknown` de clock/provenance.
- Cambios de repositorio canónico, política de refs o schema invalidan la clave anterior sin migraciones destructivas.
- Una entrada corrupta se ignora, no bloquea el comando y no se sobrescribe salvo que exista una nueva observación live válida.

### Derivación de estados

Los estados son hechos, no una máquina de transición ni instrucciones de actualización. Una evaluación puede ser, por ejemplo:

- instalación igual a publicación + checkout anterior: `updated`, `checkout-behind`, `installed-misaligned`;
- publicación posterior a instalación + checkout igual a publicación: `update-available`, `installed-misaligned`;
- checkout posterior a publicación + instalación igual a publicación: `updated`, `ahead`, `installed-misaligned`;
- remoto inaccesible con caché utilizable: `offline` más los hechos comparativos que la evidencia cacheada permita;
- remoto inaccesible sin caché y sin comparación posible: `offline`, `unknown`, con publicación unavailable.

Cada hecho incluye sujetos, valores, base (`semver`, `git-ancestry`, `identity` o `evidence-gap`) y referencias a las entradas de procedencia. El orden de `states` es exactamente el de R13-REQ-025; no depende del orden de descubrimiento asíncrono.

## Fuera de alcance funcional

- `axiom self-update apply` o cualquier variante que materialice una actualización.
- Descarga de artefactos, firma/verificación criptográfica, rollback o recuperación transaccional.
- Manifest v2 completo, catálogo de canales, rings o selección de prereleases por política de usuario.
- Launcher UI, badges, modales, notificaciones de escritorio o refresco automático.
- Publicación, tagging o promoción en CI y certificación final de releases.
- Reescritura o migración masiva de `ManagedState`.
- Modificación del remoto, refs, índice, worktree, object database o configuración Git.
- Autenticación interactiva o almacenamiento de secretos para consultar releases.
