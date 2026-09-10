# Plan de certificación de releases, E2E y documentación

> **Código**: PLAN-INC-20260909-r13-self-update-release-certification
> **Estado**: draft
> **Artefacto origen**: INC-20260909-r13-self-update-release-certification
> **Versión de spec**: v1
> **Versión de plan**: p1

## Resumen ejecutivo

Este plan ejecuta el incremento 5/5 del lote R-13.5 y convierte ACC-080 en el gate final del self-update global. No diseña de nuevo identity/check, manifest/CLI, updater transaccional ni integración Launcher: recibe esos cuatro contratos, los ensambla contra una release Git del monorepo completo y demuestra que funcionan desde clean clone en Windows y POSIX.

La salida técnica es una decisión reproducible STOP/GO para un candidato identificado por versión SemVer, ref Git completa y commit exacto. El gate valida una única versión de producto, compatibilidad Node/npm, `private: true`, lockfile, build, typecheck, suites, instalador, CLI instalada, doctor, E2E A→B, rollback/recovery y paridad CLI/Launcher. Solo después del GO técnico se corrigen documentos runtime y se integra conocimiento estable en Axiom.Spec. No se crea ni publica un package npm; un package standalone queda fuera de alcance.

Este documento es un plan, no evidencia de ejecución. No registra tags remotos, commits candidatos, PASS, receipts ni transiciones actuales.

## Objetivo técnico

Entregar un comando de certificación automatizable que falle cerrado y que responda, con evidencia, a estas preguntas:

1. ¿La ref candidata cumple `refs/tags/v<productVersion>` y resuelve al commit exacto obtenido del remoto canónico?
2. ¿Existe una sola autoridad de versión y concuerdan raíz, workspaces gobernados, CLI compilada, build identity, manifest, status y provenance?
3. ¿El monorepo sigue siendo privado y la release se instala desde Git completo, no desde un tarball CLI incompleto?
4. ¿Un clean clone se reproduce con las versiones Node/npm declaradas y `npm ci` sin outputs previos?
5. ¿La validación estándar incluye Vitest y la suite nativa `node:test` del installer?
6. ¿Una instalación A puede descubrir, planificar y aplicar B de forma real en un entorno temporal y quedar observablemente en B?
7. ¿Cada fallo deja A intacta, hace rollback completo o entra en recovery explícito sin falso éxito?
8. ¿CLI y Launcher muestran el mismo estado y outcome en Windows y POSIX?
9. ¿Los documentos describen solo capacidades demostradas y el lote puede pasar a review/cierre mediante Core?

## Principios de ejecución

- **Expected behavior first:** los criterios AC-080-01..24 son el oráculo; no se ajustan para justificar el runtime actual.
- **Gate, no quinta implementación:** cualquier gap en identidad, estado, updater o Launcher vuelve al incremento propietario.
- **Fail closed:** mismatch, falta de evidencia o warning obligatorio produce STOP.
- **Hermeticidad:** Git de E2E, homes, prefix, cache, PATH y shims viven dentro de un temp root.
- **Verdad observada:** el gate ejecuta la CLI resultante y resuelve su launcher; no confía en target solicitado ni solo en manifest.
- **Sin publicación:** `private: true` permanece y no hay paso `npm pack`/`npm publish` de release.
- **Docs después de código validado:** ningún claim de update real se adelanta al E2E.
- **Estructura por Core:** metadata, status, receipts, índices, cierre y archivo no se editan manualmente.

## Dependencias de entrada y secuencia global

### Secuencia 1→5

| Orden | Incremento | Handoff requerido hacia 5/5 | Condición STOP |
|---|---|---|---|
| 1 | `INC-20260909-r13-self-update-release-identity` | Resolución de remoto/ref/commit, SemVer, published/downloaded/installed y cache con frescura. | Ref libre, comparación incorrecta, cache como autoridad o fetch mutante. |
| 2 | `INC-20260909-r13-self-update-state-cli-contract` | Manifest v2, writer/lock, operaciones exclusivas, envelope JSON, streams y exit codes. | Dry-run mutante, manifest como autoridad, JSON no parseable o lost update. |
| 3 | `INC-20260909-r13-self-update-transactional-updater` | Plan inmutable, staging/apply, smokes, activación, rollback, journal y recover. | Apply ficticio, no-op como installed, rollback parcial oculto o falta de recovery. |
| 4 | `INC-20260909-r13-self-update-launcher-integration` | DTO/runner común, confirmación, progreso, helper externo y cierre/reinicio. | Lógica duplicada, apply al arranque, módulos A/B mezclados o outcome reinterpretado. |
| 5 | Este incremento | Validator, clean clone, E2E, matrices, docs, evidence ledger y decisión final. | Cualquier handoff anterior ausente o no certificable. |

### Handoff de entrada

Antes de editar runtime para 5/5, el builder prepara una matriz de interfaces con ruta, símbolo/comando, versión de contrato y evidencia del incremento propietario. No hace falta que 5/5 copie sus specs; sí debe poder invocar cada boundary sin mocks en al menos un E2E.

El gate se detiene si un predecesor no ha alcanzado su condición de entrega o si sus pruebas focales no pasan. El defecto se registra contra el incremento propietario; 5/5 permanece bloqueado y no introduce un segundo mecanismo.

### Handoff de cierre del lote

