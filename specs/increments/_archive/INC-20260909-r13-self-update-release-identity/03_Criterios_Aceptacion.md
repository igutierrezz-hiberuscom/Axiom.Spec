# 03 Criterios de Aceptación

## Criterios de aceptación

Los criterios se verifican contra comportamiento objetivo. Deben implementarse con fixtures controladas y no mediante la instalación, red, caché o repositorios reales del operador.

### Happy path

- **R13-AC-001 — Identidad del artefacto.** Dado un CLI global de prueba con metadata embebida válida, cuando se resuelve su identidad, entonces `installedVersion`, producto, tag, commit y build proceden del artefacto efectivo y ninguna lectura importa o consulta `@axiom/user-workspace`.
- **R13-AC-002 — Autoridad global única.** Dado un CLI global y un proyecto que contiene otra versión o binario local, cuando se ejecuta status mediante el entrypoint global, entonces la identidad instalada sigue siendo la global; la versión de proyecto solo puede aparecer como `ManagedState.runtime.version` y el candidato local se diagnostica sin sustituir la autoridad.
- **R13-AC-003 — Release publicada.** Dado un remoto Git local canónico con tags `v1.9.0`, `v1.10.0` y tags no canónicos, cuando se descubre la publicación, entonces `publishedVersion` es `1.10.0`, conserva tag/target/procedencia canónicos e ignora los tags inválidos sin orden lexicográfico.
- **R13-AC-004 — Tags anotados, ligeros y targets.** Dado un remoto con tags anotados y ligeros válidos, cuando se resuelven sus refs, entonces cada release conserva el target peeled/direct correcto. Si el objeto existe localmente y es commit, `commit` queda demostrado; si no existe, queda `targetType=unverified`, `commit=null` y no se usa para ancestry. Un target local demostrado como blob/tree se rechaza con `release-target-not-commit`. Ningún caso actualiza refs locales.
- **R13-AC-005 — Checkout por tag.** Dado un checkout cuyo HEAD coincide exactamente con un único tag canónico válido, cuando se obtiene `downloadedVersion`, entonces su clase es `tag` e incluye SemVer, tag y commit.
- **R13-AC-006 — Checkout por ref.** Dado un checkout en una rama sin tag exacto, cuando se obtiene `downloadedVersion`, entonces su clase es `ref`, incluye ref y commit y no inventa una SemVer.
- **R13-AC-007 — Checkout detached.** Dado un detached HEAD sin tag exacto, cuando se obtiene `downloadedVersion`, entonces su clase es `commit` y conserva el commit sin convertirlo en versión.
- **R13-AC-008 — Runtime project-scoped.** Dados dos proyectos con valores distintos de `ManagedState.runtime.version`, cuando el mismo CLI consulta cada uno, entonces los valores se informan como contexto de proyecto y `installedVersion` permanece idéntica en ambos; status no modifica ninguno.
- **R13-AC-009 — Estado alineado.** Dadas publicación, instalación y checkout en `1.4.0`, cuando se evalúa el estado, entonces se emite `updated`, no se emite una actualización disponible y la procedencia de cada identidad es explícita y resoluble.
- **R13-AC-010 — Actualización disponible.** Dada publicación `1.5.0`, instalación `1.4.2` y checkout comparable, cuando se evalúa, entonces se emite `update-available` para installed con base SemVer y referencias a ambas evidencias, sin descargar ni instalar nada.
- **R13-AC-011 — Checkout atrasado con CLI actualizado.** Dada publicación/instalación `1.5.0` y checkout `1.4.0`, entonces los hechos incluyen `updated`, `checkout-behind` e `installed-misaligned`; ninguno se pierde por el estado primario.
- **R13-AC-012 — Checkout ahead.** Dada publicación `1.5.0`, instalación `1.5.0` y checkout `1.6.0` o un commit descendiente demostrable, entonces se emiten `updated`, `ahead(subject=downloaded)` e `installed-misaligned` con la base y evidencias correspondientes.
- **R13-AC-013 — Salida única.** Dada una evaluación válida, las salidas text y JSON se derivan del mismo `ReleaseAssessment`; coinciden en identidades, estados, `observedAt` del assessment y procedencia.
- **R13-AC-014 — Caché de observación válida.** Dada una consulta live exitosa, cuando se persiste la caché, entonces existe una única entrada user-level validada, escrita atómicamente y claveada por schema, repositorio y política de refs.

### Validaciones y errores

