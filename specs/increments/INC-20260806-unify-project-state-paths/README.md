# Unify project runtime state paths

> **Código**: INC-20260806-unify-project-state-paths
> **Estado**: Cerrado pendiente de archivado
> **Fecha de creación**: 2026-08-06
> **Tipo de cambio**: normalizar persistencia e aislamiento de estado

## Resumen

El estado runtime ligado a un proyecto debe tener un solo hogar fisico:
`.axiom-state/<projectKey>/`. Se elimina la bifurcacion entre ese directorio
y `.axiom-state/config/<projectName>/`, y se deja `.axiom-state/local/`
exclusivamente para datos realmente locales al repo u operador.

## Contexto y motivación

R-05 encontro tres convenciones que se solapan: escritores directos bajo
`.axiom-state/<projectName>/`, el `FilesystemStore` bajo
`.axiom-state/<scope>/<projectName>/`, y memoria/workflow/bindings MCP bajo
`.axiom-state/local/` aunque exigen aislamiento por proyecto. Esto obliga a
cada consumidor a conocer excepciones, permite duplicados y hace que el nombre
`config` describa una ruta fisica que no coincide con `axiom.config/`, que es
la configuracion declarativa real.

La unificacion conserva la carpeta project-scoped ya usada por lifecycle y
reduce la superficie de migracion: el nombre canonico de maquina sera
`projectKey`, derivado de `projectId` en v2 y de un slug estable de la
identidad v1. La lectura acepta rutas antiguas durante la migracion, pero los
writers nuevos solo escriben en la ruta canonica.

## Alcance

### Incluido

- Crear una unica resolucion de `projectKey` y helpers de rutas compartidos.
- Mapear el scope `config` del `FilesystemStore` a la raiz
	`.axiom-state/<projectKey>/`, sin crear el segmento fisico `config`.
- Mantener subdirectorios solo para separar familias que colisionarian:
	`memory/`, `mcp/`, `outputs/` y `executions/<executionId>/`.
- Mover escritores y lectores project-bound de init/profile, members,
	workspace, start/sync, toolchain, tracker, plugins, skills, checkpoints,
	managed-state, model assignments, components, workflow, MCP bindings y
	memoria a la convencion comun.
- Reservar `.axiom-state/local/` para audit trail y sidecar, topology
	bindings y overrides locales; documentar esa frontera y asegurar que no se
	use para datos project-bound.
- Leer estados legacy desde las rutas antiguas con precedencia determinista,
	migrarlos de forma atomica e idempotente y evitar doble escritura.
- Actualizar tests, doctor, worktree provisioning/harvest y documentacion
	activa para las rutas reales.

### Excluido

- Cambiar `axiom.config/`, `axiom.spec/` o el registro user-level `~/.axiom/`.
- Meter `executions/<executionId>/` dentro del projectKey; su aislamiento
	por ejecucion y su lifecycle son una frontera independiente.
- Eliminar el audit trail, memoria JSON/Engram, MCP brokers o checkpoints.
- Introducir una base de datos, un indice global o un sistema empresarial de
	persistencia.
- Reescribir historia archivada; solo se migran estados runtime presentes.

## Documentos del incremento

- `Axiom/packages/filesystem-truth/src/{discovery,path-canonical}.ts`
- `Axiom/packages/core/src/paths.ts`
- `Axiom/packages/persistence/src/{types,isolation,filesystem-store}.ts`
- `Axiom/packages/{installer,versioning,workflow,memory,topology,components,model-routing}/src/`
- `Axiom/packages/cli-commands/src/commands/`
- `Axiom/apps/cli/src/commands/`
- `Axiom.Spec/specs/03_Modelo_Operativo_y_Datos.md` y contexto tecnico,
  solo en la integracion final.

## Dudas abiertas

Ninguna bloqueante. Se fija el destino canonico en
`.axiom-state/<projectKey>/`; la compatibilidad se resuelve con lectura de
legacy y migracion perezosa, no con dos writers activos.

## Decisiones funcionales cerradas

