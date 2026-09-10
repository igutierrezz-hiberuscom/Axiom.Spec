# Plan de identidad y descubrimiento de releases de self-update

> **Código**: PLAN-INC-20260909-r13-self-update-release-identity
> **Estado**: draft
> **Artefacto origen**: [INC-20260909-r13-self-update-release-identity](../../increments/INC-20260909-r13-self-update-release-identity/README.md)
> **Versión de spec**: v1
> **Versión de plan**: p1

## Resumen ejecutivo

Este plan implementa la base de ACC-077 y la identidad de release de ACC-080 en el runtime Axiom. Entrega cuatro identidades separadas (`publishedVersion`, `downloadedVersion`, `installedVersion` y `ManagedState.runtime.version`), identidad de producto/tag/commit/build, descubrimiento Git no mutante, caché offline con procedencia y evaluación tipada de estado. El resultado visible es una consulta CLI que no modifica Git, instalación ni proyecto; su única escritura permitida es la caché user-level de una observación remota válida.

Es el **incremento 1 de 5** y debe ejecutarse primero:

1. **Identidad y descubrimiento de release — este plan.**
2. Descarga/aplicación transaccional y rollback.
3. Manifest v2 completo y adopción por consumidores de release.
4. Launcher UI sobre contratos estabilizados.
5. CI de release, publicación y certificación final.

No depende de los incrementos 2–5; consume el runtime actual y las acciones ACC-077/080, y habilita a los cuatro siguientes. Su estado documental permanece **planificado/pending**. Nada en el plan afirma que el código actual ya tenga estos componentes. Core conserva `specifying`/`draft`; este plan no ejecuta transiciones.

## Objetivo técnico

Construir una frontera de aplicación única que responda “¿qué producto ejecuto, qué release está publicada, qué checkout tengo y qué runtime declara este proyecto?” con evidencia explícita y sin modificar instalación, Git o proyecto. La implementación debe:

- obtener la versión del CLI desde el artefacto global efectivo y retirar toda autoridad de `@axiom/user-workspace`;
- resolver el remoto canónico sin fallback arbitrario a `origin`;
- enumerar releases remotas sin escribir refs/objetos y ordenarlas con SemVer 2.0.0 estricto;
- diferenciar OID remoto de commit demostrado, sin traer objetos para completar evidencia;
- representar HEAD como tag, tags ambiguos, ref, commit o unavailable;
- conservar la última observación remota válida en caché user-level atómica, versionada y saneada;
- producir un `ReleaseAssessment` fechado, con hechos combinables, evidencia enlazada y orden determinista;
- exponer el resultado por consulta CLI text/JSON sin lógica duplicada;
- demostrar comportamiento y no mutación mediante tests herméticos con repos Git locales.

## Alcance incluido

- Contrato v1 de tipos, serialización, provenance, diagnostics, freshness y estados.
- Política SemVer y de tags canónicos, aliases, tags anotados/ligeros y targets no verificables/no-commit.
- Resolución de identidad del producto y del CLI global efectivo.
- Separación semántica de `ManagedState.runtime.version`, sin migración de manifest v2.
- Puerto y adapter Git local/remoto con prompting y optional locks deshabilitados.
- Resolución demostrable del repositorio canónico y sanitización de URLs.
- Caché offline con timestamp, clave, procedencia, edad y `stale=true` para toda evidencia histórica.
- Evaluación de `updated`, `update-available`, `checkout-behind`, `ahead`, `installed-misaligned`, `unknown` y `offline`.
- Registro del subcomando de consulta y presenters humano/JSON.
- Tests unitarios, integración Git local, snapshots/schemas CLI, build y suite completa.
- Revisión semántica contra 48 criterios y propuesta de integración estable en Axiom.Spec.

## Alcance excluido

- Download/apply, sustitución del ejecutable, rollback, locks de instalación o recuperación transaccional.
- Manifest v2 completo, migración general de manifests y canales/rings de rollout.
- Launcher, componentes visuales, notificaciones, watchers o procesos residentes.
- Firma, publicación, promoción y CI final de releases.
- Escrituras en checkout, refs, object database, `ManagedState`, PATH o instalación desde status.
- Reestructuración general del monorepo o introducción de un framework nuevo de actualización.
- Edición manual de metadata, índices, receipts o status de artefactos Axiom.

