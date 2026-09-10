# r13-self-update-release-identity

> **Código**: INC-20260909-r13-self-update-release-identity
> **Estado**: En Borrador
> **Fecha de creación**: 2026-09-09
> **Tipo de cambio**: Fundacional de runtime y CLI, sin mutación de instalaciones

## Resumen

Este incremento define la base de ACC-077 y la identidad de release de ACC-080. Su resultado objetivo es un contrato único para distinguir la release publicada en el Git canónico, el checkout descargado, el CLI global realmente instalado y la versión de runtime gestionada por cada proyecto. También especifica el descubrimiento remoto de solo lectura, una caché offline con procedencia verificable y la evaluación tipada de estado necesaria para construir el self-update en incrementos posteriores.

El incremento es el **primero de una secuencia global 1 → 2 → 3 → 4 → 5**: (1) identidad y descubrimiento, este incremento; (2) descarga/aplicación y rollback; (3) manifest v2 completo; (4) experiencia Launcher; y (5) automatización y certificación final de release. No depende de los otros cuatro y los habilita al estabilizar las identidades y contratos que deberán consumir.

Su estado documental es **planificado/pending**. Este texto describe comportamiento requerido, no afirma que el runtime actual ya lo implemente. Core conserva los estados estructurales `specifying` para el incremento y `draft` para el plan hasta que una transición posterior, fuera de este trabajo, satisfaga sus reglas.

## Contexto y motivación

Un self-update seguro no puede tratar “la versión de Axiom” como un único literal. En una misma máquina pueden coexistir una release publicada, un checkout de desarrollo, un CLI instalado globalmente y proyectos gestionados con su propia versión de runtime. Confundir esas identidades produciría falsos positivos de actualización, decisiones sobre el repositorio equivocado o escrituras sobre estado de proyecto que no pertenece a la instalación global.

La base debe resolver además dos restricciones: el diagnóstico de releases no puede modificar el checkout durante una consulta y debe seguir siendo informativo cuando el remoto no esté disponible. Por ello, el descubrimiento Git se diseña como read-only y best-effort; la última observación remota válida se conserva con timestamp y procedencia, y cualquier dato no demostrable se presenta como `unknown` u `offline`, nunca como una certeza inferida.

ACC-077 aporta las acciones de self-update que necesitarán esta base. ACC-080 exige una identidad de release separada y trazable. Este incremento consume esas acciones como requisitos de entrada, pero no ejecuta todavía ninguna acción de descarga, sustitución o rollback.

## Alcance

### Incluido

- Un único CLI Axiom global por usuario como instalación efectiva y autoridad de `installedVersion`; los binarios locales de proyecto no constituyen una segunda instalación soportada.
- Separación explícita de:
  - `publishedVersion`: mayor release SemVer válida observada en los tags del remoto Git canónico;
  - `downloadedVersion`: identidad del checkout local expresada como tag exacto, ref más commit o commit aislado;
  - `installedVersion`: versión embebida en el artefacto del CLI global que está ejecutándose;
  - `ManagedState.runtime.version`: versión project-scoped del runtime gestionado, sin equivalencia implícita con las tres anteriores.
- Identidad de producto y build con producto, versión, tag, commit y datos de build, cada campo acompañado por su origen cuando corresponda.
- Un módulo de identidad propiedad del runtime/CLI que elimine el literal y cualquier lookup de versión desde `@axiom/user-workspace`.
- Política determinista de remoto y refs: solo el repositorio canónico configurado puede aportar `publishedVersion`; no se elige `origin` por conveniencia si no se demuestra que corresponde al remoto canónico.
- Descubrimiento Git remoto y local de solo lectura, sin `checkout`, `merge`, `pull`, `fetch` ni escritura de refs u objetos.
- Parsing y comparación SemVer estrictos, incluida la precedencia correcta de prereleases y la ausencia de precedencia del build metadata.
- Caché user-level de la última observación remota válida, con timestamp UTC, clave de repositorio/política, método y procedencia sanitizada.
- Contrato de evaluación con los estados tipados `updated`, `update-available`, `checkout-behind`, `ahead`, `installed-misaligned`, `unknown` y `offline`.
- Comando CLI planificado de consulta, read-only respecto de Git, instalación y proyecto, con una única escritura permitida de caché user-level tras una observación live válida.
- Tests unitarios e integración hermética con repositorios Git locales y un entorno de usuario aislado.

### Excluido

