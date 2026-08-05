# Increment: Ciclo SDD completo en el launcher

Status: closed
Date: 2026-08-04
Action: ACC-007 / R-10
Dependencies: INC-20260804-launcher-onboarding-migration

Formal freeze previo, conservado como evidencia histórica:
`axiom freeze --increment INC-20260804-launcher-sdd-lifecycle --force-json`,
hash `d118591cf24dc6303f57edfa8722f16001c07f469e02696467c42effa72f566f`.
El cierre gobernado usa un freeze final y sus receipts sobre esta versión.

## Goal

Completar el launcher como superficie operativa del ciclo SDD: crear y listar
incrementos y bugs, crear y aprobar planes, iniciar/aplicar/completar roles,
ejecutar validaciones, archivar artefactos y mostrar sus relaciones sin
inventar estados ni duplicar la logica de `@axiom/workflow` y los comandos
canonicos.

## Context

El launcher ya expone catalogo de acciones, craft/execute confirmado,
registry de incrementos/bugs/planes y refresco SSE. Sin embargo, el registry
solo devuelve `id`, `title` y `status`; el catalogo no cubre de manera uniforme
validacion y archivado para todas las familias; y el front no presenta una
relacion completa entre spec, metadata, plan, roles, repositorio,
implementacion y estado.

La CLI ya dispone de `axiom-increment`, `axiom-bug`, `axiom-plan`, `axiom-role`,
`axiom validate changes`, `axiom integrate`, `axiom-qa-e2e` y `axiom state`.
El launcher debe ser un thin wrapper sobre esas rutas y reflejar sus errores y
gates, incluido freeze cuando el comando lo exija.

## Scope

- Enriquecer el registry con datos derivados de archivos reales: kind, paths,
  metadata, estado de workflow y relaciones disponibles; campos ausentes se
  omiten o se marcan como no resueltos, nunca se inventan.
- Completar el catalogo de acciones para incrementos, bugs, planes, roles y QA
  con create/refine/specify/plan/approve/start/apply/complete/validate/archive
  donde exista un comando canonico.
- Añadir preview y confirmacion uniformes para todas las acciones mutantes,
  incluyendo validacion y archive; las lecturas permanecen read-only.
- Delegar en `runIncrementSubcommand`, `runBugSubcommand`, `runPlanCreate`,
  `runPlanApprove`, `runRoleSubcommand`, `runValidateChanges`, `runIntegrate`,
  `runQaE2eSubcommand` u otros exports existentes, sin reimplementar transiciones.
- Mostrar resultado, estado siguiente, comandos reales, errores de precondicion,
  freeze/doctor/QA gates y relaciones en la UI.
- Añadir pruebas de ciclo completo y de acciones ilegales o no confirmadas.

## Non-goals

- No crear un nuevo state machine ni estados que no existan en metadata o
  `workflows.yaml`.
- No sustituir la CLI como ejecutor ni eliminar la TUI en este incremento.
- No ejecutar automaticamente roles, validaciones o archive sin confirmacion.
- No introducir integraciones externas; ACC-008 trata plugins.
- No cambiar las reglas de archive, freeze, QA o write-scope existentes.

## Acceptance criteria

- [x] `ACC-001` El registry expone incrementos, bugs y planes con datos reales de
  carpeta, metadata, estado de workflow, paths y relaciones disponibles,
  manteniendo `id`, `kind`, `title` y `status`.
- [x] `ACC-002` El launcher conserva las acciones existentes y expone las rutas
  SDD adicionales que tienen comando canonico: validate transition, validate
  changes y archive/integrate para incrementos y bugs.
- [x] `ACC-003` Toda mutacion tiene preview sin write y confirmacion explicita;
  una peticion no confirmada deja filesystem, metadata y workflow-state sin
  cambios. Las validaciones read-only no se convierten en mutaciones.
- [x] `ACC-004` Las acciones confirmadas delegan en los `run*` existentes y
  devuelven `exitCode`, mensaje, estado anterior/siguiente cuando exista,
  comando real y errores de gate.