## Roles impactados

- **Builder de runtime:** implementa tipos, puertos, adapters, CLI y tests en `Axiom`; sigue [role-builder.md](./role-builder.md).
- **Reviewer semántico:** contrasta comportamiento con R13-AC-001…048, separaciones de autoridad y ausencia de efectos mutantes.
- **Maintainer de release/build:** confirma inyección de version/tag/commit/build e identidad canónica del remoto; no diseña CI final.
- **Owner de Axiom.Spec:** integra únicamente conocimiento estable después de GO final y usa Core para cualquier transición posterior.

No se necesita un rol de Launcher ni de updater transaccional en este incremento.

## Rutas y símbolos probables

Las rutas son **candidatas**, no descripción del código actual. Fase 0 debe reemplazarlas por rutas reales antes de editar. Si no existe una capa candidata, se usa la capa actual más próxima sin crear paquetes especulativos.

| Responsabilidad | Ruta probable a confirmar | Símbolos objetivo probables |
|---|---|---|
| Tipos/reglas | `packages/core/src/**/self-update/` o módulo release actual | `ProductIdentity`, `PublishedObservation`, `DownloadedVersion`, `InstalledCliIdentity`, `ReleaseState`, `ReleaseAssessment` |
| SemVer | utility existente en `packages/core/src/**` | `parseReleaseTag`, `compareReleaseVersions`, `selectPublishedRelease` |
| Caso de uso | capa application/core existente | `getSelfUpdateStatus`, `ReleaseStatusService` |
| Puerto Git | contracts/ports existentes del core | `ReleaseDiscovery`, `GitReadPort` |
| Adapter Git Node | adapter/infrastructure existente bajo `packages/*` | `GitReleaseDiscovery`, `discoverDownloadedVersion`, `discoverPublishedVersion` |
| Caché user-level | servicio de paths/storage existente | `ReleaseCache`, `readReleaseCache`, `writeReleaseCacheAtomic` |
| Build/CLI identity | package CLI y configuración build | `BUILD_IDENTITY`, `getInstalledCliIdentity` |
| Estado gestionado | declaración/reader real de `ManagedState` | lectura de `runtime.version`, sin escritura desde status |
| Comando CLI | `apps/cli/src/**/commands/` o registry real | `self-update status`, presenters text/JSON |
| Dependencia indebida | consumidores de `@axiom/user-workspace` | retirar lookup/literal y cubrir con tests |

## Plan de ejecución ordenado

### Fase 0 — Revalidación inicial y baseline

1. Trabajar desde la raíz del runtime `Axiom`; confirmar branch, worktree y ausencia de cambios ajenos que puedan mezclarse. No modificar Axiom.Spec en esta fase.
2. Leer README, package scripts, workspaces, configuración TypeScript/Vitest y AGENTS aplicables.
3. Localizar con búsquedas dirigidas:

   ```text
   rg -n "ManagedState|runtime\.version" packages apps
   rg -n "@axiom/user-workspace|installedVersion|publishedVersion|downloadedVersion" packages apps
   rg -n "product.*version|release.*tag|commit|build" packages apps
   rg -n "ls-remote|rev-parse|symbolic-ref|describe|execFile|spawn" packages apps
   rg -n "XDG_|LOCALAPPDATA|APPDATA|homedir|cache" packages apps
   rg -n '"bin"|self-update|update.*status' package.json packages apps
   ```

4. Registrar rutas/símbolos reales de entrypoint, fuente de versión, imports user-workspace, ManagedState, runner Git, paths user-level, command registry, build/packaging y tests vecinos. Persistir `context/runtime-symbol-map.md` solo si aporta valor de handoff.
5. Identificar la fuente del repositorio canónico y probar que puede compararse con remotes saneados sin depender del nombre `origin`.
6. Confirmar cómo inyectar metadata de artefacto sin acceso al workspace del usuario.
7. Capturar baseline, sin corregir fallos ajenos:

   ```text
   npm run build
   npx vitest run
   ```