- **R13-AC-015 — SemVer estricto.** Fixtures con `v1.10.0 > v1.9.0`, prerelease inferior a su final, identificadores numéricos ordenados numéricamente, build metadata sin precedencia, versiones parciales y ceros iniciales inválidos producen exactamente la precedencia/invalidación definida por SemVer 2.0.0.
- **R13-AC-016 — Empates de build metadata.** Dados tags de igual precedencia que difieren solo en build metadata: si apuntan a targets distintos, el resultado emite `unknown` y `semver-precedence-ambiguous`; si apuntan al mismo target, el resultado conserva todos los aliases ordenados y elige como representación el único tag sin build metadata o, si no existe, el byte-wise menor. En ningún caso se afirma precedencia entre los aliases.
- **R13-AC-017 — Remoto canónico.** Dado un remote llamado `origin` cuya URL no coincide con el repositorio canónico y otro remote coincidente, solo se consulta el coincidente. Si ninguno coincide y no hay URL canónica directa válida, se emite `unknown` y no se contacta `origin` por fallback.
- **R13-AC-018 — Ref no autorizada.** Branches, tags sin prefijo `v`, refs alias no canónicas como `latest` y strings coercibles pero no estrictas no participan en `publishedVersion`; cada rechazo queda cubierto por un diagnóstico agregado o trazable.
- **R13-AC-019 — Ancestry no demostrable.** Dado un OID remoto cuyo objeto no está en el checkout o cuyo tipo commit no está demostrado, cuando no existe SemVer comparable, entonces la dirección queda `unknown` con `ancestry-unavailable`; no se invoca `fetch`.
- **R13-AC-020 — Offline con caché.** Dada una caché válida y una consulta remota que falla por red/timeout, entonces `primaryState` es `offline`, la publicación procede de `remote-cache`, tiene `freshness=cached`, `stale=true`, timestamp original y edad calculada, y puede conservar hechos comparativos claramente históricos.
- **R13-AC-021 — Offline sin caché.** Dada una consulta remota no disponible y ninguna caché válida, entonces se emiten `offline` y `unknown`; la publicación es `availability=unavailable`, `freshness=unavailable`, sin fabricar `publishedVersion` ni timestamp remoto.
- **R13-AC-022 — Caché corrupta o ajena.** Dada una entrada corrupta, con schema desconocido, repositorio distinto, política distinta, timestamp inválido, SemVer inválida o target inválido, entonces se ignora con diagnóstico; no bloquea el comando ni se presenta como live.
- **R13-AC-023 — Clock skew.** Dado un timestamp futuro respecto al reloj inyectado, la edad no es negativa; se emite diagnóstico de reloj/procedencia y no se oculta la incertidumbre.
- **R13-AC-024 — Checkout no disponible.** Fuera de un repo o con HEAD unborn, el contrato usa `downloadedVersion.kind=unavailable`; la identidad global y la publicación remota siguen siendo consultables si sus fuentes existen.
- **R13-AC-025 — Identidad instalada ausente.** Si el artefacto de desarrollo no contiene una SemVer válida, la instalación queda `unknown` con procedencia de build; no toma la versión de package metadata del proyecto ni de `ManagedState`.
- **R13-AC-026 — Múltiples candidatos CLI.** Si PATH o el proyecto exponen candidatos adicionales, el resultado identifica el entrypoint efectivo y diagnostica los demás de forma saneada. No selecciona otra instalación ni modifica PATH.
- **R13-AC-027 — Fallos esperables e internos.** Una salida remota mal formada produce `remote-invalid-output` y contrato parcial; los demás fallos esperables también se expresan mediante diagnóstico. Solo una excepción interna no clasificada produce código de salida de fallo y nunca deja JSON parcial en stdout.
- **R13-AC-028 — Sin comandos mutantes ni optional writes.** Un command runner espía confirma que la ruta de status deshabilita optional locks/refrescos y solo usa operaciones Git permitidas de lectura. La lista observada no contiene `fetch`, `pull`, `merge`, `checkout`, `switch`, `reset`, `clean`, mantenimiento, actualización de refs ni escritura de configuración.
- **R13-AC-029 — Sin mutación Git.** Snapshots antes/después de refs, HEAD, índice, worktree y object database del fixture son equivalentes tras status, tanto en éxito como en error.
- **R13-AC-030 — Sin dependencia residual.** Una búsqueda estática y el grafo de build confirman que el módulo de identidad/versionado y el comando de status no importan `@axiom/user-workspace` ni conservan un literal alternativo como fallback.

### Permisos y visibilidad