- [x] `ACC-005` Archive usa `runIntegrate` y respeta la precondicion de
  transicion terminal, `integration.status`, QA warning, write-scope/
  aprobacion cuando la validacion correspondiente se ejecuta y el move
  atomico; un fallo de move no se presenta como archive exitoso.
- [x] `ACC-006` La UI muestra relaciones entre spec, metadata, plan, roles,
  repositorio e implementacion solo cuando se derivan de metadata, paths,
  workflow-state, topology o ficheros existentes.
- [x] `ACC-007` Un fixture recorre create -> plan -> role -> validate -> archive
  o documenta con precision el gate que impide avanzar; no hay estados ni
  relaciones falsos.
- [x] `ACC-008` Build, pruebas dirigidas de launcher/workflow/increment/bug/
  plan/role/validate/integrate/QA y el E2E disponible pasan, con fallos
  preexistentes clasificados.

## Risks

- Los comandos tienen contratos y estados distintos; un mapping superficial
  puede ejecutar una transicion equivocada o saltarse un gate.
- El archive mueve carpetas y puede dejar registry stale si el refresh no se
  hace despues de la operacion.
- Las relaciones entre artifacts pueden vivir en metadata, plan o topologia y
  no siempre existen; la UI debe tolerar ausencia.

## Open questions

No hay preguntas bloqueantes. Cuando una familia no tenga una transicion
canonica, se conserva como preview informativo o se marca no ejecutable y se
documenta la fuente que lo demuestra.

## Assumptions

- `@axiom/workflow` y los `run*` actuales son la fuente de verdad del estado.
- `axiom integrate`/archive se usara solo donde sus precondiciones y tipo de
  artifact correspondan; no se sustituira por un move directo desde el front.
- La compatibilidad de clientes existentes exige conservar las claves basicas
  del registry y agregar datos de forma aditiva.

## Implementation notes

El mapping operativo verificado para este incremento es:

| Action | Comando o wrapper canonico | Gate/precondicion | Resultado esperado |
| --- | --- | --- | --- |
| `increment-new` | `runIncrementSubcommand(create)` | id/slug segun el comando; `confirmed` | metadata/skeleton y estado inicial |
| `increment-execute` | `runIncrementSubcommand(refine)` | transicion declarada | `fromState`, `toState`, `exitCode` |
| `increment-close` | `runIncrementSubcommand(specify)` | transicion declarada | estado `planned` si el graph lo declara |
| `increment-plan` / `increment-plan-approve` | `runIncrementSubcommand(plan/plan-approve)` | transicion declarada y aprobacion del workflow | estado real y `exitCode` |
| `increment-verify` | `runIncrementSubcommand(verify)` | validacion funcional declarada por el comando | estado real y `exitCode` |
| `increment-change` | no ejecutable desde launcher | `checkCandidateFreeze` vive en el wrapper CLI async; no se salta | preview informativo con fuente del gate |
| `bug-new` | `runBugSubcommand(create)` | id/slug segun el comando; `confirmed` | metadata/skeleton y estado inicial |
| `bug-execute` | `runBugSubcommand(fix-plan)` | transicion declarada | `fromState`, `toState`, `exitCode` |
| `bug-close` | `runBugSubcommand(verify)` | transicion declarada | estado resultante real |
| `plan-new` | `runPlanCreate` | titulo y afinidad de repo spec; `confirmed` | `planId`, metadata y roles derivados |
| `plan-execute` / `plan-close` | `runPlanApprove` | plan identificable, workflow y aprobacion | `plan-approved` o error real |
| `back/front-new` | `runRoleSubcommandAsync(start)` | plan aprobado, topology/role gates y `executionMode` | estado/worktree resultante real |
| `back/front-execute` | `runRoleSubcommand(apply)` | role en estado aplicable | resultado real |
| `back/front-close` | `runRoleSubcommand(complete)` | review/write-scope/role gates existentes | estado resultante real |
| `e2e-new/execute/close` | `runQaE2eSubcommand(start/verify/pass)` | qa lane y workflow declarados | resultado real |
| `e2e-fail` | `runQaE2eSubcommand(fail)` | precondiciones del workflow QA | estado `failed` si el graph lo declara |
| `*-validate` | `runValidateTransition` | workflow, command y estado declarados | read-only; commands legales y estado destino |
| `plan-validate-changes` | `runValidateChanges` | plan existente y `status: plan-approved`; write-scope real | violaciones y `exitCode` |
| `increment-archive` / `bug-archive` | `runIntegrate` | transicion `-> archived`, metadata, QA warning y move | solo exito si integrate devuelve `exitCode: 0` |
| `plan-archive` | no ejecutable | el workflow de plan termina en `plan-approved` y no expone archive/integrate | preview informativo con fuente del motivo |
| `e2e-archive` | `runQaE2eSubcommand(pass)` | precondiciones del workflow QA | resultado real; no se sustituye por move |