8. Confirmar convenciones de exit codes, JSON stdout/stderr, user data path, package boundaries y soporte de `GIT_OPTIONAL_LOCKS=0`/`--no-optional-locks`.

**Gate G0 — STOP/GO**

- **GO:** rutas propietarias, autoridad canónica, metadata de artefacto y seams Git/filesystem/clock confirmados.
- **STOP:** remoto canónico adivinable únicamente, identidad dependiente de user-workspace, entrypoint global indemostrable, optional writes imposibles de desactivar o diff no aislable. No introducir fallback ni ampliar scope.

### Fase 1 — Contratos de dominio y SemVer

1. Añadir en core identidades, `PublishedObservation`, provenance enlazada, diagnostics, estados con `StateEvidence` y assessment con `observedAt`.
2. Implementar parser estricto de tag `v` + SemVer y selector de release. Reutilizar librería aprobada; si falta una dependencia, detenerse para aprobar y fijar versión exacta.
3. Implementar empates: targets distintos → ambiguous/unknown; mismo target → aliases, preferencia por tag sin build metadata y luego orden byte-wise, conservando todos.
4. Implementar `DownloadedVersion.kind=ambiguous-tag` y sus candidatos ordenados.
5. Distinguir `GitObjectId` remoto de `GitCommit` demostrado.
6. Definir orden de states por prioridad, luego subject y clave estable de evidencia; deduplicar solo igualdad estructural.
7. Añadir exhaustive checks y unit tests SemVer, aliases, ambigüedad, serialización unavailable y referencias de provenance sin huérfanos.

**Gate G1 — STOP/GO**

- **GO:** tipos puros y tests codifican exactamente la spec, incluidos offline sin cache y timestamp del assessment.
- **STOP:** identidad colapsada, SemVer textual, freshness global falsa, evidencia no enlazable o acoplamiento a Git concreto/CLI/Launcher/apply.

### Fase 2 — Identidad de artefacto, instalación global y ManagedState

1. Crear/adaptar módulo de identidad del runtime/CLI. El build genera o inyecta producto, versión, tag, commit demostrado, build id/timestamp y dirty cuando exista evidencia.
2. Leer metadata desde el artefacto efectivo, no package.json del proyecto ni `@axiom/user-workspace`.
3. Implementar `getInstalledCliIdentity`; candidatos PATH/locales son diagnóstico saneado, no otra autoridad.
4. Migrar consumidores del literal/lookup antiguo y eliminar el fallback.
5. Exponer `ManagedState.runtime.version` como contexto opcional project-scoped sin cambiar persistencia ni escribirlo desde status.
6. Añadir tests con dos proyectos y un artefacto global, más builds con metadata completa/parcial y provenance resoluble.

**Gate G2 — STOP/GO**

- **GO:** identidad instalada reproducible desde artefacto, estable entre proyectos y sin dependencia residual de user-workspace.
- **STOP:** metadata no fiable, necesidad de manifest v2 o detección global que requiera modificar PATH/instalación.

### Fase 3 — Descubrimiento Git no mutante

1. Definir puerto con canonical repository identity, cwd opcional, timeout y runner; devolver observaciones tipadas.
2. Resolver/normalizar remotes y sanear credenciales sin cambiar identidad.
3. Ejecutar todos los procesos Git con `GIT_OPTIONAL_LOCKS=0` o equivalente y prompting deshabilitado. Usar argv estructurado y timeout.
4. Descubrir remoto con equivalente a `git ls-remote --tags`, capturando refs directas/peeled sin actualizar refs locales.
5. Conservar target OID. Si el objeto está local, verificar tipo mediante lectura; rechazar no-commit. Si no está, mantener `unverified` y no usar ancestry.
6. Descubrir HEAD/ref/tag/dirty localmente sin refrescos opcionales. Detectar tags exactos ambiguos y aliases.
7. Comparar ancestry solo entre commits ya disponibles; ausencia produce `ancestry-unavailable`.
8. Tipar no repo, unborn HEAD, Git ausente, timeout, red, auth, salida remota inválida y targets inválidos/no verificables.
9. Instrumentar allowlist de comandos/argumentos y negar fetch/pull/merge/checkout/switch/reset/clean/mantenimiento/escrituras.