1. Los builders de 1–4 entregan contratos congelados y gaps conocidos.
2. El builder de 5/5 ejecuta validator, clean-clone, E2E, fault/platform matrix y completa el ledger.
3. Un reviewer independiente reproduce los gates críticos y revisa la frontera de no duplicación.
4. Con GO técnico, se corrige documentación runtime y se ejecuta de nuevo el gate documental/técnico aplicable.
5. Se integra únicamente conocimiento estable en `specs/00..08` y el contexto propietario.
6. El reviewer emite veredicto final del lote; cualquier blocker devuelve STOP.
7. Un operador autorizado usa Axiom Core para cierre, receipts, indexado y archivo en orden de dependencia, dejando este incremento para el final. No se mueven carpetas ni se cambian estados manualmente.
8. El handoff final lista los cinco IDs, commits de implementación, evidence IDs, decisiones de compatibilidad, resultado por plataforma y ubicación de docs; no inventa releases upstream.

## Alcance incluido

- Autoridad de versión única y proyecciones de build/runtime.
- Validador `SemVer ↔ ref ↔ commit ↔ versión ↔ CLI`.
- Guard de `private: true` y de ausencia de canal package standalone.
- Decisión y declaración verificable de Node/npm.
- Scripts raíz para separar/agregar Vitest, installer `node:test`, validator, clean clone y E2E.
- Clean clone real desde candidato sin `dist`, `node_modules` ni cache productor.
- Smokes por ruta absoluta y por shim/binlink temporal.
- Repos bare A/B, updater real, provenance, no-op, faults y recovery.
- Matriz Windows/Linux y límites Node/npm; macOS opcional como cobertura POSIX adicional.
- Workflow CI de certificación si al comenzar sigue sin existir un workflow equivalente.
- Evidence ledger y revisión independiente.
- Documentación runtime exacta e integración estable en Axiom.Spec después de validar.
- Cierre y archivo por Core, nunca en este plan de forma manual.

## Alcance excluido

- Implementar de nuevo ACC-077, ACC-078, ACC-079 o el incremento Launcher.
- Publicar un package npm, tarball standalone, binario nativo o installer de sistema.
- Retirar `private: true` o hacer públicos workspaces internos.
- Restaurar TUI o diseñar una nueva superficie visual.
- Auto-update al arranque, update silencioso o bypass de confirmación.
- Firmas, SBOM o attestations externas no exigidas por otra spec.
- Crear/fetchear tags upstream como parte de esta redacción o afirmar su existencia.
- Modificar metadata, status, receipts o índices manualmente.

## Baseline técnico a reconfirmar

La inspección read-only usada para preparar este plan encontró estas superficies. El builder las reconfirma al iniciar porque no son evidencia de ejecución futura:

| Superficie | Baseline observado | Uso en el plan |
|---|---|---|
| `package.json` raíz | `build: tsc -b ...`, `test: vitest run`, `typecheck: tsc -b`, `doctor`, `readiness:first-project`; `private: true`. | Autoridad de scripts y punto de agregación estándar. |
| `package-lock.json` | Lockfile del monorepo. | Única entrada permitida para `npm ci`. |
| `vitest.config.ts` | Excluye `scripts/**`. | Mantener Vitest y añadir runner `node:test` explícito. |
| `scripts/install-global.mjs` | Instala shim `.cmd` en Windows y usa npm link/binlink en POSIX. | Boundary real que el E2E debe ejecutar y endurecer desde los predecesores. |
| `scripts/install-global.test.mjs` | Suite `node:test` focal. | Integrar en validación estándar. |
| `scripts/verify-first-project-readiness.mjs` | Smoke de proceso sobre proyecto temporal y doctor. | Extender/reutilizar sin confundirlo con instalación global aislada. |
| `apps/cli/src/index.ts` | Commander y versión CLI. | Eliminar literal independiente y proyectar versión raíz. |
| `apps/cli/tests/self-update.test.ts` | Suite focal de self-update. | Regresión; no sustituye E2E real. |
| `packages/user-workspace/tests/self-update.test.ts` | Manifest user-level. | Regresión de provenance/estado. |
| `packages/installer/tests` | Instalación project-scoped. | Regresión separada del installer global. |
| `packages/launcher/tests/launcher.test.ts` y tests app-side | Contratos Launcher existentes. | Regresión y paridad; distinguir Launcher UI del shim/binlink. |
| `packages/versioning/tests`, `apps/cli/tests/rollback.test.ts` | Versioning/rollback project-scoped. | Regresión; no usar como prueba de rollback del CLI global. |
| `packages/doctor/tests` | Checks doctor. | Regresión más smoke instalado. |
| CI | No se identificó workflow equivalente durante la inspección. | Reconfirmar; crear uno mínimo solo si sigue ausente. |
| `docs/cli/self-update.md` | No se identificó. | Crear después de GO técnico e indexar. |

## Plan de ejecución detallado

### Fase 0 — Freeze de requisitos y preflight de dependencias

**Acciones**

1. Leer las versiones finales de los cuatro incrementos anteriores y sus receipts mediante Core, sin editar sus artifacts.
2. Construir la tabla de handoff con símbolos, archivos, comandos y outcomes propietarios.
3. Ejecutar pruebas focales de cada predecesor antes de añadir el gate.
4. Identificar el remoto canónico configurado sin cambiarlo y distinguirlo de remotos de fixture.
5. Capturar `git status --short`, commit de trabajo y baseline de scripts únicamente como contexto local; no presentarlos como release.
6. Confirmar que no existe un cambio paralelo sobre versiones, installer o workflows que invalide el plan.

**STOP**

- Algún contrato no está implementado o no tiene test focal.
- La CLI y Launcher usan servicios distintos.
- Apply todavía acepta una versión arbitraria como autoridad o no cambia realmente de release.
- No existe recovery verificable.

**Evidencia:** `E-01`, `E-02`.

### Fase 1 — Decidir y declarar compatibilidad Node/npm

**Acciones**