- Descargar, instalar, sustituir, aplicar o revertir una release.
- Cualquier comando `apply`, actualización automática, modificación del checkout o escritura en `ManagedState` como efecto de la consulta.
- El manifest v2 completo o la migración general de manifests; solo se define la identidad mínima que ese manifest futuro deberá consumir.
- Launcher UI, notificaciones gráficas, watchers, procesos residentes o interacción visual reactiva.
- El pipeline CI/CD definitivo de publicación, firma, promoción o certificación de releases.
- Canales de release, selección interactiva de versiones y políticas corporativas de rollout no exigidas por ACC-077/080.
- Cambios estructurales, transiciones de estado, índices o receipts de Axiom.Spec durante esta elaboración documental.

## Documentos del incremento

- [01_Requisitos.md](./01_Requisitos.md): requisitos funcionales, políticas de SemVer, remoto, caché y evaluación.
- [02_Cambios_Modelo.md](./02_Cambios_Modelo.md): tipos de identidad, observación, procedencia, estados y compatibilidad.
- [03_Criterios_Aceptacion.md](./03_Criterios_Aceptacion.md): escenarios verificables y efectos observables.
- [04_Interacciones_UI.md](./04_Interacciones_UI.md): contrato compartido de salida y superficie CLI read-only; no define Launcher.
- [context/README.md](./context/README.md): límites del contexto auxiliar admisible.
- [Plan de implementación](../../plans/PLAN-INC-20260909-r13-self-update-release-identity/README.md): orden de ejecución, gates, validación y rollback.

## Dudas abiertas

No hay dudas funcionales bloqueantes dentro del alcance. La revalidación inicial del plan debe resolver, sin alterar estos contratos:

1. las rutas y nombres reales de los módulos de CLI, estado gestionado, ejecución Git y almacenamiento user-level en el runtime actual;
2. la fuente existente que identifica inequívocamente el remoto Git canónico; si no existe, la implementación debe detenerse en el gate correspondiente en lugar de adivinar `origin`;
3. el mecanismo de build actual capaz de inyectar versión, tag y commit sin depender de `@axiom/user-workspace`;
4. la ubicación multiplataforma ya adoptada por Axiom para datos de usuario y la forma de detectar la instalación global efectiva.

Son comprobaciones de implementación, no autorización para cambiar el comportamiento especificado ni para ampliar el alcance.

## Decisiones funcionales cerradas

- La instalación soportada es un único CLI global user-level; el proyecto no es autoridad de la versión instalada.
- `publishedVersion`, `downloadedVersion`, `installedVersion` y `ManagedState.runtime.version` son identidades distintas y no se completan unas a partir de otras.
- La versión instalada procede del artefacto ejecutable; `@axiom/user-workspace` deja de ser fuente de identidad de producto o release.
- El remoto canónico debe quedar demostrado mediante configuración/identidad; un nombre de remote no demuestra por sí solo su canonicidad.
- El descubrimiento de estado no muta Git. Si una relación requiere objetos que no están disponibles localmente, el resultado es `unknown`; no se hace `fetch` para resolverla.
- Los estados son hechos tipados combinables y el contrato incluye un `primaryState` determinista para consumidores simples. Esto permite, por ejemplo, informar simultáneamente `updated`, `checkout-behind` e `installed-misaligned`.
- Una caída de red produce `offline`. La caché puede aportar una observación histórica, pero su antigüedad y procedencia siempre son visibles y nunca se presenta como observación live.
- Los estados de dominio no ejecutan una actualización ni implican por sí mismos un código de salida de error.

## Consolidación en la spec general

La consolidación estable queda pendiente de implementación y validación. Cuando el incremento cumpla sus criterios, deberán integrarse de forma concisa —sin copiar el historial del plan— las reglas duraderas en las especificaciones propietarias: requisitos funcionales de self-update, modelo operativo y de datos, interfaces CLI e integración Git/release. Los destinos candidatos son `specs/01_Requisitos_Funcionales.md`, `specs/03_Modelo_Operativo_y_Datos.md`, `specs/05_Interfaces_Operativas.md` y `specs/06_Integraciones_y_Capacidades.md`; la revalidación decidirá cuáles son realmente propietarios.

Esta elaboración no modifica esos archivos, no crea índices y no ejecuta transiciones Core.

## Estrategia E2E

La validación prevista utiliza repositorios Git temporales locales: un bare repo representa el remoto canónico; uno o más worktrees generan tags anotados y ligeros, ramas, commits ahead/behind y HEAD detached. El HOME/directorio de datos de usuario, el reloj y la identidad instalada se inyectan o aíslan para que ninguna prueba lea la instalación, caché o configuración reales del operador.