**Gate G3 — STOP/GO**

- **GO:** fixtures prueban refs/targets y snapshots confirman que refs, índice, worktree y object database no cambian.
- **STOP:** cualquier escenario exige traer objetos, optional writes, autenticación interactiva o mutación Git.

### Fase 4 — Caché offline y procedencia

1. Reutilizar paths user-level y crear namespace keyed por repository identity + refs policy + schema; nunca usar directorio de proyecto.
2. Validar schema, key, timestamp, SemVer, aliases y target de cada lectura. Corrupta/ajena se ignora con diagnóstico.
3. Escribir solo una observación live válida mediante temp + replace/rename atómico.
4. Inyectar clock/filesystem. Al leer cache: mantener timestamp original, calcular age, fijar `freshness=cached` y `stale=true`. No existe TTL en este incremento.
5. Representar fallo remoto sin cache como published unavailable, no como live/cached.
6. Sanear antes de persistir; añadir secreto canary.
7. Probar cache válida, inexistente, corrupta, schema/repo/policy distintos, timestamp futuro y offline con/sin cache.

**Gate G4 — STOP/GO**

- **GO:** no se toca estado de proyecto, persistencia es atómica, toda cache es histórica explícita y offline sin cache es representable.
- **STOP:** cache tomada como autoridad live, umbral stale oculto, secreto persistido o migración destructiva.

### Fase 5 — Orquestación y estados

1. Implementar caso de uso con identidad, Git, cache, ManagedState reader y clock inyectados.
2. Separar recolección de evidencia y evaluator puro.
3. Generar `observedAt` al componer assessment, independiente del timestamp remoto/cache.
4. Hacer que cada identidad y estado enlace provenance IDs existentes; validar ausencia de referencias huérfanas.
5. Codificar matriz, orden normativo, desempates y primary igual al primer kind.
6. No comparar refs por nombre ni OIDs por desigualdad; usar solo SemVer, commit ancestry o identity demostrables.
7. Probar todas las combinaciones, incluyendo updated + behind + misaligned, offline + cache, offline sin cache, aliases y ambiguity.

**Gate G5 — STOP/GO**

- **GO:** cada estado tiene caso positivo/negativo, evidence completa, orden estable y ninguna salida activa apply.
- **STOP:** estado único con pérdida, provenance huérfana, identidad rellenada o runtime de proyecto confundido con CLI.

### Fase 6 — CLI y contratos de salida

1. Registrar `axiom self-update status` según registry real; resolver conflictos de nombre sin cambiar semántica.
2. Implementar presenters text/JSON sobre el mismo assessment. JSON incluye schema, assessment timestamp, observation, states/evidence, provenance y diagnostics.
3. Aislar stdout JSON; enviar diagnóstico operativo a stderr.
4. Aplicar convención existente: estados de dominio con contrato válido son éxito; uso inválido/fallo interno quedan diferenciados.
5. Mostrar target no verificado como target, nunca como commit; mostrar toda cache como stale/histórica.
6. Verificar help sin apply e imports sin Launcher.
7. Añadir command tests/snapshots para cada estado primario, combinaciones, outside-repo, offline con/sin cache y saneamiento.

**Gate G6 — STOP/GO**

- **GO:** text/JSON equivalentes, automatizables, provenance resoluble, cero secretos y única escritura de cache permitida.
- **STOP:** mutación de instalación/proyecto/Git, prompt, apply, UI acoplada o renderer que recalcula estado.

### Fase 7 — Pruebas herméticas, validación y revisión

1. Crear helper temporal con bare remote, author repo y checkout sujeto; generar commits, tags anotados/ligeros, tags a non-commit objects, aliases, ambigüedades, detached/dirty y objetos deliberadamente ausentes.
2. Aislar HOME/XDG/APPDATA/LOCALAPPDATA, clock, artifact identity y command runner.
3. Cubrir como mínimo:
   - `1.9.0` frente a `1.10.0`, prerelease/final, build aliases compatibles/incompatibles;
   - target commit demostrado, target unverified y target non-commit;
   - remoto canónico/no coincidente y salida remota inválida;
   - tag/ref/commit/ambiguous-tag/unavailable;
   - siete estados, combinaciones, orden y provenance links;
   - offline con cache (`stale=true`) y sin cache (published unavailable);
   - CLI global estable entre proyectos;
   - URL con secreto canary;
   - optional locks deshabilitados e invariantes de no mutación.