1. Probar clean clone con las versiones candidatas de Node/npm en Windows y Linux.
2. Elegir un límite inferior y una versión soportada reciente; no inferir Node desde `@types/node`.
3. Declarar rangos exactos en `package.json#engines.node` y `package.json#engines.npm` o en la autoridad versionada aprobada.
4. Fijar `packageManager` si se usa para reproducir npm/lockfile.
5. Hacer que el validator compare declaración, versión efectiva y matriz CI.
6. Documentar política de elevación/deprecación de versiones sin prometer soporte no probado.

**STOP**

- No hay valores exactos acordados.
- `npm ci` modifica lockfile o una celda declarada no compila.
- La matriz CI no representa los límites declarados.

**Evidencia:** `E-03`, `E-04`.

### Fase 2 — Autoridad de versión, release validator y package guard

**Archivos runtime previstos**

- `package.json` y manifests de workspace gobernados.
- `apps/cli/src/index.ts`.
- La autoridad existente `packages/versioning/src/version.ts` o un único módulo generado elegido tras impacto; nunca dos fuentes.
- Nuevo `scripts/release/validate-release.mjs`.
- Nuevo `scripts/release/validate-release.test.mjs`.

**Contrato del validator**

Inputs del job: `AXIOM_RELEASE_REF`, `AXIOM_RELEASE_COMMIT` y el remoto canónico ya configurado por ACC-077. En modo source-tree para pull requests puede validar coherencia de versiones/private sin certificar una ref publicada. En modo release todos los inputs son obligatorios.

Checks, en orden:

1. Árbol limpio y commit candidate exacto.
2. Ref completa con patrón `refs/tags/v<SemVer>`; extraer SemVer sin normalizaciones silenciosas.
3. Fetch explícito de la ref desde el remoto canónico y peel a commit.
4. Igualdad entre commit resuelto y `AXIOM_RELEASE_COMMIT`/HEAD candidate.
5. Igualdad entre SemVer de ref y `package.json#version` raíz.
6. Igualdad o proyección declarada de versiones de workspaces gobernados.
7. Generación/importación de build identity y `axiom --version` exacta.
8. `private: true` en raíz y workspaces; ausencia de publish config/scripts que conviertan npm en canal soportado.
9. Existencia/coherencia de lockfile y contrato Node/npm.
10. Salida JSON estable para CI y diagnóstico humano por stderr; exit no cero ante cualquier mismatch.

El validator nunca crea tags, hace push, cambia refs, publica packages ni corrige versiones. Solo observa y rechaza.

**Package guard**

- Enumerar workspaces desde la raíz, no desde una lista duplicada.
- Rechazar manifest gobernado sin `private: true`.
- Rechazar una configuración de release que apunte a `apps/cli`, `npm pack` o `npm publish` como artefacto.
- Comprobar que el installer y assets requeridos existen en el monorepo candidate.
- Incluir tests negativos que muten copias temporales de manifests, nunca el checkout real.

**STOP**

- Queda un literal de versión independiente sin guard.
- El validator puede pasar sin fetch/ref/commit en modo release.
- El guard se limita al root y omite workspaces.

**Evidencia:** `E-05`, `E-06`.

### Fase 3 — Integrar installer y definir comandos estándar

**Cambio propuesto en scripts raíz**

Los nombres se fijan para evitar que `scripts/**` quede oculto por Vitest:

```json
{
  "test:vitest": "vitest run",
  "test:installer": "node --test scripts/install-global.test.mjs",
  "test:release-validator": "node --test scripts/release/validate-release.test.mjs",
  "test:release-e2e": "node --test scripts/release/self-update-release.e2e.test.mjs",
  "test": "npm run test:vitest && npm run test:installer",
  "release:validate": "node scripts/release/validate-release.mjs",
  "release:clean-clone": "node scripts/release/verify-clean-clone.mjs",
  "release:certify": "npm run release:validate && npm run release:clean-clone && npm run test:release-e2e"
}
```

El builder puede ajustar nombres solo si existe una convención equivalente; debe conservar comandos focales y hacer que `npm test` ejecute Vitest más installer. `test:release-validator` se añade al gate y puede añadirse al estándar si su coste es unitario. El E2E pesado no tiene que ejecutarse en cada `npm test`, pero sí en `release:certify` y CI de release.

**Regresiones obligatorias**

- `npm test` falla si falla la suite installer.
- `npm run test:installer` ejecuta exactamente `node:test` una vez.
- Vitest sigue sin intentar interpretar `scripts/install-global.test.mjs`.
- Los comandos funcionan en cmd/PowerShell y shell POSIX porque la orquestación compleja reside en Node, no en quoting shell.

**Evidencia:** `E-07`.

### Fase 4 — Harness de clean clone

**Archivo previsto:** `scripts/release/verify-clean-clone.mjs`.

**Topología**

- `tempRoot/source`: source local o remote candidate de solo lectura.
- `tempRoot/clone`: clone nuevo checkout detached al commit candidato.
- `tempRoot/home`, `userprofile`, `prefix`, `npm-cache`, `tmp` y `artifacts`.
- Entorno child allowlisted; nunca heredar un `axiom` host en PATH.

**Algoritmo**