La preview muestra el comando y los gates conocidos sin escribir. Las acciones
mutantes exigen `confirmed: true`; las acciones de validacion tambien pasan por
el contrato uniforme del launcher pero sus wrappers solo leen y sus fallos se
devuelven como resultado. `increment-change` no se ejecuta porque el gate de
freeze solo esta aplicado en el wrapper Commander async; exponerlo via el
launcher requeriria duplicar o saltar esa precondicion. El registry deriva relaciones
desde `metadata.links`, `roles.roleFiles`, paths reales, `workflow-state.json`,
`topology.yaml` y ficheros que existan. Si no hay una fuente suficiente, omite
la relacion. Archive usa `runIntegrate`; hace preflight del destino, captura un
snapshot raw y, si el move falla, intenta rollback tipado y luego restauracion
raw. Si ambos mecanismos fallan, devuelve `inconsistent: true` y el launcher
lo muestra como reparacion manual; no se presenta como archive exitoso.

## Validation

- `npm run build`: OK.
- `npx vitest run apps/cli/tests/app-launcher.test.ts`: OK, 25 tests,
  incluyendo ejecucion confirmada de `back-new` y `front-new` con worktree.
- `npx vitest run apps/cli/tests/integrate.test.ts`: OK, 6 tests, incluyendo
  colision de destino y restauracion raw ante fallo del rollback tipado.
- Bateria focalizada de launcher/workflow/increment/bug/plan/role/validate/
  integrate/QA: OK, 12 archivos y 104 tests.
- `npx vitest run apps/cli/tests/e2e/launcher.e2e.test.ts`: OK, 1 test E2E.
- `node --check apps/cli/static/launcher/launcher.js`: OK.
- `get_errors` sobre `app-api.ts`, `app-launcher.ts`, `integrate.ts` y tests
  afectados: sin errores.
- `node --check apps/cli/static/launcher/launcher.js`: OK.
- No se observaron fallos preexistentes en la validacion focalizada.

El freeze formal previo
(`d118591cf24dc6303f57edfa8722f16001c07f469e02696467c42effa72f566f`) queda
como evidencia histórica de la revisión anterior. El freeze final y el
receipt `verify` deben corresponder a esta última versión documental.

## Result

Implementado y validado el contrato SDD del launcher: registry aditivo con
provenance de paths/metadata/workflow/topology, acciones canonicas de
validacion e integracion, confirmed gate uniforme, refresh tras archive, UI de
relaciones, ejecucion async real de worktree para roles y rollback de metadata
con señal estructurada si la recuperacion completa no es posible. La review
independiente inicial y su re-review N=1 quedaron resueltas; el incremento
queda `closed`; freeze, receipt, review e integracion canonica fueron
completados y la carpeta queda lista para archivado.

## General spec integration

Integrado en `specs/00..08` y `context/architecture/03-ciclo-de-vida-cli-y-
orquestacion.md`: registry basado en fuentes reales, acciones SDD delegadas,
executionMode async de worktree, gates de validacion/archive y rollback con
senal `inconsistent` cuando la recuperacion completa no es posible.