- `projectKey` es el identificador de maquina del proyecto: `projectId` v2;
	para v1, slug estable de `project.name` mediante el helper canonico de
	identidad. Nunca se usa texto humano sin normalizar como segmento nuevo.
- `axiom.config/` sigue siendo declarativo y no es un namespace de estado.
- `.axiom-state/local/` no contiene datos que deban separarse por proyecto.
- La ruta nueva tiene precedencia; las rutas legacy solo sirven para lectura
	y migracion, y una migracion exitosa no deja dos copias activas.
- `config` puede seguir existiendo como nombre de API si es necesario para
	compatibilidad, pero no como segmento fisico obligatorio.

## Consolidación en la spec general

La integracion final debe sustituir la tabla de rutas activa por el namespace
unico project-scoped, distinguirlo de `local`, `executions`, `axiom.config/`
y `~/.axiom/`, y retirar las afirmaciones de que `config/<projectName>` es una
familia fisica separada.

## Estrategia E2E

- Crear un proyecto v1 y otro v2, ejecutar init/configure/start/sync y
	comprobar que los archivos nuevos terminan bajo el mismo projectKey.
- Escribir fixtures legacy bajo `<projectName>/`, `config/<projectName>/` y
	`local/`, leerlos y verificar precedencia, migracion atomica e idempotencia.
- Confirmar aislamiento entre dos projectKeys y entre dos executionIds.
- Verificar memoria, workflow, MCP bindings, managed-state, components,
	model-routing, checkpoints y audit trail sin cruces de proyecto.
- Ejecutar build, doctor, readiness y las suites focalizadas de todos los
	paquetes afectados.

## Trazabilidad y fuentes

- Accion de auditoria: `ACC-023`.
- Hallazgos R-05: coexistencia de rutas con y sin `config` y uso incorrecto
	de `local` para estado project-bound.
- Fuentes: `FilesystemStore`, `resolveScopeDir`, `memoryFilePath`,
	`WORKFLOW_STATE_REL_PATH`, checkpoint/toolchain/installer y los lectores de
	CLI enumerados arriba.

## Implementation notes

La aplicacion de ACC-023 se limita al runtime de `Axiom`. La implementacion
introducira un helper compartido para derivar `projectKey` y construir el
namespace project-scoped, mantendra `.axiom-state/local/` para estado
repo/operador-local, y resolvera rutas legacy solo durante lectura o
migracion. La precedencia sera: namespace canonico, directorio legacy directo,
directorio legacy `config` y, unicamente para archivos conocidos, la ruta
legacy bajo `local`; una copia elegida se trasladara con escritura atomica y
la fuente legacy se retirara solo despues de confirmar el destino.

## Estado de validación humana

Implementación ejecutada y revisada independientemente. Los tres blockers de
runtime quedaron resueltos y la documentación se actualizó; el freeze y los
receipts finales se regeneran sobre esta versión antes del archivado.

## Acceptance criteria

- [x] Existe una sola regla de rutas project-scoped y un helper compartido para
			derivar `projectKey` y paths.
- [x] Ningun writer nuevo usa `.axiom-state/config/<projectName>/`.
- [x] Memoria, workflow y MCP bindings dejan de vivir bajo `local`.
- [x] `local` queda limitado a datos repo/operador-locales.
- [x] Lectura y migracion legacy son deterministas, atomicas, idempotentes y
			no mantienen doble escritura.
- [x] Dos proyectos y dos ejecuciones no comparten rutas.
- [x] Build, doctor, readiness y pruebas focalizadas pasan.
- [x] La documentacion activa coincide con las rutas emitidas.
- [x] Review independiente, freeze final, receipts e integración canónica
			quedan conservados junto al incremento.

## Risks and mitigations

- El alcance atraviesa muchos paquetes. Se debe introducir primero el helper
	de identidad/rutas y migrar por familias, manteniendo pruebas de cada writer.