1. Crear temp root con API Node y registrar cleanup seguro.
2. Clonar/fetchear el commit candidate; no copiar `node_modules`, `dist` ni `.tsbuildinfo`.
3. Verificar `git status --porcelain` vacío y que HEAD coincide con candidate.
4. Fijar HOME/USERPROFILE/prefix/cache/temp/PATH internos.
5. Registrar `node --version` y `npm --version`; validar rangos.
6. Ejecutar `npm ci`; comprobar lockfile y tracked tree sin cambios.
7. Ejecutar `npm run typecheck` y `npm run build`.
8. Ejecutar `npm test` y las suites focales listadas en “Comandos de validación”.
9. Ejecutar explícitamente `node --test scripts/install-global.test.mjs` aunque el agregado ya lo cubra, para que el ledger muestre el gate nominal.
10. Instalar con `node scripts/install-global.mjs --install` usando prefix/home temporal.
11. Resolver el launcher real; Windows verifica `.cmd`, contenido/quoting y target; POSIX resuelve binlink/realpath y permisos.
12. Ejecutar por ruta absoluta y launcher: `--version`, `--help` y `doctor` sobre proyecto temporal preparado.
13. Confirmar build identity, versión y provenance.
14. Ejecutar uninstall/cleanup si el instalador lo soporta y demostrar que no hay residuos fuera del temp root.
15. Emitir resultado estructurado; limpiar temp root solo después de conservar la evidencia autorizada.

**Reglas**

- No usar `which axiom`/`where axiom` sin verificar que el resultado está dentro del temp root.
- No aceptar un shim no vacío como smoke suficiente.
- No hacer fallback de `npm ci` a `npm install`.
- Timeout y signals son fallos con proceso hijo terminado y log preservado.

**Evidencia:** `E-08`, `E-09`.

### Fase 5 — E2E hermético release A→B

**Archivos previstos**

- `scripts/release/self-update-release.e2e.test.mjs`.
- Un helper pequeño bajo `scripts/release/` solo si evita duplicar creación de repos/env; no se crea un framework genérico.

**Fixture**

1. Copiar únicamente archivos trackeados necesarios a un productor temporal.
2. Crear release A y B como commits locales distintos con versiones SemVer de fixture y tags que siguen `v<SemVer>`.
3. Crear remoto bare local y publicar allí solo las refs del fixture.
4. Instalar A desde clone completo con npm cache/prefix temporales.
5. Ejecutar el CLI A por ruta absoluta y shim/binlink.
6. Configurar el remoto local como canónico dentro del fixture.

Las versiones/tags A/B se generan durante la prueba y se identifican expresamente como fixtures; no son claims sobre upstream.

**Recorrido happy path**

1. `status` observa A instalada.
2. `check` hace fetch local y descubre B sin checkout/apply.
3. `plan` devuelve target B, ref/commit y pasos; snapshot de filesystem demuestra read-only.
4. CLI apply exige confirmación y delega en helper externo.
5. Updater prepara B, ejecuta `npm ci`, build y smokes reales.
6. Activación cambia launcher de A a B de forma atómica o recuperable.
7. Proceso nuevo devuelve B en `--version`; `--help` y doctor pasan.
8. Manifest/status/provenance coinciden con B y con `git rev-parse` del clone activo.
9. Launcher repite la observación y produce DTO/outcome equivalente; se prueba cierre/reinicio.
10. B→B devuelve no-op, no invoca fases mutantes y conserva bytes durables permitidos.

**Aislamiento**

- Variables obligatorias: `HOME`, `USERPROFILE`, `npm_config_prefix`, `npm_config_cache`, `TMPDIR`, `TEMP`, `TMP` y PATH controlado.
- Windows usa paths con espacios y valida `%USERPROFILE%\.local\bin\axiom.cmd` o la ruta acordada por el installer final.
- POSIX valida `<prefix>/bin/axiom`, permisos, symlink/realpath y ausencia de bin host.
- El Git E2E usa paths locales/file transport autorizado; todo intento HTTP/SSH falla.
- El npm cache es temporal. El setup puede poblarlo desde el lockfile; la fase A→B se ejecuta con la política offline elegida para no depender de red durante self-update.

**Evidencia:** `E-10`, `E-11`, `E-12`, `E-13`.

### Fase 6 — Fault matrix, rollback y recovery

Fault injection se expone solo mediante dependencias/test hooks internos; una variable de producción no puede saltarse validaciones. Cada caso parte de A verificada y compara checkout, launcher, build identity, manifest y journal antes/después.

| ID | Punto de fallo | Outcome obligatorio | Estado durable esperado |
|---|---|---|---|
| F-01 | Fetch local no disponible/offline | offline/failed según operación; nunca installed | A; cache marcado stale. |
| F-02 | Ref inválida, SemVer inválido o tag ausente | failed | A, sin staging mutable. |
| F-03 | Ref resuelve a commit distinto o versión no coincide | failed | A; candidato rechazado. |
| F-04 | Checkout dirty/detached/divergente no gobernado | failed | Estado original intacto; sin stash/reset. |
| F-05 | Lock ya adquirido / dos updaters | busy/failed tipado | A; un solo owner. |
| F-06 | `npm ci` falla en staging | failed | A; staging limpiable, manifest previo. |
| F-07 | Build o carga de módulos falla | failed | A; sin activar B. |
| F-08 | `--version` devuelve valor distinto | failed | A; B no activada. |
| F-09 | `--help` o doctor smoke falla/timeout/signal | failed | A; logs conservados. |
| F-10 | Escritura/swap de shim/binlink falla | failed o recovery-required | A restaurada; si no, journal explícito. |
| F-11 | B activa pero persistencia de manifest falla | failed con rollback o recovery-required | A completa o estado recuperable; nunca success+warning. |
| F-12 | Interrupción entre backup, activación y persistencia | recovery-required al reiniciar | Journal/backup suficientes; ninguna suposición de versión. |
| F-13 | Rollback de commit/build/launcher falla | recovery-required | Evidencia preservada; exit no cero. |
| F-14 | Cleanup falla tras B completamente verificada | outcome definido y documentado | B solo si identidad/manifest ya son consistentes; residuos reportados. |
| F-15 | Journal stale o recovery repetido | recover idempotente | A o B verificada; nunca degradar. |
| F-16 | B→B | unchanged/noop | Cero fases mutantes y bytes estables. |
| F-17 | Launcher cancela antes de confirmar | cancelled/noop | A; sin home/manifest nuevo. |
| F-18 | Launcher no reinicia o intenta usar módulos A | failed | No certificar B hasta proceso limpio. |