- **R13-AC-031 — Escrituras limitadas.** El comando no escribe en checkout, `ManagedState`, configuración Git, ubicación del CLI ni archivos de proyecto. La única escritura admitida es la entrada de caché user-level después de una observación live íntegra.
- **R13-AC-032 — Entorno de usuario aislado.** Tests y E2E configuran HOME/directorio de datos temporal y demuestran que no leen ni escriben la caché o instalación reales del operador.
- **R13-AC-033 — Sin prompt de credenciales.** Las consultas remotas se ejecutan sin interacción. Un remoto que requiere autenticación produce diagnóstico tipado y no bloquea esperando input.
- **R13-AC-034 — Procedencia saneada.** URL con userinfo, token o query sensible se normaliza/sanea antes de cache y output. Snapshots de text/JSON no contienen el secreto fixture ni rutas de usuario completas innecesarias.
- **R13-AC-035 — Visibilidad del proyecto.** `ManagedState.runtime.version` solo aparece cuando el contexto de proyecto permite leerlo; su ausencia fuera de un proyecto no se trata como fallo de identidad global.
- **R13-AC-036 — Exit codes.** `updated`, `update-available`, `checkout-behind`, `ahead`, `installed-misaligned`, `unknown` y `offline` devuelven `0` cuando se emitió un contrato válido. Argumentos inválidos y fallos internos usan códigos no cero diferenciados conforme a la convención existente revalidada.

### Estados y efectos observables

- **R13-AC-037 — Unión cerrada.** Typecheck y tests exhaustivos fallan si un consumidor omite uno de los siete `ReleaseState.kind` requeridos.
- **R13-AC-038 — Orden sin pérdida.** `states` es determinista, no contiene duplicados estructuralmente idénticos y conserva todos los hechos aplicables. Se ordena por `offline` → `unknown` → `installed-misaligned` → `ahead` → `checkout-behind` → `update-available` → `updated`; dentro de un kind, por sujeto y clave estable de evidencia. `primaryState` es el kind del primer elemento.
- **R13-AC-039 — Evidencia resoluble.** Cada estado contiene todos los sujetos comparados, valores saneados, base y `provenanceIds`. Cada ID resuelve exactamente una entrada de la misma observación y no hay referencias huérfanas; cada identidad también enlaza su procedencia.
- **R13-AC-040 — Frescura observable.** Toda publicación disponible indica `live` o `cached`; toda evidencia cacheada tiene `stale=true`, timestamp y edad. Publicación ausente indica `unavailable`. El assessment conserva además su propio `observedAt`, distinto del timestamp histórico remoto.
- **R13-AC-041 — Dirty no cambia identidad.** Modificar el worktree del fixture solo altera `dirty`; no cambia tag/ref/commit ni hace que status limpie, stashée o descarte cambios.
- **R13-AC-042 — No apply implícito.** Después de cualquier estado, la versión del artefacto instalado, el contenido del checkout, PATH y `ManagedState` son idénticos a los valores previos.
- **R13-AC-043 — Contrato JSON.** `--json` produce un documento con `schemaVersion`, `observedAt`, `primaryState`, `states` con evidencia, las cuatro identidades separadas, `provenance` y `diagnostics`. Los datos no disponibles son nulos/uniones explícitas, no strings vacíos ni sustituciones.
- **R13-AC-044 — Presentación humana.** La salida humana nombra por separado “published”, “downloaded”, “installed” y “managed runtime”, indica offline/unknown sin lenguaje de certeza y no depende exclusivamente de color.
- **R13-AC-045 — Scope negativo.** No se incorporan comando apply, manifest v2 completo, imports de Launcher ni cambios de pipeline de release final en el diff del incremento.
- **R13-AC-046 — Validación de repositorio.** `npm run build`, tests Vitest focalizados y suite completa pasan, o cualquier fallo preexistente queda reproducido y clasificado antes del cambio sin atribuirle falsamente éxito al incremento.
- **R13-AC-047 — Revisión semántica.** Una revisión final traza cada criterio a código/test/evidencia y confirma que la consulta no muta Git/instalación/proyecto, las identidades no se colapsan y no se afirma una capacidad no implementada.
- **R13-AC-048 — Tags locales ambiguos.** Dado HEAD con dos tags canónicos de distinta precedencia, `downloadedVersion.kind` es `ambiguous-tag`, los candidatos aparecen ordenados, se emiten `checkout-tag-ambiguous` y `unknown`, y no se elige una versión por orden de enumeración. Si son aliases de igual precedencia y mismo commit, se aplica la regla determinista de aliases sin ambigüedad falsa.
