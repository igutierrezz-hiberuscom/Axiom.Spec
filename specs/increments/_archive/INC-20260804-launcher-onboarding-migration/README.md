# Increment: Launcher de onboarding y migracion

Status: closed
Date: 2026-08-04
Action: ACC-006 / R-13
Dependencies: INC-20260804-cli-commands-package-output

## Goal

Convertir el launcher servido por `axiom app` en la interfaz guiada completa
para instalar, unirse, configurar y adoptar un workspace Axiom, incluyendo la
migracion de spec y contexto tecnico, con preview, confirmacion y resultados
honestos. El launcher debe delegar en los comandos canonicos y no crear una
segunda implementacion de negocio.

## Context

El launcher ya ofrece `/launcher/`, seleccion de proyecto, install/join,
roles, catalogo de acciones, doctor, craft/execute y materializacion de
adapters. `app-onboarding.ts` ya tiene confirm gate y wiring real para
adapters y execution mode. La auditoria ACC-006 encontro que la adopcion de
spec/contexto y el setup completo de workspace no estan expuestos como flujo
visual completo y que algunas opciones solo se muestran como referencia.

Los comandos existentes `workspace setup`, `workspace adopt`,
`bootstrap from-legacy-sdd` y `bootstrap from-context` son la fuente de verdad.
Sus garantias de no-clobber, provenance, idempotencia y dry-run deben
conservarse desde la UI.

## Scope

- Ampliar el contrato de onboarding y sus endpoints para cubrir proyecto
  nuevo, proyecto existente, workspace setup, adopcion de spec, adopcion de
  SDD/control repo y ingest de contexto.
- Añadir al launcher la seleccion del proyecto, rutas de repos y opciones
  reales disponibles, mostrando advertencias cuando una opcion no tenga una
  ruta ejecutable.
- Exponer preview read-only antes de cualquier mutacion y confirmation gate
  explicito para cada operacion mutante.
- Delegar en `runWorkspaceSetup`, `runWorkspaceAdopt`, `runBootstrapFromLegacySdd`
  y `runBootstrapFromContext` o sus wrappers publicados; no duplicar sus
  writes ni inventar estados.
- Mostrar resultado con archivos creados, saltos por colision, warnings,
  provenance y rutas efectivamente usadas; refrescar registry y doctor.
- Cubrir single-repo, multi-repo y proyectos migrados con pruebas HTTP y, si
  existe fixture estable, E2E sobre la CLI construida.

## Non-goals

- No implementar aqui el ciclo SDD de incrementos/bugs/planes/roles; eso es
  `INC-20260804-launcher-sdd-lifecycle`.
- No ejecutar shell arbitrario ni instalar tools sin una ruta canonica.
- No retirar la TUI.
- No cambiar los primitives de migracion ni las reglas de no-clobber.
- No hacer push, commits remotos ni integraciones externas desde onboarding.

## Acceptance criteria

- [x] Un proyecto nuevo puede recorrerse desde el launcher con preview y
      confirmacion hasta `init` + workspace setup, con registry actualizado.
- [x] Un proyecto existente puede unirse y seleccionarse en el launcher sin
      perder el gate ni la posibilidad de registrar/asignar roles.
- [x] La UI puede iniciar setup y adopcion de spec/SDD/contexto usando los
      comandos existentes y devuelve un preview sin writes cuando no esta
      confirmada.
- [x] La confirmacion produce los mismos archivos y estados que el comando
      CLI correspondiente; no se duplican generadores ni se clobberan archivos.
- [x] Los casos dry-run, rechazo, repo parcialmente Axiom, colisiones y
      rerun idempotente quedan cubiertos y muestran razones legibles.
- [x] Las rutas de spec/contexto, provenance y proyecto seleccionado se
      muestran desde el estado real, no desde valores fabricados por el front.
- [x] La UI refresca registry/doctor y conserva los paneles existentes.
- [x] Build, pruebas focalizadas de onboarding/adoption y E2E disponible
  pasan sin nuevas regresiones.

## Risks

- Hay varias combinaciones de topologia y rutas; un endpoint que escriba en el
  repo equivocado rompe el aislamiento entre proyectos.
- El setup existente tiene gates y no-clobber propios; reimplementar parte en
  el launcher puede crear divergencias sutiles.
- La adopcion puede producir muchos warnings; la UI debe distinguir skip,
  warning y error sin convertir una colision esperable en fallo global.

## Open questions

No hay preguntas bloqueantes. Se usara la forma canonica de `workspace setup`
y `runWorkspaceAdopt`; si una capacidad no tiene wrapper estable, se mostrara
como pendiente en vez de simular ejecucion.

## Assumptions

- El launcher sigue siendo un servidor local estatico con API HTTP y la CLI
  sigue siendo el ejecutor canonico.
- Los paths externos se validan y resuelven antes de mutar; el servidor no
  acepta rutas relativas ambiguas para operaciones de workspace.
- El resultado puede ser parcialmente exitoso cuando un primitive existente
  reporta warnings, siempre que la UI los exponga.

## Implementation notes