Para cada fila se prueba CLI y, cuando aplica, Launcher. Windows y POSIX cubren al menos F-06, F-08, F-10, F-11, F-12, F-13 y F-16; el resto puede particionarse sin perder semántica.

**Evidencia:** `E-14`, `E-15`, `E-16`.

### Fase 7 — Matriz de plataforma y CI

### Matriz obligatoria

| SO | Launcher | Node/npm | Suite mínima |
|---|---|---|---|
| Windows runner soportado | `.cmd`, paths con espacios | límite inferior declarado | validator, clean clone, installer, happy A→B, faults críticos, recovery. |
| Windows runner soportado | `.cmd` | versión reciente declarada | mismo gate; puede dividir faults manteniendo cobertura. |
| Linux runner soportado | binlink/realpath | límite inferior declarado | validator, clean clone, installer, happy A→B, faults críticos, recovery. |
| Linux runner soportado | binlink/realpath | versión reciente declarada | mismo gate. |
| macOS, si se adopta | binlink/realpath | al menos una combinación declarada | cobertura POSIX adicional; no sustituye Linux. |

Los números exactos de Node/npm se fijan en Fase 1 y quedan literales en workflow/config. No se usan etiquetas móviles como única evidencia de compatibilidad.

### Workflow previsto

Si la reconfirmación sigue mostrando ausencia de CI equivalente, crear `.github/workflows/release-certification.yml` con:

- `pull_request`: source-tree validator, estándar y tests del validator; sin certificar una ref publicada.
- `workflow_dispatch`: ejecución manual de candidato con ref/commit explícitos, sin publish.
- `push` de tags compatibles: fetch completo y gate de release; el workflow no crea tags ni publica npm.
- `fetch-depth: 0` o fetch exacto de ref para poder validar tag↔commit.
- matriz Windows/Linux y Node/npm declarada.
- timeout por job, cancelación de duplicados y uploads de evidencia solo al sistema CI aprobado.
- permisos mínimos/read-only; ninguna credencial de package registry.
- `release:certify` como comando común local/CI para evitar YAML con lógica de negocio.

Un workflow existente equivalente se extiende en vez de duplicarse. El builder documenta la decisión.

**Evidencia:** `E-17`, `E-18`.

### Fase 8 — Documentación runtime, solo después de GO técnico

### Archivos exactos

| Archivo | Cambio esperado |
|---|---|
| `README.md` | Explicar un único CLI global distribuido desde release Git del monorepo; retirar claims ambiguos de package/TUI/update ficticio. |
| `docs/README.md` | Enlazar instalación y self-update vigentes; no presentar `npx axiom` como canal soportado. |
| `docs/overview.md` | Alinear estado de upgrade/self-update y superficies CLI/Launcher. |
| `docs/installation.md` | Separar setup de desarrollo (`npm link` si sigue útil) de instalación soportada; explicar Node/npm, clone Git, build, shim/binlink y uninstall. |
| `docs/cli/README.md` | Indexar `self-update.md`; marcar TUI como histórica. |
| `docs/cli/self-update.md` | Nueva guía operativa completa. |
| `docs/cli/tui.md` | Conservar valor histórico sin comandos vigentes `npx axiom tui` ni promesas activas. |
| `docs/cli/doctor.md` | Distinguir ejemplos de salida de evidencia actual y enlazar smoke/troubleshooting de self-update. |

### Contenido mínimo de `docs/cli/self-update.md`

1. Modelo: único CLI user-level y efecto global sobre proyectos.
2. Release Git: `productVersion`, `releaseRef`, `releaseCommit`, published/downloaded/installed.
3. Compatibilidad Node/npm y preflight.
4. `status` y `check`, incluida frescura offline.
5. `plan`, confirmación y naturaleza read-only.
6. `apply`, fases, progreso y reinicio del Launcher.
7. `--json`, stdout/stderr y exit codes.
8. No-op y cómo verificar versión/commit/provenance.
9. Rollback y `recovery-required`; operación `recover`.
10. Windows `.cmd` y POSIX binlink.
11. Troubleshooting: dirty/detached/diverged, lock, fetch, npm ci, build, version mismatch, doctor, manifest corrupto y recovery.
12. Límites: no npm package, no TUI, no auto-update, no side-by-side.

### Gate documental

- Búsqueda dirigida de `npx axiom`, `@axiom/cli`, `npm pack`, `npm publish`, tarball, TUI y afirmaciones “update real”.
- Clasificar cada match como vigente, desarrollo o histórico; eliminar/reescribir solo claims engañosos.
- Ejecutar links/checks documentales existentes; si no hay checker, inspeccionar enlaces relativos y comandos contra `--help` validado.
- No pegar outputs con versiones/tags de fixture como si fueran upstream.

**Evidencia:** `E-19`.

### Fase 9 — Review independiente, integración canónica y archivo

#### Review independiente

El reviewer no debe ser quien hizo la implementación principal. Recibe diff, spec, ledger y comandos, y revisa:

- ausencia de una segunda implementación de identity/state/updater/Launcher;
- validator fail-closed y fetch/ref/commit real;
- una sola versión sin literales divergentes;
- `private: true` y ausencia de package standalone;
- hermeticidad y resolución real del launcher;
- cobertura de faults antes/después de activación;
- rollback/recovery completos en Windows/POSIX;
- paridad CLI/Launcher y JSON/exit codes;
- documentación posterior a evidencia y sin claims no demostrados;
- cambios limitados al alcance aprobado.

Debe reproducir al menos release validator negativo/positivo de fixture, `npm test`, installer test, una celda clean clone, happy A→B, manifest-write failure, rollback failure/recovery y búsqueda documental. Cualquier blocker o evidencia no reproducible devuelve STOP.

#### Integración en `specs/00..08`

Solo con review GO:

- `00_Resumen_Ejecutivo.md`: modelo de entrega Git y CLI global único.
- `01_Requisitos_Funcionales.md`: release identity y operaciones de self-update.
- `02_Requisitos_No_Funcionales.md`: reproducibilidad, soporte Node/npm, hermeticidad, recuperación y plataformas.
- `03_Modelo_Operativo_y_Datos.md`: versión/ref/commit, provenance y estados durables.
- `04_Flujos_SDD_y_Ciclo_de_Vida.md`: check→plan→confirm→apply→restart/recover y gate de release.
- `05_Interfaces_Operativas.md`: CLI/Launcher, JSON, streams, códigos y troubleshooting.
- `06_Integraciones_y_Capacidades.md`: Git remoto canónico, npm como toolchain —no canal—, installer/doctor.
- `07_Gobierno_y_Seguridad.md`: refs confiables, no clobber, locks, aislamiento y package guard.
- `08_Glosario.md`: product/published/downloaded/installed version, release ref/commit, provenance, rollback, recovery.
- `context/`: actualizar solo el archivo propietario de arquitectura/operaciones; no duplicar ni editar índices.

Cada archivo se marca en el ledger como “actualizado” o “revisado sin cambio”. No se copia la fault matrix completa a la spec canónica.

#### Core y archivo

1. Reconfirmar comandos disponibles con `axiom --help`, `axiom increment --help` y `axiom plan --help`; no adivinar sintaxis de transición.
2. Ejecutar `axiom index validate` y `axiom doctor` cuando el lote esté listo.
3. Si Core detecta índice stale, usar el comando Core autorizado para reconstruirlo; nunca editar el índice.
4. Invocar mediante Core las transiciones de review/cierre/archivo y conservar sus receipts.
5. Archivar en orden 1→5, con certificación al final, solo si cada artifact cumple sus reglas de cierre.
6. Si Core rechaza una transición, no mover carpetas ni cambiar status; resolver la causa y repetir el comando autorizado.

Este plan no ejecuta ninguna de esas transiciones ni fija una sintaxis no verificada para close/archive.

**Evidencia:** `E-20`, `E-21`, `E-22`.

## Matriz E2E resumida

| Escenario | CLI | Launcher | Windows | POSIX | Resultado requerido |
|---|---:|---:|---:|---:|---|
| A status/check descubre B | sí | sí | sí | sí | B disponible; A activa. |
| Plan B sin mutación | sí | sí | al menos una celda | al menos una celda | Plan idéntico y snapshot estable. |
| A→B exitoso | sí | sí | sí | sí | B activa; version/ref/commit/provenance coherentes. |
| B→B | sí | sí | sí | sí | noop/unchanged, cero mutación. |
| Cancelación | sí | sí | una celda | una celda | A intacta. |
| Offline/cache stale | sí | sí | una celda | una celda | sin apply ni falsa frescura. |
| Fallo preactivación | sí | representativo | faults críticos | faults críticos | A intacta, failed. |
| Fallo postactivación + rollback | sí | representativo | sí | sí | A restaurada, failed. |
| Rollback fallido + recover | sí | sí | sí | sí | recovery-required y convergencia idempotente. |
| Paths con espacios | sí | sí | sí | recomendable | launcher correcto. |
| Dos updaters | sí | refleja busy | una celda | una celda | lock exclusivo. |
| Manifest corrupto/schema futuro | sí | sí | una celda | una celda | rechazo/recovery según ACC-079. |

“Representativo” no permite omitir semántica Launcher: se puede reducir combinatoria si el runner común ya está cubierto, pero happy path, recovery y parity deben ejecutarse en ambos.

## Comandos de validación

### Baseline y estándar

```text
npm ci
npm run typecheck
npm run build
npm test
node --test scripts/install-global.test.mjs
npm run readiness:first-project
```

### Suites focales

```text
npx vitest run apps/cli/tests/self-update.test.ts
npx vitest run packages/user-workspace/tests/self-update.test.ts
npx vitest run packages/installer/tests
npx vitest run packages/launcher/tests/launcher.test.ts
npx vitest run packages/versioning/tests apps/cli/tests/rollback.test.ts
npx vitest run packages/doctor/tests
```

El builder añade los tests específicos que creen los incrementos 1–4; esta lista no los sustituye.

### Gate nuevo esperado

```text
npm run test:release-validator
npm run release:validate
npm run release:clean-clone
npm run test:release-e2e
npm run release:certify
```

`release:validate` recibe `AXIOM_RELEASE_REF` y `AXIOM_RELEASE_COMMIT` en modo release; el job configura el remoto canónico según ACC-077. No se muestran valores ficticios en este plan.

### Smokes dentro del harness

```text
node apps/cli/dist/index.js --version
node apps/cli/dist/index.js --help
node apps/cli/dist/index.js doctor
```

Además, el harness ejecuta los mismos argumentos contra la ruta absoluta del shim/binlink temporal. No se usa el `axiom` del PATH host.

### Calidad del diff y documentación

```text
git diff --check
axiom index validate
axiom doctor
```