Los E2E deben probar al menos: release alineada; release remota posterior; checkout anterior; checkout ahead; CLI instalado distinto del checkout; tags inválidos; precedencia SemVer; remoto equivocado; remoto no disponible con y sin caché; caché obsoleta; ausencia de objetos para demostrar ancestry; y salidas text/JSON. Un command runner instrumentado debe demostrar que el descubrimiento no invoca comandos mutantes ni cambia refs, índice, worktree u object database.

## Trazabilidad y fuentes

- Objetivo de producto: base de ACC-077 e identidad de release de ACC-080.
- Dependencias funcionales: runtime actual y acciones ACC-077/080; no hay dependencia de los incrementos 2–5.
- Artefacto ejecutable: [PLAN-INC-20260909-r13-self-update-release-identity](../../plans/PLAN-INC-20260909-r13-self-update-release-identity/README.md).
- Evidencia futura requerida: diff de runtime, tests herméticos, `npm run build`, suite Vitest focalizada y completa, revisión semántica y registro de integración estable.

No se ha usado el estado actual del código como prueba de que estos requisitos estén implementados; esa comprobación pertenece a la fase 0 del plan.

## Estado de validación humana

**Pending.** La especificación queda lista para revisión humana de alcance y para revalidación técnica inicial. No se declara implementación, validación del runtime, consolidación estable ni cierre del incremento. Cualquier cambio de estado debe realizarse posteriormente mediante Axiom/Core y no mediante edición manual de estos documentos.

## Resultado de implementación y handoff

La implementación fue realizada exclusivamente en `Axiom` y revisada contra R13-AC-001..048. El runtime entrega identidades separadas, metadata de artefacto generada en TypeScript, resolución de entrypoint global/local, remoto canónico sin fallback a `origin`, Git read-only con allowlist y optional locks deshabilitados, caché histórica atómica y saneada, evaluator de estados/provenance, y `axiom self-update status` text/JSON.

La ruta pública compilada usa un bootstrap ligero para `self-update status` y `--version`; `full-index.js` conserva el CLI completo y `full-index-runtime.js` conserva su implementación original. El status lee `ManagedState.runtime.version` directamente sin `FilesystemStore`, migraciones ni escrituras. `apply`, manifest v2, Launcher, release CI y sustitución de instalación permanecen fuera de este incremento.

Archivos funcionales principales en `Axiom`: `packages/core/src/release-identity.ts`, `apps/cli/src/commands/self-update-status.ts`, `apps/cli/src/index-bootstrap.ts`, `scripts/generate-cli-build-metadata.mjs`, `scripts/prepare-cli-entrypoint.mjs` y sus pruebas focalizadas. Los `tsbuildinfo` modificados por compilación son artefactos generados y no forman parte del cambio funcional.

Validación ejecutada: `npm run build` raíz exit `0`; `npm run build --workspace @axiom/cli` exit `0`; focal Core/status `37/37`; compatibilidad legacy `30/30`; generador y smoke de proceso `3/3`; `get_errors` sin errores; `git diff --check` sin errores de whitespace, salvo warnings de normalización CRLF de derivados. La suite global terminó con exit `1`; `24` archivos y `54` tests coinciden con el baseline conocido, y el único fallo adicional observado en una ejecución concurrente (`freeze.test.ts`) pasó `8/8` aislado, por lo que no se atribuye a R13.

La revisión independiente final dio **GO técnico**. El receipt Core de verify es `2026-09-09T21-44-34.248Z-verify-success.json`, con hash `3eb62d3397fab8f59bccf48981d213c7ff1ee1e0b0e5bcefe29506f0bb487206`. El workflow Core fue transicionado a `verifying`; el archive lifecycle y la integración de este artifact se ejecutan mediante Core. La integración estable en `Axiom.Spec/specs/00..08` y `context/**` queda deliberadamente pendiente hasta el final del lote R-13.5.

Handoff habilitado: `INC-20260909-r13-self-update-state-cli-contract` puede consumir los exports actuales de `@axiom/core` (`ProductIdentity`, `InstalledCliIdentity`, `DownloadedVersion`, `PublishedVersion`, `ReleaseObservation`, `ReleaseAssessment`, provenance, diagnostics y SemVer), mantener `status`, `check` y `plan` read-only, tratar `install.json` solo como cache/receipt, y dejar `apply`/`recover` en `engine_unavailable` hasta el incremento 3. No debe redefinir identidad ni alterar el contrato de release identity.
