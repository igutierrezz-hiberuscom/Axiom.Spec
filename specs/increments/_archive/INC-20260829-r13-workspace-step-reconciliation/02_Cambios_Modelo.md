# 02 Cambios de Modelo

`WorkspaceStepDefinition` declara identidad, owner, operation kinds, targets, outputs, prerequisites y executor. `WorkspaceStepResult` proyecta outcomes tipados y warnings derivados.

La matriz de selección define qué steps usa setup y cada granular; no existe lista paralela en cada comando. Los executors reciben un plan prevalidado de E y no vuelven a resolver paths libremente.

`repair` no modifica workspace state salvo metadata operativa estrictamente necesaria; `add` sí puede declarar adapters/providers antes de materializar.
