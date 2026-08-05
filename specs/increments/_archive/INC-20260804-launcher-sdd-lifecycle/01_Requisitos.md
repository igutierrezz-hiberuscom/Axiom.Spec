# Requisitos

## RQ-001. Registry veraz

El registry del launcher debe derivar estado, rutas y relaciones de fuentes
reales y mantener compatibilidad con sus campos actuales. Para cada entrada
debe conservar `id`, `kind`, `title` y `status`, y puede agregar metadata,
paths, workflow y relaciones solamente cuando la fuente exista.

## RQ-002. Cobertura SDD

Las operaciones de incrementos, bugs, planes, roles, validacion y archive que
existan en la CLI deben ser accesibles desde el launcher. El mapping permitido
es el de `03_Criterios_Aceptacion.md`; no se crea una accion para una
transicion ausente en el workflow o para una familia sin wrapper canonico.

## RQ-003. Gates

Preview y confirmacion se aplican a toda mutacion, y los gates canonicos de
freeze, QA, write-scope, aprobacion y archive no se pueden saltar. La preview
no crea carpetas ni cambia `metadata.yml` o `workflow-state.json`; una
validacion read-only puede devolver su resultado sin mutar.

## RQ-004. Delegacion

El launcher no implementa transiciones ni moves propios; usa los run functions
o comandos ya publicados: `runIncrementSubcommand`, `runBugSubcommand`,
`runPlanCreate`, `runPlanApprove`, `runRoleSubcommand`,
`runQaE2eSubcommand`, `runValidateChanges`, `runValidateTransition` y
`runIntegrate` cuando correspondan.

## RQ-005. Relaciones

La UI debe mostrar relaciones solo cuando se puedan resolver desde metadata,
planes, roles, topologia, `workflow-state.json` o paths/ficheros existentes.
Una referencia ausente, ambigua o no resoluble se omite; no se sustituye por
un estado, repo, plan o implementacion supuesto.

## RQ-006. Archive verificable

`increment-archive` y `bug-archive` deben usar la ruta de `runIntegrate`, que
valida la transicion terminal, actualiza el eje de integracion y mueve la
carpeta sin sobreescribir un destino existente. Un error de precondicion o de
move conserva un resultado fallido y no se anuncia como archivado.