El launcher mantiene los endpoints existentes de install/join/roles y agrega
dos operaciones server-level, ambas con el mismo contrato preview/confirm:

- `POST /api/launcher/workspace/setup` recibe `name`, `controlPath`,
  `specPath`, roles, adapters, profile, overlay, providers y `register`.
  `controlPath`, `specPath` y cada path de rol deben ser absolutos. La preview
  sólo inspecciona filesystem; la confirmación construye el spec mediante el
  wrapper CLI existente y delega en `runWorkspaceSetup`.
- `POST /api/launcher/workspace/adopt` recibe el destino absoluto y las
  fuentes opcionales `adoptSpec`, `adoptSdd` e `ingestContext`, además de las
  opciones de setup. La confirmación delega en `runWorkspaceAdopt`, que a su
  vez usa los primitives canónicos de setup, bootstrap legacy y bootstrap de
  contexto. Las fuentes legacy se reportan como read-only y nunca son destino
  de escritura.

Las respuestas incluyen la operación ejecutable o pendiente, paths absolutos
reales, repos creados/existentes, conteos de creados/saltados, conformance,
warnings, provenance reportada por el primitive y las líneas canónicas. La
selección de proyecto sigue leyendo el registry real y se refresca después de
install/join/setup/adopt; no se inventa un estado local del front. Las
opciones existentes de adapters, execution mode y tools conservan sus notas de
honestidad: tools continúa siendo una selección pendiente mientras no exista
un instalador canónico.

Durante la review independiente se añadieron guards antes de cualquier
adopción: un destino con `axiom.yaml` de otro proyecto o con identidad no
determinable devuelve `409` y no migra; los paths iguales o anidados entre
control, spec y roles devuelven `400`. La adopción parcialmente exitosa
conserva HTTP 200, `executed: true`, `partial: true` y el resultado completo
para que el front muestre warnings, paths, provenance y elementos omitidos.
El setup ya no publica un `skipped.files` sintético; solo expone datos
respaldados por el primitive canónico.

El candidato congelado antes del apply fue
`45150e80579291001560fd8f56c343fc9a0388f9262093a50f613015a2729504`; ese
hash es histórico porque el README cambió después para documentar el contrato
y las correcciones de review. El cierre gobernado usa el freeze final emitido
sobre esta versión.

## Validation

Ejecutado desde `Axiom/`:

- `npm run build` — verde (`tsc -b`).
- `npx vitest run apps/cli/tests/launcher-onboarding-migration.test.ts
  apps/cli/tests/app-launcher.test.ts
  apps/cli/tests/launcher-onboarding.test.ts apps/cli/tests/bootstrap.test.ts
  apps/cli/tests/bootstrap-from-context/command.test.ts
  apps/cli/tests/bootstrap-from-context/idempotency.test.ts
  apps/cli/tests/workspace-adopt.test.ts
  apps/cli/tests/e2e/workspace-adopt.e2e.test.ts` — 7 archivos, 90 tests
  verdes en la revalidacion final; la suite de primitives de adopcion incluida
  en la evidencia previa tambien permanece verde.
- Casos HTTP nuevos: `dryRun` con confirmacion no escribe, paths de roles
  solapados devuelven `400` sin mutacion y destino foraneo devuelve `409` sin
  migracion.
- `node --check apps/cli/static/launcher/launcher.js` y `node --check
  apps/cli/static/launcher/transport.js` — ambos verdes.
- `get_errors` sobre `app-onboarding.ts` y `app-api.ts` — sin errores.

La review independiente inicial encontró tres fallos críticos y tres warnings:
adopción sobre destino foráneo, paths solapados, pérdida de resultados
parciales, selección sin registry real y un campo `skipped.files` no sustentado.
Todos fueron corregidos en el mismo slice y la re-review N=1 los marcó
`resolved` o `verified`. El riesgo residual es de profundidad de pruebas: no
hay todavía un fixture de fallo tardío para render parcial en navegador ni un
fixture separado de `axiom.yaml` sin identidad; ambas ramas tienen guards
directos y validación de build/sintaxis.

## Result

Implementado y validado. Se añadieron endpoints server-level para setup y
adopción, formularios del launcher para paths/roles/configuración, transporte
HTTP y pruebas de contrato. Los wrappers delegan en
`buildWorkspaceSetupSpecFromFlags`, `runWorkspaceSetup` y `runWorkspaceAdopt`;
las fuentes legacy se mantienen read-only y los resultados exponen paths,
created/skipped, warnings, conformance, provenance y registry. Los guards de
ownership y solapamiento evitan escribir sobre destinos inseguros, y el front
conserva resultados parciales sin convertirlos en un error opaco.

Estado: `closed`. Implementación, validación, review independiente, freeze,
receipt e integración canónica completados; la carpeta queda lista para el
archivado físico del lote.

## General spec integration

Integrado en `specs/00..08` y `context/operations/01-instalacion-y-onboarding.md`:
launcher como interfaz guiada, endpoints de setup/adopt, preview/confirmacion,
guards de ownership/overlap y resultado parcial con provenance.