- Proyectos v2 pueden tener `name` humano distinto de `projectId`. Se prueba
	que el nuevo path usa `projectId` y que la lectura legacy no elige una copia
	ambigua silenciosamente.
- El cambio de workflow/memoria puede afectar repos de roles. Se prueba cada
	repo con su propia raiz y se mantiene local-only solo para datos personales.
- Los artefactos generados no se editan manualmente; se recompila para
	actualizar `dist`.

## Validation

- `npx vitest run packages/filesystem-truth/tests packages/project-resolution/tests packages/persistence/tests packages/workflow/tests packages/memory/tests packages/installer/tests packages/components/tests packages/model-routing/tests packages/versioning/tests packages/topology/tests apps/cli/tests/init.test.ts apps/cli/tests/join.test.ts apps/cli/tests/start.test.ts apps/cli/tests/sync.test.ts apps/cli/tests/audit.test.ts apps/cli/tests/workspace-command.test.ts`: 63 archivos y 746 tests pasaron.
- `npx vitest run packages/isolation/tests/p0.test.ts packages/isolation/tests/execution-paths.test.ts packages/capability-model/tests/resolver.test.ts packages/doctor/tests/capability-model.test.ts`: 4 archivos y 57 tests pasaron.
- `npx vitest run packages/filesystem-truth/tests packages/project-resolution/tests packages/persistence/tests/filesystem-store.test.ts packages/isolation/tests packages/memory/tests/memory.test.ts packages/workflow/tests/state-store.test.ts packages/model-routing/tests/assignments.test.ts packages/model-routing/tests/mutate.test.ts`: 12 archivos y 162 tests pasaron.
- `npm run build`: OK (`tsc -b`).
- `npm run doctor`: PASS, 45/60 OK, 0 fallos, 4 advertencias y 11 omitidos.
- `npm run readiness:first-project`: PASS; `last-sync.json` se verifica ahora
	bajo `.axiom-state/<projectKey>/`.
- Se actualizo `sync` para pasar los nombres legacy al lector migratorio y se
	actualizaron las expectativas de `model-routing`, upgrade, rollback y
	workflow-state a la ruta canonica.
- Se corrijo `@axiom/isolation` para construir rutas desde `projectKey`; su
	suite y los consumidores directos pasan.
- La suite enfocada posterior a la review cubre 52 tests de checkpoints,
			 toolchain y worktree; la suite secundaria de doctor/member install cubre
			 100 tests.
- La suite completa llegó a 3324 tests; los siete fallos iniciales eran
			 expectativas que todavía leían `.axiom-state/local/workflow-state.json`
			 y se actualizaron al namespace canónico.
- Checkpoint restore remapea manifests legacy direct/config/scope/local a
			 `.axiom-state/<projectKey>/` y elimina el destino antiguo tras escribir.
- Toolchain detect/probe/repair acepta aliases legacy y `repair` los migra sin
			 dejar markers duplicados.
- Worktree provisioning pasa `execution.projectId` y aliases explícitos al
			 lector de providers.

## Result

Implementacion completada y validada en el runtime. El helper compartido de
`filesystem-truth` deriva el namespace canonico, `config` deja de ser un
segmento fisico, memoria/workflow/MCP bindings migran fuera de `local`, y los
writers nuevos usan la ruta project-scoped. Las rutas legacy se leen con
precedencia determinista y migracion atomica; las copias conflictivas no se
sobrescriben y pueden reportarse mediante warning.

Los hallazgos independientes `REVIEW-002`, `REVIEW-005` y `REVIEW-006` quedan
resueltos; `REVIEW-008` y `REVIEW-009` se resuelven con el freeze final, y
`REVIEW-010` con los artefactos estructurales completados. El incremento queda
`closed` y listo para archivado después de emitir el freeze/receipts finales.

## General spec integration

Integrado en la pasada única del lote en `Axiom.Spec/specs/00..08` y
`Axiom.Spec/context/**`; las afirmaciones activas describen únicamente
`.axiom-state/<projectKey>/`, `.axiom-state/local/` y `executions/` según su
frontera real.