`git status --short` y diff scoped sirven para revisar alcance. Los comandos Axiom anteriores son validaciones; las transiciones Core se dejan al operador de cierre.

## Evidence ledger requerido

La tabla define evidencia a producir; no afirma resultados actuales.

| ID | Criterios | Productor | Evidencia/command | Condición de GO |
|---|---|---|---|---|
| E-01 | AC-080-01 | builder 1–4 / builder 5 | Matriz de handoff con rutas y contracts. | Cero gap sin owner. |
| E-02 | AC-080-01,15 | builder 5 | Tests focales predecesores y mapa servicio común CLI/Launcher. | Todos reproducibles. |
| E-03 | AC-080-06 | builder 5 | Decisión Node/npm, ranges versionados. | Valores exactos aprobados. |
| E-04 | AC-080-06,17 | CI | Matriz de versiones efectivas por SO. | Todas las celdas requeridas pasan. |
| E-05 | AC-080-02,03,04 | builder 5 | Validator positivo/negativo, JSON y exit codes. | Todo mismatch rechazado. |
| E-06 | AC-080-05 | builder 5 | Package guard sobre copias temporales de manifests. | `private: true`; cero canal npm. |
| E-07 | AC-080-08 | builder 5 | `npm test` + `npm run test:installer`. | Installer incluido y focal. |
| E-08 | AC-080-07 | clean-clone harness | Transcript `npm ci`→typecheck→build→suites. | Exit 0 y tree coherente. |
| E-09 | AC-080-18 | clean-clone harness | Rutas/realpath, version/help/doctor. | Bin candidate, no host. |
| E-10 | AC-080-09 | E2E | Topología A/B y transcript happy path. | B activa. |
| E-11 | AC-080-10 | E2E | Manifest/status/provenance comparados con Git/build. | Igualdad exacta. |
| E-12 | AC-080-11 | E2E | Snapshot B→B. | No-op sin mutación. |
| E-13 | AC-080-15 | E2E | DTO/outcome CLI vs Launcher y restart. | Paridad semántica. |
| E-14 | AC-080-12 | fault suite | F-01..F-09. | A intacta; sin falso éxito. |
| E-15 | AC-080-13 | fault suite | F-10..F-14. | Rollback o recovery explícito. |
| E-16 | AC-080-14 | fault suite | Reinicio + recover repetido. | Convergencia/idempotencia. |
| E-17 | AC-080-16,17 | CI/E2E | Env redactado, network guard, snapshots por SO. | Cero escape del temp root. |
| E-18 | AC-080-07,17 | CI | Workflow, job URLs/receipts aprobados. | Windows/POSIX verdes. |
| E-19 | AC-080-19,20 | docs reviewer | Diff y búsqueda de claims. | Docs alineadas, links válidos. |
| E-20 | AC-080-21,22 | reviewer independiente | Informe, repro crítica, hallazgos. | Cero blocker abierto. |
| E-21 | AC-080-23 | integrador canónico | Checklist `00..08/context`. | Cada archivo actualizado o revisado. |
| E-22 | AC-080-24 | operador/Core | Receipts de validate/close/archive. | Core acepta; certificación última. |

Cada entrada ejecutada registra fecha, plataforma, arquitectura, Node/npm, source commit, release ref/commit cuando aplique, comando/argv, cwd, exit code, duración, stdout/stderr o hash/ruta aprobada y vínculo al criterio. Reintentos se conservan; no reemplazan silenciosamente fallos.

## STOP/GO final

### STOP inmediato

- Falta cualquiera de los cuatro handoffs.
- No hay rango Node/npm exacto o una celda declarada falla.
- Ref no fue obtenida del remoto canónico o no concuerda con commit/versión.
- Existe más de una autoridad de versión.
- Algún workspace pierde `private: true` o el flujo usa package/tarball CLI.
- Clean clone requiere outputs del checkout productor, usa `npm install` o modifica lockfile.
- `npm test` omite el installer.
- El launcher ejecutado no está dentro del temp root.
- E2E usa red Git externa, home/PATH host o deja residuos fuera del root temporal.
- Apply devuelve éxito sin observar B real o un no-op se informa como installed.
- Un fallo postactivación deja estado ambiguo sin `recovery-required`.
- CLI y Launcher divergen, Launcher mezcla módulos o JSON/exit codes rompen ACC-079.
- Windows o POSIX carecen de evidencia obligatoria.
- Docs prometen capacidades no validadas.
- Ledger incompleto, review con blockers o integración estable no revisada.

### GO técnico

Todos los AC-080-01..18 pasan con evidencia; el candidate validator y las matrices son reproducibles; no hay mutación host ni publicación npm. Este GO habilita la fase documental, no cierra el incremento.

### GO documental

AC-080-19..20 pasan después del GO técnico y el gate no revela regresiones tras los cambios documentales.

### GO de cierre

AC-080-21..24 pasan; review independiente sin blockers, integración estable completada y Core acepta las transiciones. Si falta una condición, el artifact no se archiva.

## Roles impactados

- **Builder (`axiom-code-builder`)**: implementa validator, scripts, tests, CI y docs runtime; produce evidencia sin ejecutar lifecycle manual.
- **Owners de incrementos 1–4**: entregan/fijan contratos y corrigen gaps de su dominio.
- **Reviewer independiente**: revisa y reproduce gates; no comparte autoría principal.
- **Integrador canónico**: actualiza `specs/00..08`/`context` solo tras GO técnico y documental.
- **Operador Axiom/Core**: valida estructura, registra receipts, cierra y archiva.
- **Usuario de CLI/Launcher**: recibe un flujo único, explícito y recuperable; no se le expone lógica de certificación interna.