4. Ejecutar cada test focalizado confirmado en fase 0 mediante `npx vitest run` con su ruta real y después:

   ```text
   npm run build
   npx vitest run
   git diff --check
   ```

5. Ejecutar smoke de pack/build no publicador solo si ya existe. No crear/publicar release.
6. Comparar con baseline; corregir regresiones y separar fallos previos.
7. Buscar exclusiones: user-workspace en identidad/status, Git mutante, Launcher, apply, manifest-v2 y CI fuera de alcance.

**Gate G7 — STOP/GO final**

- **GO:** focalizadas, build y suite verdes —o fallos previos inequívocamente reproducidos—, 48 AC trazados y revisión independiente GO.
- **STOP:** red/HOME reales, metadata no reproducible, mutación, fallo nuevo, evidencia/provenance incompleta o scope posterior.

## Estrategia E2E

Los tests no usan GitHub, Internet ni tags reales. Bare remotes locales y worktrees temporales permiten validar refs directas/peeled, targets, aliases y ancestry con Git real. Los casos offline usan URL local inexistente o runner controlado; nunca desconectan la máquina. La instalación usa entrypoint fixture.

Para probar no mutación se capturan antes/después `show-ref`, HEAD, status porcelain, hash/mtime del índice y listado/conteo de objetos. El runner registra argv/env, exige optional locks deshabilitados y falla ante comandos mutantes. La combinación evita que un mock oculte escrituras.

## Riesgos y dependencias

| Riesgo | Impacto | Mitigación / STOP |
|---|---|---|
| Sin fuente inequívoca del remoto | Consulta del producto equivocado | STOP G0; nunca fallback a `origin` |
| Metadata no llega al artefacto | Installed depende del workspace | Prototipo G2 y smoke; STOP sin literal duplicado |
| Edge cases SemVer/aliases | Release incorrecta | Librería estándar, tabla y política determinista |
| OID remoto se confunde con commit | Provenance/ancestry falsos | Target separado; verificar solo local; unknown si falta prueba |
| Git hace optional writes | Viola read-only | Deshabilitar locks/refrescos + snapshots |
| `ls-remote` pide auth | CLI bloqueado | no-prompt, timeout, error tipado |
| Cache se toma como fresca | Decisión falsa | cached siempre stale + timestamp/age |
| Cache filtra secreto | Riesgo seguridad | sanitización canary |
| Global CLI difiere por plataforma | Identidad inconsistente | path abstraction y tests; no cambiar PATH |
| ManagedState crece a manifest v2 | Scope creep | solo reader; STOP |
| Concurrencia con incrementos 2–5 | Contratos inestables | este va primero; aislar diff |

Dependencias permitidas: runtime actual, Git local, infraestructura de test/build y acciones ACC-077/080. No se acepta dependencia de código de incrementos 2–5.

## Rollback

Organizar cambios reversibles por capa: (1) desregistrar command/presenters; (2) retirar orquestador/evaluator; (3) retirar cache/adapter Git; (4) revertir consumidores a la última fuente previa solo como rollback del conjunto, no fallback permanente; (5) retirar tipos e inyección de identidad.

La cache es namespaced/opcional y versiones anteriores la ignoran. Status no modifica instalación, Git o ManagedState, por lo que no hay rollback de datos de proyecto. Si packaging falla, revertir antes de publicar. Usar reversión explícita, no reset destructivo ni edición manual de estado documental.

## Revisión semántica

Un reviewer distinto debe reconstruir entrypoint → evidencia → assessment → presenter y comprobar:

- una autoridad y provenance resoluble por identidad;
- ManagedState project-scoped y user-workspace fuera de identidad;
- remote/ref/alias policy determinista;
- target OID no llamado commit sin prueba;
- SemVer y tags locales/remotos ambiguos correctos;
- optional locks deshabilitados y cero mutaciones salvo cache;
- offline con/sin cache representable y cache siempre histórica;
- assessment timestamp separado de remote timestamp;
- states con sujetos/valores/base/provenance, ordenados sin pérdida;
- text/JSON sobre el mismo modelo;
- ausencia de apply, manifest v2 completo, Launcher y CI final;
- R13-AC-001…048 enlazados a test/evidencia.