## Estrategia E2E

La estrategia combina tres capas, ninguna sustituible por otra:

1. **Unidades/contratos:** validator, SemVer, package guard, manifest, updater, Launcher y helpers de installer.
2. **Clean clone:** reproduce el candidate completo y prueba instalación/smokes sin outputs previos.
3. **A→B transaccional:** prueba el sistema instalado, faults y recovery mediante un remoto Git local.

La cobertura se distribuye para controlar coste, pero el release gate ejecuta al menos una cadena completa por cada celda Windows/POSIX y Node/npm obligatoria. Los faults críticos que cruzan activación se repiten en ambas familias de plataforma.

## Riesgos y dependencias

| Riesgo | Mitigación / owner |
|---|---|
| Los predecesores siguen siendo plantillas o no entregan runtime. | STOP en Fase 0; owner del incremento correspondiente. |
| Duplicación actual de `0.1.0` deriva. | Fuente raíz única, generación/proyección y validator negativo. |
| `npm test` no ejecuta `scripts/**`. | Separar `test:vitest`/`test:installer` y agregarlos. |
| Installer actual enlaza checkout y puede dar falso positivo. | Predecesor 3 corrige boundary; E2E verifica target y versión real. |
| Windows usa home distinto para backup/shim. | Helper de paths único del predecesor; tests con HOME/USERPROFILE separados. |
| POSIX encuentra otro `axiom` del host. | PATH allowlisted, prefix temporal y realpath assertion. |
| npm/Node no están declarados. | Decisión bloqueante en Fase 1 y matrix literals. |
| CI no existe o no tiene Windows. | Crear workflow mínimo reusable; lógica permanece en scripts Node. |
| Fault injection se filtra a producción. | Dependency injection/test-only module; validator impide bypass en build release. |
| E2E demasiado lento/flaky. | Repos locales, cache temporal preseed, timeouts deterministas, no sleeps; conservar gate completo en release. |
| Docs se adelantan al runtime. | Fase 8 bloqueada por GO técnico. |
| Se confunde package npm con npm como toolchain. | Package guard y wording explícito: npm instala dependencias, Git entrega producto. |
| Se archiva con evidencia incompleta. | Ledger + review + Core; certificación se archiva última. |

## Cambios en contexto técnico

Durante ejecución, el plan puede crear notas acotadas en su `context/` sobre compatibilidad, topología E2E, fault map y review. La verdad estable se integra después en el archivo propietario de `Axiom.Spec/context/` que describa distribución/operaciones. No se crean índices, receipts ni duplicados manuales.

El runtime debe documentar detalles operativos en `docs/cli/self-update.md`; la spec canónica conserva invariantes y comportamiento, no comandos internos del harness.

## Consolidación y archivado

- Consolidar una sola vez tras validación y review.
- Revisar los nueve archivos `specs/00..08`; cambiar solo los propietarios del conocimiento y anotar los revisados sin cambio en el ledger.
- Actualizar un contexto propietario existente o crear uno solo mediante el flujo autorizado si no existe; Core gestiona indexado.
- No copiar logs, matrices completas ni historial de fallos a las specs generales.
- Ejecutar cierre/archivo de cada incremento mediante Core, en secuencia y con 5/5 al final.
- Si no surge conocimiento estable para un archivo, registrar “revisado sin cambio”; no forzar edits cosméticos.
- El reporte de cierre del lote incluye limitaciones residuales y deja package standalone como futuro incremento, no como deuda oculta de ACC-080.

## Validaciones y gates

Los gates son acumulativos:

- **G0 Handoff:** contratos 1–4 disponibles.
- **G1 Identity:** validator, versión única, ref/commit y package guard.
- **G2 Reproducibility:** Node/npm y clean clone.
- **G3 Standard validation:** build/typecheck/Vitest/installer/focal suites.
- **G4 Installed smoke:** version/help/doctor y target launcher.
- **G5 Transaction:** A→B, provenance y no-op.
- **G6 Resilience:** fault matrix, rollback y recover.
- **G7 Parity/platform:** CLI+Launcher, Windows+POSIX, matriz Node/npm.
- **G8 Documentation:** runtime/canonical sin claims obsoletos.
- **G9 Independent review:** cero blockers.
- **G10 Core closure:** receipts, index validation y archive autorizados.

Un gate fallido invalida los posteriores; no se documenta ni archiva alrededor de un fallo.

## Fuentes y supuestos

### Fuentes

- ACC-077..080 en `plans/PLAN-REVISION-INTEGRAL-AXIOM.md`.
- Incrementos y planes R-13.5 1–4.
- `INC-20260909-r13-self-update-release-certification/01_Requisitos.md` y `03_Criterios_Aceptacion.md`.
- Baseline runtime enumerado en este plan.

### Supuestos que deben verificarse

- Los incrementos 1–4 se implementarán antes de ejecutar el gate final.
- El remoto canónico se configura mediante ACC-077 y puede consultarse sin mutar checkout.
- El updater expone fault boundaries o dependencias inyectables suficientes para pruebas deterministas.
- El Launcher puede ejecutarse/reiniciarse en harness sin UI manual.
- La plataforma CI aprobada permite Windows y Linux.
- Existe un mecanismo Core para cierre/archivo; su sintaxis se obtiene del CLI instalado, no se inventa aquí.

### Hechos que este plan no afirma

- Que exista un tag remoto concreto.
- Que una release candidata haya pasado.
- Que los cuatro incrementos previos estén cerrados.
- Que las versiones Node/npm actuales sean el contrato definitivo.
- Que haya un workflow CI vigente.
- Que se haya ejecutado cualquier comando, review, integración o transición Core durante la redacción.