Una contradicción vuelve a su fase y reabre gate; no se “corrige” aceptando el bug en snapshots.

## Cambios en contexto técnico

Puede añadirse un mapa de símbolos confirmados y una matriz breve de no mutación. No se copian logs ni contenido canónico. Código/tests viven en `Axiom`; conocimiento estable se consolida después en Axiom.Spec.

## Consolidación y archivado

Tras G7 y revisión GO:

1. actualizar resultado/evidencia del propio incremento sin editar metadata/status manualmente;
2. identificar conocimiento estable de requisitos, modelo, interfaces e integración Git/release;
3. integrarlo, sin historial, en propietarios candidatos `specs/01_Requisitos_Funcionales.md`, `specs/03_Modelo_Operativo_y_Datos.md`, `specs/05_Interfaces_Operativas.md` y `specs/06_Integraciones_y_Capacidades.md` mediante flujo permitido;
4. validar enlaces/índices con comandos Axiom apropiados si el owner lo solicita;
5. pedir transición/cierre mediante Core solo al cumplir todas las condiciones.

Este trabajo documental no ejecuta esos pasos fuera de las carpetas autorizadas ni archiva nada. Si no hay conocimiento estable adicional, debe quedar explícito con evidencia.

## Validaciones y gates

| Gate | GO exige | STOP exige acción |
|---|---|---|
| G0 Revalidación | rutas, autoridad, metadata, optional-lock seam | escalar ambigüedad; no codificar |
| G1 Modelo | tipos, freshness, evidence y SemVer exhaustivos | corregir contrato |
| G2 Identidad | artefacto global independiente de proyecto | resolver build sin fallback |
| G3 Git | fixtures locales y cero mutaciones | eliminar operación/optional write |
| G4 Cache | atomicidad, stale/provenance, cero secretos | corregir storage/semántica |
| G5 Estados | matriz, evidence y orden completos | corregir evaluator |
| G6 CLI | text/JSON equivalentes y no mutantes | retirar prompt/apply/UI/mutación |
| G7 Final | build + tests + 48 AC + review | mantener pending y documentar |

Ningún gate parcial autoriza comenzar incrementos 2–5. Solo G7 GO y handoff estable los habilitan.

## Condición de done

El incremento puede proponerse como done solo cuando:

- ACC-077/080 trazan a las cuatro identidades;
- todos los R13-REQ y R13-AC tienen implementación/evidencia;
- installed no depende de `@axiom/user-workspace`;
- tests herméticos cubren SemVer/aliases, targets, Git, cache, provenance, estados y CLI;
- no hay mutación de Git/proyecto/instalación;
- build, focalizadas y suite cumplen G7;
- revisión semántica independiente da GO;
- no entró alcance 2–5;
- conocimiento estable se integró o se justificó no aplicable;
- resultados/fallos previos están documentados;
- Core realiza posteriormente cualquier transición autorizada.

Hasta entonces permanece planificado/pending aunque esta documentación esté completa.

## Fuentes y supuestos

- Fuente normativa: [README](../../increments/INC-20260909-r13-self-update-release-identity/README.md), [requisitos](../../increments/INC-20260909-r13-self-update-release-identity/01_Requisitos.md), [modelo](../../increments/INC-20260909-r13-self-update-release-identity/02_Cambios_Modelo.md), [criterios](../../increments/INC-20260909-r13-self-update-release-identity/03_Criterios_Aceptacion.md) e [interacción CLI](../../increments/INC-20260909-r13-self-update-release-identity/04_Interacciones_UI.md).
- Entradas: acciones ACC-077/080 y runtime actual.
- Validaciones conocidas del runtime: `npm run build` y `npx vitest run`.
- A revalidar: Git disponible; identidad canónica; path user-level; registry/exit codes; soporte efectivo para deshabilitar optional writes.
- Las rutas/símbolos son probables hasta fase 0 y no se presentan como existentes.
